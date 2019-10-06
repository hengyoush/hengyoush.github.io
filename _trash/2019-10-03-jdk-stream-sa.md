---
layout: post
title:  "JDK8 stream api源码简单分析"
date:   2019-10-06 18:12:00 +0700
categories: [jdk,sa]
---

## 学习目标
搞清楚stream操作的原理.

## 开篇
```java
ArrayList<String> arrayList = new ArrayList<>();
arrayList.add("foo");
arrayList.add("bar");
arrayList.stream().map(String::length).map(Integer::getByteValue).collect(Collectors.toList());
```
如上, 是我们在代码中经常运用到的集合流式操作,
在使用Jdk8带来的stream api和Lambda表达式的方便的同时, 我觉得我们有必要了解其背后的运行机制.

## 概念
1. spliterator
splitable iterator, 即可分割的迭代器, 除了与正常的迭代器相同的功能之外, 还可以通过trySplit方法来创造出一个新的
迭代器.
java中的接口为:java.util.Spliterator
主要方法如下:
`boolean tryAdvance(Consumer<? super T> action)`: 如果还有剩余元素存在, 那么在那个元素上执行action指定的动作, 否则返回null.
`Spliterator<T> trySplit()`: 将元素集合切分, 返回一个新的spliterator, 用于可能的并行操作.

2. sink、中间操作、终止操作
sink就是在stream操作中的每一步操作, 分为中间操作和终止操作两种.
中间操作如上所示的map, 终止操作如上所示的collect.
而中间操作又可分为无状态操作和有状态操作, map就是一个无状态的操作. 
所谓无状态操作是指对流中当前元素的操作不依赖于对其他元素进行操作的结果.
而有状态操作如distinct, skip, distinct将流中元素去重, 对每个元素进行判断是否重复是依赖于之前判断的
结果来进行的.

终止操作主要包括collect, find*, reduce, forEach等.一旦使用这些操作那么流会被消费掉, 返回一个结果.

3. characteristics
characteristics可分为Collector的characteristics, Spliterator的characteristics.
Spliterator的characteristics:

- ORDERED    = 0x00000010

- DISTINCT   = 0x00000001
- SORTED     = 0x00000004
- SIZED      = 0x00000040
- NONNULL    = 0x00000100
- IMMUTABLE  = 0x00000400
- CONCURRENT = 0x00001000
- SUBSIZED   = 0x00004000

4. 链表结构
stream中的每一步操作最终的表示形式是链表结构的.
如下图所示, 我们的示例代码最终会表示成如下形式.
![avatar](/static/img/jdk-stream-lianbiao.png)


## 源码解析

我们接下来就以上面那串代码为例, 来讲解stream api的原理.
首先看ArrayList的stream方法:
使用的是父接口的Collection的stream方法:
```java
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}
```
spliterator方法返回一个Collection专用的spliterator.如下:
```java
default Spliterator<E> spliterator() {
    return Spliterators.spliterator(this, 0);
}
```
这是Collection提供的默认方法, 在ArrayList中覆盖了这个方法:
```java
public Spliterator<E> spliterator() {
    return new ArrayListSpliterator<>(this, 0, -1, 0);
}
```

### ArrayListSpliterator源码解析
我们可以看一下ArrayListSpliterator的trySplit方法和tryAdvance方法:
```java
public ArrayListSpliterator<E> trySplit() {
    // getFence返回当前spliterator的最大index
    int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
    return (lo >= mid) ? null :
        new ArrayListSpliterator<E>(list, lo, index = mid,
                                    expectedModCount);
}
```
可以看到, trySplit将当前数组平分为2半, 返回低spliterator是lo到mid的, 而自己则剩下mid到fence的一部分.
下面是ArrayListSpliterator的tryAdvance方法.
```java
public boolean tryAdvance(Consumer<? super E> action) {
    int hi = getFence(), i = index;
    if (i < hi) {
        index = i + 1;
        E e = (E)list.elementData[i];
        action.accept(e);
        return true;
    }
    return false;
}
```
可以看到tryAdvance将index向前推进了一位, 然后使用action对元素进行处理; 如果迭代到底了那么返回false.

