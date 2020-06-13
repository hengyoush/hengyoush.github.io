---
layout: post
title:  "Rocketmq 消息消费过程源码分析"
date:   2020-06-09 10:02:00 +0700
categories: [rocketmq]
---

## 学习目标
1. 搞清楚rocketmq是如何进行消息消费的, 了解消息消费过程中业务端与broker的行为.
2. 对消息消费这一条流程中的一些要点进行重点分析,比如队列负载、顺序消息消费、延迟消费等.

## 基于推模式下的消息消费过程分析
既然有基于推模式的消息拉取也有基于拉模式下的拉取, 拉模式比较易于理解, 即业务方主动与broker交互, 发送拉取请求获取消息.

而推模式则是broker端主动推送消息到客户端, 推模式一般比拉模式在实时性上略胜一筹.但是拉模式一般需要自己去维护消费进度, 所以
在编码的复杂程度上推模式也是更加简单一些.


因为RokcetMQ对推模式的实现实际上是基于拉模式的, 只要理解了推模式的实现, 那么就自动理解了拉模式, 所以我们着重讲解基于推模式的实现.

下面我们先看一下基于推模式和拉模式的使用例子, 再结合RocketMQ的源码进行分析.

### 使用示例
首先启动Producer:
```java
    public static void main(String[] args) throws MQClientException, InterruptedException {

        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
        producer.start();

        for (int i = 0; i < 128; i++)
            try {
                {
                    Message msg = new Message("TopicTest",
                        "TagA",
                        "OrderID188",
                        "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                    SendResult sendResult = producer.send(msg);
                    System.out.printf("%s%n", sendResult);
                }

            } catch (Exception e) {
                e.printStackTrace();
            }

        producer.shutdown();
    }
```

然后我们看基于拉模式的消费者示例:
```java
    // 拉模式下我们维护了一个MessageQueue和其消费进度的map
    private static final Map<MessageQueue, Long> OFFSE_TABLE = new HashMap<MessageQueue, Long>();

    public static void main(String[] args) throws MQClientException {
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("please_rename_unique_group_name_5");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.start(); // @1

        Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("topicTest"); // @2
        for (MessageQueue mq : mqs) {
            System.out.printf("Consume from the queue: %s%n", mq);
            SINGLE_MQ:
            while (true) {
                try {
                    PullResult pullResult =
                        consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32); // @3
                    System.out.printf("%s%n", pullResult);
                    putMessageQueueOffset(mq, pullResult.getNextBeginOffset()); // @4
                    switch (pullResult.getPullStatus()) {
                        case FOUND:
                            break;
                        case NO_MATCHED_MSG:
                            break;
                        case NO_NEW_MSG:
                            break SINGLE_MQ;
                        case OFFSET_ILLEGAL:
                            break;
                        default:
                            break;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }

        consumer.shutdown();
    }

    private static long getMessageQueueOffset(MessageQueue mq) {
        Long offset = OFFSE_TABLE.get(mq);
        if (offset != null)
            return offset;

        return 0;
    }

    private static void putMessageQueueOffset(MessageQueue mq, long offset) {
        OFFSE_TABLE.put(mq, offset);
    }
```
1: 这里我们设置了nameserver的地址和消费组名称之后启动基于拉模式的RocketMQ 消费端实例.  
2: 拉取topicTest主题下所有的MessageQueue  
3: 遍历MessageQueue, 调用pullBlockIfNotFound方法获取每个MessageQueue的消息, 这里第三个参数是当前要从该MessageQueue的什么位置上  
拉取消息, 这里我们调用了getMessageQueueOffset方法获取我们自己维护的消费进度.  
4: 将消息队列以及消费进度存放到自己维护的map中.


基于推模式下的消费者示例:
```java
    public static void main(String[] args) throws InterruptedException, MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");// @1
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);// @2
        consumer.subscribe("TopicTest", "*");// @3
        consumer.registerMessageListener(new MessageListenerConcurrently() { // @4
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();// @5

        System.out.printf("Consumer Started.%n");
    }
```
1: 设置消费组名称, 这里我们使用DefaultMQPushConsumer.  
2: 设置消费进度策略, 这里我们选用CONSUME_FROM_FIRST_OFFSET, 代表如果获取不到消费进度那么从该队列的最开始消费.(还有其他策略, 关于这点我们下面会详细讨论)  
3: 设置订阅的topic.  
4: 注册MessageListenerConcurrently, 这里是一个回调逻辑, 当获取到消息后会直接回调consumeMessage方法.  
5: 启动消费者实例

可以看到, 基于推模式下的消费代码比拉模式更加精简, 而且我们不需要去关心消费进度的维护, 一切交给RocketMQ去管理.

下面我们以DefaultMQPushConsumer的start方法开始, 探索消息消费的实现.

### 消费者启动流程
```java
this.copySubscription(); // @1
this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook); // @2

this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup()); // @3
this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

this.pullAPIWrapper = new PullAPIWrapper(
                    mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode()); // @4
switch (this.defaultMQPushConsumer.getMessageModel()) { // @5
    case BROADCASTING:
        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
        break;
    case CLUSTERING:
        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
        break;
    default:
        break;
}
if (this.getMessageListenerInner() instanceof MessageListenerOrderly) { // @6
    this.consumeOrderly = true;
    this.consumeMessageService =
        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
} else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
    this.consumeOrderly = false;
    this.consumeMessageService =
        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
}
this.consumeMessageService.start();
mQClientFactory.start(); // @7
```
1: 将订阅的topic拷贝给rebalanceImpl  
2: 获取或创建mQClientFactory, mQClientFactory代表一个RocketMQ的客户端实例, 封装了与broker进行交互的逻辑.  
3: 设置rebalanceImpl的一些属性, 包括消费组、消费模式、队列分配算法、mQClientFactory.  
4: 初始化PullAPIWrapper, PullAPIWrapper封装了拉取消息相关的操作.  
5: 工具消费模型确定offset消费进度的存储方式, 广播模式因为各个消费者自由消费, 不需要共享消费进度所以直接在本地维护消费进度,  
但是集群模式下, 由于各个消费者需要共享消费进度所以在broker中进行消费进度的维护.  
6: 根据是否是顺序消费, 实例化不同的consumeMessageService, 并且启动.  
7: 启动mQClientFactory, 这里会启动一些重要的线程, 比如队列负载以及消息消费线程等.  

