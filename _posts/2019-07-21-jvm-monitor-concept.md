---
layout: post
title:  "JVM字节码monitorenter分析"
date:   2019-07-21 20:18:00 +0700
categories: [jvm]
---

## monitorenter
我们在讲锁时,经常会提到`monitorenter`, 它是对应`synchronized`关键字的字节码其中之一, 还有`monitorexit`对应同步块的退出, 下面我们来看jvm源码(OPENJDK10)中
是如何处理`monitorenter`的.

> *在阅读本文章前, 你必须具有关于JVM锁的基础知识,比如: 什么是偏向锁,什么是displaced header.*

## 源码分析
首先找到地址最高的空闲的LockRecord:
```php
        CASE(_monitorenter): {
            oop lockee = STACK_OBJECT(-1);
            // derefing's lockee ought to provoke implicit null check
            CHECK_NULL(lockee);
            // find a free monitor or one already allocated for this object
            // if we find a matching object then we need a new monitor
            // since this is recursive enter
            BasicObjectLock* limit = istate->monitor_base();
            BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
            BasicObjectLock* entry = NULL;
            while (most_recent != limit ) { // 找到一个地址最高的空闲的lockrecord
                if (most_recent->obj() == NULL) entry = most_recent;
                else if (most_recent->obj() == lockee) break; // 重入
                most_recent++;
            }
        ...
```
我们来看`BasicObjectLock`的结构:
```
BasicObjectLock
    |
    |--lock
       |--displaced_header
    |--obj
```
lock中会存放加锁对象原有的mark word, obj即加锁对象.

接下来我们继续往下看:
```php
if (entry != NULL) {
          entry->set_obj(lockee); // 设置这个对象
          int success = false;
          uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;

          markOop mark = lockee->mark(); // 获取markword
          intptr_t hash = (intptr_t) markOopDesc::no_hash;
          ...
```
接着, 将获得的lock record的_obj置为当前对象.

接下来将会进入偏向锁的逻辑中:
```php
// implies UseBiasedLocking
if (mark->has_bias_pattern()) { // 可偏向, 看markword最后是不是101
```
先判断mark word最后是不是`101`.
然后判断是否是重入锁, 是否需要撤销偏向等等.

接着是一段比较复杂的位运算:
```php
uintptr_t thread_ident; // 当前线程id
uintptr_t anticipated_bias_locking_value;
thread_ident = (uintptr_t)istate->thread();
anticipated_bias_locking_value =
    (((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) &
    ~((uintptr_t) markOopDesc::age_mask_in_place);
```
 咱们将其分解来看:
 `lockee->klass()->prototype_header()`即这个对象的klass上的markword, 这上面是没有线程id的,所以它与
 `thread_ident`相或意味着将线程id嵌入到该类的原型markword中而保留了类中的 epoch + 分代年龄 + 偏向锁标志 + 锁标志位.
 然后与对象的markword做异或运算, 这会使其中相同的bit置0, 可以想见, 当当前线程持有锁时,线程id区域一定是0.
 然后又与`markOopDesc::age_mask_in_place`的反相与,`age_mask_in_place`是markword中分代年龄的掩码, 所以最后一步操作将会
 使分代年龄区域置0.

 接下来的代码将会使用算出的这个`anticipated_bias_locking_value`值进行一系列判断:

 #### 分支1:已偏向当前线程
 ```php
if  (anticipated_bias_locking_value == 0) { // 已偏向当前线程
    // already biased towards this thread, nothing to do
    if (PrintBiasedLockingStatistics) {
    (* BiasedLocking::biased_lock_entry_count_addr())++;
    }
    success = true;
}
 ```
为什么`anticipated_bias_locking_value`为0就会是已偏向当前线程前面已经说过了, 在此不再赘述.
这段重入的代码逻辑十分简单, 直接将success置为true, 就返回了.(详细全部代码可在附录中查看)

#### 分支2:类markword偏向模式关闭
由于这里已经判断过对象的markword是偏向模式的, 故此时需要撤销该对象的偏向.
```php
else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
    // try revoke bias
    markOop header = lockee->klass()->prototype_header(); // Klass中也有一个Markword
    if (hash != markOopDesc::no_hash) {
    header = header->copy_set_hash(hash);
    }
    if (lockee->cas_set_mark(header, mark) == mark) {// 将对象的markword替换为class中的markword
    if (PrintBiasedLockingStatistics)
        (*BiasedLocking::revoked_lock_entry_count_addr())++;
    }
}
```
*`biased_lock_mask_in_place`即最后101的掩码.*

此时使用cas替换对象的markword为类中的markword.接下来由于success仍然为false, 所以这个分支会进入下面的轻量级锁逻辑.
(这里的这个轻量级锁会判断重入)

#### 分支3:epoch不等于class中的epoch，尝试重偏向
经过上面的判断,已经可以确定类的偏向模式开启并且也没有偏向当前线程.
接下来判断epoch是否与class中的不一致, 为什么会不一致呢, 是因为Jvm有批量重偏向的机制, 在批量重偏向中会更新class中的epoch
并且会更新所有**正在使用的**锁对象上的epoch, 那么会出现epoch过期的情况就只有一个:该对象不是正在使用的锁对象,所以它可以被重偏向到
当前线程.
```php
// epoch不等于class中的epoch，尝试重偏向
else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
    // try rebias
    // 构造一个偏向当前线程的markword
    markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
    if (hash != markOopDesc::no_hash) {
    new_header = new_header->copy_set_hash(hash);
    }
    if (lockee->cas_set_mark(new_header, mark) == mark) {
    if (PrintBiasedLockingStatistics)
        (* BiasedLocking::rebiased_lock_entry_count_addr())++;
    }
    else {// CAS偏向失败，使用monitorenter进入轻量级锁逻辑
    CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
    }
    success = true;
}
```
该分支接下来直接返回.

#### 分支4:线程id不一致,可能是匿名偏向
```php
else {
// 构造一个匿名偏向的markword
// try to bias towards thread in case object is anonymously biased
markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |
                                                (uintptr_t)markOopDesc::age_mask_in_place |
                                                epoch_mask_in_place));
if (hash != markOopDesc::no_hash) {
    header = header->copy_set_hash(hash);
}
markOop new_header = (markOop) ((uintptr_t) header | thread_ident);
// debugging hint
DEBUG_ONLY(entry->lock()->set_displaced_header((markOop) (uintptr_t) 0xdeaddead);)
if (lockee->cas_set_mark(new_header, header) == header) {
    if (PrintBiasedLockingStatistics)
    (* BiasedLocking::anonymously_biased_lock_entry_count_addr())++;
}
else {
    CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
}
success = true;
```
该分支逻辑简单, cas判断是否是匿名偏向否则走轻量级锁逻辑,直接返回.

### 轻量级锁重入逻辑
```php
// traditional lightweight locking
    if (!success) {
    markOop displaced = lockee->mark()->set_unlocked();
    entry->lock()->set_displaced_header(displaced);// lockrecord中存放原先的markword
    bool call_vm = UseHeavyMonitors;
    if (call_vm || lockee->cas_set_mark((markOop)entry, displaced) != displaced) {// 将对象的markword指向lockrecord
        // Is it simple recursive case? // 判断是不是重入，如果是，简单将displacedHeader设置为null
        if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
        entry->lock()->set_displaced_header(NULL);
        } else {
        CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
    }
    }
```
*实际上只有上面的分支2会走到这里, 为什么只有分支2呢, 因为分支3和分支4其出现cas冲突就意味着获得锁的是其他线程, 而分支2是有可能存在轻量级锁重入的情况的, 所以这里有轻量级锁重入逻辑.*
这里将displaced header设置为unlocked的原因是等到解锁时,会将displaced header中的markword替换回对象头上,这时对象应该时无锁的所以直接设置为
unlocked.

在`cas_set_mark((markOop)entry, displaced)`中如果失败, 则意味着对象已经被锁住,这里使用`is_lock_owned`判断是否重入,如果是,那么直接设置displaced header为null
(在轻量级锁解锁时会解释为什么这么做),如果被其他线程持有,那么进入`InterpreterRuntime::monitorenter`逻辑中.

### 重偏向与撤销
```php
if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
} else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
}
```
可见,如果开启了偏向锁则会进入fast_enter.

fast_enter的代码如下:
```php
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock,
                                    bool attempt_rebias, TRAPS) {
  if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  slow_enter(obj, lock, THREAD);
}
```
如果当前不在安全点,那么会进入revoke_and_rebias方法中, 如果在安全点,那么由VMThread执行revoke_at_safepoint方法.

