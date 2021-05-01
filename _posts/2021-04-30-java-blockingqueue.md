---
layout: post
title:  "Java源码笔记-BlockingQueue"
date:   2021-04-30 23:00:00 +0700
categories: [java,juc]
---

## BlockingQueue的实现类
有三个:  
1. ArrayBlockingQueue, 特点:固定容量.
2. LinkedBlockingQueue, 特点:长度不固定,可无限存储.
3. SynchronousQueue, 特点:容量为0.

## ArrayBlockingQueue

### ArrayBlockingQueue重要字段
```java
// 底层数组,真正存储元素的地方
final Object[] items;
// 读指针
int takeIndex;
// 写指针
int putIndex;
// 元素数量
int count;
// 读写操作使用的锁
final ReentrantLock lock;
// 两个condition对象，用于队列非空、非满时通知等待的线程
private final Condition notEmpty;
private final Condition notFull;
```

示意图如下:
![avatar](/static/img/Java-ArrayBlockingQueue示意图.png)

### ArrayBlockingQueue源码分析
```java
public boolean offer(E e) {
	checkNotNull(e);
	final ReentrantLock lock = this.lock;
	lock.lock(); // 加锁
	try {
		if (count == items.length)
			return false; // 队列已满
		else {
			enqueue(e); // 主要逻辑
			return true;
		}
	} finally {
		lock.unlock(); // 解锁
	}
}
```

主要逻辑在enqueue中。
```java
private void enqueue(E x) {
	final Object[] items = this.items;
	items[putIndex] = x;
	if (++putIndex == items.length) // 递增写指针，如果到达数组末尾，从头开始
		putIndex = 0;
	count++;
	notEmpty.signal(); // 通知等待take的线程有数据进来了
}
```
这里需要关注的一点是到达数组末尾，写指针会从头开始，但是这里不会覆盖到还没有读取的元素，因为在外层方法会进行判断，如果满了
会直接返回。


```java
public E take() throws InterruptedException {
	final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		while (count == 0) // 如果队列为空，释放锁等待
			notEmpty.await();
		return dequeue(); // 真正的逻辑
	} finally {
		lock.unlock();
	}
}
```
```java
private E dequeue() {
	final Object[] items = this.items;
	E x = (E) items[takeIndex];
	items[takeIndex] = null; // 读取之后将数组该位置设置为null
	if (++takeIndex == items.length) // 递增takeIndex，超过数组长度，重新回到数组开始位置
		takeIndex = 0;
	count--;
	if (itrs != null)
		itrs.elementDequeued();
	notFull.signal(); // 通知等待put的线程
	return x;
}
```


## LinkedBlockingQueue

### 重要字段

使用链表实现, 节点Node类定义如下:
```java
static class Node<E> {
   E item;

   Node<E> next;

   Node(E x) { item = x; }
}
```
重要字段如下:
```java
// 当前元素数量
private final AtomicInteger count = new AtomicInteger();
// 链表头
Node<E> head;
// 链表尾
Node<E> last;
//  读锁
private final ReentrantLock takeLock = new ReentrantLock();
// 写锁
private final ReentrantLock putLock = new ReentrantLock();
// 两个condition对象，用于队列非空、非满时通知等待的线程
private final Condition notFull = putLock.newCondition();
private final Condition notEmpty = takeLock.newCondition();
```
LinkedBlockingQueue使用两个锁来保证线程安全, 其中takeLock在获取元素的时候
加锁, putLock在添加元素的时候加锁.实现了锁粒度的细分, 提高了并发度.

put的时候,不断在last链表尾增加元素.
take的时候,从链表头开始获取.

下面我们分别从put和take两个方面来分析.

### 源码分析

如下是put方法的实现:

```java
public void put(E e) throws InterruptedException {
   int c = -1;
   Node<E> node = new Node<E>(e);
   final ReentrantLock putLock = this.putLock;
   final AtomicInteger count = this.count;
   // 加锁
   putLock.lockInterruptibly();
   try {
      // 如果当前队列已满,那么在notFull这个条件上等待.
      while (count.get() == capacity) { 
            notFull.await();
      }
      // 真正入队的逻辑
      enqueue(node); 
      // 递增count,如果小于容量,唤醒在notFull这个条件上等待的线程
      c = count.getAndIncrement(); 
      if (c + 1 < capacity)
            notFull.signal();
   } finally {
      putLock.unlock();
   }
   // 如果c为0,代表在count.getAndIncrement()返回的是0,而现在有了一个元素,这里
   // 对等待take的线程进行唤醒.
   if (c == 0)
      signalNotEmpty();
}

private void enqueue(Node<E> node) {
   // assert putLock.isHeldByCurrentThread();
   last = last.next = node;
}
```
逻辑大致总结如下:  
1. 如果当前队列已满,那么等待take释放空间.
2. 如果队列没满,这时在链表尾追加元素,修改tail.
3. 递增count,如果没满那么唤醒put线程,如果刚刚put的是第一个元素,那么唤醒take线程.

