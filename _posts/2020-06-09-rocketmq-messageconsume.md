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

### 队列负载

### 消费请求发送

### Broker端对消费请求的处理

### 消息拉取回调

### 消息消费过程以及消费结果处理

## 顺序消费

概述

### 队列负载以及加锁处理

### 顺序消息消费以及消费结果处理

### 定时消息

## 总结
流程图
