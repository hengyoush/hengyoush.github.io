# Kafka核心概念
## 目标
将Kafka的相关概念解释清楚,为之后的源码解析铺路.


## 概念
Consumer: 消费消息的程序
Consumer Group消费组: 多个Consumer组成一个消费组, 在消费组内的Consumer消费相同的Topic, 它们共享消费的位移, 实现消费的复杂均衡.

Topic: 消息的逻辑中转站, Producer将消息发送到某个Topic下, Consumer则去消费某个Topic下的消息, 实现了消息消费的解耦.

Partition: 分区,每个Topic下含有多个partition, 可以把他们当作逻辑队列. producer将消息均匀的发送到每个partition, 每个Consumer会被分配若干个分区, 一个分区的消息会被某个Consumer独占消费.同时分区有Leader分区和Follower分区的类别,只有Leader分区才可以被读写,而Follower分区只是做Replica备份,Follower分区会拉取Leader分区的消息进行同步.
ISR: In-Sync Replica, 即与Leader分区同步的Follower分区, 只有在ISR内的分区Leader分区故障转移的时候有没资格当选Leader分区.
高水位: 一个消息offset的标志位,在高水位之前(不包括HW当前指向的消息)的消息对Consumer是可见的,即可以被消费的;而在其之后的消息还不能被消费.每次Follower分区来拉取消息的时候会对其进行更新.
LEO:Log End Offset,即分区中下一条即将写入消息的offset,高水位不允许超过LEO.
LSO:Log Start Offset,分区中可以被消费的最小位移.LSO可能在删除过期日志(还有其他的情况?)的时候被更新.
LeaderEpoch:LeaderEpoch包含Epoch和该Leader写入的第一条消息的位移两个部分,Epoch在每次Leader分区故障转移的时候会递增.该值应该在每次消息写入的时候更新.Epoch则应该在
每次故障转移的时候更新.
为什么要有LeaderEpoch呢?
没有LeaderEpoch会发生消息丢失的问题,原因是Follower副本重启时会根据它当前的高水位进行日志的截断操作,又由于*Leader分区与Follower分区在HW的更新上不是同时发生的*,
当Follower还没有来得及更新HW时,这时候就会发生日志的截断,当截断发生后如果Leader宕机,那么Follower被选举为新的Leader,此时假如A重启同样需要执行
日志截断操作(这里是不是需要向Leader请求得到HW?),这样这条被截断的消息将不复存在.
再平衡: 当消费组内成员发生变动,或者partition的数目发生变动时,会发生消费组的重平衡.重平衡期间无法消费,是STW的.重平衡需要Coordinator组件参与.
Coordinator: 位于Broker中,每个Broker都会有一个Coordinate组件.它用于管理消费组的位移以及重平衡等.每个消费组通过确定其内部消费位移主题的分区来找到Coordinate.
消费位移: 消费组的位移存在一个特殊的topic:`__consumer_offset`中,消费组将位移提交给Coordinator.
Producer: 发送消息的程序.
Broker: 存储消息, 接收Producer发送的消息, 同时提供Consumer消费消息.
日志: 日志是Kafka存储的基本结构,消息是存在日志里的,同时索引文件也是存在日志里的.
日志文件结构: Kafka的存储结构的基础单元是Topic下的分区,每个基本的存储单元有消息日志,索引日志等.每个日志会分段,分段叫做LogSegment.日志的名称是以该分段的baseOffset命名的,即该日志的第一条消息的offset.
日志分段的规则:根据大小、根据时间等?
消息的结构:
批量消息的结构:
日志的加载与恢复:
日志清理:
日志压缩:将key相同的消息进行压缩,保留最新的消息.
磁盘存储技术:













