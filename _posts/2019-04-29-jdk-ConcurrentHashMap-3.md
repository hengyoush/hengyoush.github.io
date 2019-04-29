---
layout: post
title:  "JDK源码:ConcurrentHashMap解析(三)-put解析"
date:   2019-04-29 18:30:00 +0700
categories: [java, jdk]
---

## 概论
这篇文章来分析一下CHM的putVal方法.
通过putVal方法我们可以更好的了解CHM的并发机制.

## 源码
由于源码较长,我们将其分为三部分:
1. 预检查及shortCut
2. 链表遍历
3. treeBin遍历

#### shortCut部分
我们先来看shortCut部分:
```java
if (key == null || value == null) throw new NullPointerException();
int hash = spread(key.hashCode());
int binCount = 0;
for (Node<K,V>[] tab = table;;) {
	Node<K,V> f; int n, i, fh; K fk; V fv;
	if (tab == null || (n = tab.length) == 0)
		tab = initTable();
	else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
		if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
			break;                   // no lock when adding to empty bin
	}
	else if ((fh = f.hash) == MOVED)
		tab = helpTransfer(tab, f);
	else if (onlyIfAbsent // check first node without acquiring lock
			 && fh == hash
			 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
			 && (fv = f.val) != null)
		return fv;
```
分析以上代码:
1. 检查键值不能为null.
2. 通过hash函数获取hash值.
3. 进入循环(此循环是为了之后出现initTable或者helpTransfer而准备的)
	1. 判断是否需要初始化table
	2. 判断是否要插入的bin为空,如果为空,直接尝试cas,如果cas成功那么退出循环结束,否则继续循环
	3. 判断当前bin是否处于MOVED状态,如果是,调用helpTransfer方法帮助transfer.
	4. 判断bin的第一个节点是否是要查找的节点,如果onlyIfAbsent为true说明只有在目标key不存在的情况下才put,如果第一个节点就是要查找的节点那么不需要put,可以直接返回.
	5. 遍历链表(接下来讲到)

#### 链表遍历部分
接着来看链表遍历部分:
```java
V oldVal = null;
synchronized (f) {
	if (tabAt(tab, i) == f) {
		if (fh >= 0) {
			binCount = 1;
			for (Node<K,V> e = f;; ++binCount) {
				K ek;
				if (e.hash == hash &&
					((ek = e.key) == key ||
					 (ek != null && key.equals(ek)))) {
					oldVal = e.val;
					if (!onlyIfAbsent)
						e.val = value;
					break;
				}
				Node<K,V> pred = e;
				if ((e = e.next) == null) {
					pred.next = new Node<K,V>(hash, key, value);
					break;
				}
			}
		}
		else if (f instanceof TreeBin) {
			Node<K,V> p;
			binCount = 2;
			if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
										   value)) != null) {
				oldVal = p.val;
				if (!onlyIfAbsent)
					p.val = value;
			}
		}
		else if (f instanceof ReservationNode)
			throw new IllegalStateException("Recursive update");
	}
}
```
1. 注意到这段代码是被`synchronized (f)`包裹起来的,那么f是什么呢,可以看到shortCut部分中f被这样赋值了:`f = tabAt(tab, i = (n - 1) & hash)`,由此可见f是**当时**bin的第一个节点.
2. 接下来判断`tabAt(tab, i) == f`是为了什么呢?注意到f可能在被赋值和到`synchronized (f)`之间,bin的第一个节点会被移除,如果获取锁后不进行一次double-check的话,可能同步块锁住的是一个失效的节点.
如果这里判断是false的话,会重新进行一次外层for循环.
3. 判断f的hash值,hash值有几个种类(详见第一篇文章)
	1. 如果大于0代表是正常的节点,进入for循环,如果在for循环内找到了目标key则直接替换val,否则在末尾添加.
	2. 如果不大于0代表可能是treebin,第二个分支判断是否是treeBin,如果是进行相关逻辑
	3. 判断是否是ReservationNode,这个ReservationNode是在compute等操作时用到的.