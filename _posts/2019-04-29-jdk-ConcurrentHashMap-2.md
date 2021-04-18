---
layout: post
title:  "JDK源码:ConcurrentHashMap解析(二)-位运算与resize详解"
date:   2019-04-29 18:30:00 +0700
categories: [java, jdk]
---

## 概论
在**ConcurrentHashMap**(以下简称CHM)中,存在一些位运算,我们需要事先了解这些位运算才能更好的理解CHM的原理.  
而且个人认为resize的流程是CHM设计比较精妙的部分.下面重点讲解一下该过程.

## sizeCtl
sizeCtl是与table初始化和resize相关的变量,它使用了位操作来表示和控制初始化和resize的过程.

下面来解释sizeCtl的具体功能:
sizeCtl在正常时期(即不是初始化或者resize)是下一次进行resize时的size大小,我们可以在以下地方寻找到印证:
```java
public ConcurrentHashMap(int initialCapacity) {
	...(省略无关代码)
	this.sizeCtl = cap;
}
```
```java
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
	this.sizeCtl = DEFAULT_CAPACITY;
	...(省略无关代码)
}
```
如上所示,在构造函数中设置了sizeCtl为Entry数组的大小.

在初始化时,sc为-1.
在resize时,有点复杂,此时sc高16位为table的length的前导0个数.
*(例如:假设table.length此时为16,那么其前导0个数为:28 -- 11100)*
sc低16位为进行resize的线程数 + 1.

接着我们看有哪些地方使用到了sizeCtl,这样我们可以加深对sizeCtl的理解.

## sizeCtl的深入理解

#### initTable()
```java
private final Node<K,V>[] initTable() {
	Node<K,V>[] tab; int sc;
	while ((tab = table) == null || tab.length == 0) {
		// sizeCtl小于0表示已经有其他线程开始初始化
		if ((sc = sizeCtl) < 0)
			Thread.yield(); // lost initialization race; just spin
		else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
			try {
				if ((tab = table) == null || tab.length == 0) {
					// 上面说了sc大于0表示下一次扩容时的数组大小
					int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
					@SuppressWarnings("unchecked")
					Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
					table = tab = nt;
					// sc 为 3/4n, 即loadFactor为0.75
					sc = n - (n >>> 2);
				}
			} finally {
				sizeCtl = sc;
			}
			break;
		}
	}
	return tab;
}
```

#### addCount()
```java
private final void addCount(long x, int check) {
	CounterCell[] as; long b, s;
	if ((as = counterCells) != null ||
		!U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
		CounterCell a; long v; int m;
		boolean uncontended = true;
		if (as == null || (m = as.length - 1) < 0 ||
			(a = as[ThreadLocalRandom.getProbe() & m]) == null ||
			!(uncontended =
			  U.compareAndSetLong(a, CELLVALUE, v = a.value, v + x))) {
			fullAddCount(x, uncontended); // Striped64的逻辑
			return;
		}
		if (check <= 1)
			return;
		s = sumCount();
	}
	if (check >= 0) {
		Node<K,V>[] tab, nt; int n, sc;
		// 当前元素个数大于sc开始扩容
		while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
			   (n = tab.length) < MAXIMUM_CAPACITY) {
			int rs = resizeStamp(n);
			// sc < 0 代表已经有其他线程发起了扩容
			if (sc < 0) {
				// sc >>> RESIZE_STAMP_SHIFT 代表前导0个数,此时判断是否等于rs是为了判断是否此时的sc已经过时了
				// sc = rs + 1 代表是否最后一个线程已经完成resize
				// sc = rs + MAX_RESIZERS 代表是否resize线程到达上限
				// nextTable = null 代表扩容完成或者新的table还没被第一个resize线程初始化完成
				// transferIndex <= 0代表扩容任务已经被拉取完成
				if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
					sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
					transferIndex <= 0)
					break;
				// resize线程 + 1
				if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
					transfer(tab, nt);
			}
			// 第一个进行resize的线程需要+2,代表一个初始化1 和 一个线程 1
			else if (U.compareAndSetInt(this, SIZECTL, sc,
										 (rs << RESIZE_STAMP_SHIFT) + 2))
				transfer(tab, null); // 注意这里传递的参数nextTab是null与之前不同
			s = sumCount();
		}
	}
}
```
addCount方法的逻辑非常清晰,分为两步:
1. 先进行类似于LongAdder的累加
2. 根据相加后的sum,判断是否需要进行resize.如果需要扩容,那么是否是本线程开始扩容还是helpTransfer.

