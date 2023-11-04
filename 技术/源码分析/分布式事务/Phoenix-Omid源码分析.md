# Phoenix-Omid源码分析

* [概述](#概述)
  * [写事务流程](#写事务流程)
  * [读事务流程](#读事务流程)
* [Quikstart](#quikstart)
* [流程](#流程)
  * [开启事务](#开启事务)
  * [提交事务](#提交事务)
* [TSO-Server](#tso-server)
  * [处理请求入口](#处理请求入口)
  * [发布事件](#发布事件)
    * [发布获取时间戳的事件](#发布获取时间戳的事件)
    * [发布提交事务的事件](#发布提交事务的事件)
  * [请求事件处理](#请求事件处理)
    * [事务开始时获取时间戳](#事务开始时获取时间戳)
    * [事务提交时获取时间戳](#事务提交时获取时间戳)
  * [响应事件处理](#响应事件处理)
    * [缓存Put请求](#缓存put请求)
    * [将响应信息存放至events数组](#将响应信息存放至events数组)
    * [将缓存的Put发送至服务端、返回响应信息给客户端](#将缓存的put发送至服务端返回响应信息给客户端)
    * [将缓存的Put请求发送至Hbase服务端](#将缓存的put请求发送至hbase服务端)
    * [发送响应信息给客户端](#发送响应信息给客户端)


# 概述

## 写事务流程

1、启动事务时，客户端从TSO请求一个开始时间戳。写数据时，用开始时间戳作为数据单元格的版本。

2、提交事务时，客户端将写集合发送给TSO, TSO将检查冲突。如果没有冲突，TSO将给事务分配一个提交时间戳，生成一条开始时间戳与提交时间戳的映射数据，将数据持久化到提交表。最后提交时间戳就返回给客户端。

3、客户端收到提交时间戳，更新影子单元格（rowkey，family,qualifer with Shadow CellSuffix,start Timestamp,commit Timestamp），并从提交表中删除映射数据。

## 读事务流程

1、读事务启动时，客户端从TSO请求一个开始时间戳

2、客户端首先检查是否有影子单元格

3、如果影子单元格的提交时间小于当前读事务的开始时间，那么单元格的数据对客户端时可见的

4、如果没有影子单元格，去提交表中查找单元格的提交时间，提交表中不存在，对单元格的写事务被终止了。

# Quikstart

```java
public class Example {
    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        //事务管理器
        TransactionManager tm = HBaseTransactionManager.newBuilder()
                                                       .withConfiguration(conf)
                                                       .build();
				//创建TTable，内部根据Configuration、tableName创建HTable
        TTable tt = new TTable(conf, "EXAMPLE_TABLE");
        //rowkey1
        byte[] exampleRow1 = Bytes.toBytes("EXAMPLE_ROW1");
       //rowkey2
        byte[] exampleRow2 = Bytes.toBytes("EXAMPLE_ROW2");
        //列族
        byte[] family = Bytes.toBytes("EXAMPLE_CF");
      	//列名
        byte[] qualifier = Bytes.toBytes("foo");
        byte[] dataValue1 = Bytes.toBytes("val1");
        byte[] dataValue2 = Bytes.toBytes("val2");
				//开启事务
        Transaction tx = tm.begin();
        Put row1 = new Put(exampleRow1);
        row1.add(family, qualifier, dataValue1);
        tt.put(tx, row1);
      
        Put row2 = new Put(exampleRow2);
        row2.add(family, qualifier, dataValue2);
        tt.put(tx, row2);
      	//提交事务
        tm.commit(tx);

        tt.close();
        tm.close();
    }
}
```

# 流程

## 开启事务

com.yahoo.omid.transaction.AbstractTransactionManager#begin

```java
public final Transaction begin() throws TransactionException {

    try {
        try {
            //事务开始前执行的自定义操作
            preBegin();
        } catch (TransactionManagerException e) {
            LOG.warn(e.getMessage());
        }
        //从tso-server获取事务开始时间戳
        long startTimestamp = tsoClient.getNewStartTimestamp().get();
      	//创建HBaseTransaction
        AbstractTransaction<? extends CellId> tx =
                transactionFactory.createTransaction(startTimestamp, this);
        try {
            //事务开始后执行的自定义操作
            postBegin(tx);
        } catch (TransactionManagerException e) {
            LOG.warn(e.getMessage());
        }
        return tx;
    } catch (ExecutionException e) {
        throw new TransactionException("Could not get new timestamp", e);
    } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw new TransactionException("Interrupted getting timestamp", ie);
    }
}
```

```java
public void put(Transaction tx, Put put) throws IOException {
    throwExceptionIfOpSetsTimerange(put);
    HBaseTransaction transaction = enforceHBaseTransactionAsParam(tx);
    final long startTimestamp = transaction.getStartTimestamp(); //开始时间戳
    final Put tsput = new Put(put.getRow(), startTimestamp);
    Map<byte[], List<Cell>> kvs = put.getFamilyCellMap();
    for (List<Cell> kvl : kvs.values()) {
        for (Cell c : kvl) {
            CellUtils.validateCell(c, startTimestamp);
            // Reach into keyvalue to update timestamp.
            // It's not nice to reach into keyvalue internals,
            // but we want to avoid having to copy the whole thing
            KeyValue kv = KeyValueUtil.ensureKeyValue(c);
            Bytes.putLong(kv.getValueArray(), kv.getTimestampOffset(), startTimestamp);
            tsput.add(kv);
            transaction.addWriteSetElement(
                                           new HBaseCellId(table,
                                                           CellUtil.cloneRow(kv),
                                                           CellUtil.cloneFamily(kv),
                                                           CellUtil.cloneQualifier(kv),
                                                           kv.getTimestamp()));
        }
    }
    table.put(tsput);
}
```

## 提交事务

com.yahoo.omid.transaction.AbstractTransactionManager#commit

```java
public final void commit(Transaction transaction)
        throws RollbackException, TransactionException {
    AbstractTransaction<? extends CellId> tx = enforceAbstractTransactionAsParam(transaction);
    enforceTransactionIsInRunningState(tx);
    if (tx.isRollbackOnly()) { // If the tx was marked to rollback, do it
        rollback(tx);
        throw new RollbackException("Transaction was set to rollback");
    }
    try {
        try {
            preCommit(tx);
        } catch (TransactionManagerException e) {
            tx.cleanup();
            throw new TransactionException(e.getMessage(), e);
        }
        //tso-server获取提交时间戳，并且验证事务是否有冲突
        long commitTs = tsoClient.commit(tx.getStartTimestamp(), tx.getWriteSet()).get();
        tx.setStatus(Status.COMMITTED);
        tx.setCommitTimestamp(commitTs);
        try {
            updateShadowCells(tx);//更新shadowCell
            // Remove transaction from commit table if not failure occurred
            commitTableClient.completeTransaction(tx.getStartTimestamp()).get();
            postCommit(tx);
        } catch (TransactionManagerException e) {
            LOG.warn(e.getMessage());
        }
    } catch (ExecutionException e) {
        if (e.getCause() instanceof AbortException) { // Conflicts detected, so rollback
            // Make sure its commit timestamp is 0, so the cleanup does the right job
            tx.setCommitTimestamp(0);
            tx.setStatus(Status.ROLLEDBACK);
            tx.cleanup();
            throw new RollbackException("Conflicts detected in tx writeset. Transaction aborted.", e.getCause());
        }
        throw new TransactionException("Could not commit", e.getCause());
    } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw new TransactionException("Interrupted committing transaction", ie);
    }
}
```

# TSO-Server

## 处理请求入口

内部逻辑主要是借助Disruptor，通过事件驱动机制来实现的

com.yahoo.omid.tso.TSOHandler#messageReceived

```java
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) {
    Object msg = e.getMessage();
    if (msg instanceof TSOProto.Request) {
        TSOProto.Request request = (TSOProto.Request)msg;
        if (request.hasHandshakeRequest()) {
            checkHandshake(ctx, request.getHandshakeRequest());
            return;
        }
        if (!handshakeCompleted(ctx)) {
            LOG.info("handshake not completed");
            ctx.getChannel().close();
        }

        if (request.hasTimestampRequest()) { //开启事务
            requestProcessor.timestampRequest(ctx.getChannel());
        } else if (request.hasCommitRequest()) { //提交事务
            TSOProto.CommitRequest cr = request.getCommitRequest();
            requestProcessor.commitRequest(cr.getStartTimestamp(),
                                           cr.getCellIdList(),
                                           cr.getIsRetry(),
                                           ctx.getChannel());
        } else {
            LOG.error("Invalid request {}", request);
            ctx.getChannel().close();
        }
    } else {
        LOG.error("Unknown message type", msg);
    }
}
```

## 发布事件

### 发布获取时间戳的事件

com.yahoo.omid.tso.RequestProcessorImpl#timestampRequest

```java
public void timestampRequest(Channel c) {
    long seq = requestRing.next();
    RequestEvent e = requestRing.get(seq);
    RequestEvent.makeTimestampRequest(e, c);
    requestRing.publish(seq); //发布获取时间戳事件
}
```

### 发布提交事务的事件

com.yahoo.omid.tso.RequestProcessorImpl#commitRequest

```java
public void commitRequest(long startTimestamp, Collection<Long> writeSet, boolean isRetry, Channel c) {
    long seq = requestRing.next();
    RequestEvent e = requestRing.get(seq);
    RequestEvent.makeCommitRequest(e, startTimestamp, writeSet, isRetry, c);
    requestRing.publish(seq); //发布提交事务的事件
} 
```

## 请求事件处理

```java
public void onEvent(final RequestEvent event, final long sequence, final boolean endOfBatch)
    throws Exception
{
    if (event.getType() == RequestEvent.Type.TIMESTAMP) {
        handleTimestamp(event.getChannel()); //处理获取时间戳事件
    } else if (event.getType() == RequestEvent.Type.COMMIT) {
        handleCommit(event.getStartTimestamp(), event.writeSet(), event.isRetry(), event.getChannel()); //处理提交事务的事件
    }
}
```

### 事务开始时获取时间戳

com.yahoo.omid.tso.RequestProcessorImpl#handleTimestamp

```java
public void handleTimestamp(Channel c) {
    long timestamp;

    try {
        timestamp = timestampOracle.next();
    } catch (IOException e) {
        LOG.error("Error getting timestamp", e);
        return;
    }

    persistProc.persistTimestamp(timestamp, c);
}
```

传递TIMESTAMP事件

com.yahoo.omid.tso.PersistenceProcessorImpl#persistTimestamp

```java
public void persistTimestamp(long startTimestamp, Channel c) {
    long seq = persistRing.next();
    PersistEvent e = persistRing.get(seq);
    PersistEvent.makePersistTimestamp(e, startTimestamp, c);
    persistRing.publish(seq);
}
```

### 事务提交时获取时间戳

com.yahoo.omid.tso.RequestProcessorImpl#handleCommit

```java
public long handleCommit(long startTimestamp, Iterable<Long> writeSet, boolean isRetry, Channel c) {
    boolean committed = false;
    long commitTimestamp = 0L;

    int numCellsInWriteset = 0;
    // 0. check if it should abort
    if (startTimestamp <= lowWatermark) {
        committed = false;
    } else {
        // 1. 检测冲突
        committed = true;
        for (long cellId : writeSet) {
            long value = hashmap.getLatestWriteForCell(cellId);
            if (value != 0 && value >= startTimestamp) { //提交的时间戳大于等于开始时间戳，表明其他事务对此cell有修改
                committed = false;
                break;
            }
            numCellsInWriteset++;
        }
    }

    if (committed) {
        // 2. commit
        try {
            commitTimestamp = timestampOracle.next(); //分配提交时间戳

            if (numCellsInWriteset > 0) {
                long newLowWatermark = lowWatermark;

                for (long r : writeSet) {
                    long removed = hashmap.putLatestWriteForCell(r, commitTimestamp);
                    newLowWatermark = Math.max(removed, newLowWatermark);
                }

                lowWatermark = newLowWatermark;
                LOG.trace("Setting new low Watermark to {}", newLowWatermark);
                persistProc.persistLowWatermark(newLowWatermark);
            }
            //发布事件:hbase存储开始时间戳与提交时间戳的映射关系
            persistProc.persistCommit(startTimestamp, commitTimestamp, c);
        } catch (IOException e) {
            LOG.error("Error committing", e);
        }
    } else { // add it to the aborted list 有冲突
        persistProc.persistAbort(startTimestamp, isRetry, c);
    }

    return commitTimestamp;
}
```

## 响应事件处理

com.yahoo.omid.tso.PersistenceProcessorImpl#onEvent

```java
public void onEvent(final PersistEvent event, final long sequence, final boolean endOfBatch)
    throws Exception {

    switch (event.getType()) {
    case COMMIT: //事务提交
        // TODO: What happens when the IOException is thrown?
        writer.addCommittedTransaction(event.getStartTimestamp(), event.getCommitTimestamp());
        batch.addCommit(event.getStartTimestamp(), event.getCommitTimestamp(), event.getChannel());
        break;
    case ABORT: //事务丢弃
        sendAbortOrIdentifyFalsePositive(event.getStartTimestamp(), event.isRetry(), event.getChannel());
        break;
    case TIMESTAMP: //请求时间戳
        batch.addTimestamp(event.getStartTimestamp(), event.getChannel());
        break;
    case LOW_WATERMARK:
        writer.updateLowWatermark(event.getLowWatermark());
        break;
    }

    if (batch.isFull() || endOfBatch) {
        maybeFlushBatch(); //将缓存的Put发送至服务端、返回响应信息给客户端
    }
}
```

### 缓存Put请求

com.yahoo.omid.committable.hbase.HBaseCommitTable.HBaseWriter#addCommittedTransaction

```java
public void addCommittedTransaction(long startTimestamp, long commitTimestamp) throws IOException { //保存事务开始时间戳、提交时间戳的映射关系
    assert(startTimestamp < commitTimestamp);
    Put put = new Put(startTimestampToKey(startTimestamp), startTimestamp);
    put.add(COMMIT_TABLE_FAMILY, COMMIT_TABLE_QUALIFIER,
            encodeCommitTimestamp(startTimestamp, commitTimestamp));
    table.put(put);
}
```

### 将响应信息存放至events数组

com.yahoo.omid.tso.PersistenceProcessorImpl.Batch#addCommit

```java
void addCommit(long startTimestamp, long commitTimestamp, Channel c) {
    if (isFull()) {
        throw new IllegalStateException("batch full");
    }
    int index = numEvents++;
    PersistEvent e = events[index];
    PersistEvent.makePersistCommit(e, startTimestamp, commitTimestamp, c);
}
```

### 将缓存的Put发送至服务端、返回响应信息给客户端

com.yahoo.omid.tso.PersistenceProcessorImpl#maybeFlushBatch

```java
private void maybeFlushBatch() {
    if (batch.isFull()) {
        flush();
    } else if ((System.nanoTime() - lastFlush) > TimeUnit.MILLISECONDS.toNanos(BATCH_TIMEOUT_MS)) {
        timeoutMeter.mark();
        flush();
    }
}
```

```java
private void flush() {
    lastFlush = System.nanoTime();
    batchSizeHistogram.update(batch.getNumEvents());
    try {
        writer.flush().get();
        flushTimer.update((System.nanoTime() - lastFlush));
        batch.sendRepliesAndReset(reply, retryProc);
    } catch (ExecutionException ee) {
        panicker.panic("Error persisting commit batch", ee.getCause());
    } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        LOG.error("Interrupted after persistence");
    }
}
```

### 将缓存的Put请求发送至Hbase服务端

com.yahoo.omid.committable.hbase.HBaseCommitTable.HBaseWriter#flush

```java
public ListenableFuture<Void> flush() {
    SettableFuture<Void> f = SettableFuture.<Void>create();
    try {
        table.flushCommits();
        f.set(null);
    } catch (IOException e) {
        LOG.error("Error flushing data", e);
        f.setException(e);
    }
    return f;
}
```

### 发送响应信息给客户端

com.yahoo.omid.tso.PersistenceProcessorImpl.Batch#sendRepliesAndReset

```java
void sendRepliesAndReset(ReplyProcessor reply, RetryProcessor retryProc) {
    for (int i = 0; i < numEvents; i++) {
        PersistEvent e = events[i];
        switch (e.getType()) {
        case TIMESTAMP:
            reply.timestampResponse(e.getStartTimestamp(), e.getChannel());
            break;
        case COMMIT:
            reply.commitResponse(e.getStartTimestamp(), e.getCommitTimestamp(), e.getChannel());
            break;
        case ABORT:
            if(e.isRetry()) {
                retryProc.disambiguateRetryRequestHeuristically(e.getStartTimestamp(), e.getChannel());
            } else {
                LOG.error("We should not be receiving non-retried aborted requests in here");
                assert(false);
            }
            break;
        default:
            assert(false);
            break;
        }
    }
    numEvents = 0;
}
```
