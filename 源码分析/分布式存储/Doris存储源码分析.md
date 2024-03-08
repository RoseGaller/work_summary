# Doris源码分析

* [概述](#概述)
* [客户端](#客户端)
  * [DataStoreFactoryImpl](#datastorefactoryimpl)
    * [创建](#创建)
    * [初始化](#初始化)
  * [Operation](#operation)
    * [Put](#put)
      * [buildSimpleLogicParam](#buildsimplelogicparam)
        * [构建key](#构建key)
        * [查找节点](#查找节点)
      * [doLogicExecute](#dologicexecute)
* [Admin](#admin)
  * [节点状态检查](#节点状态检查)
  * [NodeCheckThread](#nodecheckthread)
    * [准备迁移](#准备迁移)
    * [启动迁移线程](#启动迁移线程)
      * [发送迁移命令](#发送迁移命令)
      * [监控迁移状态线程](#监控迁移状态线程)
  * [修改迁移状态](#修改迁移状态)
* [DataServer](#dataserver)
  * [CheckAction](#checkaction)
  * [MigrationAction](#migrationaction)
    * [开始迁移](#开始迁移)
    * [启动迁移任务](#启动迁移任务)
    * [切分任务](#切分任务)
    * [迁移数据](#迁移数据)


# 概述

Doris 是一种支持 Key、Value 数据结构的分布式存储系统

集群中存储节点以序列为单位进行管理，每个序列下包含多个存储节点。

写入数据的时候，根据路由算法在每个序列中计算得到一台服务器，然后同时并发写入这些服务器中;

读取数据的时候，只需要随机选择一个序列，根据相同路由算法计算得到服务器编号和地址，即可读取。

通常情况下，系统最少写入的拷贝份数是两份。

# 客户端

## DataStoreFactoryImpl

### 创建

com.alibaba.doris.client.DataStoreFactoryImpl#DataStoreFactoryImpl(java.lang.String)

```java
public DataStoreFactoryImpl(String configLocation) {
   this.configLocation = configLocation;
   initConfig();
}
```

### 初始化

com.alibaba.doris.client.DataStoreFactoryImpl#initConfig

```java
public void initConfig() {
   
   initOperationFactory();       
   
   initCallbackHandlerFactory();
   
   initConfigManager();
   
   initUserAuth();
   
   initNamespace(); //从Admin获取namespace信息，包括需要保存的份数
   
   initDataStore();  //对每个namespace创建DataStoreImpl
   
   initDataSourceManager(); //数据路由
}
```

com.alibaba.doris.client.DataStoreFactoryImpl#initDataStore

```java
protected void initDataStore() {//为每个Namespace创建DataStoreImpl
   
   Map<String,Namespace> namespaceMap = namespaceManager.getNamespaces();
   
   dataStoreMap = new HashMap<String, DataStore>();
   
   for(Map.Entry<String, Namespace> entry: namespaceMap.entrySet()) {
      Namespace namespace = entry.getValue();
      
      DataStore dataStore = new DataStoreImpl( namespace );
      dataStore.setDataStoreFactory(this);
      
      dataStoreMap.put(dataStore.getNamespace().getName() , dataStore);
   }
}
```

## Operation

### Put

com.alibaba.doris.client.DataStoreImpl#put

```java
public boolean put(Object key, Object value) throws DorisClientException {
    keyValidator.validate(key);

    Operation operation = getOperation("put");

    List<Object> args = new ArrayList<Object>();
    args.add(key);
    args.add(value);
		//操作类型put、namespace、参数
    OperationData operationData = new OperationData(operation, namespace, args);
  	//执行  
    operation.execute(operationData);

    if (operationData.getResult() != null) 
      return ((Boolean) operationData.getResult()).booleanValue();
    else 
      return false;
}
```

com.alibaba.doris.client.operation.AbstractOperation#execute

```java
public void execute(OperationData operationData) {
		//执行参数
    List<Object> args = operationData.getArgs();
    if (args.get(0) == null) {
        throw new IllegalArgumentException("Argument 0 ( key) can't be null");
    }
		
    DataSourceRouter dataSourceRouter = dataSourceManager.getDataSourceRouter();
    RouteTableConfiger routeTableConfiger = dataSourceRouter.getRouteTableConfiger();
    RouteTable routeTable = routeTableConfiger.getRouteTable();

    if (routeTable == null) {
        throw new DorisClientException("Can't get RouteTable or connect to AdminServer.");
    }
    long routeVersion = routeTable.getVersion();

    VirtualRouter virtualRouter = dataSourceManager.getDataSourceRouter().getVirtualRouter();
    operationDataConverter.setVirtualRouter(virtualRouter);

    List<OperationDataSourceGroup> operationDataSourceGroups = null;
    try {

        operationDataSourceGroups = new ArrayList<OperationDataSourceGroup>();
        List<OperationData> operationDatas = new ArrayList<OperationData>();
				//构建key、查找对应的数据源
        buildLogicParam(operationDataSourceGroups, operationDatas, operationData, operationDataConverter, routeVersion);
				//执行操作
        List<List<DataSourceOpResult>> dsOpResults = doLogicExecute(operationDataSourceGroups, operationDatas);
				//合并操作结果
        mergeOperationResult(dsOpResults, operationDatas);

        if (operationDatas.size() > 1) {
            List<Object> rl = new ArrayList<Object>(operationDatas.size());
            for (OperationData od : operationDatas) {
                rl.add(od.getResult());
            }
            operationData.setResult(rl);
        }

    } catch (AccessException e) {
        if (logger.isErrorEnabled()) {
            logger.error(e.getMessage(), e);
        }
        reportConsisitentError(operationData, operationDataSourceGroups, e.getMessage());
        throw new DorisClientException(e);
    } catch (DorisRouterException e) {
        if (logger.isErrorEnabled()) {
            logger.error(e.getMessage(), e);
        }
        reportConsisitentError(operationData, operationDataSourceGroups, e.getMessage());
        throw new DorisClientException(e);
    }
}
```

#### buildSimpleLogicParam

com.alibaba.doris.client.operation.AbstractOperation#buildSimpleLogicParam

```java
protected void buildSimpleLogicParam(List<OperationDataSourceGroup> operationDataSourceGroups,
                                     List<OperationData> operationDatas, OperationData operationData,
                                     OperationDataConverter operationDataConverter, long routeVersion)
                                                                                                      throws DorisRouterException {
    //副本数
    int opCount = getOperationCount(operationData);
    //构建key
    Key phKey = operationDataConverter.buildKey(operationData, routeVersion);
    operationData.setKey(phKey);
    OperationDataSourceGroup operationDataSourceGroup = 	  dataSourceManager.getOperationDataSourceGroup(getOperationType(operationData),                                                                                                    opCount,phKey.getPhysicalKey());                                                                                             
 
    operationDataSourceGroups.add(operationDataSourceGroup);
    operationDatas.add(operationData);
}
```

##### 构建key

com.alibaba.doris.client.operation.impl.OperationDataConverterImpl#buildKey

```java
public Key buildKey(OperationData operationData, long routeVersion) { //构建key
    validNamespace(operationData);

    Object appKey = operationData.getArgs().get(0);
    String appKeyStr = String.valueOf(appKey);
    Key key = KeyFactory.createKey(operationData.getNamespace().getId(), appKeyStr, routeVersion, Key.DEFAULT_VNODE);
		//根据namespace:key计算vnode
    int vnode = virtualRouter.findVirtualNode(key.getPhysicalKey());
    key.setVNode(vnode);
    return key;
}
```

getOperationDataSourceGroup

com.alibaba.doris.client.DataSourceManagerImpl#getOperationDataSourceGroup

```java
public OperationDataSourceGroup getOperationDataSourceGroup( OperationEnum operationEnum,int count,Object key) throws DorisRouterException {//查找对应的数据源
   String stringKey = String.valueOf( key);
   OperationDataSourceGroup operationDataSourceGroup = dataSourceRouter.getOperationDataSourceGroup( operationEnum, count,stringKey);
   return operationDataSourceGroup;
}
```

com.alibaba.doris.client.DataSourceRouterImpl#getOperationDataSourceGroup

```java
public OperationDataSourceGroup getOperationDataSourceGroup(OperationEnum operationEnum, int count, String key)   throws DorisRouterException {                                                                                                       
    RouteStrategyHolder rHolder = routeStrategyHolder;
    //根据操作类型、副本数、key计算存储的节点
    List<StoreNode> storeNodes = rHolder.routeStrategy.findNodes(operationEnum, count, key);

    List<DataSource> dataSourceGroup = new ArrayList<DataSource>();
    for (StoreNode storeNode : storeNodes) {
				//获取storeNode所处的group
        String seqNo = String.valueOf(storeNode.getSequence().getValue());
        //group内的DataSource
        List<DataSource> seqDataSources = rHolder.allDataSources.get(seqNo);
        //根据logicId获取节点对应的DataSource
        DataSource dataSource = seqDataSources.get(storeNode.getLogicId());
        dataSourceGroup.add(dataSource);
    }
	
    OperationDataSourceGroup group = new OperationDataSourceGroup(key, dataSourceGroup, storeNodes);
    return group;
}
```

##### 查找节点

com.alibaba.doris.common.router.service.RouteStrategyImpl#findNodes

```java
public List<StoreNode> findNodes(OperationEnum type, int copyCount, String key) throws DorisRouterException {
    if (rlc.getVpmrList() == null || rlc.getVpmrList().isEmpty()) { //路由算法不能为空
        log.error("Current RouterListContainer is:" + rlc);
        throw new DorisRouterException("There is no store node.");
    }
    if (copyCount > rlc.getVpmrList().size()) { //副本数不能大于路由算法数
        log.error("Current RouterListContainer is:" + rlc);
        throw new DorisRouterException("There is only " + rlc.getVpmrList().size() + " sequence, Can't support "
                                       + copyCount + " copy count!");
    }
    List<StoreNode> snList = new ArrayList<StoreNode>();

    for (int i = 0; i < copyCount; i++) {
        if (type.equals(OperationEnum.READ) || type.equals(OperationEnum.WRITE)
            || type.equals(OperationEnum.MULTIREAD)) {// 写操作一定要取得一个node，读操作只有在可读时才取node
          	//计算该key对应的logicId
            int logicId = rlc.getVpmrList().get(i).getNodeByKey(key);
						//获取所有group的List<StoreNode>
            List<List<StoreNode>> mainNodeList = rlc.getMainStoreNodeList();
          	//获取该group的所有的StoreNode
            List<StoreNode> seqNodeList = mainNodeList.get(i);
            //根据logicId获取StoreNode
            StoreNode sn = seqNodeList.get(logicId);
            snList.add(sn);

        }
    }

    if (snList.isEmpty()) { //无节点可操作
        throw new DorisRouterException("No store node can be used!");
    }

    // //////////////////////////////////////////////////////////TODO for debug
    // List<StoreNode> tl = new ArrayList<StoreNode>();
    // for (StoreNode tsn : snList) {
    // tl.add(tsn);
    // }
    // //////////////////////////////////////////////////////

    anylizeNode(type, snList, key);

    if (snList.isEmpty()) {
        // log.error("No store node for operation:" + tl + " become:" + snList);
        throw new DorisRouterException("No store node for operation");
    }
    if (type.equals(OperationEnum.WRITE) && snList.size() < copyCount) {
        // log.error("No enough store node for write:" + tl + " become:" + snList);
        throw new DorisRouterException("No enough store node for write");
    }

    if (OperationEnum.MULTIREAD.equals(type)) {
        return snList;
    }

    // if read ,then find one node
    if (OperationEnum.READ.equals(type)) { //读操作，选择一个节点
        StoreNode sn = snList.get(getHashIndex(snList.size(), key));
        List<StoreNode> tempList = new ArrayList<StoreNode>(1);
        tempList.add(sn);
        return tempList;
    }

    int size = snList.size();
    // if write, must not be all backup nodes,
    if (OperationEnum.WRITE.equals(type)) { //写操作
        boolean backTag = true;
        List<StoreNode> tempList = new ArrayList<StoreNode>(size);
        // 保证节点在访问时，同一个key会以相同的序列来访问所有node；访问顺序必须同read的顺序一致；
        // 以便保证写操作的原子性；
        for (int i = 0; i < size; i++) {
            int index = getHashIndex(size - i, key);
            StoreNode sn = snList.get(index);
            snList.remove(index);
            if (!sn.getSequence().equals(StoreNodeSequenceEnum.TEMP_SEQUENCE)) {// 只要有一个节点不是备用节点
                backTag = false;
            }
            tempList.add(sn);
        }
        if (backTag) {
            throw new DorisRouterException("All nodes is backup node.");
        }
        return tempList;
    }
    return snList;
}
```

com.alibaba.doris.common.router.service.RouteStrategyImpl#anylizeNode

```java
private void anylizeNode(OperationEnum type, List<StoreNode> snList, String key) throws DorisRouterException {

    if (type.equals(OperationEnum.READ) || type.equals(OperationEnum.MULTIREAD)) {
        Iterator<StoreNode> i = snList.iterator();
        while (i.hasNext()) {
            NodeRouteStatus status = i.next().getStatus();
            if (status.equals(NodeRouteStatus.TEMP_FAILED)) { //读操作移除临时失效节点
                i.remove();
            }
        }
    }

    if (type.equals(OperationEnum.WRITE)) {
        for (int i = 0; i < snList.size(); i++) {
            NodeRouteStatus status = snList.get(i).getStatus();
            if (status.equals(NodeRouteStatus.TEMP_FAILED)) { //写操作将失效节点替换成备用节点
                // 用备份节点写
                snList.set(i, findBackupNode(key));
            }
        }
    }

}
```

查找备用节点

com.alibaba.doris.common.router.service.RouteStrategyImpl#findBackupNode(java.lang.String)

```java
private StoreNode findBackupNode(String key) throws DorisRouterException {
    if (rlc.getBackupStoreNodeList() == null || rlc.getBackupVpmr() == null
        || rlc.getBackupStoreNodeList().isEmpty()) {
        log.error("Current RouterListContainer is:" + rlc);
        throw new DorisRouterException("rlc's temp node is null. can't find useful backup node.");
    }
  	//根据备用路由算法计算key的logicId
    int k = rlc.getBackupVpmr().getNodeByKey(key);
    //获取StoreNode
    StoreNode sn = rlc.getBackupStoreNodeList().get(k);
    if (sn.getStatus().equals(NodeRouteStatus.OK)) {
        return sn;
    }
    int x = rlc.getBackupStoreNodeList().size();
    for (int i = 0; i < x - 1; i++) {
        if (k < x - 1) {
            k++;
        } else {
            k = 0;
        }
        sn = rlc.getBackupStoreNodeList().get(k);
        if (sn.getStatus().equals(NodeRouteStatus.OK)) {
            return sn;
        }
    }
    log.error("Current backup list is:" + rlc.getBackupStoreNodeList());
    for (int i = 0; i < rlc.getBackupStoreNodeList().size(); i++) {
        log.error("all backup nodes were failed:" + sn + ":" + rlc.getBackupStoreNodeList().get(i).getStatus());
    }
    throw new DorisRouterException("at last. can't find useful backup node." + sn.getStatus());
}
```

#### doLogicExecute

com.alibaba.doris.client.operation.AbstractOperation#doLogicExecute

```java
public List<List<DataSourceOpResult>> doLogicExecute(List<OperationDataSourceGroup> operationDataSourceGroups,
                                                     List<OperationData> operationDatas) throws AccessException {
    if (operationDataSourceGroups.isEmpty() || operationDatas.isEmpty()) {
        throw new AccessException("operationDataSourceGroups or operationDatas is empty");
    }
    int size = operationDataSourceGroups.size();

    List<LogicCallback> logicCallbacks = new ArrayList<LogicCallback>(size);
  	//LogicFailoverCallbackHandler
    LogicCallbackHandler callbackHandler = callbackHandlerFactory.getLogicFailoverHandler();
    callbackHandler.setDataSourceManager(dataSourceManager);

    for (int i = 0; i < size; i++) {
        final OperationDataSourceGroup group = operationDataSourceGroups.get(i);

        LogicCallback logicCallback = new DefaultLogicCallback(operationDatas.get(i)) {

            @Override
            public LogicCallback execute() throws AccessException {
                List<DataSourceOpFuture> dsFutureList = new ArrayList<DataSourceOpFuture>(
                                                                                          group.getDataSources().size());
								//每个节点发送请求
                for (final DataSource dataSource : group.getDataSources()) {
                    // distributed computing logic here.
                    PeerCallback peerCallback = doPeerExecute(dataSource, operationData);
                    // OperationFuture<?> future = peerCallback.getOperationFuture();

                    DataSourceOpFuture dsopFuture = new DataSourceOpFuture(dataSource, peerCallback);
                    dsFutureList.add(dsopFuture); //获取DataSourceOpFuture，阻塞获取响应结果
                }
                this.dataSourceOpFutures = dsFutureList;
                return this;
            }
        };
        logicCallbacks.add(logicCallback);

    }
    List<List<DataSourceOpResult>> dataSourceOpResults =callbackHandler.doLogicExecute(logicCallbacks,operationDataSourceGroups);
                                                                                      
    // resultList.add(dataSourceOpResults);
    return dataSourceOpResults;
}
```

com.alibaba.doris.client.operation.failover.impl.LogicFailoverCallbackHandler#doLogicExecute

```java
public List<List<DataSourceOpResult>> doLogicExecute(List<LogicCallback> callbacks,
                                                     List<OperationDataSourceGroup> operationDataSourceGroups)
                                                                                                              throws AccessException {

    List<List<DataSourceOpFuture>> futuresList = new ArrayList<List<DataSourceOpFuture>>();
    for (LogicCallback callback : callbacks) {
        callback.execute(); //执行操作，发送请求
        List<DataSourceOpFuture> dataSourceOpFutures = callback.getDataSourceOpFutures();
        futuresList.add(dataSourceOpFutures);
    }

    List<List<DataSourceOpResult>> dataSourceOpResultsList = new ArrayList<List<DataSourceOpResult>>();
    int size = futuresList.size();
    for (int k = 0; k < size; k++) {
        List<DataSourceOpFuture> dataSourceOpFutures = futuresList.get(k);
        OperationDataSourceGroup group = operationDataSourceGroups.get(k);
        List<DataSourceOpResult> dataSourceOpResults = new ArrayList<DataSourceOpResult>();
        for (DataSourceOpFuture dsOpfuture : dataSourceOpFutures) {
            boolean failed = true;
            // try 5 times for one node, try 3 nodes total
            PeerCallback peerCallback = dsOpfuture.getPeerCallback();
            DataSource dataSource = peerCallback.getDataSource();
            for (int i = 1; i <= 5; i++) {
                try {
                    //阻塞获取执行结果
                    Object value = peerCallback.getOperationFuture().get(timeoutOfOperation, TimeUnit.MILLISECONDS);

                    DataSourceOpResult dataSourceOpResult = new DataSourceOpResult();
                    dataSourceOpResult.setDataSource(dataSource);
                    dataSourceOpResult.setResult(value);

                    dataSourceOpResults.add(dataSourceOpResult);
                    failed = false;
6217 0005 9000 5989 615
                    break;
                } catch (RouteVersionOutOfDateException routeException) {
                    proceeRouteException(routeException, peerCallback, group); //重新获取路由表，重试
                } catch (ClientConnectionException cce) {
                    processClientConnectionException(peerCallback); //重试
                } catch (InterruptedException e) {
                    throw new AccessException(e);
                } catch (ExecutionException e) {
                    throw new AccessException(e);
                } catch (TimeoutException e) {
                    processClientConnectionException(peerCallback); //重试
                }
                sleep(i);
            }
            
            if (failed) {
                throw new AccessException("LogicFailoverCallbackHandler could not process.");
            }
        }
        dataSourceOpResultsList.add(dataSourceOpResults);
    }

    return dataSourceOpResultsList;
}
```

doPeerExecute

封装向每个节点发送具体请求的操作

com.alibaba.doris.client.operation.AbstractOperation#doPeerExecute

```java
public PeerCallback doPeerExecute(final DataSource dataSource, final OperationData operationData)
                                                                                                 throws AccessException {

    final PeerCallbackHandler callbackHandler = callbackHandlerFactory.getPeerFailoverHandler();
    callbackHandler.setDataSourceManager(dataSourceManager);
    callbackHandler.setOperationData(operationData);
    callbackHandler.setDataSource(dataSource);
		//定义对应的get、put、delete操作
    PeerCallback callback = generatePeerCallback(dataSource, operationData);
		//执行
    return callbackHandler.doPeerExecute(callback); 
}
```

com.alibaba.doris.client.operation.failover.impl.PeerFailoverCallbackHandler#doPeerExecute

```java
public PeerCallback doPeerExecute(PeerCallback callback) throws AccessException {
    if (log.isDebugEnabled()) {
        log.debug("execute " + operationData.getOperation().getName() + " in " + dataSource.getNo() + " for "
                  + operationData.getKey());
    }

    PeerCallback pcb = null;
    boolean failed = true;
    for (int i = 1; i <= 10; i++) {
        try {
            pcb = callback.execute(); //执行具体的操作，例如put、delete、get
            failed = false;
            break;

        } catch (AccessException e) {
           if( e.getCause() instanceof NetException) {
              processAccessException(i, callback); //重试
           }else {
              throw e;
           }                
        }
    }
    if (failed) {
        throw new AccessException("Coundn't connect data server:" + callback.getDataSource());
    }
    return pcb;
}
```

com.alibaba.doris.client.operation.impl.PutOperation#generatePeerCallback

```java
protected PeerCallback generatePeerCallback(DataSource dataSource, OperationData operationData) { //具体的put操作
    return new DefaultPeerCallback(dataSource, operationData) {

        @Override
        public PeerCallback execute() throws AccessException {
            try {
              	//连接池获取连接
                Connection connection = dataSource.getConnection();
                Key commuKey = (Key) operationData.getKey();
                Value commuValue = (Value) operationData.getArgs().get(1);
								//发送put请求
                this.future = connection.put(commuKey, commuValue);
            } catch (NetException e) {
                throw new AccessException("put connection exception.", e);
            } catch (Throwable t) {
                throw new AccessException("put operation exception." + t, t);
            }
            return this;
        }
    };
}
```

processAccessException

异常重试，同一节点连续三次请求失败，向Admin发送请求判断节点是否异常，如果节点临时失效，将请求发送临时节点

com.alibaba.doris.client.operation.failover.impl.PeerBaseFailoverCallbackHandler#processAccessException

```java
protected void processAccessException(int i, PeerCallback peerCallback) throws AccessException {

    if (log.isWarnEnabled()) {
        log.warn("error occur in executing " + this.getOperationData().getOperation().getName() + " in "
                 + peerCallback.getDataSource().getSequence() + "." + peerCallback.getDataSource().getNo()
                 + " for " + this.getOperationData().getKey() + " " + i + "times");
    }
    if (i % 3 == 0) { //连续三次重试都抛出NetExcption，向Admin发送请求判断节点是否异常
        StoreNode node = this.getDataSourceManager().getDataSourceRouter().getStoreNodeOf(                                                                                peerCallback.getDataSource());
        if (!AdminServiceFactory.getCheckFailService().checkFailNode(node)) {//节点异常
            try {
              	//临时失效，查找临时节点
                StoreNode failoverNode = this.getDataSourceManager().getDataSourceRouter().getRouteStrategy().findFailoverNode(peerCallback.getOperationData().getOperation().getOperationType(peerCallback.getOperationData()),
                                                                                                                               peerCallback.getOperationData().getNamespace().getCopyCount(),
                                                                                                                               peerCallback.getOperationData().getKey().getPhysicalKey(),
                                                                                                                               node);
                // switch datasource
                DataSource ds = this.getDataSourceManager().getDataSourceRouter().getDataSourceOf(failoverNode);
                peerCallback.setDataSource(ds);
                if (log.isInfoEnabled()) {
                    log.info( "fail over " + node + " to " + failoverNode);
                }
            } catch (DorisRouterException e1) {
                if (log.isErrorEnabled()) {
                    log.error("Route error:" + e1);
                }
                throw new AccessException();
            }
        }

    }
    // data server返回路由版本异常，从异常中获取路由表，更新本地路由，并根据新路由、新版本重新请求

}
```

查找临时节点

com.alibaba.doris.common.router.service.RouteStrategyImpl#findFailoverNode

```java
public StoreNode findFailoverNode(OperationEnum type, int copyCount, String key, StoreNode sn) throws DorisRouterException {                                                                                     

    // log.error(sn + " is failed by admin check.");
    sn.setStatus(NodeRouteStatus.TEMP_FAILED);// 一定是临时失效，这个修改是client的临时修改，在新的配置实例生效前使用

    List<StoreNode> snList = findNodes(type, copyCount, key);// 重新获取节点列表，因为刚修改过一个节点状态，这里可能因为都是备用节点而导致写操作路由异常。

    if (snList == null || snList.isEmpty()) {
        log.error("Current RouterListContainer is:" + rlc);
        throw new DorisRouterException("can't find failover node.");
    }

    // 读操作,重新获取读节点
    if (type.equals(OperationEnum.READ) || type.equals(OperationEnum.MULTIREAD)) {
        return snList.get(getHashIndex(snList.size(), key));
    }

    // 写操作，查找备用节点
    if (type.equals(OperationEnum.WRITE)) {
        return findBackupNode(key);
    }

    log.error("not support this type of operation:" + type.name());
    throw new DorisRouterException("not support this type of operation:" + type.name());
}
```

# Admin

## 节点状态检查

com.alibaba.doris.admin.service.failover.check.AdminNodeCheckAction#execute

```java
public String execute(Map<String, String> params) {
     //节点物理Id
    String nodePhysicalId = params.get(AdminServiceConstants.STORE_NODE_PHYSICAL_ID);
    //查看节点状态
    NodeHealth nodeHealth = NodeCheckManager.getInstance().checkNode(nodePhysicalId);
    Boolean result = (nodeHealth == NodeHealth.OK);
    return result.toString();
}
```

com.alibaba.doris.admin.service.failover.node.check.NodeCheckManager#checkNode(com.alibaba.doris.common.StoreNode, boolean)

```java
public NodeHealth checkNode(StoreNode node, boolean needRealAccessing) {//此时needRealAccessing=true
    if (node == null) {
        return null;
    }
    lock.lock();
    NodeHealth health = null;
    try {
        health = nodeHealthStatuses.get(node.getPhId()); //节点当前状态
        if (needRealAccessing && health == NodeHealth.OK) {
            health = accessAndCacheResult(node);
        }
    } finally {
        lock.unlock();
    }

    if (health == null) {
        health = accessAndCacheResult(node);
    }
    
    return health;
}
```

com.alibaba.doris.admin.service.failover.node.check.NodeCheckManager#accessAndCacheResult

```java
private NodeHealth accessAndCacheResult(StoreNode node) {
    NodeHealth health;
    health = verifyNodeAcess(node);

    //cache health result.
    nodeHealthStatuses.put(node.getPhId(), health);
    return health;
}
```

com.alibaba.doris.admin.service.failover.node.check.NodeCheckManager#verifyNodeAcess(com.alibaba.doris.common.StoreNode)

```java
public NodeHealth verifyNodeAcess(StoreNode node) {
    if (node == null ) {
        return null;
    }
    try {
        Connection conn = NodesManager.getInstance().getNodeConnection(node);
        return verifyNodeAcess(node, conn);
    } catch (Exception e) {
        log.debug("fail to check node :" + node.getPhId(), e);
        updateNodeHealth(node, NodeHealth.NG);
        return NodeHealth.NG;
    }
}
```

com.alibaba.doris.admin.service.failover.node.check.NodeCheckManager#verifyNodeAcess(com.alibaba.doris.common.StoreNode, com.alibaba.doris.client.net.Connection)

```java
public NodeHealth verifyNodeAcess(StoreNode node, Connection conn) {

    NodeHealth nodeHealth = NodeHealth.NG;
    try {
        Type checkType = null;

        switch (node.getSequence()) {
            case TEMP_SEQUENCE: // 临时节点
                checkType = CheckType.CHECK_TEMP_NODE;
                break;
            case STANDBY_SEQUENCE: // 备用接地那
                checkType = CheckType.CHECK_STANDBY_NODE;
                break;
            case UNUSE_SEQUENCE: // 待用节点
                checkType = null;
                break;
            default: //正常（Data）节点
                checkType = CheckType.CHECK_NORMAL_NODE;
                break;
        }
        //向datanode发送检查状态的请求
        OperationFuture<CheckResult> future = conn.check(checkType);
        CheckResult checkResult = null;
        try {
            checkResult = future.get(NodeCheckThread.nodeCheckTimeout, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            log.error("check failed for node :" + node.getPhId());
            log.error(e.getMessage(), e);
            checkResult = null;
        }

        if (checkResult == null) {
            nodeHealth = NodeHealth.NG;
            if (checkResult == null) {
                log.warn("Check result is null for node :" + node.getPhId());
            }
        } else if (checkResult.isSuccess()) {
            nodeHealth = NodeHealth.OK;
        } else {
            nodeHealth = NodeHealth.NG;
            log.warn("Check result is NG for node :" + node.getPhId() + ", Message:"
                    + checkResult.getMessage());
        }

    } catch (Exception e) {
        log.debug("fail to check node :" + node.getPhId(), e);
        updateNodeHealth(node, NodeHealth.NG);
        return NodeHealth.NG;
    }

    updateNodeHealth(node, nodeHealth);

    return nodeHealth;

}
```

## NodeCheckThread

周期性节点状态检查，当有正常节点的状态由NG转为OK，开始迁移数据,向所有的临时节点发送数据迁移的命令

com.alibaba.doris.admin.service.failover.node.check.NodeCheckThread#onNodeHealthStatusChange

```java
private void onNodeHealthStatusChange(PhysicalNodeDO physicalNodeDO, NodeHealth nodeHealth,
                                      NodeHealth originalNodeHealth) {

    String physicalId = physicalNodeDO.getPhysicalId();
    boolean isNormalNode = StoreNodeSequenceEnum.isNormalSequence(String.valueOf(physicalNodeDO.getSerialId()));
    if (nodeHealth == NodeHealth.NG) { //节点下线

        StoreNode node = nodeManager.getStoreNode(physicalId);
        if (node == null) {
            log.error("Cannot find node for node with pid=" + physicalId);
            return;
        }
        NodeRouteStatus routeStatus = node.getStatus();
        if (routeStatus != NodeRouteStatus.TEMP_FAILED) {
            // 1. from NodeHealth.OK to NodeHealth.NG, generate new route configuration instance
            int status = NodeRouteStatus.TEMP_FAILED.getValue();
            if (log.isInfoEnabled()) {
                log.info("update node status for node \"" + physicalId + "\" by set status to "
                         + NodeRouteStatus.TEMP_FAILED);
            }
            nodeService.updatePhysicalNodeStatus(physicalId, status);
            try {
                RouteConfigProcessor.getInstance().refresh();
            } catch (DorisConfigServiceException e) {
                log.error("failed refresh route config", e);
                // message 'Re-Gen Route failed:' is used as dragoon warning rule, should be consistent.
                SystemLogMonitor.error(MonitorEnum.ROUTER,
                        MonitorWarnConstants.RE_GEN_ROUTE_FAILED + ":" + physicalId, e);
            }
            if (log.isInfoEnabled()) {
                log.info("refresh and generated new route config instance.");
            }
            // message 'Node Temp Failed:' is used as dragoon warning rule, should be consistent.
            SystemLogMonitor.error(MonitorEnum.NODE_HEALTH,
                    MonitorWarnConstants.NODE_TEMP_FAILED + ":" + physicalId);
        }

        boolean foreverFailed = checkForeverFail(physicalId);
        
        if (foreverFailed) {
            SystemLogMonitor.error(MonitorEnum.NODE_HEALTH,
                    MonitorWarnConstants.NODE_FOREVER_FAILED + ":" + physicalId);
        }
        
        if (foreverFailed && isNormalNode) {
            if (log.isInfoEnabled()) {
                log.info("resolve type: forever failed resolve for node :" + physicalId);
            }
            FailoverProcessor failoverProcessor = ForeverFailoverProcessor.getInstance();

            try {
                failoverProcessor.failResolve(physicalId);
                if (log.isInfoEnabled()) {
                    log.info("resolve starts for node with physical id \"" + physicalId + "\".");
                }
            } catch (AdminServiceException e) {
                log.error("resolve starts for node with physical id \"" + physicalId + "\", but failed.");
                SystemLogMonitor.error(MonitorEnum.NODE_HEALTH,
                        MonitorWarnConstants.NODE_FOREVER_FAILURE_RESOLVE_FAILED + ":" + physicalId, e);
            }

            // reset counter
            tempFailTimes.put(physicalId, null);

        }
    } else if (originalNodeHealth == NodeHealth.NG && isNormalNode) { //节点状态由NG转为OK
        //启动故障转移，将临时节点数据迁移到正常节点
        FailoverProcessor failoverProcessor = TempFailoverProcessor.getInstance();

        try {
            failoverProcessor.failResolve(physicalId);
        } catch (AdminServiceException e) {
            SystemLogMonitor.error(MonitorEnum.NODE_HEALTH,
                            MonitorWarnConstants.NODE_TEMP_FAILURE_RESOLVE_FAILED + ":" + physicalId, e);
        }
    }

}
```

### 准备迁移

com.alibaba.doris.admin.service.failover.processor.FailoverProcessor#failResolve

```java
public synchronized void failResolve(String failPhysicalNodeId) throws AdminServiceException {
    if (!isMainAdmin) {
        if (log.isErrorEnabled()) {
            log.error("This is backup admin server, doesn't support expansion migrating task.");
        }
        throw new AdminServiceException("This is backup admin server, doesn't support expansion migrating task.");
    }
    //获取迁入节点的的信息
    PhysicalNodeDO node = NodesManager.getInstance().getNode(failPhysicalNodeId);
    if (failPhysicalNodeId == null || node == null
        || node.getSerialId() < StoreNodeSequenceEnum.NORMAL_SEQUENCE_1.getValue()
        || node.getSerialId() > StoreNodeSequenceEnum.NORMAL_SEQUENCE_4.getValue()) {
        if (log.isWarnEnabled()) {
            log.warn("illegal input parameters:" + failPhysicalNodeId);
        }
        throw new AdminServiceException("illegal input parameters:" + failPhysicalNodeId + ", it's sequence:"
                                        + node.getSerialId());
    }
    //数据库载入所有节点
    NodesManager.getInstance().reLoadNodes();
    //临时节点都有效
    canFailoverMigerate(failPhysicalNodeId);

    if (MigrateManager.getInstance().isMigrating(failPhysicalNodeId)) {// 节点正在失效迁移中，kill
        MigrateThread migThread = MigrateManager.getInstance().getMigerateThread(failPhysicalNodeId);
        if (migThread != null) {
            migThread.over();
        }

        if (log.isWarnEnabled()) {
            log.warn(NodesManager.getInstance().getLogFormatNodeId(failPhysicalNodeId)
                     + " is migerating,kill and restart.");
        }
    }
    // 启动迁移线程
    startMigerateThread(failPhysicalNodeId);

    if (log.isInfoEnabled()) {
        log.info("execute failover migerate in " + failPhysicalNodeId);
    }

}
```

### 启动迁移线程

com.alibaba.doris.admin.service.failover.processor.TempFailoverProcessor#startMigerateThread

```java
protected void startMigerateThread(String failPhysicalNodeId) {
    TempFailoverMigrateThread migThread = new TempFailoverMigrateThread(failPhysicalNodeId);
}
```

com.alibaba.doris.admin.service.common.migrate.MigrateThread#run

```java
public void run() {
    if (needMigrate) {
        if (!basicCommand()) {//发送迁移命令,响应失败，终止迁移
            this.over();
            return;
        }
        //监控迁移状态
        monitorThread = new MigrateStatusMonitorThread(statusList, this);
        monitorThread.start();
        while (isGoOn) {
            try {
                sleep(sleepTime);
            } catch (InterruptedException e) {
                log.error("", e);
            }
        }
    } else {
        updateStoreNode();
        
        NodesManager.getInstance().reLoadNodes();
        
        if (log.isDebugEnabled()) {
            log.debug("Don't need migrate, but need refresh route. ");
        }
        
        try {
            RouteConfigProcessor.getInstance().refresh();//不是必须的
        } catch (DorisConfigServiceException e) {
            SystemLogMonitor.error(MonitorEnum.ROUTER,
                    MonitorWarnConstants.RE_GEN_ROUTE_FAILED, e);
            log.error("ERROR IN REFRESH CONFIG TABLE.");
        }
    }
}
```

#### 发送迁移命令

com.alibaba.doris.admin.service.common.migrate.MigrateThread#basicCommand

```java
protected boolean basicCommand() {
    boolean result = sendMigerateCommand();
    MigrateManager.getInstance().addMigerateThread(migrateKey, this);
    return result;
}
```

com.alibaba.doris.admin.service.failover.migrate.TempFailoverMigrateThread#sendMigerateCommand

```java
protected boolean sendMigerateCommand() {
    return processSendCommand(tempPhysicalNodeIdList, MigrateTypeEnum.TEMP_FAILOVER);
}
```

com.alibaba.doris.admin.service.failover.migrate.FailoverMigrateThread#processSendCommand

```java
protected boolean processSendCommand(List<PhysicalNodeDO> physicalNodeIdList,
                                     MigrateTypeEnum type) {//发送迁移命令
    List<MigrateStatus> tempList = new ArrayList<MigrateStatus>();
    tempList.add(MigrateManager.getInstance().addMigerateNode(failPhysicalNodeIdList.get(0),
                                                              type, MigrateStatusEnum.PREPARE));
    boolean ok = true;
    commandParamList.clear();
    for (PhysicalNodeDO physicalNode : physicalNodeIdList) {
        // 如果命令发送失败，则不启动迁移监控线程，使迁移无法完成。
        MigrateCommandResult mcr= MigrateCommand.executeMigerate(physicalNode.getPhysicalId(),failPhysicalNodeIdList, type);
         Boolean sentResult = mcr.getResult();
         commandParamList.add(mcr.getCommandParam());
        if (sentResult == null) {
            if (log.isInfoEnabled()) {
                log.info("No command need send for temp fail over:"
                        + physicalNode.getPhysicalId() + "-->" + failPhysicalNodeIdList);
            }
            continue;
        }
        if (sentResult) {
            // 初始化迁移状态,必需的，避免无数据迁移的临时节点立即返回完成导致迁移结束
            MigrateManager.getInstance().updateMigerateStatus(physicalNode.getPhysicalId(),
                    failPhysicalNodeIdList.get(0), 0, MigrateStatusEnum.PREPARE,
                    "sent command.");
        } else {
            if (log.isWarnEnabled()) {
                log.warn("sent temp fail migrate command failed:"
                        + physicalNode.getPhysicalId() + "-->" + failPhysicalNodeIdList);
            }
            
            ok = false;
            break;
        }
    }
    //设置迁移状态，初始为PREPARE，后续Admin会收到临时节点上报的迁移进度，修改迁移状态
    statusList = tempList;
    if (monitorThread != null) {
        monitorThread.setStatusList(tempList);
    }
    return ok;
}
```

#### 监控迁移状态线程

com.alibaba.doris.admin.service.common.migrate.status.MigrateStatusMonitorThread#run

```java
public void run() {

    if (log.isInfoEnabled()) {
        log.info(taskId + "Migerate monitor thread start:" + statusList);
    }
    while (isGoOn) {
        boolean allFinish = true;
        boolean error = false;
        for (MigrateStatus status : statusList) { //迁移状态列表
            status.resetReportTime();
            if (log.isDebugEnabled()) {
                log.debug(taskId + " " + status.toString());
            }
            if (!status.getMigerateStatus().equals(MigrateStatusEnum.FINISH)) {// 至少有一个迁移没有完成
                allFinish = false;
            }
            if (status.getMigerateStatus().equals(MigrateStatusEnum.MIGERATE_ERROR)) {// 至少有一个迁移报告错误
                error = true;
                allFinish = false;
            }
            if (status.isTimeout()) {
                // message 'Migrate Report Timeout:' is used as dragoon warning rule, should be consistent.
                SystemLogMonitor.error(MonitorEnum.MIGRATION, MonitorWarnConstants.MIGRATE_REPORT_TIMEOUT + ":"
                        + status.getPhysicalId() + ",Mig-Type:" + status.getMigerateType());
            }
        }
        if (allFinish) { //迁移结束
            callback.finishAll();
        }
        if (error) { //迁移出现异常
            callback.notifyError(); 
        }
        try {
            sleep(sleepTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 修改迁移状态

接收DataServer发送迁移状态

com.alibaba.doris.admin.service.common.migrate.AdminMigrateStatusReportAction#execute

```java
public String execute(Map<String, String> params) {
    String remoteIp = params.get(AdminServiceConstants.REMOTE_IP);
    String port = params.get(AdminServiceConstants.MIGRATE_REPORT_SOURCE_NODE_PORT);
    String sourcePhysicalId = remoteIp + ":" + port;
    if (log.isDebugEnabled()) {
        log.debug("migrate report src:" + sourcePhysicalId );
    }
    MigrateManager.getInstance().updateMigerateStatus(
                                                      sourcePhysicalId,
                                                      params.get(AdminServiceConstants.MIGRATE_REPORT_TARGET_NODE_PHYSICAL_ID),
                                                      Integer.valueOf(params.get(AdminServiceConstants.MIGRATE_REPORT_SCHEDULE)),
                                                      MigrateStatusEnum.getEnum(params.get(AdminServiceConstants.MIGRATE_REPORT_STATUS)),
                                                      params.get(AdminServiceConstants.MIGRATE_REPORT_MESSAGE));
    return "OK";
}
```

















# DataServer

## CheckAction

接收Admin发送的健康检查请求

com.alibaba.doris.dataserver.action.CheckAction#execute

```java
public void execute(Request request, Response response) {
    CheckActionData actionData = (CheckActionData) request.getActionData();

    ApplicationContext appContext = request.getApplicationContext();
    List<Module> moduleList = appContext.getModules();

    actionData.setSuccess(true);

    for (Module m : moduleList) {
        if (m instanceof ModuleStatusChecker) {
            if (!((ModuleStatusChecker) m).isReady(actionData)) {
                actionData.setSuccess(false);
                String moduleName = m.getName();
                moduleName = StringUtils.replace(moduleName, " ", "_");
                actionData.setMessage(moduleName + "_failed");
                break;
            }
        }
    }

    response.write(actionData);
}
```

com.alibaba.doris.dataserver.store.StorageModule#isReady

```java
public boolean isReady(CheckActionData checkActionData) {
    try {
        Storage storage = manager.getStorage();
        String clazzName = storage.getClass().getName();
        CheckType checkType = checkActionData.getCheckType();
        // 判断是否是Log存储层
        if (clazzName.indexOf("LogStorage") >= 0) {
            if (checkType == null || checkType == CheckType.CHECK_TEMP_NODE
                || checkType == CheckType.CHECK_STANDBY_NODE) {
                Iterator<Pair> iterator = storage.iterator();
                if (null != iterator) {
                    if (iterator instanceof ClosableIterator) {
                        ((ClosableIterator<Pair>) iterator).close();
                    }
                    return true;
                }
            }
        } else { //其他存储类型
            if (checkType == null || checkType == CheckType.CHECK_NORMAL_NODE
                || checkType == CheckType.CHECK_STANDBY_NODE) {
              	//执行put、get请求
                storage.set(key, value);
                Value v = storage.get(key);
                if (v != null) {
                    if (v.equals(value)) {
                        return true;
                    }
                }
                return true;
            }
        }
    } catch (Throwable e) {
        logger.error("Checking module isReady.", e);
    }

    return false;
}
```





## MigrationAction

接收Admin发送的迁移请求

com.alibaba.doris.dataserver.migrator.action.MigrationAction#execute

```java
public void execute(Request request, Response response) {

    if (migrationManager == null) {
        Module module = request.getApplicationContext().getModuleByName(ModuleConstances.MIGRATION_MODULE);
        ModuleContext moduleContext = module.getModuleContext();

        migrationManager = (MigrationManager) moduleContext.getAttribute(MigrationManager._MigrationManager);
    }

    migrationManager.setPort(request.getServerPort());

    MigrationActionData actionData = (MigrationActionData) request.getActionData();

    if (logger.isInfoEnabled()) {
        logger.info("Receive command, Type:  " + actionData.getActionType() + "," + actionData.getSubcommand()
                    + ", DataServer port: " + request.getServerPort());
    }

    if (!checkSubCommand(actionData)) {
        if (logger.isDebugEnabled()) logger.debug("Invalid migrate subcommand " + actionData.getSubcommand());
        response.write(actionData);
        return;
    }

    String retMsg = null;

    if (actionData.isCancelCommand()) {
        retMsg = migrationManager.cancelMigrate(actionData);

    } else if (actionData.isAllFinishedCommand()) {
        retMsg = migrationManager.allFinishMigrate(actionData);

    } else if (actionData.isStartCommand()) { //开始迁移
        // start migration
        retMsg = migrationManager.startMigrate(actionData);

    } else if (actionData.isDataCleanCommand()) {
        retMsg = migrationManager.dataClean(actionData);

    } else {
        retMsg = migrationManager.queryStatus(actionData);
    }
    actionData.setReturnMessage(retMsg);
    response.write(actionData);
}
```

### 开始迁移

com.alibaba.doris.dataserver.migrator.MigrationManager#startMigrate

```java
public String startMigrate(MigrationActionData migrationActionData) {
    controlLock.lock();
    try {
        String retMsg = startMigrate0(migrationActionData);
        return retMsg;
    } finally {
        controlLock.unlock();
    }
}
```

com.alibaba.doris.dataserver.migrator.MigrationManager#startMigrate0

```java
private String startMigrate0(MigrationActionData migrationActionData) {
    String retMsg = null;
    MigrateSubCommand subCommand = migrationActionData.getSubcommand();

    if (logger.isDebugEnabled()) {
        logger.debug("Receive new migration task, type " + subCommand + ", route:" + migrationActionData);
    }

    migrationTaskScheduler.checkAndTerminateFinishedTask();
    //创建迁移任务
    BaseMigrationTask newTask = taskFactory.createTask(this, migrationActionData);

    if (!migrationTaskScheduler.hasActiveTask()) { //没有活跃的迁移任务
        if (logger.isDebugEnabled()) {
            logger.debug("There is no active migration task. Preprare to start new one." + newTask);
        }

        retMsg = startTask0(migrationActionData, newTask); //启动任务
        migrationActionData.setSuccess(true);

        if (logger.isDebugEnabled()) {
            logger.debug("Migration task started. " + newTask.getTaskName() + ". " + newTask);
        }
    } else { // 判断优先级，如果新任务优先级高，则取消当前执行的任务，执行新的任务
        MigrationTask lastActiveTask = migrationTaskScheduler.getLastTask();

        int newTaskPriority = newTask.getMigrateType().getPriority();
        int activeTaskPriority = lastActiveTask.getMigrateType().getPriority();

        if (newTaskPriority < activeTaskPriority) {
            if (logger.isDebugEnabled()) {
                logger.debug("An active migration task exists, the active will cancel and new will start.Active Task: "
                             + lastActiveTask + ", New Task:" + newTask);
            }
            return cancelTaskAndStartNewOne(migrationActionData, lastActiveTask, newTask);
        } else if (newTaskPriority == activeTaskPriority) {
            // 都是临时失效回迁, 同时起多个任务
            if (lastActiveTask.getMigrateType() == MigrateTypeEnum.TEMP_FAILOVER
                && newTask.getMigrateType() == MigrateTypeEnum.TEMP_FAILOVER) {
                // 查看新启动的迁移任务是否已经存在；注释：修复原有的某些情况下，重新启动临时失效回迁会启动新任务的bug；
                MigrationTask existsTask = migrationTaskScheduler.getTask(newTask.getTaskKey());

                // 如果待启动的迁移任务已经存在；
                if (null != existsTask) {
                    // 如果当前task正在等待数据清理，再次收到迁移开始指令，需要取消原有task并重新启动一个迁移任务。
                    if (existsTask.getMigrateStatus() == NodeMigrateStatus.MIGRATE_NODE_FINISHED) {
                        retMsg = cancelTaskAndStartNewOne(migrationActionData, existsTask, newTask);
                    } else {
                        // 否则只打印一条任务已经存在的log信息就返回；
                        if (logger.isDebugEnabled()) {
                            logger.debug("A Tempfailover migration task exists. And same migration route is request. It's rejected!. ActiveTask:"
                                         + existsTask + ",newTask:" + newTask);
                        }
                        retMsg = Message._MIGRATION_SAME_TASK_EXISTS;
                    }

                    return retMsg;
                } else {// 是一个新的临时失效迁移任务；
                    if (logger.isDebugEnabled()) {
                        logger.debug("Start a new migration task. newTask:" + newTask);
                    }

                    startTask0(migrationActionData, newTask);
                    retMsg = Message._MIGRATION_NEW_TASK;
                    return retMsg;
                }
            } else {
                if (logger.isDebugEnabled()) {
                    logger.debug("An same prior migration task exists. Ignore new one.   ActiveTask:"
                                 + lastActiveTask + ", newTask:" + newTask);
                }

                retMsg = Message._MIGRATION_SAME_TASK_EXISTS;
                return retMsg;
            }
        } else {
            if (logger.isDebugEnabled()) {
                logger.debug("An prior active migration task exists. New command ir rejected! " + lastActiveTask);
            }

            retMsg = Message._MIGRATION_PRIOR_TASK_EXISTS;
        }
    }

    return retMsg;
}
```

### 启动迁移任务

com.alibaba.doris.dataserver.migrator.MigrationManager#startTask0

```java
private String startTask0(MigrationActionData migrationActionData, BaseMigrationTask newTask) {

    String retMsg = Message._MIGRATION_NEW_TASK;

    // newTask.setMigrateStatus( NodeMigrateStatus.MIGRATING );
    newTask.setProgress(0);

    //向Admin上报迁移的状态
    MigrationListener reportListener = new DefaultMigrationListener();
    reportListener.setMigrationManager(this);

    newTask.addListener(this); // MigrationManager 作为监听器.
    newTask.addListener(reportListener); // 报告监听器

    /**
     * 1.启动正式的迁移任务前先准备好连接；<br>
     * 2.将任务加入到迁移Task列表中，这样前端的代理将正式生效； <br>
     * 3.提交给Executor，执行真正的迁移任务；
     */
    if (newTask.prepareTask()) {
        migrationTaskScheduler.addMigrationTask(newTask);
        executor.execute(newTask);

        if (logger.isDebugEnabled()) {
            logger.debug("Start new migration task." + newTask);
        }
    } else {
        throw new RuntimeException("Prepare task connection failed!");
    }

    retMsg = Message._MIGRATION_NEW_TASK;
    return retMsg;
}
```

com.alibaba.doris.dataserver.migrator.task.BaseMigrationTask#run

```java
public void run() {
    // do
    try {
        // start0
        if (begin() == false) {
            if (logger.isDebugEnabled()) {
                logger.debug("Fail to start new migration task thread.  " + getMigrateType() + ", currrent state: "
                             + migrateStatus);
            }

            return;
        }

        if (logger.isDebugEnabled()) {
            logger.debug("Start a migrate task " + getMigrateType() + ",route: "
                         + migrationActionData.getMigrationRouteString());
        }

        migrate();

        // finish
        finish();

        dataCleanPrepare();
    } catch (MigrationException e) {
        logger.error("Migration Task Error:" + e, e);
        notifyMigrateError(e.toString());

    } catch (Throwable t) {
        logger.error("Migration Task Error:" + t, t);
        notifyMigrateError(t.toString());

    } finally {
        if (logger.isInfoEnabled()) {
            logger.info("Migration Task Thread Finished, and exit.");
        }

        finished = true;
        finishTime = System.currentTimeMillis();
        if (migrationReportTimer != null) {
            migrationReportTimer.cancel();
        }

        notifyExitMigrationTask();

        // 通知退出任务后等待2s再释放所有连接；
        sleeps(2000);

        // 释放代理过程中用到的连接；
        releaseProxyConnection();
    }
}
```

com.alibaba.doris.dataserver.migrator.task.BaseMigrationTask#migrate

```java
public void migrate() throws MigrationException {
    controlLock.lock();
    try {
        migrate0();
    } finally {
        controlLock.unlock();
    }
}
```

com.alibaba.doris.dataserver.migrator.task.BaseMigrationTask#migrate0

```java
private void migrate0() throws MigrationException {
    migrationReportTimer = getMigrationReportTimer();
    migrationReportTimer.start(); //定时触发event，上报迁移的状态

    dataMigrationExecutor = getDataMigrationExecutor();
    dataMigrationExecutor.execute(); //迁移数据
}
```

### 切分任务

com.alibaba.doris.dataserver.migrator.task.DataMigrationExecutor#migrateAllVNodes

```java
protected void migrateAllVNodes(List<MigrationRoutePair> pairs) throws MigrationException {
    int totoalMCount = 0;
    List<TaskItem> taskList = generateTasks(pairs); //切分成多个子任务

    ExecutorService executor = Executors.newFixedThreadPool(taskList.size() + 1);
    boolean isCatchException = false;

    try {
        List<Future<Integer>> taskFuterList = new ArrayList<Future<Integer>>(taskList.size());
        BlockingQueue<Pair> dataQueue = new ArrayBlockingQueue<Pair>(10000);
        List<Integer> vnodeList = new ArrayList<Integer>(pairs.size());
        //提交迁移数据的任务到线程池
        for (TaskItem task : taskList) {
            vnodeList.addAll(task.getVnodeList());
            Callable<Integer> callableTask = new MigrateDataThread(migrationTask, task.getMachine(), dataQueue);
            taskFuterList.add(executor.submit(callableTask));
        }
        //读数据
        readDataThread = new ReadDataThread(storage, vnodeList, dataQueue, taskList.size());
        executor.submit(readDataThread);

        // 阻塞并等待各子任务的执行结果。
        for (int index = 0; index < taskFuterList.size(); index++) {
            Future<Integer> future = taskFuterList.get(index);
            try {
                totoalMCount += future.get().intValue();

                TaskItem task = taskList.get(index);
                List<Integer> taskVNodeList = task.getVnodeList();
                // 迁移完所有虚拟节点 ，计算迁移进度
                for (Integer vnode : taskVNodeList) {
                    progressComputer.completeOneVNode(task.getMachine(), vnode);
                }
                //上报迁移的进度
                migrationTask.notifyMigrateProgress(progressComputer.getGrossProgress());
            } catch (Exception e) {
                logger.error("Waiting migrate result failed. we need throw this exception", e);
                isCatchException = true;
                // 一旦迁移过程中，某个线程发生异常退出，立即终止所有的迁移线程，终止迁移过程。
                readDataThread.stopReadThread();
            }
        }
    } finally {
        executor.shutdownNow();
    }

    if (isCatchException) {
        throw new MigrationException("Migrating data failed.");
    }
    //上报迁移完成
    migrationTask.notifyMigrateNodeFinish();

    if (logger.isInfoEnabled()) {
        logger.info("Complete data " + getOperationName() + " all all vnodes  " + totoalMCount + ", vnodes count: "
                    + pairs.size() + ". ");
    }
}
```

### 迁移数据

com.alibaba.doris.dataserver.migrator.task.MigrateDataThread#call

```java
public Integer call() throws Exception {
    // 获取和目标机器的连接
    Connection connection = connectionManager.getConnection(targetMachine);

    int mcount = 0;
    long start = System.currentTimeMillis();
    long totalTime = 0;

    while (true) {
        Pair pair = dataQueue.take();
        if (pair instanceof ExitPair) { //退出标志
            break;
        }

        mcount++; //迁移的数据量

        if (migrationTask.isCancel()) { //迁移任务优先级低，被删除
            if (logger.isInfoEnabled()) {
                logger.info("Migrate Task has been cancelled.Terminate it.");
            }
            break;
        }

        // TODO: 待puts实现后，可以批量传输，以加快迁移速度
        // 迁移一条数据到目标机器
        long begin = System.currentTimeMillis();
        try {
            migrateDataEntry(connection, pair); //向目标节点发送数据请求
            totalTime += System.currentTimeMillis() - begin;
        } catch (MigrationException e) {
            if (e.getCause() instanceof NetException) { //网络异常，重试
                logger.error("Fail to Migrate data " + pair.getKey().getPhysicalKey() + ". Cause:" + e.getCause(),
                             e.getCause());
                Thread.sleep(RE_CONNECTE_INTERVAL);
                connection = connectionManager.getConnection(targetMachine);
                if (!connection.isConnected()) {
                    Thread.sleep(RE_CONNECTE_INTERVAL);
                    connection = connectionManager.getConnection(targetMachine);
                    if (!connection.isConnected()) {
                        throw new MigrationException("Migration failed bacause of  target connection close."
                                                     + targetMachine);
                    }
                }

                migrateDataEntry(connection, pair);
            } else {
                logger.error("Fail to Migrate data " + pair.getKey().getPhysicalKey() + ". Cause:" + e, e);
                throw e;
            }
        }
    }

    long end = System.currentTimeMillis();
    long ellapse = end - start;
    if (logger.isDebugEnabled()) {
        logger.debug("--------Migrate  data on storage by vnodes , to " + targetMachine + ",record count:" + mcount
                     + ", time:" + ellapse + "  write remote data(ms):" + totalTime);
    }

    return mcount;
}
```

















