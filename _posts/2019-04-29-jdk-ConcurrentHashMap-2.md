---
layout: post
title:  "JDK源码:ConcurrentHashMap解析(二)-位运算详解"
date:   2019-04-29 18:30:00 +0700
categories: [java, jdk]
---

## 概论
在**ConcurrentHashMap**(以下简称CHM)中,存在一些位运算,我们需要事先了解这些位运算才能更好的理解CHM的原理.

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
				// nextTable = null 代表扩容完成
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
				transfer(tab, null); // 注意这里传递的参数与之前不同
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

#### transfer方法片段
```java
if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
	if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
		return;
	finishing = advance = true;
	i = n; // recheck before commit
}
```
这一段是一个线程transfer完成时,对sc进行修改时的代码,它做了以下判断:
1. 首先使用cas将sc的值减1(注意addCount方法中`sc == rs + 1`的判断)
2. 判断是否减2后16位就为0了,意味着这是最后一个扩容的线程.