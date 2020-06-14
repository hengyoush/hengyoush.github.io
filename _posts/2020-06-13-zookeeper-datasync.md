---
layout: post
title:  "Zookeeper Leader成员发现以及数据同步流程源码分析"
date:   2020-06-13 15:20:00 +0700
categories: [zookeeper]
---

## 学习目标
当Zookeeper完成了成员选举之后, 接下来就是要实现成员当发现了, 对于zk来说, 选举并没有实现一个集群成员当确定,  
因为zk当选举没有回复, 只有通过集群节点内部不断当"举荐“过程才能知道自己是Leader, 所以选举之后  
第一件事情就是确定集群内的成员,然后进行数据的一致性修复.

这里我们的学习目标就是搞清楚Zookeeper是如何实现ZAB协议的成员发现以及数据同步流程的.

## 成员发现
当选举成功之后, 该成为Leader的和Follower的都会进行各自的逻辑, 下面我们分别看看Leader和Follower会进行怎样的初始化流程.

### Leader的初始化过程
我们首先来看Leader的初始化过程, 我们上面提到当完成了成员选举之后, 接下来就是要实现成员当发现了, 那么成员发现是怎么做到的呢?  
我们带着问题来分析源码.

来看Leader.java的lead方法.
```java
cnxAcceptor = new LearnerCnxAcceptor(); 
cnxAcceptor.start(); 
```
在这里新建了LearnerCnxAcceptor这个对象, 这个对象的作用是accept follower通过主从端口建立的连接请求,
并且初始化一个LeanerHandler的线程来处理follower上的请求以及心跳等, 这是我们接下来分析的重点.


```java
long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());
```
这行代码的作用是确定acceptedEpoch, 什么是acceptedEpoch呢? 还有一个currentEpoch, 它们的区别是什么呢?  
因为一个新当选的Leader需要将自己的epoch自增, 所以在这里需要将epoch+1, 那么为什么有两个变量呢?  
这里其实牵扯到Zookeeper的一些历史遗留问题,有兴趣的可以看下这里的引用:  
> 当一个集群内的zk节点重启时, 它的状态变化为LEADING-LOOKING—FOLLOWING,为什么呢?  
是因为老版本的zk新节点重启后会接收到
> 其他节点的所有历史选举消息, 然而在原先的lookForLeader逻辑中,如果一个节点收到了来自集群内所有节点的消息时,它就会
> 退出LOOKING,变为LEADING,当然在这之后它会由于发送NEWLEADER消息没有ACK而回到FLE状态并且最终变为FOLOWER,但是在
> 退出LEADER的时候可能会将TnxLog写入到磁盘中,TxnLog的名称是epoch+zxid,在老版本的zk中,是没有acceptedEpoch的将epoch自增
> 是在转为LEADER之后,但是在连接到大多数节点之前,如果在上述的情况中,退出LEADER之前就将日志文件名称记录为最新的epoch中.  
> 有人会问,记录到日志文件又怎么样?    
> 实际上,zk会将磁盘中的TxnLog的文件名作为自己的epoch,这样的话此时这个重启之后的节点的epoch就比原先集群里面的大1,
> 此时转变为FOLLOWER时,由于在成员发现这一步(之后会讲到),发现LEADER的epoch竟然比自己小,那么自己又会退出FOLLOWER状态,重新进入FLE,
> 如此循环下去.  
这是对应的issue:https://issues.apache.org/jira/browse/ZOOKEEPER-975 https://issues.apache.org/jira/browse/ZOOKEEPER-790  
> 和wiki:https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab1.0#space-menu-link-content
1. acceptedEpoch记录的是接收到最后的NEWEPOCH消息中的epoch.
2. currentEpoch指接收到最新NEWLEADER消息中的epoch.

