---
layout: post
title:  "Netty内存池原理分析"
date:   2019-07-27 02:00:00 +0700
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
首先尝试在每个ChunkList中分配, 如果在其中一个ChunkList中分配成功,那么直接返回, 否则创建一个新的Chunk, 在其中分配, 然后将其加入到qInit的
ChunkList中.

ok ,我们来看ChunkList的allocate方法.(*我们在这里可以做一个小小的猜测, 在ChunkList的allocate方法中还是会调用Chunk的allocate方法*)

```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    if (normCapacity > maxCapacity) {
        return false;
    }

    for (PoolChunk<T> cur = head; cur != null; cur = cur.next) {
        if (cur.allocate(buf, reqCapacity, normCapacity)) {
            if (cur.usage() >= maxUsage) {
                remove(cur);
                nextList.add(cur);
            }
            return true;
        }
    }
    return false;
}
```

首先判断了请求大小不能超过这个ChunkList中的Chunk可以分配的最大大小, 这个限制值是由这个ChunkList带有的使用率决定的(详见上方的图).

接着遍历链表, 调用Chunk的allocate方法, 如果分配成功那么判断是否需要转移该Chunk.

ok, 归根结底还是调用了Chunk的allocate方法, 下面我们进入Chunk的allocate方法一探究竟. 不过在此之前我们需要先了解Chunk的结构组成.

---

## Chunk

### Chunk结构分析
```java
final PoolArena<T> arena; // Chunk所在的Arena
final T memory; // 实际的内存，根据是不是direct的有区分，比如direct的是ByteBuffer，heap的是byte[]
final boolean unpooled; // false
final int offset; // 相对memory的偏移（因为要内存对齐，详见PoolArena#offsetCacheLine）
private final byte[] memoryMap; // 记录可分配信息
private final byte[] depthMap; // 存放每个深度信息
private final PoolSubpage<T>[] subpages; // 存放该Chunk的subpage
private final int subpageOverflowMask; // 判断是否大于pageSize的掩码
private final int pageSize; // 默认8K
private final int pageShifts; // 默认13
private final int maxOrder; // 11
private final int chunkSize; // 16MB
private final int log2ChunkSize; // 24
private final int maxSubpageAllocs;// 1 << maxOrder
```

Chunk的最主要的属性是memoryMap和depthMap.我们知道, Chunk默认被分为2048个Page, 每个Page大小8K, Chunk采用完全二叉树来管理Page, 每个Page作为二叉树的叶节点,
那么二叉树的树高为11, 总共包含2048 * 2 - 1  = 4095个节点.

好的,我们接着来看Chunk的构造方法.
```java
PoolChunk(PoolArena<T> arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize, int offset) {
    unpooled = false;
    this.arena = arena;
    this.memory = memory;
    this.pageSize = pageSize;
    this.pageShifts = pageShifts;
    this.maxOrder = maxOrder;
    this.chunkSize = chunkSize;
    this.offset = offset;
    unusable = (byte) (maxOrder + 1); // 12
    log2ChunkSize = log2(chunkSize);
    subpageOverflowMask = ~(pageSize - 1);
    freeBytes = chunkSize;

    maxSubpageAllocs = 1 << maxOrder; // 2^11

    // Generate the memory map.
    memoryMap = new byte[maxSubpageAllocs << 1];// 一共2^12
    depthMap = new byte[memoryMap.length];// 一共2^12
    int memoryMapIndex = 1;
    for (int d = 0; d <= maxOrder; ++ d) {
        int depth = 1 << d;
        for (int p = 0; p < depth; ++ p) {
            memoryMap[memoryMapIndex] = (byte) d;// 全都初始化为对应的深度
            depthMap[memoryMapIndex] = (byte) d;
            memoryMapIndex ++;
        }
    }

    subpages = newSubpageArray(maxSubpageAllocs);
    cachedNioBuffers = new ArrayDeque<ByteBuffer>(8);
}
```

ok, 对照着构造方法中对memoryMap和depthMap的初始化过程, 我们可以发现它们在一开始都被初始化为对应节点在二叉树中的深度.

depthMap在之后不会改变, 而memoryMap在之后会随着节点的分配而改变, 比如一个叶节点A被分配, 那么这个A节点所在的memoryMap的值
会被修改为unusable即12, 其父节点的memoryMap值会根据其子节点的memoryMap的最小值来决定, 那么A的父节点的memoryMap值应该被更新为11, 
其父节点的父节点会被更新为10, 以此类推.

