---
layout: post
title:  "Dledger中日志追加流程详解"
date:   2020-02-18 15:20:00 +0700
categories: [rocketmq,raft]
---

## 学习目标
搞清楚Dledger中日志传播的流程,重点分析EntryDispatcher、EntryHandler、QuorumAckChecker三个线程.

## 源码分析

在上一篇中讲到waitAck方法会等待超过半数以上的follower接收日志后返回客户端的append请求. 
而对follower日志的分发则是通过DLedgerEntryPusher的dispatcher线程进行转发.

我们首先对DLedgerEntryPusher进行分析, 先看它的重要属性如下:
```java
private Map<Long, ConcurrentMap<String, Long>> peerWaterMarksByTerm = new ConcurrentHashMap<>();
private Map<Long, ConcurrentMap<Long, TimeoutFuture<AppendEntryResponse>>> pendingAppendResponsesByTerm = new ConcurrentHashMap<>();

private EntryHandler entryHandler = new EntryHandler(logger);

private QuorumAckChecker quorumAckChecker = new QuorumAckChecker(logger);

private Map<String, EntryDispatcher> dispatcherMap = new HashMap<>();
```
peerWaterMarksByTerm: 每个节点基于term投票轮次的水位线标记.
pendingAppendResponsesByTerm: 每个日志index对应的appendFuture.
entryHandler: 线程, 可用于follower, 用来处理leader发来的日志.
quorumAckChecker: 线程, 用于leader, 用来处理日志处理的投票结果, 判断是否commit.
dispatcherMap: 线程, 用于leader, key是peerId, EntryDispatcher就是对应于每个节点的分发器.

### EntryDispatcher

我们先查看它的doWork方法:
```java
public void doWork() {
    try {
        if (!checkAndFreshState()) { // @1
            waitForRunning(1);
            return;
        }

        if (type.get() == PushEntryRequest.Type.APPEND) {
            doAppend(); // @2
        } else {
            doCompare(); // @3
        }
        waitForRunning(1);
    } catch (Throwable t) {
    }
}
```

checkAndFreshState方法, 只有返回true才会进行分发.
```java
private boolean checkAndFreshState() {
    if (!memberState.isLeader()) { // @1
        return false;
    }
    if (term != memberState.currTerm() || leaderId == null || !leaderId.equals(memberState.getLeaderId())) { // @2
        synchronized (memberState) {
            if (!memberState.isLeader()) {
                return false;
            }
            PreConditions.check(memberState.getSelfId().equals(memberState.getLeaderId()), DLedgerResponseCode.UNKNOWN);
            term = memberState.currTerm();
            leaderId = memberState.getSelfId();
            changeState(-1, PushEntryRequest.Type.COMPARE);
        }
    }
    return true;
}
```
1: 判断当前是不是leader, 如果不是, 返回false
2: 如果是leader那么判断是否leader未设置(第一次进行分发)或者是leaderId不同(重新选举).如果是上述情况的某一种,那么会将dispatcher内的数学进行更新,并且转换为Compare模式.(转换为新的leader之后需要与follower进行日志比较确保没有不一致的日志产生)

changeState方法:
```java
private synchronized void changeState(long index, PushEntryRequest.Type target) {
    switch (target) {
        case APPEND:
            compareIndex = -1;
            updatePeerWaterMark(term, peerId, index);
            quorumAckChecker.wakeup();
            writeIndex = index + 1;
            break;
        case COMPARE:
            if (this.type.compareAndSet(PushEntryRequest.Type.APPEND, PushEntryRequest.Type.COMPARE)) {
                compareIndex = -1;
                pendingMap.clear();
            }
            break;
        case TRUNCATE:
            compareIndex = -1;
            break;
        default:
            break;
    }
    type.set(target);
}
```

