---
layout: post
title:  "线上GC问题排查"
date:   2019-09-20 20:02:00 +0700
categories: [jvm]
---

## 出现线上GC问题时处理步骤：
1.	首先保护现场，使用jstack和jmap相关命令将堆栈信息导出以备分析。
2.	然后如有需要重启服务。
3.	使用MemoryAnalyzer等工具进行问题的分析。
保存堆栈信息相关命令介绍
jstat 解释：可以使用jstat命令查看指定JVM的多项统计信息，在GC问题排查中较为常用的是jstat -gcutils. 如下所示：

•	S0: 年轻代中第一个survivor（幸存区）已使用的占当前容量百分比
•	S1: 年轻代中第二个survivor（幸存区）已使用的占当前容量百分比
•	E: 年轻代中Eden（伊甸园）已使用的占当前容量百分比
•	O: old代已使用的占当前容量百分比
•	P: perm代已使用的占当前容量百分比 （Java8以后为元空间）
•	YGC: 从应用程序启动到采样时年轻代中gc次数
•	YGCT: 从应用程序启动到采样时年轻代中gc所用时间(s)
•	FGC: 从应用程序启动到采样时old代(全gc)gc次数
•	FGCT: 从应用程序启动到采样时old代(全gc)gc所用时间(s)
•	GCT: 从应用程序启动到采样时gc用的总时间(s)

jmap -histo pid 打印出指定JVM堆的对象直方图，如下所示：（也可以使用-histo:live，只统计存活的对象）
 

jmap -dump:live,format=b,file=heap.bin pid
将指定JVM进程的堆中对象dump至指定文件内，之后可以使用MemoryAnalyzer分析。
jmap -permstat 27816
打印指定JVM进程的永久代统计信息（注意Jdk8使用-clstats），打印结果如下：
 
分别是类加载器的地址，加载的类的数量，加载的类实例的大小，父类加载器，是否存活以及类型。
jstack -l pid
解释：导出Java线程栈信息，可用于排查线程阻塞和死锁等问题。
由堆空间大小不足产生的FullGC

上面介绍了相关命令，下面来简要阐述一下思路。
首先大致浏览一下jmap -histo打印的直方图，查看占用空间排名前几位的的class， 如果幸运的话在这里就可以定位产生问题的原因了，
下面是一个线上频繁产生FullGC时打印的堆直方图：
 
可以看到，Long和Date的占用空间非常多而且数量几乎一致，这是异常现象。
再往下看，有一个ResourceDetailDTO，在检查代码，发现ResourceDetailDTO里正好含有两个个Long型的
id和两个date属性，正好符合直方图中的数据（43W * 2 = 86W），由此怀疑是ResourceDetailDTO对象太多导致FullGC。
接下来通过jmap dump出来的文件，使用MemoryAnalisys分析该文件结果如下：
  
由此可见，绝大多数空间都被SelectableConcurrentHashMap的一个实例占据，通过包名可知它是ehcache缓存框架下的，接下来查看其引用的对象：
 
进一步分析发现该接口的缓存的命中率非常低，导致每次查询都去数据库查询然后回填到缓存中，导致缓存利用率极低，
而且由于业务方设置该接口缓存的值占用空间比较大，导致频繁FullGC。

解决方案：将最大缓存数量由原先的10000调整至500，并且考虑删除该缓存。
调整之后，GC情况正常。

## PermGen耗尽造成的频繁GC
报警发现PermGen周期性增长然后到达最大值接着GC回收然后周而复始。

首先查看直方图，发现并无明显异常，且不是由于堆耗尽造成的GC，而是由于永久代耗尽
造成的FUllGC，所以本次排查中直方图的作用不大。
所以使用jmap -permstat查看类加载器的情况，与正常服务相比发现明显异常（如下图），由于报警服务集成了熔断组件所以
猜测可能与其相关，所以按照类加载的思路进行分析。
 
可以看到，异常服务中发现大量SearchingClassLoader但是分析正常服务日志发现并不存在类似信息，所以问题就定位在了为何集成了熔断之后
SearchingClassLoader出现并且大量加载类然后被卸载这一异常情况中。
我们通过本地实验集成熔断并且经过一段时间的预热，接着测试调用接口进行测试，使用VisualVM监控类加载情况（类加载的数量）
发现了一行mokito的相关代码，
HttpServletRequest req = Mockito.mock(HttpServletRequest.class);
在该代码执行过后本来稳定不变的类加载数量发生增长，那么问题就应该出现在该行代码的内部。
通过查看源码调试排查发现熔断组件每次拦截调用时都会将线程上下文加载器设置为AgentClassLoader，
导致mokito内部运行时（这大概是mokito的一个bug）会不断创建SearchingClassLoader，
接着造成了Mockit不断使用新的SearchingClassLoader加载新的mock类造成永久代爆满引发FullGC。

解决方案：
熔断在拦截接口调用前将线程类加载器设置为AppClassLoader

附录：
造成永久代不断增长的mockito代码：
```java
// SearchingClassLoader.java
private static ClassLoader combine(List<ClassLoader> parentLoaders) {
    ClassLoader loader = parentLoaders.get(parentLoaders.size()-1);
    
    for (int i = parentLoaders.size()-2; i >= 0; i--) { // 集成熔断时，这里会有两个ClassLoader，造成每次都会进入循环，执行new语句
        loader = new SearchingClassLoader(parentLoaders.get(i), loader);
    }
    
    return loader;
}
// AbstractClassGenerator.java
ClassLoader loader = getClassLoader();
Map cache2 = null;
cache2 = (Map)source.cache.get(loader);
if (cache2 == null) {
    cache2 = new HashMap();
    cache2.put(NAME_KEY, new HashSet());
    source.cache.put(loader, cache2); // 放进WeakMap，所以不会OOM
}
...(省略部分代码）
if (gen == null) {
    byte[] b = strategy.generate(this);
    String className = ClassNameReader.getClassName(new ClassReader(b));
    getClassNameCache(loader).add(className);
    gen = ReflectUtils.defineClass(className, b, loader); // 加载类
}
```





