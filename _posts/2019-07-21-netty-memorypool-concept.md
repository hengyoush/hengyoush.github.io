---
layout: post
title:  "netty内存池原理分析"
date:   2019-07-21 20:18:00 +0700
categories: [netty]
---

## Overview
netty基于JEMalloc算法构建了一套高性能的内存分配机制, 下面依据netty源码来介绍一下其原理.

最顶层的是PoolArena, Arena下分为Chunk, Chunk下分为Page, Page又可分为SubPage.
Arena下的Chunk被划分为了几个ChunkList(双向链表):
```java
private final PoolChunkList<T> q050;
private final PoolChunkList<T> q025;
private final PoolChunkList<T> q000;
private final PoolChunkList<T> qInit;
private final PoolChunkList<T> q075;
private final PoolChunkList<T> q100;
```
划分的依据是Chunk的空间使用率, 如下, x轴为使用率:
![avatar](/static/img/netty-memalloc-chunklist.webp)
Chunk的初始状态为INIT.

Chunk是一个较大的内存块, 默认为16MB, 一个Chunk默认分为2048个Page, 每个Page大小为16 * 1024 * 1024 / 2048 = 8192 = 8K,
Page由一个完全平衡二叉树管理, 其中叶子为2048个Page.
// TODO 举例子解释伙伴算法分配内存

每次请求的大小可分为如下三类:
- Tiny: 16-511 
- Small: 512-8192 (一个Page以内)
- Normal: 8K-16MB (一个page到一个Chunk)
- Huge: > 16MB
![avatar](/static/img/netty-memalloc-sizeclass.webp)
请求大小会被规范化, 最小16B, 在到达Small之前以16B递增作为可分配的大小, 然后以2倍递增.

netty中最顶层的类应该是`PooledByteBufAllocator`, 在它的构造器中会初始化大量内存池参数以及一个PoolArena列表, 如下:
```java
threadCache = new PoolThreadLocalCache(useCacheForAllThreads); // 用于在线程本地分配内存
this.tinyCacheSize = tinyCacheSize;
this.smallCacheSize = smallCacheSize;
this.normalCacheSize = normalCacheSize;
chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);
...
if (nHeapArena > 0) {
    ...
}

if (nDirectArena > 0) {
    directArenas = newArenaArray(nDirectArena);
    List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(directArenas.length);
    for (int i = 0; i < directArenas.length; i ++) {
        PoolArena.DirectArena arena = new PoolArena.DirectArena(
                this, pageSize, maxOrder, pageShifts, chunkSize, directMemoryCacheAlignment);
        directArenas[i] = arena;
        metrics.add(arena);
    }
    directArenaMetrics = Collections.unmodifiableList(metrics);
} else {
    directArenas = null;
    directArenaMetrics = Collections.emptyList();
}
```

我们以使用direct内存为例(接下来也是这样)可以看到这里使用相关参数初始化了一个`PoolArena.DirectArena`数组.
