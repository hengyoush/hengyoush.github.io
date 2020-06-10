---
layout: post
title:  "Rocketmq IndexFile源码解读"
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

#### ProcessQueue
ProcessQueue是消费端上对MessageQueue对映射,是一对一的关系, ProcessQueue中存放着从Broker上的MessageQueue中拉取的待处理的消息, 每次成功处理完消息后会将消息从其中移除, 我们来看一看它的重要的成员变量:  
```java
ReadWriteLock lockTreeMap;
TreeMap<Long, MessageExt> msgTreeMap;
volatile boolean dropped = false;
```
1. lockTreeMap: ProcessQueue中每一个对msgTreeMap的操作都会加上对应的读/写锁,防止对TreeMap的并发修改.  
2. msgTreeMap: key是消息的LogicOffset, value即消息本身.  
使用TreeMap即表示其Key是有序的, 为什么要使用TreeMap呢?  
使用TreeMap实际上具有了排序和Map的功能,让我们看看假如:  
1、只有排序的功能,没有Map的功能: 考虑并发消费的场景, 当一批消息消费完成时, 需要从ProcessQueue中移除, 由于不具有Map的功能, 只能遍历Queue, 时间复杂度时O(n).  
2、只要Map的功能没有排序的功能: 考虑顺序消费的场景, 需要从Map中取出offset最小的几个消息, 此时由于没有排序只能遍历Map, 时间复杂度依然是O(n).  
所以使用TreeMap是合理的.
3. dropped: 代表这个ProcessQueue是否因为队列重新负载导致消费者不在具有该MessageQueue的消费权利, 如果设置为true, 那么会停止该消息队列的消费,同时持久化消费进度(集群模式下持久化到broker中).

updateProcessQueueTableInRebalance的伪代码如下(省略了与主流程无关的细节):
```
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


### 消费请求发送

### Broker端对消费请求的处理

### 消息拉取回调

### 消息消费过程以及消费结果处理

### 细节: 消费进度的计算策略

## 顺序消费

概述

### 队列负载以及加锁处理

### 顺序消息消费以及消费结果处理

### 定时消息

## 总结
流程图
