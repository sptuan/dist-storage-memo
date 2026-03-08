---
title: "附件：分布式存储知识地图"
weight: 900
description: "分布式存储知识地图/技能地图：从物理层、单机 I/O 到分布式与系统设计的高频技术总览。"
---


根据笔者日常工作听到的、实际用到的高频技术总结而成一副知识地图/技能地图。

这张地图，也是我们 “分布式存储漫游指南” 的世界地图。

注：笔者委托画师朋友手工绘制这个地图。请大家尽情享受最后这几年人工创作的日子吧！

![](https://static.zdfmc.net/imgs/2026/dist-stroage-memo-map_v0.1.0-lite.jpg)
*手工绘制不断更新中，您可以在[此链接](https://github.com/sptuan/dist-storage-memo/blob/master/static/dist-stroage-memo-map_v0.1.0-full.png)获取高清电子版。




## 🗺️ The Map of Distributed Storage


### 1. 物理层与硬件 (Physical & Hardware)
> 这是所有数据的最终归宿，物理特性的限制决定了上层软件的设计。

#### **存储介质**
* **HDD**: 机械臂寻道 (**Seek latency**), 顺序写优势, **SMR** (叠瓦式) vs **CMR**。
* **SSD**: **NAND Flash**, **FTL** (闪存转换层), 写入放大 (**Write Amplification**), **GC** (垃圾回收)。
* **New Tech**: **NVMe/ZNS** (分区块存储), **PMEM** (持久内存), **Open-Channel SSD**。

#### **网络硬件**
* **NIC** (网卡), **Switch** (交换机), **RDMA** (**InfiniBand/RoCE**)。

---

### 2. Linux 内核与 OS 技能 (Linux Kernel & OS)
> 这是存储开发者必须深耕的领域，理解 OS 才能榨干硬件性能。

#### **内存管理 (Memory)**
* **Page Cache**: Buffered I/O 的核心，脏页回写 (**Writeback**), **pdflush**。
* **NUMA**: 非一致性内存访问，跨 Socket 访问的性能陷阱 (**CPU Affinity/Binding**)。
* **Allocators**: **tcmalloc** / **jemalloc** (减少内存碎片和锁竞争)。

#### **并发与调度**
* **Context Switches**: 上下文切换的成本（自旋锁 vs 互斥锁）。
* **Thread Model**: 用户态线程 (**Goroutine**) vs 内核级线程。

#### **观测与旁路**
* **eBPF**: 现代内核无侵入观测神技。
* **Kernel Bypass**: **SPDK**, **DPDK** (绕过内核直接操作硬件，极致减低延迟)。

---

### 3. 本地 I/O 栈 (Local I/O Stack)
> 数据从用户态内存流向物理介质的必经之路。

#### **VFS (虚拟文件系统)**
* **Inode / Dentry**: 文件句柄与元数据查找过程（海量小文件的性能瓶颈）。
* **Filesystems**: **Ext4** (日志机制), **XFS** (大文件并行性能), **Btrfs/ZFS** (**CoW**, 快照, 校验和)。

#### **I/O 模式**
* **Buffered I/O**: 标准 `read`/`write`（依赖 Page Cache）。
* **Direct I/O**: `O_DIRECT`, 绕过 Page Cache（数据库/自研引擎常用）。
* **Memory Mapped**: **mmap** (Zero-copy read)。

#### **高性能/异步接口**
* **io_uring**: (现代高性能标准) **Submission/Completion Queue**, **Polling mode**。
* **Linux AIO**: (旧时代的异步接口，限制较多)。

---

### 4. 单机存储引擎 (Local Storage Engine)
> 如何在一个节点上高效地组织数据。关键机制 WAL 和 Checkpoint 归位于此。

#### **数据结构**
* **LSM-Tree**: (RocksDB/LevelDB) 适合写多读少，涉及 **Compaction**, **Bloom Filter**。
* **B+ Tree**: (MySQL/BoltDB) 适合读多写少，涉及 **Page Split**, **B-Tree 变体**。
* **Log-Structured**: (Bitcask) **Append-only**, **Hash Index**。

#### **可靠性维持机制 (Durability)**
* **WAL (Write-Ahead Log)**: 顺序写盘保平安，所有状态变更先写日志，**Crash Recovery** 的基础。
* **Checkpoint / Snapshot**: 定期将内存状态持久化，缩短故障恢复时间，常结合 **CoW** 技术。

---

### 5. 分布式理论与协调 (Theory & Coordination)
> 让一堆不可靠的机器像一台机器一样工作。

#### **一致性 (Consistency)**
* **Consistency Models**: **Linearizability** (线性一致性), **Sequential**, **Eventual** (最终一致性)。
* **理论基石**: **CAP 定理**, **BASE**, **PACELC** (延迟与一致性的权衡)。

#### **共识算法 (Consensus)**
* **Paxos**: 分布式共识的基石。
* **Raft**: 工业界最流行的实现（Etcd, TiKV）。
* **ZAB**: Zookeeper 使用的原子广播协议。

#### **时钟、顺序与检测**
* **Clock**: **Logical Clocks** (Lamport, Vector Clocks), **TrueTime** (原子钟)。
* **Coordination**: **Failure Detection** (心跳检测), **Lease** (租约机制防止脑裂), **Phi Accrual Detector**。
* **Leader Election**: 选主机制。

---

### 6. 数据分布与元数据 (Distribution & Metadata)
> 决定数据切分到哪里，以及如何保证不丢。

#### **分片策略 (Sharding)**
* **Partitioning**: **Range Sharding** (按范围), **Hash Sharding**。
* **Consistent Hashing**: (一致性哈希 & 虚拟节点) 解决节点增减时的数据迁移问题。

#### **冗余策略 (Redundancy)**
* **Replication**: **Multi-Raft**, **Chain Replication**, **Primary-Backup**。
* **Erasure Coding (EC)**: **RS Codes**, **LRC** (局部重构码，降低跨机柜修复带宽)。

#### **元数据管理**
* **Management**: **Centralized** (HDFS NameNode), **Decentralized** (DHT/Kademlia), **Federation**。

---

### 7. 架构形态 (Architecture & API)
> 用户看到的系统形态与访问协议。

#### **接口形态**
* **Block (块存储)**: **Ceph**, **OpenEBS**。
* **File (文件存储)**: **NFS**, **CephFS**, **JuiceFS** (符合 POSIX 标准)。
* **Object (对象存储)**: **S3 API** (事实标准), **MinIO**。

#### **系统架构演进**
* **Shared-Nothing**: 传统分布式架构。
* **Disaggregated Storage (存算分离)**: 云原生大趋势，计算层无状态，存储层独立扩展。

---

### 8. 工程化与可靠性 (Engineering & Reliability)
> 开发者的工具箱，确保系统在混沌的环境中依然正确。

#### **正确性验证**
* **Formal Methods**: **TLA+** (形式化验证复杂算法逻辑)。
* **Testing**: **Jepsen Test** (分布式一致性黑盒测试)。

#### **混沌工程与观测**
* **Chaos Engineering**: **Fault Injection** (故障注入), **Chaos Mesh**。
* **Observability**: **Distributed Tracing** (OpenTelemetry), **Tail Latency Analysis** (P99/P999 延迟分析)。