下面是forEachRemaining方法, 用于遍历迭代器中剩余的元素
```java
public void forEachRemaining(Consumer<? super E> action) {
    int i, hi, mc; // hoist accesses and checks from loop
    ArrayList<E> lst; Object[] a;
    if ((lst = list) != null && (a = lst.elementData) != null) {
        // ...
        for (; i < hi; ++i) {
                E e = (E) a[i];
                action.accept(e);
        }
    }
    throw new ConcurrentModificationException();
}
```
可以看到, forEachRemaining的实现逻辑为遍历数组中的元素进行处理.

---

接着调用了StreamSupport的stream方法
```java
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
    Objects.requireNonNull(spliterator);
    return new ReferencePipeline.Head<>(spliterator,
                                        StreamOpFlag.fromCharacteristics(spliterator),
                                        parallel);
}
```
该方法new了一个ReferencePipeline.Head对象, 这个对象就是我们之前提到的链表结构的头部.
Head相关的类图如下所示:
![avatar](/static/img/jdk-stream-pipeline-class.png)

其中我们首先分析一下AbstractPipeline类的相关重要的属性:
- sourceStage: 指向pipeline链表的头部
- previousStage: 指向链表中上一个pipeline
- nextStage: 指向链表中下一个pipeline

这是stream方法返回的对象, 接着我们继续看map方法会做什么操作.

下面是map方法:
```java
public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                    StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
        Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
            return new Sink.ChainedReference<P_OUT, R>(sink) {
                public void accept(P_OUT u) {
                    downstream.accept(mapper.apply(u));
                }
            };
        }
    };
}
```
可以看到map方法返回了一个StatelessOp对象, 由此可见每一个中间步骤都会返回一个新的ReferencePipeline对象.
在StatelessOp对象中又一个orWrapSink方法, 这里复写了它, 关于这个方法的作用我们之后再进行分析,
先来说说Sink接口:

Sink它继承了Consumer接口, 其中Sink.ChainedReference是它的内部实现类, 稍后可以看到Sink是
不同pipeline进行交互的一个类, 通过Sink的consume方法. 其中Sink包含了真正进行元素处理的
Function或者Consumer

这里有一个downstream, 这个downstream就是下一个pipeline的sink.

接着我们分析collect方法, 在此之前我们先看一下Collector接口, 主要有如下几个方法:

- supplier: 提供一个初始值
- accumulator: 累加器
- combiner: 并行计算时用到, 将两个部分结果合并为一个结果.

如下是ReferencePipeline的collect方法, 参数接收一个Collector对象
```java
public final <R, A> R collect(Collector<? super P_OUT, A, R> collector) {
    A container;
    if (isParallel()
            && (collector.characteristics().contains(Collector.Characteristics.CONCURRENT))
            && (!isOrdered() || collector.characteristics().contains(Collector.Characteristics.UNORDERED))) {
        container = collector.supplier().get();
        BiConsumer<A, ? super P_OUT> accumulator = collector.accumulator();
        forEach(u -> accumulator.accept(container, u));
    }
    else {
        container = evaluate(ReduceOps.makeRef(collector));
    }
    return collector.characteristics().contains(Collector.Characteristics.IDENTITY_FINISH)
            ? (R) container
            : collector.finisher().apply(container);
}
```
大致逻辑如下:
1. 先判断是否可以并行处理
    1.1 collector可以并行处理, 接着构造一个ForeachTask进行并行处理
    1.2 collector不能并行处理, 那么就构造一个ReduceTask进行并行处理(如果开启了并行的话)(ForeachTask与ReduceTask的区别就是
    ReduceTask比ForeachTask多一个combine的阶段)

    *为什么ForeachTask可以没有combine阶段? 那是因为ForeachTask是在线程安全的collector存在的情况下使用的, 所以每个线程上的每一次累加都是实时存入最终结果的,而在非线程安全情况下使用的ReduceTask只有在两个subtask都完成的情况下之后在单线程内实施combine操作才能保证线程安全*
