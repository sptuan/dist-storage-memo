---
title: 分布式存储漫游指南 - 首页
cascade:
  type: docs
---

本漫游指南从现代数据中心硬件、单机 IO 性能出发，逐步探索分布式的存储产品的现状和技术点。

本电子书是笔者从事分布式存储研发经历的小结。带着曾有的疑惑，从新手村重新出发，带有主观色彩。

笔者在学习过程中，受益于诸多开发者的博客文章和技术讨论，故自己也记录下来，希望能和读者多多交流学习。

{{< callout type="info" >}}
📖 本书源代码托管在 [GitHub Repo](https://github.com/sptuan/dist-storage-memo)，欢迎 star 和催更！

✍️ 文中的谬误、遗漏信息，欢迎 issue 补充讨论！
{{< /callout >}}

## 章节目录

{{< cards >}}
  {{< card link="hardware" title="第一章：硬件" subtitle="2025年了，存储硬件啥样了？" >}}
  {{< card link="sync-io" title="第二章：Sync I/O" subtitle="我应怎样在单机上读写数据？(同步篇)" >}}
  {{< card link="async-io" title="第三章：Async I/O" subtitle="我应怎样在单机上读写数据？(异步篇)" >}}
  {{< card link="dist-101" title="第四章：分布式系统 101" subtitle="复制和分区, 我变复杂了、但也可靠了" >}}
{{< /cards >}}

## 即将推出

更多章节正在撰写中，敬请期待：

{{< cards >}}
  {{< card title="第五章：控制节点篇" subtitle="数据节点的管理、路由与迁移修复" >}}
  {{< card title="第六章：元数据篇" subtitle="元数据服务与垃圾回收 (GC)" >}}
  {{< card title="第七章：协议篇" subtitle="S3 协议, 对象存储的事实标准" >}}
  {{< card title="第八章：容灾篇" subtitle="容灾与跨区异步复制" >}}
  {{< card title="番外篇：CDN" subtitle="其实我也是存储节点" >}}
{{< /cards >}}