---
title: "8.1 理解 KV Cache：Attention、P/D 分离与 vLLM 的页式显存管理"
weight: 11
date: 2026-05-05
tags: ["LLM", "KV Cache", "Attention", "PagedAttention", "vLLM", "GPU"]
description: "面向存储工程师的 LLM 推理入门：从 Attention 的数学原理出发，逐步拆解 KV Cache 的产生原因、Prefill/Decode 的计算特征差异，以及 vLLM PagedAttention 的页式显存管理。"
---

> 从 Attention 的数学原理出发，逐步拆解 KV Cache 的产生原因、Prefill/Decode 的计算特征差异，以及 vLLM PagedAttention 的页式内存管理。

---

## 1. 从 Attention 到 KV Cache

笔者在深入 vLLM 的 KV Cache Connector 接口时发现，要理解这个接口的设计，得先回答三个问题：KV Cache 从哪来、长什么样、谁在管理、怎么使用。尤其是需要了解大模型推理领域关心什么，以及怎么使用 KV Cache。

### 1.1 Token：文本的最小单元

大语言模型不直接处理文字。输入文本先被切分为 **token**——可以粗略理解为"词片段"。比如"分布式存储"可能被切分为"分布"、"式"、"存储"三个 token。每个 token 被映射为一个固定长度的向量（**embedding**），长度记为 $d_{\text{model}}$，典型值是 4096 或 8192。后续所有计算都在这些向量上进行。

### 1.2 Self-Attention（Decoder-only）：只能往后看

主流 LLM 用的是 **Decoder-only Transformer**：在因果自注意力里，位置 $i$ 只能聚合来自位置 $\leq i$ 的 token。每个 token 仍与允许范围内的所有前序 token 算相关性。

对于输入序列中每个 token 的 embedding 向量 $x$，模型通过三个线性投影矩阵产生三组向量：

$$Q = xW_Q, \quad K = xW_K, \quad V = xW_V$$

- **Q（Query）**：当前 token 的"提问"——"我应该关注谁？"
- **K（Key）**：每个 token 的"索引标签"——"我的特征是什么？"
- **V（Value）**：每个 token 的"数据内容"——"如果你关注我，这是我能提供的信息。"

笔者认为 Q,K,V 不能直接迁移到存储工程师日常重度打交道的 KV 概念（比如 tikv 的 kv）。是大模型推理为出发点描述的 Query-Key-Value。这里可以使用类比理解，但不宜搞混。

因果 mask 相当于**查询只能命中已写入日志的前缀**，不能读到尚未生成的后缀。

**因果掩码** $M$（上三角为 $-\infty$，其余为 $0$）与缩放因子一起进入 softmax。单头情形下：

$$\text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{QK^T}{\sqrt{d_k}} + M\right) V$$

其中 $d_k$ 是 head 维度。$\sqrt{d_k}$ 防止点积随维度增大而整体偏大，把 softmax 推到梯度极小的区域。分三步：

1. **相似度**：$QK^T$ 给出 $n \times n$ 分数矩阵；加上 $M$ 后，非法位置在 softmax 中权重为 $0$。
2. **归一化**：softmax 得到下三角有效的注意力权重。
3. **加权求和**：用权重对 $V$ 做加权平均，得到每个位置的新表示。

因果 mask 同时解释了为什么 Prefill 可以并行处理整段 prompt——并行只在矩阵乘层面，语义上仍由下三角结构保证不向前偷看。这也是 KV Cache 成立的前提：缓存的始终是已出现位置的 K/V。

实际模型用 **多头注意力（MHA）**：将 Q/K/V 分成 $h$ 个 head。许多大模型还用了 **GQA/MQA** 压缩 KV：**MQA** 让所有 Q head 共享一组 K/V；**GQA** 把 Q 分成若干组，每组共享一组 K/V。这样 $H_{kv} < h$，KV 显存与带宽按 $H_{kv}$ 缩放。现代 LLM 通常用位置编码（如 RoPE）把位置信息注入 Q/K；V 本身未必直接旋转，但它来自已经携带位置和前缀上下文信息的 hidden state。关键结论：**缓存里的 K/V 是"某个位置、某个前缀下"的结果，换位置或换前缀复用都会破坏语义，跨节点传输时序必须与 token 位置一致。**

