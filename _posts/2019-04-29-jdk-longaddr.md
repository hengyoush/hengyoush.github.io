---
layout: post
title:  "JDK源码:LongAdder与Striped64解析"
date:   2019-04-29 18:11:00 +0700
categories: [java, jdk]
---

## 概述
**LongAdder**是用于多线程环境下Long类型累加计算的工具类,与**AtomicLong**相比,具有在竞争激烈的场景下性能比单纯的CAS更为高效的优点.
LongAdder继承自**Striped64**类,该类实现了主要的功能逻辑.
下面我们先来看一下**Striped64**是如何实现64位的计算逻辑的.

## 动态分段
在具体讲解**Striped64**的逻辑之前,我们先介绍**Striped64**的实现思想.
**Striped64**的实现是基于分段实现的,分段是一种在多线程环境下保持线程安全和性能的重要手段,它通过将竞争的关注点分离来实现高效并发,在**Striped64**中的表现就是它通过将每个线程映射到一个数组中的不同的slot来降低竞争程度.

那么何为"动态"分段呢?
动态体现在对slot的分配上,如果一个slot上存在冲突,那么**Striped64**会对线程进行rehash将线程重新映射到新的slot上,这就是动态分段.

## 主要流程
首先,我们介绍以下与**Striped64**中计数相关的属性:
```
// 每个cell里面有一个long,线程在cell上进行累加
transient volatile Cell[] cells;
// 当不存在冲突时或者作为存在冲突时的一个后备选择,base用作累加的目标
transient volatile long base;
// 当存在table初始化,新建cell时为1,否则为0
transient volatile int cellsBusy;
```

累加主要靠上面两个属性实现,下面我们介绍累加的主要流程:
1. 先尝试在base上进行累加(CAS),如果CAS成功则直接结束,否则继续2
2. 再尝试在线程对应的cell上进行累加(使用xOrShift计算hashcode,接着计算位置),如果成功,结束,否则进入3
3. 如果失败,则判断当前cells的长度是否大于CPU核心数,如果小于进入4,否则进入5
4. 将cells长度扩大为2倍,继续2
5. 重新计算hash值,重复2

## 具体实现
```
final void longAccumulate(long x, LongBinaryOperator fn,
						  boolean wasUncontended) {
	int h;
	if ((h = getProbe()) == 0) {// 获取线程hash值，作为插入table的索引
		ThreadLocalRandom.current(); 
		h = getProbe();
		wasUncontended = true;
	}
	boolean collide = false;                // True if last slot nonempty
	done: for (;;) {
		Cell[] cs; Cell c; int n; long v;
		if ((cs = cells) != null && (n = cs.length) > 0) {// table以前存在过争用
			if ((c = cs[(n - 1) & h]) == null) {// 如果要设置的位置没有碰撞
				if (cellsBusy == 0) {       // 尝试创建并更新table，此处首先创建Cell对象
					Cell r = new Cell(x);   
					if (cellsBusy == 0 && casCellsBusy()) {// 此处尝试获取锁
						try {               
							Cell[] rs; int m, j;
							if ((rs = cells) != null &&
								(m = rs.length) > 0 &&
								rs[j = (m - 1) & h] == null) {// 拥有锁的情况下再次判断此处是否为空
								rs[j] = r;
								break done; // 创建成功,直接退出循环
							}
						} finally {
							cellsBusy = 0;// 释放锁
						}
						continue;           // 此时创建cell失败,则slot必不空
					}
				}
				collide = false;// 这里设为false是因为没能获取自旋锁不是因为碰撞导致的,是因为table正在resize
			}
			else if (!wasUncontended)       // 到此处说明有争用
				wasUncontended = true;      // 重新hash继续
			else if (c.cas(v = c.value,// CAS尝试在不为空的槽上叠加
						   (fn == null) ? v + x : fn.applyAsLong(v, x)))
				break;
			else if (n >= NCPU || cells != cs)// 判断table大小是否达到上限或者table是否被resize
				collide = false;           
			else if (!collide)// 走到这里不论collide是否为真都有争用
				collide = true;
			else if (cellsBusy == 0 && casCellsBusy()) {// 尝试获取锁将table的长度增大为两倍
				try {
					if (cells == cs)        // Expand table unless stale
						cells = Arrays.copyOf(cs, n << 1);
				} finally {
					cellsBusy = 0;
				}
				collide = false;
				continue;                   // 使用扩大后的表继续
			}
			h = advanceProbe(h);// 除非在已有的slot上CAS成功,否则重新计算一个hash值
		}
		else if (cellsBusy == 0 && cells == cs && casCellsBusy()) 
			try {    // 进入这里说明以前没有争用但现在存在了，因为下面的casBase失败了
				if (cells == cs) {
					Cell[] rs = new Cell[2];
					rs[h & 1] = new Cell(x);
					cells = rs;
					break done;
				}
			} finally {
				cellsBusy = 0;
			}
		}
		// Fall back on using base
		else if (casBase(v = base,// 尝试直接在base上更新
						 (fn == null) ? v + x : fn.applyAsLong(v, x)))
			break done;
	}
}
```

```
public void add(long x) {
	Cell[] cs; long b, v; int m; Cell c;
	if ((cs = cells) != null || !casBase(b = base, b + x)) {
		// 当存在争用或不存在争用但cas失败时进入下面的逻辑（即检测到存在争用）
		boolean uncontended = true;
		if (cs == null || (m = cs.length - 1) < 0 ||// 依次判断是否存在过争用、是否Cells已经初始化、
													// 是否当前slot存在空位、是否在非空的空位上
													// 执行CAS成功？
			(c = cs[getProbe() & m]) == null ||
			!(uncontended = c.cas(v = c.value, v + x)))
			longAccumulate(x, null, uncontended);// 上述条件有一项为真则进入
	}
}

```

#### 关于sum方法
LongAdder的sum方法返回的不是一个快照值,如下:
```
public long sum() {
	Cell[] cs = cells;
	long sum = base;
	if (cs != null) {
		for (Cell c : cs)
			if (c != null)
				sum += c.value;
	}
	return sum;
}
```
可以看到,sum方法的实现仅仅将cell和base相加,此处并没有任何同步措施,所以sum方法在存在读写竞争的情况下返回的不是一个精确的值.