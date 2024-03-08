# Sofa-Registry源码分析

* [SessionServer](#sessionserver)
  * [注册服务](#注册服务)
    * [向datanode发送注册服务请求](#向datanode发送注册服务请求)
      * [初始化时创建ConsistentHash](#初始化时创建consistenthash)
      * [获取datanode地址](#获取datanode地址)
      * [发送请求](#发送请求)
    * [本地缓存](#本地缓存)
  * [订阅服务](#订阅服务)
    * [发送订阅事件](#发送订阅事件)
      * [SubscriberRegisterFetchTaskListener](#subscriberregisterfetchtasklistener)
      * [SubscriberRegisterFetchTask](#subscriberregisterfetchtask)
      * [DefaultSubscriberRegisterFetchTaskStrategy](#defaultsubscriberregisterfetchtaskstrategy)
    * [发布推送事件](#发布推送事件)
      * [ReceivedDataMultiPushTaskListener](#receiveddatamultipushtasklistener)
      * [DefaultPushTaskMergeProcessor](#defaultpushtaskmergeprocessor)
      * [ReceivedDataMultiPushTaskListener](#receiveddatamultipushtasklistener-1)
      * [ReceivedDataMultiPushTask](#receiveddatamultipushtask)
* [DataServer](#dataserver)
  * [初始化](#初始化)
    * [DataChangeEventCenter](#datachangeeventcenter)
    * [DataChangeHandler](#datachangehandler)
    * [IDataChangeNotifier](#idatachangenotifier)
      * [SessionServerNotifier](#sessionservernotifier)
      * [BackUpNotifier](#backupnotifier)
  * [注册服务](#注册服务-1)
  * [获取服务](#获取服务)
    * [转发请求](#转发请求)
    * [获取服务信息](#获取服务信息)
  * [通知数据同步](#通知数据同步)
* [总结](#总结)


# SessionServer

## 注册服务

com.alipay.sofa.registry.server.session.remoting.handler.PublisherHandler#reply

```java
public Object reply(Channel channel, Object message) throws RemotingException {//接收注册请求
    RegisterResponse result = new RegisterResponse();
    PublisherRegister publisherRegister = (PublisherRegister) message;
    //注册
    publisherHandlerStrategy.handlePublisherRegister(channel, publisherRegister, result);
    return result;
}
```

com.alipay.sofa.registry.server.session.strategy.impl.DefaultPublisherHandlerStrategy#handlePublisherRegister

```java
public void handlePublisherRegister(Channel channel, PublisherRegister publisherRegister,
                                    RegisterResponse registerResponse) {
    try {
        String ip = channel.getRemoteAddress().getAddress().getHostAddress();
        int port = channel.getRemoteAddress().getPort();
        publisherRegister.setIp(ip);
        publisherRegister.setPort(port);

        if (StringUtils.isBlank(publisherRegister.getZone())) {
            publisherRegister.setZone(ValueConstants.DEFAULT_ZONE);
        }

        if (StringUtils.isBlank(publisherRegister.getInstanceId())) {
            publisherRegister.setInstanceId(DEFAULT_INSTANCE_ID);
        }

        Publisher publisher = PublisherConverter.convert(publisherRegister);
        publisher.setProcessId(ip + ":" + port);
        publisher.setSourceAddress(new URL(channel.getRemoteAddress()));
        if (EventTypeConstants.REGISTER.equals(publisherRegister.getEventType())) {
            sessionRegistry.register(publisher);//注册服务
        } else if (EventTypeConstants.UNREGISTER.equals(publisherRegister.getEventType())) {
            sessionRegistry.unRegister(publisher); //下线服务
        }
        registerResponse.setSuccess(true);
        registerResponse.setVersion(publisher.getVersion());
        registerResponse.setRegistId(publisherRegister.getRegistId());
        registerResponse.setMessage("Publisher register success!");
        LOGGER.info("Publisher register success!Type:{} Info:{}",
            publisherRegister.getEventType(), publisher);
    } catch (Exception e) {
        LOGGER.error("Publisher register error!Type {}", publisherRegister.getEventType(), e);
        registerResponse.setSuccess(false);
        registerResponse.setMessage("Publisher register failed!Type:"
                                    + publisherRegister.getEventType());
    }
}
```

com.alipay.sofa.registry.server.session.registry.SessionRegistry#register

```java
public void register(StoreData storeData) {

    //check connect already existed
    checkConnect(storeData);

    switch (storeData.getDataType()) {
        case PUBLISHER: //注册服务
            Publisher publisher = (Publisher) storeData;

            dataNodeService.register(publisher); //向datanode发送注册服务请求

            sessionDataStore.add(publisher); //本地缓存服务信息
 
            sessionRegistryStrategy.afterPublisherRegister(publisher);
            break;
        case SUBSCRIBER: //订阅
            Subscriber subscriber = (Subscriber) storeData;

            sessionInterests.add(subscriber);

            sessionRegistryStrategy.afterSubscriberRegister(subscriber);
            break;
        case WATCHER: //注册watcher
            Watcher watcher = (Watcher) storeData;

            sessionWatchers.add(watcher);

            sessionRegistryStrategy.afterWatcherRegister(watcher);
            break;
        default:
            break;
    }
}
```

### 向datanode发送注册服务请求

com.alipay.sofa.registry.server.session.node.service.DataNodeServiceImpl#register

```java
public void register(final Publisher publisher) {

    try {
				//创建注册服务请求
        Request<PublishDataRequest> publisherRequest = new Request<PublishDataRequest>() {

            private URL url;

            @Override
            public PublishDataRequest getRequestBody() {
                PublishDataRequest publishDataRequest = new PublishDataRequest();
                publishDataRequest.setPublisher(publisher);
                publishDataRequest.setSessionServerProcessId(SessionProcessIdGenerator
                    .getSessionProcessId());
                return publishDataRequest;
            }

            @Override
            public URL getRequestUrl() { //获取dataserver地址
                if (url == null) {
                    url = getUrl(publisher.getDataInfoId());
                }
                return url;
            }
        };
				//发送请求，同步等待响应
        Response response = dataNodeExchanger.request(publisherRequest);

        Object result = response.getResult();
        if (result instanceof CommonResponse) {
            CommonResponse commonResponse = (CommonResponse) result;
            if (!commonResponse.isSuccess()) {
                LOGGER.error(
                    "PublishDataRequest get server response failed!target url:{},message:{}",
                    publisherRequest.getRequestUrl(), commonResponse.getMessage());
                throw new RuntimeException(
                    "PublishDataRequest get server response failed! msg:"
                            + commonResponse.getMessage());
            }
        }
    } catch (RequestException e) {
        LOGGER.error("DataNodeService register new publisher error! " + e.getRequestMessage(),
            e);
        throw new RuntimeException("DataNodeService register new publisher error! "
                                   + e.getRequestMessage(), e);
    }
}
```

#### 初始化时创建ConsistentHash

sessionServer初始化时会获取所有的datanode，DataNodeManager负责监听datanode的变化，重建ConsistentHash

```java
public void updateNodes(NodeChangeResult nodeChangeResult) {
    write.lock();
    try {
        super.updateNodes(nodeChangeResult);
        consistentHash = new ConsistentHash(sessionServerConfig.getNumberOfReplicas(),//副本数，默认1000
            getDataCenterNodes()); //集群中的datanode

    } finally {
        write.unlock();
    }
}
```

com.alipay.sofa.registry.consistency.hash.ConsistentHash#ConsistentHash(com.alipay.sofa.registry.consistency.hash.HashFunction, int, java.util.Collection<T>)

```java
public ConsistentHash(HashFunction hashFunction, int numberOfReplicas, Collection<T> nodes) { //初始化
    this.realNodes = new HashSet<>();
    this.hashFunction = hashFunction;
    this.numberOfReplicas = numberOfReplicas;
    for (T node : nodes) {
        addNode(node);
    }
}
```

com.alipay.sofa.registry.consistency.hash.ConsistentHash#addNode

```java
private void addNode(T node) {
    realNodes.add(node); //真实节点
    for (int i = 0; i < numberOfReplicas; i++) {//创建虚拟节点 
        // 根据nodename+#+i计算hash
        circle.put(hashFunction.hash(node.getNodeName() + SIGN + i), node);
    }
}
```

#### 获取datanode地址

com.alipay.sofa.registry.server.session.node.service.DataNodeServiceImpl#getUrl

```java
private URL getUrl(String dataInfoId) { //根据dataInfoId计算所属的datanode

    Node dataNode = dataNodeManager.getNode(dataInfoId);
    if (dataNode != null) {
        //meta push data node has not port
        String dataIp = dataNode.getNodeUrl().getIpAddress();
        return new URL(dataIp, sessionServerConfig.getDataServerPort());
    }
    return null;
}
```

com.alipay.sofa.registry.server.session.node.DataNodeManager#getNode

```java
public DataNode getNode(String dataInfoId) {
    DataNode dataNode = consistentHash.getNodeFor(dataInfoId);
    if (dataNode == null) {
        LOGGER.error("calculate data node error!,dataInfoId={}", dataInfoId);
        throw new RuntimeException("DataNodeManager calculate data node error!,dataInfoId="
                                   + dataInfoId);
    }
    return dataNode;
}
```

com.alipay.sofa.registry.consistency.hash.ConsistentHash#getNodeFor

```java
public T getNodeFor(Object key) {
    if (circle.isEmpty()) {
        return null;
    }
    int hash = hashFunction.hash(key);
    T node = circle.get(hash); //返回等于key的node

    if (node == null) {
        // inexact match -- find the next value in the circle
        SortedMap<Integer, T> tailMap = circle.tailMap(hash); //返回比所给key大或等于的map部分
        hash = tailMap.isEmpty() ? circle.firstKey() : tailMap.firstKey(); //返回当前map中最小的key
        node = circle.get(hash); //获取node
    }
    return node;
}
```

#### 发送请求

com.alipay.sofa.registry.server.session.remoting.DataNodeExchanger#request

```java
public Response request(Request request) throws RequestException {

    Response response;
    URL url = request.getRequestUrl();
    try {
        //获取dataserver的Client，Client维护了与每个datanode的连接
        Client sessionClient = boltExchange.getClient(Exchange.DATA_SERVER_TYPE);

        if (sessionClient == null) {
            LOGGER.warn(
                    "DataNode Exchanger get dataServer connection {} error! Connection can not be null or disconnected!",
                    url);
            //first start session maybe case sessionClient null,try to auto connect
            sessionClient = boltExchange.connect(Exchange.DATA_SERVER_TYPE, url,
                    dataClientHandlers.toArray(new ChannelHandler[dataClientHandlers.size()]));
        }
        //获取url对应的channel
        Channel channel = sessionClient.getChannel(url);
        if (channel == null) {
            channel = sessionClient.connect(url); //建立连接
        }
        EXCHANGE_LOGGER.info("DataNode Exchanger request={},url={}", request.getRequestBody(), url);
				//同步发送请求
        final Object result = sessionClient.sendSync(channel, request.getRequestBody(),
                sessionServerConfig.getDataNodeExchangeTimeOut());
        if (result == null) {
            LOGGER.error("DataNode Exchanger request data get null result!Request url:" + url);
            throw new RequestException("DataNode Exchanger request data get null result!",
                    request);
        }
        response = () -> result;
    } catch (Exception e) {
        LOGGER.error("DataNode Exchanger request data error!request={},url={}", request.getRequestBody(), url, e);
        throw new RequestException("DataNode Exchanger request data error!Request url:" + url,
                request, e);
    }

    return response;
}
```

### 本地缓存

SessionServer对服务进行保存，避免每次都请求DataServer。为保持SessionServer和DataServer中注册信息的一致性，当DataServer发现有变更时，将变更事件推送给与此连接的SessionServer，SessionServer主动向DataServer拉取最新的注册信息

```java
private Map<String/*dataInfoId*/, Map<String/*registerId*/, Publisher>> registry      = new ConcurrentHashMap<>();
```

com.alipay.sofa.registry.server.session.store.SessionDataStore#add

```java
public void add(Publisher publisher) {

    write.lock();
    try {
        Map<String, Publisher> publishers = registry.get(publisher.getDataInfoId());

        if (publishers == null) {
            ConcurrentHashMap<String, Publisher> newmap = new ConcurrentHashMap<>();
            publishers = registry.putIfAbsent(publisher.getDataInfoId(), newmap);
            if (publishers == null) {
                publishers = newmap;
            }
        }

        Publisher existingPublisher = publishers.get(publisher.getRegisterId());

        if (existingPublisher != null) {

            if (existingPublisher.getVersion() != null) {
                long oldVersion = existingPublisher.getVersion();
                Long newVersion = publisher.getVersion();
                if (newVersion == null) {
                    LOGGER.error("There is publisher input version can't be null!");
                    return;
                } else if (oldVersion > newVersion) {
                    LOGGER
                        .warn(
                            "There is publisher already exists,but old version {} higher than input {},it will not be overwrite! {}",
                            oldVersion, newVersion, existingPublisher);
                    return;
                } else if (oldVersion == newVersion) {
                    Long newTime = publisher.getRegisterTimestamp();
                    long oldTime = existingPublisher.getRegisterTimestamp();
                    if (newTime == null) {
                        LOGGER
                            .error("There is publisher input Register Timestamp can not be null!");
                        return;
                    }
                    if (oldTime > newTime) {
                        LOGGER
                            .warn(
                                "There is publisher already exists,but old timestamp {} higher than input {},it will not be overwrite! {}",
                                oldTime, newTime, existingPublisher);
                        return;
                    }
                }
            }
            LOGGER
                .warn(
                    "There is publisher already exists,version:{},it will be overwrite!Input version:{},info:{}",
                    existingPublisher.getVersion(), publisher.getVersion(), existingPublisher);
        }
        publishers.put(publisher.getRegisterId(), publisher);

        addIndex(publisher);

    } finally {
        write.unlock();
    }
}
```

```java
private Map<String/*connectId*/, Map<String/*registerId*/, Publisher>>  connectIndex  = new ConcurrentHashMap<>();//每个服务实例所注册服务
```

com.alipay.sofa.registry.server.session.store.SessionDataStore#addIndex

```java
private void addIndex(Publisher publisher) {

    addConnectIndex(publisher);
}
```

```java
private void addConnectIndex(Publisher publisher) {

    String connectId = publisher.getSourceAddress().getAddressString();
    Map<String/*registerId*/, Publisher> publisherMap = connectIndex.get(connectId);
    if (publisherMap == null) {
        Map<String/*registerId*/, Publisher> newPublisherMap = new ConcurrentHashMap<>();
        publisherMap = connectIndex.putIfAbsent(connectId, newPublisherMap);
        if (publisherMap == null) {
            publisherMap = newPublisherMap;
        }
    }

    publisherMap.put(publisher.getRegisterId(), publisher);
}
```

## 订阅服务

com.alipay.sofa.registry.server.session.strategy.impl.DefaultSessionRegistryStrategy#afterSubscriberRegister

```java
public void afterSubscriberRegister(Subscriber subscriber) { //发布订阅事件
    if (!sessionServerConfig.isStopPushSwitch()) {
        //拉取数据，推送给客户端
        TaskEvent taskEvent = new TaskEvent(subscriber,
            TaskEvent.TaskType.SUBSCRIBER_REGISTER_FETCH_TASK);
        taskLogger.info("send " + taskEvent.getTaskType() + " taskEvent:{}", taskEvent);
        taskListenerManager.sendTaskEvent(taskEvent);
    }
}
```

### 发送订阅事件

#### SubscriberRegisterFetchTaskListener

com.alipay.sofa.registry.server.session.listener.SubscriberRegisterFetchTaskListener#handleEvent

```java
public void handleEvent(TaskEvent event) {

    SessionTask subscriberRegisterFetchTask = new SubscriberRegisterFetchTask(
        sessionServerConfig, taskListenerManager, dataNodeService, sessionCacheService,
        subscriberRegisterFetchTaskStrategy);

    subscriberRegisterFetchTask.setTaskEvent(event);
		//发布SubscriberRegisterFetchTask
    getSingleTaskDispatcher().dispatch(subscriberRegisterFetchTask.getTaskId(),
        subscriberRegisterFetchTask, subscriberRegisterFetchTask.getExpiryTime());
}
```

#### SubscriberRegisterFetchTask

com.alipay.sofa.registry.server.session.scheduler.task.SubscriberRegisterFetchTask#execute

```java
public void execute() {
    subscriberRegisterFetchTaskStrategy.doSubscriberRegisterFetchTask(sessionServerConfig,
        taskListenerManager, dataNodeService, sessionCacheService, subscriber);
}
```

#### DefaultSubscriberRegisterFetchTaskStrategy

com.alipay.sofa.registry.server.session.strategy.impl.DefaultSubscriberRegisterFetchTaskStrategy#doSubscriberRegisterFetchTask

```java
public void doSubscriberRegisterFetchTask(SessionServerConfig sessionServerConfig,
                                          TaskListenerManager taskListenerManager,
                                          DataNodeService dataNodeService,
                                          CacheService sessionCacheService,
                                          Subscriber subscriber) {
    if (subscriber == null) {
        throw new IllegalArgumentException("Subscriber can not be null!");
    }
		
    List<String> subscriberRegisterIdList = Collections.singletonList(subscriber
        .getRegisterId());

    boolean isOldVersion = !BaseInfo.ClientVersion.StoreData.equals(subscriber
        .getClientVersion());
	  //根据dataInfoId从datanode获取所有对应的服务实例
    Map<String/*datacenter*/, Datum> datumMap = dataNodeService.fetchGlobal(subscriber
        .getDataInfoId());
   //推送
    if (!isOldVersion) {
        fireReceivedDataPushTaskCloud(datumMap, subscriberRegisterIdList, subscriber,
            taskListenerManager);
    } else {
        fireUserDataPushTaskCloud(datumMap, subscriber, taskListenerManager);
    }
}
```

### 发布推送事件

com.alipay.sofa.registry.server.session.strategy.impl.DefaultSubscriberRegisterFetchTaskStrategy#firePush

```java
private void firePush(ReceivedData receivedData, Subscriber subscriber,
                      TaskListenerManager taskListenerManager) {
    //trigger push to client node
    Map<ReceivedData, URL> parameter = new HashMap<>();
    parameter.put(receivedData, subscriber.getSourceAddress());
    TaskEvent taskEvent = new TaskEvent(parameter,
        TaskEvent.TaskType.RECEIVED_DATA_MULTI_PUSH_TASK);
    taskLogger.info("send {} taskURL:{},taskScope:{}", taskEvent.getTaskType(),
        subscriber.getSourceAddress(), receivedData.getScope());
    taskListenerManager.sendTaskEvent(taskEvent);
}
```

#### ReceivedDataMultiPushTaskListener

com.alipay.sofa.registry.server.session.listener.ReceivedDataMultiPushTaskListener#handleEvent

```java
public void handleEvent(TaskEvent event) {
    receiveDataTaskMergeProcessorStrategy.handleEvent(event);
}
```

#### DefaultPushTaskMergeProcessor

com.alipay.sofa.registry.server.session.strategy.impl.DefaultPushTaskMergeProcessor#handleEvent

```java
public void handleEvent(TaskEvent event) {
    pushTaskSender.executePushAsync(event);
}
```

#### ReceivedDataMultiPushTaskListener

com.alipay.sofa.registry.server.session.listener.ReceivedDataMultiPushTaskListener#executePushAsync

```java
public void executePushAsync(TaskEvent event) {

    SessionTask receivedDataMultiPushTask = new ReceivedDataMultiPushTask(sessionServerConfig, clientNodeService,
            executorManager, boltExchange, receivedDataMultiPushTaskStrategy,asyncHashedWheelTimer);
    receivedDataMultiPushTask.setTaskEvent(event);

    executorManager.getPushTaskExecutor()
            .execute(() -> clientNodeSingleTaskProcessor.process(receivedDataMultiPushTask));
}
```

#### ReceivedDataMultiPushTask

com.alipay.sofa.registry.server.session.scheduler.task.ReceivedDataMultiPushTask#execute

```java
public void execute() {

    if (sessionServerConfig.isStopPushSwitch()) {
        LOGGER
            .info(
                "Stop Push ReceivedData with switch on! dataId: {},group: {},Instance: {}, url: {}",
                receivedData.getDataId(), receivedData.getGroup(),
                receivedData.getInstanceId(), url);
        return;
    }

    Object receivedDataPush = receivedDataMultiPushTaskStrategy.convert2PushData(receivedData,
        url);

    CallbackHandler callbackHandler = new CallbackHandler() {
        @Override
        public void onCallback(Channel channel, Object message) {
            LOGGER.info(
                "Push ReceivedData success! dataId:{},group:{},Instance:{},version:{},url: {}",
                receivedData.getDataId(), receivedData.getGroup(),
                receivedData.getInstanceId(), receivedData.getVersion(), url);

            if (taskClosure != null) {
                confirmCallBack(true);
            }
        }

        @Override
        public void onException(Channel channel, Throwable exception) {
            LOGGER.error(
                "Push ReceivedData error! dataId:{},group:{},Instance:{},version:{},url: {}",
                receivedData.getDataId(), receivedData.getGroup(),
                receivedData.getInstanceId(), receivedData.getVersion(), url, exception);

            if (taskClosure != null) {
                confirmCallBack(false);
                throw new RuntimeException("Push ReceivedData got exception from callback!");
            } else {
                retrySendReceiveData(new PushDataRetryRequest(receivedDataPush, url));
            }
        }
    };

    try {
      //获取客户端连接，推送服务实例数据
        clientNodeService.pushWithCallback(receivedDataPush, url, callbackHandler);
    } catch (Exception e) {
        if (taskClosure != null) {
            confirmCallBack(false);
            throw e;
        } else {
            retrySendReceiveData(new PushDataRetryRequest(receivedDataPush, url));
        }
    }
}
```

# DataServer

## 初始化

### DataChangeEventCenter

存放sessionServer的pub请求

com.alipay.sofa.registry.server.data.change.event.DataChangeEventCenter#init

```java
public void init(DataServerConfig config) {
    if (isInited.compareAndSet(false, true)) {
        queueCount = config.getQueueCount();
     		//创建DataChangeEventQueue，存放DataChangeEvent
        dataChangeEventQueues = new DataChangeEventQueue[queueCount];
        for (int idx = 0; idx < queueCount; idx++) {
            dataChangeEventQueues[idx] = new DataChangeEventQueue(idx, config);
            dataChangeEventQueues[idx].start();
        }
    }
}
```

com.alipay.sofa.registry.server.data.change.event.DataChangeEventQueue#start

```java
public void start() {
    LOGGER.info("[{}] begin start DataChangeEventQueue", getName());
    Executor executor = ExecutorFactory.newSingleThreadExecutor(
            String.format("%s_%s", DataChangeEventQueue.class.getSimpleName(), getName()));
    executor.execute(() -> {
        while (true) {
            try {
                IDataChangeEvent event = eventQueue.take(); //eventQueue存放sessionServer发送的publish请求
                DataChangeScopeEnum scope = event.getScope();
                if (scope == DataChangeScopeEnum.DATUM) {
                    DataChangeEvent dataChangeEvent = (DataChangeEvent) event;
                    handleDatum(dataChangeEvent.getChangeType(),
                            dataChangeEvent.getSourceType(), dataChangeEvent.getDatum());
                } else if (scope == DataChangeScopeEnum.CLIENT) {
                    handleHost((ClientChangeEvent) event);
                }
            } catch (Throwable e) {
                LOGGER.error("[{}] handle change event failed", getName(), e);
            }
        }
    });
    LOGGER.info("[{}] start DataChangeEventQueue success", getName());
}
```

com.alipay.sofa.registry.server.data.change.event.DataChangeEventQueue#handleDatum

```java
private void handleDatum(DataChangeTypeEnum changeType, DataSourceTypeEnum sourceType,
                         Datum targetDatum) {
    lock.lock();
    try {
        //get changed datum
        ChangeData changeData = getChangeData(targetDatum.getDataCenter(),
            targetDatum.getDataInfoId(), sourceType, changeType);
        Datum cacheDatum = changeData.getDatum();
        if (changeType == DataChangeTypeEnum.COVER || cacheDatum == null) { //覆盖或者新注册
            changeData.setDatum(targetDatum);
        } else { //发生变更
            Map<String, Publisher> targetPubMap = targetDatum.getPubMap();
            Map<String, Publisher> cachePubMap = cacheDatum.getPubMap();
            for (Publisher pub : targetPubMap.values()) {
                String registerId = pub.getRegisterId();
                Publisher cachePub = cachePubMap.get(registerId);
                if (cachePub != null) {
                    // if the registerTimestamp of cachePub is greater than the registerTimestamp of pub, it means
                    // that pub is not the newest data, should be ignored
                    if (pub.getRegisterTimestamp() < cachePub.getRegisterTimestamp()) {
                        continue;
                    }
                    // if pub and cachePub both are publisher, and sourceAddress of both are equal,
                    // and version of cachePub is greater than version of pub, should be ignored
                    if (!(pub instanceof UnPublisher) && !(cachePub instanceof UnPublisher)
                        && pub.getSourceAddress().equals(cachePub.getSourceAddress())
                        && cachePub.getVersion() >= pub.getVersion()) {
                        continue;
                    }
                }
                cachePubMap.put(registerId, pub);
                cacheDatum.setVersion(targetDatum.getVersion());
            }
        }
    } finally {
        lock.unlock();
    }
}
```

com.alipay.sofa.registry.server.data.change.event.DataChangeEventQueue#getChangeData

```java
private ChangeData getChangeData(String dataCenter, String dataInfoId,
                                 DataSourceTypeEnum sourceType, DataChangeTypeEnum changeType) {
    //CHANGE_DATA_MAP:<dataCenter:<dataInfoId,ChangeData>>
    Map<String, ChangeData> map = CHANGE_DATA_MAP.get(dataCenter);
    if (map == null) {
        Map<String, ChangeData> newMap = new ConcurrentHashMap<>();
        map = CHANGE_DATA_MAP.putIfAbsent(dataCenter, newMap);
        if (map == null) {
            map = newMap;
        }
    }

    ChangeData changeData = map.get(dataInfoId);
    if (changeData == null) {
        ChangeData newChangeData = new ChangeData(null, this.notifyIntervalMs, sourceType,
            changeType);
        changeData = map.putIfAbsent(dataInfoId, newChangeData);
        if (changeData == null) {
            changeData = newChangeData;
        }
        CHANGE_QUEUE.put(changeData); //CHANGE_QUEUE存放新变更的ChangeData
    }
    return changeData;
}
```

### DataChangeHandler

监听变更的服务信息，通知sessionServer和DataServer

com.alipay.sofa.registry.server.data.change.DataChangeHandler#afterPropertiesSet

```java
public void afterPropertiesSet() {
    //init DataChangeEventCenter
    dataChangeEventCenter.init(dataServerBootstrapConfig);
    start();
}
```

com.alipay.sofa.registry.server.data.change.DataChangeHandler#start

```java
public void start() { 
    DataChangeEventQueue[] queues = dataChangeEventCenter.getQueues();
    int queueCount = queues.length;
    Executor executor = ExecutorFactory.newFixedThreadPool(queueCount,
            DataChangeHandler.class.getSimpleName());
    Executor notifyExecutor = ExecutorFactory.newFixedThreadPool(
            dataServerBootstrapConfig.getQueueCount() * 5, this.getClass().getSimpleName());
    for (int idx = 0; idx < queueCount; idx++) {
        final DataChangeEventQueue dataChangeEventQueue = queues[idx];
        final String name = dataChangeEventQueue.getName();
        LOGGER.info("[DataChangeHandler] begin to notify datum in queue:{}", name);
        executor.execute(() -> {
            while (true) {
                try {
                  	//获取变更的数据
                    final ChangeData changeData = dataChangeEventQueue.take();
                    notifyExecutor.execute(new ChangeNotifier(changeData, name));
                } catch (Throwable e) {
                    LOGGER.error("[DataChangeHandler][{}] notify scheduler error", name, e);
                }
            }
        });
        LOGGER.info("[DataChangeHandler] notify datum in queue:{} success", name);
    }
}
```

com.alipay.sofa.registry.server.data.change.DataChangeHandler.ChangeNotifier#run

```java
public void run() {
    Datum datum = changeData.getDatum();
    String dataCenter = datum.getDataCenter();
    String dataInfoId = datum.getDataInfoId();
    long version = datum.getVersion();
    DataSourceTypeEnum sourceType = changeData.getSourceType();
    DataChangeTypeEnum changeType = changeData.getChangeType();
    try {
        if (sourceType == DataSourceTypeEnum.CLEAN) {
            if (DatumCache.cleanDatum(dataCenter, dataInfoId)) {
                LOGGER
                    .info(
                        "[DataChangeHandler][{}] clean datum, dataCenter={}, dataInfoId={}, version={},sourceType={}, changeType={}",
                        name, dataCenter, dataInfoId, version, sourceType, changeType);
            }

        } else {
            Long lastVersion = null;

            if (sourceType == DataSourceTypeEnum.PUB_TEMP) {
                notifyTempPub(datum, sourceType, changeType);
                return;
            }
						//存放服务信息
            MergeResult mergeResult = DatumCache.putDatum(changeType, datum);
            lastVersion = mergeResult.getLastVersion();

            if (lastVersion != null
                && lastVersion.longValue() == DatumCache.ERROR_DATUM_VERSION) {
                LOGGER
                    .error(
                        "[DataChangeHandler][{}] first put unPub datum into cache error, dataCenter={}, dataInfoId={}, version={}, sourceType={},isContainsUnPub={}",
                        name, dataCenter, dataInfoId, version, sourceType,
                        datum.isContainsUnPub());
                return;
            }

            boolean changeFlag = mergeResult.isChangeFlag();

            LOGGER
                .info(
                    "[DataChangeHandler][{}] datum handle,datum={},dataCenter={}, dataInfoId={}, version={}, lastVersion={}, sourceType={}, changeType={},changeFlag={},isContainsUnPub={}",
                    name, datum.hashCode(), dataCenter, dataInfoId, version, lastVersion,
                    sourceType, changeType, changeFlag, datum.isContainsUnPub());
            //第一次注册服务信息或者版本发生了变更
            if (lastVersion == null || version != lastVersion) {
                if (changeFlag) { //发生了变更
                    for (IDataChangeNotifier notifier : dataChangeNotifiers) {
                        if (notifier.getSuitableSource().contains(sourceType)) {
                            notifier.notify(datum, lastVersion); //推送给订阅的sessionServer/同步至其他dataServer
                        }
                    }
                }
            }
        }
    } catch (Exception e) {
        LOGGER
            .error(
                "[DataChangeHandler][{}] put datum into cache error, dataCenter={}, dataInfoId={}, version={}, sourceType={},isContainsUnPub={}",
                name, dataCenter, dataInfoId, version, sourceType, datum.isContainsUnPub(),
                e);
    }

}
```

### IDataChangeNotifier

#### SessionServerNotifier

com.alipay.sofa.registry.server.data.change.notify.SessionServerNotifier#notify

```java
public void notify(Datum datum, Long lastVersion) {
    DataChangeRequest request = new DataChangeRequest(datum.getDataInfoId(),
        datum.getDataCenter(), datum.getVersion());
    //管理sessionServer的连接
    List<Connection> connections = sessionServerConnectionFactory.getConnections();
    for (Connection connection : connections) {
        doNotify(new NotifyCallback(connection, request));
    }
}
```

com.alipay.sofa.registry.server.data.change.notify.SessionServerNotifier#doNotify

```java
private void doNotify(NotifyCallback notifyCallback) { //通知sessionServer服务发生了变更
    Connection connection = notifyCallback.connection;
    DataChangeRequest request = notifyCallback.request;
    try {
        //check connection active
        if (!connection.isFine()) {
            if (LOGGER.isInfoEnabled()) {
                LOGGER
                    .info(String
                        .format(
                            "connection from sessionserver(%s) is not fine, so ignore notify, retryTimes=%s,request=%s",
                            connection.getRemoteAddress(), notifyCallback.retryTimes, request));
            }
            return;
        }
        Server sessionServer = boltExchange.getServer(dataServerBootstrapConfig.getPort());
        sessionServer.sendCallback(sessionServer.getChannel(connection.getRemoteAddress()),
            request, notifyCallback, dataServerBootstrapConfig.getRpcTimeout());
    } catch (Exception e) {
        LOGGER.error(String.format(
            "invokeWithCallback failed: sessionserver(%s),retryTimes=%s, request=%s",
            connection.getRemoteAddress(), notifyCallback.retryTimes, request), e);
        onFailed(notifyCallback);
    }
}
```

#### BackUpNotifier

com.alipay.sofa.registry.server.data.change.notify.BackUpNotifier#notify

```java
public void notify(Datum datum, Long lastVersion) { //通知dataNode服务发生了变更
    syncDataService.appendOperator(new Operator(datum.getVersion(), lastVersion, datum,
        DataSourceTypeEnum.BACKUP));
}
```

com.alipay.sofa.registry.server.data.datasync.sync.SyncDataServiceImpl#appendOperator

```java
public void appendOperator(Operator operator) {
    AcceptorStore acceptorStore = StoreServiceFactory.getStoreService(operator.getSourceType()
        .toString());
    if (acceptorStore != null) {
        acceptorStore.addOperator(operator);
    } else {
        LOGGER.error("Can't find acceptor store type {} to Append operator!",
            operator.getSourceType());
        throw new RuntimeException("Can't find acceptor store to Append operator!");
    }
}
```

com.alipay.sofa.registry.server.data.datasync.sync.AbstractAcceptorStore#addOperator

```java
public void addOperator(Operator operator) {

    Datum datum = operator.getDatum();
    String dataCenter = datum.getDataCenter();
    String dataInfoId = datum.getDataInfoId();
    try {
      //<dataCenter:<dataInfoId,Acceptor>>
        Map<String/*dataInfoId*/, Acceptor> acceptorMap = acceptors.get(dataCenter);
        if (acceptorMap == null) {
            Map<String/*dataInfoId*/, Acceptor> newMap = new ConcurrentHashMap<>();
            acceptorMap = acceptors.putIfAbsent(dataCenter, newMap);
            if (acceptorMap == null) {
                acceptorMap = newMap;
            }
        }

        Acceptor existAcceptor = acceptorMap.get(dataInfoId);
        if (existAcceptor == null) {
            Acceptor newAcceptor = new Acceptor(DEFAULT_MAX_BUFFER_SIZE, dataInfoId, dataCenter);
            existAcceptor = acceptorMap.putIfAbsent(dataInfoId, newAcceptor);
            if (existAcceptor == null) {
                existAcceptor = newAcceptor;
            }
        }
        //追加operator
        existAcceptor.appendOperator(operator);
        //put cache
        putCache(existAcceptor);
    } catch (Exception e) {
        LOGGER.error(getLogByClass("Append Operator error!"), e);
        throw new RuntimeException("Append Operator error!", e);
    }
}
```

com.alipay.sofa.registry.server.data.datasync.sync.AbstractAcceptorStore#putCache

```java
private void putCache(Acceptor acceptor) {

    String dataCenter = acceptor.getDataCenter();
    String dataInfoId = acceptor.getDataInfoId();

    try {
      	//notifyAcceptorsCache存放发生变更的dataInfoId
        Map<String/*dataInfoId*/, Acceptor> acceptorMap = notifyAcceptorsCache.get(dataCenter);
        if (acceptorMap == null) {
            Map<String/*dataInfoId*/, Acceptor> newMap = new ConcurrentHashMap<>();
            acceptorMap = notifyAcceptorsCache.putIfAbsent(dataCenter, newMap);
            if (acceptorMap == null) {
                acceptorMap = newMap;
            }
        }
        Acceptor existAcceptor = acceptorMap.putIfAbsent(dataInfoId, acceptor);
        if (existAcceptor == null) {
            addQueue(acceptor); //添加到延迟队列，每个dataInfoId对应的Acceptor只会添加一次
        }
    } catch (Exception e) {
        LOGGER.error(getLogByClass("Operator push to delay cache error!"), e);
        throw new RuntimeException("Operator push to delay cache error!", e);
    }
}
```

com.alipay.sofa.registry.server.data.datasync.sync.AbstractAcceptorStore#addQueue

```java
private void addQueue(Acceptor acceptor) {
    delayQueue.put(new DelayItem(acceptor, DEFAULT_DELAY_TIMEOUT));
}
```

com.alipay.sofa.registry.server.data.datasync.sync.AbstractAcceptorStore#changeDataCheck

```java
public void changeDataCheck() { //定时同步变更请求到DataNode
    while (true) {
        try {
            DelayItem<Acceptor> delayItem = delayQueue.take();
            Acceptor acceptor = delayItem.getItem();
            removeCache(acceptor); // compare and remove
        } catch (InterruptedException e) {
            break;
        }
    }
}
```

com.alipay.sofa.registry.server.data.datasync.sync.AbstractAcceptorStore#removeCache

```java
private void removeCache(Acceptor acceptor) {
    String dataCenter = acceptor.getDataCenter();
    String dataInfoId = acceptor.getDataInfoId();

    try {
        Map<String/*dataInfoId*/, Acceptor> acceptorMap = notifyAcceptorsCache.get(dataCenter);
        if (acceptorMap != null) {
            boolean result = acceptorMap.remove(dataInfoId, acceptor);
            if (result) {
                //data change notify
                notifyChange(acceptor);
            }
        }
    } catch (Exception e) {
        LOGGER.error(getLogByClass("Operator remove from delay cache error!"), e);
        throw new RuntimeException("Operator remove from delay cache error!", e);
    }
}
```

com.alipay.sofa.registry.server.data.datasync.sync.AbstractAcceptorStore#notifyChange

```java
private void notifyChange(Acceptor acceptor) {

    Long lastVersion = acceptor.getLastVersion();

    //may be delete by expired
    if (lastVersion == null) {
        LOGGER
            .warn(getLogByClass("There is not data in acceptor queue!maybe has been expired!"));
        lastVersion = 0L;
    }

    if (LOGGER.isDebugEnabled()) {
        acceptor.printInfo();
    }
		//同步请求
    NotifyDataSyncRequest request = new NotifyDataSyncRequest(acceptor.getDataInfoId(),
        acceptor.getDataCenter(), lastVersion, getType());
		//根据dataInfoId计算DataServer
    List<String> targetDataIps = getTargetDataIp(acceptor.getDataInfoId());
    for (String targetDataIp : targetDataIps) {

        if (DataServerConfig.IP.equals(targetDataIp)) {
            continue;
        }

        Connection connection = dataServerConnectionFactory.getConnection(targetDataIp);
        if (connection == null) {
            LOGGER.error(getLogByClass(String.format(
                "Can not get notify data server connection!ip: %s", targetDataIp)));
            continue;
        }
        LOGGER.info(getLogByClass("Notify data server {} change data {} to sync"),
            connection.getRemoteIP(), request);
      //发送NotifyDataSyncRequest
        for (int tryCount = 0; tryCount < NOTIFY_RETRY; tryCount++) { 
            try {
                Server syncServer = boltExchange.getServer(dataServerBootstrapConfig
                    .getSyncDataPort());
                syncServer.sendSync(syncServer.getChannel(connection.getRemoteAddress()),
                    request, 1000);
                break;
            } catch (Exception e) {
                LOGGER.error(getLogByClass(String.format(
                    "Notify data server %s failed, NotifyDataSyncRequest:%s", targetDataIp,
                    request)), e);
            }
        }
    }
}
```



## 注册服务

com.alipay.sofa.registry.server.data.remoting.sessionserver.handler.PublishDataHandler#doHandle

```java
public Object doHandle(Channel channel, PublishDataRequest request) {
    Publisher publisher = Publisher.processPublisher(request.getPublisher());
    if (forwardService.needForward(publisher.getDataInfoId())) {
        LOGGER.warn("[forward] Publish request refused, request: {}", request);
        CommonResponse response = new CommonResponse();
        response.setSuccess(false);
        response.setMessage("Request refused, Server status is not working");
        return response;
    }

    dataChangeEventCenter.onChange(publisher, dataServerConfig.getLocalDataCenter());
    if (publisher.getPublishType() != PublishType.TEMPORARY) {
        sessionServerConnectionFactory.registerClient(request.getSessionServerProcessId(),
            publisher.getSourceAddress().getAddressString());
    }

    return CommonResponse.buildSuccessResponse();
}
```

com.alipay.sofa.registry.server.data.change.event.DataChangeEventCenter#onChange

```java
public void onChange(Publisher publisher, String dataCenter) {
    //根据dataInfoId计算idx，所属的dataChangeEventQueues下标
    int idx = hash(publisher.getDataInfoId());
    Datum datum = new Datum(publisher, dataCenter);
    if (publisher instanceof UnPublisher) {
        datum.setContainsUnPub(true);
    }
  	//存放至对应的dataChangeEventQueue
    if (publisher.getPublishType() != PublishType.TEMPORARY) {
        dataChangeEventQueues[idx].onChange(new DataChangeEvent(DataChangeTypeEnum.MERGE,
            DataSourceTypeEnum.PUB, datum));
    } else {
        dataChangeEventQueues[idx].onChange(new DataChangeEvent(DataChangeTypeEnum.MERGE,
            DataSourceTypeEnum.PUB_TEMP, datum));
    }
}
```

## 获取服务

com.alipay.sofa.registry.server.data.remoting.sessionserver.handler.GetDataHandler#doHandle

```java
public Object doHandle(Channel channel, GetDataRequest request) {
    String dataInfoId = request.getDataInfoId();
    if (forwardService.needForward(dataInfoId)) { //当前节点不正常
        try {
          	//转发请求到其他dataServer
            LOGGER.warn("[forward] Get data request forward, request: {}", request);
            return forwardService.forwardRequest(dataInfoId, request);
        } catch (Exception e) {
            LOGGER.error("[forward] getData request error, request: {}", request, e);
            GenericResponse response = new GenericResponse();
            response.setSuccess(false);
            response.setMessage(e.getMessage());
            return response;
        }
    }

    return new GenericResponse<Map<String, Datum>>().fillSucceed(DatumCache
        .getDatumGroupByDataCenter(request.getDataCenter(), dataInfoId));
}
```

### 转发请求

com.alipay.sofa.registry.server.data.remoting.sessionserver.forward.ForwardServiceImpl#forwardRequest

```java
public Object forwardRequest(String dataInfoId, Object request) throws RemotingException {
    try {
        //获取存储节点
        List<DataServerNode> dataServerNodes = DataServerNodeFactory.computeDataServerNodes(
            dataServerBootstrapConfig.getLocalDataCenter(), dataInfoId,
            dataServerBootstrapConfig.getStoreNodes());

        if (null == dataServerNodes || dataServerNodes.size() <= 0) {
            throw new RuntimeException("No available server to forward");
        }

        boolean next = false;
        String localIp = NetUtil.getLocalAddress().getHostAddress();
        //选择dataServer转发请求
        DataServerNode nextNode = null;
        for (DataServerNode dataServerNode : dataServerNodes) {
            if (next) {
                nextNode = dataServerNode;
                break;
            }
            if (null != localIp && localIp.equals(dataServerNode.getIp())) {
                next = true;
            }
        }

        if (null == nextNode || null == nextNode.getConnection()) {
            throw new RuntimeException("No available connection to forward");
        }

        LOGGER.info("[forward] target: {}, dataInfoId: {}, request: {}, allNodes: {}",
            nextNode.getIp(), dataInfoId, request, dataServerNodes);

  			//转发请求
        final DataServerNode finalNextNode = nextNode;
        return dataNodeExchanger.request(new Request() {
            @Override
            public Object getRequestBody() {
                return request;
            }

            @Override
            public URL getRequestUrl() {
                return new URL(finalNextNode.getConnection().getRemoteIP(), finalNextNode
                    .getConnection().getRemotePort());
            }
        }).getResult();
    } catch (Exception e) {
        throw new RemotingException("ForwardServiceImpl forwardRequest error", e);
    }
}
```

com.alipay.sofa.registry.server.data.remoting.dataserver.DataServerNodeFactory#computeDataServerNodes

```java
public static List<DataServerNode> computeDataServerNodes(String dataCenter, String dataInfoId,
                                                          int backupNodes) {
  	//根据dataCenter获取ConsistentHash，每个数据中心都有对应的ConsistentHash
    ConsistentHash<DataServerNode> consistentHash = CONSISTENT_HASH_MAP.get(dataCenter);
    if (consistentHash != null) { //获取存储节点,副本数默认3
        return CONSISTENT_HASH_MAP.get(dataCenter).getNUniqueNodesFor(dataInfoId, backupNodes);
    }
    return null;
}
```

### 获取服务信息

com.alipay.sofa.registry.server.data.cache.DatumCache#getDatumGroupByDataCenter

```java
public static Map<String, Datum> getDatumGroupByDataCenter(String dataCenter, String dataInfoId) {
    Map<String, Datum> map = new HashMap<>();
    if (StringUtils.isEmpty(dataCenter)) { //数据中心为空
        map = DatumCache.get(dataInfoId); //遍历所有的数据中心，根据dataInfoId获取服务信息
    } else {
        Datum datum = DatumCache.get(dataCenter, dataInfoId); //根据dataCenter、dataInfoId获取服务信息
        if (datum != null) {
            map.put(dataCenter, datum);
        }
    }
    return map;
}
```

## 通知数据同步

当DataServer监听到服务变更后，向其他的dataNode发送NotifyDataSyncRequest请求，其他dataNode进行版本的比对，判读是否需要进行拉取最新的数据

com.alipay.sofa.registry.server.data.remoting.dataserver.handler.NotifyDataSyncHandler#doHandle

```java
public Object doHandle(Channel channel, NotifyDataSyncRequest request) {
    final Connection connection = ((BoltChannel) channel).getConnection();
    executor.execute(() -> {
        String dataInfoId = request.getDataInfoId();
        String dataCenter = request.getDataCenter();
        Datum datum = DatumCache.get(dataCenter, dataInfoId);
        Long version = (datum == null) ? null : datum.getVersion();
        Long requestVersion = request.getVersion();
      	//当前节点版本为空或者小于请求的版本，需要进行同步
        if (version == null || requestVersion == 0L || version < requestVersion) {
            LOGGER.info(
                    "[NotifyDataSyncProcessor] begin get sync data, currentVersion={},request={}", version,
                    request);
            getSyncDataHandler
                    .syncData(new SyncDataCallback(getSyncDataHandler, connection, new SyncDataRequest(dataInfoId,
                            dataCenter, version, request.getDataSourceType()), dataChangeEventCenter));
        } else { //数据已经最新，无需同步
            LOGGER.info(
                    "[NotifyDataSyncHandler] not need to sync data, version={}", version);
        }
    });
    return CommonResponse.buildSuccessResponse();
}
```

com.alipay.sofa.registry.server.data.remoting.dataserver.GetSyncDataHandler#syncData

```java
public void syncData(SyncDataCallback callback) { //同步数据
    int tryCount = callback.getRetryCount();
    if (tryCount > 0) {
        try {
            callback.setRetryCount(--tryCount); //重试次数-1
          	//发送SyncDataRequest请求
            dataNodeExchanger.request(new Request() {
                @Override
                public Object getRequestBody() {
                    return callback.getRequest();
                }

                @Override
                public URL getRequestUrl() { //源dataNode
                    return new URL(callback.getConnection().getRemoteIP(), callback
                        .getConnection().getRemotePort());
                }

                @Override
                public CallbackHandler getCallBackHandler() {
                    return new CallbackHandler() {
                        @Override
                        public void onCallback(Channel channel, Object message) {
                            callback.onResponse(message);
                        }

                        @Override
                        public void onException(Channel channel, Throwable exception) {
                            callback.onException(exception);
                        }
                    };
                }
            });
        } catch (Exception e) {
            LOGGER.error("[GetSyncDataHandler] send sync data request failed", e);
        }
    } else {
        LOGGER.info("[GetSyncDataHandler] sync data retry for three times");
    }
}
```

com.alipay.sofa.registry.server.data.remoting.dataserver.SyncDataCallback#onResponse

```java
public void onResponse(Object obj) {//同步数据的回调
    GenericResponse<SyncData> response = (GenericResponse) obj;
    if (!response.isSuccess()) { //同步失败，进行重试
        getSyncDataHandler.syncData(this);
    } else { //同步成功
        SyncData syncData = response.getData();
        Collection<Datum> datums = syncData.getDatums();
        DataSourceTypeEnum dataSourceTypeEnum = DataSourceTypeEnum.valueOf(request
            .getDataSourceType());
        LOGGER
            .info(
                "[SyncDataCallback] get syncDatas,datums size={},wholeTag={},dataCenter={},dataInfoId={}",
                datums.size(), syncData.getWholeDataTag(), syncData.getDataCenter(),
                syncData.getDataInfoId());
        if (syncData.getWholeDataTag()) {
            //handle all data, replace cache with these datum directly
            for (Datum datum : datums) {
                if (datum == null) {
                    datum = new Datum();
                    datum.setDataInfoId(syncData.getDataInfoId());
                    datum.setDataCenter(syncData.getDataCenter());
                }
                processDatum(datum);
              	//负载本地数据
                dataChangeEventCenter.sync(DataChangeTypeEnum.COVER, dataSourceTypeEnum, datum);
                break;
            }
        } else {
            //handle incremental data one by one
            if (!CollectionUtils.isEmpty(datums)) {
                for (Datum datum : datums) {
                    if (datum != null) {
                        processDatum(datum);
                        dataChangeEventCenter.sync(DataChangeTypeEnum.MERGE,
                            dataSourceTypeEnum, datum);
                    }
                }
            } else {
                LOGGER.info("[SyncDataCallback] get no syncDatas");
            }
        }
    }
}
```

# 总结

**1.任何问题都可以通过加一中间层来解决**

2.增加了中间层SessionServer，可支持海量的客户端

3.优化服务发布方频繁变更。每个服务推送前进行一定延迟等待，等待一定时间后进行一次推送，借助延迟队列实现

4、推送任务比较大的时候会进行队列缓冲，以服务ID和推送方IP为粒度，对推送任务合并