#### helpTransfer()
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
	Node<K,V>[] nextTab; int sc;
	if (tab != null && (f instanceof ForwardingNode) &&
		(nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
		int rs = resizeStamp(tab.length);
		while (nextTab == nextTable && table == tab &&
			   (sc = sizeCtl) < 0) {
			if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
				sc == rs + MAX_RESIZERS || transferIndex <= 0)
				break;
			if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
				transfer(tab, nextTab);
				break;
			}
		}
		return nextTab;
	}
	return table;
}
```
与上面的addCount方法类似,在此不赘述.

#### transfer方法详解
transfer发生在resize过程中,将旧table的Node转移到新table的node中, 因为该过程一般比较耗时, 所以CHM实现中是允许同时有多个
线程同时参与resize的, 具体是将table分成一批一批分配给每个线程去转移, 如果table中的Node为null就将该位置设置为ForwadingNode,
告知并发进行的put操作修改新table,旧table就不要操作了.

下面我们来看详细的步骤:

##### 创建新的table
这一步只能由一个线程完成, 具体如何控制的看之前addCount有讲述,下面我们直接看代码:
```java
int n = tab.length, stride;
if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE) // @1
	stride = MIN_TRANSFER_STRIDE; // subdivide range
if (nextTab == null) {            // initiating @2
	try {
		@SuppressWarnings("unchecked")
		Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
		nextTab = nt;
	} catch (Throwable ex) {      // try to cope with OOME
		sizeCtl = Integer.MAX_VALUE;
		return;
	}
	nextTable = nextTab;
	transferIndex = n;
}
int nextn = nextTab.length;
```
1. 首先设置了stride变量的值, 该值的作用是确定每个resize线程每次搬动桶的大小, 最小是16个桶.
2. 初始化nextTab, 大小为之前的2倍.

#### 抢占要搬动的桶
```java
for (int i = 0, bound = 0;;) {
	Node<K,V> f; int fh;
	while (advance) {
		int nextIndex, nextBound;
		if (--i >= bound || finishing) // @1
			advance = false;
		else if ((nextIndex = transferIndex) <= 0) { // @2
			i = -1;
			advance = false;
		}
		else if (U.compareAndSwapInt
					(this, TRANSFERINDEX, nextIndex,
					nextBound = (nextIndex > stride ?
								nextIndex - stride : 0))) { // @3
			bound = nextBound;
			i = nextIndex - 1;
			advance = false;
		}
	}
	// ...
}
```
这里我们进入了一个大循环, 创建了两个变量`i`和`bound`.
接着我们进入了一个while循环,在这个循环中,当前线程将会抢占到一个区间,这个区间范围内的桶都由它来搬动.
`@1`和`@2`这两个分支我们先跳过,因为第一次基本上是不会走到这两个分支上的, 我们先看`@3`分支.  
在这个分支中, 尝试对CHM中的transferIndex进行CAS更新, 尝试更新的值是`transferIndex-stride`,
这里可以看出,分配是从右往左分配的,每次最多分配stride个桶.  
更新完之后bound设置为左边界(close), i设置为右边界(close).
示意图如下:
![avatar](/static/img/CHM-resize-桶范围确定.png)


##### 循环处理要搬运的桶
首先我们看几个前置检查分支:
1、检查i的范围
```java
if (i < 0 || i >= n || i + n >= nextn) {
	int sc;
	if (finishing) {
		nextTable = null;
		table = nextTab;
		sizeCtl = (n << 1) - (n >>> 1);
		return;
	}
	if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
		if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
			return;
		finishing = advance = true;
		i = n;
	}
}
```
如果超过有效范围, 那么检查finishing状态, 如果为true, 那么结束resize, 清理nextable变量,
更新当前table的引用, 还原sizeCtl为下一次要resize时的size.  
如果没有结束, 那么查看自己是不是最后一个resize的线程, 如果是, 那么设置finishing变量下一次循环时就会进入上面的分支.  
如果自己不是最后一个resize线程那么直接返回.

(这一段是一个线程transfer完成时,对sc进行修改时的代码,它做了以下判断:
1. 首先使用cas将sc的值减1(注意addCount方法中`sc == rs + 1`的判断)
2. 判断是否减2后16位就为0了,意味着这是最后一个扩容的线程.)

2、检查当前桶是不是null
```java
else if ((f = tabAt(tab, i)) == null)
	advance = casTabAt(tab, i, null, fwd);