可以这么理解,一个是不稳定的新的epoch,一个是得到绝大多数节点公认的新的epoch.
```java
public long getEpochToPropose(long sid, long lastAcceptedEpoch) throws InterruptedException, IOException {
    synchronized(connectingFollowers) {
        if (!waitingForNewEpoch) {
            return epoch;
        }
        if (lastAcceptedEpoch >= epoch) {
            epoch = lastAcceptedEpoch+1;
        }
        if (isParticipant(sid)) {
            connectingFollowers.add(sid);
        }
        QuorumVerifier verifier = self.getQuorumVerifier();
        if (connectingFollowers.contains(self.getId()) &&
                                        verifier.containsQuorum(connectingFollowers)) {
            waitingForNewEpoch = false;
            self.setAcceptedEpoch(epoch);
            connectingFollowers.notifyAll();
        } else {
            long start = Time.currentElapsedTime();
            long cur = start;
            long end = start + self.getInitLimit()*self.getTickTime();
            while(waitingForNewEpoch && cur < end) {
                connectingFollowers.wait(end - cur);
                cur = Time.currentElapsedTime();
            }
            if (waitingForNewEpoch) {
                throw new InterruptedException("Timeout while waiting for epoch from quorum");
            }
        }
        return epoch;
    }
}
```
这个方法会在Leader和LearnHandler两个地方调用,这个方法就是更新自己的acceptedEpoch,取Follower最大的+1,并且判断是否大多数的FOLLOWER已经给自己发送了FOLLOWERINFO消息,如果是那么唤醒所有在此方法上等待的线程,否则在此方法上等待.

这里的FOLLOWERINFO消息实际上就是FOLLOWER告诉LEADER自己当前的信息,就是成员发现的一个步骤.

### FOLLOWER的初始化流程

接下来我们把目光转向FOLLOWER,看看FOLLOWER在选举之后进行的流程.
```java
connectToLeader(leaderServer.addr, leaderServer.hostname); // 与LEADER建立连接
long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO); // 发送握手消息以及当前的acceptedEpoch,并且返回与LEADER达成一致的新的EPOCH
```

connectToLeader没什么好说的就是与LEADER在主从通信端口上建立socket连接.但是我们要注意LEADER在连接建立时候的行为,这个我们稍后再说.先看FOLLOWER的registerWithLeader方法做了什么事.
```java
QuorumPacket qp = new QuorumPacket();                
qp.setType(pktType);
qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
LearnerInfo li = new LearnerInfo(self.getId(), 0x10000, self.getQuorumVerifier().getVersion());
writePacket(qp, true);
readPacket(qp);
```
首先构建FOLLOWERINFO消息,这里用到了QuorumPacket,这是Zookeeper通用的原来通信的结构,主要包含如下部分:
```java
  private int type; // 指明消息类别
  private long zxid; // zxid
  private byte[] data; // 数据
  private java.util.List<org.apache.zookeeper.data.Id> authinfo; // 认证信息
```

将packet发送至LEADER之后等待响应.这时让我们把目光转向LEADER,看LEADER是如何处理FOLLOWER的连接以及FOLLOWERINFO消息的.

### 创建新的epoch
LearnerHandler在FOLLOWER与LEADER连接时被创建,是用来处理每个FOLLOWER的线程,下面我们来看它的run方法.
```java
// .. 省略IO序列化相关代码
QuorumPacket qp = new QuorumPacket(); // @1
readRecord(qp, "packet");
byte learnerInfoData[] = qp.getData();
long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
long zxid = qp.getZxid();
long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
long newLeaderZxid = ZxidUtils.makeZxid(newEpoch, 0);
QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, newLeaderZxid, ver, null); // @2
oa.writeRecord(newEpochPacket, "packet");

ia.readRecord(ackEpochPacket, "packet");
ByteBuffer bbepoch = ByteBuffer.wrap(ackEpochPacket.getData());
ss = new StateSummary(bbepoch.getInt(), ackEpochPacket.getZxid());
leader.waitForEpochAck(this.getSid(), ss); // @3
```
1: 解析FOLLOWER传入的FOLLOWERINFO消息, 获取其lastAcceptedEpoch ,调用上面提到的getEpochToPropose方法,最终返回的是LEADER新的epoch,只不过这时还不能直接使用,还需要向FOLLOWER发送NEWEPOCH消息.  
2: 构建LEADERINFO消息,将包含新的epoch的zxid发给FOLLOWER.  
3: 等待FOLLOWER回复NEWEPOCH.

