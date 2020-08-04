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
#### 客户端连接初始化
```java
new Zookeeper() // @1
  ClientCnxnSocket getClientCnxnSocket()
    new ClientCnxnSocketNetty().connect()
	  bootstrap.connect() // @2
	    incomingBuffer = lenBuffer; // @3
		ClientCncx.primeConnection() // @4
		ZKClientPipelineFactory.initChannel() 
		  pipeline.addLast("handler", new ZKClientHandler()) // @5
	  
```
1. 整个过程是在创建客户端Zookeeper对象的时候进行的
2. 使用我们熟知的netty的bootstrap创建连接
3. 将incomingBuffer设置为lenBuffer,lenBuffer是专门用来读取rpc消息的长度字段的,
而且这个incomingBuffer会在lenBuffer和真正的数据buffer之间来回切换
4. 调用ClientCncx的primeConnection方法,创建server的session.
5. ZKClientPipelineFactory会将ZKClientHandler加入netty的handler链中

#### Packet
```java
requestHeader // xid type
request  // Record
replyHeader // xid zxid err
response // Record
callback
context
```
#### 请求流程图
![[zookeeper客户端请求.png]]

#### SendThread
##### SendThread的Loop
```java
if (!clientCnxnSocket.isConnected())
  clientCnxnSocket.connect(addr) // @1
clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn)
  Packet head = outgoingQueue.poll(waitTimeOut, TimeUnit.MILLISECONDS) // @2
  doWrite(pendingQueue, head, cnxn);
    while (true) {
	  p.requestHeader.setXid(cnxn.getXid()); // @3
	  synchronized (pendingQueue) { // @4
	    pendingQueue.add(p);
	  }
	  sendPktOnly(p); // 不flush @5
	  anyPacketsSent = true;
	  if (outgoingQueue.isEmpty()) // @6
	    break;
	}
	if (anyPacketsSent)
	  channel.flush(); // @7
```
1. 如果socket没有连接到server,先建立连接
2. 建立连接之后,从outgoingQueue中取出待发送到packet
3. 如果成功取到,那么将取到到packet到requestHeader设置xid,xid是自增的
4. 并且将packet加入到pendingQueue中,等待响应
5. 将packet发送出去,但是不flush缓冲区
6. 注意到这是个while(true)循环,所以会持续到outgoing全部发送完毕
7. 如果存在有packet发送的话,那么会将缓冲区flush.

##### SendThread的readResponse
```java
// 处理pings、AuthPacket、notification的响应... 略过,不是我们分析的重点
SendThread.readResponse() // @1
  packet = pendingQueue.remove(); // @2
  lastZxid = replyHdr.getZxid(); // @3
  SendThread.finishPacket() 
    if (p.cb == null) { // @4
	  p.notifyAll();
	} else { 
	  eventThread.queuePacket(p); // @5
	}
```
1. SendThread.readResponse处理返回的server端响应
2. 从pendingQueue中出队一个Packet(pendingQueue中存的是等待响应的Packet,见[[2020-08-02-zookeeper-request-handle#请求流程图]]),因为Zk的命令执行保证是FIFO的,所以响应也是按照顺序返回的.
3. 更新lastZxid
4. 如果是同步的情况,那么唤醒在此packet上等待的线程,见[[2020-08-02-zookeeper-request-handle#同步API--create]]
5. 异步的情况,将packet放入eventThread中.

#### 异步API--create
```java
Zookeeper.create()
  ClientCnxn.queuePacket()
    packet = new Packet()
	outgoingQueue.add(packet);
	sendThread.getClientCnxnSocket().packetAdded();
```
可以看到,实际上就是把请求加入到ongoingQueue中

#### 同步API--create
```java
Zookeeper.create()
	ClientCnxn.submitRequest() // @1
	  Packet packet = queuePacket()  // @2
	  while (!packet.finished) {
		packet.wait(); // @3
	  }
```
1. 调用ClientCnxn的submitRequest方法
2. 将packet入队到ongoingQueue中(queuePacket见![[2020-08-02-zookeeper-request-handle#异步API--create]]),由SendThread进行发送.
3. 无限等待,直到响应到达.

那么什么时候唤醒呢?
 ```java
 protected void channelRead0(ChannelHandlerContext ctx, ByteBuf buf) throws Exception {
	while (buf.isReadable()) {
		if (incomingBuffer.remaining() > buf.readableBytes()) {
			int newLimit = incomingBuffer.position()
					+ buf.readableBytes();
			incomingBuffer.limit(newLimit);
		}
		buf.readBytes(incomingBuffer); // @1
		incomingBuffer.limit(incomingBuffer.capacity());

		if (!incomingBuffer.hasRemaining()) {
			incomingBuffer.flip();
			if (incomingBuffer == lenBuffer) { // @2
				recvCount.getAndIncrement();
				readLength();
				  -> incomingBuffer = ByteBuffer.allocate(len);
			} else if (!initialized) { // @3
				readConnectResult();
				lenBuffer.clear();
				incomingBuffer = lenBuffer;
				initialized = true;
				updateLastHeard();
			} else {
				sendThread.readResponse(incomingBuffer); // @4
				lenBuffer.clear();
				incomingBuffer = lenBuffer;
				updateLastHeard();
			}
		}
	}
	wakeupCnxn();
}
 ```
1. 一开始我们要读取消息的头四位即len到incomingBuffer中,确定接下来消息的长度,所以不会进入到上面的if中.
2. 如果我们还在读取len的话,就调用readLength方法将incomingBuffer方法切换为读取消息体的buffer.
3. 如果我们还在一开始的初始化阶段(初始化的时候会发送建立session的请求[[2020-08-02-zookeeper-request-handle#客户端连接初始化|]])
4. 如果是正常的响应消息的话,会从incomingBuffer中读取响应,这里会调用sendThread的readResponse方法,见:[[2020-08-02-zookeeper-request-handle#SendThread的readResponse]],接下来清理lenBuffer,将incomingBuffer重新设置为lenBuffer.
## 总结
下面是流程图:


