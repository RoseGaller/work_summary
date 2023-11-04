# DDMQ

# 故障自动切换

## HAManager

org.apache.rocketmq.namesrv.ha.HAManager#start

```java
public void start() throws Exception {
    detector.start();
    stateKeeper.start();
    roleManager.start();
    haScheduledThread.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                HAManager.this.doHa();
            } catch (Throwable throwable) {
                log.error("do ha failed", throwable);
            }
        }
    }, namesrvController.getNamesrvConfig().getDetectIntervalMs(), namesrvController.getNamesrvConfig().getDetectIntervalMs(), TimeUnit.MILLISECONDS);
}
```

## StateKeeper

org.apache.rocketmq.namesrv.ha.StateKeeper#start

```java
public void start() throws Exception {
    if (StringUtils.isEmpty(namesrvController.getNamesrvConfig().getClusterName())
        || StringUtils.isEmpty(namesrvController.getNamesrvConfig().getZkPath())) {
        log.error("clusterName:{} or zk path:{} is empty",
            namesrvController.getNamesrvConfig().getClusterName(), namesrvController.getNamesrvConfig().getZkPath());
        throw new Exception("cluster name or zk path is null");
    }
    hostName = getHostName();
    //与zk集群建立连接
    zkClient = CuratorFrameworkFactory.newClient(namesrvController.getNamesrvConfig().getZkPath(),
        SESSION_TIMEOUT_MS, CONNECTION_TIMEOUT_MS, new ExponentialBackoffRetry(RETRY_INTERVAL_MS, RETRY_COUNT));
    zkClient.getConnectionStateListenable().addListener(new StateListener());
    zkClient.start();
		//创建根目录
    createRootPath();
    //选主
    registerLeaderLatch();
}
```

```java
private void createRootPath() throws Exception {	//创建根目录
    if (zkClient.checkExists().forPath(getRootPath()) == null) {
       // "/" + namesrvController.getNamesrvConfig().getClusterName();
        zkClient.create().forPath(getRootPath());
    }
}
```

```java
public synchronized void registerLeaderLatch() throws Exception { //选主
    if (zkClient.checkExists().forPath(getLeaderPath()) == null) {
        zkClient.create().forPath(getLeaderPath());
    }
    leaderLatch = new LeaderLatch(zkClient, getLeaderPath(), hostName);
    leaderLatch.start();

    log.info("leaderLatch start");
}
```

StateListener

org.apache.rocketmq.namesrv.ha.StateKeeper.StateListener#stateChanged

```go
@Override public void stateChanged(CuratorFramework client, ConnectionState newState) {
    log.warn("state changed, state:{}", newState);
    if (ConnectionState.CONNECTED == newState || ConnectionState.RECONNECTED == newState) {
        log.warn("session change:{}, register again", newState);
        registerAll(); //注册此NameServer认为存活的Broker
    }
}
```

```java
public synchronized void registerAll() {
    for (Map.Entry<String, Set<String>> entry : aliveBrokers.entrySet()) {
        try {
            if (zkClient.checkExists().forPath(getNSPath(entry.getKey())) == null) {
                zkClient.create().withMode(CreateMode.EPHEMERAL).forPath(getNSPath(entry.getKey()), toJsonString(entry.getValue()).getBytes());
            }
        } catch (Exception ex) {
            log.error("create node fail in registerAll", ex);
        }
    }
}
```

## 周期探测

```java
private void doHa() throws Exception {
    //通过向主节点发送消息，探测主节点的存活状态
    HealthStatus healthStatus = detector.detectLiveStat();
    //cluterName -> (BrokerName,boolean（true:存活))
    Map<String, Map<String, Boolean>> liveStatus = getLiveBroker(healthStatus);
    //角色转换
    checkAndSwitchRole(liveStatus);
     //把本机探测的主节点的健康状态注册到zk
    for (Map.Entry<String, Map<String, Boolean>> entry : liveStatus.entrySet()) {
        updateBrokerStatus(entry.getKey(), entry.getValue());
    }
}
```

### 探测主Broker状态

org.apache.rocketmq.namesrv.ha.Detector#detectLiveStat

