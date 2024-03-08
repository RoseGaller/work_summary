# HaloDB源码分析

* [概述](#概述)
* [写数据](#写数据)
  * [写文件](#写文件)
  * [标记之前key的数据为无效状态](#标记之前key的数据为无效状态)
  * [堆外内存保存索引](#堆外内存保存索引)
* [读数据](#读数据)
  * [获取索引信息](#获取索引信息)
  * [读取文件](#读取文件)
* [删除数据](#删除数据)
* [合并数据](#合并数据)


# 概述

HaloDB是一个用Java编写的快速而简单的嵌入式键值存储。HaloDB由两个主要组件组成:一个存储所有键的内存索引，以及顺序追加的方式保存日志到磁盘。为了减少Java垃圾收集的压力，索引在Java堆之外的本机内存中分配。

对于读请求，它最多需要进行一次磁盘查找。

HaloDB提供了一种配置，可以调优来控制写入放大空间放大，进行两者间的相互权衡的;HaloDB有一个后台压缩线程来删除过时的数据从DB。可以控制压缩文件时过期数据的百分比。增加这个值将增加空间放大而会减少写的放大。

提前写日志(Write Ahead Logs, WAL)来实现崩溃恢复。HaloDB不会立即刷新数据到磁盘，只写入操作系统页面缓存。缓存的数据链达到可配置的大小就会刷新到磁盘。在断电的情况下，未刷新到磁盘的数据将丢失。

# 写数据

com.oath.halodb.HaloDBInternal#put

```java
boolean put(byte[] key, byte[] value) throws IOException, HaloDBException {
    if (key.length > Byte.MAX_VALUE) {
        throw new HaloDBException("key length cannot exceed " + Byte.MAX_VALUE);
    }
    writeLock.lock();
    try {
        Record record = new Record(key, value);
        //唯一标志
        record.setSequenceNumber(getNextSequenceNumber());
        //创建标志
        record.setVersion(Versions.CURRENT_DATA_FILE_VERSION); 
        //写数据到文件
        InMemoryIndexMetaData entry = writeRecordToFile(record); 
        //标记之前key的写入的数据为无效状态
        markPreviousVersionAsStale(key);
        //索引信息添加到内存
        return inMemoryIndex.put(key, entry);
    } finally {
        writeLock.unlock();
    }
}
```

## 写文件

```java
private InMemoryIndexMetaData writeRecordToFile(Record record) throws IOException, HaloDBException {
  	//检测是否生成新文件
    rollOverCurrentWriteFile(record);
    return currentWriteFile.writeRecord(record);
}
```

判断是否生成新文件

```java
private void rollOverCurrentWriteFile(Record record) throws IOException {
    //追加数据的大小
    int size = record.getKey().length + record.getValue().length + Record.Header.HEADER_SIZE;
    if ((currentWriteFile == null || currentWriteFile.getWriteOffset() + size > options.getMaxFileSize())
        && !isClosing) {
        forceRollOverCurrentWriteFile();
    }
}
```

刷新之前文件的缓存到磁盘，生成新的文件

```java
private void forceRollOverCurrentWriteFile() throws IOException {
    if (currentWriteFile != null) { //刷新数据到磁盘
        currentWriteFile.flushToDisk();
        currentWriteFile.getIndexFile().flushToDisk();
    }
    //创建新文件
    currentWriteFile = createHaloDBFile(HaloDBFile.FileType.DATA_FILE);
    dbDirectory.syncMetaData();
}
```

写数据到操作系统内存，默认刷盘操作交由操作系统控制

com.oath.halodb.HaloDBFile#writeRecord

```java
InMemoryIndexMetaData writeRecord(Record record) throws IOException {
    writeToChannel(record.serialize());
    //数据大小
    int recordSize = record.getRecordSize();
    //追加的文件位置
    int recordOffset = writeOffset;
    writeOffset += recordSize;
    //文件索引
    IndexFileEntry indexFileEntry = new IndexFileEntry(
        record.getKey(), recordSize,
        recordOffset, record.getSequenceNumber(),
        Versions.CURRENT_INDEX_FILE_VERSION, -1
    );
    //写入索引文件
    indexFile.write(indexFileEntry);
    //value在文件中的位置
    int valueOffset = Utils.getValueOffset(recordOffset, record.getKey());
    //内存索引信息（文件Id、文件位置、value的长度、序列号）
    return new InMemoryIndexMetaData(fileId, valueOffset, record.getValue().length, record.getSequenceNumber());
}
```

## 标记之前key的数据为无效状态

com.oath.halodb.HaloDBInternal#markPreviousVersionAsStale(byte[])

```java
private void markPreviousVersionAsStale(byte[] key) {
   //获取key的索引信息
    InMemoryIndexMetaData recordMetaData = inMemoryIndex.get(key);
    if (recordMetaData != null) {
        markPreviousVersionAsStale(key, recordMetaData);
    }
}
```

```java
private void markPreviousVersionAsStale(byte[] key, InMemoryIndexMetaData recordMetaData) {
    //失效数据的长度
    int staleRecordSize = Utils.getRecordSize(key.length, recordMetaData.getValueSize());
    addFileToCompactionQueueIfThresholdCrossed(recordMetaData.getFileId(), staleRecordSize);
}
```

```java
void addFileToCompactionQueueIfThresholdCrossed(int fileId, int staleRecordSize) {
  //根据fileId获取文件
    HaloDBFile file = readFileMap.get(fileId);
    if (file == null)
        return; 
    //获取此文件无效数据占用的空间
    int staleSizeInFile = updateStaleDataMap(fileId, staleRecordSize);
    //无效数据占用空间超过文件大小的75%，将此文件交由合并线程清除无效数据
    if (staleSizeInFile >= file.getSize() * options.getCompactionThresholdPerFile()) {
				//避免写线程和合并线程同时操作同一个文件
        if (getCurrentWriteFileId() != fileId && compactionManager.getCurrentWriteFileId() != fileId) {
            if(compactionManager.submitFileForCompaction(fileId)) {
                staleDataPerFileMap.remove(fileId);
            }
        }
    }
}
```

## 堆外内存保存索引

com.oath.halodb.OffHeapHashTableImpl#putInternal

```java
private boolean putInternal(byte[] key, V value, boolean ifAbsent, V old) {
    if (key == null || value == null) {
        throw new NullPointerException();
    }
    int valueSize = valueSize(value);
    if (valueSize != fixedValueLength) {
        throw new IllegalArgumentException("value size " + valueSize + " greater than fixed value size " + fixedValueLength);
    }
    if (old != null && valueSize(old) != fixedValueLength) {
        throw new IllegalArgumentException("old value size " + valueSize(old) + " greater than fixed value size " + fixedValueLength);
    }
    if (key.length > Byte.MAX_VALUE) {
        throw new IllegalArgumentException("key size of " + key.length + " exceeds max permitted size of " + Byte.MAX_VALUE);
    }
		//计算key的hash值
    long hash = hasher.hash(key);
    //获取segment保存索引信息
    return segment(hash).putEntry(key, value, hash, ifAbsent, old);
}
```

```java
boolean putEntry(byte[] key, V value, long hash, boolean putIfAbsent, V oldValue) {
    boolean wasFirst = lock();
    try {
        if (oldValue != null) {
            oldValueBuffer.clear();
            valueSerializer.serialize(oldValue, oldValueBuffer);
        }
        newValueBuffer.clear();
        //序列化索引信息，newValueBuffer中存储索引信息
        valueSerializer.serialize(value, newValueBuffer);
        //根据hash值获取所属的MemoryPoolChunk的index，及offset
        MemoryPoolAddress first = table.getFirst(hash);
        for (MemoryPoolAddress address = first; address.chunkIndex >= 0; address = getNext(address)) {
          //获取MemoryPoolChunk
            MemoryPoolChunk chunk = chunks.get(address.chunkIndex);
            //比较key的长度、内容是否相同
            if (chunk.compareKey(address.chunkOffset, key)) {
                if (putIfAbsent) {
                    return false;
                }
                if (oldValue != null) {
                    if (!chunk.compareValue(address.chunkOffset, oldValueBuffer.array())) {
                        return false;
                    }
                }
                //设置新value，替换旧value
                chunk.setValue(newValueBuffer.array(), address.chunkOffset);
                putReplaceCount++;
                return true;
            }
        }

        if (oldValue != null) {
            return false;
        }
        //扩容
        if (size >= threshold) {
            rehash();
            first = table.getFirst(hash);
        }
        // 添加索引信息到chunk中
        MemoryPoolAddress nextSlot = writeToFreeSlot(key, newValueBuffer.array(), first);
        table.addAsHead(hash, nextSlot);
        size++;
        putAddCount++;
    } finally {
        unlock(wasFirst);
    }
    return true;
}
```

```java
private MemoryPoolAddress writeToFreeSlot(byte[] key, byte[] value, MemoryPoolAddress nextAddress) {
    if (!freeListHead.equals(emptyAddress)) {
        // write to the head of the free list.
        MemoryPoolAddress temp = freeListHead;
        freeListHead = chunks.get(freeListHead.chunkIndex).getNextAddress(freeListHead.chunkOffset);
        chunks.get(temp.chunkIndex).fillSlot(temp.chunkOffset, key, value, nextAddress);
        --freeListSize;
        return temp;
    }
    //当前MemoryPoolChunk已写满
    if (currentChunkIndex == -1 || chunks.get(currentChunkIndex).remaining() < fixedSlotSize) {
        if (chunks.size() > Byte.MAX_VALUE) {
            logger.error("No more memory left. Each segment can have at most {} chunks.", Byte.MAX_VALUE + 1);
            throw new OutOfMemoryError("Each segment can have at most " + (Byte.MAX_VALUE + 1) + " chunks.");
        }
        //创建新的MemoryPoolChunk
        chunks.add(MemoryPoolChunk.create(chunkSize, fixedKeyLength, fixedValueLength));
        ++currentChunkIndex;
    }
    MemoryPoolChunk currentWriteChunk = chunks.get(currentChunkIndex);
    //索引信息存储的地址
    MemoryPoolAddress slotAddress = new MemoryPoolAddress(currentChunkIndex, currentWriteChunk.getWriteOffset());
    //填充索引信息到MemoryPoolChunk
    currentWriteChunk.fillNextSlot(key, value, nextAddress);
    return slotAddress;
}
```

```java
void fillSlot(int slotOffset, byte[] key, byte[] value, MemoryPoolAddress nextAddress) {
    if (key.length > fixedKeyLength || value.length != fixedValueLength) {
        throw new IllegalArgumentException(
            String.format("Invalid request. Key length %d. fixed key length %d. Value length %d",
                          key.length, fixedKeyLength, value.length)
        );
    }
    if (chunkSize - slotOffset < fixedSlotSize) {
        throw new IllegalArgumentException(
            String.format("Invalid offset %d. Chunk size %d. fixed slot size %d",
                          slotOffset, chunkSize, fixedSlotSize)
        );
    }
  	//hashcode相同的key，会通过nextAddress串起来
  	//0:slotOffset，1:nextAddress,5:key length,6:key ,6+fixedKeyLength:value
    setNextAddress(slotOffset, nextAddress);
    Uns.putByte(address, slotOffset + ENTRY_OFF_KEY_LENGTH, (byte) key.length);
    Uns.copyMemory(key, 0, address, slotOffset + ENTRY_OFF_DATA, key.length);
    setValue(value, slotOffset);
}
```



# 读数据

com.oath.halodb.HaloDBInternal#get(byte[], int)

```java
byte[] get(byte[] key, int attemptNumber) throws IOException, HaloDBException {
    if (attemptNumber > maxReadAttempts) {
        logger.error("Tried {} attempts but read failed", attemptNumber-1);
        throw new HaloDBException("Tried " + (attemptNumber-1) + " attempts but failed.");
    }
    //获取索引信息
    InMemoryIndexMetaData metaData = inMemoryIndex.get(key);
    if (metaData == null) {
        return null;
    }
    //获取所属文件
    HaloDBFile readFile = readFileMap.get(metaData.getFileId());
    if (readFile == null) {
        logger.debug("File {} not present. Compaction job would have deleted it. Retrying ...", metaData.getFileId());
        return get(key, attemptNumber+1);
    }
    try {
        //读文件
        return readFile.readFromFile(metaData.getValueOffset(), metaData.getValueSize());
    }
    catch (ClosedChannelException e) {
        if (!isClosing) {
            logger.debug("File {} was closed. Compaction job would have deleted it. Retrying ...", metaData.getFileId());
            return get(key, attemptNumber+1);
        }
        throw e;
    }
}
```

## 获取索引信息

com.oath.halodb.OffHeapHashTableImpl#get

```java
public V get(byte[] key) {
    if (key == null) {
        throw new NullPointerException();
    }

    KeyBuffer keySource = keySource(key);
    return segment(keySource.hash()).getEntry(keySource);
}
```



```java
public V getEntry(KeyBuffer key) {
    boolean wasFirst = lock();
    try {
        for (MemoryPoolAddress address = table.getFirst(key.hash());
             address.chunkIndex >= 0;
             address = getNext(address)) {
            MemoryPoolChunk chunk = chunks.get(address.chunkIndex);
            if (chunk.compareKey(address.chunkOffset, key.buffer)) {
                hitCount++;
                return valueSerializer.deserialize(chunk.readOnlyValueByteBuffer(address.chunkOffset));
            }
        }
        missCount++;
        return null;
    } finally {
        unlock(wasFirst);
    }
}
```

## 读取文件

```java
byte[] readFromFile(int offset, int length) throws IOException {
    byte[] value = new byte[length];
    ByteBuffer valueBuf = ByteBuffer.wrap(value);
    int read = readFromFile(offset, valueBuf);
    assert read == length;
    return value;
}
```

```java
int readFromFile(long position, ByteBuffer destinationBuffer) throws IOException {
    long currentPosition = position;
    int bytesRead;
    do {
        bytesRead = channel.read(destinationBuffer, currentPosition);
        currentPosition += bytesRead;
    } while (bytesRead != -1 && destinationBuffer.hasRemaining());
    return (int)(currentPosition - position);
}
```

# 删除数据

```java
void delete(byte[] key) throws IOException {
    writeLock.lock();
    try {
        //获取索引信息
        InMemoryIndexMetaData metaData = inMemoryIndex.get(key);
        if (metaData != null) {
            //从内存中移除
            inMemoryIndex.remove(key);
            //设置删除标志位
            TombstoneEntry entry =
                new TombstoneEntry(key, getNextSequenceNumber(), -1, Versions.CURRENT_TOMBSTONE_FILE_VERSION);
            currentTombstoneFile = rollOverTombstoneFile(entry, currentTombstoneFile);
            currentTombstoneFile.write(entry);
            //标记删除的key关联的数据为无效状态
            markPreviousVersionAsStale(key, metaData);
        }
    } finally {
        writeLock.unlock();
    }
}
```

com.oath.halodb.SegmentWithMemoryPool#removeEntry

```java
public boolean removeEntry(KeyBuffer key) {
    boolean wasFirst = lock();
    try {
        MemoryPoolAddress previous = null; //前一个地址;address：当前地址
        for (MemoryPoolAddress address = table.getFirst(key.hash());
             address.chunkIndex >= 0;
             previous = address, address = getNext(address)) {
            MemoryPoolChunk chunk = chunks.get(address.chunkIndex);
            if (chunk.compareKey(address.chunkOffset, key.buffer)) {
                removeInternal(address, previous, key.hash());
                removeCount++;
                size--;
                return true;
            }
        }
        return false;
    } finally {
        unlock(wasFirst);
    }
}
```

```java
private void removeInternal(MemoryPoolAddress address, MemoryPoolAddress previous, long hash) {
    //当前索引指向的下一个地址
    MemoryPoolAddress next = chunks.get(address.chunkIndex).getNextAddress(address.chunkOffset);
    if (table.getFirst(hash).equals(address)) {//删除地址等于头地址
        table.addAsHead(hash, next);
    } else if (previous == null) {
        //this should never happen. 
        throw new IllegalArgumentException("Removing entry which is not head but with previous null");
    } else { //修改前一个索引的nextoffst为删除索引的nextoffset
        chunks.get(previous.chunkIndex).setNextAddress(previous.chunkOffset, next);
    }
    //将删除的MemoryPoolAddress加到freeListHead中
    chunks.get(address.chunkIndex).setNextAddress(address.chunkOffset, freeListHead);
    freeListHead = address;
    ++freeListSize;
}
```

# 合并数据

com.oath.halodb.CompactionManager.CompactionThread#run

```java
public void run() {
    logger.info("Starting compaction thread ...");
    int fileToCompact = -1;

    while (isRunning) {
        try {
            //合并的文件
            fileToCompact = compactionQueue.take();
            if (fileToCompact == STOP_SIGNAL) {
                logger.debug("Received a stop signal.");
                // skip rest of the steps and check status of isRunning flag.
                // while pausing/stopping compaction isRunning flag must be set to false.
                continue;
            }
            logger.debug("Compacting {} ...", fileToCompact);
            //拷贝有效数据到新文件
            copyFreshRecordsToNewFile(fileToCompact);
            logger.debug("Completed compacting {} to {}", fileToCompact, getCurrentWriteFileId());
            dbInternal.markFileAsCompacted(fileToCompact);
            //删除合并的文件
            dbInternal.deleteHaloDBFile(fileToCompact);
        }
        catch (Exception e) {
            logger.error(String.format("Error while compacting file %d to %d", fileToCompact, getCurrentWriteFileId()), e);
        }
    }
    logger.info("Compaction thread stopped.");
}
```

