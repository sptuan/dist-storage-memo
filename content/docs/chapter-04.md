
---
title: "第四章：控制节点篇"
weight: 4
draft: true
---

# 分布式存储漫游指南 4: 控制节点 ———— 数据节点的管理、路由与迁移修复

buffer io 和 direct io

direct io 需要对齐，对上层存储引擎设计有什么影响？

对内存 page cache 的两种观点

async 和 sync io
iouring？为什么老登这么喜欢强调异步和同步？

sdk 侧正确/错误的判断 + offset 由谁管理

是否允许文件有空洞

是否支持流(应用层 append 操作)

流的 build

LSM tree 天生的 append-only


RocksDB 移除 mmap 支持
```
    if (options.use_mmap_reads) {
      // Use of mmap for random reads has been removed because it
      // kills performance when storage is fast.
      // Use mmap when virtual address-space is plentiful.
```




## 5 性能专题：充分利用磁盘性能，究竟是榨干什么？

### 5.1 磁盘的三维能力

### 5.2 fsync 性能

**要求数据写入即落盘的存储引擎 (比如写 raft log)应特别关注硬件 fsync 性能**。存储引擎的日志系统一般对 fsync 的性能要求高，因为必须要等待成功落盘持久化才能进行下一步操作。文章 [^2] 提到一个小技巧，可以把 WAL 和其他内容分离。日志专门写在 fsync 性能高的介质（一般这种介质比如傲腾的容量偏小），而其他持久化内容可以写在普通介质上。

有几个比较有趣的现象:
1. 企业级的存储硬件，往往 fsync 性能远远高于消费级硬件[^2]。
2. raid 卡如果提供了非易失性缓存，对小型 io 性能有提升。硬件缓存做了一层写流量的整形。





### 模式： IO 线程池模型
 同步 + 线程池 (Go, C++)

 另一种方式：强行 hook 系统调用

例子： Go 的存储 IO 、 Rust Tokio 的阻塞处理

### Protype4： 异步 aio

Seastar 使用 AIO

Seastar 提供了提交 io 会探测是否阻塞
https://www.scylladb.com/2016/02/09/qualifying-filesystems/

批评 AIO 的行为
https://blog.libtorrent.org/2012/10/asynchronous-disk-io/
https://lwn.net/Articles/724198/

为什么不能用 epoll？
- 文件的永远就绪性质、Linux 的特性

异步到底解决了什么？

io scheduler 是在规划什么？

### Protype5： 异步 iouring

异步编程概述 + io_uring （iouring lord）
https://unixism.net/loti/async_intro.html

iouring 无锁设计

```
1. 无锁化设计（Lock-free）
生产者-消费者模型：环形缓冲区天然适合单生产者/单消费者（SPSC）场景。io_uring 中：

提交队列（SQ）：应用是生产者（提交IO请求），内核是消费者（消费请求）。

完成队列（CQ）：内核是生产者（生成完成事件），应用是消费者（处理完成事件）。

消除锁竞争：通过分离生产者和消费者的指针（如头尾指针），双方无需共享锁，仅需内存屏障（Memory Barrier）保证可见性，大幅降低同步开销。

```


### 模式：Poller、Reactor、Preactor


### 讨论：线程池 vs 异步 IO
Q: 一定要搞异步 io 才是高大上吗？


线程好？纯异步好？
https://unixism.net/2019/04/linux-applications-performance-introduction/




### Protype6： SPDK 用户态 io


### 模式：用户态 Run to Complete

例子？



Q: Go 为什么很少见到用于极低延迟存储?


RDMA 用在了哪里

RoadMap:
先写同步 io


## TODO
异步 IO 的
- raw callback
- 事件驱动模式
- 完成队列模式
- 协程模式

协程调度
bprc 协程调度，cppcoro 调度

MPMC 的考虑？init 变量的代价，使用什么技术无锁？

syscall ，c 与汇编的调用
syscall: 与操作系统的沟通桥梁


syscall ，c 与汇编的调用
syscall: 与操作系统的沟通桥梁

思考：高性能的入手思路
https://rustmagazine.github.io/rust_magazine_2021/chapter_9/rethink-async.html

异步 io 的队列深度

实测
https://kb.blockbridge.com/technote/proxmox-aio-vs-iouring/


seastar 相关博客
https://www.scylladb.com/2024/11/25/database-internals-working-with-io/

https://www.scylladb.com/2023/07/17/top-mistakes-with-scylladb-storage/

https://www.scylladb.com/2016/02/09/qualifying-filesystems/

https://www.scylladb.com/2023/10/02/introducing-database-performance-at-scale-a-free-open-source-book/

https://www.scylladb.com/2021/09/15/what-weve-learned-after-6-years-of-io-scheduling/

https://www.scylladb.com/2021/04/06/scyllas-new-io-scheduler/

https://www.scylladb.com/2022/08/03/implementing-a-new-io-scheduler-algorithm-for-mixed-read-write-workloads/



## 其他
一个 socket 竟然能通过多个进程同时 accept
```
/*
 * This function is the main server loop. It never returns. In a loop, it accepts client
 * connections and calls handle_client() to serve the request. Once the request is served,
 * it closes the client connection and goes back to waiting for a new client connection,
 * calling accept() again.
 * */

void enter_server_loop(int server_socket) {
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    signal(SIGINT, SIG_IGN);
    connect_to_redis_server();

    while (1)
    {
        int client_socket = accept(
                server_socket,
                (struct sockaddr *)&client_addr,
                &client_addr_len);
        if (client_socket == -1)
            fatal_error("accept()");

        handle_client(client_socket);
        close(client_socket);
    }
}
```