将订阅的topic拷贝给rebalanceImpl, copySubscription方法分析, 这里对retryTopic重试主题也做了处理.
```java
private void copySubscription() throws MQClientException {
    Map<String, String> sub = this.defaultMQPushConsumer.getSubscription();
    for (Map.Entry<String, String> entry : sub.entrySet()) { // @1
        String topic = entry.getKey();
        SubscriptionData subscriptionData;
        this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
    }
    switch (this.defaultMQPushConsumer.getMessageModel()) {
    case CLUSTERING: // @2
        String retryTopic;
        SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(consumerGroup, retryTopic);
        this.rebalanceImpl.getSubscriptionInner().put(retryTopic, subscriptionData);
    }   
}
```
1: 遍历通过consumer.subscribe方法订阅的topic, 将其放入rebalanceImpl中.  
2: 同时对重试主题也将其放入rebalanceImpl中, 重试主题名称是"%RETRY%" + 原topic名称.  

```java
this.mQClientAPIImpl.fetchNameServerAddr();
// Start request-response channel
this.mQClientAPIImpl.start();
// Start various schedule tasks
this.startScheduledTask();
// Start pull service
this.pullMessageService.start(); // @1
// Start rebalance service
this.rebalanceService.start(); // @2
// Start push service
this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
this.serviceState = ServiceState.RUNNING;
```
1: 启动消息拉取线程
2: 启动队列负载线程,下面我们首先对队列负载进行分析

由此可见, rebalanceService和pullMessageService都是JVM级别的, 即一个JVM中只会存在一个.

### 队列负载
队列负载的流程伪代码如下:  
&emsp;&emsp;RebalanceService.run()  
&emsp;&emsp;&emsp;MQClientInstance.doRebalance()  
&emsp;&emsp;&emsp;&emsp;for each consumer, loop:  
 &emsp;&emsp;&emsp;&emsp;&emsp;DefaultMQPushConsumerImpl.doRebalance()  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;RebalanceImpl.rebalanceByTopic(topic, isOrder)

RebalanceService是一个线程, 默认每20秒运行一次.  
然后它直接调用MQClientInstance的doRebalance方法, 对每一个consumer实例, 调用其doRebalance
方法, 因为每一个实例它订阅的topic和消息模型不同需要分开处理.  
最终调用进入到了RebalanceImpl的rebalanceByTopic方法, 对每一个topic进行消息队列的负载, 这是理所当然的.
下面我们来看一看具体队列负载的过程.

根据消息模型的不同, 分为集群模式和广播模式, 对于广播模式, 相当于每个消费组内的消费者都负载了全部的
消息队列;而集群模式下, 需要给消费组内每一个消费者分配消息队列, 需要防止一个队列分给多个消费者的情况.  
下面我们先来看集群模式下的队列分配过程.  

```java
Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic); // @1
List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup); // @2

List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
mqAll.addAll(mqSet);
Collections.sort(mqAll);
Collections.sort(cidAll); // @3

AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;
List<MessageQueue> allocateResult = strategy.allocate( // @4
                            this.consumerGroup,
                            this.mQClientFactory.getClientId(),
                            mqAll,
                            cidAll);
updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder); // @5
```
1: 根据topic获取MessageQueue集合.  
2: 根据topic和消费组获取订阅该topic的该消费组的所有消费者clientId.
3: 这一步是保持分配结果正确的重要步骤: 分别对MessageQueue和clientId列表进行排序, 这里使用了Collections的sort方法,
可以保证每个消费者排序的结果都是一样的.  
4: 接下来进行真正的分配步骤, 这里使用了策略模式AllocateMessageQueueStrategy的实现类有如下几种:
1)平均分配 2)轮询分配 3)一致性hash分配 4)根据配置固定的消息队列 5)根据部署机房名分配  
一般用的是平均分配和轮询分配.  
举个平均分配的例子:  
有3个消费者:c1,c2,c3  
8个消息队列:q1到q8  
c1分配的队列有:q1 q2 q3  
c2:q4 q5 q6  
c3:q7 q8
这样就保证了每个消费者都可以分配到不同的队列,视图是一致的.  

5:根据上一步返回的分配结果, 对ProcessQueue进行更新操作, 同时在这里会创建拉取请求pullRequest.

下面我们先讲讲ProcessQueue再继续对updateProcessQueueTableInRebalance方法的分析.

>#### ProcessQueue
> ProcessQueue是消费端上对MessageQueue对映射,是一对一的关系, ProcessQueue中存放着从Broker上的MessageQueue中拉取的待处理的消息, 每次成功处理完消息后会将消息从其中移除, 我们来看一看它的重要的成员变量:  
> ```java
> ReadWriteLock lockTreeMap;
> TreeMap<Long, MessageExt> msgTreeMap;
> volatile boolean dropped = false;
> ```
>1. lockTreeMap: ProcessQueue中每一个对msgTreeMap的操作都会加上对应的读/写锁,防止对TreeMap的并发修改.  
>2. msgTreeMap: key是消息的LogicOffset, value即消息本身.  
>使用TreeMap即表示其Key是有序的, 为什么要使用TreeMap呢?  
>使用TreeMap实际上具有了排序和Map的功能,让我们看看假如:  
1、只有排序的功能,没有Map的功能: 考虑并发消费的场景, 当一批消息消费完成时, 需要从ProcessQueue中移除, 由于不具有Map的功能, 只能遍历Queue, 时间复杂度时O(n).  
2、只要Map的功能没有排序的功能: 考虑顺序消费的场景, 需要从Map中取出offset最小的几个消息, 此时由于没有排序只能遍历Map, 时间复杂度依然是O(n).  
所以使用TreeMap是合理的.
>3. dropped: 代表这个ProcessQueue是否因为队列重新负载导致消费者不在具有该MessageQueue的消费权利, 如果设置为true, 那么会停止该消息队列的消费,同时持久化消费进度(集群模式下持久化到broker中).