### 1.3 自回归生成：KV Cache 是必要的

LLM 生成文本是自回归的：每次只产出一个 token，将它追加到序列末尾作为下一次的输入。生成第 $t$ 个 token 时，attention 层需要计算当前 token 的 Q 与所有前序 token（位置 1 到 $t-1$）的 K、V 之间的交互。

这会遇到一个问题：前序 token 的 K 和 V 每一步都被重新需要，但它们的值不会改变。在因果模型中，位置 $i$ 的某层 K/V 只依赖位置 $\leq i$ 的前缀上下文和该位置编码，不依赖更后面的 token；一旦前缀固定，后续 decode 步不会反向改写它。如果每步都把历史上所有 token 的 K/V 从头重算一遍投影，对长度累加后这部分工作量是 $O(n^2)$ 量级。

**KV Cache 的做法**：将每一步算出的 K 和 V 缓存起来，之后只算新 token 的 Q/K/V。这样 **K/V 投影在整段生成上的总工作量从 $O(n^2)$ 降为 $O(n)$**（每层、每头意义下）。

注意力本身的计算量随序列变长仍会上升：第 $t$ 步要对长度 $\sim t$ 的历史做注意力，所有 decode 步合起来对注意力的总代价仍是 $O(n^2)$ 量级。KV Cache 省掉的是重复投影和重复 materialize 历史 K/V 的浪费，不是把自注意力的渐进阶一刀切成 $O(n)$。

用存储领域的话说：KV Cache 本质是一个 **append-only log**——每生成一个 token 就追加一条 K/V 记录；decode 时仍要按因果 mask 在历史前缀上做注意力，但不必把旧记录的编码工序重做一遍。没有 KV Cache，长序列生成在时间上是根本不可接受的。

### 1.4 KV Cache 的尺寸

一个实际模型的 KV Cache 大小：

$$\text{KV Cache Size} = 2 \times L \times H_{kv} \times d_k \times S \times \text{sizeof}(\text{dtype})$$

其中：
- $2$：K 和 V 各一份
- $L$：Transformer 层数
- $H_{kv}$：每层的 KV head 数（GQA/MQA 下少于 Q head 数）
- $d_k$：每个 head 的维度
- $S$：序列长度
- $\text{sizeof}(\text{dtype})$：每元素字节数（FP16 = 2 字节，FP8 = 1 字节）

以 **Llama-3 70B** 为例：$L=80$，$H_{kv}=8$（GQA），$d_k=128$，FP16。一个 4096 token 的序列：

$$2 \times 80 \times 8 \times 128 \times 4096 \times 2 = 1{,}342{,}177{,}280 \text{ bytes} = \textbf{1.25 GiB} \approx 1.34 \text{ GB}$$

**单个请求的 KV Cache 就占 1.25 GiB 显存。** 同时服务 32 个请求，仅 KV Cache 就需要约 40 GiB——接近一张 A100 80GB 卡一半的显存容量。

这组数字说清楚了一件事：**KV Cache 管理是推理系统工程的核心问题。** 当 KV Cache 总量超出单节点 GPU 显存时，外部存储和跨节点传输就成了必需品而非可选项。

---

## 2. Prefill 与 Decode：两种截然不同的计算特征

知道了 KV Cache 是什么，下一个问题：它是怎么被生产和消费的？推理过程有两个计算特征完全不同的阶段——正是这种差异，导致了 KV Cache 需要跨节点搬运。

### 2.1 Prefill：批量构建 KV Cache

用户发送一个 prompt（比如 2000 个 token），模型一次性处理所有输入 token，为每一层计算出完整的 K 和 V 向量并写入 KV Cache。这个阶段叫 **Prefill**。