我们先看revoke_and_rebias方法.
```php
  if (mark->is_biased_anonymously() && !attempt_rebias) { // hashcode方法
    markOop biased_value       = mark;
    markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age()); // prototype是nohash + 1(无锁)
    markOop res_mark = obj->cas_set_mark(unbiased_prototype, mark);
    if (res_mark == biased_value) { // 如果成功设置,说明没有竞争
      return BIAS_REVOKED;
    }
  }
```
如果是匿名偏向且不允许重偏向,那么一般是计算对象的hashcode方法, 此时直接给对象设置一个无锁的markword.

如果对象存在偏向那么走下面的逻辑:
```php
else if (mark->has_bias_pattern()) {
    Klass* k = obj->klass();
    markOop prototype_header = k->prototype_header();
    if (!prototype_header->has_bias_pattern()) { // class关闭了偏向，简单cas即可，可能race codition，但是不要紧
      markOop biased_value       = mark;
      markOop res_mark = obj->cas_set_mark(prototype_header, mark);
      assert(!obj->mark()->has_bias_pattern(), "even if we raced, should still be revoked");
      return BIAS_REVOKED;
    } else if (prototype_header->bias_epoch() != mark->bias_epoch()) { // epoch过时,该对象不是锁
      if (attempt_rebias) {
        assert(THREAD->is_Java_thread(), "");
        markOop biased_value       = mark;
        markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
        markOop res_mark = obj->cas_set_mark(rebiased_prototype, mark); // 尝试重偏向到当前线程
        if (res_mark == biased_value) {
          return BIAS_REVOKED_AND_REBIASED; // 注意这里偏向成功则不会进入slow_enter
        }
      } else {
        markOop biased_value       = mark;
        markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());// 置为无锁
        markOop res_mark = obj->cas_set_mark(unbiased_prototype, mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED;
        }
      }
    }
  }
```
如果class关闭了偏向而对象开启偏向,那么说明在其他线程中发生了批量撤销, 直接将mark换成无锁的即可.

如果epoch过时,说明发生了批量重偏向,那么如果允许重偏向,则直接尝试cas掉对象的markword,将threadid换成当前线程的, 如果成功直接返回.
如果不允许重偏向,那么将markword换成无锁的.

如果上述cas中发生了竞争那么,会进入update_heuristics方法中,进行批量重偏向和批量撤销计数器的更新(重偏向阈值低于批量撤销),
根据该类的撤销次数会进行不同的逻辑:

- 单个撤销
- 批量重偏向
- 批量撤销

先看单个撤销,单个撤销的逻辑分为当前该对象偏向的线程是不是当前线程:
```php
else if (heuristics == HR_SINGLE_REVOKE) {
    Klass *k = obj->klass();
    markOop prototype_header = k->prototype_header();
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {// 需要撤销当前线程的偏向
      ResourceMark rm;
      log_info(biasedlocking)("Revoking bias by walking my own stack:");
      EventBiasedLockSelfRevocation event;
      BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD, NULL);
      ((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
      assert(cond == BIAS_REVOKED, "why not?");
      if (event.should_commit()) {
        post_self_revocation_event(&event, k);
      }
      return cond;
    } else { // 如果不是偏向当前线程需要到safepoint再撤销
      EventBiasedLockRevocation event;
      VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
      VMThread::execute(&revoke);
      if (event.should_commit() && revoke.status_code() != NOT_BIASED) {
        post_revocation_event(&event, k, &revoke);
      }
      return revoke.status_code();
    }
  }
```
如上,撤销当前线程的偏向不需要到安全点,因为不可能发生线程安全问题.如果不是当前线程需要push到VMThread中.

批量逻辑如下所示:
```php
EventBiasedLockClassRevocation event;
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
```

### 单个对象偏向撤销
```php
uint age = mark->age();
markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);
```
首先构造了一个匿名偏向的mark和一个无锁的mark.

接着判断当前偏向线程是否存活, 如果不存活那么设置好mark之后立即返回已撤销:
```php
bool thread_is_alive = false;
  if (requesting_thread == biased_thread) {
    thread_is_alive = true;
  } else {
    ThreadsListHandle tlh;
    thread_is_alive = tlh.includes(biased_thread);
  }
  if (!thread_is_alive) {
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      obj->set_mark(unbiased_prototype);
    }
    return BiasedLocking::BIAS_REVOKED;
```

