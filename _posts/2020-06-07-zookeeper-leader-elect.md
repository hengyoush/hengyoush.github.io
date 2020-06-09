---
layout: post
title:  "Zookeeper Leader选举流程源码解读"
date:   2020-06-01 15:20:00 +0700
categories: [zookeeper]
---

## 学习目标
搞清楚Zookeeper是如何实现ZAB协议的领导者选举的.

## 源码分析
废话不多说, 让我进入源码分析的环节.
发起选举的节点状态为LOOKING状态,我们看org.apache.zookeeper.server.quorum.QuorumPeer#run方法:
```java
while (running) {
                switch (getPeerState()) {
                case LOOKING:
                    LOG.info("LOOKING");

                    if (Boolean.getBoolean("readonlymode.enabled")) {...} else {
                        try {
                           reconfigFlagClear();
                           ...
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        }                        
                    }
                    break;
                case FLOWING: ...
                case ...
```
这里我们只关注LOOKING的case,其中注意`makeLEStrategy().lookForLeader()`,
makeLEStrategy方法选择了一种领导选举算法的实现,lookForLeader是真正进行选举的代码,这里makeLEStrategy
方法返回的就是org.apache.zookeeper.server.quorum.FastLeaderElection, 接下来让我们直接分析FastLeaderElection
的lookForLeader方法吧.

```java
HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();
HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();
int notTimeout = finalizeWait;
synchronized(this){
    logicalclock.incrementAndGet();
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}

sendNotifications();
```

首先创建了两个Map, recvset和outofelection.

这里logicalclock自增1,然后更新自己的选票,这里logicalclock的初始值是-1, 用来表示当前的epoch,
但是接下来将会被自己的epoch更新.

getInitId返回myid, getInitLastLoggedZxid返回当前dataTree中最大的zxid, getPeerEpoch返回节点当前epoch.
这里其实是将选票投给自己.然后通过sendNotifications方法将vote发给集群内部所有节点.

```java
for (long sid : self.getCurrentAndNextConfigVoters()) {
    QuorumVerifier qv = self.getQuorumVerifier();
    ToSend notmsg = new ToSend(ToSend.mType.notification, // type
            proposedLeader, // 投给谁? 这里是myid
            proposedZxid, // zxid
            logicalclock.get(), // 本地投票的epoch
            QuorumPeer.ServerState.LOOKING,
            sid, // 对面的id
            proposedEpoch, // 当前投的leader的epoch
            qv.toString().getBytes());
    sendqueue.offer(notmsg); // 放入发送队列中, 另外一个WorkSender线程会去发送
}
```
sendNotifications方法会将投票信息广播,当然也会发给自己,对于发给自己的投票直接将其放入RecvQueue中,
同时对于其他节点发回来的响应也放在RecvQueue中,等待lookForLeadder取.
在lookForLeadder的下一步就是不断从RecvQueue中取出响应,进行处理:
```java
while ((self.getPeerState() == ServerState.LOOKING) && (!stop)){
    Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);
    if (validVoter(n.sid) && validVoter(n.leader)) {
        switch (n.state) {
            case LOOKING:
            case FOLLOWING AND LEADING:
        }
    }
}

```
可以看到分别对当前发送投票的节点当前的状态做了处理,当其他节点正在选举和已经确定了Leader两种情况,
我们首先看其他节点也正在选举的情况.

我们对这种情况分两步来看,第一步对一个投票进行处理
```java
if (n.electionEpoch > logicalclock.get()) {
    logicalclock.set(n.electionEpoch); // 设置新的投票轮次
    recvset.clear(); // 当前进行的投票过时了, 将recvset清除
    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
            getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
        updateProposal(n.leader, n.zxid, n.peerEpoch);
    } else {
        updateProposal(getInitId(),
                getInitLastLoggedZxid(),
                getPeerEpoch());
    }
    sendNotifications();
} else if (n.electionEpoch < logicalclock.get()) {
    if(LOG.isDebugEnabled()){
        LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                + Long.toHexString(n.electionEpoch)
                + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
    }
    break; // break switch
} else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
        proposedLeader, proposedZxid, proposedEpoch)) {
    updateProposal(n.leader, n.zxid, n.peerEpoch);
    sendNotifications();
}
recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
```
首先明确下投票规则:

1. 判断epoch, epoch大的获胜
2. epoch相同的情况, zxid大的获胜
3. 以上两个都相同的情况, myid大的获胜

