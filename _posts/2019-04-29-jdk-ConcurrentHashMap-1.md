---
layout: post
title:  "JDK源码:ConcurrentHashMap解析(一)-get方法详解"
date:   2019-04-29 18:30:00 +0700
categories: [java, jdk]
---

## 概述
**ConcurrentHashMap**是JUC包下的一个并发集合类.

## 重要属性解析
```java
// 集合的最大容量,只用了30位,前面两位是状态位.下面会讲到
private static final int MAXIMUM_CAPACITY = 1 << 30;
// 用在resize时的transfer步骤,每次一个线程最小搬运16个bin到新的table中
private static final int MIN_TRANSFER_STRIDE = 16;

private static final int RESIZE_STAMP_BITS = 16;
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
// 对Node的hashcode进行编码
static final int MOVED     = -1; // 表示是一个forward node
static final int TREEBIN   = -2; // 表示红黑树的root
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

/** 下面是成员变量 */
transient volatile Node<K,V>[] table;
// resize时新建的table
private transient volatile Node<K,V>[] nextTable;
// 扩容时,为负值,高16位为表示对长度为n的表进行扩容,低16位表示当前扩容的线程数
private transient volatile int sizeCtl;
// 下一个进行transfer的index
private transient volatile int transferIndex;
// size方法用到,相当于Striped64的base
private transient volatile long baseCount;
// 进行resize或者创建新的cell
private transient volatile int cellsBusy;
// 相当于Striped64的cells
private transient volatile CounterCell[] counterCells;
```

## 读方法解析

### get方法解析
```java
public V get(Object key) {
	Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
	int h = spread(key.hashCode());
	if ((tab = table) != null && (n = tab.length) > 0 &&
		(e = tabAt(tab, (n - 1) & h)) != null) {
		if ((eh = e.hash) == h) { // 判断bin的第一个node是否就是目标
			if ((ek = e.key) == key || (ek != null && key.equals(ek)))
				return e.val;
		}
		else if (eh < 0) // 不是正常的node,正常node的hashcode是大于0的
			return (p = e.find(h, key)) != null ? p.val : null;
		while ((e = e.next) != null) { // 遍历链表
			if (e.hash == h &&
				((ek = e.key) == key || (ek != null && key.equals(ek))))
				return e.val;
		}
	}
	return null;
}
```
可以看到,get方法没有任何同步,说明CHM是允许多线程并发读的,我们可以看一下Node内部类的属性定义:
```java
final int hash;
final K key;
volatile V val;
volatile Node<K,V> next;
```
可以看到val是volatile的,这样在put更新的时候,所有读都可以看到最新的val.

我们接着分析Node的find方法,在分析find方法之前,我们先看一下Node的子类,根据上面Node的一些特殊种类,在CHM中也定义了相关的子类如下:
- TreeBin以及TreeNode:用于红黑树相关
- ForwardingNode: resize时从旧table到新table的传送口
- ReservationNode: 用于computeIfAbsent和compute方法

##### ForwardingNode
我们先来看ForwardingNode
```java
final Node<K,V>[] nextTable;
```
ForwardingNode只有一个成员变量,nextTable保存在ForwardingNode中是为了防止nextTable在Transfer完成后被置为null.

接下来看find方法
```java
Node<K,V> find(int h, Object k) {
	// loop to avoid arbitrarily deep recursion on forwarding nodes
	outer: for (Node<K,V>[] tab = nextTable;;) {
		Node<K,V> e; int n;
		if (k == null || tab == null || (n = tab.length) == 0 ||
			(e = tabAt(tab, (n - 1) & h)) == null)
			return null;
		for (;;) {
			int eh; K ek;
			if ((eh = e.hash) == h &&
				((ek = e.key) == k || (ek != null && k.equals(ek))))
				return e;
			if (eh < 0) {
				if (e instanceof ForwardingNode) {
					tab = ((ForwardingNode<K,V>)e).nextTable;
					continue outer;
				}
				else
					return e.find(h, k);
			}
			if ((e = e.next) == null)
				return null;
		}
	}
}
```
可以看到有两层循环.
第一层循环是从当前ForwardingNode跳转到下一个ForwardingNode所用的.
第二层循环是遍历当前链表用到的.