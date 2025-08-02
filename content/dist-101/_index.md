---
title: "第四章：分布式系统 101"
weight: 4
cascade:
  type: docs
---

让我们正式进入分布式的世界。本章更加偏向于分布式存储开发者关心的话题。相对地，其他系统（如分布式计算、分布式流处理）关心的主题可能略有不同，但总体思想是一致的。

### 为什么需要了解分布式系统理论？

什么是分布式系统？最朴素地理解，就是单机的能力无法性能和容灾要求时，将原本同样的需求扩展到一组节点组成的系统上。一般来讲，分布式系统实现了更大规模的数据处理能力和容灾能力，代价是引入了巨大的复杂性。

为什么要以理论的形式讨论分布式系统？因为我们需要一套理论来回答以下问题

- 系统能否容忍节点失效？如何容忍？代价是什么？
- 数据安全性和用户可用性能否一箭双雕、既要还要？
- 有哪些手段组织数据？优缺点是什么？哪个适合我的系统？
- 是否现有算法实现分布式共识？
- ......

### 设计分布式系统，是在设计什么？
分布式存储系统永远是围绕着两个主题展开的：分区(Parititon) 和复制(Replicate)。

- 撰写分布式系统的技术方案，就是要回答这两个问题。
- 评审分布式系统的技术方案，就是要帮忙思考和推演这两个问题。

### 我是新来的，想总览全局？

笔者推荐先通篇阅读材料[^dist_book]，这本书籍诙谐有趣。

然后阅读数个经典工程论文，以及近年的系统设计论文，对照上书尝试思考开发者问什么要如此设计。比如

- GFS: The Google File System [^GFS]
- Tectonic: Facebook's Tectonic Filesystem: Efficiency from Exascale [^tectonic]

[^GFS]: [The Google File System](https://research.google.com/archive/gfs-sosp2003.pdf)
[^tectonic]: [Facebook's Tectonic Filesystem: Efficiency from Exascale](https://www.usenix.org/system/files/fast21-pan.pdf)

实际工程经验总结的博文也非常宝贵。比如[^dist-notes]。



### 1 EiB 是什么概念？
开发者可能对系统的延迟量级 (下表) 比较熟悉。表格通过对比，展示了 cpu 时间和网络时间的巨大差异。

| **操作**               | **实际延迟** | **相对 CPU 周期 (1周期=0.3ns)** | **CPU 视角的比喻**                     |
|------------------------|-------------|----------------------------------|-----------------------------------|
| **1 CPU 周期**         | 0.3 ns      | **1x**                           | "一瞬间"（CPU 的基本时间单位）     |
| **L1 缓存访问**        | 1 ns        | **3x**                           | "等 3 步，像走完自家客厅"         |
| **L2 缓存访问**        | 4 ns        | **13x**                          | "等 13 步，像走到邻居家拿东西"    |
| **主内存访问 (DRAM)**  | 100 ns      | **333x**                         | "等 5 分钟，像下楼取外卖"         |
| **SSD 随机读取**       | 16 μs       | **53,333x**                      | "等 2 天，像网购快递送达"         |
| **机械硬盘寻道**       | 2 ms        | **6,666,666x**                   | "等 2 年，像移民去另一个国家"     |
| **局域网往返 (ping)**  | 0.5 ms      | **1,666,666x**                   | "等 8 个月，像跨省搬家"           |
| **互联网往返（跨洲）** | 150 ms      | **500,000,000x**                 | "等 500 年，像人类文明更迭"       |

对于分布式存储系统的数据量，我们也可以用表格表示，便于读者感受量级大小。

| **存储层级**               | **典型容量** | **1Gbps 下载时间**       | **时间直观对比**         |
|---------------------------|--------------|--------------------------|------------------------|
| **CPU L3 Cache**          | 64 MB        | 0.51 秒                  | 一次眨眼的时间          |
| **单机内存 (RAM)**      | 256 GB       | 34 分钟                  | 一集电视剧的时间        |
| **单块 SSD**          | 4 TB         | 8.9 小时                 | 一个工作日              |
| **单块 HDD**          | 12 TB        | 26.7 小时                | 一天多                  |
| **单机磁盘**     | 300 TB       | 27.8 天                  | 一次国际旅行            |
| **机架磁盘**      | 3 PB         | 0.76 年                   | 一年中所有工作日        |
| **单数据中心**            | 50 PB        | 12.7 年                   | 工业革命至今的时间      |
| **大型互联网对象存储**    | 5 EB         | 1,268 年               | 相当于从宋朝（南宋）到现在的时间跨度  |
| **全球数据中心总容量**    | 10 ZB        | 2,536,780 年           | 相当于从最早人类祖先（能人）出现至今的时间      |

足以窥见数据规模之巨。

分布式存储虽**不一定非要**应对数 PiB/EiB 的数据。比如块存储更加看重性能，而不会追求单盘扩展到数 PiB。

但如果要处理如此规模的持久化存储，只依靠单机显然是无法满足性能和容灾的要求。


### Tips: 不要小看单机硬件的发展速度

本章虽然讨论分布式存储内容，但还请读者不要小看近年来的硬件发展速度。摩尔定律虽然不生效，但 CPU、内存、磁盘每年的发展速度仍然是惊人的。

比如超融合[^fusion]硬件，虽然对上层软件透明视为单机存储，但底层为全闪阵列存储和虚拟化的计算能力。

[^fusion]: [超融合基础设施 HCI](https://e.huawei.com/cn/products/storage/hci)

被一些云计算唱衰者敬若圭臬的案例，37singal 的下云实践[^37signal]也指出，人们可能轻视了现代数据中心的硬件的性能。

[^37signal]: [The Big Cloud Exit FAQ - 37singal](https://world.hey.com/dhh/the-big-cloud-exit-faq-20274010)

> 早在 2004 年，我们在单核赛扬服务器上推出了 Basecamp，只有 256MB 的 RAM， 使用 7,200 RPM 机械硬盘。
>
> ......
>
> （而现在）每一台 R7625 都包含两个 AMD EPYC 9454 CPU，运行频率为 2.75GHz，具有 48 核/96 线程。这意味着我们将在本地机群中增加近 4,000 个 vCPU！还有荒谬的 7,680 GB RAM！和 384TB 的第 4 代 NVMe 存储！[^37signal_2]

[^37signal_2]: [The hardware we need for our cloud exit has arrived - 37signal](https://world.hey.com/dhh/the-hardware-we-need-for-our-cloud-exit-has-arrived-99d66966)

很多 10 年前需要使用分布式系统设计解决的需求，在今天可能单服务器即可完全满足。

可以确定的是，若**横向扩展单机系统能满足需求，其永远比扩展到分布式要简便、便宜 ^_^**。


### 推荐阅读

笔者阅读后认为帮助很大的阅读材料。

- Distributed systems for fun and profit [^dist_book]
- Notes on Distributed Systems for Young Bloods [^dist-notes]

[^dist_book]: [Distributed systems for fun and profit](https://book.mixu.net/distsys/index.html)
[^dist-notes]: [Notes on Distributed Systems for Young Bloods](https://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/)
