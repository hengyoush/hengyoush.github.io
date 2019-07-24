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
        if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {// 先从线程本地缓存中尝试分配, 如果成功直接返回, 如果没有可供分配的缓存, 那么尝试在Subpage数组中分配, 见如下分析.
            // 如果在线程本地分配成功, 那么直接返回
            return;
        }
        tableIdx = tinyIdx(normCapacity); // tinyIdx方法根据请求大小获得在tinySubpagePools中的数组索引, 即寻找具有合适的Subpage大小的Page链表
        table = tinySubpagePools;
    } else {
        if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) { // 同上, 只不过分配Small
            return;
        }
        tableIdx = smallIdx(normCapacity);
        table = smallSubpagePools;
    }

    final PoolSubpage<T> head = table[tableIdx]; // 获取链表头节点

    // 在SubpagePool中尝试分配
    // 需要在链表头节点上同步, 因为Chunk中的分配Subpage和free时也会修改链表
    synchronized (head) {
        final PoolSubpage<T> s = head.next;
        if (s != head) { // 如果链表中有除了头节点以外的节点, 即其他Chunk已经分配的Page
            assert s.doNotDestroy && s.elemSize == normCapacity;
            long handle = s.allocate();
            assert handle >= 0;
            s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
            incTinySmallAllocation(tiny);
            return;
        }
    }

    // 
    synchronized (this) {
        allocateNormal(buf, reqCapacity, normCapacity);
    }

    incTinySmallAllocation(tiny);
    return;
}// normal大小的请求
if (normCapacity <= chunkSize) {
    if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {// 尝试在线程本地缓存中分配
        // was able to allocate out of the cache so move on
        return;
    }
    synchronized (this) {
        allocateNormal(buf, reqCapacity, normCapacity);
        ++allocationsNormal;
    }
} else {
    // Huge allocations are never served via the cache so just call allocateHuge
    allocateHuge(buf, reqCapacity); // huge大小的分配不会缓存
}
}
```
---

## 线程本地缓存分配
首先我们看`PoolThreadCache`的`allocateTiny`方法.
```java
boolean allocateTiny(PoolArena<?> area, PooledByteBuf<?> buf, int reqCapacity, int normCapacity) {
    return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
}
```
首先调用了`cacheForTiny`方法获取一个`MemoryRegionCache`, 这个`MemoryRegionCache`又是什么呢?

### MemoryRegionCache
我们首先看`MemoryRegionCache`在`PoolThreadCache`中的初始化以及在`PoolThreadCache`中的作用:
```java
private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;
private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;
private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
private final MemoryRegionCache<byte[]>[] normalHeapCaches;
private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;
```
可以看到, 在`PoolThreadCache`中存在着不同SizeClass的`MemoryRegionCache`数组, 猜测它应该是缓存的封装.
接着我们看`MemoryRegionCache`的初始化过程来理解`MemoryRegionCache`的构造参数的意思.(*略去部分代码*)
```java
tinySubPageDirectCaches = createSubPageCaches(
        tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
smallSubPageDirectCaches = createSubPageCaches(
        smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);

private static <T> MemoryRegionCache<T>[] createSubPageCaches(
        int cacheSize, int numCaches, SizeClass sizeClass) {
        MemoryRegionCache<T>[] cache = new MemoryRegionCache[numCaches];
        for (int i = 0; i < cache.length; i++) {
            cache[i] = new SubPageMemoryRegionCache<T>(cacheSize, sizeClass);
        }
        return cache;
}       
```
可以看到, `MemoryRegionCache`数组的长度来自Arena中的`numTinySubpagePools`和`numSmallSubpagePools`,所以`MemoryRegionCache`实际上也是和Arena中的SubpagePool对应, 也是根据数组索引来区分Subpage大小不同的Page的.

对于`MemoryRegionCache`本身来说, 主要由`cacheSize`和`sizeClass`来控制, 我们来看`MemoryRegionCache`本身的构造.

`MemoryRegionCache`的属性和构造方法如下:
```java
private final int size; // 控制内部缓存个数
private final Queue<Entry<T>> queue; // 内部的缓存队列
private final SizeClass sizeClass; 
private int allocations; // 分配次数

MemoryRegionCache(int size, SizeClass sizeClass) {
    this.size = MathUtil.safeFindNextPositivePowerOfTwo(size); // 规范化
    queue = PlatformDependent.newFixedMpscQueue(this.size); // 构造线程安全的Mpsc队列
    this.sizeClass = sizeClass;
}
```

接着我们看add方法, add方法用于将一个Chunk不用的buffer回收到内部的队列中.
```java
public final boolean add(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle) {
    Entry<T> entry = newEntry(chunk, nioBuffer, handle);
    boolean queued = queue.offer(entry);
    if (!queued) {
        entry.recycle(); // 队列满了, 直接回收完事
    }

    return queued;
}
```
首先这里构造了一个Entry对象, 用于封装Buffer存储在队列中, Entry详情如下:
```java
static final class Entry<T> {
    final Handle<Entry<?>> recyclerHandle;
    PoolChunk<T> chunk;
    ByteBuffer nioBuffer;
    long handle = -1;

    Entry(Handle<Entry<?>> recyclerHandle) {
        this.recyclerHandle = recyclerHandle;
    }

    void recycle() {
        chunk = null;
        nioBuffer = null;
        handle = -1;
        recyclerHandle.recycle(this);
    }
}
```
再看allocate方法, 它尝试分配一个队列中的缓存
```java
public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
    Entry<T> entry = queue.poll();
    if (entry == null) {
        return false;
    }
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
    entry.recycle();
    ++ allocations;
    return true;
}
```
首先尝试从队列中poll, 如果poll返回的是null, 说明当前队列为空, 直接返回`false`.

否则使用entry中的缓存来初始化传入参数的PooledByteBuf, 接着回收entry更新分配计数器结束.

以上就是`MemoryRegionCache`的大体作用, 接下来我们回到`PoolThreadCache`的`cacheForTiny`方法中来.

---

如下是`cacheForTiny`以及其相关调用方法的代码:
```java
private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
    int idx = PoolArena.tinyIdx(normCapacity);
    if (area.isDirect()) {
        return cache(tinySubPageDirectCaches, idx);
    }
    return cache(tinySubPageHeapCaches, idx);
}

