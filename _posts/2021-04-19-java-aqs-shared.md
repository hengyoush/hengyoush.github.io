---
layout: post
title:  "Java同步组件AQS原理详解——共享模式"
date:   2021-04-19 21:57:00 +0700
categories: [java]
---

## 前言
与可重入锁一样，Semaphore的内部也只有一个继承自AQS的Sync类， 也分为公平和非公平模式， 但是和可重入锁不同，
Semaphore对state的操作不同，state在信号量初始化时就设置了一个正整数，表示可以获得的最大资源，每次调用
acquire()或者acquire(int)方法实际上是获取了一定数量的信号量，如果当前信号量不足那么会阻塞等待其他线程释放信号量。

在信号量实现中，同步器也分为公平和非公平模式，我们这里只描述非公平模式的工作流程。

## 请求信号量
```java
public void acquire() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}
```
调用的是AQS的acquireSharedInterruptibly方法，
```java
public final void acquireSharedInterruptibly(int arg)
		throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
	if (tryAcquireShared(arg) < 0)
		doAcquireSharedInterruptibly(arg);
}
```
先不关注关于线程中断的处理，可以看到这里先调用了tryAcquireShared方法，该方法的返回值如果是
正数或者0代表请求成功，如果小于0代表请求失败，这里请求失败的话会进入队列等待。  
tryAcquireShared和tryAcquire一样需要同步器自定义实现，我们先看tryAcquireShared方法的实现：
```java
protected int tryAcquireShared(int acquires) {
	return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
	for (;;) {
		int available = getState();
		int remaining = available - acquires;
		if (remaining < 0 ||
			compareAndSetState(available, remaining))
			return remaining;
	}
}
```
实际上调用了nonfairTryAcquireShared方法：
1. 首先获取当前可用的state
2. 将当前可用的state减去需要的信号量得到剩余信号量
3. 如果大于等于0那么尝试直接CAS否则返回剩余的值，如果成功直接返回否则重试
4. 重试直到剩余数量小于0或者CAS成功。

假设这里剩余数量小于0，回到代码：
```java
if (tryAcquireShared(arg) < 0)
	doAcquireSharedInterruptibly(arg);
```
进入doAcquireSharedInterruptibly方法，该方法是AQS内部实现：
```java
private void doAcquireSharedInterruptibly(int arg)
	throws InterruptedException {
	final Node node = addWaiter(Node.SHARED); // @1
	boolean failed = true;
	try {
		for (;;) {
			final Node p = node.predecessor(); // @2
			if (p == head) {
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					failed = false;
					return;
				}
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				throw new InterruptedException();
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```
1. 创建了一个共享模式的节点，加入AQS的内部链表中
2. 进入循环尝试，如果当前节点的前面是Head节点，说明此时有可能竞争成功，此时调用tryAcquireShared方法，该方法上面已经描述，此处不再赘述。如果
成功那么调用setHeadAndPropagate方法，将当前节点设置为Head，并且可能唤醒后续节点（这里我们之后再详述）。
3. 如果前驱节点不是Head或者是Head但是尝试获取失败，那么会进入shouldParkAfterFailedAcquire方法，将前驱节点的状态更新为SIGNAL，然后再尝试一遍当前的逻辑，
如果还是获取不到那么就阻塞自己，等待其他线程释放唤醒自己。

要注意共享模式和独占模式的区别在于，共享模式允许多个Node同时持有state的一部分，
在一个线程获取成功时可能其他线程在释放信号量，那么要注意此时入队的线程，如果可以让它获得信号量的话一定要唤醒它。

具体进入setHeadAndPropagate方法：
```java
private void setHeadAndPropagate(Node node, int propagate) {
	Node h = head; 
	// 更新head，相当于出队
	setHead(node);
	// 当1.剩余信号量大于0
	// 2.当前head是PROPAGATE或SIGNAL
	// 3.当前节点是SIGNAL
	// 唤醒后继节点（如果有的话）
	if (propagate > 0 || h == null || h.waitStatus < 0 ||
		(h = head) == null || h.waitStatus < 0) {
		Node s = node.next;
		if (s == null || s.isShared())
			doReleaseShared();
	}
}
```
在这里做了一系列判断是否要唤醒后继，如果这里判断不对，而恰好在判断时，后继节点即将阻塞而释放线程
在其即将阻塞的时候释放，那么这时后继节点永远也得不到唤醒。
在这里我们有几个比较重要的判断propagate > 0、oldHead.waitStatus < 0, newHead.waitStatus < 0
对于第一个条件，比较好理解，因为这意味着还有剩余的信号量，所以去唤醒。
对于第二个条件，oldHead的status为负数代表其为SIGNAL或者PROPAGATE，如果是SIGNAL代表有后继节点等待唤醒，这时去唤醒，
如果为PROPAGATE，那么代表此时有另外一个线程在释放，将Head的状态由0更新为PROPAGATE
这里的意思是虽然没有后继节点但是可能后继节点还没有更新前驱的状态为SIGNAL，设置这个值就是尽快让后继节点苏醒。
第三个条件比较好理解，因为此时更新当前节点的状态的只有后继节点或者新的释放线程，要么更新为SIGNAL代表需要唤醒后继，
要么为PROPAGATE，代表有线程释放了信号量，这两种情况都要唤醒后继。

具体如何唤醒呢，下面进入了doReleaseShared方法,该方法在线程释放信号量时也会调用。
```java
private void doReleaseShared() {
	for (;;) {
		// 获取当前head
		Node h = head;
		// 队列中存在等待节点的话进入下面的逻辑
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			// 如果是SIGNAL代表有节点等待被唤醒
			if (ws == Node.SIGNAL) {
				// 尝试CAS，如果失败，那么可能其他线程也执行到了这里，重试
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;
				// 唤醒后继节点	
				unparkSuccessor(h);
			}
			// 如果ws==0，意味着后面可能存在节点在尝试获取但是可能还没有
			// 更新前驱为SIGNAL，此时为了快速唤醒线程，更新为PROPAGATE
			else if (ws == 0 &&
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;
		}
		if (h == head) 
			break;
	}
}
```
网上有人说使用PROPAGATE是为了防止在两个线程请求两个线程释放的情况下队列hang住的问题，实际上即使
不使用PROPAGATE在setHeadAndPropagate方法中的5个判断也完全不会发生hang住的情况。

## 总结
以上就是Semaphore以及AQS共享模式的内部原理, 接下来我会接着对读写锁、CountDownLatch、CyclicBarrier进行分析.