doAppend方法
```java
private void doAppend() throws Exception {
    while (true) {
        if (!checkAndFreshState()) { // @1
            break;
        }
        if (type.get() != PushEntryRequest.Type.APPEND) { // @2
            break;
        }
        if (writeIndex > dLedgerStore.getLedgerEndIndex()) { // @3
            doCommit();
            doCheckAppendResponse();
            break;
        }
        if (pendingMap.size() >= maxPendingSize || (DLedgerUtils.elapsed(lastCheckLeakTimeMs) > 1000)) { // @4
            long peerWaterMark = getPeerWaterMark(term, peerId);
            for (Long index : pendingMap.keySet()) {
                if (index < peerWaterMark) {
                    pendingMap.remove(index);
                }
            }
            lastCheckLeakTimeMs = System.currentTimeMillis();
        }
        if (pendingMap.size() >= maxPendingSize) { // @5
            doCheckAppendResponse();
            break;
        }
        doAppendInner(writeIndex); // @6
        writeIndex++;
    }
}
```
1. 检查当前状态是否是leader, 如果不是直接退出
2. 检查是不是APPEND模式, 如果不是直接退出
3. 判断writeIndex是否超出ledgerEndIndex, 如果超过,那么发送commit请求,并且尝试发送超时的append请求.出现此情况的原因是pendingMap(指的是Pusher的Map不是Dispatcher的)超过阈值(1w),此时会阻止追加, 在这个时候会出现dispatcher分发的速度超过leader的pending请求处理的速度)
4. 如果此时该节点的pending请求(已经发送还未收到响应,但此时可能存在过期的记录)超过一个阈值(1000)或者距离上一次检查pendingMap泄漏的时间超过了1s,那么会进行pendingMap的清理,具体逻辑是根据当前peer的水位线清除之前的pendingMap记录.
5. 如果此时pendingMap仍然大于阈值(此时是真正的没有收到响应的记录数), 那么此时会尝试重新发送最后一个请求.
6. 发送新的append请求

```java
private void doAppendInner(long index) throws Exception {
    DLedgerEntry entry = dLedgerStore.get(index);
    PreConditions.check(entry != null, DLedgerResponseCode.UNKNOWN, "writeIndex=%d", index);
    checkQuotaAndWait(entry); // @1
    PushEntryRequest request = buildPushRequest(entry, PushEntryRequest.Type.APPEND); // @2
    CompletableFuture<PushEntryResponse> responseFuture = dLedgerRpcService.push(request); // @3
    pendingMap.put(index, System.currentTimeMillis()); // @4
    responseFuture.whenComplete((x, ex) -> {
        try {
            PreConditions.check(ex == null, DLedgerResponseCode.UNKNOWN);
            DLedgerResponseCode responseCode = DLedgerResponseCode.valueOf(x.getCode());
            switch (responseCode) {
                case SUCCESS: // @5
                    pendingMap.remove(x.getIndex());
                    updatePeerWaterMark(x.getTerm(), peerId, x.getIndex());
                    quorumAckChecker.wakeup();
                    break;
                case INCONSISTENT_STATE: // @6
                    logger.info("[Push-{}]Get INCONSISTENT_STATE when push index={} term={}", peerId, x.getIndex(), x.getTerm());
                    changeState(-1, PushEntryRequest.Type.COMPARE);
                    break;
                default:
                    logger.warn("[Push-{}]Get error response code {} {}", peerId, responseCode, x.baseInfo());
                    break;
            }
        } catch (Throwable t) {
            logger.error("", t);
        }
    });
    lastPushCommitTimeMs = System.currentTimeMillis();
}
```
1 检查配额

- 首先触发条件：append 挂起请求数已超过最大允许挂起数；基于文件存储并主从差异超过300m，可通过 peerPushThrottlePoint 配置。
- 每秒追加的日志超过 20m(可通过 peerPushQuota 配置)，则会 sleep 1s中后再追加。

2 构造append请求
```java
private PushEntryRequest buildPushRequest(DLedgerEntry entry, PushEntryRequest.Type target) {
    PushEntryRequest request = new PushEntryRequest();
    request.setGroup(memberState.getGroup());
    request.setRemoteId(peerId);
    request.setLeaderId(leaderId);
    request.setTerm(term);
    request.setEntry(entry);
    request.setType(target);
    request.setCommitIndex(dLedgerStore.getCommittedIndex());
    return request;
}
```

3 使用dLedgerRpcService发送append请求

4 将当前发送日志的index和当前时间记录进pendingMap中.

5 对future的结果进行处理, 如果SUCCESS, 那么移除pendingMap中的记录, 并且更新水位线记录, 唤醒quorumAckChecker线程.

6 如果失败, 那么转换状态为COMPARE, 进行日志的比较重发.

关于follower如何处理append请求, 在之后的EntryHandler中会解释.

doCompare
```java
while (true) {
    if (!checkAndFreshState()) {
        break;
    }
    if (type.get() != PushEntryRequest.Type.COMPARE
        && type.get() != PushEntryRequest.Type.TRUNCATE) {
        break;
    }
    if (compareIndex == -1 && dLedgerStore.getLedgerEndIndex() == -1) {
        break;
    }
    // 对compareIndex进行校验修改
    if (compareIndex == -1) {
        compareIndex = dLedgerStore.getLedgerEndIndex();
    } else if (compareIndex > dLedgerStore.getLedgerEndIndex() || compareIndex < dLedgerStore.getLedgerBeginIndex()) {
        compareIndex = dLedgerStore.getLedgerEndIndex();
    }

    DLedgerEntry entry = dLedgerStore.get(compareIndex); // @1
    PushEntryRequest request = buildPushRequest(entry, PushEntryRequest.Type.COMPARE);
    CompletableFuture<PushEntryResponse> responseFuture = dLedgerRpcService.push(request); // @2
    PushEntryResponse response = responseFuture.get(3, TimeUnit.SECONDS); // @3
    long truncateIndex = -1;

    if (response.getCode() == DLedgerResponseCode.SUCCESS.getCode()) { 
        if (compareIndex == response.getEndIndex()) { // @4
            changeState(compareIndex, PushEntryRequest.Type.APPEND);
            break;
        } else { // @5
            truncateIndex = compareIndex;
        }
    } else if (response.getEndIndex() < dLedgerStore.getLedgerBeginIndex()
        || response.getBeginIndex() > dLedgerStore.getLedgerEndIndex()) { // @6
        truncateIndex = dLedgerStore.getLedgerBeginIndex();
    } else if (compareIndex < response.getBeginIndex()) { // @7
        truncateIndex = dLedgerStore.getLedgerBeginIndex();
    } else if (compareIndex > response.getEndIndex()) { // @8
        compareIndex = response.getEndIndex();
    } else { // @9
        compareIndex--;
    }
    if (compareIndex < dLedgerStore.getLedgerBeginIndex()) { // @10
        truncateIndex = dLedgerStore.getLedgerBeginIndex();
    }
    if (truncateIndex != -1) {
        changeState(truncateIndex, PushEntryRequest.Type.TRUNCATE);
        doTruncate(truncateIndex);
        break;
    }
}
```
1 和 2 :构造compare请求, 发送

3 阻塞获取结果

4 如果返回结果为SUCCESS棒球compareIndex与follower的endIndex相同, 那么转为APPEND.返回结果为SUCCESS说明发送的日志与follower的日志比对成功, 说明只有compareIndex之后的日志需要重发,但是compareIndex与follower的endIndex相同意味着只需要进行APPEND就可以了.

5 如果compareIndex与follower的endIndex不同,那么意味着follower存在脏数据,所以需要进行truncate, leader会将truncateIndex(不包含)之后的日志发送给follower.

6 如果follower的endIndex和beginIndex不在leader的日志范围之内,那么将会发送leader所有的日志,设置truncateIndex为leader的beginIndex.

7 和 8 : 如果compareIndex小于follower的beginIndex,那么说明follower存在着磁盘故障(7很少发生), 如果compareIndex大于follower的endIndex, 那么需要从follwer的endIndex开始比较.

9 这种情况是比较常见的情况,此时compareIndex需要自减, 因为当前比较的结果失败, 需要找到最后一个leader和follower相同的日志Index, 如果找到了, 同时意味着之前所有的日志也都一致, 这是raft算法保证的.

10 如果truncateIndex不为-1,说明需要进行truncate,转为truncate状态,开始进行truncate.

doTruncate
源码如下:
```java
private void doTruncate(long truncateIndex) throws Exception {
    DLedgerEntry truncateEntry = dLedgerStore.get(truncateIndex);
    PushEntryRequest truncateRequest = buildPushRequest(truncateEntry, PushEntryRequest.Type.TRUNCATE);
    PushEntryResponse truncateResponse = dLedgerRpcService.push(truncateRequest).get(3, TimeUnit.SECONDS);
    lastPushCommitTimeMs = System.currentTimeMillis();
    changeState(truncateIndex, PushEntryRequest.Type.APPEND);
}
```
这里没什么好说的,根据truncateIndex构造请求发送即可,发送之后就可以进入APPEND状态.

如上,就是Dispatcher的逻辑, 下面进入与follower关系密切的EntryHandler的代码中, 找到follower上面对APPEND、COMPARE、TRUNCATE等的处理.

### EntryHandler

重要属性:
- `ConcurrentMap<Long, Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>>> writeRequestMap`: 用来根据index存放对应的请求PushEntryRequest和存放结果的future.
- `BlockingQueue<Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>>> compareOrTruncateRequests`: 阻塞队列,存放compare和truncate请求PushEntryRequest和对应的future.

主流程doWork:
```java
public void doWork() {
    try {
        if (!memberState.isFollower()) {
            waitForRunning(1);
            return;
        }
        if (compareOrTruncateRequests.peek() != null) { // @1
            Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> pair = compareOrTruncateRequests.poll();
            PreConditions.check(pair != null, DLedgerResponseCode.UNKNOWN);
            switch (pair.getKey().getType()) { 
                case TRUNCATE:// @2
                    handleDoTruncate(pair.getKey().getEntry().getIndex(), pair.getKey(), pair.getValue());
                    break;
                case COMPARE: // @3
                    handleDoCompare(pair.getKey().getEntry().getIndex(), pair.getKey(), pair.getValue());
                    break;
                case COMMIT: // @4
                    handleDoCommit(pair.getKey().getCommitIndex(), pair.getKey(), pair.getValue());
                    break;
                default:
                    break;
            }
        } else {
            long nextIndex = dLedgerStore.getLedgerEndIndex() + 1; // @5
            Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> pair = writeRequestMap.remove(nextIndex);
            if (pair == null) {
                checkAbnormalFuture(dLedgerStore.getLedgerEndIndex()); // @6
                waitForRunning(1);
                return;
            }
            PushEntryRequest request = pair.getKey();
            handleDoAppend(nextIndex, request, pair.getValue()); // @7
        }
    } catch (Throwable t) {
        DLedgerEntryPusher.logger.error("Error in {}", getName(), t);
        DLedgerUtils.sleep(100);
    }
}
```
1: 如果阻塞队列中存在元素, 那么取出, 根据请求类型分别进行处理.(TRUNCATE和COMPARE和COMMIT)

2: 对TRUNCATE请求进行处理

3: 对COMPARE请求进行处理

4: 对COMMIT请求进行处理

5: 如果阻塞队列中没有元素, 那么查看writeRequestMap中是否存在本地日志下一个要写入的index的日志.

6: 如果不存在, 那么检查是否是leader推送的日志丢失了, 如果是那么需要leader重新发送COMPARE.

7: 处理APPEND请求

我们比较好奇writeRequestMap和compareOrTruncateRequests的数据是从哪里来的呢.答案在EntryHandler的handlePush方法中, 调用路径为:
DLedgerServer#handlePush -> DLedgerEntryPusher#handlePush -> EntryHandler#handlePush

让我们分别看一下上述 2 3 4的源码逻辑.
```java
private CompletableFuture<PushEntryResponse> handleDoTruncate(long truncateIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        long index = dLedgerStore.truncate(request.getEntry(), request.getTerm(), request.getLeaderId());
        PreConditions.check(index == truncateIndex, DLedgerResponseCode.INCONSISTENT_STATE);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
        dLedgerStore.updateCommittedIndex(request.getTerm(), request.getCommitIndex());
    } catch (Throwable t) {
        logger.error("[HandleDoTruncate] truncateIndex={}", truncateIndex, t);
        future.complete(buildResponse(request, DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
    }
    return future;
}
```
truncate逻辑很简单, 根据truncateIndex调用dLedgerStore的truncate方法,进行日志的truncate.

下面看COMPARE的逻辑.
```java
private CompletableFuture<PushEntryResponse> handleDoCompare(long compareIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        DLedgerEntry local = dLedgerStore.get(compareIndex);
        PreConditions.check(request.getEntry().equals(local), DLedgerResponseCode.INCONSISTENT_STATE);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
    } catch (Throwable t) {
        logger.error("[HandleDoCompare] compareIndex={}", compareIndex, t);
        future.complete(buildResponse(request, DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
    }
    return future;
}
```
根据compareIdex取到本地的日志entry与leader发送的entry做比较,如果相同那么返回SUCCESS, 否则返回INCONSISTENT_STATE.
这里和我们之前对dispatcher的分析吻合.

COMMIT的逻辑:
```java
private CompletableFuture<PushEntryResponse> handleDoCommit(long committedIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        dLedgerStore.updateCommittedIndex(request.getTerm(), committedIndex);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
    } catch (Throwable t) {
        logger.error("[HandleDoCommit] committedIndex={}", request.getCommitIndex(), t);
        future.complete(buildResponse(request, DLedgerResponseCode.UNKNOWN.getCode()));
    }
    return future;
}
```
逻辑是否简单,仅仅是更新dLedgerStore的commitedIndex.

APPEND的逻辑如下:
```java
private void handleDoAppend(long writeIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        DLedgerEntry entry = dLedgerStore.appendAsFollower(request.getEntry(), request.getTerm(), request.getLeaderId());
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
        dLedgerStore.updateCommittedIndex(request.getTerm(), request.getCommitIndex());
    } catch (Throwable t) {
        logger.error("[HandleDoWrite] writeIndex={}", writeIndex, t);
        future.complete(buildResponse(request, DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
    }
}
```
使用dLedgerStore进行日志的存储,接着尝试更新leader传来的commitIndex.
主要关注appendAsFollower方法, 但是这里对具体存储的部分我们不进行展开, 具体的我们下篇再详细讲解appendAsFollower的逻辑.

