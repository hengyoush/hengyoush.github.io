---
layout: post
title:  "Rocketmq IndexFile源码解读"
date:   2019-09-15 10:02:00 +0700
categories: [rocketmq]
---

## 学习目标
1. 搞清楚rocketmq中IndexFile机制原理, 了解IndexFile是什么以及它的作用, 组成原理以及运行原理.

## IndexFile的作用
IndexFile的主要作用是作为CommitLog的索引文件, 可以快速根据给定的key查找到对应消息在CommitLog的物理位移.

## IndexFile的组成
IndexFile主要由一下三个部分组成:
IndexHeader + HashSlot + IndexItem

IndexHeader:
IndexFile的头部, 大小40字节.
- beginTimestamp: 该IndexFile文件中包含消息的最小存储时间.
- endTimeStamp: 该IndexFile文件中包含消息的最大存储时间.
- beginPhyoffset: 该IndexFile文件中包含消息的最小物理位移.
- endPhyoffset: 该IndexFile文件中包含消息的最大物理位移.
- hashSlotcount: 该IndexFile文件中包含的hashSlot的总数.
- indexCount: 该IndexFile文件中使用的Index条目个数.

HashSlot:
默认包含500w个hash槽, 每个hash槽中存储的是最近一个hash条目的位置.

IndexItem:
默认最多包含2000w个indexItem.
其组成如下所示:
- keyHash: 消息key的hash, 当根据key搜索时比较的是其hash, 所以在之后会比较key本身.
- phyOffset: 消息的物理位移.
- timeDiff: 与第一条消息的时间戳差值.
- prevIndex: hash冲突的上一个indexItem的位置.

## Hash冲突解决方案
IndexFile是如何解决hash冲突的呢?

如果发生Hash冲突, 首先查看hashSlot存储的值, 该值是当前发生冲突的hash的IndexItem的位置, 然后
使用当前hashSlotcount + 1的值覆盖该hashSlot.

在Item区新建一个索引条目, 要注意prevIndex, 这里存放的是之前从hashslot中拿到的值, 这样就组成了一条映射到同一个slot上的链.
只要找到了hashSlot的值, 再根据prevIndex构成的链式结构就一定能够找到对应key的消息的物理位移.

实现代码如下(IndexFile的putKey方法):
1. 根据key的hash值找到对应的hashSlot, 确定上一条相同slot在IndexFile中的绝对位移:
```java
int keyHash = indexKeyHashMethod(key);
int slotPos = keyHash % this.hashSlotNum;
int absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos * hashSlotSize;
```

2. 确定本次要插入的位置, 可以看到, 插入位置是不断在IndexFile的尾部进行append.

```java
int absIndexPos =
      IndexHeader.INDEX_HEADER_SIZE + this.hashSlotNum * hashSlotSize
      + this.indexHeader.getIndexCount() * indexSize;
```

3. 进行插入
```java
// hash
this.mappedByteBuffer.putInt(absIndexPos, keyHash);
// 物理位移
this.mappedByteBuffer.putLong(absIndexPos + 4, phyOffset);
// 与beginTimestamp相对的时间位移
this.mappedByteBuffer.putInt(absIndexPos + 4 + 8, (int) timeDiff);
// 记录“链表”的上一个indexItem的位置
this.mappedByteBuffer.putInt(absIndexPos + 4 + 8 + 4, slotValue);

// 更新hashslot的值, 将其值设置为本次插入的位置(位置指第几个indexItem)
this.mappedByteBuffer.putInt(absSlotPos, this.indexHeader.getIndexCount());
```

## 根据key定位条目物理位移流程源码解读
该流程的代码主要在IndexFile的selectPhyOffset方法内, 该方法有以下几个参数:
- key: 根据该key查找对应的消息
- maxNum: 本次查找返回最多的消息个数
- begin: 查找该时间戳以后的消息
- end: 查找该时间戳以前的消息

1. 首先确定key的hash值并且根据hashSlot找到该hash值对应的最新的IndexItem的位置.
```java
int keyHash = indexKeyHashMethod(key);
int slotPos = keyHash % this.hashSlotNum;
int absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos * hashSlotSize;
```

2. 根据prevIndex组成的链式结构, 将符合条件的消息的物理位移加入到结果中
```java
int absIndexPos =
      IndexHeader.INDEX_HEADER_SIZE + this.hashSlotNum * hashSlotSize
            + nextIndexToRead * indexSize;

int keyHashRead = this.mappedByteBuffer.getInt(absIndexPos);
long phyOffsetRead = this.mappedByteBuffer.getLong(absIndexPos + 4);

long timeDiff = (long) this.mappedByteBuffer.getInt(absIndexPos + 4 + 8);
int prevIndexRead = this.mappedByteBuffer.getInt(absIndexPos + 4 + 8 + 4);

if (timeDiff < 0) {
      break;
}

timeDiff *= 1000L;

long timeRead = this.indexHeader.getBeginTimestamp() + timeDiff;
boolean timeMatched = (timeRead >= begin) && (timeRead <= end);

if (keyHash == keyHashRead && timeMatched) {
      phyOffsets.add(phyOffsetRead);
}

if (prevIndexRead <= invalidIndex
      || prevIndexRead > this.indexHeader.getIndexCount()
      || prevIndexRead == nextIndexToRead || timeRead < begin) {
      break;
}
```


