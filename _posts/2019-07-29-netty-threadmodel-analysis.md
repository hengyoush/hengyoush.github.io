---
layout: post
title:  "Netty线程模型分析"
date:   2019-07-27 02:00:00 +0700
categories: [netty]
---

## 概述
本文通过Netty源码分析Netty的线程模型.

![avatar](/static/img/netty-threadmodel-eventloopclasspic.png)

我们先来概述每个类和接口的作用:

1. `EventExecutorGroup`: 负责通过`next`方法提供`EventExecutor`, 除此之外, 还提供了一系列方法用于管理`EventExecutor`
的生命周期.

2. `EventExecutor`: 一种特殊的`EventExecutorGroup`, 它的`next`方法返回它自己. 它提供了`inEventLoop`方法判断一个线程是否在`eventLoop`中.

3. `EventLoopGroup`: `EventExecutor`的子接口, 可以注册一个Channel到`EventLoopGroup`上, 并且可通过`next`方法产生`EventLoop`.

4. `OrderedEventExecutor`: 标记接口, 表示使用顺序执行处理提交的task(`EventExecutor`submit提交的task)

5. `EventLoop`: 提供parent方法用于返回所属的`EventLoopGroup`.

6. `AbstractEventExecutor`: 对ExecutorService的基本实现, 不支持定时调度功能(即:`ScheduledExecutorService`接口功能)

7. `AbstractScheduledEventExecutor`: 继承自`AbstractEventExecutor`, 增加定时调度功能, 内含一个优先级队列.

8. `SingleThreadEventExecutor`: 继承自`AbstractScheduledEventExecutor`, 内部含有一个task队列, 用于存放外部执行请求.

9. `SingleThreadEventLoop`: 继承自`SingleThreadEventExecutor`, 实现了`EventLoop`接口, 增加了注册channel的功能.

10. `NioEventLoop`: 继承自`SingleThreadEventLoop`, 实现了`SingleThreadEventExecutor`的run方法, 内含一个事件循环, 使用NIO selector执行IO任务和非IO任务.

11. `AbstractEventExecutorGroup`

12. `MultithreadEventExecutorGroup`

13. `MultithreadEventLoopGroup`

14. `NioEventLoopGroup`