updateProcessQueueTableInRebalance的伪代码如下(省略了与主流程无关的细节):
```java
for each Pair <MessageQueue,ProcessQueue> matches the topic, loop:
    if allocateResult NOT contains messageQueue
        ProcessQueue.setDropped(true) // @1
        removeUnnecessaryMessageQueue()
            OffsetStore.persist()
            OffsetStore.removeOffset()
        
for each Allocated MessageQueue, loop:
    if processQueueTable NOT containsKey mq // @2
        if isOrder 
            lock(mq)
        long offset = computePullFromWhere(mq) 
        dispatchPullRequest(new PullRequest(mq, pq, offset, ...)) 
            defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest)
```
1: 遍历当前消费者拥有的MessageQueue和ProcessQueue, 如果本次分配结果不存在的话, 那么
设置ProcessQueue的dropped为true, 停止该队列的消费, 同时持久化该队列的消费进度让其他消费者知道当前的消费进度.(这里应该会出现重复消费)  
2: 遍历每一个本次分配到的MessageQueue, 如果没有其对应的ProcessQueue, 那么会构造一个pullRequest请求立即执行拉取消息. 同时如果这里是顺序消费, 那么会将mq加锁, 这里详细的等到讲解顺序消息时再说明.  
需要注意的是这里computePullFromWhere方法计算消费进度, 这里是一个细节, 我们先略过, 到后面再细说.

接下来我们看defaultMQPushConsumerImpl.executePullRequestImmediately对拉取消息请求做了什么处理.

### 消费请求发送
defaultMQPushConsumerImpl.executePullRequestImmediately的方法调用伪代码如下:
```java
defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest)
    mQClientFactory.getPullMessageService().executePullRequestImmediately(pullRequest)
        pullRequestQueue.put(pullRequest)
```
可见这里最终将pullRequest放在了PullMessageService的pullRequestQueue中.

由于上面我们说了,PullMessageService是一个ServiceThread线程,下面我们看它的run方法.
```java
PullRequest pullRequest = this.pullRequestQueue.take();
this.pullMessage(pullRequest);
    consumer = mQClientFactory.selectConsumer(pullRequest.getConsumerGroup())
    consumer.pullMessage(pullRequest)
        pullAPIWrapper.pullKernelImpl(messageQueue,queueOffset,pullBatchSize,pullCallback,...)
```
如果没有拉取请求,会一直阻塞在pullRequestQueue的take方法上.从队列中拿到拉取请求之后,
最终调用了pullAPIWrapper的pullKernelImpl方法(中间省略了一些流控代码和顺序消息相关的代码)去进行远程调用,
从meeageQueue中可以得到要进行远程调用的broker地址,然后发送请求即可.  
这里传入了一个pullCallBack,是在请求返回时会进行回调,稍后我们会对其进行分析.

下面我们来分析broker端对拉取消息请求的处理, 我们关心的逻辑有对于push的请求,应该会有挂起请求的机制.


### Broker端对消费请求的处理
对拉取请求处理的主要逻辑在PullMessageProcessor的processRequest方法内,下面我们对其重点步骤进行分析(省略一些校验逻辑):
```java
GetMessageResult getMessageResult = messageStore.getMessage(consumerGroup, topic, queueId, queueOffset, msgNums, messageFilter);
```
从messageStore中获取消息, 根据消费组、主题、queueId可以确定ConsumeQueue文件, 根据QueueOffset和msgNums和messageFilter可以拿到我们想要的消息.

下面我们进入getMessage方法内分析一下:
```java
minOffset = consumeQueue.getMinOffsetInQueue();
maxOffset = consumeQueue.getMaxOffsetInQueue();

if (maxOffset == 0) {
    status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
    nextBeginOffset = nextOffsetCorrection(offset, 0);
} else if (offset < minOffset) {
    status = GetMessageStatus.OFFSET_TOO_SMALL;
    nextBeginOffset = nextOffsetCorrection(offset, minOffset);
} else if (offset == maxOffset) {
    status = GetMessageStatus.OFFSET_OVERFLOW_ONE;
    nextBeginOffset = nextOffsetCorrection(offset, offset);
} else if (offset > maxOffset) {
    status = GetMessageStatus.OFFSET_OVERFLOW_BADLY;
    if (0 == minOffset) {
        nextBeginOffset = nextOffsetCorrection(offset, minOffset);
    } else {
        nextBeginOffset = nextOffsetCorrection(offset, maxOffset);
    }
} 
```
在拉取消息前需要对queueOffset做一次校正, 上图中的代码就是做校正的过程.  
首先minOffset和maxOffset分别是该MessageQueue的最大逻辑位移和最小逻辑位移.  
1: 如果当前队列最大位移为0代表当前队列尚无消息, 此时校正下次拉取位移为0.  
2: 要拉取的位移小于最小位移, 那么校正位移为最小位移.  
3: 如果与maxOffset正好相等, 下次拉取位移依然是offset.  
4: 如大于最大位移,表示偏移量越界,返回status=OFFSET_OVERFLOW_BADLY,同样校对下次拉取的偏移量.  

