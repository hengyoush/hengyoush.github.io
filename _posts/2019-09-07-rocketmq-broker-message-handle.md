---
layout: post
title:  "Rocketmq broker消息存储流程"
date:   2019-09-08 02:06:00 +0700
categories: [rocketmq]
---

## 源码阅读目标
搞清楚rocketmq如何将接收的消息根据不同的请求命令进行不同的处理, 和消息存储的流程.

## 前置准备

### Netty服务端
在Broker启动过程中,会启动一个Netty服务用来接收producer或者consumer发来的请求, 对请求进行解码, 然后根据不同的RequestCommandCode将
不同的请求转发到不同的Processor中.
具体代码如下:
```java
ServerBootstrap childHandler =
    this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
        // ...省略TCP socket配置
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline()
                    .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                    .addLast(defaultEventExecutorGroup,
                        encoder,
                        new NettyDecoder(), // 注意这里是请求解码器
                        new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                        connectionManageHandler,
                        serverHandler
                    );
            }
        });
```
如上, rocketmq注册了几个handler, 我们主要关注解码, 下面对其他的ChannelHandler简要解释:
1. encoder, 编码器,用于将返回消息进行编码,默认使用Json.
2. decoder, 解码器, 下面会对其进行解释.
3. IdleStateHandler, Netty内置的处理器, 第一个参数指明Channel多长时间没有数据读入后触发一个READER_IDLE事件, 第二个参数.
指明写操作空闲的时间触发WRITER_IDLE, 第三个参数控制读写操作均未发生的时间到达时, 触发READER_IDLE和WRITER_IDLE事件.
4. connectionManageHandler, 当连接状态发生改变时, 触发相应的事件, 并且处理IdleStateHandler触发的READER_IDLE和WRITER_IDLE事件.
5. 进行消息派发的地方, 是我们接下来重点阐述的部分.

#### 消息解码
NettyDecoder继承LengthFieldBasedFrameDecoder, LengthFieldBasedFrameDecoder根据消息内部的length field获得消息的长度.
具体不解释其工作原理, 以后可以详细分析.
下面时NettyDecoder的decode方法:
```java
public Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    ByteBuf frame = null;
    try {
        frame = (ByteBuf) super.decode(ctx, in);

        ByteBuffer byteBuffer = frame.nioBuffer();

        return RemotingCommand.decode(byteBuffer);
    } catch (Exception e) {
        RemotingUtil.closeChannel(ctx.channel());
    } finally {
        if (null != frame) {
            frame.release();
        }
    }

    return null;
}
```
为了简化, 我省略了部分校验等代码.
可以看到, 首先调用父类LengthFieldBasedFrameDecoder的decode方法, 将消息根据length取出, 然后调用RemotingCommand
的decode方法.
```java
public static RemotingCommand decode(final ByteBuffer byteBuffer) {
    int length = byteBuffer.limit();
    int oriHeaderLen = byteBuffer.getInt();
    int headerLength = getHeaderLength(oriHeaderLen);

    byte[] headerData = new byte[headerLength];
    byteBuffer.get(headerData);

    RemotingCommand cmd = headerDecode(headerData, getProtocolType(oriHeaderLen));

    int bodyLength = length - 4 - headerLength;
    byte[] bodyData = null;
    if (bodyLength > 0) {
        bodyData = new byte[bodyLength];
        byteBuffer.get(bodyData);
    }
    cmd.body = bodyData;

    return cmd;
}
```
简要说明一下该方法的流程:
1. 解码header.
2. 将body简单从ByteBuffer取出放在字节数组中. (这里netty如果使用的是直接内存的话, 那么会发生一次copy?)

##### 消息分发
接下来进入消息分发的步骤.
在此之前, 我们先了解一下broker在启动的时候注册处理器的相关逻辑.
在BrokerController中的registerProcessor方法里注册了大量相关处理器:
```java
this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
this.remotingServer.registerProcessor(RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);
this.remotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);
```
可以看到同时注册的还有线程池, 这样做可以保证各种业务不会互相影响.

