---
layout: post
title:  "Netty内存池原理分析"
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
![avatar](/static/img/netty-memalloc-huoban.webp)
如上图所示, 二叉树高12, 叶节点数目为2^11 = 2048, 正好为一个Chunk中Page的数目.
假设我们要在一个尚未开始分配的Chunk中分配一个Tiny类型的空间, 那么它会先判断这个大小会在那一层分配, Tiny小于一个Page的大小所以在
11层分配, 接下来如果分配一个18KB的大小, 该大小需要4个Page(分配空间的规范化, 下面会讲到, 实际会分配32KB), 故需要在第九层分配, 512这个节点无法满足, 那么
选取的是513这个节点进行分配.

每次请求的大小可分为如下三类:
- Tiny: 16-511 
- Small: 512-8192 (一个Page以内)
- Normal: 8K-16MB (一个page到一个Chunk)
- Huge: > 16MB
![avatar](/static/img/netty-memalloc-sizeclass.webp)

请求大小会被规范化, 最小16B, 在到达Small之前以16B递增作为可分配的大小, 然后以2倍递增.

对于Tiny和Small会在一个Page内分配, Netty对Page进一步切分为SubPage, SubPage的大小即第一次分配的请求大小, 同样切分大小的Page会组成一个双向链表
存在Chunk所属的Arena中.

---

## PooledByteBufAllocator

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

好了, 分析完PooledByteBufAllocator的初始化, 接下来我们看PooledByteBufAllocator的内存分配代码:
```java
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    PoolThreadCache cache = threadCache.get();
    PoolArena<ByteBuffer> directArena = cache.directArena;

    final ByteBuf buf;
    if (directArena != null) {
        buf = directArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        buf = PlatformDependent.hasUnsafe() ?
                UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }

    return toLeakAwareBuffer(buf);
}
```
该方法根据一个初始容量和最大容量分配一个`DirectBuffer`.
首先判断`PoolThreadCache`中的`directArena`是否为空, 这里PoolThreadCache中的directArena是在什么时候设置的呢?

答案是在上面`PooledByteBufAllocator`的构造函数开始部分的`threadCache = new PoolThreadLocalCache(useCacheForAllThreads);`,
这里构造了一个`PoolThreadLocalCache`的对象, 我们看一下该对象的情况, 它继承了`FastThreadLocal`, 并且重写了`initialValue()`方法,
从`FastThreadLocal`的名称我们就可以猜到`initialValue()`方法是做什么的, 该方法提供一个线程本地变量的初始值, 
该初始值是一个`PoolThreadCache`类型的对象, 我们看一下该方法:
```java
protected synchronized PoolThreadCache initialValue() {
    final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
    final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

    final Thread current = Thread.currentThread();
    if (useCacheForAllThreads || current instanceof FastThreadLocalThread) {
        final PoolThreadCache cache = new PoolThreadCache(
                heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);

        if (DEFAULT_CACHE_TRIM_INTERVAL_MILLIS > 0) {
            final EventExecutor executor = ThreadExecutorMap.currentExecutor();
            if (executor != null) {
                executor.scheduleAtFixedRate(trimTask, DEFAULT_CACHE_TRIM_INTERVAL_MILLIS,
                        DEFAULT_CACHE_TRIM_INTERVAL_MILLIS, TimeUnit.MILLISECONDS);
            }
        }
        return cache;
    }
    // No caching so just use 0 as sizes.
    return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
}
```
忽略该方法关于直接内存和对内存之间的判断, 我们可以看到首先调用`leastUsedArena`方法从Arena数组中获取最少被`PoolThreadCache`使用的Arena.
然后构造一个`PoolThreadCache`对象, 对于该类的详细原理我们待会再说, 先继续`newDirectBuffer`的逻辑.

如果我们按照上述流程正常走下去, `PoolThreadCache`中应该是存在Arena的, 可以到DirectArena的allocate方法内.


---

