# Jraft-rheaPD源码分析

* [PD启动](#pd启动)
  * [初始化](#初始化)
    * [Init_PlacementDriverService](#init_placementdriverservice)
    * [Init_DefaultPipeline](#init_defaultpipeline)
      * [添加Handler](#添加handler)
    * [Init_RheaKVStore](#init_rheakvstore)
      * [RegionEngine创建](#regionengine创建)
      * [RegionEngine选举](#regionengine选举)
    * [注册PDProcessor](#注册pdprocessor)
* [请求处理](#请求处理)
  * [处理store心跳请求](#处理store心跳请求)
    * [验证store统计信息](#验证store统计信息)
    * [存储store统计信息](#存储store统计信息)
  * [处理Region心跳请求](#处理region心跳请求)
    * [验证region统计信息](#验证region统计信息)
    * [存储region统计信息](#存储region统计信息)
    * [检测是否转让Leader](#检测是否转让leader)
      * [查找转让节点](#查找转让节点)
    * [检测是否切分region](#检测是否切分region)
  * [处理获取集群信息请求](#处理获取集群信息请求)
* [Pipeling执行流程](#pipeling执行流程)
  * [创建Future](#创建future)
  * [执行自定义业务逻辑](#执行自定义业务逻辑)


# PD启动

com.alipay.sofa.jraft.rhea.PlacementDriverStartup#main

```java
public static void main(String[] args) throws Exception {
    if (args.length != 1) {
        LOG.error("Usage: com.alipay.sofa.jraft.rhea.PlacementDriverStartup <ConfigFilePath>");
        System.exit(1);
    }
    final String configPath = args[0];
    final ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
    final PlacementDriverServerOptions opts = mapper.readValue(new File(configPath),
        PlacementDriverServerOptions.class);
    final PlacementDriverServer pdServer = new PlacementDriverServer();
    if (!pdServer.init(opts)) { //初始化
        throw new PlacementDriverServerStartupException("Fail to start [PlacementDriverServer].");
    }
    LOG.info("Starting PlacementDriverServer with config: {}.", opts);
}
```

## 初始化

com.alipay.sofa.jraft.rhea.PlacementDriverServer#init

```java
public synchronized boolean init(final PlacementDriverServerOptions opts) {
    if (this.started) {
        LOG.info("[PlacementDriverServer] already started.");
        return true;
    }
    Requires.requireNonNull(opts, "opts");
    final RheaKVStoreOptions rheaOpts = opts.getRheaKVStoreOptions();
    Requires.requireNonNull(rheaOpts, "opts.rheaKVStoreOptions");

    //创建并初始化DefaultRheaKVStore
    this.rheaKVStore = new DefaultRheaKVStore();
    if (!this.rheaKVStore.init(rheaOpts)) {
        LOG.error("Fail to init [RheaKVStore].");
        return false;
    }

    //创建并初始化PDservice
    this.placementDriverService = new DefaultPlacementDriverService(this.rheaKVStore);
    if (!this.placementDriverService.init(opts)) {
        LOG.error("Fail to init [PlacementDriverService].");
        return false;
    }
    //获取存储引擎
    final StoreEngine storeEngine = ((DefaultRheaKVStore) this.rheaKVStore).getStoreEngine();
    Requires.requireNonNull(storeEngine, "storeEngine");
    //获取所有regionEngine
    final List<RegionEngine> regionEngines = storeEngine.getAllRegionEngines();
    if (regionEngines.isEmpty()) {
        throw new IllegalArgumentException("Non region for [PlacementDriverServer]");
    }
    //PD只有一个region
    if (regionEngines.size() > 1) {
        throw new IllegalArgumentException("Only support single region for [PlacementDriverServer]");
    }
    //获取regionEngine
    this.regionEngine = regionEngines.get(0);
    this.rheaKVStore.addLeaderStateListener(this.regionEngine.getRegion().getId(),
        ((DefaultPlacementDriverService) this.placementDriverService));

    //注册请求处理器
    addPlacementDriverProcessor(storeEngine.getRpcServer());
    LOG.info("[PlacementDriverServer] start successfully, options: {}.", opts);
    return this.started = true;
}
```

### Init_PlacementDriverService

com.alipay.sofa.jraft.rhea.DefaultPlacementDriverService#init

```java
public synchronized boolean init(final PlacementDriverServerOptions opts) {
    if (this.started) {
        LOG.info("[DefaultPlacementDriverService] already started.");
        return true;
    }
    Requires.requireNonNull(opts, "placementDriverServerOptions");
    this.metadataStore = new DefaultMetadataStore(this.rheaKVStore);
    final ThreadPoolExecutor threadPool = createPipelineExecutor(opts);
    if (threadPool != null) {
        this.pipelineInvoker = new DefaultHandlerInvoker(threadPool);
    }
    //创建DefaultPipeline，与Netty的Pipeline类似，处理StoreHeart、RegionHeart请求
    this.pipeline = new DefaultPipeline();
    initPipeline(this.pipeline);
    LOG.info("[DefaultPlacementDriverService] start successfully, options: {}.", opts);
    return this.started = true;
}
```

com.alipay.sofa.jraft.rhea.DefaultPlacementDriverService#initPipeline

```java
protected void initPipeline(final Pipeline pipeline) {
    final List<Handler> sortedHandlers = JRaftServiceLoader.load(Handler.class) //加载Handler
        .sort(); //排序

    // default handlers and order:
    //
    // 1. logHandler
    // 2. storeStatsValidator
    // 3. regionStatsValidator
    // 4. storeStatsPersistence
    // 5. regionStatsPersistence
    // 6. regionLeaderBalance
    // 7. splittingJudgeByApproximateKeys
    // 8: placementDriverTail
    for (final Handler h : sortedHandlers) {
        pipeline.addLast(h); //加入Pipeline，维护handler
    }

    // first handler
    pipeline.addFirst(this.pipelineInvoker, "logHandler", new LogHandler()); //头
    // last handler
    pipeline.addLast("placementDriverTail", new PlacementDriverTailHandler()); //尾
}
```

### Init_DefaultPipeline

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline#DefaultPipeline

```java
public DefaultPipeline() { //双向链表
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

#### 添加Handler

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline#addLast(com.alipay.sofa.jraft.rhea.util.pipeline.Handler...)

```java
public Pipeline addLast(Handler... handlers) { //添加handler
    return addLast(null, handlers);
}
```

```java
public Pipeline addLast(HandlerInvoker invoker, Handler... handlers) {
    if (handlers == null) {
        throw new NullPointerException("handlers");
    }

    for (Handler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(invoker, null, h); //添加
    }

    return this;
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline#addLast(com.alipay.sofa.jraft.rhea.util.pipeline.HandlerInvoker, java.lang.String, com.alipay.sofa.jraft.rhea.util.pipeline.Handler)

```java
public Pipeline addLast(HandlerInvoker invoker, String name, Handler handler) {
    name = filterName(name, handler); //生成Handler名称
    //封装成DefaultHandlerContext
    addLast0(new DefaultHandlerContext(this, invoker, name, handler)); 
    return this;
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline#addLast0

```java
private void addLast0(AbstractHandlerContext newCtx) {
    checkMultiplicity(newCtx);
	  //加入双向链表，并没有考虑线程安全问题，在PD中只有在初始化时添加handler
    AbstractHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
		//执行AbstractHandlerContext的handlerAdded方法
    callHandlerAdded(newCtx);
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline#callHandlerAdded

```java
private void callHandlerAdded(final AbstractHandlerContext ctx) {
    try {
        ctx.handler().handlerAdded(ctx); //执行handler的handlerAdded方法
    } catch (Throwable t) {
        boolean removed = false;
        try {
            remove(ctx);
            removed = true;
        } catch (Throwable t2) {
            if (LOG.isWarnEnabled()) {
                LOG.warn("Failed to remove a handler: {}, {}.", ctx.name(), StackTraceUtil.stackTrace(t2));
            }
        }

    fireExceptionCaught(null, new PipelineException(
            ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; " + (removed ? "removed." :  "also failed to remove."), t));
    }
}
```

### Init_RheaKVStore

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

#### RegionEngine创建

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
    //获取store，每个节点有一个store，存储本地所有的region
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

#### RegionEngine选举

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
        final RawKVStore rawKVStore = this.storeEngine.getRawKVStore();
        final Executor readIndexExecutor = this.storeEngine.getReadIndexExecutor();
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

### 注册PDProcessor

com.alipay.sofa.jraft.rhea.PlacementDriverServer#addPlacementDriverProcessor

```java
private void addPlacementDriverProcessor(final RpcServer rpcServer) {
  	//注册PD请求处理器
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(RegionHeartbeatRequest.class,
        this.placementDriverService, this.pdExecutor));
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(StoreHeartbeatRequest.class,
        this.placementDriverService, this.pdExecutor));
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(GetClusterInfoRequest.class,
        this.placementDriverService, this.pdExecutor));
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(GetStoreIdRequest.class,
        this.placementDriverService, this.pdExecutor));
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(GetStoreInfoRequest.class,
        this.placementDriverService, this.pdExecutor));
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(SetStoreInfoRequest.class,
        this.placementDriverService, this.pdExecutor));
    rpcServer.registerProcessor(new PlacementDriverProcessor<>(CreateRegionIdRequest.class,
        this.placementDriverService, this.pdExecutor));
}
```

# 请求处理

## 处理store心跳请求

com.alipay.sofa.jraft.rhea.DefaultPlacementDriverService#handleStoreHeartbeatRequest

```java
public void handleStoreHeartbeatRequest(final StoreHeartbeatRequest request,
                                        final RequestProcessClosure<BaseRequest, BaseResponse> closure) { //处理存储节点的统计信息，每个存储节点对应一个store
    final StoreHeartbeatResponse response = new StoreHeartbeatResponse();
    response.setClusterId(request.getClusterId());
    if (!this.isLeader) { 
        response.setError(Errors.NOT_LEADER);
        closure.sendResponse(response);
        return;
    }
    //必须是Leader节点
    try {
        //封装成StorePingEvent交由DefaultPipeline处理
        final StorePingEvent storePingEvent = new StorePingEvent(request, this.metadataStore);
      	
        final PipelineFuture<Object> future = this.pipeline.invoke(storePingEvent);
        future.whenComplete((ignored, throwable) -> {
            if (throwable != null) {
                LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(throwable));
                response.setError(Errors.forException(throwable));
            }
            closure.sendResponse(response);
        });
    } catch (final Throwable t) {
        LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(t));
        response.setError(Errors.forException(t));
        closure.sendResponse(response);
    }
}
```

### 验证store统计信息

com.alipay.sofa.jraft.rhea.pipeline.handler.StoreStatsValidator#readMessage

```java
public void readMessage(final HandlerContext ctx, final StorePingEvent event) throws Exception {//验证store信息
    final MetadataStore metadataStore = event.getMetadataStore();
    final StoreHeartbeatRequest request = event.getMessage();
    final StoreStats storeStats = request.getStats();
    if (storeStats == null) {
        LOG.error("Empty [StoreStats] by event: {}.", event);
        throw Errors.INVALID_STORE_STATS.exception();
    }
    final StoreStats currentStoreStats = metadataStore.getStoreStats(request.getClusterId(),
        storeStats.getStoreId());
    if (currentStoreStats == null) {
        return; // new data
    }
    final TimeInterval interval = storeStats.getInterval();
    if (interval == null) {
        LOG.error("Empty [TimeInterval] by event: {}.", event);
        throw Errors.INVALID_STORE_STATS.exception();
    }
    final TimeInterval currentInterval = currentStoreStats.getInterval();
    if (interval.getEndTimestamp() < currentInterval.getEndTimestamp()) {
        LOG.error("The [TimeInterval] is out of date: {}.", event);
        throw Errors.STORE_HEARTBEAT_OUT_OF_DATE.exception();
    }
}
```

### 存储store统计信息

com.alipay.sofa.jraft.rhea.pipeline.handler.StoreStatsPersistenceHandler#readMessage

```java
public void readMessage(final HandlerContext ctx, final StorePingEvent event) throws Exception { //存储store信息
    final MetadataStore metadataStore = event.getMetadataStore();
    final StoreHeartbeatRequest request = event.getMessage();
    metadataStore.updateStoreStats(request.getClusterId(), request.getStats()).get(); // sync
}
```

## 处理Region心跳请求

com.alipay.sofa.jraft.rhea.DefaultPlacementDriverService#handleRegionHeartbeatRequest

```java
public void handleRegionHeartbeatRequest(final RegionHeartbeatRequest request,
                                         final RequestProcessClosure<BaseRequest, BaseResponse> closure) { //处理region的统计信息
    final RegionHeartbeatResponse response = new RegionHeartbeatResponse();
    response.setClusterId(request.getClusterId());
    if (!this.isLeader) {
        response.setError(Errors.NOT_LEADER);
        closure.sendResponse(response);
        return;
    }
  	//必须是Leader节点
    try {
      	 //封装成RegionPingEvent交由DefaultPipeline处理
        // 1. 保存数据
        // 2. 检查是否需要发送调度指令
        final RegionPingEvent regionPingEvent = new RegionPingEvent(request, this.metadataStore);
        final PipelineFuture<List<Instruction>> future = this.pipeline.invoke(regionPingEvent);
        future.whenComplete((instructions, throwable) -> {
            if (throwable == null) {
                response.setValue(instructions);
            } else {
                LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(throwable));
                response.setError(Errors.forException(throwable));
            }
            closure.sendResponse(response);
        });
    } catch (final Throwable t) {
        LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(t));
        response.setError(Errors.forException(t));
        closure.sendResponse(response);
    }
}
```

### 验证region统计信息

com.alipay.sofa.jraft.rhea.pipeline.handler.RegionStatsValidator#readMessage

```java
public void readMessage(final HandlerContext ctx, final RegionPingEvent event) throws Exception { //验证region的合法性
    final MetadataStore metadataStore = event.getMetadataStore();
    final RegionHeartbeatRequest request = event.getMessage();
    final List<Pair<Region, RegionStats>> regionStatsList = request.getRegionStatsList();
    if (regionStatsList == null || regionStatsList.isEmpty()) {
        LOG.error("Empty [RegionStatsList] by event: {}.", event);
        throw Errors.INVALID_REGION_STATS.exception();
    }
    for (final Pair<Region, RegionStats> pair : regionStatsList) {
        final Region region = pair.getKey();
        if (region == null) {
            LOG.error("Empty [Region] by event: {}.", event);
            throw Errors.INVALID_REGION_STATS.exception();
        }
        final RegionEpoch regionEpoch = region.getRegionEpoch();
        if (regionEpoch == null) {
            LOG.error("Empty [RegionEpoch] by event: {}.", event);
            throw Errors.INVALID_REGION_STATS.exception();
        }
        final RegionStats regionStats = pair.getValue();
        if (regionStats == null) {
            LOG.error("Empty [RegionStats] by event: {}.", event);
            throw Errors.INVALID_REGION_STATS.exception();
        }
        final Pair<Region, RegionStats> currentRegionInfo = metadataStore.getRegionStats(request.getClusterId(),
            region);
        if (currentRegionInfo == null) {
            return; // new data
        }
        final Region currentRegion = currentRegionInfo.getKey();
        if (regionEpoch.compareTo(currentRegion.getRegionEpoch()) < 0) {
            LOG.error("The region epoch is out of date: {}.", event);
            throw Errors.REGION_HEARTBEAT_OUT_OF_DATE.exception();
        }
        final TimeInterval interval = regionStats.getInterval();
        if (interval == null) {
            LOG.error("Empty [TimeInterval] by event: {}.", event);
            throw Errors.INVALID_REGION_STATS.exception();
        }
        final TimeInterval currentInterval = currentRegionInfo.getValue().getInterval();
        if (interval.getEndTimestamp() < currentInterval.getEndTimestamp()) {
            LOG.error("The [TimeInterval] is out of date: {}.", event);
            throw Errors.REGION_HEARTBEAT_OUT_OF_DATE.exception();
        }
    }
}
```

### 存储region统计信息

com.alipay.sofa.jraft.rhea.pipeline.handler.RegionStatsPersistenceHandler#readMessage

```java
public void readMessage(final HandlerContext ctx, final RegionPingEvent event) throws Exception { //存储region信息
    final MetadataStore metadataStore = event.getMetadataStore();
    final RegionHeartbeatRequest request = event.getMessage();
    metadataStore.batchUpdateRegionStats(request.getClusterId(), request.getRegionStatsList()).get(); // sync
}
```

### 检测是否转让Leader

com.alipay.sofa.jraft.rhea.pipeline.handler.RegionLeaderBalanceHandler#readMessage

```java
public void readMessage(final HandlerContext ctx, final RegionPingEvent event) throws Exception {//检测是否触发region的Leader的转让，避免同一节点上的Leader过多
    if (event.isReady()) {
        return;
    }
    final MetadataStore metadataStore = event.getMetadataStore();
    final RegionHeartbeatRequest request = event.getMessage();
    final long clusterId = request.getClusterId();
    final long storeId = request.getStoreId();
    final ClusterStatsManager clusterStatsManager = ClusterStatsManager.getInstance(clusterId);
    //上报leader是本节点的region对应的统计信息
    final List<Pair<Region, RegionStats>> regionStatsList = request.getRegionStatsList();
    for (final Pair<Region, RegionStats> stats : regionStatsList) {
        final Region region = stats.getKey();
        clusterStatsManager.addOrUpdateLeader(storeId, region.getId()); //维护节点上的 region leader
    }

    //获取leader个数最多的节点
    final Pair<Set<Long>, Integer> modelWorkers = clusterStatsManager.findModelWorkerStores(1);
    final Set<Long> modelWorkerStoreIds = modelWorkers.getKey();
    final int modelWorkerLeaders = modelWorkers.getValue();
    if (!modelWorkerStoreIds.contains(storeId)) {
        return;
    }

    LOG.info("[Cluster: {}] model worker stores is: {}, it has {} leaders.", clusterId, modelWorkerStoreIds, modelWorkerLeaders);
		//遍历当前上报的region统计信息。每次遍历只会转让一个region的Leader，不会转让多个
    for (final Pair<Region, RegionStats> pair : regionStatsList) { 
        final Region region = pair.getKey();
        final List<Peer> peers = region.getPeers();
        if (peers == null) {
            continue;
        }
        final List<Endpoint> endpoints = Lists.transform(peers, Peer::getEndpoint);
        //storeId -> EndPoint
        final Map<Long, Endpoint> storeIds = metadataStore.unsafeGetStoreIdsByEndpoints(clusterId, endpoints);
        // 获取leader数量最少的store
        final List<Pair<Long, Integer>> lazyWorkers = clusterStatsManager.findLazyWorkerStores(storeIds.keySet());
        if (lazyWorkers.isEmpty()) {
            return;
        }
        for (int i = lazyWorkers.size() - 1; i >= 0; i--) {
            final Pair<Long, Integer> worker = lazyWorkers.get(i);
            if (modelWorkerLeaders - worker.getValue() <= 1) { // no need to transfer 各个存储节点上的leader数量差距不大，不需要转让
                lazyWorkers.remove(i); //移除
            }
        }
        if (lazyWorkers.isEmpty()) {
            continue;
        }
        //查找转让leader到哪个存储节点
        final Pair<Long, Integer> laziestWorker = tryToFindLaziestWorker(clusterId, metadataStore, lazyWorkers);
        if (laziestWorker == null) {
            continue;
        }
        //转让到此节点
        final Long lazyWorkerStoreId = laziestWorker.getKey();
        LOG.info("[Cluster: {}], lazy worker store is: {}, it has {} leaders.", clusterId, lazyWorkerStoreId,
                laziestWorker.getValue());
        //leader转让指令
        final Instruction.TransferLeader transferLeader = new Instruction.TransferLeader();
        transferLeader.setMoveToStoreId(lazyWorkerStoreId);
        transferLeader.setMoveToEndpoint(storeIds.get(lazyWorkerStoreId));
        final Instruction instruction = new Instruction();
        instruction.setRegion(region.copy());
        instruction.setTransferLeader(transferLeader);
        event.addInstruction(instruction);
        LOG.info("[Cluster: {}], send 'instruction.transferLeader': {} to region: {}.", clusterId, instruction, region);
        break; // Only do one thing at a time
    }
}
```

#### 查找转让节点

com.alipay.sofa.jraft.rhea.pipeline.handler.RegionLeaderBalanceHandler#tryToFindLaziestWorker

```java
private Pair<Long, Integer> tryToFindLaziestWorker(final long clusterId, final MetadataStore metadataStore, final List<Pair<Long, Integer>> lazyWorkers) {//根据store统计信息查找转让节点
  	 //获取store统计信息
    final List<Pair<Pair<Long, Integer>, StoreStats>> storeStatsList = Lists.newArrayList();
    for (final Pair<Long, Integer> worker : lazyWorkers) {
        final StoreStats stats = metadataStore.getStoreStats(clusterId, worker.getKey());
        if (stats != null) {
            // TODO check timeInterval
            storeStatsList.add(Pair.of(worker, stats));
        }
    }
    if (storeStatsList.isEmpty()) {
        return null;
    }
    if (storeStatsList.size() == 1) {
        return storeStatsList.get(0).getKey();
    }

    final Pair<Pair<Long, Integer>, StoreStats> min = Collections.min(storeStatsList, (o1, o2) -> {
        final StoreStats s1 = o1.getValue();
        final StoreStats s2 = o2.getValue();
        int val = Boolean.compare(s1.isBusy(), s2.isBusy()); //是否正在切分region
        if (val != 0) {
            return val;
        }
        val = Integer.compare(s1.getRegionCount(), s2.getRegionCount()); //region的数量
        if (val != 0) {
            return val;
        }
        val = Long.compare(s1.getBytesWritten(), s2.getBytesWritten()); //写的字节数
        if (val != 0) {
            return val;
        }
        val = Long.compare(s1.getBytesRead(), s2.getBytesRead()); //读的字节数
        if (val != 0) {
            return val;
        }
        val = Long.compare(s1.getKeysWritten(), s2.getKeysWritten()); //写入key的数量
        if (val != 0) {
            return val;
        }
        val = Long.compare(s1.getKeysRead(), s2.getKeysRead()); //读取key的数量
        if (val != 0) {
            return val;
        }
        return Long.compare(-s1.getAvailable(), -s2.getAvailable()); //可用空间
    });
    return min.getKey();
}
```

### 检测是否切分region

com.alipay.sofa.jraft.rhea.pipeline.handler.SplittingJudgeByApproximateKeysHandler#readMessage

```java
public void readMessage(final HandlerContext ctx, final RegionPingEvent event) throws Exception { //检测是否需要切分region，只考虑store多region少情况下的切分
    if (event.isReady()) {
        return;
    }
    final MetadataStore metadataStore = event.getMetadataStore();
    final RegionHeartbeatRequest request = event.getMessage();
  	//获取集群统计信息
    final long clusterId = request.getClusterId();
    final ClusterStatsManager clusterStatsManager = ClusterStatsManager.getInstance(clusterId);
    clusterStatsManager.addOrUpdateRegionStats(request.getRegionStatsList());
  	//获取集群下的storeId
    final Set<Long> stores = metadataStore.unsafeGetStoreIds(clusterId);
    if (stores == null || stores.isEmpty()) {
        return;
    }
    if (clusterStatsManager.regionSize() >= stores.size()) {
        // one store one region is perfect
        return;
    }
    //store多，但是region少的情况下进行region切分
  	//获取区间内含有key的数量最多的region以及统计信息
    final Pair<Region, RegionStats> modelWorker = clusterStatsManager.findModelWorkerRegion();
    if (!isSplitNeeded(request, modelWorker)) { //是否需要切分region
        return;
    }

    LOG.info("[Cluster: {}] model worker region is: {}.", clusterId, modelWorker);
		//创建newRegionId
    final Long newRegionId = metadataStore.createRegionId(clusterId);
  	//创建切分的指令，存储节点上报region信息时，会注册回调函数，回调函数中执行PD返回的指令
    final Instruction.RangeSplit rangeSplit = new Instruction.RangeSplit();
    rangeSplit.setNewRegionId(newRegionId);
    final Instruction instruction = new Instruction();
    instruction.setRegion(modelWorker.getKey().copy());
    instruction.setRangeSplit(rangeSplit);
    event.addInstruction(instruction);
}
```

com.alipay.sofa.jraft.rhea.pipeline.handler.SplittingJudgeByApproximateKeysHandler#isSplitNeeded

```java
private boolean isSplitNeeded(final RegionHeartbeatRequest request, final Pair<Region, RegionStats> modelWorker) {//是否需要切分
    if (modelWorker == null) {
        return false;
    }
    final long modelApproximateKeys = modelWorker.getValue().getApproximateKeys();
    if (request.getLeastKeysOnSplit() > modelApproximateKeys) {//默认情况，region含有的key的数量超过10_000才会进行切分
        return false;
    }
    final Region modelRegion = modelWorker.getKey();
    final List<Pair<Region, RegionStats>> regionStatsList = request.getRegionStatsList();
    for (final Pair<Region, RegionStats> p : regionStatsList) {
        if (modelRegion.equals(p.getKey())) {
            return true;
        }
    }
    return false;
}
```

## 处理获取集群信息请求

com.alipay.sofa.jraft.rhea.DefaultPlacementDriverService#handleGetClusterInfoRequest

```java
public void handleGetClusterInfoRequest(final GetClusterInfoRequest request,
                                        final RequestProcessClosure<BaseRequest, BaseResponse> closure) { //获取集群信息
    final long clusterId = request.getClusterId();
    final GetClusterInfoResponse response = new GetClusterInfoResponse();
    response.setClusterId(clusterId);
    if (!this.isLeader) {
        response.setError(Errors.NOT_LEADER);
        closure.sendResponse(response);
        return;
    }
    //必须是Leader节点
    try {
        final Cluster cluster = this.metadataStore.getClusterInfo(clusterId);
        response.setCluster(cluster);
    } catch (final Throwable t) {
        LOG.error("Failed to handle: {}, {}.", request, StackTraceUtil.stackTrace(t));
        response.setError(Errors.forException(t));
    }
    closure.sendResponse(response);
}
```

com.alipay.sofa.jraft.rhea.DefaultMetadataStore#getClusterInfo

```java
public Cluster getClusterInfo(final long clusterId) { //获取集群信息
  	//获取集群下的所有存储节点
    final Set<Long> storeIds = getClusterIndex(clusterId);
    if (storeIds == null) {
        return null;
    }
    final List<byte[]> storeKeys = Lists.newArrayList();
    for (final Long storeId : storeIds) {
        final String storeInfoKey = MetadataKeyHelper.getStoreInfoKey(clusterId, storeId);
        storeKeys.add(BytesUtil.writeUtf8(storeInfoKey));
    }
  	//获取Store信息，store包含了region的信息
    final Map<ByteArray, byte[]> storeInfoBytes = this.rheaKVStore.bMultiGet(storeKeys);
    final List<Store> stores = Lists.newArrayListWithCapacity(storeInfoBytes.size());
    for (final byte[] storeBytes : storeInfoBytes.values()) {
        final Store store = this.serializer.readObject(storeBytes, Store.class);
        stores.add(store);
    }
    return new Cluster(clusterId, stores);
}
```

# Pipeling执行流程

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline#invoke(com.alipay.sofa.jraft.rhea.util.pipeline.event.InboundMessageEvent<M>)

```java
public <R, M> PipelineFuture<R> invoke(InboundMessageEvent<M> event) {
    return invoke(event, -1);
}
```

```java
public <R, M> PipelineFuture<R> invoke(InboundMessageEvent<M> event, long timeoutMillis) {
    PipelineFuture<R> future = DefaultPipelineFuture.with(event.getInvokeId(), timeoutMillis); 
    head.fireInbound(event);
    return future;
}
```

## 创建Future

com.alipay.sofa.jraft.rhea.util.pipeline.future.DefaultPipelineFuture#with

```java
public static <T> DefaultPipelineFuture<T> with(final long invokeId, final long timeoutMillis) {//创建Future
    return new DefaultPipelineFuture<>(invokeId, timeoutMillis);
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.future.DefaultPipelineFuture#DefaultPipelineFuture

```java
private DefaultPipelineFuture(long invokeId, long timeoutMillis) {
    this.invokeId = invokeId;
    this.timeout = timeoutMillis > 0 ? TimeUnit.MILLISECONDS.toNanos(timeoutMillis) : DEFAULT_TIMEOUT_NANOSECONDS;
    futures.put(invokeId, this); //维护全部的Future对象
}
```

## 执行自定义业务逻辑

com.alipay.sofa.jraft.rhea.util.pipeline.AbstractHandlerContext#fireInbound

```java
public HandlerContext fireInbound(final InboundMessageEvent<?> event) {
  	//查找下一个Inbound类型的HandlerContext，直到执行到TrailContext，再执行outBound，将处理的指令信息返回给存储节点
    final AbstractHandlerContext next = findContextInbound(); 
    next.invoker().invokeInbound(next, event); //执行方法
    return this;
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultHandlerInvoker#invokeInbound

```java
public void invokeInbound(final HandlerContext ctx, final InboundMessageEvent<?> event) {
    if (this.executor == null) {
        HandlerInvokerUtil.invokeInboundNow(ctx, event);
    } else { //线程池处理
        this.executor.execute(() -> HandlerInvokerUtil.invokeInboundNow(ctx, event));
    }
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.HandlerInvokerUtil#invokeInboundNow

```java
public static void invokeInboundNow(final HandlerContext ctx, final InboundMessageEvent<?> event) {
    try {
        ((InboundHandler) ctx.handler()).handleInbound(ctx, event);
    } catch (final Throwable t) {
        notifyHandlerException(ctx, event, t);
    }
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.InboundHandlerAdapter#handleInbound

```java
public void handleInbound(final HandlerContext ctx, final InboundMessageEvent<?> event) throws Exception {
    if (isAcceptable(event)) { //事件类型是否匹配
        readMessage(ctx, (I) event); //执行自定义的逻辑
    }
    ctx.fireInbound(event); //执行下一个HandlerContext
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.InboundHandlerAdapter#isAcceptable

```java
public boolean isAcceptable(final MessageEvent<?> event) { //匹配事件类型
    return matcher.match(event);
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.InboundHandlerAdapter#InboundHandlerAdapter

```java
protected InboundHandlerAdapter() {//创建Handler时同时创建matcher对象
    this.matcher = TypeParameterMatcher.find(this, InboundHandlerAdapter.class, "I");
}
```

PlacementDriverTailHandler

com.alipay.sofa.jraft.rhea.pipeline.handler.PlacementDriverTailHandler#handleInbound

```java
public void handleInbound(final HandlerContext ctx, final InboundMessageEvent<?> event) throws Exception {
    if (isAcceptable(event)) {
        PingEvent<?> ping = (PingEvent<?>) event;
          ctx.fireOutbound(new PongEvent(ping.getInvokeId(),   	 	Lists.newArrayList(ping.getInstructions()))); //调用outBound，，最终调用HeadContext的outbound返回指令给存储节点
    }
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.DefaultPipeline.HeadContext#handleOutbound

```java
public void handleOutbound(HandlerContext ctx, OutboundMessageEvent<?> event) throws Exception {
    if (event != null) {
        DefaultPipelineFuture.received(event.getInvokeId(), event.getMessage());
    }
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.future.DefaultPipelineFuture#received

```java
public static void received(final long invokeId, final Object response) {
    final DefaultPipelineFuture<?> future = futures.remove(invokeId);//获取Future
    if (future == null) {
        LOG.warn("A timeout response [{}] finally returned.", response);
        return;
    }
    future.doReceived(response); //唤醒阻塞，执行回调
}
```

com.alipay.sofa.jraft.rhea.util.pipeline.future.DefaultPipelineFuture#doReceived

```java
private void doReceived(final Object response) {
    if (response instanceof Throwable) {
        completeExceptionally((Throwable) response);
    } else {
        complete((V) response);
    }
}
```
