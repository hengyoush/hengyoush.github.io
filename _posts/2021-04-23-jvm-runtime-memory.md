---
layout: post
title:  "JVM笔记-内存区域以及GC收集器"
date:   2021-04-24 19:20:00 +0700
categories: [java]
---

|名称|是否线程私有|可能抛出的异常|作用|  
|-------|-------|-----|--------------|  
|PC、程序计数器|私有|无|用于当前线程执行字节码的行号,字节码解释器就通过改变这个计数器的值来选取下一条要执行的字节码指令.  |  
|本地方法栈|私有|StackOverflow、OOM|本地方法栈为本地方法服务|  
|JAVA堆|公有|OOM|用于存放对象的实例,由GC收集器管理  |  
|方法区|公有|OOM|用于存储被虚拟机加载的类型信息、常量、静态变量、代码缓存等.Hotspot在JDK8之后将这部分由原先存储在堆的永久代中改为本地内存.里面包含运行时常量池,在Class文件中的Constant Pool Table内的内容会在类加载完成之后放入到常量池中.  |  
|直接内存|公有|OOM|比较典型的例子是DirectByteBuffer,它真正存储数据的地方在本地内存中,不受Java堆管理.  |  


![avatar](/static/img/JVM-运行时内存.png)


**对象的创建步骤**  
1. 分配内存
2. 初始化零值
3. 设置对象头:类型信息+MarkWord+数组长度

**如何分配内存?**
内存是一块连续的区域.方法有两个:  
- 指针碰撞:常用在具备压缩整理的GC堆中,如Serial、ParNew.
- 空闲列表:常用在基于清除算法的GC堆中,如CMS.

**TLAB(Thread Local Allocation Buffer)**
目的是解决如下问题:创建对象修改指针是很频繁的行为,如果多线程同时操作为了达到线程安全,必须加锁.
为了提高性能,每个线程在JAVA堆中实际上都有一小块内存,称为线程本地分配缓冲,只有本地缓冲用完了,
才会同步锁定.

**找出存活对象的算法**
1. 引用计数
2. 可达性分析

**分代收集**  
1. 弱分代假说:绝大多数对象都是朝生夕灭的
2. 强分代假说:熬过越多次垃圾收集的对象就越难以消亡.
3. 跨代引用假说:跨代引用相对于同代引用来说只是少数.

**记忆集(谁引用了我)**   
CMS垃圾收集时,为了不去扫描整个old区,特在eden区中建立一个全局的数据结构:RememberSet,用于记录有哪些老年代的区域
引用了新生代,扫描GCRoots的时候只扫描那一部分.
具体来说就是记录非收集区域到收集区域的指针集合的数据结构.在改变引用关系的时候需要记录维护记忆集,这是通过写屏障进行的.

## 收集算法   
|算法名称|过程|优缺点|备注|  
|---------|----------|----------|-----------------|  
标记-清除|标记所有需要回收的对象,进行清除|1. 执行效率不稳定,如果存在大量对象是要回收的,标记与清除的执行效率随对象数量增长而降低;2. 容易造成内存碎片问题|无  
标记-复制|将内存分为两个区,然后标记所有存活对象,将存活对象拷贝至另外一个区中|优点:解决了内存碎片的问题,实现简单 缺点:只适用于新生代,大量对象朝生夕灭的情况. | 可能存在复制的时候另外一个区容量不够的情况,这时候需要分配担保  
标记-整理|标记所有存活对象,将存活对象移动到内存的一端|优点:解决内存碎片问题   缺点:GC时要求STW||

## Hostspot垃圾收集术语解释

### 根节点枚举
要在在常量、静态变量和执行上下文中搜索.
借助于OomMap,我们可以快速知道对象内的哪一个偏移量是引用也可以知道栈和寄存器哪些位置是引用.

### 安全点
安全点:那些比较耗时的操作,比如方法调用,循环,跳转等.

只有在安全点才会加入OopMap信息.因此要在STW时让用户线程跑到有安全点的指令上.
两种方法:
抢先式中断:由JVM中断所有线程,如果发现有线程不在安全点,恢复其运行直到跑到安全点上.
主动式中断:在STW时,JVM只设置一个标志位,在编译后的指令中插入轮询标志位的代码,如果线程发现标志位设置了,那么就跑到安全点上挂起.
目前的实现是在安全点的地方就插入轮询指令.

