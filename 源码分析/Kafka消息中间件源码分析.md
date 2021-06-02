# Kafka源码分析

* [网络通信](#网络通信)
  * [初始化](#初始化)
  * [创建控制面板](#创建控制面板)
  * [创建数据面板](#创建数据面板)
  * [创建Acceptor](#创建acceptor)
  * [启动Acceptor](#启动acceptor)
  * [创建Processor](#创建processor)
  * [启动Processor](#启动processor)
* [请求处理](#请求处理)
  * [创建KafkaRequestHandlerPool](#创建kafkarequesthandlerpool)
    * [创建KafkaRequestHandler](#创建kafkarequesthandler)
    * [动态调整线程池大小](#动态调整线程池大小)
  * [启动KafkaRequestHandler](#启动kafkarequesthandler)
* [KafkaController](#kafkacontroller)
  * [选举](#选举)
* [消息追加](#消息追加)
* [索引](#索引)
  * [追加索引](#追加索引)
  * [查找索引](#查找索引)
* [消息读取](#消息读取)
* [总结](#总结)


# 网络通信

kafka.server.KafkaServer#startup

```scala
socketServer = new SocketServer(config, metrics, time, credentialProvider)
socketServer.startup(startupProcessors = false)
```

## 初始化

将请求分为控制请求和数据请求，控制类请求会影响数据类请求.控制请求比较少有3种类型：LeaderAndIsr、StopReplica、UpdateMetadata

kafka.network.SocketServer#startup

```scala
def startup(startupProcessors: Boolean = true): Unit = {
  this.synchronized {
    connectionQuotas = new ConnectionQuotas(config, time)
    createControlPlaneAcceptorAndProcessor(config.controlPlaneListener) //控制面板，处理controller发送的请求
    createDataPlaneAcceptorsAndProcessors(config.numNetworkThreads, config.dataPlaneListeners) //数据面板，处理客户端、其他broker发送的请求
    if (startupProcessors) { //启动Processor
      startControlPlaneProcessor(Map.empty) //启动控制面板
      startDataPlaneProcessors(Map.empty) //启动数据面板
    }
  }
}
```

## 创建控制面板

kafka.network.SocketServer#createControlPlaneAcceptorAndProcessor

```scala
//创建控制面板的Acceptors和Processors线程（control.plane.listener.name=CONTROLLE）；控制面板的EndPoint只有一个
private def createControlPlaneAcceptorAndProcessor(endpointOpt: Option[EndPoint]): Unit = synchronized {
  endpointOpt.foreach { endpoint =>
    connectionQuotas.addListener(config, endpoint.listenerName)
    //创建Acceptor
    val controlPlaneAcceptor = createAcceptor(endpoint, ControlPlaneMetricPrefix)
    //只需一个Processor处理控制类请求
    val controlPlaneProcessor = newProcessor(nextProcessorId, controlPlaneRequestChannelOpt.get, connectionQuotas, endpoint.listenerName, endpoint.securityProtocol, memoryPool)
    controlPlaneAcceptorOpt = Some(controlPlaneAcceptor)
    controlPlaneProcessorOpt = Some(controlPlaneProcessor)
    val listenerProcessors = new ArrayBuffer[Processor]()
    listenerProcessors += controlPlaneProcessor
    //RequestChannel维护Processor,id -> Processor
    controlPlaneRequestChannelOpt.foreach(_.addProcessor(controlPlaneProcessor))
    nextProcessorId += 1
    //将Processor绑定到Acceptor
    controlPlaneAcceptor.addProcessors(listenerProcessors, ControlPlaneThreadPrefix)
    //启动Acceptor
    KafkaThread.nonDaemon(s"${ControlPlaneThreadPrefix}-kafka-socket-acceptor-${endpoint.listenerName}-${endpoint.securityProtocol}-${endpoint.port}", controlPlaneAcceptor).start()
    //等待Acceptor初始化完成
    controlPlaneAcceptor.awaitStartup()
    info(s"Created control-plane acceptor and processor for endpoint : $endpoint")
  }
}
```

## 创建数据面板

kafka.network.SocketServer#createDataPlaneAcceptorsAndProcessors

```scala
//创建数据面板的Acceptors和Processors线程
private def createDataPlaneAcceptorsAndProcessors(dataProcessorsPerListener: Int,
                                                  endpoints: Seq[EndPoint]): Unit = synchronized {
  //istener.security.protocol.map=CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNA
  //listeners=CONTROLLER://192.1.1.8:9091,INTERNAL://192.1.1.8:9092
  //control.plane.listener.name=CONTROLLE
  endpoints.foreach { endpoint => //host:port:listenerName:securityProtocol
    connectionQuotas.addListener(config, endpoint.listenerName)
    //创建Acceptor
    val dataPlaneAcceptor = createAcceptor(endpoint, DataPlaneMetricPrefix)
    //创建Processors，将其绑定到Acceptor
    addDataPlaneProcessors(dataPlaneAcceptor, endpoint, dataProcessorsPerListener)
    //启动Acceptor
    KafkaThread.nonDaemon(s"data-plane-kafka-socket-acceptor-${endpoint.listenerName}-${endpoint.securityProtocol}-${endpoint.port}", dataPlaneAcceptor).start()
    //等待Acceptor初始化完成
    dataPlaneAcceptor.awaitStartup()
    dataPlaneAcceptors.put(endpoint, dataPlaneAcceptor)
    info(s"Created data-plane acceptor and processors for endpoint : $endpoint")
  }
}
```

## 创建Acceptor

kafka.network.SocketServer#createAcceptor

```scala
//创建Accpetor
private def createAcceptor(endPoint: EndPoint, metricPrefix: String) : Acceptor = synchronized {
  val sendBufferSize = config.socketSendBufferBytes
  val recvBufferSize = config.socketReceiveBufferBytes
  val brokerId = config.brokerId
  new Acceptor(endPoint, sendBufferSize, recvBufferSize, brokerId, connectionQuotas, metricPrefix)
}
```

## 启动Acceptor

```scala
def run(): Unit = {
  //注册Accept事件
  serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
  startupComplete()
  try {
    var currentProcessorIndex = 0
    while (isRunning) {
      try {
        val ready = nioSelector.select(500)
        if (ready > 0) {
          val keys = nioSelector.selectedKeys()
          val iter = keys.iterator()
          while (iter.hasNext && isRunning) {
            try {
              val key = iter.next
              iter.remove()
              if (key.isAcceptable) { //有客户端连接事件发生
                accept(key).foreach { socketChannel =>
                  var retriesLeft = synchronized(processors.length)
                  var processor: Processor = null
                  do {
                    retriesLeft -= 1
                    processor = synchronized {
                      //选择一个Processor，每个SocketChannel对应一个Processor，一个Processor可以处理多个SocketChannel的读写事件
                      currentProcessorIndex = currentProcessorIndex % processors.length 
                      processors(currentProcessorIndex)
                    }
                    currentProcessorIndex += 1
                    //绑定SocketChannel和Processor，将SocketChannel放入newConnections队列
                  } while (!assignNewConnection(socketChannel, processor, retriesLeft == 0))
                }
              } else
                throw new IllegalStateException("Unrecognized key state for acceptor thread.")
            } catch {
              case e: Throwable => error("Error while accepting connection", e)
            }
          }
        }
      }
      catch {
        case e: ControlThrowable => throw e
        case e: Throwable => error("Error occurred", e)
      }
    }
  } finally {
    debug("Closing server socket and selector.")
    CoreUtils.swallow(serverChannel.close(), this, Level.ERROR)
    CoreUtils.swallow(nioSelector.close(), this, Level.ERROR)
    shutdownComplete()
  }
}
```

## 创建Processor

kafka.network.SocketServer#addDataPlaneProcessors

```scala
private def addDataPlaneProcessors(acceptor: Acceptor, endpoint: EndPoint, newProcessorsPerListener: Int): Unit = synchronized {
  val listenerName = endpoint.listenerName
  val securityProtocol = endpoint.securityProtocol
  val listenerProcessors = new ArrayBuffer[Processor]()
  for (_ <- 0 until newProcessorsPerListener) {
    //创建Processors
    val processor = newProcessor(nextProcessorId, dataPlaneRequestChannel, connectionQuotas, listenerName, securityProtocol, memoryPool)
    listenerProcessors += processor
    //数据面板的RequestChannel,维护Processor,id->Processor
    dataPlaneRequestChannel.addProcessor(processor)
    nextProcessorId += 1
  }
  listenerProcessors.foreach(p => dataPlaneProcessors.put(p.id, p))
  //将Processors绑定到Acceptor，并启动Processor
  acceptor.addProcessors(listenerProcessors, DataPlaneThreadPrefix)
}
```

## 启动Processor

kafka.network.Processor#run

```scala
override def run(): Unit = {
  startupComplete()
  try {
    while (isRunning) {
      try {
        configureNewConnections() //处理Acceptor接收的新连接,注册READ事件
        processNewResponses() // 发送响应信息。每个Processor有单独的responseQueue队列，存放响应信息
        poll()
        processCompletedReceives() //将接受的数据封装成Request，放入RequestChannel
        processCompletedSends()//处理发送成功的响应
        processDisconnected() //处理断开的连接
        closeExcessConnections() //关闭channel
      } catch {
        case e: Throwable => processException("Processor got uncaught exception.", e)
      }
    }
  } finally {
    debug(s"Closing selector - processor $id")
    CoreUtils.swallow(closeAll(), this, Level.ERROR)
    shutdownComplete()
  }
}
```

# 请求处理

控制请求和数据请求，分别使用不同的线程池处理各自的请求

kafka.server.KafkaServer#startup

```scala
//数据面板
dataPlaneRequestProcessor = new KafkaApis(socketServer.dataPlaneRequestChannel, replicaManager, adminManager, groupCoordinator, transactionCoordinator,
  kafkaController, zkClient, config.brokerId, config, metadataCache, metrics, authorizer, quotaManagers,
  fetchManager, brokerTopicStats, clusterId, time, tokenManager)
dataPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.dataPlaneRequestChannel, dataPlaneRequestProcessor, time,
  config.numIoThreads, s"${SocketServer.DataPlaneMetricPrefix}RequestHandlerAvgIdlePercent", SocketServer.DataPlaneThreadPrefix)
//控制面板
socketServer.controlPlaneRequestChannelOpt.foreach { controlPlaneRequestChannel =>
  controlPlaneRequestProcessor = new KafkaApis(controlPlaneRequestChannel, replicaManager, adminManager, groupCoordinator, transactionCoordinator,
    kafkaController, zkClient, config.brokerId, config, metadataCache, metrics, authorizer, quotaManagers,
    fetchManager, brokerTopicStats, clusterId, time, tokenManager)
  controlPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.controlPlaneRequestChannelOpt.get, controlPlaneRequestProcessor, time,
    1, s"${SocketServer.ControlPlaneMetricPrefix}RequestHandlerAvgIdlePercent", SocketServer.ControlPlaneThreadPrefix)
}
```

## 创建KafkaRequestHandlerPool

```scala
val brokerId: Int, //Broker标志
val requestChannel: RequestChannel, //Processors将接收的请求存放到RequestChannel的requestQueue中
val apis: KafkaApis, //真正处理请求的类
numThreads: Int, //线程池大小，可动态调整， num.io.threads,
```

### 创建KafkaRequestHandler

kafka.server.KafkaRequestHandlerPool#createHandler

```scala
def createHandler(id: Int): Unit = synchronized {
  runnables += new KafkaRequestHandler(id, brokerId, aggregateIdleMeter, threadPoolSize, requestChannel, apis, time)
  KafkaThread.daemon(logAndThreadNamePrefix + "-kafka-request-handler-" + id, runnables(id)).start()
}
```

### 动态调整线程池大小

```scala
def resizeThreadPool(newSize: Int): Unit = synchronized {
  val currentSize = threadPoolSize.get
  info(s"Resizing request handler thread pool size from $currentSize to $newSize")
  if (newSize > currentSize) { //增加IO线程池大小
    for (i <- currentSize until newSize) {
      createHandler(i)
    }
  } else if (newSize < currentSize) { //减小IO线程池大小
    for (i <- 1 to (currentSize - newSize)) {
      runnables.remove(currentSize - i).stop() //停止线程
    }
  }
  threadPoolSize.set(newSize)
}
```

## 启动KafkaRequestHandler

kafka.server.KafkaRequestHandler#run

```java
def run(): Unit = {
  while (!stopped) { //线程尚未关闭，继续循环执行
    val startSelectTime = time.nanoseconds
    val req = requestChannel.receiveRequest(300) //从请求队列中获取待处理的请求
    val endTime = time.nanoseconds
    val idleTime = endTime - startSelectTime
    aggregateIdleMeter.mark(idleTime / totalHandlerThreads.get)
    req match {
      case RequestChannel.ShutdownRequest => //关闭请求
        debug(s"Kafka request handler $id on broker $brokerId received shut down command")
        shutdownComplete.countDown()
        return
      case request: RequestChannel.Request => //普通请求
        try {
          request.requestDequeueTimeNanos = endTime
          trace(s"Kafka request handler $id on broker $brokerId handling request $request")
          apis.handle(request) //调用KafkaApis的handler方法进行相应的处理
        } catch {
          case e: FatalExitError =>
            shutdownComplete.countDown()
            Exit.exit(e.statusCode)
          case e: Throwable => error("Exception when handling request", e)
        } finally {
          request.releaseBuffer()
        }
      case null => // continue
    }
  }
  shutdownComplete.countDown()
}
```

# KafkaController

## 选举

在 Kafka 集群中，只有一台Broker可以成为Controller。选举过程依赖 ZooKeeper完成，通过创建 /controller 临时节点（Ephemeral Node），协助完成 Controller 的选举，所有的broker会监听这个节点是否存在，如果节点不存在，就会触发Controller选举

kafka.controller.KafkaController#startup

```scala
 def startup() = {
    zkClient.registerStateChangeHandler(new StateChangeHandler {
      override val name: String = StateChangeHandlers.ControllerHandler
      override def afterInitializingSession(): Unit = {
        eventManager.put(RegisterBrokerAndReelect)
      }
      override def beforeInitializingSession(): Unit = {
        val queuedEvent = eventManager.clearAndPut(Expire)
        queuedEvent.awaitProcessing()
      }
    })
    eventManager.put(Startup) //写入Startup事件到事件队列
    eventManager.start() //启动事件处理线程
  }
```

kafka.controller.KafkaController#processStartup

```scala
private def processStartup(): Unit = {
  zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
  elect()
}
```

```scala
private def elect(): Unit = {
  activeControllerId = zkClient.getControllerId.getOrElse(-1)
  /*
   * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition,
   * it's possible that the controller has already been elected when we get here. This check will prevent the following
   * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.
   */
  if (activeControllerId != -1) {
    debug(s"Broker $activeControllerId has been elected as the controller, so stopping the election process.")
    return
  }

  try {
    val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)
    controllerContext.epoch = epoch
    controllerContext.epochZkVersion = epochZkVersion
    activeControllerId = config.brokerId

    info(s"${config.brokerId} successfully elected as the controller. Epoch incremented to ${controllerContext.epoch} " +
      s"and epoch zk version is now ${controllerContext.epochZkVersion}")

    onControllerFailover()
  } catch {
    case e: ControllerMovedException =>
      maybeResign()

      if (activeControllerId != -1)
        debug(s"Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}", e)
      else
        warn("A controller has been elected but just resigned, this will result in another round of election", e)

    case t: Throwable =>
      error(s"Error while electing or becoming controller on broker ${config.brokerId}. " +
        s"Trigger controller movement immediately", t)
      triggerControllerMove()
  }
}
```

# 消息追加

kafka.log.Log#append

```scala
private def append(records: MemoryRecords,
                   origin: AppendOrigin,
                   interBrokerProtocolVersion: ApiVersion,
                   assignOffsets: Boolean,
                   leaderEpoch: Int): LogAppendInfo = {
  maybeHandleIOException(s"Error while appending records to $topicPartition in dir ${dir.getParent}") {
    //验证消息
    val appendInfo = analyzeAndValidateRecords(records, origin)
    //无有效数据
    if (appendInfo.shallowCount == 0)
      return appendInfo
    //删除无效消息
    var validRecords = trimInvalidBytes(records, appendInfo)
    lock synchronized {
      checkIfMemoryMappedBufferClosed()
      //分配位移
      if (assignOffsets) {
        val offset = new LongRef(nextOffsetMetadata.messageOffset)
        appendInfo.firstOffset = Some(offset.value)
        val now = time.milliseconds
        //验证消息、分配逻辑位移，压缩类型不一致导致重压缩
        val validateAndOffsetAssignResult = try {
          LogValidator.validateMessagesAndAssignOffsets(validRecords,
            topicPartition,
            offset,
            time,
            now,
            appendInfo.sourceCodec,
            appendInfo.targetCodec,
            config.compact,
            config.messageFormatVersion.recordVersion.value,
            config.messageTimestampType,
            config.messageTimestampDifferenceMaxMs,
            leaderEpoch,
            origin,
            interBrokerProtocolVersion,
            brokerTopicStats)
        } catch {
          case e: IOException =>
            throw new KafkaException(s"Error validating messages while appending to log $name", e)
        }
        validRecords = validateAndOffsetAssignResult.validatedRecords
        appendInfo.maxTimestamp = validateAndOffsetAssignResult.maxTimestamp
        appendInfo.offsetOfMaxTimestamp = validateAndOffsetAssignResult.shallowOffsetOfMaxTimestamp
        appendInfo.lastOffset = offset.value - 1
        appendInfo.recordConversionStats = validateAndOffsetAssignResult.recordConversionStats
        if (config.messageTimestampType == TimestampType.LOG_APPEND_TIME)
          appendInfo.logAppendTime = now

        // re-validate message sizes if there's a possibility that they have changed (due to re-compression or message
        // format conversion)
        //重新验证消息大小，有可能由于重压缩或者消息格式转换导致消息大小发生改变
        if (validateAndOffsetAssignResult.messageSizeMaybeChanged) {
          for (batch <- validRecords.batches.asScala) {
            if (batch.sizeInBytes > config.maxMessageSize) {
              // we record the original message set size instead of the trimmed size
              // to be consistent with pre-compression bytesRejectedRate recording
              brokerTopicStats.topicStats(topicPartition.topic).bytesRejectedRate.mark(records.sizeInBytes)
              brokerTopicStats.allTopicsStats.bytesRejectedRate.mark(records.sizeInBytes)
              throw new RecordTooLargeException(s"Message batch size is ${batch.sizeInBytes} bytes in append to" +
                s"partition $topicPartition which exceeds the maximum configured size of ${config.maxMessageSize}.")
            }
          }
        }
      } else { //offset无序
        // we are taking the offsets we are given
        if (!appendInfo.offsetsMonotonic)
          throw new OffsetsOutOfOrderException(s"Out of order offsets found in append to $topicPartition: " +
                                               records.records.asScala.map(_.offset))

        if (appendInfo.firstOrLastOffsetOfFirstBatch < nextOffsetMetadata.messageOffset) {
          // we may still be able to recover if the log is empty
          // one example: fetching from log start offset on the leader which is not batch aligned,
          // which may happen as a result of AdminClient#deleteRecords()
          val firstOffset = appendInfo.firstOffset match {
            case Some(offset) => offset
            case None => records.batches.asScala.head.baseOffset()
          }

          val firstOrLast = if (appendInfo.firstOffset.isDefined) "First offset" else "Last offset of the first batch"
          throw new UnexpectedAppendOffsetException(
            s"Unexpected offset in append to $topicPartition. $firstOrLast " +
            s"${appendInfo.firstOrLastOffsetOfFirstBatch} is less than the next offset ${nextOffsetMetadata.messageOffset}. " +
            s"First 10 offsets in append: ${records.records.asScala.take(10).map(_.offset)}, last offset in" +
            s" append: ${appendInfo.lastOffset}. Log start offset = $logStartOffset",
            firstOffset, appendInfo.lastOffset)
        }
      }

      // update the epoch cache with the epoch stamped onto the message by the leader
      validRecords.batches.asScala.foreach { batch =>
        if (batch.magic >= RecordBatch.MAGIC_VALUE_V2) {
          maybeAssignEpochStartOffset(batch.partitionLeaderEpoch, batch.baseOffset)
        } else {
          // In partial upgrade scenarios, we may get a temporary regression to the message format. In
          // order to ensure the safety of leader election, we clear the epoch cache so that we revert
          // to truncation by high watermark after the next leader election.
          leaderEpochCache.filter(_.nonEmpty).foreach { cache =>
            warn(s"Clearing leader epoch cache after unexpected append with message format v${batch.magic}")
            cache.clearAndFlush()
          }
        }
      }

      // check messages set size may be exceed config.segmentSize
      if (validRecords.sizeInBytes > config.segmentSize) {
        throw new RecordBatchTooLargeException(s"Message batch size is ${validRecords.sizeInBytes} bytes in append " +
          s"to partition $topicPartition, which exceeds the maximum configured segment size of ${config.segmentSize}.")
      }

      //如果activeLogsegment已经写满不足以存放当前写入的消息，则生成新的Logsegment
      // maybe roll the log if this segment is full
      val segment = maybeRoll(validRecords.sizeInBytes, appendInfo)

      val logOffsetMetadata = LogOffsetMetadata(
        messageOffset = appendInfo.firstOrLastOffsetOfFirstBatch,
        segmentBaseOffset = segment.baseOffset,
        relativePositionInSegment = segment.size)
      //验证事务状态
      // now that we have valid records, offsets assigned, and timestamps updated, we need to
      // validate the idempotent/transactional state of the producers and collect some metadata
      val (updatedProducers, completedTxns, maybeDuplicate) = analyzeAndValidateProducerState(
        logOffsetMetadata, validRecords, origin)

      maybeDuplicate.foreach { duplicate =>
        appendInfo.firstOffset = Some(duplicate.firstOffset)
        appendInfo.lastOffset = duplicate.lastOffset
        appendInfo.logAppendTime = duplicate.timestamp
        appendInfo.logStartOffset = logStartOffset
        return appendInfo
      }

      //执行真正的消息写入操作
      segment.append(largestOffset = appendInfo.lastOffset,
        largestTimestamp = appendInfo.maxTimestamp,
        shallowOffsetOfMaxTimestamp = appendInfo.offsetOfMaxTimestamp,
        records = validRecords)

      //LEO是消息集合中最后一条消息位移值+1
      updateLogEndOffset(appendInfo.lastOffset + 1)

      // update the producer state
      for (producerAppendInfo <- updatedProducers.values) {
        producerStateManager.update(producerAppendInfo)
      }

      // update the transaction index with the true last stable offset. The last offset visible
      // to consumers using READ_COMMITTED will be limited by this value and the high watermark.
      for (completedTxn <- completedTxns) {
        val lastStableOffset = producerStateManager.lastStableOffset(completedTxn)
        segment.updateTxnIndex(completedTxn, lastStableOffset)
        producerStateManager.completeTxn(completedTxn)
      }

      // always update the last producer id map offset so that the snapshot reflects the current offset
      // even if there isn't any idempotent data being written
      producerStateManager.updateMapEndOffset(appendInfo.lastOffset + 1)

      // update the first unstable offset (which is used to compute LSO)
      maybeIncrementFirstUnstableOffset()

      trace(s"Appended message set with last offset: ${appendInfo.lastOffset}, " +
        s"first offset: ${appendInfo.firstOffset}, " +
        s"next offset: ${nextOffsetMetadata.messageOffset}, " +
        s"and messages: $validRecords")
      //是否需要手动落盘
      if (unflushedMessages >= config.flushInterval)
        flush()
      appendInfo
    }
  }
}
```

kafka.log.LogSegment#append

```scala
def append(largestOffset: Long,
           largestTimestamp: Long,
           shallowOffsetOfMaxTimestamp: Long,
           records: MemoryRecords): Unit = {
  if (records.sizeInBytes > 0) {
    trace(s"Inserting ${records.sizeInBytes} bytes at end offset $largestOffset at position ${log.sizeInBytes} " +
          s"with largest timestamp $largestTimestamp at shallow offset $shallowOffsetOfMaxTimestamp")
    //物理位置
    val physicalPosition = log.sizeInBytes()
    if (physicalPosition == 0)
      rollingBasedTimestamp = Some(largestTimestamp)
    ensureOffsetInRange(largestOffset)
    // 追加消息
    val appendedBytes = log.append(records)
    trace(s"Appended $appendedBytes to ${log.file} at end offset $largestOffset")
    if (largestTimestamp > maxTimestampSoFar) { //修改最大的时间戳以及对应的位移
      maxTimestampSoFar = largestTimestamp
      offsetOfMaxTimestampSoFar = shallowOffsetOfMaxTimestamp
    }
    //稀疏索引
    if (bytesSinceLastIndexEntry > indexIntervalBytes) { //创建索引
      offsetIndex.append(largestOffset, physicalPosition)  //追加位移索引
      timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar) //时间戳索引
      bytesSinceLastIndexEntry = 0
    }
    bytesSinceLastIndexEntry += records.sizeInBytes
  }
```

# 索引

## 追加索引

稀疏索引，每条索引占用8个字节，相对逻辑位移、物理位移

kafka.log.OffsetIndex#append

```scala
def append(offset: Long, position: Int): Unit = {
  inLock(lock) {
    //判断文件未写满
    require(!isFull, "Attempt to append to a full index (size = " + _entries + ").")
    if (_entries == 0 || offset > _lastOffset) {
      trace(s"Adding index entry $offset => $position to ${file.getAbsolutePath}")
      mmap.putInt(relativeOffset(offset)) // 写入相对位移值
      mmap.putInt(position) //写入物理位置信息
      _entries += 1 //更新索引个数
      _lastOffset = offset //更新当前索引文件最大位移值
      // 确保写入索引项格式符合要求
      require(_entries * entrySize == mmap.position(), entries + " entries but file position in index is " + mmap.position() + ".")
    } else {
      throw new InvalidOffsetException(s"Attempt to append an offset ($offset) to position $entries no larger than" +
        s" the last offset appended (${_lastOffset}) to ${file.getAbsolutePath}.")
    }
  }
}
```

```scala
def relativeOffset(offset: Long): Int = {// 写入相对位移值
  val relativeOffset = toRelative(offset)
  if (relativeOffset.isEmpty)
  // 如果无法转换成功（比如差值超过了整型表示范围)，则抛出异常
    throw new IndexOffsetOverflowException(s"Integer overflow for offset: $offset (${file.getAbsoluteFile})")
  relativeOffset.get
}
```

## 查找索引

```
当Kafka在查询索引的时候，原版的二分查找算法并没有考虑到缓存的问题，因此很可能会导致一些不必要的缺页中断（Page Fault）。此时，Kafka 线程会被阻塞，等待对应的索引项从物理磁盘中读出并放入到页缓存
```

kafka.log.OffsetIndex#lookup

```scala
def lookup(targetOffset: Long): OffsetPosition = {
  maybeLock(lock) {
    val idx = mmap.duplicate
    val slot = largestLowerBoundSlotFor(idx, targetOffset, IndexSearchType.KEY)
    if(slot == -1)
      OffsetPosition(baseOffset, 0)
    else
      parseEntry(idx, slot)
  }
}
```

```scala
protected def largestLowerBoundSlotFor(idx: ByteBuffer, target: Long, searchEntity: IndexSearchEntity): Int =
  indexSlotRangeFor(idx, target, searchEntity)._1
```

```scala
private def indexSlotRangeFor(idx: ByteBuffer, target: Long, searchEntity: IndexSearchEntity): (Int, Int) = {
  // 索引文件为空
  if(_entries == 0)
    return (-1, -1)

  def binarySearch(begin: Int, end: Int) : (Int, Int) = { //二分查找
    // binary search for the entry
    var lo = begin
    var hi = end
    while(lo < hi) {
      val mid = (lo + hi + 1) >>> 1
      val found = parseEntry(idx, mid)
      val compareResult = compareIndexEntry(found, target, searchEntity)
      if(compareResult > 0)
        hi = mid - 1
      else if(compareResult < 0)
        lo = mid
      else
        return (mid, mid)
    }
    (lo, if (lo == _entries - 1) -1 else lo + 1)
  }
  //默认_warmEntries=1024
  val firstHotEntry = Math.max(0, _entries - 1 - _warmEntries)
  //检测target offset是否在热区
  if(compareIndexEntry(parseEntry(idx, firstHotEntry), target, searchEntity) < 0) {//target > parseEntry(idx, firstHotEntry)
    return binarySearch(firstHotEntry, _entries - 1) //二分查找
  }
  //检测目标offset是否比索引文件中最小的offset还小
  if(compareIndexEntry(parseEntry(idx, 0), target, searchEntity) > 0)
    return (-1, 0)
  //冷区查找，可能触发缺页
  binarySearch(0, firstHotEntry)
}
```

# 消息读取

kafka.log.Log#read

```scala
def read(startOffset: Long,
         maxLength: Int,
         isolation: FetchIsolation,
         minOneMessage: Boolean): FetchDataInfo = {
  maybeHandleIOException(s"Exception while reading from $topicPartition in dir ${dir.getParent}") {
    trace(s"Reading $maxLength bytes from offset $startOffset of length $size bytes")
    val includeAbortedTxns = isolation == FetchTxnCommitted
    val endOffsetMetadata = nextOffsetMetadata
    val endOffset = nextOffsetMetadata.messageOffset
    if (startOffset == endOffset) //无消息可消费
      return emptyFetchDataInfo(endOffsetMetadata, includeAbortedTxns)
    // 找到startOffset值所在的日志段对象
    var segmentEntry = segments.floorEntry(startOffset)
    //要读取的消息位移超过了LEO值、没找到对应的日志段对象、要读取的消息在Log Start Offset之下（发生了合并），同样是对外不可见的消息
    if (startOffset > endOffset || segmentEntry == null || startOffset < logStartOffset)
      throw new OffsetOutOfRangeException(s"Received request for offset $startOffset for partition $topicPartition, " +
        s"but we only have log segments in the range $logStartOffset to $endOffset.")

    val maxOffsetMetadata = isolation match {
      case FetchLogEnd => nextOffsetMetadata
      case FetchHighWatermark => fetchHighWatermarkMetadata
      case FetchTxnCommitted => fetchLastStableOffsetMetadata
    }

    // 如果要读取的起始位置超过了能读取的最大位置，返回空的消息集合，因为没法读取任何消息
    if (startOffset > maxOffsetMetadata.messageOffset) {
      val startOffsetMetadata = convertToOffsetMetadataOrThrow(startOffset)
      return emptyFetchDataInfo(startOffsetMetadata, includeAbortedTxns)
    }
    while (segmentEntry != null) {
      val segment = segmentEntry.getValue
      val maxPosition = {
        if (maxOffsetMetadata.segmentBaseOffset == segment.baseOffset) {
          maxOffsetMetadata.relativePositionInSegment
        } else {
          segment.size //当前segment已经写入的字节数
        }
      }
      // 调用logsement的read方法执行真正的读取消息操作
      val fetchInfo = segment.read(startOffset, maxLength, maxPosition, minOneMessage)
      if (fetchInfo == null) {
        //如果没有返回任何消息，去下一个Logsegment读取消息
        segmentEntry = segments.higherEntry(segmentEntry.getKey)
      } else {
        return if (includeAbortedTxns)
          addAbortedTransactions(startOffset, segmentEntry, fetchInfo)
        else
          fetchInfo
      }
    }
    FetchDataInfo(nextOffsetMetadata, MemoryRecords.EMPTY)
  }
}
```

kafka.log.LogSegment#read

```scala
def read(startOffset: Long,
         maxSize: Int,
         maxPosition: Long = size,
         minOneMessage: Boolean = false): FetchDataInfo = {
  if (maxSize < 0)
    throw new IllegalArgumentException(s"Invalid max size $maxSize for log read from segment $log")
  //从索引文件根据改进的二分法查找第比startoffset小的最大索引，
  //然后从日志文件读取数据，获取大于等于该startOffset的物理位移
  //获取读取的位置、读取的字节数
  val startOffsetAndSize = translateOffset(startOffset)

  // if the start position is already off the end of the log, return null
  if (startOffsetAndSize == null)
    return null
  //开始读取的物理位移
  val startPosition = startOffsetAndSize.position
  val offsetMetadata = LogOffsetMetadata(startOffset, this.baseOffset, startPosition)

  //调整最大字节，避免了消息体过大，超过了maxSize，导致读不出完整的数据
  val adjustedMaxSize =
    if (minOneMessage) math.max(maxSize, startOffsetAndSize.size)
    else maxSize

  // return a log segment but with zero size in the case below
  if (adjustedMaxSize == 0)
    return FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY)

  //计算可拉取的字节数，
  val fetchSize: Int = min((maxPosition - startPosition).toInt, adjustedMaxSize)

  FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize),
    firstEntryIncomplete = adjustedMaxSize < startOffsetAndSize.size)
}
```

# 总结

1、对请求分类，避免了互相影响

2、顺序写，对于已写入的数据不会修改。新写入的数据都放在操作系统内存中，定时将内存数据刷新到磁盘。

3、大部分情况下都是尾部读，即读取最新写入的数据，而最新写入的数据一般都在内存中，避免了磁盘读取，降低了延迟

4、Zero-copy 零拷⻉，少了一次内存交换。 

5、批量处理,合并小的请求 

6、消费端使用拉模式进行消息的获取消费，与消费端处理能力相符
