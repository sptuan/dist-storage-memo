---
title: "第三章：异步 I/O (Async I/O)"
weight: 3
---

上篇文章中我们探索了同步 IO 的来龙去脉。本篇文章将踏入异步 IO 王国，研究异步编程思想。

本文依旧聚焦于 Linux 平台。总览异步编程，表格如下：

| **技术**         | **核心原理**               | **编程接口**               | **I/O 类型支持**       | **性能特点**                     | **局限性**                                   |
|------------------|--------------------------|--------------------------|-----------------------|--------------------------------|----------------------------------|
| **io_uring**     | 环形队列 + SQ/CQ 无锁队列  | `io_uring_setup`<br>`io_uring_enter`<br>`liburing` 封装库 | **全异步**<br>(read/write, fsync, socket, etc.) | <br>• 零拷贝设计<br>• 批处理提交/完成<br>• 内核旁路优化 | 需 Linux 5.1+<br>API 较复杂需封装          |
| **epoll**        | 就绪事件通知 (红黑树+链表) | `epoll_create`<br>`epoll_ctl`<br>`epoll_wait` | **网络 I/O 为主**<br>(支持 socket, pipe) | <br>• O(1) 事件就绪检测<br>• 水平/边缘触发  | **仅支持文件描述符就绪通知**<br>磁盘 I/O 需配合线程池 |
| **libaio**       | 原生内核 AIO              | `io_setup`<br>`io_submit`<br>`io_getevents` | **Direct I/O 磁盘**<br>(O_DIRECT) | <br>• 真异步磁盘 I/O<br>• 但存在阻塞点    | 仅支持 O_DIRECT<br>部分操作非异步 (fsync)<br>存在内存对齐限制 |
| **POSIX AIO**    | 用户态线程模拟             | `aio_read`<br>`aio_write`<br>`aio_error` | 文件/网络 (假异步)    | <br>• 实质是线程池包装<br>• 上下文切换开销大   | **性能差**<br>不适用于高性能存储             |

其中，`POSIX AIO` 笔者很少见到使用。`epoll` 多见于网络 socket 的异步处理，不能处理 disk io。

鉴于本文为写于 2025 年的存储小品文，kernel 早已经推进至 `6.15`, 故将重点放于 `libaio` 和 `io_uring`。



{{< section-cards >}}