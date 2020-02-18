---
layout: post
title:  "Dledger中Candidate选举流程"
date:   2020-01-08 22:20:00 +0700
categories: [rocketmq,raft]
---

## 学习目标
搞清楚Dledger中Candidate的选举流程, 并且与Raft算法原本的步骤进行比较, 了解其中的不同之处以及原因

## 源码分析
流程的入口在`DLedgerLeaderElector#maintainAsCandidate`方法处:

```java
if (System.currentTimeMillis() < nextTimeToRequestVote && !needIncreaseTermImmediately) {
    return;
}
```
首先判断nextTimeToRequestVote和needIncreaseTermImmediately(这两个值的设置在后文中有解释), 尚不需要请求投票，直接返回。

```java
long term;
long ledgerEndTerm;
long ledgerEndIndex;
```
term: 请求投票时使用的term
ledgerEndTerm: 当前log的最新term
ledgerEndIndex：当前log的最新index

```java
synchronized (memberState) {
    if (!memberState.isCandidate()) {
        return;
    }
    if (lastParseResult == VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT || needIncreaseTermImmediately) {
        long prevTerm = memberState.currTerm();
        term = memberState.nextTerm();
        logger.info("{}_[INCREASE_TERM] from {} to {}", memberState.getSelfId(), prevTerm, term);
        lastParseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
    } else {
        term = memberState.currTerm();
    }
    ledgerEndIndex = memberState.getLedgerEndIndex();
    ledgerEndTerm = memberState.getLedgerEndTerm();
}
```
这段代码的作用是计算出当前请求投票的term，当lastParseResult为WAIT_TO_VOTE_NEXT或者needIncreaseTermImmediately为true时，会将当前term自增1，否则为当前的term。
这里的lastParseResult和needIncreaseTermImmediately在后面有详细说明。

```java
long startVoteTimeMs = System.currentTimeMillis();
final List<CompletableFuture<VoteResponse>> quorumVoteResponses = voteForQuorumResponses(term, ledgerEndTerm, ledgerEndIndex);
```
接着进入voteForQuorumResponses方法开始给集群内节点发送投票请求。
```java
private List<CompletableFuture<VoteResponse>> voteForQuorumResponses(long term, long ledgerEndTerm,
    long ledgerEndIndex) throws Exception {
    List<CompletableFuture<VoteResponse>> responses = new ArrayList<>();
    for (String id : memberState.getPeerMap().keySet()) {
        VoteRequest voteRequest = new VoteRequest();
        voteRequest.setGroup(memberState.getGroup());
        voteRequest.setLedgerEndIndex(ledgerEndIndex);
        voteRequest.setLedgerEndTerm(ledgerEndTerm);
        voteRequest.setLeaderId(memberState.getSelfId());
        voteRequest.setTerm(term);
        voteRequest.setRemoteId(id);
        CompletableFuture<VoteResponse> voteResponse;
        if (memberState.getSelfId().equals(id)) {
            voteResponse = handleVote(voteRequest, true);
        } else {
            //async
            voteResponse = dLedgerRpcService.vote(voteRequest);
        }
        responses.add(voteResponse);

    }
    return responses;
}
```
可以看到，通过调用`handleVote`方法给自己投票，通过调用`dLedgerRpcService.vote`方法向远程节点请求。

## 处理投票请求
通过对比请求的term与自己的term的大小以及log的新旧来进行投票与否。
如果请求term小，返回REJECT_EXPIRED_VOTE_TERM。
如果term相等但是当前已经投票给某个节点，那么返回REJECT_ALREADY_VOTED。或者已经有正在生效的leader那么返回REJECT_ALREADY__HAS_LEADER。
如果自己的term比较小，那么先转为Candidate，并且将needIncreaseTermImmediately设置为true，返回REJECT_TERM_NOT_READY。设置needIncreaseTermImmediately为true，会影响到之后Candidate进行选举时计算term（见上面），而且可以想见计算的term一定不大于当前生效leader的term，而且极有可能在投票后会收到来自leader的心跳转为follower（见下方对nextTimeToRequestVote的分析）。
代码如下：
```java
if (request.getTerm() < memberState.currTerm()) {
    return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.REJECT_EXPIRED_VOTE_TERM));
} else if (request.getTerm() == memberState.currTerm()) {
    if (memberState.currVoteFor() == null) {
        //let it go
    } else if (memberState.currVoteFor().equals(request.getLeaderId())) {
        //repeat just let it go
    } else {
        if (memberState.getLeaderId() != null) {
            return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.REJECT_ALREADY__HAS_LEADER));
        } else {
            return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.REJECT_ALREADY_VOTED));
        }
    }
} else {
    //stepped down by larger term
    changeRoleToCandidate(request.getTerm());
    needIncreaseTermImmediately = true;
    //only can handleVote when the term is consistent
    return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.REJECT_TERM_NOT_READY));
}
```

以上情况中只有term相等且当前没有投票给任何人才会进行下一步log的比较。
首先判断LedgerEndTerm，如果LedgerEndTerm相等则接下来判断LedgerEndIndex的是否是request比较大，如果是那么接受这个投票请求，否则拒绝。代码如下：
```java
if (request.getLedgerEndTerm() < memberState.getLedgerEndTerm()) {
    return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.REJECT_EXPIRED_LEDGER_TERM));
} else if (request.getLedgerEndTerm() == memberState.getLedgerEndTerm() && request.getLedgerEndIndex() < memberState.getLedgerEndIndex()) {
    return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.REJECT_SMALL_LEDGER_END_INDEX));
}

if (request.getTerm() < memberState.getLedgerEndTerm()) {
    return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.getLedgerEndTerm()).voteResult(VoteResponse.RESULT.REJECT_TERM_SMALL_THAN_LEDGER));
}

memberState.setCurrVoteFor(request.getLeaderId());
return CompletableFuture.completedFuture(new VoteResponse(request).term(memberState.currTerm()).voteResult(VoteResponse.RESULT.ACCEPT);
```

## 对投票结果的处理
```java
CountDownLatch voteLatch = new CountDownLatch(1);
        for (CompletableFuture<VoteResponse> future : quorumVoteResponses) {
    // ...
}
```
这里使用了CountDownLatch，当一些条件满足时释放该闭锁使流程前进，否则阻塞直到超时时间到达。

对每个投票结果的future，对以下几种投票结果进行处理：
```java
public enum RESULT {
    UNKNOWN,
    ACCEPT, // 接受投票
    REJECT_UNKNOWN_LEADER, // 发起投票的机器不在集群中
    REJECT_UNEXPECTED_LEADER, // 
    REJECT_EXPIRED_VOTE_TERM, // 请求的term过期
    REJECT_ALREADY_VOTED, // 已投票给某个节点
    REJECT_ALREADY__HAS_LEADER, // 已经存在有效的leader，且该leader是其他机器
    REJECT_TERM_NOT_READY, // 本节点term过小（接下来转为Candidate）
    REJECT_TERM_SMALL_THAN_LEDGER, // 请求的term小于本机log的term
    REJECT_EXPIRED_LEDGER_TERM, // 请求log的term小于本机log的term
    REJECT_SMALL_LEDGER_END_INDEX;// 请求log的index小于本机log的index
}
```

- 对于每一个ACCEPT，累加acceptedNum
- 对于REJECT_ALREADY__HAS_LEADER，alreadyHasLeader设置为true
- 对于REJECT_TERM_SMALL_THAN_LEDGER，REJECT_EXPIRED_VOTE_TERM，设置knownMaxTermInGroup（最大已知的term）
- 对于REJECT_EXPIRED_LEDGER_TERM、REJECT_SMALL_LEDGER_END_INDEX，累加biggerLedgerNum
- 对于REJECT_TERM_NOT_READY：累加notReadyTermNum

```java
switch (x.getVoteResult()) {
    case ACCEPT:
        acceptedNum.incrementAndGet();
        break;
    case REJECT_ALREADY_VOTED:
        break;
    case REJECT_ALREADY__HAS_LEADER:
        alreadyHasLeader.compareAndSet(false, true);
        break;
    case REJECT_TERM_SMALL_THAN_LEDGER:
    case REJECT_EXPIRED_VOTE_TERM:
        if (x.getTerm() > knownMaxTermInGroup.get()) {
            knownMaxTermInGroup.set(x.getTerm());
        }
        break;
    case REJECT_EXPIRED_LEDGER_TERM:
    case REJECT_SMALL_LEDGER_END_INDEX:
        biggerLedgerNum.incrementAndGet();
        break;
    case REJECT_TERM_NOT_READY:
        notReadyTermNum.incrementAndGet();
        break;
    default:
        break;

}
```

这些变量在接下来Candidate的下一步操作中起到控制作用。

如果结果返回过程中出现以下情况：

1. alreadyHasLeader为true，即集群中已经存在有效leader且term相等
2. acceptedNum占半数以上
3. acceptedNum + notReadyTermNum占半数以上

voteLatch立即countdown。

接下来开始解析投票结果, 如果存在已知更大的term那么就再次以该term进行选举:
```java
if (knownMaxTermInGroup.get() > term) {
    parseResult = VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT;
    nextTimeToRequestVote = getNextTimeToRequestVote();
    changeRoleToCandidate(knownMaxTermInGroup.get());
}
```

否则如果集群中已经有了一个同term的leader,那么继续投票(term增加),但是下一次投票的时间变长,
极有可能在间隔期间接收到来自leader的heartbeat.其中 heartBeatTimeIntervalMs 为一次心跳间隔时间， maxHeartBeatLeak 为 允许最大丢失的心跳
```java
else if (alreadyHasLeader.get()) {
    parseResult = VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT;
    nextTimeToRequestVote = getNextTimeToRequestVote() + heartBeatTimeIntervalMs * maxHeartBeatLeak;
}
```

否则如果在选举超时期间内收到的回复不到一半,那么进行重新投票(term不变)
```java
else if (!memberState.isQuorum(validNum.get())) {
    parseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
    nextTimeToRequestVote = getNextTimeToRequestVote();
}
```

如果收到的同意投票数超过一半,那么转变为leader
```java
 else if (memberState.isQuorum(acceptedNum.get())) {
    parseResult = VoteResponse.ParseResult.PASSED;
} 
if (parseResult == VoteResponse.ParseResult.PASSED) {
    logger.info("[{}] [VOTE_RESULT] has been elected to be the leader in term {}", memberState.getSelfId(), term);
    changeRoleToLeader(term);
}
```

否则如果此时同意投票的和NOT_READY(即term数小于请求term的节点会转变为Candidate)的超过半数,
那么立即进行下一轮投票(因为此时极有可能赢得选举)
```java
else if (memberState.isQuorum(acceptedNum.get() + notReadyTermNum.get())) {
    parseResult = VoteResponse.ParseResult.REVOTE_IMMEDIATELY;
}
```

否则如果此时同意的和因为log过时拒绝的回复加在一起超过半数的话,那么重新投票
```java
else if (memberState.isQuorum(acceptedNum.get() + biggerLedgerNum.get())) {
    parseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
    nextTimeToRequestVote = getNextTimeToRequestVote();
}
```

其他情况:
```java
else {
    parseResult = VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT;
    nextTimeToRequestVote = getNextTimeToRequestVote();
}
```

## 转变为Leader
调用memberstate的changeToLeader方法转变为leader:
```java
public synchronized void changeToLeader(long term) {
    PreConditions.check(currTerm == term, DLedgerResponseCode.ILLEGAL_MEMBER_STATE, "%d != %d", currTerm, term);
    this.role = LEADER;
    this.leaderId = selfId;
}
```
在接下来StateMaintainer的循环中就会根据当memberState的role进行leader的流程操作.

## 与原本的Raft算法的不同之处

1. 当节点接收到一个term比自己大的Candidate节点A的vote请求时不会直接将票投给它,而是返回REJECT_TERM_NOT_READY,
接着自己变成Candidate立即自增term发起投票,但是由于此时term必然不可能比原先被请求投票的那个节点A的大, 所以极有可能
收到再次来自A的投票请求.