Prefill 的核心运算是矩阵-矩阵乘法（GEMM）：$Q_{[n \times d]} \cdot K_{[n \times d]}^T$，其中 $n$ 是 prompt 长度。特征：

- **大批量计算**：一次处理数千 token，矩阵维度大
- **Compute-bound**：运算量 $O(n^2 \cdot d)$，GPU 算力是瓶颈
- **GPU 利用率高**：GEMM 是 GPU 最擅长的操作，Tensor Core 可以充分利用

用存储类比：**Prefill 就像批量顺序写入**——大量数据一次性写入，吞吐量高，硬件利用率好。

### 2.2 Decode：逐 token 追加

Prefill 完成后，模型进入自回归生成。每一步只计算一个新 token 的 Q 向量，然后与 KV Cache 中所有历史 K/V 做 attention。这个阶段叫 **Decode**。

Decode 的核心运算是两次矩阵-向量乘法（GEMV）（实现上常融合成一个 kernel）：先用 $Q_{[1 \times d]} \cdot K_{[t \times d]}^T$ 得到注意力权重，再用该权重与 $V_{[t \times d]}$ 做加权求和。特征：

- **单 token 计算**：每次只有一行新 Q，算量远小于 Prefill
- **Memory-bound**：瓶颈在于从显存读取历史 K 与 V
- **GPU 利用率低**：大量时间花在等显存读取，算力严重闲置

用算术强度量化一下。以 batch=1、单头的简化模型为例：$QK^T$ 与加权后的 $V$ 两段 GEMV 各约 $2td$ FLOPs，合计 **$4td$ FLOPs**；需从 HBM 读取 K、V 各 $td$ 个元素，FP16 下共 **$4td$ 字节**。算术强度为 $4td / (4td) = 1$ FLOP/byte。A100 的峰值算力 312 TFLOPS 对应显存带宽 2 TB/s，计算-带宽比约 156 FLOP/byte。**Decode 阶段大致只用了 GPU 峰值算力的 $1/156 \approx 0.6\%$ 量级。**

MHA/GQA/MQA、批解码和 kernel 融合会改变这个常数——多个 Q head 共享较少 KV head 时，K/V 读取可被更多 Q 复用；更大的 batch 也能提高占用。但这些优化只是把常数变大，**Decode 的核心瓶颈仍是从 HBM 反复读历史 KV。**

用存储类比：**Decode 就像随机点读**——每次只写入一条记录，但需要读取全部历史数据来计算。

### 2.3 P/D 分离：让不同的硬件做不同的事

Prefill 和 Decode 的硬件需求截然相反：

| 特征 | Prefill | Decode |
|---|---|---|
| 计算模式 | 矩阵-矩阵乘法 (GEMM) | 矩阵-向量乘法 (GEMV) |
| 瓶颈 | 算力 (Compute-bound) | 显存带宽 (Memory-bound) |
| GPU 利用率 | 高 | 低 |
| 延迟要求 | 用户可接受较长首 token 延迟 | 需要极低的每 token 延迟 |

如果两者混跑在同一组 GPU 上，Prefill 的大批量计算会抢占 Decode 的带宽和调度资源，Decode 的小算量又浪费 Prefill 本可利用的算力。**把它们放在一起，谁都跑不爽。**

**Prefill/Decode 分离（P/D Disaggregation）** 的思路：一组 GPU 专门跑 Prefill，另一组专门跑 Decode，各自的硬件配置和调度策略独立优化。

笔者观察到，在各类领域的工程问题上都会有经典的 trade-off。比如存储工程师的经典 trade-off 是 iops vs 带宽的工程平衡。此推理场景可以描述为 GPU 显存带宽 vs 算力的均衡。而手段都是分布式系统经典的几板斧，比如流水线优化(pipeline)、批次操作(batch)等等。

### 2.4 分离带来的核心问题：KV Cache 需要传输

P/D 分离引出一个直接的工程问题：Prefill 节点产出的 KV Cache 必须传输到 Decode 节点才能继续生成。对于一个 70B 模型、4096 token 的请求，这意味着约 1.25 GiB 的数据需要跨节点搬运。