```java
public HealthStatus detectLiveStat() throws Exception {
    if (producer == null) {
        producer = new DefaultMQProducer("BrokerDetector");
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();
    }
    if (routeInfoManager == null) {
        log.warn("routeInfoManager is null");
        return null;
    }
    //clusterName -> Set<brokerName>
    HashMap<String, Set<String>> clusterAddrTable = routeInfoManager.getClusterAddrTable();
    for (Map.Entry<String, Set<String>> entry : clusterAddrTable.entrySet()) {
        String clusterName = entry.getKey();
        log.info("begin to detect cluster:{}", clusterName);
        for (String brokerName : entry.getValue()) {
            log.info("begin to detect broker:{}", brokerName);
            //brokerName ->BrokerData
            BrokerData brokerData = routeInfoManager.getBrokerData(brokerName);
            if (brokerData == null) {
                log.warn("get broker data failed, broker name:{}", brokerName);
                continue;
            }
            //主节点brokerId=0，获取主节点地址
            //<brokerId> -> <broker address>
            String brokerAddr = brokerData.getBrokerAddrs() == null ? null : brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
            if (brokerAddr == null) {//此Cluster下无Broker或者无主Broker
                log.warn("no master of broker name:{}", brokerName);
                healthStatus.updateHealthStatus(clusterName, brokerName, false);
                continue;
            }
            //向主Broker上的此主题发送消息，探测主节点状态
            String topicName = MixAll.LIVE_STATE_DETECT_TOPIC;
            Message msg = new Message(topicName, String.valueOf(System.currentTimeMillis()).getBytes());
            msg.setWaitStoreMsgOK(false);//do not wait sync to slave
            MessageQueue mq = new MessageQueue(topicName, brokerName, 0);
            SendMessageRequestHeader requestHeader = getMessageRequestHeader(msg, mq);
            boolean sendOk = true;
            try {
                SendResult sendResult = producer.getDefaultMQProducerImpl().getmQClientFactory().getMQClientAPIImpl().sendMessage(
                    brokerAddr,
                    mq.getBrokerName(),
                    msg,
                    requestHeader,
                    namesrvController.getNamesrvConfig().getDetectMessageSendTimeoutMs(),
                    CommunicationMode.SYNC,
                    null,
                    null);
                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                    log.error("send message failed, status:{}", sendResult.getSendStatus());
                    sendOk = false;
                }
            } catch (MQBrokerException brokerException) {
                if (brokerException.getResponseCode() == ResponseCode.NO_PERMISSION || brokerException.getResponseCode() == ResponseCode.TOPIC_NOT_EXIST) {
                    log.warn("brokerAddr {} is not writable, cause:{}", brokerAddr, brokerException.getErrorMessage());
                } else {
                    log.error("send message failed", brokerException);
                    sendOk = false;
                }
            } catch (Exception ex) {
                log.error("send message failed", ex);
                sendOk = false;
            }
            //更改此主节点的健康状态
            //cluster name ->(broker name -> NodeHealthStatus)
            healthStatus.updateHealthStatus(clusterName, brokerName, sendOk);
        }
        log.info("detect cluster:{} end", clusterName);
    }
    //delete not in routeInfoManager
    healthStatus.deleteDead();
    return healthStatus;
}
```

org.apache.rocketmq.namesrv.ha.HealthStatus#updateHealthStatus

```java
public void updateHealthStatus(String clusterName, String brokerName, boolean status) {
     //修改集群下Broker的状态
    if (clusterStatus.get(clusterName) == null) {
        Map<String, NodeHealthStatus> brokerInfo = new HashMap<>(16);
        NodeHealthStatus nodeHealthStatus = new NodeHealthStatus();
        brokerInfo.put(brokerName, nodeHealthStatus);
        clusterStatus.put(clusterName, brokerInfo);
    } else if (clusterStatus.get(clusterName).get(brokerName) == null) {
        NodeHealthStatus nodeHealthStatus = new NodeHealthStatus();
        clusterStatus.get(clusterName).put(brokerName, nodeHealthStatus);
    } else {
        clusterStatus.get(clusterName).get(brokerName).updateStatus(status);
    }
}
```

### 检测并转换角色

