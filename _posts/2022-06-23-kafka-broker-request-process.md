---
layout: post
title:  "Kafka源码学习笔记 之 Broker端是如何处理请求的"
date:   2020-07-13 15:20:00 +0700
categories: [kafka]
---

## 学习目标
学习kafka的broker端请求处理源码(源码位置:kafka/network/SocketServer.scala)

## SocketServer的初始化
伪代码:
```java
kafka.network.SocketServer#startup
createDataPlaneAcceptorsAndProcessors // 1
	for each endpoint:
		dataPlaneAcceptor = createAcceptor()
			new Acceoptor
		addDataPlaneProcessors(acceptor, processorNum) // 2
			new Processor // loop processorNum
			dataPlaneRequestChannel.addProcessor(processor) 
			acceptor.addProcessors // 3
startProcessingRequests // 4
	startDataPlaneProcessorsAndAcceptors
		acceptor.startProcessors
			for each processor.start()
```
1. 启动DataPlane的Acceptor和Processor.  
实际上对每个Kafka监听的hostport,都会创建一个Acceptor线程,该线程接收入站连接,并将其分配给Processor.  
2. 创建好Acceptor后,为其创建processorNum个processor线程(这个变量由num.network.threads配置控制),同时将其加入到dataPlaneRequestChannel中.
3. 创建好所有processors之后将其添加到Acceptor中.
4. startProcessingRequests方法启动DataPlane的所有Acceptor和其Processor,实际上调用了acceptor的startProcessors方法,最终遍历调用processor的start方法.


## 连接的Accept
Acceptor类是一个线程,让我们看看它的run方法做了什么事.
```java
serverChannel = openServerSocket(endPoint.host, endPoint.port) // 1
	serverChannel = ServerSocketChannel.open()
	serverChannel.socket.bind(socketAddress)
serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT) // 2
while(isRunning)
	ready = nioSelector.select(500)
	if (ready > 0) {
		keys = nioSelector.selectedKeys() //3
		for key in keys is acceptable
		socketChannel = accept(key)
			socketChannel = serverSocketChannel.accept()
		assignNewConnection(socketChannel, processor) // 4
			processor.accept(socketChannel)
				newConnections.offer(socketChannel)
				~ async
				processor.configureNewConnections // 5
					channel = newConnections.poll()
					selector.register(connectionId(channel.socket), channel)
						registerChannel(id, socketChannel, SelectionKey.OP_READ);
					...
	}
```
1. 这一步准确来说不是在run方法里面的,实际上是在Acceptor初始化的时候做的,不过不妨拿来一说.  
 openServerSocket实际上使用NIO打开了一个ServerSocketChannel用来接收入站连接.
 2. 同样使用NIO的register方法注册感兴趣的事件为ACCEPT.
 3. 如果发生了ACCEPT事件,那么获取key,调用serverSocketChannel的accept方法获取SocketChannel.
 4. 接下来把得到的与client的连接,分配给acceptor拥有的其中一个processor线程,实际上调用processor的accept方法将连接塞到了processor的阻塞队列里.
 5. 那么什么时候processor从队列中取出呢?  
 答案是在processor的configureNewConnections方法中,取出连接后注册READ事件,从channel中读取数据.
 
 可以看到,Kafka的这个架构是一个非常经典的Reactor模式的应用.

## Processor线程循环
processor同样是一个线程,我们来看它的一些重要属性:
```scala
private val newConnections = new ArrayBlockingQueue[SocketChannel](connectionQueueSize)
private val inflightResponses = mutable.Map[String, RequestChannel.Response]()
private val responseQueue = new LinkedBlockingDeque[RequestChannel.Response]()
private val selector = createSelector(...)
```
- newConnections是**新接收**的连接,不是processor当前处理的client的连接,这个要区分清楚.
- inflightResponses是当前返回给client端的response,这个属性用于之后的response返回之后的回调.
- responseQueue,是一个阻塞队列,I/O线程处理请求完成之后的响应会存放在这里
- selector:对nio的selector的封装.

下面我们来看它的run方法:
```scala
while (isRunning) {
  configureNewConnections() // 1
  processNewResponses() // 2
  poll() // 3
  processCompletedReceives() // 4
  processCompletedSends() // 5
  processDisconnected() // 6
  closeExcessConnections() // 7
}
```
1. 处理新的客户端连接,实际上去注册READ事件.
2. 从responseQueue中取出response,如果是SendResponse的话,调用selector的send方法,将responseSend返回给client.
3. 底层调用selector的poll方法,针对不同的SelectionKey做不同处理.  
	- 对read事件,读取channel中的请求数据将其放入到selecor的completedReceives属性中,将channel置为mute状态.(参见:[[2022-06-23-kafka-broker-request-process#细节 处理client端请求的有序性是如何保证的]])
	- 对write事件,调用channel的write方法,将response写出.  
4. 处理上一步中存放在selecor的completedReceives属性中的请求,调用requestChannel的sendRequest方法,将其放入到requestChannel的请求队列中.
5. 这一步处理已经完成的发送响应,调用其回调方法,并且将client的channel置为unmute.
6. 处理连接断开的channel
7. 处理超出max.connections连接的情况,选择优先级最低的channel进行关闭.

### 细节:处理client端请求的有序性是如何保证的


## RequestHandler





