#### 安全区域
如线程在SLEEP、BLOCK时,在安全区域内,不会发生引用关系的改变.
线程进入安全区域时,会标记自己,如果线程尝试离开,会查询JVM是否已经完成枚举,如果完成了直接离开,如果没完成必须等待直到收到
可以离开安全区域的信号为止.


#### 记忆集与卡表
有哪些地方引用了我,这个精度可以是字长、对象、内存区域.
hotspot使用的是内存区域为精度,实现是卡表.一个内存区域是2的9次方,512字节为一个区域,实现是一个数组:
`boolean[max_size >>> 9]`,如果为1代表该卡是脏的,存在引用关系,扫描的时候扫描这部分就可以.

卡表元素如何维护?
使用**写屏障**,在对引用赋值的地方,加一个类似Around切面,在这个切面里完成卡表的维护

#### 可达性分析
三色标记:
- 白色:未被访问过
- 黑色:已被访问过,且其所有引用已被访问过
- 灰色:已被访问过,但是其引用没有全部被访问过.

增量更新:插入黑色到白色的引用时记录下来,标记结束时再次从这些黑色对象开始扫描
原始快照:在删除灰色到白色对象的引用时记录下来,标记结束时,再次从这些灰色对象开始扫描

#### GC收集器

|收集器名称|收集区域|是否并行|是否并发|使用算法|优点|缺点|
|----|---|--|---|--|---------|--------|
|Serial|新生代|否|否|标记-复制|简单高效|全程STW,暂停用户线程|
|ParNew|新生代|是|否|标记-复制|相比Serial实现了多线程,在cpu资源充足的情况下效率比Serial高|仍然需要暂停用户线程|
|Parallel Scavenge|新生代|是|否|标记-复制|关注点是可控的吞吐量,吞吐量=(运行用户代码时间)/(运行用户代码时间+运行GC时间).提供了两个参数来控制吞吐量,最大停顿时间:-XX:MaxGcPauseMillis和-XX:GCTimeRatio.还可以使用-XX:UseAdaptiveSizePolicy参数来动态调整新生代、老年代大小以及晋升对象大小|仍然需要暂停用户线程|  
|Serial Old|老年代|否|否|标记-整理|简单高效,作为CMS在发生Concurrent Mode Failure时使用|全程STW,暂停用户线程|  
|Parallel Old|老年代|是|否|标记-整理|和Parallel Scavenge相同|和Parallel Scavenge相同|  
|CMS|老年代|是|是(并发标记时,在初始、最终标记时依然要STW)|标记-清除|实现了并发与低停顿|1.无法处理浮动垃圾,有可能发生Concurrent Mode Failure导致另一次完全STW的FullGC(因为在并发标记和并发清理的时候,垃圾仍然在不断产生,所以此时要预留一些Old区的空间),触发条件是参数:-XX:CMSInitiatingOccupancyFraction,默认为到达old区的68%就开始FullGC. 2. 基于标记-清除算法会造成内存碎片化,如果分配对象时无法找到足够大的连续空间,那么会造成FullGC|  
|G1GC|全区域|是|是|宏观:标记-整理,微观:标记-复制|可指定停顿时间、不会产生内存碎片|1.记忆集占有堆内存大约10%到20%,内存消耗比CMS大.|

#### CMS收集器收集步骤

1. 初始标记-STW
2. 并发标记-并发
3. 最终标记-STW
4. 并发清理-并发

使用增量更新处理并发标记中改变的引用.

#### G1GC
1. 初始标记-STW,在YoungGC中完成
2. 并发标记-并发
3. 最终标记-STW
4. 筛选回收-STW,并行

##### G1GC日志分析
## MinorGC
2021-04-25T08:45:45.592+0800: 26304.309: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0612649 secs]
   [Parallel Time: 52.0 ms, GC Workers: 43]