下面回到FOLLOWER,让我们看看FOLLOWER是如何处理NEWEPOCH消息的, 还是registerWithLeader方法
```java
readPacket(qp);
final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
if (newEpoch > self.getAcceptedEpoch()) {
    wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
    self.setAcceptedEpoch(newEpoch);
} else {
    throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
}
QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
writePacket(ackNewEpoch, true);
return ZxidUtils.makeZxid(newEpoch, 0);
```
如果是LEADER比较大那么更新自己的AcceptedEpoch. 否则报错,进入FLE开始选举.  
如果一切正常的话那么此时返回ACKEPOCH消息,代表自己接受LEADER的epoch,并且返回新的zxid.

接下里回到LEADER的线程中,我们看到Leader会阻塞在getEpochToPropose这个方法上,当所有节点都连接上之后Leader会这么样呢?  
```java
waitForEpochAck(self.getId(), leaderStateSummary);
self.setCurrentEpoch(epoch);

waitForNewLeaderAck(self.getId(), zk.getZxid());
```
首先回复自己的epochAck,然后当waitForEpochAck返回时也就代表着大多数节点已经接收了自己的epoch,那么下面就设置currentEpoch作为正式的epoch.(FOLLOWER什么时候设置currentEpoch呢?别急,接下里会看到)

然后leader开始等待newleader了,newLeader消息是在数据同步之后FOLLOWER才会ackLEADER的,所以这里我们先看数据同步的流程.

至此成员发现这一步算是结束了,接下来进入到数据同步的流程中.

## 数据同步
由于FOLLOWER在进入选举之前会将本地没有提交的事务全部提交(org.apache.zookeeper.server.quorum.Follower#shutdown),所以在新的LEADER当选的时候,一定存在某些日志缺失的,或者日志冲突的现象.这时候我们要将事务一致化,就需要LEADER将FOLLOWER的zxid(ACKEPOCH消息中)与自己的最新提交的zxid做比对,根据不同情况作出不同的处理.

我们定义以下几个变量:
```java
maxCommittedLog: 提交的最大zxid
minCommittedLog: 内存中提交的最小zxid
lastProcessedZxid: dataTree上提交的最大zxid
peerLastZxid: follower的最大zxid,这个zxid是日志中的,可能是没有提交的.
```
其中maxCommittedLog和minCommittedLog是zk在内存中维护的一张链表,用于快速进行follower的日志同步.  
有如下几种情况:
1. lastProcessedZxid == peerLastZxid, 这种情况无需同步
2. peerLastZxid > maxCommittedLog, 这种情况需要进行TRUNC,truncate到maxCommittedLog即可
3. minCommittedLog <= peerLastZxid <= maxCommittedLog,直接使用zk中的链表进行DIFF同步
4. peerLastZxid < minCommittedLog, 这时需要读取磁盘上的事务日志和内存中的链表进行DIFF同步或者采用SNAP方式进行同步

### 数据同步--DIFF、TRUNC还是SNAP?
关键代码如下:
```java
if (lastProcessedZxid == peerLastZxid) {
    queueOpPacket(Leader.DIFF, peerLastZxid);
} else if (peerLastZxid > maxCommittedLog && !isPeerNewEpochZxid) {
    queueOpPacket(Leader.TRUNC, maxCommittedLog);
    currentZxid = maxCommittedLog;
} else if ((maxCommittedLog >= peerLastZxid) && (minCommittedLog <= peerLastZxid)) {
    Iterator<Proposal> itr = db.getCommittedLog().iterator();
    currentZxid = queueCommittedProposals(itr, peerLastZxid,null, maxCommittedLog);
} else if (peerLastZxid < minCommittedLog && txnLogSyncEnabled) {
    long sizeLimit = db.calculateTxnLogSizeLimit();
    Iterator<Proposal> txnLogItr = db.getProposalsFromTxnLog(
                        peerLastZxid, sizeLimit);
    if (txnLogItr.hasNext()) {
        currentZxid = queueCommittedProposals(txnLogItr, peerLastZxid,
                                                minCommittedLog, maxCommittedLog);

        Iterator<Proposal> committedLogItr = db.getCommittedLog().iterator();
        currentZxid = queueCommittedProposals(committedLogItr, currentZxid,
                                                null, maxCommittedLog);
    } else {
        snap = true;
    }
}
```
对应上面的几种情况, 接下里我们对FOLLOWER端进行分析:

queueCommittedProposals代码:
```java
long prevProposalZxid = -1;
while (itr.hasNext()) {
    Proposal propose = itr.next();
    long packetZxid = propose.packet.getZxid();
                // abort if we hit the limit
    if ((maxZxid != null) && (packetZxid > maxZxid)) {
        break;
    }
        // skip the proposals the peer already has
    if (packetZxid < peerLastZxid) {
        prevProposalZxid = packetZxid;
        continue;
    }

    if (needOpPacket) {
        if (packetZxid == peerLastZxid) {
            queueOpPacket(Leader.DIFF, lastCommittedZxid);
            needOpPacket = false;
            continue;
        }

        if (isPeerNewEpochZxid) {
            queueOpPacket(Leader.DIFF, lastCommittedZxid);
            needOpPacket = false;
        } else if (packetZxid > peerLastZxid  ) {
            // Peer have some proposals that the leader hasn't seen yet
            // it may used to be a leader
            queueOpPacket(Leader.TRUNC, prevProposalZxid);
            needOpPacket = false;
        }
    }
    queuePacket(propose.packet);
    queueOpPacket(Leader.COMMIT, packetZxid);
    queuedZxid = packetZxid;
}
```

根据LEADER发送的不同请求类型:  
DIFF: 接下来从leader那里会收到一系列的proposal和commit的packet.  
TRUNC: 根据leader传来的maxCommittedLog, truncate掉从maxCommittedLog开始的所有日志  
SNAP: 清空自己的zkDatabase, 反序列化leader传来的快照作为自己的zkDatabase

接下来就开始从leader接收proposal和commit包了.
将PROPOSAL放到packetsNotCommitted中临时存储, 等待匹配的COMMIT包, 将其应用到dataTree中.

同步完成之后收到NEWLEADER消息会返回ACK:
```java
writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
```

那么当Leader做完数据同步包的发送之后,节开始发送NEWLEADER:
```java
QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
        newLeaderZxid, leader.self.getLastSeenQuorumVerifier()
                .toString().getBytes(), null);
queuedPackets.add(newLeaderQP);

leader.waitForNewLeaderAck(getSid(), qp.getZxid());
```
然后阻塞在waitForNewLeaderAck,等待大多数节点同步完成.

当大多数节点同步完成之后发送UPTODATE包,告诉FOLLOWER可以使用同步的数据了.

接下来LearnerHandler开始根据FOLLOWER传来的包进行判断根据不同的类型进行不同的处理,开始进入原子广播阶段:
```java
case Leader.ACK:
    leader.processAck(this.sid, qp.getZxid(), sock.getLocalSocketAddress());
case Leader.PING:
    leader.zk.touch(sess, to);
case Leader.REQUEST:
    leader.zk.submitLearnerRequest(si);
```

对于follower也是一样的,如下:
```java
QuorumPacket qp = new QuorumPacket();
while (this.isRunning()) {
    readPacket(qp);
    processPacket(qp);
}
```
开始不断接收leader传来的proposal和commit请求.

在leader的主线程里,等待完成newLeader之后,开始准备接收客户端和follower的请求:
```java
startZkServer();
// 判断心跳超时时间内是否与绝大多数follower保持连接
```
startZkServer开始建立requestProcessor链,它用来处理用户的请求,这个我们之后在原子广播中详述.

## 总结
下面是流程图:

![avatar](/static/img/成员发现&数据同步.png)

