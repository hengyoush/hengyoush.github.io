---
layout: post
title:  "Kafka源码学习笔记"
date:   2020-06-13 15:20:00 +0700
categories: [kafka]
---

## 学习目标
学习kafka的消息存储、消息客户端发送、消费.
队列负载、元数据与zk同步等,消息消费进度管理.

### 分区副本
指消息发送者将消息发送至哪个分区,在这一过程中使用的策略.
轮询策略
随机策略

主题下会分区,将消息分成几个逻辑队列分散至各个broker.每个分区都有一个Leader分区和数个Follower分区.
Follower没有提供读写功能仅仅是为了备份.

当Leader分区所在的broker挂了之后,如何选举出下一个Leader分区.  

从ISR中选出,如果ISR为空,那么看是否开启unclean选举.

#### ISR
这就涉及到ISR,即In-Sync Replicas,指与Leader同步的副本.
ISR一定包含Leader本身.

什么样的副本算作是ISR呢.
只要Follower落后Leader副本的时间间隔没有超过replica.lag.time.max.ms(默认10s),那么就算是ISR内的副本

### 消息发送的坑,要注意避免
1. 使用带回调的send方法send(msg, callback)
2. 设置acks=all,所有的ISR副本所在的broker都要接收到消息,才算提交成功
3. 设置retries为较大的值,发送失败时自动重试.
4. 禁用unclean选举
5. 多设置副本数》=3
6. min.insync.replicas,设置消息提交副本数的下限值,为了防止ISR中只有Leader一个,如果该值设置为1,此时Leader挂了,那么即使设置了acks=all消息也丢失了.
7. enableAutoCommit设置为false.

### 拦截器

### 事务型 Producer
将消息原子性的写入多个分区中,要么一起成功要么一起失败

### 重平衡触发的条件
1. 消费组成员发生变更
2. topic的分区数发生变更
3. 订阅主题数发生变更

重平衡发生的时候会STW.

重平衡过程中,所有Consumer都要参与,由Coordinator进行协调,完成订阅主题的分配.

Coordinator:Broker端的组件专门为 Consumer Group 服务，负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理.  
Consumer端提交位移实际上是向Coordinator提交位移,由Coordinator完成ConsumerGroup的注册,成员管理,元数据管理等.  
所有Broker都有Coordinator组件,所以重平衡的第一步是确定Coordinator.  

确定Coordinator算法:
1. partitionId = groupId.hashCode % partitionCount,partitionCount为位移主题的分区数.
2. 找到位移主题的partitionId所在的Leader副本的Broker即Coordinator所在的Broker.

不必要的重平衡:
1. 未能及时发送心跳,导致Consumer被踢出消费组.session.timeout.ms表示心跳的超时时间.heartbeat.interval.ms心跳发送时间间隔.
2. 消费能力跟不上,max.poll.interval.ms这个参数指定消费者两次poll之间的最大间隔,如果在这个间隔内没有消费完,
那么Consumer会主动发起离开组的请求.

### 重平衡详解
重平衡流程由Coordinator发起,通过心跳响应完成这个的通知.

重平衡有几个重要的状态.
Empty Stable PreparingRebalance CompleteingRebalance Dead

当新成员入组或者心跳过期时会从Stable转入PreparingRebalance.

消费端重平衡分为两步,1-加入组,2-等待LeaderConsumer指定分配方案:
1. JoinGroup,消费者向Broker发送自己需要订阅的主题列表.
2. SyncGroup,领导者向Broker发送分配方案,其他消费这也会发送SyncGroup只不过发送的是空请求.由Coordinator响应给
各个消费者自己负责的分区.

这里有一个问题,如果在SyncGroup中又有新的成员加入或者离开该怎么办?

Broker端重平衡.



### 消费位移管理
位移主题: __consumer_offsets
位移管理的机制: 将Consumer的位移作为一条普通Kafka消息发送至__consumer_offsets主题中.

位移消息格式是Kafka自己定义的,用户无法修改.