接下来到了撤销逻辑的关键部分, 主要逻辑如下:
遍历对象当前偏向线程的所有lockrecord,查找出class相符合的,将lockrecord的displacedheader设置为null,
接着设置地址最高的lockrecord的displacedheader为该对象无锁的mark,并且对象的mark设置为地址最高的lockrecord的地址:
```php
GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    if (oopDesc::equals(mon_info->owner(), obj)) {
      log_trace(biasedlocking)("   mon_info->owner (" PTR_FORMAT ") == obj (" PTR_FORMAT ")",
                               p2i((void *) mon_info->owner()),
                               p2i((void *) obj));
      // Assume recursive case and fix up highest lock later
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      highest_lock->set_displaced_header(mark);
    } else {
      log_trace(biasedlocking)("   mon_info->owner (" PTR_FORMAT ") != obj (" PTR_FORMAT ")",
                               p2i((void *) mon_info->owner()),
                               p2i((void *) obj));
    }
  }
  if (highest_lock != NULL) {
    highest_lock->set_displaced_header(unbiased_prototype);
    obj->release_set_mark(markOopDesc::encode(highest_lock));
```

### 批量逻辑

批量重偏向的代码如下:
```php
if (bulk_rebias) {
      if (klass->prototype_header()->has_bias_pattern()) {
        int prev_epoch = klass->prototype_header()->bias_epoch();
        klass->set_prototype_header(klass->prototype_header()->incr_bias_epoch());
        int cur_epoch = klass->prototype_header()->bias_epoch();

        // Now walk all threads' stacks and adjust epochs of any biased
        // and locked objects of this data type we encounter
        for (; JavaThread *thr = jtiwh.next(); ) {
          GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
          for (int i = 0; i < cached_monitor_info->length(); i++) {
            MonitorInfo* mon_info = cached_monitor_info->at(i);
            oop owner = mon_info->owner();
            markOop mark = owner->mark();
            if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
              // We might have encountered this object already in the case of recursive locking
              assert(mark->bias_epoch() == prev_epoch || mark->bias_epoch() == cur_epoch, "error in bias epoch adjustment");
              owner->set_mark(mark->set_bias_epoch(cur_epoch));
            }
          }
        }
      }

      // At this point we're done. All we have to do is potentially
      // adjust the header of the given object to revoke its bias.
      revoke_bias(o, attempt_rebias_of_object && klass->prototype_header()->has_bias_pattern(), true, requesting_thread, NULL);
```
首先将klass中mark的epoch自增,然后遍历所有java线程中的当前正在使用的该类型的偏向锁对象,将其epoch更新,
从另一方面来说,也就使那些已偏向但偏向线程已退出或dead掉的该类型对象的epoch过期, 这就给了其他线程一个提示,
即可直接替换该对象中mark的threadId来完成重偏向.

批量撤销的逻辑如下:
```php
    klass->set_prototype_header(markOopDesc::prototype());

    // Now walk all threads' stacks and forcibly revoke the biases of
    // any locked and biased objects of this data type we encounter.
    for (; JavaThread *thr = jtiwh.next(); ) {
    GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
    for (int i = 0; i < cached_monitor_info->length(); i++) {
        MonitorInfo* mon_info = cached_monitor_info->at(i);
        oop owner = mon_info->owner();
        markOop mark = owner->mark();
        if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
        revoke_bias(owner, false, true, requesting_thread, NULL);
        }
    }
    }

    // Must force the bias of the passed object to be forcibly revoked
    // as well to ensure guarantees to callers
    revoke_bias(o, false, true, requesting_thread, NULL);
```
将klass中的mark设置为了不可偏向, 然后遍历所有线程栈并且撤销所有该类的偏向锁.

为什么要设置批量重偏向和批量撤销呢?

因为有可能出现一个线程创建了大量对象并且使用同步操作, 之后另一个线程使用这些对象作为锁则会出现大量的撤销操作,
为了防止这种情况, 设置批量重偏向可以给这个类型的对象一个重新使用偏向锁的机会而不是直接膨胀到轻量级锁.

但是当撤销次数继续增长达到批量撤销的阈值时,我们可以认为在该类型的对象上不适合使用偏向锁, 进而把当前所有偏向锁膨胀,而且新建对象也不使用偏向.

## 重量级锁

以上是膨胀至轻量级锁的逻辑, 接下来我们看如果fast_enter中没有成功重偏向会发生什么(没有开启偏向锁模式也会进入slow_enter):

```php
if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
        if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  slow_enter(obj, lock, THREAD); // 进入slow-enter
```

