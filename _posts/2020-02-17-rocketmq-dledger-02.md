---
layout: post
title:  "Dledger中Leader和Follower流程详解"
date:   2020-02-17 15:20:00 +0700
categories: [rocketmq,raft]
---

## 学习目标
搞清楚Dledger中Leader和Follower的流程处理,重点关注心跳包的处理,因为心跳包是维持集群内一致性的重要手段之一,通过对心跳包的处理我们可以更加全面的了解Raft算法在数据不一致时的状态流转过程.

## 源码分析

首先来看Leader的状态维护流程

### Leader
首先看maintainAsLeader方法:

```java
    private void maintainAsLeader() throws Exception {
        if (DLedgerUtils.elapsed(lastSendHeartBeatTime) > heartBeatTimeIntervalMs) {
            long term;
            String leaderId;
            synchronized (memberState) {
                if (!memberState.isLeader()) {
                    //stop sending
                    return;
                }
                term = memberState.currTerm();
                leaderId = memberState.getLeaderId();
                lastSendHeartBeatTime = System.currentTimeMillis();
            }
            sendHeartbeats(term, leaderId); // 发送心跳
        }
    }
```
代码很简单, 我们主要关注sendHeartbeats方法,下面是发送请求的部分代码:
```java
for (String id : memberState.getPeerMap().keySet()) {
    if (memberState.getSelfId().equals(id)) {
        continue;
    }
    /**
    构造请求, 并且使用dLedgerRpcService发送请求.
    */
    HeartBeatRequest heartBeatRequest = new HeartBeatRequest();
    heartBeatRequest.setGroup(memberState.getGroup());
    heartBeatRequest.setLocalId(memberState.getSelfId());
    heartBeatRequest.setRemoteId(id);
    heartBeatRequest.setLeaderId(leaderId);
    heartBeatRequest.setTerm(term);
    CompletableFuture<HeartBeatResponse> future = dLedgerRpcService.heartBeat(heartBeatRequest);
    future.whenComplete((HeartBeatResponse x, Throwable ex) -> {
        try {

            if (ex != null) {
                throw ex;
            }
            switch (DLedgerResponseCode.valueOf(x.getCode())) {
                case SUCCESS:
                    succNum.incrementAndGet(); // 成功, 增加succNum
                    break;
                case EXPIRED_TERM:
                    maxTerm.set(x.getTerm()); // 存在更大的term
                    break;
                case INCONSISTENT_LEADER:
                    inconsistLeader.compareAndSet(false, true); // 不一致的leader
                    break;
                case TERM_NOT_READY:
                    notReadyNum.incrementAndGet(); // term比leader小
                    break;
                default:
                    break;
            }
            if (memberState.isQuorum(succNum.get())
                || memberState.isQuorum(succNum.get() + notReadyNum.get())) {
                beatLatch.countDown();
            }
        } catch (Throwable t) {
            logger.error("Parse heartbeat response failed", t);
        } finally {
            allNum.incrementAndGet();
            if (allNum.get() == memberState.peerSize()) {
                beatLatch.countDown();
            }
        }
    });
}
```
1. 构造请求, 并且使用dLedgerRpcService发送请求.
2. 根据请求的不同结果, 进行不同的计数器累加, 或者标识位的设置.
    1) SUCCESS, 增加succNum计数器
    2) EXPIRED_TERM, 说明存在更大的term,将该term设置到maxterm.
    3) INCONSISTENT_LEADER, 集群中存在另一个leader
    4) TERM_NOT_READY, term比leader小.


对心跳结果进行处理
```java
if (memberState.isQuorum(succNum.get())) {
    lastSuccHeartBeatTime = System.currentTimeMillis();
} else {
    if (memberState.isQuorum(succNum.get() + notReadyNum.get())) {
        lastSendHeartBeatTime = -1;
    } else if (maxTerm.get() > term) {
        changeRoleToCandidate(maxTerm.get());
    } else if (inconsistLeader.get()) {
        changeRoleToCandidate(term);
    } else if (DLedgerUtils.elapsed(lastSuccHeartBeatTime) > maxHeartBeatLeak * heartBeatTimeIntervalMs) {
        changeRoleToCandidate(term);
    }
}
```
1. 如果succNum占半数以上, 那么设置lastSuccHeartBeatTime, 返回, 这是正常情况.

    否则对异常情况进行判断.