位移消息的Key:<ConsumerGroupId,Topic,QueueId>
消息体:位移值

__consumer_offsets中的消息格式
1. 上述存放消费位移的
2. 保存ConsumerGroup信息的消息
3. 删除ConsumerGroup过期位移,或者删除Group的消息.即tombstone消息,消息体是null

默认50个分区,副本数是3.

清理过期消息,什么是过期消息,同一个Key,旧消息是过期消息.
Kafka中的LogCleaner线程用来清理过期消息.

### Producer创建连接的时机
- 创建KafkaProducer实例时,创建Sender线程,向bootstrap.servers请求集群元数据.
- 首次更新元数据后会再次创建与集群所有broker之间的连接.如果其中存在不需要发送消息的broker,那么会在connections.max.idle.ms之后被关闭.
- 发送消息时,如果发现与该broker没有创建连接,那么也会创建连接.
- 设置Producer 端 connections.max.idle.ms 参数大于 0,那么第一步创建的连接将自动关闭.

### Consumer如何管理连接
连接是在KafkaConsumer.poll方法调用时被创建的,具体来说分为以下几种情况:
- 发起FindCoordinate请求时.Coordinator负责消费组的进度管理,在消费消息之前需要确定Coordinator所在的broker.
- 连接Coordinator时.
- 消费消息时,需要创建各个分区的Leader副本所在的Broker.

### Broker如何处理请求
经典的Reactor架构.
Acceptor+ IO 线程 + 业务线程
Acceptor接收到客户端的连接请求,将请求负载到IO线程上,之后IO线程就负责接收客户端的请求,然后将请求放到
一个请求队列中,然后业务线程将请求队列中的请求取出,进行处理,将处理后的结果通过IO线程的响应队列返回给IO线程,
然后IO线程将响应队列中的响应写出给客户端.

### 高水位与Leader Epoch
高水位是一个位移标记,所有高于该标记的都是未提交消息,不能被Consumer消费,只有低于HW的才可以被消费.
它的作用主要有以下两点:
- 定义消息可见性,用来标示分区下哪些消息是可以被消费的.
- 帮助完成副本同步

Broker中的每一个分区都保存有HW(高水位)和Log End Offset(简称LEO).
如下所示,假设Broker0上有一个Leader副本,Broker1上有对应的Follower副本.Broker0上也会管理
远程副本即Broker1上的副本的LEO但不会管理HW.

高水位更新机制:
Broker0上Leader LEO: 在Producer发的消息写入磁盘之后就会更新.
Broker0上Leader HW: 更新时机为Leader更新LEO和更新远程副本的LEO之后,更新为当前HW和所有远程副本LEO最小值中的较大的那一个.
Broker0上远程副本的LEO: Follower在拉取Leader副本消息时,会告诉Leader从哪里开始拉取,Leader使用这个值更新远程副本的LEO
Broker1上的LEO: Follower从Leader上拉取消息写入磁盘之后即会更新LEO
Broker1上的HW: Follower更新完LEO之后,会取当前Leader的HW和自己的LEO的最小值作为更新之后的HW.

Leader副本处理Producer请求流程如下:
1. 写入消息到本地磁盘,更新LEO
2. 更新HW:
    2.1 取当前所有远程副本的LEO值
    2.2 取当前HW
    2.3 取max{当前HW, min{远程副本LEO}}作为更新的HW

处理Follower拉取消息逻辑如下:
1. 读取磁盘中的消息
2. 根据Follower请求中的位移更新远程副本的LEO
3. 更新Leader的HW


Follower:
1. 写入消息到本地磁盘中
2. 更新LEO
3. 取min{LEO, Leader发来的当前HW}作为更新的HW

Leader Epoch:每个Leader副本都有一个Leader Epoch,它记录了当前的Epoch和该Epoch开始时的起始位移(即
该Leader写入的第一条消息的位移).

每次Leader分区发生变更时都会追加Epoch的值.