```java
private void checkAndSwitchRole(Map<String, Map<String, Boolean>> liveStatus) {
    for (Map.Entry<String, Map<String, Boolean>> cluster : liveStatus.entrySet()) {
        if (!stateKeeper.isLeader()) {
            continue;
        }
        String clusterName = cluster.getKey();
        for (Map.Entry<String, Boolean> broker : cluster.getValue().entrySet()) {
            if (broker.getValue()) {
                continue;
            }
            String brokerName = broker.getKey();
            log.info("this name server detect that the broker is unhealthy, broker name:{}, cluster:{}", brokerName, clusterName);
            if (!isInRouteInfoManager(clusterName, brokerName)) {
                log.warn("not in route info, not need to change");
                continue;
            }
            //根据所有nameserver探测的结果，来决定broker的存活状态
            boolean isHealthyInOtherNS = stateKeeper.isHealthyInOtherNS(clusterName, brokerName);
            log.info("healthy status in other ns:{}", isHealthyInOtherNS);
            if (!isHealthyInOtherNS) {
                log.warn("broker is unhealthy in other ns, broker name:{}, cluster:{}", brokerName, clusterName);
            }
            //选择主节点，切换主从角色
            if (!isHealthyInOtherNS && isSwitchRole(clusterName, brokerName)) {
                RoleChangeInfo roleChangeInfo = selectNewMaster(clusterName, brokerName);
                if (roleChangeInfo == null) {
                    log.warn("can not get a new master, clusterName:{}, brokerName:{}", clusterName, brokerName);
                    continue;
                }
                log.info("nodes would be changed {}", roleChangeInfo);
                if (roleChangeInfo.oldMaster == null) {
                    log.warn("no old master, just change a slave to master");
                    //slave to new master
                    if (!roleManager.change2Master(brokerName, roleChangeInfo.newMaster.addr, true)) {
                        log.error("change slave to master failed, stop. clusterName:{}, brokerName:{}, brokerAddr:{}", clusterName,
                            brokerName, roleChangeInfo.newMaster.addr);
                        continue;
                    }
                } else {
                    //向主节点发送UPDATE_BROKER_CONFIG请求，角色改为slave
                    if (!roleManager.change2Slave(brokerName, roleChangeInfo.oldMaster.addr, roleChangeInfo.oldMaster.expectId, false)) {
                        log.error("change master to slave failed, stop. clusterName:{}, brokerName:{}, brokerAddr:{}", clusterName,
                            brokerName, roleChangeInfo.oldMaster.addr);
                        continue;
                    }
                    //向从节点发送UPDATE_BROKER_CONFIG请求，角色改为master
                    if (!roleManager.change2Master(brokerName, roleChangeInfo.newMaster.addr, true)) {
                        log.error("change slave to master failed, stop. clusterName:{}, brokerName:{}, brokerAddr:{}", clusterName,
                            brokerName, roleChangeInfo.newMaster.addr);
                        continue;
                    }
                    //change new slave id
                    long slaveId;
                    BrokerData brokerData = namesrvController.getRouteInfoManager().getBrokerData(brokerName);
                    if (brokerData != null) {
                        SortedSet<Long> ids = new TreeSet<>(brokerData.getBrokerAddrs().keySet());
                        slaveId = selectSlaveId(ids);
                    } else {
                        slaveId = roleChangeInfo.newMaster.oldId;
                    }
                  	//为旧的master分配salveId
                    if (!roleManager.changeId(brokerName, roleChangeInfo.oldMaster.addr, slaveId, false)) {
                        log.error("change id failed, stop. clusterName:{}, brokerName:{}, brokerAddr:{}", clusterName,
                            brokerName, roleChangeInfo.oldMaster.addr);
                        continue;
                    }
                }
                log.info("cluster:{}, broker:{}, change role success", clusterName, brokerName);
                //clear old detect info
                detector.reset(clusterName, brokerName);
            }
        }
    }
}
```

org.apache.rocketmq.namesrv.ha.StateKeeper#isHealthyInOtherNS

