---
layout: post
title:  "Java源码笔记-ReentrantReadWriteLock"
date:   2021-04-28 22:00:00 +0700
categories: [java,juc]
---

## ReentrantReadWriteLock的要点

- 可重入性: 该锁允许读锁重入读锁、写锁可重入写锁.除此之外,一个写锁可以获取读锁,但是相反不行,即一个读锁不可重入写锁.
- 锁降级: 写锁可以降级为读锁, 通过获取了一个写锁然后获取读锁,然后释放写锁.锁升级锁不可以的.

## 代码示例

下面的代码片段锁JDK官方的示例片段, 它的逻辑大意是: 读取缓存数据, 如果发现缓存数据失效那么尝试更新它.  
```java
 class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
       // 在获取写锁之前必须先释放读锁
       rwl.readLock().unlock();
       rwl.writeLock().lock();
       try {
         // 获取写锁之后要再进行检查
         if (!cacheValid) {
           data = ...
           cacheValid = true;
         }
         // 通过在释放写锁之前获取读锁完成从写锁到读锁的降级.
         rwl.readLock().lock();
       } finally {
          // 释放写锁, 依然持有读锁
         rwl.writeLock().unlock(); 
       }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }
```

## 源码解析

### 外围方法解析

首先看它的成员变量.  
```java
private final ReentrantReadWriteLock.ReadLock readerLock;
private final ReentrantReadWriteLock.WriteLock writerLock;
final Sync sync; // 自定义同步器
```


接下来看ReentrantReadWriteLock以及内部类ReadLock和WriteLock的构造器.    
```java
public ReentrantReadWriteLock(boolean fair) {
   sync = fair ? new FairSync() : new NonfairSync();
   readerLock = new ReadLock(this);
   writerLock = new WriteLock(this);
}
protected ReadLock(ReentrantReadWriteLock lock) {
   sync = lock.sync;
}
protected WriteLock(ReentrantReadWriteLock lock) {
   sync = lock.sync;
}
```
ReentrantReadWriteLock同样实现了一个继承自AQS的同步器, 并且它的读锁和写锁使用的都是同一个Sync对象.

下面我们看ReadLock和WriteLock的lock和unlock方法.  
```java
// ReadLock::lock
public void lock() {
   sync.acquireShared(1);
}
// ReadLock::unlock
public void unlock() {
   sync.releaseShared(1);
}

// WriteLock::lock
public void lock() {
   sync.acquire(1);
}
// WriteLock::unlock
public void unlock() {
   sync.release(1);
}
```
可以看到对于ReadLock它的加锁和解锁都是基于共享模式的, 对于WriteLock来说它的加锁和解锁都是基于独占模式的.

下面我们来看它的Sync同步器实现.

### Sync自定义同步器

该同步器中的state分为两部分,即32位分为高16位和低16位, 高16位表示读锁持有者数量, 低16位表示写锁持有量.  
另外每个线程都有一个读锁计数器(类为HoldCounter), 即ThreadLocal变量readHolds.
```java
static final class HoldCounter {
   int count = 0;
   final long tid = getThreadId(Thread.currentThread());
}

static final class ThreadLocalHoldCounter
   extends ThreadLocal<HoldCounter> {
   public HoldCounter initialValue() {
         return new HoldCounter();
   }
}

private transient ThreadLocalHoldCounter readHolds;
```
加入这个变量是为了实现getReadHoldCount方法的,该方法返回当前线程持有读锁的重入次数.因为不是主要逻辑,我们在此分析中会把这部分略过.

我们首先看独占锁的获取与释放.

### 独占锁的获取与释放

