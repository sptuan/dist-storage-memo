---
title: "Linux AIO"
weight: 3
---

`Linux Kernel Asynchronous I/O` 于 2003 年进入内核，`libaio` 是其用户态接口的库封装。

时至今日看，`linux aio` 存在一系列的批评声音[^LWN_aio]。笔者很少见到新的项目选择使用 `linux aio` 了，大部分转而去拥抱 `io_uring`。但不可否认的是，AIO 切切实实为 Linux 磁盘异步 IO 曾经的生产实践。


[^LWN_aio]: [Toward non-blocking asynchronous I/O - LWN.net](https://lwn.net/Articles/724198/)

### 1 异步 I/O 基础

Linux AIO 的基本使用，体现了异步 IO 的基本范式: **提交-执行-收割**。

```mermaid
graph LR
A[提交队列 Submit] --> B[内核执行]
B --> C[完成队列 Completion]
```

一般开发者不会直接调用 syscall，而是使用为用户封装的库 `libaio`。具体来讲，`libaio` 的关键调用如下：

| 阶段           | 系统调用      | 功能                                  |
|----------------|---------------|---------------------------------------|
| 上下文初始化   | io_setup()    | 创建事件队列（最大深度 128-4096）     |
| 准备异步请求  | io_prep_pread/pwrite() | 准备异步 io 请求结构体 |
| I/O 提交       | io_submit()   | 批量提交请求（iocb 结构体）           |
| 事件获取       | io_getevents()| 阻塞/非阻塞获取完成事件               |

其余关键控制函数还有 prepare、cancel 等操作，可参考 `libaio` 文档。


### 2 AIO 的缺陷

LWN.net 2017 年的一篇博客[^LWN_aio]精辟地总结了开发者们对 linux aio 的批评，大致包含以下几点。

#### 2.1 某些条件下意料外的阻塞

虽然我们以异步非阻塞的方式提交了 io，但还要验证其在任何文件系统、内核版本都是以异步方式运行的。

Seastar 甚至推出一个专门的工具[^seastar_detect_aio_block]，通过测量上下文切换次数判断是否内核进行了阻塞 IO。

[^seastar_detect_aio_block]: [Qualifying Filesystems for Seastar and ScyllaDB](https://www.scylladb.com/2016/02/09/qualifying-filesystems/)

这是很多开发者不太能接受的，尤其是用户应用代码在不同的内核、文件系统/块设备、磁盘负载下，表现出不一致的阻塞性。一个纯 async 的进程可能突然在某种条件下 hang 住一段时间，造成性能大幅下降。


#### 2.2 只支持 Direct I/O

Linux AIO 被设计时主要聚焦于避免阻塞的磁盘 I/O，且强制要求使用 `O_DIRECT` 标志。这其实限制了其范围、设计思路、接口定义。

一方面，从应用开发者：只有部分需要使用 AIO 的，比如数据库、存储引擎应用才会去重度使用[^Seastar5]。使用 `O_DIRECT` 需要自己处理一系列的内存对齐、自己构建上层的 buffer 系统[^Seastar5]。

另一方面，从内核开发者：网络读写、甚至任何 syscall 是不是都可以有一套异步接口？Linux AIO 一直也被持续改进，但是被认为还是需要有一个通用的、异步调用 syscall 的机制，而不是扩展已有的 AIO 接口[^LWN_671]。

[^Seastar5]: [How io_uring and eBPF Will Revolutionize Programming in Linux](https://www.scylladb.com/2020/05/05/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/)

[^LWN_671]: [Re: [PATCH 09/13] aio: add support for async openat()](https://lwn.net/Articles/671657/)


### 3 ScyllaDB/Seastar 与 AIO
ScyllaDB/Seastar 重度使用了 linux aio。博客有一系列关于磁盘 io 的文章，非常适合阅读。尤其是 io 调度主题。
- [Database Internals: Working with I/O - ScyllaDB Blog](https://www.scylladb.com/2024/11/25/database-internals-working-with-io/)
- [Top Mistakes with ScyllaDB Storage - ScyllaDB Blog](https://www.scylladb.com/2023/07/17/top-mistakes-with-scylladb-storage/)
- [On-Demand Webinar: Different I/O Access Methods for Linux - What We Chose for ScyllaDB and Why - ScyllaDB Blog](https://www.scylladb.com/webinar/on-demand-webinar-different-i-o-access-methods-for-linux-what-we-chose-for-scylladb-and-why/)
- [What We've Learned After 6 Years of I/O Scheduling - ScyllaDB Blog](https://www.scylladb.com/2021/09/15/what-weve-learned-after-6-years-of-io-scheduling/)
- [How io_uring and eBPF Will Revolutionize Programming in Linux - ScyllaDB Blog](https://www.scylladb.com/2020/05/05/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/)
- [Scylla's New I/O Scheduler - ScyllaDB Blog](https://www.scylladb.com/2021/04/06/scyllas-new-io-scheduler/)
- [I/O Access Methods in Scylla - ScyllaDB Blog](https://www.scylladb.com/2017/10/05/io-access-methods-scylla/)


