# TubeMQ消息中间件源码分析

* [SSD辅助存储](#ssd辅助存储)
  * [请求SSD转存](#请求ssd转存)
  * [处理转存请求](#处理转存请求)
    * [获取需要转存的文件数据](#获取需要转存的文件数据)
    * [检查是否转存](#检查是否转存)
    * [拷贝文件到SSD](#拷贝文件到ssd)
* [内存块缓存](#内存块缓存)


# SSD辅助存储

## 请求SSD转存

com.tencent.tubemq.server.broker.msgstore.ssd.MsgSSDStoreManager#requestSsdTransfer

```java
public boolean requestSsdTransfer(final String partStr, final String storeKey,
                                  final long startOffset, final int segCnt) {
    if (this.isStart && !this.closed) {
        try {
            this.reqSSDEvents.put(new SSDSegEvent(partStr, storeKey, startOffset, segCnt));
            return true;
        } catch (Throwable e1) {
            logger.warn("[SSD Store] request SSD Event failure : ", e1);
        }
    }
    return false;
}
```

## 处理转存请求

com.tencent.tubemq.server.broker.msgstore.ssd.MsgSSDStoreManager.SsdStoreRunner#run

```java
public void run() {
    final StringBuilder strBuffer = new StringBuilder(512);
    logger.info("[SSD Manager] start process SSD  transfer requests");
    try {
        MetadataManage metadataManage = msgStoreMgr.getMetadataManage();
        while (!closed) {
            try {
                SSDSegEvent ssdSegEvent =
                        reqSSDEvents.poll(500, TimeUnit.MILLISECONDS); //获取转存请求
                if (ssdSegEvent == null) {
                    ThreadUtils.sleep(1000);
                    continue;
                }
              //consumer group - topic - partition id  --> consumer info
                Map<String, ConsumerNodeInfo> consumerNodeInfoMap =
             msgStoreMgr.getTubeBroker().getBrokerServiceServer().getConsumerRegisterMap();
                ConsumerNodeInfo consumerNodeInfo =
                        consumerNodeInfoMap.get(ssdSegEvent.partStr);
                if (consumerNodeInfo == null) {
                    continue;
                }
                final int index = ssdSegEvent.storeKey.lastIndexOf('-');
                if (index < 0) {
                    logger.warn(strBuffer.append("[SSD Manager] Ignore invlaid storeKey=")
                            .append(ssdSegEvent.storeKey).toString());
                    strBuffer.delete(0, strBuffer.length());
                    continue;
                }
              //主题名
                final String topic = ssdSegEvent.storeKey.substring(0, index);
                TopicMetadata topicMetadata =
                        metadataManage.getTopicMetadata(topic);
                if (topicMetadata == null) {
                    logger.warn(strBuffer
                            .append("[SSD Manager] No valid topic config for storeKey=")
                            .append(ssdSegEvent.storeKey).toString());
                    strBuffer.delete(0, strBuffer.length());
                    continue;
                }
                final int storeId =
                        Integer.parseInt(ssdSegEvent.storeKey.substring(index + 1));
              //获取SSDSegment
                SSDSegFound ssdSegFound =
                        msgStoreMgr.getSourceSegment(topic, storeId, ssdSegEvent.startOffset, 15);
                if (!ssdSegFound.isSuccess) {
                    if (ssdSegFound.reason > 0) {
                        logger.warn(strBuffer
                                .append("[SSD Manager] Found source store error, storeKey=")
                                .append(ssdSegEvent.storeKey).append(", reason=")
                                .append(ssdSegFound.reason).toString());
                        strBuffer.delete(0, strBuffer.length());
                    }
                    consumerNodeInfo.setSSDTransferFinished(false,
                            ssdSegFound.startOffset, ssdSegFound.endOffset);
                    continue;
                }
                checkSegmentTransferStatus(topic, ssdSegEvent,
                        ssdSegFound, consumerNodeInfo, strBuffer);
            } catch (InterruptedException e) {
                break;
            } catch (Throwable e2) {
                logger.error("[SSD Manager] process SSD  transfer request failure", e2);
            }
        }
        logger.info("[SSD Manager] stopped process SSD  transfer requests");
    } catch (Throwable e) {
        logger.error("[SSD Manager] Error during process SSD transfer requests", e);
    }
}
```

### 获取需要转存的文件数据

com.tencent.tubemq.server.broker.msgstore.MessageStoreManager#getSourceSegment

```java
public SSDSegFound getSourceSegment(final String topic, final int storeId,
                                    final long offset, final int rate) throws IOException {
    ConcurrentHashMap<Integer, MessageStore> map = this.dataStores.get(topic);
    if (map == null) {
        return new SSDSegFound(false, 1, null);
    }
    MessageStore messageStore = map.get(storeId);
    if (messageStore == null) {
        return new SSDSegFound(false, 2, null);
    }
    return messageStore.getSourceSegment(offset, rate);
}
```

com.tencent.tubemq.server.broker.msgstore.MessageStore#getSourceSegment

```java
public SSDSegFound getSourceSegment(final long offset,
                                    final int rate) throws IOException {
    return this.msgFileStore.getSourceSegment(offset, rate);
}
```

```java
public SSDSegFound getSourceSegment(final long offset, final int rate) throws IOException {
     // 二分法查找包含offset的segment
    final Segment segment = this.dataSegments.findSegment(offset);
    if (segment == null) {
        return new SSDSegFound(false, -1, null);
    }
   //文件大小
    long dataSize = segment.getCachedSize();
    //如果需要读取的数据小于文件大小的15%，不再进行转存
    if ((dataSize - offset + segment.getStart()) < dataSize * rate / 100) {
        return new SSDSegFound(false, -2, null,
                segment.getStart(), segment.getStart() + segment.getCachedSize());
    }
    return new SSDSegFound(true, 0, segment.getFile(),
            segment.getStart(), segment.getStart() + segment.getCachedSize());
}
```

### 检查是否转存

com.tencent.tubemq.server.broker.msgstore.ssd.MsgSSDStoreManager#checkSegmentTransferStatus

```java
private void checkSegmentTransferStatus(final String topicName, final SSDSegEvent ssdSegEvent,
                                        final SSDSegFound ssdSegFound, final ConsumerNodeInfo consumerNodeInfo,
                                        final StringBuilder strBuffer) {
    consumerNodeInfo.setSSDProcing();
    //storeKey-><Long, MsgSSDSegment>
    ConcurrentHashMap<Long, MsgSSDSegment> msgSegmentMap =
            ssdSegmentsMap.get(ssdSegEvent.storeKey);
    if (msgSegmentMap == null) {
        ConcurrentHashMap<Long, MsgSSDSegment> tmpMsgSsdSegMap =
                new ConcurrentHashMap<Long, MsgSSDSegment>();
        msgSegmentMap =
                ssdSegmentsMap.putIfAbsent(ssdSegEvent.storeKey, tmpMsgSsdSegMap);
        if (msgSegmentMap == null) {
            msgSegmentMap = tmpMsgSsdSegMap;
        }
    }
    MsgSSDSegment curMsgSsdSegment =
            msgSegmentMap.get(ssdSegFound.startOffset);
    if (curMsgSsdSegment != null) {//转存数据已经存在
        curMsgSsdSegment.setInUse();
        ConcurrentHashSet<SSDSegIndex> ssdSegIndices =
                ssdPartStrMap.get(ssdSegEvent.partStr);
        if (ssdSegIndices == null) {
            ConcurrentHashSet<SSDSegIndex> tmpSsdSegIndices =
                    new ConcurrentHashSet<SSDSegIndex>();
            ssdSegIndices =
                    ssdPartStrMap.putIfAbsent(ssdSegEvent.partStr, tmpSsdSegIndices);
            if (ssdSegIndices == null) {
                ssdSegIndices = tmpSsdSegIndices;
            }
        }
        ssdSegIndices.add(new SSDSegIndex(ssdSegEvent.storeKey,
                ssdSegFound.startOffset, ssdSegFound.endOffset));
        //转存请求结束
        consumerNodeInfo.setSSDTransferFinished(true,
                ssdSegFound.startOffset, ssdSegFound.endOffset);
    } else { //加载文件
        if (totalSsdFileCnt.get() >= tubeConfig.getMaxSSDTotalFileCnt()
                || totalSsdFileSize.get() >= tubeConfig.getMaxSSDTotalFileSizes()) {//是否达到SSD转存的阈值（最大的文件数、最大的占用空间）
            consumerNodeInfo.setSSDTransferFinished(false, -2, -2);
            return;
        }
        //复制到SSD
        boolean result = copyFileToSSD(ssdSegEvent.storeKey, topicName,
                ssdSegEvent.partStr, ssdSegFound.sourceFile,
                ssdSegFound.startOffset, ssdSegFound.endOffset, strBuffer);
        if (result) { //复制成功
            consumerNodeInfo.setSSDTransferFinished(true,
                    ssdSegFound.startOffset, ssdSegFound.endOffset);
        } else { //复制失败
            consumerNodeInfo.setSSDTransferFinished(false,
                    ssdSegFound.startOffset, ssdSegFound.endOffset);
        }
    }
}
```

### 拷贝文件到SSD

com.tencent.tubemq.server.broker.msgstore.ssd.MsgSSDStoreManager#copyFileToSSD

```java
private boolean copyFileToSSD(final String storeKey, final String topic,
                              final String partStr, File fromFile,
                              final long startOffset, final long endOffset,
                              final StringBuilder sb) {
    // #lizard forgives
    logger.info(sb.append("[SSD Manager] Begin to copy file to SSD, source data path:")
            .append(fromFile.getAbsolutePath()).toString());
    sb.delete(0, sb.length());

    if (!fromFile.exists()) {
        logger.warn(sb.append("[SSD Manager] source file not exists, path:")
                .append(fromFile.getAbsolutePath()).toString());
        sb.delete(0, sb.length());
        return false;
    }

    if (!fromFile.isFile()) {
        logger.warn(sb.append("[SSD Manager] source file not File, path:")
                .append(fromFile.getAbsolutePath()).toString());
        sb.delete(0, sb.length());
        return false;
    }
    if (!fromFile.canRead()) {
        logger.warn(sb.append("[SSD Manager] source file not canRead, path:")
                .append(fromFile.getAbsolutePath()).toString());
        sb.delete(0, sb.length());
        return false;
    }

    boolean isNew = false;
    final String filename = fromFile.getName();
    final long start =
            Long.parseLong(filename.substring(0, filename.length() - DATA_FILE_SUFFIX.length()));

    final File toFileDir =
            new File(sb.append(secdStorePath)
                    .append(File.separator).append(storeKey).toString());
    sb.delete(0, sb.length());
    FileUtil.checkDir(toFileDir);

    ConcurrentHashMap<Long, MsgSSDSegment> msgSsdSegMap =
            ssdSegmentsMap.get(storeKey);

    if (msgSsdSegMap == null) {
        ConcurrentHashMap<Long, MsgSSDSegment> tmpMsgSsdSegMap =
                new ConcurrentHashMap<Long, MsgSSDSegment>();
        msgSsdSegMap = ssdSegmentsMap.putIfAbsent(storeKey, tmpMsgSsdSegMap);
        if (msgSsdSegMap == null) {
            msgSsdSegMap = tmpMsgSsdSegMap;
        }
    }

    MsgSSDSegment msgSsdSegment1 = msgSsdSegMap.get(start);
    if (msgSsdSegment1 == null) { //写文件到SSD
        boolean isSuccess = false;
        FileInputStream fosfrom = null;
        FileOutputStream fosto = null;
        final File targetFile =
                new File(toFileDir, DataStoreUtils.nameFromOffset(start, DATA_FILE_SUFFIX));
        try {
            fosfrom = new FileInputStream(fromFile);
            fosto = new FileOutputStream(targetFile);
            byte[] bt = new byte[65536];
            int c;
            while ((c = fosfrom.read(bt)) > 0) {
                fosto.write(bt, 0, c);
            }
            isSuccess = true;
            isNew = true;
        } catch (FileNotFoundException e) {
            e.printStackTrace();
            logger.warn("[SSD Manager] copy File to SSD FileNotFoundException failure ", e);
        } catch (IOException e) {
            e.printStackTrace();
            logger.warn("[SSD Manager] copy File to SSD IOException failure ", e);
        } finally {
            if (fosfrom != null) {
                try {
                    fosfrom.close();
                } catch (Throwable e2) {
                    logger.warn("[SSD Manager] Close FileInputStream failure! ", e2);
                }
            }
            if (fosto != null) {
                try {
                    fosto.close();
                } catch (Throwable e2) {
                    logger.warn("[SSD Manager] Close FileOutputStream failure! ", e2);
                }
            }
        }

        if (!isSuccess) { //写失败
            try {
                targetFile.delete();
            } catch (Throwable e2) {
                logger.warn("[SSD Manager] remove SSD target file failure ", e2);
            }
            return false;
        }

        try {
            MsgSSDSegment tmpMsgSsdSegment =
                    new MsgSSDSegment(storeKey, topic, start, targetFile);
            if (tmpMsgSsdSegment.getDataMinOffset() != startOffset
                    || tmpMsgSsdSegment.getDataMaxOffset() != endOffset) { //验证合法性
                logger.warn(sb
                        .append("[SSD Manager] created SSD file getCachedSize not equal source: sourceFile=")
                        .append(fromFile.getAbsolutePath()).append(",srcStartOffset=")
                        .append(startOffset).append(",srcEndOffset=").append(endOffset)
                        .append(",dstStartOffset=").append(tmpMsgSsdSegment.getDataMinOffset())
                        .append(",dstEndOffset=").append(tmpMsgSsdSegment.getDataMaxOffset()).toString());
                sb.delete(0, sb.length());
                tmpMsgSsdSegment.close();
                return false;
            }
            msgSsdSegment1 = msgSsdSegMap.putIfAbsent(start, tmpMsgSsdSegment);
            if (msgSsdSegment1 == null) { //存放到map之前不存在
                msgSsdSegment1 = tmpMsgSsdSegment;
                //统计SSD的占用文件、存储的文件数
                this.totalSsdFileSize.addAndGet(tmpMsgSsdSegment.getDataSizeInBytes());
                this.totalSsdFileCnt.incrementAndGet();
            } else { //已经存在，将tmpMsgSsdSegment设置为无效状态
                tmpMsgSsdSegment.close();
            }
        } catch (Throwable e1) {
            logger.warn("[SSD Manager] create SSD FileSegment failure", e1);
            return false;
        }
    }
    //存储对应的索引信息（storeKey、startOffset、endOffset）
    ConcurrentHashSet<SSDSegIndex> ssdSegIndices = this.ssdPartStrMap.get(partStr);
    if (ssdSegIndices == null) {
        ConcurrentHashSet<SSDSegIndex> tmpSsdSegIndices = new ConcurrentHashSet<SSDSegIndex>();
        ssdSegIndices = this.ssdPartStrMap.putIfAbsent(partStr, tmpSsdSegIndices);
        if (ssdSegIndices == null) {
            ssdSegIndices = tmpSsdSegIndices;
        }

    }
    SSDSegIndex ssdSegIndex =
            new SSDSegIndex(storeKey, start, msgSsdSegment1.getDataMaxOffset());
    if (!ssdSegIndices.contains(ssdSegIndex)) {
        ssdSegIndices.add(ssdSegIndex);
    }
    logger.info(sb.append("[SSD Manager] copy file to SSD success, data segment ")
            .append(msgSsdSegment1.getStoreKey()).append(",start=").append(start)
            .append(", isNew=").append(isNew).toString());
    sb.delete(0, sb.length());
    return true;
}
```

# 内存块缓存

写数据时先写入内存，默认大小2M，内存填满时将数据刷新到磁盘

com.tencent.tubemq.server.broker.msgstore.MessageStore#appendMsg

```java
public boolean appendMsg(final long msgId, final int dataLength,
                         final int dataCheckSum, final byte[] data,
                         final int msgTypeCode, final int msgFlag,
                         final int partitionId, final int sentAddr) throws IOException {
    if (this.closed.get()) {
        throw new IllegalStateException(new StringBuilder(512)
                .append("[Data Store] Closed MessageStore for storeKey ")
                .append(this.storeKey).toString());
    }
    int msgBufLen = DataStoreUtils.STORE_DATA_HEADER_LEN + dataLength;
    final long receivedTime = System.currentTimeMillis();
    //追加的数据填充到ByteBuffer
    final ByteBuffer buffer = ByteBuffer.allocate(msgBufLen);
    buffer.putInt(DataStoreUtils.STORE_DATA_PREFX_LEN + dataLength);
    buffer.putInt(DataStoreUtils.STORE_DATA_TOKER_BEGIN_VALUE);
    buffer.putInt(dataCheckSum);
    buffer.putInt(partitionId);
    buffer.putLong(-1L);
    buffer.putLong(receivedTime);
    buffer.putInt(sentAddr);
    buffer.putInt(msgTypeCode);
    buffer.putLong(msgId);
    buffer.putInt(msgFlag);
    buffer.put(data);
    buffer.flip();
    int count = 3; //重试次数
    //写数据到memstore
    do {
        this.writeCacheMutex.readLock().lock();
        try {
            if (this.msgMemStore.appendMsg(msgMemStatisInfo,
                    partitionId, msgTypeCode, receivedTime, msgBufLen, buffer)) {
                return true;
            }
        } finally {
            this.writeCacheMutex.readLock().unlock();
        }
        if (triggerFlushAndAddMsg(partitionId, msgTypeCode,
                receivedTime, msgBufLen, true, buffer, false)) {
            return true;
        }
        ThreadUtils.sleep(1);
    } while (count-- >= 0);
    msgMemStatisInfo.addWriteFailCount();
    return false;
}
```

com.tencent.tubemq.server.broker.msgstore.mem.MsgMemStore#appendMsg

```java
public boolean appendMsg(final MsgMemStatisInfo msgMemStatisInfo,
                         final int partitionId, final int keyCode,
                         final long timeRecv, final int entryLength,
                         final ByteBuffer entry) {
    int alignedSize = (entryLength + 64 - 1) & MASK_64_ALIGN;
    boolean fullDataSize = false;
    boolean fullIndexSize = false;
    boolean fullCount = false;
    this.writeLock.lock();
    try {
        //　判断是否可以写入，默认大小为2M，最多写入的数据条数5 * 1024、索引占用的空间
        int dataOffset = this.cacheDataOffset.get();
        int indexOffset = this.cacheIndexOffset.get();
        if ((fullDataSize = (dataOffset + alignedSize > this.maxDataCacheSize))
                || (fullIndexSize = (indexOffset + this.indexUnitLength > this.maxIndexCacheSize))
                || (fullCount = (this.curMessageCount.get() + 1 > this.maxAllowedMsgCount))) {
            msgMemStatisInfo.addFullTypeCount(timeRecv, fullDataSize, fullIndexSize, fullCount);
            return false;
        }
        // conduct message with filling process
        entry.putLong(DataStoreUtils.STORE_HEADER_POS_QUEUE_LOGICOFF,
                this.writeIndexStartPos + this.cacheIndexSize.get());
        //写数据
        this.cacheDataSegment.position(dataOffset);
        this.cacheDataSegment.put(entry.array());
        //写索引
        this.cachedIndexSegment.position(indexOffset);
        this.cachedIndexSegment.putInt(partitionId);
        this.cachedIndexSegment.putInt(keyCode);
        this.cachedIndexSegment.putInt(dataOffset);
        this.cachedIndexSegment.putInt(entryLength);
        this.cachedIndexSegment.putLong(timeRecv);
        this.cachedIndexSegment
                .putLong(this.writeDataStartPos + this.cacheDataSize.getAndAdd(entryLength));

        Integer indexSizePos =
                this.cacheIndexSize.getAndAdd(DataStoreUtils.STORE_INDEX_HEAD_LEN);

        this.queuesMap.put(partitionId, indexSizePos);
        this.keysMap.put(keyCode, indexSizePos);

        this.cacheDataOffset.getAndAdd(alignedSize);
        this.cacheIndexOffset.getAndAdd(this.indexUnitLength);
        this.curMessageCount.getAndAdd(1);
        msgMemStatisInfo.addMsgSizeStatis(timeRecv, entryLength);
    } finally {
        this.writeLock.unlock();
      
    }
    return true;
}
```