[GC Worker Start (ms): Min: 26304311.9, Avg: 26304312.3, Max: 26304312.7, Diff: 0.8]
[Ext Root Scanning (ms): Min: 7.7, Avg: 9.8, Max: 10.9, Diff: 3.2, Sum: 423.2]
[Update RS (ms): Min: 4.9, Avg: 6.7, Max: 9.4, Diff: 4.4, Sum: 290.0]
[Processed Buffers: Min: 1, Avg: 17.0, Max: 40, Diff: 39, Sum: 733]
[Scan RS (ms): Min: 0.1, Avg: 0.5, Max: 0.7, Diff: 0.6, Sum: 19.5]
[Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
[Object Copy (ms): Min: 30.5, Avg: 33.1, Max: 34.7, Diff: 4.2, Sum: 1424.3]
[Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.9]
[Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 43]
[GC Worker Other (ms): Min: 0.0, Avg: 0.3, Max: 1.1, Diff: 1.0, Sum: 13.9]
[GC Worker Total (ms): Min: 49.9, Avg: 50.5, Max: 51.1, Diff: 1.2, Sum: 2172.0]
[GC Worker End (ms): Min: 26304362.5, Avg: 26304362.8, Max: 26304363.1, Diff: 0.6]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 1.4 ms]
   [Other: 7.8 ms]
[Choose CSet: 0.0 ms]
[Ref Proc: 1.4 ms]
[Ref Enq: 0.0 ms]
[Redirty Cards: 0.8 ms]
[Humongous Register: 0.0 ms]
[Humongous Reclaim: 0.0 ms]
[Free CSet: 2.4 ms]
[Eden: 3396.0M(3396.0M)->0.0B(3408.0M) Survivors: 140.0M->128.0M Heap: 7226.3M(8192.0M)->3832.9M(8192.0M)]
[Times: user=0.84 sys=1.01, real=0.06 secs]
1. Parallel Time这一段是并行进行的，不是并发，是STW的
2. GC Worker Start，GC worker线程启动的时间
3. Ext Root Scanning，根节点枚举
4. Update RS，每个线程更新RS的时间，用户线程在改变对象图和指向特定region的引用时，我们保持对这些改动的跟踪，将这些改动记录到Update Buffers中。
5. Processed Buffers, 对Update Buffers处理的时间
6. Scan RS，线程扫描RS消耗的时间，这个阶段扫描RS中的卡表，寻找指向CSet中的所有引用
7. Object Copy，将Cset中的存活对象复制到其他region
8. Termination，终止时间。
9. GC Worker Other，单线程处理的其他任务
10. Clear CT，清理卡表，单线程
11.Ref Proc和Ref Enq，处理引用类型，并将其入队的时间（Pending Reference）


## MixedGC
这里有一个initial-mark，代表mixed的初始标记在这次younggc中完成。
```
2021-04-25T08:45:45.654+0800: 26304.371: [GC concurrent-root-region-scan-start]
2021-04-25T08:45:45.680+0800: 26304.397: [GC concurrent-root-region-scan-end, 0.0262306 secs]
2021-04-25T08:45:45.680+0800: 26304.397: [GC concurrent-mark-start]
2021-04-25T08:45:45.880+0800: 26304.597: [GC concurrent-mark-end, 0.2003583 secs]
2021-04-25T08:45:45.883+0800: 26304.601: [GC remark 2021-04-25T08:45:45.883+0800: 26304.601: [Finalize Marking, 0.0016711 secs] 2021-04-25T08:45:45.885
+0800: 26304.602: [GC ref-proc, 3.7604799 secs] 2021-04-25T08:45:49.645+0800: 26308.363: [Unloading, 0.0557080 secs], 3.8326448 secs]
[Times: user=3.86 sys=2.58, real=3.83 secs]
2021-04-25T08:45:49.718+0800: 26308.436: [GC cleanup 3902M->3878M(8192M), 0.0058618 secs]
[Times: user=0.11 sys=0.00, real=0.00 secs]
2021-04-25T08:45:49.725+0800: 26308.442: [GC concurrent-cleanup-start]
2021-04-25T08:45:49.725+0800: 26308.442: [GC concurrent-cleanup-end, 0.0000819 secs]
```

