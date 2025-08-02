---
title: "2.1 同步 I/O (Sync I/O)"
weight: 1
---

一次性读完整个 Linux IO 编程接口文档，再进行编程，这也太难了！我们不妨把需求简化到极致：**仅仅读写一次文件**，先不计较任何的并发和性能，写完就可以交差！

### 1 posix 标准接口

对于这种简单需求，posix 标准为我们提供了一些 api 接口。

| 函数名    | 原型                                                                 |
|-----------|----------------------------------------------------------------------|
| `lseek`   | `off_t lseek(int fd, off_t offset, int whence);`                     |
| `write`   | `ssize_t write(int fd, const void *buf, size_t count);`              |
| `read`    | `ssize_t read(int fd, void *buf, size_t count);`                     |
| `pwrite`  | `ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);` |
| `pread`   | `ssize_t pread(int fd, void *buf, size_t count, off_t offset);`      |

其中：
- `fd`: 文件描述符
- `buf`: 数据缓冲区（`void*` 类型）
- `count`: 操作字节数（`size_t` 类型）
- `offset`: 偏移量（`off_t` 类型）
- `whence`: 基准位置（`SEEK_SET`/`SEEK_CUR`/`SEEK_END`）

在 posix 世界中，所有的文件操作，都要打开一个这个文件，得到 int 类型文件描述符 `fd`。为了读写它，有一个偏移游标 `offset`。

我们要么先去 `seek` 到这个 offset，然后读写文件。这期间还要注意线程安全，`seek`+`read/write` 并不是原子的。

要么使用 `pwrite/pread` 在一次调用中原子性地指定 `offset` 完成读写。

| api               | `lseek`                  | `write`/`read`          | `pwrite`/`pread`        |
|--------------------|--------------------------|-------------------------|-------------------------|
| **用途**           | 移动文件指针             | 基础读写操作            | **定位读写**（不移动指针） |
| **POSIX 标准**     | POSIX.1-1988             | POSIX.1-1988            | POSIX.1-2001 (XSI 扩展) |
| **原子性**         | &#x274c; 非原子                | &#x274c; 非原子               | &#x2705; **原子操作**          |
| **线程安全**       | &#x274c; 需额外保证            | &#x274c; 需保证               | &#x2705; **线程安全**          |
| **文件指针影响**   | &#x2705; 修改指针位置          | &#x2705; 读写后指针移动       | &#x274c; **不影响指针位置**    |
| **典型使用场景**   | 随机访问文件             | 顺序读写                | 多线程/多进程并发读写   |

### 2 Code Snippet
```cpp
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main() {
  int fd = open("testfile.txt", O_RDWR | O_CREAT, 0644);
  if (fd == -1) {
    perror("open failed");
    exit(1);
  }

  // 使用write写入数据
  const char *msg1 = "Hello, world!\n";
  write(fd, msg1, strlen(msg1));

  // 使用lseek移动指针并写入
  lseek(fd, 100, SEEK_SET);
  const char *msg2 = "At position 100\n";
  write(fd, msg2, strlen(msg2));

  // 使用pwrite在特定位置写入(不移动指针)
  const char *msg3 = "Written with pwrite at 200\n";
  pwrite(fd, msg3, strlen(msg3), 200);

  // 读取文件内容
  char buffer[256];
  lseek(fd, 0, SEEK_SET); // 回到文件开头

  ssize_t bytes_read;
  while ((bytes_read = read(fd, buffer, sizeof(buffer))) > 0) {
    write(STDOUT_FILENO, buffer, bytes_read);
  }

  // 使用pread从特定位置读取
  printf("\nReading with pread from position 100:\n");
  bytes_read = pread(fd, buffer, sizeof(buffer), 100);
  write(STDOUT_FILENO, buffer, bytes_read);

  close(fd);
  return 0;
}
```

运行一下
```shell
➜  snip git:(master) ✗ g++ -Wall -Wextra -g -o 01 ./01_sync_io.cpp

➜  snip git:(master) ✗ ./01 
Hello, world!
At position 100
Written with pwrite at 200

Reading with pread from position 100:
At position 100
Written with pwrite at 200
```

### 3 什么是“同步”? 为什么关注阻塞？
“同步”，指的是 `Synchronous I/O`，简写为 `Sync I/O`。是指调用的进程这期间会被阻塞 `Block`。我们不得不回顾一下线程模型。

***强如 128 cores 的 Linux Server 其实是个大单片机！***

