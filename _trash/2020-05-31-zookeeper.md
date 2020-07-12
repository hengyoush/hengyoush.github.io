---
layout: post
title:  "Dledger中日志追加流程详解"
date:   2020-02-18 15:20:00 +0700
categories: [rocketmq,raft]
---

## 学习目标


## 源码分析
添加一个事务日志
```java
    public synchronized boolean append(TxnHeader hdr, Record txn)
        throws IOException
    {
        // 新建文件
        if (logStream==null) {
           logFileWrite = new File(logDir, Util.makeLogName(hdr.getZxid())); // 根据zxid生成文件名
           fos = new FileOutputStream(logFileWrite);
           logStream=new BufferedOutputStream(fos);
           oa = BinaryOutputArchive.getArchive(logStream); // 获取jute的序列化工具类
           FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
           fhdr.serialize(oa, "fileheader");
           // Make sure that the magic number is written before padding.
           logStream.flush();
           filePadding.setCurrentSize(fos.getChannel().position());
           streamsToFlush.add(fos);
        }
        // 扩展文件
        filePadding.padFile(fos.getChannel());
        // 序列化事务
        byte[] buf = Util.marshallTxnEntry(hdr, txn);
        Checksum crc = makeChecksumAlgorithm();
        crc.update(buf, 0, buf.length);
        // 使用jute写入文件
        oa.writeLong(crc.getValue(), "txnEntryCRC");
        Util.writeTxnBytes(oa, buf);

        return true;
    }
    // 64MB
    long padFile(FileChannel fileChannel) throws IOException {
        long newFileSize = calculateFileSizeWithPadding(fileChannel.position(), currentSize, preAllocSize);
        if (currentSize != newFileSize) {
            fileChannel.write((ByteBuffer) fill.position(0), newFileSize - fill.remaining());
            currentSize = newFileSize;
        }
        return currentSize;
    }
    // 存在两种情况进行两种不同的padding
    public static long calculateFileSizeWithPadding(long position, long fileSize, long preAllocSize) {
        // If preAllocSize is positive and we are within 4KB of the known end of the file calculate a new file size
        if (preAllocSize > 0 && position + 4096 >= fileSize) {
            // If we have written more than we have previously preallocated we need to make sure the new
            // file size is larger than what we already have
            if (position > fileSize) {
                fileSize = position + preAllocSize;
                fileSize -= fileSize % preAllocSize;
            } else {
                fileSize += preAllocSize;
            }
        }

        return fileSize;
    }
```

快照部分:
Zk的DataTree和DataNode,DataNode即Zk中的节点,相关重要属性如下:
```java
    /** the data for this datanode */
    byte data[];
    /**
     * the list of children for this node. note that the list of children string
     * does not contain the parent path -- just the last part of the path. This
     * should be synchronized on except deserializing (for speed up issues).
     */
    private Set<String> children = null;
```
序列化方法如下:
```java
    synchronized public void deserialize(InputArchive archive, String tag)
            throws IOException {
        archive.startRecord("node");
        data = archive.readBuffer("data");
        acl = archive.readLong("acl");
        stat = new StatPersisted();
        stat.deserialize(archive, "statpersisted");
        archive.endRecord("node");
    }

    synchronized public void serialize(OutputArchive archive, String tag)
            throws IOException {
        archive.startRecord(this, "node");
        archive.writeBuffer(data, "data");
        archive.writeLong(acl, "acl");
        stat.serialize(archive, "statpersisted");
        archive.endRecord(this, "node");
    }
```

DataTree即表示整个Zk的节点存储,里面又一个重要属性nodes,其类型为:
ConcurrentHashMap<String, DataNode>,String即节点的全路径, 
Zk的数据快照大部分主要就是存储DataTree的,让我们看一下DataTree的序列化方法:
```java
    void serializeNode(OutputArchive oa, StringBuilder path) throws IOException {
        String pathString = path.toString();
        DataNode node = getNode(pathString);
        if (node == null) {
            return;
        }
        String children[] = null;
        DataNode nodeCopy;
        synchronized (node) {
            StatPersisted statCopy = new StatPersisted();
            copyStatPersisted(node.stat, statCopy);
            //we do not need to make a copy of node.data because the contents
            //are never changed
            nodeCopy = new DataNode(node.data, node.acl, statCopy);
            Set<String> childs = node.getChildren();
            children = childs.toArray(new String[childs.size()]);
        }
        serializeNodeData(oa, pathString, nodeCopy);
        path.append('/');
        int off = path.length();
        for (String child : children) {
            // since this is single buffer being resused
            // we need
            // to truncate the previous bytes of string.
            path.delete(off, Integer.MAX_VALUE);
            path.append(child);
            serializeNode(oa, path);
        }
    }

    // visiable for test
    public void serializeNodeData(OutputArchive oa, String path, DataNode node) throws IOException {
        oa.writeString(path, "path");
        oa.writeRecord(node, "node");
    }

    public void serialize(OutputArchive oa, String tag) throws IOException {
        aclCache.serialize(oa);
        serializeNode(oa, new StringBuilder(""));
        // / marks end of stream
        // we need to check if clear had been called in between the snapshot.
        if (root != null) {
            oa.writeString("/", "path");
        }
    }
```

