---
title: "第二章：同步 I/O (Sync I/O)"
weight: 2
---

在扩展到分布式之前，我们先来弄明白单机 IO 的手段。同步/异步/Poller/线程池，眼花缭乱的名词，是否在故弄玄虚？


<!--more-->
[![](https://steinslab.io/wp-content/uploads/2025/05/io-banner.jpg)](https://steinslab.io/wp-content/uploads/2025/05/io-banner.jpg)

我工作使用的第一门编程语言是 Go，享受了大量原生 Goroutine、GMP 调度器的便利，我以为编程语言天生具备并发能力是一件很自然的事情。

从事分布式存储工作后，发觉有经验的同事在做系统设计时，一定会重点关注 IO 和线程模型。IO 包括用户请求到本机 IO 整个链路。而线程模型和 IO 是否阻塞、负载和性能需求息息相关。必须掌握这些，才能针对不同的用户需求设计出最佳的存储系统。

这也迫使我从系统编程语言的角度重新思考、实践了一些常见的 IO 模式。这个过程反而令我更加理解了 Goroutine 和 Go Runtime 的设计动机、了解了更多 syscall 的基础概念。

本文以小型原型为线索，记录了笔者在此主题上的所见所闻。受限于笔者的经验，本文讨论的 IO 模型以及代码实验限定在 Linux 平台上。如有谬误，感谢读者交流、指正！

Linux IO 可根据以下性质分类[^self]
- 是否被 Kernel Page Cache 缓存（**&#x1f4e6; Buffered/&#x1f3af; Direct**）
- 是否会发生阻塞 (**&#x23f3; Sync/&#x26a1;Aysnc**)

| I/O Type       |&#x23f3; Sync I/O (Blocking)      |&#x26a1; Async I/O (Non-blocking) |
|----------------|---------------------------|----------------------------|
| &#x1f4e6; **Buffered I/O**    | `read()`, `write()`        | `io_uring`, `libaio`        |
| &#x1f3af; **Direct I/O**    | `read()`, `write()` with `O_DIRECT` flag | `io_uring`, `libaio` with `O_DIRECT` flag |

你可以根据自己的需求选择任意一种、甚至多种合适的方式一起使用，比如写时候使用 `Sync + Bufferd I/O`，读的时候使用 `Async + Direct I/O`。

如果你是新手开发者，可以暂时搁置所有 `async` 相关的疑惑。不要急，今天就让我们只在 **同步 IO (Sync I/O)** 的世界四处看看！

[^self]: [我选了几个 emoji 希望能帮助读者理解，但让文章产生一股子 AI 味道 &#x1f44a;&#x1f916;&#x1f525;]()





## 参考资料
[^1]: [Ensuring data reaches disk - LWN.net](https://lwn.net/Articles/457667/)
[^2]: [从共识算法开谈 - 硬盘性能的最大几个误解](https://zhuanlan.zhihu.com/p/55658164)
[^3]: [浅谈存储引擎数据结构](https://haobin.work/2024/05/24/%E7%AE%97%E6%B3%95/%E6%B5%85%E8%B0%88%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[^openman]: [open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)
[^youjiali]: [ScyllaDB 学习(六) – disk I/O](https://youjiali1995.github.io/scylladb/disk-io/)
[^rocksdb_issue]: [Fixing mmap performance for RocksDB](https://smalldatum.blogspot.com/2022/06/fixing-mmap-performance-for-rocksdb.html)
[^rocksdb_man]: [RocksDB Wiki - IO](https://github.com/facebook/rocksdb/wiki/IO)