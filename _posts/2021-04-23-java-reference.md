---
layout: post
title:  "Java引用原理剖析"
date:   2021-04-24 02:20:00 +0700
categories: [java]
---

## 引用类型简介
引用类型在Java中以`java.lang.ref.Reference`类表示, 对于引用我们知道有四种类型, 如下所示:

名称|特点
-|-
强引用|我们在代码中直接使用的局部变量、静态变量、类字段等都属于强引用,强引用的对象不会被GC回收
软引用|只有在内存不足时,才会被回收,以SoftReference类表示
弱引用|每次GC都会回收,以WeakReference表示
虚引用|只被虚引用指向的对象永远无法拿到其原本的引用(在JDK8之前虚引用是可以拿到里面的对象的,甚至在GC后放入引用队列被通知都可以拿到,如果不进行手动clear的话可能会造成内存泄露.jdk12就不存在这个问题了),以PhantomReference类表示.

## Reference类介绍
Reference内部有如下几个字段比较重要:
- referent: 表示被引用的底层对象
- queue: ReferenceQueue类型,在引用被GC的时候可以从这里拿到通知,对虚引用来说尤为重要,是一个链表结构.
- next: 表示在ReferenceQueue链表中的下一个被通知的引用
- discovered: 在discoveredList中表示下一个要被检查的引用. 在移动到pending-Reference链表后表示下一个元素.


## Reference引用GC触发通知原理
首先我们看如何创建一个引用, 触发通知, 代码如下:
```java
public static void main(String[] args) throws InterruptedException {
	Object o = new Object();
	ReferenceQueue q = new ReferenceQueue();
	PhantomReference r = new PhantomReference(o, q); // 1
	o = null;
	System.gc(); // 2
	Reference remove = q.remove(); // 3
	System.out.println(remove == r);
}
```
1: 这里我们创建了一个Object类型的变量, 然后创建了一个虚引用, 注册到一个ReferenceQueue中.
2: 由于我们想将object gc掉以触发通知, 所以我们将object置为null, 然后手动调用`System.gc()`触发fullgc.
3: 这里object被回收,我们可以获得一个通知,结果这个引用和我们之前创建的引用是同一个.

### 内部实现原理
首先我们要明白,在GC的时候,JVM会发现回收的对象不是一个强引用, 那么会将它放到一个叫做discoveredList的队列中,
在该队列中针对不同的引用做不同的处理,比如当前内存充足,那么就会将软引用从队列中移除,最终留下的是要进行回收的引用,会把它们移动
到pending-Reference链表中,而这些引用最终将会得到通知.

在Native层会做:  
1.JVM会发现回收的对象不是一个强引用, 那么会将它放到一个叫做discoveredList的队列中, 然后根据不同的引用类型进行过滤.(代码位置:
ReferenceProcessor::discover_reference) 
2.移动到pending-Reference链表(DiscoveredListIterator::complete_enqueue)

在Java层有一个专门从pending-Reference链表中拿引用然后将引用放到我们之前注册的引用队列的线程:ReferenceHandler,代码如下:
```java
Reference<Object> pendingList;
synchronized (processPendingLock) {
	pendingList = getAndClearReferencePendingList();// 1
	processPendingActive = true;
}
while (pendingList != null) {
	Reference<Object> ref = pendingList;
	pendingList = ref.discovered; // 2
	ref.discovered = null;

	if (ref instanceof Cleaner) { // 3
		((Cleaner)ref).clean();
		synchronized (processPendingLock) {
			processPendingLock.notifyAll();
		}
	} else {
		ReferenceQueue<? super Object> q = ref.queue;
		if (q != ReferenceQueue.NULL) q.enqueue(ref);
	}
}
```
1. 获取Native代码中的pending-Reference链表, 具体代码在(jvm.cpp::JVM_ENTRY(jobject, JVM_GetAndClearReferencePendingList(JNIEnv* env)))
2. 通过discovered字段获取下一个pending-Reference链表的下一个元素.
3. 如果该引用是一个Cleaner对象, 那么就不放到队列里面,直接指向clean方法否则放到给该引用注册的队列中.

下面我们看看native层的实现:

### 处理SoftReference的代码
```c++
size_t ReferenceProcessor::process_soft_ref_reconsider_work(DiscoveredList&    refs_list,
                                                            ReferencePolicy*   policy,
                                                            BoolObjectClosure* is_alive,
                                                            OopClosure*        keep_alive,
                                                            VoidClosure*       complete_gc) {
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  // Decide which softly reachable refs should be kept alive.
  while (iter.has_next()) { // 1
    iter.load_ptrs(DEBUG_ONLY(!discovery_is_atomic() /* allow_null_referent */));
    bool referent_is_dead = (iter.referent() != NULL) && !iter.is_referent_alive();
    if (referent_is_dead &&
        !policy->should_clear_reference(iter.obj(), _soft_ref_timestamp_clock)) { // 2
      log_dropped_ref(iter, "by policy");
      // Remove Reference object from list
      iter.remove();
      // keep the referent around
      iter.make_referent_alive();
      iter.move_to_next();
    } else {
      iter.next();
    }
  }
  // Close the reachable set
  complete_gc->do_void();

  log_develop_trace(gc, ref)(" Dropped " SIZE_FORMAT " dead Refs out of " SIZE_FORMAT " discovered Refs by policy, from list " INTPTR_FORMAT,
                             iter.removed(), iter.processed(), p2i(&refs_list));
  return iter.removed();
}
```
1. 遍历DiscoveredList,对其中不符合回收标准的从列表中移除.
2. 真正进行判断的地方,什么样的软引用符合回收的标准呢? 判断方式如下:
JVM内部定义了几种策略,默认的是这种:

```c++
// Capture state (of-the-VM) information needed to evaluate the policy
void LRUMaxHeapPolicy::setup() {
  size_t max_heap = MaxHeapSize;
  max_heap -= Universe::get_heap_used_at_last_gc();
  max_heap /= M;

  _max_interval = max_heap * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

// The oop passed in is the SoftReference object, and not
// the object the SoftReference points to.
bool LRUMaxHeapPolicy::should_clear_reference(oop p,
                                             jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) {
    return false;
  }

  return true;
}
```
这种策略在内部不足时对软引用空闲的时间要求越短. 在这里可以看到通过比较软引用内部的一个字段timestamp与上一次GC时间做比较.

### 处理软引用、弱引用的地方

```c++
// process_soft_weak_final_refs方法
void ReferenceProcessor::process_soft_weak_final_refs(BoolObjectClosure* is_alive,
                                                      OopClosure* keep_alive,
                                                      VoidClosure* complete_gc,
                                                      AbstractRefProcTaskExecutor*  task_executor,
                                                      ReferenceProcessorPhaseTimes* phase_times) {
  // ...														  
  if (_processing_is_mt) {
	  // ...
  } else {
    {
      size_t removed = 0;
      for (uint i = 0; i < _max_num_queues; i++) {
        removed += process_soft_weak_final_refs_work(_discoveredSoftRefs[i], is_alive, keep_alive, true /* do_enqueue */); 
      }
    }
    {
      size_t removed = 0;

      RefProcSubPhasesWorkerTimeTracker tt2(WeakRefSubPhase2, phase_times, 0);
      for (uint i = 0; i < _max_num_queues; i++) {
        removed += process_soft_weak_final_refs_work(_discoveredWeakRefs[i], is_alive, keep_alive, true /* do_enqueue */);
      }
    }
	// ...
  }
}

//process_soft_weak_final_refs_work方法

size_t ReferenceProcessor::process_soft_weak_final_refs_work(DiscoveredList&    refs_list,
                                                             BoolObjectClosure* is_alive,
                                                             OopClosure*        keep_alive,
                                                             bool               do_enqueue_and_clear) {
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  while (iter.has_next()) {
    if (iter.referent() == NULL) {
      iter.remove();
      iter.move_to_next();
    } else if (iter.is_referent_alive()) { // 依然存活
      iter.remove();
      iter.make_referent_alive();
      iter.move_to_next();
    } else {
      if (do_enqueue_and_clear) { // weak和soft都为true
        iter.clear_referent();
        iter.enqueue(); // 将discoveredList的下一个元素放到当前引用的discovered字段上
        log_enqueued_ref(iter, "cleared");
      }
      // Keep in discovered list
      iter.next();
    }
  }
  if (do_enqueue_and_clear) {
    iter.complete_enqueue(); // 将discovered队列移到pending-referenece队列,实际是通知Java层当前有pending的reference需要处理
    refs_list.clear();
  }
  return iter.removed();
}
```
1. 如上首先分别对soft和weak引用做处理,将它们加入到pending-reference链表上,注意这里可以看到对weak引用的处理与soft有怎样的区别,
因为在前面soft只有在内存不足的情况下才会加入到这个链表中而weak则是二话不说就加进去了,这也印证了我们之前的说法.


### 处理虚引用的地方
```c++
size_t ReferenceProcessor::process_phantom_refs_work(DiscoveredList&    refs_list,
                                          BoolObjectClosure* is_alive,
                                          OopClosure*        keep_alive,
                                          VoidClosure*       complete_gc) {
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  while (iter.has_next()) {

    oop const referent = iter.referent();

    if (referent == NULL || iter.is_referent_alive()) { // 1
      iter.make_referent_alive();
      iter.remove();
      iter.move_to_next();
    } else { // 2
      iter.clear_referent();
      iter.enqueue();
      iter.next();
    }
  }
  iter.complete_enqueue();
  complete_gc->do_void();
  refs_list.clear();

  return iter.removed();
}
```
1. 判断对象是否存活,如果存活则从DiscoveredList中移除
2. 如果死亡那么将其加入到pending-reference列表中.