## PoolArena初步解析
在分析`PoolArena`的内存分配流程之前, 我们先梳理一下`PoolArena`的大体结构.
`PoolArena`的重要属性如下所示:
```java
private final int maxOrder; // chunk二叉树的高度，默认11
final int pageSize; // 单个page大小，默认8192
final int pageShifts;  //
final int chunkSize; // chunk大小
final int subpageOverflowMask; // 用于计算是否大于tiny/small
final int numSmallSubpagePools; // small请求的链表头个数
final int directMemoryCacheAlignment; // 对齐内存单位
final int directMemoryCacheAlignmentMask; // 用于对齐内存
private final PoolSubpage<T>[] tinySubpagePools; // 元素为tiny请求的Subpage双向链表头，下标表示subpage的大小，下同
private final PoolSubpage<T>[] smallSubpagePools;// 元素为small请求的Subpage双向链表头
// 各个利用率的chunkList
private final PoolChunkList<T> q050;
private final PoolChunkList<T> q025;
private final PoolChunkList<T> q000;
private final PoolChunkList<T> qInit;
private final PoolChunkList<T> q075;
private final PoolChunkList<T> q100;
```
各个ChunkList的关系如下所示:
![avatar](/static/img/netty-memalloc-chunklist.png)

在构造方法中, 除了对上述ChunkList的构造之外, 对一些其他的重要参数也有处理, 如下:
```java
this.parent = parent; // PooledByteBufAllocator
this.pageSize = pageSize;
this.maxOrder = maxOrder;
this.pageShifts = pageShifts;
this.chunkSize = chunkSize;
directMemoryCacheAlignment = cacheAlignment;
directMemoryCacheAlignmentMask = cacheAlignment - 1;
subpageOverflowMask = ~(pageSize - 1);
tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);
for (int i = 0; i < tinySubpagePools.length; i ++) {
    tinySubpagePools[i] = newSubpagePoolHead(pageSize);
}

numSmallSubpagePools = pageShifts - 9;// pageShifts默认为13，则此处为4，即512，1k，2k，4k
smallSubpagePools = newSubpagePoolArray(numSmallSubpagePools);
for (int i = 0; i < smallSubpagePools.length; i ++) {
    smallSubpagePools[i] = newSubpagePoolHead(pageSize);
}
```
需要说明的是`tinySubpagePools`和`smallSubpagePools`, 在上面的属性中也可以看到, 这两个数组中的元素都是链表的头,
数组下标的不同表示SubPage的大小, 头是不参与内存的分配的. 在Chunk中对SubPage的初始分配都会将对应的Page加入到对应的链表中.

好的, 分析完了`PoolArena`的构造方法, 接下来我们可以开始内存分配的分析了, 回到`PoolArena`的`allocate`方法中来:
```java
PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
    PooledByteBuf<T> buf = newByteBuf(maxCapacity);
    allocate(cache, buf, reqCapacity);
    return buf;
}
```
首先通过`newByteBuf`方法构造一个未分配内存的`PooledByteBuf`, 接下来进入重载的`allocate`方法中:
```java
final int normCapacity = normalizeCapacity(reqCapacity); // 规范化请求内存大小
if (isTinyOrSmall(normCapacity)) { // 是否是tiny或者small, 即判断是否小于pageSize
    int tableIdx;
    PoolSubpage<T>[] table;
    boolean tiny = isTiny(normCapacity);
    if (tiny) { // < 512
        if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {// 先从线程本地缓存中尝试分配, 如果成功直接返回, 如果没有可供分配的缓存, 那么尝试在Subpage数组中分配.
            // was able to allocate out of the cache so move on
            return;
        }
        tableIdx = tinyIdx(normCapacity);
        table = tinySubpagePools;
    } else {
        if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        tableIdx = smallIdx(normCapacity);
        table = smallSubpagePools;
    }

    final PoolSubpage<T> head = table[tableIdx];

    /**
      * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
      * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
      */
    synchronized (head) {
        final PoolSubpage<T> s = head.next;
        if (s != head) {
            assert s.doNotDestroy && s.elemSize == normCapacity;
            long handle = s.allocate();
            assert handle >= 0;
            s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
            incTinySmallAllocation(tiny);
            return;
        }
    }
    synchronized (this) {
        allocateNormal(buf, reqCapacity, normCapacity);
    }

    incTinySmallAllocation(tiny);
    return;
}
if (normCapacity <= chunkSize) {
    if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
        // was able to allocate out of the cache so move on
        return;
    }
    synchronized (this) {
        allocateNormal(buf, reqCapacity, normCapacity);
        ++allocationsNormal;
    }
} else {
    // Huge allocations are never served via the cache so just call allocateHuge
    allocateHuge(buf, reqCapacity);
}
}
```