2. 执行完毕之后根据是否需要对结果进行转换进行不同的操作, 如果collecor提供了Finisher的话使用funisher进行转换.


接下来我们来看Collecotrs.toList()返回的是一个什么Collecotr?
```java
public static <T>
Collector<T, ?, List<T>> toList() {
    return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                (left, right) -> { left.addAll(right); return left; },
                                CH_ID);
}
```
可以看到supplier是new了一个ArrayList, 累加器是List::add, 并行操作下结果的合并使用的是Lisy::add.


 ### TerminalOp执行过程解析
 我们首先看AbstractPipeline::evaluate方法:
 ```java
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
    // ...(省略部分代码)
    return isParallel()
            ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
            : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
}
 ```
根据是否设置了sourceStage的parallel决定是否使用并行计算(可以使用AbstractPipeline::parallel方法设置)
首先看sourceSpliteratorff方法:
```java
private Spliterator<?> sourceSpliterator(int terminalFlags) {
    // 省略校验代码...

    if (isParallel() && sourceStage.sourceAnyStateful) {
        // 提前计算statefulOp
        int depth = 1;
        for (AbstractPipeline u = sourceStage, p = sourceStage.nextStage, e = this;
                u != e;
                u = p, p = p.nextStage) {

            int thisOpFlags = p.sourceOrOpFlags;
            if (p.opIsStateful()) {
                // 下一个stage是有状态的话, 将下一个stage的depth清零(depth在wrapSink的时候用到)
                depth = 0;

                if (StreamOpFlag.SHORT_CIRCUIT.isKnown(thisOpFlags)) {
                    thisOpFlags = thisOpFlags & ~StreamOpFlag.IS_SHORT_CIRCUIT;
                }
                // 包装一个新的spliterator
                spliterator = p.opEvaluateParallelLazy(u, spliterator);

                thisOpFlags = spliterator.hasCharacteristics(Spliterator.SIZED)
                        ? (thisOpFlags & ~StreamOpFlag.NOT_SIZED) | StreamOpFlag.IS_SIZED
                        : (thisOpFlags & ~StreamOpFlag.IS_SIZED) | StreamOpFlag.NOT_SIZED;
            }
            p.depth = depth++;
            p.combinedFlags = StreamOpFlag.combineOpFlags(thisOpFlags, u.combinedFlags);
        }
    }

    if (terminalFlags != 0)  {
        // Apply flags from the terminal operation to last pipeline stage
        combinedFlags = StreamOpFlag.combineOpFlags(terminalFlags, combinedFlags);
    }

    return spliterator;
}
```
可以看到, sourceSpliterator主要是为了获取能够遍历元素的spliterator,
其中对于并行的有状态操作进行了处理, 将spliterator进行包装, 包装后的
spliterator存储着中间状态, 比如DistinctOp的spliterator的中间状态——一个维持唯一的Set,
SliceOp(limit和skip会用到)存储着当前切分的索引和边界.

接下来我们首先看evaluateParallel方法:
evaluateParallel方法有四种实现分别对应着不同的终结操作:

- Find
- ForEach
- Match
- Reduce

