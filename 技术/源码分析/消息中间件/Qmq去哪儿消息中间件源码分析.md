# Qmq源码分析

* [概述](#概述)
* [存储模型](#存储模型)
* [服务端入口](#服务端入口)
* [消息追加](#消息追加)
  * [合法性检测](#合法性检测)
  * [单条数据追加数据](#单条数据追加数据)
  * [获取当前LogSegment](#获取当前logsegment)
  * [追加消息到文件](#追加消息到文件)
* [消息刷盘](#消息刷盘)
* [索引构建](#索引构建)
  * [初始化](#初始化)
  * [启动](#启动)
    * [校验回放位置的合法性](#校验回放位置的合法性)
    * [迭代消息，发布MessageLogRecord事件](#迭代消息发布messagelogrecord事件)
    * [构建索引](#构建索引)
    * [索引刷盘](#索引刷盘)
      * [事件刷盘](#事件刷盘)
      * [周期刷盘](#周期刷盘)
* [消息拉取](#消息拉取)
  * [doFindUnAckMessages](#dofindunackmessages)
  * [findNewExistMessages](#findnewexistmessages)
    * [获取ConsumeQueue](#获取consumequeue)
    * [读取消息](#读取消息)
    * [追加ActionLog](#追加actionlog)
* [消费者注册](#消费者注册)
  * [注册入口](#注册入口)
  * [添加subscriber](#添加subscriber)
  * [更新心跳时间](#更新心跳时间)
  * [检测消费者](#检测消费者)
    * [启动入口](#启动入口)
    * [检测消费者状态](#检测消费者状态)
      * [暂时下线处理](#暂时下线处理)
      * [永久下线处理](#永久下线处理)
    * [对FOREVER_OFFLINE类型的Action进行处理](#对forever_offline类型的action进行处理)
* [消费组隔离](#消费组隔离)
* [分布式事务](#分布式事务)
  * [流程](#流程)
  * [实现](#实现)
    * [1、SpringTransactionProvider](#1springtransactionprovider)
    * [2、MessageTracker](#2messagetracker)
    * [3、发送消息](#3发送消息)
    * [4、事务拦截处理](#4事务拦截处理)
* [延时/定时消息](#延时定时消息)
  * [重放MessageLog](#重放messagelog)
  * [加入ScheduleLog](#加入schedulelog)
  * [加入时间轮](#加入时间轮)
* [References](#references)
* [总结](#总结)


# 概述

QMQ是去哪儿网开源的消息中间件，支持各种时长的延迟消息、弱化了当消费者和分区发生变化时触发的重平衡，支持事务消息

# 存储模型

所有topic的消息都存储到同一个文件（MessageLog），与RocketMQ类似

topic没有分区的概念，所以每个topic只有一个索引文件（ConsumeLog），不像RocketMQ每个topic的分区都有有对应的索引文件，

每个消费者实例多都有对应的Pull log文件，用于记录从ConsumeLog拉取的逻辑sequence，根据逻辑sequence计算出ConsumeLog中具体位置的信息

**无重平衡的概念，灵活、容易的扩展消费者实例，解决积压的问题**

# 服务端入口

qunar.tc.qmq.container.Bootstrap#main

```java
public static void main(String[] args) throws Exception {
    //加载配置，封装成DynamicConfig
    DynamicConfig config = DynamicConfigLoader.load("broker.properties");
    //创建ServerWrapper
    ServerWrapper wrapper = new ServerWrapper(config);
    //创建关闭时的钩子函数
    Runtime.getRuntime().addShutdownHook(new Thread(wrapper::destroy));
    //启动ServerWrapper
    wrapper.start(true);

    if (wrapper.isSlave()) {//salve角色
        final ServletContextHandler context = new ServletContextHandler(ServletContextHandler.SESSIONS);
        context.setContextPath("/");
        context.setResourceBase(System.getProperty("java.io.tmpdir"));
        final QueryMessageServlet servlet = new QueryMessageServlet(config, wrapper.getStorage());

        ServletHolder servletHolder = new ServletHolder(servlet);
        servletHolder.setAsyncSupported(true);
        context.addServlet(servletHolder, "/api/broker/message");

        final int port = config.getInt("slave.server.http.port", 8080);
        final Server server = new Server(port);
        server.setHandler(context);
        server.start();
        server.join();
    }
}
```

# 消息追加

处理追加请求

qunar.tc.qmq.processor.SendMessageProcessor#processRequest

```java
public CompletableFuture<Datagram> processRequest(ChannelHandlerContext ctx, RemotingCommand command) {
    List<RawMessage> messages;
    try {
        //反序列化原始消息，封装成RawMessage	
        messages = deserializeRawMessages(command);
    } catch (Exception e) {
        LOG.error("received invalid message. channel: {}", ctx.channel(), e);
        QMon.brokerReceivedInvalidMessageCountInc();
        final Datagram response = RemotingBuilder.buildEmptyResponseDatagram(CommandCode.BROKER_ERROR, command.getHeader());
        return CompletableFuture.completedFuture(response);
    }
  //统计一分钟内追加的数据量
    BrokerStats.getInstance().getLastMinuteSendRequestCount().add(messages.size());
    final ListenableFuture<Datagram> result = sendMessageWorker.receive(messages, command);
    final CompletableFuture<Datagram> future = new CompletableFuture<>();
    Futures.addCallback(result, new FutureCallback<Datagram>() {
                @Override
                public void onSuccess(Datagram datagram) {
                    future.complete(datagram);
                }
                @Override
                public void onFailure(Throwable ex) {
                    future.completeExceptionally(ex);
                }
            }
    );
    return future;
}
```

qunar.tc.qmq.processor.SendMessageWorker#receive

```java
ListenableFuture<Datagram> receive(final List<RawMessage> messages, final RemotingCommand cmd) {
    final List<SettableFuture<ReceiveResult>> futures = new ArrayList<>(messages.size());
    for (final RawMessage message : messages) {
        final MessageHeader header = message.getHeader();
        monitorMessageReceived(header.getCreateTime(), header.getSubject());
        final ReceivingMessage receivingMessage = new ReceivingMessage(message, cmd.getReceiveTime());
        futures.add(receivingMessage.promise());
        invoker.invoke(receivingMessage);
    }
    return Futures.transform(Futures.allAsList(futures),
            (Function<? super List<ReceiveResult>, ? extends Datagram>) input -> RemotingBuilder.buildResponseDatagram(CommandCode.SUCCESS, cmd.getHeader(), new SendResultPayloadHolder(input)));
}
```

## 合法性检测

qunar.tc.qmq.processor.SendMessageWorker#doInvoke

```java
private void doInvoke(ReceivingMessage message) {
    if (BrokerConfig.isReadonly()) { //默认只读
        brokerReadOnly(message);
        return;
    }
    //避免从节点缺失太多数据
    if (bigSlaveLag()) { //等待slave同步完成 && 同步请求数量大于50000
        brokerReadOnly(message); //返回BROKER_READ_ONLY，后台统计出现此情况的次数，关注slave的同步情况
        return;
    }
    final String subject = message.getSubject();
    if (SubjectUtils.isInValid(subject)) { //subject无效
        QMon.receivedIllegalSubjectMessagesCountInc(subject);
        if (isRejectIllegalSubject()) { //拒绝无效的subject
            notAllowed(message);
            return;
        }
    }
    try {
        ReceiveResult result = messageStore.putMessage(message); //追加消息
        offer(message, result); //
    } catch (Throwable t) {
        error(message, t);
    }
}
```

## 单条数据追加数据

qunar.tc.qmq.store.MessageStoreWrapper#putMessage

```java
public ReceiveResult putMessage(final ReceivingMessage message) {
    final RawMessage rawMessage = message.getMessage();
    final MessageHeader header = rawMessage.getHeader();
    final String msgId = header.getMessageId();
    final long start = System.currentTimeMillis();
    try {
        final PutMessageResult putMessageResult = storage.appendMessage(rawMessage);
        final PutMessageStatus status = putMessageResult.getStatus();
        if (status != PutMessageStatus.SUCCESS) {
            LOG.error("put message error, message:{} {}, status:{}", header.getSubject(), msgId, status.name());
            QMon.storeMessageErrorCountInc(header.getSubject());
            return new ReceiveResult(msgId, MessageProducerCode.STORE_ERROR, status.name(), -1);
        }
        AppendMessageResult<MessageSequence> result = putMessageResult.getResult();
        final long endOffsetOfMessage = result.getWroteOffset() + result.getWroteBytes();
        return new ReceiveResult(msgId, MessageProducerCode.SUCCESS, "", endOffsetOfMessage);
    } catch (Throwable e) {
        LOG.error("put message error, message:{} {}", header.getSubject(), header.getMessageId(), e);
        QMon.storeMessageErrorCountInc(header.getSubject());
        return new ReceiveResult(msgId, MessageProducerCode.STORE_ERROR, "", -1);
    } finally {
        //记录追加耗时
        QMon.putMessageTime(header.getSubject(), System.currentTimeMillis() - start);
    }
}
```

## 获取当前LogSegment

qunar.tc.qmq.store.MessageLog#putMessage

```java
public PutMessageResult putMessage(final RawMessage message) {
    final AppendMessageResult<MessageSequence> result;
    //获取当前追加的LogSegment
    LogSegment segment = logManager.latestSegment();
    if (segment == null) {
        segment = logManager.allocNextSegment();//新建LogSegment
    }
    if (segment == null) {
        return new PutMessageResult(PutMessageStatus.CREATE_MAPPED_FILE_FAILED, null);
    }
    //追加消息
    result = segment.append(message, messageAppender);
    switch (result.getStatus()) {
        case SUCCESS: //成功
            break;
        case END_OF_FILE: //文件空间不足
            LogSegment logSegment = logManager.allocNextSegment();
            if (logSegment == null) {
                return new PutMessageResult(PutMessageStatus.CREATE_MAPPED_FILE_FAILED, null);
            }
            return putMessage(message);
        case MESSAGE_SIZE_EXCEEDED:
            return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
        default:
            return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
    }
    return new PutMessageResult(PutMessageStatus.SUCCESS, result);
}
```

## 追加消息到文件

qunar.tc.qmq.store.LogSegment#append

```java
public <T, R> AppendMessageResult<R> append(final T message, final MessageAppender<T, R> appender) {
    final int currentPos = wrotePosition.get();
    if (currentPos > fileSize) {
        return new AppendMessageResult<>(AppendMessageStatus.UNKNOWN_ERROR);
    }
    if (currentPos == fileSize) {
        return new AppendMessageResult<>(AppendMessageStatus.END_OF_FILE);
    }
    //新缓冲区的pos为0、capacity和limit将是该缓冲区中剩余的字节数
    final ByteBuffer buffer = mappedByteBuffer.slice();
    //从该位置开始追加
    buffer.position(currentPos);
    final AppendMessageResult<R> result = appender.doAppend(getBaseOffset(), buffer, fileSize - currentPos, message);
    //可见性，追加的消息对消费者可见
    this.wrotePosition.addAndGet(result.getWroteBytes());
    return result;
}
```

qunar.tc.qmq.store.MessageLog.RawMessageAppender#doAppend

```java
public AppendMessageResult<MessageSequence> doAppend(long baseOffset, ByteBuffer targetBuffer, int freeSpace, RawMessage message) {
    workingBuffer.clear(); //清空数据；重复使用
    final String subject = message.getHeader().getSubject();
    final byte[] subjectBytes = subject.getBytes(StandardCharsets.UTF_8);
    //消息写入的位置
    final long wroteOffset = baseOffset + targetBuffer.position();
    final int recordSize = recordSize(subjectBytes.length, message.getBodySize());
    //文件空间不足，填充字节0,返回END_OF_FILE
    if (recordSize != freeSpace && recordSize + MIN_RECORD_BYTES > freeSpace) {
        workingBuffer.limit(MIN_RECORD_BYTES);
        workingBuffer.putInt(MagicCode.MESSAGE_LOG_MAGIC_V3);
        workingBuffer.put(ATTR_EMPTY_RECORD);
        workingBuffer.putLong(System.currentTimeMillis());
        targetBuffer.put(workingBuffer.array(), 0, MIN_RECORD_BYTES);
        int fillZeroLen = freeSpace - MIN_RECORD_BYTES;
        if (fillZeroLen > 0) {
            targetBuffer.put(fillZero(fillZeroLen));
        }
        return new AppendMessageResult<>(AppendMessageStatus.END_OF_FILE, wroteOffset, freeSpace, null);
    } else {
        //获取sequence
        final long sequence = consumerLogManager.getOffsetOrDefault(subject, 0);
        int headerSize = recordSize - message.getBodySize();
        workingBuffer.limit(headerSize);
        workingBuffer.putInt(MagicCode.MESSAGE_LOG_MAGIC_V3);
        workingBuffer.put(ATTR_MESSAGE_RECORD);
        workingBuffer.putLong(System.currentTimeMillis());
        workingBuffer.putLong(sequence);
        workingBuffer.putShort((short) subjectBytes.length);
        workingBuffer.put(subjectBytes);
        workingBuffer.putLong(message.getHeader().getBodyCrc());
        workingBuffer.putInt(message.getBodySize());
        targetBuffer.put(workingBuffer.array(), 0, headerSize);
        targetBuffer.put(message.getBody().nioBuffer());
        //设置下一个sequence
        consumerLogManager.incOffset(subject);
        //value写入位置
        final long payloadOffset = wroteOffset + headerSize;
        return new AppendMessageResult<>(AppendMessageStatus.SUCCESS, wroteOffset, recordSize, new MessageSequence(sequence, payloadOffset));
    }
}
```

# 消息刷盘

qunar.tc.qmq.store.DefaultStorage#DefaultStorage

```java
this.messageLogFlushService = new PeriodicFlushService(new MessageLogFlushProvider());
```

qunar.tc.qmq.store.DefaultStorage#start

```java
messageLogFlushService.start();
```

qunar.tc.qmq.store.DefaultStorage.MessageLogFlushProvider#flush

```java
public void flush() {
    messageLog.flush(); //消息刷盘
}
```



```java
public void flush() {
    final long start = System.currentTimeMillis();
    try {
        logManager.flush();
    } finally {
        QMon.flushMessageLogTimer(System.currentTimeMillis() - start); //记录刷盘时间
    }
}
```

qunar.tc.qmq.store.LogManager#flush

```java
public boolean flush() {
    //根据flushedOffset获取需要刷新的Log
    ConcurrentNavigableMap<Long, LogSegment> beingFlushView = findBeingFlushView();
    int lastOffset = -1;
    long lastBaseOffset = -1;
    for (Map.Entry<Long, LogSegment> entry : beingFlushView.entrySet()) {
        try {
            LogSegment segment = entry.getValue();
            //返回文件写入的位置
            lastOffset = segment.flush();
            //文件的开始offset
            lastBaseOffset = segment.getBaseOffset();
        } catch (Exception e) {
            break;
        }
    }
    if (lastBaseOffset == -1 || lastOffset == -1) return false;
    //计算flushoffset
    final long where = lastBaseOffset + lastOffset;
    boolean result = where != this.flushedOffset;
    this.flushedOffset = where;
    return result;
}
```

# 索引构建

## 初始化

qunar.tc.qmq.store.DefaultStorage#DefaultStorage

```java
this.messageEventBus = new FixedExecOrderEventBus();
if (config.isSMTEnable()) {
    this.messageEventBus.subscribe(MessageLogRecord.class, new BuildMessageMemTableEventListener(config, memTableManager, sortedMessagesTable));
    this.messageEventBus.subscribe(MessageLogRecord.class, event -> messageEventBus.post(new ConsumerLogWroteEvent(event.getSubject(), true)));
} else {
    this.messageEventBus.subscribe(MessageLogRecord.class, new BuildConsumerLogEventListener(consumerLogManager)); //构建索引事件
    this.messageEventBus.subscribe(MessageLogRecord.class, consumerLogFlusher); //刷盘事件
}
//回放MessageLog；需要记录上次回放的位置，从上次回放的位置开始构建索引
this.messageLogIterateService = new LogIterateService<>("ReplayMessageLog", config.getLogDispatcherPauseMillis(), messageLog, checkpointManager.getMessageCheckpointOffset(), messageEventBus);
```

## 启动

qunar.tc.qmq.store.DefaultStorage#start

```java
messageLogIterateService.start();
```

### 校验回放位置的合法性

qunar.tc.qmq.store.LogIterateService#initialMessageIterateFrom

```java
private long initialMessageIterateFrom(final Visitable<T> log, final long checkpoint) { // 获取回放的位置
    if (checkpoint <= 0) {
        return log.getMinOffset();
    }
    if (checkpoint > log.getMaxOffset()) {
        return log.getMaxOffset();
    }
    return checkpoint;
}
```

### 迭代消息，发布MessageLogRecord事件

qunar.tc.qmq.store.LogIterateService.Dispatcher#processLog

```java
private void processLog() {
    final long startOffset = iterateFrom.longValue();
    try (AbstractLogVisitor<T> visitor = visitable.newVisitor(startOffset)) {
        if (startOffset != visitor.getStartOffset()) {
            LOG.info("reset iterate from offset from {} to {}", startOffset, visitor.getStartOffset());
            iterateFrom.reset();
            iterateFrom.add(visitor.getStartOffset());
        }
        while (true) {
            final LogVisitorRecord<T> record = visitor.nextRecord();
            if (record.isNoMore()) {
                break;
            }
            if (record.hasData()) {
                dispatcher.post(record.getData());
            }
        }
        iterateFrom.add(visitor.visitedBufferSize());
    }

    try {
        TimeUnit.MILLISECONDS.sleep(dispatcherPauseMills);
    } catch (InterruptedException e) {
        LOG.warn("log dispatcher sleep interrupted");
    }
}
```

### 构建索引

qunar.tc.qmq.store.DefaultStorage.BuildConsumerLogEventListener#onEvent

```java
public void onEvent(final MessageLogRecord event) {
    if (isFirstEventOfLogSegment(event)) {
        LOG.info("first event of log segment. event: {}", event);
        // TODO(keli.wang): need catch all exception here?
        consumerLogManager.createOffsetFileFor(event.getBaseOffset(), offsets);
    }
    updateOffset(event);
    //获取或创建ConsumerLog，每个主题对应一个ConsumerLog
    final ConsumerLog consumerLog = consumerLogManager.getOrCreateConsumerLog(event.getSubject());
    if (consumerLog.nextSequence() != event.getSequence()) {
        LOG.error("next sequence not equals to max sequence. subject: {}, received seq: {}, received offset: {}, diff: {}",
                event.getSubject(), event.getSequence(), event.getWroteOffset(), event.getSequence() - consumerLog.nextSequence());
    }
    //追加索引，流程与追加消息类似
    final boolean success = consumerLog.writeMessageLogIndex(event.getSequence(), event.getWroteOffset(), event.getWroteBytes(), event.getHeaderSize());
    checkpointManager.updateMessageReplayState(event);
    //发送索引构建完成的事件
    messageEventBus.post(new ConsumerLogWroteEvent(event.getSubject(), success));
}
```



### 索引刷盘

#### 事件刷盘

默认每追加10000条索引调用刷盘操作

qunar.tc.qmq.store.ConsumerLogFlusher#onEvent

```java
public void onEvent(final MessageLogRecord event) {
    final long count = counter.incrementAndGet();
    if (count < config.getMessageCheckpointInterval()) { //默认10_000
        return;
    }

    QMon.consumerLogFlusherExceedCheckpointIntervalCountInc();
    submitFlushTask();
}
```

#### 周期刷盘

默认每隔1分钟调用刷盘操作

qunar.tc.qmq.store.ConsumerLogFlusher#ConsumerLogFlusher

```java
scheduleForceFlushTask();
```

```java
private void scheduleForceFlushTask() {
    flushExecutor.scheduleWithFixedDelay(this::tryForceSubmitFlushTask, 1, 1, TimeUnit.MINUTES);
}
```

提交刷盘任务

```java
private synchronized void submitFlushTask() {
    counter.set(0);
    latestFlushTime = System.currentTimeMillis();
    final Snapshot<MessageCheckpoint> snapshot = checkpointManager.createMessageCheckpointSnapshot();
    flushExecutor.submit(() -> {
        final long start = System.currentTimeMillis();
        try {
            consumerLogManager.flush();
            checkpointManager.saveMessageCheckpointSnapshot(snapshot);
        } catch (Exception e) {
            QMon.consumerLogFlusherFlushFailedCountInc();
            LOG.error("flush consumer log failed. offset: {}", snapshot.getVersion(), e);
        } finally {
            QMon.consumerLogFlusherElapsedPerExecute(System.currentTimeMillis() - start);
        }
    });
}
```

```

```

# 消息拉取

qunar.tc.qmq.store.MessageStoreWrapper#findMessages

```java
public PullMessageResult findMessages(final PullRequest pullRequest) {
    try {
        //查找未ack的数据
        final PullMessageResult unAckMessages = findUnAckMessages(pullRequest);
        if (unAckMessages.getMessageNum() > 0) {
            return unAckMessages;
        }
        if (PullMessageResult.WAIT_PULLLOG_REPLAY == unAckMessages) {
            return PullMessageResult.EMPTY;
        }
        return findNewExistMessages(pullRequest);
    } catch (Throwable e) {
        LOG.error("find messages error, consumer: {}", pullRequest, e);
        QMon.findMessagesErrorCountInc(pullRequest.getSubject(), pullRequest.getGroup());
    }
    return PullMessageResult.EMPTY;
}
```

## doFindUnAckMessages

```java
private PullMessageResult findUnAckMessages(final PullRequest pullRequest) {
    final long start = System.currentTimeMillis();
    try {
        if (pullRequest.isBroadcast()) return PullMessageResult.EMPTY;
        return doFindUnAckMessages(pullRequest);
    } finally {
        QMon.findLostMessagesTime(pullRequest.getSubject(), pullRequest.getGroup(), System.currentTimeMillis() - start);
    }
}
```



```java
private PullMessageResult doFindUnAckMessages(final PullRequest pullRequest) {
    final String subject = pullRequest.getSubject();
    final String group = pullRequest.getGroup();
    final String consumerId = pullRequest.getConsumerId();
    //<topic,group,consumerId> -> ConsumerSequence,<topic,group,consumerId> -> Pull log
    //消费者拉取以及ack的sequence,这里的sequence指的是pull log中的sequence
    final ConsumerSequence consumerSequence = consumerSequenceManager.getOrCreateConsumerSequence(subject, group, consumerId);
    final long ackSequence = consumerSequence.getAckSequence(); //已经ack的offset
    long pullLogSequenceInConsumer = pullRequest.getPullOffsetLast(); //此次请求消费者拉取的offset
    if (pullLogSequenceInConsumer < ackSequence) { //此次请求的offset小于已经ack的offset
        pullLogSequenceInConsumer = ackSequence;//修改请求的offset为ack的offset，避免重复拉取
    }
    final long pullLogSequenceInServer = consumerSequence.getPullSequence(); //此消费者上次拉取的offset
    if (pullLogSequenceInServer <= pullLogSequenceInConsumer) { //拉取的消息都已经ack
        return PullMessageResult.EMPTY;
    }
    //上次拉取消息时追加的action log，还没有回放出pull log
    long pullLogReplaySequence = storage.getPullLogReplayState(subject, group, consumerId);
    if (pullLogReplaySequence < pullLogSequenceInServer) {
        return PullMessageResult.WAIT_PULLLOG_REPLAY;
    }
    LOG.warn("consumer need find lost ack messages, pullRequest: {}, consumerSequence: {}", pullRequest, consumerSequence);
    //请求拉取的消息数
    final int requestNum = pullRequest.getRequestNum();
    final List<Buffer> buffers = new ArrayList<>(requestNum);
    long firstValidSeq = -1;
    int totalSize = 0;
    //第一个缺失ack的pull log sequence
    final long firstLostAckPullLogSeq = pullLogSequenceInConsumer + 1;
    for (long seq = firstLostAckPullLogSeq; buffers.size() < requestNum && seq <= pullLogSequenceInServer; seq++) {
        try {
            //从pull log获取consume log sequence
            final long consumerLogSequence = getConsumerLogSequence(pullRequest, seq);
            //deleted message
            if (consumerLogSequence < 0) {
                LOG.warn("find no consumer log for this pull log sequence, req: {}, pullLogSeq: {}, consumerLogSeq: {}", pullRequest, seq, consumerLogSequence);
                if (firstValidSeq == -1) {
                    continue;
                } else {
                    break;
                }
            }
            //根据subject、consumerLogSequence从consume log获取索引信息，再从MessageLog获取一条数据
            final GetMessageResult getMessageResult = storage.getMessage(subject, consumerLogSequence);
            if (getMessageResult.getStatus() != GetMessageStatus.SUCCESS || getMessageResult.getMessageNum() == 0) {
                LOG.error("getMessageResult error, consumer:{}, consumerLogSequence:{}, pullLogSequence:{}, getMessageResult:{}", pullRequest, consumerLogSequence, seq, getMessageResult);
                QMon.getMessageErrorCountInc(subject, group);
                if (firstValidSeq == -1) {
                    continue;
                } else {
                    break;
                }
            }
            //有过滤器则进行过滤
            final Buffer segmentBuffer = getMessageResult.getBuffers().get(0);
            if (!noPullFilter(pullRequest) && !pullMessageFilterChain.needKeep(pullRequest, segmentBuffer)) {
                segmentBuffer.release();
                if (firstValidSeq != -1) {
                    break;
                } else {
                    continue;
                }
            }
            if (firstValidSeq == -1) {
                firstValidSeq = seq;
            }
            buffers.add(segmentBuffer);
            totalSize += segmentBuffer.getSize();
            //超过一次读取的内存限制
            if (totalSize >= MAX_MEMORY_LIMIT) break;
        } catch (Exception e) {
            LOG.error("error occurs when find messages by pull log offset, request: {}, consumerSequence: {}", pullRequest, consumerSequence, e);
            QMon.getMessageErrorCountInc(subject, group);
            if (firstValidSeq != -1) {
                break;
            }
        }
    }
    if (buffers.size() > 0) {
        //说明pull log里有一段对应的消息已经被清理掉了，需要调整一下位置
        if (firstValidSeq > firstLostAckPullLogSeq) {
            consumerSequence.setAckSequence(firstValidSeq - 1);
        }
    } else {
        LOG.error("find lost messages empty, consumer:{}, consumerSequence:{}, pullLogSequence:{}", pullRequest, consumerSequence, pullLogSequenceInServer);
        QMon.findLostMessageEmptyCountInc(subject, group);
        firstValidSeq = pullLogSequenceInServer;
        consumerSequence.setAckSequence(pullLogSequenceInServer);
        // auto ack all deleted pulled message
        LOG.info("auto ack deleted pulled message. subject: {}, group: {}, consumerId: {}, firstSeq: {}, lastSeq: {}",
                subject, group, consumerId, firstLostAckPullLogSeq, firstValidSeq);
        consumerSequenceManager.putAction(new RangeAckAction(subject, group, consumerId, System.currentTimeMillis(), firstLostAckPullLogSeq, firstValidSeq));
    }
    final PullMessageResult result = new PullMessageResult(firstValidSeq, buffers, totalSize, buffers.size());
    QMon.findLostMessageCountInc(subject, group, result.getMessageNum());
    LOG.info("found lost ack messages request: {}, consumerSequence: {}, result: {}", pullRequest, consumerSequence, result);
    return result;
}
```

## findNewExistMessages

1、获取ConsumeQueue，主要记录了<group,topic>下次拉取的位置

2、根据下次拉取的位置从索引文件读取索引信息，然后再从消息文件获取消息

3、根据消费者提供的过滤器进行消息的过滤

4、追加ActionLog

```java
private PullMessageResult findNewExistMessages(final PullRequest pullRequest) {
    final String subject = pullRequest.getSubject();
    final String group = pullRequest.getGroup();
    final String consumerId = pullRequest.getConsumerId();
    final boolean isBroadcast = pullRequest.isBroadcast();
    final long start = System.currentTimeMillis();
    try {
        //Map<topic,Map<subject,ConsumeQueue>>
        //主要记录下次拉取的位置
        ConsumeQueue consumeQueue = storage.locateConsumeQueue(subject, group);
        //读取消息
        final GetMessageResult getMessageResult = consumeQueue.pollMessages(pullRequest.getRequestNum());
        switch (getMessageResult.getStatus()) {
            case SUCCESS:
                if (getMessageResult.getMessageNum() == 0) {
                    consumeQueue.setNextSequence(getMessageResult.getNextBeginSequence());
                    return PullMessageResult.EMPTY;
                }

                if (noPullFilter(pullRequest)) { //没有Filter
                  	//追加ActionLog
                    final WritePutActionResult writeResult = consumerSequenceManager.putPullActions(subject, group, consumerId, isBroadcast, getMessageResult);
                    if (writeResult.isSuccess()) {
                        consumeQueue.setNextSequence(getMessageResult.getNextBeginSequence());
                        return new PullMessageResult(writeResult.getPullLogOffset(), getMessageResult.getBuffers(), getMessageResult.getBufferTotalSize(), getMessageResult.getMessageNum());
                    } else {
                        getMessageResult.release();
                        return PullMessageResult.EMPTY;
                    }
                }
								//有过滤器
                return doPullResultFilter(pullRequest, getMessageResult, consumeQueue);
            case OFFSET_OVERFLOW:
                LOG.warn("get message result not success, consumer:{}, result:{}", pullRequest, getMessageResult);
                QMon.getMessageOverflowCountInc(subject, group);
            default:
                consumeQueue.setNextSequence(getMessageResult.getNextBeginSequence());
                return PullMessageResult.EMPTY;
        }
    } finally {
        QMon.findNewExistMessageTime(subject, group, System.currentTimeMillis() - start);
    }
}
```

### 获取ConsumeQueue

qunar.tc.qmq.store.DefaultStorage#locateConsumeQueue

```java
public ConsumeQueue locateConsumeQueue(String subject, String group) {
    return consumeQueueManager.getOrCreate(subject, group);
}
```

qunar.tc.qmq.store.ConsumeQueueManager#getOrCreate

```java
public ConsumeQueue getOrCreate(final String subject, final String group) {
    ConcurrentMap<String, ConsumeQueue> consumeQueues = queues.computeIfAbsent(subject, s -> new ConcurrentHashMap<>());
    return consumeQueues.computeIfAbsent(group, g -> {
        Optional<Long> lastMaxSequence = getLastMaxSequence(subject, g);
        if (lastMaxSequence.isPresent()) {
            return new ConsumeQueue(storage, subject, g, lastMaxSequence.get() + 1);
        } else {
            OffsetBound consumerLogBound = storage.getSubjectConsumerLogBound(subject);
            long maxSequence = consumerLogBound == null ? 0 : consumerLogBound.getMaxOffset();
            LOGGER.info("发现新的group：{} 订阅了subject: {},从最新的消息开始消费, 最新的sequence为：{}！", g, subject, maxSequence);
            return new ConsumeQueue(storage, subject, g, maxSequence);
        }
    });
}
```

### 读取消息

qunar.tc.qmq.store.ConsumeQueue#pollMessages

```java
public synchronized GetMessageResult pollMessages(final int maxMessages) { //同步锁
    enableLagMonitor();
    //消费组下次消费的位置，nextSequence表示的是consume log的sequence
    long currentSequence = nextSequence.get();
    if (RetrySubjectUtils.isRetrySubject(subject)) {
        return storage.pollMessages(subject, currentSequence, maxMessages, this::isDelayReached);
    } else {
        //读取数据
        final GetMessageResult result = storage.pollMessages(subject, currentSequence, maxMessages);
        long actualSequence = result.getNextBeginSequence() - result.getBuffers().size();
        long delta = actualSequence - currentSequence;
        if (delta > 0) {
            QMon.expiredMessagesCountInc(subject, group, delta);
            LOG.error("next sequence skipped. subject: {}, group: {}, nextSequence: {}, result: {}", subject, group, currentSequence, result);
        }
        return result;
    }
}
```

qunar.tc.qmq.store.DefaultStorage#pollFromConsumerLog

```java
private GetMessageResult pollFromConsumerLog(String subject, long consumerLogSequence, int maxMessages, MessageFilter filter, boolean strictly) {//读取索引信息，从数据文件读取消息
    final GetMessageResult result = new GetMessageResult();
    if (maxMessages <= 0) { //拉取的最多条数为0
        result.setNextBeginSequence(consumerLogSequence);
        result.setStatus(GetMessageStatus.NO_MESSAGE);
        return result;
    }
    //获取主题对应的索引文件
    final ConsumerLog consumerLog = consumerLogManager.getConsumerLog(subject);
    if (consumerLog == null) {
        result.setNextBeginSequence(0);
        result.setStatus(GetMessageStatus.SUBJECT_NOT_FOUND);
        return result;
    }
    final OffsetBound bound = consumerLog.getOffsetBound();
    final long minSequence = bound.getMinOffset();
    final long maxSequence = bound.getMaxOffset();
    if (maxSequence == 0) {
        result.setNextBeginSequence(maxSequence);
        result.setStatus(GetMessageStatus.NO_MESSAGE);
        return result;
    }
    if (consumerLogSequence < 0) {
        result.setConsumerLogRange(new OffsetRange(maxSequence, maxSequence));
        result.setNextBeginSequence(maxSequence);
        result.setStatus(GetMessageStatus.SUCCESS);
        return result;
    }
    if (consumerLogSequence > maxSequence) {
        result.setNextBeginSequence(maxSequence);
        result.setStatus(GetMessageStatus.OFFSET_OVERFLOW);
        return result;
    }
    if (strictly && consumerLogSequence < minSequence) {
        result.setNextBeginSequence(consumerLogSequence);
        result.setStatus(GetMessageStatus.EMPTY_CONSUMER_LOG);
        return result;
    }
    //计算开始拉取的位置
    final long start = consumerLogSequence < minSequence ? minSequence : consumerLogSequence;
    //从索引文件读取索引信息，再从MessageLog获取消息
    final SegmentBuffer consumerLogBuffer = consumerLog.selectIndexBuffer(start);
    if (consumerLogBuffer == null) { //无索引信息
        result.setNextBeginSequence(start);
        result.setStatus(GetMessageStatus.EMPTY_CONSUMER_LOG);
        return result;
    }
    if (!consumerLogBuffer.retain()) {
        result.setNextBeginSequence(start);
        result.setStatus(GetMessageStatus.EMPTY_CONSUMER_LOG);
        return result;
    }
    long nextBeginSequence = start;
    try {
        //根据索引信息从消息文件获取消息
        final int maxMessagesInBytes = maxMessages * consumerLog.getUnitBytes();
        for (int i = 0; i < maxMessagesInBytes; i += consumerLog.getUnitBytes()) {
            if (i >= consumerLogBuffer.getSize()) {
                break;
            }
            //获取索引信息
            final ByteBuffer buffer = consumerLogBuffer.getBuffer();
            final ConsumerLog.Unit unit = consumerLog.readUnit(buffer);
            if (unit.getType() == ConsumerLog.PayloadType.MESSAGE_LOG_INDEX) {
                final ConsumerLog.MessageLogIndex index = (ConsumerLog.MessageLogIndex) unit.getIndex();
                if (!filter.filter(index::getTimestamp)) break;
                //根据索引信息获取数据
                if (!readFromMessageLog(subject, index, result)) {
                    if (result.getMessageNum() > 0) {
                        break;
                    }
                }
            } else if (unit.getType() == ConsumerLog.PayloadType.SMT_INDEX) {
                final ConsumerLog.SMTIndex index = (ConsumerLog.SMTIndex) unit.getIndex();
                if (!filter.filter(index::getTimestamp)) {
                    break;
                }
                if (!readFromSMT(index.getTabletId(), index.getPosition(), index.getSize(), result)) {
                    if (result.getMessageNum() > 0) {
                        break;
                    }
                }
            } else {
                throw new RuntimeException("unknown consumer log unit type " + unit.getType());
            }
            nextBeginSequence += 1;
        }
    } finally {
        consumerLogBuffer.release();
    }
    //消费者下次拉取的位置
    result.setNextBeginSequence(nextBeginSequence);
    //此次消费者拉取的范围
    result.setConsumerLogRange(new OffsetRange(start, nextBeginSequence - 1));
    result.setStatus(GetMessageStatus.SUCCESS);
    return result;
}
```

### 追加ActionLog

qunar.tc.qmq.consumer.ConsumerSequenceManager#putPullActions

```java
public WritePutActionResult putPullActions(final String subject, final String group, final String consumerId, final boolean isBroadcast, final GetMessageResult getMessageResult) {
    //本次拉取的consume log范围
    final OffsetRange consumerLogRange = getMessageResult.getConsumerLogRange();
    //消费已经拉取的、ack的pull log sequence
    final ConsumerSequence consumerSequence = getOrCreateConsumerSequence(subject, group, consumerId);
    if (consumerLogRange.getEnd() - consumerLogRange.getBegin() + 1 != getMessageResult.getMessageNum()) {
        LOG.debug("consumer offset range error, subject:{}, group:{}, consumerId:{}, isBroadcast:{}, getMessageResult:{}", subject, group, consumerId, isBroadcast, getMessageResult);
        QMon.consumerLogOffsetRangeError(subject, group);
    }
    consumerSequence.pullLock();
    try {
        //因为消息堆积等原因，可能会导致历史消息已经被删除了。所以可能得出这种情况：一次拉取100条消息，然后前20条已经删除了，
        //所以不能使用begin，要使用end减去消息条数这种方式
        //记录consume log的sequence
        final long firstConsumerLogSequence = consumerLogRange.getEnd() - getMessageResult.getMessageNum() + 1;
        final long lastConsumerLogSequence = consumerLogRange.getEnd();
        //记录pull log的sequence
        final long firstPullSequence = isBroadcast ? firstConsumerLogSequence : consumerSequence.getPullSequence() + 1;
        final long lastPullSequence = isBroadcast ? lastConsumerLogSequence : consumerSequence.getPullSequence() + getMessageResult.getMessageNum();
        final Action action = new PullAction(subject, group, consumerId,
                System.currentTimeMillis(), isBroadcast,
                firstPullSequence, lastPullSequence,
                firstConsumerLogSequence, lastConsumerLogSequence);
        //追加Action，
        if (!putAction(action)) {
            return new WritePutActionResult(false, -1);
        }
        //pull log sequence
        consumerSequence.setPullSequence(lastPullSequence);
        return new WritePutActionResult(true, firstPullSequence);
    } catch (Exception e) {
        LOG.error("write action log failed, subject: {}, group: {}, consumerId: {}", subject, group, consumerId, e);
        return new WritePutActionResult(false, -1);
    } finally {
        consumerSequence.pullUnlock();
    }
}
```

# 消费者注册

## 注册入口

```java
public CompletableFuture<Datagram> processRequest(final ChannelHandlerContext ctx, final RemotingCommand command) {
    final PullRequest pullRequest = pullRequestSerde.read(command.getHeader().getVersion(), command.getBody());
    BrokerStats.getInstance().getLastMinutePullRequestCount().add(1);
    QMon.pullRequestCountInc(pullRequest.getSubject(), pullRequest.getGroup());
    //检测、修正拉取请求的参数
    if (!checkAndRepairPullRequest(pullRequest)) {
        return CompletableFuture.completedFuture(crateErrorParamResult(command));
    }
    //Broker端维护group、subject、consumerId 信息
    subscribe(pullRequest);
    final PullEntry entry = new PullEntry(pullRequest, command.getHeader(), ctx);
    //分发拉取请求
    pullMessageWorker.pull(entry);
    return null;
}
```

qunar.tc.qmq.processor.PullMessageProcessor#subscribe

```java
private void subscribe(PullRequest pullRequest) {
    if (pullRequest.isBroadcast()) return; //广播消费
    final String subject = pullRequest.getSubject();
    final String group = pullRequest.getGroup();
    final String consumerId = pullRequest.getConsumerId();
    // Map<group-subject,Map<consumerId,Subscriber>> 
    subscriberStatusChecker.addSubscriber(subject, group, consumerId); //添加subscriber
    subscriberStatusChecker.heartbeat(consumerId, subject, group);//更新心跳时间
}
```

## 添加subscriber

```java
public void addSubscriber(String subject, String group, String consumerId) {
    final String groupAndSubject = GroupAndSubject.groupAndSubject(subject, group);
    addSubscriber(groupAndSubject, consumerId);
}
//添加Subscriber
private void addSubscriber(String groupAndSubject, String consumerId) {
    ConcurrentMap<String, Subscriber> m = subscribers.get(groupAndSubject);
    if (m == null) {
        m = new ConcurrentHashMap<>();
        final ConcurrentMap<String, Subscriber> old = subscribers.putIfAbsent(groupAndSubject, m);
        if (old != null) {
            m = old;
        }
    }
    if (!m.containsKey(consumerId)) {
        m.putIfAbsent(consumerId, createSubscriber(groupAndSubject, consumerId));
    }
}
```



```java
private Subscriber createSubscriber(String groupAndSubject, String consumerId) {
    final Subscriber subscriber = new Subscriber(this, groupAndSubject, consumerId);
    final RetryTask retryTask = new RetryTask(config, consumerSequenceManager, subscriber);
    final OfflineTask offlineTask = new OfflineTask(consumerSequenceManager, subscriber);
    subscriber.setRetryTask(retryTask); //暂时下线。3分钟无心跳
    subscriber.setOfflineTask(offlineTask);//永久下线。2天无心跳
    return subscriber;
}
```

## 更新心跳时间

qunar.tc.qmq.consumer.SubscriberStatusChecker#heartbeat

```java
public void heartbeat(String consumerId, String subject, String group) {
    final String realSubject = RetrySubjectUtils.getRealSubject(subject);
    final String retrySubject = RetrySubjectUtils.buildRetrySubject(realSubject, group);
    refreshSubscriber(realSubject, group, consumerId);
    refreshSubscriber(retrySubject, group, consumerId);
}
```



```java
private void refreshSubscriber(final String subject, final String group, final String consumerId) {
    final Subscriber subscriber = getSubscriber(subject, group, consumerId);
    if (subscriber != null) {
        subscriber.heartbeat();
    }
}
```



```java
void heartbeat() {
    renew();
    retryTask.cancel();
    offlineTask.cancel();
}
```



```java
void renew() {
    lastUpdate = System.currentTimeMillis();
}
```

## 检测消费者

### 启动入口

qunar.tc.qmq.startup.ServerWrapper#startConsumerChecker

```java
private void startConsumerChecker() {
    if (BrokerConfig.getBrokerRole() == BrokerRole.MASTER) {
        subscriberStatusChecker.start();
    }
}
```



```java
public void run() {
    try {
        check();
    } catch (Throwable e) {
        LOG.error("consumer status checker task failed.", e);
    }
}
```



```java
private void check() {
    for (final ConcurrentMap<String, Subscriber> m : subscribers.values()) {
        for (final Subscriber subscriber : m.values()) {
            if (needSkipCheck()) {
                subscriber.renew();
            } else {
                //检测消费者当前的状态
                if (subscriber.status() == Subscriber.Status.ONLINE) continue;
                subscriber.reset();
                actorSystem.dispatch("status-checker-" + subscriber.getGroup(), subscriber, this);
            }
        }
    }
}
```



```java
public boolean process(Subscriber subscriber, ActorSystem.Actor<Subscriber> self) {
    subscriber.checkStatus();
    return true;
}
```

### 检测消费者状态

```java
void checkStatus() {
    try {
        Status status = status(); //获取消费者的状态
        if (status == Status.OFFLINE) { //暂时下线处理
            if (processed.compareAndSet(false, true)) {
                retryTask.run();
            }
        }
        if (status == Status.FOREVER) { //永久下线处理
            if (processed.compareAndSet(false, true)) {
                offlineTask.run();
            }
        }
    } finally {
        processed.set(false);
    }
}
```



```java
Status status() {
    long now = System.currentTimeMillis();
    long interval = now - lastUpdate;
    if (interval >= FOREVER_LEASE_MILLIS) { //默认超过2天
        return Status.FOREVER;
    }
    if (interval >= OFFLINE_LEASE_MILLIS) { //默认超过3分钟
        return Status.OFFLINE;
    }
    return Status.ONLINE;
}
```

#### 暂时下线处理

查看消费者是否有已经拉取但是为ack的消费，如果有将未ack的消息重放到MessageLog文件中。

qunar.tc.qmq.consumer.RetryTask#run

```java
void run() {
    if (cancel) return;
    //获取已经拉取、ack的pull log sequence
    final ConsumerSequence consumerSequence = consumerSequenceManager.getConsumerSequence(subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
    if (consumerSequence == null) {
        return;
    }
    QMon.retryTaskExecuteCountInc(subscriber.getSubject(), subscriber.getGroup());
    while (true) {
        limiter.acquire();
        if (!consumerSequence.tryLock()) return;
        try {
            if (cancel) return;
            final long firstNotAckedSequence = consumerSequence.getAckSequence() + 1;
            final long lastPulledSequence = consumerSequence.getPullSequence();
            //表明拉取的消息都已经ack
            if (lastPulledSequence < firstNotAckedSequence) return;
            subscriber.renew();
            LOG.info("put need retry message in retry task, subject: {}, group: {}, consumerId: {}, ack offset: {}, pull offset: {}",
                    subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId(), firstNotAckedSequence, lastPulledSequence);
            //把未ack的消息重新放到MessageLog文件中（而不是将offset位移信息放到ConsumeLog文件中，应该是为了避免’挖坟‘，将尾部数据、热数据挤出内存）
            consumerSequenceManager.putNeedRetryMessages(subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId(), firstNotAckedSequence, firstNotAckedSequence);
            // 追加action消息，表示消息已经被ack
            final Action action = new RangeAckAction(subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId(), System.currentTimeMillis(), firstNotAckedSequence, firstNotAckedSequence);
            if (consumerSequenceManager.putAction(action)) {
                consumerSequence.setAckSequence(firstNotAckedSequence);
                QMon.consumerAckTimeoutErrorCountInc(subscriber.getConsumerId(), 1);
            }
        } finally {
            consumerSequence.unlock();
        }
    }
}
```

#### 永久下线处理

qunar.tc.qmq.consumer.OfflineTask#run

```java
void run() {
    if (cancel) return;
    LOG.info("run offline task for {}/{}/{}.", subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
    QMon.offlineTaskExecuteCountInc(subscriber.getSubject(), subscriber.getGroup());
    final ConsumerSequence consumerSequence = consumerSequenceManager.getOrCreateConsumerSequence(subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
    if (!consumerSequence.tryLock()) return;
    try {
        if (cancel) return;
        if (isProcessedComplete(consumerSequence)) { //拉取的消息都已经ack
            if (unSubscribe()) {
                LOG.info("offline task destroyed subscriber for {}/{}/{}", subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
            }
        } else {
            LOG.info("offline task skip destroy subscriber for {}/{}/{}", subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
        }
    } finally {
        consumerSequence.unlock();
    }
}
```



```java
private boolean unSubscribe() {
    if (cancel) return false;
  //追加ForeverOfflineAction到ActionLog
    final boolean success = consumerSequenceManager.putForeverOfflineAction(subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
    if (!success) {
        return false;
    }
    consumerSequenceManager.remove(subscriber.getSubject(), subscriber.getGroup(), subscriber.getConsumerId());
    subscriber.destroy();
    return true;
}
```

### 对FOREVER_OFFLINE类型的Action进行处理

```java
final OfflineActionHandler handler = new OfflineActionHandler(storage);
this.storage.registerActionEventListener(handler);
this.storage.start();
```

qunar.tc.qmq.consumer.OfflineActionHandler#onEvent

```java
public void onEvent(ActionEvent event) {
    switch (event.getAction().type()) {
        case FOREVER_OFFLINE:
            foreverOffline(event.getAction());
            break;
        default:
            break;

    }
}
```



```java
private void foreverOffline(final Action action) {
    final String subject = action.subject();
    final String group = action.group();
    final String consumerId = action.consumerId();

    LOG.info("execute offline task, will remove pull log and checkpoint entry for {}/{}/{}", subject, group, consumerId);
    storage.destroyPullLog(subject, group, consumerId);
}
```

qunar.tc.qmq.store.DefaultStorage#destroyPullLog

```java
public void destroyPullLog(String subject, String group, String consumerId) {
    if (pullLogManager.destroy(subject, group, consumerId)) { //销毁此消费者对应的Pull Log
        checkpointManager.removeConsumerProgress(subject, group, consumerId); //移除消费进度
    }
}
```

# 消费组隔离

比如有一个消息主题，有几个不同的消费组来消费，消费组里是包含多个消费者的。假设A消费组有100个消费者，而B消费组只有两个消费者，这样Server就会收到大量来自A消费组的拉请求，而来自B消费组的拉请求要少得多，服务端大部分资源都用来处理A消费组的请求，B消费组的请求有可能来不及处理，导致超时。

引入隔离，可以保证服务端可以均衡资源（时间片，每个消费组只会执行一个时间片，时间片耗尽执行其他消费组）处理不同消费组的请求

qunar.tc.qmq.processor.PullMessageProcessor#processRequest

```java
public CompletableFuture<Datagram> processRequest(final ChannelHandlerContext ctx, final RemotingCommand command) {
    final PullRequest pullRequest = pullRequestSerde.read(command.getHeader().getVersion(), command.getBody());
    BrokerStats.getInstance().getLastMinutePullRequestCount().add(1);
    QMon.pullRequestCountInc(pullRequest.getSubject(), pullRequest.getGroup());
    //检测、修正拉取请求的参数
    if (!checkAndRepairPullRequest(pullRequest)) {
        return CompletableFuture.completedFuture(crateErrorParamResult(command));
    }
    //Broker端维护group、subject、consumerId 信息
    subscribe(pullRequest);
    final PullEntry entry = new PullEntry(pullRequest, command.getHeader(), ctx);
    //消息消费隔离入口
    pullMessageWorker.pull(entry);
    return null;
}
```

```java
void pull(PullMessageProcessor.PullEntry pullEntry) {
  //创建actorPath，以subject_group为单位对消费请求进行调度
    final String actorPath = ConsumerGroupUtils.buildConsumerGroupKey(pullEntry.subject, pullEntry.group);
    actorSystem.dispatch(actorPath, pullEntry, this);
}
```

qunar.tc.qmq.concurrent.ActorSystem#dispatch

```java
public <E> void dispatch(String actorPath, E msg, Processor<E> processor) {
    Actor<E> actor = createOrGet(actorPath, processor); //根据subject、group获取Actor，或者创建Actor
    actor.dispatch(msg); //添加到Actor的queue中
    schedule(actor, true);
}
```



```java
private <E> boolean schedule(Actor<E> actor, boolean hasMessageHint) {
    if (!actor.canBeSchedule(hasMessageHint))//是否可以调度
      	return false;
    if (actor.setAsScheduled()) { //修改为调度状态成功
        actor.submitTs = System.currentTimeMillis();
        this.executor.execute(actor); //加入线程池，等待调度
        return true;
    }
    return false;
}
```

qunar.tc.qmq.concurrent.ActorSystem.Actor#canBeSchedule

```java
private boolean canBeSchedule(boolean hasMessageHint) {
    int s = currentStatus();
    if (s == Open || s == Scheduled) return hasMessageHint || !queue.isEmpty();
    return false;
}
```

qunar.tc.qmq.concurrent.ActorSystem.Actor#setAsScheduled

```java
final boolean setAsScheduled() {
    while (true) {
        int s = currentStatus();
        if ((s & shouldScheduleMask) != Open) return false;
        //修改为调度状态
        if (updateStatus(s, s | Scheduled)) return true;
    }
}
```

qunar.tc.qmq.concurrent.ActorSystem.Actor#run

```java
public void run() {
    long start = System.currentTimeMillis();
    String old = Thread.currentThread().getName();
    try {
        Thread.currentThread().setName(systemName + "-" + name);
        if (shouldProcessMessage()) { //是否执行
            processMessages();
        }
    } finally {
        long duration = System.currentTimeMillis() - start;
        total += duration;
        QMon.actorProcessTime(name, duration);

        Thread.currentThread().setName(old);
        setAsIdle();
        this.actorSystem.schedule(this, false);
    }
}
```

```java
void processMessages() {
    long deadline = System.currentTimeMillis() + QUOTA; //时间限制
    while (true) {
        E message = queue.peek(); //拉取请求
        if (message == null) return;
        boolean process = processor.process(message, this); //执行请求
        if (!process) return;

        queue.pollNode();
        if (System.currentTimeMillis() >= deadline) return; //时间片耗尽
    }
}
```

# 分布式事务

**本地事务消息表**

## 流程

1. 开启本地事务
2. 执行业务操作
3. 向同实例消息库插入消息
4. 事务提交
5. 向server发送消息
6. 如果server回复成功则删除消息
7.  开启补偿任务，扫描未发送消息
8. 发送补偿消息
9. 删除补偿成功的消息

## 实现

对TransactionSynchronization进行扩展，在事务提交之前，将事务消息插入数据库，在事务提交之后将消息发送到消息服务器，发送消息回复成功将消息删除。**避免在事务中执行发送消息的操作，因为发送消息涉及到网络传输，网络传输耗时长，容易出现长事务，长时间占用数据库锁，降低数据库的吞吐量**

### 1、SpringTransactionProvider

qunar.tc.qmq.producer.tx.spring.SpringTransactionProvider#SpringTransactionProvider(javax.sql.DataSource)

```java
public SpringTransactionProvider(DataSource bizDataSource) { 
    this.store = new DefaultMessageStore(bizDataSource);
}
```

### 2、MessageTracker

将TransactionProvider注入到MessageProducerProvider

qunar.tc.qmq.producer.MessageProducerProvider#setTransactionProvider

```java
public void setTransactionProvider(TransactionProvider transactionProvider) {
    this.messageTracker = new MessageTracker(transactionProvider);
}
```

### 3、发送消息

qunar.tc.qmq.producer.MessageProducerProvider#sendMessage(qunar.tc.qmq.Message, qunar.tc.qmq.MessageSendStateListener)

```java
if (!messageTracker.trackInTransaction(pm)) { //非事务消息
    spanBuilder.withTag("messageType", "PersistenceMessage");
    scope = spanBuilder.startActive(true);
    tagValues = new String[]{message.getSubject(), "PersistenceMessage"};
    pm.send();
} else { //事务消息
    spanBuilder.withTag("messageType", "TransactionMessage");
    scope = spanBuilder.startActive(true);
    tagValues = new String[]{message.getSubject(), "TransactionMessage"};
}
```

qunar.tc.qmq.producer.tx.MessageTracker#trackInTransaction

```java
public boolean trackInTransaction(ProduceMessage message) {
    MessageStore messageStore = this.transactionProvider.messageStore();
    message.setStore(messageStore);
    if (transactionProvider.isInTransaction()) { //事务中
        this.transactionListener.beginTransaction(messageStore); //绑定线程私有的TransactionMessageHolder，存储需要发送的消息
        this.transactionProvider.setTransactionListener(transactionListener);
        messageStore.beginTransaction();
        this.transactionListener.addMessage(message); //存放到线程私有的TransactionMessageHolder的队列中
        return true;
    } else { //无事务
        try {
            message.save();
        } catch (Exception e) {
            message.getBase().setStoreAtFailed(true);
        }
        return false;
    }
}
```

### 4、事务拦截处理

qunar.tc.qmq.producer.tx.spring.SpringTransactionProvider

对Spring事务的扩展 ，拦截事务操作

```java
//事务提交之前
@Override
public void beforeCommit(boolean readOnly) {
    if (readOnly) return;
    if (transactionListener != null) transactionListener.beforeCommit(); //写入消息表
}
//事务提交之后
@Override
public void afterCommit() {
    if (transactionListener != null) transactionListener.afterCommit(); //发送消息
}
//操作完成之后
@Override
public void afterCompletion(int status) {
    if (transactionListener != null) transactionListener.afterCompletion();
}
```

qunar.tc.qmq.producer.tx.DefaultTransactionListener

```java
//事务执行前，将事务消息插入数据库
@Override
public void beforeCommit() {
    List<ProduceMessage> list = get();
    for (ProduceMessage msg : list) {
        msg.save();
    }
}

//事务提交成功之后，发送消息
@Override
public void afterCommit() {
    List<ProduceMessage> list = remove();
    for (int i = 0; i < list.size(); ++i) {
        ProduceMessage msg = list.get(i);
        try {
            msg.send();
        } catch (Throwable t) {
            logger.error("消息发送失败{}", msg.getMessageId(), t);
        }
    }
}
```

# 延时/定时消息

QMQ提供任意时间的延时/定时消息，延时消息和普通分开存储，需要单独部署qmq-delay-server。当写入消息时，直接顺序追加到磁盘文件末尾。开启线程读取文件，判断消息是否可以加入内存中的时间轮，默认存放1小时内的消息，否则将消息存放到对应的schedulelog文件中

qunar.tc.qmq.delay.DefaultDelayLogFacade#DefaultDelayLogFacade

```java
public DefaultDelayLogFacade(final StoreConfiguration config, final Function<ScheduleIndex, Boolean> func) {
    this.messageLog = new MessageLog(config);
    this.scheduleLog = new ScheduleLog(config);
    this.dispatchLog = new DispatchLog(config);
    this.offsetManager = new IterateOffsetManager(config.getCheckpointStorePath(), scheduleLog::flush); //负责存储MessageLog回放的位置，定时保存到文件，启动时从读文件，加载到内存
    FixedExecOrderEventBus bus = new FixedExecOrderEventBus();
    bus.subscribe(MessageLogRecord.class, e -> {
        AppendLogResult<ScheduleIndex> result = appendScheduleLog(e); //根据延迟时间追加到对应的schedulelog文件中（存放消息在MessageLog文件中的offet、消息的大小）
        int code = result.getCode();
        if (MessageProducerCode.SUCCESS != code) {
            LOGGER.error("appendMessageLog schedule log error,log:{} {},code:{}", e.getSubject(), e.getMessageId(), code);
            throw new AppendException("appendScheduleLogError");
        }
        func.apply(result.getAdditional()); //是否要放入内存时间轮
    });
    bus.subscribe(MessageLogRecord.class, e -> { //更新回放MessageLog的位置，下次重启从此位置开始回放
        long checkpoint = e.getStartWroteOffset() + e.getRecordSize();
        updateIterateOffset(checkpoint);
    });
    //1、根据之前的offset，回放MessageLog；
    //2、迭代message，派发给FixedExecOrderEventBus；
    //3、FixedExecOrderEventBus将message放置到对应的schedulelog上；
    //4、FixedExecOrderEventBus更新回放的offset
    //5、判断是否需要将ScheduleIndex放入到内存时间轮
    this.messageLogIterateService = new LogIterateService<>("message-log", 5, messageLog, initialMessageIterateFrom(), bus);

    this.logFlusher = new LogFlusher(messageLog, offsetManager, dispatchLog);
    this.cleaner = new LogCleaner(config, dispatchLog, scheduleLog, messageLog);
}
```

## 重放MessageLog

qunar.tc.qmq.store.LogIterateService.Dispatcher#run

```java
public void run() {
    while (!stop) {
        try {
            processLog();
        } catch (Throwable e) {
            QMon.replayLogFailedCountInc(name + "Failed");
            LOG.error("replay log failed, will retry.", e);
        }
    }
}
```

```java
private void processLog() {
    final long startOffset = iterateFrom.longValue();
    try (AbstractLogVisitor<T> visitor = visitable.newVisitor(startOffset)) {
        if (startOffset != visitor.getStartOffset()) {
            LOG.info("reset iterate from offset from {} to {}", startOffset, visitor.getStartOffset());
            iterateFrom.reset();
            iterateFrom.add(visitor.getStartOffset());
        }

        while (true) {
            final LogVisitorRecord<T> record = visitor.nextRecord();
            if (record.isNoMore()) {
                break;
            }

            if (record.hasData()) {
                dispatcher.post(record.getData()); //发布事件
            }
        }
        iterateFrom.add(visitor.visitedBufferSize());
    }

    try {
        TimeUnit.MILLISECONDS.sleep(dispatcherPauseMills);
    } catch (InterruptedException e) {
        LOG.warn("log dispatcher sleep interrupted");
    }
}
```

## 加入ScheduleLog

根据延迟时间计算所属DelaySegment，将消息在messageLog中的索引信息保存到DelaySegment

qunar.tc.qmq.delay.DefaultDelayLogFacade#appendScheduleLog

```java
public AppendLogResult<ScheduleIndex> appendScheduleLog(LogRecord event) {
    return scheduleLog.append(event);
}
```

qunar.tc.qmq.delay.store.log.ScheduleLog#append

```java
public AppendLogResult<ScheduleIndex> append(LogRecord record) {
    if (!open.get()) {
        return new AppendLogResult<>(MessageProducerCode.STORE_ERROR, "schedule log closed");
    }
    AppendLogResult<RecordResult<ScheduleSetSequence>> result = scheduleSet.append(record);
    int code = result.getCode();
    if (MessageProducerCode.SUCCESS != code) {
        LOGGER.error("appendMessageLog schedule set error,log:{} {},code:{}", record.getSubject(), record.getMessageId(), code);
        return new AppendLogResult<>(MessageProducerCode.STORE_ERROR, "appendScheduleSetError");
    }

    RecordResult<ScheduleSetSequence> recordResult = result.getAdditional();
    //存放在MessageLog中的offset
    ScheduleIndex index = new ScheduleIndex(
            record.getSubject(),
            record.getScheduleTime(),
            recordResult.getResult().getWroteOffset(),
            recordResult.getResult().getWroteBytes(),
            recordResult.getResult().getAdditional().getSequence());

    return new AppendLogResult<>(MessageProducerCode.SUCCESS, "", index);
}
```

qunar.tc.qmq.delay.store.log.AbstractDelayLog#append

```java
public AppendLogResult<RecordResult<T>> append(LogRecord record) {
    String subject = record.getSubject();
    RecordResult<T> result = container.append(record); //写数据
    PutMessageStatus status = result.getStatus();
    if (PutMessageStatus.SUCCESS != status) {
        LOGGER.error("appendMessageLog schedule set file error,subject:{},status:{}", subject, status.name());
        return new AppendLogResult<>(MessageProducerCode.STORE_ERROR, status.name(), null);
    }

    return new AppendLogResult<>(MessageProducerCode.SUCCESS, status.name(), result);
}
```

qunar.tc.qmq.delay.store.log.AbstractDelaySegmentContainer#append

```java
public RecordResult<T> append(LogRecord record) {
    long scheduleTime = record.getScheduleTime();
    DelaySegment<T> segment = locateSegment(scheduleTime);//根据延迟时间获取DelaySegment
    if (null == segment) {
        segment = allocNewSegment(scheduleTime); //创建DelaySegment
    }

    if (null == segment) {
        return new NopeRecordResult(PutMessageStatus.CREATE_MAPPED_FILE_FAILED);
    }

    return retResult(segment.append(record, appender)); //写数据
}
```

qunar.tc.qmq.delay.store.log.AbstractDelaySegmentContainer#locateSegment

```java
DelaySegment<T> locateSegment(long scheduleTime) {
    long baseOffset = resolveSegment(scheduleTime, segmentScale);//根据延迟时间计算baseOffset
    return segments.get(baseOffset); //根据baseOffset获取DelaySegment
}
```

## 加入时间轮

qunar.tc.qmq.delay.startup.ServerWrapper#iterateCallback

```java
private boolean iterateCallback(final ScheduleIndex index) {
    long scheduleTime = index.getScheduleTime();
    long offset = index.getOffset();
    if (wheelTickManager.canAdd(scheduleTime, offset)) { //是否可以加入时间轮
        wheelTickManager.addWHeel(index); //加入时间轮
        return true;
    }
    return false;
}
```

qunar.tc.qmq.delay.wheel.WheelTickManager#canAdd

```java
public boolean canAdd(long scheduleTime, long offset) { //最近1小时的消息加入时间轮
    WheelLoadCursor.Cursor currentCursor = loadingCursor.cursor();
    long currentBaseOffset = currentCursor.getBaseOffset(); //默认-1
    long currentOffset = currentCursor.getOffset();//默认-1

    long baseOffset = resolveSegment(scheduleTime, segmentScale); //根据延迟时间计算baseOffset
    if (baseOffset < currentBaseOffset) return true; 

    if (baseOffset == currentBaseOffset) {
        return currentOffset <= offset;
    }
    return false;
}
```

qunar.tc.qmq.delay.wheel.WheelTickManager#addWHeel

```java
public void addWHeel(ScheduleIndex index) { //加入时间轮
    refresh(index);
}
```

qunar.tc.qmq.delay.wheel.WheelTickManager#refresh

```java
private void refresh(ScheduleIndex index) {
    long now = System.currentTimeMillis();
    long scheduleTime = now;
    try {
        scheduleTime = index.getScheduleTime();
        timer.newTimeout(index, scheduleTime - now, TimeUnit.MILLISECONDS);
    } catch (Throwable e) {
        LOGGER.error("wheel refresh error, scheduleTime:{}, delay:{}", scheduleTime, scheduleTime - now);
        throw Throwables.propagate(e);
    }
}
```

qunar.tc.qmq.delay.wheel.WheelTickManager#process

```java
public void process(ScheduleIndex index) { //延迟任务
    QMon.scheduleDispatch(); 
    sender.send(index);//将到期的延时消息发送给实时server
}
```

# References

[官方文档](https://github.com/qunarcorp/qmq)

# 总结

1、**计算机中所有问题都可以通过添加一个中间层来解决** 

2、请求隔离，避免相互影响