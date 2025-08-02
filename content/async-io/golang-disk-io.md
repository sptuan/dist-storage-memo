---
title: "3.4 Go 的磁盘 IO"
weight: 5
---

Go 的程序员视角看起来，无论收发网络请求，还是读写磁盘，runtime 已经包装成异步形式，自然地使用协程视角去统一处理，不必担心阻塞问题。既然我们已经理解磁盘 I/O 和网络 I/O 在系统调用上的不同，以及应对阻塞、提高效率的方法。那么 Go 语言是如何处理的呢？磁盘 IO 使用了 linux 的异步技术吗？

### 1 GMP 模型与磁盘 I/O 的交互
1. G（Goroutine）：
表示一个Go程序的用户级线程，它包含了一个程序计数器和栈等信息。
2. P（Processor）：
代表一个逻辑处理器，负责调度和执行goroutine。每个P关联一个goroutine队列。
3. M（Machine）：
代表一个操作系统线程，负责实际的执行。一个M可以绑定一个P。

**磁盘I/O流程**：

1. 当某个 goroutine 发起磁盘读写操作时，该 goroutine 会被分配到系统线程（M）上执行。由于同步 I/O 操作具有阻塞特性，会导致当前 M 进入阻塞状态。

2. 为避免单个 M 的阻塞影响处理器（P）及其管理的其他 goroutine 的执行效率，Go 运行时系统会智能地进行资源重组：
    - 在满足特定条件时（如 I/O 完成或超时触发）
    - 运行时会将 P 从阻塞的 M 上解绑

3. 同时，运行时系统会创建新的 M 并与解绑的 P 重新关联，确保该 P 能继续调度执行其他就绪的 goroutine，维持程序的并发性能。

4. 当原始 M 完成 I/O 操作后：
    - 首先尝试重新获取可用的 P 继续执行
    - 若无法立即获取 P，则该 M 会转入空闲状态
    - 被阻塞的 goroutine 在获得执行资源后会被重新调度

原来如此，其实算是一种我们熟悉的模式：**同步 IO 线程池模型**。核心思想是既然阻塞，就专门扔到一个线程池去做，IO 完成后原来的协程继续执行。在用户层面这个操作是 “异步” 的。

这种技术选择，往往和项目启动时的技术栈、可移植性等等很多因素有关，有些 issue[^go_io_uring] 也探讨了 Go io_uring 的可行性和收益。

[^go_io_uring]: [internal/poll: transparently support new linux io_uring interface #31908](https://github.com/golang/go/issues/31908)


### 2 Tokio 的阻塞处理

在 Rust 异步框架 `Tokio` 中，也是类似的思路。提供 `tokio::task::spawn_blocking` 接口，供用户将阻塞操作自行扔进专用线程池，避免阻塞整个 runtime。

```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

async fn write_file() -> std::io::Result<()> {
    // tokio 框架自带的异步文件 IO
    let mut file = File::create("foo.txt").await?;
    file.write_all(b"Hello, Tokio!").await?;

    // 如果必须用同步库（如std::fs）
    tokio::task::spawn_blocking(|| {
        std::fs::write("bar.txt", b"Blocking write").unwrap();
    }).await?;

    Ok(())
}
```