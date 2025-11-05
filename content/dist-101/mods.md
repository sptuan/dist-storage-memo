---
title: "4.5 元数据与存储引擎"
weight: 5
---

{{< callout type="info" >}}
✋🏻😭✋🏻 本小节编辑中 ✍️✍️✍️
{{< /callout >}}

## 前言

本文围绕分布式存储（块存储/对象存储/文件存储）话题的两大组件（元数据 & 存储引擎）展开，力求每一条论述都有实际系统案例作为支撑。

将规律抽象出来是困难的。尤其是超越了不同类型的分布式系统(数据库/存储/消息队列)，在上层总结出的理论和经验。这也是我们初看《DDIA》这本书时一头雾水的原因 —— **一旦超出了我们认知的具体用例，仿佛读到了什么，又仿佛什么都没读到**。我们一定事先明确研究内容和背景。

其他背景(比如数据库)的读者，也可作为综述参考不同的设计视角。

## 元数据与存储引擎

元数据和存储引擎的分层设计，常见于对象存储和文件存储产品形态中。

大型分布式对象/文件集群需要动辄支撑千亿、万亿级别的元数据，一个重要的流派就是元数据管理逐渐独立出来，作为并行于存储引擎的组件。甚至直接使用 NoSQL 数据库，以得到近似线性的扩容支撑能力。

有综述论文[^meta_overview]描述了元数据系统的发展里程碑：

[^meta_overview]: H. Dai, Y. Wang, K. B. Kent, L. Zeng and C. Xu, "[The State of the Art of Metadata Managements in Large-Scale Distributed File Systems — Scalability, Performance and Availability](https://ieeexplore.ieee.org/document/9768784)" in IEEE Transactions on Parallel and Distributed Systems, vol. 33, no. 12, pp. 3850-3869, 1 Dec. 2022, doi: 10.1109/TPDS.2022.3170574.

![](https://static.zdfmc.net/imgs/2025/11/0eae61fc3b510d9e.png)

而块存储主要为用户提供块设备的线性空间，元数据管理的量级较小，一般使用单组高可用共识节点即可管理。


{{< callout type="info" >}}
✋🏻😭✋🏻 本小节仍然在编辑中，请稍后回来 ✍️✍️✍️
{{< /callout >}}