ok, 了解了内存分配对memoryMap的影响, 接下来我们回到allocate的逻辑中来.

### allocate分析
```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    final long handle;
    if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
        handle =  allocateRun(normCapacity);// 分配多个Page
    } else {
        handle = allocateSubpage(normCapacity);// 在一个Page内分配
    }

    if (handle < 0) {
        return false;
    }
    ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
    initBuf(buf, nioBuffer, handle, reqCapacity);
    return true;
}
```

首先判断要分配的大小是否小于一个Page, 如果小于那么直接进入SubPage的逻辑中否则分配多个Page.

我们首先看`allocateSubpage`.

#### allocateSubpage
```java
private long allocateSubpage(int normCapacity) {
    // 获取对应arena中的Subpage链表中的head
    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
    int d = maxOrder; // d为要分配的深度, 此处是Subpage,故一定在叶节点, d = 11;
    synchronized (head) {
        int id = allocateNode(d);// 重要逻辑在这
        if (id < 0) {
            return id;
        }

        final PoolSubpage<T>[] subpages = this.subpages;
        final int pageSize = this.pageSize;

        freeBytes -= pageSize;

        int subpageIdx = subpageIdx(id);// 获取subpage在Chunk中的位置
        PoolSubpage<T> subpage = subpages[subpageIdx];
        if (subpage == null) {
            subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        } else {
            subpage.init(head, normCapacity);
        }
        return subpage.allocate();
    }
}
```

1. 首先确定要分配的深度, 这里是叶节点, 故d为maxOrder.
2. 接着调用allocateNode方法,传入参数是深度, 在接下来我们可以看到allocateRun其实也调用了该方法, 只不过深度小于maxOrder.
3. 获得分配的节点后需要初始化subpage并且添加到arena中 的subpagePool中去.

我们先看allocateNode方法:
```java
private int allocateNode(int d) {
    int id = 1;
    int initial = - (1 << d); // 最后d位为0, 其他为1
    byte val = value(id); // 获取memoryMap的值, 这里id=1, 二叉树的root, 从root开始往下
    if (val > d) { // 当memoryMap的值大于某个值说明该节点下面没有足够的空间分配了
        return -1;
    }
    while (val < d || (id & initial) == 0) { // 只要当前id对应的节点还能分配(有余) || id还在d层的上面但节点无余(val==d) -> 继续向下
        id <<= 1; // 继续向下
        val = value(id);
        if (val > d) { // 如果左节点不行,那么尝试右节点(右节点一定可以, 因为已经通过了while条件的判断了)
            id ^= 1;
            val = value(id);
        }
    }
    byte value = value(id);
    assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
            value, id & initial, d);
    setValue(id, unusable); // 将获取到的节点的memoryMap设置为unusable
    updateParentsAlloc(id); // 更新祖先节点
    return id;
}

/** Chunk#value **/
private byte value(int id) {
    return memoryMap[id];
}
/** Chunk#updateParentsAlloc **/
private void updateParentsAlloc(int id) {
    while (id > 1) { // 没到达root, root没有父节点
        int parentId = id >>> 1; // 父节点
        byte val1 = value(id);
        byte val2 = value(id ^ 1);
        byte val = val1 < val2 ? val1 : val2; // 获取自己和兄弟节点的value, 选取小的作为parent的value
        setValue(parentId, val);
        id = parentId;
    }
}
```
首先我们假设d = 11, 即分配的是叶节点来解释allocate的代码, 这段代码实现了伙伴算法的分配思想.
1. 判断当前root是否可以分配, 这里会有同学想问`val = d`可不可以,当然不可以,因为这相当于要分配一个Page, er这个大小是不能通过`(normCapacity & subpageOverflowMask) != 0`这个检查的.
2. 进入while循环, 这个循环的作用主要是找到能够满足当前需要的最低级节点, 而且是从左往右分配. 然后我们再来解读这个while循环的条件:<br><br>
首先判断`val < d`, 这个式子成立的意思是当前节点的两个节点都满足分配; <br><br> 第二个式子是`(id & initial) == 0`, initial只有最后11位是0, 其他全是1, 这意味着想要这个条件成立,必须id小于2^11,这就意味着id不能是目标深度节点才可以进入循环, 因为进入循环就意味着要向下一层前进, 而且第二个式子的隐含条件是`val==d`,即只有右孩子节点是正好满足要求的.
3. 进入循环, 首先id左移, 相当于深度加一, 如果左孩子无法分配, 那么与1异或(左孩子永远都是偶数), 得到右孩子, 继续循环.
4. 跳出循环, 这里id就是我们要找的节点, 我们将其设置为12,并且更新节点的memoryMap的值知道父节点(如前所述).返回id, id即节点在二叉树中的索引.