接下来我们分析消息派发的逻辑:
NettyRemotingAbstract#processMessageReceived
```java
public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) {
    final RemotingCommand cmd = msg;
    if (cmd != null) {
        switch (cmd.getType()) {
            case REQUEST_COMMAND:
                processRequestCommand(ctx, cmd);
                break;
            case RESPONSE_COMMAND:
                processResponseCommand(ctx, cmd);
                break;
            default:
                break;
        }
    }
}
```
根据消息是请求还是响应分别进入不同逻辑中, 我们关注的是请求, 所以进入processRequestCommand方法中.
processRequestCommand的核心代码如下:
```java
final Pair<NettyRequestProcessor, ExecutorService> pair = this.processorTable.get(cmd.getCode());
Runnable run = new Runnable() {
            @Override
            public void run() {
                final RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);
                ctx.writeAndFlush(response);
        };
final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
pair.getObject2().submit(requestTask);
```
很简单, 就是根据命令code从先前注册的code -> processor, executor 映射中找到对应的processor,
然后在对应的线程池中使用找出来的processor进行消息的处理.

## CommitLog消息存储
有了以上的基础, 我们现在开始真正的分析.

### 消息处理
在SendMessageProcessor中的sendMessage方法, 其核心代码如下:
```java
private RemotingCommand sendMessage(final ChannelHandlerContext ctx, // 用于获取channel相关信息
                                        final RemotingCommand request, // 消息请求, 主要使用里面的body
                                        final SendMessageContext sendMessageContext, // 开启消息跟踪时的消息上下文信息
                                        final SendMessageRequestHeader requestHeader) { // 用于获取消息的相关信息,比如topic等
    // 检查topic是否可写, 如果topic可自动且topic不存在的情况下则创建topic, 否则返回TOPIC_NOT_EXIST异常
    // 进行queueId的范围校验等
    super.msgCheck(ctx, requestHeader, response);
    // MessageExtBrokerInner是消息在broker内部的表现形式, 其中不仅包含body也包含topic ,queueId等信息
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(requestHeader.getTopic());
    msgInner.setQueueId(queueIdInt);
    msgInner.setBody(body);
    // 消费端消费失败时, 会将重试消息发到所属consumerGroup的retry topic
    // 这里handleRetryAndDLQ就是用来控制重试次数的, 如果重试次数超过最大重试次数时,
    // 会进入DLQ死信队列
    if (!handleRetryAndDLQ(requestHeader, response, request, msgInner, topicConfig)) {
        return response;
    }
    // 将消息存储到MessageStore中
    putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
    // 对存储结果进行处理 TODO
    return handlePutMessageResult(putMessageResult, response, request, msgInner, responseHeader, sendMessageContext, ctx, queueIdInt);
}
```

### CommitLog的putMessage方法解析
MessageStore的putMessage就是存储消息的核心代码了, MessageStore在rocketmq内的实现DefaultMessageStore采用一个CommitLog
的结构来存储消息, 所有topic的消息都存放在一个CommitLog里.

CommitLog以文件形式进行存储, 其中含有一个文件队列: MappedFileQueue, 消息写入是顺序写入.
Commitlog文件存储目录为${ROCKET_HOME}/store/commitlog目录，每一个文件默认1G，一个文件写满后再创建另外一个，
以该文件中第一个偏移量为文件名，偏移量小于20位用0补齐。

在写入CommitLog的时候会加锁, 所以写入CommitLog的过程是单线程的.
下面的代码在CommitLog类中putMessage方法中.
```java
long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
this.beginTimeInLock = beginLockTimestamp;

// Here settings are stored timestamp, in order to ensure an orderly
// global
msg.setStoreTimestamp(beginLockTimestamp);

if (null == mappedFile || mappedFile.isFull()) {
    mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
}
if (null == mappedFile) {
    log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
    beginTimeInLock = 0;
    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
}
```
要进行顺序写消息的话就必须获取当前最后的文件来进行append, 所以上面的代码首先尝试获取最后一个mappedFile, 如果为空或者已满, 那么会使用getLastMappedFile(0)方法来新建一个mappedFile.
具体如何新建mappedFile在稍后进行解释.