下面我们看offset在minOffset和maxOffset之间的情况, 也是我们可以拉取消息的唯一一种情况.
```java
GetMessageResult getResult = new GetMessageResult();
SelectMappedBufferResult bufferConsumeQueue = consumeQueue.getIndexBuffer(offset);
for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
    long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
    int sizePy = bufferConsumeQueue.getByteBuffer().getInt();
    long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
    SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
    getResult.addMessage(selectResult);
    nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
}
```
首先根据逻辑位移offset获取ConsumeQueue的文件Buffer, 然后遍历该buffer的条目, 对每一个
条目都获取其对应消息的物理位移和消息大小.  
然后在CommitLog中根据物理位移和消息大小获取具体的消息, 将其加入到返回结果中.  
同时这里会计算下一次拉取的位移,会加上本次拉取的消息个数.

回到PullMessageProcessor中, 接下来我们会对GetMessageResult的几种不同状况, 返回不同的状态码:
```java
case FOUND:
    response.setCode(ResponseCode.SUCCESS);
...
case NO_MATCHED_LOGIC_QUEUE,NO_MESSAGE_IN_QUEUE,OFFSET_FOUND_NULL
    OFFSET_OVERFLOW_ONE:
    response.setCode(ResponseCode.PULL_NOT_FOUND);
...
```
对于FOUND, 这里返回SUCCESS, 下面当返回码为SUCCESS时, 将消息发送给客户端.  
对于其他几种无法获取消息的状态,返回PULL_NOT_FOUND.

下面我们针对这两种码值进行分析.  
1. SUCCESS:
```java
if (this.brokerController.getBrokerConfig().isTransferMsgByHeap()) {
    final byte[] r = this.readGetMessageResult(getMessageResult, requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId());
    response.setBody(r);
} else {
    try {
        FileRegion fileRegion =
            new ManyMessageTransfer(response.encodeHeader(getMessageResult.getBufferTotalSize()), getMessageResult);
        channel.writeAndFlush(fileRegion).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                getMessageResult.release();
            }
        });
    } 
}
```
可以看到, 对于传输的方式, 第一种将CommitLog的buffer中的消息从堆外内存拷贝到堆中,而第二种则使用了FileRegion,利用DMA技术(关于这一点我需要进一步学习)从文件直接发送到网卡缓冲区,是ZeroCopy的应用.

2.PULL_NOT_FOUND:
```java
if (brokerAllowSuspend && hasSuspendFlag) {
    long pollingTimeMills = suspendTimeoutMillisLong; // @1
    if (!this.brokerController.getBrokerConfig().isLongPollingEnable()) {
        pollingTimeMills = this.brokerController.getBrokerConfig().getShortPollingTimeMills(); // @2
    }

    String topic = requestHeader.getTopic();
    long offset = requestHeader.getQueueOffset();
    int queueId = requestHeader.getQueueId();
    PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
        this.brokerController.getMessageStore().now(), offset, subscriptionData, messageFilter); // @3
    this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest); // @5
    response = null;
    break;
}
```
1: 挂起时间,默认15秒  
2: 如果broker配置不支持长轮询, 那么使用短轮询配置:1秒  
3: 创建一个PullRequest(与客户端创建的不一样), 包含开始挂起的时间戳和挂起时间.
4: 调用PullRequestHoldService的suspendPullRequest进行请求的挂起.  

下面我们看一下PullRequestHoldService的实现:

#### Broker端长轮询的实现
PullRequestHoldService中的重要成员属性如下:
```java
private ConcurrentMap<String/* topic@queueId */, ManyPullRequest> pullRequestTable =
        new ConcurrentHashMap<String, ManyPullRequest>(1024);
```
pullRequestTable即存放挂起请求的Map, key为topic@queueId, value即复数个拉取请求.  
suspendPullRequest方法实现即将拉取请求放入到Map中.

同时PullRequestHoldService是一个线程, 我们看它的run方法:
```java
while (!this.isStopped()) {
    try {
        if (this.brokerController.getBrokerConfig().isLongPollingEnable()) { // @1
            this.waitForRunning(5 * 1000);
        } else {
            this.waitForRunning(this.brokerController.getBrokerConfig().getShortPollingTimeMills());
        }

        this.checkHoldRequest(); // @2
    }
}
```
1: 判断是否是支持长轮询的, 然后等待相应的时间.
2: checkHoldRequest方法对Map中的挂起请求做消息到来的通知处理.下面我们分析一下checkHoldRequest方法:
```java
for (String key : this.pullRequestTable.keySet()) {
    ...
    long offset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
    this.notifyMessageArriving(topic, queueId, offset);
}
```
实际上是获取队列中的最大位移,然后调用notifyMessageArriving方法.  
由于位移可能和之前一样,所以猜测notifyMessageArriving方法中一定会对位移做判断,如果没变,那么不做处理.  
下面我我们看notifyMessageArriving方法:
```java
List<PullRequest> requestList = pullRequestTable.get(key).cloneListAndClear();
List<PullRequest> replayList = new ArrayList<PullRequest>();
for (PullRequest request : requestList) { // @1
    long newestOffset = maxOffset;
     if (newestOffset > request.getPullFromThisOffset()) { // @2
         this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),request.getRequestCommand());
    }
    if (System.currentTimeMillis() >= (request.getSuspendTimestamp() + request.getTimeoutMillis())) { // @3
        this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                                request.getRequestCommand());
        continue;
    }
    replayList.add(request); // @4
}
```
1: 遍历对应队列下的request列表
2: 如果当前队列最大位移超过了挂起请求的位移,说明有新消息到来,此时调用PullMessageProcessor的executeRequestWhenWakeup方法如下:
```java
public void executeRequestWhenWakeup(final Channel channel,
    final RemotingCommand request) throws RemotingCommandException {
    Runnable run = new Runnable() {
        public void run() {
            final RemotingCommand response = PullMessageProcessor.this.processRequest(channel, request, false);
            if (response != null) {
                channel.writeAndFlush(response).addListener(new ChannelFutureListener() {...});
            }
        }
    };
    this.brokerController.getPullMessageExecutor().submit(new RequestTask(run, channel, request));
}
```
可以看到只是重新调用了PullMessageProcessor的processRequest进行处理(这里第三个参数设置为false,代表不挂起),然后将处理结果写入到channel中返回给客户端.  
3: 如果超过了挂起时间,那么此时也立即执行一次executeRequestWhenWakeup方法.
4: 如果可以执行到这一步的话说明这个队列没有新的消息到达且没超时,此时将其加入到replayList中继续放入到Map里等待下一次check.