最后,我们分析最为复杂的doWork的代码@6部分,也就是checkAbnormalFuture方法.
```java
private void checkAbnormalFuture(long endIndex) {
    if (DLedgerUtils.elapsed(lastCheckFastForwardTimeMs) < 1000) {
        return;
    }
    lastCheckFastForwardTimeMs  = System.currentTimeMillis();
    if (writeRequestMap.isEmpty()) {
        return;
    }
    long minFastForwardIndex = Long.MAX_VALUE;
    for (Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> pair : writeRequestMap.values()) {
        long index = pair.getKey().getEntry().getIndex();
        //Fall behind
        if (index <= endIndex) {
            try {
                DLedgerEntry local = dLedgerStore.get(index);
                PreConditions.check(pair.getKey().getEntry().equals(local), DLedgerResponseCode.INCONSISTENT_STATE);
                pair.getValue().complete(buildResponse(pair.getKey(), DLedgerResponseCode.SUCCESS.getCode()));
                logger.warn("[PushFallBehind]The leader pushed an entry index={} smaller than current ledgerEndIndex={}, maybe the last ack is missed", index, endIndex);
            } catch (Throwable t) {
                logger.error("[PushFallBehind]The leader pushed an entry index={} smaller than current ledgerEndIndex={}, maybe the last ack is missed", index, endIndex, t);
                pair.getValue().complete(buildResponse(pair.getKey(), DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
            }
            writeRequestMap.remove(index);
            continue;
        }
        //Just OK
        if (index ==  endIndex + 1) {
            //The next entry is coming, just return
            return;
        }
        //Fast forward
        TimeoutFuture<PushEntryResponse> future  = (TimeoutFuture<PushEntryResponse>) pair.getValue();
        if (!future.isTimeOut()) {
            continue;
        }
        if (index < minFastForwardIndex) {
            minFastForwardIndex = index;
        }
    }
    if (minFastForwardIndex == Long.MAX_VALUE) {
        return;
    }
    Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> pair = writeRequestMap.get(minFastForwardIndex);
    if (pair == null) {
        return;
    }
    logger.warn("[PushFastForward] ledgerEndIndex={} entryIndex={}", endIndex, minFastForwardIndex);
    pair.getValue().complete(buildResponse(pair.getKey(), DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
}
```

遍历writeRequestMap中待写入的日志,分别对以下情况进行处理:
1. 如果index小于等于follower的endIndex, 那么说明该日志应该在follow而本地有存储, 比对它们之间的entry是否相同,如果不同那么返回INCONSISTENT_STATE,
让leader发送COMPARE.
2. 如果index正好等于endIndex+1, 那么意味着要写入的日志进入队列中,退出循环.
3. 否则就是index > endIndex + 1的场合, 去判断该index对应的future是否已经超时, 如果已经超时那么记录该index到minFastForwardIndex中.

4. 循环之外,说明此处endIndex+1的日志仍未到达, 如果记录了minFastForwardIndex, 那么判断其是否在仍然在待写入队列中, 如果仍然存在
那么返回INCONSISTENT_STATE,要求leader重新比对日志.

上面就是follower接收到leader请求的动作, 下面我们看leader对follwer的返回结果是如何处理的.

### QuorumAckChecker
代码比较长, 我们分几块来看.
```java
if (pendingAppendResponsesByTerm.size() > 1) {
    for (Long term : pendingAppendResponsesByTerm.keySet()) {
        if (term == currTerm) {
            continue;
        }
        for (Map.Entry<Long, TimeoutFuture<AppendEntryResponse>> futureEntry : pendingAppendResponsesByTerm.get(term).entrySet()) {
            AppendEntryResponse response = new AppendEntryResponse();
            response.setGroup(memberState.getGroup());
            response.setIndex(futureEntry.getKey());
            response.setCode(DLedgerResponseCode.TERM_CHANGED.getCode());
            response.setLeaderId(memberState.getLeaderId());
            logger.info("[TermChange] Will clear the pending response index={} for term changed from {} to {}", futureEntry.getKey(), term, currTerm);
            futureEntry.getValue().complete(response);
        }
        pendingAppendResponsesByTerm.remove(term);
    }
}
if (peerWaterMarksByTerm.size() > 1) {
    for (Long term : peerWaterMarksByTerm.keySet()) {
        if (term == currTerm) {
            continue;
        }
        logger.info("[TermChange] Will clear the watermarks for term changed from {} to {}", term, currTerm);
        peerWaterMarksByTerm.remove(term);
    }
}
```
清理pendingAppendResponsesByTerm和peerWaterMarksByTerm.

```java
Map<String, Long> peerWaterMarks = peerWaterMarksByTerm.get(currTerm);

long quorumIndex = -1;
for (Long index : peerWaterMarks.values()) {
    int num = 0;
    for (Long another : peerWaterMarks.values()) {
        if (another >= index) {
            num++;
        }
    }
    if (memberState.isQuorum(num) && index > quorumIndex) {
        quorumIndex = index;
    }
}
```
根据当前term找出当前占半数以上最大的index, 存入变量quorumIndex中.

```java
if (quorumIndex >= 0) {
    for (Long i = quorumIndex; i >= 0; i--) {
        try {
            CompletableFuture<AppendEntryResponse> future = responses.remove(i);
            if (future == null) {
                needCheck = lastQuorumIndex != -1 && lastQuorumIndex != quorumIndex && i != lastQuorumIndex;
                break;
            } else if (!future.isDone()) {
                AppendEntryResponse response = new AppendEntryResponse();
                response.setGroup(memberState.getGroup());
                response.setTerm(currTerm);
                response.setIndex(i);
                response.setLeaderId(memberState.getSelfId());
                response.setPos(((AppendFuture) future).getPos());
                future.complete(response);
            }
            ackNum++; // 本次完成确认提交的日志数
        } catch (Throwable t) {
            logger.error("Error in ack to index={} term={}", i, currTerm, t);
        }
    }
}
```
从quorumIndex, 递减处理, 判断index对应的客户端future是否完成, 如果未完成, 那么构造reponse并且调用CompletedFuture的complete方法.
如果递减直到future以完成, 那么获取needChech的值,
needCheck为true,需要满足下面几个条件:
1. 最后仲裁的日志号不为-1
2. 最后仲裁的日志号不等于quorumIndex
3. 当前日志号不等于最后仲裁的日志号.

needcheck是为了判断responses这个Map是否发生了泄漏存在的,观察needChechk为true的条件,
可以发现对于条件1: 即这次仲裁是第一次仲裁, 所以不存在泄漏的future, 为什么呢, 因为如果存在的话, 那么此时future==null如何解释?因为本次term第一次执行本方法,所以没有任何机会清理这个Map.
对于条件2: 即本次没有发生任何新增的需要确认的日志, 那么也就不存在泄漏的future.
对于条件3: 当前日志号等于lastQuorumIndex就意味着不存在泄漏的日志, 因为在上一次执行方法的时候就清理完了lastQuorumIndex之前的所有future.

```java
if (ackNum == 0) {
    for (long i = quorumIndex + 1; i < Integer.MAX_VALUE; i++) {
        TimeoutFuture<AppendEntryResponse> future = responses.get(i);
        if (future == null) {
            break;
        } else if (future.isTimeOut()) {
            AppendEntryResponse response = new AppendEntryResponse();
            response.setGroup(memberState.getGroup());
            response.setCode(DLedgerResponseCode.WAIT_QUORUM_ACK_TIMEOUT.getCode());
            response.setTerm(currTerm);
            response.setIndex(i);
            response.setLeaderId(memberState.getSelfId());
            future.complete(response);
        } else {
            break;
        }
    }
    waitForRunning(1);
}
```
如果本次确认的日志数为0, 那么尝试从quorumIndex开始判断是否存在客户端的append的future是否超时,如果超时,
那么设置结果.

```java
if (DLedgerUtils.elapsed(lastCheckLeakTimeMs) > 1000 || needCheck) {
    updatePeerWaterMark(currTerm, memberState.getSelfId(), dLedgerStore.getLedgerEndIndex());
    for (Map.Entry<Long, TimeoutFuture<AppendEntryResponse>> futureEntry : responses.entrySet()) {
        if (futureEntry.getKey() < quorumIndex) {
            AppendEntryResponse response = new AppendEntryResponse();
            response.setGroup(memberState.getGroup());
            response.setTerm(currTerm);
            response.setIndex(futureEntry.getKey());
            response.setLeaderId(memberState.getSelfId());
            response.setPos(((AppendFuture) futureEntry.getValue()).getPos());
            futureEntry.getValue().complete(response);
            responses.remove(futureEntry.getKey());
        }
    }
    lastCheckLeakTimeMs = System.currentTimeMillis();
}
lastQuorumIndex = quorumIndex;
```
检查responses是否产生泄漏, 即判断index是否小于当前确认的日志index即quorumIndex, 如果小于, 那么直接完成对应的future.