```java
public boolean isHealthyInOtherNS(String clusterName, String broker) {
  //其他NameServer检测到的Broker状态
    List<String> nameServers = null;
    try {
      	//集群下的所有Nameserver
        nameServers = zkClient.getChildren().forPath(getIdsPath(clusterName));
    } catch (Exception ex) {
        log.error("get child failed", ex);
    }

    if (nameServers == null || nameServers.size() < NAME_SERVER_SIZE_MIN) {
        log.warn("name servers would not be null or empty, nameServers:{}", nameServers);
        return true;
    }

    int healthyCount = 0;
    for (String ns : nameServers) {
        if (hostName.equals(ns)) {
            continue;
        }
      	//其他Nameserver
        try {
          	//获取Nameserver下的Broker
            Set brokers = fromJsonBytes(zkClient.getData().forPath(getNSPath(clusterName, ns)), Set.class);
            if (brokers != null && brokers.contains(broker)) {
                healthyCount++;
                log.info("host:{} think it is healthy, cluster:{}, broker:{}", ns, clusterName, broker);
            }
        } catch (Exception ex) {
            log.warn("get data failed, path:" + getNSPath(clusterName, ns), ex);
        }
    }
		//不健康的个数
    int unhealthyCount = nameServers.size() - healthyCount;
  //最大的不健康个数
    int maxUnhealthyCount = (int) Math.ceil(nameServers.size() * namesrvController.getNamesrvConfig().getUnhealthyRateAllNs());

    log.info("total count:{}, unhealthy count:{}, unhealthy rate:{}, threshold:{}", nameServers.size(),
        nameServers.size() - healthyCount, namesrvController.getNamesrvConfig().getUnhealthyRateAllNs(), maxUnhealthyCount);

    if (unhealthyCount >= maxUnhealthyCount) { //Broker不健康
        return false;
    }

    return true;
}
```

org.apache.rocketmq.namesrv.ha.HAManager#isSwitchRole

```java
private boolean isSwitchRole(String clusterName, String brokerName) {
  	//是否自动切换角色
    if (namesrvController.getNamesrvConfig().isRoleAutoSwitchEnable()) {//默认false
        return true;
    }

    boolean isSwitch = false;
    String key = getClusterBrokerKey(clusterName, brokerName);
    if (brokerEnableSwitch.containsKey(key)) { //启用切换角色，默认15秒内有效
        if (System.currentTimeMillis() - brokerEnableSwitch.get(key) < namesrvController.getNamesrvConfig().getEnableValidityPeriodMs()) {
            log.info("broker:{} enable to switch", brokerName);
            isSwitch = true;
        }
        brokerEnableSwitch.remove(key);
    }

    return isSwitch;
}
```

从slave中选举新的master

org.apache.rocketmq.namesrv.ha.HAManager#selectNewMaster

```java
public RoleChangeInfo selectNewMaster(String cluster, String brokerName) {
		//根据BrokerName获取所有的Broker
    BrokerData brokerData = namesrvController.getRouteInfoManager().getBrokerData(brokerName);
    if (brokerData == null || !cluster.equals(brokerData.getCluster())) {
        log.warn("no broker data for broker name:{}, broker data:{}", brokerName, brokerData);
        return null;
    }

    HashMap<Long, String> brokerAddrs = new HashMap<>(brokerData.getBrokerAddrs());
    for (Iterator<Map.Entry<Long, String>> it = brokerAddrs.entrySet().iterator(); it.hasNext(); ) {
        Map.Entry<Long, String> item = it.next();
        if (item.getKey() > namesrvController.getNamesrvConfig().getMaxIdForRoleSwitch()) {
            it.remove();
        }
    }

    //无broker
    if (brokerAddrs == null || brokerAddrs.isEmpty()) {
        log.warn("no broker addrs, for broker name:{}, broker data:{}", brokerName, brokerData);
        return null;
    }
	
    //只有一个master
    if (brokerAddrs.size() == 1 && brokerAddrs.get(MixAll.MASTER_ID) != null) {
        log.warn("only on broker, but it is current master");
        return null;
    }

    //slave exist
    RoleChangeInfo roleChangeInfo = new RoleChangeInfo();
    SortedSet<Long> ids = new TreeSet<>(brokerAddrs.keySet());
    if (ids.first() == MixAll.MASTER_ID) {//MASTER_ID=0L
        roleChangeInfo.oldMaster = new RoleInChange(brokerAddrs.get(ids.first()), ids.first(), ids.last() + 1);
    }
		//选择master
    long newMasterId = pickMaster(brokerAddrs);
    if (newMasterId == -1) { //没有选出master
        //newMasterId = ids.last();
        log.error("do not get master, broker name:{}", brokerName);
        return null;
    }
    roleChangeInfo.newMaster = new RoleInChange(brokerAddrs.get(newMasterId), newMasterId, MixAll.MASTER_ID);

    return roleChangeInfo;
}
```