接下来的逻辑就是否直接了, 只不过有对mappedFile空间不够的处理
```java
result = mappedFile.appendMessage(msg, this.appendMessageCallback);
switch (result.getStatus()) {
    case PUT_OK:
        break;
    case END_OF_FILE:
        unlockMappedFile = mappedFile;
        // Create a new file, re-write the message
        mappedFile = this.mappedFileQueue.getLastMappedFile(0);
        if (null == mappedFile) {
            // XXX: warn and notify me
            log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
        }
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        break;
        ...
}
```
根据mappedFile的appendMessage方法的结果进行处理, 如果是EOF, 那么意味着需要新建一个mappedFile
来进行append.

值得注意的是在EOF的处理中有这个变量:unlockMappedFile, 它的作用是稍后会将它从内存中解除锁定:
```java
if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
    this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
}
// MappedFile.java#munlock
public void munlock() {
    final long beginTime = System.currentTimeMillis();
    final long address = ((DirectBuffer) (this.mappedByteBuffer)).address();
    Pointer pointer = new Pointer(address);
    int ret = LibC.INSTANCE.munlock(pointer, new NativeLong(this.fileSize));
    log.info("munlock {} {} {} ret = {} time consuming = {}", address, this.fileName, this.fileSize, ret, System.currentTimeMillis() - beginTime);
}
```

注意此时仅仅是将内容写到ByteBuffer中, 还没有flush到磁盘中, 稍后会解释这一异步操作.
`handleDiskFlush(result, putMessageResult, msg);`
接下来Master写入成功, 那么还有写入从broker中:
`handleHA(result, putMessageResult, msg);`
关于HA同步我们有机会再介绍.

以上到工作做完之后接下来就是返回一个putMessageResult了:
```java
PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);
return putMessageResult;
```
我们总结一下上述流程:
1. 写入之前先确保mappedFile存在
2, 写入mappedFile, 同时对EOF情况做处理
3. 如果是同步刷盘模式下需要进行flush
4. 如果是同步复制的模式的话要进行HA同步操作
5. 返回PUT_OK结果

在CommitLog的存储就是以上这些, 在前面遗留了几个比较重要问题:
1. MappedFile的创建
2. MappedFile写入消息和刷盘的详细实现

### MappedFile的创建
我们先来看上面的一个问题MappedFile的创建.
代码在MappedFileQueue.java的getLastMappedFile方法
```java
public MappedFile getLastMappedFile(final long startOffset, boolean needCreate) {
    long createOffset = -1;
    MappedFile mappedFileLast = getLastMappedFile();

    if (mappedFileLast == null) {
        createOffset = startOffset - (startOffset % this.mappedFileSize);
    }

    if (mappedFileLast != null && mappedFileLast.isFull()) {
        createOffset = mappedFileLast.getFileFromOffset() + this.mappedFileSize;
    }

    if (createOffset != -1 && needCreate) {
        String nextFilePath = this.storePath + File.separator + UtilAll.offset2FileName(createOffset);
        String nextNextFilePath = this.storePath + File.separator
            + UtilAll.offset2FileName(createOffset + this.mappedFileSize);
        MappedFile mappedFile = null;

        if (this.allocateMappedFileService != null) {
            mappedFile = this.allocateMappedFileService.putRequestAndReturnMappedFile(nextFilePath,
                nextNextFilePath, this.mappedFileSize);
        } else {
            try {
                mappedFile = new MappedFile(nextFilePath, this.mappedFileSize);
            } catch (IOException e) {
                log.error("create mappedFile exception", e);
            }
        }

        if (mappedFile != null) {
            if (this.mappedFiles.isEmpty()) {
                mappedFile.setFirstCreateInQueue(true);
            }
            this.mappedFiles.add(mappedFile);
        }

        return mappedFile;
    }

    return mappedFileLast;
}
```
在上一节的分析中, 我们知道是这样调用的:`getLastMappedFile(0, true)`.
1. 文件名的确定
传入startOffset的作用是确定mappedFile的名字, 名字是通过createOffset控制的.
首先如果当前没有文件, 那么通过startOffset与mappedFile的文件大小进行对齐得到createOffset即可.
如果mappedFile不为空且没满, 那么`createOffset =- 1`不变, 直接返回.
如果已满那么根据这个满的mappedFile得到下一个文件的起始offset.