其实除了这种定时检查的方式,Broker在接收到消息时也会立即通知PullRequestHoldService.  
在ReputMessageService.doReput方法中,有如下逻辑:
```java
if (dispatchRequest.isSuccess()) {
    DefaultMessageStore.this.doDispatch(dispatchRequest);
    if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
    && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) {
    DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
        dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
        dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
        dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
}
}
```
在分发Message到IndexFIle和ConsumeQueue之后,会调用messageArrivingListener的arriving方法通知消息的到来.

好的,以上就是broker端处理拉取请求的过程,接下来我们看客户端收到响应之后的回调逻辑PullCallBack.


### 消息拉取回调
PullCallBack首先根据返回的状态码做响应的处理,对于FOUND,会将消息发送给ConsumeMessageService,同时构造下一个拉取请求放入到PullMessageService里.  
对于NO_NEW_MSG(PULL_NOT_FOUND)和NO_MATCHED_MSG(offset过大或过小)都会使用broker返回的offset进行校正并且重新拉取.  
下面我们看FOUND的处理逻辑:
```java
pullRequest.setNextOffset(pullResult.getNextBeginOffset()); // @1
boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList()); // @2
DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                    pullResult.getMsgFoundList(),
                                    processQueue,
                                    pullRequest.getMessageQueue(),
                                    dispatchToConsume); // @3
DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest); // @4
```
1: 设置pullRequest下次拉取的offset为broker计算返回的NextBeginOffset  
2: 将拉取到的消息放入到processQueue中, 供consumeMessageService使用  
3: 提交一个消费请求到consumeMessageService中  
4: 执行下一次拉取请求

下面我们开始分析consumeMessageService是如何实现消费消息的

### 消息消费过程以及消费结果处理
ConsumeMessageService有两个实现类, 分别对应顺序消费和并发消费, 这里我们先分析并发消费,并发消费对应的实现类是ConsumeMessageConcurrentlyService.

我们先看一下ConsumeMessageConcurrentlyService的重要成员变量:
```java
ThreadPoolExecutor consumeExecutor;
BlockingQueue<Runnable> consumeRequestQueue;
MessageListenerConcurrently messageListener;
String consumerGroup;
```
consumeExecutor用来并发执行消费请求.  
consumeRequestQueue用来存放消费请求.  
messageListener是用户注册的业务逻辑.  
consumerGroup代表消费组,每个Consumer实例都有一个ConsumeMessageService.

下面我们看submitConsumeRequest方法:
```java
if (msgs.size() <= consumeBatchSize) {
    ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
    this.consumeExecutor.submit(consumeRequest);
} else {
    ... // 分片批量提交
}
```
可以看到逻辑十分简单,就是将消费请求放到线程池中, 对于大量的消息,分片提交.

然后我们看ConsumeRequest的run方法,其中包含了消费的全部逻辑.  
我们可以把消费过程提取为几步:  
1. 调用业务代码注册的listener
2. 消费进度的更新

下面我们来看代码:
```java
status = listener.consumeMessage(Collections.unmodifiableList(msgs), context); // @1
if (null == status) {
    if (hasException) {
        returnType = ConsumeReturnType.EXCEPTION;
    } else {
        returnType = ConsumeReturnType.RETURNNULL;
    }
} else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
    returnType = ConsumeReturnType.TIME_OUT;
} else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
    returnType = ConsumeReturnType.FAILED;
} else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
     returnType = ConsumeReturnType.SUCCESS;
}
if (!processQueue.isDropped()) {
    ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this); // @2
}     
```
1: 调用业务代码进行消息的处理  
2: 根据消费的结果进行进度的更新

下面我们来看processConsumeResult是如何对进度进行更新的:
```java
switch (this.defaultMQPushConsumer.getMessageModel()) {
    case BROADCASTING:
    ...// 如果失败,打印日志
    case CLUSTERING:
        for each 失败消息:
            boolean result = this.sendMessageBack(msg, context);
            if (!result) {
                msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                msgBackFailed.add(msg);
            }
        if (!msgBackFailed.isEmpty()) {
            consumeRequest.getMsgs().removeAll(msgBackFailed);

            this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
        }
}
long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
    this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
}
```
对于广播模式下的消息,如果消费失败,那么仅仅打印日志,  
对于集群模式下的消息, 如果消费失败,那么将消息重新发回到broker,重新消费.如果发送失败,那么重新发送到ConsumeMessageConcurrentlyService中尝试再次进行消费.

最后,不管成功与否,都从ProcessQueue中移除这一批消息并且返回当前ProcessQUeue中剩余消息的offset的最小值,将其作为新的消费进度提交.

对于通过sendMessageBack方法发回给broker的消息实际上进行了定时消费的处理,这个我们稍后会进行讨论,不过到此为止,关于普通消息的消费过程基本上已经分析完了,逻辑上还是比较清晰的,各个处理流程之间可以很明确的看到是完全异步化的.

### 细节: 消费进度的计算策略

回到之前略过去的计算消费进度的逻辑, 我们首先要明确这个策略是控制什么的,是在什么场景下发生的?  
考虑一个topic,在创建之初,我们给它发送了一些消息,但是此时我们没有消费者消费.  
过了一段时间之后,我们加了消费者去消费它,这时我们想要从头开始消费,还是从消费者开始消费的那一刻起开始消费忽略之前的消息呢?

