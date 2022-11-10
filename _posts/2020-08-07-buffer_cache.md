---
layout: post
title: 内存中的Buffer和Cache
date: 2020-08-07
tags: 系统知识
---

### 1.free 数据的来源
用 man 命令查询 free 的文档，就可以找到对应指标的详细说明。
比如，我们执行 man free ，就可以看到下面这个界面。

```
buffers
Memory used by kernel buffers (Buffers in /proc/meminfo)

cache  
Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

buff/cache
Sum of buffers and cache
```

从 free 的手册中，你可以看到 buffer 和 cache 的说明。

- Buffers 是内核缓冲区用到的内存，对应的是 /proc/meminfo 中的 Buffers 值。
- Cache 是内核页缓存和 Slab 用到的内存，对应的是 /proc/meminfo 中的 Cached 与 SReclaimable 之和。

### 2.proc 文件系统

/proc 是 Linux 内核提供的一种特殊文件系统，是用户跟内核交互的接口。比方说，用户可以从 /proc 中查询内核的运行状态和配置选项，查询进程的运行状态、统计数据等，当然，你也可以通过 /proc  来修改内核的配置。

proc 文件系统同时也是很多性能工具的最终数据来源。比如我们刚才看到的 free ，就是通过读取 /proc/meminfo  ，得到内存的使用情况。

继续说回 /proc/meminfo，既然 Buffers、Cached、SReclaimable 这几个指标不容易理解，那我们还得继续查 proc 文件系统，获取它们的详细定义。

执行 man proc  ，你就可以得到 proc 文件系统的详细文档。注意这个文档比较长，你最好搜索一下（比如搜索 meminfo），以便更快定位到内存部分。

```
Buffers %lu
Relatively temporary storage for raw disk blocks that shouldn't get tremendously large (20MB or so).
 
Cached %lu
In-memory cache for files read from the disk (the page cache).  Doesn't include SwapCached.
 ...
SReclaimable %lu (since Linux 2.6.19)
Part of Slab, that might be reclaimed, such as caches.
 
SUnreclaim %lu (since Linux 2.6.19)
Part of Slab, that cannot be reclaimed on memory pressure.
```

通过这个文档，我们可以看到：

- Buffers 是对原始磁盘块的临时存储，也就是用来缓存磁盘的数据，通常不会特别大（20MB 左右）。这样，内核就可以把分散的写集中起来，统一优化磁盘的写入，比如可以把多次小的写合并成单次大的写等等。
- Cached 是从磁盘读取文件的页缓存，也就是用来缓存从文件读取的数据。这样，下次访问这些文件数据时，就可以直接从内存中快速获取，而不需要再次访问缓慢的磁盘。
- SReclaimable 是 Slab 的一部分。Slab 包括两部分，其中的可回收部分，用 SReclaimable 记录；而不可回收部分，用 SUnreclaim 记录。

### 3.linux清除缓存：需要root权限

```
$sync; echo 3 >/proc/sys/vm/drop_caches
```

上面的echo 3 是清理所有缓存

echo 0 是不释放缓存

echo 1 是释放页缓存

ehco 2 是释放dentries和inodes缓存

echo 3 是释放 1 和 2 中说道的的所有缓存