再看上面的代码, 首先判断投票节点的投票时钟和自己的大小, 如果投票节点获胜, 那么再进行votePK,
如果投票节点获胜那么更新自己的选票并且将新的vote发送出去.
对于投票时钟相同的情况, 进行votePK, 如果投票胜出那么更新自己的投票,发送新的投票到各个节点.

代码如下:
```java
    protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
      return ((newEpoch > curEpoch) ||
                ((newEpoch == curEpoch) &&
                ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
    }
```

最后将接收到的投票放入recvset中:`recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch))`.

每接收到投票, 都要判断一次当前轮次是否有人已经获胜, 代码如下:
```java
if (termPredicate(recvset,
        new Vote(proposedLeader, proposedZxid,
                logicalclock.get(), proposedEpoch))) {
    // 此处在确定在finalizeWait时间内没有其他获胜者存在,
    // 因为zk在退出follower之后会提交所有没提交的txn, 所以不会
    // 像raft那样确保大多数节点具有最新提交的事务.
    while((n = recvqueue.poll(finalizeWait,
            TimeUnit.MILLISECONDS)) != null){
        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                proposedLeader, proposedZxid, proposedEpoch)){
            recvqueue.put(n);
            break;
        }
    }
    if (n == null) {
        self.setPeerState((proposedLeader == self.getId()) ?
                ServerState.LEADING: learningState());
        Vote endVote = new Vote(proposedLeader,
                proposedZxid, logicalclock.get(), 
                proposedEpoch);
        leaveInstance(endVote);
        return endVote;
    }
}
```
termPredicate方法判断当前节点选中的节点是否已经获胜(即集群中大多数的选票), 如果是那么将自己的状态
设置为LEADER或者FOLLOWER, 结束选举.

上面是从其他LOOKING节点上接收的选票,如果是LEADER或者是FOLLOWER状态的节点,那么会如何处理呢?
```java
if(n.electionEpoch == logicalclock.get()){
    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
    if(termPredicate(recvset, new Vote(n.version, n.leader,
                    n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                    && checkLeader(outofelection, n.leader, n.electionEpoch)) {
        self.setPeerState((n.leader == self.getId()) ?
                ServerState.LEADING: learningState());
        Vote endVote = new Vote(n.leader, 
                n.zxid, n.electionEpoch, n.peerEpoch);
        leaveInstance(endVote);
        return endVote;
    }
}

outofelection.put(n.sid, new Vote(n.version, n.leader, 
        n.zxid, n.electionEpoch, n.peerEpoch, n.state));
if (termPredicate(outofelection, new Vote(n.version, n.leader,
        n.zxid, n.electionEpoch, n.peerEpoch, n.state))
        && checkLeader(outofelection, n.leader, n.electionEpoch)) {
    synchronized(this){
        logicalclock.set(n.electionEpoch);
        self.setPeerState((n.leader == self.getId()) ?
                ServerState.LEADING: learningState());
    }
    Vote endVote = new Vote(n.leader, n.zxid, 
            n.electionEpoch, n.peerEpoch);
    leaveInstance(endVote);
    return endVote;
}
```
1. 当前LEADER与自己属于同一轮投票, 那么判断当recvset中该节点是否已经获得了大多数选票,
如果是, 那么设置自己为LEADER或者FOLLOWING, 退出, 否则继续接收投票.

2. 否则, 将投票放在outofelection中, 同样对outofelection进行LEADER有效判断, 如果有效
那么设置状态返回.

这里的checkLeader方法可以看一下:
```java
    protected boolean checkLeader(
            Map<Long, Vote> votes,
            long leader,
            long electionEpoch){

        boolean predicate = true;

        if(leader != self.getId()){
            if(votes.get(leader) == null) predicate = false;
            else if(votes.get(leader).getState() != ServerState.LEADING) predicate = false;
        } else if(logicalclock.get() != electionEpoch) {
            predicate = false;
        }

        return predicate;
    }
```
我们要确定一个LEADER是否合法有以下要点:
1. 该LEADER必须获取了足够的选票
2. 该LEADER必须是存活的

第一点已经被保证(termPredicate), 那么只需要确认第二点.
如果别人认为我是LEADER,说明其他节点已经达成共识,我是LEADER,那么第二点自动满足.
如果认为别人是LEADER,那么需要防止该LEADER获取足够多选票之后宕机, 但是它的选票已经被
大多数节点持有并且不断发送, 就会陷入一种一台宕掉的节点获胜的情况, 为了防止这一点, 对有没有从
这个LEADER上拿到投票这一点做了判断并且是自认为是LEADER之后的vote.

## 总结
下面是流程图:

![avatar](/static/img/选举流程.png)