2. 文件的真正创建
判断allocateMappedFileService是否为空.(按当前的实现, 是不可能为空的, 所以下面仅分析不为空的情况)
接着调用allocateMappedFileService的putRequestAndReturnMappedFile方法创建一个mappedFile.

核心代码如下所示:
AllocateMappedFileService.java#mmapOperation方法
```java
MappedFile mappedFile;
if (messageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
    try {
        mappedFile = ServiceLoader.load(MappedFile.class).iterator().next();
        mappedFile.init(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
    } catch (RuntimeException e) {
        log.warn("Use default implementation.");
        mappedFile = new MappedFile(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
    }
} else {
    mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());
}
```
这里解释一下TransientStorePoolEnable选项的作用, 如果开启了TransientStorePoolEnable选项的话, 
broker会创建一个堆外buffer池(默认5个buffer, 每个1G), 同时将它们锁定在内存中(使用jna).
这样在消息写入commitLog时会有一些不同: 先将消息写入到这个堆外内存中, 然后再commit到FileChannel的
OS Page Cache中, 最后再flush到磁盘里.不开启这个选项的话是没有commit的步骤直接写入到FileChannel映射的
OS Page Cache中.

注意, TransientStorePoolEnable仅在开启异步刷盘的时候才有效, 因为如果是同步的话那么就没必要使用堆外内存了,
因为每次处理消息都要直接flush到磁盘上.

值得一提的是, 在AllocateMappedFileService新建mappedFile到时候会根据是否开启warmMapedFileEnable
这个选项来决定是否预先向FileChannel映射到ByteBuffer中写入数据来保证不发生缺页中断.(注意这里处理的是FileChannel的映射的
逻辑地址而不是上面TransientStorePool的堆外内存的地址, TransientStorePool在创建池内的buffer的时候就已经处理过了)

### 消息append
入口代码在MappedFile.java的appendMessagesInner方法中:
```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    int currentPos = this.wrotePosition.get();

    if (currentPos < this.fileSize) {
        // 首先我们要确定消息首先写入哪里? 如果我们开启了TransientStorePoolEnable那么就需要写入到writeBuffer——这是我们之前
        // 从TransientStorePool中borrow的buffer, 否则就写入到文件映射buffer中
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        byteBuffer.position(currentPos);
        AppendMessageResult result;
        if (messageExt instanceof MessageExtBrokerInner) {
            // 这里是消息append的核心实现
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```
*参数的AppendMessageCallback会传入CommitLog的内部类DefautlAppendMessageCallback*
接下来我们分析DefautlAppendMessageCallback的关于消息append的核心代码.

DefautlAppendMessageCallback关于消息写入的方法是doAppend方法, 由于其代码比较多, 我在这里先简要
概括一下它的大概流程:(其中有针对事务消息的特殊处理, 由于不是分析的主要目标就不详细说明了)

1. 确保文件空间足够, 如果不够则返回EOF
2. 将消息的相关信息和body写入到DefautlAppendMessageCallback的成员变量中的msgStoreItemMemory
3. 将msgStoreItemMemory的内容put到mappedFile的buffer中.

### 消息commit以及刷盘分析
我们回到CommitLog的putMessage方法中, 根据之前的分析我们知道在消息put到mappedFile后会进行
commit和flush到处理, 现在我们来分析一下.

在CommitLog中有两个Service是用来处理commit和flush的:
```java
private final FlushCommitLogService flushCommitLogService;
private final FlushCommitLogService commitLogService = new CommitRealTimeService();
```
*flushCommitLogService用来处理flush, commitLogService用来处理commit.对于commit和flush的区别上面已经说明了.*

其中flushCommitLogService根据刷盘模式有不同的实现:
```java
if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
    this.flushCommitLogService = new GroupCommitService();
} else {
    this.flushCommitLogService = new FlushRealTimeService();
}
```

接着我们看一下handleDiskFlush方法的具体实现:
```java
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    // 同步刷盘, flushCommitLogService使用GroupCommitService实现, 调用GroupCommitRequest的waitForFlush方法
    // 同步等待刷盘
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        if (messageExt.isWaitStoreMsgOK()) {
            GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
            service.putRequest(request);
            boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
            ...
        } else {
            service.wakeup();
        }
    }
    // 异步刷盘, 如果开启了TransientStorePoolEnable的话会先commit
    else {
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            flushCommitLogService.wakeup();
        } else {
            commitLogService.wakeup();
        }
    }
}
```