2. 如果term相同的 + term较小的占半数以上, 那么设置`lastSendHeartBeatTime = -1`, 即立即开始下一轮心跳包的发送.
3. 如果存在更大的term或者集群中存在另一个leader, 那么转为candidate.

4. 如果距离发送上一个心跳包的间隔时间超过了`maxHeartBeatLeak * heartBeatTimeIntervalMs`,那么此时转变为Candidate.(为什么要转为Candidate呢,在之后的follower篇,我们可以看到当follower超过`maxHeartBeatLeak * heartBeatTimeIntervalMs`没有收到心跳包时,会转为Candidate,此时大概率自己的term已经过期,所以转为Candidate较为合适).

### Follower

我们首先来看Follower如何处理心跳包.
```java
if (request.getTerm() < memberState.currTerm()) {
    return CompletableFuture.completedFuture(new HeartBeatResponse().term(memberState.currTerm()).code(DLedgerResponseCode.EXPIRED_TERM.getCode()));
} else if (request.getTerm() == memberState.currTerm()) {
    if (request.getLeaderId().equals(memberState.getLeaderId())) {
        lastLeaderHeartBeatTime = System.currentTimeMillis();
        return CompletableFuture.completedFuture(new HeartBeatResponse());
    }
}
```

1. 如果leader的term小于自己的那么返回EXPIRED_TERM.

2. 如果相等, 那么返回接收成功.

这里是一次不加锁的fast判断, 如果不满足这里的分支, 那么会进入下面的加锁逻辑中.

```java
synchronized (memberState) {
    if (request.getTerm() < memberState.currTerm()) {
        return CompletableFuture.completedFuture(new HeartBeatResponse().term(memberState.currTerm()).code(DLedgerResponseCode.EXPIRED_TERM.getCode()));
    } else if (request.getTerm() == memberState.currTerm()) {
        if (memberState.getLeaderId() == null) {
            changeRoleToFollower(request.getTerm(), request.getLeaderId());
            return CompletableFuture.completedFuture(new HeartBeatResponse());
        } else if (request.getLeaderId().equals(memberState.getLeaderId())) {
            lastLeaderHeartBeatTime = System.currentTimeMillis();
            return CompletableFuture.completedFuture(new HeartBeatResponse());
        } else {
            //this should not happen, but if happened
            logger.error("[{}][BUG] currTerm {} has leader {}, but received leader {}", memberState.getSelfId(), memberState.currTerm(), memberState.getLeaderId(), request.getLeaderId());
            return CompletableFuture.completedFuture(new HeartBeatResponse().code(DLedgerResponseCode.INCONSISTENT_LEADER.getCode()));
        }
    } else {
        //To make it simple, for larger term, do not change to follower immediately
        //first change to candidate, and notify the state-maintainer thread
        changeRoleToCandidate(request.getTerm());
        needIncreaseTermImmediately = true;
        return CompletableFuture.completedFuture(new HeartBeatResponse().code(DLedgerResponseCode.TERM_NOT_READY.getCode()));
    }
}
```
1. 如果leader的term小于自己的那么返回EXPIRED_TERM.

2. 如果term相同, 那么判断leaderId, local的leaderId为null或者相同时, 会返回成功否则返回`INCONSISTENT_LEADER`.

3. 如果local的term比较小, 那么此时返回TERM_NOT_READY, 不直接转为Follower,而是转为Candidate, 等待leader再次发送心跳包, 进入上面leaderId为null的逻辑, 转为follower.

maintainAsFollower

```java
private void maintainAsFollower() {
    if (DLedgerUtils.elapsed(lastLeaderHeartBeatTime) > 2 * heartBeatTimeIntervalMs) {
        synchronized (memberState) {
            if (memberState.isFollower() && (DLedgerUtils.elapsed(lastLeaderHeartBeatTime) > maxHeartBeatLeak * heartBeatTimeIntervalMs)) {
                logger.info("[{}][HeartBeatTimeOut] lastLeaderHeartBeatTime: {} heartBeatTimeIntervalMs: {} lastLeader={}", memberState.getSelfId(), new Timestamp(lastLeaderHeartBeatTime), heartBeatTimeIntervalMs, memberState.getLeaderId());
                changeRoleToCandidate(memberState.currTerm());
            }
        }
    }
}
```

这里follower的逻辑非常简单, 判断距上次接收心跳包的时间是否超过`maxHeartBeatLeak * heartBeatTimeIntervalMs`, 如果超过, 那么判断leader不可用, 转为Candidate, 进行选举.