```java
public E take() throws InterruptedException {
   E x;
   int c = -1;
   final AtomicInteger count = this.count;
   final ReentrantLock takeLock = this.takeLock;
   // 加锁
   takeLock.lockInterruptibly();
   try {
      // 如果队列中没有元素,那么等待put线程
      while (count.get() == 0) {
            notEmpty.await();
      }
      // 执行出队
      x = dequeue();
      // 递减count,如果队列中还有元素,那么唤醒take线程
      c = count.getAndDecrement();
      if (c > 1)
            notEmpty.signal();
   } finally {
      takeLock.unlock();
   }
   // 刚刚的出队让满队列变为不满的队列,唤醒put线程.
   if (c == capacity)
      signalNotFull();
   return x;
}

private E dequeue() {
   Node<E> h = head;
   Node<E> first = h.next;
   h.next = h; // help GC
   head = first;
   E x = first.item;
   first.item = null;
   return x;
}
```

逻辑大致总结如下:  
1. 如果当前队列为空,那么等待put新增元素.
2. 如果队列有元素,执行出队,修改head.
3. 递减count,如果还有元素那么唤醒take线程,如果刚刚take的元素让满队列变为非满,那么唤醒put线程.

可以看到,take和put之间不需要同步的前提是:
它们修改的状态是分离的(除了count),put修改tail,take修改head;而它们共同修改的变量count则是Atomic变量.


## SynchronousQueue
SynchronousQueue的每一个put操作都必须等待另一个线程的take操作,可以将其看作没有容量的阻塞队列.
对于没有容量的队列,任何take操作都无法获取元素,因为队列中没有元素;任何put操作也无法成功,因为队列永远处于
满队列的状态.

支持非公平和公平两种模式,不同模式内部实现的数据结构也不同,下面我们着重说一下非公平模式的实现.

内部重要字段只有一个:`Transferer<E> transferer;`,Transferer定义如下:  
```java
abstract static class Transferer<E> {
   // e为null代表
   abstract E transfer(E e, boolean timed, long nanos);
}
```




有两种实现,我们着重说一下非公平模式的实现.

### 非公平模式源码解读

#### TransferStack解读
首先看节点类的状态定义:
```java
/** 表示一个尚未满足的consumer */
static final int REQUEST    = 0;
/** 表示一个尚未满足的producer */
static final int DATA       = 1;
/** 表示这个节点正在满足另外一个未满足的DATA或者REQUEST节点 */
static final int FULFILLING = 2;
```

节点的内部结构定义如下

```java
static final class SNode {
   volatile SNode next;        // 栈中的下一个节点
   volatile SNode match;       // 正在满足这个节点的另一个节点
   volatile Thread waiter;     // 需要等待满足的线程
   Object item;                // 具体的数据,对于REQUEST来说是null
   int mode;                   // 不再赘述
   // ...
}
```

Snode的其他基础方法:
```java
void tryCancel() {
      UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
}
```
取消操作把match字段cas为自身.

尝试将s作为自己的匹配节点,返回是否成功匹配.
```java
boolean tryMatch(SNode s) {
      if (match == null &&
         UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) { // CAS自身的match字段为s
         Thread w = waiter;
         if (w != null) {    // 唤醒等待线程
            waiter = null;
            LockSupport.unpark(w);
         }
         return true;
      }
      return match == s; // 可能CAS失败,因为可能发生取消
}
```

#### 核心方法transfer解读