![](https://static.zdfmc.net/imgs/2025/05/843909d174fb0482.png)
图: 单线程程序被阻塞

我们启动的单线程程序在调用 `write/read` 时，会进入阻塞状态。这期间 CPU 虽然可以调度给其他程序，但我们进程傻傻地（也只能傻傻地）等待这个调用返回。

示例程序的规模完全不需要担心阻塞带来的性能问题。当我们单机需要处理几十万 iops 和 数十 GBps 的流量时，阻塞以及线程上下文切换，对我们系统设计、性能的影响是巨大的。

必须意识到，实际的产品中，我们整个程序除了执行 IO 操作，还需要处理用户请求 socket、执行相关的业务逻辑、编解码等。阻塞会导致该线程强行 “闲置”。为了达到性能要求，榨干现代存储硬件给我们提供的吞吐能力，整个工程必须合理地安排线程工作内容。

因此，使用线程池管理所有阻塞 IO 的模式应运而生。我们将在稍后探索 **同步 IO 的线程池模式**。

### 4 Stream IO
除了 posix read/write 外，还有一种不同角度考虑的 IO 方式，流式 I/O (Stream I/O)。流式 I/O 在低级 IO 接口上构建了一层缓冲区，可以攒一些输入输出后在进行刷盘，减少系统调用和读写次数。

C 语言提供的常用流式 IO 接口如下。

| 函数                | 头文件      | 描述                                                                 |
|---------------------|-------------|----------------------------------------------------------------------|
| `fopen()`          | `<stdio.h>` | 打开文件并关联到 `FILE*` 流对象                                      |
| `fclose()`         | `<stdio.h>` | 关闭流并刷新缓冲区                                                   |
| `fread()`/`fwrite` | `<stdio.h>` | 二进制数据的缓冲读写                                                 |
| `fgets()`/`fputs`  | `<stdio.h>` | 文本行的缓冲读写                                                     |
| `fprintf()`/`fscanf` | `<stdio.h>` | 格式化的缓冲读写                                                    |
| `setbuf()`/`setvbuf` | `<stdio.h>` | 手动控制缓冲区策略                                                   |
| `fflush()`         | `<stdio.h>` | 强制刷新输出缓冲区                                                   |



**构建存储引擎时，开发者更常见希望自己针对需求自行构建刷盘、缓冲策略**，因此较少见到使用 Stream IO 构建存储引擎。其多见于日志系统的存储和读取。因此本篇不会详细介绍。

注意：此处的 Stream I/O 不是指 kernel 提供的 `buffer/cache` 缓存。此处指的是编程语言为我们包装的带缓冲的 I/O 库。比如我们上面 C 语言的 `stdio.h`，Go 语言提供的 `bufio` 包。

### 5 确保数据到达硬盘

LWN 上一篇文章 Ensuring data reaches disk [^1] 向存储系统程序员强调了持久化的认知。

![](https://static.zdfmc.net/imgs/2025/05/6467fda5d41d49bc.png)
图： 数据读写的全链路 [^1]

除了我们应用程序构建的缓存外，还可能经过 Stream IO 库提供的缓冲、Kernel 提供的 Page Cache、存储硬件上的易失/非易失性缓存，最终落在磁盘上。只有数据到达了非易失存储，才能认为数据安全保存。

**一种方式是通过显式调用 `sync` 接口强制刷盘。另一种方式是打开文件时候指定 `O_SYNC` 或 `O_DSYNC`，在文件写入时候会被立即写入稳定存储。**

值得注意的是，以上调用都是在应用程序角度尽可能执行落盘操作。例如 kernel 挂载磁盘时使用了 `nobarrier`，也无法保证磁盘控制器缓存的实际刷新。

性能方面，使用 `O_SYNC`，也代表每次写入都会写入 Kernel Page Cache 后强制落盘，性能预期会有比较大的下降。使用 `fio` 压测磁盘性能时，可以使用 `--sync=1` 观察 sync 写入性能。

例如：

```shell
 fio --name=dsync_test --filename=fio_testfile --size=10G --rw=rw --rwmixread=50 --bs=4096 --ioengine=io_uring  --iodepth=16 --direct=1  --sync=1 --numjobs=8 --runtime=60 --time_based  --group_reporting
```

[^1]: [Ensuring data reaches disk - LWN.net](https://lwn.net/Articles/457667/)
[^2]: [从共识算法开谈 - 硬盘性能的最大几个误解](https://zhuanlan.zhihu.com/p/55658164)
[^3]: [浅谈存储引擎数据结构](https://haobin.work/2024/05/24/%E7%AE%97%E6%B3%95/%E6%B5%85%E8%B0%88%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[^openman]: [open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)
[^youjiali]: [ScyllaDB 学习(六) – disk I/O](https://youjiali1995.github.io/scylladb/disk-io/)
[^rocksdb_issue]: [Fixing mmap performance for RocksDB](https://smalldatum.blogspot.com/2022/06/fixing-mmap-performance-for-rocksdb.html)
[^rocksdb_man]: [RocksDB Wiki - IO](https://github.com/facebook/rocksdb/wiki/IO)