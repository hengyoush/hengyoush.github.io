---
layout: post
title:  "Java关键字volatile原理解析"
date:   2021-04-14 20:30:00 +0700
categories: [java]
---

## volatile是什么, 它解决了什么问题
在多核机器上, 当多线程对同一个共享变量操作时, 会出现数据不一致的问题.
volatile是Java的一个关键字, 它的作用是保证对变量读写在多线程环境下的可见性,
也就说说即使在多线程环境下, 变量的修改对其中每个线程都是可见的.  
除此之外, volatile还解决了指令重排序及有序性的问题, 在特殊场景下发生指令重排序可能造成意想不到的后果,比如
我们下面举的例子.

## volatile的实际应用场景——DCL初始化
```java
private volatile static X x;

public X getX() {
	if (x == null) {
		synchronized (this) {
			if (x == null) {
				// initialize x...
				x = new X(1);
			}
		}
	}
	return x;
}
```
接下来让我们理解一下为什么在双重锁定初始化的代码中, i一定要被volatile修饰的原因. 
其中对i的赋值编译为字节码之后实际上是这个样子:  
1. 分配空间(new指令,这里仅分配空间不涉及初始化)
2. dup(下面的invokespecial会消费掉操作栈上的引用而返回值是void,所以这里要复制一个引用给后面的putField指令使用)
3. 调用Integer的构造器完成初始化
4. 将初始化完成的对象赋值给成员变量i

如果我们不用volatile对x进行修饰, 由于存在指令重排序, 可能第四步被排序到了第三步之前执行, 这么一来, 当其他线程同时间调用该方法时就会获得一个尚未
初始化完成的x变量, 此时对x操作可能发生意想不到的情况.  
然而对volatile变量的读写会禁止相关指令的重排序, 保证执行的有序性(虽然这是以牺牲了一点性能为代价做到的), 从而解决了该问题.

## volatile的实际应用场景之二——多线程变量可见性
```java
static boolean a;
public static void main(String[] args) throws InterruptedException {
	// 启动线程，如果a为false那么就一直循环，否则打印"退出循环"
	Thread threadB = new Thread(() -> {
		while (!a) {
		}
		System.out.println("退出循环");
	});
	thread.setDaemon(true);
	thread.start();
	
	TimeUnit.SECONDS.sleep(1);
	a = true;
	TimeUnit.SECONDS.sleep(1);
}
```
在上面的代码中, 简单做了一个实验, 首先启动一个线程B, 它的作用是当静态变量为false时一直死循环, 当a为true
时跳出循环,打印“退出循环”.

而在主线程中, 等待1s然后将a设置为true.

我们期望上述线程会打印出"退出循环", 然而现实是打印不出来, 为什么呢?
因为我们在主线程对a作出的修改, 线程B看不到, 即对B不可见.
而我们给a的前面加上volatile关键字之后,我们就可以看到打印的信息了, 也就是说主线程对a作出的修改对线程B可见了, 这就是
volatile对可见性对保证.

## 原理篇

### volatile到底做了什么
如下是对上面DCL初始化的代码进行编译得到的汇编指令:
```asm
  0x0000000110424692: movabs $0x11ba3e000,%rax
  0x000000011042469c: movb   $0x0,(%rsi,%rax,1)
  0x00000001104246a0: lock addl $0x0,(%rsp)     ;*putstatic abc
                                                ; - Scratch::getX@24 (line 14)

```
注意最后的:`lock addl $0x0,(%rsp)`, 如果我们把volatile去掉, 那么编译出来的汇编指令为:
```asm
  0x000000010b9b9012: movabs $0x106f74000,%rax
  0x000000010b9b901c: movb   $0x0,(%rsi,%rax,1)  ;*putstatic abc
                                                ; - Scratch::getX@24 (line 14)
```
可见volatile的影响是增加了一个`lock addl $0x0,(%rsp)`的指令, 这条指令的作用相当于一个内存屏障,
这里的`addl $0x0,(%rsp)`是一个空操作, 而前面的`lock`的作用是将处理器的缓存写入内存, 这个写入动作也会无效化
其他处理器中的缓存, 让volatile的修改对其他处理器立即可见.这也意味着该指令之前的操作都已经完成(写到了主存或者缓存行中,总而言之能对其他cpu可见), 达到了指令之后的操作无法排序到该指令之前执行的效果, 即禁止重排序的效果.

### 内存屏障
其实在JMM定义中, 对于内存屏障是有四种分类的:
屏障类型|指令示例|说明
-|-|-
LoadLoad屏障|Load1;LoadLoad;Load2|确保在Load2加载数据之前加载Load1的数据.考虑这样的程序, Thread1执行:`a=1; b=1`, Thread2执行:`while(b==0){} assert a == 1;`, 按顺序执行的话断言应当成功,但是如果发生Load和Load之间的重排序,那么会发生a先取到的是0,断言时从缓存行中获取造成断言失败.然而当b为volatile变量时,在后面增加了LoadLoad屏障,在Load a时获取到的一定是从主存中获取, 而又因为b=1前面有StoreStore屏障,a一定写入了主存.  
LoadStore屏障|Load1;LoadStore;Store2|确保在Store2之前加载Load1的数据.考虑如下程序, Thread1:`while(b==0) {} a = 1;`, Thread2:`assert a == 0; b = 1;`, 按顺序的话,断言不会报错,因为b在a之前被赋值,由于断言在b赋值之前,所以a一定为0.但是实际发生了b=1的赋值被重排序到了断言之前,造成了`b=1;assert a == 0;`这种情况. 如果我们给a加上volatile, 由于volatile读之后会加上LoadStore屏障,因此b=1不会被提前到断言之前,保证断言一定成功.
StoreStore屏障|Store1;StoreStore;Store2|确保在Store2之前将Store1的数据全部刷到主存中.考虑Thread1:`a=1;b=1;`, Thread2:`while(b==0){} assert a==1`, 假如没有屏障, 那么可能发生重排序:`b=1;a=1;`, 造成断言失败.如果给b声明volatile,那么根据StoreStore屏障,a一定先于b刷到主存,所以不会出现断言失败的情况.
StoreLoad屏障|Store1;StoreLoad;Load2|确保Store1存储数据一定先于后续的读取,这个比较好理解,即将当前工作内存中的数据全部刷到主内存中,对所有其他线程可见.


### JMM——Java内存模型
JMM——Java Memory Model, 要想了解JMM, 首先我们要知道JMM它要解决的问题是什么.

在多路处理器架构下, 每个处理器都有自己的高速缓存, 而它们又共享同一个主内存, 当多个处理器同时处理某个内存数据的任务时,
可能在每个处理器中看到的同一块主存中的数据是不一致的, 这个时候为了解决哪一个处理器中的数据是有效的问题, 我们引入了缓存协议,
如MESI等, 然而即使有了缓存协议, 仍然不能完全解决一致性的问题.这是可见性的问题.

另外一个问题和cpu处理指令时的优化有关, 为了处理器资源的有效利用, 处理器可能对输入的代码乱序执行. 这是有序性的问题.

为了屏蔽底层操作系统和硬件的差异, Java定义了自己的一套内存模型规范, 它定义了程序中变量访问的规范.  

![avatar](/static/img/Java关键字volatile原理解析-JMM示意图.png)















