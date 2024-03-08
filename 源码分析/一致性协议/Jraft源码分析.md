# 	Jraft源码分析

* [Leader选举](#leader选举)
  * [预投票](#预投票)
    * [选举超时定时器](#选举超时定时器)
    * [处理选举超时](#处理选举超时)
      * [判断是否允许选举](#判断是否允许选举)
      * [预投票](#预投票-1)
      * [回调](#回调)
  * [投票](#投票)
    * [投票定时器](#投票定时器)
    * [投票响应回调](#投票响应回调)
  * [当选Leader](#当选leader)
* [数据同步](#数据同步)
  * [创建Replicator](#创建replicator)
  * [心跳超时检测](#心跳超时检测)
  * [发送空请求](#发送空请求)
    * [填充请求](#填充请求)
    * [回调方法](#回调方法)
    * [追加消息的回调](#追加消息的回调)
  * [发送消息](#发送消息)
    * [获取发送的索引](#获取发送的索引)
    * [复制数据](#复制数据)
      * [从磁盘读取数据](#从磁盘读取数据)
* [快照](#快照)
  * [生成快照](#生成快照)
    * [定时器触发](#定时器触发)
    * [快照请求触发](#快照请求触发)
    * [实现](#实现)
  * [安装快照*](#安装快照)
* [处理请求](#处理请求)
  * [初始化Disruptor](#初始化disruptor)
  * [发布客户端请求事件](#发布客户端请求事件)
  * [处理客户端请求事件](#处理客户端请求事件)
  * [发布磁盘写事件](#发布磁盘写事件)
  * [处理磁盘写事件](#处理磁盘写事件)
  * [写磁盘回调](#写磁盘回调)
  * [发布提交事件](#发布提交事件)
  * [处理提交事件](#处理提交事件)
  * [应用到状态机](#应用到状态机)
    * [CounterStateMachine](#counterstatemachine)
    * [AtomicStateMachine](#atomicstatemachine)
* [线性一致读](#线性一致读)
  * [实现1](#实现1)
  * [实现2](#实现2)
    * [接收readIndex请求](#接收readindex请求)
    * [发布readIndex事件](#发布readindex事件)
    * [处理readIndex事件](#处理readindex事件)
      * [Leaer节点处理](#leaer节点处理)
        * [ReadOnlySafe](#readonlysafe)
          * [心跳回调](#心跳回调)
      * [线性读回调](#线性读回调)
      * [唤醒阻塞ReadIndex请求](#唤醒阻塞readindex请求)
* [扩容](#扩容)
  * [添加follower节点](#添加follower节点)
  * [添加Learner节点](#添加learner节点)
* [Leader转让](#leader转让)
    * [Leader节点处理转让](#leader节点处理转让)
      * [转让leader给从节点](#转让leader给从节点)
      * [转让超时](#转让超时)
  * [Follower接收转让请求](#follower接收转让请求)
* [总结](#总结)

# 快速入门

com.alipay.sofa.jraft.example.counter.CounterServer#main

```java
public static void main(final String[] args) throws IOException {
    if (args.length != 4) {
        System.out
            .println("Useage : java com.alipay.sofa.jraft.example.counter.CounterServer {dataPath} {groupId} {serverId} {initConf}");
        System.out
            .println("Example: java com.alipay.sofa.jraft.example.counter.CounterServer /tmp/server1 counter 127.0.0.1:8081 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083");
        System.exit(1);
    }
    final String dataPath = args[0];
    final String groupId = args[1];
    final String serverIdStr = args[2];
    final String initConfStr = args[3];

    final NodeOptions nodeOptions = new NodeOptions();
    // 为了测试,调整 snapshot 间隔等参数
    // 设置选举超时时间为 1 秒
    nodeOptions.setElectionTimeoutMs(1000);
    // 关闭 CLI 服务。
    nodeOptions.setDisableCli(false);
    // 每隔30秒做一次 snapshot
    nodeOptions.setSnapshotIntervalSecs(30);
    // 解析参数
    final PeerId serverId = new PeerId();
    if (!serverId.parse(serverIdStr)) {
        throw new IllegalArgumentException("Fail to parse serverId:" + serverIdStr);
    }
    final Configuration initConf = new Configuration();
    if (!initConf.parse(initConfStr)) {
        throw new IllegalArgumentException("Fail to parse initConf:" + initConfStr);
    }
    // 设置初始集群配置
    nodeOptions.setInitialConf(initConf);

    // 启动
    final CounterServer counterServer = new CounterServer(dataPath, groupId, serverId, nodeOptions);
    System.out.println("Started counter server at port:"
                       + counterServer.getNode().getNodeId().getPeerId().getPort());
}
```

com.alipay.sofa.jraft.example.counter.CounterServer#CounterServer

```java
public CounterServer(final String dataPath, final String groupId, final PeerId serverId,
                     final NodeOptions nodeOptions) throws IOException {
    // 初始化路径
    FileUtils.forceMkdir(new File(dataPath));

    // 这里让 raft RPC 和业务 RPC 使用同一个 RPC server, 通常也可以分开
    final RpcServer rpcServer = RaftRpcServerFactory.createRaftRpcServer(serverId.getEndpoint());
    // 注册业务处理器
    CounterService counterService = new CounterServiceImpl(this);
    rpcServer.registerProcessor(new GetValueRequestProcessor(counterService));
    rpcServer.registerProcessor(new IncrementAndGetRequestProcessor(counterService));
    // 初始化状态机
    this.fsm = new CounterStateMachine();
    // 设置状态机到启动参数
    nodeOptions.setFsm(this.fsm);
    // 设置存储路径
    // 日志, 必须
    nodeOptions.setLogUri(dataPath + File.separator + "log");
    // 元信息, 必须
    nodeOptions.setRaftMetaUri(dataPath + File.separator + "raft_meta");
    // snapshot, 可选, 一般都推荐
    nodeOptions.setSnapshotUri(dataPath + File.separator + "snapshot");
    // 初始化 raft group 服务框架
    this.raftGroupService = new RaftGroupService(groupId, serverId, nodeOptions, rpcServer);
    // 启动
    this.node = this.raftGroupService.start();
}
```

# 启动

com.alipay.sofa.jraft.RaftGroupService#start()

```java
public synchronized Node start() {
    return start(true);
}
```

```java
public synchronized Node start(final boolean startRpcServer) {
    if (this.started) {
        return this.node;
    }
    if (this.serverId == null || this.serverId.getEndpoint() == null
        || this.serverId.getEndpoint().equals(new Endpoint(Utils.IP_ANY, 0))) {
        throw new IllegalArgumentException("Blank serverId:" + this.serverId);
    }
    if (StringUtils.isBlank(this.groupId)) {
        throw new IllegalArgumentException("Blank group id" + this.groupId);
    }
    //Adds RPC server to Server.
    NodeManager.getInstance().addAddress(this.serverId.getEndpoint());

    //创建并启动NodeImpl
    this.node = RaftServiceFactory.createAndInitRaftNode(this.groupId, this.serverId, this.nodeOptions);
    if (startRpcServer) {
        this.rpcServer.init(null); //启动RPCserver
    } else {
        LOG.warn("RPC server is not started in RaftGroupService.");
    }
    this.started = true;
    LOG.info("Start the RaftGroupService successfully.");
    return this.node;
}
```

```java
public static Node createAndInitRaftNode(final String groupId, final PeerId serverId, final NodeOptions opts) {
    final Node ret = createRaftNode(groupId, serverId);
    if (!ret.init(opts)) {
        throw new IllegalStateException("Fail to init node, please see the logs to find the reason.");
    }
    return ret;
}
```

```java
public static Node createRaftNode(final String groupId, final PeerId serverId) {
    return new NodeImpl(groupId, serverId);
}
```

```java
public NodeImpl(final String groupId, final PeerId serverId) {
    super();
    if (groupId != null) {
        Utils.verifyGroupId(groupId);
    }
    this.groupId = groupId;
    this.serverId = serverId != null ? serverId.copy() : null;
    this.state = State.STATE_UNINITIALIZED; //尚未初始化
    this.currTerm = 0;
    updateLastLeaderTimestamp(Utils.monotonicMs());
    this.confCtx = new ConfigurationCtx(this);
    this.wakingCandidate = null;
    final int num = GLOBAL_NUM_NODES.incrementAndGet();
    LOG.info("The number of active nodes increment to {}.", num);
}
```

```java
public boolean init(final NodeOptions opts) {
    Requires.requireNonNull(opts, "Null node options");
    Requires.requireNonNull(opts.getRaftOptions(), "Null raft options");
    Requires.requireNonNull(opts.getServiceFactory(), "Null jraft service factory");
    this.serviceFactory = opts.getServiceFactory();
    this.options = opts;
    this.raftOptions = opts.getRaftOptions();
    this.metrics = new NodeMetrics(opts.isEnableMetrics());
    this.serverId.setPriority(opts.getElectionPriority());
    this.electionTimeoutCounter = 0;

    if (this.serverId.getIp().equals(Utils.IP_ANY)) {
        LOG.error("Node can't started from IP_ANY.");
        return false;
    }

    if (!NodeManager.getInstance().serverExists(this.serverId.getEndpoint())) {
        LOG.error("No RPC server attached to, did you forget to call addService?");
        return false;
    }

    this.timerManager = TIMER_FACTORY.getRaftScheduler(this.options.isSharedTimerPool(),
        this.options.getTimerPoolSize(), "JRaft-Node-ScheduleThreadPool");

    // Init timers
    final String suffix = getNodeId().toString();
    String name = "JRaft-VoteTimer-" + suffix;
    this.voteTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs(), TIMER_FACTORY.getVoteTimer(
        this.options.isSharedVoteTimer(), name)) {

        @Override
        protected void onTrigger() {
            handleVoteTimeout();
        }

        @Override
        protected int adjustTimeout(final int timeoutMs) {
            return randomTimeout(timeoutMs);
        }
    };
    name = "JRaft-ElectionTimer-" + suffix;
    this.electionTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs(),
        TIMER_FACTORY.getElectionTimer(this.options.isSharedElectionTimer(), name)) {

        @Override
        protected void onTrigger() {
            handleElectionTimeout();
        }

        @Override
        protected int adjustTimeout(final int timeoutMs) {
            return randomTimeout(timeoutMs);
        }
    };
    name = "JRaft-StepDownTimer-" + suffix;
    this.stepDownTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs() >> 1,
        TIMER_FACTORY.getStepDownTimer(this.options.isSharedStepDownTimer(), name)) {

        @Override
        protected void onTrigger() {
            handleStepDownTimeout();
        }
    };
    name = "JRaft-SnapshotTimer-" + suffix;
    this.snapshotTimer = new RepeatedTimer(name, this.options.getSnapshotIntervalSecs() * 1000,
        TIMER_FACTORY.getSnapshotTimer(this.options.isSharedSnapshotTimer(), name)) {

        private volatile boolean firstSchedule = true;

        @Override
        protected void onTrigger() {
            handleSnapshotTimeout();
        }

        @Override
        protected int adjustTimeout(final int timeoutMs) {
            if (!this.firstSchedule) {
                return timeoutMs;
            }

            // Randomize the first snapshot trigger timeout
            this.firstSchedule = false;
            if (timeoutMs > 0) {
                int half = timeoutMs / 2;
                return half + ThreadLocalRandom.current().nextInt(half);
            } else {
                return timeoutMs;
            }
        }
    };

    this.configManager = new ConfigurationManager();

    this.applyDisruptor = DisruptorBuilder.<LogEntryAndClosure> newInstance() //
        .setRingBufferSize(this.raftOptions.getDisruptorBufferSize()) //
        .setEventFactory(new LogEntryAndClosureFactory()) //
        .setThreadFactory(new NamedThreadFactory("JRaft-NodeImpl-Disruptor-", true)) //
        .setProducerType(ProducerType.MULTI) //
        .setWaitStrategy(new BlockingWaitStrategy()) //
        .build();
    this.applyDisruptor.handleEventsWith(new LogEntryAndClosureHandler());
    this.applyDisruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
    this.applyQueue = this.applyDisruptor.start();
    if (this.metrics.getMetricRegistry() != null) {
        this.metrics.getMetricRegistry().register("jraft-node-impl-disruptor",
            new DisruptorMetricSet(this.applyQueue));
    }

    this.fsmCaller = new FSMCallerImpl();
    if (!initLogStorage()) {
        LOG.error("Node {} initLogStorage failed.", getNodeId());
        return false;
    }
    if (!initMetaStorage()) {
        LOG.error("Node {} initMetaStorage failed.", getNodeId());
        return false;
    }
    if (!initFSMCaller(new LogId(0, 0))) {
        LOG.error("Node {} initFSMCaller failed.", getNodeId());
        return false;
    }
    this.ballotBox = new BallotBox();
    final BallotBoxOptions ballotBoxOpts = new BallotBoxOptions();
    ballotBoxOpts.setWaiter(this.fsmCaller);
    ballotBoxOpts.setClosureQueue(this.closureQueue);
    if (!this.ballotBox.init(ballotBoxOpts)) {
        LOG.error("Node {} init ballotBox failed.", getNodeId());
        return false;
    }

    if (!initSnapshotStorage()) {
        LOG.error("Node {} initSnapshotStorage failed.", getNodeId());
        return false;
    }

    final Status st = this.logManager.checkConsistency();
    if (!st.isOk()) {
        LOG.error("Node {} is initialized with inconsistent log, status={}.", getNodeId(), st);
        return false;
    }
    this.conf = new ConfigurationEntry();
    this.conf.setId(new LogId());
    // if have log using conf in log, else using conf in options
    if (this.logManager.getLastLogIndex() > 0) {
        checkAndSetConfiguration(false);
    } else {
        this.conf.setConf(this.options.getInitialConf());
        // initially set to max(priority of all nodes)
        this.targetPriority = getMaxPriorityOfNodes(this.conf.getConf().getPeers());
    }

    if (!this.conf.isEmpty()) {
        Requires.requireTrue(this.conf.isValid(), "Invalid conf: %s", this.conf);
    } else {
        LOG.info("Init node {} with empty conf.", this.serverId);
    }

    // TODO RPC service and ReplicatorGroup is in cycle dependent, refactor it
    this.replicatorGroup = new ReplicatorGroupImpl();
    this.rpcService = new DefaultRaftClientService(this.replicatorGroup);
    final ReplicatorGroupOptions rgOpts = new ReplicatorGroupOptions();
    rgOpts.setHeartbeatTimeoutMs(heartbeatTimeout(this.options.getElectionTimeoutMs()));
    rgOpts.setElectionTimeoutMs(this.options.getElectionTimeoutMs());
    rgOpts.setLogManager(this.logManager);
    rgOpts.setBallotBox(this.ballotBox);
    rgOpts.setNode(this);
    rgOpts.setRaftRpcClientService(this.rpcService);
    rgOpts.setSnapshotStorage(this.snapshotExecutor != null ? this.snapshotExecutor.getSnapshotStorage() : null);
    rgOpts.setRaftOptions(this.raftOptions);
    rgOpts.setTimerManager(this.timerManager);

    // Adds metric registry to RPC service.
    this.options.setMetricRegistry(this.metrics.getMetricRegistry());

    if (!this.rpcService.init(this.options)) {
        LOG.error("Fail to init rpc service.");
        return false;
    }
    this.replicatorGroup.init(new NodeId(this.groupId, this.serverId), rgOpts);

    this.readOnlyService = new ReadOnlyServiceImpl();
    final ReadOnlyServiceOptions rosOpts = new ReadOnlyServiceOptions();
    rosOpts.setFsmCaller(this.fsmCaller);
    rosOpts.setNode(this);
    rosOpts.setRaftOptions(this.raftOptions);

    if (!this.readOnlyService.init(rosOpts)) {
        LOG.error("Fail to init readOnlyService.");
        return false;
    }

    // set state to follower
    this.state = State.STATE_FOLLOWER;

    if (LOG.isInfoEnabled()) {
        LOG.info("Node {} init, term={}, lastLogId={}, conf={}, oldConf={}.", getNodeId(), this.currTerm,
            this.logManager.getLastLogId(false), this.conf.getConf(), this.conf.getOldConf());
    }

    if (this.snapshotExecutor != null && this.options.getSnapshotIntervalSecs() > 0) {
        LOG.debug("Node {} start snapshot timer, term={}.", getNodeId(), this.currTerm);
        this.snapshotTimer.start();
    }

    if (!this.conf.isEmpty()) {
        stepDown(this.currTerm, false, new Status());
    }

    if (!NodeManager.getInstance().add(this)) {
        LOG.error("NodeManager add {} failed.", getNodeId());
        return false;
    }

    // Now the raft node is started , have to acquire the writeLock to avoid race
    // conditions
    this.writeLock.lock();
    if (this.conf.isStable() && this.conf.getConf().size() == 1 && this.conf.getConf().contains(this.serverId)) {
        // The group contains only this server which must be the LEADER, trigger
        // the timer immediately.
        electSelf();
    } else {
        this.writeLock.unlock();
    }

    return true;
}
```

# Leader选举

## 预投票

### 选举超时定时器

```java
this.electionTimer = new RepeatedTimer("JRaft-ElectionTimer", this.options.getElectionTimeoutMs()) {
    @Override
    protected void onTrigger() {
        handleElectionTimeout(); 
    }
    @Override
    protected int adjustTimeout(int timeoutMs) {
        return randomTimeout(timeoutMs);
    }
}; //选举超时定时器
```

### 处理选举超时

```java
private void handleElectionTimeout() {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
      //启动之初默认为STATE_FOLLOWER
      if (this.state != State.STATE_FOLLOWER) {
        return;
      }
      //Leader租约仍然有效，距离上次选举还未超过electionTimeoutMs，默认1秒
      if (isCurrentLeaderValid()) {
        return;
      }
      resetLeaderId(PeerId.emptyPeer(), new Status(RaftError.ERAFTTIMEDOUT, "Lost connection from leader %s.",
                                                   this.leaderId));
 			//判断是否允许运行选举
      if (!allowLaunchElection()) {
        return;
      }
			//不释放锁，在预投票方法中释放锁
      doUnlock = false;
      preVote();//预投票

    } finally {
      if (doUnlock) {
        this.writeLock.unlock();
      }
    }
}
```

#### 判断是否允许选举

com.alipay.sofa.jraft.core.NodeImpl#allowLaunchElection

```java
private boolean allowLaunchElection() {//是否允许选举

    //priority=0,不参与选举
    if (this.serverId.isPriorityNotElected()) { 
        return false;
    }

  	//priority<=-1,可以参加选举
    if (this.serverId.isPriorityDisabled()) { 
        return true;
    }

    // If current node's priority < target_priority, it does not initiate leader,
    // election and waits for the next election timeout.
    if (this.serverId.getPriority() < this.targetPriority) {
        this.electionTimeoutCounter++;

        // If next leader is not elected until next election timeout, it
        // decays its local target priority exponentially.
        if (this.electionTimeoutCounter > 1) {
            decayTargetPriority();
            this.electionTimeoutCounter = 0;
        }

        if (this.electionTimeoutCounter == 1) {
            return false;
        }
    }

    return this.serverId.getPriority() >= this.targetPriority;
}
```

#### 预投票

com.alipay.sofa.jraft.core.NodeImpl#preVote

```java
private void preVote() {
    long oldTerm;
    try {
        LOG.info("Node {} term {} start preVote", this.getNodeId(), this.currTerm);
       //接收到Leader发送的InstallSnapshotRequest请求，正在安装快照
        if (this.snapshotExecutor != null && snapshotExecutor.isInstallingSnapshot()) {
            LOG.warn(
                "Node {} term {} doesn't do preVote when installing snapshot as the configuration may be out of date.",
                getNodeId());
            return;
        }
        //当前节点的serverId不合法
        if (!this.conf.contains(this.serverId)) { 
            LOG.warn("Node {} can't do preVote as it is not in conf <{}>", this.getNodeId(), this.conf.getConf());
            return;
        }
        oldTerm = this.currTerm; //当前term
    } finally {
        writeLock.unlock();
    }
    //最新的LogId，包含了index、term
    final LogId lastLogId = this.logManager.getLastLogId(true);
    boolean doUnlock = true;
    writeLock.lock();
    try {
      	//term发生了变更
        if (oldTerm != currTerm) {
            LOG.warn("Node {} raise term {} when get lastLogId", this.getNodeId(), this.currTerm);
            return;
        }
        //设置quorum为集群节点个数的一半加1
        this.prevVoteCtx.init(conf.getConf(), conf.isStable() ? null : conf.getOldConf());
        //向集群内其他节点发送预投票请求
        for (final PeerId peer : this.conf.listPeers()) {
            if (peer.equals(this.serverId)) {
                continue;
            }
            //建立连接
            if (!this.rpcService.connect(peer.getEndpoint())) {
                LOG.warn("Node {} channel init failed, addr: {}", this.getNodeId(), peer.getEndpoint());
                continue;
            }
            //接收到响应之后，执行的回调方法
            final OnPreVoteRpcDone done = new OnPreVoteRpcDone(peer, currTerm);
            final RequestVoteRequest.Builder reqBuilder = RequestVoteRequest.newBuilder();
            reqBuilder.setPreVote(true); //表明预投票请求
            reqBuilder.setGroupId(this.groupId);
            reqBuilder.setServerId(this.serverId.toString());
            reqBuilder.setPeerId(peer.toString());
            reqBuilder.setTerm(currTerm + 1); //本节点的term不会增加
            reqBuilder.setLastLogIndex(lastLogId.getIndex()); //本机日志的最新index
            reqBuilder.setLastLogTerm(lastLogId.getTerm()); //本机日志最新的term
            done.request = reqBuilder.build();
            //发送请求
            rpcService.preVote(peer.getEndpoint(), done.request, done);
        }
        
        prevVoteCtx.grant(this.serverId);//当前节点的响应
        if (prevVoteCtx.isGranted()) { //判断是否收到一半节点的响应
            doUnlock = false;
            electSelf(); //投票
        }
    } finally {
        if (doUnlock) {
            writeLock.unlock();
        }
    }
}
```

com.alipay.sofa.jraft.entity.Ballot#isGranted

```java
public boolean isGranted() { //quorum原为集群节点个数的一半+1
    return this.quorum <= 0 && this.oldQuorum <= 0;
}
```

#### 回调

com.alipay.sofa.jraft.core.NodeImpl.OnPreVoteRpcDone#run

```java
public void run(final Status status) {
    NodeImpl.this.metrics.recordLatency("pre-vote", Utils.monotonicMs() - this.startMs);
    if (!status.isOk()) {
        LOG.warn("Node {} PreVote to {} error: {}.", getNodeId(), this.peer, status);
    } else {
        handlePreVoteResponse(this.peer, this.term, getResponse());
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl#handlePreVoteResponse

```java
public void handlePreVoteResponse(final PeerId peerId, final long term, final RequestVoteResponse response) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_FOLLOWER) { //节点状态发生了变更
            LOG.warn("Node {} received invalid PreVoteResponse from {}, state not in STATE_FOLLOWER but {}.",
                getNodeId(), peerId, this.state);
            return;
        }
        if (term != this.currTerm) { //当前节点的term发生了变更
            LOG.warn("Node {} received invalid PreVoteResponse from {}, term={}, currTerm={}.", getNodeId(),
                peerId, term, this.currTerm);
            return;
        }
        if (response.getTerm() > this.currTerm) { //响应的term大于本机term
            LOG.warn("Node {} received invalid PreVoteResponse from {}, term {}, expect={}.", getNodeId(), peerId,
                response.getTerm(), this.currTerm);
            stepDown(response.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                "Raft node receives higher term pre_vote_response."));
            return;
        }
      	//正常响应
        LOG.info("Node {} received PreVoteResponse from {}, term={}, granted={}.", getNodeId(), peerId,
            response.getTerm(), response.getGranted());
        // check granted quorum?
        if (response.getGranted()) {
            this.prevVoteCtx.grant(peerId);
            if (this.prevVoteCtx.isGranted()) {//判断是否收到一半节点的响应
                doUnlock = false;
                electSelf(); //选举
            }
        }
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

## 投票

com.alipay.sofa.jraft.core.NodeImpl#electSelf

```java
private void electSelf() {
    long oldTerm;
    try {
        LOG.info("Node {} term {} start vote and grant vote self", this.getNodeId(), this.currTerm);
        if (!this.conf.contains(this.serverId)) { //节点不合法
            LOG.warn("Node {} can't do electSelf as it is not in {}", this.getNodeId(), this.conf.getConf());
            return;
        }
       
        if (state == State.STATE_FOLLOWER) { //当前状态为STATE_FOLLOWER
            LOG.debug("Node {} term {} stop election timer", this.getNodeId(), this.currTerm);
            this.electionTimer.stop(); //停止electionTimer定时器
        }
        final PeerId emptyId = new PeerId();
        //重置LeaderId
        resetLeaderId(emptyId, new Status(RaftError.ERAFTTIMEDOUT,
                "A follower's leader_id is reset to NULL as it begins to request_vote."));
        //当前状态修改为STATE_CANDIDATE
        this.state = State.STATE_CANDIDATE;
        //增加term
        this.currTerm++;
        this.votedId = this.serverId.copy();
        LOG.debug("Node {} term {} start vote_timer", this.getNodeId(), this.currTerm);
        //开启投票超时定时器
        this.voteTimer.start();   
        //设置quorum为集群内节点的大多数
        this.voteCtx.init(this.conf.getConf(), this.conf.isStable() ? null : this.conf.getOldConf());
        oldTerm = this.currTerm;
    } finally {
        writeLock.unlock();
    }
    final LogId lastLogId = logManager.getLastLogId(true);
    writeLock.lock();
    try {	
        // vote need defense ABA after unlock&writeLock
        if (oldTerm != this.currTerm) {
            LOG.warn("Node {} raise term {} when getLastLogId.", getNodeId(), this.currTerm);
            return;
        }
        //向集群内各节点发送投票请求
        for (final PeerId peer : this.conf.listPeers()) {
            if (peer.equals(this.serverId)) {
                continue;
            }
            //建立连接
            if (!this.rpcService.connect(peer.getEndpoint())) {
                LOG.warn("Node {} channel init failed, addr: {}", this.getNodeId(), peer.getEndpoint());
                continue;
            }
            //回调
            final OnRequestVoteRpcDone done = new OnRequestVoteRpcDone(peer, this.currTerm, this);
            final RequestVoteRequest.Builder reqBuilder = RequestVoteRequest.newBuilder();
            reqBuilder.setPreVote(false); //It's not a pre-vote request.
            reqBuilder.setGroupId(groupId);
            reqBuilder.setServerId(this.serverId.toString());
            reqBuilder.setPeerId(peer.toString());
            reqBuilder.setTerm(currTerm);
            reqBuilder.setLastLogIndex(lastLogId.getIndex());
            reqBuilder.setLastLogTerm(lastLogId.getTerm());
            done.request = reqBuilder.build();
            //发送请求
            rpcService.requestVote(peer.getEndpoint(), done.request, done);
        }
        this.metaStorage.setTermAndVotedFor(this.currTerm, serverId); //本机投票信息
        voteCtx.grant(serverId);//响应当前节点
        if (voteCtx.isGranted()) { //超过大多数
            becomeLeader(); //成为Leader
        }
    } finally {
        writeLock.unlock();
    }
}
```

### 投票定时器

```java
this.voteTimer = new RepeatedTimer(name, this.options.getElectionTimeoutMs(), TIMER_FACTORY.getVoteTimer(
    this.options.isSharedVoteTimer(), name)) {

    @Override
    protected void onTrigger() {
      	//处于STATE_CANDIDATE状态，启动此定时器，投票请求超时
        handleVoteTimeout();
    }

    @Override
    protected int adjustTimeout(final int timeoutMs) {
        return randomTimeout(timeoutMs);
    }
};
```

com.alipay.sofa.jraft.core.NodeImpl#handleVoteTimeout

```java
private void handleVoteTimeout() {
    this.writeLock.lock();
    if (this.state != State.STATE_CANDIDATE) {
        this.writeLock.unlock();
        return;
    }

    if (this.raftOptions.isStepDownWhenVoteTimedout()) {//默认true
        LOG.warn(
            "Candidate node {} term {} steps down when election reaching vote timeout: fail to get quorum vote-granted.",
            this.nodeId, this.currTerm);
        stepDown(this.currTerm, false, new Status(RaftError.ETIMEDOUT,
            "Vote timeout: fail to get quorum vote-granted."));
        // unlock in preVote
        preVote(); //预投票
    } else {
        LOG.debug("Node {} term {} retry to vote self.", getNodeId(), this.currTerm);
        // unlock in electSelf
        electSelf(); //投票
    }
}
```

### 投票响应回调

com.alipay.sofa.jraft.core.NodeImpl.OnRequestVoteRpcDone#run

```java
public void run(final Status status) {
    NodeImpl.this.metrics.recordLatency("request-vote", Utils.monotonicMs() - this.startMs);
    if (!status.isOk()) {
        LOG.warn("Node {} RequestVote to {} error: {}.", this.node.getNodeId(), this.peer, status);
    } else {
        this.node.handleRequestVoteResponse(this.peer, this.term, getResponse());
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl#handleRequestVoteResponse

```java
public void handleRequestVoteResponse(final PeerId peerId, final long term, final RequestVoteResponse response) {
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_CANDIDATE) { //当前状态不再是STATE_CANDIDATE，直接返回
            LOG.warn("Node {} received invalid RequestVoteResponse from {}, state not in STATE_CANDIDATE but {}.",
                getNodeId(), peerId, this.state);
            return;
        }
        // check stale term
        if (term != this.currTerm) { //term发生了变更
            LOG.warn("Node {} received stale RequestVoteResponse from {}, term={}, currTerm={}.", getNodeId(),
                peerId, term, this.currTerm);
            return;
        }
        // check response term
        if (response.getTerm() > this.currTerm) { //其他节点的term大于本机的term
            LOG.warn("Node {} received invalid RequestVoteResponse from {}, term={}, expect={}.", getNodeId(),
                peerId, response.getTerm(), this.currTerm);
            stepDown(response.getTerm(), false, new Status(RaftError.EHIGHERTERMRESPONSE,
                "Raft node receives higher term request_vote_response."));
            return;
        }
        // check granted quorum?
        if (response.getGranted()) { //是否投票给本机
            this.voteCtx.grant(peerId); 
            if (this.voteCtx.isGranted()) { //超一半节点投票给本机
                becomeLeader(); //当选leader
            }
        }
    } finally {
        this.writeLock.unlock();
    }
}
```

## 当选Leader

com.alipay.sofa.jraft.core.NodeImpl#becomeLeader

```java
private void becomeLeader() {
    Requires.requireTrue(this.state == State.STATE_CANDIDATE, "Illegal state: " + this.state);
    LOG.info("Node {} become leader of group, term={}, conf={}, oldConf={}.", getNodeId(), this.currTerm,
        this.conf.getConf(), this.conf.getOldConf());
    // cancel candidate vote timer
    stopVoteTimer(); //关闭投票定时器
    this.state = State.STATE_LEADER; //修改为Leader
    this.leaderId = this.serverId.copy();
    this.replicatorGroup.resetTerm(this.currTerm); //设置当前的term
    // Start follower's replicators
    for (final PeerId peer : this.conf.listPeers()) { //启动Follower的复制器
        if (peer.equals(this.serverId)) {
            continue;
        }
        LOG.debug("Node {} add a replicator, term={}, peer={}.", getNodeId(), this.currTerm, peer);
        if (!this.replicatorGroup.addReplicator(peer)) {
            LOG.error("Fail to add a replicator, peer={}.", peer);
        }
    }

    // Start learner's replicators
    for (final PeerId peer : this.conf.listLearners()) {//启动learner的复制器
        LOG.debug("Node {} add a learner replicator, term={}, peer={}.", getNodeId(), this.currTerm, peer);
        if (!this.replicatorGroup.addReplicator(peer, ReplicatorType.Learner)) {
            LOG.error("Fail to add a learner replicator, peer={}.", peer);
        }
    }

    // init commit manager
    this.ballotBox.resetPendingIndex(this.logManager.getLastLogIndex() + 1);
    // Register _conf_ctx to reject configuration changing before the first log
    // is committed.
    if (this.confCtx.isBusy()) {
        throw new IllegalStateException();
    }
    //追加一条最新的日志，解决’幽灵复现‘
    this.confCtx.flush(this.conf.getConf(), this.conf.getOldConf()); 
    this.stepDownTimer.start();
}
```

# 数据同步

Leader选举成功之后，开始向集群内的节点同步数据

Leader负责维护各个节点的Replicator

## 创建Replicator

com.alipay.sofa.jraft.core.ReplicatorGroupImpl#addReplicator

```java
public boolean addReplicator(PeerId peer) {
    Requires.requireTrue(commonOptions.getTerm() != 0);
    if (replicatorMap.get(peer) != null) {
        this.failureReplicators.remove(peer);
        return true;
    }
    final ReplicatorOptions options = commonOptions.copy();
    options.setPeerId(peer);
  	//创建Replicator
    final ThreadId rid = Replicator.start(options, this.raftOptions);
    if (rid == null) {
        LOG.error("Fail to start replicator to peer={}", peer);
        this.failureReplicators.add(peer);
        return false;
    }
    return this.replicatorMap.put(peer, rid) == null;
}
```

com.alipay.sofa.jraft.core.Replicator#start

```java
public static ThreadId start(final ReplicatorOptions opts, final RaftOptions raftOptions) {
    if (opts.getLogManager() == null || opts.getBallotBox() == null || opts.getNode() == null) {
        throw new IllegalArgumentException("Invalid ReplicatorOptions.");
    }
    //创建Replicator
    final Replicator r = new Replicator(opts, raftOptions);
    //建立连接
    if (!r.rpcService.connect(opts.getPeerId().getEndpoint())) {
        LOG.error("Fail to init sending channel to {}.", opts.getPeerId());
        // Return and it will be retried later.
        return null;
    }
    // Register replicator metric set.
    final MetricRegistry metricRegistry = opts.getNode().getNodeMetrics().getMetricRegistry();
    if (metricRegistry != null) {
        try {
            final String replicatorMetricName = getReplicatorMetricName(opts);
            if (!metricRegistry.getNames().contains(replicatorMetricName)) {
                metricRegistry.register(replicatorMetricName, new ReplicatorMetricSet(opts, r));
            }
        } catch (final IllegalArgumentException e) {
            // ignore
        }
    }
    // Start replication
    r.id = new ThreadId(r, r);
    r.id.lock();
    notifyReplicatorStatusListener(r, ReplicatorEvent.CREATED);
    LOG.info("Replicator={}@{} is started", r.id, r.options.getPeerId());
    r.catchUpClosure = null;
    r.lastRpcSendTimestamp = Utils.monotonicMs();
    //心跳超时检测
    r.startHeartbeatTimer(Utils.nowMs());
    // id.unlock in sendEmptyEntries
    //发送prob请求
    r.sendEmptyEntries(false);
    return r.id;
}
```

## 心跳超时检测

com.alipay.sofa.jraft.core.Replicator#startHeartbeatTimer

```java
private void startHeartbeatTimer(final long startMs) {
    final long dueTime = startMs + this.options.getDynamicHeartBeatTimeoutMs();
    try {
        this.heartbeatTimer = this.timerManager.schedule(() -> onTimeout(this.id), dueTime - Utils.nowMs(),
            TimeUnit.MILLISECONDS);
    } catch (final Exception e) {
        LOG.error("Fail to schedule heartbeat timer", e);
        onTimeout(this.id);
    }
}
```



```java
private static void onTimeout(final ThreadId id) {
    if (id != null) {
      //设置心跳超时标志
        id.setError(RaftError.ETIMEDOUT.getNumber());
    } else {
        LOG.warn("Replicator id is null when timeout, maybe it's destroyed.");
    }
}
```

## 发送空请求

com.alipay.sofa.jraft.core.Replicator#sendEmptyEntries

```java
private void sendEmptyEntries(final boolean isHeartbeat,
                              final RpcResponseClosure<AppendEntriesResponse> heartBeatClosure) {//发送Prob或者心跳请求
    //构建请求
    final AppendEntriesRequest.Builder rb = AppendEntriesRequest.newBuilder();
    //填充请求
    if (!fillCommonFields(rb, this.nextIndex - 1, isHeartbeat)) {
        installSnapshot(); //发送安装快照的请求
        if (isHeartbeat && heartBeatClosure != null) {
            Utils.runClosureInThread(heartBeatClosure, new Status(RaftError.EAGAIN,
                "Fail to send heartbeat to peer %s", this.options.getPeerId()));
        }
        return;
    }
    try {
        final long monotonicSendTimeMs = Utils.monotonicMs();
        final AppendEntriesRequest request = rb.build();
        if (isHeartbeat) { //心跳请求
            // Sending a heartbeat request
            this.heartbeatCounter++;
            RpcResponseClosure<AppendEntriesResponse> heartbeatDone;
            // Prefer passed-in closure.
            if (heartBeatClosure != null) {
                heartbeatDone = heartBeatClosure;
            } else {
                heartbeatDone = new RpcResponseClosureAdapter<AppendEntriesResponse>() {
                    @Override
                    public void run(final Status status) {
                        onHeartbeatReturned(Replicator.this.id, status, request, getResponse(), monotonicSendTimeMs);
                    }
                }; //回调
            }
            //发送心跳请求
            this.heartbeatInFly = this.rpcService.appendEntries(this.options.getPeerId().getEndpoint(), request,
                this.options.getElectionTimeoutMs() / 2, heartbeatDone);
        } else { //prob请求
            this.statInfo.runningState = RunningState.APPENDING_ENTRIES;
            this.statInfo.firstLogIndex = this.nextIndex;
            this.statInfo.lastLogIndex = this.nextIndex - 1;
            this.appendEntriesCounter++;
            this.state = State.Probe; //Probe
            final int stateVersion = this.version;
            final int seq = getAndIncrementReqSeq();
            final Future<Message> rpcFuture = this.rpcService.appendEntries(this.options.getPeerId().getEndpoint(),
                request, -1, new RpcResponseClosureAdapter<AppendEntriesResponse>() {
                    @Override
                    public void run(final Status status) {
                        onRpcReturned(Replicator.this.id, RequestType.AppendEntries, status, request,getResponse(), seq, stateVersion, monotonicSendTimeMs);
                    }
                }); //发送Probe请求
            //加入inflights双端队列，保存已经发送但是尚未接收响应的请求
            addInflight(RequestType.AppendEntries, this.nextIndex, 0, 0, seq, rpcFuture);
        }
    } finally {
        this.id.unlock();
    }
}
```

### 填充请求

com.alipay.sofa.jraft.core.Replicator#fillCommonFields

```java
private boolean fillCommonFields(final AppendEntriesRequest.Builder rb, long prevLogIndex, final boolean isHeartbeat) {
    //前一条日志项的term
    final long prevLogTerm = this.options.getLogManager().getTerm(prevLogIndex);
    if (prevLogTerm == 0 && prevLogIndex != 0) {
        if (!isHeartbeat) { //不是心跳请求
            Requires.requireTrue(prevLogIndex < this.options.getLogManager().getFirstLogIndex());
            LOG.debug("logIndex={} was compacted", prevLogIndex);
            return false;
        } else { //心跳请求
            prevLogIndex = 0;
        }
    }
    //填充请求
    rb.setTerm(this.options.getTerm());
    rb.setGroupId(this.options.getGroupId());
    rb.setServerId(this.options.getServerId().toString());
    rb.setPeerId(this.options.getPeerId().toString());
    //前一条日志项的index、term
    rb.setPrevLogIndex(prevLogIndex);
    rb.setPrevLogTerm(prevLogTerm);
    //Leader节点已经提交的index
    rb.setCommittedIndex(this.options.getBallotBox().getLastCommittedIndex());
    return true;
}
```

com.alipay.sofa.jraft.core.Replicator#addInflight

```java
private void addInflight(final RequestType reqType, final long startIndex, final int count, final int size,
                         final int seq, final Future<Message> rpcInfly) {
    //最新发送的请求
    this.rpcInFly = new Inflight(reqType, startIndex, count, size, seq, rpcInfly);
    this.inflights.add(this.rpcInFly);//暂存已经发送但是尚未接收响应的请求
    this.nodeMetrics.recordSize("replicate-inflights-count", this.inflights.size());
}
```

### 回调方法

发送快照、prob请求、同步请求之后

com.alipay.sofa.jraft.core.Replicator#onRpcReturned

```java
static void onRpcReturned(final ThreadId id, final RequestType reqType, final Status status, final Message request,final Message response, final int seq, final int stateVersion, final long rpcSendTime) {
    if (id == null) {
        return;
    }
    final long startTimeMs = Utils.nowMs();
    Replicator r;
    if ((r = (Replicator) id.lock()) == null) {
        return;
    }
    if (stateVersion != r.version) {
        id.unlock();
        return;
    }
    //存放接收的响应，根据seq进行排序
    final PriorityQueue<RpcResponse> holdingQueue = r.pendingResponses;
    holdingQueue.add(new RpcResponse(reqType, seq, status, request, response, rpcSendTime));
    //积压请求太多
    if (holdingQueue.size() > r.raftOptions.getMaxReplicatorInflightMsgs()) {
        LOG.warn("Too many pending responses {} for replicator {}, maxReplicatorInflightMsgs={}",
            holdingQueue.size(), r.options.getPeerId(), r.raftOptions.getMaxReplicatorInflightMsgs());
        r.resetInflights(); //清空inflights
        r.state = State.Probe; //发送Probe请求
        r.sendEmptyEntries(false);
        return;
    }
    //是否继续发送
    boolean continueSendEntries = false;
    final boolean isLogDebugEnabled = LOG.isDebugEnabled();
    StringBuilder sb = null;
    if (isLogDebugEnabled) {
        sb = new StringBuilder("Replicator ").append(r).append(" is processing RPC responses,");
    }
    try {
        int processed = 0;
        while (!holdingQueue.isEmpty()) {
            final RpcResponse queuedPipelinedResponse = holdingQueue.peek();
            // Sequence 乱序,等待下一个响应
            if (queuedPipelinedResponse.seq != r.requiredNextSeq) {
                if (processed > 0) { //本轮有响应可以处理
                    if (isLogDebugEnabled) {
                        sb.append("has processed ").append(processed).append(" responses,");
                    }
                    break;
                } else { //没有处理任何响应，不再继续发送数据
                    continueSendEntries = false;
                    id.unlock();
                    return;
                }
            }
            //将响应从holdingQueue移除
            holdingQueue.remove();
            processed++;
            final Inflight inflight = r.pollInflight();
            if (inflight == null) {//已从inflights移除
                if (isLogDebugEnabled) {
                    sb.append("ignore response because request not found:").append(queuedPipelinedResponse)
                        .append(",\n");
                }
                continue;
            }
            //出现乱序
            if (inflight.seq != queuedPipelinedResponse.seq) {
                LOG.warn(
                    "Replicator {} response sequence out of order, expect {}, but it is {}, reset state to try again.",
                    r, inflight.seq, queuedPipelinedResponse.seq);
                r.resetInflights();
                //设置Prob状态
                r.state = State.Probe;
                continueSendEntries = false; //不再继续发送
                r.block(Utils.nowMs(), RaftError.EREQUEST.getNumber());
                return;
            }
            try {
                switch (queuedPipelinedResponse.requestType) {
                    case AppendEntries: //追加数据
                        continueSendEntries = onAppendEntriesReturned(id, inflight, queuedPipelinedResponse.status,
                            (AppendEntriesRequest) queuedPipelinedResponse.request,
                            (AppendEntriesResponse) queuedPipelinedResponse.response, rpcSendTime, startTimeMs, r);
                        break;
                    case Snapshot: //快照
                        continueSendEntries = onInstallSnapshotReturned(id, r, queuedPipelinedResponse.status,
                            (InstallSnapshotRequest) queuedPipelinedResponse.request,
                            (InstallSnapshotResponse) queuedPipelinedResponse.response);
                        break;
                }
            } finally {
                if (continueSendEntries) {
                    r.getAndIncrementRequiredNextSeq();
                } else {
                    break;
                }
            }
        }
    } finally {
        if (isLogDebugEnabled) {
            sb.append(", after processed, continue to send entries: ").append(continueSendEntries);
            LOG.debug(sb.toString());
        }
        if (continueSendEntries) {
            // unlock in sendEntries.
            r.sendEntries(); //继续发送消息
        }
    }
}
```

### 追加消息的回调

com.alipay.sofa.jraft.core.Replicator#onAppendEntriesReturned

```java
private static boolean onAppendEntriesReturned(final ThreadId id, final Inflight inflight, final Status status,
                                               final AppendEntriesRequest request,
                                               final AppendEntriesResponse response, final long rpcSendTime,
                                               final long startTimeMs, final Replicator r) {
   
    if (inflight.startIndex != request.getPrevLogIndex() + 1) {//不合法
        r.resetInflights(); //重置inflights
        r.state = State.Probe; //发送Probe请求
        r.sendEmptyEntries(false);
        return false;
    }
    if (request.getEntriesCount() > 0) {
        r.nodeMetrics.recordLatency("replicate-entries", Utils.monotonicMs() - rpcSendTime);
        r.nodeMetrics.recordSize("replicate-entries-count", request.getEntriesCount());
        r.nodeMetrics.recordSize("replicate-entries-bytes", request.getData() != null ? request.getData().size()
            : 0);
    }
    final boolean isLogDebugEnabled = LOG.isDebugEnabled();
    StringBuilder sb = null;
    if (isLogDebugEnabled) {
        sb = new StringBuilder("Node "). //
            append(r.options.getGroupId()).append(":").append(r.options.getServerId()). //
            append(" received AppendEntriesResponse from "). //
            append(r.options.getPeerId()). //
            append(" prevLogIndex=").append(request.getPrevLogIndex()). //
            append(" prevLogTerm=").append(request.getPrevLogTerm()). //
            append(" count=").append(request.getEntriesCount());
    }
    if (!status.isOk()) { //响应信息异常
        if (isLogDebugEnabled) {
            sb.append(" fail, sleep.");
            LOG.debug(sb.toString());
        }
        notifyReplicatorStatusListener(r, ReplicatorEvent.ERROR, status);
        if (++r.consecutiveErrorTimes % 10 == 0) {
            LOG.warn("Fail to issue RPC to {}, consecutiveErrorTimes={}, error={}", r.options.getPeerId(),
                r.consecutiveErrorTimes, status);
        }
       //发送Probe请求
        r.resetInflights();
        r.state = State.Probe;
        r.block(startTimeMs, status.getCode());
        return false;
    }
    r.consecutiveErrorTimes = 0;
    if (!response.getSuccess()) {
        if (response.getTerm() > r.options.getTerm()) {//Follower的term比leader的term高
            if (isLogDebugEnabled) {
                sb.append(" fail, greater term ").append(response.getTerm()).append(" expect term ").append(r.options.getTerm());
                LOG.debug(sb.toString());
            }
            final NodeImpl node = r.options.getNode();
            r.notifyOnCaughtUp(RaftError.EPERM.getNumber(), true);
            r.destroy();
            node.increaseTermTo(response.getTerm(), new Status(RaftError.EHIGHERTERMRESPONSE,
                "Leader receives higher term heartbeat_response from peer:%s", r.options.getPeerId()));
            return false;
        }
        if (isLogDebugEnabled) {
            sb.append(" fail, find nextIndex remote lastLogIndex ").append(response.getLastLogIndex())
                .append(" local nextIndex ").append(r.nextIndex);
            LOG.debug(sb.toString());
        }
        if (rpcSendTime > r.lastRpcSendTimestamp) {
            r.lastRpcSendTimestamp = rpcSendTime;
        }
        r.resetInflights();
        if (response.getLastLogIndex() + 1 < r.nextIndex) {
            LOG.debug("LastLogIndex at peer={} is {}", r.options.getPeerId(), response.getLastLogIndex());
            r.nextIndex = response.getLastLogIndex() + 1;
        } else {
            if (r.nextIndex > 1) {
                LOG.debug("logIndex={} dismatch", r.nextIndex);
                r.nextIndex--;
            } else {
                LOG.error("Peer={} declares that log at index=0 doesn't match, which is not supposed to happen",
                    r.options.getPeerId());
            }
        }
        r.sendEmptyEntries(false);
        return false;
    }
    if (isLogDebugEnabled) {
        sb.append(", success");
        LOG.debug(sb.toString());
    }
    // 响应成功
    if (response.getTerm() != r.options.getTerm()) { //term不匹配
        r.resetInflights();
        r.state = State.Probe;
        LOG.error("Fail, response term {} dismatch, expect term {}", response.getTerm(), r.options.getTerm());
        id.unlock();
        return false;
    }
    if (rpcSendTime > r.lastRpcSendTimestamp) {
        r.lastRpcSendTimestamp = rpcSendTime; //最新发送的时间戳
    }
    //区分追加请求的类型，有数据为同步请求，无数据为prob请求
    final int entriesSize = request.getEntriesCount();
    if (entriesSize > 0) { //同步请求
        if (r.options.getReplicatorType().isFollower()) { //节点类型为Follower，提交index
            r.options.getBallotBox().commitAt(r.nextIndex, r.nextIndex + entriesSize - 1, r.options.getPeerId());
        }
    } else {//表明是prob请求，将状态修改为Replicate
        r.state = State.Replicate;
    }
    //下次同步的index
    r.nextIndex += entriesSize;
    r.hasSucceeded = true;
    r.notifyOnCaughtUp(RaftError.SUCCESS.getNumber(), false);
    if (r.timeoutNowIndex > 0 && r.timeoutNowIndex < r.nextIndex) {
        r.sendTimeoutNow(false, false);
    }
    return true;
}
```

## 发送消息

com.alipay.sofa.jraft.core.Replicator#sendEntries()

```java
void sendEntries() { //发送尽可能多的消息
    boolean doUnlock = true;
    try {
        long prevSendIndex = -1;
        while (true) {
            //获取下一条发送的消息的索引，异常情况返回-1
            final long nextSendingIndex = getNextSendIndex();
            if (nextSendingIndex > prevSendIndex) {
                if (sendEntries(nextSendingIndex)) {//发送消息
                    prevSendIndex = nextSendingIndex;
                } else {
                    doUnlock = false;
                    // id already unlock in sendEntries when it returns false.
                    break;
                }
            } else {
                break;
            }
        }
    } finally {
        if (doUnlock) {
            this.id.unlock();
        }
    }
}
```

### 获取发送的索引

com.alipay.sofa.jraft.core.Replicator#getNextSendIndex

```java
long getNextSendIndex() { //获取下一条发送的消息的索引
    // Fast path
    if (this.inflights.isEmpty()) {
        return this.nextIndex;
    }
    // 积压太多尚未响应的请求
    if (this.inflights.size() > this.raftOptions.getMaxReplicatorInflightMsgs()) {
        return -1L;
    }
    // 最新发送的请求不为空，并且是追加请求类型，发送的消息数不能为0
    if (this.rpcInFly != null && this.rpcInFly.isSendingLogEntries()) {
        return this.rpcInFly.startIndex + this.rpcInFly.count;
    }
    return -1L;
}
```

### 复制数据

com.alipay.sofa.jraft.core.Replicator#sendEntries(long)

```java
private boolean sendEntries(final long nextSendingIndex) {
    final AppendEntriesRequest.Builder rb = AppendEntriesRequest.newBuilder();
    if (!fillCommonFields(rb, nextSendingIndex - 1, false)) { //填充属性信息
        // unlock id in installSnapshot
        installSnapshot(); //发送安装快照的请求
        return false;
    }

    ByteBufferCollector dataBuf = null;
    //默认一次最多发送1024条消息
    final int maxEntriesSize = this.raftOptions.getMaxEntriesSize();
    final RecyclableByteBufferList byteBufList = RecyclableByteBufferList.newInstance();
    try {
        for (int i = 0; i < maxEntriesSize; i++) {
            final RaftOutter.EntryMeta.Builder emb = RaftOutter.EntryMeta.newBuilder();
            //获取日志消息
            if (!prepareEntry(nextSendingIndex, i, emb, byteBufList)) {
                break;
            }
            rb.addEntries(emb.build());
        }
        if (rb.getEntriesCount() == 0) { //无新增消息
            if (nextSendingIndex < this.options.getLogManager().getFirstLogIndex()) {
                installSnapshot(); //日志发生了合并，发送快照
                return false;
            }
            // _id is unlock in _wait_more
            waitMoreEntries(nextSendingIndex); //等待更多的消息
            return false;
        }
        if (byteBufList.getCapacity() > 0) {
            dataBuf = ByteBufferCollector.allocateByRecyclers(byteBufList.getCapacity());
            for (final ByteBuffer b : byteBufList) {
                dataBuf.put(b);
            }
            final ByteBuffer buf = dataBuf.getBuffer();
            buf.flip();
            rb.setData(ZeroByteStringHelper.wrap(buf));
        }
    } finally {
        RecycleUtil.recycle(byteBufList);
    }
		//创建追加请求
    final AppendEntriesRequest request = rb.build();
    if (LOG.isDebugEnabled()) {
        LOG.debug(
            "Node {} send AppendEntriesRequest to {} term {} lastCommittedIndex {} prevLogIndex {} prevLogTerm {} logIndex {} count {}",
            this.options.getNode().getNodeId(), this.options.getPeerId(), this.options.getTerm(),
            request.getCommittedIndex(), request.getPrevLogIndex(), request.getPrevLogTerm(), nextSendingIndex,
            request.getEntriesCount());
    }
    this.statInfo.runningState = RunningState.APPENDING_ENTRIES;
    this.statInfo.firstLogIndex = rb.getPrevLogIndex() + 1;
    this.statInfo.lastLogIndex = rb.getPrevLogIndex() + rb.getEntriesCount();

    final Recyclable recyclable = dataBuf;
    final int v = this.version;
    final long monotonicSendTimeMs = Utils.monotonicMs();
    final int seq = getAndIncrementReqSeq();
    //发送追加请求
    final Future<Message> rpcFuture = this.rpcService.appendEntries(this.options.getPeerId().getEndpoint(),
        request, -1, new RpcResponseClosureAdapter<AppendEntriesResponse>() {

            @Override
            public void run(final Status status) {
                RecycleUtil.recycle(recyclable);
                onRpcReturned(Replicator.this.id, RequestType.AppendEntries, status, request, getResponse(), seq,
                    v, monotonicSendTimeMs);
            }

        });
    addInflight(RequestType.AppendEntries, nextSendingIndex, request.getEntriesCount(), request.getData().size(),
        seq, rpcFuture);
    return true;

}
```

#### 从磁盘读取数据

com.alipay.sofa.jraft.storage.impl.LogManagerImpl#getEntry

```java
public LogEntry getEntry(final long index) {
    this.readLock.lock();
    try {
        if (index > this.lastLogIndex || index < this.firstLogIndex) { //验证index
            return null;
        }
        final LogEntry entry = getEntryFromMemory(index); //先从内存中读取
        if (entry != null) {
            return entry;
        }
    } finally {
        this.readLock.unlock();
    }
    //磁盘读取
    final LogEntry entry = this.logStorage.getEntry(index);
    if (entry == null) {
        reportError(RaftError.EIO.getNumber(), "Corrupted entry at index=%d, not found", index);
    }
    // 验证校验和，数据是否被损坏
    if (entry != null && this.raftOptions.isEnableLogEntryChecksum() && entry.isCorrupted()) {
        String msg = String.format("Corrupted entry at index=%d, term=%d, expectedChecksum=%d, realChecksum=%d",
            index, entry.getId().getTerm(), entry.getChecksum(), entry.checksum());
        // Report error to node and throw exception.
        reportError(RaftError.EIO.getNumber(), msg);
        throw new LogEntryCorruptedException(msg);
    }
    return entry;
}
```

# 快照

## 生成快照

### 定时器触发

NodeImpl初始化时，创建快照定时器，间隔时间默认1小时

```java
this.snapshotTimer = new RepeatedTimer("JRaft-SnapshotTimer", this.options.getSnapshotIntervalSecs() * 1000) {

    @Override
    protected void onTrigger() {
        handleSnapshotTimeout();
    }
};
```

com.alipay.sofa.jraft.core.NodeImpl#handleSnapshotTimeout

```java
private void handleSnapshotTimeout() {
    writeLock.lock();
    try {
        if (!this.state.isActive()) {
            return;
        }
    } finally {
        writeLock.unlock();
    }
    //在其他线程中做快照避免阻塞定时线程
    Utils.runInThread(() -> doSnapshot(null));
}
```

### 快照请求触发

com.alipay.sofa.jraft.rpc.impl.cli.SnapshotRequestProcessor#processRequest0

```java
protected Message processRequest0(CliRequestContext ctx, SnapshotRequest request, RpcRequestClosure done) {
    LOG.info("Receive SnapshotRequest to {} from {}", ctx.node.getNodeId(), request.getPeerId());
    ctx.node.snapshot(done);
    return null;
}
```

### 实现

com.alipay.sofa.jraft.core.NodeImpl#doSnapshot

```java
private void doSnapshot(Closure done) {
    if (this.snapshotExecutor != null) {
        snapshotExecutor.doSnapshot(done);
    } else {
        if (done != null) {
            final Status status = new Status(RaftError.EINVAL, "Snapshot is not supported");
            Utils.runClosureInThread(done, status);
        }
    }
}
```

com.alipay.sofa.jraft.storage.snapshot.SnapshotExecutorImpl#doSnapshot

```java
public void doSnapshot(final Closure done) {
    boolean doUnlock = true;
    this.lock.lock();
    try {
        if (this.stopped) {//已关闭
            Utils.runClosureInThread(done, new Status(RaftError.EPERM, "Is stopped."));
 	           return;
        }
        if (this.downloadingSnapshot.get() != null) { //正在加载快照
            Utils.runClosureInThread(done, new Status(RaftError.EBUSY, "Is loading another snapshot."));
            return;
        }

        if (this.savingSnapshot) { //正在生成快照，之前的触发的快照尚未结束
            Utils.runClosureInThread(done, new Status(RaftError.EBUSY, "Is saving another snapshot."));
            return;
        }
				//自上次快照之后无新增的数据
        if (this.fsmCaller.getLastAppliedIndex() == this.lastSnapshotIndex) {
            // There might be false positive as the getLastAppliedIndex() is being
            // updated. But it's fine since we will do next snapshot saving in a
            // predictable time.
            doUnlock = false;
            this.lock.unlock();
            this.logManager.clearBufferedLogs();
            Utils.runClosureInThread(done);
            return;
        }
				//自从上次快照之后，写入的数据数
        final long distance = this.fsmCaller.getLastAppliedIndex() - this.lastSnapshotIndex;
        if (distance < this.node.getOptions().getSnapshotLogIndexMargin()) {//新增数据过少，不做快照
            if (this.node != null) {
                LOG.debug(
                    "Node {} snapshotLogIndexMargin={}, distance={}, so ignore this time of snapshot by snapshotLogIndexMargin setting.",
                    this.node.getNodeId(), distance, this.node.getOptions().getSnapshotLogIndexMargin());
            }
            doUnlock = false;
            this.lock.unlock();
            Utils.runClosureInThread(done);
            return;
        }
				//创建SnapshotWriter
        final SnapshotWriter writer = this.snapshotStorage.create();
        if (writer == null) {
            Utils.runClosureInThread(done, new Status(RaftError.EIO, "Fail to create writer."));
            reportError(RaftError.EIO.getNumber(), "Fail to create snapshot writer.");
            return;
        }
        //标志正在生成快照
        this.savingSnapshot = true;
        final SaveSnapshotDone saveSnapshotDone = new SaveSnapshotDone(writer, done, null);
        //发布生成快照的事件
      	if (!this.fsmCaller.onSnapshotSave(saveSnapshotDone)) {
          	//服务端关闭或者Disruptor的RingBuffer已满
            Utils.runClosureInThread(done, new Status(RaftError.EHOSTDOWN, "The raft node is down."));
            return;
        }
        this.runningJobs.incrementAndGet();
    } finally {
        if (doUnlock) {
            this.lock.unlock();
        }
    }

}
```

com.alipay.sofa.jraft.core.FSMCallerImpl#doSnapshotSave

```java
private void doSnapshotSave(final SaveSnapshotClosure done) {
    Requires.requireNonNull(done, "SaveSnapshotClosure is null");
    final long lastAppliedIndex = this.lastAppliedIndex.get();
  	//生成快照时的配置信息
    final RaftOutter.SnapshotMeta.Builder metaBuilder = RaftOutter.SnapshotMeta.newBuilder() //
        .setLastIncludedIndex(lastAppliedIndex) //
        .setLastIncludedTerm(this.lastAppliedTerm);
    final ConfigurationEntry confEntry = this.logManager.getConfiguration(lastAppliedIndex);
    if (confEntry == null || confEntry.isEmpty()) {
        LOG.error("Empty conf entry for lastAppliedIndex={}", lastAppliedIndex);
        Utils.runClosureInThread(done, new Status(RaftError.EINVAL, "Empty conf entry for lastAppliedIndex=%s",
            lastAppliedIndex));
        return;
    }
    for (final PeerId peer : confEntry.getConf()) {
        metaBuilder.addPeers(peer.toString());
    }
    for (final PeerId peer : confEntry.getConf().getLearners()) {
        metaBuilder.addLearners(peer.toString());
    }
    if (confEntry.getOldConf() != null) {
        for (final PeerId peer : confEntry.getOldConf()) {
            metaBuilder.addOldPeers(peer.toString());
        }
        for (final PeerId peer : confEntry.getOldConf().getLearners()) {
            metaBuilder.addOldLearners(peer.toString());
        }
    }
    final SnapshotWriter writer = done.start(metaBuilder.build());
    if (writer == null) {
        done.run(new Status(RaftError.EINVAL, "snapshot_storage create SnapshotWriter failed"));
        return;
    }
    this.fsm.onSnapshotSave(writer, done);
}
```

## 发送快照

Follower落后的数据太多，Leader发送快照同步数据

com.alipay.sofa.jraft.core.Replicator#installSnapshot

```java
void installSnapshot() {
    if (this.state == State.Snapshot) {//正在通过快照同步数据
        LOG.warn("Replicator {} is installing snapshot, ignore the new request.", this.options.getPeerId());
        this.id.unlock();
        return;
    }
    boolean doUnlock = true;
    try {
        Requires.requireTrue(this.reader == null,
            "Replicator %s already has a snapshot reader, current state is %s", this.options.getPeerId(),
            this.state);
      //用于读取Leader节点的快照数据
        this.reader = this.options.getSnapshotStorage().open();
        if (this.reader == null) {
            final NodeImpl node = this.options.getNode();
            final RaftException error = new RaftException(EnumOutter.ErrorType.ERROR_TYPE_SNAPSHOT);
            error.setStatus(new Status(RaftError.EIO, "Fail to open snapshot"));
            this.id.unlock();
            doUnlock = false;
            node.onError(error);
            return;
        }
      	//拷贝快照的地址（remote://endpoint/readerId）
        final String uri = this.reader.generateURIForCopy();
        if (uri == null) {
            final NodeImpl node = this.options.getNode();
            final RaftException error = new RaftException(EnumOutter.ErrorType.ERROR_TYPE_SNAPSHOT);
            error.setStatus(new Status(RaftError.EIO, "Fail to generate uri for snapshot reader"));
            releaseReader();
            this.id.unlock();
            doUnlock = false;
            node.onError(error);
            return;
        }
        final RaftOutter.SnapshotMeta meta = this.reader.load();
        if (meta == null) {
            final String snapshotPath = this.reader.getPath();
            final NodeImpl node = this.options.getNode();
            final RaftException error = new RaftException(EnumOutter.ErrorType.ERROR_TYPE_SNAPSHOT);
            error.setStatus(new Status(RaftError.EIO, "Fail to load meta from %s", snapshotPath));
            releaseReader();
            this.id.unlock();
            doUnlock = false;
            node.onError(error);
            return;
        }
      	//发送安装快照的请求
        final InstallSnapshotRequest.Builder rb = InstallSnapshotRequest.newBuilder();
        rb.setTerm(this.options.getTerm());
        rb.setGroupId(this.options.getGroupId());
        rb.setServerId(this.options.getServerId().toString());
        rb.setPeerId(this.options.getPeerId().toString());
        rb.setMeta(meta);
        rb.setUri(uri);

        this.statInfo.runningState = RunningState.INSTALLING_SNAPSHOT;
        this.statInfo.lastLogIncluded = meta.getLastIncludedIndex();
        this.statInfo.lastTermIncluded = meta.getLastIncludedTerm();

        final InstallSnapshotRequest request = rb.build();
        this.state = State.Snapshot;
        // noinspection NonAtomicOperationOnVolatileField
        this.installSnapshotCounter++;
        final long monotonicSendTimeMs = Utils.monotonicMs();
        final int stateVersion = this.version;
        final int seq = getAndIncrementReqSeq();
        final Future<Message> rpcFuture = this.rpcService.installSnapshot(this.options.getPeerId().getEndpoint(),
            request, new RpcResponseClosureAdapter<InstallSnapshotResponse>() {

                @Override
                public void run(final Status status) {
                    onRpcReturned(Replicator.this.id, RequestType.Snapshot, status, request, getResponse(), seq,
                        stateVersion, monotonicSendTimeMs);
                }
            });
        addInflight(RequestType.Snapshot, this.nextIndex, 0, 0, seq, rpcFuture);
    } finally {
        if (doUnlock) {
            this.id.unlock();
        }
    }
}
```

## 安装快照*

com.alipay.sofa.jraft.core.NodeImpl#handleInstallSnapshot

```java
public Message handleInstallSnapshot(final InstallSnapshotRequest request, final RpcRequestClosure done) {
    if (this.snapshotExecutor == null) {
        return RpcResponseFactory.newResponse(RaftError.EINVAL, "Not supported snapshot");
    }
    final PeerId serverId = new PeerId();
    if (!serverId.parse(request.getServerId())) { //请求的节点不合法
        LOG.warn("Node {} ignore InstallSnapshotRequest from {} bad server id.", getNodeId(), request.getServerId());
        return RpcResponseFactory.newResponse(RaftError.EINVAL, "Parse serverId failed: %s", request.getServerId());
    }

    this.writeLock.lock();
    try {
        if (!this.state.isActive()) { //节点不正常
            LOG.warn("Node {} ignore InstallSnapshotRequest as it is not in active state {}.", getNodeId(),
                this.state);
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Node %s:%s is not in active state, state %s.",
                this.groupId, this.serverId, this.state.name());
        }

        if (request.getTerm() < this.currTerm) { //小于当前的term
            LOG.warn("Node {} ignore stale InstallSnapshotRequest from {}, term={}, currTerm={}.", getNodeId(),
                request.getPeerId(), request.getTerm(), this.currTerm);
            return InstallSnapshotResponse.newBuilder() //
                .setTerm(this.currTerm) // 返回当前term
                .setSuccess(false) // 返回false
                .build();
        }

        checkStepDown(request.getTerm(), serverId);

        if (!serverId.equals(this.leaderId)) { //请求来自Leader
            LOG.error("Another peer {} declares that it is the leader at term {} which was occupied by leader {}.",
                serverId, this.currTerm, this.leaderId);
            // Increase the term by 1 and make both leaders step down to minimize the
            // loss of split brain
            stepDown(request.getTerm() + 1, false, new Status(RaftError.ELEADERCONFLICT,
                "More than one leader in the same term."));
            return InstallSnapshotResponse.newBuilder() //
                .setTerm(request.getTerm() + 1) //
                .setSuccess(false) //
                .build();
        }

    } finally {
        this.writeLock.unlock();
    }
    final long startMs = Utils.monotonicMs();
    try {
        if (LOG.isInfoEnabled()) {
            LOG.info(
                "Node {} received InstallSnapshotRequest from {}, lastIncludedLogIndex={}, lastIncludedLogTerm={}, lastLogId={}.",
                getNodeId(), request.getServerId(), request.getMeta().getLastIncludedIndex(), request.getMeta()
                    .getLastIncludedTerm(), this.logManager.getLastLogId(false));
        }
        this.snapshotExecutor.installSnapshot(request, InstallSnapshotResponse.newBuilder(), done);
        return null;
    } finally {
        this.metrics.recordLatency("install-snapshot", Utils.monotonicMs() - startMs);
    }
}
```

com.alipay.sofa.jraft.storage.snapshot.SnapshotExecutorImpl#installSnapshot

```java
public void installSnapshot(final InstallSnapshotRequest request, final InstallSnapshotResponse.Builder response,
                            final RpcRequestClosure done) {
    final SnapshotMeta meta = request.getMeta();
    final DownloadingSnapshot ds = new DownloadingSnapshot(request, response, done);
    // DON'T access request, response, and done after this point
    // as the retry snapshot will replace this one.
    if (!registerDownloadingSnapshot(ds)) {
        LOG.warn("Fail to register downloading snapshot.");
        // This RPC will be responded by the previous session
        return;
    }
    Requires.requireNonNull(this.curCopier, "curCopier");
    try {
        this.curCopier.join();
    } catch (final InterruptedException e) {
        Thread.currentThread().interrupt();
        LOG.warn("Install snapshot copy job was canceled.");
        return;
    }

    loadDownloadingSnapshot(ds, meta);
}
```

## 获取快照

com.alipay.sofa.jraft.rpc.impl.core.GetFileRequestProcessor#processRequest

```java
public Message processRequest(final GetFileRequest request, final RpcRequestClosure done) {
    return FileService.getInstance().handleGetFile(request, done);
}
```

com.alipay.sofa.jraft.storage.FileService#handleGetFile

```java
public Message handleGetFile(final GetFileRequest request, final RpcRequestClosure done) {
  	//校验合法性
    if (request.getCount() <= 0 || request.getOffset() < 0) {
        return RpcResponseFactory.newResponse(RaftError.EREQUEST, "Invalid request: %s", request);
    }
     //获取FileReader，生成快照的时候会放入fileReaderMap
    final FileReader reader = this.fileReaderMap.get(request.getReaderId());
    if (reader == null) {
        return RpcResponseFactory.newResponse(RaftError.ENOENT, "Fail to find reader=%d", request.getReaderId());
    }

    if (LOG.isDebugEnabled()) {
        LOG.debug("GetFile from {} path={} filename={} offset={} count={}", done.getRpcCtx().getRemoteAddress(),
            reader.getPath(), request.getFilename(), request.getOffset(), request.getCount());
	    }
		//ByteBuffer分配器
    final ByteBufferCollector dataBuffer = ByteBufferCollector.allocate();
    final GetFileResponse.Builder responseBuilder = GetFileResponse.newBuilder();
    try {
      	//读取文件内容放入dataBuffer
        final int read = reader
            .readFile(dataBuffer, request.getFilename(), request.getOffset(), request.getCount());
        responseBuilder.setReadSize(read);
        responseBuilder.setEof(read == FileReader.EOF);
        final ByteBuffer buf = dataBuffer.getBuffer();
        buf.flip();	
        if (!buf.hasRemaining()) { //空数据
            responseBuilder.setData(ByteString.EMPTY);
        } else {
            responseBuilder.setData(ZeroByteStringHelper.wrap(buf));
        }
        return responseBuilder.build();
    } catch (final RetryAgainException e) {
        return RpcResponseFactory.newResponse(RaftError.EAGAIN,
            "Fail to read from path=%s filename=%s with error: %s", reader.getPath(), request.getFilename(),
            e.getMessage());
    } catch (final IOException e) {
        LOG.error("Fail to read file path={} filename={}", reader.getPath(), request.getFilename(), e);
        return RpcResponseFactory.newResponse(RaftError.EIO, "Fail to read from path=%s filename=%s",
            reader.getPath(), request.getFilename());
    }
}
```

# 处理请求

## 初始化Disruptor

com.alipay.sofa.jraft.core.NodeImpl#init

```java
this.applyDisruptor = DisruptorBuilder.<LogEntryAndClosure> newInstance() //
    .setRingBufferSize(this.raftOptions.getDisruptorBufferSize()) //数组长度，默认16384，2的幂次方
    .setEventFactory(new LogEntryAndClosureFactory()) //负责事先生成事件，填充数组
    .setThreadFactory(new NamedThreadFactory("JRaft-NodeImpl-Disruptor-", true)) //
    .setProducerType(ProducerType.MULTI) // 多生产者模式
    .setWaitStrategy(new BlockingWaitStrategy()) // 等待策略
    .build();
this.applyDisruptor.handleEventsWith(new LogEntryAndClosureHandler()); //处理事件
this.applyDisruptor.setDefaultExceptionHandler(new LogExceptionHandler<Object>(getClass().getSimpleName()));
this.applyQueue = this.applyDisruptor.start();
```

## 发布客户端请求事件

com.alipay.sofa.jraft.core.NodeImpl#apply

```java
public void apply(final Task task) {
    if (this.shutdownLatch != null) {
        Utils.runClosureInThread(task.getDone(), new Status(RaftError.ENODESHUTDOWN, "Node is shutting down."));
        throw new IllegalStateException("Node is shutting down");
    }
    Requires.requireNonNull(task, "Null task");

    final LogEntry entry = new LogEntry();
    entry.setData(task.getData());
    int retryTimes = 0;
    try {
        //负责生成事件，基于Disruptor实现
        final EventTranslator<LogEntryAndClosure> translator = (event, sequence) -> {
            event.reset();
            event.done = task.getDone();
            event.entry = entry;
            event.expectedTerm = task.getExpectedTerm();
        };
        while (true) {
            if (this.applyQueue.tryPublishEvent(translator)) { //发布事件
                break;
            } else {
                retryTimes++;
                if (retryTimes > MAX_APPLY_RETRY_TIMES) { //默认重试3次
                    Utils.runClosureInThread(task.getDone(),
                        new Status(RaftError.EBUSY, "Node is busy, has too many tasks."));
                    LOG.warn("Node {} applyQueue is overload.", getNodeId());
                    this.metrics.recordTimes("apply-task-overload-times", 1);
                    return;
                }
                ThreadHelper.onSpinWait();
            }
        }

    } catch (final Exception e) {
        LOG.error("Fail to apply task.", e);
        Utils.runClosureInThread(task.getDone(), new Status(RaftError.EPERM, "Node is down."));
    }
}
```

## 处理客户端请求事件

com.alipay.sofa.jraft.core.NodeImpl.LogEntryAndClosureHandler#onEvent

```java
public void onEvent(LogEntryAndClosure event, long sequence, boolean endOfBatch) throws Exception {
    if (event.shutdownLatch != null) {
        if (!tasks.isEmpty()) {
            executeApplyingTasks(tasks);
        }
        GLOBAL_NUM_NODES.decrementAndGet();
        event.shutdownLatch.countDown();
        return;
    }
		//暂存事件，批量处理
    tasks.add(event);
    if (tasks.size() >= raftOptions.getApplyBatch() || endOfBatch) {
        executeApplyingTasks(tasks);
        tasks.clear();
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl#executeApplyingTasks

```java
private void executeApplyingTasks(final List<LogEntryAndClosure> tasks) {
    this.writeLock.lock();
    try {
        final int size = tasks.size();
        if (this.state != State.STATE_LEADER) { //不再是Leader
            final Status st = new Status();
            if (this.state != State.STATE_TRANSFERRING) {
                st.setError(RaftError.EPERM, "Is not leader.");
            } else {
                st.setError(RaftError.EBUSY, "Is transferring leadership.");
            }
            LOG.debug("Node {} can't apply, status={}.", getNodeId(), st);
            final List<LogEntryAndClosure> savedTasks = new ArrayList<>(tasks);
            //线程池中调用回调方法
            Utils.runInThread(() -> {
                for (int i = 0; i < size; i++) {
                    savedTasks.get(i).done.run(st);
                }
            });
            return;
        }
        final List<LogEntry> entries = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            final LogEntryAndClosure task = tasks.get(i);
            if (task.expectedTerm != -1 && task.expectedTerm != this.currTerm) {//term不匹配
                LOG.debug("Node {} can't apply task whose expectedTerm={} doesn't match currTerm={}.", getNodeId(),
                    task.expectedTerm, this.currTerm);
              	//调用回调方法
                if (task.done != null) {
                    final Status st = new Status(RaftError.EPERM, "expected_term=%d doesn't match current_term=%d",
                        task.expectedTerm, this.currTerm);
                    Utils.runClosureInThread(task.done, st); //返回响应给客户端
                }
                continue;
            }
            if (!this.ballotBox.appendPendingTask(this.conf.getConf(),
                this.conf.isStable() ? null : this.conf.getOldConf(), task.done)) {
                Utils.runClosureInThread(task.done, new Status(RaftError.EINTERNAL, "Fail to append task."));
                continue;
            }
            // set task entry info before adding to list.
            task.entry.getId().setTerm(this.currTerm);
            task.entry.setType(EnumOutter.EntryType.ENTRY_TYPE_DATA); //数据类型
            entries.add(task.entry);
        }
        this.logManager.appendEntries(entries, new LeaderStableClosure(entries));
        // update conf.first
        checkAndSetConfiguration(true);
    } finally {
        this.writeLock.unlock();
    }
}
```

com.alipay.sofa.jraft.core.BallotBox#appendPendingTask

```java
public boolean appendPendingTask(final Configuration conf, final Configuration oldConf, final Closure done) {
    final Ballot bl = new Ballot();
    if (!bl.init(conf, oldConf)) {
        LOG.error("Fail to init ballot.");
        return false;
    }
    final long stamp = this.stampedLock.writeLock();
    try {
        if (this.pendingIndex <= 0) {
            LOG.error("Fail to appendingTask, pendingIndex={}.", this.pendingIndex);
            return false;
        }
        this.pendingMetaQueue.add(bl);//暂存客户端请求
        this.closureQueue.appendPendingClosure(done);
        return true;
    } finally {
        this.stampedLock.unlockWrite(stamp);
    }
}
```

## 发布磁盘写事件

写入的数据会暂存在内存中，避免了复制数据时的磁盘读取，降低复制延迟

com.alipay.sofa.jraft.storage.impl.LogManagerImpl#appendEntries

```java
public void appendEntries(final List<LogEntry> entries, final StableClosure done) {
    Requires.requireNonNull(done, "done");
    if (this.hasError) {
        entries.clear();
        Utils.runClosureInThread(done, new Status(RaftError.EIO, "Corrupted LogStorage"));
        return;
    }
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
        if (!entries.isEmpty() && !checkAndResolveConflict(entries, done)) {
            entries.clear();
            Utils.runClosureInThread(done, new Status(RaftError.EINTERNAL, "Fail to checkAndResolveConflict."));
            return;
        }
        
        for (int i = 0; i < entries.size(); i++) {
            final LogEntry entry = entries.get(i);
            // Set checksum after checkAndResolveConflict
            if (this.raftOptions.isEnableLogEntryChecksum()) {
                entry.setChecksum(entry.checksum()); //校验和
            }
            if (entry.getType() == EntryType.ENTRY_TYPE_CONFIGURATION) {
                Configuration oldConf = new Configuration();
                if (entry.getOldPeers() != null) {
                    oldConf = new Configuration(entry.getOldPeers(), entry.getOldLearners());
                }
                final ConfigurationEntry conf = new ConfigurationEntry(entry.getId(),
                    new Configuration(entry.getPeers(), entry.getLearners()), oldConf);
                this.configManager.add(conf);
            }
        }
        if (!entries.isEmpty()) {
            done.setFirstLogIndex(entries.get(0).getId().getIndex()); //第一条日志的index
            this.logsInMemory.addAll(entries); //暂存到内存中
        }
        done.setEntries(entries);

        int retryTimes = 0;
        //负责生成StableClosureEvent
        final EventTranslator<StableClosureEvent> translator = (event, sequence) -> {
            event.reset();
            event.type = EventType.OTHER;
            event.done = done;
        };
        while (true) {
            if (tryOfferEvent(done, translator)) { //发布磁盘事件
                break;
            } else {
                retryTimes++;
                if (retryTimes > APPEND_LOG_RETRY_TIMES) { //默认50，磁盘负载过重
                    reportError(RaftError.EBUSY.getNumber(), "LogManager is busy, disk queue overload.");
                    return;
                }
                ThreadHelper.onSpinWait();
            }
        }
        doUnlock = false;
        if (!wakeupAllWaiter(this.writeLock)) {
            notifyLastLogIndexListeners();
        }
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
}
```

## 处理磁盘写事件

```java
LogId               lastId  = LogManagerImpl.this.diskId;
List<StableClosure> storage = new ArrayList<>(256);
AppendBatcher       ab      = new AppendBatcher(this.storage, 256, new ArrayList<>(),
                                LogManagerImpl.this.diskId);
```

com.alipay.sofa.jraft.storage.impl.LogManagerImpl.StableClosureEventHandler#onEvent

```java
public void onEvent(final StableClosureEvent event, final long sequence, final boolean endOfBatch)                                                                                               throws Exception {
    if (event.type == EventType.SHUTDOWN) {//关闭事件
        this.lastId = this.ab.flush(); //刷新到磁盘
        setDiskId(this.lastId);
        LogManagerImpl.this.shutDownLatch.countDown();
        return;
    }
    final StableClosure done = event.done;

    if (done.getEntries() != null && !done.getEntries().isEmpty()) { //追加数据不为空
        this.ab.append(done); //暂存到AppendBatcher
    } else {
        this.lastId = this.ab.flush(); //刷新到磁盘
        boolean ret = true;
        switch (event.type) {
            case LAST_LOG_ID:
                ((LastLogIdClosure) done).setLastLogId(this.lastId.copy());
                break;
            case TRUNCATE_PREFIX:
                long startMs = Utils.monotonicMs();
                try {
                    final TruncatePrefixClosure tpc = (TruncatePrefixClosure) done;
                    LOG.debug("Truncating storage to firstIndexKept={}.", tpc.firstIndexKept);
                    ret = LogManagerImpl.this.logStorage.truncatePrefix(tpc.firstIndexKept);
                } finally {
                    LogManagerImpl.this.nodeMetrics.recordLatency("truncate-log-prefix", Utils.monotonicMs()
                                                                                         - startMs);
                }
                break;
            case TRUNCATE_SUFFIX:
                startMs = Utils.monotonicMs();
                try {
                    final TruncateSuffixClosure tsc = (TruncateSuffixClosure) done;
                    LOG.warn("Truncating storage to lastIndexKept={}.", tsc.lastIndexKept);
                    ret = LogManagerImpl.this.logStorage.truncateSuffix(tsc.lastIndexKept);
                    if (ret) {
                        this.lastId.setIndex(tsc.lastIndexKept);
                        this.lastId.setTerm(tsc.lastTermKept);
                        Requires.requireTrue(this.lastId.getIndex() == 0 || this.lastId.getTerm() != 0);
                    }
                } finally {
                    LogManagerImpl.this.nodeMetrics.recordLatency("truncate-log-suffix", Utils.monotonicMs()
                                                                                         - startMs);
                }
                break;
            case RESET:
                final ResetClosure rc = (ResetClosure) done;
                LOG.info("Resetting storage to nextLogIndex={}.", rc.nextLogIndex);
                ret = LogManagerImpl.this.logStorage.reset(rc.nextLogIndex);
                break;
            default:
                break;
        }

        if (!ret) {
            reportError(RaftError.EIO.getNumber(), "Failed operation in LogStorage");
        } else {
            done.run(Status.OK());
        }
    }
    if (endOfBatch) {
        this.lastId = this.ab.flush();
        setDiskId(this.lastId);
    }
}
```

com.alipay.sofa.jraft.storage.impl.LogManagerImpl.AppendBatcher#flush

```java
LogId flush() {
    if (this.size > 0) { //暂存的数据不为空
        this.lastId = appendToStorage(this.toAppend);
        for (int i = 0; i < this.size; i++) {
            this.storage.get(i).getEntries().clear();
            Status st = null;
            try {
                if (LogManagerImpl.this.hasError) {
                    st = new Status(RaftError.EIO, "Corrupted LogStorage");
                } else {
                    st = Status.OK();
                }
                this.storage.get(i).run(st); //调用LeaderStableClosure
            } catch (Throwable t) {
                LOG.error("Fail to run closure with status: {}.", st, t);
            }
        }
        this.toAppend.clear();
        this.storage.clear();

    }
    this.size = 0;
    this.bufferSize = 0;
    return this.lastId;
}
```

## 写磁盘回调

com.alipay.sofa.jraft.core.NodeImpl.LeaderStableClosure#run

```java
public void run(final Status status) {
    if (status.isOk()) {
        NodeImpl.this.ballotBox.commitAt(this.firstLogIndex, this.firstLogIndex + this.nEntries - 1,
            NodeImpl.this.serverId);
    } else {
        LOG.error("Node {} append [{}, {}] failed, status={}.", getNodeId(), this.firstLogIndex,
            this.firstLogIndex + this.nEntries - 1, status);
    }
}
```

com.alipay.sofa.jraft.core.BallotBox#commitAt

```java
public boolean commitAt(final long firstLogIndex, final long lastLogIndex, final PeerId peer) {
    // TODO  use lock-free algorithm here?
    final long stamp = this.stampedLock.writeLock();
    long lastCommittedIndex = 0;
    try {
        if (this.pendingIndex == 0) { //初始状态或者已被重置
            return false;
        }
        if (lastLogIndex < this.pendingIndex) { //已经被提交过了
            return true;
        }

        if (lastLogIndex >= this.pendingIndex + this.pendingMetaQueue.size()) {//越界，不合法
            throw new ArrayIndexOutOfBoundsException();
        }
				//获取较大值，从此值之后开始提交
        final long startAt = Math.max(this.pendingIndex, firstLogIndex);
        Ballot.PosHint hint = new Ballot.PosHint();
        for (long logIndex = startAt; logIndex <= lastLogIndex; logIndex++) {
          	//获取bolt
            final Ballot bl = this.pendingMetaQueue.get((int) (logIndex - this.pendingIndex));
            hint = bl.grant(peer, hint);
            if (bl.isGranted()) { //集群半数节点都已提交
                lastCommittedIndex = logIndex; //修改提交的日志index
            }
        }
        if (lastCommittedIndex == 0) {
            return true;
        }
        // When removing a peer off the raft group which contains even number of
        // peers, the quorum would decrease by 1, e.g. 3 of 4 changes to 2 of 3. In
        // this case, the log after removal may be committed before some previous
        // logs, since we use the new configuration to deal the quorum of the
        // removal request, we think it's safe to commit all the uncommitted
        // previous logs, which is not well proved right now
        this.pendingMetaQueue.removeFromFirst((int) (lastCommittedIndex - this.pendingIndex) + 1);
        LOG.debug("Committed log fromIndex={}, toIndex={}.", this.pendingIndex, lastCommittedIndex);
        this.pendingIndex = lastCommittedIndex + 1; //修改pendingIndex
        this.lastCommittedIndex = lastCommittedIndex; //修改lastCommittedIndex
    } finally {
        this.stampedLock.unlockWrite(stamp);
    }
    this.waiter.onCommitted(lastCommittedIndex); //发布提交事件
    return true;
}
```

## 发布提交事件

com.alipay.sofa.jraft.core.FSMCallerImpl#onCommitted

```java
public boolean onCommitted(final long committedIndex) {
    return enqueueTask((task, sequence) -> {
        task.type = TaskType.COMMITTED;
        task.committedIndex = committedIndex;
    });
}
```

## 处理提交事件

com.alipay.sofa.jraft.core.FSMCallerImpl.ApplyTaskHandler#onEvent

```java
public void onEvent(final ApplyTask event, final long sequence, final boolean endOfBatch) throws Exception {
    this.maxCommittedIndex = runApplyTask(event, this.maxCommittedIndex, endOfBatch);
}
```

com.alipay.sofa.jraft.core.FSMCallerImpl#runApplyTask

```java
private long runApplyTask(final ApplyTask task, long maxCommittedIndex, final boolean endOfBatch) {
    CountDownLatch shutdown = null;
    if (task.type == TaskType.COMMITTED) { //提交类型
        if (task.committedIndex > maxCommittedIndex) {
            maxCommittedIndex = task.committedIndex; //修改maxCommittedIndex
        }
    } else {
        if (maxCommittedIndex >= 0) {
            this.currTask = TaskType.COMMITTED;
            doCommitted(maxCommittedIndex);
            maxCommittedIndex = -1L; // reset maxCommittedIndex
        }
        final long startMs = Utils.monotonicMs();
        try {
            switch (task.type) {
                case COMMITTED:
                    Requires.requireTrue(false, "Impossible");
                    break;
                case SNAPSHOT_SAVE:
                    this.currTask = TaskType.SNAPSHOT_SAVE;
                    if (passByStatus(task.done)) {
                        doSnapshotSave((SaveSnapshotClosure) task.done);
                    }
                    break;
                case SNAPSHOT_LOAD:
                    this.currTask = TaskType.SNAPSHOT_LOAD;
                    if (passByStatus(task.done)) {
                        doSnapshotLoad((LoadSnapshotClosure) task.done);
                    }
                    break;
                case LEADER_STOP:
                    this.currTask = TaskType.LEADER_STOP;
                    doLeaderStop(task.status);
                    break;
                case LEADER_START:
                    this.currTask = TaskType.LEADER_START;
                    doLeaderStart(task.term);
                    break;
                case START_FOLLOWING:
                    this.currTask = TaskType.START_FOLLOWING;
                    doStartFollowing(task.leaderChangeCtx);
                    break;
                case STOP_FOLLOWING:
                    this.currTask = TaskType.STOP_FOLLOWING;
                    doStopFollowing(task.leaderChangeCtx);
                    break;
                case ERROR:
                    this.currTask = TaskType.ERROR;
                    doOnError((OnErrorClosure) task.done);
                    break;
                case IDLE:
                    Requires.requireTrue(false, "Can't reach here");
                    break;
                case SHUTDOWN:
                    this.currTask = TaskType.SHUTDOWN;
                    shutdown = task.shutdownLatch;
                    break;
                case FLUSH:
                    this.currTask = TaskType.FLUSH;
                    shutdown = task.shutdownLatch;
                    break;
            }
        } finally {
            this.nodeMetrics.recordLatency(task.type.metricName(), Utils.monotonicMs() - startMs);
        }
    }
    try {
        if (endOfBatch && maxCommittedIndex >= 0) { //批量提交
            this.currTask = TaskType.COMMITTED;
            doCommitted(maxCommittedIndex); //真正的提交
            maxCommittedIndex = -1L; // reset maxCommittedIndex
        }
        this.currTask = TaskType.IDLE;
        return maxCommittedIndex;
    } finally {
        if (shutdown != null) {
            shutdown.countDown();
        }
    }
}
```

com.alipay.sofa.jraft.core.FSMCallerImpl#doCommitted

```java
private void doCommitted(final long committedIndex) {
    if (!this.error.getStatus().isOk()) {
        return;
    }
    final long lastAppliedIndex = this.lastAppliedIndex.get();
    // We can tolerate the disorder of committed_index
    if (lastAppliedIndex >= committedIndex) { //已经被应用到状态机
        return;
    }
    final long startMs = Utils.monotonicMs();
    try {
        final List<Closure> closures = new ArrayList<>();
        final List<TaskClosure> taskClosures = new ArrayList<>();
      	//closureQueue存放客户端回调方法
        final long firstClosureIndex = this.closureQueue.popClosureUntil(committedIndex, closures, taskClosures);

        // Calls TaskClosure#onCommitted if necessary
        onTaskCommitted(taskClosures);

        Requires.requireTrue(firstClosureIndex >= 0, "Invalid firstClosureIndex");
        final IteratorImpl iterImpl = new IteratorImpl(this.fsm, this.logManager, closures, firstClosureIndex,
            lastAppliedIndex, committedIndex, this.applyingIndex);
        while (iterImpl.isGood()) {
            final LogEntry logEntry = iterImpl.entry();
            if (logEntry.getType() != EnumOutter.EntryType.ENTRY_TYPE_DATA) { //不是数据类型
                if (logEntry.getType() == EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION) {//配置类型
                    if (logEntry.getOldPeers() != null && !logEntry.getOldPeers().isEmpty()) {
                        // Joint stage is not supposed to be noticeable by end users.
                        this.fsm.onConfigurationCommitted(new Configuration(iterImpl.entry().getPeers()));
                    }
                }
                if (iterImpl.done() != null) {
                    // For other entries, we have nothing to do besides flush the
                    // pending tasks and run this closure to notify the caller that the
                    // entries before this one were successfully committed and applied.
                    iterImpl.done().run(Status.OK());
                }
                iterImpl.next();
                continue;
            }

            // Apply data task to user state machine
            doApplyTasks(iterImpl); //应用到状态机
        }

        if (iterImpl.hasError()) {
            setError(iterImpl.getError());
            iterImpl.runTheRestClosureWithError();
        }
        final long lastIndex = iterImpl.getIndex() - 1;
        final long lastTerm = this.logManager.getTerm(lastIndex);
        final LogId lastAppliedId = new LogId(lastIndex, lastTerm);
        this.lastAppliedIndex.set(lastIndex);
        this.lastAppliedTerm = lastTerm;
        this.logManager.setAppliedId(lastAppliedId); //清除内存中暂存的数据
        notifyLastAppliedIndexUpdated(lastIndex);
    } finally {
        this.nodeMetrics.recordLatency("fsm-commit", Utils.monotonicMs() - startMs);
    }
}
```

com.alipay.sofa.jraft.core.FSMCallerImpl#doApplyTasks

```java
private void doApplyTasks(final IteratorImpl iterImpl) {
    final IteratorWrapper iter = new IteratorWrapper(iterImpl);
    final long startApplyMs = Utils.monotonicMs();
    final long startIndex = iter.getIndex();
    try {
        this.fsm.onApply(iter);
    } finally {
        this.nodeMetrics.recordLatency("fsm-apply-tasks", Utils.monotonicMs() - startApplyMs);
        this.nodeMetrics.recordSize("fsm-apply-tasks-count", iter.getIndex() - startIndex);
    }
    if (iter.hasNext()) {
        LOG.error("Iterator is still valid, did you return before iterator reached the end?");
    }
    // Try move to next in case that we pass the same log twice.
    iter.next();
}
```

## 应用到状态机

### CounterStateMachine

com.alipay.sofa.jraft.example.counter.CounterStateMachine#onApply

```java
public void onApply(final Iterator iter) { //实现计数
    while (iter.hasNext()) {
        long current = 0;
        CounterOperation counterOperation = null;

        CounterClosure closure = null;
        if (iter.done() != null) { //客户端请求的回调函数不为空
            closure = (CounterClosure) iter.done();
            counterOperation = closure.getCounterOperation(); //计数操作
        } else {
            // Have to parse FetchAddRequest from this user log.
            final ByteBuffer data = iter.getData();
            try {
                counterOperation = SerializerManager.getSerializer(SerializerManager.Hessian2).deserialize(
                    data.array(), CounterOperation.class.getName());
            } catch (final CodecException e) {
                LOG.error("Fail to decode IncrementAndGetRequest", e);
            }
        }
        if (counterOperation != null) {
            switch (counterOperation.getOp()) {
                case GET: //从状态机读取计数值
                    current = this.value.get(); //状态机最新值
                    LOG.info("Get value={} at logIndex={}", current, iter.getIndex());
                    break;
                case INCREMENT: //增加计数
                    final long delta = counterOperation.getDelta();
                    final long prev = this.value.get();
                    current = this.value.addAndGet(delta);//状态机最新值
                    LOG.info("Added value={} by delta={} at logIndex={}", prev, delta, iter.getIndex());
                    break;
            }

            if (closure != null) { //调用回调，返回响应给客户端
                closure.success(current);//设置状态机最新值
                closure.run(Status.OK()); //设置响应状态
            }
        }
        iter.next();
    }
}
```

### AtomicStateMachine

com.alipay.sofa.jraft.test.atomic.server.AtomicStateMachine#onApply

```java
public void onApply(final Iterator iter) { //原子性状态机
    while (iter.hasNext()) {
        final Closure done = iter.done();
        CommandType cmdType;
        final ByteBuffer data = iter.getData();
        Object cmd = null;
        LeaderTaskClosure closure = null;
        if (done != null) {
            closure = (LeaderTaskClosure) done;
            cmdType = closure.getCmdType();
            cmd = closure.getCmd();
        } else {
            final byte b = data.get();
            final byte[] cmdBytes = new byte[data.remaining()];
            data.get(cmdBytes);
            cmdType = CommandType.parseByte(b); //获取命令类型
            switch (cmdType) { //反序列化为BaseRequestCommand
                case GET:
                    cmd = CommandCodec.decodeCommand(cmdBytes, GetCommand.class);
                    break;
                case SET:
                    cmd = CommandCodec.decodeCommand(cmdBytes, SetCommand.class);
                    break;
                case CAS:
                    cmd = CommandCodec.decodeCommand(cmdBytes, CompareAndSetCommand.class);
                    break;
                case INC:
                    cmd = CommandCodec.decodeCommand(cmdBytes, IncrementAndGetCommand.class);
                    break;
            }
        }
        final String key = ((BaseRequestCommand) cmd).getKey(); //获取key
        final AtomicLong counter = getCounter(key, true);
        Object response = null;
        switch (cmdType) {
            case GET: //查找
                response = new ValueCommand(counter.get());
                break;
            case SET: //设置
                final SetCommand setCmd = (SetCommand) cmd;
                counter.set(setCmd.getValue());
                response = new BooleanCommand(true);
                break;
            case CAS: //修改
                final CompareAndSetCommand casCmd = (CompareAndSetCommand) cmd;
                response = new BooleanCommand(counter.compareAndSet(casCmd.getExpect(), casCmd.getNewValue()));
                break;
            case INC: //递增
                final IncrementAndGetCommand incCmd = (IncrementAndGetCommand) cmd;
                final long ret = counter.addAndGet(incCmd.getDetal());
                response = new ValueCommand(ret);
                break;
        }
        if (closure != null) { //执行回调方法，放回响应信息
            closure.setResponse(response);
            closure.run(Status.OK());
        }
        iter.next();
    }
}
```

# 线性一致读

当一个 Client 向集群发起写操作的请求并且得到成功响应之后，该写操作的结果要对所有后来的读请求可见

## 实现1

走 Raft 协议，读请求和写请求的处理流程一样，leader存储Log ，复制 Log 到follower，等待应用到状态机再读取数据，然后把结果返回给 Client,带来刷盘开销、存储开销、网络开销

## 实现2

ReadIndex,直接从 Leader 读取结果：所有已经复制到多数派上的 Log（可视为写操作）就可以被视为安全的 Log

保证写入成功的数据，从任何节点读取都可以获取到最新写入的数据

### 接收readIndex请求

com.alipay.sofa.jraft.core.NodeImpl#readIndex

```java
public void readIndex(final byte[] requestContext, final ReadIndexClosure done) {
    if (this.shutdownLatch != null) {//服务端关闭
        Utils.runClosureInThread(done, new Status(RaftError.ENODESHUTDOWN, "Node is shutting down."));
        throw new IllegalStateException("Node is shutting down");
    }
    Requires.requireNonNull(done, "Null closure");
  	//只读服务ReadOnlyServiceImpl，Disruptor异步处理
    this.readOnlyService.addRequest(requestContext, done);
}
```

### 发布readIndex事件

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl#addRequest

```java
public void addRequest(final byte[] reqCtx, final ReadIndexClosure closure) {
    if (this.shutdownLatch != null) {
        Utils.runClosureInThread(closure, new Status(RaftError.EHOSTDOWN, "Was stopped"));
        throw new IllegalStateException("Service already shutdown.");
    }
    try {
      	//负责生成ReadIndexEvent，交由Disruptor处理
        EventTranslator<ReadIndexEvent> translator = (event, sequence) -> {
            event.done = closure;
            event.requestContext = new Bytes(reqCtx);
            event.startTime = Utils.monotonicMs();
        };
        int retryTimes = 0;
        while (true) {
            if (this.readIndexQueue.tryPublishEvent(translator)) { //发布事件成功，退出循环
                break;
            } else {
                retryTimes++;
                //当readIndexQueue田填满时，默认重试3次
                if (retryTimes > MAX_ADD_REQUEST_RETRY_TIMES) {
                  	//积压太多的请求
                    Utils.runClosureInThread(closure,
                        new Status(RaftError.EBUSY, "Node is busy, has too many read-only requests."));
                    this.nodeMetrics.recordTimes("read-index-overload-times", 1);
                    LOG.warn("Node {} ReadOnlyServiceImpl readIndexQueue is overload.", this.node.getNodeId());
                    return;
                }
              	//默认调用Thread.yield方法
                ThreadHelper.onSpinWait();
            }
        }
    } catch (final Exception e) {
        Utils.runClosureInThread(closure, new Status(RaftError.EPERM, "Node is down."));
    }
}
```

### 处理readIndex事件

```f
ReadIndexEventHandler负责处理只读请求
```

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl.ReadIndexEventHandler#onEvent

```java
public void onEvent(final ReadIndexEvent newEvent, final long sequence, final boolean endOfBatch) throws Exception {
    if (newEvent.shutdownLatch != null) {
        executeReadIndexEvents(this.events);
        this.events.clear();
        newEvent.shutdownLatch.countDown();
        return;
    }
		//放入events列表
    this.events.add(newEvent);
  	//当累积请求超过32或者Disruptor的RingBuffer中无积压的请求（endOfBatch=true）时，批量处理
    if (this.events.size() >= ReadOnlyServiceImpl.this.raftOptions.getApplyBatch() || endOfBatch) {
        executeReadIndexEvents(this.events);
        this.events.clear();
    }
}
```

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl#executeReadIndexEvents

```java
private void executeReadIndexEvents(final List<ReadIndexEvent> events) {
  	//处理只读ReadIndex事件
    if (events.isEmpty()) {
        return;
    }
  	//创建ReadIndexRequest
    final ReadIndexRequest.Builder rb = ReadIndexRequest.newBuilder() //
        .setGroupId(this.node.getGroupId()) //
        .setServerId(this.node.getServerId().toString());

    final List<ReadIndexState> states = new ArrayList<>(events.size());

    for (final ReadIndexEvent event : events) {
        rb.addEntries(ZeroByteStringHelper.wrap(event.requestContext.get()));
        states.add(new ReadIndexState(event.requestContext, event.done, event.startTime));
    }
    final ReadIndexRequest request = rb.build();
		//交由NodeImpl处理ReadIndex请求
    this.node.handleReadIndexRequest(request, new ReadIndexResponseClosure(states, request));
}	
```

com.alipay.sofa.jraft.core.NodeImpl#handleReadIndexRequest

```java
public void handleReadIndexRequest(final ReadIndexRequest request, final RpcResponseClosure<ReadIndexResponse> done) {
    final long startMs = Utils.monotonicMs();
    this.readLock.lock();//获取读锁
    try {
        switch (this.state) {
            case STATE_LEADER:
                readLeader(request, ReadIndexResponse.newBuilder(), done);
                break;
            case STATE_FOLLOWER:
                readFollower(request, done);
                break;
            case STATE_TRANSFERRING:
                done.run(new Status(RaftError.EBUSY, "Is transferring leadership."));
                break;
            default:
                done.run(new Status(RaftError.EPERM, "Invalid state for readIndex: %s.", this.state));
                break;
        }
    } finally {
        this.readLock.unlock();
        this.metrics.recordLatency("handle-read-index", Utils.monotonicMs() - startMs);
        this.metrics.recordSize("handle-read-index-entries", request.getEntriesCount());
    }
}
```

#### Leader节点处理

com.alipay.sofa.jraft.core.NodeImpl#readLeader

```java
private void readLeader(final ReadIndexRequest request, final ReadIndexResponse.Builder respBuilder,
                        final RpcResponseClosure<ReadIndexResponse> closure) {
    final int quorum = getQuorum(); //获取集群节点个数一半+1
    if (quorum <= 1) { //当前只有一个节点
        // Only one peer, fast path.
        respBuilder.setSuccess(true) //
            //从投票箱获取最新提交的索引lastCommittedIndex
            .setIndex(this.ballotBox.getLastCommittedIndex());
        closure.setResponse(respBuilder.build());
        closure.run(Status.OK());//执行回调ReadIndexResponseClosure
        return;
    }
 		//从投票箱获取最新提交的索引lastCommittedIndex
    final long lastCommittedIndex = this.ballotBox.getLastCommittedIndex();
    //当前任期内leader未曾提交任何数据，拒绝只读请求
    if (this.logManager.getTerm(lastCommittedIndex) != this.currTerm) {
        closure //执行回调ReadIndexResponseClosure
            .run(new Status(
                RaftError.EAGAIN,
                "ReadIndex request rejected because leader has not committed any log entry at its term, logIndex=%d, currTerm=%d.",
                lastCommittedIndex, this.currTerm));
        return;
    }
 		 //设置最新提交的索引
    respBuilder.setIndex(lastCommittedIndex); 
   
		//不为空说明请求来自follwer或者learner
    if (request.getPeerId() != null) {
        final PeerId peer = new PeerId();
        peer.parse(request.getServerId());
        //验证peerId的合法性
        if (!this.conf.contains(peer) && !this.conf.containsLearner(peer)) {
            closure
                .run(new Status(RaftError.EPERM, "Peer %s is not in current configuration: %s.", peer, this.conf));
            return;
        }
    }
		//默认ReadOnlySafe，还支持ReadOnlyLeaseBased
    ReadOnlyOption readOnlyOpt = this.raftOptions.getReadOnlyOptions();
    //当ReadOnlyOption配置为ReadOnlyLeaseBased时,确认Leader租约是否有效,即检查Heartbeat间隔是否小于 election timeout时间
    if (readOnlyOpt == ReadOnlyOption.ReadOnlyLeaseBased && !isLeaderLeaseValid()) {
        // 如果租约超时，改为ReadIndex
        readOnlyOpt = ReadOnlyOption.ReadOnlySafe;
    }

    switch (readOnlyOpt) {
        case ReadOnlySafe: //ReadIndex Read
        		//获取follower节点
            final List<PeerId> peers = this.conf.getConf().getPeers();
            Requires.requireTrue(peers != null && !peers.isEmpty(), "Empty peers");
        		//向follower发送心跳的回调
            final ReadIndexHeartbeatResponseClosure heartbeatDone = new ReadIndexHeartbeatResponseClosure(closure,respBuilder, quorum, peers.size());
            // 向follower发送心跳，确保当前leader仍是leader
            for (final PeerId peer : peers) {
                if (peer.equals(this.serverId)) {
                    continue;
                }
                this.replicatorGroup.sendHeartbeat(peer, heartbeatDone);
            }
            break; 
        case ReadOnlyLeaseBased: //Lease Read,保证CPU时钟同步
            // Responses to followers and local node.
            respBuilder.setSuccess(true);
            closure.setResponse(respBuilder.build());
            closure.run(Status.OK());//执行回调ReadIndexResponseClosure
            break;
    }
}
```

##### ReadOnlySafe

向follower发送心跳请求，确保当前leader仍然有效

com.alipay.sofa.jraft.core.NodeImpl.ReadIndexHeartbeatResponseClosure#run

```java
public synchronized void run(final Status status) {//心跳回调
    if (this.isDone) {
        return;
    }
    if (status.isOk() && getResponse().getSuccess()) { //心跳响应成功
        this.ackSuccess++; //成功累积次数
    } else { //响应失败
        this.ackFailures++; //失败累积次数
    }
    // +1代表包含了leader节点
    if (this.ackSuccess + 1 >= this.quorum) { 
      //超过一半节点响应成功，说明leader仍然是leaeder
        this.respBuilder.setSuccess(true);
        this.closure.setResponse(this.respBuilder.build());
        this.closure.run(Status.OK()); //执行回调ReadIndexResponseClosure
        this.isDone = true;
    } else if (this.ackFailures >= this.failPeersThreshold) {
       //一半响应失败
        this.respBuilder.setSuccess(false);
        this.closure.setResponse(this.respBuilder.build());
        this.closure.run(Status.OK());
        this.isDone = true;
    }
}
```

##### ReadOnlyLeaseBased

```java
//默认ReadOnlySafe
ReadOnlyOption readOnlyOpt = this.raftOptions.getReadOnlyOptions();
if (readOnlyOpt == ReadOnlyOption.ReadOnlyLeaseBased && !isLeaderLeaseValid()) {
    //如果Leader租约无效在改成ReadOnlySafe
    readOnlyOpt = ReadOnlyOption.ReadOnlySafe;
}
 case ReadOnlyLeaseBased:
      // Responses to followers and local node.
      respBuilder.setSuccess(true);
      closure.setResponse(respBuilder.build());
			//直接调用线性读回调
      closure.run(Status.OK());
      break;
```

```java
private boolean isLeaderLeaseValid() {
    final long monotonicNowMs = Utils.monotonicMs();
    if (checkLeaderLease(monotonicNowMs)) {
        return true;
    }
  		//检查过半节点的心跳是否正常
    checkDeadNodes0(this.conf.getConf().getPeers(), monotonicNowMs, false, null);
    return checkLeaderLease(monotonicNowMs);
}
```

```java
private boolean checkLeaderLease(final long monotonicNowMs) {
  	//Leader在追加请求时会修改lastLeaderTimestamp
    return monotonicNowMs - this.lastLeaderTimestamp < this.options.getLeaderLeaseTimeoutMs(); //默认900ms
}
```

```java
private boolean checkDeadNodes0(final List<PeerId> peers, final long monotonicNowMs, final boolean checkReplicator,
                                final Configuration deadNodes) {
  	//检查过半节点的心跳是否正常
    final int leaderLeaseTimeoutMs = this.options.getLeaderLeaseTimeoutMs();
    int aliveCount = 0;
    long startLease = Long.MAX_VALUE;
    for (final PeerId peer : peers) {
        if (peer.equals(this.serverId)) {
            aliveCount++;
            continue;
        }
        if (checkReplicator) {
            checkReplicator(peer);
        }
        final long lastRpcSendTimestamp = this.replicatorGroup.getLastRpcSendTimestamp(peer);
        if (monotonicNowMs - lastRpcSendTimestamp <= leaderLeaseTimeoutMs) {
            aliveCount++;
            if (startLease > lastRpcSendTimestamp) {
                startLease = lastRpcSendTimestamp;
            }
            continue;
        }
        if (deadNodes != null) {
            deadNodes.addPeer(peer);
        }
    }
    if (aliveCount >= peers.size() / 2 + 1) {
        updateLastLeaderTimestamp(startLease);
        return true;
    }
    return false;
}
```

##### 线性读回调

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl.ReadIndexResponseClosure#run

```java
public void run(final Status status) { //过半节点认返回心跳成功，执行此方法
    if (!status.isOk()) { //失败
        notifyFail(status); //执行用户自定义回调
        return;
    }
    //心跳响应成功
    final ReadIndexResponse readIndexResponse = getResponse();
    if (!readIndexResponse.getSuccess()) {
        notifyFail(new Status(-1, "Fail to run ReadIndex task, maybe the leader stepped down."));
        return;
    }
    // Success
    final ReadIndexStatus readIndexStatus = new ReadIndexStatus(this.states, this.request,
        readIndexResponse.getIndex());
    for (final ReadIndexState state : this.states) {
        // 设置处理ReadIndex请求时，Leader上最新的commitIndex
        state.setIndex(readIndexResponse.getIndex());
    }

    boolean doUnlock = true;
    ReadOnlyServiceImpl.this.lock.lock();
    try {
        if (readIndexStatus.isApplied(ReadOnlyServiceImpl.this.fsmCaller.getLastAppliedIndex())) {
          	//已提交的日志项都已应用到状态机
            ReadOnlyServiceImpl.this.lock.unlock();
            doUnlock = false;
            notifySuccess(readIndexStatus); //用户自定义回调
        } else {
            //leader最新提交的日志项尚未应用到状态机，pendingNotifyStatus对其暂存
            ReadOnlyServiceImpl.this.pendingNotifyStatus
            .computeIfAbsent(readIndexStatus.getIndex(), k -> new ArrayList<>(10)) //
            .add(readIndexStatus);
        }
    } finally {
        if (doUnlock) { //释放锁的标志
            ReadOnlyServiceImpl.this.lock.unlock();
        }
    }
}
```

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl#notifySuccess

```java
private void notifySuccess(final ReadIndexStatus status) {
    final long nowMs = Utils.monotonicMs();
    final List<ReadIndexState> states = status.getStates();
    final int taskCount = states.size();
    for (int i = 0; i < taskCount; i++) {
        final ReadIndexState task = states.get(i);
        final ReadIndexClosure done = task.getDone(); // stack copy
        if (done != null) {
            this.nodeMetrics.recordLatency("read-index", nowMs - task.getStartTimeMs());
            done.setResult(task.getIndex(), task.getRequestContext().get());
            done.run(Status.OK()); //用户自定义的回调，交由状态机处理
        }
    }
}
```

##### 唤醒阻塞ReadIndex请求

ReadIndex请求必须等待提交的索引都应用到状态机

com.alipay.sofa.jraft.core.FSMCallerImpl#notifyLastAppliedIndexUpdated

```java
private void notifyLastAppliedIndexUpdated(final long lastAppliedIndex) { //应用到状态机之后，调用此方法，默认ReadOnlyServiceImpl的
    for (final LastAppliedLogIndexListener listener : this.lastAppliedLogIndexListeners) {
      	//定时调用或者当lastAppliedIndex被更新的时候调用
        listener.onApplied(lastAppliedIndex);
    }
}
```

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl#init

```java
//ReadOnlyServiceImpl启动时，定时调用
this.scheduledExecutorService.scheduleAtFixedRate(() -> onApplied(this.fsmCaller.getLastAppliedIndex()),
    this.raftOptions.getMaxElectionDelayMs(), this.raftOptions.getMaxElectionDelayMs(), TimeUnit.MILLISECONDS); 
```

com.alipay.sofa.jraft.core.ReadOnlyServiceImpl#onApplied

```java
public void onApplied(final long appliedIndex) {
    // 处理暂存的ReadIndex请求
    List<ReadIndexStatus> pendingStatuses = null;
    this.lock.lock();
    try {
        if (this.pendingNotifyStatus.isEmpty()) { //为空直接返回
            return;
        }
        //查找所有小于等于appliedIndex的status
        final Map<Long, List<ReadIndexStatus>> statuses = this.pendingNotifyStatus.headMap(appliedIndex, true);
        if (statuses != null) {
            pendingStatuses = new ArrayList<>(statuses.size() << 1);

            final Iterator<Map.Entry<Long, List<ReadIndexStatus>>> it = statuses.entrySet().iterator();
            while (it.hasNext()) {
                final Map.Entry<Long, List<ReadIndexStatus>> entry = it.next();
                pendingStatuses.addAll(entry.getValue());
               //从pendingNotifyStatus移除
                it.remove(); 
            }

        }

        /*
         * Remaining pending statuses are notified by error if it is presented.
         * When the node is in error state, consider following situations:
         * 1. If commitIndex > appliedIndex, then all pending statuses should be notified by error status.
         * 2. When commitIndex == appliedIndex, there will be no more pending statuses.
         */
        if (this.error != null) { 
            resetPendingStatusError(this.error.getStatus());
        }
    } finally {
        this.lock.unlock();
        if (pendingStatuses != null && !pendingStatuses.isEmpty()) {
            for (final ReadIndexStatus status : pendingStatuses) {
                notifySuccess(status); //执行用户自定义回调
            }
        }
    }
}
```



#### Follower节点处理

com.alipay.sofa.jraft.core.NodeImpl#readFollower

```java
private void readFollower(final ReadIndexRequest request, final RpcResponseClosure<ReadIndexResponse> closure) {
  	//尚未知晓集群中的Leader
    if (this.leaderId == null || this.leaderId.isEmpty()) {
        closure.run(new Status(RaftError.EPERM, "No leader at term %d.", this.currTerm));
        return;
    }
    // 发送请求给Leader
    final ReadIndexRequest newRequest = ReadIndexRequest.newBuilder() //
        .mergeFrom(request) //
        .setPeerId(this.leaderId.toString()) //
        .build();
    this.rpcService.readIndex(this.leaderId.getEndpoint(), newRequest, -1, closure);
}
```

# 扩容

## 添加follower节点

com.alipay.sofa.jraft.core.NodeImpl#addPeer

```java
public void addPeer(final PeerId peer, final Closure done) {//添加follower节点
    Requires.requireNonNull(peer, "Null peer");
    this.writeLock.lock();
    try {
        Requires.requireTrue(!this.conf.getConf().contains(peer), "Peer already exists in current configuration");
				//创建新的配置
        final Configuration newConf = new Configuration(this.conf.getConf());
        newConf.addPeer(peer); //新增的节点
        unsafeRegisterConfChange(this.conf.getConf(), newConf, done);
    } finally {
        this.writeLock.unlock();
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl#unsafeRegisterConfChange

```java
private void unsafeRegisterConfChange(final Configuration oldConf, final Configuration newConf, final Closure done) {

    Requires.requireTrue(newConf.isValid(), "Invalid new conf: %s", newConf);
    // The new conf entry(will be stored in log manager) should be valid
    Requires.requireTrue(new ConfigurationEntry(null, newConf, oldConf).isValid(), "Invalid conf entry: %s",newConf);

    if (this.state != State.STATE_LEADER) { //判断是否仍是leader节点
        LOG.warn("Node {} refused configuration changing as the state={}.", getNodeId(), this.state);
        if (done != null) {
            final Status status = new Status();
            if (this.state == State.STATE_TRANSFERRING) {
                status.setError(RaftError.EBUSY, "Is transferring leadership.");
            } else {
                status.setError(RaftError.EPERM, "Not leader");
            }
            Utils.runClosureInThread(done, status);
        }
        return;
    }
  	//STAGE_CATCHING_UP -> STAGE_JOINT->STAGE_STABLE
    if (this.confCtx.isBusy()) { //检查当前的配置变更
        LOG.warn("Node {} refused configuration concurrent changing.", getNodeId());
        if (done != null) {
            Utils.runClosureInThread(done, new Status(RaftError.EBUSY, "Doing another configuration change."));
        }
        return;
    }
  
    // Return immediately when the new peers equals to current configuration
    if (this.conf.getConf().equals(newConf)) { //配置没有发生变更
        Utils.runClosureInThread(done);
        return;
    }
    this.confCtx.start(oldConf, newConf, done);
}
```

com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#start

```java
void start(final Configuration oldConf, final Configuration newConf, final Closure done) {
    if (isBusy()) {
        if (done != null) {
            Utils.runClosureInThread(done, new Status(RaftError.EBUSY, "Already in busy stage."));
        }
        throw new IllegalStateException("Busy stage");
    }
    if (this.done != null) {
        if (done != null) {
            Utils.runClosureInThread(done, new Status(RaftError.EINVAL, "Already have done closure."));
        }
        throw new IllegalArgumentException("Already have done closure");
    }
    this.done = done;
    this.stage = Stage.STAGE_CATCHING_UP; //修改stage为STAGE_CATCHING_UP
    this.oldPeers = oldConf.listPeers();
    this.newPeers = newConf.listPeers();
    this.oldLearners = oldConf.listLearners();
    this.newLearners = newConf.listLearners();
    final Configuration adding = new Configuration();
    final Configuration removing = new Configuration();
    newConf.diff(oldConf, adding, removing);
    this.nchanges = adding.size() + removing.size();

    addNewLearners();
    if (adding.isEmpty()) {
        nextStage();
        return;
    }
    addNewPeers(adding); 	//添加follower节点
}
```

com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#addNewPeers

```java
private void addNewPeers(final Configuration adding) {//添加follower节点
    this.addingPeers = adding.listPeers();
    LOG.info("Adding peers: {}.", this.addingPeers);
    for (final PeerId newPeer : this.addingPeers) {
        if (!this.node.replicatorGroup.addReplicator(newPeer)) { //同步数据
            LOG.error("Node {} start the replicator failed, peer={}.", this.node.getNodeId(), newPeer);
            onCaughtUp(this.version, newPeer, false);
            return;
        }
        final OnCaughtUp caughtUp = new OnCaughtUp(this.node, this.node.currTerm, newPeer, this.version);
        final long dueTime = Utils.nowMs() + this.node.options.getElectionTimeoutMs();
      	//catchupMargin:默认1000，主从同步的差距小于1000时，将变更的配置同步至集群中的其他节点
        if (!this.node.replicatorGroup.waitCaughtUp(newPeer, this.node.options.getCatchupMargin(), dueTime, caughtUp)) {
            LOG.error("Node {} waitCaughtUp, peer={}.", this.node.getNodeId(), newPeer);
            onCaughtUp(this.version, newPeer, false);
            return;
        }
    }
}
```

com.alipay.sofa.jraft.core.ReplicatorGroupImpl#waitCaughtUp

```java
public boolean waitCaughtUp(final PeerId peer, final long maxMargin, final long dueTime, final CatchUpClosure done) {
    final ThreadId rid = this.replicatorMap.get(peer);
    if (rid == null) {
        return false;
    }
		//等待同步差距小于1000
    Replicator.waitForCaughtUp(rid, maxMargin, dueTime, done);
    return true;
}
```

com.alipay.sofa.jraft.core.Replicator#waitForCaughtUp

```java
public static void waitForCaughtUp(final ThreadId id, final long maxMargin, final long dueTime,
                                   final CatchUpClosure done) {
    final Replicator r = (Replicator) id.lock();

    if (r == null) {
        Utils.runClosureInThread(done, new Status(RaftError.EINVAL, "No such replicator"));
        return;
    }
    try {
        if (r.catchUpClosure != null) { //之前的wait_for_caught_up尚未完成
            LOG.error("Previous wait_for_caught_up is not over");
            Utils.runClosureInThread(done, new Status(RaftError.EINVAL, "Duplicated call"));
            return;
        }
        done.setMaxMargin(maxMargin); //默认1000
        if (dueTime > 0) {
            done.setTimer(r.timerManager.schedule(() -> onCatchUpTimedOut(id), dueTime - Utils.nowMs(),TimeUnit.MILLISECONDS)); //设置延迟任务，触发超时事件
        }
        r.catchUpClosure = done;
    } finally {
        id.unlock();
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl.OnCaughtUp#run

```java
public void run(final Status status) {//主从同步的差距小于1000时，会执行此方法
    this.node.onCaughtUp(this.peer, this.term, this.version, status);
}
```

com.alipay.sofa.jraft.core.NodeImpl#onCaughtUp

```java
private void onCaughtUp(final PeerId peer, final long term, final long version, final Status st) { //leader每次给follower同步完数据之后，都会执行此方法
    this.writeLock.lock();
    try {
        // check current_term and state to avoid ABA problem
        if (term != this.currTerm && this.state != State.STATE_LEADER) { //发生了leader变更
            // term has changed and nothing should be done, otherwise there will be
            // an ABA problem.
            return;
        }
        if (st.isOk()) { //追赶成功
            // Caught up successfully
            this.confCtx.onCaughtUp(version, peer, true);
            return;
        }
        // 如果节点仍然存活，重试
        if (st.getCode() == RaftError.ETIMEDOUT.getNumber()
            && Utils.monotonicMs() - this.replicatorGroup.getLastRpcSendTimestamp(peer) <= this.options .getElectionTimeoutMs()) {
            LOG.debug("Node {} waits peer {} to catch up.", getNodeId(), peer);
            final OnCaughtUp caughtUp = new OnCaughtUp(this, term, peer, version);
            final long dueTime = Utils.nowMs() + this.options.getElectionTimeoutMs();
            if (this.replicatorGroup.waitCaughtUp(peer, this.options.getCatchupMargin(), dueTime, caughtUp)) {
                return;
            }
            LOG.warn("Node {} waitCaughtUp failed, peer={}.", getNodeId(), peer);
        }
        LOG.warn("Node {} caughtUp failed, status={}, peer={}.", getNodeId(), st, peer);
        this.confCtx.onCaughtUp(version, peer, false);
    } finally {
        this.writeLock.unlock();
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#onCaughtUp

```java
void onCaughtUp(final long version, final PeerId peer, final boolean success) {
    if (version != this.version) {
        LOG.warn("Ignore onCaughtUp message, mismatch configuration context version, expect {}, but is {}.",
            this.version, version);
        return;
    }
    Requires.requireTrue(this.stage == Stage.STAGE_CATCHING_UP, "Stage is not in STAGE_CATCHING_UP");
    if (success) { //追赶成功
        this.addingPeers.remove(peer);
        if (this.addingPeers.isEmpty()) {
            nextStage();
            return;
        }
        return;
    }
    LOG.warn("Node {} fail to catch up peer {} when trying to change peers from {} to {}.",
        this.node.getNodeId(), peer, this.oldPeers, this.newPeers);
    reset(new Status(RaftError.ECATCHUP, "Peer %s failed to catch up.", peer));
}
```

com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#nextStage

```java
void nextStage() {
    Requires.requireTrue(isBusy(), "Not in busy stage");
    switch (this.stage) {
        case STAGE_CATCHING_UP:
            if (this.nchanges > 1) {
                this.stage = Stage.STAGE_JOINT; //修改状态
                this.node.unsafeApplyConfiguration(new Configuration(this.newPeers, this.newLearners),
                    new Configuration(this.oldPeers), false);
                return;
            }
            // Skip joint consensus since only one peers has been changed here. Make
            // it a one-stage change to be compatible with the legacy
            // implementation.
        case STAGE_JOINT:
            this.stage = Stage.STAGE_STABLE;
            this.node.unsafeApplyConfiguration(new Configuration(this.newPeers, this.newLearners), null, false);
            break;
        case STAGE_STABLE:
            final boolean shouldStepDown = !this.newPeers.contains(this.node.serverId);
            reset(new Status());
            if (shouldStepDown) {
                this.node.stepDown(this.node.currTerm, true, new Status(RaftError.ELEADERREMOVED,
                    "This node was removed."));
            }
            break;
        case STAGE_NONE:
            // noinspection ConstantConditions
            Requires.requireTrue(false, "Can't reach here");
            break;
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl#unsafeApplyConfiguration

```java
private void unsafeApplyConfiguration(final Configuration newConf, final Configuration oldConf,
                                      final boolean leaderStart) {
    Requires.requireTrue(this.confCtx.isBusy(), "ConfigurationContext is not busy");
    final LogEntry entry = new LogEntry(EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION);
    entry.setId(new LogId(0, this.currTerm));
    entry.setPeers(newConf.listPeers());
    entry.setLearners(newConf.listLearners());
    if (oldConf != null) {
        entry.setOldPeers(oldConf.listPeers());
        entry.setOldLearners(oldConf.listLearners());
    }
    final ConfigurationChangeDone configurationChangeDone = new ConfigurationChangeDone(this.currTerm, leaderStart);
    // Use the new_conf to deal the quorum of this very log
    if (!this.ballotBox.appendPendingTask(newConf, oldConf, configurationChangeDone)) {
        Utils.runClosureInThread(configurationChangeDone, new Status(RaftError.EINTERNAL, "Fail to append task."));
        return;
    }
    final List<LogEntry> entries = new ArrayList<>();
    entries.add(entry);
  	//写磁盘、同步至从节点
    this.logManager.appendEntries(entries, new LeaderStableClosure(entries));
    checkAndSetConfiguration(false);
}
```

## 添加Learner节点

com.alipay.sofa.jraft.rpc.impl.cli.AddLearnersRequestProcessor#processRequest0

```java
protected Message processRequest0(final CliRequestContext ctx, final AddLearnersRequest request, final RpcRequestClosure done) {
    final List<PeerId> oldLearners = ctx.node.listLearners();
    final List<PeerId> addingLearners = new ArrayList<>(request.getLearnersCount());

    for (final String peerStr : request.getLearnersList()) { //learner节点可以批量添加
        final PeerId peer = new PeerId();
        if (!peer.parse(peerStr)) {
            return RpcResponseFactory.newResponse(RaftError.EINVAL, "Fail to parse peer id %", peerStr);
        }
        addingLearners.add(peer);
    }

    LOG.info("Receive AddLearnersRequest to {} from {}, adding {}.", ctx.node.getNodeId(),
        done.getRpcCtx().getRemoteAddress(), addingLearners);
  	
  	//添加learner节点
    ctx.node.addLearners(addingLearners, status -> {
        if (!status.isOk()) {
            done.run(status);
        } else {
            final LearnersOpResponse.Builder rb = LearnersOpResponse.newBuilder();

            for (final PeerId peer : oldLearners) {
                rb.addOldLearners(peer.toString());
                rb.addNewLearners(peer.toString());
            }

            for (final PeerId peer : addingLearners) {
                if (!oldLearners.contains(peer)) {
                    rb.addNewLearners(peer.toString());
                }
            }

            done.sendResponse(rb.build());
        }
    });

    return null;
}
```

com.alipay.sofa.jraft.core.NodeImpl#addLearners

```java
public void addLearners(final List<PeerId> learners, final Closure done) {//与添加Follower节点类似
    checkPeers(learners);
    this.writeLock.lock();
    try {
        final Configuration newConf = new Configuration(this.conf.getConf());
        for (final PeerId peer : learners) {
            newConf.addLearner(peer);
        }
        unsafeRegisterConfChange(this.conf.getConf(), newConf, done);
    } finally {
        this.writeLock.unlock();
    }
}
```

com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#start

```java
void start(final Configuration oldConf, final Configuration newConf, final Closure done) {
    if (isBusy()) {
        if (done != null) {
            Utils.runClosureInThread(done, new Status(RaftError.EBUSY, "Already in busy stage."));
        }
        throw new IllegalStateException("Busy stage");
    }
    if (this.done != null) {
        if (done != null) {
            Utils.runClosureInThread(done, new Status(RaftError.EINVAL, "Already have done closure."));
        }
        throw new IllegalArgumentException("Already have done closure");
    }
    this.done = done;
    this.stage = Stage.STAGE_CATCHING_UP;
    this.oldPeers = oldConf.listPeers();
    this.newPeers = newConf.listPeers();
    this.oldLearners = oldConf.listLearners();
    this.newLearners = newConf.listLearners();
    final Configuration adding = new Configuration();
    final Configuration removing = new Configuration();
    newConf.diff(oldConf, adding, removing);
    this.nchanges = adding.size() + removing.size();

    addNewLearners(); //添加Learner节点,同步数据
    if (adding.isEmpty()) { //无Follower节点添加
        nextStage(); //直接同步变更的节点配置信息
        return;
    }
    addNewPeers(adding);
}
```

com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#addNewLearners

```java
private void addNewLearners() {
    final Set<PeerId> addingLearners = new HashSet<>(this.newLearners);
    addingLearners.removeAll(this.oldLearners);
    LOG.info("Adding learners: {}.", this.addingPeers);
    for (final PeerId newLearner : addingLearners) {
        //创建Replicator,同步数据
        if (!this.node.replicatorGroup.addReplicator(newLearner, ReplicatorType.Learner)) {
            LOG.error("Node {} start the learner replicator failed, peer={}.", this.node.getNodeId(),
                newLearner);
        }
    }
}
```

# Leader转让

### Leader节点处理转让

com.alipay.sofa.jraft.core.NodeImpl#transferLeadershipTo

```java
public Status transferLeadershipTo(final PeerId peer) {
    Requires.requireNonNull(peer, "Null peer");
    this.writeLock.lock();
    try {
      	//当前节点必须是leader节点
        if (this.state != State.STATE_LEADER) { 
            LOG.warn("Node {} can't transfer leadership to peer {} as it is in state {}.", getNodeId(), peer,
                this.state);
            return new Status(this.state == State.STATE_TRANSFERRING ? RaftError.EBUSY : RaftError.EPERM,
                    "Not a leader");
        }
        //配置发生了变更
        if (this.confCtx.isBusy()) { 
            LOG.warn(
                "Node {} refused to transfer leadership to peer {} when the leader is changing the configuration.",
                getNodeId(), peer);
            return new Status(RaftError.EBUSY, "Changing the configuration");
        }

        PeerId peerId = peer.copy();
        // if peer_id is ANY_PEER(0.0.0.0:0:0), the peer with the largest
        // last_log_id will be selected.
        if (peerId.equals(PeerId.ANY_PEER)) {
            LOG.info("Node {} starts to transfer leadership to any peer.", getNodeId());
            if ((peerId = this.replicatorGroup.findTheNextCandidate(this.conf)) == null) {
                return new Status(-1, "Candidate not found for any peer");
            }
        }
      	//转让给本节点
        if (peerId.equals(this.serverId)) {
            LOG.info("Node {} transferred leadership to self.", this.serverId);
            return Status.OK();
        }
        //非法节点
        if (!this.conf.contains(peerId)) {
            LOG.info("Node {} refused to transfer leadership to peer {} as it is not in {}.", getNodeId(), peer,
                this.conf);
            return new Status(RaftError.EINVAL, "Not in current configuration");
        }

      	//leader节点最新的日志索引
        final long lastLogIndex = this.logManager.getLastLogIndex();
      	//转让,设置Replicator的timeoutNowIndex等于lastLogIndex，同步进度追赶上leader时，给转让节点发送TimeoutNowRequest请求
        if (!this.replicatorGroup.transferLeadershipTo(peerId, lastLogIndex)) {
            LOG.warn("No such peer {}.", peer);
            return new Status(RaftError.EINVAL, "No such peer %s", peer);
        }
        //修改为STATE_TRANSFERRING
        this.state = State.STATE_TRANSFERRING; 
        final Status status = new Status(RaftError.ETRANSFERLEADERSHIP,
            "Raft leader is transferring leadership to %s", peerId);
        onLeaderStop(status);
        LOG.info("Node {} starts to transfer leadership to peer {}.", getNodeId(), peer);
        
        //创建转让延迟任务
        final StopTransferArg stopArg = new StopTransferArg(this, this.currTerm, peerId);
        this.stopTransferArg = stopArg;
        this.transferTimer = this.timerManager.schedule(() -> onTransferTimeout(stopArg),
            this.options.getElectionTimeoutMs(), TimeUnit.MILLISECONDS);

    } finally {
        this.writeLock.unlock();
    }
    return Status.OK();
}
```

#### 转让leader角色给从节点

com.alipay.sofa.jraft.core.ReplicatorGroupImpl#transferLeadershipTo

```java
public boolean transferLeadershipTo(final PeerId peer, final long logIndex) {
    final ThreadId rid = this.replicatorMap.get(peer);
    return rid != null && Replicator.transferLeadership(rid, logIndex);
}
```



```java
public static boolean transferLeadership(final ThreadId id, final long logIndex) {
    final Replicator r = (Replicator) id.lock();
    if (r == null) {
        return false;
    }
    // dummy is unlock in _transfer_leadership
    return r.transferLeadership(logIndex);
}
```

com.alipay.sofa.jraft.core.Replicator#transferLeadership(long)

```java
private boolean transferLeadership(final long logIndex) {
    if (this.hasSucceeded && this.nextIndex > logIndex) {
        // _id is unlock in _send_timeout_now
        sendTimeoutNow(true, false);
        return true;
    }
    // Register log_index so that _on_rpc_return trigger
    // _send_timeout_now if _next_index reaches log_index
    this.timeoutNowIndex = logIndex;
    this.id.unlock();
    return true;
}
```

com.alipay.sofa.jraft.core.Replicator#sendTimeoutNow(boolean, boolean, int)

```java
private void sendTimeoutNow(final boolean unlockId, final boolean stopAfterFinish, final int timeoutMs) { //当同步的索引大于发生转让时的leader的索引，执行此方法，发送TimeoutNowRequest
    final TimeoutNowRequest.Builder rb = TimeoutNowRequest.newBuilder();
    rb.setTerm(this.options.getTerm());
    rb.setGroupId(this.options.getGroupId());
    rb.setServerId(this.options.getServerId().toString());
    rb.setPeerId(this.options.getPeerId().toString());
    try {
        if (!stopAfterFinish) {
            // This RPC is issued by transfer_leadership, save this call_id so that
            // the RPC can be cancelled by stop.
            this.timeoutNowInFly = timeoutNow(rb, false, timeoutMs);
            this.timeoutNowIndex = 0;
        } else {
            timeoutNow(rb, true, timeoutMs);
        }
    } finally {
        if (unlockId) {
            this.id.unlock();
        }
    }
}
```

#### 转让超时

com.alipay.sofa.jraft.core.NodeImpl#handleTransferTimeout

```java
private void handleTransferTimeout(final long term, final PeerId peer) {
    LOG.info("Node {} failed to transfer leadership to peer {}, reached timeout.", getNodeId(), peer);
    this.writeLock.lock();
    try {
        if (term == this.currTerm) {
            this.replicatorGroup.stopTransferLeadership(peer);
            if (this.state == State.STATE_TRANSFERRING) {
                this.fsmCaller.onLeaderStart(term);
                this.state = State.STATE_LEADER;
                this.stopTransferArg = null;
            }
        }
    } finally {
        this.writeLock.unlock();
    }
}
```

## Follower接收转让请求

com.alipay.sofa.jraft.core.NodeImpl#handleTimeoutNowRequest

```java
public Message handleTimeoutNowRequest(final TimeoutNowRequest request, final RpcRequestClosure done) {
    boolean doUnlock = true;
    this.writeLock.lock();
    try {
      	//1、term没有发生变更
        if (request.getTerm() != this.currTerm) {
            final long savedCurrTerm = this.currTerm;
            if (request.getTerm() > this.currTerm) {
                stepDown(request.getTerm(), false, new Status(RaftError.EHIGHERTERMREQUEST,
                    "Raft node receives higher term request"));
            }
            LOG.info("Node {} received TimeoutNowRequest from {} while currTerm={} didn't match requestTerm={}.",
                getNodeId(), request.getPeerId(), savedCurrTerm, request.getTerm());
            return TimeoutNowResponse.newBuilder() //
                .setTerm(this.currTerm) //
                .setSuccess(false) //
                .build();
        }
      	//2、转让的节点必须是follower类型的节点
        if (this.state != State.STATE_FOLLOWER) {
            LOG.info("Node {} received TimeoutNowRequest from {}, while state={}, term={}.", getNodeId(),
                request.getServerId(), this.state, this.currTerm);
            return TimeoutNowResponse.newBuilder() //
                .setTerm(this.currTerm) //
                .setSuccess(false) //
                .build();
        }
				//3、返回响应
        final long savedTerm = this.currTerm;
        final TimeoutNowResponse resp = TimeoutNowResponse.newBuilder() //
            .setTerm(this.currTerm + 1) //
            .setSuccess(true) //
            .build();
        // Parallelize response and election
        done.sendResponse(resp);
        doUnlock = false;
        //4、触发选举 
        electSelf();
        LOG.info("Node {} received TimeoutNowRequest from {}, term={}.", getNodeId(), request.getServerId(), savedTerm);
    } finally {
        if (doUnlock) {
            this.writeLock.unlock();
        }
    }
    return null;
}
```

# 总结

1、设置不同的线程池，避免了不同类型请求的相互影响

2、leader持久化日志和向follower发送日志是并行的

3、leader向各个follower发送日志是相互独立且并发的

4、基于Disruptor，实现事件异步化

5、批量化，批量任务的提交、批量网络发送、批量IO写入、状态机的批量应用

6、复制流水线，利用 Pipeline 的通信方式来提高日志复制的效率

