---
title: ""
cascade:
  type: docs
toc: false
sidebar:
  open: true
prev: false
next: false
breadcrumbs: false
---

<div align="center">

<h1 class="center-title">分布式存储漫游指南</h1>

<div align="center">

<p align="center">
  <h5 class="center-title"><em>分布式存储技术探索之旅 by SPtuan</em></h5>
</p>

<picture>
  <img src="https://static.zdfmc.net/imgs/2025/10/591fc7cf520b7a51.jpg" alt="分布式存储漫游指南" width="330">
</picture>

</div>

<p align="center">
  <a href="https://github.com/sptuan/dist-storage-memo">
    <img src="https://img.shields.io/github/stars/sptuan/dist-storage-memo?style=for-the-badge&logo=github&color=yellow" alt="GitHub Stars">
  </a>
    <a href="https://github.com/sptuan">
    <img src="https://img.shields.io/badge/GitHub-sptuan-blue?style=flat-square&logo=github" alt="GitHub">
  </a>
  <a href="https://steinslab.io">
    <img src="https://img.shields.io/badge/Blog-steinslab.io-green?style=flat-square&logo=blogger" alt="Blog">
  </a>
  <a href="mailto:sptuan@steinslab.io">
    <img src="https://img.shields.io/badge/Email-sptuan@steinslab.io-red?style=flat-square&logo=gmail" alt="Email">
  </a>
</p>

</div>
本漫游指南从现代数据中心硬件、单机 IO 性能出发，逐步探索分布式的存储产品的现状和技术点。

本电子书是笔者从事分布式存储研发经历的小结。带着曾有的疑惑，从新手村重新出发，带有主观色彩。

笔者在学习过程中，受益于诸多开发者的博客文章和技术讨论，故自己也记录下来，希望能和读者多多交流学习。

## 适合读者

- **分布式系统开发者** - 存储系统架构设计与实现
- **存储系统工程师** - 深入理解存储技术栈
- **学生和研究者** - 对存储技术感兴趣的学习者
- **技术人员** - 希望了解分布式存储架构

## 章节目录

{{< cards >}}
  {{< card link="hardware" title="第一章：硬件" subtitle="2025年了，存储硬件啥样了？" >}}
  {{< card link="sync-io" title="第二章：Sync I/O" subtitle="我应怎样在单机上读写数据？(同步篇)" >}}
  {{< card link="async-io" title="第三章：Async I/O" subtitle="我应怎样在单机上读写数据？(异步篇)" >}}
  {{< card link="dist-101" title="第四章：分布式系统 101" subtitle="复制和分区, 我变复杂了、但也可靠了" >}}
{{< /cards >}}

## 即将推出

更多章节正在撰写中：

{{< cards >}}
  {{< card title="第五章：控制节点篇" subtitle="数据节点的管理、路由与迁移修复" >}}
  {{< card title="第六章：元数据篇" subtitle="元数据服务与垃圾回收 (GC)" >}}
  {{< card title="第七章：协议篇" subtitle="S3 协议, 对象存储的事实标准" >}}
  {{< card title="第八章：容灾篇" subtitle="容灾与跨区异步复制" >}}
  {{< card title="番外篇：CDN" subtitle="其实我也是存储节点" >}}
{{< /cards >}}

## 贡献与反馈

{{< callout type="info" >}}
📖 本书源代码托管在 [GitHub Repo](https://github.com/sptuan/dist-storage-memo)，欢迎 star 和催更！

✍️ 文中的谬误、遗漏信息，欢迎 issue 补充讨论！
{{< /callout >}}

### 如何贡献

- **发现错误** - [提交 Issue](https://github.com/sptuan/dist-storage-memo/issues)
- **内容建议** - [功能请求](https://github.com/sptuan/dist-storage-memo/issues)  
- **内容贡献** - [Pull Request](https://github.com/sptuan/dist-storage-memo/pulls)
- **点赞支持** - [⭐ Star 项目](https://github.com/sptuan/dist-storage-memo)

## 关于作者

**SPtuan** - 一名普通的工程师，最大的愿望是度过平静的时光。

- [GitHub](https://github.com/sptuan)
- [Blog](https://steinslab.io)
- Email: sptuan#steinslab.io (# 替换为 @)

## 读者交流闲聊群

<picture>
  <img src="https://static.zdfmc.net/imgs/2025/11/110b88837543d1e8.png" alt="WeChat" width="300">
</picture>
<picture>
  <img src="https://static.zdfmc.net/imgs/2025/10/0d8fc3f543265714.png" alt="WeChat" width="300">
</picture>

世界很大，圈子很小！
让世界更热闹一些吧 (球球了)！
所有的读者都大大大欢迎！
您的反馈是我前进的动力！
WeChat: dangotech