反序列化:
```java
    public void deserialize(InputArchive ia, String tag) throws IOException {
        aclCache.deserialize(ia);
        nodes.clear();
        pTrie.clear();
        String path = ia.readString("path");
        while (!"/".equals(path)) {
            DataNode node = new DataNode();
            ia.readRecord(node, "node");
            nodes.put(path, node);
            synchronized (node) {
                aclCache.addUsage(node.acl);
            }
            int lastSlash = path.lastIndexOf('/');
            if (lastSlash == -1) {
                root = node;
            } else {
                String parentPath = path.substring(0, lastSlash);
                DataNode parent = nodes.get(parentPath);
                if (parent == null) {
                    throw new IOException("Invalid Datatree, unable to find " +
                            "parent " + parentPath + " of path " + path);
                }
                parent.addChild(path.substring(lastSlash + 1));
                long eowner = node.stat.getEphemeralOwner();
                EphemeralType ephemeralType = EphemeralType.get(eowner);
                if (ephemeralType == EphemeralType.CONTAINER) {
                    containers.add(path);
                } else if (ephemeralType == EphemeralType.TTL) {
                    ttls.add(path);
                } else if (eowner != 0) {
                    HashSet<String> list = ephemerals.get(eowner);
                    if (list == null) {
                        list = new HashSet<String>();
                        ephemerals.put(eowner, list);
                    }
                    list.add(path);
                }
            }
            path = ia.readString("path");
        }
        nodes.put("/", root);
        // we are done with deserializing the
        // the datatree
        // update the quotas - create path trie
        // and also update the stat nodes
        setupQuota();

        aclCache.purgeUnused();
    }
```

Zookeeper客户端网络连接初始化
```java
    void connect(InetSocketAddress addr) throws IOException {
        firstConnect = new CountDownLatch(1);

        Bootstrap bootstrap = new Bootstrap()
                .group(eventLoopGroup)
                .channel(NettyUtils.nioOrEpollSocketChannel())
                .option(ChannelOption.SO_LINGER, -1)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(new ZKClientPipelineFactory(addr.getHostString(), addr.getPort()));
        bootstrap = configureBootstrapAllocator(bootstrap);
        bootstrap.validate();

        connectLock.lock();
        try {
            connectFuture = bootstrap.connect(addr);
            connectFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture channelFuture) throws Exception {
                    // this lock guarantees that channel won't be assigned after cleanup().
                    boolean connected = false;
                    connectLock.lock();
                    try {
                        if (!channelFuture.isSuccess()) {
                            LOG.info("future isn't success, cause:", channelFuture.cause());
                            return;
                        } else if (connectFuture == null) {
                            LOG.info("connect attempt cancelled");
                            // If the connect attempt was cancelled but succeeded
                            // anyway, make sure to close the channel, otherwise
                            // we may leak a file descriptor.
                            channelFuture.channel().close();
                            return;
                        }
                        // setup channel, variables, connection, etc.
                        channel = channelFuture.channel();

                        disconnected.set(false);
                        initialized = false;
                        lenBuffer.clear();
                        incomingBuffer = lenBuffer;

                        sendThread.primeConnection();
                        updateNow();
                        updateLastSendAndHeard();

                        if (sendThread.tunnelAuthInProgress()) {
                            waitSasl.drainPermits();
                            needSasl.set(true);
                            sendPrimePacket();
                        } else {
                            needSasl.set(false);
                        }
                        connected = true;
                    } finally {
                        connectFuture = null;
                        connectLock.unlock();
                        if (connected) {
                            LOG.info("channel is connected: {}", channelFuture.channel());
                        }
                        // need to wake on connect success or failure to avoid
                        // timing out ClientCnxn.SendThread which may be
                        // blocked waiting for first connect in doTransport().
                        wakeupCnxn();
                        firstConnect.countDown();
                    }
                }
            });
        } finally {
            connectLock.unlock();
        }
    }

        protected void initChannel(SocketChannel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            if (clientConfig.getBoolean(ZKClientConfig.SECURE_CLIENT)) {
                initSSL(pipeline);
            }
            pipeline.addLast("handler", new ZKClientHandler());
        }
```

Packet:
```java
Packet {
        RequestHeader requestHeader;

        ReplyHeader replyHeader;

        Record request;

        Record response;

        ByteBuffer bb;
```

ZKClientHandler

运动会 开幕式 熙熙攘攘 热闹 气氛 礼炮 五彩缤纷 我们的死亡 墓碑 寂静的死去
family 单词 无知 只记住了这个 对家庭的渴望