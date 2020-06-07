---
layout: post
title:  "Dledger中日志追加流程概述"
date:   2020-02-18 15:20:00 +0700
categories: [rocketmq,raft]
---

## 学习目标
搞清楚Dledger中Leader和Follower中对日志追加的流程.

## 源码分析

下面代码是leader处理客户端append请求.(DledgerServer#handleAppend)
```java
long currTerm = memberState.currTerm();
if (dLedgerEntryPusher.isPendingFull(currTerm)) { //        1
    AppendEntryResponse appendEntryResponse = new AppendEntryResponse();
    appendEntryResponse.setGroup(memberState.getGroup());
    appendEntryResponse.setCode(DLedgerResponseCode.LEADER_PENDING_FULL.getCode());
    appendEntryResponse.setTerm(currTerm);
    appendEntryResponse.setLeaderId(memberState.getSelfId());
    return AppendFuture.newCompletedFuture(-1, appendEntryResponse);
} else {                                           //       2
    DLedgerEntry dLedgerEntry = new DLedgerEntry();
    dLedgerEntry.setBody(request.getBody());
    DLedgerEntry resEntry = dLedgerStore.appendAsLeader(dLedgerEntry);
    return dLedgerEntryPusher.waitAck(resEntry);
}
```
1: 由dLedgerEntryPusher判断当前term的pending请求是否大于阈值, 如果大于那么直接拒绝.
2: 如果未超过阈值, 那么首先构造dLedgerEntry并且存入leader的本地存储, 接着调用waitAck方法, 等待超过半数以上的follower接收日志返回.

appendAsLeader
存储结构大致与rocketmq的存储相同, 同样分为存储日志的部分和存储index的部分.在存储时需要构造这两个部分.

日志的结构如下:
magic - size - index - term - pos - channel - chain crc - body crc - body size - body
pos: 物理偏移量

index的结构如下:
magic - pos - size - index - term

只保留了关键代码
```java
public DLedgerEntry appendAsLeader(DLedgerEntry entry) {
    ByteBuffer dataBuffer = localEntryBuffer.get();
    ByteBuffer indexBuffer = localIndexBuffer.get();
    DLedgerEntryCoder.encode(entry, dataBuffer);
    int entrySize = dataBuffer.remaining();
    synchronized (memberState) {
        long nextIndex = ledgerEndIndex + 1;
        entry.setIndex(nextIndex);
        entry.setTerm(memberState.currTerm());
        entry.setMagic(CURRENT_MAGIC);
        DLedgerEntryCoder.setIndexTerm(dataBuffer, nextIndex, memberState.currTerm(), CURRENT_MAGIC);
        long prePos = dataFileList.preAppend(dataBuffer.remaining()); // @1
        entry.setPos(prePos);
        DLedgerEntryCoder.setPos(dataBuffer, prePos);
        long dataPos = dataFileList.append(dataBuffer.array(), 0, dataBuffer.remaining()); // @2
        DLedgerEntryCoder.encodeIndex(dataPos, entrySize, CURRENT_MAGIC, nextIndex, memberState.currTerm(), indexBuffer);
        long indexPos = indexFileList.append(indexBuffer.array(), 0, indexBuffer.remaining(), false);
        ledgerEndIndex++;
        ledgerEndTerm = memberState.currTerm();
        if (ledgerBeginIndex == -1) {
            ledgerBeginIndex = ledgerEndIndex;
        }
        updateLedgerEndIndexAndTerm();
        return entry;
    }
}
```
这里的逻辑较为简单:
1. 构造dataBuffer并且将日志条目的信息补充到buffer里去,然后将buffer写入到文件中.
2. 构造indexBuffer, 将buffer也写入到文件中.
3. 更新ledgerEndIndex和ledgerEndTerm,返回日志条目.

值得注意的细节:
```java
long prePos = dataFileList.preAppend(dataBuffer.remaining());
```

此处调用dataFileList的preAppend方法主要是为了获取到日志条目的物理position.
dataBuffer的remaining获取到整个日志条目的大小.接着我们看preAppend方法.(省略非核心代码)
```java
public long preAppend(int len, boolean useBlank) {
    MmapFile mappedFile = getLastMappedFile(); // 1
    if (null == mappedFile || mappedFile.isFull()) {
        mappedFile = getLastMappedFile(0);
    }
    int blank = useBlank ? MIN_BLANK_LEN : 0;
    if (len + blank > mappedFile.getFileSize() - mappedFile.getWrotePosition()) { // 2
        ByteBuffer byteBuffer = ByteBuffer.allocate(mappedFile.getFileSize() - mappedFile.getWrotePosition());
        byteBuffer.putInt(BLANK_MAGIC_CODE);
        byteBuffer.putInt(mappedFile.getFileSize() - mappedFile.getWrotePosition());
        if (mappedFile.appendMessage(byteBuffer.array())) {
            //need to set the wrote position
            mappedFile.setWrotePosition(mappedFile.getFileSize());
        } else {
            logger.error("Append blank error for {}", storePath);
            return -1;
        }
        mappedFile = getLastMappedFile(0);
        if (null == mappedFile) {
            logger.error("Create mapped file for {}", storePath);
            return -1;
        }
    }
    return mappedFile.getFileFromOffset() + mappedFile.getWrotePosition(); // 3

}
```
1. 首先获取到最后一个MmapFile.(如果不存在,那么此处会进行创建)
2. 接着判断MmapFile有没有足够空间写入日志条目, 如果没有, 那么会创建新的文件.
3. 不论2是否创建新文件, 返回file当前的物理位移+file当前的wrotePosition作为要写入日志条目的起始物理位移.


让我们回到handleAppend方法里, leader自己写入日志条目之后就需要将日志推送到集群中到其他节点, 半数以上写入成功后commit, 才可以
返回客户端写入成功.

好的, 让我们开始分析waitAck方法:
```java
public CompletableFuture<AppendEntryResponse> waitAck(DLedgerEntry entry) {
    updatePeerWaterMark(entry.getTerm(), memberState.getSelfId(), entry.getIndex()); // 1. 更新水位线
    if (memberState.getPeerMap().size() == 1) { // 2. 只有一个节点
        AppendEntryResponse response = new AppendEntryResponse();
        response.setGroup(memberState.getGroup());
        response.setLeaderId(memberState.getSelfId());
        response.setIndex(entry.getIndex());
        response.setTerm(entry.getTerm());
        response.setPos(entry.getPos());
        return AppendFuture.newCompletedFuture(entry.getPos(), response);
    } else {
        checkTermForPendingMap(entry.getTerm(), "waitAck");
        AppendFuture<AppendEntryResponse> future = new AppendFuture<>(dLedgerConfig.getMaxWaitAckTimeMs());
        future.setPos(entry.getPos());
        wakeUpDispatchers(); // 唤醒dispatcher, 方法日志
        return future;
    }
}
```
1. 更新leader的水位线, 水位线关联的是一个Map`Map<Long<* term *>, ConcurrentMap<String<* peerId *>, Long<* index *>>`,用于记录集群内各个节点
的日志情况.
2. 如果只有一个节点, 那么直接返回即可, 如果不是, 那么调用wakeUpDispatchers, 唤醒dispatcher线程, 进行分发.