ok, 分析完了allocateNode方法, 我们继续回到allocateSubpage方法中

```java
int id = allocateNode(d);// 我们这里返回了在二叉树中的索引
if (id < 0) {
    return id;
}

final PoolSubpage<T>[] subpages = this.subpages;
final int pageSize = this.pageSize;

freeBytes -= pageSize;

int subpageIdx = subpageIdx(id);// 获取subpage在Chunk中的位置
PoolSubpage<T> subpage = subpages[subpageIdx];
if (subpage == null) {
    subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity); // runOffset(id)获取的是在Chunk中的内存偏移量
    subpages[subpageIdx] = subpage;
} else {
    subpage.init(head, normCapacity);
}
return subpage.allocate();

/** subpageIdx **/
private int subpageIdx(int memoryMapIdx) {
    return memoryMapIdx ^ maxSubpageAllocs; // remove highest set bit, to get offset
}
```
我们获取了分配节点在Chunk二叉树中的索引, 接着调用`subpageIdx`方法获取其在subpages中的偏移量(因为叶节点的id是大于subpages的长度即1 << 11 的, 所以此处将最高位的1移除).

获取到subpages数组中对应位置的PoolSubpage后根据其是否已经分配进行后续的逻辑:

- 未分配: 初始化一个PoolSubpage,放到Chunk的subpages中.
- 已分配: 说明以前分配过但现在free了, 重新init即可.

我们分别来看这两种情况之前需要分析`PoolSubpage`的组成.

### PoolSubpage结构分析
```java
final PoolChunk<T> chunk; // 所属chunk
private final int memoryMapIdx; // memoryMap数组的索引
private final int runOffset; // 在chunk中的内存偏移量
private final int pageSize; 
private final long[] bitmap; // 表示Subpage的分配信息，长度为pageSize / 16 / 64，64代表是long，16代表每个位16B(对应最小16B), 默认为8

PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
    this.chunk = chunk;
    this.memoryMapIdx = memoryMapIdx;
    this.runOffset = runOffset;
    this.pageSize = pageSize;
    bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64
    init(head, elemSize);
}

void init(PoolSubpage<T> head, int elemSize) {
    doNotDestroy = true;
    this.elemSize = elemSize;
    if (elemSize != 0) {
        maxNumElems = numAvail = pageSize / elemSize; // 可分配个数
        nextAvail = 0;
        bitmapLength = maxNumElems >>> 6; // 可分配个数除以64, 计算bitmap的长度
        if ((maxNumElems & 63) != 0) { // 此处补上一步整除的余数, 假设maxNumElems = 100, 那么上一步得出的结果为1, 此处再加一
            bitmapLength ++;
        }

        for (int i = 0; i < bitmapLength; i ++) {// 初始化bitmap
            bitmap[i] = 0;
        }
    }
    addToPool(head);
}

private void addToPool(PoolSubpage<T> head) { // 简单插入head的next中
    assert prev == null && next == null;
    prev = head;
    next = head.next;
    next.prev = this;
    head.next = this;
}
```
init方法参数head为Arena中Subpage链表的head, elemSize为Subpage的分配大小.

如上所示, PoolSubpage的初始化过程主要是bitmap数组的初始化, bitmap主要表示一个Page中的分配情况,
bitmap是long数组, 每个long中的每一位表示一次分配.每一次分配都会在bitmap中的long中的一位置1, 在每一个long中从右向左分配.

例如, 假设该Page的可分配大小为1K, 那么可分配个数为8, 所以一个long就可以表示完成表示.

假设分配大小为32, 那么可分配个数就为256, 需要long的个数为4, 所以bitmap的长度为4.

ok, 分析完了subpage的大致结构, 我们再来看它的allocate方法.