```java
protected final boolean tryAcquire(int acquires) {
   /*
      * Walkthrough:
      * 1. If read count nonzero or write count nonzero
      *    and owner is a different thread, fail.
      * 2. If count would saturate, fail. (This can only
      *    happen if count is already nonzero.)
      * 3. Otherwise, this thread is eligible for lock if
      *    it is either a reentrant acquire or
      *    queue policy allows it. If so, update state
      *    and set owner.
      */
   Thread current = Thread.currentThread();
   int c = getState();
   int w = exclusiveCount(c);
   if (c != 0) { // 存在读锁或写锁持有者
         // (注意: 如果 c != 0 并且 w == 0 则说明 shared count != 0, 获取写锁失败)
         if (w == 0 || current != getExclusiveOwnerThread())
            return false;
         // 经过上面的过滤此处已经可以确认没有读锁了
         if (w + exclusiveCount(acquires) > MAX_COUNT) // 重入超过最大值
            throw new Error("Maximum lock count exceeded");
         // 重入获取
         setState(c + acquires);
         return true;
   }
   // 尝试CAS获取锁,如果成功则返回true
   if (writerShouldBlock() ||
         !compareAndSetState(c, c + acquires))
         return false;
   setExclusiveOwnerThread(current);
   return true;
}
```
基本逻辑:
1. 判断当前是否存在读锁, 或者是否其他线程持有写锁, 如果是这两个情况中的一种, 返回false.
2. 如果是重入, 那么返回true
3. 到这里说明不存在写锁以及读锁, 此时尝试CAS更新state, 如果成功返回true否则返回false.


下面是释放逻辑:

```java
protected final boolean tryRelease(int releases) {
   // 是否占有写锁,如果没有,抛出异常
   if (!isHeldExclusively())
         throw new IllegalMonitorStateException();
   int nextc = getState() - releases;
   boolean free = exclusiveCount(nextc) == 0;
   if (free)
         setExclusiveOwnerThread(null);
   setState(nextc);
   return free;
}
```
1. 首先判断当前线程是否占有写锁,如果没有,抛出异常
2. 更新state, 如果state为0,那么返回true,AQS下一步会唤醒后继线程.

### 共享锁的获取与释放

```java
protected final int tryAcquireShared(int unused) {
   Thread current = Thread.currentThread();
   int c = getState();
   // 当前有线程占有写锁, 返回失败
   if (exclusiveCount(c) != 0 &&
         getExclusiveOwnerThread() != current)
         return -1;
   // 尝试CAS更新state
   int r = sharedCount(c);
   // readerShouldBlock在下一个等待线程是请求写锁时会阻塞自己,防止饥饿
   if (!readerShouldBlock() &&
         r < MAX_COUNT &&
         compareAndSetState(c, c + SHARED_UNIT)) {
         // ... 更新计数器逻辑,略过
         return 1;
   }
   // CAS失败, 进入下面的方法,该方法有无限循环
   return fullTryAcquireShared(current);
}

// Non-fair的逻辑,即使是非公平实际上也有公平性至少不会让请求写锁的线程一直得不到处理
final boolean readerShouldBlock() {
   return apparentlyFirstQueuedIsExclusive();
}

final boolean apparentlyFirstQueuedIsExclusive() {
   Node h, s;
   return (h = head) != null &&
      (s = h.next)  != null &&
      !s.isShared()         && // head.next是独占模式的节点返回true
      s.thread != null;
}
```
基本逻辑如下:
1. 判断当前是否有线程占有写锁, 返回失败.
2. 尝试CAS更新state,如果成功直接返回否则进入fullTryAcquireShared方法中.

这里有一个值得注意的点, readerShouldBlock体现了一个防饥饿设计, 假设当前
该锁被读锁占据, 模式是非公平的, 队列中有很多写锁正在等待读锁释放, 但是此时不断有
读锁请求, 由于此时是共享模式, 那么读锁请求一定成功, 即使读锁请求不是那么频繁,
但只要有连续但读锁请求, 那么就会发生写锁一直得不到的情况, 即“饥饿情况”的出现.

为了防止这一现象, 实现中会判断当前队列头节点的下一个是不是独占模式节点,
这个独占模式节点正在等待当前读线程的释放, 如果是那么这个读线程乖乖阻塞,
给请求写锁的线程一点机会.

下面我们看fullTryAcquireShared方法,该方法除了有一个循环大体逻辑和上面的有点重复:
```java
        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    // 其他线程占有写锁,返回失败
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                } else if (readerShouldBlock()) {
                   if (非重入) {
                      return -1; // 当前非重入且根据readerShouldBlock判断应该阻塞,那么进入队列中
                   } 
                   // 否则当前是重入状态,那么进入下面的CAS更新步骤
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                   // ...计数器逻辑
                   return 1;
                }
            }
        }
```
1. 先判断其他线程是否占有写锁,如果是返回失败.
2. 判断当前是否是非重入状态,如果是那么根据readerShouldBlock的结果,我们应该阻塞.
3. 否则如果当前没有写线程在等待或者当前是重入状态,那么直接尝试CAS更新,如果失败再次循环,否则返回成功.






