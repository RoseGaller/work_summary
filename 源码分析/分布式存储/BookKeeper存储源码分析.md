# BookKeeper源码分析

* [处理写请求](#处理写请求)
  * [写请求入口](#写请求入口)
  * [追加数据](#追加数据)
  * [写内存](#写内存)
  * [内存快照](#内存快照)
  * [快照数据写入磁盘](#快照数据写入磁盘)
* [写数据到文件](#写数据到文件)
* [写索引到内存](#写索引到内存)
  * [获取LedgerEntryPage](#获取ledgerentrypage)
    * [内存中查找page](#内存中查找page)
    * [创建回收page](#创建回收page)
  * [写入索引信息](#写入索引信息)
* [读数据](#读数据)
  * [磁盘读取](#磁盘读取)
    * [从索引中获取存储位置](#从索引中获取存储位置)
    * [从磁盘读取数据](#从磁盘读取数据)
  * [内存读取](#内存读取)
* [OrderedExecutor](#orderedexecutor)
  * [创建线程池](#创建线程池)
  * [提交任务](#提交任务)
* [SkipListArena](#skiplistarena)
  * [获取当前chunk](#获取当前chunk)
  * [初始化chunk](#初始化chunk)
  * [分配bytes](#分配bytes)
* [总结](#总结)


# 处理写请求

根据请求的不同类型划分不同的线程池，避免慢请求干扰其他类型请求的执行；readThreadPool、writeThreadPool、longPollThreadPool、highPriorityThreadPool；

org.apache.bookkeeper.proto.BookieRequestProcessor#processAddRequestV3（根据请求的优先级选择不同的线程池异步追加数据）

```java
  private void processAddRequestV3(final BookkeeperProtocol.Request r, final Channel c) {
        WriteEntryProcessorV3 write = new WriteEntryProcessorV3(r, c, this);
        final OrderedExecutor threadPool;
        if (RequestUtils.isHighPriority(r)) { //高优先级，请求头属性priority>0,默认0
            threadPool = highPriorityThreadPool;
        } else { //普通追加请求
            threadPool = writeThreadPool;
        }
        if (null == threadPool) {
            write.run(); //直接运行
        } else {
            try {
                //根据ledgerId选择线程池
                threadPool.executeOrdered(r.getAddRequest().getLedgerId(), write);
            } catch (RejectedExecutionException e) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Failed to process request to add entry at {}:{}. Too many pending requests",
                              r.getAddRequest().getLedgerId(), r.getAddRequest().getEntryId());
                }
                BookkeeperProtocol.AddResponse.Builder addResponse = BookkeeperProtocol.AddResponse.newBuilder()
                        .setLedgerId(r.getAddRequest().getLedgerId())
                        .setEntryId(r.getAddRequest().getEntryId())
                        .setStatus(BookkeeperProtocol.StatusCode.ETOOMANYREQUESTS);
                BookkeeperProtocol.Response.Builder response = BookkeeperProtocol.Response.newBuilder()
                        .setHeader(write.getHeader())
                        .setStatus(addResponse.getStatus())
                        .setAddResponse(addResponse);
                BookkeeperProtocol.Response resp = response.build();
                write.sendResponse(addResponse.getStatus(), resp, requestStats.getAddRequestStats());
            }
        }
    }
```

## 写请求入口

org.apache.bookkeeper.proto.WriteEntryProcessorV3#safeRun

```java
public void safeRun() {
    //追加数据
    AddResponse addResponse = getAddResponse();
    if (null != addResponse) {
        // This means there was an error and we should send this back.
        Response.Builder response = Response.newBuilder()
                .setHeader(getHeader())
                .setStatus(addResponse.getStatus())
                .setAddResponse(addResponse);
        Response resp = response.build();
        sendResponse(addResponse.getStatus(), resp,
                     requestProcessor.getRequestStats().getAddRequestStats());
    }
}
```

org.apache.bookkeeper.proto.WriteEntryProcessorV3#getAddResponse

```java
private AddResponse getAddResponse() {
    final long startTimeNanos = MathUtils.nowInNano();
    AddRequest addRequest = request.getAddRequest();
    long ledgerId = addRequest.getLedgerId();
    long entryId = addRequest.getEntryId();

    final AddResponse.Builder addResponse = AddResponse.newBuilder()
            .setLedgerId(ledgerId)
            .setEntryId(entryId);

    if (!isVersionCompatible()) {
        addResponse.setStatus(StatusCode.EBADVERSION);
        return addResponse.build();
    }

    if (requestProcessor.getBookie().isReadOnly()
        && !(RequestUtils.isHighPriority(request)
                && requestProcessor.getBookie().isAvailableForHighPriorityWrites())) {
        logger.warn("BookieServer is running as readonly mode, so rejecting the request from the client!");
        addResponse.setStatus(StatusCode.EREADONLY);
        return addResponse.build();
    }
    //回调方法
    BookkeeperInternalCallbacks.WriteCallback wcb = new BookkeeperInternalCallbacks.WriteCallback() {
        @Override
        public void writeComplete(int rc, long ledgerId, long entryId,
                                  BookieSocketAddress addr, Object ctx) {
            if (BookieProtocol.EOK == rc) {
                requestProcessor.getRequestStats().getAddEntryStats()
                    .registerSuccessfulEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
            } else {
                requestProcessor.getRequestStats().getAddEntryStats()
                    .registerFailedEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
            }

            StatusCode status;
            switch (rc) {
                case BookieProtocol.EOK:
                    status = StatusCode.EOK;
                    break;
                case BookieProtocol.EIO:
                    status = StatusCode.EIO;
                    break;
                default:
                    status = StatusCode.EUA;
                    break;
            }
            addResponse.setStatus(status);
            Response.Builder response = Response.newBuilder()
                    .setHeader(getHeader())
                    .setStatus(addResponse.getStatus())
                    .setAddResponse(addResponse);
            Response resp = response.build();
          //发送响应
            sendResponse(status, resp, requestProcessor.getRequestStats().getAddRequestStats());
        }
    };
    final EnumSet<WriteFlag> writeFlags;
    if (addRequest.hasWriteFlags()) {
        writeFlags = WriteFlag.getWriteFlags(addRequest.getWriteFlags());
    } else {
        writeFlags = WriteFlag.NONE;
    }
    final boolean ackBeforeSync = writeFlags.contains(WriteFlag.DEFERRED_SYNC);
    StatusCode status = null;
    byte[] masterKey = addRequest.getMasterKey().toByteArray();
    //分配非池化的内存
    ByteBuf entryToAdd = Unpooled.wrappedBuffer(addRequest.getBody().asReadOnlyByteBuffer());
    try {
        if (RequestUtils.hasFlag(addRequest, AddRequest.Flag.RECOVERY_ADD)) {
            requestProcessor.getBookie().recoveryAddEntry(entryToAdd, wcb, channel, masterKey);
        } else { //追加数据
            requestProcessor.getBookie().addEntry(entryToAdd, ackBeforeSync, wcb, channel, masterKey);
        }
        status = StatusCode.EOK;
    } catch (OperationRejectedException e) {
        if (logger.isDebugEnabled()) {
            logger.debug("Operation rejected while writing {}", request, e);
        }
        status = StatusCode.EIO;
    } catch (IOException e) {
        logger.error("Error writing entry:{} to ledger:{}",
                entryId, ledgerId, e);
        status = StatusCode.EIO;
    } catch (BookieException.LedgerFencedException e) {
        logger.error("Ledger fenced while writing entry:{} to ledger:{}",
                entryId, ledgerId, e);
        status = StatusCode.EFENCED;
    } catch (BookieException e) {
        logger.error("Unauthorized access to ledger:{} while writing entry:{}",
                ledgerId, entryId, e);
        status = StatusCode.EUA;
    } catch (Throwable t) {
        logger.error("Unexpected exception while writing {}@{} : ",
                entryId, ledgerId, t);
        // some bad request which cause unexpected exception
        status = StatusCode.EBADREQ;
    }
    if (!status.equals(StatusCode.EOK)) {
        addResponse.setStatus(status);
        return addResponse.build();
    }
    return null;
}
```

## 追加数据

1、先写内存，再写日志

2、内存达到上限，将内存数据写入磁盘文件，建立索引

org.apache.bookkeeper.bookie.Bookie#addEntry

```java
public void addEntry(ByteBuf entry, boolean ackBeforeSync, WriteCallback cb, Object ctx, byte[] masterKey)
            throws IOException, BookieException, InterruptedException {
        long requestNanos = MathUtils.nowInNano();
        boolean success = false;
        int entrySize = 0;
        try {
            //获取对应的LedgerDescriptor
            LedgerDescriptor handle = getLedgerForEntry(entry, masterKey);
            synchronized (handle) {
                if (handle.isFenced()) {
                    throw BookieException
                            .create(BookieException.Code.LedgerFencedException);
                }
                entrySize = entry.readableBytes();
                //追加数据
                addEntryInternal(handle, entry, ackBeforeSync, cb, ctx, masterKey);
            }
            success = true;
        } catch (NoWritableLedgerDirException e) {
            stateManager.transitionToReadOnlyMode();
            throw new IOException(e);
        } finally {
          	//统计信息
            long elapsedNanos = MathUtils.elapsedNanos(requestNanos);
            if (success) {
                bookieStats.getAddEntryStats().registerSuccessfulEvent(elapsedNanos, TimeUnit.NANOSECONDS);
                bookieStats.getAddBytesStats().registerSuccessfulValue(entrySize);
            } else {
                bookieStats.getAddEntryStats().registerFailedEvent(elapsedNanos, TimeUnit.NANOSECONDS);
                bookieStats.getAddBytesStats().registerFailedValue(entrySize);
            }
            //释放内存
            entry.release();
        }
    }
```

org.apache.bookkeeper.bookie.Bookie#addEntryInternal

```java
long ledgerId = handle.getLedgerId();
long entryId = handle.addEntry(entry); //追加数据到内存
bookieStats.getWriteBytes().add(entry.readableBytes()); //统计写入的字节数
if (masterKeyCache.get(ledgerId) == null) {
    byte[] oldValue = masterKeyCache.putIfAbsent(ledgerId, masterKey);
    if (oldValue == null) {
        ByteBuffer bb = ByteBuffer.allocate(8 + 8 + 4 + masterKey.length);
        bb.putLong(ledgerId);
        bb.putLong(METAENTRY_ID_LEDGER_KEY);
        bb.putInt(masterKey.length);
        bb.put(masterKey);
        bb.flip();
        getJournal(ledgerId).logAddEntry(bb, false /* ackBeforeSync */, new NopWriteCallback(), null);
    }
}
if (LOG.isTraceEnabled()) {
    LOG.trace("Adding {}@{}", entryId, ledgerId);
}
//根据ledgerId获取Journal，将entry放入Journal的queue中 
getJournal(ledgerId).logAddEntry(entry, ackBeforeSync, cb, ctx); 
```

## 写内存

org.apache.bookkeeper.bookie.LedgerDescriptorImpl#addEntry

```java
long ledgerId = entry.getLong(entry.readerIndex());
if (ledgerId != this.ledgerId) {
    throw new IOException("Entry for ledger " + ledgerId + " was sent to " + this.ledgerId);
}
return ledgerStorage.addEntry(entry);
```

org.apache.bookkeeper.bookie.SortedLedgerStorage#addEntry

```java
long ledgerId = entry.getLong(entry.readerIndex() + 0);
long entryId = entry.getLong(entry.readerIndex() + 8);
long lac = entry.getLong(entry.readerIndex() + 16);
//写内存
memTable.addEntry(ledgerId, entryId, entry.nioBuffer(), this);
interleavedLedgerStorage.ledgerCache.updateLastAddConfirmed(ledgerId, lac);
return entryId;
```

org.apache.bookkeeper.bookie.EntryMemTable#addEntry

```java
public long addEntry(long ledgerId, long entryId, final ByteBuffer entry, final CacheCallback cb)
        throws IOException {
    long size = 0;
    long startTimeNanos = MathUtils.nowInNano();
    boolean success = false;
    try {
        //EntrySkipList容量达到上限64M或者之前的刷盘没有成功
        if (isSizeLimitReached() || (!previousFlushSucceeded.get())) {
            Checkpoint cp = snapshot(); //生成快照，将当前的kvmap赋值给snapshot，并生成新的kvmap
            if ((null != cp) || (!previousFlushSucceeded.get())) {
                cb.onSizeLimitReached(cp);
            }
        }

        final int len = entry.remaining();
        if (!skipListSemaphore.tryAcquire(len)) {
            memTableStats.getThrottlingCounter().inc();
            final long throttlingStartTimeNanos = MathUtils.nowInNano();
            skipListSemaphore.acquireUninterruptibly(len);
            memTableStats.getThrottlingStats()
                .registerSuccessfulEvent(MathUtils.elapsedNanos(throttlingStartTimeNanos), TimeUnit.NANOSECONDS);
        }

        this.lock.readLock().lock();
        try {
            //避免内存碎片，使用分配器分配内存，存储entry
            EntryKeyValue toAdd = cloneWithAllocator(ledgerId, entryId, entry);
            //放入ConcurrentSkipListMap
            size = internalAdd(toAdd);
        } finally {
            this.lock.readLock().unlock();
        }
        success = true;
        return size;
    } finally {
        if (success) {
            memTableStats.getPutEntryStats()
                .registerSuccessfulEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        } else {
            memTableStats.getPutEntryStats()
                .registerFailedEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        }
    }
}
```

## 内存快照

org.apache.bookkeeper.bookie.EntryMemTable#snapshot(org.apache.bookkeeper.bookie.CheckpointSource.Checkpoint)

```java
Checkpoint snapshot(Checkpoint oldCp) throws IOException {
    Checkpoint cp = null;
    // No-op if snapshot currently has entries
    if (this.snapshot.isEmpty() && this.kvmap.compareTo(oldCp) < 0) {
        final long startTimeNanos = MathUtils.nowInNano();
        this.lock.writeLock().lock();
        try {
            if (this.snapshot.isEmpty() && !this.kvmap.isEmpty()
                    && this.kvmap.compareTo(oldCp) < 0) {
                this.snapshot = this.kvmap;
                this.kvmap = newSkipList(); //创建新的EntrySkipList
                // get the checkpoint of the memtable.
                cp = this.kvmap.cp;
                // Reset heap to not include any keys
                this.size.set(0);
                // Reset allocator so we get a fresh buffer for the new EntryMemTable
                this.allocator = new SkipListArena(conf);
            }
        } finally {
            this.lock.writeLock().unlock();
        }

        if (null != cp) {
            memTableStats.getSnapshotStats()
                .registerSuccessfulEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        } else {
            memTableStats.getSnapshotStats()
                .registerFailedEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        }
    }
    return cp;
}
```

## 快照数据写入磁盘

org.apache.bookkeeper.bookie.SortedLedgerStorage#onSizeLimitReached

```java
public void onSizeLimitReached(final Checkpoint cp) throws IOException {
    LOG.info("Reached size {}", cp);
    scheduler.execute(new Runnable() {
        @Override
        public void run() {
            try {
                LOG.info("Started flushing mem table.");
                interleavedLedgerStorage.getEntryLogger().prepareEntryMemTableFlush();
                memTable.flush(SortedLedgerStorage.this);
                if (interleavedLedgerStorage.getEntryLogger().commitEntryMemTableFlush()) {
                    interleavedLedgerStorage.checkpointer.startCheckpoint(cp);
                }
            } catch (Exception e) {
                stateManager.transitionToReadOnlyMode();
                LOG.error("Exception thrown while flushing skip list cache.", e);
            }
        }
    });
}
```

org.apache.bookkeeper.bookie.EntryMemTable#flushSnapshot

```java
long flushSnapshot(final SkipListFlusher flusher, Checkpoint checkpoint) throws IOException {
    long size = 0;
    if (this.snapshot.compareTo(checkpoint) < 0) {
        long ledger, ledgerGC = -1;
        synchronized (this) {
            EntrySkipList keyValues = this.snapshot;
            if (keyValues.compareTo(checkpoint) < 0) {
                for (EntryKey key : keyValues.keySet()) {
                    EntryKeyValue kv = (EntryKeyValue) key;
                    size += kv.getLength();
                    ledger = kv.getLedgerId();
                    if (ledgerGC != ledger) {
                        try {
                            flusher.process(ledger, kv.getEntryId(), kv.getValueAsByteBuffer());
                        } catch (NoLedgerException exception) {
                            ledgerGC = ledger;
                        }
                    }
                }
                memTableStats.getFlushBytesCounter().add(size);
                clearSnapshot(keyValues);
            }
        }
    }
    skipListSemaphore.release((int) size);
    return size;
}
```

org.apache.bookkeeper.bookie.InterleavedLedgerStorage#processEntry(long, long, io.netty.buffer.ByteBuf, boolean)

```java
protected void processEntry(long ledgerId, long entryId, ByteBuf entry, boolean rollLog)
        throws IOException {
    somethingWritten.set(true);
    //写数据到文件
    long pos = entryLogger.addEntry(ledgerId, entry, rollLog);
    //写索引到内存
    ledgerCache.putEntryOffset(ledgerId, entryId, pos);
}
```

# 写数据到文件

org.apache.bookkeeper.bookie.EntryLogManagerBase#addEntry

```java
public long addEntry(long ledger, ByteBuf entry, boolean rollLog) throws IOException {
    int entrySize = entry.readableBytes() + 4; // Adding 4 bytes to prepend the size
    BufferedLogChannel logChannel = getCurrentLogForLedgerForAddEntry(ledger, entrySize, rollLog);
    ByteBuf sizeBuffer = sizeBufferForAdd.get();
    sizeBuffer.clear();
    sizeBuffer.writeInt(entry.readableBytes());
    logChannel.write(sizeBuffer);

    long pos = logChannel.position();
    logChannel.write(entry);
    logChannel.registerWrittenEntry(ledger, entrySize);
    return (logChannel.getLogId() << 32L) | pos;
}
```

# 写索引到内存

1、获取所属的page

2、写入文件中的位置信息

org.apache.bookkeeper.bookie.IndexInMemPageMgr#putEntryOffset

```java
void putEntryOffset(long ledger, long entry, long offset) throws IOException {
    //默认entriesPerPage=1024，获取在page中的offset
    int offsetInPage = (int) (entry % entriesPerPage);
    //获取所属page的开始offset
    long pageEntry = entry - offsetInPage;
    LedgerEntryPage lep = null;
    try {
      	//获取所属的page
        lep = getLedgerEntryPage(ledger, pageEntry);
        assert lep != null;
        //写入offset
        lep.setOffset(offset, offsetInPage * LedgerEntryPage.getIndexEntrySize());
    } catch (FileInfo.FileInfoDeletedException e) {
        throw new Bookie.NoLedgerException(ledger);
    } finally {
        if (null != lep) {
            lep.releasePage();
        }
    }
}
```

## 获取LedgerEntryPage

```java
LedgerEntryPage getLedgerEntryPage(long ledger,
                                   long pageEntry) throws IOException {
  //从内存中查找LedgerEntryPage
  LedgerEntryPage lep = getLedgerEntryPageFromCache(ledger, pageEntry, false);
  if (lep == null) {
    ledgerCacheMissCounter.inc();
    //创建或者回收page或者刷新page到磁盘
    lep = grabLedgerEntryPage(ledger, pageEntry);
  } else {
    ledgerCacheHitCounter.inc();
  }
  return lep;
}
```

### 内存中查找page

```java
LedgerEntryPage getLedgerEntryPageFromCache(long ledger,
                                                   long firstEntry,
                                                   boolean onlyDirty) {
   //从内存中查找LedgerEntryPage
    LedgerEntryPage lep = pageMapAndList.getPage(ledger, firstEntry);
    if (onlyDirty && null != lep && lep.isClean()) {
        return null;
    }
    if (null != lep) {
        lep.usePage();
    }
    return lep;
}
```

### 创建回收page

org.apache.bookkeeper.bookie.IndexInMemPageMgr#grabLedgerEntryPage

```java
private LedgerEntryPage grabLedgerEntryPage(long ledger, long pageEntry) throws IOException {
  	//创建回收page
    LedgerEntryPage lep = grabCleanPage(ledger, pageEntry);
    try {
        Stopwatch readPageStopwatch = Stopwatch.createStarted();
        boolean isNewPage = indexPersistenceManager.updatePage(lep);
        if (!isNewPage) {
            ledgerCacheReadPageStats.registerSuccessfulEvent(
                    readPageStopwatch.elapsed(TimeUnit.MICROSECONDS),
                    TimeUnit.MICROSECONDS);
        }
    } catch (IOException ie) {
        lep.releasePageNoCallback();
        pageMapAndList.addToListOfFreePages(lep);
        throw ie;
    }
    LedgerEntryPage oldLep;
    if (lep != (oldLep = pageMapAndList.putPage(lep))) {
        lep.releasePageNoCallback();
        pageMapAndList.addToListOfFreePages(lep);
        oldLep.usePage();
        lep = oldLep;
    }
    return lep;
}
```

org.apache.bookkeeper.bookie.IndexInMemPageMgr#grabCleanPage

```java
private LedgerEntryPage grabCleanPage(long ledger, long entry) throws IOException {
    if (entry % entriesPerPage != 0) {
        throw new IllegalArgumentException(entry +" is not a multiple of " +entriesPerPage);
    }
    while (true) {
        boolean canAllocate = false;
        if (pageCount.incrementAndGet() <= pageLimit) { //没有达到上限
            canAllocate = true;
        } else {
            pageCount.decrementAndGet();
        }
        //可以分配，创建LedgerEntryPage
        if (canAllocate) {
            LedgerEntryPage lep = new LedgerEntryPage(pageSize, entriesPerPage, pageMapAndList);
            lep.setLedgerAndFirstEntry(ledger, entry);
            lep.usePage();
            return lep;
        }
        //回收干净page
        LedgerEntryPage lep = pageMapAndList.grabCleanPage(ledger, entry);
        if (null != lep) {
            return lep;
        }
        LOG.info("Could not grab a clean page for ledger {}, entry {}, force flushing dirty ledgers.",ledger, entry);
        //刷新LedgerEntryPage到磁盘
        flushOneOrMoreLedgers(false);
    }
}
```

## 写入索引信息

org.apache.bookkeeper.bookie.LedgerEntryPage#setOffset

```java
public void setOffset(long offset, int position) {
    checkPage();
    page.putLong(position, offset);
    version.incrementAndGet();
    if (last < position / getIndexEntrySize()) {
        last = position / getIndexEntrySize();
    }
    this.clean = false;

    if (null != callback) {
        callback.onSetDirty(this);
    }
}
```

# 读数据

根据ledgerId、entryId读取数据

org.apache.bookkeeper.bookie.SortedLedgerStorage#getEntry

```java
public ByteBuf getEntry(long ledgerId, long entryId) throws IOException {
    if (entryId == BookieProtocol.LAST_ADD_CONFIRMED) {
        return getLastEntryId(ledgerId);
    }
    ByteBuf buffToRet;
    try {
        //1.从磁盘读取
        buffToRet = interleavedLedgerStorage.getEntry(ledgerId, entryId);
    } catch (Bookie.NoEntryException nee) {
        //2.从内存读取
        EntryKeyValue kv = memTable.getEntry(ledgerId, entryId);
        if (null == kv) {
            //从文件读取
            buffToRet = interleavedLedgerStorage.getEntry(ledgerId, entryId);
        } else {
            buffToRet = kv.getValueAsByteBuffer();
        }
    }
    return buffToRet;
}
```

## 磁盘读取

org.apache.bookkeeper.bookie.InterleavedLedgerStorage#getEntry

```java
public ByteBuf getEntry(long ledgerId, long entryId) throws IOException {
    long offset;
    if (entryId == BookieProtocol.LAST_ADD_CONFIRMED) {
        entryId = ledgerCache.getLastEntry(ledgerId); //返回上次写入的entryId
    }
    // Get Offset
    long startTimeNanos = MathUtils.nowInNano();
    boolean success = false;
    try {
        //1.根据ledgerId、entryId获取offset
        offset = ledgerCache.getEntryOffset(ledgerId, entryId);
        if (offset == 0) { //不存在
            throw new Bookie.NoEntryException(ledgerId, entryId);
        }
        success = true;
    } finally {
        if (success) {
            getOffsetStats.registerSuccessfulEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        } else {
            getOffsetStats.registerFailedEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        }
    }
   //Get Entry
    startTimeNanos = MathUtils.nowInNano();
    success = false;
    try {
        //2.从文件读取数据
        ByteBuf retBytes = entryLogger.readEntry(ledgerId, entryId, offset);
        success = true;
        return retBytes;
    } finally {
        if (success) {
            getEntryStats.registerSuccessfulEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        } else {
            getEntryStats.registerFailedEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        }
    }
}
```

### 从索引中获取存储位置

org.apache.bookkeeper.bookie.IndexInMemPageMgr#getEntryOffset

```java
long getEntryOffset(long ledger, long entry) throws IOException {
    //计算在page中的offset
    int offsetInPage = (int) (entry % entriesPerPage);
    // find the id of the first entry of the page that has the entry
    // we are looking for
    //计算所属page的开始offset
    long pageEntry = entry - offsetInPage;
    LedgerEntryPage lep = null;
    try {
        //获取LedgerEntryPage
        lep = getLedgerEntryPage(ledger, pageEntry);
        //获取位置信息
        return lep.getOffset(offsetInPage  * LedgerEntryPage.getIndexEntrySize());
    } finally {
        if (lep != null) {
            lep.releasePage();
        }
    }
}
```

### 从磁盘读取数据

org.apache.bookkeeper.bookie.EntryLogger#internalReadEntry

```java
public ByteBuf internalReadEntry(long ledgerId, long entryId, long location, boolean validateEntry)
        throws IOException, Bookie.NoEntryException {
    //计算logId
    long entryLogId = logIdForOffset(location);
    //计算pos
    long pos = posForOffset(location);
    BufferedReadChannel fc = null;
    int entrySize = -1;
    try {
        fc = getFCForEntryInternal(ledgerId, entryId, entryLogId, pos);
        //获取size、legerId、entryId
        ByteBuf sizeBuff = readEntrySize(ledgerId, entryId, entryLogId, pos, fc);
        entrySize = sizeBuff.getInt(0);
        if (validateEntry) { //验证Entry
            validateEntry(ledgerId, entryId, entryLogId, pos, sizeBuff);
        }
    } catch (EntryLookupException.MissingEntryException entryLookupError) {
        throw new Bookie.NoEntryException("Short read from entrylog " + entryLogId,
                ledgerId, entryId);
    } catch (EntryLookupException e) {
        throw new IOException(e.toString());
    }
    //分配ByteBuf
    ByteBuf data = allocator.buffer(entrySize, entrySize);
    //读取数据
    int rc = readFromLogChannel(entryLogId, fc, data, pos);
    if (rc != entrySize) {
        data.release();
        throw new Bookie.NoEntryException("Short read for " + ledgerId + "@"
                                          + entryId + " in " + entryLogId + "@"
                                          + pos + "(" + rc + "!=" + entrySize + ")", ledgerId, entryId);
    }
    data.writerIndex(entrySize);
    return data;
}
```

## 内存读取

org.apache.bookkeeper.bookie.EntryMemTable#getEntry

```java
public EntryKeyValue getEntry(long ledgerId, long entryId) throws IOException {
    EntryKey key = new EntryKey(ledgerId, entryId);
    EntryKeyValue value = null;
    long startTimeNanos = MathUtils.nowInNano();
    boolean success = false;
    this.lock.readLock().lock();
    try {
      //从当前正在写的EntrySkipList中获取数据
        value = this.kvmap.get(key);
        if (value == null) {
            value = this.snapshot.get(key); //从之前的EntrySkipList获取数据
        }
        success = true;
    } finally {
        this.lock.readLock().unlock();
        if (success) {
            memTableStats.getGetEntryStats()
                .registerSuccessfulEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        } else {
            memTableStats.getGetEntryStats()
                .registerFailedEvent(MathUtils.elapsedNanos(startTimeNanos), TimeUnit.NANOSECONDS);
        }
    }

    return value;
}
```

# OrderedExecutor

创建多个线程数为1的线程池，并将每个线程和CPU绑定

提交任务时，根据ledgerId计算使用哪个线程池执行任务。legerId相同的任务都在同一线程池运行

## 创建线程池

org.apache.bookkeeper.common.util.OrderedExecutor#OrderedExecutor

```java
protected OrderedExecutor(String baseName, int numThreads, ThreadFactory threadFactory,
                            StatsLogger statsLogger, boolean traceTaskExecution,
                            boolean preserveMdcForTaskExecution, long warnTimeMicroSec, int maxTasksInQueue,
                            boolean enableBusyWait) {
    checkArgument(numThreads > 0);
    checkArgument(!StringUtils.isBlank(baseName));

    this.maxTasksInQueue = maxTasksInQueue;
    this.warnTimeMicroSec = warnTimeMicroSec;
    this.enableBusyWait = enableBusyWait;
    name = baseName;
    threads = new ExecutorService[numThreads];
    threadIds = new long[numThreads];
    for (int i = 0; i < numThreads; i++) {
        //创建线程数为1的线程池
        ThreadPoolExecutor thread = createSingleThreadExecutor(
                new ThreadFactoryBuilder().setNameFormat(name + "-" + getClass().getSimpleName() + "-" + i + "-%d")
                .setThreadFactory(threadFactory).build());

        //对ThreadPoolExecutor进行功能增强，限制执行的任务数
        threads[i] = addExecutorDecorators(getBoundedExecutor(thread));

        final int idx = i;
        try {
            //将线程和cpu绑定
            threads[idx].submit(() -> {
                threadIds[idx] = Thread.currentThread().getId();

                if (enableBusyWait) {
                    // Try to acquire 1 CPU core to the executor thread. If it fails we
                    // are just logging the error and continuing, falling back to
                    // non-isolated CPUs.
                    try {
                        CpuAffinity.acquireCore();
                    } catch (Throwable t) {
                        log.warn("Failed to acquire CPU core for thread {}", Thread.currentThread().getName(),
                                t.getMessage(), t);
                    }
                }
            }).get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Couldn't start thread " + i, e);
        } catch (ExecutionException e) {
            throw new RuntimeException("Couldn't start thread " + i, e);
        }

        // Register gauges
        statsLogger.registerGauge(String.format("%s-queue-%d", name, idx), new Gauge<Number>() {
            @Override
            public Number getDefaultValue() {
                return 0;
            }

            @Override
            public Number getSample() {
                return thread.getQueue().size();
            }
        });
        statsLogger.registerGauge(String.format("%s-completed-tasks-%d", name, idx), new Gauge<Number>() {
            @Override
            public Number getDefaultValue() {
                return 0;
            }

            @Override
            public Number getSample() {
                return thread.getCompletedTaskCount();
            }
        });
        statsLogger.registerGauge(String.format("%s-total-tasks-%d", name, idx), new Gauge<Number>() {
            @Override
            public Number getDefaultValue() {
                return 0;
            }

            @Override
            public Number getSample() {
                return thread.getTaskCount();
            }
        });
    }
    // Stats
    this.taskExecutionStats = statsLogger.scope(name).getOpStatsLogger("task_execution");
    this.taskPendingStats = statsLogger.scope(name).getOpStatsLogger("task_queued");
    this.traceTaskExecution = traceTaskExecution;
    this.preserveMdcForTaskExecution = preserveMdcForTaskExecution;
}
```

## 提交任务

org.apache.bookkeeper.common.util.OrderedExecutor#executeOrdered(long, org.apache.bookkeeper.common.util.SafeRunnable)

```java
public void executeOrdered(long orderingKey, SafeRunnable r) {
    chooseThread(orderingKey).execute(r); //先选择线程池
}
```

```java
public ExecutorService chooseThread(long orderingKey) {
    if (threads.length == 1) {
        return threads[0];
    }

    return threads[MathUtils.signSafeMod(orderingKey, threads.length)];
}
```

org.apache.bookkeeper.common.util.MathUtils#signSafeMod

```java
public static int signSafeMod(long dividend, int divisor) {
    int mod = (int) (dividend % divisor);

    if (mod < 0) {
        mod += divisor;
    }

    return mod;
}
```

# SkipListArena

以chunk为单位进行内存的申请，默认大小4M，避免了内存碎片。

如果分配大小超过默认值128Kb，不再通过分配器分配。

## 获取当前chunk

org.apache.bookkeeper.bookie.SkipListArena#getCurrentChunk

```java
private Chunk getCurrentChunk() {
    while (true) {
        // 获取chunk
        Chunk c = curChunk.get();
        if (c != null) {
            return c;
        }
				//初次获取或者之前的已经被释放
        c = new Chunk(chunkSize);//默认4M
        if (curChunk.compareAndSet(null, c)) {
            c.init();//初始化chunk
            return c;
        }
        // lost race
    }
}
```

## 初始化chunk

org.apache.bookkeeper.bookie.SkipListArena.Chunk#init

```java
public void init() {
    //nextFreeOffset:下次分配的开始偏移量
    assert nextFreeOffset.get() == UNINITIALIZED;
    try {
        data = new byte[size];
    } catch (OutOfMemoryError e) {
        boolean failInit = nextFreeOffset.compareAndSet(UNINITIALIZED, OOM);
        assert failInit; // should be true.
        throw e;
    }
    // Mark that it's ready for use
    boolean okInit = nextFreeOffset.compareAndSet(UNINITIALIZED, 0);
    assert okInit;    // single-threaded call
}
```

## 分配bytes

org.apache.bookkeeper.bookie.SkipListArena#allocateBytes

```java
public MemorySlice allocateBytes(int size) {
    assert size >= 0;
    if (size > maxAlloc) { //默认128kb，过大不通过分配器进行分配
        return null;
    }

    while (true) {
        //获取当前的Chunk
        Chunk c = getCurrentChunk();
				//尝试从当前的Chunk分配内存
        int allocOffset = c.alloc(size);//chunk中的偏移量
        if (allocOffset != -1) {//成功分配
            return new MemorySlice(c.data, allocOffset);
        }
        //当前chunk内存不足，释放此chunk
        retireCurrentChunk(c);
    }
}
```

org.apache.bookkeeper.bookie.SkipListArena.Chunk#alloc

```java
public int alloc(int size) {
    while (true) {
        int oldOffset = nextFreeOffset.get();
        if (oldOffset == UNINITIALIZED) {//当前的chunk尚未初始化
            Thread.yield();
            continue;
        }
        if (oldOffset == OOM) { //发生了OOM
            return -1;
        }

        if (oldOffset + size > data.length) { //chunk内存不足
            return -1; // alloc doesn't fit
        }

        // Try to atomically claim this chunk
        if (nextFreeOffset.compareAndSet(oldOffset, oldOffset + size)) { //修改下次分配的开始偏移量
            allocCount.incrementAndGet();
            return oldOffset;
        }
    }
}
```

# 总结

1、请求隔离，不同的请求使用不同的线程池执行，避免慢请求对其他请求造成延迟

2、将线程绑定CPU，避免线程的来回切换，提高缓存命中率

3、根据场景的不同，选择合适的队列，比如多生产者单消费者场景下选择MpscArrayQueue

4、避免内存碎片，创建内存分配器，将内存碎片的粒度增大