/** PoolArena#tinyIdx **/
static int tinyIdx(int normCapacity) {
    return normCapacity >>> 4;
}

/** PoolThreadCache#cache **/
private static <T> MemoryRegionCache<T> cache(MemoryRegionCache<T>[] cache, int idx) {
    if (cache == null || idx > cache.length - 1) {
        return null;
    }
    return cache[idx];
}
```
首先使用tinyIdx方法对请求大小除以16, 这很好理解, 因为Tiny大小本身小于512, 所以除以16就是为了得出在
`MemoryRegionCache`数组中的索引, `PoolThreadCache#cache`方法简单对索引做了范围check直接返回cache[idx]了, 所以其逻辑还是非常简单的.

下面我们进入`allocate`方法中:
```java
private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
    boolean allocated = cache.allocate(buf, reqCapacity);
    if (++ allocations >= freeSweepAllocationThreshold) {
        allocations = 0;
        trim();
    }
    return allocated;
}
```
通过调用MemoryRegionCache的allocate方法(上面已经分析了), 如果在MemoryRegionCache中分配完成的话那么会返回true否则返回false.

好的, 关于线程本地缓存进行分配的分析到此结束, 我们回到Arena的分配流程中去.

## SubpagePool分配
我们知道, 在一开始, 线程本地缓存是空的, 因为线程本地缓存的本质是本应回收的内存存放在线程本地变量中用于之后的分配, 所以根据接下来Arena分配的逻辑, 对于Tiny/Small大小的请求来说, 会进入SubpagePool分配的逻辑当中.

首先我们会很想知道SubpagePool中的PoolSubpage是从哪来的, 毕竟在Arena的构造方法中并没有初始化它们, 我们先带着这个问题, 我会在之后解释.

好的,回到Arena的逻辑中来, 如果线程本地缓存分配失败, 接下来:
```java
tableIdx = tinyIdx(normCapacity);
table = tinySubpagePools;

// or

tableIdx = smallIdx(normCapacity);
table = smallSubpagePools;
```
获取对应的索引和上面的MemoryRegionCache数组获取索引的意义相同, 不再赘述.

获取到索引之后, 接着:
```java
final PoolSubpage<T> head = table[tableIdx];

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
```
在head上同步, 接着调用PoolSubpage的allocate方法(*注意这个方法没有带参数,比如请求大小什么的,因为Subpage可分配带大小是在一开始就固定的*)
对于PoolSubpage的allocate方法的详细实现我们待会再解释, 我们只要知道它返回的handle——一个long值代表着:

- 高32位: 分配的内存在SubPage中的位置
- 低32位: Subpage在Chunk中的位置

接着我们使用这个handle初始化一开始传入的PooledByteBuf.(待会解释详细初始化过程) ***TODO***

接着完成之后返回.

## ChunkList分配
如果上述分配都不成功的话(一开始的话就会走到这里), 会进入ChunkList分配的阶段, 这里也是一定会完成分配的阶段, 首先我们来看`allocateNormal`的源码:
```java
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }

    // Add a new chunk.
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    boolean success = c.allocate(buf, reqCapacity, normCapacity);
    assert success;
    qInit.add(c);
}
```