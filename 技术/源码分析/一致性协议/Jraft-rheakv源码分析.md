# Jraft-rheakv源码分析

* [服务端启动](#服务端启动)
  * [DefaultRheaKVStore](#defaultrheakvstore)
    * [RegionRouteTable初始化](#regionroutetable初始化)
    * [StoreEngine初始化](#storeengine初始化)
      * [注册KV处理器](#注册kv处理器)
      * [创建RawKVStore](#创建rawkvstore)
      * [初始化RegionEngine](#初始化regionengine)
        * [RegionEngine选举](#regionengine选举)
    * [HeartbeatSender初始化](#heartbeatsender初始化)
      * [StoreHeartbeat](#storeheartbeat)
      * [RegionHeartbeat](#regionheartbeat)
* [请求处理](#请求处理)
  * [GetRequest](#getrequest)
  * [PutRequest](#putrequest)
  * [RangeSplit](#rangesplit)
* [总结](#总结)


# 服务端启动

```java
public class Server1 {

    public static void main(final String[] args) throws Exception {
        final PlacementDriverOptions pdOpts= 			PlacementDriverOptionsConfigured.newConfigured()
                .withFake(true) //表示在无PD模式下启动
                .config();
        final StoreEngineOptions storeOpts = StoreEngineOptionsConfigured.newConfigured() //
                .withStorageType(StorageType.RocksDB)      .withRocksDBOptions(RocksDBOptionsConfigured.newConfigured().withDbPath(Configs.DB_PATH).config())
                .withRaftDataPath(Configs.RAFT_DATA_PATH)
                .withServerAddress(new Endpoint("127.0.0.1", 8181))
                .config();
        final RheaKVStoreOptions opts = RheaKVStoreOptionsConfigured.newConfigured() //
                .withClusterName(Configs.CLUSTER_NAME) //
                .withInitialServerList(Configs.ALL_NODE_ADDRESSES)
                .withStoreEngineOptions(storeOpts) //存储配置
                .withPlacementDriverOptions(pdOpts) //PD配置
                .config();
        System.out.println(opts);
        final Node node = new Node(opts);
        node.start();
        Runtime.getRuntime().addShutdownHook(new Thread(node::stop));
        System.out.println("server1 start OK");
    }
}
```

com.alipay.sofa.jraft.example.rheakv.Node#start

```java
public void start() {
    this.rheaKVStore = new DefaultRheaKVStore();
    this.rheaKVStore.init(this.options);
}
```

## DefaultRheaKVStore

com.alipay.sofa.jraft.rhea.client.DefaultRheaKVStore#init

```java
public synchronized boolean init(final RheaKVStoreOptions opts) {
    if (this.started) {
        LOG.info("[DefaultRheaKVStore] already started.");
        return true;
    }
    this.opts = opts;
    // 初始化PD
    final PlacementDriverOptions pdOpts = opts.getPlacementDriverOptions();
    final String clusterName = opts.getClusterName();
    Requires.requireNonNull(pdOpts, "opts.placementDriverOptions");
    Requires.requireNonNull(clusterName, "opts.clusterName");
    if (Strings.isBlank(pdOpts.getInitialServerList())) {
        // if blank, extends parent's value
        pdOpts.setInitialServerList(opts.getInitialServerList());
    }

    //创建PD客户端
    if (pdOpts.isFake()) {
        this.pdClient = new FakePlacementDriverClient(opts.getClusterId(), clusterName);
    } else {
        this.pdClient = new RemotePlacementDriverClient(opts.getClusterId(), clusterName);
    }
    //初始化region路由表
    if (!this.pdClient.init(pdOpts)) {
        LOG.error("Fail to init [PlacementDriverClient].");
        return false;
    }

    // 初始化存储引擎
    final StoreEngineOptions stOpts = opts.getStoreEngineOptions();
    if (stOpts != null) {
        stOpts.setInitialServerList(opts.getInitialServerList());
        this.storeEngine = new StoreEngine(this.pdClient, this.stateListenerContainer);
        if (!this.storeEngine.init(stOpts)) {
            LOG.error("Fail to init [StoreEngine].");
            return false;
        }
    }

    final Endpoint selfEndpoint = this.storeEngine == null ? null : this.storeEngine.getSelfEndpoint();
    final RpcOptions rpcOpts = opts.getRpcOptions();
    Requires.requireNonNull(rpcOpts, "opts.rpcOptions");

    //创建并初始化DefaultRheaKVRpcService
    this.rheaKVRpcService = new DefaultRheaKVRpcService(this.pdClient, selfEndpoint) {

        @Override
        public Endpoint getLeader(final long regionId, final boolean forceRefresh, final long timeoutMillis) {
            final Endpoint leader = getLeaderByRegionEngine(regionId);
            if (leader != null) {
                return leader;
            }
            return super.getLeader(regionId, forceRefresh, timeoutMillis);
        }
    };
    if (!this.rheaKVRpcService.init(rpcOpts)) {
        LOG.error("Fail to init [RheaKVRpcService].");
        return false;
    }

    this.failoverRetries = opts.getFailoverRetries();
    this.futureTimeoutMillis = opts.getFutureTimeoutMillis();
    this.onlyLeaderRead = opts.isOnlyLeaderRead();
    if (opts.isUseParallelKVExecutor()) {
        final int numWorkers = Utils.cpus();
        final int bufSize = numWorkers << 4;
        final String name = "parallel-kv-executor";
        final ThreadFactory threadFactory = Constants.THREAD_AFFINITY_ENABLED
                ? new AffinityNamedThreadFactory(name, true) : new NamedThreadFactory(name, true);
        this.kvDispatcher = new TaskDispatcher(bufSize, numWorkers, WaitStrategyType.LITE_BLOCKING_WAIT, threadFactory);
    }
    
    this.batchingOpts = opts.getBatchingOptions();
    if (this.batchingOpts.isAllowBatching()) { //默认true
        this.getBatching = new GetBatching(KeyEvent::new, "get_batching",
                new GetBatchingHandler("get", false));
        this.getBatchingOnlySafe = new GetBatching(KeyEvent::new, "get_batching_only_safe",
                new GetBatchingHandler("get_only_safe", true));
        this.putBatching = new PutBatching(KVEvent::new, "put_batching",
                new PutBatchingHandler("put"));
    }
    LOG.info("[DefaultRheaKVStore] start successfully, options: {}.", opts);
    return this.started = true;
}
```

### RegionRouteTable初始化

com.alipay.sofa.jraft.rhea.client.pd.RemotePlacementDriverClient#init

```java
public synchronized boolean init(final PlacementDriverOptions opts) {
    if (this.started) {
        LOG.info("[RemotePlacementDriverClient] already started.");
        return true;
    }
  	//本地配置中获取
    super.init(opts);
    this.pdGroupId = opts.getPdGroupId();
    if (Strings.isBlank(this.pdGroupId)) {
        throw new IllegalArgumentException("opts.pdGroup id must not be blank");
    }
    //PDServer地址列表
    final String initialPdServers = opts.getInitialPdServerList();
    if (Strings.isBlank(initialPdServers)) {
        throw new IllegalArgumentException("opts.initialPdServerList must not be blank");
    }
    RouteTable.getInstance().updateConfiguration(this.pdGroupId, initialPdServers);
  	//从PDServer获取集群下的region信息
    this.metadataRpcClient = new MetadataRpcClient(super.pdRpcService, 3);
    refreshRouteTable();
    LOG.info("[RemotePlacementDriverClient] start successfully, options: {}.", opts);
    return this.started = true;
}
```

### StoreEngine初始化

com.alipay.sofa.jraft.rhea.StoreEngine#init

```java
public synchronized boolean init(final StoreEngineOptions opts) { //初始化存储引擎
    if (this.started) {
        LOG.info("[StoreEngine] already started.");
        return true;
    }
    this.storeOpts = Requires.requireNonNull(opts, "opts");
    Endpoint serverAddress = Requires.requireNonNull(opts.getServerAddress(), "opts.serverAddress");
    final int port = serverAddress.getPort();
    final String ip = serverAddress.getIp();
    if (ip == null || Utils.IP_ANY.equals(ip)) {
        serverAddress = new Endpoint(NetUtil.getLocalCanonicalHostName(), port);
        opts.setServerAddress(serverAddress);
    }
    final long metricsReportPeriod = opts.getMetricsReportPeriod();

    // 初始化region配置
    List<RegionEngineOptions> rOptsList = opts.getRegionEngineOptionsList();
    if (rOptsList == null || rOptsList.isEmpty()) {
        //创建初始region
        final RegionEngineOptions rOpts = new RegionEngineOptions();
        rOpts.setRegionId(Constants.DEFAULT_REGION_ID); // -1
        rOptsList = Lists.newArrayList();
        rOptsList.add(rOpts);
        opts.setRegionEngineOptionsList(rOptsList);
    }
    //获取集群名称
    final String clusterName = this.pdClient.getClusterName();
    //填充region配置
    for (final RegionEngineOptions rOpts : rOptsList) {
        rOpts.setRaftGroupId(JRaftHelper.getJRaftGroupId(clusterName, rOpts.getRegionId()));
        rOpts.setServerAddress(serverAddress);
        rOpts.setInitialServerList(opts.getInitialServerList());
        if (rOpts.getNodeOptions() == null) {
            // copy common node options
            rOpts.setNodeOptions(opts.getCommonNodeOptions() == null ? new NodeOptions() : opts
                .getCommonNodeOptions().copy());
        }
        if (rOpts.getMetricsReportPeriod() <= 0 && metricsReportPeriod > 0) {
            // extends store opts
            rOpts.setMetricsReportPeriod(metricsReportPeriod);
        }
    }
    //从PD获取store或者根据本地配置创建store并上报至PD，strore存储本地所有的region,每个节点对应一个store
    final Store store = this.pdClient.getStoreMetadata(opts);
    if (store == null || store.getRegions() == null || store.getRegions().isEmpty()) {
        LOG.error("Empty store metadata: {}.", store);
        return false;
    }
    this.storeId = store.getId();
    // init executors
    if (this.readIndexExecutor == null) {
        this.readIndexExecutor = StoreEngineHelper.createReadIndexExecutor(opts.getReadIndexCoreThreads());
    }
    if (this.raftStateTrigger == null) {
        this.raftStateTrigger = StoreEngineHelper.createRaftStateTrigger(opts.getLeaderStateTriggerCoreThreads());
    }
    if (this.snapshotExecutor == null) {
        this.snapshotExecutor = StoreEngineHelper.createSnapshotExecutor(opts.getSnapshotCoreThreads(),
            opts.getSnapshotMaxThreads());
    }

    // 初始化线程池，根据请求类型分类
    final boolean useSharedRpcExecutor = opts.isUseSharedRpcExecutor();
    if (!useSharedRpcExecutor) {
        if (this.cliRpcExecutor == null) {
            this.cliRpcExecutor = StoreEngineHelper.createCliRpcExecutor(opts.getCliRpcCoreThreads());
        }
        if (this.raftRpcExecutor == null) {
            this.raftRpcExecutor = StoreEngineHelper.createRaftRpcExecutor(opts.getRaftRpcCoreThreads());
        }
        if (this.kvRpcExecutor == null) {
            this.kvRpcExecutor = StoreEngineHelper.createKvRpcExecutor(opts.getKvRpcCoreThreads());
        }
    }
    // init metrics
    startMetricReporters(metricsReportPeriod);
    // init rpc server
    this.rpcServer = RaftRpcServerFactory.createRaftRpcServer(serverAddress, this.raftRpcExecutor,
        this.cliRpcExecutor);
    //注册KV处理器
    StoreEngineHelper.addKvStoreRequestProcessor(this.rpcServer, this);
    if (!this.rpcServer.init(null)) {
        LOG.error("Fail to init [RpcServer].");
        return false;
    }
    // 初始化底层存储
    if (!initRawKVStore(opts)) {
        return false;
    }
    if (this.rawKVStore instanceof Describer) {
        DescriberManager.getInstance().addDescriber((Describer) this.rawKVStore);
    }
    // 初始化regionEngine
    if (!initAllRegionEngine(opts, store)) {
        LOG.error("Fail to init all [RegionEngine].");
        return false;
    }
    //发送心跳到PD
    if (this.pdClient instanceof RemotePlacementDriverClient) { //fake为false
        HeartbeatOptions heartbeatOpts = opts.getHeartbeatOptions();
        if (heartbeatOpts == null) {
            heartbeatOpts = new HeartbeatOptions();
        }
        this.heartbeatSender = new HeartbeatSender(this);
        if (!this.heartbeatSender.init(heartbeatOpts)) {
            LOG.error("Fail to init [HeartbeatSender].");
            return false;
        }
    }
    this.startTime = System.currentTimeMillis();
    LOG.info("[StoreEngine] start successfully: {}.", this);
    return this.started = true;
}
```

#### 注册KV处理器

com.alipay.sofa.jraft.rhea.StoreEngineHelper#addKvStoreRequestProcessor

```java
public static void addKvStoreRequestProcessor(final RpcServer rpcServer, final StoreEngine engine) {
    rpcServer.registerProcessor(new KVCommandProcessor<>(GetRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(MultiGetRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(ContainsKeyRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(GetSequenceRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(ResetSequenceRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(ScanRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(PutRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(GetAndPutRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(CompareAndPutRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(MergeRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(PutIfAbsentRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(KeyLockRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(KeyUnlockRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(BatchPutRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(DeleteRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(DeleteRangeRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(BatchDeleteRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(NodeExecuteRequest.class, engine));
    rpcServer.registerProcessor(new KVCommandProcessor<>(RangeSplitRequest.class, engine));
}
```

#### 创建RawKVStore

com.alipay.sofa.jraft.rhea.StoreEngine#initRawKVStore

```java
private boolean initRawKVStore(final StoreEngineOptions opts) { //创建RawKVStore
    final StorageType storageType = opts.getStorageType();
    switch (storageType) {
        case RocksDB:
            return initRocksDB(opts);
        case Memory:
            return initMemoryDB(opts);
        default:
            throw new UnsupportedOperationException("unsupported storage type: " + storageType);
    }
}
```

#### 初始化RegionEngine

com.alipay.sofa.jraft.rhea.StoreEngine#initAllRegionEngine

```java
private boolean initAllRegionEngine(final StoreEngineOptions opts, final Store store) {
    Requires.requireNonNull(opts, "opts");
    Requires.requireNonNull(store, "store");
    String baseRaftDataPath = opts.getRaftDataPath();
    if (Strings.isNotBlank(baseRaftDataPath)) {
        try {
            FileUtils.forceMkdir(new File(baseRaftDataPath));
        } catch (final Throwable t) {
            LOG.error("Fail to make dir for raftDataPath: {}.", baseRaftDataPath);
            return false;
        }
    } else {
        baseRaftDataPath = "";
    }
    final Endpoint serverAddress = opts.getServerAddress();
    //获取region的配置
    final List<RegionEngineOptions> rOptsList = opts.getRegionEngineOptionsList();
    final List<Region> regionList = store.getRegions();
    Requires.requireTrue(rOptsList.size() == regionList.size());
  	//创建RegionEngine
    for (int i = 0; i < rOptsList.size(); i++) {
        final RegionEngineOptions rOpts = rOptsList.get(i);
        final Region region = regionList.get(i);
       	//设置region的存储目录
        if (Strings.isBlank(rOpts.getRaftDataPath())) {
            final String childPath = "raft_data_region_" + region.getId() + "_" + serverAddress.getPort();
            rOpts.setRaftDataPath(Paths.get(baseRaftDataPath, childPath).toString());
        }
        Requires.requireNonNull(region.getRegionEpoch(), "regionEpoch");
        //创建RegionEngine
        final RegionEngine engine = new RegionEngine(region, this);
        //初始化
        if (engine.init(rOpts)) {
            //regionId ->DefaultRegionKVService
            final RegionKVService regionKVService = new DefaultRegionKVService(engine);
            registerRegionKVService(regionKVService);
            //regionId ->RegionEngine
            this.regionEngineTable.put(region.getId(), engine);
        } else {
            LOG.error("Fail to init [RegionEngine: {}].", region);
            return false;
        }
    }
    return true;
}
```

##### RegionEngine选举

com.alipay.sofa.jraft.rhea.RegionEngine#init

```java
public synchronized boolean init(final RegionEngineOptions opts) {//为region选举Leader
    if (this.started) {
        LOG.info("[RegionEngine: {}] already started.", this.region);
        return true;
    }
    this.regionOpts = Requires.requireNonNull(opts, "opts");
    this.fsm = new KVStoreStateMachine(this.region, this.storeEngine);

    // 获取或者创建NodeOptions，用于region的leader选举
    NodeOptions nodeOpts = opts.getNodeOptions();
    if (nodeOpts == null) {
        nodeOpts = new NodeOptions();
    }
    final long metricsReportPeriod = opts.getMetricsReportPeriod();
    if (metricsReportPeriod > 0) {
        // metricsReportPeriod > 0 means enable metrics
        nodeOpts.setEnableMetrics(true);
    }
    final Configuration initialConf = new Configuration();
    if (!initialConf.parse(opts.getInitialServerList())) {
        LOG.error("Fail to parse initial configuration {}.", opts.getInitialServerList());
        return false;
    }
    //region所在的服务器
    nodeOpts.setInitialConf(initialConf);
    //设置状态机
    nodeOpts.setFsm(this.fsm);
    final String raftDataPath = opts.getRaftDataPath();
    try {
        FileUtils.forceMkdir(new File(raftDataPath));
    } catch (final Throwable t) {
        LOG.error("Fail to make dir for raftDataPath {}.", raftDataPath);
        return false;
    }
    if (Strings.isBlank(nodeOpts.getLogUri())) {
        final Path logUri = Paths.get(raftDataPath, "log");
        nodeOpts.setLogUri(logUri.toString());
    }
    if (Strings.isBlank(nodeOpts.getRaftMetaUri())) {
        final Path meteUri = Paths.get(raftDataPath, "meta");
        nodeOpts.setRaftMetaUri(meteUri.toString());
    }
    if (Strings.isBlank(nodeOpts.getSnapshotUri())) {
        final Path snapshotUri = Paths.get(raftDataPath, "snapshot");
        nodeOpts.setSnapshotUri(snapshotUri.toString());
    }
    LOG.info("[RegionEngine: {}], log uri: {}, raft meta uri: {}, snapshot uri: {}.", this.region,
        nodeOpts.getLogUri(), nodeOpts.getRaftMetaUri(), nodeOpts.getSnapshotUri());

    final Endpoint serverAddress = opts.getServerAddress();
    final PeerId serverId = new PeerId(serverAddress, 0);
    final RpcServer rpcServer = this.storeEngine.getRpcServer();
    //创建并初始化node，为region选举leader
    this.raftGroupService = new RaftGroupService(opts.getRaftGroupId(), serverId, nodeOpts, rpcServer, true);
    this.node = this.raftGroupService.start(false);
    RouteTable.getInstance().updateConfiguration(this.raftGroupService.getGroupId(), nodeOpts.getInitialConf());
    if (this.node != null) {
      	//所有的region共用同一个RawKVStore
        final RawKVStore rawKVStore = this.storeEngine.getRawKVStore();
        final Executor readIndexExecutor = this.storeEngine.getReadIndexExecutor();
        //装饰RawKVStore
        this.raftRawKVStore = new RaftRawKVStore(this.node, rawKVStore, readIndexExecutor);
        this.metricsRawKVStore = new MetricsRawKVStore(this.region.getId(), this.raftRawKVStore);
        // metrics config
        if (this.regionMetricsReporter == null && metricsReportPeriod > 0) {
            final MetricRegistry metricRegistry = this.node.getNodeMetrics().getMetricRegistry();
            if (metricRegistry != null) {
                final ScheduledExecutorService scheduler = this.storeEngine.getMetricsScheduler();
                // start raft node metrics reporter
                this.regionMetricsReporter = Slf4jReporter.forRegistry(metricRegistry) //
                    .prefixedWith("region_" + this.region.getId()) //
                    .withLoggingLevel(Slf4jReporter.LoggingLevel.INFO) //
                    .outputTo(LOG) //
                    .scheduleOn(scheduler) //
                    .shutdownExecutorOnStop(scheduler != null) //
                    .build();
                this.regionMetricsReporter.start(metricsReportPeriod, TimeUnit.SECONDS);
            }
        }
        this.started = true;
        LOG.info("[RegionEngine] start successfully: {}.", this);
    }
    return this.started;
}
```

### HeartbeatSender初始化

com.alipay.sofa.jraft.rhea.client.pd.HeartbeatSender#init

```java
public synchronized boolean init(final HeartbeatOptions opts) {
    if (this.started) {
        LOG.info("[HeartbeatSender] already started.");
        return true;
    }
  	//统计信息收集器
    this.statsCollector = new StatsCollector(this.storeEngine);
    this.instructionProcessor = new InstructionProcessor(this.storeEngine);
  	//创建时间轮
    this.heartbeatTimer = new HashedWheelTimer(new NamedThreadFactory("heartbeat-timer", true), 50,
        TimeUnit.MILLISECONDS, 4096);
    this.heartbeatRpcTimeoutMillis = opts.getHeartbeatRpcTimeoutMillis();
    if (this.heartbeatRpcTimeoutMillis <= 0) {
        throw new IllegalArgumentException("Heartbeat rpc timeout millis must > 0, "
                                           + this.heartbeatRpcTimeoutMillis);
    }
  	//执行心跳回调的线程池
    final String name = "rheakv-heartbeat-callback";
    this.heartbeatRpcCallbackExecutor = ThreadPoolUtil.newBuilder() //
        .poolName(name) //
        .enableMetric(true) //
        .coreThreads(4) //
        .maximumThreads(4) //
        .keepAliveSeconds(120L) //
        .workQueue(new ArrayBlockingQueue<>(1024)) //
        .threadFactory(new NamedThreadFactory(name, true)) //
        .rejectedHandler(new DiscardOldPolicyWithReport(name)) //
        .build();
    final long storeHeartbeatIntervalSeconds = opts.getStoreHeartbeatIntervalSeconds();
    final long regionHeartbeatIntervalSeconds = opts.getRegionHeartbeatIntervalSeconds();
    if (storeHeartbeatIntervalSeconds <= 0) {
        throw new IllegalArgumentException("Store heartbeat interval seconds must > 0, "
                                           + storeHeartbeatIntervalSeconds);
    }
    if (regionHeartbeatIntervalSeconds <= 0) {
        throw new IllegalArgumentException("Region heartbeat interval seconds must > 0, "
                                           + regionHeartbeatIntervalSeconds);
    }
    final long now = System.currentTimeMillis();
  //创建上报store信息的任务
    final StoreHeartbeatTask storeHeartbeatTask = new StoreHeartbeatTask(storeHeartbeatIntervalSeconds, now, false);
  //创建上报region信息的任务
    final RegionHeartbeatTask regionHeartbeatTask = new RegionHeartbeatTask(regionHeartbeatIntervalSeconds, now,
        false);
    this.heartbeatTimer.newTimeout(storeHeartbeatTask, storeHeartbeatTask.getNextDelay(), TimeUnit.SECONDS);
    this.heartbeatTimer.newTimeout(regionHeartbeatTask, regionHeartbeatTask.getNextDelay(), TimeUnit.SECONDS);
    LOG.info("[HeartbeatSender] start successfully, options: {}.", opts);
    return this.started = true;
}
```

#### StoreHeartbeat

com.alipay.sofa.jraft.rhea.client.pd.HeartbeatSender#sendStoreHeartbeat

```java
private void sendStoreHeartbeat(final long nextDelay, final boolean forceRefreshLeader, final long lastTime) { //上报store信息
    final long now = System.currentTimeMillis();
    final StoreHeartbeatRequest request = new StoreHeartbeatRequest();
    request.setClusterId(this.storeEngine.getClusterId());
    final TimeInterval timeInterval = new TimeInterval(lastTime, now);
    //获取store信息
    final StoreStats stats = this.statsCollector.collectStoreStats(timeInterval);
    request.setStats(stats);
    final HeartbeatClosure<Object> closure = new HeartbeatClosure<Object>() {

        @Override
        public void run(final Status status) {
            final boolean forceRefresh = !status.isOk() && ErrorsHelper.isInvalidPeer(getError());
            final StoreHeartbeatTask nexTask = new StoreHeartbeatTask(nextDelay, now, forceRefresh);
            heartbeatTimer.newTimeout(nexTask, nexTask.getNextDelay(), TimeUnit.SECONDS);
        }
    };
    final Endpoint endpoint = this.pdClient.getPdLeader(forceRefreshLeader, this.heartbeatRpcTimeoutMillis);
    callAsyncWithRpc(endpoint, request, closure);
}
```

```java
public StoreStats collectStoreStats(final TimeInterval timeInterval) {
    final StoreStats stats = new StoreStats();
    stats.setStoreId(this.storeEngine.getStoreId());
    // Capacity for the store
    stats.setCapacity(this.storeEngine.getTotalSpace()); //store总共空间
    // Available size for the store
    stats.setAvailable(this.storeEngine.getUsableSpace()); //可用空间
    // Total region count in this store
    stats.setRegionCount(this.storeEngine.getRegionCount()); //region的数量
    // Leader region count in this store
    stats.setLeaderRegionCount(this.storeEngine.getLeaderRegionCount()); //leader是本节点的region数量
    // Current sending snapshot count
    // TODO
    // Current receiving snapshot count
    // TODO
    // How many region is applying snapshot
    // TODO
    // When the store is started (unix timestamp in milliseconds)
    stats.setStartTime(this.storeEngine.getStartTime()); //启动时间
    // If the store is busy
    stats.setBusy(this.storeEngine.isBusy()); //是否正在执行region切分
    // Actually used space by db
    stats.setUsedSize(this.storeEngine.getStoreUsedSpace()); //已用空间
    // Bytes written for the store during this period
    stats.setBytesWritten(getStoreBytesWritten(true)); //周期内写的字节数
    // Bytes read for the store during this period
    stats.setBytesRead(getStoreBytesRead(true)); //周期内读的字节数
    // Keys written for the store during this period
    stats.setKeysWritten(getStoreKeysWritten(true)); //周期内写的key的数量
    // Keys read for the store during this period
    stats.setKeysRead(getStoreKeysRead(true)); //周期内读的key的数量
    // Actually reported time interval
    stats.setInterval(timeInterval);
    LOG.info("Collect [StoreStats]: {}.", stats);
    return stats;
}
```

#### RegionHeartbeat

com.alipay.sofa.jraft.rhea.client.pd.HeartbeatSender#sendRegionHeartbeat

```java
private void sendRegionHeartbeat(final long nextDelay, final long lastTime, final boolean forceRefreshLeader) { //上报leader是本机节点的region信息
    final long now = System.currentTimeMillis();
    final RegionHeartbeatRequest request = new RegionHeartbeatRequest();
    request.setClusterId(this.storeEngine.getClusterId()); //所属集群
    request.setStoreId(this.storeEngine.getStoreId()); //所属的节点
  	//默认10_000，region内含有的key的数量超过此值，才会进行切分
    request.setLeastKeysOnSplit(this.storeEngine.getStoreOpts().getLeastKeysOnSplit());
    final List<Long> regionIdList = this.storeEngine.getLeaderRegionIds(); //获取leader本机的region
    if (regionIdList.isEmpty()) {
        // So sad, there is no even a region leader :(
        final RegionHeartbeatTask nextTask = new RegionHeartbeatTask(nextDelay, now, false);
        this.heartbeatTimer.newTimeout(nextTask, nextTask.getNextDelay(), TimeUnit.SECONDS);
        if (LOG.isInfoEnabled()) {
            LOG.info("So sad, there is no even a region leader on [clusterId:{}, storeId: {}, endpoint:{}].",
                this.storeEngine.getClusterId(), this.storeEngine.getStoreId(), this.storeEngine.getSelfEndpoint());
        }
        return;
    }
    final List<Pair<Region, RegionStats>> regionStatsList = Lists.newArrayListWithCapacity(regionIdList.size());
    final TimeInterval timeInterval = new TimeInterval(lastTime, now);
    for (final Long regionId : regionIdList) {
        final Region region = this.pdClient.getRegionById(regionId);
      //获取region的信息
        final RegionStats stats = this.statsCollector.collectRegionStats(region, timeInterval);
        if (stats == null) {
            continue;
        }
        regionStatsList.add(Pair.of(region, stats));
    }
    request.setRegionStatsList(regionStatsList);
    final HeartbeatClosure<List<Instruction>> closure = new HeartbeatClosure<List<Instruction>>() {

        @Override
        public void run(final Status status) {
            final boolean isOk = status.isOk();
            if (isOk) {
                final List<Instruction> instructions = getResult();
                if (instructions != null && !instructions.isEmpty()) {
                    instructionProcessor.process(instructions);
                }
            }
            final boolean forceRefresh = !isOk && ErrorsHelper.isInvalidPeer(getError());
            final RegionHeartbeatTask nextTask = new RegionHeartbeatTask(nextDelay, now, forceRefresh);
            heartbeatTimer.newTimeout(nextTask, nextTask.getNextDelay(), TimeUnit.SECONDS);
        }
    };
  	//上报region信息
    final Endpoint endpoint = this.pdClient.getPdLeader(forceRefreshLeader, this.heartbeatRpcTimeoutMillis);
    callAsyncWithRpc(endpoint, request, closure);
}
```

com.alipay.sofa.jraft.rhea.client.pd.StatsCollector#collectRegionStats

```java
public RegionStats collectRegionStats(final Region region, final TimeInterval timeInterval) {
    final RegionStats stats = new RegionStats();
    stats.setRegionId(region.getId()); //regionId
    // region的leader信息
    stats.setLeader(new Peer(region.getId(), this.storeEngine.getStoreId(), 		this.storeEngine.getSelfEndpoint()));
    // Leader considers that these peers are down
    // TODO
    // Pending peers are the peers that the leader can't consider as working followers
    // TODO
    // Bytes written for the region during this period
    stats.setBytesWritten(getRegionBytesWritten(region, true));//周期内写的字节数
    // Bytes read for the region during this period
    stats.setBytesRead(getRegionBytesRead(region, true)); //周期内读的字节数
    // Keys written for the region during this period
    stats.setKeysWritten(getRegionKeysWritten(region, true)); //周期内写的次数
    // Keys read for the region during this period
    stats.setKeysRead(getRegionKeysRead(region, true)); //周期读的次数
    // Approximate region size
    // TODO very important
    // Approximate number of keys 区间内key的数量
    stats.setApproximateKeys(this.rawKVStore.getApproximateKeysInRange(region.getStartKey(), region.getEndKey()));
    // Actually reported time interval
    stats.setInterval(timeInterval);
    LOG.info("Collect [RegionStats]: {}.", stats);
    return stats;
}
```

# 请求处理

com.alipay.sofa.jraft.rhea.KVCommandProcessor#handleRequest

```java
public void handleRequest(final RpcContext rpcCtx, final T request) {
    Requires.requireNonNull(request, "request");
    final RequestProcessClosure<BaseRequest, BaseResponse<?>> closure = new RequestProcessClosure<>(request, rpcCtx);
  	//StoreEngine维护了region和RegionKVService的对应关系
    final RegionKVService regionKVService = 		    this.storeEngine.getRegionKVService(request.getRegionId());
    if (regionKVService == null) { //未获取到对应的RegionKVService
        final NoRegionFoundResponse noRegion = new NoRegionFoundResponse();
        noRegion.setRegionId(request.getRegionId());
        noRegion.setError(Errors.NO_REGION_FOUND);
        noRegion.setValue(false);
        closure.sendResponse(noRegion);
        return;
    }
    switch (request.magic()) {
        case BaseRequest.PUT:
            regionKVService.handlePutRequest((PutRequest) request, closure);
            break;
        case BaseRequest.BATCH_PUT:
            regionKVService.handleBatchPutRequest((BatchPutRequest) request, closure);
            break;
        case BaseRequest.PUT_IF_ABSENT:
            regionKVService.handlePutIfAbsentRequest((PutIfAbsentRequest) request, closure);
            break;
        case BaseRequest.GET_PUT:
            regionKVService.handleGetAndPutRequest((GetAndPutRequest) request, closure);
            break;
        case BaseRequest.COMPARE_PUT:
            regionKVService.handleCompareAndPutRequest((CompareAndPutRequest) request, closure);
            break;
        case BaseRequest.DELETE:
            regionKVService.handleDeleteRequest((DeleteRequest) request, closure);
            break;
        case BaseRequest.DELETE_RANGE:
            regionKVService.handleDeleteRangeRequest((DeleteRangeRequest) request, closure);
            break;
        case BaseRequest.BATCH_DELETE:
            regionKVService.handleBatchDeleteRequest((BatchDeleteRequest) request, closure);
            break;
        case BaseRequest.MERGE:
            regionKVService.handleMergeRequest((MergeRequest) request, closure);
            break;
        case BaseRequest.GET:
            regionKVService.handleGetRequest((GetRequest) request, closure);
            break;
        case BaseRequest.MULTI_GET:
            regionKVService.handleMultiGetRequest((MultiGetRequest) request, closure);
            break;
        case BaseRequest.CONTAINS_KEY:
            regionKVService.handleContainsKeyRequest((ContainsKeyRequest) request, closure);
            break;
        case BaseRequest.SCAN:
            regionKVService.handleScanRequest((ScanRequest) request, closure);
            break;
        case BaseRequest.GET_SEQUENCE:
            regionKVService.handleGetSequence((GetSequenceRequest) request, closure);
            break;
        case BaseRequest.RESET_SEQUENCE:
            regionKVService.handleResetSequence((ResetSequenceRequest) request, closure);
            break;
        case BaseRequest.KEY_LOCK:
            regionKVService.handleKeyLockRequest((KeyLockRequest) request, closure);
            break;
        case BaseRequest.KEY_UNLOCK:
            regionKVService.handleKeyUnlockRequest((KeyUnlockRequest) request, closure);
            break;
        case BaseRequest.NODE_EXECUTE:
            regionKVService.handleNodeExecuteRequest((NodeExecuteRequest) request, closure);
            break;
        case BaseRequest.RANGE_SPLIT:
            regionKVService.handleRangeSplitRequest((RangeSplitRequest) request, closure);
            break;
        default:
            throw new RheaRuntimeException("Unsupported request type: " + request.getClass().getName());
    }
}
```

## GetRequest

com.alipay.sofa.jraft.rhea.DefaultRegionKVService#handleGetRequest

```java
public void handleGetRequest(final GetRequest request,
                             final RequestProcessClosure<BaseRequest, BaseResponse<?>> closure) {
    final GetResponse response = new GetResponse();
    response.setRegionId(getRegionId());
    response.setRegionEpoch(getRegionEpoch());
    try {
        KVParameterRequires.requireSameEpoch(request, getRegionEpoch());
        final byte[] key = KVParameterRequires.requireNonNull(request.getKey(), "get.key");
      	//此rawKVStore为MetricsRawKVStore
        this.rawKVStore.get(key, request.isReadOnlySafe(), new BaseKVStoreClosure() {

            @Override
            public void run(final Status status) {
                if (status.isOk()) {
                    response.setValue((byte[]) getData());
                } else {
                    setFailure(request, response, status, getError());
                }
                closure.sendResponse(response);
            }
        });
    } catch (final Throwable t) {
        LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(t));
        response.setError(Errors.forException(t));
        closure.sendResponse(response);
    }
}
```

com.alipay.sofa.jraft.rhea.storage.MetricsRawKVStore#get

```java
public void get(final byte[] key, final boolean readOnlySafe, final KVStoreClosure closure) {
    final KVStoreClosure c = metricsAdapter(closure, GET, 1, 0);
    this.rawKVStore.get(key, readOnlySafe, c);
}
```

com.alipay.sofa.jraft.rhea.storage.RaftRawKVStore#get

```java
public void get(final byte[] key, final boolean readOnlySafe, final KVStoreClosure closure) {
    if (!readOnlySafe) { //默认为true，线性读取
        this.kvStore.get(key, false, closure); //直接从rocksdb读取
        return;
    }
   //线性读取
    this.node.readIndex(BytesUtil.EMPTY_BYTES, new ReadIndexClosure() {

        @Override
        public void run(final Status status, final long index, final byte[] reqCtx) {
            if (status.isOk()) {
                RaftRawKVStore.this.kvStore.get(key, true, closure);
                return;
            }
            RaftRawKVStore.this.readIndexExecutor.execute(() -> {
                if (isLeader()) {
                    LOG.warn("Fail to [get] with 'ReadIndex': {}, try to applying to the state machine.", status);
                    // If 'read index' read fails, try to applying to the state machine at the leader node
                    applyOperation(KVOperation.createGet(key), closure);
                } else {
                    LOG.warn("Fail to [get] with 'ReadIndex': {}.", status);
                    // Client will retry to leader node
                    new KVClosureAdapter(closure, null).run(status);
                }
            });
        }
    });
}
```

## PutRequest

com.alipay.sofa.jraft.rhea.DefaultRegionKVService#handlePutRequest

```java
public void handlePutRequest(final PutRequest request,
                             final RequestProcessClosure<BaseRequest, BaseResponse<?>> closure) {
    final PutResponse response = new PutResponse();
    response.setRegionId(getRegionId());
    response.setRegionEpoch(getRegionEpoch());
    try {
        KVParameterRequires.requireSameEpoch(request, getRegionEpoch());
        final byte[] key = KVParameterRequires.requireNonNull(request.getKey(), "put.key");
        final byte[] value = KVParameterRequires.requireNonNull(request.getValue(), "put.value");
        this.rawKVStore.put(key, value, new BaseKVStoreClosure() {

            @Override
            public void run(final Status status) {
                if (status.isOk()) {
                    response.setValue((Boolean) getData());
                } else {
                    setFailure(request, response, status, getError());
                }
                closure.sendResponse(response);
            }
        });
    } catch (final Throwable t) {
        LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(t));
        response.setError(Errors.forException(t));
        closure.sendResponse(response);
    }
}
```

com.alipay.sofa.jraft.rhea.storage.MetricsRawKVStore#put

```java
public void put(final byte[] key, final byte[] value, final KVStoreClosure closure) {
    final KVStoreClosure c = metricsAdapter(closure, PUT, 1, value.length);
    this.rawKVStore.put(key, value, c); //RaftRawKVStore
}
```

com.alipay.sofa.jraft.rhea.storage.RaftRawKVStore#put

```java
public void put(final byte[] key, final byte[] value, final KVStoreClosure closure) {
    applyOperation(KVOperation.createPut(key, value), closure);
}
```

com.alipay.sofa.jraft.rhea.storage.RaftRawKVStore#applyOperation

```java
private void applyOperation(final KVOperation op, final KVStoreClosure closure) {
    if (!isLeader()) {//先判断是否是Leader
        closure.setError(Errors.NOT_LEADER);
        closure.run(new Status(RaftError.EPERM, "Not leader"));
        return;
    }
  	//发布事件
    final Task task = new Task();
    task.setData(ByteBuffer.wrap(Serializers.getDefault().writeObject(op)));
    task.setDone(new KVClosureAdapter(closure, op));
    this.node.apply(task);
}
```

## RangeSplit

com.alipay.sofa.jraft.rhea.DefaultRegionKVService#handleRangeSplitRequest

```java
public void handleRangeSplitRequest(final RangeSplitRequest request,
                                    final RequestProcessClosure<BaseRequest, BaseResponse<?>> closure) { //切分region
    final RangeSplitResponse response = new RangeSplitResponse();
    response.setRegionId(getRegionId());
    response.setRegionEpoch(getRegionEpoch());
    try {
        //切分之后新region的Id
        final Long newRegionId = KVParameterRequires.requireNonNull(request.getNewRegionId(),
            "rangeSplit.newRegionId");
        this.regionEngine.getStoreEngine().applySplit(request.getRegionId(), newRegionId, new BaseKVStoreClosure() {
            @Override
            public void run(final Status status) {
                if (status.isOk()) {
                    response.setValue((Boolean) getData());
                } else {
                    setFailure(request, response, status, getError());
                }
                closure.sendResponse(response);
            }
        });
    } catch (final Throwable t) {
        LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(t));
        response.setError(Errors.forException(t));
        closure.sendResponse(response);
    }
}
```

com.alipay.sofa.jraft.rhea.StoreEngine#applySplit

```java
public void applySplit(final Long regionId, final Long newRegionId, final KVStoreClosure closure) { //进行切分
    Requires.requireNonNull(regionId, "regionId");
    Requires.requireNonNull(newRegionId, "newRegionId");
    if (this.regionEngineTable.containsKey(newRegionId)) { //newRegionId对应的region已经存在
        closure.setError(Errors.CONFLICT_REGION_ID);
        closure.run(new Status(-1, "Conflict region id %d", newRegionId));
        return;
    }
    if (!this.splitting.compareAndSet(false, true)) { //有其他切分请求在执行
        closure.setError(Errors.SERVER_BUSY);
        closure.run(new Status(-1, "Server is busy now"));
        return;
    }
    final RegionEngine parentEngine = getRegionEngine(regionId);
    if (parentEngine == null) { //切分的region不存在
        closure.setError(Errors.NO_REGION_FOUND);
        closure.run(new Status(-1, "RegionEngine[%s] not found", regionId));
        this.splitting.set(false);
        return;
    }
    if (!parentEngine.isLeader()) { //被切分的region不是Leader
        closure.setError(Errors.NOT_LEADER);
        closure.run(new Status(-1, "RegionEngine[%s] not leader", regionId));
        this.splitting.set(false);
        return;
    }
    final Region parentRegion = parentEngine.getRegion();
    final byte[] startKey = BytesUtil.nullToEmpty(parentRegion.getStartKey()); //开始key
    final byte[] endKey = parentRegion.getEndKey(); //结束key
  	//粗略估计区间内的key的数量
    final long approximateKeys = this.rawKVStore.getApproximateKeysInRange(startKey, endKey);
    final long leastKeysOnSplit = this.storeOpts.getLeastKeysOnSplit();//默认10_000
    if (approximateKeys < leastKeysOnSplit) { //少于10_000不进行切分
        closure.setError(Errors.TOO_SMALL_TO_SPLIT);
        closure.run(new Status(-1, "RegionEngine[%s]'s keys less than %d", regionId, leastKeysOnSplit));
        this.splitting.set(false);
        return;
    }
  	//找到region的中间key
    final byte[] splitKey = this.rawKVStore.jumpOver(startKey, approximateKeys >> 1);
    if (splitKey == null) {
        closure.setError(Errors.STORAGE_ERROR);
        closure.run(new Status(-1, "Fail to scan split key"));
        this.splitting.set(false);
        return;
    }
  	//创建切分Operation，并将其同步至Follower，收到超过半数节点的响应后执行状态机切分
    final KVOperation op = KVOperation.createRangeSplit(splitKey, regionId, newRegionId);
    final Task task = new Task();
    task.setData(ByteBuffer.wrap(Serializers.getDefault().writeObject(op)));
    task.setDone(new KVClosureAdapter(closure, op));
    parentEngine.getNode().apply(task);
}
```

com.alipay.sofa.jraft.rhea.storage.KVStoreStateMachine#doSplit

```java
private void doSplit(final KVStateOutputList kvStates) { //状态机执行切分操作
    final byte[] parentKey = this.region.getStartKey();
    for (final KVState kvState : kvStates) {
        final KVOperation op = kvState.getOp();
        final long currentRegionId = op.getCurrentRegionId();
        final long newRegionId = op.getNewRegionId();
        final byte[] splitKey = op.getKey();
        final KVStoreClosure closure = kvState.getDone();
        try {
            this.rawKVStore.initFencingToken(parentKey, splitKey);
            this.storeEngine.doSplit(currentRegionId, newRegionId, splitKey);
            if (closure != null) {
                // null on follower
                closure.setData(Boolean.TRUE);
                closure.run(Status.OK());
            }
        } catch (final Throwable t) {
            LOG.error("Fail to split, regionId={}, newRegionId={}, splitKey={}.", currentRegionId, newRegionId,
                BytesUtil.toHex(splitKey));
            setCriticalError(closure, t);
        }
    }
}
```

com.alipay.sofa.jraft.rhea.storage.RocksRawKVStore#initFencingToken

```java
public void initFencingToken(final byte[] parentKey, final byte[] childKey) {
    final Timer.Context timeCtx = getTimeContext("INIT_FENCING_TOKEN");
    final Lock readLock = this.readWriteLock.readLock();
    readLock.lock();
    try {
        final byte[] realKey = BytesUtil.nullToEmpty(parentKey);
        final byte[] parentBytesVal = this.db.get(this.fencingHandle, realKey);
        if (parentBytesVal == null) {
            return;
        }
        this.db.put(this.fencingHandle, this.writeOptions, childKey, parentBytesVal);
    } catch (final RocksDBException e) {
        throw new StorageException("Fail to init fencing token.", e);
    } finally {
        readLock.unlock();
        timeCtx.stop();
    }
}
```

com.alipay.sofa.jraft.rhea.StoreEngine#doSplit

```java
public void doSplit(final Long regionId, final Long newRegionId, final byte[] splitKey) {
    try {
        Requires.requireNonNull(regionId, "regionId");
        Requires.requireNonNull(newRegionId, "newRegionId");
        final RegionEngine parent = getRegionEngine(regionId);
      	//创建新的region
        final Region region = parent.getRegion().copy();
      	//region的配置信息
        final RegionEngineOptions rOpts = parent.copyRegionOpts();
        region.setId(newRegionId);
        region.setStartKey(splitKey);
        region.setRegionEpoch(new RegionEpoch(-1, -1));

        rOpts.setRegionId(newRegionId);
        rOpts.setStartKeyBytes(region.getStartKey());
        rOpts.setEndKeyBytes(region.getEndKey());
        rOpts.setRaftGroupId(JRaftHelper.getJRaftGroupId(this.pdClient.getClusterName(), newRegionId));
        rOpts.setRaftDataPath(null);

        String baseRaftDataPath = this.storeOpts.getRaftDataPath();
        if (Strings.isBlank(baseRaftDataPath)) {
            baseRaftDataPath = "";
        }
        rOpts.setRaftDataPath(baseRaftDataPath + "raft_data_region_" + region.getId() + "_"
                              + getSelfEndpoint().getPort());
        //创建RegionEngine
        final RegionEngine engine = new RegionEngine(region, this);
        if (!engine.init(rOpts)) { //为新的region选举leader
            LOG.error("Fail to init [RegionEngine: {}].", region);
            throw Errors.REGION_ENGINE_FAIL.exception();
        }

        // update parent conf
        final Region pRegion = parent.getRegion();
        final RegionEpoch pEpoch = pRegion.getRegionEpoch();
        final long version = pEpoch.getVersion();
        pEpoch.setVersion(version + 1); // 修改版本
        pRegion.setEndKey(splitKey); // 修改被切分region的结束key

        // the following two lines of code can make a relation of 'happens-before' for
        // read 'pRegion', because that a write to a ConcurrentMap happens-before every
        // subsequent read of that ConcurrentMap.
      
      	//regionId -> RegionEngine
        this.regionEngineTable.put(region.getId(), engine);
      	//regionId -> DefaultRegionKVService
        registerRegionKVService(new DefaultRegionKVService(engine));

        // 更新本地region路由表
        this.pdClient.getRegionRouteTable().splitRegion(pRegion.getId(), region);
    } finally {
        this.splitting.set(false); //切分完成
    }
}
```

# 总结

1、请求隔离，不同类型的请求，使用不同的线程池

2、异步、回调，请求流程无阻塞
