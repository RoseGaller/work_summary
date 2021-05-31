# Dledger源码分析

* [启动流程](#启动流程)
  * [构造DLedgerServer](#构造dledgerserver)
    * [构造MemberState](#构造memberstate)
    * [创建DLedgerStore](#创建dledgerstore)
    * [创建DLedgerRpcNettyService](#创建dledgerrpcnettyservice)
    * [创建DLedgerEntryPusher](#创建dledgerentrypusher)
    * [创建DLedgerLeaderElector](#创建dledgerleaderelector)
  * [启动DLedgerServer](#启动dledgerserver)
    * [启动DLedgerStore](#启动dledgerstore)
    * [启动DLedgerRpcService](#启动dledgerrpcservice)
    * [启动DLedgerEntryPusher](#启动dledgerentrypusher)
    * [启动DLedgerLeaderElector](#启动dledgerleaderelector)
* [Leader](#leader)
  * [EntryDispatcher](#entrydispatcher)
    * [对比数据](#对比数据)
      * [主从数据一致](#主从数据一致)
      * [截断文件](#截断文件)
    * [追加数据](#追加数据)
      * [重发数据](#重发数据)
      * [同步数据](#同步数据)
  * [handleAppend](#handleappend)
    * [写本地磁盘](#写本地磁盘)
    * [等待过半节点的ACK](#等待过半节点的ack)
  * [QuorumAckChecker](#quorumackchecker)
  * [Preferred Leader](#preferred-leader)
* [Follower](#follower)
  * [接收push请求](#接收push请求)
  * [执行push请求](#执行push请求)
    * [handleDoAppend](#handledoappend)
    * [handleDoCompare](#handledocompare)
    * [handleDoTruncate](#handledotruncate)
    * [handleDoCommit](#handledocommit)


# 启动流程

```java
public class DLedger {

    private static Logger logger = LoggerFactory.getLogger(DLedger.class);

    public static void main(String args[]) {
        DLedgerConfig dLedgerConfig = new DLedgerConfig();
        //解析参数，封装成DLedgerConfig
        JCommander.newBuilder().addObject(dLedgerConfig).build().parse(args);
        //根据配置文件创建DLedgerServer
        DLedgerServer dLedgerServer = new DLedgerServer(dLedgerConfig);
        // 启动DLedgerServer
        dLedgerServer.startup();
        logger.info("[{}] group {} start ok with config {}", dLedgerConfig.getSelfId(), dLedgerConfig.getGroup(), JSON.toJSONString(dLedgerConfig));
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            private volatile boolean hasShutdown = false;

            @Override
            public void run() {
                synchronized (this) {
                    logger.info("Shutdown hook was invoked");
                    if (!this.hasShutdown) {
                        this.hasShutdown = true;
                        long beginTime = System.currentTimeMillis();
                        dLedgerServer.shutdown();
                        long consumingTimeTotal = System.currentTimeMillis() - beginTime;
                        logger.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
                    }
                }
            }
        }, "ShutdownHook"));
    }
}
```

## 构造DLedgerServer

io.openmessaging.storage.dledger.DLedgerServer#DLedgerServer

```java
public DLedgerServer(DLedgerConfig dLedgerConfig) {
    this.dLedgerConfig = dLedgerConfig;
    //成员状态
    this.memberState = new MemberState(dLedgerConfig);
    //默认文件存储
    this.dLedgerStore = createDLedgerStore(dLedgerConfig.getStoreType(), this.dLedgerConfig, this.memberState);
    //负责接收客户端请求、向远端发起请求
    dLedgerRpcService = new DLedgerRpcNettyService(this);
    //负责leader向Follower数据同步
    dLedgerEntryPusher = new DLedgerEntryPusher(dLedgerConfig, memberState, dLedgerStore, dLedgerRpcService);
    //负责leaer选举
    dLedgerLeaderElector = new DLedgerLeaderElector(dLedgerConfig, memberState, dLedgerRpcService);
    executorService = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread t = new Thread(r);
        t.setDaemon(true);
        t.setName("DLedgerServer-ScheduledExecutor");
        return t;
    });
}
```

### 构造MemberState

io.openmessaging.storage.dledger.MemberState#MemberState

```java
public MemberState(DLedgerConfig config) {
    this.group = config.getGroup();
    this.selfId = config.getSelfId();
    this.peers = config.getPeers();
    for (String peerInfo : this.peers.split(";")) {
        String peerSelfId = peerInfo.split("-")[0];
        String peerAddress = peerInfo.substring(selfId.length() + 1);
        peerMap.put(peerSelfId, peerAddress);
    }
    this.dLedgerConfig = config;
    loadTerm(); //从文件加载
}
```

io.openmessaging.storage.dledger.MemberState#loadTerm

```java
private void loadTerm() { //文件加载term、currVoteFor
    try {
        String data = IOUtils.file2String(dLedgerConfig.getDefaultPath() + File.separator + TERM_PERSIST_FILE);
        Properties properties = IOUtils.string2Properties(data);
        if (properties == null) {
            return;
        }
        if (properties.containsKey(TERM_PERSIST_KEY_TERM)) {
            currTerm = Long.valueOf(String.valueOf(properties.get(TERM_PERSIST_KEY_TERM)));
        }
        if (properties.containsKey(TERM_PERSIST_KEY_VOTE_FOR)) {
            currVoteFor = String.valueOf(properties.get(TERM_PERSIST_KEY_VOTE_FOR));
            if (currVoteFor.length() == 0) {
                currVoteFor = null;
            }
        }
    } catch (Throwable t) {
        logger.error("Load last term failed", t);
    }
}
```

### 创建DLedgerStore

io.openmessaging.storage.dledger.DLedgerServer#createDLedgerStore

```java
private DLedgerStore createDLedgerStore(String storeType, DLedgerConfig config, MemberState memberState) {
    if (storeType.equals(DLedgerConfig.MEMORY)) {
        return new DLedgerMemoryStore(config, memberState);
    } else {
        return new DLedgerMmapFileStore(config, memberState);
    }
}
```

### 创建DLedgerRpcNettyService

io.openmessaging.storage.dledger.DLedgerRpcNettyService#DLedgerRpcNettyService

```java
public DLedgerRpcNettyService(DLedgerServer dLedgerServer) {
    this.dLedgerServer = dLedgerServer;
    this.memberState = dLedgerServer.getMemberState();
    //请求处理器
    NettyRequestProcessor protocolProcessor = new NettyRequestProcessor() {
        @Override
        public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws Exception {
            return DLedgerRpcNettyService.this.processRequest(ctx, request);
        }

        @Override public boolean rejectRequest() {
            return false;
        }
    };
    //start the remoting server
    NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(Integer.valueOf(memberState.getSelfAddr().split(":")[1]));
    //创建NettyRemotingServer，接收远端请求
    this.remotingServer = new NettyRemotingServer(nettyServerConfig, null);
    //注册不同请求对应的处理器
    this.remotingServer.registerProcessor(DLedgerRequestCode.METADATA.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.APPEND.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.GET.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.PULL.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.PUSH.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.VOTE.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.HEART_BEAT.getCode(), protocolProcessor, null);
    this.remotingServer.registerProcessor(DLedgerRequestCode.LEADERSHIP_TRANSFER.getCode(), protocolProcessor, null);

    //创建NettyRemotingClient，向远端发起请求
    this.remotingClient = new NettyRemotingClient(new NettyClientConfig(), null);

}
```

### 创建DLedgerEntryPusher

io.openmessaging.storage.dledger.DLedgerEntryPusher#DLedgerEntryPusher

```java
public DLedgerEntryPusher(DLedgerConfig dLedgerConfig, MemberState memberState, DLedgerStore dLedgerStore,
    DLedgerRpcService dLedgerRpcService) {
    this.dLedgerConfig = dLedgerConfig;
    this.memberState = memberState;
    this.dLedgerStore = dLedgerStore;
    this.dLedgerRpcService = dLedgerRpcService;
    for (String peer : memberState.getPeerMap().keySet()) {
        if (!peer.equals(memberState.getSelfId())) { //集群中每个节点创建EntryDispatcher
            dispatcherMap.put(peer, new EntryDispatcher(peer, logger));//负责同步数据
        }
    }
}
```

### 创建DLedgerLeaderElector

io.openmessaging.storage.dledger.DLedgerLeaderElector#DLedgerLeaderElector

```java
public DLedgerLeaderElector(DLedgerConfig dLedgerConfig, MemberState memberState,
    DLedgerRpcService dLedgerRpcService) {
    this.dLedgerConfig = dLedgerConfig;
    this.memberState = memberState;
    this.dLedgerRpcService = dLedgerRpcService;
    refreshIntervals(dLedgerConfig); //刷新配置文件
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#refreshIntervals

```java
private void refreshIntervals(DLedgerConfig dLedgerConfig) {
    this.heartBeatTimeIntervalMs = dLedgerConfig.getHeartBeatTimeIntervalMs();
    this.maxHeartBeatLeak = dLedgerConfig.getMaxHeartBeatLeak();
    this.minVoteIntervalMs = dLedgerConfig.getMinVoteIntervalMs();
    this.maxVoteIntervalMs = dLedgerConfig.getMaxVoteIntervalMs();
}
```

## 启动DLedgerServer

io.openmessaging.storage.dledger.DLedgerServer#startup

```java
public void startup() {
    //加载数据、恢复
    this.dLedgerStore.startup();
    //绑定端口、初始化远程通信
    this.dLedgerRpcService.startup();
    //同步数据
    this.dLedgerEntryPusher.startup();
    //leader选举
    this.dLedgerLeaderElector.startup();
    //定时检测优先 Leader
    executorService.scheduleAtFixedRate(this::checkPreferredLeader, 1000, 1000, TimeUnit.MILLISECONDS);
}
```

### 启动DLedgerStore

io.openmessaging.storage.dledger.store.file.DLedgerMmapFileStore#startup

```java
public void startup() {
    load(); //加载数据文件、索引文件
    recover(); //恢复
    flushDataService.start(); //刷新数据
    cleanSpaceService.start(); //清理数据
}
```

### 启动DLedgerRpcService

io.openmessaging.storage.dledger.DLedgerRpcNettyService#startup

```java
public void startup() {
    this.remotingServer.start(); //绑定端口，监听客户端连接、请求
    this.remotingClient.start(); //远程通信客户端启动
}
```

### 启动DLedgerEntryPusher

io.openmessaging.storage.dledger.DLedgerEntryPusher#startup

```java
public void startup() {
    entryHandler.start(); //Follower节点接收leader的同步数据的请求
    quorumAckChecker.start();//检测数据是否有已经同步到过半的节点
    for (EntryDispatcher dispatcher : dispatcherMap.values()) {
        dispatcher.start(); //leader节点向Follower同步数据
    }
}
```

### 启动DLedgerLeaderElector

io.openmessaging.storage.dledger.DLedgerLeaderElector#startup

```java
public void startup() {
    stateMaintainer.start();
    for (RoleChangeHandler roleChangeHandler : roleChangeHandlers) {
        roleChangeHandler.startup();
    }
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector.StateMaintainer#doWork

```java
 public void doWork() {
    try {
        if (DLedgerLeaderElector.this.dLedgerConfig.isEnableLeaderElector()) {
            DLedgerLeaderElector.this.refreshIntervals(dLedgerConfig);
            DLedgerLeaderElector.this.maintainState();
        }
        sleep(10);
    } catch (Throwable t) {
        DLedgerLeaderElector.logger.error("Error in heartbeat", t);
    }
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#maintainState

```java
private void maintainState() throws Exception {
    if (memberState.isLeader()) { //当前为leader
        maintainAsLeader();
    } else if (memberState.isFollower()) { //当前为follower
        maintainAsFollower();
    } else { //当前为candidate(初始状态)
        maintainAsCandidate();
    }
}
```

发起投票

io.openmessaging.storage.dledger.DLedgerLeaderElector#maintainAsCandidate

```java
private void maintainAsCandidate() throws Exception {
    //for candidate
    if (System.currentTimeMillis() < nextTimeToRequestVote && !needIncreaseTermImmediately) {
        return;
    }
    long term;
    long ledgerEndTerm;
    long ledgerEndIndex;
    synchronized (memberState) {
        if (!memberState.isCandidate()) {
            return;
        }
        if (lastParseResult == VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT || needIncreaseTermImmediately) {
            long prevTerm = memberState.currTerm();
            term = memberState.nextTerm();
            logger.info("{}_[INCREASE_TERM] from {} to {}", memberState.getSelfId(), prevTerm, term);
            lastParseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
        } else {
            term = memberState.currTerm();
        }
        ledgerEndIndex = memberState.getLedgerEndIndex();
        ledgerEndTerm = memberState.getLedgerEndTerm();
    }
    if (needIncreaseTermImmediately) {
        nextTimeToRequestVote = getNextTimeToRequestVote();
        needIncreaseTermImmediately = false;
        return;
    }

    long startVoteTimeMs = System.currentTimeMillis();
    //向集群中节点发送投票请求
    final List<CompletableFuture<VoteResponse>> quorumVoteResponses = voteForQuorumResponses(term, ledgerEndTerm, ledgerEndIndex);
    final AtomicLong knownMaxTermInGroup = new AtomicLong(-1);
    final AtomicInteger allNum = new AtomicInteger(0);
    final AtomicInteger validNum = new AtomicInteger(0);
    final AtomicInteger acceptedNum = new AtomicInteger(0);
    final AtomicInteger notReadyTermNum = new AtomicInteger(0);
    final AtomicInteger biggerLedgerNum = new AtomicInteger(0);
    final AtomicBoolean alreadyHasLeader = new AtomicBoolean(false);

    CountDownLatch voteLatch = new CountDownLatch(1);
    for (CompletableFuture<VoteResponse> future : quorumVoteResponses) {
        future.whenComplete((VoteResponse x, Throwable ex) -> { //处理投票结果
            try {
                if (ex != null) {
                    throw ex;
                }
                logger.info("[{}][GetVoteResponse] {}", memberState.getSelfId(), JSON.toJSONString(x));
                if (x.getVoteResult() != VoteResponse.RESULT.UNKNOWN) {
                    validNum.incrementAndGet();
                }
                synchronized (knownMaxTermInGroup) {
                    switch (x.getVoteResult()) {
                        case ACCEPT:
                            acceptedNum.incrementAndGet();
                            break;
                        case REJECT_ALREADY_VOTED:
                        case REJECT_TAKING_LEADERSHIP:
                            break;
                        case REJECT_ALREADY_HAS_LEADER:
                            alreadyHasLeader.compareAndSet(false, true);
                            break;
                        case REJECT_TERM_SMALL_THAN_LEDGER:
                        case REJECT_EXPIRED_VOTE_TERM:
                            if (x.getTerm() > knownMaxTermInGroup.get()) {
                                knownMaxTermInGroup.set(x.getTerm());
                            }
                            break;
                        case REJECT_EXPIRED_LEDGER_TERM:
                        case REJECT_SMALL_LEDGER_END_INDEX:
                            biggerLedgerNum.incrementAndGet();
                            break;
                        case REJECT_TERM_NOT_READY:
                            notReadyTermNum.incrementAndGet();
                            break;
                        default:
                            break;

                    }
                }
                if (alreadyHasLeader.get()
                    || memberState.isQuorum(acceptedNum.get())
                    || memberState.isQuorum(acceptedNum.get() + notReadyTermNum.get())) {
                    voteLatch.countDown();
                }
            } catch (Throwable t) {
                logger.error("vote response failed", t);
            } finally {
                allNum.incrementAndGet();
                if (allNum.get() == memberState.peerSize()) {
                    voteLatch.countDown();
                }
            }
        });

    }
    try {
        //阻塞，直到收到过半的结果
        voteLatch.await(3000 + random.nextInt(maxVoteIntervalMs), TimeUnit.MILLISECONDS);
    } catch (Throwable ignore) {

    }
    lastVoteCost = DLedgerUtils.elapsed(startVoteTimeMs);
    VoteResponse.ParseResult parseResult;
    if (knownMaxTermInGroup.get() > term) {
        parseResult = VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT;
        nextTimeToRequestVote = getNextTimeToRequestVote();
        changeRoleToCandidate(knownMaxTermInGroup.get());
    } else if (alreadyHasLeader.get()) {
        parseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
        nextTimeToRequestVote = getNextTimeToRequestVote() + heartBeatTimeIntervalMs * maxHeartBeatLeak;
    } else if (!memberState.isQuorum(validNum.get())) {
        parseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
        nextTimeToRequestVote = getNextTimeToRequestVote();
    } else if (memberState.isQuorum(acceptedNum.get())) {
        parseResult = VoteResponse.ParseResult.PASSED;
    } else if (memberState.isQuorum(acceptedNum.get() + notReadyTermNum.get())) {
        parseResult = VoteResponse.ParseResult.REVOTE_IMMEDIATELY;
    } else if (memberState.isQuorum(acceptedNum.get() + biggerLedgerNum.get())) {
        parseResult = VoteResponse.ParseResult.WAIT_TO_REVOTE;
        nextTimeToRequestVote = getNextTimeToRequestVote();
    } else {
        parseResult = VoteResponse.ParseResult.WAIT_TO_VOTE_NEXT;
        nextTimeToRequestVote = getNextTimeToRequestVote();
    }
    lastParseResult = parseResult;
    logger.info("[{}] [PARSE_VOTE_RESULT] cost={} term={} memberNum={} allNum={} acceptedNum={} notReadyTermNum={} biggerLedgerNum={} alreadyHasLeader={} maxTerm={} result={}",
        memberState.getSelfId(), lastVoteCost, term, memberState.peerSize(), allNum, acceptedNum, notReadyTermNum, biggerLedgerNum, alreadyHasLeader, knownMaxTermInGroup.get(), parseResult);

    if (parseResult == VoteResponse.ParseResult.PASSED) { //当选leader
        logger.info("[{}] [VOTE_RESULT] has been elected to be the leader in term {}", memberState.getSelfId(), term);
        changeRoleToLeader(term);
    }

}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#changeRoleToLeader

```java
public void changeRoleToLeader(long term) {
    synchronized (memberState) {
        if (memberState.currTerm() == term) {
            memberState.changeToLeader(term);
            lastSendHeartBeatTime = -1;
            handleRoleChange(term, MemberState.Role.LEADER);
            logger.info("[{}] [ChangeRoleToLeader] from term: {} and currTerm: {}", memberState.getSelfId(), term, memberState.currTerm());
        } else {
            logger.warn("[{}] skip to be the leader in term: {}, but currTerm is: {}", memberState.getSelfId(), term, memberState.currTerm());
        }
    }
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#handleRoleChange

```java
private void handleRoleChange(long term, MemberState.Role role) {
    try {
        takeLeadershipTask.check(term, role);
    } catch (Throwable t) {
        logger.error("takeLeadershipTask.check failed. ter={}, role={}", term, role, t);
    }

    for (RoleChangeHandler roleChangeHandler : roleChangeHandlers) {
        try {
            roleChangeHandler.handle(term, role);
        } catch (Throwable t) {
            logger.warn("Handle role change failed term={} role={} handler={}", term, role, roleChangeHandler.getClass(), t);
        }
    }
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#maintainAsLeader

```java
private void maintainAsLeader() throws Exception {
    //每隔2s，进行一次心跳请求
    if (DLedgerUtils.elapsed(lastSendHeartBeatTime) > heartBeatTimeIntervalMs) {
        long term;
        String leaderId;
        synchronized (memberState) {
            if (!memberState.isLeader()) {
                //stop sending
                return;
            }
            term = memberState.currTerm();
            leaderId = memberState.getLeaderId();
            lastSendHeartBeatTime = System.currentTimeMillis();
        }
        sendHeartbeats(term, leaderId);
    }
}
```

# Leader

## EntryDispatcher

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#doWork

```java
public void doWork() { //主从数据同步
    try {
        if (!checkAndFreshState()) {
            waitForRunning(1);
            return;
        }
        //leader节点执行，系统启动时type=Compare,先进行文件的对比，决定后续时截断还是同步数据
        if (type.get() == PushEntryRequest.Type.APPEND) {
            if (dLedgerConfig.isEnableBatchPush()) { //是否批量同步数据，默认false
                doBatchAppend(); //批量同步
            } else {
                doAppend(); //单条同步
            }
        } else {
            doCompare(); //比较、截断
        }
        waitForRunning(1);
    } catch (Throwable t) {
        DLedgerEntryPusher.logger.error("[Push-{}]Error in {} writeIndex={} compareIndex={}", peerId, getName(), writeIndex, compareIndex, t);
        DLedgerUtils.sleep(500);
    }
}
```

### 对比数据

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#doCompare

```java
private void doCompare() throws Exception {
    while (true) {
        if (!checkAndFreshState()) {
            break;
        }
        //leader节点继续执行
        if (type.get() != PushEntryRequest.Type.COMPARE
            && type.get() != PushEntryRequest.Type.TRUNCATE) {
            break;
        }
        //leader节点尚无数据写入
        if (compareIndex == -1 && dLedgerStore.getLedgerEndIndex() == -1) {
            break;
        }
        //revise the compareIndex
        if (compareIndex == -1) {
            compareIndex = dLedgerStore.getLedgerEndIndex(); //leader节点最后一条数据的index
            logger.info("[Push-{}][DoCompare] compareIndex=-1 means start to compare", peerId);
        } else if (compareIndex > dLedgerStore.getLedgerEndIndex() || compareIndex < dLedgerStore.getLedgerBeginIndex()) {
            logger.info("[Push-{}][DoCompare] compareIndex={} out of range {}-{}", peerId, compareIndex, dLedgerStore.getLedgerBeginIndex(), dLedgerStore.getLedgerEndIndex());
            compareIndex = dLedgerStore.getLedgerEndIndex();
        }
        //leader根据compareIndex获取DLedgerEntry
        DLedgerEntry entry = dLedgerStore.get(compareIndex);
        PreConditions.check(entry != null, DLedgerResponseCode.INTERNAL_ERROR, "compareIndex=%d", compareIndex);
        //构建COMPARE类型的PushEntryRequest
        PushEntryRequest request = buildPushRequest(entry, PushEntryRequest.Type.COMPARE);
        //向Follower发送PushEntryRequest
        CompletableFuture<PushEntryResponse> responseFuture = dLedgerRpcService.push(request);
        PushEntryResponse response = responseFuture.get(3, TimeUnit.SECONDS);
        PreConditions.check(response != null, DLedgerResponseCode.INTERNAL_ERROR, "compareIndex=%d", compareIndex);
        PreConditions.check(response.getCode() == DLedgerResponseCode.INCONSISTENT_STATE.getCode() || response.getCode() == DLedgerResponseCode.SUCCESS.getCode()
            , DLedgerResponseCode.valueOf(response.getCode()), "compareIndex=%d", compareIndex);
        long truncateIndex = -1;

        if (response.getCode() == DLedgerResponseCode.SUCCESS.getCode()) {
            if (compareIndex == response.getEndIndex()) { //leader、Follower数据已经同步，
                changeState(compareIndex, PushEntryRequest.Type.APPEND);
                break;
            } else {
                truncateIndex = compareIndex; //截断的索引
            }
        } else if (response.getEndIndex() < dLedgerStore.getLedgerBeginIndex()
            || response.getBeginIndex() > dLedgerStore.getLedgerEndIndex()) {
            //follower节点的数据与leader无交集，通常是由于follower宕机了一段时间，leader删除了过期的数据
            truncateIndex = dLedgerStore.getLedgerBeginIndex();
        } else if (compareIndex < response.getBeginIndex()) {
            //这种情况很少发生，通常意味着磁盘损坏。
            truncateIndex = dLedgerStore.getLedgerBeginIndex();
        } else if (compareIndex > response.getEndIndex()) {
            compareIndex = response.getEndIndex();
        } else {
            compareIndex--;
        }
        if (compareIndex < dLedgerStore.getLedgerBeginIndex()) {
            truncateIndex = dLedgerStore.getLedgerBeginIndex(); //Follower截断的位置
        }
        //执行截断
        if (truncateIndex != -1) {
            changeState(truncateIndex, PushEntryRequest.Type.TRUNCATE);//当前修改为TRUNCATE
            doTruncate(truncateIndex);
            break;
        }
    }
}
```

#### 主从数据一致

Follower节点和Leader节点的数据保持一致，改为APPEND状态

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#changeState

```java
private synchronized void changeState(long index, PushEntryRequest.Type target) {
    logger.info("[Push-{}]Change state from {} to {} at {}", peerId, type.get(), target, index);
    switch (target) {
        case APPEND:
            compareIndex = -1;
            updatePeerWaterMark(term, peerId, index); //记录在任期内，每个节点的同步的位置
            quorumAckChecker.wakeup();// 唤醒阻塞的quorumAckChecker
            writeIndex = index + 1;//从writeIndex开始给从节点复制数据
            if (dLedgerConfig.isEnableBatchPush()) { //默认不开启批量同步
                resetBatchAppendEntryRequest(); //重置PushEntryRequest
            }
            break;
        case COMPARE:
            if (this.type.compareAndSet(PushEntryRequest.Type.APPEND, PushEntryRequest.Type.COMPARE)) {
                compareIndex = -1;
                if (dLedgerConfig.isEnableBatchPush()) {
                    batchPendingMap.clear();
                } else {
                    pendingMap.clear();
                }
            }
            break;
        case TRUNCATE:
            compareIndex = -1;
            break;
        default:
            break;
    }
    //修改当前的状态
    type.set(target);
}
```

#### 截断文件

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#doTruncate

```java
private void doTruncate(long truncateIndex) throws Exception {
    PreConditions.check(type.get() == PushEntryRequest.Type.TRUNCATE, DLedgerResponseCode.UNKNOWN);
    DLedgerEntry truncateEntry = dLedgerStore.get(truncateIndex);
    PreConditions.check(truncateEntry != null, DLedgerResponseCode.UNKNOWN);
    logger.info("[Push-{}]Will push data to truncate truncateIndex={} pos={}", peerId, truncateIndex, truncateEntry.getPos());
    PushEntryRequest truncateRequest = buildPushRequest(truncateEntry, PushEntryRequest.Type.TRUNCATE);
    PushEntryResponse truncateResponse = dLedgerRpcService.push(truncateRequest).get(3, TimeUnit.SECONDS);
    PreConditions.check(truncateResponse != null, DLedgerResponseCode.UNKNOWN, "truncateIndex=%d", truncateIndex);
    PreConditions.check(truncateResponse.getCode() == DLedgerResponseCode.SUCCESS.getCode(), DLedgerResponseCode.valueOf(truncateResponse.getCode()), "truncateIndex=%d", truncateIndex);
    lastPushCommitTimeMs = System.currentTimeMillis();
    //Follower响应成功，改变当前的状态为APPEND
    changeState(truncateIndex, PushEntryRequest.Type.APPEND);
}
```

### 追加数据

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#doAppend

```java
private void doAppend() throws Exception {
    while (true) {
        if (!checkAndFreshState()) {
            break;
        }
        //leader节点执行，并且当前状态为APPEND
        if (type.get() != PushEntryRequest.Type.APPEND) {
            break;
        }
        //Follower已经追赶上leader
        if (writeIndex > dLedgerStore.getLedgerEndIndex()) {
            doCommit();
            doCheckAppendResponse();
            break;
        }
        if (pendingMap.size() >= maxPendingSize || (DLedgerUtils.elapsed(lastCheckLeakTimeMs) > 1000)) {
            long peerWaterMark = getPeerWaterMark(term, peerId);
            for (Long index : pendingMap.keySet()) {
                if (index < peerWaterMark) {
                    pendingMap.remove(index);
                }
            }
            lastCheckLeakTimeMs = System.currentTimeMillis();
        }
        //尚未返回响应的请求数大于1000
        if (pendingMap.size() >= maxPendingSize) {
            doCheckAppendResponse(); //重发数据
            break; 
        }
        //同步数据
        doAppendInner(writeIndex);
        writeIndex++;
        }
    }
```

#### 重发数据

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#doCheckAppendResponse

```java
private void doCheckAppendResponse() throws Exception {
    long peerWaterMark = getPeerWaterMark(term, peerId);
    Long sendTimeMs = pendingMap.get(peerWaterMark + 1);
    if (sendTimeMs != null && System.currentTimeMillis() - sendTimeMs > dLedgerConfig.getMaxPushTimeOutMs()) {
        logger.warn("[Push-{}]Retry to push entry at {}", peerId, peerWaterMark + 1);
        doAppendInner(peerWaterMark + 1);
    }
}
```

#### 同步数据

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryDispatcher#doAppendInner

```java
private void doAppendInner(long index) throws Exception {
    //根据index获取数据
    DLedgerEntry entry = getDLedgerEntryForAppend(index);
    if (null == entry) {
        return;
    }
    checkQuotaAndWait(entry);
    //构建APPEND类型的请求
    PushEntryRequest request = buildPushRequest(entry, PushEntryRequest.Type.APPEND);
    CompletableFuture<PushEntryResponse> responseFuture = dLedgerRpcService.push(request);
    //暂存尚未响应的请求，默认暂存1000条
    pendingMap.put(index, System.currentTimeMillis());
    responseFuture.whenComplete((x, ex) -> {
        try {
            PreConditions.check(ex == null, DLedgerResponseCode.UNKNOWN);
            DLedgerResponseCode responseCode = DLedgerResponseCode.valueOf(x.getCode());
            switch (responseCode) {
                case SUCCESS:
                    pendingMap.remove(x.getIndex());//移除
                    updatePeerWaterMark(x.getTerm(), peerId, x.getIndex()); //修改任期内同步的位置
                    quorumAckChecker.wakeup(); //唤醒
                    break;
                case INCONSISTENT_STATE: //leader和Follower数据不一致
                    logger.info("[Push-{}]Get INCONSISTENT_STATE when push index={} term={}", peerId, x.getIndex(), x.getTerm());
                    changeState(-1, PushEntryRequest.Type.COMPARE); //进入比较状态
                    break;
                default:
                    logger.warn("[Push-{}]Get error response code {} {}", peerId, responseCode, x.baseInfo());
                    break;
            }
        } catch (Throwable t) {
            logger.error("", t);
        }
    });
    lastPushCommitTimeMs = System.currentTimeMillis();
}
```

## handleAppend

io.openmessaging.storage.dledger.DLedgerServer#handleAppend

```java
public CompletableFuture<AppendEntryResponse> handleAppend(AppendEntryRequest request) throws IOException { //处理客户端的数据追加请求
    try {
        PreConditions.check(memberState.getSelfId().equals(request.getRemoteId()), DLedgerResponseCode.UNKNOWN_MEMBER, "%s != %s", request.getRemoteId(), memberState.getSelfId());
        PreConditions.check(memberState.getGroup().equals(request.getGroup()), DLedgerResponseCode.UNKNOWN_GROUP, "%s != %s", request.getGroup(), memberState.getGroup());
        PreConditions.check(memberState.isLeader(), DLedgerResponseCode.NOT_LEADER);
        PreConditions.check(memberState.getTransferee() == null, DLedgerResponseCode.LEADER_TRANSFERRING);
        long currTerm = memberState.currTerm();
        if (dLedgerEntryPusher.isPendingFull(currTerm)) { //限流，防止堆积过多的AppendEntryResponse
            AppendEntryResponse appendEntryResponse = new AppendEntryResponse();
            appendEntryResponse.setGroup(memberState.getGroup());
            appendEntryResponse.setCode(DLedgerResponseCode.LEADER_PENDING_FULL.getCode());
            appendEntryResponse.setTerm(currTerm);
            appendEntryResponse.setLeaderId(memberState.getSelfId());
            return AppendFuture.newCompletedFuture(-1, appendEntryResponse);
        } else {
            DLedgerEntry dLedgerEntry = new DLedgerEntry();
            dLedgerEntry.setBody(request.getBody());
            DLedgerEntry resEntry = dLedgerStore.appendAsLeader(dLedgerEntry); //追加到本地磁盘
            return dLedgerEntryPusher.waitAck(resEntry);//等待过半节点返回ack
        }
    } catch (DLedgerException e) {
        logger.error("[{}][HandleAppend] failed", memberState.getSelfId(), e);
        AppendEntryResponse response = new AppendEntryResponse();
        response.copyBaseInfo(request);
        response.setCode(e.getCode().getCode());
        response.setLeaderId(memberState.getLeaderId());
        return AppendFuture.newCompletedFuture(-1, response);
    }
}
```

### 写本地磁盘

io.openmessaging.storage.dledger.store.file.DLedgerMmapFileStore#appendAsLeader

```java
public DLedgerEntry appendAsLeader(DLedgerEntry entry) {
    PreConditions.check(memberState.isLeader(), DLedgerResponseCode.NOT_LEADER);
    PreConditions.check(!isDiskFull, DLedgerResponseCode.DISK_FULL);
    ByteBuffer dataBuffer = localEntryBuffer.get();
    ByteBuffer indexBuffer = localIndexBuffer.get();
    DLedgerEntryCoder.encode(entry, dataBuffer);
    int entrySize = dataBuffer.remaining();
    synchronized (memberState) {
        PreConditions.check(memberState.isLeader(), DLedgerResponseCode.NOT_LEADER, null);
        PreConditions.check(memberState.getTransferee() == null, DLedgerResponseCode.LEADER_TRANSFERRING, null);

        long nextIndex = ledgerEndIndex + 1;
        entry.setIndex(nextIndex);
        entry.setTerm(memberState.currTerm());
        entry.setMagic(CURRENT_MAGIC);
        DLedgerEntryCoder.setIndexTerm(dataBuffer, nextIndex, memberState.currTerm(), CURRENT_MAGIC);

        long prePos = dataFileList.preAppend(dataBuffer.remaining());
        entry.setPos(prePos);
        PreConditions.check(prePos != -1, DLedgerResponseCode.DISK_ERROR, null);
        DLedgerEntryCoder.setPos(dataBuffer, prePos);

        for (AppendHook writeHook : appendHooks) {
            writeHook.doHook(entry, dataBuffer.slice(), DLedgerEntry.BODY_OFFSET);
        }

        long dataPos = dataFileList.append(dataBuffer.array(), 0, dataBuffer.remaining());
        PreConditions.check(dataPos != -1, DLedgerResponseCode.DISK_ERROR, null);
        PreConditions.check(dataPos == prePos, DLedgerResponseCode.DISK_ERROR, null);

        DLedgerEntryCoder.encodeIndex(dataPos, entrySize, CURRENT_MAGIC, nextIndex, memberState.currTerm(), indexBuffer);
        long indexPos = indexFileList.append(indexBuffer.array(), 0, indexBuffer.remaining(), false);
        PreConditions.check(indexPos == entry.getIndex() * INDEX_UNIT_SIZE, DLedgerResponseCode.DISK_ERROR, null);
        if (logger.isDebugEnabled()) {
            logger.info("[{}] Append as Leader {} {}", memberState.getSelfId(), entry.getIndex(), entry.getBody().length);
        }
        ledgerEndIndex++;
        ledgerEndTerm = memberState.currTerm();
        if (ledgerBeginIndex == -1) {
            ledgerBeginIndex = ledgerEndIndex;
        }
        updateLedgerEndIndexAndTerm();
        return entry;
    }
}
```

### 等待过半节点的ACK

io.openmessaging.storage.dledger.DLedgerEntryPusher#waitAck

```java
public CompletableFuture<AppendEntryResponse> waitAck(DLedgerEntry entry) {
    updatePeerWaterMark(entry.getTerm(), memberState.getSelfId(), entry.getIndex());
    if (memberState.getPeerMap().size() == 1) { //集群只有一个节点，直接返回响应信息
        AppendEntryResponse response = new AppendEntryResponse();
        response.setGroup(memberState.getGroup());
        response.setLeaderId(memberState.getSelfId());
        response.setIndex(entry.getIndex());
        response.setTerm(entry.getTerm());
        response.setPos(entry.getPos());
        return AppendFuture.newCompletedFuture(entry.getPos(), response);
    } else { //集群中有多个节点
        checkTermForPendingMap(entry.getTerm(), "waitAck");
        //默认最大等待时间2500ms
        AppendFuture<AppendEntryResponse> future = new AppendFuture<>(dLedgerConfig.getMaxWaitAckTimeMs());
        future.setPos(entry.getPos());
        //暂存到pendingAppendResponsesByTerm
        CompletableFuture<AppendEntryResponse> old = pendingAppendResponsesByTerm.get(entry.getTerm()).put(entry.getIndex(), future);
        if (old != null) {
            logger.warn("[MONITOR] get old wait at index={}", entry.getIndex());
        }
        return future;
    }
}
```

## QuorumAckChecker

每个追加请求得到半数节点的ack，返回响应给客户端

io.openmessaging.storage.dledger.DLedgerEntryPusher.QuorumAckChecker#doWork

```java
 public void doWork() {
        try {
            if (DLedgerUtils.elapsed(lastPrintWatermarkTimeMs) > 3000) {
                logger.info("[{}][{}] term={} ledgerBegin={} ledgerEnd={} committed={} watermarks={}",
                    memberState.getSelfId(), memberState.getRole(), memberState.currTerm(), dLedgerStore.getLedgerBeginIndex(), dLedgerStore.getLedgerEndIndex(), dLedgerStore.getCommittedIndex(), JSON.toJSONString(peerWaterMarksByTerm));
            lastPrintWatermarkTimeMs = System.currentTimeMillis();
        }
        if (!memberState.isLeader()) {
            waitForRunning(1);
            return;
        }
        //leader节点执行
        long currTerm = memberState.currTerm();
        checkTermForPendingMap(currTerm, "QuorumAckChecker");
        checkTermForWaterMark(currTerm, "QuorumAckChecker");

        //删除非当前term内的AppendEntryResponse
        if (pendingAppendResponsesByTerm.size() > 1) {
            for (Long term : pendingAppendResponsesByTerm.keySet()) {
                if (term == currTerm) {
                    continue;
                }
                //任期term已经发生了改变
                for (Map.Entry<Long, TimeoutFuture<AppendEntryResponse>> futureEntry : pendingAppendResponsesByTerm.get(term).entrySet()) {
                    AppendEntryResponse response = new AppendEntryResponse();
                    response.setGroup(memberState.getGroup());
                    response.setIndex(futureEntry.getKey());
                    response.setCode(DLedgerResponseCode.TERM_CHANGED.getCode());
                    response.setLeaderId(memberState.getLeaderId());
                    logger.info("[TermChange] Will clear the pending response index={} for term changed from {} to {}", futureEntry.getKey(), term, currTerm);
                    futureEntry.getValue().complete(response);
                }
                pendingAppendResponsesByTerm.remove(term);
            }
        }

        //删除非当前term内的Follower的同步进度
        if (peerWaterMarksByTerm.size() > 1) {
            for (Long term : peerWaterMarksByTerm.keySet()) {
                if (term == currTerm) {
                    continue;
                }
                logger.info("[TermChange] Will clear the watermarks for term changed from {} to {}", term, currTerm);
                peerWaterMarksByTerm.remove(term);
            }
        }

        //获取当前任期内，每个节点同步的进度（PeerId:Index）
        Map<String, Long> peerWaterMarks = peerWaterMarksByTerm.get(currTerm);

        //获取集群内接收到超过半数节点的ack的最大index，quorumIndex之前的数据至少有半数节点复制
        long quorumIndex = -1;
        for (Long index : peerWaterMarks.values()) {
            int num = 0;
            for (Long another : peerWaterMarks.values()) {
                if (another >= index) { //判断其他节点的同步进度是否快与自己
                    num++;
                }
            }
            //确保接收到超过集群半数节点的ack
            if (memberState.isQuorum(num) && index > quorumIndex) {
                quorumIndex = index;
            }
        }

        //更新committedIndex（索引小于等于quorumIndex的数据已经发送到集群内过半的节点上了）
        dLedgerStore.updateCommittedIndex(currTerm, quorumIndex);

        ConcurrentMap<Long, TimeoutFuture<AppendEntryResponse>> responses = pendingAppendResponsesByTerm.get(currTerm);
        boolean needCheck = false;
        int ackNum = 0;
        if (quorumIndex >= 0) {
            for (Long i = quorumIndex; i >= 0; i--) {
                try {
                    CompletableFuture<AppendEntryResponse> future = responses.remove(i);
                    if (future == null) { //可能follower返回的ack很快，但是dLedgerEntryPusher.waitAck执行太慢未将future放入pendingAppendResponsesByTerm
                        needCheck = lastQuorumIndex != -1 && lastQuorumIndex != quorumIndex && i != lastQuorumIndex;
                        break;
                    } else if (!future.isDone()) {
                        AppendEntryResponse response = new AppendEntryResponse();
                        response.setGroup(memberState.getGroup());
                        response.setTerm(currTerm);
                        response.setIndex(i);
                        response.setLeaderId(memberState.getSelfId());
                        response.setPos(((AppendFuture) future).getPos());
                        future.complete(response);
                    }
                    ackNum++;
                } catch (Throwable t) {
                    logger.error("Error in ack to index={}  term={}", i, currTerm, t);
                }
            }
        }

        if (ackNum == 0) { // quorumIndex= -1,说明没有一条消息的ack数量超过集群内的半数节点的数量
            for (long i = quorumIndex + 1; i < Integer.MAX_VALUE; i++) {
                TimeoutFuture<AppendEntryResponse> future = responses.get(i);
                if (future == null) {
                    break;
                } else if (future.isTimeOut()) { //判断是否等待超时，判断是否等待超时，响应超时请求
                    AppendEntryResponse response = new AppendEntryResponse();
                    response.setGroup(memberState.getGroup());
                    response.setCode(DLedgerResponseCode.WAIT_QUORUM_ACK_TIMEOUT.getCode());
                    response.setTerm(currTerm);
                    response.setIndex(i);
                    response.setLeaderId(memberState.getSelfId());
                    future.complete(response);
                } else {
                    break;
                }
            }
            waitForRunning(1);
        }

        //检测时间超过1s或者防止出现LedgerEntryPusher.waitAck执行太慢，没来得及将future放入pendingAppendResponsesByTerm,半数的节点已经返回ack的情况
        if (DLedgerUtils.elapsed(lastCheckLeakTimeMs) > 1000 || needCheck) {
            updatePeerWaterMark(currTerm, memberState.getSelfId(), dLedgerStore.getLedgerEndIndex());
            for (Map.Entry<Long, TimeoutFuture<AppendEntryResponse>> futureEntry : responses.entrySet()) {
                if (futureEntry.getKey() < quorumIndex) {
                    AppendEntryResponse response = new AppendEntryResponse();
                    response.setGroup(memberState.getGroup());
                    response.setTerm(currTerm);
                    response.setIndex(futureEntry.getKey());
                    response.setLeaderId(memberState.getSelfId());
                    response.setPos(((AppendFuture) futureEntry.getValue()).getPos());
                    futureEntry.getValue().complete(response);

                    responses.remove(futureEntry.getKey());
                }
            }
            lastCheckLeakTimeMs = System.currentTimeMillis();
        }
        lastQuorumIndex = quorumIndex;
    } catch (Throwable t) {
        DLedgerEntryPusher.logger.error("Error in {}", getName(), t);
        DLedgerUtils.sleep(100);
    }
}
```

## Preferred Leader

io.openmessaging.storage.dledger.DLedgerServer#checkPreferredLeader

```java
private void checkPreferredLeader() { //Leader节点主动触发，将Leader转让给优先节点
    if (!memberState.isLeader()) {
        return;
    }
    //leader节点继续往下执行
    String preferredLeaderId = dLedgerConfig.getPreferredLeaderId(); //优先选为leader的节点
    if (preferredLeaderId == null || preferredLeaderId.equals(dLedgerConfig.getSelfId())) { //已经转让完成
        return;
    }

    if (!memberState.isPeerMember(preferredLeaderId)) { //不合法
        logger.warn("preferredLeaderId = {} is not a peer member", preferredLeaderId);
        return;
    }

    if (memberState.getTransferee() != null) { //当前leader正在转移
        return;
    }

    if (!memberState.getPeersLiveTable().containsKey(preferredLeaderId) ||
        memberState.getPeersLiveTable().get(preferredLeaderId) == Boolean.FALSE) {//Preferred Leader不健康 				
        logger.warn("preferredLeaderId = {} is not online", preferredLeaderId);
        return;
    }
    //落后leader的数据小于1000，才会进行切换
    long fallBehind = dLedgerStore.getLedgerEndIndex() - dLedgerEntryPusher.getPeerWaterMark(memberState.currTerm(), preferredLeaderId);
    logger.info("transferee fall behind index : {}", fallBehind);
    if (fallBehind < dLedgerConfig.getMaxLeadershipTransferWaitIndex()) { //默认1000
        LeadershipTransferRequest request = new LeadershipTransferRequest();
        request.setTerm(memberState.currTerm());
        request.setTransfereeId(dLedgerConfig.getPreferredLeaderId()); //新的leader Id

        try {
            long startTransferTime = System.currentTimeMillis();
            //发起leader切换
            LeadershipTransferResponse response = dLedgerLeaderElector.handleLeadershipTransfer(request).get();
            logger.info("transfer finished. request={},response={},cost={}ms", request, response, DLedgerUtils.elapsed(startTransferTime));
        } catch (Throwable t) {
            logger.error("[checkPreferredLeader] error, request={}", request, t);
        }
    }
}
```

向优先节点发送Leader转让请求

io.openmessaging.storage.dledger.DLedgerLeaderElector#handleLeadershipTransfer

```java
public CompletableFuture<LeadershipTransferResponse> handleLeadershipTransfer(
    LeadershipTransferRequest request) throws Exception {
    logger.info("handleLeadershipTransfer: {}", request);
    synchronized (memberState) {
        if (memberState.currTerm() != request.getTerm()) { //term发生变更
            logger.warn("[BUG] [HandleLeaderTransfer] currTerm={} != request.term={}", memberState.currTerm(), request.getTerm());
            return CompletableFuture.completedFuture(new LeadershipTransferResponse().term(memberState.currTerm()).code(DLedgerResponseCode.INCONSISTENT_TERM.getCode()));
        }

        if (!memberState.isLeader()) { //不再是leader
            logger.warn("[BUG] [HandleLeaderTransfer] selfId={} is not leader", request.getLeaderId());
            return CompletableFuture.completedFuture(new LeadershipTransferResponse().term(memberState.currTerm()).code(DLedgerResponseCode.NOT_LEADER.getCode()));
        }

        if (memberState.getTransferee() != null) { //之前的leader转让尚未完成
            logger.warn("[BUG] [HandleLeaderTransfer] transferee={} is already set", memberState.getTransferee());
            return CompletableFuture.completedFuture(new LeadershipTransferResponse().term(memberState.currTerm()).code(DLedgerResponseCode.LEADER_TRANSFERRING.getCode()));
        }
        //正在转让leader，禁止写操作
        memberState.setTransferee(request.getTransfereeId());
    }
    //封装leader转让请求
    LeadershipTransferRequest takeLeadershipRequest = new LeadershipTransferRequest();
    takeLeadershipRequest.setGroup(memberState.getGroup());
    takeLeadershipRequest.setLeaderId(memberState.getLeaderId());
    takeLeadershipRequest.setLocalId(memberState.getSelfId());
    takeLeadershipRequest.setRemoteId(request.getTransfereeId());
    takeLeadershipRequest.setTerm(request.getTerm());
    takeLeadershipRequest.setTakeLeadershipLedgerIndex(memberState.getLedgerEndIndex());
    takeLeadershipRequest.setTransferId(memberState.getSelfId());
    takeLeadershipRequest.setTransfereeId(request.getTransfereeId());
    if (memberState.currTerm() != request.getTerm()) { //term发生了变更
        logger.warn("[HandleLeaderTransfer] term changed, cur={} , request={}", memberState.currTerm(), request.getTerm());
        return CompletableFuture.completedFuture(new LeadershipTransferResponse().term(memberState.currTerm()).code(DLedgerResponseCode.EXPIRED_TERM.getCode()));
    }
    //发起请求
    return dLedgerRpcService.leadershipTransfer(takeLeadershipRequest).thenApply(response -> {
        synchronized (memberState) {
            if (memberState.currTerm() == request.getTerm() && memberState.getTransferee() != null) {
                logger.warn("leadershipTransfer failed, set transferee to null");
                memberState.setTransferee(null);
            }
        }
        return response;
    });
}
```

处理转让

io.openmessaging.storage.dledger.DLedgerServer#handleLeadershipTransfer

```java
public CompletableFuture<LeadershipTransferResponse> handleLeadershipTransfer(LeadershipTransferRequest request) throws Exception {
    try {
        PreConditions.check(memberState.getSelfId().equals(request.getRemoteId()), DLedgerResponseCode.UNKNOWN_MEMBER, "%s != %s", request.getRemoteId(), memberState.getSelfId());
        PreConditions.check(memberState.getGroup().equals(request.getGroup()), DLedgerResponseCode.UNKNOWN_GROUP, "%s != %s", request.getGroup(), memberState.getGroup());
        if (memberState.getSelfId().equals(request.getTransferId())) { //当前的leader收到transfer command
            //It's the leader received the transfer command.
            PreConditions.check(memberState.isPeerMember(request.getTransfereeId()), DLedgerResponseCode.UNKNOWN_MEMBER, "transferee=%s is not a peer member", request.getTransfereeId());
            PreConditions.check(memberState.currTerm() == request.getTerm(), DLedgerResponseCode.INCONSISTENT_TERM, "currTerm(%s) != request.term(%s)", memberState.currTerm(), request.getTerm());
            PreConditions.check(memberState.isLeader(), DLedgerResponseCode.NOT_LEADER, "selfId=%s is not leader=%s", memberState.getSelfId(), memberState.getLeaderId());

            // 检测是否落后太多数据
            long transfereeFallBehind = dLedgerStore.getLedgerEndIndex() - dLedgerEntryPusher.getPeerWaterMark(request.getTerm(), request.getTransfereeId());
            PreConditions.check(transfereeFallBehind < dLedgerConfig.getMaxLeadershipTransferWaitIndex(),
                DLedgerResponseCode.FALL_BEHIND_TOO_MUCH, "transferee fall behind too much, diff=%s", transfereeFallBehind);
            return dLedgerLeaderElector.handleLeadershipTransfer(request);
        } else if (memberState.getSelfId().equals(request.getTransfereeId())) { //优先leader节点收到leader的转让
            // It's the transferee received the take leadership command.
            PreConditions.check(request.getTransferId().equals(memberState.getLeaderId()), DLedgerResponseCode.INCONSISTENT_LEADER, "transfer=%s is not leader", request.getTransferId());

            return dLedgerLeaderElector.handleTakeLeadership(request);
        } else {
            return CompletableFuture.completedFuture(new LeadershipTransferResponse().term(memberState.currTerm()).code(DLedgerResponseCode.UNEXPECTED_ARGUMENT.getCode()));
        }
    } catch (DLedgerException e) {
        logger.error("[{}][handleLeadershipTransfer] failed", memberState.getSelfId(), e);
        LeadershipTransferResponse response = new LeadershipTransferResponse();
        response.copyBaseInfo(request);
        response.setCode(e.getCode().getCode());
        response.setLeaderId(memberState.getLeaderId());
        return CompletableFuture.completedFuture(response);
    }
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#handleTakeLeadership

```java
public CompletableFuture<LeadershipTransferResponse> handleTakeLeadership(LeadershipTransferRequest request) throws Exception { //优先节点处理leaer转让
    logger.debug("handleTakeLeadership.request={}", request);
    synchronized (memberState) {
        if (memberState.currTerm() != request.getTerm()) {
            logger.warn("[BUG] [handleTakeLeadership] currTerm={} != request.term={}", memberState.currTerm(), request.getTerm());
            return CompletableFuture.completedFuture(new LeadershipTransferResponse().term(memberState.currTerm()).code(DLedgerResponseCode.INCONSISTENT_TERM.getCode()));
        }

        long targetTerm = request.getTerm() + 1; //term + 1
        memberState.setTermToTakeLeadership(targetTerm); //设置转让term
        CompletableFuture<LeadershipTransferResponse> response = new CompletableFuture<>();
        takeLeadershipTask.update(request, response);//takeLeadershipTask负责检测转让是否完成
        changeRoleToCandidate(targetTerm); //状态改变为CANDIDATE
        needIncreaseTermImmediately = true;
        return response;
    }
}
```

io.openmessaging.storage.dledger.DLedgerLeaderElector#changeRoleToCandidate

```java
public void changeRoleToCandidate(long term) {
    synchronized (memberState) {
        if (term >= memberState.currTerm()) {
            memberState.changeToCandidate(term);
            handleRoleChange(term, MemberState.Role.CANDIDATE);
            logger.info("[{}] [ChangeRoleToCandidate] from term: {} and currTerm: {}", memberState.getSelfId(), term, memberState.currTerm());
        } else {
            logger.info("[{}] skip to be candidate in term: {}, but currTerm: {}", memberState.getSelfId(), term, memberState.currTerm());
        }
    }
}
```

# Follower

## 接收push请求

根据请求的类型，分别放入不同的集合队列中。系统启动时开启线程，从队列获取请求开始执行

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryHandler#handlePush

```java
public CompletableFuture<PushEntryResponse> handlePush(PushEntryRequest request) throws Exception {
    //The timeout should smaller than the remoting layer's request timeout
    CompletableFuture<PushEntryResponse> future = new TimeoutFuture<>(1000);
    switch (request.getType()) {
        case APPEND: //追加请求，将请求放入writeRequestMap（index ->Pair(request,future) ）
            if (dLedgerConfig.isEnableBatchPush()) { //批量
                PreConditions.check(request.getBatchEntry() != null && request.getCount() > 0, DLedgerResponseCode.UNEXPECTED_ARGUMENT);
                long firstIndex = request.getFirstEntryIndex();
                writeRequestMap.put(firstIndex, new Pair<>(request, future));
            } else { //单条
                PreConditions.check(request.getEntry() != null, DLedgerResponseCode.UNEXPECTED_ARGUMENT);
                long index = request.getEntry().getIndex();
                Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> old = writeRequestMap.putIfAbsent(index, new Pair<>(request, future));
                if (old != null) {
                    logger.warn("[MONITOR]The index {} has already existed with {} and curr is {}", index, old.getKey().baseInfo(), request.baseInfo());
                    future.complete(buildResponse(request, DLedgerResponseCode.REPEATED_PUSH.getCode()));
                }
            }
            break;
        case COMMIT: //提交
            compareOrTruncateRequests.put(new Pair<>(request, future));
            break;
        case COMPARE: //对比
        case TRUNCATE: //截断
            PreConditions.check(request.getEntry() != null, DLedgerResponseCode.UNEXPECTED_ARGUMENT);
            writeRequestMap.clear();
            compareOrTruncateRequests.put(new Pair<>(request, future));
            break;
        default:
            logger.error("[BUG]Unknown type {} from {}", request.getType(), request.baseInfo());
            future.complete(buildResponse(request, DLedgerResponseCode.UNEXPECTED_ARGUMENT.getCode()));
            break;
    }
    return future;
}
```

## 执行push请求

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryHandler#doWork

```java
public void doWork() {
    try {
        if (!memberState.isFollower()) { //非Follower节点直接返回
            waitForRunning(1);
            return;
        }
        //Follower节点继续执行，优先处理对比截断请求
        if (compareOrTruncateRequests.peek() != null) {
            Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> pair = compareOrTruncateRequests.poll();
            PreConditions.check(pair != null, DLedgerResponseCode.UNKNOWN);
            switch (pair.getKey().getType()) {
                case TRUNCATE:
                    handleDoTruncate(pair.getKey().getEntry().getIndex(), pair.getKey(), pair.getValue());
                    break;
                case COMPARE:
                    handleDoCompare(pair.getKey().getEntry().getIndex(), pair.getKey(), pair.getValue());
                    break;
                case COMMIT:
                    handleDoCommit(pair.getKey().getCommitIndex(), pair.getKey(), pair.getValue());
                    break;
                default:
                    break;
            }
        } else { //追加请求
            long nextIndex = dLedgerStore.getLedgerEndIndex() + 1;
            Pair<PushEntryRequest, CompletableFuture<PushEntryResponse>> pair = writeRequestMap.remove(nextIndex);
            if (pair == null) { //追加请求有误
                checkAbnormalFuture(dLedgerStore.getLedgerEndIndex());
                waitForRunning(1);
                return;
            }
            PushEntryRequest request = pair.getKey();
            if (dLedgerConfig.isEnableBatchPush()) {
                handleDoBatchAppend(nextIndex, request, pair.getValue());
            } else {
                handleDoAppend(nextIndex, request, pair.getValue());
            }
        }
    } catch (Throwable t) {
        DLedgerEntryPusher.logger.error("Error in {}", getName(), t);
        DLedgerUtils.sleep(100);
    }
}
```

### handleDoAppend

直接追加数据到磁盘文件

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryHandler#handleDoAppend

```java
private void handleDoAppend(long writeIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) { //写磁盘
    try {
        PreConditions.check(writeIndex == request.getEntry().getIndex(), DLedgerResponseCode.INCONSISTENT_STATE);
        DLedgerEntry entry = dLedgerStore.appendAsFollower(request.getEntry(), request.getTerm(), request.getLeaderId());
        PreConditions.check(entry.getIndex() == writeIndex, DLedgerResponseCode.INCONSISTENT_STATE);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
        dLedgerStore.updateCommittedIndex(request.getTerm(), request.getCommitIndex());
    } catch (Throwable t) {
        logger.error("[HandleDoWrite] writeIndex={}", writeIndex, t);
        future.complete(buildResponse(request, DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
    }
}
```

### handleDoCompare

根据索引获取数据，和Leader的数据比较，判断是否相同，如果不同返回INCONSISTENT_STATE。如果相同，返回SUCCESS，后续Leader开始向Follower同步数据

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryHandler#handleDoCompare

```java
private CompletableFuture<PushEntryResponse> handleDoCompare(long compareIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        PreConditions.check(compareIndex == request.getEntry().getIndex(), DLedgerResponseCode.UNKNOWN);
        PreConditions.check(request.getType() == PushEntryRequest.Type.COMPARE, DLedgerResponseCode.UNKNOWN);
        //获取数据
        DLedgerEntry local = dLedgerStore.get(compareIndex);
        //如果Follower和Leader节点的数据不一致，返回INCONSISTENT_STATE
        PreConditions.check(request.getEntry().equals(local), DLedgerResponseCode.INCONSISTENT_STATE);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
    } catch (Throwable t) {
        logger.error("[HandleDoCompare] compareIndex={}", compareIndex, t);
        future.complete(buildResponse(request, DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
    }
    return future;
}
```

### handleDoTruncate

将与Leader不一致的数据删除

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryHandler#handleDoTruncate

```java
private CompletableFuture<PushEntryResponse> handleDoTruncate(long truncateIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        logger.info("[HandleDoTruncate] truncateIndex={} pos={}", truncateIndex, request.getEntry().getPos());
        PreConditions.check(truncateIndex == request.getEntry().getIndex(), DLedgerResponseCode.UNKNOWN);
        PreConditions.check(request.getType() == PushEntryRequest.Type.TRUNCATE, DLedgerResponseCode.UNKNOWN);
      //文件截断
        long index = dLedgerStore.truncate(request.getEntry(), request.getTerm(), request.getLeaderId());
        PreConditions.check(index == truncateIndex, DLedgerResponseCode.INCONSISTENT_STATE);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
        dLedgerStore.updateCommittedIndex(request.getTerm(), request.getCommitIndex());
    } catch (Throwable t) {
        logger.error("[HandleDoTruncate] truncateIndex={}", truncateIndex, t);
        future.complete(buildResponse(request, DLedgerResponseCode.INCONSISTENT_STATE.getCode()));
    }
    return future;
}
```

### handleDoCommit

io.openmessaging.storage.dledger.DLedgerEntryPusher.EntryHandler#handleDoCommit

```java
private CompletableFuture<PushEntryResponse> handleDoCommit(long committedIndex, PushEntryRequest request,
    CompletableFuture<PushEntryResponse> future) {
    try {
        PreConditions.check(committedIndex == request.getCommitIndex(), DLedgerResponseCode.UNKNOWN);
        PreConditions.check(request.getType() == PushEntryRequest.Type.COMMIT, DLedgerResponseCode.UNKNOWN);
        dLedgerStore.updateCommittedIndex(request.getTerm(), committedIndex);
        future.complete(buildResponse(request, DLedgerResponseCode.SUCCESS.getCode()));
    } catch (Throwable t) {
        logger.error("[HandleDoCommit] committedIndex={}", request.getCommitIndex(), t);
        future.complete(buildResponse(request, DLedgerResponseCode.UNKNOWN.getCode()));
    }
    return future;
}
```

io.openmessaging.storage.dledger.store.file.DLedgerMmapFileStore#updateCommittedIndex

```java
public void updateCommittedIndex(long term, long newCommittedIndex) {
    if (newCommittedIndex == -1
        || ledgerEndIndex == -1
        || term < memberState.currTerm()
        || newCommittedIndex == this.committedIndex) {
        return;
    }
    if (newCommittedIndex < this.committedIndex
        || newCommittedIndex < this.ledgerBeginIndex) { //不合法
        logger.warn("[MONITOR]Skip update committed index for new={} < old={} or new={} < beginIndex={}", newCommittedIndex, this.committedIndex, newCommittedIndex, this.ledgerBeginIndex);
        return;
    }
    long endIndex = ledgerEndIndex;
    if (newCommittedIndex > endIndex) {
        //如果Follower落后太多，committedIndex大于enIndex
        newCommittedIndex = endIndex; //修改newCommittedIndex
    }
    //根据索引获取数据
    DLedgerEntry dLedgerEntry = get(newCommittedIndex);
    //数据不能为空
    PreConditions.check(dLedgerEntry != null, DLedgerResponseCode.DISK_ERROR);
    //修改提交的索引
    this.committedIndex = newCommittedIndex;
    //修改提交的位置
    this.committedPos = dLedgerEntry.getPos() + dLedgerEntry.getSize();
}
```