```java
long allocate() {
    if (elemSize == 0) {
        return toHandle(0);
    }

    if (numAvail == 0 || !doNotDestroy) { // 分配大小过大或者doNotDestroy为false,返回分配失败
        return -1;
    }

    final int bitmapIdx = getNextAvail(); // 获取可用的bitmap索引,第一次分配时为0
    int q = bitmapIdx >>> 6; // 第几个long?
    int r = bitmapIdx & 63; // long中的第几个?
    assert (bitmap[q] >>> r & 1) == 0; // 一定没分配
    bitmap[q] |= 1L << r; // 把对应的位置为1

    if (-- numAvail == 0) {
        removeFromPool(); // 从链表中unlink
    }
    // 0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx;
    return toHandle(bitmapIdx);  // 即前32位放bitmap, 后32位放memoryMap的索引即Page在Chunk中的位置
}
/**---------------------------**/
private int getNextAvail() {
    int nextAvail = this.nextAvail;
    if (nextAvail >= 0) { // 初始时为0
        this.nextAvail = -1;
        return nextAvail;
    }
    return findNextAvail(); // 如果不是第一次分配
}
/**---------------------------**/
private int findNextAvail() {
    final long[] bitmap = this.bitmap;
    final int bitmapLength = this.bitmapLength;
    for (int i = 0; i < bitmapLength; i ++) { // 遍历bitmap,寻找没有被占用的空间
        long bits = bitmap[i];
        if (~bits != 0) { // bits的反不为0说明bits存在0, 即Subpage没有分配的位置
            return findNextAvail0(i, bits); // i为当前第几个long, bits为当前long
        }
    }
    return -1;
}
/**---------------------------**/
private int findNextAvail0(int i, long bits) {
    final int maxNumElems = this.maxNumElems;
    final int baseVal = i << 6; 

    for (int j = 0; j < 64; j ++) {// 遍历bits的每一位, 寻找0, 在分配时1是从右向左分配的
        if ((bits & 1) == 0) { // 找到0了
            int val = baseVal | j; 
            if (val < maxNumElems) {
                return val;
            } else {
                break;
            }
        }
        bits >>>= 1; // 这一位是1,继续向左寻找
    }
    return -1;
}
```
1. 首先获取下一个可用的bitmapIdx,.(*bitmapIdx = 64 * (bitmap数组索引) + long中第某位比如bitmapIdx = 106, 那么就意味着在bitmap中的第2个long中的第42位(从右往左)* )<br>
1.1 nextAvail不为负数时,说明已经计算好了(可能时第一次分配)此时直接返回即可,否则进入1.2<br>
1.2 遍历bitmap, 找到bitmap不全为1的long, 然后在这个long中找到位为0的位置, 即findNextAvail0中, 找到之后,返回`baseVal | j`, baseVal为当前long的起始idx, j为0在当前long的位置, 故此处直接或就可以得到0在整个bitmap中的位置.

2. 找到了0之后,将此处的0置为1,即`bitmap[q] |= 1L << r`.

3. 如果该Page经过这次分配无法再分配了, 那么从subpagePool中移除.

4. 返回一个handle的long值, 前32位表示在Subpage中的位置, 后32位表示Page在Chunk中的位置.

ok,此处返回了handle之后, 我们再回过头看Chunk的逻辑:
```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    final long handle;
    if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
        handle =  allocateRun(normCapacity);
    } else {
        handle = allocateSubpage(normCapacity); // 我们刚刚分析的是这个方法
    }

    if (handle < 0) {
        return false;
    }
    ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
    initBuf(buf, nioBuffer, handle, reqCapacity); 
    return true;
}
```
我们刚刚分析的是allocateSubpage的一系列方法, 接下来我们看allocateRun方法, allocateRun方法比较简单, 其中allocateNode
方法我们已经分析过了, 就不再赘述了,我们快速过一遍即可.
```java
private long allocateRun(int normCapacity) {
    int d = maxOrder - (log2(normCapacity) - pageShifts); // 找到第几层
    int id = allocateNode(d); // 在该层上分配, 获取id, id为memoryMap的索引
    if (id < 0) {
        return id;
    }
    freeBytes -= runLength(id); // freeBytes减去id节点分配的大小
    return id;
}

private int runLength(int id) {
    return 1 << log2ChunkSize - depth(id); // id代表的节点有多大
}
```

ok, 家下来我们进入`initBuf(buf, nioBuffer, handle, reqCapacity); `方法, 分析PooledByteBuffer是如何根据分配到的handle初始化的.