slowenter代码如下:
```php
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  if (mark->is_neutral()) {// 如果当前无锁，那么尝试使用lockrecord的地址cas掉对象的mark
    lock->set_displaced_header(mark);
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      TEVENT(slow_enter: release stacklock);
      return;
    }
    // cas失败, 膨胀至重量级锁
  } else if (mark->has_locker() &&
             THREAD->is_lock_owned((address)mark->locker())) { // 处理重入
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }

  // 将displaced——header设置为0(unused)
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD, // 膨胀
                              obj(),
                              inflate_cause_monitor_enter)->enter(THREAD);
}
```

接下来进入膨胀重量级锁的逻辑中:

```php
for (;;) {
    const markOop mark = object->mark();
    assert(!mark->has_bias_pattern(), "invariant");

    // 当前对象的markword
    // *  Inflated-已膨胀     - 其他线程将其膨胀了,直接返回
    // *  Stack-locked-轻量级锁 - 尝试膨胀
    // *  INFLATING-膨胀中    - 等待其他线程将其膨胀完成
    // *  Neutral-无锁      - 尝试膨胀
    // *  BIASED-偏向状态       - 非法,如果处于偏向锁模式,说明没有撤销,应该先撤销

    // CASE: inflated--已膨胀,直接返回
    if (mark->has_monitor()) {
      ObjectMonitor * inf = mark->monitor();
      assert(inf->header()->is_neutral(), "invariant");
      assert(oopDesc::equals((oop) inf->object(), object), "invariant");
      assert(ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
      return inf;
    }

    // CASE: 正在膨胀,等待自旋然后返回
    if (mark == markOopDesc::INFLATING()) {
      TEVENT(Inflate: spin while INFLATING); // 自旋
      ReadStableMark(object);
      continue;
    }

    // CASE: stack-locked-轻量级锁,尝试设置INFLATING,如果成功替换掉对象的markword
    // 为当前monitor地址

    if (mark->has_locker()) {
      ObjectMonitor * m = omAlloc(Self);
      // 优化:先准备好monitor,然后再设置INFLATING,防止INFLATING时间过长
      m->Recycle();
      m->_Responsible  = NULL;
      m->_recursions   = 0;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;   // Consider: maintain by type/class

      markOop cmp = object->cas_set_mark(markOopDesc::INFLATING(), mark);
      if (cmp != mark) { // 其他线程成功设置, 重试
        omRelease(Self, m, true);
        continue;       // Interference -- just retry
      }

      // 为什么使用一个INFLATING而不是直接设置monitor呢?
      // 这是因为在轻量级锁解锁的过程中,要将displacedHeader还原到对象头中,这时我们设置一个INFLATING
      // 可以让它cas失败,然后进入重量级锁的释放流程,而不是直接还原,造成hashcode值莫名其妙的改变
      markOop dmw = mark->displaced_mark_helper();
      assert(dmw->is_neutral(), "invariant");

      // Setup monitor fields to proper values -- prepare the monitor
      m->set_header(dmw);

      // 将owner设置为lockrecord
      m->set_owner(mark->locker());
      m->set_object(object);

      guarantee(object->mark() == markOopDesc::INFLATING(), "invariant");
      object->release_set_mark(markOopDesc::encode(m)); // 设置对象的mark为monitor的地址

      ...

      return m;
    }

    // CASE: 无锁,尝试直接替换markword

    assert(mark->is_neutral(), "invariant");
    ObjectMonitor * m = omAlloc(Self);
    // prepare m for installation - set monitor to initial state
    m->Recycle();
    m->set_header(mark);
    m->set_owner(NULL);
    m->set_object(object);
    m->_recursions   = 0;
    m->_Responsible  = NULL;
    m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;       // consider: keep metastats by type/class

    if (object->cas_set_mark(markOopDesc::encode(m), mark) != mark) {
      m->set_object(NULL);
      m->set_owner(NULL);
      m->Recycle();
      omRelease(Self, m, true);
      m = NULL;
      continue;
      // interference - the markword changed - just retry.
      // The state-transitions are one-way, so there's no chance of
      // live-lock -- "Inflated" is an absorbing state.
    }
```

以上就是对应当前对象的锁状态的膨胀逻辑, 具体就是设置了一个objectMonitor对象到对象的markword中.

### 重量级锁竞争逻辑
代码在`objetcMonitor::enter`中.

```php
void ObjectMonitor::enter(TRAPS) {
  Thread * const Self = THREAD;

  void * cur = Atomic::cmpxchg(Self, &_owner, (void*)NULL);// 尝试cas掉owner字段,只有在owner为空的情况下才成功.
  if (cur == NULL) {
    // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
    assert(_recursions == 0, "invariant");
    assert(_owner == Self, "invariant");
    return;
  }

  if (cur == Self) { // 重入,简单自增_recursions
    // TODO-FIXME: check for integer overflow!  BUGID 6557169.
    _recursions++;
    return;
  }

  if (Self->is_lock_owned ((address)cur)) { // 当前线程就是轻量级锁的拥有者,直接设置_owner
    assert(_recursions == 0, "internal state error");
    _recursions = 1;
    _owner = Self;
    return;
  }

  Self->_Stalled = intptr_t(this);

  // 尝试自旋
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
    assert(_owner == Self, "invariant");
    assert(_recursions == 0, "invariant");
    assert(((oop)(object()))->mark() == markOopDesc::encode(this), "invariant");
    Self->_Stalled = 0;
    return;
  }

  Atomic::inc(&_count);

  EventJavaMonitorEnter event;
  if (event.should_commit()) {
    event.set_monitorClass(((oop)this->object())->klass());
    event.set_address((uintptr_t)(this->object_addr()));
  }

  { // 设置java线程状态
    JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);

    Self->set_current_pending_monitor(this);

    OSThreadContendState osts(Self->osthread());
    ThreadBlockInVM tbivm(jt);

    for (;;) {
        // 用于suspend支持,这块没看太懂,但是现在基本不会使用suspend方法了,所以没多大用?
      jt->set_suspend_equivalent();

      EnterI(THREAD); // 进入队列_cxq竞争锁

      if (!ExitSuspendEquivalent(jt)) break;

      // We have acquired the contended monitor, but while we were
      // waiting another thread suspended us. We don't want to enter
      // the monitor while suspended because that would surprise the
      // thread that suspended us.
      //
      _recursions = 0;
      _succ = NULL;
      exit(false, Self);

      jt->java_suspend_self();
    }
    Self->set_current_pending_monitor(NULL);
  }

  Atomic::dec(&_count);
  Self->_Stalled = 0;

  ...
}
```
可以看到,先进行了一些特殊情况的尝试,然后进入EnterI方法,下面我们来看EnterI方法.

EnterI可分为几个步骤:
1. 插入cxq队列首部
2. 作为`_Responsible`或者普通线程在队列中等待
3. 获取锁后从cxq或者EntryList中移除(为什么会在EntryList接下来会解释)

_Responsible的park是有限时间的park,它会定期检查monitor的_owner字段,之所以这样设计是防止出现“stranding”即搁浅.

下面是具体代码:
```php
void ObjectMonitor::EnterI(TRAPS) {
  Thread * const Self = THREAD;
  assert(Self->is_Java_thread(), "invariant");
  assert(((JavaThread *) Self)->thread_state() == _thread_blocked, "invariant");

  // Try the lock - TATAS
  if (TryLock (Self) > 0) {
    assert(_succ != Self, "invariant");
    assert(_owner == Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }

  DeferredInitialize();

  if (TrySpin (Self) > 0) {
    assert(_owner == Self, "invariant");
    assert(_succ != Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }

  ObjectWaiter node(Self);
  Self->_ParkEvent->reset();
  node._prev   = (ObjectWaiter *) 0xBAD;
  node.TState  = ObjectWaiter::TS_CXQ;

  ObjectWaiter * nxt;
  for (;;) { // 插入队首
    node._next = nxt = _cxq;
    if (Atomic::cmpxchg(&node, &_cxq, nxt) == nxt) break;

    // 再次尝试TATAS
    if (TryLock (Self) > 0) {
      assert(_succ != Self, "invariant");
      assert(_owner == Self, "invariant");
      assert(_Responsible != Self, "invariant");
      return;
    }
  }
    // 当EntryList为空时,说明没有正在等待的线程,直接将_Responsible设置为自己
  if ((SyncFlags & 16) == 0 && nxt == NULL && _EntryList == NULL) {
    // Try to assume the role of responsible thread for the monitor.
    // CONSIDER:  ST vs CAS vs { if (Responsible==null) Responsible=Self }
    Atomic::replace_if_null(Self, &_Responsible);
  }

  int nWakeups = 0;
  int recheckInterval = 1;

  for (;;) {

    if (TryLock(Self) > 0) break; // 再次尝试获取TATAS
    assert(_owner != Self, "invariant");

    if ((SyncFlags & 2) && _Responsible == NULL) {
      Atomic::replace_if_null(Self, &_Responsible);
    }

    // 获取锁失败，park自己，分为是不是_Responsible，如果是那么是timedpark，
    // 间隔一定时间后尝试获取_owner
    if (_Responsible == Self || (SyncFlags & 1)) {
      TEVENT(Inflated enter - park TIMED);
      Self->_ParkEvent->park((jlong) recheckInterval);
      // Increase the recheckInterval, but clamp the value.
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) {
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      TEVENT(Inflated enter - park UNTIMED);
      Self->_ParkEvent->park();
    }

    if (TryLock(Self) > 0) break;

    ...

    ++nWakeups;

    if ((Knob_SpinAfterFutile & 1) && TrySpin(Self) > 0) break;

    ...

    if (_succ == Self) _succ = NULL; // 释放锁时，succ会被选取
  }


  // 从cxq或者entrylist上unlink自己
  assert(_owner == Self, "invariant");
  assert(object() != NULL, "invariant");
  // I'd like to write:
  //   guarantee (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
  // but as we're at a safepoint that's not safe.

  UnlinkAfterAcquire(Self, &node);
  if (_succ == Self) _succ = NULL;

  ...

  return;
}
```

### 重量级锁的释放
释放逻辑简单来说分为如下几步(非重入释放):
while(true) {
    1. 设置owner为null
    2. 如果有必要唤醒一个线程,那么唤醒(需要重新获得锁),否则直接退出
    3. 将cxq以某种方式拼接到EntryList中
    4. 如果此时EntryList不为空那么设置succ和owner字段,唤醒线程返回.否则进入5
    5. 再次将cxq插入到EntryList中, 如果EntryList不为空那么唤醒, 否则continue
}
```php
for (;;) {
    assert(THREAD == _owner, "invariant");

    if (Knob_ExitPolicy == 0) {// 默认为0
      OrderAccess::release_store(&_owner, (void*)NULL);   // 设置owner为null
      OrderAccess::storeload();                      
      if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) { // _succ也会在TrySpin中设置，如果存在的话就不用唤醒其他线程了
        TEVENT(Inflated exit - simple egress);
        return;
      }
      TEVENT(Inflated exit - complex egress);
      
      // 由于之后我们要操作cxq和entrylist, 所以需要重新获取锁,如果此处失败那么返回即可,由新的锁拥有者来处理唤醒线程
      if (!Atomic::replace_if_null(THREAD, &_owner)) {
        return;
      }
      TEVENT(Exit - Reacquired);
    } else {
      ...
    }

    ObjectWaiter * w = NULL;
    int QMode = Knob_QMode;

    // Qmode为2,那么直接从cxq中唤醒
    if (QMode == 2 && _cxq != NULL) {
      w = _cxq;
      assert(w != NULL, "invariant");
      assert(w->TState == ObjectWaiter::TS_CXQ, "Invariant");
      ExitEpilog(Self, w);
      return;
    }

    // QMode为3, 那么将cxq 抽干到EntryList中(放在EntryList末尾)
    if (QMode == 3 && _cxq != NULL) {
      w = _cxq;
      for (;;) {
        assert(w != NULL, "Invariant");
        ObjectWaiter * u = Atomic::cmpxchg((ObjectWaiter*)NULL, &_cxq, w); // 将cxq设置为null
        if (u == w) break;
        w = u;
      }
      assert(w != NULL, "invariant");

      ObjectWaiter * q = NULL;
      ObjectWaiter * p;
      for (p = w; p != NULL; p = p->_next) { // 更改状态为TS_ENTER
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
      }

      // 将cxq放在EntryList末尾
      ObjectWaiter * Tail;
      for (Tail = _EntryList; Tail != NULL && Tail->_next != NULL;
           Tail = Tail->_next)
        /* empty */;
      if (Tail == NULL) {
        _EntryList = w;
      } else {
        Tail->_next = w;
        w->_prev = Tail;
      }

    }

    // / QMode为3, 那么将cxq 抽干到EntryList中(放在EntryList头部)
    if (QMode == 4 && _cxq != NULL) {
      w = _cxq;
      for (;;) {
        assert(w != NULL, "Invariant");
        ObjectWaiter * u = Atomic::cmpxchg((ObjectWaiter*)NULL, &_cxq, w);
        if (u == w) break;
        w = u;
      }
      assert(w != NULL, "invariant");

      ObjectWaiter * q = NULL;
      ObjectWaiter * p;
      for (p = w; p != NULL; p = p->_next) {
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
      }

      // Prepend the RATs to the EntryList
      if (_EntryList != NULL) {
        q->_next = _EntryList;
        _EntryList->_prev = q;
      }
      _EntryList = w;

      // Fall thru into code that tries to wake a successor from EntryList
    }

    // _EntryList不为空,唤醒_EntryList的头部线程
    w = _EntryList;
    if (w != NULL) {
      assert(w->TState == ObjectWaiter::TS_ENTER, "invariant");
      ExitEpilog(Self, w);
      return;
    }

    // cxq和_EntryList都为空,那么重新将不空的cxq放在_EntryList中
    w = _cxq;
    if (w == NULL) continue;

    for (;;) {
      assert(w != NULL, "Invariant");
      ObjectWaiter * u = Atomic::cmpxchg((ObjectWaiter*)NULL, &_cxq, w);
      if (u == w) break;
      w = u;
    }

    // QMode为1, 将cxq逆序
    if (QMode == 1) {
      // QMode == 1 : drain cxq to EntryList, reversing order
      // We also reverse the order of the list.
      ObjectWaiter * s = NULL;
      ObjectWaiter * t = w;
      ObjectWaiter * u = NULL;
      while (t != NULL) {
        guarantee(t->TState == ObjectWaiter::TS_CXQ, "invariant");
        t->TState = ObjectWaiter::TS_ENTER;
        u = t->_next;
        t->_prev = u;
        t->_next = s;
        s = t;
        t = u;
      }
      _EntryList  = s;
      assert(s != NULL, "invariant");
    } else {
      // QMode == 0 or QMode == 2, 将cxq正序插入, 最晚的线程在最前面
      _EntryList = w;
      ObjectWaiter * q = NULL;
      ObjectWaiter * p;
      for (p = w; p != NULL; p = p->_next) {
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
      }
    }

    // context-switch rate.
    if (_succ != NULL) continue;

    w = _EntryList;
    if (w != NULL) {
      guarantee(w->TState == ObjectWaiter::TS_ENTER, "invariant");
      ExitEpilog(Self, w);
      return;
    }
  }
```