接下来concurrent-root-region-scan-start代表开始扫描GCRoot引用的对象，这个过程没有暂停应用线程；是后台线程并行处理的。
这个阶段不能被YGC所打断、因此后台线程有足够的CPU时间很关键。如果Young区空间恰好在Root扫描的时候满了、
YGC必须等待root扫描之后才能进行。带来的影响是YGC暂停时间会相应的增加。
```
350.994: [GC pause (young)
351.093: [GC concurrent-root-region-scan-end, 0.6100090]
351.093: [GC concurrent-mark-start],0.37559600 secs]
```

之后进入并发标记阶段  
```
2021-04-25T08:45:45.680+0800: 26304.397: [GC concurrent-mark-start]
2021-04-25T08:45:45.880+0800: 26304.597: [GC concurrent-mark-end, 0.2003583 secs]
```

之后进入最终标记阶段，这时候会短暂暂停（G1使用的是原始快照方式）
```
2021-04-25T08:45:45.883+0800: 26304.601: [GC remark 2021-04-25T08:45:45.883+0800: 26304.601: [Finalize Marking, 0.0016711 secs] 2021-04-25T08:45:45.885
+0800: 26304.602: [GC ref-proc, 3.7604799 secs] 2021-04-25T08:45:49.645+0800: 26308.363: [Unloading, 0.0557080 secs], 3.8326448 secs]
[Times: user=3.86 sys=2.58, real=3.83 secs]
```

清理阶段，STW
```
2021-04-25T08:45:49.718+0800: 26308.436: [GC cleanup 3902M->3878M(8192M), 0.0058618 secs]
[Times: user=0.11 sys=0.00, real=0.00 secs]
```

并发清理
```
2021-04-25T08:45:49.725+0800: 26308.442: [GC concurrent-cleanup-start]
2021-04-25T08:45:49.725+0800: 26308.442: [GC concurrent-cleanup-end, 0.0000819 secs]
```

接下来进入mixed清理阶段
第一次：
```
2021-04-25T08:46:17.637+0800: 26336.354: [GC pause (G1 Evacuation Pause) (mixed), 0.0754858 secs]
   [Parallel Time: 66.7 ms, GC Workers: 43]
[GC Worker Start (ms): Min: 26336355.9, Avg: 26336356.2, Max: 26336356.6, Diff: 0.8]
[Ext Root Scanning (ms): Min: 2.6, Avg: 3.0, Max: 3.4, Diff: 0.8, Sum: 131.1]
[Update RS (ms): Min: 4.0, Avg: 5.0, Max: 9.3, Diff: 5.3, Sum: 212.9]
[Processed Buffers: Min: 2, Avg: 16.4, Max: 44, Diff: 42, Sum: 705]
[Scan RS (ms): Min: 6.5, Avg: 9.7, Max: 11.2, Diff: 4.8, Sum: 415.2]
[Code Root Scanning (ms): Min: 0.0, Avg: 0.1, Max: 1.8, Diff: 1.8, Sum: 5.5]
[Object Copy (ms): Min: 46.1, Avg: 47.5, Max: 48.4, Diff: 2.3, Sum: 2043.0]
[Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 1.1]
[Termination Attempts: Min: 1, Avg: 1.0, Max: 2, Diff: 1, Sum: 44]
[GC Worker Other (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 3.3]
[GC Worker Total (ms): Min: 65.0, Avg: 65.4, Max: 65.8, Diff: 0.8, Sum: 2812.1]
[GC Worker End (ms): Min: 26336421.6, Avg: 26336421.6, Max: 26336421.8, Diff: 0.2]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 1.4 ms]
   [Other: 7.3 ms]
[Choose CSet: 0.4 ms]
[Ref Proc: 1.8 ms]
[Ref Enq: 0.0 ms]
[Redirty Cards: 1.0 ms]
[Humongous Register: 0.1 ms]
[Humongous Reclaim: 0.0 ms]
[Free CSet: 1.9 ms]
   [Eden: 276.0M(276.0M)->0.0B(356.0M) Survivors: 132.0M->52.0M Heap: 4101.1M(8192.0M)->3390.8M(8192.0M)]
[Times: user=1.44 sys=1.30, real=0.07 secs]
```
第二次...

在mixed回收结束后，会回到回收young区的阶段.