1.25GB 看起来不大不小，但必须考虑严苛的延迟需求。1.25 GiB 的数据在 100 Gbps 的网络下传输需要约 100 毫秒，这在苛刻的 TTFT（首字延迟）SLA 要求下是一个巨大的 overhead。底层网络瓶颈（如 RDMA/InfiniBand vs TCP）使得 KV 序列化的内存拷贝和传输耗时成为 P/D 分离的核心挑战。

跨节点传输只是场景之一。更广义地说，以下需求都要求 KV Cache 能被外部系统管理：

- **CPU Offload**：GPU 显存不足时将 KV Cache 暂存到主机内存
- **跨请求复用**：多个请求共享相同 prompt 前缀的 KV Cache，避免重复 Prefill
- **持久化缓存**：将热点 prompt 的 KV Cache 持久化到 SSD/远端存储，重启后免重算
- **弹性调度**：将 KV Cache 从负载高的节点迁移到空闲节点

这些场景指向同一个抽象：需要一个标准化的接口来 **读取、写入、传输** KV Cache 数据。

---

## 3. PagedAttention 与 Block 管理：给 GPU 显存装上页表

知道了 KV Cache 长什么样、为什么需要传输，下一个问题：vLLM 是怎么在 GPU 显存里管理这些数据的？

### 3.1 类比页式内存管理

朴素做法是为每个请求预分配一段连续显存存放 KV Cache，但这会带来严重的内存碎片——就像早期操作系统的连续内存分配。vLLM 的解决方案借用了操作系统最经典的思想：**页式虚拟内存。**

vLLM 的 PagedAttention 与操作系统虚拟内存管理几乎一一对应：

| 操作系统概念 | vLLM 对应概念 | 含义 |
|---|---|---|
| 页帧（Page Frame） | **Block** | GPU 显存中固定大小的物理存储单元 |
| 页表（Page Table） | **Block Table** | 将逻辑位置映射到物理 block 的映射表 |
| 页内偏移（Offset） | **Slot 偏移** | block 内部的 token 位置 |
| 页大小 | **block_size** | 每个 block 容纳的 token 数（默认 16） |
| 虚拟地址 | **Slot 编号** | token 的全局逻辑位置 |

和操作系统一样，这种设计的核心价值是**消除外部碎片**：物理 block 不需要连续，任何空闲 block 都可以分配给任何请求；请求结束后释放的 block 可以立即被其他请求复用，不会出现"显存还有 2GB 空闲但找不到连续大块"的尴尬。内部碎片仍在——每个逻辑块的最后一个物理 block 可能只用到部分 slot——但浪费量有上界，通常每序列不到一个 block。

### 3.2 Block 的物理布局

从分配器视角看，block 是固定大小的物理页，存放 `block_size` 个 token 的 K 和 V 向量。单层、单个 block 在 **HND**（Head-first）布局下的逻辑形状：

```
[2, num_kv_heads, block_size, head_dim]
 │       │            │          │
 │       │            │          └── 每个 head 的维度（如 128）
 │       │            └── block 内的 token 数（如 16）
 │       └── KV head 数量（如 8）
 └── K 和 V 两份
```

其中 `2` 代表 K 和 V 各一份。对于 GQA（$H_{kv}=8$）、$d_k=128$、`block_size=16`、FP16 的模型，单层一个 block 占用：

$$2 \times 8 \times 16 \times 128 \times 2 = 65{,}536 \text{ bytes} = 64 \text{ KB}$$

vLLM 还支持 **NHD** 布局，单 block 逻辑形状为 `[2, block_size, num_kv_heads, head_dim]`，即 token 维度和 head 维度互换。两种布局对外部存储引擎来说不改变 block 的分配和生命周期，但会改变序列化、反序列化和 RDMA 注册后按字节搬运时的维序理解。

### 3.3 Slot 寻址：从逻辑位置到物理地址

对于序列中第 $i$ 个 token（0-indexed），vLLM 通过两级寻址定位：

