---
layout: post
title:  "Zookeeper请求处理全流程"
date:   2020-08-02 17:20:00 +0700
categories: [zookeeper]
---

## 学习目标
对于Zk来说,除了Leader选举和成员发现、数据同步之外,更重要的是ZK在正常状态:原子广播模式下的运作,
所以这篇文章力求搞清楚Zk是如何处理客户端请求的,我们主要以setData为例.

## 客户端网络
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

#### eventThread
在之前readResponse流程中,我们知道,对于异步请求来说,响应会调用eventThread的queuePacket方法:
[[2020-08-02-zookeeper-request-handle#SendThread的readResponse]]
实际上eventThread中有一个队列waitingEvents,里面会存放返回的响应Packet,所以queuePacket实际上将Packet入队.
然后eventThread本身也是个线程,我们看它的运行主逻辑:
```java
while (true) {
  Object event = waitingEvents.take(); // @1
  processEvent(event); // @2
  ...
	  if (p.response instanceof Create2Response) {
		cb.processResult()
	  }
}
```
1. 从waitingEvents出队
2. 根据不同请求的响应进行callback的调用.

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

## 服务端请求处理
### 网络请求处理
NettyServerCnxnFactory.java
```java
void channelActive()
  NettyServerCnxn cnxn = new NettyServerCnxn()
  ctx.channel().attr(CONNECTION_ATTRIBUTE).set(cnxn); // @1
```
关于Netty如何建立服务端我们不再赘述,让我们直接看要点.
1. 在netty的channelActive方法中代表一个连接的新建,这里每建立一个新的连接都会创建一个NettyServerCnxn,然后将它与channel关联起来,为后续的channelRead作准备

下面看channelRead:
```java
void channelRead()
  NettyServerCnxn cnxn = ctx.channel().attr(CONNECTION_ATTRIBUTE).get(); // @1
  cnxn.processMessage((ByteBuf) msg); // @2
    NettyServerCnxn.receiveMessage(buf);
	  ZooKeeperServer.processPacket(buf);
	    ZooKeeperServer.submitRequest(Request);
		  firstProcessor.processRequest(Request); // @3
```
1. 取出该channel关联的NettyServerCnxn
2. 调用NettyServerCnxn的processMessage方法处理数据
3. 最终由ZooKeeperServer中的firstProcessor进行请求的处理

那么firstProcessor是在那里配置的呢?
实际上对于Follower和Leader来说,Processor的配置是不一样的,但它们都是一个串起来的链所组成的形式.
下面就来研究一下这个Processor链的处理过程.

### RequestProcessor处理链
请求在Zk中是要经过RequestProcessor处理链的,处理链在Follower和Leader上是不同的,而且处理读请求与写请求之间也有细微的差别.
关于RequestProcessor接口定义如下:
```java
void processRequest(Request request);
void shutdown();
```
其中processRequest在RequestProcessor实现类中一般都是将请求放到实现类中的阻塞队列就OK了(除了FinalRequestProcessor和ToBeAppliedRequestProcessor).

由于Follower上的处理流程比较简单,我们首先来看写请求在Follower上的处理流程.

#### 写请求在Follower上的处理流程
Follower上的RequestProcessor处理链为:
![[Follower处理链.png]]
其中SyncRequestProcessor主要处理来自Leader的Proposal,SendAck用来处理向Leader发送Proposal的响应.

正常的提交到follower的请求走的是上面这个链路.

Request表示的是在RequestProcessor链中移动的请求,让我们来看它有哪些重要的字段:
```java
sessionId
cxid
type
TxnHeader hdr
zxid
```
我们首先来看FollowerRequestProcessor:

##### FollowerRequestProcessor
FollowerRequestProcessor的processRequest就是将请求放到队列中就返回了,我们主要看它的run方法(FollowerRequestProcessor本身是一个线程)
```java
loop:
  Request request = queuedRequests.take(); // @1
  nextProcessor.processRequest(request); // @2
  switch (request.type) {
    case OpCode.create 
	...
	  zks.getFollower().request(request); // @3
  }
```
1. 从内部的队列中取出一个请求
2. 提交给下一个处理器进行处理,下一个处理器是Commit处理器,而Commit处理器的processRequest的实现也是提交到队列中,所以这一步是异步的.
3. 如果是写请求,需要向Leader发送请求.

接下来就到了Commit处理器,Commit处理器可能是最复杂的处理器了,我会对其进行详解.

##### CommitProcessor
Zk为了保证最大化并发度但是又必须保证数据的一致性做出了以下约束:
1. 读请求可以并发执行
2. 读请求和写请求互斥
3. 任何时候只能有**一个**写请求在处理
我们可以在接下来的代码分析中看到zk是如何确保这几点的.

首先请看下面的图:
![[队列关系.png]]
可以看到在CommitProcessor中有两个队列:queuedRequests和committedRequests,两个Request容器:nextPending和currentlyCommitting.  
- queuedRequests的作用和之前的处理器一样,是为了异步执行解耦用的.
- committedRequests用来存放由Leader发送的Commit请求,这里的Commit请求可能是别的follower的请求结果也可能就是当前Follower的.  
- nextPending存放的是正在等待Leader回复Commit消息的request,在nextPending存在期间,zk是不会处理任何queuedRequests中的请求的.
- currentlyCommitting.代表正在执行FInalRequestProcssor的请求,同样当currentlyCommitting不为空的时候也不允许其他请求执行.

其流程大致如下:
1. 首先由上一个处理器发来请求放到queuedRequests中
2. CommitProcessor主线程判断当前是否可以执行请求,如果可以就从queue中拿出来请求.这里判断的标准是不能有任何正在等待提交或者等待处理已提交的请求(即判断nextPending和currentlyCommitting是否为空),如果存在,那么挂起执行线程直到其为空.
3. 根据其是否是读请求,如果是读请求,直接进入下一个处理器,如果是写请求,则需要放到nextPending中等待FollowerRequestProcessor中提交给Leader的Proposed的commit响应[[2020-08-02-zookeeper-request-handle#FollowerRequestProcessor]]
4. 由Leader返回的commit响应会放到committedRequests这个队列中,这一步会wakeup CommitProcessor的处理线程.注意这一步中的commit消息可能不是nextPending中等待的commit.
5. 由committedRequests中取出commit消息与nextPending中等待的request尝试匹配,匹配的字段是cxid和sessionId,如果都相同则匹配否则不匹配
6. 无论是否匹配,都会设置currentlyCommitting,如果匹配的话还需要清除nextPending
7. 进入最后的Final处理器处理,主要是修改内存中的dataTree,这里可能会返回响应给客户端

##### FinalRequestProcessor
FinalRequestProcessor:最后一个processor,将事务应用到DataTree中同时把响应返回给客户端.
伪代码:
```java
processRequest:
  zks.processTxn(request); // @1
    dataTree.processTxn(hdr, txn);
  if (request.isQuorum()) // @2
    zks.getZKDatabase().addCommittedProposal(request);
    zks.getZKDatabase().addCommittedProposal(request);
  rsp = new CreateResponse(); // @3
  cnxn.sendResponse(rsp)
```
1. 将request应用到dataTree中.
2. 如果是写请求则需要加入到ZkDataBase维护的一个列表,这是用来`fast follower synchronization`的,关于这一点待补充
3. 最后将请求处理的响应发给客户端.

关于Follower接收客户端请求的流程我们已经说完了,下面我们来分析Follower的第二条处理链,Sync -> SendAck.  
先来看SyncRequestProcessor:
##### SyncRequestProcessor
Follower从什么地方进入到SyncRequestProcessor的呢?
下面是进入的伪代码步骤:
```java
Follower#processPacket
  case Leader.PROPOSAL:
    FollowerZooKeeperServer.logRequest(hdr, txn);
	  syncProcessor.processRequest(request);
```
可以看到,是在Follower接收到Leader发来的Proposal的时候才会进入Sync中.

我们来看SyncRequestProcessor的主要功能是什么:  
负责把事务记录到磁盘中,在ZK里就是txnLog和snap,而且做了**group commit**的优化.
只有将request代表的事务记录到磁盘中,才会继续往下传递request.
下面是对SyncRequestProcessor源码上注释的翻译:
```
SyncRequestProcessor用于以下三个场景:
1. Leader: 将request写到磁盘,并且转发给AckRequestProcessor,AckRequestProcessor的作用是回复一个ack给自己.
2. Follower: 将request写到磁盘,并且转发给SendAckRequestProcessor,它将给Leader发送响应
3. Observer: 将已提交的请求刷至磁盘,它的下一个processor是null,也就说它不会给Leader发送响应.
```
SyncRequestProcessor接收请求:
```java
public void processRequest(Request request) {
	// request.addRQRec(">sync");
	queuedRequests.add(request);
}
```
该方法只有一行,将请求加入到queuedRequests队列中.

SyncRequestProcessor主流程伪代码:
```java
loop:
  si = queuedRequests.poll(); // @1
  if si == null // @2
    flush(toFlush)
  zks.getZKDatabase().append(si) // @3
  toFlush.add(si) // @4
  if (toFlush.size() > 1000)  // @5
    flush(toFlush);
```
1. 从queuedRequests中尝试取出一个请求
2. 如果当前没有请求,说明我们很闲😄,这时候去flush
3. 这一步实际上调用txnLog的append方法,将其写入日志中.注意这一步仅仅是write,并没有flush.另外只有写请求会append,读请求不会.
4. 将请求加入待flush列表中.
5. 如果等待flush的请求大于1000,那么进行flush,这一步实际上会将所有待flush的请求向后面的processor转发.

##### SendAckRequestProcessor
这一步是Follower第二条链的最后一个处理器,看名字就能知道它的作用是向Leader发送确认消息.
实际上代码确实这么简单:
```java
QuorumPacket qp = new QuorumPacket
  learner.writePacket
```

#### 写请求在Leader上的处理流程
在[[2020-08-02-zookeeper-request-handle#FollowerRequestProcessor]]的处理流程中我们可以看到,Follower会把请求发给Leader,而Leader会在[[2020-06-13-zookeeper-datasync#数据同步的收尾--进入原子广播阶段]]这里接收Follower的请求,我们来看submitLearnerRequest的伪代码:
```java
prepRequestProcessor.processRequest(request);
```
它只有一行代码,就是将我们的请求提交到了PrepRequestProcessor,在看PrepRequestProcessor之前,我们先把Leader的整个处理器链路给画出来.
![[Leader处理链.png]]

我们首先来看PrepRequestProcessor
##### PrepRequestProcessor
PrepRequestProcessor的processRequest方法,将请求放在内部的submittedRequests阻塞队列中.
```java
public void processRequest(Request request) {
	submittedRequests.add(request);
}
```
它同时也是一个线程,它会从submittedRequests中取出请求,进行处理:
```java
while (true) {
	Request request = submittedRequests.take();
	if (Request.requestOfDeath == request) { // "毒丸"设计模式
		break;
	}
	pRequest(request);
}
}
```
pRequest实际上根据request的type去做相应的处理,具体做什么处理呢,是将写请求加入到ZookeeperServer到outstandingChanges和outstandingChangesForPath属性中,我们举个例子来说吗这两个属性的作用:
当session关闭时,需要清理该会话下的所有临时节点,但是实际场景下,closeSession请求处理之前如果已经有下面两类请求到达服务器并且正在处理:
- 节点(包含临时与非临时)删除请求，删除的目标节点正好是上述临时节点中的一个。
- 临时节点创建，修改请求，目标节点正好是上述临时节点中的一个。  

对于第一类,由于我们closeSession请求的目的就是删除临时节点,所以到closeSession请求处理时会造成重复删除,所以我们将closeSession请求待删除列表中移除该节点.  
对于第二类,我们将临时节点创建，修改请求对应的节点路径加入到待删除列表中.

我们以create操作为例:
```java
pRequest2Txn(request.type, zks.getNextZxid(), request, create2Request, true)
  request.setHdr(new TxnHeader(request.sessionId, request.cxid, zxid, Time.currentWallTime(), type)); // @1
  pRequest2TxnCreate(type, request, record, deserialize);
    int newCversion = parentRecord.stat.getCversion()+1; // @2
	request.setTxn(new CreateTxn(path, data, listACL, createMode.isEphemeral(),
                    newCversion)); // @3 ?
	parentRecord.childCount++; // @4
	parentRecord.stat.setCversion(newCversion); 
	addChangeRecord(parentRecord); // @5
	  zks.outstandingChanges.add(c);
	  zks.outstandingChangesForPath.put(c.path, c);
	addChangeRecord(new ChangeRecord(request.getHdr().getZxid(), path, s, 0, listACL)); // @6
```
1. 设置事务头,包含sessionId、创建node时的zxid(即当前的zxid)、下一个zxid等
2. 设置父节点等cversion自增
3. 设置事务数据,包含本次创建等path和数据等
4. 父节点等child数量自增同时更新cversion
5. 将父节点改变记录加入到outstandingChanges中.
6. 将节点改变记录加入到outstandingChanges中.

##### ProposalRequestProcessor
ProposalRequestProcessor的作用主要是对写请求会给follower发送提案获得大多数follower的ack后进行提交,它的代码也是直观的:
```java
nextProcessor.processRequest(request);
if (request.getHdr() != null) {
	// We need to sync and get consensus on any transactions
	try {
		zks.getLeader().propose(request);
		  QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data);
		  outstandingProposals.put(lastProposed, p);
		  sendPacket(pp);
	} catch (XidRolloverException e) {
		throw new RequestProcessorException(e.getMessage(), e);
	}
	syncProcessor.processRequest(request);
}
```
可以看到我们会首先将请求传递给下一个processor,而下一个processor实际上就是commitprocessor(见:[[2020-08-02-zookeeper-request-handle#CommitProcessor]]).
同时在Leader的propose方法中除了发送请求之外还会将提案加入到outstandingProposals以便和后续的commit请求对应起来.
对于写请求还会把它传递给SyncProcessor进行事务的持久化(见:[[2020-08-02-zookeeper-request-handle#SyncRequestProcessor]]).

##### AckRequestProcessor
AckRequestProcessor实际上就是Leader给自己的proposal返回ack的过程,代码过于简单就不详细说明了.
```java
QuorumPeer self = leader.self;
if(self != null)
	leader.processAck(self.getId(), request.zxid, null);
```

##### 提案ACK消息的处理
上面我们说到了在ProposalRequestProcessor中我们会发送Proposal给Follower,我们来看Follower是如何处理的.
使用伪代码描述流程如下:
```java
Follower.followLeader
  while (this.isRunning()) {
	readPacket(qp);
	processPacket(qp);
	  case Leader.PROPOSAL:
	    // 反序列化...
		syncProcessor.processRequest(request);
  }
```
我们可以看到最后是由syncProcessor进行处理的,之前我们已经分析过Follower的第二条处理链的执行步骤了,这里就不赘述了,总之最后第二条链进入FInal处理器会发送Ack给Leader,让我们看Leader是在哪里处理的.

```java
LearnerHandler.run
  ...
  case Leader.ACK: // @1
    leader.processAck
	  Proposal p = outstandingProposals.get(zxid); // @2
	  tryToCommit(p, zxid, followerAddr) 
	    if (p.hasAllQuorums()) // @3
		{
		  outstandingProposals.remove(zxid); // @3.1
		  toBeApplied.add(p); // @3.2
		  zk.commitProcessor.commit(p.request); // @3.3
		}
```
1. 在LeanerHandler线程中,接收到Follower的消息,判断是ACK的话进入到processAck的方法中
2. 在这里我们从outstandingProposals中尝试获取到我们在ProposalRequestProcessor中设置进去的提案[[2020-08-02-zookeeper-request-handle#ProposalRequestProcessor]]
3. 如果获得了大多数Follower的确认的才进入下面的步骤,否则直接返回
	1. 从outstandingProposals移除提案
	2. 加入提案到toBeApplied,toBeApplied代表已经提交但还没有应用到dataTree中的提案
	3. 调用commitProcessor的commit方法,会与之前设置的nextPending做匹配,详细分析见:[[2020-08-02-zookeeper-request-handle#CommitProcessor]]


## 总结
以上就是zookeeper对于客户端请求的整体解读.
我们首先看了客户端上如何发送请求给zookeeper server的,接着又看了server端的处理流程,重点分析了Follower和Leader的请求处理器链以及Follower与Leader之间的交互,可以看到ZAB协议的原子广播在这里的应用.


