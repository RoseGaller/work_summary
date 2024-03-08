# LevelDB源码分析

* [写数据](#写数据)
  * [写前检查](#写前检查)
  * [写WAL](#写wal)
  * [写Memstore](#写memstore)
* [写sst文件](#写sst文件)
* [读数据](#读数据)
  * [sst文件中查找](#sst文件中查找)
* [删除数据](#删除数据)
* [快照snapshot](#快照snapshot)
* [合并](#合并)
  * [minor compaction](#minor-compaction)
  * [Pick Compaction](#pick-compaction)
  * [Major compaction](#major-compaction)
* [生成新版本](#生成新版本)
  * [记录删除、新增的文件](#记录删除新增的文件)
  * [计算新版本的各level对应的文件](#计算新版本的各level对应的文件)
  * [预先计算下次合并的level、score](#预先计算下次合并的levelscore)
  * [将新版本写入MANIFEST文件](#将新版本写入manifest文件)
  * [更改版本](#更改版本)
  * [释放旧版本](#释放旧版本)


# 写数据

org.iq80.leveldb.impl.DbImpl#writeInternal

```java
public Snapshot writeInternal(WriteBatchImpl updates, WriteOptions options)
        throws DBException
{
    mutex.lock();
    try {
        makeRoomForWrite(false);
        //生成开始、结束序列号
        final long sequenceBegin = versions.getLastSequence() + 1;
        final long sequenceEnd = sequenceBegin + updates.size() - 1;
        // Reserve this sequence in the version set
        versions.setLastSequence(sequenceEnd);
        // Log write
        Slice record = writeWriteBatch(updates, sequenceBegin);
        try {
            //写入WAL文件
            log.addRecord(record, options.sync());
        }
        catch (IOException e) {
            throw Throwables.propagate(e);
        }
        // 写入memstore
        updates.forEach(new InsertIntoHandler(memTable, sequenceBegin));
        return new SnapshotImpl(sequenceEnd);
    }
    finally {
        mutex.unlock();
    }
}
```

## 写前检查

```java
private void makeRoomForWrite(boolean force)
{
    Preconditions.checkState(mutex.isHeldByCurrentThread());
    boolean allowDelay = !force;
    while (true) {
        //level0的文件数大于8个，延迟写操作
        if (allowDelay && versions.numberOfFilesInLevel(0) > L0_SLOWDOWN_WRITES_TRIGGER) {
            try {
                mutex.unlock();
                Thread.sleep(1);
            }
            catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            } finally {
                mutex.lock();
            }
           //只会延迟一次写操作
            allowDelay = false;
        }
        else if (!force && memTable.approximateMemoryUsage() <= options.writeBufferSize()) { //memstore有剩余空间，中断循环
            break;
        }
        else if (immutableMemTable != null) { //没有剩余空间，并且之前的memstore还没有刷新到磁盘。有可能磁盘IO压力大，刷盘操作受阻
            backgroundCondition.awaitUninterruptibly();//阻塞直到之前的memstore刷新到磁盘
        }
        else if (versions.numberOfFilesInLevel(0) >= L0_STOP_WRITES_TRIGGER) { //level0存放文件超过12个，阻塞
            backgroundCondition.awaitUninterruptibly();
        }
        else {
            Preconditions.checkState(versions.getPrevLogNumber() == 0);
            try {
                //关闭之前的wal文件
                log.close();
            }
            catch (IOException e) {
                throw new RuntimeException("Unable to close log file " + log.getFile(), e);
            }
            //创建新的LogWriter
            long logNumber = versions.getNextFileNumber();
            try {
                this.log = Logs.createLogWriter(new File(databaseDir, Filename.logFileName(logNumber)), logNumber);
            }
            catch (IOException e) {
                throw new RuntimeException("Unable to open new log file " +
                        new File(databaseDir, Filename.logFileName(logNumber)).getAbsoluteFile(), e);
            }
            //创建新的mem table
            immutableMemTable = memTable;
            memTable = new MemTable(internalKeyComparator);
            force = false;
            //主动触发后台合并
            maybeScheduleCompaction();
        }
    }
}
```

## 写WAL

org.iq80.leveldb.impl.FileChannelLogWriter#addRecord

```java
public synchronized void addRecord(Slice record, boolean force)
        throws IOException
{
    checkState(!closed.get(), "Log has been closed");

    SliceInput sliceInput = record.input();
    boolean begin = true;
    do {
        //Block = 32kb
        int bytesRemainingInBlock = BLOCK_SIZE - blockOffset;
        checkState(bytesRemainingInBlock >= 0);
        if (bytesRemainingInBlock < HEADER_SIZE) { //当前block可用空间小于7字节
            if (bytesRemainingInBlock > 0) {
                // 剩余空间用0填充
                fileChannel.write(ByteBuffer.allocate(bytesRemainingInBlock));
            }
            blockOffset = 0;
            bytesRemainingInBlock = BLOCK_SIZE - blockOffset;
        }
        //减去7个字节的之后的剩余空间大小
        int bytesAvailableInBlock = bytesRemainingInBlock - HEADER_SIZE;
        checkState(bytesAvailableInBlock >= 0);
        boolean end;
        //本次可以写入磁盘的字节数
        int fragmentLength;
        //写入的数据太多，当前block存不下
        if (sliceInput.available() > bytesAvailableInBlock) {
            end = false;
            fragmentLength = bytesAvailableInBlock;
        }
        else { //当前的block可以存放写入的数据
            end = true;
            fragmentLength = sliceInput.available();
        }
        //决定block的类型
        LogChunkType type;
        if (begin && end) { //在一个block完全存下
            type = LogChunkType.FULL;
        }
        else if (begin) {//只存了开始一部分
            type = LogChunkType.FIRST;
        }
        else if (end) { //只存了最后一部分
            type = LogChunkType.LAST;
        }
        else { //只存了中间一部分
            type = LogChunkType.MIDDLE;
        }
        // write the chunk
        writeChunk(type, sliceInput.readSlice(fragmentLength));
        // we are no longer on the first chunk
        begin = false;
    } while (sliceInput.isReadable());
    //强制刷磁盘
    if (force) {
        fileChannel.force(false);
    }
}
```

## 写Memstore

org.iq80.leveldb.impl.DbImpl.InsertIntoHandler#put

```java
public void put(Slice key, Slice value)
{
    memTable.add(sequence++, VALUE, key, value);
}
```

# 写sst文件

org.iq80.leveldb.impl.DbImpl#buildTable

```java
private FileMetaData buildTable(SeekingIterable<InternalKey, Slice> data)
        throws IOException
{
    long fileNumber = versions.getNextFileNumber(); //文件编号
    pendingOutputs.add(fileNumber);
    File file = new File(databaseDir, Filename.tableFileName(fileNumber)); //创建文件
    try {
        FileChannel channel = new FileOutputStream(file).getChannel();
        TableBuilder tableBuilder = new TableBuilder(options, channel, new InternalUserComparator(internalKeyComparator));
        InternalKey smallest = null;
        InternalKey largest = null;
        for (Entry<InternalKey, Slice> entry : data) {
            // update keys
            InternalKey key = entry.getKey();
            if (smallest == null) {
                smallest = key;
            }
            largest = key;
            tableBuilder.add(key.encode(), entry.getValue()); //写数据
        }
        tableBuilder.finish();
        channel.force(true);
        channel.close();
        if (smallest == null) {
            return null;
        }
        FileMetaData fileMetaData = new FileMetaData(fileNumber, file.length(), smallest, largest);
        // verify table can be opened
        tableCache.newIterator(fileMetaData);
        pendingOutputs.remove(fileNumber);
        return fileMetaData;
    }
    catch (IOException e) {
        file.delete();
        throw e;
    }
}
```

org.iq80.leveldb.table.TableBuilder#add(org.iq80.leveldb.util.Slice, org.iq80.leveldb.util.Slice)

```java
public void add(Slice key, Slice value)
        throws IOException
{
    requireNonNull(key, "key is null");
    requireNonNull(value, "value is null");
    checkState(!closed, "table is finished");
    if (entryCount > 0) { //确保新写入的key大于上次写入的key，数据按照key排序
        assert (userComparator.compare(key, lastKey) > 0) : "key must be greater than last key";
    }
 
    if (pendingIndexEntry) {   //写索引
        checkState(dataBlockBuilder.isEmpty(), "Internal error: Table has a pending index entry but data block builder is empty");
        Slice shortestSeparator = userComparator.findShortestSeparator(lastKey, key);
        Slice handleEncoding = BlockHandle.writeBlockHandle(pendingHandle);
        indexBlockBuilder.add(shortestSeparator, handleEncoding);
        pendingIndexEntry = false;
    }
    lastKey = key;
    entryCount++;
    dataBlockBuilder.add(key, value); //写数据到dataBlockBuilder
    int estimatedBlockSize = dataBlockBuilder.currentSizeEstimate();
    if (estimatedBlockSize >= blockSize) { //超过一个块的大小
        flush();//将数据刷写到磁盘
    }
}
```

org.iq80.leveldb.table.BlockBuilder#add(org.iq80.leveldb.util.Slice, org.iq80.leveldb.util.Slice)

```java
public void add(Slice key, Slice value)
{
    requireNonNull(key, "key is null");
    requireNonNull(value, "value is null");
    checkState(!finished, "block is finished");
    checkPositionIndex(restartBlockEntryCount, blockRestartInterval);

    checkArgument(lastKey == null || comparator.compare(key, lastKey) > 0, "key must be greater than last key");

    int sharedKeyBytes = 0;
    if (restartBlockEntryCount < blockRestartInterval) { //默认每16条数据开始重新统计一次前缀
        sharedKeyBytes = calculateSharedBytes(key, lastKey); //获取共同前缀的长度
    }
    else { //写入完整的key
        // restart prefix compression
        restartPositions.add(block.size());
        restartBlockEntryCount = 0;
    }

    int nonSharedKeyBytes = key.length() - sharedKeyBytes;
    VariableLengthQuantity.writeVariableLengthInt(sharedKeyBytes, block); //共享的字节数
    VariableLengthQuantity.writeVariableLengthInt(nonSharedKeyBytes, block); //非共享的字节数
    VariableLengthQuantity.writeVariableLengthInt(value.length(), block); //value的长度

    block.writeBytes(key, sharedKeyBytes, nonSharedKeyBytes); //写入当前key与上一个key差异的部分

    block.writeBytes(value, 0, value.length()); //写入value
    lastKey = key;
    entryCount++;
    restartBlockEntryCount++;
}
```

org.iq80.leveldb.table.TableBuilder#flush

```java
private void flush()
        throws IOException
{
    checkState(!closed, "table is finished");
    if (dataBlockBuilder.isEmpty()) {
        return;
    }

    checkState(!pendingIndexEntry, "Internal error: Table already has a pending index entry to flush");

    pendingHandle = writeBlock(dataBlockBuilder); //写数据，获取写索引时的value
    pendingIndexEntry = true; //表明可以写索引
}
```

org.iq80.leveldb.table.TableBuilder#writeBlock

```java
private BlockHandle writeBlock(BlockBuilder blockBuilder)
        throws IOException
{
    // close the block
    Slice raw = blockBuilder.finish();

    // attempt to compress the block
    Slice blockContents = raw;
    CompressionType blockCompressionType = CompressionType.NONE;
    if (compressionType == CompressionType.SNAPPY) { //只支持Snappy压缩
        ensureCompressedOutputCapacity(maxCompressedLength(raw.length()));
        try {
          //压缩后占用的空间
            int compressedSize = Snappy.compress(raw.getRawArray(), raw.getRawOffset(), raw.length(), compressedOutput.getRawArray(), 0);

            ////压缩比小于12.5%不开启压缩
            if (compressedSize < raw.length() - (raw.length() / 8)) {
                blockContents = compressedOutput.slice(0, compressedSize);
                blockCompressionType = CompressionType.SNAPPY;
            }
        }
        catch (IOException ignored) {
            // compression failed, so just store uncompressed form
        }
    }

    // 压缩类型、校验和
    BlockTrailer blockTrailer = new BlockTrailer(blockCompressionType, crc32c(blockContents, blockCompressionType));
    Slice trailer = BlockTrailer.writeBlockTrailer(blockTrailer);

    // 写入的文件位置、写入的数据大小
    BlockHandle blockHandle = new BlockHandle(position, blockContents.length());

    // 写入数据、压缩类型和校验和
    position += fileChannel.write(new ByteBuffer[] {blockContents.toByteBuffer(), trailer.toByteBuffer()});

    blockBuilder.reset(); //重置blockBuilder

    return blockHandle;
}
```

# 读数据

1、先在memTable中查找

2、immutableMemTable中查找

3、sst文件中查找

org.iq80.leveldb.impl.DbImpl#get(byte[], ReadOptions)



```java

public byte[] get(byte[] key, ReadOptions options)
        throws DBException
{
    LookupKey lookupKey;
    mutex.lock();
    try {
        //获取快照号，默认获取最新的序列号
        long snapshot = getSnapshotNumber(options);
        //封装查询的key
        lookupKey = new LookupKey(Slices.wrappedBuffer(key), snapshot);
        //1、先在memTable中查找
        LookupResult lookupResult = memTable.get(lookupKey);
        if (lookupResult != null) {
            Slice value = lookupResult.getValue();
            if (value == null) {
                return null;
            }
            return value.getBytes();
        }
        //2、immutableMemTable中查找
        if (immutableMemTable != null) {
            lookupResult = immutableMemTable.get(lookupKey);
            if (lookupResult != null) {
                return lookupResult.getValue().getBytes();
            }
        }
    }
    finally {
        mutex.unlock();
    }
    //3、sst文件中查找
    LookupResult lookupResult = versions.get(lookupKey);
    // schedule compaction if necessary
    mutex.lock();
    try {
        if (versions.needsCompaction()) {//判断是否需要合并
            maybeScheduleCompaction(); //调度合并任务
        }
    }
    finally {
        mutex.unlock();
    }

    if (lookupResult != null) {
        Slice value = lookupResult.getValue();
        if (value != null) {
            return value.getBytes();
        }
    }
    return null;
}
```

## sst文件中查找

org.iq80.leveldb.impl.VersionSet#get

```java
public LookupResult get(LookupKey key)
{
    return current.get(key);
}
```

org.iq80.leveldb.impl.Version#get

```java
public LookupResult get(LookupKey key)
{
		
    ReadStats readStats = new ReadStats(); //读数据时的统计信息
    LookupResult lookupResult = level0.get(key, readStats); //先从level0查找
    if (lookupResult == null) { //其他level查找
        for (Level level : levels) {
            lookupResult = level.get(key, readStats);
            if (lookupResult != null) {
                break;
            }
        }
    }
    updateStats(readStats.getSeekFileLevel(), readStats.getSeekFile());
    return lookupResult;
}
```

org.iq80.leveldb.impl.Level0#get

```java
public LookupResult get(LookupKey key, ReadStats readStats)
{
    if (files.isEmpty()) {
        return null;
    }
    //查找文件:smallestKey <= lookupKey <= largetKey
    List<FileMetaData> fileMetaDataList = new ArrayList<>(files.size());
    for (FileMetaData fileMetaData : files) {
        if (internalKeyComparator.getUserComparator().compare(key.getUserKey(), fileMetaData.getSmallest().getUserKey()) >= 0 &&
                internalKeyComparator.getUserComparator().compare(key.getUserKey(), fileMetaData.getLargest().getUserKey()) <= 0) {
            fileMetaDataList.add(fileMetaData);
        }
    }
    //由于level0层的文件之间有重叠，需要按照文件名称倒序排序，number越大，文件越新
    Collections.sort(fileMetaDataList, NEWEST_FIRST);
    readStats.clear(); //清空统计信息
    for (FileMetaData fileMetaData : fileMetaDataList) {
        // open the iterator
        InternalTableIterator iterator = tableCache.newIterator(fileMetaData);
        // seek to the key
        iterator.seek(key.getInternalKey());
        if (iterator.hasNext()) {
            // parse the key in the block
            Entry<InternalKey, Slice> entry = iterator.next();
            InternalKey internalKey = entry.getKey();
            checkState(internalKey != null, "Corrupt key for %s", key.getUserKey().toString(UTF_8));

            // if this is a value key (not a delete) and the keys match, return the value
            if (key.getUserKey().equals(internalKey.getUserKey())) {
                if (internalKey.getValueType() == ValueType.DELETION) {
                    return LookupResult.deleted(key);
                }
                else if (internalKey.getValueType() == VALUE) {
                    return LookupResult.ok(key, entry.getValue());
                }
            }
        }

        if (readStats.getSeekFile() == null) {
            // We have had more than one seek for this read.  Charge the first file.
            readStats.setSeekFile(fileMetaData);
            readStats.setSeekFileLevel(0);
        }
    }

    return null;
}
```

org.iq80.leveldb.impl.Level#get

```java
public LookupResult get(LookupKey key, ReadStats readStats)
{
    if (files.isEmpty()) {
        return null;
    }

    List<FileMetaData> fileMetaDataList = new ArrayList<>(files.size());
    if (levelNumber == 0) {
        for (FileMetaData fileMetaData : files) {
            if (internalKeyComparator.getUserComparator().compare(key.getUserKey(), fileMetaData.getSmallest().getUserKey()) >= 0 &&
                    internalKeyComparator.getUserComparator().compare(key.getUserKey(), fileMetaData.getLargest().getUserKey()) <= 0) {
                fileMetaDataList.add(fileMetaData);
            }
        }
    }
    else {
        // Binary search to find earliest index whose largest key >= ikey.
        //二分查找
        int index = ceilingEntryIndex(Lists.transform(files, FileMetaData::getLargest), key.getInternalKey(), internalKeyComparator);

        // did we find any files that could contain the key?
        if (index >= files.size()) {
            return null;
        }

        // check if the smallest user key in the file is less than the target user key
        FileMetaData fileMetaData = files.get(index);
        if (internalKeyComparator.getUserComparator().compare(key.getUserKey(), fileMetaData.getSmallest().getUserKey()) < 0) {//查找的key小于文件中最小的key
            return null;
        }

        // search this file
        fileMetaDataList.add(fileMetaData);
    }

    FileMetaData lastFileRead = null;
    int lastFileReadLevel = -1;
    readStats.clear();
    for (FileMetaData fileMetaData : fileMetaDataList) {
        if (lastFileRead != null && readStats.getSeekFile() == null) {
            // We have had more than one seek for this read.  Charge the first file.
            readStats.setSeekFile(lastFileRead);
            readStats.setSeekFileLevel(lastFileReadLevel);
        }

        lastFileRead = fileMetaData;
        lastFileReadLevel = levelNumber;

        // open the iterator
        InternalTableIterator iterator = tableCache.newIterator(fileMetaData);

        // seek to the key
        iterator.seek(key.getInternalKey());

        if (iterator.hasNext()) {
            // parse the key in the block
            Entry<InternalKey, Slice> entry = iterator.next();
            InternalKey internalKey = entry.getKey();
            checkState(internalKey != null, "Corrupt key for %s", key.getUserKey().toString(UTF_8));

            // if this is a value key (not a delete) and the keys match, return the value
            if (key.getUserKey().equals(internalKey.getUserKey())) {
                if (internalKey.getValueType() == ValueType.DELETION) {
                    return LookupResult.deleted(key);
                }
                else if (internalKey.getValueType() == VALUE) {
                    return LookupResult.ok(key, entry.getValue());
                }
            }
        }
    }

    return null;
}
```

# 删除数据

跟追加数据类似，只是把value设置为空

```java
public void delete(byte[] key)
        throws DBException
{
    writeInternal(new WriteBatchImpl().delete(key), new WriteOptions());
}
```

# 快照snapshot

org.iq80.leveldb.impl.DbImpl#getSnapshot

```java
public Snapshot getSnapshot()
{
    mutex.lock();
    try {
        return new SnapshotImpl(versions.getLastSequence());//根据当前最新的sequence做快照，数据的sequence小于当前的sequence都可读
    }
    finally {
        mutex.unlock();
    }
}
```

# 合并

org.iq80.leveldb.impl.DbImpl#maybeScheduleCompaction

```java
private void maybeScheduleCompaction()
{//启动时调用
    Preconditions.checkState(mutex.isHeldByCurrentThread());
    if (backgroundCompaction != null) { //后台合并正在运行
    }
    else if (shuttingDown.get()) { //已经关闭
    }
    else if (immutableMemTable == null &&
            manualCompaction == null &&
            !versions.needsCompaction()) { //不需要合并(冻结的memstore为空、没有手动指定合并、没有搜索触发的合并、启动恢复时计算的合并分数小于1)
    }
    else {
        backgroundCompaction = compactionExecutor.submit(new Callable<Void>()
        {
            @Override
            public Void call()
                    throws Exception
            {
                try {
                    backgroundCall(); //后台合并
                }
                catch (DatabaseShutdownException ignored) {
                }
                catch (Throwable e) {
                    e.printStackTrace();
                }
                return null;
            }
        });
    }
}
```

org.iq80.leveldb.impl.DbImpl#backgroundCall

```java
private void backgroundCall()
        throws IOException
{ //后台调度合并任务
    mutex.lock(); //获取锁
    try {
        if (backgroundCompaction == null) {
            return;
        }
        try {
            if (!shuttingDown.get()) {
                backgroundCompaction(); //后台合并
            }
        }
        finally {
            backgroundCompaction = null; //当前合并已经完成
        }
        //先前的合并过程中在同一level又产生了许多的文件，然后重新调度合并
        maybeScheduleCompaction();
    }
    finally {
        try {
            backgroundCondition.signalAll(); //唤醒写阻塞的线程
        }
        finally {
            mutex.unlock(); //释放锁
        }
    }
}
```

后台合并

org.iq80.leveldb.impl.DbImpl#backgroundCompaction

```java
private void backgroundCompaction()
        throws IOException
{
    Preconditions.checkState(mutex.isHeldByCurrentThread());
    compactMemTableInternal(); //将immutableMemTable写入level0
    Compaction compaction;
    if (manualCompaction != null) {  //指定手动合并
        compaction = versions.compactRange(manualCompaction.level,
                new InternalKey(manualCompaction.begin, MAX_SEQUENCE_NUMBER, ValueType.VALUE),
                new InternalKey(manualCompaction.end, 0, ValueType.DELETION));
    } else { //数据太多触发的合并、搜索触发的合并
        compaction = versions.pickCompaction();
    }
  
    if (compaction == null) {
      
    } else if (manualCompaction == null && compaction.isTrivialMove()) {//level层只挑选一个文件并且与level+1层无重叠数据，直接将其移动到level+1，不涉及level+1层文件的读取
        // 将文件移动到level+1层
        Preconditions.checkState(compaction.getLevelInputs().size() == 1);
        FileMetaData fileMetaData = compaction.getLevelInputs().get(0);
        //记录将文件从level层移除
        compaction.getEdit().deleteFile(compaction.getLevel(), fileMetaData.getNumber());
        //记录将文件移动到level层
        compaction.getEdit().addFile(compaction.getLevel() + 1, fileMetaData);
        //应用文件版本变更
        versions.logAndApply(compaction.getEdit());
    } else {
        CompactionState compactionState = new CompactionState(compaction);
        //合并工作
        doCompactionWork(compactionState);
        //清理工作
        cleanupCompaction(compactionState);
    }
    if (manualCompaction != null) { //手动合并完成
        manualCompaction = null;
    }
```

## minor compaction

将immutableMemTable中的数据刷新到level0，相比其他level的合并具有更高的优先级

org.iq80.leveldb.impl.DbImpl#compactMemTableInternal

```java
private void compactMemTableInternal()
        throws IOException
{
    Preconditions.checkState(mutex.isHeldByCurrentThread());
    if (immutableMemTable == null) {
        return;
    }
    try {
        // 存储生成的SSTable的信息
        VersionEdit edit = new VersionEdit();
        //维护了各层中文件信息
        Version base = versions.getCurrent();
        //写immutableMemTable到level0
        writeLevel0Table(immutableMemTable, edit, base);
        if (shuttingDown.get()) {
            throw new DatabaseShutdownException("Database shutdown during memtable compaction");
        }
        edit.setPreviousLogNumber(0);
        // 之前的wal文件不再需要
        edit.setLogNumber(log.getFileNumber());  
        //将新增的sst文件应用到versions
        versions.logAndApply(edit);
        immutableMemTable = null;
        deleteObsoleteFiles();
    }
    finally {
        backgroundCondition.signalAll(); //唤醒因immutableMemTable!=null而导致写操作阻塞的线程
    }
}
```

org.iq80.leveldb.impl.DbImpl#writeLevel0Table

```java
private void writeLevel0Table(MemTable memTable, VersionEdit edit, Version base)
        throws IOException
{//写内存中不可变数据到level0,如果生成的文件在level-0层中没有重叠，尽可能放到更高层，默认最大推2层
    Preconditions.checkState(mutex.isHeldByCurrentThread());
    if (memTable.isEmpty()) {
        return;
    }
    FileMetaData fileMetaData;
    mutex.unlock();
    try {
      //创建sst文件
        fileMetaData = buildTable(memTable);
    }
    finally {
        mutex.lock();
    }
  	//sst文件的最小key
    Slice minUserKey = fileMetaData.getSmallest().getUserKey();
    //sst文件的最大key
    Slice maxUserKey = fileMetaData.getLargest().getUserKey();
    //生成的sst文件放到哪一个层
    int level = 0;
    //level0如果没有重合的文件，尽可能的将新文件放到更高的level
    if (base != null && !base.overlapInLevel(0, minUserKey, maxUserKey)) {
        while (level < MAX_MEM_COMPACT_LEVEL && !base.overlapInLevel(level + 1, minUserKey, maxUserKey)) {
            level++;
        }
    }
    //增加文件
    edit.addFile(level, fileMetaData);
}
```



## Pick Compaction

挑选合并的文件

org.iq80.leveldb.impl.VersionSet#pickCompaction

```java
public Compaction pickCompaction()
{
    //优先处理由某个level数据太多（level0数据文件太多，其他level数据量太多）触发的压缩
  	//而不是由搜索触发的压缩。
    //启动恢复时或者合并完成后会计算每层的合并分数
    boolean sizeCompaction = (current.getCompactionScore() >= 1);
    boolean seekCompaction = (current.getFileToCompact() != null);//根据key查找数据时，可能更改此值
    int level;
    List<FileMetaData> levelInputs;
    if (sizeCompaction) {//数据太多触发的合并
        level = current.getCompactionLevel();  //对compactionLevel进行合并
        checkState(level >= 0);
        checkState(level + 1 < NUM_LEVELS);
        // Pick the first file that comes after compact_pointer_[level]
        levelInputs = new ArrayList<>();
        for (FileMetaData fileMetaData : current.getFiles(level)) {
            if (!compactPointers.containsKey(level) ||
                    internalKeyComparator.compare(fileMetaData.getLargest(), compactPointers.get(level)) > 0) {
                levelInputs.add(fileMetaData);
                break;
            }
        }
        if (levelInputs.isEmpty()) {
            // Wrap-around to the beginning of the key space
            levelInputs.add(current.getFiles(level).get(0));
        }
    }
    else if (seekCompaction) { //搜索次数
        level = current.getFileToCompactLevel();
        levelInputs = ImmutableList.of(current.getFileToCompact());
    }
    else {
        return null;
    }
    //level0中每个文件都有可能重叠，挑出所有重叠文件，计算smallestKey、LargestKey
    if (level == 0) {
        Entry<InternalKey, InternalKey> range = getRange(levelInputs);
        levelInputs = getOverlappingInputs(0, range.getKey(), range.getValue());
        checkState(!levelInputs.isEmpty());
    }
    //从level+1层找出合并的合并wen'jian
    Compaction compaction = setupOtherInputs(level, levelInputs);
    return compaction;
}
```

org.iq80.leveldb.impl.VersionSet#setupOtherInputs

获取level、level+1甚至level+2层合并的文件

```java
private Compaction setupOtherInputs(int level, List<FileMetaData> levelInputs)
{
    //计算level层需要合并的文件的key范围，smallestKey、largestKey
    Entry<InternalKey, InternalKey> range = getRange(levelInputs);
    InternalKey smallest = range.getKey();
    InternalKey largest = range.getValue();
    //计算level+1层中重叠的文件
    List<FileMetaData> levelUpInputs = getOverlappingInputs(level + 1, smallest, largest);
    //计算level、level+1所有需合并文件的smallestKey、largestKey
    range = getRange(levelInputs, levelUpInputs);
    InternalKey allStart = range.getKey();
    InternalKey allLimit = range.getValue();
    // See if we can grow the number of inputs in "level" without
    // changing the number of "level+1" files we pick up.
    if (!levelUpInputs.isEmpty()) {
        //尽可能的增加level层合并的文件，但是不会引起level+1层需要合并的文件数
        List<FileMetaData> expanded0 = getOverlappingInputs(level, allStart, allLimit); //重新计算level层合并的文件
        if (expanded0.size() > levelInputs.size()) { //level层合并的文件有变
            range = getRange(expanded0); //获取变化之后的合并范围
            InternalKey newStart = range.getKey();
            InternalKey newLimit = range.getValue();
            List<FileMetaData> expanded1 = getOverlappingInputs(level + 1, newStart, newLimit); //level+1层合并的文件
            if (expanded1.size() == levelUpInputs.size()) { //level+1层合并的文件数没有发生变化
                smallest = newStart; //level层
                largest = newLimit; //level层
                levelInputs = expanded0;
                levelUpInputs = expanded1;
                range = getRange(levelInputs, levelUpInputs); //获取level、level+1 层文件的范围
                allStart = range.getKey(); //总体的最小key
                allLimit = range.getValue(); //总体的最大key
            }
        }
    }
    List<FileMetaData> grandparents = ImmutableList.of();
    if (level + 2 < NUM_LEVELS) { //尚未达到最大level
        grandparents = getOverlappingInputs(level + 2, allStart, allLimit); //获取level + 2层重叠的文件
    }
    //level：对哪一层合并
    //levelInputs：level层需要合并的文件
    //levelUpInputs: level+1层需要合并的文件
    //grandparents:level+2层需要合并的文件
    Compaction compaction = new Compaction(current, level, levelInputs, levelUpInputs, grandparents);
    compactPointers.put(level, largest);
    compaction.getEdit().setCompactPointer(level, largest); //对level层合并的最大key
    return compaction;
}
```

org.iq80.leveldb.impl.Compaction#Compaction

```java
public Compaction(Version inputVersion, int level, List<FileMetaData> levelInputs, List<FileMetaData> levelUpInputs, List<FileMetaData> grandparents)
{
    this.inputVersion = inputVersion;
    this.level = level;
    this.levelInputs = levelInputs;
    this.levelUpInputs = levelUpInputs;
    this.grandparents = ImmutableList.copyOf(requireNonNull(grandparents, "grandparents is null"));
    this.maxOutputFileSize = VersionSet.maxFileSizeForLevel(level);
    this.inputs = new List[] {levelInputs, levelUpInputs}; //level、level+1层文件
}
```

## Major compaction

org.iq80.leveldb.impl.DbImpl#doCompactionWork

```java
private void doCompactionWork(CompactionState compactionState)
        throws IOException
{
    Preconditions.checkState(mutex.isHeldByCurrentThread());
    Preconditions.checkArgument(versions.numberOfBytesInLevel(compactionState.getCompaction().getLevel()) > 0);
    Preconditions.checkArgument(compactionState.builder == null);
    Preconditions.checkArgument(compactionState.outfile == null);
    // todo track snapshots
    compactionState.smallestSnapshot = versions.getLastSequence();
    // Release mutex while we're actually doing the compaction work
    mutex.unlock(); //释放锁
    try {
        //通过优先级队列实现排序
        MergingIterator iterator = versions.makeInputIterator(compactionState.compaction);
        Slice currentUserKey = null;
        boolean hasCurrentUserKey = false;

        long lastSequenceForKey = MAX_SEQUENCE_NUMBER;
        while (iterator.hasNext() && !shuttingDown.get()) {

            mutex.lock(); //获取锁
            try {
                compactMemTableInternal(); //优先合并immutableMemTable
            }
            finally {
                mutex.unlock();
            }

            InternalKey key = iterator.peek().getKey();
            if (compactionState.compaction.shouldStopBefore(key) && compactionState.builder != null) {
                finishCompactionOutputFile(compactionState);
            }

            // Handle key/value, add to state, etc.
            boolean drop = false;
            // todo if key doesn't parse (it is corrupted),
            if (false /*!ParseInternalKey(key, &ikey)*/) {
                // do not hide error keys
                currentUserKey = null;
                hasCurrentUserKey = false;
                lastSequenceForKey = MAX_SEQUENCE_NUMBER;
            }
            else {
                if (!hasCurrentUserKey || internalKeyComparator.getUserComparator().compare(key.getUserKey(), currentUserKey) != 0) {
                    // First occurrence of this user key
                    currentUserKey = key.getUserKey();
                    hasCurrentUserKey = true;
                    lastSequenceForKey = MAX_SEQUENCE_NUMBER;
                }

                if (lastSequenceForKey <= compactionState.smallestSnapshot) {
                    // Hidden by an newer entry for same user key
                    drop = true; // (A)
                }
                else if (key.getValueType() == ValueType.DELETION &&
                        key.getSequenceNumber() <= compactionState.smallestSnapshot &&
                        compactionState.compaction.isBaseLevelForKey(key.getUserKey())) {

                    // For this user key:
                    // (1) there is no data in higher levels
                    // (2) data in lower levels will have larger sequence numbers
                    // (3) data in layers that are being compacted here and have
                    //     smaller sequence numbers will be dropped in the next
                    //     few iterations of this loop (by rule (A) above).
                    // Therefore this deletion marker is obsolete and can be dropped.
                    drop = true;
                }

                lastSequenceForKey = key.getSequenceNumber();
            }

            if (!drop) {//此key没有被删除
                // Open output file if necessary
                if (compactionState.builder == null) {
                    openCompactionOutputFile(compactionState); //创建新的sst文件存储数据
                }
                if (compactionState.builder.getEntryCount() == 0) {
                    compactionState.currentSmallest = key;//设置sst文件的最小key
                }
                compactionState.currentLargest = key;//设置sst文件最大key，文件中的数据按照key排序
                compactionState.builder.add(key.encode(), 
                                            iterator.peek().getValue());//写数据到sst文件

                // Close output file if it is big enough
                if (compactionState.builder.getFileSize() >=
                      compactionState.compaction.getMaxOutputFileSize()) {//sst文件大小达到上限
                    finishCompactionOutputFile(compactionState);
                }
            }
            iterator.next();
        }

        if (shuttingDown.get()) {
            throw new DatabaseShutdownException("DB shutdown during compaction");
        }
        if (compactionState.builder != null) {
            finishCompactionOutputFile(compactionState);
        }
    }
    finally {
        mutex.lock();
    }

    // todo port CompactionStats code

    installCompactionResults(compactionState);
}
```

创建MergingIterator

org.iq80.leveldb.impl.VersionSet#makeInputIterator

```java
public MergingIterator makeInputIterator(Compaction c)
{
  	//inputs[0] = levelInpus
    //inputs[1] = levelUpInputs
    //level-0使用Level0Iterator
    //其他level使用LevelIterator
    List<InternalIterator> list = new ArrayList<>();
    for (int which = 0; which < 2; which++) {
        if (!c.getInputs()[which].isEmpty()) {
            if (c.getLevel() + which == 0) { //level0
                List<FileMetaData> files = c.getInputs()[which];
                list.add(new Level0Iterator(tableCache, files, internalKeyComparator));
            }
            else { //非level 0
                // Create concatenating iterator for the files from this level
                list.add(Level.createLevelConcatIterator(tableCache, c.getInputs()[which], internalKeyComparator));
            }
        }
    }
   
    return new MergingIterator(list, internalKeyComparator);
}
```

创建Level0Iterator

org.iq80.leveldb.util.Level0Iterator#Level0Iterator(org.iq80.leveldb.impl.TableCache, java.util.List<org.iq80.leveldb.impl.FileMetaData>, java.util.Comparator<org.iq80.leveldb.impl.InternalKey>)

```java
public Level0Iterator(TableCache tableCache, List<FileMetaData> files, Comparator<InternalKey> comparator)
{
    Builder<InternalTableIterator> builder = ImmutableList.builder();
    for (FileMetaData file : files) {
        builder.add(tableCache.newIterator(file));
    }
    this.inputs = builder.build();
    this.comparator = comparator;
		//创建优先级队列
    this.priorityQueue = new PriorityQueue<>(Iterables.size(inputs) + 1);
    resetPriorityQueue(comparator);
}
```

org.iq80.leveldb.util.Level0Iterator#resetPriorityQueue

```java
private void resetPriorityQueue(Comparator<InternalKey> comparator)
{
    int i = 0;
    for (InternalTableIterator input : inputs) {
        if (input.hasNext()) { //把每个文件的第一条数据封装成ComparableIterator放入优先级队列
            priorityQueue.add(new ComparableIterator(input, comparator, i++, input.next()));
        }
    }
}
```

创建非level 0 的LevelIterator

org.iq80.leveldb.impl.Level#createLevelConcatIterator

```java
public static LevelIterator createLevelConcatIterator(TableCache tableCache, List<FileMetaData> files, InternalKeyComparator internalKeyComparator)
{
    return new LevelIterator(tableCache, files, internalKeyComparator);
}
```

org.iq80.leveldb.util.LevelIterator#LevelIterator

```java
public LevelIterator(TableCache tableCache, List<FileMetaData> files, InternalKeyComparator comparator)
{
    this.tableCache = tableCache;
    this.files = files;
    this.comparator = comparator;
}
```

org.iq80.leveldb.util.MergingIterator#MergingIterator

```java
public MergingIterator(List<? extends InternalIterator> levels, Comparator<InternalKey> comparator)
{
    this.levels = levels; //合并的文件
    this.comparator = comparator; //比较器

    this.priorityQueue = new PriorityQueue<>(levels.size() + 1);
    resetPriorityQueue(comparator);
}
```



org.iq80.leveldb.impl.DbImpl#installCompactionResults

```java
private void installCompactionResults(CompactionState compact)
        throws IOException
{
    Preconditions.checkState(mutex.isHeldByCurrentThread());

    //记录删除的文件
    compact.compaction.addInputDeletions(compact.compaction.getEdit());
    int level = compact.compaction.getLevel();
    //合并中新建的文件放到level+1层
    for (FileMetaData output : compact.outputs) {
        compact.compaction.getEdit().addFile(level + 1, output);
        pendingOutputs.remove(output.getNumber());
    }
    compact.outputs.clear();

    try {
      	//合并过程中发生了文件变更，创建新的Version，维护每层对应的文件
        versions.logAndApply(compact.compaction.getEdit());
        deleteObsoleteFiles();
    }
    catch (IOException e) {
        // Discard any files we may have created during this failed compaction
        for (FileMetaData output : compact.outputs) {
            new File(databaseDir, Filename.tableFileName(output.getNumber())).delete();
        }
    }
}
```

# 生成新版本

org.iq80.leveldb.impl.VersionSet#logAndApply

```java
public void logAndApply(VersionEdit edit)
        throws IOException
{//合并过程涉及到文件的新增、删除，将变更的文件与current version合并，生成new version
    if (edit.getLogNumber() != null) {
        checkArgument(edit.getLogNumber() >= logNumber);
        checkArgument(edit.getLogNumber() < nextFileNumber.get());
    }
    else {
        edit.setLogNumber(logNumber);
    }
    if (edit.getPreviousLogNumber() == null) {
        edit.setPreviousLogNumber(prevLogNumber);
    }
    edit.setNextFileNumber(nextFileNumber.get());
    edit.setLastSequenceNumber(lastSequence);
    //新版本，将新增的文件与之前存在的文件合并、删除合并中删除的文件
    Version version = new Version(this);
    Builder builder = new Builder(this, current);
    //记录每个level删除、新增的文件
    builder.apply(edit);
    //填充新版本
    builder.saveTo(version);
    //计算合并level、合并分数
    finalizeVersion(version);
    boolean createdNewManifest = false;
    try {
        if (descriptorLog == null) {
            edit.setNextFileNumber(nextFileNumber.get());
            //MANIFEST文件
            descriptorLog = Logs.createLogWriter(new File(databaseDir, Filename.descriptorFileName(manifestFileNumber)), manifestFileNumber);
            //写当前版本的快照到MANIFEST文件
            writeSnapshot(descriptorLog);
            createdNewManifest = true;
        }
        //当前合并中涉及到的文件变更记录到MANIFEST文件
        Slice record = edit.encode();
        descriptorLog.addRecord(record, true);
        if (createdNewManifest) { //将MANIFEST文件名记录到CURRENT文件中
            Filename.setCurrentFile(databaseDir, descriptorLog.getFileNumber());
        }
    }
    catch (IOException e) {
        if (createdNewManifest) {
            descriptorLog.close();
            new File(databaseDir, Filename.logFileName(descriptorLog.getFileNumber())).delete();
            descriptorLog = null;
        }
        throw e;
    }
    //每次合并生成新的版本
    appendVersion(version);
    logNumber = edit.getLogNumber();
    prevLogNumber = edit.getPreviousLogNumber();
}
```

## 记录删除、新增的文件

org.iq80.leveldb.impl.VersionSet.Builder#apply

```java
public void apply(VersionEdit edit)
{ //记录每个level删除、新增的文件
    // Update compaction pointers
    for (Entry<Integer, InternalKey> entry : edit.getCompactPointers().entrySet()) {
        Integer level = entry.getKey();
        InternalKey internalKey = entry.getValue();
        versionSet.compactPointers.put(level, internalKey);
    }
    //删除文件
    // Delete files
    for (Entry<Integer, Long> entry : edit.getDeletedFiles().entries()) {
        Integer level = entry.getKey();
        Long fileNumber = entry.getValue();
        levels.get(level).deletedFiles.add(fileNumber);
        // todo missing update to addedFiles?
    }
    // 新增文件
    for (Entry<Integer, FileMetaData> entry : edit.getNewFiles().entries()) {
        Integer level = entry.getKey();
        FileMetaData fileMetaData = entry.getValue();
        //设置经过多少次seek之后，将此文件进行合并
        int allowedSeeks = (int) (fileMetaData.getFileSize() / 16384);
        if (allowedSeeks < 100) {
            allowedSeeks = 100;
        }
        fileMetaData.setAllowedSeeks(allowedSeeks);
        levels.get(level).deletedFiles.remove(fileMetaData.getNumber());
        levels.get(level).addedFiles.add(fileMetaData);
    }
}
```

## 计算新版本的各level对应的文件

org.iq80.leveldb.impl.VersionSet.Builder#saveTo

```java
public void saveTo(Version version)
        throws IOException
{ //将合并前的文件和合并中新增的文件进行排序，过滤删除的文件，添加到new Version
    FileMetaDataBySmallestKey cmp = new FileMetaDataBySmallestKey(versionSet.internalKeyComparator);
    for (int level = 0; level < baseVersion.numberOfLevels(); level++) {
        //合并之前存在的文件
        Collection<FileMetaData> baseFiles = baseVersion.getFiles().asMap().get(level);
        if (baseFiles == null) {
            baseFiles = ImmutableList.of();
        }
        //合并中新增的文件
        SortedSet<FileMetaData> addedFiles = levels.get(level).addedFiles;
        if (addedFiles == null) {
            addedFiles = ImmutableSortedSet.of();
        }
        //排序
        ArrayList<FileMetaData> sortedFiles = new ArrayList<>(baseFiles.size() + addedFiles.size());
        sortedFiles.addAll(baseFiles);
        sortedFiles.addAll(addedFiles);
        Collections.sort(sortedFiles, cmp);

        //排序后的sst文件封装到new version中
        for (FileMetaData fileMetaData : sortedFiles) {
           //把文件放入对应的level，非level0层的文件之间无重叠
            maybeAddFile(version, level, fileMetaData);
        }
        version.assertNoOverlappingFiles(); //确保非level0层的文件之间无重叠
    }
}
```

org.iq80.leveldb.impl.VersionSet.Builder#maybeAddFile

```java
private void maybeAddFile(Version version, int level, FileMetaData fileMetaData)
        throws IOException
{
    if (levels.get(level).deletedFiles.contains(fileMetaData.getNumber())) { //文件删除
    }
    else {
        //
        List<FileMetaData> files = version.getFiles(level);
        //确保高于0层的文件之间无重叠
        if (level > 0 && !files.isEmpty()) {
            // Must not overlap
            boolean filesOverlap = versionSet.internalKeyComparator.compare(files.get(files.size() - 1).getLargest(), fileMetaData.getSmallest()) >= 0;
            if (filesOverlap) { //有重叠
                throw new IOException(String.format("Compaction is obsolete: Overlapping files %s and %s in level %s",
                        files.get(files.size() - 1).getNumber(),
                        fileMetaData.getNumber(), level));
            }
        }
        //新版本维护level -> file of level
        version.addFile(level, fileMetaData);
    }
}
```

## 预先计算下次合并的level、score

org.iq80.leveldb.impl.VersionSet#finalizeVersion

```java
private void finalizeVersion(Version version)
{//预先计算下次合并时的compactionLevel、compactionScore
    int bestLevel = -1;
    double bestScore = -1;
    for (int level = 0; level < version.numberOfLevels() - 1; level++) {
        double score;
        if (level == 0) {
            //通过限定文件的数量来处理level0而不是字节数
            //写缓冲区很大的话，不需要做许多的level0合并
            //level-0中的文件在每次读取和读取时合并因此，我们希望在个别时避免过多的文件 文件大小很小(可能是因为写缓冲区很小设置，或者非常高的压缩比，或者很多覆盖/删除)。
            score = 1.0 * version.numberOfFilesInLevel(level) / L0_COMPACTION_TRIGGER;
        }
        else {
            long levelBytes = 0;
            for (FileMetaData fileMetaData : version.getFiles(level)) {
                levelBytes += fileMetaData.getFileSize();
            }
            //计算百分比=当前leverl实际的字节大小/当前level的上限字节大小
            score = 1.0 * levelBytes / maxBytesForLevel(level);
        }
        if (score > bestScore) {
            bestLevel = level;
            bestScore = score;
        }
    }
    version.setCompactionLevel(bestLevel); //合并level
    version.setCompactionScore(bestScore); //合并分数
}
```

## 将新版本写入MANIFEST文件

```java
Slice record = edit.encode();
descriptorLog.addRecord(record, true); //刷写磁盘
if (createdNewManifest) { //将MANIFEST文件名记录到CURRENT文件中
    Filename.setCurrentFile(databaseDir, descriptorLog.getFileNumber());
 }
//CURRENT文件记录最新的MANIFEST文件名
```

## 更改版本

org.iq80.leveldb.impl.VersionSet#appendVersion

```java
private void appendVersion(Version version)
{
    requireNonNull(version, "version is null");
    checkArgument(version != current, "version is the current version");
    Version previous = current;
    current = version; //替换旧版本
    //新的版本加入到activeVersions中
    activeVersions.put(version, new Object());
    if (previous != null) {
        //释放旧的版本
        previous.release();
    }
}
```

## 释放旧版本

org.iq80.leveldb.impl.Version#release

```java
public void release()
{
    int now = retained.decrementAndGet(); //引用计数减一
    assert now >= 0 : "Version was released after it was disposed.";
    if (now == 0) { //无外部引用时，将此版本删除
        versionSet.removeVersion(this);
    }
}
```