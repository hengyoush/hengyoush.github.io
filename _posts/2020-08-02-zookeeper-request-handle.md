---
layout: post
title:  "Zookeeper请求处理全流程"
date:   2020-08-02 17:20:00 +0700
categories: [zookeeper]
---

## 学习目标
对于Zk来说,除了Leader选举和成员发现、数据同步之外,更重要的是ZK在正常状态:原子广播模式下的运作,
所以这篇文章力求搞清楚Zk是如何处理客户端请求的,我们主要以setData为例.

## temp
客户端连接初始化
```java
new Zookeeper() // @1
  ClientCnxnSocket getClientCnxnSocket()
    new ClientCnxnSocketNetty().connect()
	  bootstrap.connect() // @2
	    incomingBuffer = lenBuffer; // @3
		ZKClientPipelineFactory.initChannel() 
		  pipeline.addLast("handler", new ZKClientHandler()) // @4
	  
```
1. 整个过程是在创建客户端Zookeeper对象的时候进行的
2. 使用我们熟知的netty的bootstrap创建连接
3. 将incomingBuffer设置为lenBuffer,lenBuffer是专门用来读取rpc消息的长度字段的,
而且这个incomingBuffer会在lenBuffer和真正的数据buffer之间来回切换
4. ZKClientPipelineFactory会将ZKClientHandler加入netty的handler链中

#### Packet
```java
requestHeader // xid type
request  // Record
replyHeader // xid zxid err
response // Record
callback
context
```
![[zookeeper客户端请求.png]]
## 总结
下面是流程图:


