---
layout: post
title:  "Java关键字synchornized原理解析"
date:   2021-04-18 16:30:00 +0700
categories: [java]
---

## 字节码
synchornized在编译为class文件时, 对应为两个字节码: monitorenter和monitorexit.

## 对象头
![avatar](/static/img/对象头布局.png)

## 锁的分类
锁的级别从高到底分别是:
- 无锁
- 偏向锁
- 轻量级锁
- 重量级锁

对于不同级别的锁状态其MarkWord中的内容也不同, 具体如下.

**无锁状态**
|25bit|4bit|1bit|2bit|  
|-|-|-|-|  
|对象的hashCode|对象的分代年龄|0 是否是偏向锁|01 锁标志位|


**偏向锁状态**
|23bit|2bit|4bit|1bit|2bit|  
|-|-|-|-|-|  
|线程id|epoch|对象的分代年龄|1 是否是偏向锁|01 锁标志位|

**轻量级锁**
|30bit|2bit|  
|-|-|  
|指向栈中锁记录的指针|00|

**重量级锁**
|30bit|2bit|  
|-|-|  
|指向互斥量的指针|10|

### 锁记录与MarkWord
如图所示,该图表达的是如下程序当前的锁记录与锁对象之间的关系.
```java
synchronized(obj) {
    synchronized (obj) {
        synchronized (obj) {
            // ...
        }
    }
}
```
![avatar](/static/img/JAVA关键字synchronized-锁记录结构.png)



### 偏向锁
线程在获取锁时会尝试在对象头中设置自己的线程id, 如果设置成功那么之后线程进入退出同步块时就不需要通过CAS来加锁、解锁等,只需要看一下
MarkWord中是否包含自己的线程id即可, 具体流程如下:
1. 判断当前MarkWord中是否已偏向当前线程, 如果是那么获取锁成功.
2. CAS将偏向线程改为当前线程(使用的旧值为匿名偏向,即没有偏向任何线程), 如果成功直接返回, 否则存在多线程竞争, 进入轻量级锁模式.  

#### 偏向锁的撤销

**加锁逻辑**  
假设线程A**当前或者曾经**是该偏向锁的持有者, 此时线程B尝试获取锁, 它发现MarkWord中的线程id不是匿名的, 所以首先进入撤销偏向锁的逻辑中, 
撤销锁的逻辑如下:  
1. 判断当前线程A是否存活或者在同步块中, 如果未存活, 则先把MarkWord设置为无锁状态, 然后再升级为轻量级锁
2. 如果存活且在同步块中, 那么将偏向线程的所有Displaced Mark Word设置为null, 同时将处于最高位的LockRecord
的DisplacedLockRecord设置为无锁状态(最高位的LockRecord就是第一次进入同步块时的LockRecord), 然后将对象头指向
最高位的LockRecord.

**解锁逻辑**  
从低往高遍历栈的LockRecord,如果当前线程持有的偏向锁没有被撤销(即没有升级为轻量级锁),那么直接将LockRecord释放就好了;  
对于已经撤销升级到轻量级锁的,需要将LockRecord中的Displaced Mark Word cas到对象的MarkWord中.  
如果这里CAS失败或者是重量级锁, 会在接下来讲解.

### 轻量级锁
**加锁逻辑**  
1. 如果是无锁状态会尝试把对象的MarkWord设置到LockRecord的Displaced MarkWord中, 并且把LockRecord的地址设置到对象头的MarkWord中,
如果这一步成功了, 那么成功获取了锁, 否则膨胀为重量级锁.
2. 如果是重入状态, 那么此时的LockRecord的Displaced MarkWord设置的是null. 由于是重入状态, 所以不需要cas, 直接返回即可.
3. 如果在1中失败了或者轻量级锁被其他线程持有, 那么会进入重量级锁逻辑中.

**解锁逻辑**  
从低往高遍历栈的LockRecord进行释放,将最后一个非重入的LockRecord的Displaced MarkWord CAS 到对象头的MarkWord.
CAS失败会进入monitorexit方法.

### 重量级锁
ObjectMonitor示意图:
![avatar](/static/img/JAVA关键字synchronized-重量级锁ObjectMonitor结构.png)


**膨胀逻辑**  
从无锁状态或者轻量级锁进入重量级锁要进行锁的膨胀,首先创建一个ObjectMonitor对象, 将它的地址cas到对象头中,
设置LockRecord的Displaced MarkWord为一个特殊值, 代表该锁正在使用一个重量级锁的Monitor.

**加锁逻辑**  
1. 如果成功CAS设置ObjectMonitor的owner或者是重入, 则直接返回.
2. 如果CAS失败, 那么进行自旋, 如果通过自旋获取了锁, 那么直接返回.
3. 自旋失败, 进入ObjectMonitor cxq队列的队首.
4. 阻塞自己
5. 被唤醒后再次尝试获取锁.

**释放锁逻辑**
1. 将owner字段设置为null
2. 如果当前没有需要唤醒的线程(entrylist和cxq为空)那么直接返回.
3. 当前线程重新获取锁,因为之后要操作cxq和entrylist唤醒线程.
4. 根据策略的不同进行不同的唤醒方法.
	- 策略1: 直接从cxq中获取队首元素唤醒, 结束.
	- 策略2: 把cxq队列插入entrylist尾部
	- 策略3: 把cxq队列插入entrylist头部
	- 策略4: 什么都不做
5. 如果entrylist不为空, 获取第一个元素唤醒返回.
6. 否则获取cxq的第一个元素唤醒返回.

## 总结:synchronized与ReentrantLock的区别
|比较点|synchronized|ReentrantLock|  
|-|-|-|  
|实现层次|JVM|JDK|  
|是否可中断|否|可|  
|公平性|非公平,通常情况下后来线程先获得锁|非公平与公平均可设置|  
|是否可查看锁的状态|否|是|  
|发生异常是否自动释放|是|否,需要手动释放|  







