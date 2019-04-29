---
layout: post
title:  "JDK源码:ThreadLocal解析"
date:   2019-04-29 18:11:00 +0700
categories: [java, jdk源码]
---

## 概述
每一个 **ThreadLocal** ,存放在线程对象中的 **threadLocals** 变量中, **threadLocals** 是一个 **ThreadLocalMap** 类型的变量.
**ThreadLocalMap** 是一个键值对的集合类型,与 **HashMap** 不同的是,它采用线性探测法解决冲突;而且其 **Entry** 继承 **WeakReference**:
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
		/** The value associated with this ThreadLocal. */
		Object value;
		Entry(ThreadLocal<?> k, Object v) {
		super(k);
		value = v;
	}
}
```
这就意味着如果ThreadLocal对象不再拥有强引用的情况下,会被GC回收掉,这样就防止了存放在ThreadLocal中影响GC.
但是还是需要注意使用线程池的情况,及时remove释放掉,防止内存泄漏.

下面我们看一下 **ThreadLocalMap** 的具体实现吧!

## getEntry方法
```java
private Entry getEntry(ThreadLocal<?> key) {
	int i = key.threadLocalHashCode & (table.length - 1);
	Entry e = table[i];
	if (e != null && e.get() == key)
		return e;
	else
		return getEntryAfterMiss(key, i, e);
}
```
分析:
1. 首先获取槽的索引
2. 判断槽中的entry中的ThreadLocal是否与key相等,如果相等那么直接返回,否则进入 **getEntryAfterMiss** 方法.

#### getEntryAfterMiss方法猜测
getEntryAfterMiss方法必须根据线性探测法的特性向后寻找相等的key.
回想起之前Entry继承了 **WeakReference** 这个事情,那么在Entry数组中一定存在着entry中的key为空的情况(即已经被回收),那么我们需要对这些 **stale slot**  进行清理,防止它们一直占用槽位.

#### getEntryAfterMiss方法解析
首先看下源码:
```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
	Entry[] tab = table;
	int len = tab.length;

	while (e != null) {
		ThreadLocal<?> k = e.get();
		if (k == key)
			return e;
		if (k == null)
			expungeStaleEntry(i);
		else
			i = nextIndex(i, len);
		e = tab[i];
	}
	return null;
}
```
三个参数含义分别如下:
- 要查找的key
- 该ThreadLocal对应的槽,即从哪开始找的位置
- 对应槽的Entry

我们可以很容易的看出其中的逻辑:
对每一个遍历到的槽做如下判断:
1. 首先判断key是否相等,相等则直接返回否则进入下一步
2. 如果key为空,则说明该槽的ThreadLocal已经被回收,此时调用 **expungeStaleEntry** 方法清理槽位(该方法接下来会讲到)
3. 如果key不相等且不为空,则继续查找,通过 **nextIndex** 方法获取下一个探查的位置(这其实是一个探测函数),继续判断
4. 终结:如果e为空,即查找失败(这里为什么查找失败还需要看下Knuth的书),返回null
*(如我们之前所猜测的,确实在查找过程中进行了清理)*

## set方法解析

#### 解析前的猜测
set方法有可能会遇到冲突,根据线性探测法,其实现应该调用 **nextIndex** 方法获取一个没有被占用的槽来进行插入.
在查找的过程中可能遇见 **stale slot** ,那么需要对其进行清理.
同时,由于可能entry数组大小超过了 **capacity * threshold** , 需要 **resize** ,这也是需要考虑的.

#### Let us 解析!
```java
private void set(ThreadLocal<?> key, Object value) {
	Entry[] tab = table;
	int len = tab.length;
	int i = key.threadLocalHashCode & (len-1);
	for (Entry e = tab[i]; // 遍历查找空槽
		 e != null;			// 注意这个循环条件
		 e = tab[i = nextIndex(i, len)]) {
		ThreadLocal<?> k = e.get();
		if (k == key) { // 如果已经存在,那么覆盖value
			e.value = value;
			return;
		}
		if (k == null) { // 如果遍历过程中,发现stale slot,那么要进行替换
			replaceStaleEntry(key, value, i);
			return;
		}
	}
	tab[i] = new Entry(key, value); // 查找到空槽
	int sz = ++size;
	if (!cleanSomeSlots(i, sz) && sz >= threshold) // 是否超限
		rehash();
}
```
与猜测有所不同,如果寻找到 **stale slot** ,进行了一个replace操作,猜测是将要存放的值存放在stale slot中.
其他的猜测都命中了,比如:resize, 查找到空槽来进行插入等.
接下来我们看一下 **replaceStaleEntry** 的实现.

#### replaceStaleEntry解析
```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
							   int staleSlot) {
	Entry[] tab = table;
	int len = tab.length;
	Entry e;

	// 在当前 run 中找到第一个stale slot, 用slotToExpunge记录
	int slotToExpunge = staleSlot;
	for (int i = prevIndex(staleSlot, len);
		 (e = tab[i]) != null;
		 i = prevIndex(i, len))
		if (e.get() == null)
			slotToExpunge = i;

	// 从staleSlot开始遍历,直到null,如果在这之中存在key,那么进行交换并更新value
	for (int i = nextIndex(staleSlot, len);
		 (e = tab[i]) != null;
		 i = nextIndex(i, len)) {
		ThreadLocal<?> k = e.get();

		// 找到了key,那么需要将它放到staleSlot中,将它们的entry进行交换,然后开始清理
		if (k == key) {
			e.value = value;

			tab[i] = tab[staleSlot];
			tab[staleSlot] = e;

			// Start expunge at preceding stale entry if it exists
			if (slotToExpunge == staleSlot)
				slotToExpunge = i;
			cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
			return;
		}

		// k=null表明该槽需要清理
		if (k == null && slotToExpunge == staleSlot)
			slotToExpunge = i;
	}

	// If key not found, put new entry in stale slot
	// 在run中没有发现key,那么新建一个entry,将其放在stale slot上.
	tab[staleSlot].value = null;
	tab[staleSlot] = new Entry(key, value);

	// 开始清理
	if (slotToExpunge != staleSlot)
		cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```
1. 首先从staleSlot向前查找直到null,获取第一个需要清理的slot索引,存放在slotToExpunge中,后面需要使用slotToExpunge进行清理.
2. 开始从staleSlot向后查找,看是不是有和key相同的槽,如果在遍历中发现存在相同的槽,那么将其与Entry[staleSlot]中的entry进行交换,将那个相同的槽放到staleSlot对应的槽中,接着开始清理,return.
3. 如果在循环中没有相同key的槽,那么新建一个Entry,将其放在staleSlot上.
4. 清理返回

## 槽的清理

#### expungeStaleEntry方法解析
```java
private int expungeStaleEntry(int staleSlot) {
	Entry[] tab = table;
	int len = tab.length;

	// 清理在staleSlot上的entry
	tab[staleSlot].value = null;
	tab[staleSlot] = null;
	size--;

	// 在遇到null之前对每个没有 stale 的槽进行rehash
	Entry e;
	int i;
	for (i = nextIndex(staleSlot, len);
		 (e = tab[i]) != null;
		 i = nextIndex(i, len)) {
		ThreadLocal<?> k = e.get();
		if (k == null) {
			e.value = null;
			tab[i] = null;
			size--;
		} else {
			int h = k.threadLocalHashCode & (len - 1);
			if (h != i) {
				tab[i] = null;

				// Unlike Knuth 6.4 Algorithm R, we must scan until
				// null because multiple entries could have been stale.
				while (tab[h] != null)
					h = nextIndex(h, len);
				tab[h] = e;
			}
		}
	}
	return i;
}
```
如上所示,代码逻辑非常简单,
1. 先将staleSlot对应的槽释放掉
2. 从staleSlot的下一个slot开始直到null,对其之间的slot作如下处理:
	1. 如果该槽已经stale,那么将其释放,相应的size--
	2. 如果该槽没有stale,那么将其rehash(这一步十分关键,是为什么上面的查找步骤只要遇到null就结束查找的原因)
3. 返回staleSlot之后的null slot.

#### cleanSomeSlots方法解析
```java
private boolean cleanSomeSlots(int i, int n) {
	boolean removed = false;
	Entry[] tab = table;
	int len = tab.length;
	do {
		i = nextIndex(i, len);
		Entry e = tab[i];
		if (e != null && e.get() == null) {
			n = len;
			removed = true;
			i = expungeStaleEntry(i);
		}
	} while ( (n >>>= 1) != 0);
	return removed;
}
```
对Entry数组进行扫描,找到stale slot进行expunge.
具体是从i开始,扫描n对数个slot,如果在其中发现了stale slot,那么n会变成table.length.

##### 备注
本文章基于JDK10分析