我们这里以Foreach的实现为例, ForeachTask
```java
// 首先是ForEachOp::evaluateParallel, 创建了ForeachTask的任务用于进行Fork/Join并行运算
public <S> Void evaluateParallel(PipelineHelper<T> helper,
                                    Spliterator<S> spliterator) {
    if (ordered)
        new ForEachOrderedTask<>(helper, spliterator, this).invoke();
    else
        new ForEachTask<>(helper, spliterator, helper.wrapSink(this)).invoke();
    return null;
}
// AbstractPipeline::wrapSink方法, 从尾至头将链表中的sink串起来, 注意如果
// 链表中含有有状态操作那么“头”是指最后一个有状态操作
final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
    Objects.requireNonNull(sink);

    for ( @SuppressWarnings("rawtypes") AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
        sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
    }
    return (Sink<P_IN>) sink;
}

// ForeachTask::compute, 具体进行服务切分和运算的逻辑
public void compute() {
    Spliterator<S> rightSplit = spliterator, leftSplit;
    long sizeEstimate = rightSplit.estimateSize(), sizeThreshold;
    if ((sizeThreshold = targetSize) == 0L)
        targetSize = sizeThreshold = AbstractTask.suggestTargetSize(sizeEstimate);
    boolean isShortCircuit = StreamOpFlag.SHORT_CIRCUIT.isKnown(helper.getStreamAndOpFlags());
    boolean forkRight = false;
    Sink<S> taskSink = sink;
    ForEachTask<S, T> task = this;
    while (!isShortCircuit || !taskSink.cancellationRequested()) {
        // 当前元素数小于sizeThreshold或者切分出来的leftSplit为null
        // 那么直接将迭代器中的元素拷贝到sink中, 否则进行切分
        if (sizeEstimate <= sizeThreshold ||
            (leftSplit = rightSplit.trySplit()) == null) {
            task.helper.copyInto(taskSink, rightSplit);
            break;
        }
        // 创建新的leftSubTask
        ForEachTask<S, T> leftTask = new ForEachTask<>(task, leftSplit);
        task.addToPendingCount(1);
        ForEachTask<S, T> taskToFork;
        // 左右交替切分
        if (forkRight) {
            forkRight = false;
            rightSplit = leftSplit;
            taskToFork = task;
            task = leftTask;
        }
        else {
            forkRight = true;
            taskToFork = leftTask;
        }
        taskToFork.fork();
        sizeEstimate = rightSplit.estimateSize();
    }
    task.spliterator = null;
    task.propagateCompletion();
}
}
```
下面简单说下compute的逻辑
1. 首先根据当前task元素数判断是否需要创建subTask
2. 如果不需要, 调用copyInto方法将元素拷贝到Sink中
3. 如果需要, 那么调用迭代器的trySplit方法将元素集合切分.

trySplit方法在ArrayList中实现就是从中间切分数组.
下面我们看copyInto方法的实现:
```java
// AbstractPipeline::copyInto, 遍历迭代器中的元素, 使用组合后的sink进行处理
final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
    Objects.requireNonNull(wrappedSink);

    if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
        wrappedSink.begin(spliterator.getExactSizeIfKnown());
        spliterator.forEachRemaining(wrappedSink);
        wrappedSink.end();
    }
    else {
        copyIntoWithCancel(wrappedSink, spliterator);
    }
}
```
如上, 可以看到这里使用了迭代器的forEachRemaing方法遍历迭代器中的元素将其使用
sink处理.

以上是对于并行流的处理的解析, 关于串行流的分析这里不再赘述.

## 总结
我们以ArrayList的stream为例, 讲解了关于stream api的一些原理, 

对于我们例子中的情况: 中间操作无状态 + 串行流 + Foreach终止操作

1. 首先构造一个sourceSpiterator提供遍历元素和拆分流的功能(这里是串行所以没有用到拆分流的功能)
2. 之后构造一个pipeline的链表, 直到终止操作
3. 遇到了终止操作, 我们根据pipeline构造一个将所有pipeline的将所有中间操作都包装好的sink
4. 将sourceSpliterator中的所有元素使用sink遍历, 完成元素的中间操作的处理

可以说例子中的十分简单, 还有更加复杂的形式, 比如:
有状态中间操作 + 并行流 + Reduce终止操作, 这是我们下次文章将要解析的东西了.