```
如果为null, 那么就要设置ForwardNode提醒其他线程.

3、检查当前是不是ForwardNode
```java
else if ((fh = f.hash) == MOVED)
	advance = true; // already processed
```
如果是那么跳过这个桶,继续遍历下一个桶.

4、开始进行真正的桶迁移操作
我们以链表桶为主要讲解部分, 红黑树部分和链表大同小异不再赘述.
```java
synchronized (f) {
	if (tabAt(tab, i) == f) {
		Node<K,V> ln, hn;
		if (fh >= 0) {
			// 链表部分
		} else if (...) {
			// 红黑树部分
		}
	}
}
```
首先在链表头上加锁, 同时判断链表头的类型, 下面我们看链表的操作部分.

在此之前, 我们要知道每个桶的迁移实际上只有两种选择:
1. 留在原地.
2. 迁移到新table的n+i位置.  
假设我们的table大小为8(二进制为1000), 这次要扩展到16(10000), 在table索引为3(00011)的地方有一个链表, 链表上有三个对象,
一个对象的hash值为3(00011),另一个为11((01011)),另外一个是19(10011)它们通过mod 8都hash到了3这个位置.  
然而当表的大小扩展到16之后, 我们再看它们的位置:
1. 对于3来说,它的位置是 3 mod 16 = 3
2. 对于11来说, 它的位置是 11 mod 16 = 11
3. 对于19来说, 它的位置是 19 mod 16 = 3
可看出来, 对于新table的位置来说, 要么在原位置,要么在原位置 + 原table大小的位置.

```java
for (Node<K,V> p = f; p != lastRun; p = p.next) {
	int ph = p.hash; K pk = p.key; V pv = p.val;
	if ((ph & n) == 0)
		ln = new Node<K,V>(ph, pk, pv, ln);
	else
		hn = new Node<K,V>(ph, pk, pv, hn);
}
setTabAt(nextTab, i, ln); // 老位置
setTabAt(nextTab, i + n, hn); // 新位置
setTabAt(tab, i, fwd);
advance = true;
```
这里就将ln作为原位置的链表, 将hn作为新位置的链表, 代码比较简单就不详细说明了.
到这里一个for循环就结束了, 什么?你不知道是哪个for循环, 好吧,其实我也忘了😓, 回头看前面:
```java
for (int i = 0, bound = 0;;) {
	Node<K,V> f; int fh;
	while (advance) {
		int nextIndex, nextBound;
		if (--i >= bound || finishing) // @1
			advance = false;
		else if ((nextIndex = transferIndex) <= 0) { // @2
			i = -1;
			advance = false;
		}
		else if (U.compareAndSwapInt
					(this, TRANSFERINDEX, nextIndex,
					nextBound = (nextIndex > stride ?
								nextIndex - stride : 0))) { // @3
			bound = nextBound;
			i = nextIndex - 1;
			advance = false;
		}
	}
	// ...
}
```
@1: 迁移完成一个桶之后, 我们会到i这里将之前的i减一, 继续搬运这批桶的下一个桶;  
如果发现i-1 < bound了,就说明分配的这批桶全部迁移完了, 要分配下一批桶了.

@2: 走到这里说明要分配下一批桶了, 判断`transferIndex <= 0`代表所有桶都搬运完了(可能是其他线程搬运的),
这时候将i设置为-1,是为了之后进入`if (i < 0 || i >= n || i + n >= nextn)`的分支中结束当前resize.

@3: 走到这里就尝试CAS分配下一批桶了, 之后的流程就不再赘述了.

