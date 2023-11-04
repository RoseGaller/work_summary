# DeFiBus源码分析

* [DeFiBus源码分析](#defibus源码分析)
* [概述](#概述)
* [RPC调用Request-Reply模式](#rpc调用request-reply模式)
* [Producer端的隔离](#producer端的隔离)
  * [隔离的实现](#隔离的实现)
  * [队列的健康管理器](#队列的健康管理器)
* [Consumer端的隔离](#consumer端的隔离)
  * [启动线程，开始拉取消息](#启动线程开始拉取消息)
  * [Broker健康检测](#broker健康检测)
  * [正常拉取消息](#正常拉取消息)
  * [隔离线程拉取消息](#隔离线程拉取消息)
  * [Broker健康管理器](#broker健康管理器)
* [自动伸缩Queue](#自动伸缩queue)
  * [伸缩readQueue](#伸缩readqueue)
  * [伸缩writeQueue](#伸缩writequeue)

# 概述

DeFiBus=RPC+MQ，提供了RPC同步调用

Decentralized Financial Message Bus -- 分布式金融级消息总线

# RPC调用Request-Reply模式

cn.webank.defibus.producer.DeFiBusProducer#request(Message, long)

```java
public Message request(Message msg, long timeout) //超时时间
    throws InterruptedException, RemotingException, MQClientException, MQBrokerException {
    return deFiBusProducerImpl.request(msg, timeout);
}
```

```java
public Message request(Message requestMsg,
    long timeout) throws InterruptedException, RemotingException, MQClientException, MQBrokerException {
    return request(requestMsg, null, null, timeout);
}
```

```java
public Message request(Message requestMsg, final SendCallback sendCallback, RRCallback rrCallback, long timeout)
    throws InterruptedException, RemotingException, MQClientException, MQBrokerException {

    boolean isAsyncRR = (rrCallback != null);
		//创建请求的唯一标志
    final String uniqueRequestId = DeFiBusRequestIDUtil.createUniqueName("w");
    DefaultMQProducer producer = deFiBusProducer.getDefaultMQProducer();
    requestMsg.putUserProperty(DeFiBusConstant.KEY, DeFiBusConstant.PERSISTENT);
    requestMsg.putUserProperty(DeFiBusConstant.PROPERTY_RR_REQUEST_ID, uniqueRequestId);
    requestMsg.putUserProperty(DeFiBusConstant.PROPERTY_MESSAGE_REPLY_TO, producer.buildMQClientId());
    requestMsg.putUserProperty(DeFiBusConstant.PROPERTY_MESSAGE_TTL, String.valueOf(timeout));

   //创建RRResponseFuture，封装RRCallback，返回响应后执行RRCallback
    final RRResponseFuture responseFurture = new RRResponseFuture(rrCallback, timeout);

    String topic = requestMsg.getTopic();
    boolean hasRouteData = deFiBusProducer.getDefaultMQProducer().getDefaultMQProducerImpl().getmQClientFactory().getTopicRouteTable().containsKey(topic);
    Boolean isSendHeartbeatOk = topicInitMap.get(topic);
    if (isSendHeartbeatOk == null) {
        isSendHeartbeatOk = false;
    }
    if (!hasRouteData || !isSendHeartbeatOk) {
        long startTimestamp = System.currentTimeMillis();
        synchronized (this) {
            boolean hasRouteDataSync = deFiBusProducer.getDefaultMQProducer().getDefaultMQProducerImpl().getmQClientFactory().getTopicRouteTable().containsKey(topic);
            if (!hasRouteDataSync) {
                LOGGER.info("no topic route info for " + topic + ", send heartbeat to nameserver");
                deFiBusProducer.getDefaultMQProducer().getDefaultMQProducerImpl().getmQClientFactory().updateTopicRouteInfoFromNameServer(topic);
                deFiBusProducer.getDefaultMQProducer().getDefaultMQProducerImpl().getmQClientFactory().sendHeartbeatToAllBrokerWithLock();
                topicInitMap.put(topic, true);
            }
        }
        long cost = System.currentTimeMillis() - startTimestamp;
        if (cost > 500) {
            LOGGER.warn("get topic route info for {} before request cost {} ms.", topic, cost);
        }
    }
		//暂存 uniqueRequestId->responseFurture,返回响应后根据请求标志获取对应的responseFurture
    ResponseTable.getRrResponseFurtureConcurrentHashMap().put(uniqueRequestId, responseFurture);
    if (isAsyncRR) { //异步
        this.publish(requestMsg, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                if (sendCallback != null) {
                    sendCallback.onSuccess(sendResult);
                }
            }

            @Override
            public void onException(Throwable e) {
                LOGGER.warn("except when publish async rr message, uniqueId :{} {} ", uniqueRequestId, e.getMessage());
                ResponseTable.getRrResponseFurtureConcurrentHashMap().remove(uniqueRequestId);
                if (sendCallback != null) {
                    sendCallback.onException(e);
                }
            }
        }, timeout);
        return null;

    } else {
        publish(requestMsg, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                if (sendCallback != null) {
                    sendCallback.onSuccess(sendResult);
                }
            }

            @Override
            public void onException(Throwable e) {
                LOGGER.warn("except when publish sync rr message, uniqueId :{} {}", uniqueRequestId, e.getMessage());
                ResponseTable.getRrResponseFurtureConcurrentHashMap().remove(uniqueRequestId);
                if (sendCallback != null) {
                    sendCallback.onException(e);
                }
            }
        }, timeout);
       //等待响应，直到超时
        Message retMessage = responseFurture.waitResponse(timeout);
      //移除暂存的responseFurture
        ResponseTable.getRrResponseFurtureConcurrentHashMap().remove(uniqueRequestId);
        if (retMessage == null) {
            LOGGER.warn("request {} is sent, constant is :{}, but no rr response ", topic, uniqueRequestId);
        }
        return retMessage;
    }
}
```

```java
public void publish(final Message msg, final SendCallback sendCallback,
    final long timeout) throws MQClientException, RemotingException, InterruptedException {
    if (msg.getUserProperty(DeFiBusConstant.PROPERTY_MESSAGE_TTL) == null) {
        msg.putUserProperty(DeFiBusConstant.PROPERTY_MESSAGE_TTL, DeFiBusConstant.DEFAULT_TTL);
    }

    final AtomicReference<MessageQueue> selectorArgs = new AtomicReference<MessageQueue>();
    AsynCircuitBreakSendCallBack asynCircuitBreakSendCallBack = new AsynCircuitBreakSendCallBack();
    asynCircuitBreakSendCallBack.setMsg(msg);
    asynCircuitBreakSendCallBack.setProducer(this.deFiBusProducer);
    asynCircuitBreakSendCallBack.setSelectorArg(selectorArgs);
    asynCircuitBreakSendCallBack.setSendCallback(sendCallback);//用户自定义的回调函数

    String topic = msg.getTopic();
    boolean hasRouteData = deFiBusProducer.getDefaultMQProducer().getDefaultMQProducerImpl().getmQClientFactory().getTopicRouteTable().containsKey(topic);
    if (!hasRouteData) {
        LOGGER.info("no topic route info for " + topic + ", send heartbeat to nameserver");
        deFiBusProducer.getDefaultMQProducer().getDefaultMQProducerImpl().getmQClientFactory().updateTopicRouteInfoFromNameServer(topic);
    }
		//发送消息
    DeFiBusProducerImpl.this.deFiBusProducer.getDefaultMQProducer().send(msg, messageQueueSelector, selectorArgs, asynCircuitBreakSendCallBack, timeout);
}
```

# Producer端的隔离

```
Producer在往Topic发送消息时，会按照MessageQueueSelector定义的选择策略，从Topic的所有MessageQueue中选择一个作为目标队列发送消息。当往某个队列发送消息失败时，将队列标记为隔离状态，在隔离过期之前将不再往这个队列发送消息，避免再次失败，降低失败概率。
```

## 隔离的实现

cn.webank.defibus.client.impl.producer.DeFiBusProducerImpl.AsynCircuitBreakSendCallBack#onSuccess

```java
public void onSuccess(SendResult sendResult) { //响应成功
    //发送成功，MessageQueueHealthManager移除此MessageQueue
 messageQueueSelector.getMessageQueueHealthManager().markQueueHealthy(sendResult.getMessageQueue());
    if (sendCallback != null) { //执行用户自定义的回调方法
        sendCallback.onSuccess(sendResult);
    }
}
```

cn.webank.defibus.client.impl.producer.DeFiBusProducerImpl.AsynCircuitBreakSendCallBack#onException

```java
public void onException(Throwable e) { //响应异常
    try {
        MessageQueueHealthManager messageQueueHealthManager
            = ((HealthyMessageQueueSelector) messageQueueSelector).getMessageQueueHealthManager();
        MessageQueue messageQueue = ((AtomicReference<MessageQueue>) selectorArg).get();
        if (messageQueue != null) {
            //标志此messagequeue异常
            messageQueueSelector.getMessageQueueHealthManager().markQueueFault(messageQueue);
            if (messageQueueSelector.getMessageQueueHealthManager().isQueueFault(messageQueue)) {
                LOGGER.warn("isolate send failed mq. {} cause: {}", messageQueue, e.getMessage());
            }
        }
        //此队列对应的消费者消费速度太慢，防止消息积压，进行熔断，将消息发送到其他队列
        if (e.getMessage().contains("CODE: " + DeFiBusResponseCode.CONSUME_DIFF_SPAN_TOO_LONG)) {
            //first retry initialize 第一次回调
            if (queueCount == 0) {
                //获取该Topic的MessageQueue信息
                List<MessageQueue> messageQueueList = producer.getDefaultMQProducer().getDefaultMQProducerImpl().getTopicPublishInfoTable()
                    .get(msg.getTopic()).getMessageQueueList();
                queueCount = messageQueueList.size(); //设置MessageQueueList大小
                String clusterPrefix = deFiBusProducer.getDeFiBusClientConfig().getClusterPrefix(); //获取集群前缀
                if (!StringUtils.isEmpty(clusterPrefix)) {
                    for (MessageQueue mq : messageQueueList) {
                        if (messageQueueHealthManager.isQueueFault(mq)) { //判断该messageQueue是否可以正常发送消息
                            queueCount--;
                        }
                    }
                }
            }
            //重试次数
            int retryTimes = Math.min(queueCount, deFiBusProducer.getDeFiBusClientConfig().getRetryTimesWhenSendAsyncFailed());
            if (circuitBreakRetryTimes.get() < retryTimes) {
                circuitBreakRetryTimes.incrementAndGet();
                LOGGER.warn("fuse:send to [{}] circuit break, retry no.[{}] times, msgKey:[{}]", messageQueue.toString(), circuitBreakRetryTimes.intValue(), msg.getKeys());
                producer.getDefaultMQProducer().send(msg, messageQueueSelector, selectorArg, this);
                //no exception to client when retry
                return;
            } else {
                LOGGER.warn("fuse:send to [{}] circuit break after retry {} times, msgKey:[{}]", messageQueue.toString(), retryTimes, msg.getKeys());
            }
        } else { //其他异常处理
            int maxRetryTimes = producer.getDeFiBusClientConfig().getRetryTimesWhenSendAsyncFailed();
            if (sendRetryTimes.getAndIncrement() < maxRetryTimes) {
                LOGGER.info("send message fail, retry {} now, msgKey: {}, cause: {}", sendRetryTimes.get(), msg.getKeys(), e.getMessage());
                producer.getDefaultMQProducer().send(msg, messageQueueSelector, selectorArg, this);
                return;
            } else {
                LOGGER.warn("send message fail, after retry {} times, msgKey:[{}]", maxRetryTimes, msg.getKeys());
            }
        }

        if (sendCallback != null) {
            sendCallback.onException(e);
        }
    } catch (Exception e1) {
        LOGGER.warn("onExcept fail", e1);
        if (sendCallback != null) {
            sendCallback.onException(e);
        }
    }
}
```

## 队列的健康管理器

cn.webank.defibus.client.impl.producer.MessageQueueHealthManager

```java
private long isoTime = 60 * 1000L; //默认隔离期1min
```

设置此队列的隔离期

cn.webank.defibus.client.impl.producer.MessageQueueHealthManager#markQueueFault

```java
public void markQueueFault(MessageQueue mq) {
    faultMap.put(mq, System.currentTimeMillis() + isoTime);
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("mark queue {} fault for time {}", mq, isoTime);
    }
}
```

标记队列为健康状态，将其从队列中移除

cn.webank.defibus.client.impl.producer.MessageQueueHealthManager#markQueueHealthy

```java
public void markQueueHealthy(MessageQueue mq) {
    if (faultMap.containsKey(mq)) {
        LOGGER.info("mark queue healthy. {}", mq);
        faultMap.remove(mq);
    }
}
```

判断此队列是否仍然在隔离期内

cn.webank.defibus.client.impl.producer.MessageQueueHealthManager#isQueueFault

```java
public boolean isQueueFault(MessageQueue mq) {
    Long isolateUntilWhen = faultMap.get(mq);
    return isolateUntilWhen != null && System.currentTimeMillis() < isolateUntilWhen;
}
```

# Consumer端的隔离

```
Consumer端的消息拉取只由一个消息拉取线程完成。在正常情况下，每次拉消息请求的执行都很快，不会有卡顿。一旦有Broker故障导致PullRequest的执行发生了卡顿，则该Consumer监听的所有Queue都会因拉取请求执行的延迟而出现消息消费延迟。对于RR同步请求的场景，这种是不能够接受的。
```

```
DeFiBus在Consumer拉消息的过程中增加了对拉消息任务的隔离，此处的隔离指的是将疑似有问题的任务隔离到另外的线程中执行，保证拉消息线程能够正常处理其他正常的任务。当发现执行拉消息耗时超过设定的阈值时，将该拉消息任务对应的Broker列入“隔离名单”中，在隔离过期之前，隔离Broker的拉消息请求都转交给另外线程执行，避免阻塞拉消息主线程，从而避免故障的Broker影响健康Broker的消息消费时效。
```

对RocketMq的扩展

```
DeFiBusPullMessageService继承自PullMessageService，重写run方法，添加额外的线程池处理疑似有问题的任务
```

```
DeFiBusClientInstance继承自MQClientInstance，通过反射替换MQClientInstance中的PullMessageService为DeFiBusPullMessageService
```

```java
 DeFiBusPullMessageService deFiBusPullMessageService = new DeFiBusPullMessageService(this);
 ReflectUtil.setSimpleProperty(MQClientInstance.class, this, "pullMessageService",deFiBusPullMessageService);
```

## 启动线程，开始拉取消息

cn.webank.defibus.client.impl.consumer.DeFiBusPullMessageService#run

```java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            PullRequest pullRequest = this.pullRequestQueue.take();
            //拉取消息前判断Broker状态
            this.pullMessageWithHealthyManage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }
    log.info(this.getServiceName() + " service end");
}
```

## Broker健康检测

cn.webank.defibus.client.impl.consumer.DeFiBusPullMessageService#pullMessageWithHealthyManage

```java
private void pullMessageWithHealthyManage(final PullRequest pullRequest) {
  	//判断Broker是否健康
    boolean brokerAvailable = brokerHealthyManager.isBrokerAvailable(pullRequest.getMessageQueue().getBrokerName());
    if (brokerAvailable) { //Broker正常
        pullMessage(pullRequest);
    } else { //Broker在隔离名单中
        runInRetryThread(pullRequest);
    }
}
```

## 正常拉取消息

cn.webank.defibus.client.impl.consumer.DeFiBusPullMessageService#pullMessage

```java
private void pullMessage(final PullRequest pullRequest) {
    final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        long beginPullRequestTime = System.currentTimeMillis();
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        log.debug("begin Pull Message, {}", pullRequest);
        impl.pullMessage(pullRequest);
        long rt = System.currentTimeMillis() - beginPullRequestTime;
      //如果超过隔离阈值，将此Broker加入隔离名单
        if (rt >= brokerHealthyManager.getIsolateThreshold()) { 
          //标记Broker为隔离状态
          brokerHealthyManager.isolateBroker(pullRequest.getMessageQueue().getBrokerName());
        }
    } else {
        log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
    }
}
```

## 隔离线程拉取消息

cn.webank.defibus.client.impl.consumer.DeFiBusPullMessageService#runInRetryThread

```java
private void runInRetryThread(PullRequest pullRequest) {
    try {
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                pullMessage(pullRequest);
            }
        });
    } catch (Exception ex) {
        log.info("execute pull message in retry thread fail.", ex);
        super.executePullRequestLater(pullRequest, 100);
    }
}
```

## Broker健康管理器

```java
private long isolateThreshold = 500; //拉取耗时超过此阈值，进行隔离
private long ISOLATE_TIMEOUT = 5 * 60 * 1000; //隔离时长
```

判断Broker是否正常，超过隔离期就认为正常，从队列中移除

cn.webank.defibus.client.impl.consumer.DeFiBusPullMessageService.BrokerHealthyManager#isBrokerAvailable

```java
public boolean isBrokerAvailable(String brokerName) {
    boolean brokerIsolated = isolatedBroker.containsKey(brokerName);
    if (brokerIsolated) {
        boolean isolatedTimeout = System.currentTimeMillis() - isolatedBroker.get(brokerName) > ISOLATE_TIMEOUT;
        if (isolatedTimeout) { //超过隔离时长，从名单中移除Broker
            removeIsolateBroker(brokerName);
            return true;
        } else {
            return false;
        }
    } else {
        return true;
    }
}
```

Broker设置为隔离状态

cn.webank.defibus.client.impl.consumer.DeFiBusPullMessageService.BrokerHealthyManager#isolateBroker

```java
public void isolateBroker(String brokerName) {
    isolatedBroker.put(brokerName, System.currentTimeMillis());
    if (isolatedBroker.containsKey(brokerName)) {
        log.info("isolate broker for slow pull message success, {}", brokerName);
    }
}
```

# 自动伸缩Queue

cn.webank.defibus.broker.client.AdjustQueueNumStrategy#adjustQueueNumByConsumerCount

```java
private void adjustQueueNumByConsumerCount(String topic, AdjustType scaleType) {
    if (BrokerRole.SLAVE == this.deFiBrokerController.getMessageStoreConfig().getBrokerRole()) {
        log.info("skip adjust queue num in slave.");
        return;
    }
    if (topic.startsWith(MixAll.DLQ_GROUP_TOPIC_PREFIX) || topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
        log.info("skip adjust queue num for topic [{}]", topic);
        return;
    }
    switch (scaleType) {
        case INCREASE_QUEUE_NUM: //伸
            adjustReadQueueNumByConsumerCount(topic, 0, scaleType); //先扩readQueue
            adjustWriteQueueNumByConsumerCount(topic, 10 * 1000, scaleType); //再扩writeQueue
            break;

        case DECREASE_QUEUE_NUM: //缩
            adjustWriteQueueNumByConsumerCount(topic, 0, scaleType); //先缩writeQueue
            long delayTimeMillis = deFiBrokerController.getDeFiBusBrokerConfig().getScaleQueueSizeDelayTimeMinute() * 60 * 1000; //再缩readQueue
            adjustReadQueueNumByConsumerCount(topic, delayTimeMillis, scaleType);
            break;
    }
}
```

## 伸缩readQueue

```java
private void adjustReadQueueNumByConsumerCount(String topic, long delayMills, AdjustType mode) {
    Runnable scaleQueueTask = new Runnable() {
        private int alreadyRetryTimes = 0;
        @Override
        public void run() {
            TopicConfig topicConfig = deFiBrokerController.getTopicConfigManager().getTopicConfigTable().get(topic);
            if (topicConfig != null) {
                synchronized (topicConfig.getTopicName()) {
                    //先获取topic的信息
                    topicConfig = deFiBrokerController.getTopicConfigManager().getTopicConfigTable().get(topic);
                    //调整后的readQueue的大小
                    int adjustReadQueueSize = adjustQueueSizeByMaxConsumerCount(topic);

                    if (AdjustType.INCREASE_QUEUE_NUM == mode && adjustReadQueueSize < topicConfig.getReadQueueNums()) {//扩展后，adjustReadQueueSize应该大于readQueueNums
                        log.info("can not decrease read queue size to {} for [{}], prev: {}, {}", adjustReadQueueSize, topic, topicConfig.getReadQueueNums(), mode);
                        return;
                    }
                    if (AdjustType.DECREASE_QUEUE_NUM == mode && adjustReadQueueSize > topicConfig.getReadQueueNums()) {//缩后，adjustReadQueueSize应该小于ReadQueueNums
                        log.info("can not increase read queue size to {} for [{}], prev: {}, {}", adjustReadQueueSize, topic, topicConfig.getReadQueueNums(), mode);
                        return;
                    }
										//伸缩后的读队列大小不等于之前的读队列大小
                    if (adjustReadQueueSize != topicConfig.getReadQueueNums()) {
                        log.info("try adjust read queue size to {} for [{}], prev: {}, {}", adjustReadQueueSize, topic, topicConfig.getReadQueueNums(), mode);
                        if (adjustReadQueueSize < topicConfig.getWriteQueueNums()) {
                           //小于写队列大小，不合法
                            log.info("adjust read queues to {} for [{}] fail. read queue size can't less than write queue size[{}]. {}",
                                adjustReadQueueSize, topic, topicConfig.getWriteQueueNums(), mode);
                            return;
                        }
                      //判断是否可以调整（所有消息都已被消费）
                        boolean canAdjustReadQueueSize = isCanAdjustReadQueueSize(topic, adjustReadQueueSize);
                        if (canAdjustReadQueueSize) {
                            if (adjustReadQueueSize >= topicConfig.getWriteQueueNums() && adjustReadQueueSize < 1024) {
                                if (mode == AdjustType.INCREASE_QUEUE_NUM && adjustReadQueueSize > 4) {
                                    log.warn("[NOTIFY]auto adjust queues more than 4 for [{}]. {}", topic, mode);
                                }
                              //创建新的TopicConfig
                                TopicConfig topicConfigNew = generateNewTopicConfig(topicConfig, topicConfig.getWriteQueueNums(), adjustReadQueueSize);
//修改topic配置
                              deFiBrokerController.getTopicConfigManager().updateTopicConfig(topicConfigNew);
                              
                                deFiBrokerController.registerBrokerAll(true, false, true);
                              //通知消费者topic信息发生变更
                                notifyWhenTopicConfigChange(topic);
                            } else if (adjustReadQueueSize >= 1024) {
                                log.warn("[NOTIFY]auto adjust queue num is limited to 1024 for [{}]. {}", topic, mode);
                            }
                        } else {
                            if (this.alreadyRetryTimes < deFiBrokerController.getDeFiBusBrokerConfig().getScaleQueueRetryTimesMax()) {
                                log.info("try adjust read queue size to {} for [{}] fail. retry times: [{}]. {}", adjustReadQueueSize, topic, this.alreadyRetryTimes, mode);
                                this.alreadyRetryTimes++;
                                scheduleAdjustQueueSizeTask(this, delayMills, topic, mode);
                                log.info("adjustQueueSizeScheduleExecutor queued: {}", autoScaleQueueSizeExecutorService.getQueue().size());
                            } else {
                                log.warn("try adjust read queue size to {} for [{}] fail. ignore after retry {} times. {}", adjustReadQueueSize, topic, this.alreadyRetryTimes, mode);
                            }
                        }
                    } else {
                        log.info("no need to adjust read queue size for [{}]. now [w:{}/r:{}]. {}", topic, topicConfig.getWriteQueueNums(), topicConfig.getReadQueueNums(), mode);
                    }
                }
            } else {
                log.info("skip adjust read queue size for [{}]. topicConfig is null.", topic);
            }
        }
    };
    this.scheduleAdjustQueueSizeTask(scaleQueueTask, delayMills, topic, mode);
}
```

## 伸缩writeQueue

```java
private void adjustWriteQueueNumByConsumerCount(String topic, long delayMills, AdjustType mode) {
    Runnable scaleTask = new Runnable() {
        @Override
        public void run() {
            TopicConfig topicConfig = deFiBrokerController.getTopicConfigManager().getTopicConfigTable().get(topic);
            if (topicConfig != null) {
                synchronized (topicConfig.getTopicName()) {

                    //query again to ensure it's newest
                    topicConfig = deFiBrokerController.getTopicConfigManager().getTopicConfigTable().get(topic);
                    int adjustWriteQueueSize = adjustQueueSizeByMaxConsumerCount(topic);

                    if (AdjustType.INCREASE_QUEUE_NUM == mode && adjustWriteQueueSize < topicConfig.getWriteQueueNums()) {
                        log.info("can not decrease write queue size to {} for [{}], prev: {}, {}", adjustWriteQueueSize, topic, topicConfig.getWriteQueueNums(), mode);
                        return;
                    }
                    if (AdjustType.DECREASE_QUEUE_NUM == mode && adjustWriteQueueSize > topicConfig.getWriteQueueNums()) {
                        log.info("can not increase write queue size to {} for [{}], prev: {}, {}", adjustWriteQueueSize, topic, topicConfig.getWriteQueueNums(), mode);
                        return;
                    }

                    if (adjustWriteQueueSize != topicConfig.getWriteQueueNums()) {
                        log.info("try adjust write queue size to {} for [{}], prev: {}. {}", adjustWriteQueueSize, topic, topicConfig.getWriteQueueNums(), mode);
                        if (adjustWriteQueueSize >= 0 && adjustWriteQueueSize <= topicConfig.getReadQueueNums()) {
                            TopicConfig topicConfigNew = generateNewTopicConfig(topicConfig, adjustWriteQueueSize, topicConfig.getReadQueueNums());
                            deFiBrokerController.getTopicConfigManager().updateTopicConfig(topicConfigNew);
                            deFiBrokerController.registerBrokerAll(true, false, true);
                            notifyWhenTopicConfigChange(topic);
                        } else {
                            log.info("adjust write queues to {} for [{}] fail. target write queue size can't less than 0 or greater than read queue size[{}]. mode: {}",
                                adjustWriteQueueSize, topic, topicConfig.getReadQueueNums(), mode);
                        }
                    } else {
                        log.info("no need to adjust write queue size for [{}]. now [w:{}/r:{}]. {}", topic, topicConfig.getWriteQueueNums(), topicConfig.getReadQueueNums(), mode);
                    }
                }
            } else {
                log.info("skip adjust write queue size for [{}]. topicConfig is null.", topic);
            }
        }
    };
    this.scheduleAdjustQueueSizeTask(scaleTask, delayMills, topic, mode);
}
```