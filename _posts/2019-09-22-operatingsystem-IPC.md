---
layout: post
title:  "操作系统: 进程间通信相关概念简介"
date:   2019-09-22 20:02:00 +0700
categories: [os]
---

## 学习目标
首先了解多进程/线程间会出现的问题, 比如race condition, producer/consumer等问题, 然后学习使用
各种进程间通信的方法来避免出现这些问题.

## Race Condition
又称竞态条件, 指当存在多个进程/线程同时对某个共享数据进行读写, 并且最终的结果依赖于它们执行的顺序, 称做竞态条件.

## Critical Region
临界区.
程序中的共享数据被同时read/write的部分会造成Race Condition的称为临界区.
为了解决Race Condition, 需要同时满足以下四个条件:
1. 两个进程无法同时进入临界区.
2. 不要对cpu的执行和数量做任何假设.
3. 在临界区外的线程不会阻塞任何线程
4. 进入临界区的进程一定会在有限时间内执行完毕.

## 解决Race Condition的几种方法--基于busy waiting

### 关掉中断
通过关掉中断让单处理器系统的进程在进入临界区之后不会被中断, 执行完毕后再恢复中断.
这种方法只能对单处理器有效, 因为让中断失效的命令只能作用于当前运行该进程的处理器上, 对于其他处理器上的进程依然可以访问
共享内存.

### Lock变量
假设有一个共享变量--lock, 当进程想要进入临界区时先测试这个lock是否为0, 如果为0就把它设置为1; 如果这个lock是1就等待
它直到为0.
这种方法并不能解决RC, 同样的RC会发生, 只不过发生的变量首先在lock上了而已, 这里不再赘述.

### 严格交替(Strict Alternation)
代码如下:
```c++
// process0
while(TRUE) {
    while (turn != 0)
    critical_region();
    turn = 1;
    noncritical_region();
}

// process1
while(TRUE) {
    while (turn != 1)
    critical_region();
    turn = 0;
    noncritical_region();
}
```
假设turn一开始为0, process0可以通过while(turn != 0)的循环,进入临界区;而process1
则不能通过while循环, 当process0执行完毕临界区后将turn设置为1, process1可以进入临界区.

但是仔细分析这段代码就可以发现, process0和process1进入临界区的行为是**严格交替**的.
我们接着上面的过程, process1现在执行完毕临界区进入noncritical_region方法中, 但是process0
的执行速度比较快, 它很快又执行了一边临界区, 将turn设置为1进入非临界区代码后很快执行完毕, 此时process1
则慢吞吞的, process0因为turn被自己设置为1而无法继续进行, 必须要等process1执行完毕非临界区再次进入
临界区才可以.

由此可见, 它们是严格交替执行的, 并且process1明明没有在临界区内, 却让process0无法继续执行, 这是不符合我们上面提到的解决方案
的标准的.

同时注意到, while循环会不断执行, 这在锁的持有时间较短的情况下还不会有很大问题, 但是在锁的持有时间
很长的情况下会造成很大的cpu资源浪费.

### Peterson算法
代码如下:
```c
#define FALSE 0
#define TRUE 1
#define N 2

int turn;
int interested[N];

void enter_region(int process) { // process is 1 or 0
    int other;
    other = 1 - process;
    interested[process] = TRUE;
    turn = process;
    while(turn == process && interested[other] == TRUE) ()
}

void leave_region(int process) {
    interested[process] = FALSE;
}
```
解释一下while(turn == process && interested[other] == TRUE)的条件.
1. turn == process: 乍一看这个条件好像必须成立, 其实不然. 假设两个进程几乎同时进入enter_region方法, 假设没有这个条件,
这两个进程都将会阻塞在interested[other]==true这个条件上.
2. interested[other] == TRUE: 如果为TRUE说明至少另一个进程还在临界区中或者是在进入临界区之前, 当前线程慢了一步(哈哈..)

这种方法是可以解决RC问题的.

### Test And Lock
代码如下:
```assembly
enter_region:
    TSL REGISTER,LOCK   // 读Lock的值并且存入REGISTER中, 然后设置一个非零值给Lock
    CMP REGISTER,#0     // 比较刚刚读的值与0
    JNE enter_region    // 如果不为0那么说明lock被持有了重新来过
    RET                 // 返回, 进入临界区

leave_region:
    MOV LOCK,#0         // 简单将LOCK设置为0
    RET
```

## Sleep和Wakeup
上述的解决方法都有一个缺点, 那就是会浪费cpu时间, 接下来介绍几种基于sleep/wakeup的几种
进程间同步的方法.

首先我们抛出一个问题: producer/consumer问题, 我们使用sleep/wakeup的方案的代码如下:


 



 