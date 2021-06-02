# RocketMQ源码分析

* [生产者](#生产者)
  * [启动生产者](#启动生产者)
  * [初始化生产者](#初始化生产者)
  * [发送消息](#发送消息)
  * [发送事务消息](#发送事务消息)
  * [处理Broker的事务回查](#处理broker的事务回查)
* [消费者](#消费者)
  * [订阅主题](#订阅主题)
  * [初始化](#初始化)
  * [重平衡服务](#重平衡服务)
    * [启动重平衡服务](#启动重平衡服务)
    * [重平衡](#重平衡)
    * [分发拉取请求](#分发拉取请求)
    * [消费分区发生变化](#消费分区发生变化)
    * [执行拉取请求](#执行拉取请求)
    * [处理业务逻辑](#处理业务逻辑)
      * [并发消费](#并发消费)
      * [顺序消费](#顺序消费)
* [Broker](#broker)
  * [消息追加](#消息追加)
    * [刷盘](#刷盘)
      * [同步刷盘](#同步刷盘)
      * [异步刷盘](#异步刷盘)
        * [FlushRealTimeService](#flushrealtimeservice)
        * [CommitRealTimeService](#commitrealtimeservice)
    * [等待slave同步消息](#等待slave同步消息)
  * [索引建立](#索引建立)
    * [建立ConsumeQueue](#建立consumequeue)
    * [建立IndexFile](#建立indexfile)
  * [消息拉取](#消息拉取)
  * [拉取请求的挂起](#拉取请求的挂起)
    * [轮询检测](#轮询检测)
      * [存放挂起的拉取请求](#存放挂起的拉取请求)
      * [检测挂起的请求](#检测挂起的请求)
    * [写入及时唤醒](#写入及时唤醒)
  * [消息同步](#消息同步)
    * [HAClient](#haclient)
      * [处理接收的数据](#处理接收的数据)
      * [上报同步的进度](#上报同步的进度)
    * [AcceptSocketService](#acceptsocketservice)
    * [HAConnection](#haconnection)
      * [ReadSocketService](#readsocketservice)
      * [WriteSocketService](#writesocketservice)
    * [GroupTransferService](#grouptransferservice)
  * [延迟消息](#延迟消息)
    * [load](#load)
    * [start](#start)
  * [事务消息](#事务消息)
    * [处理EndTransactionRequest](#处理endtransactionrequest)
    * [定时检测half消息](#定时检测half消息)
  * [请求快速失败](#请求快速失败)
  * [禁用消费太慢的Group](#禁用消费太慢的group)
* [总结](#总结)


# 生产者

## 启动生产者

org.apache.rocketmq.client.producer.DefaultMQProducer#start

```java
public void start() throws MQClientException {
    //设置生产者组
    this.setProducerGroup(withNamespace(this.producerGroup));
    //启动生产者实现类
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

## 初始化生产者

org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#start(boolean)

```java
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST: //初始状态
            this.serviceState = ServiceState.START_FAILED;
            //参数校验
            this.checkConfig();
            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }
            //根据生产者组名获取或者初始化一个MQClientInstance
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
            //维护producerGroup -> DefaultMQProducerImpl
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
            if (startFactory) {
                mQClientFactory.start();
            }
            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }
    //发送心跳
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    //扫描过期的请求,针对request->reply类型的请求
    this.timer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            try {
                RequestFutureTable.scanExpiredRequest();
            } catch (Throwable e) {
                log.error("scan RequestFutureTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

org.apache.rocketmq.client.impl.factory.MQClientInstance#start

```java
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST: //初始状态
                this.serviceState = ServiceState.START_FAILED;
                // 获取nameserver地址
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                //网络通信初始化
                this.mQClientAPIImpl.start();
                //定时任务(定时发送心跳包括consumer注册)
                this.startScheduledTask();
                //开启拉取服务
                this.pullMessageService.start();
                // 开启重新平衡服务
                this.rebalanceService.start();
                //开启发送消息服务
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

## 发送消息

设置重试次数，默认2次

设置超时时间，默认3秒，

设置发送方式SYNC、ASYNC、ONEWAY

设置回调函数

设置发送失败时，是否可以像其他的Broker发送消息，默认false，retryAnotherBrokerWhenNotStoreOK

```java
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    this.makeSureStateOK();
    //验证数据的长度，不能为空，不能超过最大长度4M
    Validators.checkMessage(msg, this.defaultMQProducer);
    final long invokeID = random.nextLong();
    long beginTimestampFirst = System.currentTimeMillis();
    long beginTimestampPrev = beginTimestampFirst;
    long endTimestamp = beginTimestampFirst;
    //获取Topic的信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        boolean callTimeout = false;
        MessageQueue mq = null;
        Exception exception = null;
        SendResult sendResult = null;
        //发送总数
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            //不启用容错策略的话，随机选择broker
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                mq = mqSelected;
                brokersSent[times] = mq.getBrokerName(); //记录发送的Broker
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    if (times > 0) { //表示重试
                        //Reset topic with namespace during resend.
                        msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                    }
                    long costTime = beginTimestampPrev - beginTimestampFirst;
                    if (timeout < costTime) { //发送超时
                        callTimeout = true;
                        break;
                    }
                    //发送
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                    endTimestamp = System.currentTimeMillis();
                  //记录broker的延迟时间
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                //发送失败。尝试往其他broker发送；默认为false
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }
                            return sendResult;
                        default:
                            break;
                    }
                } catch (RemotingException e) {
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    exception = e;
                    continue;
                } catch (MQClientException e) {
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    exception = e;
                    continue;
                } catch (MQBrokerException e) {
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    exception = e;
                    switch (e.getResponseCode()) {
                        case ResponseCode.TOPIC_NOT_EXIST:
                        case ResponseCode.SERVICE_NOT_AVAILABLE:
                        case ResponseCode.SYSTEM_ERROR:
                        case ResponseCode.NO_PERMISSION:
                        case ResponseCode.NO_BUYER_ID:
                        case ResponseCode.NOT_IN_CURRENT_UNIT:
                            continue;
                        default:
                            if (sendResult != null) {
                                return sendResult;
                            }

                            throw e;
                    }
                } catch (InterruptedException e) {
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());

                    log.warn("sendKernelImpl exception", e);
                    log.warn(msg.toString());
                    throw e;
                }
            } else {
                break;
            }
        }

        if (sendResult != null) {
            return sendResult;
        }

        String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
            times,
            System.currentTimeMillis() - beginTimestampFirst,
            msg.getTopic(),
            Arrays.toString(brokersSent));

        info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

        MQClientException mqClientException = new MQClientException(info, exception);
        if (callTimeout) {
            throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
        }

        if (exception instanceof MQBrokerException) {
            mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
        } else if (exception instanceof RemotingConnectException) {
            mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
        } else if (exception instanceof RemotingTimeoutException) {
            mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
        } else if (exception instanceof MQClientException) {
            mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
        }

        throw mqClientException;
    }

    validateNameServerSetting();

    throw new MQClientException("No route info of this topic: " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
        null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
}
```

```java
private SendResult sendKernelImpl(final Message msg,
    final MessageQueue mq,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    //根据brokername获取broker地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        tryToFindTopicPublishInfo(mq.getTopic()); //从nameserver同步topic数据
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());//根据brokername获取broker地址
    }
    SendMessageContext context = null;
    if (brokerAddr != null) {
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
        byte[] prevBody = msg.getBody();
        try {
            //for MessageBatch,ID has been set in the generating process
            //设置唯一Id
            if (!(msg instanceof MessageBatch)) {
                MessageClientIDSetter.setUniqID(msg);
            }
            boolean topicWithNamespace = false;
            if (null != this.mQClientFactory.getClientConfig().getNamespace()) {
                msg.setInstanceId(this.mQClientFactory.getClientConfig().getNamespace());
                topicWithNamespace = true;
            }
            int sysFlag = 0;
            boolean msgBodyCompressed = false;
            //body超过4kb启用压缩，默认zip压缩
            if (this.tryToCompressMessage(msg)) {
                sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
                msgBodyCompressed = true;
            }
            //事务Half消息，对消费者不可见
            final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
            if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
            }
            if (hasCheckForbiddenHook()) {
                CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
                checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
                checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
                checkForbiddenContext.setCommunicationMode(communicationMode);
                checkForbiddenContext.setBrokerAddr(brokerAddr);
                checkForbiddenContext.setMessage(msg);
                checkForbiddenContext.setMq(mq);
                checkForbiddenContext.setUnitMode(this.isUnitMode());
                this.executeCheckForbiddenHook(checkForbiddenContext);
            }

            if (this.hasSendMessageHook()) {
                context = new SendMessageContext();
                context.setProducer(this);
                context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                context.setCommunicationMode(communicationMode);
                context.setBornHost(this.defaultMQProducer.getClientIP());
                context.setBrokerAddr(brokerAddr);
                context.setMessage(msg);
                context.setMq(mq);
                context.setNamespace(this.defaultMQProducer.getNamespace());
                String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                if (isTrans != null && isTrans.equals("true")) {
                    context.setMsgType(MessageType.Trans_Msg_Half);
                }

                if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
                    context.setMsgType(MessageType.Delay_Msg);
                }
                this.executeSendMessageHookBefore(context);
            }
            //设置发送消息请求头
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            requestHeader.setTopic(msg.getTopic());
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setSysFlag(sysFlag);
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            requestHeader.setFlag(msg.getFlag());
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            requestHeader.setBatch(msg instanceof MessageBatch);
            //topic以%RETRY%开头，
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                String reconsumeTimes = MessageAccessor.getReconsumeTime(msg); //重试次数
                if (reconsumeTimes != null) {
                    requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                }
                String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                if (maxReconsumeTimes != null) {
                    requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                }
            }
            SendResult sendResult = null;
            switch (communicationMode) {
                case ASYNC: //异步发送
                    Message tmpMessage = msg;
                    boolean messageCloned = false;
                    if (msgBodyCompressed) {
                        //If msg body was compressed, msgbody should be reset using prevBody.
                        //Clone new message using commpressed message body and recover origin massage.
                        //Fix bug:https://github.com/apache/rocketmq-externals/issues/66
                        tmpMessage = MessageAccessor.cloneMessage(msg);
                        messageCloned = true;
                        msg.setBody(prevBody);
                    }

                    if (topicWithNamespace) {
                        if (!messageCloned) {
                            tmpMessage = MessageAccessor.cloneMessage(msg);
                            messageCloned = true;
                        }
                        msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
                    }

                    long costTimeAsync = System.currentTimeMillis() - beginStartTime;
                    if (timeout < costTimeAsync) {
                        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                    }
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        tmpMessage,
                        requestHeader,
                        timeout - costTimeAsync,
                        communicationMode,
                        sendCallback,
                        topicPublishInfo,
                        this.mQClientFactory,
                        this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                        context,
                        this);
                    break;
                case ONEWAY: //不关心响应
                case SYNC: //阻塞，直到收到响应
                    //从开始发送到真正发送的耗时
                    long costTimeSync = System.currentTimeMillis() - beginStartTime;
                    if (timeout < costTimeSync) {
                        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                    }
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout - costTimeSync, //修改超时时间
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }

            if (this.hasSendMessageHook()) {
                context.setSendResult(sendResult);
                this.executeSendMessageHookAfter(context);
            }

            return sendResult;
        } catch (RemotingException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } catch (MQBrokerException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } catch (InterruptedException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } finally {
            msg.setBody(prevBody);
            msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
        }
    }
    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

## 发送事务消息

1、发送Half消息

2、发送成功，调用TransactionListener的executeLocalTransaction方法，执行本地事务，返回状态码

3、根据状态码，发送事务结束的请求，设置事务消息的最终状态（COMMIT_MESSAGE、ROLLBACK_MESSAGE、UNKNOW）

4、TransactionListener的checkLocalTransaction，供Broker端对Half消息进行回查

org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendMessageInTransaction

```java
public TransactionSendResult sendMessageInTransaction(final Message msg,
    final LocalTransactionExecuter localTransactionExecuter, final Object arg)
    throws MQClientException {
    //实现本地事务、回查本地事务
    TransactionListener transactionListener = getCheckListener();
    if (null == localTransactionExecuter && null == transactionListener) {
        throw new MQClientException("tranExecutor is null", null);
    }
    //事务类消息不能设置延迟时间
    if (msg.getDelayTimeLevel() != 0) {
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_DELAY_TIME_LEVEL);
    }
    Validators.checkMessage(msg, this.defaultMQProducer);
    SendResult sendResult = null;
    //设置事务类型
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
    try {
        //发送消息，和普通消息发送逻辑相同
        sendResult = this.send(msg);
    } catch (Exception e) {
        throw new MQClientException("send message Exception", e);
    }

    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
    Throwable localException = null;
    switch (sendResult.getSendStatus()) {
        case SEND_OK: { //发送成功
            try {
                if (sendResult.getTransactionId() != null) {
                    msg.putUserProperty("__transactionId__", sendResult.getTransactionId());
                }
                String transactionId = msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                if (null != transactionId && !"".equals(transactionId)) {
                    msg.setTransactionId(transactionId);
                }
                //执行本地事务，返回状态码
                if (null != localTransactionExecuter) {
                    localTransactionState = localTransactionExecuter.executeLocalTransactionBranch(msg, arg);
                } else if (transactionListener != null) {
                    log.debug("Used new transaction API");
                    localTransactionState = transactionListener.executeLocalTransaction(msg, arg);
                }
                if (null == localTransactionState) {
                    localTransactionState = LocalTransactionState.UNKNOW;
                }
                if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
                    log.info("executeLocalTransactionBranch return {}", localTransactionState);
                    log.info(msg.toString());
                }
            } catch (Throwable e) {
                log.info("executeLocalTransactionBranch exception", e);
                log.info(msg.toString());
                localException = e;
            }
        }
        break;
        case FLUSH_DISK_TIMEOUT:
        case FLUSH_SLAVE_TIMEOUT:
        case SLAVE_NOT_AVAILABLE:
            localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
            break;
        default:
            break;
    }
    //发送事务结束请求
    try {
        this.endTransaction(sendResult, localTransactionState, localException);
    } catch (Exception e) {
        log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
    }
    TransactionSendResult transactionSendResult = new TransactionSendResult();
    transactionSendResult.setSendStatus(sendResult.getSendStatus());
    transactionSendResult.setMessageQueue(sendResult.getMessageQueue());
    transactionSendResult.setMsgId(sendResult.getMsgId());
    transactionSendResult.setQueueOffset(sendResult.getQueueOffset());
    transactionSendResult.setTransactionId(sendResult.getTransactionId());
    transactionSendResult.setLocalTransactionState(localTransactionState);
    return transactionSendResult;
}
```

```java
public void endTransaction(
    final SendResult sendResult,
    final LocalTransactionState localTransactionState,
    final Throwable localException) throws RemotingException, MQBrokerException, InterruptedException, UnknownHostException {
    final MessageId id;
    if (sendResult.getOffsetMsgId() != null) {
        id = MessageDecoder.decodeMessageId(sendResult.getOffsetMsgId());
    } else {
        id = MessageDecoder.decodeMessageId(sendResult.getMsgId());
    }
    String transactionId = sendResult.getTransactionId();
    final String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(sendResult.getMessageQueue().getBrokerName());
    EndTransactionRequestHeader requestHeader = new EndTransactionRequestHeader();
    requestHeader.setTransactionId(transactionId);
    requestHeader.setCommitLogOffset(id.getOffset());
    //设置事务的最终状态
    switch (localTransactionState) {
        case COMMIT_MESSAGE://本地事务执行成功
            requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
            break;
        case ROLLBACK_MESSAGE: //本地事务执行失败
            requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
            break;
        case UNKNOW: //未知
            requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
            break;
        default:
            break;
    }
    requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
    requestHeader.setTranStateTableOffset(sendResult.getQueueOffset());
    requestHeader.setMsgId(sendResult.getMsgId());
    String remark = localException != null ? ("executeLocalTransactionBranch exception: " + localException.toString()) : null;
    //以oneway方式发送
    this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, requestHeader, remark,
        this.defaultMQProducer.getSendMsgTimeout());
}
```

## 处理Broker的事务回查

org.apache.rocketmq.client.impl.ClientRemotingProcessor#checkTransactionState

```java
public RemotingCommand checkTransactionState(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    final CheckTransactionStateRequestHeader requestHeader =
        (CheckTransactionStateRequestHeader) request.decodeCommandCustomHeader(CheckTransactionStateRequestHeader.class);
    final ByteBuffer byteBuffer = ByteBuffer.wrap(request.getBody());
    final MessageExt messageExt = MessageDecoder.decode(byteBuffer); //解码为MessageExt
    if (messageExt != null) {
        if (StringUtils.isNotEmpty(this.mqClientFactory.getClientConfig().getNamespace())) {
            messageExt.setTopic(NamespaceUtil
                .withoutNamespace(messageExt.getTopic(), this.mqClientFactory.getClientConfig().getNamespace()));
        }
        //获取事务Id
        String transactionId = messageExt.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
        if (null != transactionId && !"".equals(transactionId)) {
            messageExt.setTransactionId(transactionId);
        }
        final String group = messageExt.getProperty(MessageConst.PROPERTY_PRODUCER_GROUP);
        if (group != null) {
            //根据producerGroup获取对应的MQProducerInner
            MQProducerInner producer = this.mqClientFactory.selectProducer(group);
            if (producer != null) {
                //获取远端地址
                final String addr = RemotingHelper.parseChannelRemoteAddr(ctx.channel());
                //检测事务状态
                producer.checkTransactionState(addr, messageExt, requestHeader);
            } else {
                log.debug("checkTransactionState, pick producer by group[{}] failed", group);
            }
        } else {
            log.warn("checkTransactionState, pick producer group failed");
        }
    } else {
        log.warn("checkTransactionState, decode message failed");
    }

    return null;
}
```

org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#checkTransactionState

```java
public void checkTransactionState(final String addr, final MessageExt msg,
    final CheckTransactionStateRequestHeader header) {
    Runnable request = new Runnable() {
        private final String brokerAddr = addr;
        private final MessageExt message = msg;
        private final CheckTransactionStateRequestHeader checkRequestHeader = header;
        private final String group = DefaultMQProducerImpl.this.defaultMQProducer.getProducerGroup();

        @Override
        public void run() {
            //获取事务检查监听器
            TransactionCheckListener transactionCheckListener = DefaultMQProducerImpl.this.checkListener();
            TransactionListener transactionListener = getCheckListener();
            if (transactionCheckListener != null || transactionListener != null) {
                LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
                Throwable exception = null;
                try {
                  	//回查本地事务
                    if (transactionCheckListener != null) {
                        localTransactionState = transactionCheckListener.checkLocalTransactionState(message);
                    } else if (transactionListener != null) {
                        log.debug("Used new check API in transaction message");
                        localTransactionState = transactionListener.checkLocalTransaction(message);
                    } else {
                        log.warn("CheckTransactionState, pick transactionListener by group[{}] failed", group);
                    }
                } catch (Throwable e) {
                    log.error("Broker call checkTransactionState, but checkLocalTransactionState exception", e);
                    exception = e;
                }
								//处理事务的最终状态
                this.processTransactionState(
                    localTransactionState,
                    group,
                    exception);
            } else {
                log.warn("CheckTransactionState, pick transactionCheckListener by group[{}] failed", group);
            }
        }

        private void processTransactionState(
            final LocalTransactionState localTransactionState,
            final String producerGroup,
            final Throwable exception) {
            final EndTransactionRequestHeader thisHeader = new EndTransactionRequestHeader();
            thisHeader.setCommitLogOffset(checkRequestHeader.getCommitLogOffset());
            thisHeader.setProducerGroup(producerGroup);
            thisHeader.setTranStateTableOffset(checkRequestHeader.getTranStateTableOffset());
            thisHeader.setFromTransactionCheck(true);

            String uniqueKey = message.getProperties().get(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
            if (uniqueKey == null) {
                uniqueKey = message.getMsgId();
            }
            thisHeader.setMsgId(uniqueKey);
            thisHeader.setTransactionId(checkRequestHeader.getTransactionId());
            switch (localTransactionState) {
                case COMMIT_MESSAGE:
                    thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
                    break;
                case ROLLBACK_MESSAGE:
                    thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
                    log.warn("when broker check, client rollback this transaction, {}", thisHeader);
                    break;
                case UNKNOW:
                    thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
                    log.warn("when broker check, client does not know this transaction state, {}", thisHeader);
                    break;
                default:
                    break;
            }

            String remark = null;
            if (exception != null) {
                remark = "checkLocalTransactionState Exception: " + RemotingHelper.exceptionSimpleDesc(exception);
            }

            try {      //向Broker发送事务的最终状态
              DefaultMQProducerImpl.this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, thisHeader, remark,
                    3000);
            } catch (Exception e) {
                log.error("endTransactionOneway exception", e);
            }
        }
    };

    this.checkExecutor.submit(request);
}
```

# 消费者

## 订阅主题

org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#subscribe

```java
public void subscribe(final String topic, final MessageSelector messageSelector) throws MQClientException {
    try {
        if (messageSelector == null) {
            subscribe(topic, SubscriptionData.SUB_ALL);
            return;
        }
        //订阅信息：消费的主题、tag过滤
        SubscriptionData subscriptionData = FilterAPI.build(topic,
            messageSelector.getExpression(), messageSelector.getExpressionType());
        this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
        if (this.mQClientFactory != null) {
          	//发送心跳给Broker
            this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        }
    } catch (Exception e) {
        throw new MQClientException("subscription exception", e);
    }
}
```

## 初始化

org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start

```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;
            //检测配置信息是否合法
            this.checkConfig();
            this.copySubscription();
            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {//集群模式
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }
            //创建MQClientInstance，clientId -> MQClientInstance
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
            //设置重平衡需要的信息
            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);
            //封装拉取数据的操作
            this.pullAPIWrapper = new PullAPIWrapper(
                mQClientFactory,
                this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
            //获取上次消费到的offset信息；默认初始化时为空
            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                //集群消费：offset保存在Broker端，由RemoteBrokerOffsetStore管理
                //广播消费：offet保存在本地，由LocalFileOffsetStore管理
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            //加载offset(BROADCASTING:从本地文件加载;CLUSTERING:实现为空，从远端拉取offset)
            this.offsetStore.load();
            //顺序消费还是并发消费
            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {//顺序消费
                this.consumeOrderly = true;
                this.consumeMessageService =
                    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) { //并发消费
                this.consumeOrderly = false;
                this.consumeMessageService =
                    new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }
            //ConsumeMessageService
            this.consumeMessageService.start();
            //注册（groupName->PushConsumer)
            boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                this.consumeMessageService.shutdown();
                throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }
            mQClientFactory.start();
            log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }
    //从nameserver获取topic的详细信息
    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    this.mQClientFactory.checkClientInBroker();
    //发送心跳，注册消费者
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    //唤醒重平衡线程
    this.mQClientFactory.rebalanceImmediately();
```

## 重平衡服务

### 启动重平衡服务

org.apache.rocketmq.client.impl.consumer.RebalanceService#run

```java
public void run() {
    log.info(this.getServiceName() + " service started");
    while (!this.isStopped()) {
        //默认每隔20s执行一次mqClientFactory.doRebalance方法
        this.waitForRunning(waitInterval);
        //消费者之间的消费平衡；同一个group内的消费者，消费某个主题的某个分区
        this.mqClientFactory.doRebalance();
    }
    log.info(this.getServiceName() + " service end");
}
```

consumerTable维护消费组下的消费者实例，groupName -> MQConsumerInner

org.apache.rocketmq.client.impl.factory.MQClientInstance#doRebalance

```java
public void doRebalance() {
    //遍历注册的消费者,对消费者执行doRebalance()方法
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTableconsumerTable.entrySet()) {
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            try {
                impl.doRebalance();
            } catch (Throwable e) {
                log.error("doRebalance exception", e);
            }
        }
    }
}
```

### 重平衡

org.apache.rocketmq.client.impl.consumer.RebalanceImpl#doRebalance

```java
public void doRebalance(final boolean isOrder) {
  //获取此消费者的订阅信息
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
            final String topic = entry.getKey();
            try {
              	//以topic为单位为每个消费者配置消费分区
                this.rebalanceByTopic(topic, isOrder);
            } catch (Throwable e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("rebalanceByTopic Exception", e);
                }
            }
        }
    }
    this.truncateMessageQueueNotMyTopic();
}
```

org.apache.rocketmq.client.impl.consumer.RebalanceImpl#rebalanceByTopic

```java
private void rebalanceByTopic(final String topic, final boolean isOrder) {
    switch (messageModel) {
        case BROADCASTING: { //广播消费
            Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
            if (mqSet != null) {
                boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
                if (changed) {
                    this.messageQueueChanged(topic, mqSet, mqSet);
                    log.info("messageQueueChanged {} {} {} {}",
                        consumerGroup,
                        topic,
                        mqSet,
                        mqSet);
                }
            } else {
                log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
            }
            break;
        }
        case CLUSTERING: { //集群消费
            //根据topic获取分区信息
            Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
            //获取消费组下的消费者（初始化时会发送心跳请求，broker会对consumer进行注册）
            List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
            if (null == mqSet) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
                }
            }
            if (null == cidAll) { //group下的consumers为空
                log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
            }
            if (mqSet != null && cidAll != null) { //同一个group下的consumers不为空
                List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
                mqAll.addAll(mqSet);
                Collections.sort(mqAll); //分区排序
                Collections.sort(cidAll); //consumerId排序
                //MessageQueue分配策略
                AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;
                List<MessageQueue> allocateResult = null;
                try {
                    //给当前的消费者分配消费的queue
                    allocateResult = strategy.allocate(
                        this.consumerGroup,
                        this.mQClientFactory.getClientId(),
                        mqAll,
                        cidAll);
                } catch (Throwable e) {
                    log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                        e);
                    return;
                }
                //当前客户端被分配的MessageQueue
                Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
                if (allocateResult != null) {
                    allocateResultSet.addAll(allocateResult);
                }
                //根据本次分配的Messagequeue，判断是否发生变化
                boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
                if (changed) { //发生改变
                    log.info(
                        "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
                        strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
                        allocateResultSet.size(), allocateResultSet);
                    this.messageQueueChanged(topic, mqSet, allocateResultSet);
                }
            }
            break;
        }
        default:
            break;
    }
}
```

重平衡之后，修改processQueueTable

重平衡之后，某些分区不再被消费，需将其从processQueueTable移除，并提交已经消费的offset

如果是顺序消费，需要向Broker申请锁，申请存放，才能拉取消息

分发拉取请求

```java
private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet,
    final boolean isOrder) {
    boolean changed = false;
    //上次平衡时维护的数据
    Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<MessageQueue, ProcessQueue> next = it.next();
        MessageQueue mq = next.getKey();
        ProcessQueue pq = next.getValue();

        if (mq.getTopic().equals(topic)) {//之前有消费此topic的数据
            if (!mqSet.contains(mq)) {//平衡之后不再消费此分区
                pq.setDropped(true); //此messagequeue不再被消费
                if (this.removeUnnecessaryMessageQueue(mq, pq)) { //提交移除messageueue的offset
                    it.remove();//从processQueueTable删除
                    changed = true;//标志发生变化
                    log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
                }
            } else if (pq.isPullExpired()) { //此请求超时，距离上次拉取超过120000mills
                switch (this.consumeType()) {
                    case CONSUME_ACTIVELY: //pull
                        break;
                    case CONSUME_PASSIVELY: //push
                        pq.setDropped(true);
                        if (this.removeUnnecessaryMessageQueue(mq, pq)) { //移除不是必需的Messagequeue
                            it.remove();
                            changed = true;//标志发生变化
                            log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
                                consumerGroup, mq);
                        }
                        break;
                    default:
                        break;
                }
            }
        }
    }

    //组装拉取请求
    List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
    for (MessageQueue mq : mqSet) {
        if (!this.processQueueTable.containsKey(mq)) { //之前未曾消费过此Messagequeue
            if (isOrder && !this.lock(mq)) { //顺序消费，需要发送LOCK_BATCH_MQ请求
                log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
                continue;
            }
            //移除之前的offset
            this.removeDirtyOffset(mq);
            ProcessQueue pq = new ProcessQueue();
            //计算拉取位置，默认CONSUME_FROM_LAST_OFFSET
            long nextOffset = this.computePullFromWhere(mq);
            if (nextOffset >= 0) {
                ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
                if (pre != null) { //messagequeue已经存在
                    log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
                } else {
                    log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
                    PullRequest pullRequest = new PullRequest(); //创建拉取请求
                    pullRequest.setConsumerGroup(consumerGroup); //消费组
                    pullRequest.setNextOffset(nextOffset); //消费位置
                    pullRequest.setMessageQueue(mq); //消费分区
                    pullRequest.setProcessQueue(pq); // 限流
                    pullRequestList.add(pullRequest);
                    changed = true; //标志发生改变
                }
            } else {
                log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
            }
        }
    }
    //分发拉取请求，交由PullMessageService处理拉取请求
    this.dispatchPullRequest(pullRequestList);
    return changed;
}
```

### 分发拉取请求

org.apache.rocketmq.client.impl.consumer.RebalancePushImpl#dispatchPullRequest

```java
public void dispatchPullRequest(List<PullRequest> pullRequestList) {
    for (PullRequest pullRequest : pullRequestList) {
        this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest);
        log.info("doRebalance, {}, add a new pull request {}", consumerGroup, pullRequest);
    }
}
```

### 消费分区发生变化

重计算流控：每个分区应该拉取消息的阈值，比如100条；每个分区缓存的消息大小，比如100MB

```java
public void messageQueueChanged(String topic, Set<MessageQueue> mqAll, Set<MessageQueue> mqDivided) {
    SubscriptionData subscriptionData = this.subscriptionInner.get(topic);
    long newVersion = System.currentTimeMillis();
    log.info("{} Rebalance changed, also update version: {}, {}", topic, subscriptionData.getSubVersion(), newVersion);
    subscriptionData.setSubVersion(newVersion);//设置新版本
    int currentQueueCount = this.processQueueTable.size(); //消费的分区数量
    //流控
    if (currentQueueCount != 0) {
        //主题级别的流控，默认无限制；如果配置1000，当前消费被分配10个分区，pullThresholdForQueue就是100
        int pullThresholdForTopic = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getPullThresholdForTopic();
        if (pullThresholdForTopic != -1) { // 默认-1:无限制
            int newVal = Math.max(1, pullThresholdForTopic / currentQueueCount);
            log.info("The pullThresholdForQueue is changed from {} to {}",
                this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getPullThresholdForQueue(), newVal);
            this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().setPullThresholdForQueue(newVal);
        }
        //控制缓存的消息大小，默认无限制；如果配置1000MB，当前消费被分配10个分区，pullThresholdForQueue就是100MB
        int pullThresholdSizeForTopic = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getPullThresholdSizeForTopic();
        if (pullThresholdSizeForTopic != -1) {
            int newVal = Math.max(1, pullThresholdSizeForTopic / currentQueueCount);
            log.info("The pullThresholdSizeForQueue is changed from {} to {}",
                this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getPullThresholdSizeForQueue(), newVal);
            this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().setPullThresholdSizeForQueue(newVal);
        }
    }
    //发送心跳给所有的Broker
    this.getmQClientFactory().sendHeartbeatToAllBrokerWithLock();
}
```

### 执行拉取请求

在初始化阶段，启动此线程。由于是单线程拉取，当某个拉取请求耗时较长时，就会影响其他的拉取请求。可以采用线程隔离的技术消除有问题的拉取请求带来的延迟问题。执行完拉取请求后，计算拉取耗时，如果超过一定阈值，将此请求放到隔离线程池中处理。

```java
//存放拉取请求
private final LinkedBlockingQueue<PullRequest> pullRequestQueue = new LinkedBlockingQueue<PullRequest>();
//线程池；处理延迟调度的拉取请求
private final ScheduledExecutorService scheduledExecutorService = Executors
    .newSingleThreadScheduledExecutor(new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "PullMessageServiceScheduledThread");
        }
    });
```

org.apache.rocketmq.client.impl.consumer.PullMessageService#run

```java
public void run() {
    log.info(this.getServiceName() + " service started");
    while (!this.isStopped()) {
        try {
            //pullRequestQueue存放拉取请求
            PullRequest pullRequest = this.pullRequestQueue.take();
            //发送拉取消息的请求
            this.pullMessage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }
    log.info(this.getServiceName() + " service end");
}
```

```java
private void pullMessage(final PullRequest pullRequest) {
    //根据请求的group获取DefaultMQPushConsumerImpl，发送拉取请求
    final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        impl.pullMessage(pullRequest);
    } else {
        log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
    }
}
```

org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage

```java
public void pullMessage(final PullRequest pullRequest) {
    final ProcessQueue processQueue = pullRequest.getProcessQueue();
    if (processQueue.isDropped()) {
        log.info("the pull request[{}] is dropped.", pullRequest.toString());
        return;
    }
    //设置拉取时间
    pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());
    try {
        this.makeSureStateOK();
    } catch (MQClientException e) {
        log.warn("pullMessage exception, consumer state not ok", e);
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        return;
    }
    //如果处理队列被挂起,延迟1s后再执行
    if (this.isPause()) {
        log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
        return;
    }
    //缓存的数据数量（分区级别）
    long cachedMessageCount = processQueue.getMsgCount().get();
    //缓存的数据大小（分区级别）
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);
    //如果消费端累积的数据数量超过1000，延迟50mills执行
    if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }
    //如果消费端累积的数据大小超过100，延迟50mills执行
    if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }
    if (!this.consumeOrderly) {//不是顺序消费
        if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {//跨度超过2000
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
            if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
                    processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
                    pullRequest, queueMaxSpanFlowControlTimes);
            }
            return;
        }
    } else {
        if (processQueue.isLocked()) {
            if (!pullRequest.isLockedFirst()) {
                final long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
                boolean brokerBusy = offset < pullRequest.getNextOffset();
                log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
                    pullRequest, offset, brokerBusy);
                if (brokerBusy) {
                    log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
                        pullRequest, offset);
                }

                pullRequest.setLockedFirst(true);
                pullRequest.setNextOffset(offset);
            }
        } else {
            this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
            log.info("pull message later because not locked in broker, {}", pullRequest);
            return;
        }
    }
    //根据topic获取SubscriptionData
    final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (null == subscriptionData) {
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        log.warn("find the consumer's subscription failed, {}", pullRequest);
        return;
    }
    //开始拉取时间
    final long beginTimestamp = System.currentTimeMillis();
    //拉取的回调函数
    PullCallback pullCallback = new PullCallback() {
        @Override
        public void onSuccess(PullResult pullResult) { //拉取数据成功
            if (pullResult != null) {
                //对拉取的数据过滤
                pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                    subscriptionData);
                switch (pullResult.getPullStatus()) {
                    case FOUND:
                        long prevRequestOffset = pullRequest.getNextOffset(); //之前请求的offset
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset()); //下次请求的offset
                        //记录拉取耗时
                        long pullRT = System.currentTimeMillis() - beginTimestamp;
                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullRT);
                        long firstMsgOffset = Long.MAX_VALUE;
                        if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) { //拉取数据为空，加入到pullRequestQueue队列中
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        } else {
                            firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();
                            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());
                            //存到Treemap，是否分发给consume；用于顺序消费
                            boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                            //线程池处理拉取的数据
                            DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                pullResult.getMsgFoundList(),
                                processQueue,
                                pullRequest.getMessageQueue(),
                                dispatchToConsume);

                            //默认拉取间隔为0，立即拉取，否则等待一定间隔，在进行拉取
                            if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                    DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                            } else {
                                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            }
                        }
                        //消息异常
                        if (pullResult.getNextBeginOffset() < prevRequestOffset
                            || firstMsgOffset < prevRequestOffset) {
                            log.warn(
                                "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                                pullResult.getNextBeginOffset(),
                                firstMsgOffset,
                                prevRequestOffset);
                        }

                        break;
                    case NO_NEW_MSG:
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                        DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        break;
                    case NO_MATCHED_MSG:
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                        DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        break;
                    case OFFSET_ILLEGAL:
                        log.warn("the pull request offset illegal, {} {}",
                            pullRequest.toString(), pullResult.toString());
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                        pullRequest.getProcessQueue().setDropped(true);
                        DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

                            @Override
                            public void run() {
                                try {
                                    DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                        pullRequest.getNextOffset(), false);

                                    DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());

                                    DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());

                                    log.warn("fix the pull request offset, {}", pullRequest);
                                } catch (Throwable e) {
                                    log.error("executeTaskLater Exception", e);
                                }
                            }
                        }, 10000);
                        break;
                    default:
                        break;
                }
            }
        }

        @Override
        public void onException(Throwable e) { //拉取出现异常，50mills之后再次拉取
            if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                log.warn("execute the pull request exception", e);
            }

            DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        }
    };

    boolean commitOffsetEnable = false; //是否要提交offset
    long commitOffsetValue = 0L; //提交的offset
    if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) { //集群消费
        commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
        if (commitOffsetValue > 0) { //大于0，提交offset
            commitOffsetEnable = true;
        }
    }

    String subExpression = null;
    boolean classFilter = false;
    SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (sd != null) {
        if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
            subExpression = sd.getSubString();
        }

        classFilter = sd.isClassFilterMode();
    }

    int sysFlag = PullSysFlag.buildSysFlag(
        commitOffsetEnable, // commitOffset
        true, // suspend
        subExpression != null, // subscription
        classFilter // class filter
    );

    try {
        //获取broker地址，组装拉取请求，发送远端请求
        this.pullAPIWrapper.pullKernelImpl(
            pullRequest.getMessageQueue(),
            subExpression,
            subscriptionData.getExpressionType(),
            subscriptionData.getSubVersion(),
            pullRequest.getNextOffset(),
            this.defaultMQPushConsumer.getPullBatchSize(),
            sysFlag,
            commitOffsetValue,
            BROKER_SUSPEND_MAX_TIME_MILLIS,
            CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
            CommunicationMode.ASYNC,
            pullCallback
        );
    } catch (Exception e) {
        log.error("pullKernelImpl exception", e);
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    }
}
```

### 处理业务逻辑

#### 并发消费

org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService#submitConsumeRequest

```java
public void submitConsumeRequest(
    final List<MessageExt> msgs,
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final boolean dispatchToConsume) {
    //将拉取的消息封装成ConsumeRequest，提交到线程池
    //分批处理
    //如果提交时抛出RejectedExecutionException，则会调用submitConsumeRequestLater延迟5s执行
    final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
    if (msgs.size() <= consumeBatchSize) {
        ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
        try {
            this.consumeExecutor.submit(consumeRequest);
        } catch (RejectedExecutionException e) {
            this.submitConsumeRequestLater(consumeRequest);
        }
    } else {
        for (int total = 0; total < msgs.size(); ) {
            List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
            for (int i = 0; i < consumeBatchSize; i++, total++) {
                if (total < msgs.size()) {
                    msgThis.add(msgs.get(total));
                } else {
                    break;
                }
            }
            ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
            try {
                this.consumeExecutor.submit(consumeRequest);
            } catch (RejectedExecutionException e) {
                for (; total < msgs.size(); total++) {
                    msgThis.add(msgs.get(total));
                }
                this.submitConsumeRequestLater(consumeRequest);
            }
        }
    }
}
```

org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService.ConsumeRequest#run

```java
public void run() {
    if (this.processQueue.isDropped()) {
        log.info("the message queue not be able to consume, because it's dropped. group={} {}", ConsumeMessageConcurrentlyService.this.consumerGroup, this.messageQueue);
        return;
    }

    MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
    ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
    ConsumeConcurrentlyStatus status = null;
    defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());

    ConsumeMessageContext consumeMessageContext = null;
    if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext = new ConsumeMessageContext();
        consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
        consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());
        consumeMessageContext.setProps(new HashMap<String, String>());
        consumeMessageContext.setMq(messageQueue);
        consumeMessageContext.setMsgList(msgs);
        consumeMessageContext.setSuccess(false);
        ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
    }

    long beginTimestamp = System.currentTimeMillis();
    boolean hasException = false;
    ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
    try {
        if (msgs != null && !msgs.isEmpty()) {
            for (MessageExt msg : msgs) {
                MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
            }
        }
        //处理业务逻辑
        status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
    } catch (Throwable e) {
        log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
            RemotingHelper.exceptionSimpleDesc(e),
            ConsumeMessageConcurrentlyService.this.consumerGroup,
            msgs,
            messageQueue);
        hasException = true;
    }
    //记录处理消息的时间
    long consumeRT = System.currentTimeMillis() - beginTimestamp;
    //设置返回类型
    if (null == status) { //处理出现异常
        if (hasException) {
            returnType = ConsumeReturnType.EXCEPTION;
        } else {
            returnType = ConsumeReturnType.RETURNNULL;
        }
    } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) { //15min
        returnType = ConsumeReturnType.TIME_OUT;
    } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
        returnType = ConsumeReturnType.FAILED;
    } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
        returnType = ConsumeReturnType.SUCCESS;
    }

    if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
    }
    //设置处理后的状态
    if (null == status) {
        log.warn("consumeMessage return null, Group: {} Msgs: {} MQ: {}",
            ConsumeMessageConcurrentlyService.this.consumerGroup,
            msgs,
            messageQueue);
        status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }

    if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext.setStatus(status.toString());
        consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);
        ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
    }
    //记录处理耗时
    ConsumeMessageConcurrentlyService.this.getConsumerStatsManager()
        .incConsumeRT(ConsumeMessageConcurrentlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

    if (!processQueue.isDropped()) { //处理消费的结果
        ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
    } else {
        log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
    }
}
```

```java
public void processConsumeResult(
    final ConsumeConcurrentlyStatus status,
    final ConsumeConcurrentlyContext context,
    final ConsumeRequest consumeRequest
) {
    int ackIndex = context.getAckIndex();
    if (consumeRequest.getMsgs().isEmpty())
        return;
    //记录统计数据
    switch (status) {
        case CONSUME_SUCCESS: //消费成功
            if (ackIndex >= consumeRequest.getMsgs().size()) {
                ackIndex = consumeRequest.getMsgs().size() - 1;
            }
            int ok = ackIndex + 1;
            int failed = consumeRequest.getMsgs().size() - ok;
            this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), ok);
            this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), failed);
            break;
        case RECONSUME_LATER: //处理失败
            ackIndex = -1;
            this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(),
                consumeRequest.getMsgs().size());
            break;
        default:
            break;
    }
    switch (this.defaultMQPushConsumer.getMessageModel()) {
        case BROADCASTING:
            for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                MessageExt msg = consumeRequest.getMsgs().get(i);
                log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
            }
            break;
        case CLUSTERING:
            List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
            for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                MessageExt msg = consumeRequest.getMsgs().get(i);
                //发送给broker
                boolean result = this.sendMessageBack(msg, context);
                if (!result) { //发送失败
                    msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                    msgBackFailed.add(msg);
                }
            }
            if (!msgBackFailed.isEmpty()) {
                consumeRequest.getMsgs().removeAll(msgBackFailed); //移除发送失败的消息
                //对发送失败的消息之后再执行
                this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
            }
            break;
        default:
            break;
    }
    //从TreeMap中移除成功的消息
    long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
    if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
        //提交offset
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
    }
}
```

#### 顺序消费

org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService#submitConsumeRequest

```java
public void submitConsumeRequest(
    final List<MessageExt> msgs,
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final boolean dispathToConsume) {
    if (dispathToConsume) {
        ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
        this.consumeExecutor.submit(consumeRequest);
    }
}
```

org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService.ConsumeRequest#run

```java
public void run() {
    if (this.processQueue.isDropped()) {
        log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
        return;
    }
    //获取消费队列的锁
    final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
    synchronized (objLock) {
        //在重平衡时，在broker端已经申请到锁，并且锁尚未过期
        if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
            || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
            final long beginTime = System.currentTimeMillis();
            for (boolean continueConsume = true; continueConsume; ) {
                if (this.processQueue.isDropped()) { //此消费队列已经删除，不能再消费
                    log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                    break;
                }
                if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    && !this.processQueue.isLocked()) {
                    log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
                    //稍后向broker申请锁
                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                    break;
                }

                if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    && this.processQueue.isLockExpired()) { //锁过期，稍后消费
                    log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                    break;
                }

                long interval = System.currentTimeMillis() - beginTime;
                if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) { //超过最大持续消费的时间，稍后消费
                    ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
                    break;
                }

                final int consumeBatchSize =
                    ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize(); //批量消费的大小，默认1条数据
                List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);
                defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());
                if (!msgs.isEmpty()) {
                    final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);
                    ConsumeOrderlyStatus status = null;
                    ConsumeMessageContext consumeMessageContext = null;
                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext = new ConsumeMessageContext();
                        consumeMessageContext
                            .setConsumerGroup(ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumerGroup());
                        consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
                        consumeMessageContext.setMq(messageQueue);
                        consumeMessageContext.setMsgList(msgs);
                        consumeMessageContext.setSuccess(false);
                        // init the consume context type
                        consumeMessageContext.setProps(new HashMap<String, String>());
                        ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
                    }
                    long beginTimestamp = System.currentTimeMillis();
                    ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
                    boolean hasException = false;
                    try {
                        this.processQueue.getLockConsume().lock();
                        if (this.processQueue.isDropped()) {
                            log.warn("consumeMessage, the message queue not be able to consume, because it's dropped. {}",
                                this.messageQueue);
                            break;
                        }
                        status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
                    } catch (Throwable e) {
                        log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
                            RemotingHelper.exceptionSimpleDesc(e),
                            ConsumeMessageOrderlyService.this.consumerGroup,
                            msgs,
                            messageQueue);
                        hasException = true;
                    } finally {
                        this.processQueue.getLockConsume().unlock(); //释放消费的本地锁
                    }

                    if (null == status
                        || ConsumeOrderlyStatus.ROLLBACK == status
                        || ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                        log.warn("consumeMessage Orderly return not OK, Group: {} Msgs: {} MQ: {}",
                            ConsumeMessageOrderlyService.this.consumerGroup,
                            msgs,
                            messageQueue);
                    }

                    long consumeRT = System.currentTimeMillis() - beginTimestamp;
                    if (null == status) {
                        if (hasException) {
                            returnType = ConsumeReturnType.EXCEPTION;
                        } else {
                            returnType = ConsumeReturnType.RETURNNULL;
                        }
                    } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
                        returnType = ConsumeReturnType.TIME_OUT;
                    } else if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                        returnType = ConsumeReturnType.FAILED;
                    } else if (ConsumeOrderlyStatus.SUCCESS == status) {
                        returnType = ConsumeReturnType.SUCCESS;
                    }

                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
                    }

                    if (null == status) {
                        status = ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                    }

                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext.setStatus(status.toString());
                        consumeMessageContext
                            .setSuccess(ConsumeOrderlyStatus.SUCCESS == status || ConsumeOrderlyStatus.COMMIT == status);
                        ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
                    }

                    ConsumeMessageOrderlyService.this.getConsumerStatsManager()
                        .incConsumeRT(ConsumeMessageOrderlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

                    continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
                } else {
                    continueConsume = false;
                }
            }
        } else {
            if (this.processQueue.isDropped()) {
                log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                return;
            }
            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
        }
    }
}
```

# Broker

## 消息追加

org.apache.rocketmq.store.CommitLog#putMessage

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // Set the storage time
    msg.setStoreTimestamp(System.currentTimeMillis());
    // Set the message body BODY CRC (consider the most appropriate setting
    // on the client)
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
    // Back to Results
    AppendMessageResult result = null;
    StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();
    String topic = msg.getTopic();
    int queueId = msg.getQueueId();
    final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
        || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        // Delay Delivery
        if (msg.getDelayTimeLevel() > 0) { //判断是否是定时消息
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) { //如果大于最大level，就设置为最大level
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }
           //定时消息所属的Topic
            topic = ScheduleMessageService.SCHEDULE_TOPIC;
            //定时消息所属的Queue
            queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());
            //将原消息的Topic、queueId备份到消息的Properties中
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));
            //修改Topic、QueueId为定时消息所属的Topic、QueueId
            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

    InetSocketAddress bornSocketAddress = (InetSocketAddress) msg.getBornHost();
    if (bornSocketAddress.getAddress() instanceof Inet6Address) {
        msg.setBornHostV6Flag();
    }
    InetSocketAddress storeSocketAddress = (InetSocketAddress) msg.getStoreHost();
    if (storeSocketAddress.getAddress() instanceof Inet6Address) {
        msg.setStoreHostAddressV6Flag();
    }
    //锁住的时间
    long elapsedTimeInLock = 0;
    MappedFile unlockMappedFile = null;
    //获取最新的MappedFile
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
    putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
    try {
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        //记录锁的时间
        this.beginTimeInLock = beginLockTimestamp;
        //设置存储时间
        msg.setStoreTimestamp(beginLockTimestamp);
        //获取最新的MappedFile
        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
        }
        if (null == mappedFile) {
            log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
        }
        //追加数据
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            case PUT_OK: //成功
                break;
            case END_OF_FILE:
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                }
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
            case UNKNOWN_ERROR:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            default:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        }

        elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        putMessageLock.unlock();
    }
    if (elapsedTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
    }
		//unlockMappedFile不为空，说明新建了MappedFile，如果启用了文件预热，将之前的文件解锁，即释放锁住的内存区域
    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }
    //写消息的响应
    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);
    // Statistics
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());
    //刷新磁盘
    handleDiskFlush(result, putMessageResult, msg);
    //slave同步
    handleHA(result, putMessageResult, msg);
    return putMessageResult;
}
```

### 刷盘

org.apache.rocketmq.store.CommitLog#handleDiskFlush

```java
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    // 同步刷盘
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
      	//GroupCommitService
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        if (messageExt.isWaitStoreMsgOK()) {
            //下一条消息的offset
            GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes()); 
            service.putRequest(request);
            CompletableFuture<PutMessageStatus> flushOkFuture = request.future();
            PutMessageStatus flushStatus = null;
            try {
                //等待刷盘完成
                flushStatus = flushOkFuture.get(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout(),
                        TimeUnit.MILLISECONDS);
            } catch (InterruptedException | ExecutionException | TimeoutException e) {
                //flushOK=false;
            }
            if (flushStatus != PutMessageStatus.PUT_OK) {
                log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                    + " client address: " + messageExt.getBornHostString());
                putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
            }
        } else {
            service.wakeup();
        }
    }
    // 异步刷磁盘
    else {
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {//FlushRealTimeService
            flushCommitLogService.wakeup();	//唤醒阻塞的刷盘线程
        } else {//CommitRealTimeService
            commitLogService.wakeup();//唤醒阻塞的刷盘线程
        }
    }
}
```

####  同步刷盘

org.apache.rocketmq.store.CommitLog.GroupCommitService#doCommit

```java
private void doCommit() {
    synchronized (this.requestsRead) {
        if (!this.requestsRead.isEmpty()) {
            for (GroupCommitRequest req : this.requestsRead) {
                // There may be a message in the next file, so a maximum of
                // two times the flush
                boolean flushOK = false;
                for (int i = 0; i < 2 && !flushOK; i++) {
                    //判断刷新点是否大于当前请求的下一个offset，如果大于等于表示此请求的刷新已完成；小于说明有新数据写数据写入们可以进行刷新
                    flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();
                    if (!flushOK) {
                        CommitLog.this.mappedFileQueue.flush(0); //磁盘刷新
                    }
                }
                //唤醒等待刷新磁盘的请求
                req.wakeupCustomer(flushOK);
            }
            long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
            if (storeTimestamp > 0) {
                CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
            }
            this.requestsRead.clear();
        } else {
            // Because of individual messages is set to not sync flush, it
            // will come to this process
            CommitLog.this.mappedFileQueue.flush(0);
        }
    }
}
```

#### 异步刷盘

##### FlushRealTimeService

org.apache.rocketmq.store.CommitLog.FlushRealTimeService#run

```java
public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");
    while (!this.isStopped()) {
        //默认false
        boolean flushCommitLogTimed = CommitLog.this.defaultMessageStore.getMessageStoreConfig().isFlushCommitLogTimed();
        //刷盘数据到磁盘的间隔默认500
        int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushIntervalCommitLog();
        //默认4
        int flushPhysicQueueLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogLeastPages();
        //默认10秒
        int flushPhysicQueueThoroughInterval =
            CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogThoroughInterval();
        boolean printFlushProgress = false;
        // Print flush progress
        long currentTimeMillis = System.currentTimeMillis();
        if (currentTimeMillis >= (this.lastFlushTimestamp + flushPhysicQueueThoroughInterval)) {
            this.lastFlushTimestamp = currentTimeMillis;
            flushPhysicQueueLeastPages = 0;
            printFlushProgress = (printTimes++ % 10) == 0; //是否输出刷新进度的标志
        }
        try {
            if (flushCommitLogTimed) {
                Thread.sleep(interval);
            } else {
                this.waitForRunning(interval);
            }
            if (printFlushProgress) {
                this.printFlushProgress(); //输出刷新磁盘的进度
            }
            long begin = System.currentTimeMillis();
            //刷新数据到磁盘；默认刷盘4个页，刷盘间隔超过10s，将写入的数据全部刷盘
            CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages); 
            long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
            if (storeTimestamp > 0) {        CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
            }
            long past = System.currentTimeMillis() - begin;
            if (past > 500) {
                log.info("Flush data to disk costs {} ms", past);
            }
        } catch (Throwable e) {
            CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
            this.printFlushProgress();
        }
    }
    // Normal shutdown, to ensure that all the flush before exit
    boolean result = false;
    for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
        result = CommitLog.this.mappedFileQueue.flush(0);
        CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
    }
    this.printFlushProgress();
    CommitLog.log.info(this.getServiceName() + " service end");
}
```

##### CommitRealTimeService

org.apache.rocketmq.store.CommitLog.CommitRealTimeService#run

```java
public void run() {// 开启了TransientStorePool，先将DirectBuffer中的数据刷新到FileChannel，再异步刷盘
    CommitLog.log.info(this.getServiceName() + " service started");
    while (!this.isStopped()) {
        //默认200
        int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitIntervalCommitLog();
        //默认4
        int commitDataLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogLeastPages();
        //默认200
        int commitDataThoroughInterval =
            CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogThoroughInterval();
        long begin = System.currentTimeMillis();
        if (begin >= (this.lastCommitTimestamp + commitDataThoroughInterval)) {
            this.lastCommitTimestamp = begin;
            commitDataLeastPages = 0;
        }
        try {
            //将DirectBuffer中新增的的数据刷新到FileChannel
            boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
            long end = System.currentTimeMillis();
            if (!result) {
                this.lastCommitTimestamp = end; // result = false means some data committed.
                //唤醒FlushCommitLogService，刷新数据到磁盘
                flushCommitLogService.wakeup();
            }
            if (end - begin > 500) {
                log.info("Commit data to file costs {} ms", end - begin);
            }
            this.waitForRunning(interval);
        } catch (Throwable e) {
            CommitLog.log.error(this.getServiceName() + " service has exception. ", e);
        }
    }
    boolean result = false;
    for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
        result = CommitLog.this.mappedFileQueue.commit(0);
        CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
    }
    CommitLog.log.info(this.getServiceName() + " service end");
}
```

### 等待slave同步消息

如果slave落后master的消息不多，则等待slave同步完成，否则直接返回SLAVE_NOT_AVAILABLE。

org.apache.rocketmq.store.CommitLog#handleHA

```java
public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    //当前Broker是Mater
    if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
        HAService service = this.defaultMessageStore.getHaService();
        //等待slave同步完成
        if (messageExt.isWaitStoreMsgOK()) {
            //传入master已经写入的位置
            if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                //GroupTransferService负责检测slave是否已经完成消息同步
                service.putRequest(request);
                service.getWaitNotifyObject().wakeupAll();
                PutMessageStatus replicaStatus = null;
                try {
                    replicaStatus = request.future().get(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout(),
                            TimeUnit.MILLISECONDS);
                } catch (InterruptedException | ExecutionException | TimeoutException e) {
                }
                if (replicaStatus != PutMessageStatus.PUT_OK) {
                    log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                        + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                }
            }
            // 默认salve落后256M，就认为SLAVE_NOT_AVAILABLE
            else {
                // Tell the producer, slave not available
                putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
            }
        }
    }
```

org.apache.rocketmq.store.ha.HAService#isSlaveOK

```java
public boolean isSlaveOK(final long masterPutWhere) {//判断slave的状态
    boolean result = this.connectionCount.get() > 0; //说明有slave正在复制数据
    result =
        result
            && ((masterPutWhere - this.push2SlaveMaxOffset.get()) < this.defaultMessageStore
            .getMessageStoreConfig().getHaSlaveFallbehindMax()); //haSlaveFallbehindMax默认256M，slave落后master小于256M时，返回true，需要等待slave同步完成
    return result;
}
```

## 索引建立

**ReputMessageService负责索引的建立**。每个主题的每个分区对应一个索引文件。首先确定开始建立索引的位置，然后根据根据位置读取CommitLog消息建立索引

### 建立ConsumeQueue

org.apache.rocketmq.store.DefaultMessageStore.ReputMessageService#doReput

```java
private void doReput() {
    if (this.reputFromOffset < DefaultMessageStore.this.commitLog.getMinOffset()) {
        log.warn("The reputFromOffset={} is smaller than minPyOffset={}, this usually indicate that the dispatch behind too much and the commitlog has expired.",
            this.reputFromOffset, DefaultMessageStore.this.commitLog.getMinOffset());
        this.reputFromOffset = DefaultMessageStore.this.commitLog.getMinOffset();
    }
    for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {
        if (DefaultMessageStore.this.getMessageStoreConfig().isDuplicationEnable()
            && this.reputFromOffset >= DefaultMessageStore.this.getConfirmOffset()) {
            break;
        }
        //根据reputFromOffset找到对应的MappFile，然后读取数据
        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
        if (result != null) {
            try {
                this.reputFromOffset = result.getStartOffset();
                for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                    //读取数据封装成DispatchRequest
                    DispatchRequest dispatchRequest =
 DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                    int size = dispatchRequest.getBufferSize() == -1 ? dispatchRequest.getMsgSize() : dispatchRequest.getBufferSize();
                    if (dispatchRequest.isSuccess()) {
                        if (size > 0) {
                          //分发请求
                            DefaultMessageStore.this.doDispatch(dispatchRequest);
                            if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
                                    && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) {
                                //唤醒挂起的拉取请求
DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                        dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                        dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                        dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
                            }

                            this.reputFromOffset += size;
                            readSize += size;
                            if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).incrementAndGet();
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
                                    .addAndGet(dispatchRequest.getMsgSize());
                            }
                        } else if (size == 0) {
                            this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
                            readSize = result.getSize();
                        }
                    } else if (!dispatchRequest.isSuccess()) {

                        if (size > 0) {
                            log.error("[BUG]read total count not equals msg total size. reputFromOffset={}", reputFromOffset);
                            this.reputFromOffset += size;
                        } else {
                            doNext = false;
                            // If user open the dledger pattern or the broker is master node,
                            // it will not ignore the exception and fix the reputFromOffset variable
                            if (DefaultMessageStore.this.getMessageStoreConfig().isEnableDLegerCommitLog() ||
                                DefaultMessageStore.this.brokerConfig.getBrokerId() == MixAll.MASTER_ID) {
                                log.error("[BUG]dispatch message to consume queue error, COMMITLOG OFFSET: {}",
                                    this.reputFromOffset);
                                this.reputFromOffset += result.getSize() - readSize;
                            }
                        }
                    }
                }
            } finally {
                result.release();
            }
        } else {
            doNext = false;
        }
    }
}
```

```java
public void doDispatch(DispatchRequest req) {
    for (CommitLogDispatcher dispatcher : this.dispatcherList) {
        dispatcher.dispatch(req);
    }
}
```

org.apache.rocketmq.store.DefaultMessageStore#putMessagePositionInfo

```java
public void putMessagePositionInfo(DispatchRequest dispatchRequest) {
    //创建或者查找ConsumeQueue（topic->(queueId->ConsumeQueue))
    ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(), dispatchRequest.getQueueId());
    cq.putMessagePositionInfoWrapper(dispatchRequest);
}
```

org.apache.rocketmq.store.ConsumeQueue#putMessagePositionInfo

```java
private boolean putMessagePositionInfo(final long offset, final int size, final long tagsCode,
    final long cqOffset) {
    if (offset + size <= this.maxPhysicOffset) {
        log.warn("Maybe try to build consume queue repeatedly maxPhysicOffset={} phyOffset={}", maxPhysicOffset, offset);
        return true;
    }
    //每条索引消息20字节
    this.byteBufferIndex.flip();
    this.byteBufferIndex.limit(CQ_STORE_UNIT_SIZE); 
    this.byteBufferIndex.putLong(offset);//commitLog文件位置
    this.byteBufferIndex.putInt(size); //消息的字节大小
    this.byteBufferIndex.putLong(tagsCode); //tag的hashcode，方便服务端比较
    //cqOffset表示消息文件索引文件的第几条消息，根据CQ_STORE_UNIT_SIZE，可以方便得知索引文件中的位置，进而得知所处的commitlog位置
    final long expectLogicOffset = cqOffset * CQ_STORE_UNIT_SIZE;
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile(expectLogicOffset);
    if (mappedFile != null) {
        if (mappedFile.isFirstCreateInQueue() && cqOffset != 0 && mappedFile.getWrotePosition() == 0) {
            this.minLogicOffset = expectLogicOffset;
            this.mappedFileQueue.setFlushedWhere(expectLogicOffset);
            this.mappedFileQueue.setCommittedWhere(expectLogicOffset);
            this.fillPreBlank(mappedFile, expectLogicOffset);
            log.info("fill pre blank space " + mappedFile.getFileName() + " " + expectLogicOffset + " "
                + mappedFile.getWrotePosition());
        }

        if (cqOffset != 0) {
            long currentLogicOffset = mappedFile.getWrotePosition() + mappedFile.getFileFromOffset();
						//重复建立
            if (expectLogicOffset < currentLogicOffset) {
                log.warn("Build  consume queue repeatedly, expectLogicOffset: {} currentLogicOffset: {} Topic: {} QID: {} Diff: {}",
                    expectLogicOffset, currentLogicOffset, this.topic, this.queueId, expectLogicOffset - currentLogicOffset);
                return true;
            }

            if (expectLogicOffset != currentLogicOffset) {
                LOG_ERROR.warn(
                    "[BUG]logic queue order maybe wrong, expectLogicOffset: {} currentLogicOffset: {} Topic: {} QID: {} Diff: {}",
                    expectLogicOffset,
                    currentLogicOffset,
                    this.topic,
                    this.queueId,
                    expectLogicOffset - currentLogicOffset
                );
            }
        }
        //最大的物理位置
        this.maxPhysicOffset = offset + size;
        return mappedFile.appendMessage(this.byteBufferIndex.array());
    }
    return false;
}
```

###  建立IndexFile

org.apache.rocketmq.store.DefaultMessageStore.CommitLogDispatcherBuildIndex#dispatch

```java
class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {
    @Override
    public void dispatch(DispatchRequest request) {
        //默认true
        if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
            DefaultMessageStore.this.indexService.buildIndex(request);
        }
    }
}
```

```java
public void buildIndex(DispatchRequest req) {
  //尝试获取、创建IndexFile
    IndexFile indexFile = retryGetAndCreateIndexFile();
    if (indexFile != null) {
        long endPhyOffset = indexFile.getEndPhyOffset();
        DispatchRequest msg = req;
        String topic = msg.getTopic();
        String keys = msg.getKeys();
       //避免重复建立索引
        if (msg.getCommitLogOffset() < endPhyOffset) {
            return;
        }
        final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
        switch (tranType) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE:
            case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                break;
            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                return;
        }
      //对唯一key建立索引
        if (req.getUniqKey() != null) {
            indexFile = putKey(indexFile, msg, buildKey(topic, req.getUniqKey()));
            if (indexFile == null) {
                log.error("putKey error commitlog {} uniqkey {}", req.getCommitLogOffset(), req.getUniqKey());
                return;
            }
        }

        if (keys != null && keys.length() > 0) {
            String[] keyset = keys.split(MessageConst.KEY_SEPARATOR);
            for (int i = 0; i < keyset.length; i++) {
                String key = keyset[i];
                if (key.length() > 0) {
                    indexFile = putKey(indexFile, msg, buildKey(topic, key));
                    if (indexFile == null) {
                        log.error("putKey error commitlog {} uniqkey {}", req.getCommitLogOffset(), req.getUniqKey());
                        return;
                    }
                }
            }
        }
    } else {
        log.error("build index error, stop building index");
    }
}
```

Header：40字节 ，存储slot使用数量、索引数、phyOffset、storeTimestamp

hashSlotNum：默认500 0000，

indexNum:默认500 0000 * 4，

文件总大小：40 + 500 0000 * 4 + 500 0000 * 4*20

org.apache.rocketmq.store.index.IndexFile#putKey

```java
public boolean putKey(final String key, final long phyOffset, final long storeTimestamp) {
    if (this.indexHeader.getIndexCount() < this.indexNum) { // 500 0000 * 4
        int keyHash = indexKeyHashMethod(key); //计算hash
        int slotPos = keyHash % this.hashSlotNum; //500 0000 （计算处于哪个slot table）
        int absSlotPos = IndexHeader.INDEX_HEADER_SIZE  + slotPos * hashSlotSize; // 40 + slotPos * 4，此slot在IndexFile中的位置
        FileLock fileLock = null;
        try {
            // fileLock = this.fileChannel.lock(absSlotPos, hashSlotSize,
            // false);
          	//存储此slot的头
            int slotValue = this.mappedByteBuffer.getInt(absSlotPos);
            if (slotValue <= invalidIndex || slotValue > this.indexHeader.getIndexCount()) {
                slotValue = invalidIndex;
            }
            long timeDiff = storeTimestamp - this.indexHeader.getBeginTimestamp();
            timeDiff = timeDiff / 1000;
            if (this.indexHeader.getBeginTimestamp() <= 0) {
                timeDiff = 0;
            } else if (timeDiff > Integer.MAX_VALUE) {
                timeDiff = Integer.MAX_VALUE;
            } else if (timeDiff < 0) {
                timeDiff = 0;
            }
            int absIndexPos =
                IndexHeader.INDEX_HEADER_SIZE + this.hashSlotNum * hashSlotSize
                    + this.indexHeader.getIndexCount() * indexSize; //计算key存放的位置
            this.mappedByteBuffer.putInt(absIndexPos, keyHash); // key的hash
            this.mappedByteBuffer.putLong(absIndexPos + 4, phyOffset); //key 偏移量
            this.mappedByteBuffer.putInt(absIndexPos + 4 + 8, (int) timeDiff); //时间戳
            this.mappedByteBuffer.putInt(absIndexPos + 4 + 8 + 4, slotValue); //slot table 存储的单向列表的头
            this.mappedByteBuffer.putInt(absSlotPos, this.indexHeader.getIndexCount()); //更新slot table指向的头指针
            if (this.indexHeader.getIndexCount() <= 1) {
                this.indexHeader.setBeginPhyOffset(phyOffset);
                this.indexHeader.setBeginTimestamp(storeTimestamp);
            }

            this.indexHeader.incHashSlotCount();
            this.indexHeader.incIndexCount();
            this.indexHeader.setEndPhyOffset(phyOffset);
            this.indexHeader.setEndTimestamp(storeTimestamp);

            return true;
        } catch (Exception e) {
            log.error("putKey exception, Key: " + key + " KeyHashCode: " + key.hashCode(), e);
        } finally {
            if (fileLock != null) {
                try {
                    fileLock.release();
                } catch (IOException e) {
                    log.error("Failed to release the lock", e);
                }
            }
        }
    } else {
        log.warn("Over index file capacity: index count = " + this.indexHeader.getIndexCount()
            + "; index max num = " + this.indexNum);
    }
    return false;
}
```

按照Message Key查询消息

1、根据Key计算hashcode

2、根据hashcode和hashslotnum获取所属的slot

3、根据所属的slot获取此slot的第一条索引信息

org.apache.rocketmq.store.index.IndexFile#selectPhyOffset

```java
public void selectPhyOffset(final List<Long> phyOffsets, final String key, final int maxNum,
    final long begin, final long end, boolean lock) {
    if (this.mappedFile.hold()) {
        int keyHash = indexKeyHashMethod(key);
        int slotPos = keyHash % this.hashSlotNum;
        int absSlotPos = IndexHeader.INDEX_HEADER_SIZE + slotPos * hashSlotSize;
        FileLock fileLock = null;
        try {
            if (lock) {
                // fileLock = this.fileChannel.lock(absSlotPos,
                // hashSlotSize, true);
            }
            int slotValue = this.mappedByteBuffer.getInt(absSlotPos);
            // if (fileLock != null) {
            // fileLock.release();
            // fileLock = null;
            // }
            if (slotValue <= invalidIndex || slotValue > this.indexHeader.getIndexCount()
                || this.indexHeader.getIndexCount() <= 1) {
            } else {
                for (int nextIndexToRead = slotValue; ; ) {
                    if (phyOffsets.size() >= maxNum) {
                        break;
                    }
                    //此slot在indexfile中的位置
                    int absIndexPos =
                        IndexHeader.INDEX_HEADER_SIZE + this.hashSlotNum * hashSlotSize
                            + nextIndexToRead * indexSize;
                    //hashcode
                    int keyHashRead = this.mappedByteBuffer.getInt(absIndexPos);
                    //物理位置
                    long phyOffsetRead = this.mappedByteBuffer.getLong(absIndexPos + 4);
                    long timeDiff = (long) this.mappedByteBuffer.getInt(absIndexPos + 4 + 8);
                    //同一个slot table中下一个数据的位置
                    int prevIndexRead = this.mappedByteBuffer.getInt(absIndexPos + 4 + 8 + 4);
                    if (timeDiff < 0) {
                        break;
                    }
                    timeDiff *= 1000L;
                    long timeRead = this.indexHeader.getBeginTimestamp() + timeDiff;
                    boolean timeMatched = (timeRead >= begin) && (timeRead <= end);
                    //判断hashcode和时间
                    if (keyHash == keyHashRead && timeMatched) {
                        phyOffsets.add(phyOffsetRead);
                    }
                    if (prevIndexRead <= invalidIndex
                        || prevIndexRead > this.indexHeader.getIndexCount()
                        || prevIndexRead == nextIndexToRead || timeRead < begin) {
                        break;
                    }
                    nextIndexToRead = prevIndexRead;
                }
            }
        } catch (Exception e) {
            log.error("selectPhyOffset exception ", e);
        } finally {
            if (fileLock != null) {
                try {
                    fileLock.release();
                } catch (IOException e) {
                    log.error("Failed to release the lock", e);
                }
            }
            this.mappedFile.release();
        }
    }
}
```

##  消息拉取

根据topic和queueId获取ConsumeQueue

从ConsumeQueue获取minOffset和maxOffset，判断用户请求的offset的合理性

计算可以拉取的最大数量

根据请求的offet读取消费者可见的索引消息

根据索引消息获取消息，再根据tag的hashcode在服务端进行过滤

记录topic-queueId积压的数据

计算下次拉取的offset

如果积压的数据超过总内存的40%，建议从slave拉取。

```java
public GetMessageResult getMessage(final String group, final String topic, final int queueId, final long offset,
    final int maxMsgNums,
    final MessageFilter messageFilter) {
    if (this.shutdown) {
        log.warn("message store has shutdown, so getMessage is forbidden");
        return null;
    }
    if (!this.runningFlags.isReadable()) {
        log.warn("message store is not readable, so getMessage is forbidden " + this.runningFlags.getFlagBits());
        return null;
    }
    long beginTime = this.getSystemClock().now();
    GetMessageStatus status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
    long nextBeginOffset = offset;
    long minOffset = 0;
    long maxOffset = 0;
    GetMessageResult getResult = new GetMessageResult();
    final long maxOffsetPy = this.commitLog.getMaxOffset();
    //根据topic、queueId获取ConsumeQueue
    ConsumeQueue consumeQueue = findConsumeQueue(topic, queueId);
    if (consumeQueue != null) {
        //获取consumeQueue中的minOffset、maxOffset
        minOffset = consumeQueue.getMinOffsetInQueue();
        maxOffset = consumeQueue.getMaxOffsetInQueue();
        //判断offset的合法性
        if (maxOffset == 0) {
            status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
            nextBeginOffset = nextOffsetCorrection(offset, 0);
        } else if (offset < minOffset) {
            status = GetMessageStatus.OFFSET_TOO_SMALL;
            nextBeginOffset = nextOffsetCorrection(offset, minOffset);
        } else if (offset == maxOffset) {
            status = GetMessageStatus.OFFSET_OVERFLOW_ONE;
            nextBeginOffset = nextOffsetCorrection(offset, offset);
        } else if (offset > maxOffset) {
            status = GetMessageStatus.OFFSET_OVERFLOW_BADLY;
            if (0 == minOffset) {
                nextBeginOffset = nextOffsetCorrection(offset, minOffset);
            } else {
                nextBeginOffset = nextOffsetCorrection(offset, maxOffset);
            }
        } else {
            //根据offset计算索引文件读取位置
            SelectMappedBufferResult bufferConsumeQueue = consumeQueue.getIndexBuffer(offset);
            if (bufferConsumeQueue != null) {
                try {
                    status = GetMessageStatus.NO_MATCHED_MESSAGE;
                    long nextPhyFileStartOffset = Long.MIN_VALUE;
                    long maxPhyOffsetPulling = 0;
                    int i = 0;
                    final int maxFilterMessageCount = Math.max(16000, maxMsgNums * ConsumeQueue.CQ_STORE_UNIT_SIZE);
                    //默认true
                    final boolean diskFallRecorded = this.messageStoreConfig.isDiskFallRecorded();
                    ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                    for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                        long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
                        int sizePy = bufferConsumeQueue.getByteBuffer().getInt();
                        long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
                        maxPhyOffsetPulling = offsetPy;
                        if (nextPhyFileStartOffset != Long.MIN_VALUE) {
                            if (offsetPy < nextPhyFileStartOffset)
                                continue;
                        }
                        boolean isInDisk = checkInDiskByCommitOffset(offsetPy, maxOffsetPy);
                        if (this.isTheBatchFull(sizePy, maxMsgNums, getResult.getBufferTotalSize(), getResult.getMessageCount(),
                            isInDisk)) {
                            break;
                        }
                        boolean extRet = false, isTagsCodeLegal = true;
                        if (consumeQueue.isExtAddr(tagsCode)) {
                            extRet = consumeQueue.getExt(tagsCode, cqExtUnit);
                            if (extRet) {
                                tagsCode = cqExtUnit.getTagsCode();
                            } else {
                                // can't find ext content.Client will filter messages by tag also.
                                log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}, topic={}, group={}",
                                    tagsCode, offsetPy, sizePy, topic, group);
                                isTagsCodeLegal = false;
                            }
                        }

                        if (messageFilter != null
                            && !messageFilter.isMatchedByConsumeQueue(isTagsCodeLegal ? tagsCode : null, extRet ? cqExtUnit : null)) {
                            if (getResult.getBufferTotalSize() == 0) {
                                status = GetMessageStatus.NO_MATCHED_MESSAGE;
                            }
                            continue;
                        }
                        //读取数据
                        SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
                        if (null == selectResult) {
                            if (getResult.getBufferTotalSize() == 0) {
                                status = GetMessageStatus.MESSAGE_WAS_REMOVING;
                            }
                            nextPhyFileStartOffset = this.commitLog.rollNextFile(offsetPy);
                            continue;
                        }
                        if (messageFilter != null
                            && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null)) {
                            if (getResult.getBufferTotalSize() == 0) {
                                status = GetMessageStatus.NO_MATCHED_MESSAGE;
                            }
                            // release...
                            selectResult.release();
                            continue;
                        }
                        this.storeStatsService.getGetMessageTransferedMsgCount().incrementAndGet();
                        getResult.addMessage(selectResult);
                        status = GetMessageStatus.FOUND;
                        nextPhyFileStartOffset = Long.MIN_VALUE;
                    }
                    //默认true，记录此group下，topic-queueId积压的数据
                    if (diskFallRecorded) {
                        long fallBehind = maxOffsetPy - maxPhyOffsetPulling;
                        brokerStatsManager.recordDiskFallBehindSize(group, topic, queueId, fallBehind);
                    }
                    //计算下次请求的offset
                    nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                    //积压的数据
                    long diff = maxOffsetPy - maxPhyOffsetPulling;
                    //默认24G*(40/100)
                    long memory = (long) (StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE
                        * (this.messageStoreConfig.getAccessMessageInMemoryMaxRatio() / 100.0));
                    //默认情况下，如果积压的数据超过总内存的的40%，建议从slave拉取数据；避免'挖坟'导致尾部消息缓存命中率下降
                    getResult.setSuggestPullingFromSlave(diff > memory);
                } finally {
                    bufferConsumeQueue.release();
                }
            } else {
                status = GetMessageStatus.OFFSET_FOUND_NULL;
                nextBeginOffset = nextOffsetCorrection(offset, consumeQueue.rollNextFile(offset));
                log.warn("consumer request topic: " + topic + "offset: " + offset + " minOffset: " + minOffset + " maxOffset: "
                    + maxOffset + ", but access logic queue failed.");
            }
        }
    } else {
        status = GetMessageStatus.NO_MATCHED_LOGIC_QUEUE;
        nextBeginOffset = nextOffsetCorrection(offset, 0);
    }

    if (GetMessageStatus.FOUND == status) {
        this.storeStatsService.getGetMessageTimesTotalFound().incrementAndGet();
    } else {
        this.storeStatsService.getGetMessageTimesTotalMiss().incrementAndGet();
    }
    long elapsedTime = this.getSystemClock().now() - beginTime;
    this.storeStatsService.setGetMessageEntireTimeMax(elapsedTime);
    getResult.setStatus(status);
    getResult.setNextBeginOffset(nextBeginOffset); //设置下次读取的offset
    getResult.setMaxOffset(maxOffset);
    getResult.setMinOffset(minOffset);
    return getResult;
}
```

## 拉取请求的挂起

如果消费者没有拉取到消息，先将拉取请求挂起。

1、定时轮询查看是否有新消息写入，如果有则将挂起的请求唤醒，再次执行（PullRequestHoldService）

2、直接通知，当有消息写入时，就唤醒挂起的请求，即时感知到消息的写入（NotifyMessageArrivingListener）

### 轮询检测

org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest

```java
PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
    this.brokerController.getMessageStore().now(), offset, subscriptionData, messageFilter);
this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
```

#### 存放挂起的拉取请求

org.apache.rocketmq.broker.longpolling.PullRequestHoldService#suspendPullRequest

```java
public void suspendPullRequest(final String topic, final int queueId, final PullRequest pullRequest) {
  //topic@queudId -> ManyPullRequest
    String key = this.buildKey(topic, queueId);
    ManyPullRequest mpr = this.pullRequestTable.get(key);
    if (null == mpr) {
        mpr = new ManyPullRequest();
        ManyPullRequest prev = this.pullRequestTable.putIfAbsent(key, mpr);
        if (prev != null) {
            mpr = prev;
        }
    }
    mpr.addPullRequest(pullRequest);
}
```

#### 检测挂起的请求

org.apache.rocketmq.broker.longpolling.PullRequestHoldService#run

```java
public void run() {
    log.info("{} service started", this.getServiceName());
    while (!this.isStopped()) {
        try {
            if (this.brokerController.getBrokerConfig().isLongPollingEnable()) {
                this.waitForRunning(5 * 1000);
            } else {
this.waitForRunning(this.brokerController.getBrokerConfig().getShortPollingTimeMills());
            }
            long beginLockTimestamp = this.systemClock.now();
            //检测挂起的请求
            this.checkHoldRequest();
            long costTime = this.systemClock.now() - beginLockTimestamp;
            if (costTime > 5 * 1000) {
                log.info("[NOTIFYME] check hold request cost {} ms.", costTime);
            }
        } catch (Throwable e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }
    log.info("{} service end", this.getServiceName());
}
```

```java
private void checkHoldRequest() {
    for (String key : this.pullRequestTable.keySet()) {
        String[] kArray = key.split(TOPIC_QUEUEID_SEPARATOR);
        if (2 == kArray.length) {
            String topic = kArray[0];
            int queueId = Integer.parseInt(kArray[1]);
          //获取ConsumeQueue中最大的offset
            final long offset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
            try {
                this.notifyMessageArriving(topic, queueId, offset);
            } catch (Throwable e) {
                log.error("check hold request failed. topic={}, queueId={}", topic, queueId, e);
            }
        }
    }
}
```

org.apache.rocketmq.broker.longpolling.PullRequestHoldService#notifyMessageArriving

```java
public void notifyMessageArriving(final String topic, final int queueId, final long maxOffset, final Long tagsCode,
    long msgStoreTime, byte[] filterBitMap, Map<String, String> properties) {
    String key = this.buildKey(topic, queueId);
    ManyPullRequest mpr = this.pullRequestTable.get(key);
    if (mpr != null) {
        List<PullRequest> requestList = mpr.cloneListAndClear();
        if (requestList != null) {
            List<PullRequest> replayList = new ArrayList<PullRequest>();
            for (PullRequest request : requestList) {
                long newestOffset = maxOffset;
                //再获取一次ConsumeQueue最大的offset
                if (newestOffset <= request.getPullFromThisOffset()) {
                    newestOffset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
                }
                //大于用户请求的offset，写入了新消息
                if (newestOffset > request.getPullFromThisOffset()) {
                    boolean match = request.getMessageFilter().isMatchedByConsumeQueue(tagsCode,
                        new ConsumeQueueExt.CqExtUnit(tagsCode, msgStoreTime, filterBitMap));
                    // match by bit map, need eval again when properties is not null.
                    if (match && properties != null) {
                        match = request.getMessageFilter().isMatchedByCommitLog(null, properties);
                    }
                    if (match) {
                        try {
                            //执行拉取请求
 this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                                request.getRequestCommand());
                        } catch (Throwable e) {
                            log.error("execute request when wakeup failed.", e);
                        }
                        continue;
                    }
                }
                //挂起请求已经超时
                if (System.currentTimeMillis() >= (request.getSuspendTimestamp() + request.getTimeoutMillis())) {
                    try {
                        ////执行拉取请求
                        this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                            request.getRequestCommand());
                    } catch (Throwable e) {
                        log.error("execute request when wakeup failed.", e);
                    }
                    continue;
                }
                replayList.add(request);
            }

            if (!replayList.isEmpty()) {
                mpr.addPullRequest(replayList);
            }
        }
    }
}
```

### 写入及时唤醒

org.apache.rocketmq.store.DefaultMessageStore.ReputMessageService#doReput

```java
DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
        dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
        dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
        dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
```

org.apache.rocketmq.broker.longpolling.NotifyMessageArrivingListener#arriving

```java
public void arriving(String topic, int queueId, long logicOffset, long tagsCode,
    long msgStoreTime, byte[] filterBitMap, Map<String, String> properties) {
    this.pullRequestHoldService.notifyMessageArriving(topic, queueId, logicOffset, tagsCode,
        msgStoreTime, filterBitMap, properties);
}
```

## 消息同步

### HAClient

org.apache.rocketmq.store.ha.HAService.HAClient#run

```java
public void run() {
    log.info(this.getServiceName() + " service started");
    while (!this.isStopped()) {
        try {
            if (this.connectMaster()) { //是否已经连接到master
                if (this.isTimeToReportOffset()) { //是否需要上报offfset
                    boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset); //向master发送offset
                    if (!result) {
                        this.closeMaster();
                    }
                }
                this.selector.select(1000);

                boolean ok = this.processReadEvent(); //处理从master接收到的数据
                if (!ok) { //处理失败，关闭连接重新连接master
                    this.closeMaster();
                }

                //上报slave节点的offset
                if (!reportSlaveMaxOffsetPlus()) {
                    continue;
                }
								//写间隔，默认20s
                long interval =
                    HAService.this.getDefaultMessageStore().getSystemClock().now()
                        - this.lastWriteTimestamp;
                if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig()
                    .getHaHousekeepingInterval()) { //master响应超时，重新连接
                    log.warn("HAClient, housekeeping, found this connection[" + this.masterAddress
                        + "] expired, " + interval);
                    this.closeMaster();
                    log.warn("HAClient, master not response some time, so close connection");
                }
            } else {
                this.waitForRunning(1000 * 5);
            }
        } catch (Exception e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
            this.waitForRunning(1000 * 5);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```

#### 处理接收的数据

org.apache.rocketmq.store.ha.HAService.HAClient#processReadEvent

```java
private boolean processReadEvent() {
    int readSizeZeroTimes = 0;
    while (this.byteBufferRead.hasRemaining()) {
        try {
          	//读取master发送的数据，暂存至byteBufferRead
            int readSize = this.socketChannel.read(this.byteBufferRead);
            if (readSize > 0) { //读取字节数大于0
                readSizeZeroTimes = 0;
                boolean result = this.dispatchReadRequest();//处理读到的数据
                if (!result) {
                    log.error("HAClient, dispatchReadRequest error");
                    return false;
                }
            } else if (readSize == 0) {
                if (++readSizeZeroTimes >= 3) {
                    break;
                }
            } else {
                log.info("HAClient, processReadEvent read socket < 0");
                return false;
            }
        } catch (IOException e) {
            log.info("HAClient, processReadEvent read socket exception", e);
            return false;
        }
    }
    return true;
}
```

org.apache.rocketmq.store.ha.HAService.HAClient#dispatchReadRequest

```java
private boolean dispatchReadRequest() {
    final int msgHeaderSize = 8 + 4; // phyoffset + size
    int readSocketPos = this.byteBufferRead.position(); //开始position

    while (true) {
        int diff = this.byteBufferRead.position() - this.dispatchPosition;
        if (diff >= msgHeaderSize) { //大于12字节
            //master节点的offset
            long masterPhyOffset = this.byteBufferRead.getLong(this.dispatchPosition);
            //消息长度
            int bodySize = this.byteBufferRead.getInt(this.dispatchPosition + 8);
            //获取slave节点的commitLog的offset
            long slavePhyOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();

            if (slavePhyOffset != 0) {
                 //slave节点的offset与master的offset不一致
                if (slavePhyOffset != masterPhyOffset) { 
                    log.error("master pushed offset not equal the max phy offset in slave, SLAVE: "
                        + slavePhyOffset + " MASTER: " + masterPhyOffset);
                    return false;
                }
            }

            if (diff >= (msgHeaderSize + bodySize)) { //至少读取到一条完整数据
                //读取消息到bodyData
                byte[] bodyData = new byte[bodySize];
                //修改position
                this.byteBufferRead.position(this.dispatchPosition + msgHeaderSize);
              	//读取完整的数据
                this.byteBufferRead.get(bodyData);

                //追加消息到commitlog
                HAService.this.defaultMessageStore.appendToCommitLog(masterPhyOffset, bodyData);
								//重置byteBufferRead的position
                this.byteBufferRead.position(readSocketPos);
                //已经读取到的位置
                this.dispatchPosition += msgHeaderSize + bodySize;

                if (!reportSlaveMaxOffsetPlus()) { //上报salve同步的进度
                    return false;
                }

                continue;
            }
        }

        if (!this.byteBufferRead.hasRemaining()) {
            this.reallocateByteBuffer();
        }

        break;
    }

    return true;
}
```

#### 上报同步的进度

org.apache.rocketmq.store.ha.HAService.HAClient#reportSlaveMaxOffsetPlus

```java
private boolean reportSlaveMaxOffsetPlus() {
    boolean result = true;
    //commitLog最大的offset
    long currentPhyOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();
    if (currentPhyOffset > this.currentReportedOffset) {//大于之前上报的offst
        this.currentReportedOffset = currentPhyOffset; //修改上报的offset
        result = this.reportSlaveMaxOffset(this.currentReportedOffset); //上报
        if (!result) { //上报出现异常
            this.closeMaster(); //关闭连接重连
            log.error("HAClient, reportSlaveMaxOffset error, " + this.currentReportedOffset);
        }
    }
    return result;
}
```

org.apache.rocketmq.store.ha.HAService.HAClient#reportSlaveMaxOffset

```java
private boolean reportSlaveMaxOffset(final long maxOffset) {
    this.reportOffset.position(0);
    this.reportOffset.limit(8);
    this.reportOffset.putLong(maxOffset); //slave节点的offset
    this.reportOffset.position(0); //重置bytebuf
    this.reportOffset.limit(8);

    for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
        try {
            //将reportOffset的数据发送到master
            this.socketChannel.write(this.reportOffset); 
        } catch (IOException e) {
            log.error(this.getServiceName()
                + "reportSlaveMaxOffset this.socketChannel.write exception", e);
            return false;
        }
    }

    lastWriteTimestamp = HAService.this.defaultMessageStore.getSystemClock().now();
    return !this.reportOffset.hasRemaining();
}
```

### AcceptSocketService

org.apache.rocketmq.store.ha.HAService.AcceptSocketService#run

```java
public void run() { //Master监听slave的连接
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.selector.select(1000);
            Set<SelectionKey> selected = this.selector.selectedKeys();

            if (selected != null) {
                for (SelectionKey k : selected) {
                    if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                        SocketChannel sc = ((ServerSocketChannel) k.channel()).accept();

                        if (sc != null) {
                            HAService.log.info("HAService receive new connection, "
                                + sc.socket().getRemoteSocketAddress());

                            try {
                               //为每个slave创建HAConnection
                                HAConnection conn = new HAConnection(HAService.this, sc);
                                conn.start();
                                //管理HAConnection
                                HAService.this.addConnection(conn);
                            } catch (Exception e) {
                                log.error("new HAConnection exception", e);
                                sc.close();
                            }
                        }
                    } else {
                        log.warn("Unexpected ops in select " + k.readyOps());
                    }
                }

                selected.clear();
            }
        } catch (Exception e) {
            log.error(this.getServiceName() + " service has exception.", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```

### HAConnection

#### ReadSocketService

org.apache.rocketmq.store.ha.HAConnection.ReadSocketService#run

```java
public void run() { //读取slave上报的同步进度
    HAConnection.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.selector.select(1000);
            boolean ok = this.processReadEvent();
            if (!ok) {
                HAConnection.log.error("processReadEvent error");
                break;
            }

            long interval = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastReadTimestamp;
            if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaHousekeepingInterval()) {
                log.warn("ha housekeeping, found this connection[" + HAConnection.this.clientAddr + "] expired, " + interval);
                break;
            }
        } catch (Exception e) {
            HAConnection.log.error(this.getServiceName() + " service has exception.", e);
            break;
        }
    }

    this.makeStop();

    writeSocketService.makeStop();

    haService.removeConnection(HAConnection.this);

    HAConnection.this.haService.getConnectionCount().decrementAndGet();

    SelectionKey sk = this.socketChannel.keyFor(this.selector);
    if (sk != null) {
        sk.cancel();
    }

    try {
        this.selector.close();
        this.socketChannel.close();
    } catch (IOException e) {
        HAConnection.log.error("", e);
    }

    HAConnection.log.info(this.getServiceName() + " service end");
}
```

org.apache.rocketmq.store.ha.HAService#notifyTransferSome

```java
public void notifyTransferSome(final long offset) {
  	//push2SlaveMaxOffset存放salve同步的最大offset
    for (long value = this.push2SlaveMaxOffset.get(); offset > value; ) {
        boolean ok = this.push2SlaveMaxOffset.compareAndSet(value, offset); //修改offset
        if (ok) {
            this.groupTransferService.notifyTransferSome(); //唤醒等待salve复制成功的写入线程
            break;
        } else {
            value = this.push2SlaveMaxOffset.get();
        }
    }
}
```

#### WriteSocketService

org.apache.rocketmq.store.ha.HAConnection.WriteSocketService#run

```java
public void run() { //master向slave同步数据
    HAConnection.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.selector.select(1000);

            if (-1 == HAConnection.this.slaveRequestOffset) {
                Thread.sleep(10);
                continue;
            }

            if (-1 == this.nextTransferFromWhere) {
                if (0 == HAConnection.this.slaveRequestOffset) {
                    long masterOffset = HAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
                    masterOffset =
                        masterOffset
                            - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                            .getMappedFileSizeCommitLog());

                    if (masterOffset < 0) {
                        masterOffset = 0;
                    }

                    this.nextTransferFromWhere = masterOffset;
                } else {
                    this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
                }

                log.info("master transfer data from " + this.nextTransferFromWhere + " to slave[" + HAConnection.this.clientAddr
                    + "], and slave request " + HAConnection.this.slaveRequestOffset);
            }

            if (this.lastWriteOver) {

                long interval =
                    HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;

                if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                    .getHaSendHeartbeatInterval()) {

                    // Build Header
                    this.byteBufferHeader.position(0);
                    this.byteBufferHeader.limit(headerSize);
                    this.byteBufferHeader.putLong(this.nextTransferFromWhere);
                    this.byteBufferHeader.putInt(0);
                    this.byteBufferHeader.flip();

                    this.lastWriteOver = this.transferData();
                    if (!this.lastWriteOver)
                        continue;
                }
            } else {
                this.lastWriteOver = this.transferData();
                if (!this.lastWriteOver)
                    continue;
            }

            SelectMappedBufferResult selectResult =
                HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
            if (selectResult != null) {
                int size = selectResult.getSize();
                if (size > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize()) {
                    size = HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize();
                }

                long thisOffset = this.nextTransferFromWhere;
                this.nextTransferFromWhere += size;

                selectResult.getByteBuffer().limit(size);
                this.selectMappedBufferResult = selectResult;

                // Build Header
                this.byteBufferHeader.position(0);
                this.byteBufferHeader.limit(headerSize);
                this.byteBufferHeader.putLong(thisOffset);
                this.byteBufferHeader.putInt(size);
                this.byteBufferHeader.flip();

                this.lastWriteOver = this.transferData();
            } else {

                HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
            }
        } catch (Exception e) {

            HAConnection.log.error(this.getServiceName() + " service has exception.", e);
            break;
        }
    }

    HAConnection.this.haService.getWaitNotifyObject().removeFromWaitingThreadTable();

    if (this.selectMappedBufferResult != null) {
        this.selectMappedBufferResult.release();
    }

    this.makeStop();

    readSocketService.makeStop();

    haService.removeConnection(HAConnection.this);

    SelectionKey sk = this.socketChannel.keyFor(this.selector);
    if (sk != null) {
        sk.cancel();
    }

    try {
        this.selector.close();
        this.socketChannel.close();
    } catch (IOException e) {
        HAConnection.log.error("", e);
    }

    HAConnection.log.info(this.getServiceName() + " service end");
}
```

### GroupTransferService

org.apache.rocketmq.store.ha.HAService.GroupTransferService#putRequest

```java
public synchronized void putRequest(final CommitLog.GroupCommitRequest request) { //存放等待slave同步完成的写入请求
    synchronized (this.requestsWrite) {
        this.requestsWrite.add(request);
    }
    if (hasNotified.compareAndSet(false, true)) {
        waitPoint.countDown(); // 唤醒线程
    }
}
```

org.apache.rocketmq.store.ha.HAService.GroupTransferService#doWaitTransfer

```java
private void doWaitTransfer() {
    synchronized (this.requestsRead) {
        if (!this.requestsRead.isEmpty()) {
            for (CommitLog.GroupCommitRequest req : this.requestsRead) {
                //salve同步的offset超过req的nextOffset，表示slave已经同步完成
                boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
                //请求失效时间
                long waitUntilWhen = HAService.this.defaultMessageStore.getSystemClock().now()
                    + HAService.this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout();
                //一直阻塞到超时或者同步完成
                while (!transferOK && HAService.this.defaultMessageStore.getSystemClock().now() < waitUntilWhen) {
                    this.notifyTransferObject.waitForRunning(1000);
                    transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
                }
                if (!transferOK) {
                    log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
                }
                //有可能超时或者同步完成
                req.wakeupCustomer(transferOK);
            }
            this.requestsRead.clear();
        }
    }
}
```

## 延迟消息

只支持指定level的延迟，1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h，

单独的topic存储延迟消息，每个level对应一个分区

### load

org.apache.rocketmq.store.schedule.ScheduleMessageService#load

```java
public boolean load() {
    boolean result = super.load(); //加载持久化的offsetTable；记录已经消费的位置
    result = result && this.parseDelayLevel(); // 解析支持的消息级别
    return result;
}
```

### start

org.apache.rocketmq.store.schedule.ScheduleMessageService#start

```java
public void start() {
    if (started.compareAndSet(false, true)) {
        this.timer = new Timer("ScheduleMessageTimerThread", true);
        for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
            Integer level = entry.getKey();
            Long timeDelay = entry.getValue();
            Long offset = this.offsetTable.get(level);
            if (null == offset) {
                offset = 0L;
            }
            //为每个level创建DeliverDelayedMessageTimerTask，周期性检查延迟消息是否已经到期，将其存放到指定的topic中
            if (timeDelay != null) {
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
            }
        }

        //周期持久化offsetTable即每个level消费的位置，
        this.timer.scheduleAtFixedRate(new TimerTask() {

            @Override
            public void run() {
                try {
                    if (started.get()) ScheduleMessageService.this.persist();
                } catch (Throwable e) {
                    log.error("scheduleAtFixedRate flush exception", e);
                }
            }
        }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
    }
}
```

org.apache.rocketmq.store.schedule.ScheduleMessageService.DeliverDelayedMessageTimerTask#executeOnTimeup

```java
public void executeOnTimeup() {
    //每个delayLevel都有单独的ConsumeQueue
    ConsumeQueue cq =
            ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(SCHEDULE_TOPIC,
                    delayLevel2QueueId(delayLevel));

    long failScheduleOffset = offset;

    if (cq != null) {
        SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
        if (bufferCQ != null) {
            try {
                long nextOffset = offset;
                int i = 0;
                ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) { //一条索引信息占20字节
                    long offsetPy = bufferCQ.getByteBuffer().getLong();
                    int sizePy = bufferCQ.getByteBuffer().getInt();
                    long tagsCode = bufferCQ.getByteBuffer().getLong();

                    //*****
                    if (cq.isExtAddr(tagsCode)) {
                        if (cq.getExt(tagsCode, cqExtUnit)) {
                            tagsCode = cqExtUnit.getTagsCode();
                        } else {
                            //can't find ext content.So re compute tags code.
                            log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}",
                                tagsCode, offsetPy, sizePy);
                            long msgStoreTime = defaultMessageStore.getCommitLog().pickupStoreTimestamp(offsetPy, sizePy);
                            tagsCode = computeDeliverTimestamp(delayLevel, msgStoreTime);
                        }
                    }
                    //******

                    long now = System.currentTimeMillis();
                    long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode);
										
                    nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

                    long countdown = deliverTimestamp - now;

                    if (countdown <= 0) { //到期
                       //根据offset、size查找数据
                        MessageExt msgExt =
                        ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(
                                offsetPy, sizePy); 

                        if (msgExt != null) {
                            try {
                                MessageExtBrokerInner msgInner = this.messageTimeup(msgExt); //
                                if (MixAll.RMQ_SYS_TRANS_HALF_TOPIC.equals(msgInner.getTopic())) { //过滤事务半消息
                                    log.error("[BUG] the real topic of schedule msg is {}, discard the msg. msg={}",
                                            msgInner.getTopic(), msgInner);
                                    continue;
                                }
                                PutMessageResult putMessageResult =
                                    ScheduleMessageService.this.writeMessageStore
                                        .putMessage(msgInner); //再次写入磁盘

                                if (putMessageResult != null
                                    && putMessageResult.getPutMessageStatus() == PutMessageStatus.PUT_OK) {
                                    continue;
                                } else {
                                    // XXX: warn and notify me
                                    log.error(
                                        "ScheduleMessageService, a message time up, but reput it failed, topic: {} msgId {}",
                                        msgExt.getTopic(), msgExt.getMsgId());
                                    ScheduleMessageService.this.timer.schedule(
                                        new DeliverDelayedMessageTimerTask(this.delayLevel,
                                            nextOffset), DELAY_FOR_A_PERIOD);
                                    ScheduleMessageService.this.updateOffset(this.delayLevel,
                                        nextOffset);
                                    return;
                                }
                            } catch (Exception e) {
                                /*
                                 * XXX: warn and notify me



                                 */
                                log.error(
                                    "ScheduleMessageService, messageTimeup execute error, drop it. msgExt="
                                        + msgExt + ", nextOffset=" + nextOffset + ",offsetPy="
                                        + offsetPy + ",sizePy=" + sizePy, e);
                            }
                        }
                    } else { //没有到期，再次调度
                        ScheduleMessageService.this.timer.schedule(
                            new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),
                            countdown);
                        //修改消费的进度
                        ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                        return;
                    }
                } // end of for

                nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(
                    this.delayLevel, nextOffset), DELAY_FOR_A_WHILE);
                ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                return;
            } finally {

                bufferCQ.release();
            }
        } // end of if (bufferCQ != null)
        else {

            long cqMinOffset = cq.getMinOffsetInQueue();
            if (offset < cqMinOffset) {
                failScheduleOffset = cqMinOffset;
                log.error("schedule CQ offset invalid. offset=" + offset + ", cqMinOffset="
                    + cqMinOffset + ", queueId=" + cq.getQueueId());
            }
        }
    } // end of if (cq != null)

    ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel,
        failScheduleOffset), DELAY_FOR_A_WHILE);
}
```

## 事务消息

### 处理EndTransactionRequest

org.apache.rocketmq.broker.processor.EndTransactionProcessor#processRequest

```java
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws
    RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final EndTransactionRequestHeader requestHeader =
        (EndTransactionRequestHeader)request.decodeCommandCustomHeader(EndTransactionRequestHeader.class);
    LOGGER.debug("Transaction request:{}", requestHeader);
    if (BrokerRole.SLAVE == brokerController.getMessageStoreConfig().getBrokerRole()) {
        response.setCode(ResponseCode.SLAVE_NOT_AVAILABLE);
        LOGGER.warn("Message store is slave mode, so end transaction is forbidden. ");
        return response;
    }

    if (requestHeader.getFromTransactionCheck()) { //回查
        switch (requestHeader.getCommitOrRollback()) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE: {
                LOGGER.warn("Check producer[{}] transaction state, but it's pending status."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                return null;
            }

            case MessageSysFlag.TRANSACTION_COMMIT_TYPE: {
                LOGGER.warn("Check producer[{}] transaction state, the producer commit the message."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());

                break;
            }

            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE: {
                LOGGER.warn("Check producer[{}] transaction state, the producer rollback the message."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                break;
            }
            default:
                return null;
        }
    } else {
        switch (requestHeader.getCommitOrRollback()) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE: {
                LOGGER.warn("The producer[{}] end transaction in sending message,  and it's pending status."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                return null;
            }

            case MessageSysFlag.TRANSACTION_COMMIT_TYPE: {
                break;
            }

            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE: {
                LOGGER.warn("The producer[{}] end transaction in sending message, rollback the message."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                break;
            }
            default:
                return null;
        }
    }
    OperationResult result = new OperationResult();
  	//事务提交
    if (MessageSysFlag.TRANSACTION_COMMIT_TYPE == requestHeader.getCommitOrRollback()) {
        //根据commitLogOffset获取prepareMessage
        result = this.brokerController.getTransactionalMessageService().commitMessage(requestHeader);
        if (result.getResponseCode() == ResponseCode.SUCCESS) {
            //验证prepareMessage合法性
            RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
            if (res.getCode() == ResponseCode.SUCCESS) {
                MessageExtBrokerInner msgInner = endMessageTransaction(result.getPrepareMessage());
                msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), requestHeader.getCommitOrRollback()));
                msgInner.setQueueOffset(requestHeader.getTranStateTableOffset());
                msgInner.setPreparedTransactionOffset(requestHeader.getCommitLogOffset());
                msgInner.setStoreTimestamp(result.getPrepareMessage().getStoreTimestamp());
                MessageAccessor.clearProperty(msgInner, MessageConst.PROPERTY_TRANSACTION_PREPARED);
                RemotingCommand sendResult = sendFinalMessage(msgInner); //写入到真正的主题、队列中
                if (sendResult.getCode() == ResponseCode.SUCCESS) {//事务提交成功，将prepareMessage删除
                    this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
                }
                return sendResult;
            }
            return res;
        }
    } else if (MessageSysFlag.TRANSACTION_ROLLBACK_TYPE == requestHeader.getCommitOrRollback()) { //事务回滚
        result = this.brokerController.getTransactionalMessageService().rollbackMessage(requestHeader); //查询prepareMessage
        if (result.getResponseCode() == ResponseCode.SUCCESS) {
            RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
            if (res.getCode() == ResponseCode.SUCCESS) {//将prepareMessage删除
                this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
            }
            return res;
        }
    }
    response.setCode(result.getResponseCode());
    response.setRemark(result.getResponseRemark());
    return response;
}
```

### 定时检测half消息

org.apache.rocketmq.broker.transaction.queue.TransactionalMessageServiceImpl#check

```java
public void check(long transactionTimeout, int transactionCheckMax,
    AbstractTransactionalMessageCheckListener listener) {
    try {
        String topic = MixAll.RMQ_SYS_TRANS_HALF_TOPIC;
        //存放half消息
        Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues(topic);
        if (msgQueues == null || msgQueues.size() == 0) {
            log.warn("The queue of topic is empty :" + topic);
            return;
        }
        log.debug("Check topic={}, queues={}", topic, msgQueues);
        for (MessageQueue messageQueue : msgQueues) { //存放half消息的MessageQueue
            long startTime = System.currentTimeMillis();
            MessageQueue opQueue = getOpQueue(messageQueue); //存放op消息的MessageQueue
            long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);
            long opOffset = transactionalMessageBridge.fetchConsumeOffset(opQueue);
            log.info("Before check, the queue={} msgOffset={} opOffset={}", messageQueue, halfOffset, opOffset);
            if (halfOffset < 0 || opOffset < 0) {
                log.error("MessageQueue: {} illegal offset read: {}, op offset: {},skip this queue", messageQueue,
                    halfOffset, opOffset);
                continue;
            }

            List<Long> doneOpOffset = new ArrayList<>();
            HashMap<Long, Long> removeMap = new HashMap<>();
            PullResult pullResult = fillOpRemoveMap(removeMap, opQueue, opOffset, halfOffset, doneOpOffset);
            if (null == pullResult) {
                log.error("The queue={} check msgOffset={} with opOffset={} failed, pullResult is null",
                    messageQueue, halfOffset, opOffset);
                continue;
            }
            // single thread
            int getMessageNullCount = 1;
            long newOffset = halfOffset;
            long i = halfOffset;
            while (true) {
                if (System.currentTimeMillis() - startTime > MAX_PROCESS_TIME_LIMIT) {
                    log.info("Queue={} process time reach max={}", messageQueue, MAX_PROCESS_TIME_LIMIT);
                    break;
                }
                if (removeMap.containsKey(i)) { //half消息已经被提交或者回滚
                    log.info("Half offset {} has been committed/rolled back", i);
                    Long removedOpOffset = removeMap.remove(i);
                    doneOpOffset.add(removedOpOffset);
                } else { //half消息尚未处理
                    GetResult getResult = getHalfMsg(messageQueue, i); //获取half消息
                    MessageExt msgExt = getResult.getMsg();
                    if (msgExt == null) {
                        if (getMessageNullCount++ > MAX_RETRY_COUNT_WHEN_HALF_NULL) {
                            break;
                        }
                        if (getResult.getPullResult().getPullStatus() == PullStatus.NO_NEW_MSG) {
                            log.debug("No new msg, the miss offset={} in={}, continue check={}, pull result={}", i,
                                messageQueue, getMessageNullCount, getResult.getPullResult());
                            break;
                        } else {
                            log.info("Illegal offset, the miss offset={} in={}, continue check={}, pull result={}",
                                i, messageQueue, getMessageNullCount, getResult.getPullResult());
                            i = getResult.getPullResult().getNextBeginOffset();
                            newOffset = i;
                            continue;
                        }
                    }

                    //检测达到15次或者存活时间超过72小时
                    if (needDiscard(msgExt, transactionCheckMax) || needSkip(msgExt)) {
                        listener.resolveDiscardMsg(msgExt); //将此消息存储到TRANS_CHECK_MAXTIME_TOPIC
                        newOffset = i + 1;
                        i++;
                        continue;
                    }
                    if (msgExt.getStoreTimestamp() >= startTime) {
                        log.debug("Fresh stored. the miss offset={}, check it later, store={}", i,
                            new Date(msgExt.getStoreTimestamp()));
                        break;
                    }
                    //half消息存活的时间，存活时间超过配置的阈值，才会进行回调检查
                    long valueOfCurrentMinusBorn = System.currentTimeMillis() - msgExt.getBornTimestamp();
                    long checkImmunityTime = transactionTimeout; //服务端默认6s，超过6s才会进行回查
                    //客户端设置的回查时间
                    String checkImmunityTimeStr = msgExt.getUserProperty(MessageConst.PROPERTY_CHECK_IMMUNITY_TIME_IN_SECONDS);
                    if (null != checkImmunityTimeStr) {
                        //如果客户端设置-1，则取服务端的配置，否则取客户端配置
                        checkImmunityTime = getImmunityTime(checkImmunityTimeStr, transactionTimeout);
                        if (valueOfCurrentMinusBorn < checkImmunityTime) {
                            if (checkPrepareQueueOffset(removeMap, doneOpOffset, msgExt)) {
                                newOffset = i + 1;
                                i++;
                                continue;
                            }
                        }
                    } else {
                        if ((0 <= valueOfCurrentMinusBorn) && (valueOfCurrentMinusBorn < checkImmunityTime)) {
                            log.debug("New arrived, the miss offset={}, check it later checkImmunity={}, born={}", i,
                                checkImmunityTime, new Date(msgExt.getBornTimestamp()));
                            break;
                        }
                    }

                    List<MessageExt> opMsg = pullResult.getMsgFoundList();
                    boolean isNeedCheck = (opMsg == null && valueOfCurrentMinusBorn > checkImmunityTime)//尚未收到客户端提交、回滚事务的请求并且存活时间超过了阈值
                        || (opMsg != null && (opMsg.get(opMsg.size() - 1).getBornTimestamp() - startTime > transactionTimeout))
                        || (valueOfCurrentMinusBorn <= -1);

                    if (isNeedCheck) { //需要回查
                        if (!putBackHalfMsgQueue(msgExt, i)) { //写回half消息
                            continue;
                        }
                        //回查half消息
                        listener.resolveHalfMsg(msgExt);
                    } else { //不需要回查
                        pullResult = fillOpRemoveMap(removeMap, opQueue, pullResult.getNextBeginOffset(), halfOffset, doneOpOffset);
                        log.debug("The miss offset:{} in messageQueue:{} need to get more opMsg, result is:{}", i,
                            messageQueue, pullResult);
                        continue;
                    }
                }
                newOffset = i + 1;
                i++;
            }
            if (newOffset != halfOffset) {
                transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);
            }
            long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);
            if (newOpOffset != opOffset) {
                transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);
            }
        }
    } catch (Throwable e) {
        log.error("Check error", e);
    }
}
```

## 请求快速失败

org.apache.rocketmq.broker.latency.BrokerFastFailure#start

```java
public void start() {
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            if (brokerController.getBrokerConfig().isBrokerFastFailureEnable()) {//默认为true
                cleanExpiredRequest(); //清除过期请求
            }
        }
    }, 1000, 10, TimeUnit.MILLISECONDS);
}
```

org.apache.rocketmq.broker.latency.BrokerFastFailure#cleanExpiredRequest

```java
private void cleanExpiredRequest() {
    //首先检测操作系统的page cache是否繁忙
    while (this.brokerController.getMessageStore().isOSPageCacheBusy()) { 
        try {
            if (!this.brokerController.getSendThreadPoolQueue().isEmpty()) {
                final Runnable runnable = this.brokerController.getSendThreadPoolQueue().poll(0, TimeUnit.SECONDS);
                if (null == runnable) {
                    break;
                }

                final RequestTask rt = castRunnable(runnable);
                rt.returnResponse(RemotingSysResponseCode.SYSTEM_BUSY, String.format("[PCBUSY_CLEAN_QUEUE]broker busy, start flow control for a while, period in queue: %sms, size of queue: %d", System.currentTimeMillis() - rt.getCreateTimestamp(), this.brokerController.getSendThreadPoolQueue().size()));
            } else {
                break;
            }
        } catch (Throwable ignored) {
        }
    }

    cleanExpiredRequestInQueue(this.brokerController.getSendThreadPoolQueue(),
        this.brokerController.getBrokerConfig().getWaitTimeMillsInSendQueue());

    cleanExpiredRequestInQueue(this.brokerController.getPullThreadPoolQueue(),
        this.brokerController.getBrokerConfig().getWaitTimeMillsInPullQueue());

    cleanExpiredRequestInQueue(this.brokerController.getHeartbeatThreadPoolQueue(),
        this.brokerController.getBrokerConfig().getWaitTimeMillsInHeartbeatQueue());

    cleanExpiredRequestInQueue(this.brokerController.getEndTransactionThreadPoolQueue(), this
        .brokerController.getBrokerConfig().getWaitTimeMillsInTransactionQueue());
}
```

org.apache.rocketmq.store.DefaultMessageStore#isOSPageCacheBusy

```java
public boolean isOSPageCacheBusy() { //判断osPageCache是否繁忙
    long begin = this.getCommitLog().getBeginTimeInLock(); //写入数据时记录当前时间
    long diff = this.systemClock.now() - begin; //距离开始写入数据的时间差值（写数据到pagecache、刷新数据从pagecache到磁盘、slave拉取数据）

    return diff < 10_000_000
        && diff > this.messageStoreConfig.getOsPageCacheBusyTimeOutMills();//默认1000
}
```

org.apache.rocketmq.broker.latency.BrokerFastFailure#cleanExpiredRequestInQueue

```java
void cleanExpiredRequestInQueue(final BlockingQueue<Runnable> blockingQueue, final long maxWaitTimeMillsInQueue) {//清除过期请求，默认最大等待200ms
    while (true) {
        try {
            if (!blockingQueue.isEmpty()) {
                final Runnable runnable = blockingQueue.peek();
                if (null == runnable) {
                    break;
                }
                final RequestTask rt = castRunnable(runnable);//转成RequestTask
                if (rt == null || rt.isStopRun()) {
                    break;
                }

                final long behind = System.currentTimeMillis() - rt.getCreateTimestamp();
                if (behind >= maxWaitTimeMillsInQueue) {
                    if (blockingQueue.remove(runnable)) { //从队列移除
                        rt.setStopRun(true);
                        rt.returnResponse(RemotingSysResponseCode.SYSTEM_BUSY, String.format("[TIMEOUT_CLEAN_QUEUE]broker busy, start flow control for a while, period in queue: %sms, size of queue: %d", behind, blockingQueue.size())); //返回结果响应
                    }
                } else {
                    break;
                }
            } else {
                break;
            }
        } catch (Throwable ignored) {
        }
    }
}
```

## 禁用消费太慢的Group

org.apache.rocketmq.broker.BrokerController#protectBroker

```java
public void protectBroker() {
    if (this.brokerConfig.isDisableConsumeIfConsumerReadSlowly()) { //默认false
        final Iterator<Map.Entry<String, MomentStatsItem>> it = this.brokerStatsManager.getMomentStatsItemSetFallSize().getStatsItemTable().entrySet().iterator(); //在拉取数据时记录group下给topic-queue积压的数据
        while (it.hasNext()) {
            final Map.Entry<String, MomentStatsItem> next = it.next();
            final long fallBehindBytes = next.getValue().getValue().get();
            if (fallBehindBytes > this.brokerConfig.getConsumerFallbehindThreshold()) {//超过16G
                final String[] split = next.getValue().getStatsKey().split("@");
                final String group = split[2];
                LOG_PROTECTION.info("[PROTECT_BROKER] the consumer[{}] consume slowly, {} bytes, disable it", group, fallBehindBytes);
                this.subscriptionGroupManager.disableConsume(group); //禁止此group消费
            }
        }
    }
}
```

# 总结

1、请求隔离、队列隔离、线程池隔离

2、对MappedFile进行预热，分配真实的内存，避免了频繁的缺页中断。通过调用mlock,锁住指定的内存区域，确保不会被内核回收，保证读取写入数据时不会发生缺页中断。调用madvise方法，指定参数WILLNEED,提示内核指定的数据即将被访问 ，立即预读数据到page cache