```go
private long pickMaster(HashMap<Long, String> brokerAddrs) {//挑选master
    long maxOffset = -1;
    long brokerId = -1;
    for (Map.Entry<Long, String> broker : brokerAddrs.entrySet()) {
        if (broker.getKey() == MixAll.MASTER_ID) { //过滤掉旧的master
            continue;
        }
        long offset = namesrvController.getRouteInfoManager().getBrokerMaxPhyOffset(broker.getValue());
        if (offset > maxOffset) { //获取offset最大的broker
            brokerId = broker.getKey();
            maxOffset = offset;
        }
    }
    log.info("get new master id:{}, maxOffset:{}", brokerId, maxOffset);
    return brokerId;
}
```

# 延迟消息

```
开发者使用DDMQ的生产SDK将延迟消息生产到PProxy中，然后PProxy会将延迟消息写入到提前配置好的 
inner topic 中。之后 Chronos 会消费 inner topic的消息并存储到内置的 RocksDB 存储引擎中。
Chronos 内置的时间轮服务会将到期的消息再次发送给 DDMQ 供业务方消费。
```

ChronosMain

com.xiaojukeji.chronos.ChronosMain#main

```java
public static void main(String[] args) {
    Thread.setDefaultUncaughtExceptionHandler((thread, exception) ->
            LOGGER.error("UncaughtException in Thread " + thread.toString(), exception));

    if (args.length < 1) {
        LOGGER.error("params error!");
        return;
    }

    ChronosStartup startup = new ChronosStartup(args[0]);
    try {
        startup.start();
    } catch (Exception e) {
        LOGGER.error("error while start chronos, err:{}", e.getMessage(), e);
        startup.stop();
    }
}
```

ChronosStartup

com.xiaojukeji.chronos.ChronosStartup#start

```java
public void start() throws Exception {
    LOGGER.info("start to launch chronos...");
    final long start = System.currentTimeMillis();

    Runtime.getRuntime().addShutdownHook(new Thread() {
        @Override
        public void run() {
            try {
                LOGGER.info("start to stop chronos...");
                final long start = System.currentTimeMillis();
                ChronosStartup.this.stop();
                final long cost = System.currentTimeMillis() - start;
                LOGGER.info("succ stop chronos, cost:{}ms", cost);
            } catch (Exception e) {
                LOGGER.error("error while shutdown chronos, err:{}", e.getMessage(), e);
            } finally {
                /* shutdown log4j2 */
                LogManager.shutdown();
            }
        }
    });

    /* 注意: 以下初始化顺序有先后次序 */

    /* init config */
    ConfigManager.initConfig(configFilePath);

    /* init metrics */
    if (!MetricService.init()) {
        System.exit(-1);
    }

    /* init rocksdb */
    RDB.init(ConfigManager.getConfig().getDbConfig().getDbPath());

    /* init zk */
    ZkUtils.init();

    /* init seektimestamp */
    MetaService.load();

    waitForShutdown = new CountDownLatch(1);

    if (ConfigManager.getConfig().isStandAlone()) {
        /* standalone */
        MasterElection.standAlone();
    } else {
        /* 集群模式 master election */
        MasterElection.election(waitForShutdown);
    }

    /* init pull worker */
    if (ConfigManager.getConfig().isPullOn()) {
        pullWorker = PullWorker.getInstance();
        pullWorker.start();
    }

    /* init push worker */
    if (ConfigManager.getConfig().isPushOn()) {
        pushWorker = PushWorker.getInstance();
        pushWorker.start();
    }

    /* init delete worker */
    if (ConfigManager.getConfig().isDeleteOn()) {
        deleteBgWorker = DeleteBgWorker.getInstance();
        deleteBgWorker.start();
    }

    final long cost = System.currentTimeMillis() - start;
    LOGGER.info("succ start chronos, cost:{}ms", cost);

    /* init http server */
    nettyHttpServer = NettyHttpServer.getInstance();
    nettyHttpServer.start();

    waitForShutdown.await();
}
```

