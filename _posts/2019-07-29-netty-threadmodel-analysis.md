---
layout: post
title:  "Netty线程模型分析"
date:   2019-08-03 21:50:00 +0700
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

## Reactor模式简介
![avatar](/static/img/netty-threadmodel-reactor.png)
下面解释一下这张图片里几个组件的作用.
1. Handle: 可以是任意的资源, Selector在这些资源上等待事件发生. 例如一个电典型的客户端/服务端程序, 一个服务端保持着多个客户端的连接,
这种连接即Handle, Selector在这些连接上等待读写等事件的发生.

2. Selecotr: 经典的Reacotr模式中, 它叫做`Synchronous Event Demultiplexer`(同步事件分离器). 它在一个handle_set上等待事件的发生.

3. Initiation Dispatcher:  定义了一系列管理Event Handler的接口, 包括注册, 注销Event Handler.一般来说, Selector在返回后会通知Initiation Dispatcher
,然后Initiation Dispatcher回调应用指定的Handler.

4. Event Handler: 定义了用于事件发生时的回调方法, 来处理这些事件.

## Reactor模式流程
在此我们将以最经典的Reactor模型来阐述Reactor模式的工作流程.
![avatar](/static/img/netty-threadmodel-subreactor.png)

1. MainReactor: 对应于Initiation Dispatcher, 它只有一个Handler, 即Acceptor作为注册的Event Handler.在MainReactor包含一个Event Loop, 该Loop调用select方法, , 当有新的连接请求到来, 调用Acceptor的回调方法进行处理. 
2. Acceptor: 初始化时, 注册到MainReactor中, 并且指定监听的端口(监听的端口即Handle). Acceptor处理连接事件的逻辑是: 将新的连接注册到SubReacotr上, 并且指定监听事件类型是可读/可写等, 这样该连接的后续事件都将由SubReactor进行处理.
3. SubReactor: 负责分配读写事件给注册到该SubReactor的Handler, 一个SubReactor可同时管理多个连接(Handle).
4. ThreadPool: 负责处理非IO任务.

## Netty的线程模型
![avatar](/static/img/netty-threadmodel-eventloop.png)

如上所示: 
BossEventLoopGroup相当于MainReactor, 它用来处理新的连接请求, 一旦有新的连接请求到来, BossEventLoopGroup相当于MainReactor会将这个请求分配给ServerBootstrapAcceptor, ServerBootstrapAcceptor将会把这个新的连接注册到ChildEventLoopGroup中的一个EventLoop中, 后续的读写事件都由这个EventLoop进行处理.

## 关键源码分析
### 一. `ServerBootstrapAcceptor`作用分析
`ServerBootstrap#bind(int inetPort)` <br>
    -`doBind(localAddress)`<br>
    -`initAndRegister()`<br>
    -`ServerBootstrap#init(channel)`<br>

如下是`ServerBootstrap#init(channel)`中关于Acceptor创建的代码:

```java
void init(Channel channel) throws Exception {
    // 省略部分代码
    ... 

    p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) throws Exception {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});
}
```
init方法参数的channel即即将注册到BossEventLoop中到channel, 负责监听本地端口, 此时在init方法内还在进行channel以及channel对应到pipeline到构造.此处即为channel的pipeline添加了一个`ServerBootstrapAcceptor`.

`ServerBootstrapAcceptor`是`ChannelInboundHandlerAdapter`的子类, 即它是处理入站事件的处理器.

我们依次解释一下构造`ServerBootstrapAcceptor`所需的参数:
1. ch: 即ServerSocketChannel
2. currentChildGroup: ServerBootstrap中的currentChildGroup, 在构造ServerBootstrap中的currentChildGroup时传入
3. currentChildOptions: 子Channel的Option
4. currentChildAttrs: 子Channel的AttributeKey

下面查看`ServerBootstrapAcceptor`的源码, 可以想到`ServerBootstrapAcceptor`既然是Acceptor那么它一定有将新的连接请求注册到childEventLoopGroup中的逻辑.

```java
// ServerBootstrapAcceptor#channelRead
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    // childHandler
    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```
处理逻辑如下:
1. 将msg强制转型为Channel(关于这里为什么可以正常转型我们在其他文章中详述), 然后添加子Channel对应的Handler(很可能是我们配置bootstrap时的ChannelInitializer)
2. 设置Channel的Option和Attr
3. 最关键到来了: **将子Channel注册到childGroup上**.

### 二. ChildGroup的注册Channel流程
`MultithreadEventLoopGroup#register(Channel channel)`<br>
->`SingleThreadEventLoop#register(Channel channel)`<br>
->`register(final ChannelPromise promise)`<br>
->`Unsafe#register(EventLoop eventLoop, final ChannelPromise promise)`<br>
->`register0(ChannelPromise promise)`<br>
->`AbstractNioUnsafe#doRegister()`<br>


```java
// MultithreadEventLoopGroup#register
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```
next方法以近似于一种轮训的方式获取group中的下一个EventLoop, 进行实际的注册.

```java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

public ChannelFuture register(final ChannelPromise promise) {
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

可以看到最终还是调用了Unsafe类的register方法.
```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;

        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```
上述代码的逻辑如下:
1. 首先确保Channel没有close
2. 调用doRegister
3. 调用`pipeline.invokeHandlerAddedIfNeeded()`, 因为在channel注册完成之前可能有handler加入到pipeline中, 我们需要等待注册完毕才触发`HandlerAddedI`.
4. 只有第一次注册时才触发channel active事件.

下面我我们看`doRegister`的逻辑, `doRegister`在AbstractUnsafe中是一个空实现, 我们看`AbstractNioUnsafe`的实现如下:
```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```

实际上就是调用了javaChannel的register方法.<br>
注意这里的异常处理, 这里catch了`CancelledKeyException`, 这里有可能发生该异常的原因是该channel已经注册在selector上了,但是当前channle对应的selectionKey已经取消了, 如果再次调用register方法的话会造成此异常.
所以此处再次selectNow, 因为selectionKey的cancel方法会将当前key放入selector的cancel-set中,只有再一次select才会从selector中移除.