#### 消息commit的具体分析
代码在CommitRealTimeService的run方法中.
核心代码如下:
```java
boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
```

猜想如下:
调用mappedFileQueue的commit方法, 应是通过getLastMappedFile获取当前的mappedFile,
再调用mappedFile的commit方法, mappedFile的commit方法的逻辑应该是读取writeBuffer的flushPos到
writePos之间的数据写入到mappedByteBuffer中.

如下是appedFileQueue的commit方法
```java
public boolean commit(final int commitLeastPages) {
    boolean result = true;
    MappedFile mappedFile = this.findMappedFileByOffset(this.committedWhere, this.committedWhere == 0);
    if (mappedFile != null) {
        int offset = mappedFile.commit(commitLeastPages);
        long where = mappedFile.getFileFromOffset() + offset;
        result = where == this.committedWhere;
        this.committedWhere = where;
    }

    return result;
}
```
*与我到猜想大致相同, 但是不是获取最后一个file, 而是通过记录的上一次commitOffset获取的file.*

接下来我们看mappedFile的commit方法:
```java
public int commit(final int commitLeastPages) {
    if (writeBuffer == null) {
        // 如果writeBuffer为空说明没有开启TransientPool, 不需要commit, 直接返回即可
        return this.wrotePosition.get();
    }
    // isAbleToCommit方法用来判断当前脏数据是否满足commitLeastPages的需求
    if (this.isAbleToCommit(commitLeastPages)) {
        if (this.hold()) {
            commit0(commitLeastPages);
            this.release();
        } else {
            log.warn("in commit, hold failed, commit offset = " + this.committedPosition.get());
        }
    }

    // 当前writeBuffer的所有脏数据都写到FileChannel上了, 那么需要新建mappedFile, 重新向TransientPool申请一个
    // writeBuffer, 此时将writeBuffer归还给pool
    if (writeBuffer != null && this.transientStorePool != null && this.fileSize == this.committedPosition.get()) {
        this.transientStorePool.returnBuffer(writeBuffer);
        this.writeBuffer = null;
    }

    return this.committedPosition.get();
}

protected void commit0(final int commitLeastPages) {
    int writePos = this.wrotePosition.get();
    int lastCommittedPosition = this.committedPosition.get();

    if (writePos - this.committedPosition.get() > 0) {
        try {
            ByteBuffer byteBuffer = writeBuffer.slice();
            // 此处确实如我所料, 将上次commit的pos到wrotePos之间的数据写入到FileChannel中
            byteBuffer.position(lastCommittedPosition);
            byteBuffer.limit(writePos);
            // 相比写入mappedByteBuffer, 直接写入FileChannel更直接一些
            // 虽然都需要flush到磁盘
            // 应该是对性能有所提升吧
            this.fileChannel.position(lastCommittedPosition);
            this.fileChannel.write(byteBuffer);
            this.committedPosition.set(writePos);
        } catch (Throwable e) {
            log.error("Error occurred when commit data to FileChannel.", e);
        }
    }
}
```
整体的逻辑与我之前预期的差不多.

#### 消息刷盘的具体分析
我们直接看mappedFile的flush方法:
```java
public int flush(final int flushLeastPages) {
    // 判断当前数据是否符合flushLeastPages的要求
    if (this.isAbleToFlush(flushLeastPages)) {
        if (this.hold()) {
            int value = getReadPosition();

            try {
                // 使用TransientPool是直接写入到FileChannel中的, 所以使用fileChannel的force方法
                if (writeBuffer != null || this.fileChannel.position() != 0) {
                    this.fileChannel.force(false);
                } else {
                    this.mappedByteBuffer.force();
                }
            } catch (Throwable e) {
                log.error("Error occurred when force data to disk.", e);
            }
            // 刷新flushedPos, 这里的value指的是当前commitPos, 
            // 如果使用TransientPool, 那么value就是commitPos
            // 否则就是当前的wrotePos
            this.flushedPosition.set(value);
            this.release();
        } else {
            log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
            this.flushedPosition.set(getReadPosition());
        }
    }
    return this.getFlushedPosition();
}
```