### PooledByteBuf在Chunk中的初始化
```java
void initBuf(PooledByteBuf<T> buf, ByteBuffer nioBuffer, long handle, int reqCapacity) {
    int memoryMapIdx = memoryMapIdx(handle);// 获取handle的低32位
    int bitmapIdx = bitmapIdx(handle);// 获取高32位
    if (bitmapIdx == 0) {
        byte val = value(memoryMapIdx); // memoryMap[memoryMapIdx]
        assert val == unusable : String.valueOf(val);
        buf.init(this, nioBuffer, handle, runOffset(memoryMapIdx) + offset,
                reqCapacity, runLength(memoryMapIdx), arena.parent.threadCache());
    } else {
      int memoryMapIdx = memoryMapIdx(handle);
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        buf.init(
            this, nioBuffer, handle,
            runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize + offset,
                reqCapacity, subpage.elemSize, arena.parent.threadCache());
    }
}

private int runOffset(int id) {
    // represents the 0-based offset in #bytes from start of the byte-array chunk
    int shift = id ^ 1 << depth(id);
    return shift * runLength(id);
}

private int runLength(int id) {
    // represents the size in #bytes supported by node 'id' in the tree
    return 1 << log2ChunkSize - depth(id);
}
```
我们来分析一下initBuf方法.
根据bitmapIdx是否为0分为两个分支, bitmapIdx如果为0说明是第一次在Page上分配或者是normal分配, 否则则是Page已经有分配, 所以要计算一个位移.
我们可以对比两个分支的buf.init方法的参数的不同.
1. 第一次分配或normal分配:`runOffset(memoryMapIdx) + offset`,runOffset根据memoryId计算二叉树节点在Chunk中的位移, `offset`则是
Chunk中实际开始分配的内存相对于memory的位移(详见PoolArena#offsetCacheLine), 两者加在一起则是此次分配在Chunk中的位移.<br>
第二次分配则是:`unOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize + offset`, 比第一次多了个
`(bitmapIdx & 0x3FFFFFFF) * subpage.elemSize`, Chunk的最大大小是1<<30(见PooledByteBufAllocator#MAX_CHUNK_SIZE), 所以bitmapIdx一定在1<<30之内, 而0x3FFFFFFF为2^30 - 1.这个多出来的位移正是Subpage上其他分配的内存所占用的.
2. 第一次分配或normal分配:`runLength(memoryMapIdx)`, 这个参数的意思是分配的最大大小, 和reqCapacity相对照.`subpage.elemSize`也是同样.

最终会调用到pooledBytebuf的init0方法:
```java
private void init0(PoolChunk<T> chunk, ByteBuffer nioBuffer,
                    long handle, int offset, int length, int maxLength, PoolThreadCache cache) {
    assert handle >= 0;
    assert chunk != null;

    this.chunk = chunk;
    memory = chunk.memory;
    tmpNioBuf = nioBuffer;
    allocator = chunk.arena.parent;
    this.cache = cache;
    this.handle = handle;
    this.offset = offset;
    this.length = length;
    this.maxLength = maxLength;
}
```
ok, 仅仅是设置了一些属性值, PooledByteBuf及其子类的实现都比较简单, 在此不再赘述.

ok以上就是netty内存池内存分配相关的代码解读, 接下来我们开始解读另一半: 内存的回收与释放.
我们按照: Subpage -> Chunk -> ChunkList -> Arena的顺序解读内存的回收.

## 内存的回收
### Subpage内存的释放
```java
boolean free(PoolSubpage<T> head, int bitmapIdx) {
    if (elemSize == 0) {
        return true;
    }
    int q = bitmapIdx >>> 6;
    int r = bitmapIdx & 63;
    assert (bitmap[q] >>> r & 1) != 0;
    bitmap[q] ^= 1L << r; // 将bitmap中的位置为0

    setNextAvail(bitmapIdx); // 设置当前bitmapIdx为nextAvail

    if (numAvail ++ == 0) {// 又可以用了， 加入到SubpagePool中
        addToPool(head);
        return true;
    }

    if (numAvail != maxNumElems) { // Subpage还存在被分配的
        return true;
    } else {
        if (prev == next) { // 链表至少要保证一个存在
            return true;
        }

        doNotDestroy = false; // 可以释放了
        removeFromPool();
        return false;
    }
}
```
1. 首先将bitmap还原
2. 设置nextAvail
3. 如果必要,将当前Subpage加入到SubpagePool中去
4. 根据Subpage是否还在使用返回true or false

### Chunk内存的释放
```java
void free(long handle, ByteBuffer nioBuffer) {
    int memoryMapIdx = memoryMapIdx(handle);
    int bitmapIdx = bitmapIdx(handle);

    if (bitmapIdx != 0) { // 释放Subpage
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage != null && subpage.doNotDestroy;

        PoolSubpage<T> head = arena.findSubpagePoolHead(subpage.elemSize);
        synchronized (head) {
            if (subpage.free(head, bitmapIdx & 0x3FFFFFFF)) { // Subpage还在使用直接返回否则释放Subpage并且从pool中移除
                return;
            }
        }
    }
    freeBytes += runLength(memoryMapIdx);
    setValue(memoryMapIdx, depth(memoryMapIdx)); // 还原memoryMap节点值
    updateParentsFree(memoryMapIdx); // 更新祖先节点memoryMap

    if (nioBuffer != null && cachedNioBuffers != null &&
            cachedNioBuffers.size() < PooledByteBufAllocator.DEFAULT_MAX_CACHED_BYTEBUFFERS_PER_CHUNK) {
        cachedNioBuffers.offer(nioBuffer);
    }
}
```
释放Subpage的时候需注意是否是链表的最后一个, 如果是那么即使完全没有分配也继续留在链表里,否则从链表中移除
并且还原memoryMap, 这样就可以继续在Chunk中分配给其他请求.

为什么这里链表只需要一个就够了呢, 因为多个并不能带来什么增益反而会因为SUbpage个数爆炸得不到释放造成Chunk的可分配内存下降, 而保证只有一个
可以满足Arena分配内存的fast-path(避免重新在Chunk中分配).

### ChunkList中内存的释放
```java
boolean free(PoolChunk<T> chunk, long handle, ByteBuffer nioBuffer) {
    chunk.free(handle, nioBuffer);
    if (chunk.usage() < minUsage) {
        remove(chunk);
        return move0(chunk);
    }
    return true;
}

private boolean move0(PoolChunk<T> chunk) {
    if (prevList == null) {
        // There is no previous PoolChunkList so return false which result in having the PoolChunk destroyed and
        // all memory associated with the PoolChunk will be released.
        assert chunk.usage() == 0;
        return false;
    }
    return prevList.move(chunk);
}

private boolean move(PoolChunk<T> chunk) {
    assert chunk.usage() < maxUsage;

    if (chunk.usage() < minUsage) {
        // Move the PoolChunk down the PoolChunkList linked-list.
        return move0(chunk);
    }

    // PoolChunk fits into this PoolChunkList, adding it here.
    add0(chunk);
    return true;
}
```
调用上述的Chunk释放内存的方法, 主要的逻辑是根据释放后Chunk的内存使用情况,将其下移至其他ChunkList.
该方法的返回值:true代表Chunk不可释放,false代表`chunk.usage() == 0`

### Arena中内存的释放
```java
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass, ByteBuffer nioBuffer, boolean finalizer) {
    final boolean destroyChunk;
    ...
    if (destroyChunk) {
        // 调用unsafe相关方法释放Chunk内存
        destroyChunk(chunk);
    }
}
```

free单个PooledByteBuf
```java
void free(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle, int normCapacity, PoolThreadCache cache) {
    if (chunk.unpooled) {
        ...
    } else {
        SizeClass sizeClass = sizeClass(normCapacity);
        if (cache != null && cache.add(this, chunk, nioBuffer, handle, normCapacity, sizeClass)) { 
          // 调用线程本地缓存.将对应Chunk中handle对应的内存缓存起来不释放
            return;
        }

        freeChunk(chunk, handle, sizeClass, nioBuffer, false);
    }
}

/** PoolThreadCache#add **/
boolean add(PoolArena<?> area, PoolChunk chunk, ByteBuffer nioBuffer,
            long handle, int normCapacity, SizeClass sizeClass) {
    MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
    if (cache == null) {
        return false;
    }
    return cache.add(chunk, nioBuffer, handle);
}
```

## 总结
以上就是Netty的内存分配以及回收代码的解读, 
关于其中代码的细节部分可能会有很多解读不到位的地方, 欢迎各位在我的邮箱留言或者在github上提issue.