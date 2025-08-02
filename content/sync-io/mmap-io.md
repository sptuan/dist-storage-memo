---
title: "2.3 内存映射 I/O (mmap)"
weight: 3
---

### 1 概念

内存映射 mmap 则是从完全不同的角度处理文件读写：将整个文件透明地映射成一段内存，像操作指针一样去读写这块内存区域。

首先使用 `mmap` 映射打开的文件，随后可以像操作内存一样，直接使用 `strncpy` 之类的操作。最后使用 `munmap` 解除映射并清理资源。

### 2 Code Snippet

```cpp
#include <iostream>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>

int main() {
    const char* filepath = "mmap_example.txt";
    const size_t filesize = 4096;  // 4KB文件大小
    
    // 1. 创建并打开文件
    int fd = open(filepath, O_RDWR | O_CREAT, (mode_t)0600);
    if (fd == -1) {
        perror("Error opening file for writing");
        return 1;
    }
    
    // 2. 调整文件大小
    if (lseek(fd, filesize-1, SEEK_SET) == -1) {
        close(fd);
        perror("Error calling lseek() to stretch the file");
        return 1;
    }
    if (write(fd, "", 1) == -1) {
        close(fd);
        perror("Error writing last byte of the file");
        return 1;
    }
    
    // 3. 将文件映射到内存
    char* map = (char*)mmap(0, filesize, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (map == MAP_FAILED) {
        close(fd);
        perror("Error mmapping the file");
        return 1;
    }
    
    // 4. 写入数据到内存映射区
    const char* text = "Hello, mmap world!";
    strncpy(map, text, strlen(text));
    
    // 5. 从内存映射区读取数据
    std::cout << "Read from mmap: " << map << std::endl;
    
    // 6. 清理
    if (munmap(map, filesize) == -1) {
        close(fd);
        perror("Error un-mmapping the file");
        return 1;
    }
    
    close(fd);
    return 0;
}
```

运行一下

```shell
➜  snip git:(master) ✗ g++ -Wall -Wextra -g -o 03 ./03_mmap.cpp     
➜  snip git:(master) ✗ ./03 
Read from mmap: Hello, mmap world!
```

### 3 优势和局限性

`mmap` 数据直接从磁盘映射到用户空间，避免了内核缓冲区到用户缓冲区的拷贝。

还有一种用得比较多的方式是作为进程间通信使用，直接共享内存交换数据。

笔者很少见到使用 `mmap` 作为读写数据引擎的基本方式。在构建存储引擎时，有开发者批评 `mmap` 的劣势在于不能完全掌控背后内核的内存管理和刷盘机制[^youjiali]。

开发者如果使用 `mmap`，尤其是文件远远大于机器内存的情况下需要调优。RocksDB 曾遇到一个未设置 `fadvise` 导致性能下降的 issue[^rocksdb_issue]。Rocksdb 手册[^rocksdb_man] 提到如果数据在内存 fs 中，开启 `mmap` 会带来比较大的性能提升，否则应该谨慎试用 `mmap` 选项。


[^1]: [Ensuring data reaches disk - LWN.net](https://lwn.net/Articles/457667/)
[^2]: [从共识算法开谈 - 硬盘性能的最大几个误解](https://zhuanlan.zhihu.com/p/55658164)
[^3]: [浅谈存储引擎数据结构](https://haobin.work/2024/05/24/%E7%AE%97%E6%B3%95/%E6%B5%85%E8%B0%88%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[^openman]: [open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)
[^youjiali]: [ScyllaDB 学习(六) – disk I/O](https://youjiali1995.github.io/scylladb/disk-io/)
[^rocksdb_issue]: [Fixing mmap performance for RocksDB](https://smalldatum.blogspot.com/2022/06/fixing-mmap-performance-for-rocksdb.html)
[^rocksdb_man]: [RocksDB Wiki - IO](https://github.com/facebook/rocksdb/wiki/IO)