```java
E transfer(E e, boolean timed, long nanos) {
   /**
      有三种状况:
      1. 如果当前栈为空或者栈中的元素与当前的模式相同,那么此时尝试向栈上push一个新节点
      2. 如果当前存在一个互补的节点,那么尝试push一个fulfilling状态的节点到栈上, 和互补的节点匹配,
      最后将互补的节点和刚刚push的节点一起从栈上pop出来. 当然可能其他线程“帮助”完成了这一操作,即第三个情况
      3. 如果当前栈顶已经存在了另外一个fulfilling的节点,那么帮助它完成match和pop.
   */
   SNode s = null;
   int mode = (e == null) ? REQUEST : DATA;

   for (;;) {
         SNode h = head;
         if (h == null || h.mode == mode) {  // 空 或者 相同模式
            if (timed && nanos <= 0) {      // 超时了
               if (h != null && h.isCancelled())
                     casHead(h, h.next);     // pop出来取消的节点
               else
                     return null;
            } else if (casHead(h, s = snode(s, e, h, mode))) {
               SNode m = awaitFulfill(s, timed, nanos); // 等待匹配
               if (m == s) {               // 是否已取消
                     clean(s);
                     return null;
               }
               if ((h = head) != null && h.next == s) // 未取消, 帮助pop
                     casHead(h, s.next);
               return (E) ((mode == REQUEST) ? m.item : s.item);
            }
         } else if (!isFulfilling(h.mode)) { // mode不同,尝试满足
            if (h.isCancelled())            // 已取消
               casHead(h, h.next);         // pop然后重试
            else if (casHead(h, s=snode(s, e, h/*next*/, FULFILLING|mode))) { // push一个FULFILLING节点
               for (;;) { // 循环直到1.匹配或者2.互补节点不存在
                     SNode m = s.next;       // m is s's match
                     if (m == null) {        // 互补节点不存在
                        casHead(s, null);   // 将刚刚push的FULFILLING节点pop出来.重试
                        s = null;           
                        break;              
                     }
                     SNode mn = m.next; // 互补节点还在,尝试match,如果成功那么pop出自身和互补节点
                     if (m.tryMatch(s)) {
                        casHead(s, mn);
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                     } else                 
                        s.casNext(m, mn);   // 将m.next设置为当前节点的后继,继续尝试满足
               }
            }
         } else {                            // 帮助满足线程
            SNode m = h.next;               
            if (m == null)                  // waiter取消了,pop出fulfilling节点
               casHead(h, null);           
            else {
               SNode mn = m.next;
               if (m.tryMatch(h))          // waiter还在,帮助match和pop
                     casHead(h, mn);        
               else                        
                     h.casNext(m, mn);       
            }
         }
   }
}
```

对于情况2我画了一张示意图:
一开始,栈中包含数个未满足的REQUEST请求:
![avatar](/static/img/JAVA-TransferStack工作流程-1.png)

接着一个DATA请求来了,发现栈上有互补模式的节点,它创建一个新的Fulfilling模式的节点
push到栈上.
![avatar](/static/img/JAVA-TransferStack工作流程-2.png)

最后将栈上的节点pop出去.
![avatar](/static/img/JAVA-TransferStack工作流程-3.png)

#### 公平模式代码简单解读

公平模式为了保证公平性使用的数据结构是队列而不是栈.
如下是示意图:
![avatar](/static/img/JAVA-TransferQueue示意图.png)


队列节点定义:

```java
static final class QNode {
   volatile QNode next;          // 队列的下一个节点
   volatile Object item;         // 如果是take此处会从null被更新为non-null,put相反
   volatile Thread waiter;       // 等待的线程
   final boolean isData;         // take-false,put-true
   // ...
}
```

transfer方法解读:

```java
E transfer(E e, boolean timed, long nanos) {
   /**
      主要分为两种情况:
      1. 如果队列为空或者持有相同mode的节点,那么尝试新增节点加入到队列中并且等待被满足(或者主动取消)
      2. 如果队列中包含的是互补节点,那么尝试通过CAS互补节点的item字段,并且出队,然后返回匹配的item.
   */
   QNode s = null; // constructed/reused as needed
   boolean isData = (e != null);

   for (;;) {
         QNode t = tail;
         QNode h = head;
         if (h == t || t.isData == isData) { // 空或者是队列中只含有相同模式的节点
            QNode tn = t.next;
            // .. 省略了一些一致性校验,我们这里只关心主流程
            if (s == null)
               s = new QNode(e, isData);
            if (!t.casNext(null, s))        // 入队,尝试更新tail的next字段
               continue;

            advanceTail(t, s);              // 更新tail的指向
            Object x = awaitFulfill(s, e, timed, nanos); // 等待被满足
            if (x == s) {                   // 主动取消
               clean(t, s);
               return null;
            }

            if (!s.isOffList()) {           // 出队
               advanceHead(t, s);          
               if (x != null)              
                     s.item = s;
               s.waiter = null;
            }
            return (x != null) ? (E)x : e;

         } else {                            // 如果队列中含有的是互补模式的节点
            QNode m = h.next;               // 从队头开始匹配,m即要匹配的节点

            Object x = m.item;
            if (isData == (x != null) ||    // m的item已经被更新,说明已经在匹配了
               x == m ||                   // m的等待节点取消了等待.
               !m.casItem(x, e)) {         // 尝试更新m的item字段失败
               advanceHead(h, m);          // 出队重试
               continue;
            }
            advanceHead(h, m);              // 成功的更新了m的item字段,说明匹配成功
            LockSupport.unpark(m.waiter); // 唤醒等待线程
            return (x != null) ? (E)x : e;
         }
   }
}
```