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

3. 