**第一级**：逻辑 block 编号和 block 内偏移

$$\text{logical\_block\_id} = \lfloor i / \text{block\_size} \rfloor$$
$$\text{token\_offset} = i \mod \text{block\_size}$$

**第二级**：通过 block table 查找物理 block

$$\text{physical\_block\_id} = \text{block\_table}[\text{logical\_block\_id}]$$

最终得到全局 slot 编号：

$$\text{slot} = \text{physical\_block\_id} \times \text{block\_size} + \text{token\_offset}$$

这个 **slot** 是 paged KV buffer 上的逻辑槽位编号（扁平化索引）；要算 GPU 字节偏移还需结合维序、`head_dim`、`num_kv_heads` 与张量的 `stride()`，不能直接当成线性物理地址。

### 3.4 一个具体例子

假设 `block_size=4`（为简化说明），一个序列有 10 个 token（编号 0-9）。Scheduler 为它分配了 3 个物理 block：

| 逻辑 block | 物理 block ID | 包含的 token |
|---|---|---|
| 0 | 7 | token 0, 1, 2, 3 |
| 1 | 3 | token 4, 5, 6, 7 |
| 2 | 12 | token 8, 9, _(空)_, _(空)_ |

该请求的 block table 为 `[7, 3, 12]`。

定位 token 6 的 KV Cache：
- 逻辑 block = $\lfloor 6/4 \rfloor = 1$，偏移 = $6 \mod 4 = 2$
- 物理 block = block_table[1] = 3
- slot = $3 \times 4 + 2 = 14$

对外部 KV Cache 引擎来说，slot 编号 14 指向第 14 个 token 槽位；真正读取这个 token 的某个 head、某个 K/V 分量时，还要结合布局和 tensor stride 计算具体地址。

### 3.5 Block 的生命周期

block 的分配和释放由 Scheduler 管理：

1. **分配**：新请求到来或现有请求需要更多空间时，Scheduler 从空闲池中分配物理 block
2. **使用**：Worker 的 attention kernel 根据 block table 和 slot 编号读写 KV 数据
3. **释放**：请求完成（或被抢占）后，Scheduler 将其占用的 block 归还空闲池

这里有一个对外部存储引擎十分关键的细节：当引擎正在异步保存某个 block 的 KV 数据到外部存储时，这个 block **不能被释放和覆写**。这就是 vLLM 中 `request_finished` 接口返回 `True` 以接管 block 生命周期的原因——外部引擎需要延迟 block 释放，直到数据安全落盘。

### 3.6 对外部 KV Cache 引擎的含义

从外部 KV Cache 引擎的视角，PagedAttention 传递的核心信息：

- **数据不连续**：一个请求的 KV Cache 分散在多个物理 block 中，传输时需要做 gather/scatter
- **寻址靠 block ID**：引擎收到的是一组 block ID，需要用 block ID 定位 tensor 中的物理页，再按布局和 stride 计算页内偏移
- **粒度是 block**：block 是分配、释放、传输的最小单位，一个 block 包含 `block_size` 个 token 的完整 KV 数据

---

## 小结

这三层——数学、计算、内存——勾勒了 KV Cache 从产生到管理的核心链路。

对于正在了解和定制 KV Cache 引擎的存储工程师来说，笔者需要处理的数据格式是分页的 K/V tensor，核心问题是：**如何用存储系统的方法论和领域知识，为 GPU 显存这个最昂贵的存储介质做容量和带宽的定制扩展。**

笔者在梳理这条链路的过程中深切体会到，KV Cache 的每一个设计决策，从分页管理到 P/D 分离，背后都是对硬件极限的精确回应。后续如果有机会，希望能深入研究并理解外部 KV Cache 引擎的具体实现：Connector 怎么在 Scheduler 侧做缓存命中判断、怎么在 Worker 侧做异步 RDMA 传输、`request_finished` 的 block 生命周期接管如何实现。

受限于笔者的背景和水平，文中如有疏漏，希望读者不吝指正。