ExitEpilog方法很简单,就不解释了:
```php
void ObjectMonitor::ExitEpilog(Thread * Self, ObjectWaiter * Wakee) {
  assert(_owner == Self, "invariant");

  // Exit protocol:
  // 1. ST _succ = wakee
  // 2. membar #loadstore|#storestore;
  // 2. ST _owner = NULL
  // 3. unpark(wakee)

  _succ = Knob_SuccEnabled ? Wakee->_thread : NULL;
  ParkEvent * Trigger = Wakee->_event;

  // Hygiene -- once we've set _owner = NULL we can't safely dereference Wakee again.
  // The thread associated with Wakee may have grabbed the lock and "Wakee" may be
  // out-of-scope (non-extant).
  Wakee  = NULL;

  // Drop the lock
  OrderAccess::release_store(&_owner, (void*)NULL);
  OrderAccess::fence();                               // ST _owner vs LD in unpark()

  if (SafepointMechanism::poll(Self)) {
    TEVENT(unpark before SAFEPOINT);
  }

  DTRACE_MONITOR_PROBE(contended__exit, this, object(), Self);
  Trigger->unpark();

  // Maintain stats and report events to JVMTI
  OM_PERFDATA_OP(Parks, inc());
}
```

关于同步关键字的源码实现分析到此结束, 对于wait/notify等原理的分析会以后再分析吧.

吐槽: 我是个没什么毅力的人, 写这个也只是心血来潮, 而且写的较乱, 一些基本的概念没有介绍,比如markword的不同结构以及ObjectMonitor的组成
,这些可以画一张图的, 这些以后补充吧.

JDK中的AQS和ObjectMonitor设计的有相似之处,但是也有很多不同, 比如这里有Responsible线程, 还有_succ字段, 这些在AQS里面是没有的,
而且JVM的这个monitor只有非公平模式, 而AQS可以是非公平也可以是公平的.还有很多其他的不同这里就不列举了.