## ConsumeQueue
RocketMQ基于主题订阅模式实现消息消费，消费者关心的是一个主题下的所有消息，但由于同一主题的消息不连续地存储在commitlog文件中，试想一下如果消息消费者直接从消息存储文件（commitlog）中去遍历查找订阅主题下的消息，效率将极其低下，RocketMQ为了适应消息消费的检索需求，设计了消息消费队列文件（Consumequeue），该文件可以看成是Commitlog关于消息消费的“索引”文件，consumequeue的第一级目录为消息主题，第二级目录为主题的消息队列.

consume queue 条目结构如下所示:
```
commitlog offset   +    size    +    tag hashcode
-----8bytes-----    ---4bytes--- ------8bytes-----
```
关于consume queue的写入核心代码如下所示, 首先是ReputMessageService,:
```java
SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
for (int readSize = 0; readSize < result.getSize() && doNext; ) {
// 构造DispatchRequest对象
 DispatchRequest dispatchRequest =
    DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
int size = dispatchRequest.getBufferSize() == -1 ? dispatchRequest.getMsgSize() : dispatchRequest.getBufferSize();

if (dispatchRequest.isSuccess()) {
    if (size > 0) {
        // 真正进行分发的地方
        DefaultMessageStore.this.doDispatch(dispatchRequest);
        ...
        this.reputFromOffset += size;
        readSize += size;
        ...
    } else if (size == 0) {
        // 还记得当commitlog的mappedFile最后空间不够了怎么办吗? 👆(在之前有相关解释)
        this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
        readSize = result.getSize();
    }
}
}
```
该service thread每隔1ms会从当前reputFromOffset开始从commitlog中将消息信息取出放入
consumequeue和indexFile中.
它会构造一个DispatchRequest对象, 里面包含消息的相关信息:
```java
private final String topic;
private final int queueId;
private final long commitLogOffset;
private int msgSize;
private final long tagsCode;
private final long storeTimestamp;
private final long consumeQueueOffset;
private final String keys;
private final boolean success;
private final String uniqKey;

private final int sysFlag;
private final long preparedTransactionOffset;
private final Map<String, String> propertiesMap;
private byte[] bitMap;
```

接下来我们看DefaultMessageStore的doDispatch方法
```java
public void doDispatch(DispatchRequest req) {
    for (CommitLogDispatcher dispatcher : this.dispatcherList) {
        dispatcher.dispatch(req);
    }
}
```
在DefaultMessageStore的构造方法中初始化了两个分发器,分别是consumequeue的和indexFile的, 我们先看consume queue的:
```java
this.dispatcherList = new LinkedList<>();
this.dispatcherList.addLast(new CommitLogDispatcherBuildConsumeQueue());
this.dispatcherList.addLast(new CommitLogDispatcherBuildIndex());
```

我们先猜想一下实现逻辑:
1. 首先从consume queue的mappedFileQueue中获得最后一个file
2. 将DispatchRequest对象中的物理位移, size和tagscode取出, 放入到file中, 至于是否刷盘尚不清楚.

好的现在我们来看对应的代码吧:(DefaultMessageStore.java#putMessagePositionInfo)
```java
public void putMessagePositionInfo(DispatchRequest dispatchRequest) {
    ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(), dispatchRequest.getQueueId());
    cq.putMessagePositionInfoWrapper(dispatchRequest);
}
```
如上所示, 先找到对应的consume queue对象, 然后将dispatchRequest放入其中.
findConsumeQueue逻辑非常简单, 就是根据topic和queueid获取相应的记录即可.
下面我们看真正进行写入的逻辑:(DefaultMessageStore的putMessagePositionInfo方法)
```java
this.byteBufferIndex.flip();
this.byteBufferIndex.limit(CQ_STORE_UNIT_SIZE);
this.byteBufferIndex.putLong(offset);
this.byteBufferIndex.putInt(size);
this.byteBufferIndex.putLong(tagsCode);

MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile(expectLogicOffset);
this.maxPhysicOffset = offset + size;
return mappedFile.appendMessage(this.byteBufferIndex.array());
```
可以看到,我们猜想的逻辑与代码中几乎一致, 当然我在这里省略了一些代码, 但大致的逻辑是这样的.