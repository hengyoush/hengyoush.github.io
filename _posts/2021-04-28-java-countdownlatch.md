---
layout: post
title:  "Java源码笔记-CountDownLatch"
date:   2021-04-28 21:20:00 +0700
categories: [java,juc]
---

## CountDownLatch的用处
CountDownLatch是一种同步机制, 让一个或多个线程等待其他线程执行的操作完成.
它与CyclicBarrier不同的一点在于它不可重置, 而CyclicBarrier可以.


## 源码解析

### 入口代码

```java
public void await() throws InterruptedException {
   sync.acquireSharedInterruptibly(1);
}

public void countDown() {
   sync.releaseShared(1);
}
```
和ReentrantLock一样, 它也定义了自己的Sync类, 该类也继承自AQS, 下面我们来看该类的实现.

> If the current count is zero then this method returns immediately. 
对于一个已经countdown完成的latch来说, 再次调用它的await方法会立即返回.

### Sync类解析

```java
private static final class Sync extends AbstractQueuedSynchronizer {
   Sync(int count) {
      setState(count);
   }

   protected int tryAcquireShared(int acquires) {
      return (getState() == 0) ? 1 : -1;
   }

   protected boolean tryReleaseShared(int releases) {
      // Decrement count; signal when transition to zero
      for (;;) {
            int c = getState();
            if (c == 0)
               return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
               return nextc == 0;
      }
   }
}
```
首先我们看到它和Semaphore一样实现了共享模式的方法, 为什么要实现共享模式呢, 接下来我们会明白的.  

它的构造器接收一个int型变量作为state的初始值, 但是它和Semaphore的作用是相反的.

接着看到tryAcquireShared方法, 初看它的实现可能和Semaphore比起来有点奇怪, 但是它符合上面await的语义:  
对于一个已经为0的latch, await会立即返回, 因此这里的逻辑是如果当前状态为0那么立即返回, 否则进入队列等待.  
从这里可以看出await方法并没有修改state.

那么对于它的release方法——即countdown方法来说, 它就不能和Semaphore一样去增加state了, 相反这里是减少state,
每次countdown都减一,这样如果state为0, 那么tryReleaseShared方法会返回true, 返回true会发生什么呢?  
根据我们之前对Semaphore的解析我们知道如果tryReleaseShared返回true, 那么会进入doReleaseShared方法
唤醒队列中的线程.







