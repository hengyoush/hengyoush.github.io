---
layout: post
title:  "Java源码笔记-CyclicBarrier"
date:   2021-04-29 22:40:00 +0700
categories: [java,juc]
---

## CyclicBarrier的作用

CyclicBarrier可以用来让一组线程彼此等待, 直到所有线程都都到达一个屏障点. CyclicBarrier之所以前面有一个Cyclic是因为它在所有等待线程释放之后可以重用.  
CyclicBarrier支持一个可选的Runnable类型的参数, 在最后一个线程到达屏障点之后在第一个线程释放之前会执行这个参数中的逻辑.

## 源码解析

### 内部字段
```java
/** The lock for guarding barrier entry */
private final ReentrantLock lock = new ReentrantLock();
/** 等待一次屏障触发 */
private final Condition trip = lock.newCondition();
/** 线程数量 */
private final int parties;
/* 当屏障触发时运行的命令 */
private final Runnable barrierCommand;
/** 当前generation */
private Generation generation = new Generation();
/** 当前正在屏障上等待的线程数量,每个generation从parties降为0. */
private int count;

private static class Generation {
   boolean broken = false;
}
```

### 主要流程解析
```java
public int await() throws InterruptedException, BrokenBarrierException {
   try {
      return dowait(false, 0L);
   } catch (TimeoutException toe) {
      throw new Error(toe); // cannot happen
   }
}

private int dowait(boolean timed, long nanos) {
   final ReentrantLock lock = this.lock;
   // 加锁,下面要对count和generation做修改,保证原子性
   lock.lock();
   try {
      final Generation g = generation;

      // 异常流程,忽略

      int index = --count;
      if (index == 0) {  // 如果count为0,那么代表触发了屏障,此时要先处理barrierCommand
            boolean ranAction = false;
            try {
               final Runnable command = barrierCommand;
               if (command != null)
                  command.run();
               ranAction = true;
               nextGeneration(); // 在这里唤醒等待的线程
               return 0;
            } finally {
               if (!ranAction)
                  breakBarrier();
            }
      }

      // 如果不是最后一个线程,那么进入等待
      for (;;) {
            try {
               if (!timed)
                  trip.await();
               else if (nanos > 0L)
                  nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
               // 异常流程,忽略
            }
            // 异常流程,忽略
      }
   } finally {
      lock.unlock();
   }
}

private void nextGeneration() {
   // 唤醒所有线程
   trip.signalAll();
   // 重置count
   count = parties;
   generation = new Generation();
}
```

主要逻辑如下:  
1. 将count减一,判断count是否为0,即当前线程是否是最后一个到达的线程,如果是那么执行barrierCommand,
并且之后在nextGeneration方法中唤醒所有等待的线程,并且重置count,返回.
2. 如果当前线程不是最后一个,那么进入在trip上等待,直到唤醒或者超时.