这取决于我们消息的性质,需要根据业务来衡量.  

这里使用到了我们接下来提到的几个策略,分别是:  
1. CONSUME_FROM_LAST_OFFSET 从尾部开始消费
2. CONSUME_FROM_FIRST_OFFSET 从头部开始消费
3. CONSUME_FROM_TIMESTAMP 消费半小时以前的消息

这里面我们重点分析前两个.

具体代码在RebalancePushImpl.computePullFromWhere方法:
```java
switch (consumeFromWhere) {
    case CONSUME_FROM_LAST_OFFSET: {
        long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
        if (lastOffset >= 0) {
            result = lastOffset;
        }
        // First start,no offset
        else if (-1 == lastOffset) {
            if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                result = 0L;
            } else {
                try {
                    result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
                } catch (MQClientException e) {
                    result = -1;
                }
            }
        } else {
            result = -1;
        }
        break;
    }
}
```
对于CONSUME_FROM_LAST_OFFSET, 如果从远程存储中读到的lastOffset为-1(代表此时该MessageQueue并没有消费记录),
那么才会进入到CONSUME_FROM_LAST_OFFSET的策略中.  

如果是非重试队列, 从broker上获取该MessageQueue的最大位移返回.这就是我们说的从尾部开始消费.

对于CONSUME_FROM_FIRST_OFFSET,代码如下:
```java
case CONSUME_FROM_FIRST_OFFSET: {
    long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
    if (lastOffset >= 0) {
        result = lastOffset;
    } else if (-1 == lastOffset) {
        result = 0L;
    } else {
        result = -1;
    }
    break;
}
```
注意当offsetStore返回的offset为-1时,直接将0作为offset返回,符合对该策略的描述.

## 顺序消费
对于顺序消费, 理论上来说只需要consumer这边做好顺序消费就可以,但实际业务中,顺序消费也要求发送方对发送的消息做好控制,一般是具有某一种特征的消息发送到同一个MessageQueue上.

### 使用示例

下面是一个简单的使用示例:
```java
MQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
producer.start();

String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
for (int i = 0; i < 100; i++) {
    int orderId = i % 10; // @1
    Message msg =
        new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
            ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
    SendResult sendResult = producer.send(msg, new MessageQueueSelector() { // @2
        @Override
        public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
            Integer id = (Integer) arg;
            int index = id % mqs.size();
            return mqs.get(index);
        }
    }, orderId) // @3;

    System.out.printf("%s%n", sendResult);
}

producer.shutdown();
```
上面是生产者的示例代码  
1: 循环100次,根据次数对10取模  
2: 这里我们传入了一个MessageQueueSelector,表示MessageQueue的选择策略, 这里我们的选择策略是根据第三个参数orderId, 使用orderId对队列数进行取模运算.  
3: select方法中的第三个参数

假如消息队列有10个,根据上面的方式发送消息的话最终结果应该是Hello RocketMQ 1 ~ Hello RocketMQ 10会分别发送到q1~q10,
同理Hello RocketMQ 11 ~ Hello RocketMQ 20也会按顺序发送到q1~q10.  

对于消费端如何实现顺序消费呢?  
示例代码如下:
```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_3");

consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

consumer.subscribe("TopicTest", "TagA || TagC || TagD");

consumer.registerMessageListener(new MessageListenerOrderly() {
    AtomicLong consumeTimes = new AtomicLong(0);

    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
        context.setAutoCommit(true);
        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
        this.consumeTimes.incrementAndGet();
        if ((this.consumeTimes.get() % 2) == 0) {
            return ConsumeOrderlyStatus.SUCCESS;
        } else if ((this.consumeTimes.get() % 3) == 0) {
            return ConsumeOrderlyStatus.ROLLBACK;
        } else if ((this.consumeTimes.get() % 4) == 0) {
            return ConsumeOrderlyStatus.COMMIT;
        } else if ((this.consumeTimes.get() % 5) == 0) {
            context.setSuspendCurrentQueueTimeMillis(3000);
            return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
        }

        return ConsumeOrderlyStatus.SUCCESS;
    }
});

consumer.start();
```
这里我们注册的listener是MessageListenerOrderly的实例,所以ConsumeMessageService也不再是ConsumeMessageConcurrentlyService
而是ConsumeMessageOrderlyService.

### 队列负载以及加锁处理
在之前对队列负载进行分析的时候,有这么一段代码:
```java
for (MessageQueue mq : mqSet) {
    if (isOrder && !this.lock(mq)) {
        log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
        continue;
    }
    ...
}
```
意思是在创建ProcessQUeue和pullRequest的时候,会对MessageQueue进行加锁,如果失败,那么将跳过该MessageQueue对应的ProcessQueue的创建处理.

为什么要这样做呢?

仔细想想, 顺序消息消费与普通消息最大的不同在于我们必须保证消息消费的顺序和MessageQueue的offset大小顺序一致,  
如果我们没有加锁的处理,那么就会出现ConsumerA在处理mq1的消息,此时发生重新负载,mq1被分配到了ConsumerB上,于是ConsumerB也去
消费消息,此时就不能保证消息消费的顺序性.  
所以我们必须有一种机制可以保证ConsumerB在消费的时候,其他Consumer必须不能拥有正在消费中的消息.  
这种机制就是mq的锁.

加锁会发送到broker端进行处理,我们看broker端是如何进行处理的.

在Broker端,我们维护了一个Map,key是consumerGroup,value是该消费组拥有端锁,也是一个Map如下:
`ConcurrentMap<String ConcurrentHashMap<MessageQueue, LockEntry>>`,
key是mq,value是对应端锁.

同时锁是有过期时间的,过期时间是1分钟,而重新负载的间隔是20s,所以正常情况下是不会超时的.  
LockEntry的属性如下:
```java
private String clientId;
private volatile long lastUpdateTimestamp = System.currentTimeMillis();
```
clientId即对应的消费端的clientId, lastUpdateTimestamp即每次lock时都会更新的时间戳.

加锁逻辑在RebalanceLockManager#tryLockBatch方法中, 逻辑比较简单, 对于请求的
mq, 尝试对过期的进行加锁,并且返回自己已经拥有的锁.

对于成功加锁的mq,在客户端也会对其对应的ProcessQueue进行加锁:
```java
for (MessageQueue mmqq : lockedMq) {
    ProcessQueue processQueue = this.processQueueTable.get(mmqq);
    if (processQueue != null) {
        processQueue.setLocked(true);
        processQueue.setLastLockTimestamp(System.currentTimeMillis());
    }
}
```

同时在拉取消息时,也会判断队列是否加锁:
```java
if (processQueue.isLocked()) {
    if (!pullRequest.isLockedFirst()) {
        final long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
        boolean brokerBusy = offset < pullRequest.getNextOffset();
        log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
            pullRequest, offset, brokerBusy);
        if (brokerBusy) {
            log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
                pullRequest, offset);
        }

        pullRequest.setLockedFirst(true);
        pullRequest.setNextOffset(offset);
    }
} else {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    log.info("pull message later because not locked in broker, {}", pullRequest);
    return;
}
```
下面我们看顺序消息是如何进行消费的以及对消费结果的处理与普通消息有什么不同.

### 顺序消息消费以及消费结果处理
首先看ConsumeMessageOrderlyService有哪些不一样的成员变量:
```java
MessageListenerOrderly messageListener;
BlockingQueue<Runnable> consumeRequestQueue;
ThreadPoolExecutor consumeExecutor;
MessageQueueLock messageQueueLock = new MessageQueueLock(); @1
```
1: 属性中有一个MessageQueueLock, 它的作用是确保每一个mq只能被线程池中的一个线程消费, 它维护了一个map来实现这个功能:`private ConcurrentMap<MessageQueue, Object> mqLockTable = new ConcurrentHashMap<MessageQueue, Object>();`  
一个mq的锁会被一个线程持续占有一段时间(最大60s).

下面我们看它的ConsumeRequest的run方法:
```java
Object objLock = messageQueueLock.fetchLockObject(this.messageQueue); // @1
if (BROADCASTING || processQueue.isLocked) {
    List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize); // @2
    this.processQueue.getLockConsume().lock();
    status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context); // @3

    if (null == status) {
        if (hasException) {
            returnType = ConsumeReturnType.EXCEPTION;
        } else {
            returnType = ConsumeReturnType.RETURNNULL;
        }
    } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
        returnType = ConsumeReturnType.TIME_OUT;
    } else if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
        returnType = ConsumeReturnType.FAILED;
    } else if (ConsumeOrderlyStatus.SUCCESS == status) {
        returnType = ConsumeReturnType.SUCCESS;
    }    
    // 是否由这个线程继续消费
    continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this); // @4
}
```
1: 获取线程池内线程的队列锁  
2: 从ProcessQueue中获取消息并且将消息放到consumingMsgOrderlyTreeMap中  
3: 消费消息  
4: 对消费结果进行处理  

为什么要将正在消费的消息放到consumingMsgOrderlyTreeMap中呢?  
因为这样方便回滚,即将consumingMsgOrderlyTreeMap再put回msgTreeMap中.  
为什么需要回滚?  
因为与普通消息不同,可以直接发回broker重新消费,如果顺序消费中间出错为了保持顺序性会一直消费下去,卡在原地.

下面我们对第四步进行分析:
```java
case SUCCESS:
    commitOffset = consumeRequest.getProcessQueue().commit(); // @1
    break;
case SUSPEND_CURRENT_QUEUE_A_MOMENT:
    if (checkReconsumeTimes(msgs)) { // @2
        consumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs); // @3
        this.submitConsumeRequestLater( 
            consumeRequest.getProcessQueue(),
            consumeRequest.getMessageQueue(),
            context.getSuspendCurrentQueueTimeMillis());
        continueConsume = false;
    } else {
        commitOffset = consumeRequest.getProcessQueue().commit();
    }
    break;

if (commitOffset >= 0 && !consumeRequest.getProcessQueue().isDropped()) { // @4
    this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);
}
```
1: 如果消费成功,那么调用processQueue的commit方法,该方法会将consumingMsgOrderlyTreeMap清空,并且返回consumingMsgOrderlyTreeMap中最大的offset+1.  
2: 如果消费失败,那么校验当前消息是否重新消费次数大于阈值,如果大于那么直接提交否则继续重试,实际上这个阈值默认是Long.MAX_VALUE,也就是说默认的行为是永远重试.  
3: 将consumingMsgOrderlyTreeMap中的msg重新放入daomsgTreeMap中,重新提交一个消费请求进行消费.  
4: 提交消费进度

### 定时消息
上面说到, 当普通消息消费失败时, 会将消费失败的消息重新发往broker:
```java
this.defaultMQPushConsumerImpl.sendMessageBack(msg, delayLevel, context.getMessageQueue().getBrokerName());
```
delayLevel小于0代表不延迟,直接进入DLQ,等于0代表由broker端控制延迟,大于0代表由客户端控制,这里为0,表示由broker控制延迟.

对应的RequestCode时RequestCode.CONSUMER_SEND_MSG_BACK, 让我们来看broker端是如何处理的:
代码在SendMessageProcessor#processRequest中:

```java
processRequest(ChannelHandlerContext ctx, RemotingCommand request)
    consumerSendMsgBack(ctx, request)
        String newTopic = MixAll.getRetryTopic(requestHeader.getGroup()); // @1
        MessageExt msgExt = this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getOffset()); // @2
        if (msgExt.getReconsumeTimes() >= maxReconsumeTimes || delayLevel < 0) { // @3
            // 进入DLQ
        } else { // @4
            if (0 == delayLevel) {
                    delayLevel = 3 + msgExt.getReconsumeTimes();
            }
        }
        MessageExtBrokerInner msgInner = new MessageExtBrokerInner(); // @5
        msgInner.setTopic(newTopic);
        msgInner.setReconsumeTimes(msgExt.getReconsumeTimes() + 1);
        // set body,queueId,originmsgId etc
        PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner); // @6
```
1: 根据消费组创建该消费组的重试主题, 这里可以看出重试主题是以消费组为单位创建的.  
2: 从commitLog中得到实际的消息, 这里会存放重新消费的次数.  
3: 如果最大重试次数超过重试次数阈值(默认16次)或者客户端传来的delayLevel小于0,代表将要进入DLQ.  
4: 否则根据客户端传来的delayLevel是否大于0,如果是,代表由客户端控制延迟时间,否则将reconsumeTimes加3获得延迟.这里可以看到
如果是0的话即第一次重试会delayLevel的值是3.  
 "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"  
 这是rocketMQ支持的延迟消费的时间,delayLevel值是3的对应的是10s.
5: 创建新的消息,设置topic为重试主题,以及将重新消费次数加一
6: 最后,将其存储到commitLog中.

下面我们分析延迟消息是如何存储以及再次消费的.在putMessage中会对消息的延迟等级做判断,如果大于0说明是延迟消息,需要做特殊处理.

```java
if (msg.getDelayTimeLevel() > 0) {
    topic = ScheduleMessageService.SCHEDULE_TOPIC; // SCHEDULE_TOPIC_XXXX
    queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel()); // delayLevel - 1

    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
    msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

    msg.setTopic(topic);
    msg.setQueueId(queueId);
}
```
关键代码如上,
首先获取延迟消费的topic, 然后根据延迟等级获得queueId, 接着将原topic和queueId放入message的properties中,
最后将延迟主题和队列id替换原来的.

之后的代码就和之前的文章分析的类似的,此处就不再赘述. 

既然我们根据延迟等级将消息放入延迟消费主题的不同的queue中,那么对于延迟消费主题来说应该有一个特殊的线程去将
不同队列的消息以一定的时间间隔将原topic和queueId还原并且放入到ConsumeQueue中.

下面我们就来分析ScheduleMessageService:
首先来看它的重要的属性:
```java
    private final ConcurrentMap<Integer /* level */, Long/* delay timeMillis */> delayLevelTable =
        new ConcurrentHashMap<Integer, Long>(32);
```
delayLevelTabley用来存放延迟等级和延迟时间对应的关系.  

下面我们来看ScheduleMessageService是如何启动的.
```java
public void start() {
    if (started.compareAndSet(false, true)) {
        this.timer = new Timer("ScheduleMessageTimerThread", true);
        for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) { // @1
            Integer level = entry.getKey();
            Long timeDelay = entry.getValue();
            Long offset = this.offsetTable.get(level); // @2
            if (null == offset) {
                offset = 0L;
            }

            if (timeDelay != null) {
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME); // @3
            }
        }

        this.timer.scheduleAtFixedRate(new TimerTask() { // @4
            @Override
            public void run() {
                try {
                    if (started.get()) ScheduleMessageService.this.persist();
                } catch (Throwable e) {
                    log.error("scheduleAtFixedRate flush exception", e);
                }
            }
        }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
    }
}
```
1: 遍历延迟等级  
2: 对每个延迟等级也即每个延迟主题下的队列, 去取上次处理到的offset  
3: 使用定时器, 定时执行DeliverDelayedMessageTimerTask, 注意此处并没有类似scheduleAtFixDelay这种, 所以猜测DeliverDelayedMessageTimerTask一定有再次调度的方法.  
4: 定时持久化进度到磁盘上  

可以知道主要逻辑都在DeliverDelayedMessageTimerTask中, 所以我们来看DeliverDelayedMessageTimerTask的处理逻辑:

```java
ConsumeQueue cq =
    ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(SCHEDULE_TOPIC, // @1
            delayLevel2QueueId(delayLevel));
SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
    long offsetPy = bufferCQ.getByteBuffer().getLong();
    int sizePy = bufferCQ.getByteBuffer().getInt();
    long tagsCode = bufferCQ.getByteBuffer().getLong(); // @2 
    long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode); // @3
    nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
    long countdown = deliverTimestamp - now; // @4
    if (countdown <= 0) {
        // @5
    } else { // @6
        ScheduleMessageService.this.timer.schedule(
            new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),
            countdown);
        ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
    }
}
```
1: 根据topic和延迟等级找到对应的ConsumeQueue  
2: 遍历ConsumeQueue条目, 注意这里的tagsCode, 在延迟消息的情况下,分发到ConsumeQueue时这里做了一个“手脚“,即将tagsCode存储为:storeTimeStamp + delayTime,消息存储到CommitLog的时间戳 + 延迟时间.  
3: 判断now + delayTime和理论上需要分发的时间的大小, (这块没看懂...)  
4: 判断距离分发时间还有多长时间  
5: 如果已经到了分发时间了,那么开始分发  
6: 还没到,那么创建一个新的分发任务,在countdown后执行.  

上述步骤中开始分发意味着我们要将该消息原来的topic和queueId还原,然后重新放入到CommitLog中一次.  
```java
MessageExt msgExt = defaultMessageStore.lookMessageByOffset; // @1
MessageExtBrokerInner msgInner = this.messageTimeup(msgExt);  // @2
messageStore.putMessage(msgInner); // @3
```
1: 从CommitLog中根据物理位移和消息大小取出消息  
2: 将message中的queue和queueId还原  
3: 重新投递到commitLog中  

重新放入到CommitLog中去也就意味着回分发到ConsumeQUeue中,这时消费者就可以消费到延迟消费的消息了.

## 总结
流程图
![avatar](/static/img/消息消费流程.png)