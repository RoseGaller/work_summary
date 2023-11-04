# 启动

embed/etcd.go

```go
func startEtcd(cfg *config) (<-chan struct{}, error) {
	urlsmap, token, err := getPeerURLsMapAndToken(cfg, "etcd")
	if err != nil {
		return nil, fmt.Errorf("error setting up initial cluster: %v", err)
	}

	if !cfg.peerTLSInfo.Empty() {
		plog.Infof("peerTLS: %s", cfg.peerTLSInfo)
	}
	plns := make([]net.Listener, 0)
	for _, u := range cfg.lpurls {//监听集群内其他节点的请求
		if u.Scheme == "http" && !cfg.peerTLSInfo.Empty() {
			plog.Warningf("The scheme of peer url %s is http while peer key/cert files are presented. Ignored peer key/cert files.", u.String())
		}
		var l net.Listener
		l, err = rafthttp.NewListener(u, cfg.peerTLSInfo)
		if err != nil {
			return nil, err
		}

		urlStr := u.String()
		plog.Info("listening for peers on ", urlStr)
		defer func() {
			if err != nil {
				l.Close()
				plog.Info("stopping listening for peers on ", urlStr)
			}
		}()
		plns = append(plns, l)
	}

	if !cfg.clientTLSInfo.Empty() {
		plog.Infof("clientTLS: %s", cfg.clientTLSInfo)
	}
	clns := make([]net.Listener, 0)
	for _, u := range cfg.lcurls { //监听客户端发送的请求
		if u.Scheme == "http" && !cfg.clientTLSInfo.Empty() {
			plog.Warningf("The scheme of client url %s is http while client key/cert files are presented. Ignored client key/cert files.", u.String())
		}
		var l net.Listener
		l, err = net.Listen("tcp", u.Host)
		if err != nil {
			return nil, err
		}

		if fdLimit, err := runtimeutil.FDLimit(); err == nil {
			if fdLimit <= reservedInternalFDNum {
				plog.Fatalf("file descriptor limit[%d] of etcd process is too low, and should be set higher than %d to ensure internal usage", fdLimit, reservedInternalFDNum)
			}
			l = transport.LimitListener(l, int(fdLimit-reservedInternalFDNum))
		}

		// Do not wrap around this listener if TLS Info is set.
		// HTTPS server expects TLS Conn created by TLSListener.
		l, err = transport.NewKeepAliveListener(l, u.Scheme, cfg.clientTLSInfo)
		if err != nil {
			return nil, err
		}

		urlStr := u.String()
		plog.Info("listening for client requests on ", urlStr)
		defer func() {
			if err != nil {
				l.Close()
				plog.Info("stopping listening for client requests on ", urlStr)
			}
		}()
		clns = append(clns, l)
	}

	var v3l net.Listener
	if cfg.v3demo { //监听grpc请求
		v3l, err = net.Listen("tcp", cfg.gRPCAddr)
		if err != nil {
			plog.Fatal(err)
		}
		plog.Infof("listening for client rpc on %s", cfg.gRPCAddr)
	}
	//创建ServerConfig实例
	srvcfg := &etcdserver.ServerConfig{
		Name:                    cfg.name,
		ClientURLs:              cfg.acurls, //监听客户端
		PeerURLs:                cfg.apurls, //监听节点
		DataDir:                 cfg.dir, //存储目录
		DedicatedWALDir:         cfg.walDir,
		SnapCount:               cfg.snapCount, //快照间隔数
		MaxSnapFiles:            cfg.maxSnapFiles,
		MaxWALFiles:             cfg.maxWalFiles,
		InitialPeerURLsMap:      urlsmap,
		InitialClusterToken:     token,
		DiscoveryURL:            cfg.durl,
		DiscoveryProxy:          cfg.dproxy,
		NewCluster:              cfg.isNewCluster(),
		ForceNewCluster:         cfg.forceNewCluster,
		PeerTLSInfo:             cfg.peerTLSInfo,
		TickMs:                  cfg.TickMs,
		ElectionTicks:           cfg.electionTicks(),
		V3demo:                  cfg.v3demo,
		AutoCompactionRetention: cfg.autoCompactionRetention, //自动合并
		StrictReconfigCheck:     cfg.strictReconfigCheck,
		EnablePprof:             cfg.enablePprof,
	}
	var s *etcdserver.EtcdServer
  //创建etcdServer
	s, err = etcdserver.NewServer(srvcfg)
	if err != nil {
		return nil, err
	}
  //启动etcdserver
	s.Start()
	osutil.RegisterInterruptHandler(s.Stop)

	if cfg.corsInfo.String() != "" {
		plog.Infof("cors = %s", cfg.corsInfo)
	}
	ch := &cors.CORSHandler{
		Handler: etcdhttp.NewClientHandler(s, srvcfg.ReqTimeout()),
		Info:    cfg.corsInfo,
	}
	ph := etcdhttp.NewPeerHandler(s)//处理集群内其他节点发送的请求
	// Start the peer server in a goroutine
	for _, l := range plns {
		go func(l net.Listener) {
			plog.Fatal(serveHTTP(l, ph, 5*time.Minute))
		}(l)
	}
	// Start a client server goroutine for each listen address
	for _, l := range clns { //处理客户端发送的请求
		go func(l net.Listener) {
			// read timeout does not work with http close notify
			// TODO: https://github.com/golang/go/issues/9524
			plog.Fatal(serveHTTP(l, ch, 0))
		}(l)
	}

	if cfg.v3demo {
		//处理grpc请求
		tls := &cfg.clientTLSInfo
		if cfg.clientTLSInfo.Empty() {
			tls = nil
		}
		grpcServer, err := v3rpc.Server(s, tls)
		if err != nil {
			s.Stop()
			<-s.StopNotify()
			return nil, err
		}
		go func() { plog.Fatal(grpcServer.Serve(v3l)) }()
	}

	return s.StopNotify(), nil
}
```

## 创建EtcdServer

etcdserver/server.go

```go
func NewServer(cfg *ServerConfig) (srv *EtcdServer, err error) {
	st := store.New(StoreClusterPrefix, StoreKeysPrefix) //v2版本的store 

	var (
		w  *wal.WAL//负责预写入日志
		n  raft.Node
		s  *raft.MemoryStorage
		id types.ID
		cl *membership.RaftCluster
	)

	if cfg.MaxRequestBytes > recommendedMaxRequestBytes { //请求数据大小不能超过10M
		plog.Warningf("MaxRequestBytes %v exceeds maximum recommended size %v", cfg.MaxRequestBytes, recommendedMaxRequestBytes)
	}

	if terr := fileutil.TouchDirAll(cfg.DataDir); terr != nil {
		return nil, fmt.Errorf("cannot access data directory: %v", terr)
	}
	//是否存在wal
	haveWAL := wal.Exist(cfg.WALDir())
  //创建快照目录
	if err = fileutil.TouchDirAll(cfg.SnapDir()); err != nil {
		plog.Fatalf("create snapshot directory error: %v", err)
	}
  //创建Snapshotter（负责保存、加载快照）
	ss := snap.New(cfg.SnapDir())
	//v3版本存储
	bepath := cfg.backendPath()
	beExist := fileutil.Exist(bepath)
	be := openBackend(cfg)

	defer func() {
		if err != nil {
			be.Close()
		}
	}()
	prt, err := rafthttp.NewRoundTripper(cfg.PeerTLSInfo, cfg.peerDialTimeout())
	if err != nil {
		return nil, err
	}
	var (
		remotes  []*membership.Member
		snapshot *raftpb.Snapshot
	)
	//NewCluster为true，表示强制创建一个新的集群
	switch {
	case !haveWAL && !cfg.NewCluster:
		if err = cfg.VerifyJoinExisting(); err != nil {
			return nil, err
		}
		cl, err = membership.NewClusterFromURLsMap(cfg.InitialClusterToken, cfg.InitialPeerURLsMap)
		if err != nil {
			return nil, err
		}
		existingCluster, gerr := GetClusterFromRemotePeers(getRemotePeerURLs(cl, cfg.Name), prt)
		if gerr != nil {
			return nil, fmt.Errorf("cannot fetch cluster info from peer urls: %v", gerr)
		}
		if err = membership.ValidateClusterAndAssignIDs(cl, existingCluster); err != nil {
			return nil, fmt.Errorf("error validating peerURLs %s: %v", existingCluster, err)
		}
		if !isCompatibleWithCluster(cl, cl.MemberByName(cfg.Name).ID, prt) {
			return nil, fmt.Errorf("incompatible with current running cluster")
		}

		remotes = existingCluster.Members()
		cl.SetID(existingCluster.ID())
		cl.SetStore(st)
		cl.SetBackend(be)
		cfg.Print()
		id, n, s, w = startNode(cfg, cl, nil)
	case !haveWAL && cfg.NewCluster:
		if err = cfg.VerifyBootstrap(); err != nil {
			return nil, err
		}
		cl, err = membership.NewClusterFromURLsMap(cfg.InitialClusterToken, cfg.InitialPeerURLsMap)
		if err != nil {
			return nil, err
		}
		m := cl.MemberByName(cfg.Name)
		if isMemberBootstrapped(cl, cfg.Name, prt, cfg.bootstrapTimeout()) {
			return nil, fmt.Errorf("member %s has already been bootstrapped", m.ID)
		}
		if cfg.ShouldDiscover() {
			var str string
			str, err = discovery.JoinCluster(cfg.DiscoveryURL, cfg.DiscoveryProxy, m.ID, cfg.InitialPeerURLsMap.String())
			if err != nil {
				return nil, &DiscoveryError{Op: "join", Err: err}
			}
			var urlsmap types.URLsMap
			urlsmap, err = types.NewURLsMap(str)
			if err != nil {
				return nil, err
			}
			if checkDuplicateURL(urlsmap) {
				return nil, fmt.Errorf("discovery cluster %s has duplicate url", urlsmap)
			}
			if cl, err = membership.NewClusterFromURLsMap(cfg.InitialClusterToken, urlsmap); err != nil {
				return nil, err
			}
		}
		cl.SetStore(st)
		cl.SetBackend(be)
		cfg.PrintWithInitial()
		id, n, s, w = startNode(cfg, cl, cl.MemberIDs())
	case haveWAL: //存在wal
    //判断Member目录是否可写
		if err = fileutil.IsDirWriteable(cfg.MemberDir()); err != nil {
			return nil, fmt.Errorf("cannot write to member directory: %v", err)
		}
   //判断WAL目录是否可写
		if err = fileutil.IsDirWriteable(cfg.WALDir()); err != nil {
			return nil, fmt.Errorf("cannot write to WAL directory: %v", err)
		}

		if cfg.ShouldDiscover() {
			plog.Warningf("discovery token ignored since a cluster has already been initialized. Valid log found at %q", cfg.WALDir())
		}
    //加载最新的快照（对快照文件按照时间排序，最新的加载出错，加载次新的快照）
		snapshot, err = ss.Load()
		if err != nil && err != snap.ErrNoSnapshot {
			return nil, err
		}
		if snapshot != nil {
			if err = st.Recovery(snapshot.Data); err != nil { //根据快照数据恢复
				plog.Panicf("recovered store from snapshot error: %v", err)
			}
			plog.Infof("recovered store from snapshot at index %d", snapshot.Metadata.Index)
			if be, err = recoverSnapshotBackend(cfg, be, *snapshot); err != nil {
				plog.Panicf("recovering backend from snapshot error: %v", err)
			}
		}
		cfg.Print()
		if !cfg.ForceNewCluster {
			id, cl, n, s, w = restartNode(cfg, snapshot) //重启
		} else {
			id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
		}
		cl.SetStore(st)
		cl.SetBackend(be)
		cl.Recover(api.UpdateCapability)
		if cl.Version() != nil && !cl.Version().LessThan(semver.Version{Major: 3}) && !beExist {
			os.RemoveAll(bepath)
			return nil, fmt.Errorf("database file (%v) of the backend is missing", bepath)
		}
	default:
		return nil, fmt.Errorf("unsupported bootstrap config")
	}

	if terr := fileutil.TouchDirAll(cfg.MemberDir()); terr != nil {
		return nil, fmt.Errorf("cannot access member directory: %v", terr)
	}

	sstats := stats.NewServerStats(cfg.Name, id.String())
	lstats := stats.NewLeaderStats(id.String())
	//心跳间隔
	heartbeat := time.Duration(cfg.TickMs) * time.Millisecond
  //创建etcdServer实例
	srv = &EtcdServer{
		readych:     make(chan struct{}),
		Cfg:         cfg,
		snapCount:   cfg.SnapCount,
		errorc:      make(chan error, 1),
		store:       st,
		snapshotter: ss,
		r: *newRaftNode(
			raftNodeConfig{
				isIDRemoved: func(id uint64) bool { return cl.IsIDRemoved(types.ID(id)) },
				Node:        n,
				heartbeat:   heartbeat,
				raftStorage: s,
				storage:     NewStorage(w, ss),
			},
		),
		id:            id,
		attributes:    membership.Attributes{Name: cfg.Name, ClientURLs: cfg.ClientURLs.StringSlice()},
		cluster:       cl,
		stats:         sstats,
		lstats:        lstats,
		SyncTicker:    time.NewTicker(500 * time.Millisecond),
		peerRt:        prt,
		reqIDGen:      idutil.NewGenerator(uint16(id), time.Now()),
		forceVersionC: make(chan struct{}),
	}
	serverID.With(prometheus.Labels{"server_id": id.String()}).Set(1)
	//v2版本
	srv.applyV2 = &applierV2store{store: srv.store, cluster: srv.cluster}

	srv.be = be //v3版本存储
  //最小存活时间
	minTTL := time.Duration((3*cfg.ElectionTicks)/2) * heartbeat
	//租约
	srv.lessor = lease.NewLessor(srv.be, int64(math.Ceil(minTTL.Seconds())))
	srv.kv = mvcc.New(srv.be, srv.lessor, &srv.consistIndex)
	if beExist {
		kvindex := srv.kv.ConsistentIndex() //幂等
		if snapshot != nil && kvindex < snapshot.Metadata.Index {
			if kvindex != 0 {
				return nil, fmt.Errorf("database file (%v index %d) does not match with snapshot (index %d).", bepath, kvindex, snapshot.Metadata.Index)
			}
			plog.Warningf("consistent index never saved (snapshot index=%d)", snapshot.Metadata.Index)
		}
	}
	newSrv := srv // since srv == nil in defer if srv is returned as nil
	defer func() {
		// closing backend without first closing kv can cause
		// resumed compactions to fail with closed tx errors
		if err != nil {
			newSrv.kv.Close()
		}
	}()

	srv.consistIndex.setConsistentIndex(srv.kv.ConsistentIndex())
	tp, err := auth.NewTokenProvider(cfg.AuthToken,
		func(index uint64) <-chan struct{} {
			return srv.applyWait.Wait(index)
		},
	)
	if err != nil {
		plog.Errorf("failed to create token provider: %s", err)
		return nil, err
	}
	srv.authStore = auth.NewAuthStore(srv.be, tp) //鉴权存储
	if h := cfg.AutoCompactionRetention; h != 0 { //周期性合并
		srv.compactor = compactor.NewPeriodic(h, srv.kv, srv)
		srv.compactor.Run()
	}

	srv.applyV3Base = &applierV3backend{srv}
	if err = srv.restoreAlarms(); err != nil { //配额告警
		return nil, err
	}

	// 负责与其他节点通信
	tr := &rafthttp.Transport{
		TLSInfo:     cfg.PeerTLSInfo,
		DialTimeout: cfg.peerDialTimeout(),
		ID:          id,
		URLs:        cfg.PeerURLs,
		ClusterID:   cl.ID(),
		Raft:        srv,
		Snapshotter: ss,
		ServerStats: sstats,
		LeaderStats: lstats,
		ErrorC:      srv.errorc,
	}
	if err = tr.Start(); err != nil {
		return nil, err
	}
	// add all remotes into transport
	for _, m := range remotes {
		if m.ID != id {
			tr.AddRemote(m.ID, m.PeerURLs)
		}
	}
	for _, m := range cl.Members() {
		if m.ID != id {
			tr.AddPeer(m.ID, m.PeerURLs)
		}
	}
	srv.r.transport = tr

	return srv, nil
}
```

## 创建raftNode

```go
func newRaftNode(cfg raftNodeConfig) *raftNode {
   r := &raftNode{
      tickMu:         new(sync.Mutex),
      raftNodeConfig: cfg,
      //检测发往同一节点的心跳是否超时
      td:         contention.NewTimeoutDetector(2 * cfg.heartbeat),
      readStateC: make(chan raft.ReadState, 1),
      msgSnapC:   make(chan raftpb.Message, maxInFlightMsgSnap),
      applyc:     make(chan apply),
      stopped:    make(chan struct{}),
      done:       make(chan struct{}),
   }
  //ticker逻辑逻辑时钟，每次触发推进一次选举计时器和心跳计时器
   if r.heartbeat == 0 {
      r.ticker = &time.Ticker{}
   } else {
      r.ticker = time.NewTicker(r.heartbeat)
   }
   return r
}
```

## 启动EtcdServer

etcdserver/server.go

```go
func (s *EtcdServer) Start() {
   s.start()//执行etcdServer的run方法
   s.goAttach(func() { s.adjustTicks() })
   s.goAttach(func() { s.publish(s.Cfg.ReqTimeout()) })
   s.goAttach(s.purgeFile) //定时清理wal和快照文件
   s.goAttach(func() { monitorFileDescriptor(s.getLogger(), s.stopping) })
   s.goAttach(s.monitorVersions)
   s.goAttach(s.linearizableReadLoop) //实现线性读功能
   s.goAttach(s.monitorKVHash)
}
```

### Start

etcd/etcdserver/server.go

```go
func (s *EtcdServer) start() {
	if s.snapCount == 0 {
		plog.Infof("set snapshot count to default %d", DefaultSnapCount)
		s.snapCount = DefaultSnapCount
	}
	s.w = wait.New()
	s.applyWait = wait.NewTimeList()
	s.done = make(chan struct{})
	s.stop = make(chan struct{})
	s.stopping = make(chan struct{})
	s.ctx, s.cancel = context.WithCancel(context.Background())
	s.readwaitc = make(chan struct{}, 1)
	s.readNotifier = newNotifier()
	if s.ClusterVersion() != nil {
		plog.Infof("starting server... [version: %v, cluster version: %v]", version.Version, version.Cluster(s.ClusterVersion().String()))
		membership.ClusterVersionMetrics.With(prometheus.Labels{"cluster_version": version.Cluster(s.ClusterVersion().String())}).Set(1)
	} else {
		plog.Infof("starting server... [version: %v, cluster version: to_be_decided]", version.Version)
	}
	// TODO: if this is an empty log, writes all peer infos
	// into the first entry
	go s.run()
}
```



```go
func (s *EtcdServer) run() {
	sn, err := s.r.raftStorage.Snapshot()
	if err != nil {
		plog.Panicf("get snapshot from raft storage error: %v", err)
	}
	//FIFO调度器
	sched := schedule.NewFIFOScheduler()

	var (
		smu   sync.RWMutex
		syncC <-chan time.Time
	)
	setSyncC := func(ch <-chan time.Time) {
		smu.Lock()
		syncC = ch
		smu.Unlock()
	}
	getSyncC := func() (ch <-chan time.Time) {
		smu.RLock()
		ch = syncC
		smu.RUnlock()
		return
	}
	rh := &raftReadyHandler{
		updateLeadership: func(newLeader bool) {
			if !s.isLeader() {
				if s.lessor != nil {
					s.lessor.Demote()
				}
				if s.compactor != nil {
					s.compactor.Pause()
				}
				setSyncC(nil)
			} else {
				if newLeader {
					t := time.Now()
					s.leadTimeMu.Lock()
					s.leadElectedTime = t
					s.leadTimeMu.Unlock()
				}
				setSyncC(s.SyncTicker.C)
				if s.compactor != nil {
					s.compactor.Resume()
				}
			}

			// TODO: remove the nil checking
			// current test utility does not provide the stats
			if s.stats != nil {
				s.stats.BecomeLeader()
			}
		},
		updateCommittedIndex: func(ci uint64) {
			cci := s.getCommittedIndex()
			if ci > cci {
				s.setCommittedIndex(ci)
			}
		},
	}
  //启动raftNode
	s.r.start(rh)

	ep := etcdProgress{
		confState: sn.Metadata.ConfState,
		snapi:     sn.Metadata.Index,
		appliedt:  sn.Metadata.Term,
		appliedi:  sn.Metadata.Index,
	}

	defer func() {
		s.wgMu.Lock() // block concurrent waitgroup adds in goAttach while stopping
		close(s.stopping)
		s.wgMu.Unlock()
		s.cancel()

		sched.Stop()

		// wait for gouroutines before closing raft so wal stays open
		s.wg.Wait()

		s.SyncTicker.Stop()

		// must stop raft after scheduler-- etcdserver can leak rafthttp pipelines
		// by adding a peer after raft stops the transport
		s.r.stop()

		// kv, lessor and backend can be nil if running without v3 enabled
		// or running unit tests.
		if s.lessor != nil {
			s.lessor.Stop()
		}
		if s.kv != nil {
			s.kv.Close()
		}
		if s.authStore != nil {
			s.authStore.Close()
		}
		if s.be != nil {
			s.be.Close()
		}
		if s.compactor != nil {
			s.compactor.Stop()
		}
		close(s.done)
	}()

	var expiredLeaseC <-chan []*lease.Lease
	if s.lessor != nil { //获取过期的Lease，lessor会启动协程间监听过期的lease
		expiredLeaseC = s.lessor.ExpiredLeasesC()
	}

	for {
		select {
		case ap := <-s.r.apply(): //监听raftNode的apply实例
			f := func(context.Context) { s.applyAll(&ep, &ap) }
			sched.Schedule(f) //应用到状态机任务交由调度器
		case leases := <-expiredLeaseC: //监听过期的Lease
			s.goAttach(func() {
				//通过并行化增加过期租约删除过程的吞吐量
				c := make(chan struct{}, maxPendingRevokes)//默认16个并行度
				for _, lease := range leases {
					select {
					case c <- struct{}{}: //加入并行通道
					case <-s.stopping:
						return
					}
					lid := lease.ID
					s.goAttach(func() {
            //发起raft协议，撤销lease
						_, lerr := s.LeaseRevoke(s.ctx, &pb.LeaseRevokeRequest{ID: int64(lid)})
						if lerr == nil {
							leaseExpired.Inc()
						} else {
							plog.Warningf("failed to revoke %016x (%q)", lid, lerr.Error())
						}

						<-c //撤销lease完成
					})
				}
			})
		case err := <-s.errorc:
			plog.Errorf("%s", err)
			plog.Infof("the data-dir used by this member must be removed.")
			return
		case <-getSyncC(): //定时发送sync消息
			if s.store.HasTTLKeys() { //如果v2只有永久节点，无需发送sync消息
				s.sync(s.Cfg.ReqTimeout())
			}
		case <-s.stop:
			return
		}
	}
}
```



```go
func (s *EtcdServer) goAttach(f func()) {
   s.wgMu.RLock() // this blocks with ongoing close(s.stopping)
   defer s.wgMu.RUnlock()
   select {
   case <-s.stopping: //检测EtcdServer实例是否已经停止
      plog.Warning("server has stopped (skipping goAttach)")
      return
   default:
   }

   s.wg.Add(1) //调用etcdServer的stop方法时，需要等待协程运行结束才返回
   go func() {
      defer s.wg.Done() //协程运行结束时执行
      f()
   }()
}
```



```go
func (s *EtcdServer) publish(timeout time.Duration) {
   b, err := json.Marshal(s.attributes)
   if err != nil {
      plog.Panicf("json marshal error: %v", err)
      return
   }
   req := pb.Request{
      Method: "PUT",
      Path:   membership.MemberAttributesStorePath(s.id),
      Val:    string(b),
   }

   for {
      ctx, cancel := context.WithTimeout(s.ctx, timeout)
      _, err := s.Do(ctx, req)
      cancel()
      switch err {
      case nil:
         close(s.readych)
         plog.Infof("published %+v to cluster %s", s.attributes, s.cluster.ID())
         return
      case ErrStopped:
         plog.Infof("aborting publish because server is stopped")
         return
      default:
         plog.Errorf("publish error: %v", err)
      }
   }
}
```

### purgeFile

```go
func (s *EtcdServer) purgeFile() {
   var dberrc, serrc, werrc <-chan error
   var dbdonec, sdonec, wdonec <-chan struct{}
  //启动两个goroutine,每隔30s清理一次wal和快照文件
   if s.Cfg.MaxSnapFiles > 0 {
      dbdonec, dberrc = fileutil.PurgeFileWithDoneNotify(s.Cfg.SnapDir(), "snap.db", s.Cfg.MaxSnapFiles, purgeFileInterval, s.stopping)
      sdonec, serrc = fileutil.PurgeFileWithDoneNotify(s.Cfg.SnapDir(), "snap", s.Cfg.MaxSnapFiles, purgeFileInterval, s.stopping)
   }
   if s.Cfg.MaxWALFiles > 0 {
      wdonec, werrc = fileutil.PurgeFileWithDoneNotify(s.Cfg.WALDir(), "wal", s.Cfg.MaxWALFiles, purgeFileInterval, s.stopping)
   }
   select {
   case e := <-dberrc:
      plog.Fatalf("failed to purge snap db file %v", e)
   case e := <-serrc:
      plog.Fatalf("failed to purge snap file %v", e)
   case e := <-werrc:
      plog.Fatalf("failed to purge wal file %v", e)
   case <-s.stopping:
      if dbdonec != nil {
         <-dbdonec
      }
      if sdonec != nil {
         <-sdonec
      }
      if wdonec != nil {
         <-wdonec
      }
      return
   }
}
```

### linearizableReadLoop

```go
func (s *EtcdServer) linearizableReadLoop() {
   var rs raft.ReadState

   for {
      ctxToSend := make([]byte, 8)
      id1 := s.reqIDGen.Next()
      binary.BigEndian.PutUint64(ctxToSend, id1)

      select {
      case <-s.readwaitc:
      case <-s.stopping:
         return
      }

      nextnr := newNotifier()

      s.readMu.Lock()
      nr := s.readNotifier
      s.readNotifier = nextnr
      s.readMu.Unlock()

      cctx, cancel := context.WithTimeout(context.Background(), s.Cfg.ReqTimeout())
      if err := s.r.ReadIndex(cctx, ctxToSend); err != nil {
         cancel()
         if err == raft.ErrStopped {
            return
         }
         plog.Errorf("failed to get read index from raft: %v", err)
         readIndexFailed.Inc()
         nr.notify(err)
         continue
      }
      cancel()

      var (
         timeout bool
         done    bool
      )
      for !timeout && !done {
         select {
         case rs = <-s.r.readStateC:
            done = bytes.Equal(rs.RequestCtx, ctxToSend)
            if !done {
               // a previous request might time out. now we should ignore the response of it and
               // continue waiting for the response of the current requests.
               id2 := uint64(0)
               if len(rs.RequestCtx) == 8 {
                  id2 = binary.BigEndian.Uint64(rs.RequestCtx)
               }
               plog.Warningf("ignored out-of-date read index response; local node read indexes queueing up and waiting to be in sync with leader (request ID want %d, got %d)", id1, id2)
               slowReadIndex.Inc()
            }

         case <-time.After(s.Cfg.ReqTimeout()):
            plog.Warningf("timed out waiting for read index response (local node might have slow network)")
            nr.notify(ErrTimeout)
            timeout = true
            slowReadIndex.Inc()

         case <-s.stopping:
            return
         }
      }
      if !done {
         continue
      }

      if ai := s.getAppliedIndex(); ai < rs.Index {
         select {
         case <-s.applyWait.Wait(rs.Index):
         case <-s.stopping:
            return
         }
      }
      // unblock all l-reads requested at indices before rs.Index
      nr.notify(nil)
   }
}
```

### applyAll

```go
func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply) {
	s.applySnapshot(ep, apply)
	s.applyEntries(ep, apply)

	proposalsApplied.Set(float64(ep.appliedi))
	s.applyWait.Trigger(ep.appliedi)
	// wait for the raft routine to finish the disk writes before triggering a
	// snapshot. or applied index might be greater than the last index in raft
	// storage, since the raft routine might be slower than apply routine.
	<-apply.notifyc

	s.triggerSnapshot(ep)//触发生成快照文件
	select {
	case m := <-s.r.msgSnapC: //发送快照，快速同步数据
		merged := s.createMergedSnapshotMessage(m, ep.appliedt, ep.appliedi, ep.confState)
		s.sendMergedSnap(merged)
	default:
	}
}
```

#### applySnapshot

```go
func (s *EtcdServer) applySnapshot(ep *etcdProgress, apply *apply) {
   if raft.IsEmptySnap(apply.snapshot) { 
      return
   }

   plog.Infof("applying snapshot at index %d...", ep.snapi)
   defer plog.Infof("finished applying incoming snapshot at index %d", ep.snapi)

   if apply.snapshot.Metadata.Index <= ep.appliedi {
      plog.Panicf("snapshot index [%d] should > appliedi[%d] + 1",
         apply.snapshot.Metadata.Index, ep.appliedi)
   }

   // 等待raftNode将快照持久化到磁盘
   <-apply.notifyc

   newbe, err := openSnapshotBackend(s.Cfg, s.snapshotter, apply.snapshot)
   if err != nil {
      plog.Panic(err)
   }

   // always recover lessor before kv. When we recover the mvcc.KV it will reattach keys to its leases.
   // If we recover mvcc.KV first, it will attach the keys to the wrong lessor before it recovers.
   if s.lessor != nil {
      plog.Info("recovering lessor...")
      s.lessor.Recover(newbe, func() lease.TxnDelete { return s.kv.Write() })
      plog.Info("finished recovering lessor")
   }

   plog.Info("restoring mvcc store...")

   if err := s.kv.Restore(newbe); err != nil {
      plog.Panicf("restore KV error: %v", err)
   }
   s.consistIndex.setConsistentIndex(s.kv.ConsistentIndex())

   plog.Info("finished restoring mvcc store")

   // Closing old backend might block until all the txns
   // on the backend are finished.
   // We do not want to wait on closing the old backend.
   s.bemu.Lock()
   oldbe := s.be
   go func() {
      plog.Info("closing old backend...")
      defer plog.Info("finished closing old backend")

      if err := oldbe.Close(); err != nil {
         plog.Panicf("close backend error: %v", err)
      }
   }()

   s.be = newbe
   s.bemu.Unlock()

   plog.Info("recovering alarms...")
   if err := s.restoreAlarms(); err != nil {
      plog.Panicf("restore alarms error: %v", err)
   }
   plog.Info("finished recovering alarms")

   if s.authStore != nil {
      plog.Info("recovering auth store...")
      s.authStore.Recover(newbe)
      plog.Info("finished recovering auth store")
   }

   plog.Info("recovering store v2...")
   if err := s.store.Recovery(apply.snapshot.Data); err != nil {
      plog.Panicf("recovery store error: %v", err)
   }
   plog.Info("finished recovering store v2")

   s.cluster.SetBackend(s.be)
   plog.Info("recovering cluster configuration...")
   s.cluster.Recover(api.UpdateCapability)
   plog.Info("finished recovering cluster configuration")

   plog.Info("removing old peers from network...")
   // recover raft transport
   s.r.transport.RemoveAllPeers()
   plog.Info("finished removing old peers from network")

   plog.Info("adding peers from new cluster configuration into network...")
   for _, m := range s.cluster.Members() {
      if m.ID == s.ID() {
         continue
      }
      s.r.transport.AddPeer(m.ID, m.PeerURLs)
   }
   plog.Info("finished adding peers from new cluster configuration into network...")

   ep.appliedt = apply.snapshot.Metadata.Term
   ep.appliedi = apply.snapshot.Metadata.Index
   ep.snapi = ep.appliedi
   ep.confState = apply.snapshot.Metadata.ConfState
}
```

#### applyEntries

```go
func (s *EtcdServer) applyEntries(ep *etcdProgress, apply *apply) {
   if len(apply.entries) == 0 {
      return
   }
   firsti := apply.entries[0].Index
   if firsti > ep.appliedi+1 { //检测待应用的第一条entry记录是否合法
      plog.Panicf("first index of committed entry[%d] should <= appliedi[%d] + 1", firsti, ep.appliedi)
   }
   var ents []raftpb.Entry
   if ep.appliedi+1-firsti < uint64(len(apply.entries)) { //忽略已经应用的entry
      ents = apply.entries[ep.appliedi+1-firsti:]
   }
   if len(ents) == 0 {
      return
   }
   var shouldstop bool
  //执行apply方法，保存到v2或者v3存储
   if ep.appliedi, shouldstop = s.apply(ents, &ep.confState); shouldstop {
      go s.stopWithDelay(10*100*time.Millisecond, fmt.Errorf("the member has been permanently removed from the cluster"))
   }
}
```



```go
func (s *EtcdServer) apply(es []raftpb.Entry, confState *raftpb.ConfState) (appliedt uint64, appliedi uint64, shouldStop bool) {
   for i := range es {
      e := es[i]
      switch e.Type {
      case raftpb.EntryNormal: //正常数据插入
         s.applyEntryNormal(&e)
      case raftpb.EntryConfChange: //配置变更
         // 更新consistentIndex,保证幂等
         if e.Index > s.consistIndex.ConsistentIndex() {
            s.consistIndex.setConsistentIndex(e.Index)
         }
         var cc raftpb.ConfChange
         pbutil.MustUnmarshal(&cc, e.Data) //反序列成ConfChange实例
         removedSelf, err := s.applyConfChange(cc, confState) //处理ConfChange
         s.setAppliedIndex(e.Index) //更新appliedIndex字段
        //removeSelf为true表示将当前节点从集群删除
         shouldStop = shouldStop || removedSelf
        //唤醒阻塞的请求
         s.w.Trigger(cc.ID, &confChangeResponse{s.cluster.Members(), err})
      default:
         plog.Panicf("entry type should be either EntryNormal or EntryConfChange")
      }
     //更新raftNode的index、term字段
      atomic.StoreUint64(&s.r.index, e.Index)
      atomic.StoreUint64(&s.r.term, e.Term)
      appliedt = e.Term
      appliedi = e.Index
   }

   return appliedt, appliedi, shouldStop
}
```

##### applyRequest

```go
func (s *EtcdServer) applyRequest(r pb.Request) Response {//应用到状态机
  //创建响应
   f := func(ev *store.Event, err error) Response { 
      return Response{Event: ev, err: err}
   }
   expr := timeutil.UnixNanoToTime(r.Expiration)
   switch r.Method {
   case "POST":
      return f(s.store.Create(r.Path, r.Dir, r.Val, true, expr))
   case "PUT":
      exists, existsSet := pbutil.GetBool(r.PrevExist)
      switch {
      case existsSet:
         if exists {
            if r.PrevIndex == 0 && r.PrevValue == "" {
               return f(s.store.Update(r.Path, r.Val, expr))
            } else {
               return f(s.store.CompareAndSwap(r.Path, r.PrevValue, r.PrevIndex, r.Val, expr))
            }
         }
         return f(s.store.Create(r.Path, r.Dir, r.Val, false, expr))
      case r.PrevIndex > 0 || r.PrevValue != "":
         return f(s.store.CompareAndSwap(r.Path, r.PrevValue, r.PrevIndex, r.Val, expr))
      default:
         if storeMemberAttributeRegexp.MatchString(r.Path) {
            id := mustParseMemberIDFromKey(path.Dir(r.Path))
            var attr Attributes
            if err := json.Unmarshal([]byte(r.Val), &attr); err != nil {
               log.Panicf("unmarshal %s should never fail: %v", r.Val, err)
            }
            s.Cluster.UpdateAttributes(id, attr)
         }
         return f(s.store.Set(r.Path, r.Dir, r.Val, expr))
      }
   case "DELETE":
      switch {
      case r.PrevIndex > 0 || r.PrevValue != "":
         return f(s.store.CompareAndDelete(r.Path, r.PrevValue, r.PrevIndex))
      default:
         return f(s.store.Delete(r.Path, r.Dir, r.Recursive))
      }
   case "QGET":
      return f(s.store.Get(r.Path, r.Recursive, r.Sorted))
   case "SYNC":
      s.store.DeleteExpiredKeys(time.Unix(0, r.Time))
      return Response{}
   default:
      // This should never be reached, but just in case:
      return Response{err: ErrUnknownMethod}
   }
}
```

##### applyV3Request

```go
func (s *EtcdServer) applyV3Request(r *pb.InternalRaftRequest) interface{} {
   kv := s.getKV()
   le := s.lessor

   ar := &applyResult{}

   switch {
   case r.Range != nil:
      ar.resp, ar.err = applyRange(noTxn, kv, r.Range)
   case r.Put != nil:
      ar.resp, ar.err = applyPut(noTxn, kv, le, r.Put)
   case r.DeleteRange != nil:
      ar.resp, ar.err = applyDeleteRange(noTxn, kv, r.DeleteRange)
   case r.Txn != nil:
      ar.resp, ar.err = applyTxn(kv, le, r.Txn)
   case r.Compaction != nil:
      ar.resp, ar.err = applyCompaction(kv, r.Compaction)
   case r.LeaseCreate != nil:
      ar.resp, ar.err = applyLeaseCreate(le, r.LeaseCreate)
   case r.LeaseRevoke != nil:
      ar.resp, ar.err = applyLeaseRevoke(le, r.LeaseRevoke)
   case r.AuthEnable != nil:
      ar.resp, ar.err = applyAuthEnable(s)
   default:
      panic("not implemented")
   }

   return ar
}
```

etcd/store/store.go

```go
func (s *store) Create(nodePath string, dir bool, value string, unique bool, expireTime time.Time) (*Event, error) {
   s.worldLock.Lock() //写锁
   defer s.worldLock.Unlock()
   e, err := s.internalCreate(nodePath, dir, value, unique, false, expireTime, Create)

   if err == nil {
      e.EtcdIndex = s.CurrentIndex
      s.WatcherHub.notify(e)
      s.Stats.Inc(CreateSuccess)
   } else {
      s.Stats.Inc(CreateFail)
   }

   return e, err
}
```

etcd/pkg/wait/wait.go

```go
func (w *List) Trigger(id uint64, x interface{}) {
   w.l.Lock()
   ch := w.m[id]
   delete(w.m, id)
   w.l.Unlock()
   if ch != nil {
      ch <- x
      close(ch)
   }
}
```

raft/storage.go

```go
func NewMemoryStorage() *MemoryStorage {
   return &MemoryStorage{
      // When starting from scratch populate the list with a dummy entry at term zero.
      ents: make([]pb.Entry, 1),
   }
}
```

#### triggerSnapshot

```go
func (s *EtcdServer) triggerSnapshot(ep *etcdProgress) {
   if ep.appliedi-ep.snapi <= s.snapCount {
      return
   }

   plog.Infof("start to snapshot (applied: %d, lastsnap: %d)", ep.appliedi, ep.snapi)
   s.snapshot(ep.appliedi, ep.confState) //参考日志和快照管理中的的快照管理
   ep.snapi = ep.appliedi
}
```

## 启动raftNode

```go
func (r *raftNode) start(rh *raftReadyHandler) {
   internalTimeout := time.Second

   go func() {
      defer r.onStop()
      islead := false

      for {
         select {
         case <-r.ticker.C://触发选举或者发送心跳，每次时间推进TickMs
            r.tick()
         case rd := <-r.Ready()://node实例的ready
           //softState封装集群的leader信息和当前节点的角色
            if rd.SoftState != nil {
               newLeader := rd.SoftState.Lead != raft.None && atomic.LoadUint64(&r.lead) != rd.SoftState.Lead //true表明leader发生了变更
               if newLeader {
                  leaderChanges.Inc()
               }

               if rd.SoftState.Lead == raft.None {
                  hasLeader.Set(0)
               } else {
                  hasLeader.Set(1)
               }

               atomic.StoreUint64(&r.lead, rd.SoftState.Lead) //更新为新的leader节点的ID
               islead = rd.RaftState == raft.StateLeader //当前节点是否是leader
               if islead {
                  isLeader.Set(1)
               } else {
                  isLeader.Set(0)
               }
               rh.updateLeadership(newLeader)  //leader变更后执行的操作
               r.td.Reset() //重置探测器
            }

            if len(rd.ReadStates) != 0 {
               select {
                 //将readStates的最后一项写入readStateC
               case r.readStateC <- rd.ReadStates[len(rd.ReadStates)-1]:
               case <-time.After(internalTimeout):
                 //通道满了阻塞，等待1秒，如果依然写入不了，则输出警告日志
                  plog.Warningf("timed out sending read state")
               case <-r.stopped:
                  return
               }
            }

            notifyc := make(chan struct{}, 1)
            ap := apply{
               entries:  rd.CommittedEntries, //已提交、待应用的entry
               snapshot: rd.Snapshot,//带持久化的快照数据
               notifyc:  notifyc, //协调协程间通信
            }
						//更新EtcdServer中记录的已经提交位置
            updateCommittedIndex(&ap, rh)

            select {
            case r.applyc <- ap: //将apply实例applyc通道，交由上层模块（EtcdServer）处理
            case <-r.stopped:
               return
            }

            if islead { //当前节点是leader
               // gofail: var raftBeforeLeaderSend struct{}
               r.transport.Send(r.processMessages(rd.Messages))
            }

            // 将HardState、entries写入wal日志文件
            if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
               plog.Fatalf("raft save state and entries error: %v", err)
            }
            if !raft.IsEmptyHardState(rd.HardState) {
               proposalsCommitted.Set(float64(rd.HardState.Commit))
            }
            // gofail: var raftAfterSave struct{}

            if !raft.IsEmptySnap(rd.Snapshot) {
               //将快照数据保存到磁盘
               if err := r.storage.SaveSnap(rd.Snapshot); err != nil {
                  plog.Fatalf("raft save snapshot error: %v", err)
               }
               // Etcdserver现在声明快照已经持久化到磁盘上
               notifyc <- struct{}{}

               // 将快照数据保存到MemoryStorage
               r.raftStorage.ApplySnapshot(rd.Snapshot)
               plog.Infof("raft applied incoming snapshot at index %d", rd.Snapshot.Metadata.Index)
            }
						//将数据写入MemoryStorage
            r.raftStorage.Append(rd.Entries)

            if !islead {
               // finish processing incoming messages before we signal raftdone chan
               msgs := r.processMessages(rd.Messages)

               // now unblocks 'applyAll' that waits on Raft log disk writes before triggering snapshots
               notifyc <- struct{}{}

               // Candidate or follower needs to wait for all pending configuration
               // changes to be applied before sending messages.
               // Otherwise we might incorrectly count votes (e.g. votes from removed members).
               // Also slow machine's follower raft-layer could proceed to become the leader
               // on its own single-node cluster, before apply-layer applies the config change.
               // We simply wait for ALL pending entries to be applied for now.
               // We might improve this later on if it causes unnecessary long blocking issues.
               waitApply := false
               for _, ent := range rd.CommittedEntries {
                  if ent.Type == raftpb.EntryConfChange {
                     waitApply = true
                     break
                  }
               }
               if waitApply {
                  // blocks until 'applyAll' calls 'applyWait.Trigger'
                  // to be in sync with scheduled config-change job
                  // (assume notifyc has cap of 1)
                  select {
                  case notifyc <- struct{}{}:
                  case <-r.stopped:
                     return
                  }
               }

               r.transport.Send(msgs)
            } else {
               notifyc <- struct{}{}
            }

            r.Advance()
         case <-r.stopped:
            return
         }
      }
   }()
}
```



```go

```



# node结构体

## 初次启动

raft/node.go

```go
func StartNode(c *Config, peers []Peer) Node {
	r := newRaft(c)//根据配置创建raft实例
	// become the follower at term 1 and apply initial configuration
	// entries of term 1
	r.becomeFollower(1, None)
	for _, peer := range peers {
		cc := pb.ConfChange{Type: pb.ConfChangeAddNode, NodeID: peer.ID, Context: peer.Context}
		d, err := cc.Marshal()
		if err != nil {
			panic("unexpected marshal error")
		}
		e := pb.Entry{Type: pb.EntryConfChange, Term: 1, Index: r.raftLog.lastIndex() + 1, Data: d}
		r.raftLog.append(e)
	}
	r.raftLog.committed = r.raftLog.lastIndex()
	for _, peer := range peers {
		r.addNode(peer.ID)
	}

	n := newNode() //初始化node实例
	go n.run(r) //处理node中各通道的数据
	return &n
}
```

newNode

raft/raft.go

```go
func newNode() node {
   return node{
      propc:      make(chan pb.Message),
      recvc:      make(chan pb.Message),
      confc:      make(chan pb.ConfChange),
      confstatec: make(chan pb.ConfState),
      readyc:     make(chan Ready),
      advancec:   make(chan struct{}),
      tickc:      make(chan struct{}),
      done:       make(chan struct{}),
      stop:       make(chan struct{}),
      status:     make(chan chan Status),
   }
}
```

```go
func RestartNode(c *Config) Node { //重新启动
   r := newRaft(c)

   n := newNode()
   go n.run(r)
   return &n
}
```

## run方法

raft/node.go

```go
func (n *node) run(r *raft) {
   var propc chan pb.Message
   var readyc chan Ready
   var advancec chan struct{}
   var prevLastUnstablei, prevLastUnstablet uint64
   var havePrevLastUnstablei bool
   var prevSnapi uint64
   var rd Ready

   lead := None
   prevSoftSt := r.softState()
   prevHardSt := emptyState

   for {
      if advancec != nil {
         readyc = nil
      } else {
         rd = newReady(r, prevSoftSt, prevHardSt)
         if rd.containsUpdates() {
            readyc = n.readyc
         } else {
            readyc = nil
         }
      }

      if lead != r.lead {
         if r.hasLeader() {
            if lead == None {
               log.Printf("raft.node: %x elected leader %x at term %d", r.id, r.lead, r.Term)
            } else {
               log.Printf("raft.node: %x changed leader from %x to %x at term %d", r.id, lead, r.lead, r.Term)
            }
            propc = n.propc
         } else {
            log.Printf("raft.node: %x lost leader %x at term %d", r.id, lead, r.Term)
            propc = nil
         }
         lead = r.lead
      }

      select {
        case m := <-propc: //获取msgPropc消息，交给raft.step()方法处理
         m.From = r.id
         r.Step(m)
      case m := <-n.recvc: //获取非msgPropc消息，交给raft.step()方法处理
         if _, ok := r.prs[m.From]; ok || !IsResponseMsg(m) {
            r.Step(m) // raft never returns an error
         }
      case cc := <-n.confc: //配置变更、节点上线、节点下线
         if cc.NodeID == None {
            r.resetPendingConf()
            select {
            case n.confstatec <- pb.ConfState{Nodes: r.nodes()}:
            case <-n.done:
            }
            break
         }
         switch cc.Type {
         case pb.ConfChangeAddNode:
            r.addNode(cc.NodeID)
         case pb.ConfChangeRemoveNode:
            // block incoming proposal when local node is
            // removed
            if cc.NodeID == r.id {
               n.propc = nil
            }
            r.removeNode(cc.NodeID)
         case pb.ConfChangeUpdateNode:
            r.resetPendingConf()
         default:
            panic("unexpected conf type")
         }
         select {
         case n.confstatec <- pb.ConfState{Nodes: r.nodes()}:
         case <-n.done:
         }
      case <-n.tickc: //由上层模块向tickc通道写入信号
         r.tick() //选举或者心跳
      case readyc <- rd: //将创建的ready实例写入readyc通道，等待上层raft模块处理
         if rd.SoftState != nil {
            prevSoftSt = rd.SoftState
         }
         if len(rd.Entries) > 0 {
            prevLastUnstablei = rd.Entries[len(rd.Entries)-1].Index
            prevLastUnstablet = rd.Entries[len(rd.Entries)-1].Term
            havePrevLastUnstablei = true
         }
         if !IsEmptyHardState(rd.HardState) {
            prevHardSt = rd.HardState
         }
         if !IsEmptySnap(rd.Snapshot) {
            prevSnapi = rd.Snapshot.Metadata.Index
         }
         r.msgs = nil
         advancec = n.advancec
      case <-advancec: //上层模块处理ready实例之后，向advances通道写入信号
         if prevHardSt.Commit != 0 {
            r.raftLog.appliedTo(prevHardSt.Commit)
         }
         if havePrevLastUnstablei {
            r.raftLog.stableTo(prevLastUnstablei, prevLastUnstablet)
            havePrevLastUnstablei = false
         }
         r.raftLog.stableSnapTo(prevSnapi)
         advancec = nil
      case c := <-n.status:
         c <- getStatus(r)
      case <-n.stop:
         close(n.done)
         return
      }
   }
}
```

```go
func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
   rd := Ready{
      Entries:          r.raftLog.unstableEntries(),
      CommittedEntries: r.raftLog.nextEnts(),
      Messages:         r.msgs,
   }
   if softSt := r.softState(); !softSt.equal(prevSoftSt) {
      rd.SoftState = softSt
   }
   if hardSt := r.hardState(); !isHardStateEqual(hardSt, prevHardSt) {
      rd.HardState = hardSt
   }
   if r.raftLog.unstable.snapshot != nil {
      rd.Snapshot = *r.raftLog.unstable.snapshot
   }
   return rd
}
```

# raft实现

## 初始化

etcd/raft/raft.go

```go
func newRaft(c *Config) *raft {
	if err := c.validate(); err != nil {
		panic(err.Error())
	}
	raftlog := newLog(c.Storage, c.Logger)
	hs, cs, err := c.Storage.InitialState()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	peers := c.peers
	if len(cs.Nodes) > 0 {
		if len(peers) > 0 {
			// TODO(bdarnell): the peers argument is always nil except in
			// tests; the argument should be removed and these tests should be
			// updated to specify their nodes through a snapshot.
			panic("cannot specify both newRaft(peers) and ConfState.Nodes)")
		}
		peers = cs.Nodes
	}
	r := &raft{
		id:               c.ID,
		lead:             None,
		raftLog:          raftlog,
		maxMsgSize:       c.MaxSizePerMsg,//已发送但是未收到响应的数据字节数
		maxInflight:      c.MaxInflightMsgs, //已发送但是未收到响应的数据条数
		prs:              make(map[uint64]*Progress),
		electionTimeout:  c.ElectionTick,
		heartbeatTimeout: c.HeartbeatTick,
		logger:           c.Logger,
		checkQuorum:      c.CheckQuorum,
	}
	r.rand = rand.New(rand.NewSource(int64(c.ID)))
	for _, p := range peers {//节点的同步状态
		r.prs[p] = &Progress{Next: 1, ins: newInflights(r.maxInflight)}
	}
	if !isHardStateEqual(hs, emptyState) {
		r.loadState(hs)
	}	
	if c.Applied > 0 {
		raftlog.appliedTo(c.Applied)
	}
	r.becomeFollower(r.Term, None) //将节点状态切换成Follower

	nodesStrs := make([]string, 0)
	for _, n := range r.nodes() {
		nodesStrs = append(nodesStrs, fmt.Sprintf("%x", n))
	}

	r.logger.Infof("newRaft %x [peers: [%s], term: %d, commit: %d, applied: %d, lastindex: %d, lastterm: %d]",
		r.id, strings.Join(nodesStrs, ","), r.Term, r.raftLog.committed, r.raftLog.applied, r.raftLog.lastIndex(), r.raftLog.lastTerm())
	return r
}
```

## 切换状态

### becomeFollower

raft/raft.go

```go
func (r *raft) becomeFollower(term uint64, lead uint64) {
 	//初始状态为StateFollower
   r.step = stepFollower
   r.reset(term)
   r.tick = r.tickElection
   r.lead = lead
   r.state = StateFollower
   log.Printf("raft: %x became follower at term %d", r.id, r.Term)
}
```

```go
func (r *raft) reset(term uint64) {
   if r.Term != term {
      r.Term = term
      r.Vote = None
   }
   r.lead = None

   r.electionElapsed = 0
   r.heartbeatElapsed = 0

   r.votes = make(map[uint64]bool)
   for id := range r.prs {
      r.prs[id] = &Progress{Next: r.raftLog.lastIndex() + 1, ins: newInflights(r.maxInflight)}
      if id == r.id {
         r.prs[id].Match = r.raftLog.lastIndex()
      }
   }
   r.pendingConf = false
}
```

tickElection

raft/raft.go

```go
func (r *raft) tickElection() {
	r.electionElapsed++

	if r.promotable() && r.pastElectionTimeout() {//选举超时
		r.electionElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```



```go
func (r *raft) pastElectionTimeout() bool {
  //超时时间随机化，防止同一时间多个节点同时发生选举，[electiontimeout, 2 * electiontimeout - 1]
   return r.electionElapsed >= r.randomizedElectionTimeout
}
```



```go
func (r *raft) Step(m pb.Message) error {
   if m.Type == pb.MsgHup { //选举超时
      log.Printf("raft: %x is starting a new election at term %d", r.id, r.Term)
      r.campaign()
      r.Commit = r.raftLog.committed
      return nil
   }

   switch {
   case m.Term == 0:
      // local message
   case m.Term > r.Term:
      lead := m.From
      if m.Type == pb.MsgVote {
         lead = None
      }
      log.Printf("raft: %x [term: %d] received a %s message with higher term from %x [term: %d]",
         r.id, r.Term, m.Type, m.From, m.Term)
      r.becomeFollower(m.Term, lead)
   case m.Term < r.Term:
      // ignore
      log.Printf("raft: %x [term: %d] ignored a %s message with lower term from %x [term: %d]",
         r.id, r.Term, m.Type, m.From, m.Term)
      return nil
   }
   r.step(r, m)
   r.Commit = r.raftLog.committed
   return nil
}
```



```go
func (r *raft) campaign() {
   //状态变为StateCandidate
   r.becomeCandidate()
   if r.q() == r.poll(r.id, true) { //获得过半节点的投票
      r.becomeLeader() //状态变为StateLeader
      return
   }
  //发送投票信息
   for i := range r.prs {
      if i == r.id {
         continue
      }
      log.Printf("raft: %x [logterm: %d, index: %d] sent vote request to %x at term %d",
         r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), i, r.Term)
      r.send(pb.Message{To: i, Type: pb.MsgVote, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm()})
   }
}
```

```go
func (r *raft) q() int { return len(r.prs)/2 + 1 } //过半节点数
```

```go
func (r *raft) poll(id uint64, v bool) (granted int) { //获得的投票数
   if v {
      log.Printf("raft: %x received vote from %x at term %d", r.id, id, r.Term)
   } else {
      log.Printf("raft: %x received vote rejection from %x at term %d", r.id, id, r.Term)
   }
   if _, ok := r.votes[id]; !ok {
      r.votes[id] = v
   }
   for _, vv := range r.votes {
      if vv {
         granted++
      }
   }
   return granted
}
```

### becomeCandidate

```go
func (r *raft) becomeCandidate() {
   // TODO(xiangli) remove the panic when the raft implementation is stable
   if r.state == StateLeader {
      panic("invalid transition [leader -> candidate]")
   }
   r.step = stepCandidate
   r.reset(r.Term + 1)
   r.tick = r.tickElection
   r.Vote = r.id
   r.state = StateCandidate
   log.Printf("raft: %x became candidate at term %d", r.id, r.Term)
}
```

### becomeLeader

```go
func (r *raft) becomeLeader() {
   // TODO(xiangli) remove the panic when the raft implementation is stable
   if r.state == StateFollower {
      panic("invalid transition [follower -> leader]")
   }
   r.step = stepLeader
   r.reset(r.Term)
   r.tick = r.tickHeartbeat
   r.lead = r.id
   r.state = StateLeader
  //未提交的日志项中是否有EntryConfChange类型的消息
   for _, e := range r.raftLog.entries(r.raftLog.committed + 1) {
      if e.Type != pb.EntryConfChange {
         continue
      }
      if r.pendingConf {
         panic("unexpected double uncommitted config entry")
      }
      r.pendingConf = true 
   }
  //追加空日志，防止”幽灵复现“
   r.appendEntry(pb.Entry{Data: nil})
   log.Printf("raft: %x became leader at term %d", r.id, r.Term)
}
```

appendEntry

```go
func (r *raft) appendEntry(es ...pb.Entry) {
 //最后一条记录的索引值
   li := r.raftLog.lastIndex()
  //修改entry的term和index
   for i := range es {
      es[i].Term = r.Term
      es[i].Index = li + 1 + uint64(i)
   }
  //追加
   r.raftLog.append(es...)
   r.prs[r.id].update(r.raftLog.lastIndex())
  //可能修改committed（日志项已经复制到过半节点）
   r.maybeCommit()
}
```

lastIndex

etcd/raft/log.go

```go
func (l *raftLog) lastIndex() uint64 {//最后一条记录的索引值
   if i, ok := l.unstable.maybeLastIndex(); ok {
      return i
   }
   i, err := l.storage.LastIndex()
   if err != nil {
      panic(err) // TODO(bdarnell)
   }
   return i
}
```

raft/log.go

```go
func (l *raftLog) append(ents ...pb.Entry) uint64 {
   if len(ents) == 0 {
      return l.lastIndex()
   }
   if after := ents[0].Index - 1; after < l.committed {
      log.Panicf("after(%d) is out of range [committed(%d)]", after, l.committed)
   }
   l.unstable.truncateAndAppend(ents)
   return l.lastIndex()
}
```

raft/log_unstable.go

```go
func (u *unstable) truncateAndAppend(ents []pb.Entry) {
   after := ents[0].Index - 1
   switch {
   case after == u.offset+uint64(len(u.entries))-1:
      // after is the last index in the u.entries
      // directly append
      u.entries = append(u.entries, ents...)
   case after < u.offset:
      log.Printf("raftlog: replace the unstable entries from index %d", after+1)
      // The log is being truncated to before our current offset
      // portion, so set the offset and replace the entries
      u.offset = after + 1
      u.entries = ents
   default:
      // truncate to after and copy to u.entries
      // then append
      log.Printf("raftlog: truncate the unstable entries to index %d", after)
      u.entries = append([]pb.Entry{}, u.slice(u.offset, after+1)...)
      u.entries = append(u.entries, ents...)
   }
}
```

maybeUpdate

```go
func (pr *Progress) maybeUpdate(n uint64) bool {
   var updated bool
   if pr.Match < n {
      pr.Match = n //n之前的数据都写入raftlog
      updated = true
      pr.resume() //leader可以向其他节点复制数据
   }
   if pr.Next < n+1 {
      pr.Next = n + 1 //下次复制的数据从next开始
   }
   return updated
}
```

maybeCommit

```go
func (r *raft) maybeCommit() bool {
   // TODO(bmizerany): optimize.. Currently naive
   mis := make(uint64Slice, 0, len(r.prs))
   for id := range r.prs {
      mis = append(mis, r.prs[id].Match)
   }
   sort.Sort(sort.Reverse(mis)) //对match值排序
   mci := mis[r.quorum()-1]//如果5个实例，r.quorum()-1=2，获取索引为2对应的match值
   return r.raftLog.maybeCommit(mci, r.Term)//更新commited值
}
```

## 处理消息	

```go
func (r *raft) Step(m pb.Message) error { //由node.run()调用或者选举、心跳超时时调用
   if m.Type == pb.MsgHup {
      if r.state != StateLeader { //非leader节点处理此类型消息
         r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
         r.campaign() //切换角色，发起投票
      } else {
         r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
      }
      return nil
   }

   switch {
   case m.Term == 0:
   case m.Term > r.Term://消息中的term大于本机term
      lead := m.From
      if m.Type == pb.MsgVote {
         lead = None
      }
      r.logger.Infof("%x [term: %d] received a %s message with higher term from %x [term: %d]", r.id, r.Term, m.Type, m.From, m.Term)
      r.becomeFollower(m.Term, lead)
   case m.Term < r.Term:
      // ignore
      r.logger.Infof("%x [term: %d] ignored a %s message with lower term from %x [term: %d]",
         r.id, r.Term, m.Type, m.From, m.Term)
      return nil
   }
   r.step(r, m)
   return nil
}
```

### stepCandidate

```go
func stepCandidate(r *raft, m pb.Message) {//主要处理投票响应信息
   switch m.Type {
   case pb.MsgProp:
      r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
      return
   case pb.MsgApp:
      r.becomeFollower(r.Term, m.From)
      r.handleAppendEntries(m)
   case pb.MsgHeartbeat:
      r.becomeFollower(r.Term, m.From)
      r.handleHeartbeat(m)
   case pb.MsgSnap:
      r.becomeFollower(m.Term, m.From)
      r.handleSnapshot(m)
   case pb.MsgVote: //不接受其他节点的投票信息
      r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected vote from %x [logterm: %d, index: %d] at term %d",
         r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.From, m.LogTerm, m.Index, r.Term)
      r.send(pb.Message{To: m.From, Type: pb.MsgVoteResp, Reject: true})
   case pb.MsgVoteResp: //投票响应信息
      gr := r.poll(m.From, !m.Reject)//是否超过半数节点支持此节点当选leader
      r.logger.Infof("%x [quorum:%d] has received %d votes and %d vote rejections", r.id, r.quorum(), gr, len(r.votes)-gr)
      switch r.quorum() { //len(r.prs)/2 + 1
      case gr:
         r.becomeLeader() //切换成leader
         r.bcastAppend()  //发送entry
      case len(r.votes) - gr:
         r.becomeFollower(r.Term, None)
      }
   }
}
```

```go
func (r *raft) bcastAppend() { //同步数据
   for i := range r.prs {
      if i == r.id { //过滤本机
         continue
      }
      r.sendAppend(i)
   }
}
```

```go
func (r *raft) sendAppend(to uint64) {
	pr := r.prs[to]
	if pr.isPaused() {
		return
	}
	m := pb.Message{}
	m.To = to

	term, errt := r.raftLog.term(pr.Next - 1)
  //获取发送的数据
	ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)

	if errt != nil || erre != nil { // 发送快照
		if !pr.RecentActive {
			r.logger.Debugf("ignore sending snapshot to %x since it is not recently active", to)
			return
		}

		m.Type = pb.MsgSnap
		snapshot, err := r.raftLog.snapshot()
		if err != nil {
			if err == ErrSnapshotTemporarilyUnavailable {
				r.logger.Debugf("%x failed to send snapshot to %x because snapshot is temporarily unavailable", r.id, to)
				return
			}
			panic(err) // TODO(bdarnell)
		}
		if IsEmptySnap(snapshot) {
			panic("need non-empty snapshot")
		}
		m.Snapshot = snapshot
		sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
		r.logger.Debugf("%x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
			r.id, r.raftLog.firstIndex(), r.raftLog.committed, sindex, sterm, to, pr)
		pr.becomeSnapshot(sindex)
		r.logger.Debugf("%x paused sending replication messages to %x [%s]", r.id, to, pr)
	} else { //发送数据
		m.Type = pb.MsgApp
		m.Index = pr.Next - 1
		m.LogTerm = term
		m.Entries = ents
		m.Commit = r.raftLog.committed
		if n := len(m.Entries); n != 0 {
			switch pr.State {
			// optimistically increase the next when in ProgressStateReplicate
			case ProgressStateReplicate:
				last := m.Entries[n-1].Index
				pr.optimisticUpdate(last) //修改下次同步数据的位置
				pr.ins.add(last)//存放未响应的消息
			case ProgressStateProbe:
				pr.pause()
			default:
				r.logger.Panicf("%x is sending append in unhandled state %s", r.id, pr.State)
			}
		}
	}
	r.send(m)
}
```

```go
func (r *raft) send(m pb.Message) {
   m.From = r.id
   if m.Type != pb.MsgProp {
      m.Term = r.Term
   }
   r.msgs = append(r.msgs, m) 
}
```

### stepLeader

```go
func stepLeader(r *raft, m pb.Message) {

	//来自本机的消息
	switch m.Type {
	case pb.MsgBeat: //发送心跳
		r.bcastHeartbeat()
		return
	case pb.MsgCheckQuorum: //leader节点，是否与大多数节点保持连通状态（在选举超时时触发）
		if !r.checkQuorumActive() {
			r.logger.Warningf("%x stepped down to follower since quorum is not active", r.id)
			r.becomeFollower(r.Term, None) //切换成Follower
		}
		return
	case pb.MsgProp: //同步数据
		if len(m.Entries) == 0 {
			r.logger.Panicf("%x stepped empty MsgProp", r.id)
		}
		if _, ok := r.prs[r.id]; !ok { //节点被移除
			return
		}
    //是否有EntryConfChange类型消息
		for i, e := range m.Entries {
			if e.Type == pb.EntryConfChange {
				if r.pendingConf {
					m.Entries[i] = pb.Entry{Type: pb.EntryNormal}
				}
				r.pendingConf = true
			}
		}
		r.appendEntry(m.Entries...)
		r.bcastAppend()
		return
	case pb.MsgVote: //投票请求
		r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected vote from %x [logterm: %d, index: %d] at term %d",
			r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.From, m.LogTerm, m.Index, r.Term)
		r.send(pb.Message{To: m.From, Type: pb.MsgVoteResp, Reject: true /*拒绝投票*/})
		return
	}

	//来自其他节点的消息
	pr, prOk := r.prs[m.From]
	if !prOk {
		r.logger.Debugf("no progress available for %x", m.From)
		return
	}
	switch m.Type {
	case pb.MsgAppResp:
		pr.RecentActive = true

		if m.Reject {
			r.logger.Debugf("%x received msgApp rejection(lastindex: %d) from %x for index %d",
				r.id, m.RejectHint, m.From, m.Index)
			if pr.maybeDecrTo(m.Index, m.RejectHint) {
				r.logger.Debugf("%x decreased progress of %x to [%s]", r.id, m.From, pr)
				if pr.State == ProgressStateReplicate {
					pr.becomeProbe()
				}
				r.sendAppend(m.From)
			}
		} else {
			oldPaused := pr.isPaused()
			if pr.maybeUpdate(m.Index) {
				switch {
				case pr.State == ProgressStateProbe:
					pr.becomeReplicate()
				case pr.State == ProgressStateSnapshot && pr.maybeSnapshotAbort():
					r.logger.Debugf("%x snapshot aborted, resumed sending replication messages to %x [%s]", r.id, m.From, pr)
					pr.becomeProbe()
				case pr.State == ProgressStateReplicate:
					pr.ins.freeTo(m.Index)
				}

				if r.maybeCommit() {
					r.bcastAppend()
				} else if oldPaused {
					// update() reset the wait state on this node. If we had delayed sending
					// an update before, send it now.
					r.sendAppend(m.From)
				}
			}
		}
	case pb.MsgHeartbeatResp: //收到心跳响应
		pr.RecentActive = true //修改为true，收到同步数据的响应时也会修改为true

		// free one slot for the full inflights window to allow progress.
		if pr.State == ProgressStateReplicate && pr.ins.full() {
			pr.ins.freeFirstOne() //释放空间
		}
		if pr.Match < r.raftLog.lastIndex() {
			r.sendAppend(m.From) //尚未与leader的数据保持一致同步数据
		}
	case pb.MsgSnapStatus: //收到快照响应 
		if pr.State != ProgressStateSnapshot { //如果当前不是ProgressStateSnapshot，拒绝此响应
			return
		}
		if !m.Reject { //其他节点接收快照同步
			pr.becomeProbe()
			r.logger.Debugf("%x snapshot succeeded, resumed sending replication messages to %x [%s]", r.id, m.From, pr)
		} else {//其他节点拒绝快照同步
			pr.snapshotFailure()
			pr.becomeProbe()
			r.logger.Debugf("%x snapshot failed, resumed sending replication messages to %x [%s]", r.id, m.From, pr)
		}
		// If snapshot finish, wait for the msgAppResp from the remote node before sending
		// out the next msgApp.
		// If snapshot failure, wait for a heartbeat interval before next try
		pr.pause()
	case pb.MsgUnreachable: //远端节点不可达
		if pr.State == ProgressStateReplicate {
			pr.becomeProbe()
		}
		r.logger.Debugf("%x failed to send message to %x because it is unreachable [%s]", r.id, m.From, pr)
	}
}
```

#### bcastHeartbeat

```go
func (r *raft) bcastHeartbeat() {
   for id := range r.prs {
      if id == r.id {
         continue
      }
      r.sendHeartbeat(id)
      r.prs[id].resume()
   }
}
```

#### checkQuorumActive

```go
func (r *raft) checkQuorumActive() bool {
   var act int
   for id := range r.prs {
      if id == r.id { // self is always active
         act++
         continue
      }
      if r.prs[id].RecentActive {
         act++
      }
      r.prs[id].RecentActive = false
   }
   return act >= r.quorum()
}
```

### stepFollower

```go
func stepFollower(r *raft, m pb.Message) {
   switch m.Type {
   case pb.MsgProp:
      if r.lead == None { //leader没有选举出来
         r.logger.Infof("%x no leader at term %d; dropping proposal", r.id, r.Term)
         return
      }
      m.To = r.lead
      r.send(m) //将MsgProp类型的消息发送给leader
   case pb.MsgApp:  //收到leader的数据同步请求
      r.electionElapsed = 0
      r.lead = m.From
      r.handleAppendEntries(m) //追加数据
   case pb.MsgHeartbeat: //收到leader的心跳请求
      r.electionElapsed = 0
      r.lead = m.From
      r.handleHeartbeat(m) //处理心跳
   case pb.MsgSnap: //收到leader的快照同步请求
      r.electionElapsed = 0
      r.handleSnapshot(m)
   case pb.MsgVote: //收到其他节点的投票请求
      if (r.Vote == None || r.Vote == m.From) && r.raftLog.isUpToDate(m.Index, m.LogTerm) {//没有收到其他节点的投票或者之前收到了此节点的投票请求，并且比本节点的数据更新
         r.electionElapsed = 0
         r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] voted for %x [logterm: %d, index: %d] at term %d",
            r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.From, m.LogTerm, m.Index, r.Term)
         r.Vote = m.From
         r.send(pb.Message{To: m.From, Type: pb.MsgVoteResp})
      } else { //已经收到过其他节点的投票，拒绝此投票请求
         r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected vote from %x [logterm: %d, index: %d] at term %d",
            r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.From, m.LogTerm, m.Index, r.Term)
         r.send(pb.Message{To: m.From, Type: pb.MsgVoteResp, Reject: true})
      }
   }
}
```

```go
func (l *raftLog) isUpToDate(lasti, term uint64) bool {
  //先比较term，在比较lastIndex
   return term > l.lastTerm() || (term == l.lastTerm() && lasti >= l.lastIndex())
}
```

#### handleAppendEntries

```go
func (r *raft) handleAppendEntries(m pb.Message) {
   if m.Index < r.raftLog.committed {
     //防止数据重复同步
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
      return
   }

   if mlastIndex, ok := r.raftLog.maybeAppend(m.Index, m.LogTerm, m.Commit, m.Entries...); ok {
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: mlastIndex}) //返回最后一条记录的索引
   } else { //同步的数据有冲突
      r.logger.Debugf("%x [logterm: %d, index: %d] rejected msgApp [logterm: %d, index: %d] from %x",
         r.id, r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(m.Index)), m.Index, m.LogTerm, m.Index, m.From)
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: m.Index, Reject: true, RejectHint: r.raftLog.lastIndex()})
   }
}
```

etcd/raft/log.go:76

```go
func (l *raftLog) maybeAppend(index, logTerm, committed uint64, ents ...pb.Entry) (lastnewi uint64, ok bool) {
   lastnewi = index + uint64(len(ents))
   if l.matchTerm(index, logTerm) { //查看index处的term是否与请求中的term相同，避免数据出现遗漏
      ci := l.findConflict(ents) //查看是否有冲突
      switch {
      case ci == 0: //无冲突
      case ci <= l.committed:
         l.logger.Panicf("entry %d conflict with committed entry [committed(%d)]", ci, l.committed)
      default:
         offset := index + 1
         l.append(ents[ci-offset:]...)
      }
      l.commitTo(min(committed, lastnewi)) //更新committed
      return lastnewi, true
   }
   return 0, false
}
```

#### handleHeartbeat

```go
func (r *raft) handleHeartbeat(m pb.Message) {
   r.raftLog.commitTo(m.Commit)
   r.send(pb.Message{To: m.From, Type: pb.MsgHeartbeatResp})
}
```

#### handleSnapshot

```go
func (r *raft) handleSnapshot(m pb.Message) {
   sindex, sterm := m.Snapshot.Metadata.Index, m.Snapshot.Metadata.Term
   if r.restore(m.Snapshot) {
      r.logger.Infof("%x [commit: %d] restored snapshot [index: %d, term: %d]",
         r.id, r.raftLog.committed, sindex, sterm)
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.lastIndex()})
   } else {
      r.logger.Infof("%x [commit: %d] ignored snapshot [index: %d, term: %d]",
         r.id, r.raftLog.committed, sindex, sterm)
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
   }
}
```

# ClientHandler

```go
func NewClientHandler(server *etcdserver.EtcdServer, timeout time.Duration) http.Handler {
   //负责处理http请求
   go capabilityLoop(server)

   sec := auth.NewStore(server, timeout)

   kh := &keysHandler{
      sec:     sec,
      server:  server,
      cluster: server.Cluster(),
      timer:   server,
      timeout: timeout,
   }

   sh := &statsHandler{
      stats: server,
   }

   mh := &membersHandler{
      sec:     sec,
      server:  server,
      cluster: server.Cluster(),
      timeout: timeout,
      clock:   clockwork.NewRealClock(),
   }

   dmh := &deprecatedMachinesHandler{
      cluster: server.Cluster(),
   }

   sech := &authHandler{
      sec:     sec,
      cluster: server.Cluster(),
   }

   mux := http.NewServeMux()
   mux.HandleFunc("/", http.NotFound)
   mux.Handle(healthPath, healthHandler(server))
   mux.HandleFunc(versionPath, versionHandler(server.Cluster(), serveVersion))
   mux.Handle(keysPrefix, kh)
   mux.Handle(keysPrefix+"/", kh)
   mux.HandleFunc(statsPrefix+"/store", sh.serveStore)
   mux.HandleFunc(statsPrefix+"/self", sh.serveSelf)
   mux.HandleFunc(statsPrefix+"/leader", sh.serveLeader)
   mux.HandleFunc(varsPath, serveVars)
   mux.HandleFunc(configPath+"/local/log", logHandleFunc)
   mux.Handle(metricsPath, prometheus.Handler())
   mux.Handle(membersPrefix, mh)
   mux.Handle(membersPrefix+"/", mh)
   mux.Handle(deprecatedMachinesPrefix, dmh)
   handleAuth(mux, sech)

   if server.IsPprofEnabled() {
      plog.Infof("pprof is enabled under %s", pprofPrefix)

      mux.HandleFunc(pprofPrefix, pprof.Index)
      mux.HandleFunc(pprofPrefix+"/profile", pprof.Profile)
      mux.HandleFunc(pprofPrefix+"/symbol", pprof.Symbol)
      mux.HandleFunc(pprofPrefix+"/cmdline", pprof.Cmdline)
      // TODO: currently, we don't create an entry for pprof.Trace,
      // because go 1.4 doesn't provide it. After support of go 1.4 is dropped,
      // we should add the entry.

      mux.Handle(pprofPrefix+"/heap", pprof.Handler("heap"))
      mux.Handle(pprofPrefix+"/goroutine", pprof.Handler("goroutine"))
      mux.Handle(pprofPrefix+"/threadcreate", pprof.Handler("threadcreate"))
      mux.Handle(pprofPrefix+"/block", pprof.Handler("block"))
   }

   return requestLogger(mux)
}
```

## keysHandler

etcd/etcdserver/etcdhttp/client.go

```go
func (h *keysHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   if !allowMethod(w, r.Method, "HEAD", "GET", "PUT", "POST", "DELETE") {
      return
   }

   w.Header().Set("X-Etcd-Cluster-ID", h.cluster.ID().String())

   ctx, cancel := context.WithTimeout(context.Background(), h.timeout)
   defer cancel()
   clock := clockwork.NewRealClock()
   startTime := clock.Now()
   rr, err := parseKeyRequest(r, clock)
   if err != nil {
      writeKeyError(w, err)
      return
   }
   // The path must be valid at this point (we've parsed the request successfully).
   if !hasKeyPrefixAccess(h.sec, r, r.URL.Path[len(keysPrefix):], rr.Recursive) {
      writeKeyNoAuth(w)
      return
   }
   if !rr.Wait {
      reportRequestReceived(rr)
   }
   resp, err := h.server.Do(ctx, rr)
   if err != nil {
      err = trimErrorPrefix(err, etcdserver.StoreKeysPrefix)
      writeKeyError(w, err)
      reportRequestFailed(rr, err)
      return
   }
   switch {
   case resp.Event != nil:
      if err := writeKeyEvent(w, resp.Event, h.timer); err != nil {
         // Should never be reached
         plog.Errorf("error writing event (%v)", err)
      }
      reportRequestCompleted(rr, resp, startTime)
   case resp.Watcher != nil:
      ctx, cancel := context.WithTimeout(context.Background(), defaultWatchTimeout)
      defer cancel()
      handleKeyWatch(ctx, w, resp.Watcher, rr.Stream, h.timer)
   default:
      writeKeyError(w, errors.New("received response with no Event/Watcher!"))
   }
}
```

etcdserver/server.go:

```go
func (s *EtcdServer) Do(ctx context.Context, r pb.Request) (Response, error) {
   r.ID = s.reqIDGen.Next()
   if r.Method == "GET" && r.Quorum {
      r.Method = "QGET"
   }
   switch r.Method {
   case "POST", "PUT", "DELETE", "QGET": //变更数据
      data, err := r.Marshal()
      if err != nil {
         return Response{}, err
      }
      ch := s.w.Register(r.ID)
     s.r.Propose(ctx, data) //处理请求(可能来自客户端或者Follower节点)
      select {
      case x := <-ch://等待响应
         resp := x.(Response)
         return resp, resp.err
      case <-ctx.Done():
         s.w.Trigger(r.ID, nil) // GC wait
         return Response{}, parseCtxErr(ctx.Err())
      case <-s.done:
         return Response{}, ErrStopped
      }
   case "GET":
      switch {
      case r.Wait: //注册watch
         wc, err := s.store.Watch(r.Path, r.Recursive, r.Stream, r.Since)
         if err != nil {
            return Response{}, err
         }
         return Response{Watcher: wc}, nil
      default: //获取数据
         ev, err := s.store.Get(r.Path, r.Recursive, r.Sorted)
         if err != nil {
            return Response{}, err
         }
         return Response{Event: ev}, nil
      }
   case "HEAD":
      ev, err := s.store.Get(r.Path, r.Recursive, r.Sorted)
      if err != nil {
         return Response{}, err
      }
      return Response{Event: ev}, nil
   default:
      return Response{}, ErrUnknownMethod
   }
}
```

raft/node.go

```go
func (n *node) Propose(ctx context.Context, data []byte) error {
   return n.step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}
```

```go
func (n *node) step(ctx context.Context, m pb.Message) error {
   ch := n.recvc 
   if m.Type == pb.MsgProp {
      ch = n.propc
   }
   select {
   case ch <- m: //将消息写入recvc或者propc通道
      return nil
   case <-ctx.Done():
      return ctx.Err()
   case <-n.done:
      return ErrStopped
   }
}
```

获取数据

store/store.go

```go
func (s *store) Get(nodePath string, recursive, sorted bool) (*Event, error) {
   s.worldLock.RLock()
   defer s.worldLock.RUnlock()

   nodePath = path.Clean(path.Join("/", nodePath))

   n, err := s.internalGet(nodePath)

   if err != nil {
      s.Stats.Inc(GetFail)
      return nil, err
   }

   e := newEvent(Get, nodePath, n.ModifiedIndex, n.CreatedIndex)
   e.EtcdIndex = s.CurrentIndex
   e.Node.loadInternalNode(n, recursive, sorted, s.clock)

   s.Stats.Inc(GetSuccess)

   return e, nil
}
```

```go
func stepLeader(r *raft, m pb.Message) {//leader处理请求
   switch m.Type {
   case pb.MsgBeat: //心跳请求
      r.bcastHeartbeat()
   case pb.MsgProp: //事务类请求
      if len(m.Entries) == 0 {
         log.Panicf("raft: %x stepped empty MsgProp", r.id)
      }
     //配置变更
      for i, e := range m.Entries {
         if e.Type == pb.EntryConfChange {
            if r.pendingConf {
               m.Entries[i] = pb.Entry{Type: pb.EntryNormal}
            }
            r.pendingConf = true
         }
      }
     //追加entries
      r.appendEntry(m.Entries...)
     //复制到其他节点
      r.bcastAppend()
   case pb.MsgAppResp: //同步响应
      if m.Reject { //拒绝同步请求
         log.Printf("raft: %x received msgApp rejection(lastindex: %d) from %x for index %d",
            r.id, m.RejectHint, m.From, m.Index)
         if r.prs[m.From].maybeDecrTo(m.Index, m.RejectHint) {
            log.Printf("raft: %x decreased progress of %x to [%s]", r.id, m.From, r.prs[m.From])
            r.sendAppend(m.From)
         }
      } else {
         oldWait := r.prs[m.From].shouldWait()
         r.prs[m.From].update(m.Index)
         if r.maybeCommit() {
            r.bcastAppend()
         } else if oldWait {
            // update() reset the wait state on this node. If we had delayed sending
            // an update before, send it now.
            r.sendAppend(m.From)
         }
      }
   case pb.MsgHeartbeatResp: //心跳响应
      if r.prs[m.From].Match < r.raftLog.lastIndex() {
         r.sendAppend(m.From)
      }
   case pb.MsgVote: //投票请求
      log.Printf("raft: %x [logterm: %d, index: %d, vote: %x] rejected vote from %x [logterm: %d, index: %d] at term %d",
         r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.From, m.LogTerm, m.Index, r.Term)
      r.send(pb.Message{To: m.From, Type: pb.MsgVoteResp, Reject: true})
   }
}
```

appendEntry

```go
func (r *raft) appendEntry(es ...pb.Entry) {
 //最新的追加索引
   li := r.raftLog.lastIndex()
  //修改entry的term和index
   for i := range es {
      es[i].Term = r.Term
      es[i].Index = li + 1 + uint64(i)
   }
  //追加
   r.raftLog.append(es...)
   r.prs[r.id].update(r.raftLog.lastIndex())
  //可能修改committed（日志项已经复制到过半节点）
   r.maybeCommit()
}
```

raft/log.go

```go
func (l *raftLog) append(ents ...pb.Entry) uint64 {
   if len(ents) == 0 {
      return l.lastIndex()
   }
   if after := ents[0].Index - 1; after < l.committed {
      log.Panicf("after(%d) is out of range [committed(%d)]", after, l.committed)
   }
   l.unstable.truncateAndAppend(ents)
   return l.lastIndex()
}
```

raft/log_unstable.go

```go
func (u *unstable) truncateAndAppend(ents []pb.Entry) {
   after := ents[0].Index - 1
   switch {
   case after == u.offset+uint64(len(u.entries))-1:
      // after is the last index in the u.entries
      // directly append
      u.entries = append(u.entries, ents...)
   case after < u.offset:
      log.Printf("raftlog: replace the unstable entries from index %d", after+1)
      // The log is being truncated to before our current offset
      // portion, so set the offset and replace the entries
      u.offset = after + 1
      u.entries = ents
   default:
      // truncate to after and copy to u.entries
      // then append
      log.Printf("raftlog: truncate the unstable entries to index %d", after)
      u.entries = append([]pb.Entry{}, u.slice(u.offset, after+1)...)
      u.entries = append(u.entries, ents...)
   }
}
```

同步数据

```go
func (r *raft) bcastAppend() { //同步数据
   for i := range r.prs {
      if i == r.id {
         continue
      }
      r.sendAppend(i)
   }
}
```

```go
func (r *raft) sendAppend(to uint64) {
   pr := r.prs[to]
   if pr.shouldWait() {
      return
   }
   m := pb.Message{}
   m.To = to
   if r.needSnapshot(pr.Next) {
      m.Type = pb.MsgSnap//同步快照
      snapshot, err := r.raftLog.snapshot()
      if err != nil {
         panic(err) // TODO(bdarnell)
      }
      if IsEmptySnap(snapshot) {
         panic("need non-empty snapshot")
      }
      m.Snapshot = snapshot
      sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
      log.Printf("raft: %x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
         r.id, r.raftLog.firstIndex(), r.Commit, sindex, sterm, to, pr)
      pr.waitSet(r.electionTimeout)
   } else { //同步日志项
      m.Type = pb.MsgApp
      m.Index = pr.Next - 1
      m.LogTerm = r.raftLog.term(pr.Next - 1)
      m.Entries = r.raftLog.entries(pr.Next)
      m.Commit = r.raftLog.committed
      // optimistically increase the next if the follower
      // has been matched.
      if n := len(m.Entries); pr.Match != 0 && n != 0 {
         pr.optimisticUpdate(m.Entries[n-1].Index)
      } else if pr.Match == 0 {
         // TODO (xiangli): better way to find out if the follower is in good path or not
         // a follower might be in bad path even if match != 0, since we optimistically
         // increase the next.
         pr.waitSet(r.heartbeatTimeout)
      }
   }
  //发送
   r.send(m)
}
```

# 租约服务

 etcd server 保证在约定的有效期内（TTL），不会删除你关联到此 Lease 上的 key-value

不同 key 若 TTL 相同，可复用同一个 Lease，显著减少了 Lease 数量

通过 gRPC HTTP/2 实现了多路复用，流式传输，同一连接可支持为多个 Lease 续期，大大减少了连接数

**存在问题：1、查找过期的Lease时间复杂度为O（n），需要遍历所有的Lease**

​					**2、每个Lease只会存储TTL时间，并不会存储剩余存活时间，当重建的时候相当于自动续期，导致					Lease无法删除，大量的key堆积，db 大小超过配额**

## 创建GRPC

etcd/etcdmain/etcd.go

```go
func startEtcd(cfg *config) (<-chan struct{}, error) {//截取lease相关的代码
		if cfg.v3demo {
			tls := &cfg.clientTLSInfo
			if cfg.clientTLSInfo.Empty() {
				tls = nil
		 }
		grpcServer, err := v3rpc.Server(s, tls)
		if err != nil {
				s.Stop()
				<-s.StopNotify()
				return nil, err
		}
		go func() { plog.Fatal(grpcServer.Serve(v3l)) }()
	}
}
```

```go
func Server(s *etcdserver.EtcdServer, tls *transport.TLSInfo) (*grpc.Server, error) {
   var opts []grpc.ServerOption
   if tls != nil {
      creds, err := credentials.NewServerTLSFromFile(tls.CertFile, tls.KeyFile)
      if err != nil {
         return nil, err
      }
      opts = append(opts, grpc.Creds(creds))
   }

   grpcServer := grpc.NewServer(opts...)
   pb.RegisterKVServer(grpcServer, NewKVServer(s))
   pb.RegisterWatchServer(grpcServer, NewWatchServer(s))
   pb.RegisterLeaseServer(grpcServer, NewLeaseServer(s)) //注册lease服务
   pb.RegisterClusterServer(grpcServer, NewClusterServer(s))
   pb.RegisterAuthServer(grpcServer, NewAuthServer(s))
   pb.RegisterMaintenanceServer(grpcServer, NewMaintenanceServer(s))
   return grpcServer, nil
}
```

etcd/etcdserver/api/v3rpc/lease.go:31

```go
func NewLeaseServer(le etcdserver.Lessor) pb.LeaseServer {
   return &LeaseServer{le: le/*EtcdServer*/}
}
```

## 堆外暴露服务

### 创建租约

```go
func (ls *LeaseServer) LeaseCreate(ctx context.Context, cr *pb.LeaseCreateRequest) (*pb.LeaseCreateResponse, error) {	//创建租约，走raft协议
  
   resp, err := ls.le.LeaseCreate(ctx, cr)
   if err == lease.ErrLeaseExists {
      return nil, rpctypes.ErrLeaseExist
   }
   return resp, err
}
```

etcd/etcdserver/v3_server.go

```go
func (s *EtcdServer) LeaseCreate(ctx context.Context, r *pb.LeaseCreateRequest) (*pb.LeaseCreateResponse, error) {
   for r.ID == int64(lease.NoLease) {
      r.ID = int64(s.reqIDGen.Next() & ((1 << 63) - 1)) //生成Lease的唯一标志
   }
  //走raft协议，进行持久化
   result, err := s.processInternalRaftRequest(ctx, pb.InternalRaftRequest{LeaseCreate: r})
   if err != nil {
      return nil, err
   }
   return result.resp.(*pb.LeaseCreateResponse), result.err
}
```

```go
func (s *EtcdServer) processInternalRaftRequest(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error) {
  	//生成请求Id 
  r.ID = s.reqIDGen.Next()

   data, err := r.Marshal()
   if err != nil {
      return nil, err
   }
	//请求数据不能超过1.5M
   if len(data) > maxRequestBytes {
      return nil, ErrRequestTooLarge
   }
	//维护未完成的请求
   ch := s.w.Register(r.ID)
	//处理事务类请求
   s.r.Propose(ctx, data)

   select {
   case x := <-ch: //等待请求处理完成
      return x.(*applyResult), nil
   case <-ctx.Done():
      s.w.Trigger(r.ID, nil) // GC wait
      return nil, ctx.Err()
   case <-s.done:
      return nil, ErrStopped
   }
}
```

etcd/etcdserver/v3_server.go

```go
func applyLeaseCreate(le lease.Lessor, lc *pb.LeaseCreateRequest) (*pb.LeaseCreateResponse, error) {
   l, err := le.Grant(lease.LeaseID(lc.ID), lc.TTL)
   resp := &pb.LeaseCreateResponse{}
   if err == nil {
      resp.ID = int64(l.ID)
      resp.TTL = l.TTL
   }
   return resp, err
}
```

etcd/lease/lessor.go

```go
func (le *lessor) Grant(id LeaseID, ttl int64) (*Lease, error) {
   if id == NoLease {
      return nil, ErrLeaseNotFound
   }
	//创建Lease对象，包含唯一标志、存活时间、关联的key
   l := &Lease{ID: id, TTL: ttl, itemSet: make(map[LeaseItem]struct{})}

   le.mu.Lock()
   defer le.mu.Unlock()

   if _, ok := le.leaseMap[id]; ok {
      return nil, ErrLeaseExists //租约已经存在
   }

   if le.primary {
      l.refresh(0)
   } else {
      l.forever()
   }

   le.leaseMap[id] = l //保存到map
   l.persistTo(le.b) //保存到boltdb

   return l, nil
}
```

### 删除租约

```go
func (ls *LeaseServer) LeaseRevoke(ctx context.Context, rr *pb.LeaseRevokeRequest) (*pb.LeaseRevokeResponse, error) { //删除租约，走raft协议
   r, err := ls.le.LeaseRevoke(ctx, rr)
   if err != nil {
      return nil, rpctypes.ErrLeaseNotFound
   }
   return r, nil
}
```

```go
func (s *EtcdServer) LeaseRevoke(ctx context.Context, r *pb.LeaseRevokeRequest) (*pb.LeaseRevokeResponse, error) {
	result, err := s.processInternalRaftRequest(ctx, pb.InternalRaftRequest{LeaseRevoke: r})
	if err != nil {
		return nil, err
	}
	if result.err != nil {
		return nil, result.err
	}
	return result.resp.(*pb.LeaseRevokeResponse), nil
}
```

### 续租

```go
func (ls *LeaseServer) LeaseKeepAlive(stream pb.Lease_LeaseKeepAliveServer) error {
  //续租请求交由leader处理，不会走raft协议（因为续约相对比较频繁，如果也走raft协议，会占用大量的磁盘和网络，导致db不断膨胀甚至超过配额）
   for {
      req, err := stream.Recv()
      if err == io.EOF {
         return nil
      }
      if err != nil {
         return err
      }

      ttl, err := ls.le.LeaseRenew(lease.LeaseID(req.ID))
      if err == lease.ErrLeaseNotFound {
         return rpctypes.ErrLeaseNotFound
      }

      if err != nil && err != lease.ErrLeaseNotFound {
         return err
      }

      resp := &pb.LeaseKeepAliveResponse{ID: req.ID, TTL: ttl}
      err = stream.Send(resp)
      if err != nil {
         return err
      }
   }
}
```

etcd/etcdserver/v3demo_server.go

```go
func (s *EtcdServer) LeaseRenew(ctx context.Context, id lease.LeaseID) (int64, error) {
	ttl, err := s.lessor.Renew(id) //本地续约
	if err == nil {
		return ttl, nil
	}
	if err != lease.ErrNotPrimary {
		return -1, err
	}

	// 此请求不会走raft，直接转发给Leader
	cctx, cancel := context.WithTimeout(ctx, s.Cfg.ReqTimeout())
	defer cancel()

	// renewals don't go through raft; forward to leader manually
	for cctx.Err() == nil && err != nil {
		leader, lerr := s.waitLeader(cctx)
		if lerr != nil {
			return -1, lerr
		}
		for _, url := range leader.PeerURLs {
			lurl := url + "/leases"
			ttl, err = leasehttp.RenewHTTP(cctx, id, lurl, s.peerRt)
			if err == nil || err == lease.ErrLeaseNotFound {
				return ttl, err
			}
		}
	}
	return -1, ErrTimeout
}
```



# 租约

## Version-3.0

存在问题

1、重启之后TTL自动续期，可能导致Lease永不淘汰

2、寻找过期的Lease的时间复杂度为O(N)

### 创建Lessor

etcd/lease/lessor.go:148

```go
func NewLessor(b backend.Backend) Lessor {
   return newLessor(b)
}
```

```go
func newLessor(b backend.Backend) *lessor {
   l := &lessor{
      leaseMap: make(map[LeaseID]*Lease),
      b:        b,
      // expiredC is a small buffered chan to avoid unnecessary blocking.
      expiredC: make(chan []*Lease, 16),
      stopC:    make(chan struct{}),
      doneC:    make(chan struct{}),
   }
   l.initAndRecover() //恢复

   go l.runLoop() //启动协程，定期删除过期的lease

   return l
}
```

#### 初始化恢复

```go
func (le *lessor) initAndRecover() {
   tx := le.b.BatchTx()
   tx.Lock()
	//创建单独的bucket用于存放租约数据
   tx.UnsafeCreateBucket(leaseBucketName)
  //从单独的桶中获取存储的Lease（leaseId、TTL）
   _, vs := tx.UnsafeRange(leaseBucketName, int64ToBytes(0), int64ToBytes(math.MaxInt64), 0)
   for i := range vs {
      var lpb leasepb.Lease
      err := lpb.Unmarshal(vs[i])//反序列
      if err != nil {
         panic("failed to unmarshal lease proto item")
      }
      ID := LeaseID(lpb.ID)
      if lpb.TTL < le.minLeaseTTL {
         lpb.TTL = le.minLeaseTTL
      }
      le.leaseMap[ID] = &Lease{
         ID:  ID,
         ttl: lpb.TTL, //存活时间，重启之后相当于延长了存活时间
         itemSet: make(map[LeaseItem]struct{}), //存放关联的key
         expiry:  forever,
         revokec: make(chan struct{}),
      }
   }
   tx.Unlock()

   le.b.ForceCommit()
}
```

#### 定时删除过期Lease

```go
func (le *lessor) runLoop() {
   defer close(le.doneC)

   for {
      var ls []*Lease

      le.mu.Lock()
      if le.primary { //leader节点
         ls = le.findExpiredLeases() //查找过期的租约
      }
      le.mu.Unlock()

      if len(ls) != 0 {
         select {
         case <-le.stopC:
            return
         case le.expiredC <- ls://将过期的lease放入chan，交由EtcdServer处理
         default:
         }
      }

      select {
      case <-time.After(500 * time.Millisecond):
      case <-le.stopC:
         return
      }
   }
}
```



```go
func (le *lessor) findExpiredLeases() []*Lease { //查找过期的lease，复杂度O(n)后续改为最小堆
   leases := make([]*Lease, 0, 16)
   now := time.Now()

   for _, l := range le.leaseMap {
      if l.expiry.Sub(now) <= 0 {
         leases = append(leases, l)
      }
   }

   return leases
}
```

etcd/etcdserver/server.go:506

```go
func (s *EtcdServer) run() { //截取lease撤销相关的代码
  var expiredLeaseC <-chan []*lease.Lease
  if s.lessor != nil {
     expiredLeaseC = s.lessor.ExpiredLeasesC()
  }

  for {
     case leases := <-expiredLeaseC:
        go func() {
           for _, l := range leases {
             //撤销lease，走raft协议
              s.LeaseRevoke(context.TODO(), &pb.LeaseRevokeRequest{ID: int64(l.ID)})
           }
        }()
  }
}
```

### Revoke

etcd/lease/lessor.go:

```go
func (le *lessor) Revoke(id LeaseID) error {
   le.mu.Lock()
   l := le.leaseMap[id]
   if l == nil { 
      le.mu.Unlock()
      return ErrLeaseNotFound
   }
   le.mu.Unlock()

   if le.rd != nil {
      for item := range l.itemSet { //关联的key
         le.rd.DeleteRange([]byte(item.Key), nil)
      }
   }

   le.mu.Lock()
   defer le.mu.Unlock()
   delete(le.leaseMap, l.ID) //从map移除
   l.removeFrom(le.b) //从boltdb删除

   return nil
}
```

### 关联key

```go
func (le *lessor) Attach(id LeaseID, items []LeaseItem) error {//key管理leaseId
   le.mu.Lock()
   defer le.mu.Unlock()

   l := le.leaseMap[id]  //leaseId -> Lease
   if l == nil {
      return ErrLeaseNotFound
   }

   for _, it := range items {
      l.itemSet[it] = struct{}{}  //将items和lease绑定
   }
   return nil
}
```

### 取消关联key

```go
func (le *lessor) Detach(id LeaseID, items []LeaseItem) error {//取消key与leaseId的关联
   le.mu.Lock()
   defer le.mu.Unlock()

   l := le.leaseMap[id]
   if l == nil {
      return ErrLeaseNotFound
   }

   for _, it := range items {
      delete(l.itemSet, it)
   }
   return nil
}
```

## Version-3.4

1、用小顶堆维护Lease，寻找过期Lease的时间复杂度降为O(logN)

### 创建Lessor

```java
func newLessor(lg *zap.Logger, b backend.Backend, cfg LessorConfig) *lessor {
   checkpointInterval := cfg.CheckpointInterval
   expiredLeaseRetryInterval := cfg.ExpiredLeasesRetryInterval
   if checkpointInterval == 0 {
      checkpointInterval = defaultLeaseCheckpointInterval //定时做checkpoint的间隔
   }
   if expiredLeaseRetryInterval == 0 {
      expiredLeaseRetryInterval = defaultExpiredleaseRetryInterval //定时删除过期租约的间隔
   }
   l := &lessor{
      leaseMap:                  make(map[LeaseID]*Lease),
      itemMap:                   make(map[LeaseItem]LeaseID),
      leaseExpiredNotifier:      newLeaseExpiredNotifier(), //小顶堆存放租约
      leaseCheckpointHeap:       make(LeaseQueue, 0), //存放租约的数组
      b:                         b,
      minLeaseTTL:               cfg.MinLeaseTTL,
      checkpointInterval:        checkpointInterval,
      expiredLeaseRetryInterval: expiredLeaseRetryInterval,
      // expiredC is a small buffered chan to avoid unnecessary blocking.
      expiredC: make(chan []*Lease, 16),
      stopC:    make(chan struct{}),
      doneC:    make(chan struct{}),
      lg:       lg,
   }
   l.initAndRecover()

   go l.runLoop()

   return l
}
```

#### 初始化恢复

```go
func (le *lessor) initAndRecover() {
  //从boltdb查找lease
   tx := le.b.BatchTx()
   tx.Lock()

   tx.UnsafeCreateBucket(leaseBucketName)
   _, vs := tx.UnsafeRange(leaseBucketName, int64ToBytes(0), int64ToBytes(math.MaxInt64), 0)
   // TODO: copy vs and do decoding outside tx lock if lock contention becomes an issue.
   for i := range vs {
      var lpb leasepb.Lease
      err := lpb.Unmarshal(vs[i])
      if err != nil {
         tx.Unlock()
         panic("failed to unmarshal lease proto item")
      }
      ID := LeaseID(lpb.ID)
      if lpb.TTL < le.minLeaseTTL {
         lpb.TTL = le.minLeaseTTL
      }
      le.leaseMap[ID] = &Lease{
         ID:  ID,
         ttl: lpb.TTL,
         // itemSet will be filled in when recover key-value pairs
         // set expiry to forever, refresh when promoted
         itemSet: make(map[LeaseItem]struct{}),
         expiry:  forever,
         revokec: make(chan struct{}),
      }
   }
   le.leaseExpiredNotifier.Init() //初始化leaseExpiredNotifier
   heap.Init(&le.leaseCheckpointHeap)
   tx.Unlock()

   le.b.ForceCommit()
}
```

#### 删除过期租约

```go
func (le *lessor) runLoop() {
   defer close(le.doneC)

   for {
      le.revokeExpiredLeases() //删除过期租约
      le.checkpointScheduledLeases() //做checkpoint

      select {
      case <-time.After(500 * time.Millisecond):
      case <-le.stopC:
         return
      }
   }
}
```

##### 删除过期租约

```go
func (le *lessor) revokeExpiredLeases() {
   var ls []*Lease

   //限速：一次删除上限500
   revokeLimit := leaseRevokeRate / 2

   le.mu.RLock()
   if le.isPrimary() {
      ls = le.findExpiredLeases(revokeLimit) //查找过期租约
   }
   le.mu.RUnlock()

   if len(ls) != 0 {
      select {
      case <-le.stopC:
         return
      case le.expiredC <- ls:
      default:
         // the receiver of expiredC is probably busy handling
         // other stuff
         // let's try this next time after 500ms
      }
   }
}
```

```go
func (le *lessor) findExpiredLeases(limit int) []*Lease {
   leases := make([]*Lease, 0, 16)

   for {
      l, ok, next := le.expireExists() //查找过期的租约
      if !ok && !next {
         break
      }
      if !ok {
         continue
      }
      if next {
         continue
      }

      if l.expired() { //过期
         leases = append(leases, l)
         if len(leases) == limit { //达到上限
            break
         }
      }
   }

   return leases
}
```

```go
func (le *lessor) expireExists() (l *Lease, ok bool, next bool) { //从小顶堆的堆顶获取即将过期的租约
   if le.leaseExpiredNotifier.Len() == 0 {
      return nil, false, false
   }

   item := le.leaseExpiredNotifier.Poll()
   l = le.leaseMap[item.id]
   if l == nil {
      // lease has expired or been revoked
      // no need to revoke (nothing is expiry)
      le.leaseExpiredNotifier.Unregister() // O(log N)
      return nil, false, true
   }
   now := time.Now()
   if now.UnixNano() < item.time /* expiration time */ {
      // Candidate expirations are caught up, reinsert this item
      // and no need to revoke (nothing is expiry)
      return l, false, false
   }

   // recheck if revoke is complete after retry interval
   item.time = now.Add(le.expiredLeaseRetryInterval).UnixNano()
   le.leaseExpiredNotifier.RegisterOrUpdate(item)
   return l, true, false
}
```

### 创建租约

```go
func (le *lessor) Grant(id LeaseID, ttl int64) (*Lease, error) {
   if id == NoLease {
      return nil, ErrLeaseNotFound
   }

   if ttl > MaxLeaseTTL {
      return nil, ErrLeaseTTLTooLarge
   }

   // TODO: when lessor is under high load, it should give out lease
   // with longer TTL to reduce renew load.
   l := &Lease{
      ID:      id,
      ttl:     ttl,
      itemSet: make(map[LeaseItem]struct{}),
      revokec: make(chan struct{}),
   }

   le.mu.Lock()
   defer le.mu.Unlock()

   if _, ok := le.leaseMap[id]; ok {
      return nil, ErrLeaseExists
   }

   if l.ttl < le.minLeaseTTL {
      l.ttl = le.minLeaseTTL
   }

   if le.isPrimary() {
      l.refresh(0)
   } else {
      l.forever()
   }

   le.leaseMap[id] = l
   item := &LeaseWithTime{id: l.ID, time: l.expiry.UnixNano()}
   le.leaseExpiredNotifier.RegisterOrUpdate(item) //注册或更新
   l.persistTo(le.b) //持久化

   leaseTotalTTLs.Observe(float64(l.ttl))
   leaseGranted.Inc()

   if le.isPrimary() {
      le.scheduleCheckpointIfNeeded(l) //记录做checkpoint的租约
   }

   return l, nil
}
```

#### 小顶堆维护

```go
func (mq *LeaseExpiredNotifier) RegisterOrUpdate(item *LeaseWithTime) {
   if old, ok := mq.m[item.id]; ok { //已经存在
      old.time = item.time //修改过期时间
      heap.Fix(&mq.queue, old.index) //调整小顶堆
   } else { //不存在
      heap.Push(&mq.queue, item) //放入小顶堆
      mq.m[item.id] = item
   }
}
```

#### checkpoint记录

```go
func (le *lessor) scheduleCheckpointIfNeeded(lease *Lease) {
   if le.cp == nil {
      return
   }

   if lease.RemainingTTL() > int64(le.checkpointInterval.Seconds()) { //剩余时间大于做checkpoint的间隔
      if le.lg != nil {
         le.lg.Debug("Scheduling lease checkpoint",
            zap.Int64("leaseID", int64(lease.ID)),
            zap.Duration("intervalSeconds", le.checkpointInterval),
         )
      }
     //维护小顶堆
      heap.Push(&le.leaseCheckpointHeap, &LeaseWithTime{
         id:   lease.ID,
         time: time.Now().Add(le.checkpointInterval).UnixNano(),
      })
   }
}
```

# 接收v3版本的客户端请求

etcd/etcdserver/v3demo_server.go

```go
func (s *EtcdServer) applyV3Request(r *pb.InternalRaftRequest) interface{} {
   kv := s.getKV()
   le := s.lessor

   ar := &applyResult{}

   switch {
   case r.Range != nil:
      ar.resp, ar.err = applyRange(noTxn, kv, r.Range)
   case r.Put != nil:
      ar.resp, ar.err = applyPut(noTxn, kv, le, r.Put) //key关联leaseId
   case r.DeleteRange != nil:
      ar.resp, ar.err = applyDeleteRange(noTxn, kv, r.DeleteRange)
   case r.Txn != nil:
      ar.resp, ar.err = applyTxn(kv, le, r.Txn)
   case r.Compaction != nil:
      ar.resp, ar.err = applyCompaction(kv, r.Compaction)
   case r.LeaseCreate != nil:
      ar.resp, ar.err = applyLeaseCreate(le, r.LeaseCreate)
   case r.LeaseRevoke != nil:
      ar.resp, ar.err = applyLeaseRevoke(le, r.LeaseRevoke)
   case r.AuthEnable != nil:
      ar.resp, ar.err = applyAuthEnable(s)
   default:
      panic("not implemented")
   }

   return ar
}
```

```go
func applyPut(txnID int64, kv dstorage.KV, le lease.Lessor, p *pb.PutRequest) (*pb.PutResponse, error) {
   resp := &pb.PutResponse{}
   resp.Header = &pb.ResponseHeader{}
   var (
      rev int64
      err error
   )
   if txnID != noTxn {
      rev, err = kv.TxnPut(txnID, p.Key, p.Value, lease.LeaseID(p.Lease))
      if err != nil {
         return nil, err
      }
   } else { //默认noTxn
      leaseID := lease.LeaseID(p.Lease)
      if leaseID != lease.NoLease {
         if l := le.Lookup(leaseID); l == nil { //查找lease
            return nil, lease.ErrLeaseNotFound
         }
      }
      rev = kv.Put(p.Key, p.Value, leaseID)
   }
   resp.Header.Revision = rev
   return resp, nil
}
```

etcd/storage/kvstore.go

```go
func (s *store) Put(key, value []byte, lease lease.LeaseID) int64 {
   id := s.TxnBegin()
   s.put(key, value, lease)
   s.txnEnd(id)

   putCounter.Inc()

   return int64(s.currentRev.main)
}
```

```go
func (s *store) put(key, value []byte, leaseID lease.LeaseID) {
   rev := s.currentRev.main + 1
   c := rev
   oldLease := lease.NoLease

   // if the key exists before, use its previous created and
   // get its previous leaseID
   grev, created, ver, err := s.kvindex.Get(key, rev)
   if err == nil {
      c = created.main
      ibytes := newRevBytes()
      revToBytes(grev, ibytes)
      _, vs := s.tx.UnsafeRange(keyBucketName, ibytes, nil, 0)
      var kv storagepb.KeyValue
      if err = kv.Unmarshal(vs[0]); err != nil {
         log.Fatalf("storage: cannot unmarshal value: %v", err)
      }
      oldLease = lease.LeaseID(kv.Lease)
   }

   ibytes := newRevBytes()
   revToBytes(revision{main: rev, sub: s.currentRev.sub}, ibytes)

   ver = ver + 1
   kv := storagepb.KeyValue{
      Key:            key,
      Value:          value,
      CreateRevision: c,
      ModRevision:    rev,
      Version:        ver,
      Lease:          int64(leaseID),
   }

   d, err := kv.Marshal()
   if err != nil {
      log.Fatalf("storage: cannot marshal event: %v", err)
   }

   s.tx.UnsafeSeqPut(keyBucketName, ibytes, d)
   s.kvindex.Put(key, revision{main: rev, sub: s.currentRev.sub})
   s.changes = append(s.changes, kv)
   s.currentRev.sub += 1

   if oldLease != lease.NoLease {
      if s.le == nil {
         panic("no lessor to detach lease")
      }
			//取消key与leaseId的关联
      err = s.le.Detach(oldLease, []lease.LeaseItem{{Key: string(key)}})
      if err != nil {
         panic("unexpected error from lease detach")
      }
   }

   if leaseID != lease.NoLease {
      if s.le == nil {
         panic("no lessor to attach lease")
      }

      err = s.le.Attach(leaseID, []lease.LeaseItem{{Key: string(key)}})//key关联leaseId
      if err != nil {
         panic("unexpected error from lease Attach")
      }
   }
}
```

# Watch机制

## HTTP处理watch请求

服务端默认保留1000个历史event事件，如果超过1000，会出现event被覆盖的情况，也就是出现event丢失

基于HTTP的long poll实现，本质就是http长连接，每个watch请求都需要维护一个TCP连接，高并发下导致服务端消耗太多的资源维护TCP连接、

watch只能关注一个key以及子节点，不能同时关注多个key

curl http://127.0.0.1:2379/v2/keys/foo?wait=true

```go
func (h *keysHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   if !allowMethod(w, r.Method, "HEAD", "GET", "PUT", "POST", "DELETE") {
      return
   }

   w.Header().Set("X-Etcd-Cluster-ID", h.cluster.ID().String())

   ctx, cancel := context.WithTimeout(context.Background(), h.timeout)
   defer cancel()
   clock := clockwork.NewRealClock()
   startTime := clock.Now()
   rr, err := parseKeyRequest(r, clock)
   if err != nil {
      writeKeyError(w, err)
      return
   }
   // The path must be valid at this point (we've parsed the request successfully).
   if !hasKeyPrefixAccess(h.sec, r, r.URL.Path[len(keysPrefix):], rr.Recursive) {
      writeKeyNoAuth(w)
      return
   }
   if !rr.Wait {
      reportRequestReceived(rr)
   }
   resp, err := h.server.Do(ctx, rr)
   if err != nil {
      err = trimErrorPrefix(err, etcdserver.StoreKeysPrefix)
      writeKeyError(w, err)
      reportRequestFailed(rr, err)
      return
   }
   switch {
   case resp.Event != nil:
      if err := writeKeyEvent(w, resp.Event, h.timer); err != nil {
         // Should never be reached
         plog.Errorf("error writing event (%v)", err)
      }
      reportRequestCompleted(rr, resp, startTime)
   case resp.Watcher != nil: //watch请求,阻塞等待触发watch，直到超时
      ctx, cancel := context.WithTimeout(context.Background(), defaultWatchTimeout)
      defer cancel()
      handleKeyWatch(ctx, w, resp.Watcher, rr.Stream, h.timer)
   default:
      writeKeyError(w, errors.New("received response with no Event/Watcher!"))
   }
}
```



```go
func handleKeyWatch(ctx context.Context, w http.ResponseWriter, wa store.Watcher, stream bool, rt etcdserver.RaftTimer) {
   defer wa.Remove()
   ech := wa.EventChan()
   var nch <-chan bool
   if x, ok := w.(http.CloseNotifier); ok {
      nch = x.CloseNotify()
   }

   w.Header().Set("Content-Type", "application/json")
   w.Header().Set("X-Etcd-Index", fmt.Sprint(wa.StartIndex()))
   w.Header().Set("X-Raft-Index", fmt.Sprint(rt.Index()))
   w.Header().Set("X-Raft-Term", fmt.Sprint(rt.Term()))
   w.WriteHeader(http.StatusOK)

   // Ensure headers are flushed early, in case of long polling
   w.(http.Flusher).Flush()

   for {
      select {
      case <-nch:
         // Client closed connection. Nothing to do.
         return
      case <-ctx.Done(): //超时
         // Timed out. net/http will close the connection for us, so nothing to do.
         return
      case ev, ok := <-ech: //key发生了变更，触发了watch
         if !ok {
            return
         }
         ev = trimEventPrefix(ev, etcdserver.StoreKeysPrefix)
         if err := json.NewEncoder(w).Encode(ev); err != nil {
            plog.Warningf("error writing event (%v)", err)
            return
         }
         if !stream {
            return
         }
         w.(http.Flusher).Flush()
      }
   }
}
```

etcd/etcdserver/server.go:

```go
func (s *EtcdServer) Do(ctx context.Context, r pb.Request) (Response, error) {
  //watch相关部分代码
	r.ID = s.reqIDGen.Next()
	if r.Method == "GET" && r.Quorum {
		r.Method = "QGET"
	}
	switch r.Method {
  case "GET":
     switch {
       //
     case r.Wait://watch,不走raft
        wc, err := s.store.Watch(r.Path, r.Recursive, r.Stream, r.Since)
        if err != nil {
           return Response{}, err
        }
        return Response{Watcher: wc}, nil
     default://get 
        ev, err := s.store.Get(r.Path, r.Recursive, r.Sorted)
        if err != nil {
           return Response{}, err
        }
        return Response{Event: ev}, nil
     }
 }
```

### 注册watch

etcd/store/store.go

```go
func (s *store) Watch(key string, recursive, stream bool, sinceIndex uint64) (Watcher, error) {
   s.worldLock.RLock()
   defer s.worldLock.RUnlock()

   key = path.Clean(path.Join("/", key))
   if sinceIndex == 0 {
      sinceIndex = s.CurrentIndex + 1
   }
   // WatcherHub does not know about the current index, so we need to pass it in
   w, err := s.WatcherHub.watch(key, recursive, stream, sinceIndex, s.CurrentIndex)
   if err != nil {
      return nil, err
   }

   return w, nil
}
```



```go
func (wh *watcherHub) watch(key string, recursive, stream bool, index, storeIndex uint64) (Watcher, *etcdErr.Error) {
 //用map维护key所关联的watcher列表，一个key会被多个客户端注册watcher
   reportWatchRequest()
  //查找是否有大于index的historyEvent
   event, err := wh.EventHistory.scan(key, recursive, index)

   if err != nil {
      err.Index = storeIndex
      return nil, err
   }

   w := &watcher{
      eventChan:  make(chan *Event, 100), // use a buffered channel
      recursive:  recursive,
      stream:     stream,
      sinceIndex: index,
      startIndex: storeIndex,
      hub:        wh,
   }

   wh.mutex.Lock()
   defer wh.mutex.Unlock()
  
   if event != nil { //有大于index的event
      ne := event.Clone()
      ne.EtcdIndex = storeIndex
      w.eventChan <- ne
      return w, nil
   }
	//获取key对应的watcher列表
   l, ok := wh.watchers[key]

   var elem *list.Element

   if ok { // add the new watcher to the back of the list
      elem = l.PushBack(w) //将创建的watcher加入列表
   } else { // create a new list and add the new watcher
      l = list.New()
      elem = l.PushBack(w)
      wh.watchers[key] = l
   }

   w.remove = func() { //移除
      if w.removed { // 避免移除两次
         return
      }
      w.removed = true
      l.Remove(elem)
      atomic.AddInt64(&wh.count, -1)
      reportWatcherRemoved()
      if l.Len() == 0 {
         delete(wh.watchers, key)
      }
   }

   atomic.AddInt64(&wh.count, 1)
   reportWatcherAdded()

   return w, nil
}
```

### 通知客户端

etcd/store/watcher_hub.go

```go
func (wh *watcherHub) notify(e *Event) {//当key发生变更时，触发watcher
  //添加event到固定长度的环形队列，可能导致event丢失
   e = wh.EventHistory.addEvent(e) 
	//切分key，比如”/foo/bar“切分成/、/foo、/foo/bar
   segments := strings.Split(e.Node.Key, "/")
   currPath := "/"
   for _, segment := range segments {
      currPath = path.Join(currPath, segment)
      wh.notifyWatchers(e, currPath, false)
   }

```

```go
func (wh *watcherHub) notifyWatchers(e *Event, nodePath string, deleted bool) {
   wh.mutex.Lock()
   defer wh.mutex.Unlock()
   l, ok := wh.watchers[nodePath] //获取watcher
   if ok {
      curr := l.Front()
      for curr != nil {
         next := curr.Next() // save reference to the next one in the list
         w, _ := curr.Value.(*watcher)
         originalPath := (e.Node.Key == nodePath)
         if (originalPath || !isHidden(nodePath, e.Node.Key)) && w.notify(e, originalPath, deleted) {
            if !w.stream { // do not remove the stream watcher
               // if we successfully notify a watcher
               // we need to remove the watcher from the list
               // and decrease the counter
               w.removed = true
               l.Remove(curr) //移除watcher
               atomic.AddInt64(&wh.count, -1)
               reportWatcherRemoved()
            }
         }
         curr = next // update current to the next element in the list
      }
      if l.Len() == 0 {
         delete(wh.watchers, nodePath)
      }
   }
}
```

## etcd-2.3的watch实现机制

不仅支持对个单个key注册watcher，也可以对某个范围注册watcher并通过区间树进行维护

使用grpc处理watch请求，grpc使用了HTTP2.0的tcp多路复用，一个client的不同watch可以共享使用同一个tcp连接、

### serverWatchStream

etcd/etcdserver/api/v3rpc/watch.go

```go
func (ws *watchServer) Watch(stream pb.Watch_WatchServer) error {
  //为每个客户端创建一个serverWatchStream
   sws := serverWatchStream{
      clusterID:   ws.clusterID,
      memberID:    ws.memberID,
      raftTimer:   ws.raftTimer,
      gRPCStream:  stream,
      watchStream: ws.watchable.NewWatchStream(),
      // chan for sending control response like watcher created and canceled.
      ctrlStream: make(chan *pb.WatchResponse, ctrlStreamBufLen),
      progress:   make(map[storage.WatchID]bool),
      closec:     make(chan struct{}),
   }
   defer sws.close()

   go sws.sendLoop() //推送响应
   return sws.recvLoop() //接收请求
}
```

#### recvLoop

```go
func (sws *serverWatchStream) recvLoop() error {//处理收到的watch请求
   for {
      req, err := sws.gRPCStream.Recv()
      if err == io.EOF {
         return nil
      }
      if err != nil {
         return err
      }

      switch uv := req.RequestUnion.(type) {
      case *pb.WatchRequest_CreateRequest: //创建watch请求
         if uv.CreateRequest == nil {
            break
         }

         creq := uv.CreateRequest
         if len(creq.Key) == 0 {
            // \x00 is the smallest key
            creq.Key = []byte{0}
         }
         if len(creq.RangeEnd) == 1 && creq.RangeEnd[0] == 0 {
            // support  >= key queries
            creq.RangeEnd = []byte{}
         }
         wsrev := sws.watchStream.Rev()
         rev := creq.StartRevision
         if rev == 0 {
            rev = wsrev + 1
         }
        //创建watch，返回watch标志
         id := sws.watchStream.Watch(creq.Key, creq.RangeEnd, rev)
         if id != -1 && creq.ProgressNotify {
            sws.progress[id] = true
         }
        //将创建响应放入chan
         sws.ctrlStream <- &pb.WatchResponse{
            Header:   sws.newResponseHeader(wsrev),
            WatchId:  int64(id),
            Created:  true,
            Canceled: id == -1,
         }
      case *pb.WatchRequest_CancelRequest: //删除watch请求
         if uv.CancelRequest != nil {
            id := uv.CancelRequest.WatchId
           //根据watchId删除
            err := sws.watchStream.Cancel(storage.WatchID(id))
            if err == nil {
               sws.ctrlStream <- &pb.WatchResponse{
                  Header:   sws.newResponseHeader(sws.watchStream.Rev()),
                  WatchId:  id,
                  Canceled: true,
               }
               delete(sws.progress, storage.WatchID(id))
            }
         }
         // TODO: do we need to return error back to client?
      default:
         panic("not implemented")
      }
   }
}
```

#### sendLoop

```go
func (sws *serverWatchStream) sendLoop() { //推送watch响应
   // watch ids that are currently active
   ids := make(map[storage.WatchID]struct{})
   // watch responses pending on a watch id creation message
   pending := make(map[storage.WatchID][]*pb.WatchResponse)

   progressTicker := time.NewTicker(ProgressReportInterval)
   defer progressTicker.Stop()

   for {
      select {
      case wresp, ok := <-sws.watchStream.Chan(): //触发的事件
         if !ok {
            return
         }

         evs := wresp.Events
         events := make([]*storagepb.Event, len(evs))
         for i := range evs {
            events[i] = &evs[i]
         }

         wr := &pb.WatchResponse{
            Header:          sws.newResponseHeader(wresp.Revision),
            WatchId:         int64(wresp.WatchID),
            Events:          events,
            CompactRevision: wresp.CompactRevision,
         }

         if _, hasId := ids[wresp.WatchID]; !hasId {
            // watchId尚未发送给客户端，缓存触发的事件
            wrs := append(pending[wresp.WatchID], wr)
            pending[wresp.WatchID] = wrs
            continue
         }

         storage.ReportEventReceived()
         if err := sws.gRPCStream.Send(wr); err != nil {
            return
         }

         if _, ok := sws.progress[wresp.WatchID]; ok {
            sws.progress[wresp.WatchID] = false
         }

      case c, ok := <-sws.ctrlStream: //响应信息
         if !ok {
            return
         }

         if err := sws.gRPCStream.Send(c); err != nil {
            return
         }

         // track id creation
         wid := storage.WatchID(c.WatchId)
         if c.Canceled {
            delete(ids, wid)
            continue
         }
         if c.Created {
            // 发送缓存的待发事件
            ids[wid] = struct{}{} //将watch放入ids数组
            for _, v := range pending[wid] {
               storage.ReportEventReceived()
               if err := sws.gRPCStream.Send(v); err != nil {
                  return	
               }
            }
            delete(pending, wid)
         }
      case <-progressTicker.C:
         for id, ok := range sws.progress {
            if ok {
               sws.watchStream.RequestProgress(id)
            }
            sws.progress[id] = true
         }
      case <-sws.closec:
         // drain the chan to clean up pending events
         for range sws.watchStream.Chan() {
            storage.ReportEventReceived()
         }
         for _, wrs := range pending {
            for range wrs {
               storage.ReportEventReceived()
            }
         }
      }
   }
}
```

### watchStream

#### 注册watch

etcd/storage/watcher.go

```go
func (ws *watchStream) Watch(key, end []byte, startRev int64) WatchID {//一个key或者一个范围
   ws.mu.Lock()
   defer ws.mu.Unlock()
   if ws.closed {
      return -1
   }

   id := ws.nextID
   ws.nextID++

   w, c := ws.watchable.watch(key, end, startRev, id, ws.ch)

   ws.cancels[id] = c
   ws.watchers[id] = w
   return id
}
```

etcd/storage/watchable_store.go

```go
func (s *watchableStore) watch(key, end []byte, startRev int64, id WatchID, ch chan<- WatchResponse) (*watcher, cancelFunc) {
   s.mu.Lock()
   defer s.mu.Unlock()

   wa := &watcher{
      key: key,
      end: end,
      cur: startRev,
      id:  id,
      ch:  ch,
   }

   s.store.mu.Lock()
  //未指定版本或大于当前的版本
   synced := startRev > s.store.currentRev.main || startRev == 0
   if synced {
      wa.cur = s.store.currentRev.main + 1
      if startRev > wa.cur {
         wa.cur = startRev
      }
   }
   s.store.mu.Unlock()
   if synced {
      s.synced.add(wa) //存放关注的事件尚未发生，监听的数据都已经同步完毕，在等待新的变更。
   } else {
      slowWatcherGauge.Inc()
      s.unsynced.add(wa) //监听的数据还未同步完成
   }
   watcherGauge.Inc()

   cancel := cancelFunc(func() { //删除watch时调用此函数，移除watch
      s.mu.Lock()
      defer s.mu.Unlock()
      // remove references of the watcher
      if s.unsynced.delete(wa) {
         slowWatcherGauge.Dec()
         watcherGauge.Dec()
         return
      }

      if s.synced.delete(wa) {
         watcherGauge.Dec()
      }
   })

   return wa, cancel
}
```

etcd/storage/watcher_group.go

```go
func (wg *watcherGroup) add(wa *watcher) {
   wg.watchers.add(wa)
   if wa.end == nil { //维护单个key对应的watchers
      wg.keyWatchers.add(wa)
      return
   }
	//使用区间树维护监听的key，不仅可以监听单个key，还可以监听key范围、key前缀
   ivl := adt.NewStringAffineInterval(string(wa.key), string(wa.end))
   if iv := wg.ranges.Find(ivl); iv != nil 
      //此key的区间已经注册过watcher
      iv.Val.(watcherSet).add(wa)
      return
   }

   // 没有注册过，将watcher放入区间树
   ws := make(watcherSet)
   ws.add(wa)
   wg.ranges.Insert(ivl, ws)
}
```

#### 取消watch

etcd/storage/watcher.go

```go
func (ws *watchStream) Cancel(id WatchID) error {
   cancel, ok := ws.cancels[id]
   if !ok {
      return ErrWatcherNotExist
   }
   cancel()
   delete(ws.cancels, id)
   delete(ws.watchers, id)
   return nil
}
```

#### 通知客户端

etcd/storage/watchable_store.go

```go
func (s *watchableStore) Put(key, value []byte, lease lease.LeaseID) (rev int64) {
   s.mu.Lock()
   defer s.mu.Unlock()

   rev = s.store.Put(key, value, lease) //写入数据
   changes := s.store.getChanges()
   if len(changes) != 1 {
      log.Panicf("unexpected len(changes) != 1 after put")
   }

   ev := storagepb.Event{
      Type: storagepb.PUT,
      Kv:   &changes[0],
   }
   //通知客户端（notify是被同步调用的，必须轻量，否则会严重影响集群写性能）
   s.notify(rev, []storagepb.Event{ev}) 
   return rev
}
```

```go
func (s *watchableStore) notify(rev int64, evs []storagepb.Event) {
   for w, eb := range newWatcherBatch(&s.synced, evs) { //每个watcher对应的事件
      if eb.revs != 1 {
         panic("unexpected multiple revisions in notification")
      }
      select {
        //将触发的响应放入chan,此chan的大小默认为1024
      case w.ch <- WatchResponse{WatchID: w.id, Events: eb.evs, Revision: s.Rev()}:
         pendingEventsGauge.Add(float64(len(eb.evs)))
      default:
        //如果出现网络抖动、高负载导致推送缓慢，chan满了不会丢失数据
        //会将watcher从syncd移动到unsynced
         w.cur = rev
         s.unsynced.add(w)
         s.synced.delete(w)
         slowWatcherGauge.Inc()
      }
   }
}
```

### watchableStore

etcd/storage/watchable_store.go:

```go
func newWatchableStore(b backend.Backend, le lease.Lessor) *watchableStore {
   s := &watchableStore{
      store:    NewStore(b, le),
      unsynced: newWatcherGroup(),
      synced:   newWatcherGroup(),
      stopc:    make(chan struct{}),
   }
   if s.le != nil {
      // use this store as the deleter so revokes trigger watch events
      s.le.SetRangeDeleter(s)
   }
   s.wg.Add(1)
   go s.syncWatchersLoop()
   return s
}
```

#### syncWatchersLoop

```go
func (s *watchableStore) syncWatchersLoop() { //负责将unsynced中的watcher关注的事件推送给客户端，这些watcher关注的事件版本低于当前版本
   defer s.wg.Done()

   for {
      s.mu.Lock()
      s.syncWatchers()
      s.mu.Unlock()

      select {
      case <-time.After(100 * time.Millisecond):
      case <-s.stopc:
         return
      }
   }
}
```

```go
func (s *watchableStore) syncWatchers() {
   s.store.mu.Lock()
   defer s.store.mu.Unlock()

   if s.unsynced.size() == 0 {
      return
   }

   curRev := s.store.currentRev.main
   compactionRev := s.store.compactMainRev
  //查询unsynced中watcher关注的最小版本
   minRev := s.unsynced.scanMinRev(curRev, compactionRev)
   minBytes, maxBytes := newRevBytes(), newRevBytes()
   revToBytes(revision{main: minRev}, minBytes)
   revToBytes(revision{main: curRev + 1}, maxBytes)

   // UnsafeRange returns keys and values. And in boltdb, keys are revisions.
   // values are actual key-value pairs in backend.
   tx := s.store.b.BatchTx()
   tx.Lock()
  //查询此版本区间范围的key
   revs, vs := tx.UnsafeRange(keyBucketName, minBytes, maxBytes, 0)
   evs := kvsToEvents(&s.unsynced, revs, vs)
   tx.Unlock()

   wb := newWatcherBatch(&s.unsynced, evs)

   for w, eb := range wb {
      select {
      case w.ch <- WatchResponse{WatchID: w.id, Events: eb.evs, Revision: s.store.currentRev.main}:
         pendingEventsGauge.Add(float64(len(eb.evs)))
      default:
         continue
      }
      if eb.moreRev != 0 {
         w.cur = eb.moreRev
         continue
      }
      w.cur = curRev
      s.synced.add(w) //watcher移动到synced
      s.unsynced.delete(w) //watcher从unsynced移除
   }

   for w := range s.unsynced.watchers {
      if !wb.contains(w) { //过滤出watcher,版本低于合并时的版本
         w.cur = curRev //修改为当前最新版本
         s.synced.add(w) //watcher移动到synced
         s.unsynced.delete(w)//watcher从unsynced移除
      }
   }

   slowWatcherGauge.Set(float64(s.unsynced.size()))
}
```



```go
func (wg *watcherGroup) scanMinRev(curRev int64, compactRev int64) int64 {
   minRev := int64(math.MaxInt64)
   for w := range wg.watchers {
      if w.cur > curRev { //watch而的版本大于当前的版本
         panic("watcher current revision should not exceed current revision")
      }
      if w.cur < compactRev {  //低于合并时的版本
         select {
         case w.ch <- WatchResponse{WatchID: w.id, CompactRevision: compactRev}:
            wg.delete(w) //删除watcher
         default:
            // retry next time
         }
         continue
      }
      if minRev > w.cur {
         minRev = w.cur
      }
   }
   return minRev //返回最小的版本
}
```

# 日志和快照管理

## 日志管理

### wal结构体

```go
type WAL struct {
   dir      string           // the living directory of the underlay files
   metadata []byte           // metadata recorded at the head of each WAL
   state    raftpb.HardState // hardstate recorded at the head of WAL

   start   walpb.Snapshot // snapshot to start reading
   decoder *decoder       // decoder to decode records

   mu      sync.Mutex
   f       *os.File // underlay file opened for appending, sync
   seq     uint64   // sequence of the wal file currently used for writes
   enti    uint64   // index of the last entry saved to the wal
   encoder *encoder // encoder to encode records

   locks []fileutil.Lock // the file locks the WAL is holding (the name is increasing)
}
```

### wal文件初始化

etcd/wal/wal.go

```go
func Create(dirpath string, metadata []byte) (*WAL, error) {
   if Exist(dirpath) { //目录存在
      return nil, os.ErrExist
   }

   if err := os.MkdirAll(dirpath, privateDirMode); err != nil { //创建目录
      return nil, err
   }

   p := path.Join(dirpath, walName(0, 0)) //创建文件名
   f, err := os.OpenFile(p, os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0600) //创建文件
   if err != nil {
      return nil, err
   }
   l, err := fileutil.NewLock(f.Name()) //获取文件锁
   if err != nil {
      return nil, err
   }
   if err = l.Lock(); err != nil {
      return nil, err
   }
	//创建wal
   w := &WAL{
      dir:      dirpath,
      metadata: metadata,
      seq:      0,
      f:        f,
      encoder:  newEncoder(f, 0),
   }
  //维护wal文件锁
   w.locks = append(w.locks, l)
   if err := w.saveCrc(0); err != nil { //crc
      return nil, err
   }
   if err := w.encoder.encode(&walpb.Record{Type: metadataType, Data: metadata}); err != nil { //元信息
      return nil, err
   }
   if err := w.SaveSnapshot(walpb.Snapshot{}); err != nil {//快照信息
      return nil, err 
   }
   return w, nil

```

### 回放wal日志

etcd/etcdserver/storage.go

```go
func readWAL(waldir string, snap walpb.Snapshot) (w *wal.WAL, id, cid types.ID, st raftpb.HardState, ents []raftpb.Entry) {
   var (
      err       error
      wmetadata []byte
   )

   repaired := false
   for {
      if w, err = wal.Open(waldir, snap); err != nil {//根据快照之后创建的日志文件创建wal
         plog.Fatalf("open wal error: %v", err)
      }
      if wmetadata, st, ents, err = w.ReadAll(); err != nil {//读取wal文件,将数据放入Memorystore
         w.Close()
         // we can only repair ErrUnexpectedEOF and we never repair twice.
         if repaired || err != io.ErrUnexpectedEOF {
            plog.Fatalf("read wal error (%v) and cannot be repaired", err)
         }
         if !wal.Repair(waldir) {
            plog.Fatalf("WAL error (%v) cannot be repaired", err)
         } else {
            plog.Infof("repaired WAL error (%v)", err)
            repaired = true
         }
         continue
      }
      break
   }
   var metadata pb.Metadata
   pbutil.MustUnmarshal(&metadata, wmetadata)
   id = types.ID(metadata.NodeID)
   cid = types.ID(metadata.ClusterID)
   return
}
```

```go
func (w *WAL) ReadAll() (metadata []byte, state raftpb.HardState, ents []raftpb.Entry, err error) {
   w.mu.Lock()
   defer w.mu.Unlock()

   rec := &walpb.Record{}
   decoder := w.decoder

   var match bool
   for err = decoder.decode(rec); err == nil; err = decoder.decode(rec) {
      switch rec.Type {
      case entryType:
         e := mustUnmarshalEntry(rec.Data)
         if e.Index > w.start.Index {
            ents = append(ents[:e.Index-w.start.Index-1], e)
         }
         w.enti = e.Index
      case stateType:
         state = mustUnmarshalState(rec.Data)
      case metadataType:
         if metadata != nil && !reflect.DeepEqual(metadata, rec.Data) {
            state.Reset()
            return nil, state, nil, ErrMetadataConflict
         }
         metadata = rec.Data
      case crcType:
         crc := decoder.crc.Sum32()
         // current crc of decoder must match the crc of the record.
         // do no need to match 0 crc, since the decoder is a new one at this case.
         if crc != 0 && rec.Validate(crc) != nil {
            state.Reset()
            return nil, state, nil, ErrCRCMismatch
         }
         decoder.updateCRC(rec.Crc)
      case snapshotType:
         var snap walpb.Snapshot
         pbutil.MustUnmarshal(&snap, rec.Data)
         if snap.Index == w.start.Index {
            if snap.Term != w.start.Term {
               state.Reset()
               return nil, state, nil, ErrSnapshotMismatch
            }
            match = true
         }
      default:
         state.Reset()
         return nil, state, nil, fmt.Errorf("unexpected block type %d", rec.Type)
      }
   }

   switch w.f {
   case nil:
      // We do not have to read out all entries in read mode.
      // The last record maybe a partial written one, so
      // ErrunexpectedEOF might be returned.
      if err != io.EOF && err != io.ErrUnexpectedEOF {
         state.Reset()
         return nil, state, nil, err
      }
   default:
      // We must read all of the entries if WAL is opened in write mode.
      if err != io.EOF {
         state.Reset()
         return nil, state, nil, err
      }
   }

   err = nil
   if !match {
      err = ErrSnapshotNotFound
   }

   // close decoder, disable reading
   w.decoder.close()
   w.start = walpb.Snapshot{}

   w.metadata = metadata

   if w.f != nil {
      // create encoder (chain crc with the decoder), enable appending
      w.encoder = newEncoder(w.f, w.decoder.lastCRC())
      w.decoder = nil
      lastIndexSaved.Set(float64(w.enti))
   }

   return metadata, state, ents, err
}
```

### 根据快照创建wal

```go
func Open(dirpath string, snap walpb.Snapshot) (*WAL, error) {
  //找到快照之后的wal
   return openAtIndex(dirpath, snap, true)
}
```

```go
func openAtIndex(dirpath string, snap walpb.Snapshot, write bool) (*WAL, error) {
   names, err := fileutil.ReadDir(dirpath) //目录下的所有wal文件
   if err != nil {
      return nil, err
   }
   names = checkWalNames(names) //文件名集合
   if len(names) == 0 {
      return nil, ErrFileNotFound
   }
   //获取大于snap.index的wal文件
   nameIndex, ok := searchIndex(names, snap.Index)
   if !ok || !isValidSeq(names[nameIndex:]) {
      return nil, ErrFileNotFound
   }

   // open the wal files for reading
   rcs := make([]io.ReadCloser, 0)
   ls := make([]fileutil.Lock, 0)
   for _, name := range names[nameIndex:] {
      f, err := os.Open(path.Join(dirpath, name))
      if err != nil {
         return nil, err
      }
      l, err := fileutil.NewLock(f.Name())
      if err != nil {
         return nil, err
      }
      err = l.TryLock()
      if err != nil {
         if write {
            return nil, err
         }
      }
      rcs = append(rcs, f)
      ls = append(ls, l)
   }
   rc := MultiReadCloser(rcs...) //生成快照之后创建的wal文件集合

   // create a WAL ready for reading
   w := &WAL{
      dir:     dirpath,
      start:   snap,
      decoder: newDecoder(rc),
      locks:   ls,
   }

   if write {
      // 打开最后一个wal文件
      seq, _, err := parseWalName(names[len(names)-1])
      if err != nil {
         rc.Close()
         return nil, err
      }
      last := path.Join(dirpath, names[len(names)-1])

      f, err := os.OpenFile(last, os.O_WRONLY|os.O_APPEND, 0)
      if err != nil {
         rc.Close()
         return nil, err
      }
     //为此文件预先分配空间，默认64M
      err = fileutil.Preallocate(f, segmentSizeBytes)
      if err != nil {
         rc.Close()
         plog.Errorf("failed to allocate space when creating new wal file (%v)", err)
         return nil, err
      }

      w.f = f
      w.seq = seq
   }

   return w, nil
}
```

### wal追加日志项

```go
func (w *WAL) Save(st raftpb.HardState, ents []raftpb.Entry) error {
   w.mu.Lock()
   defer w.mu.Unlock()

   // short cut, do not call sync
   if raft.IsEmptyHardState(st) && len(ents) == 0 {
      return nil
   }

  	//是否刷新数据到磁盘
   mustSync := mustSync(st, w.state, len(ents))

   // TODO(xiangli): no more reference operator
   for i := range ents {
      if err := w.saveEntry(&ents[i]); err != nil { //保存日志项
         return err
      }
   }
   if err := w.saveState(&st); err != nil {  //保存HardState
      return err
   }

   fstat, err := w.f.Stat() //返回FileInfo
   if err != nil {
      return err
   }
   if fstat.Size() < segmentSizeBytes { //小于64M
      if mustSync {
         return w.sync() //刷新磁盘
      }
      return nil
   }
  //切换wal文件
   return w.cut()
}
```

### 切换wal文件

```go
func (w *WAL) cut() error {
   // 关闭旧的wal文件
   if err := w.sync(); err != nil {
      return err
   }
   if err := w.f.Close(); err != nil {
      return err
   }
		//新的wal文件名称
   fpath := path.Join(w.dir, walName(w.seq+1, w.enti+1))
   ftpath := fpath + ".tmp"

   // create a temp wal file with name sequence + 1, or truncate the existing one
   ft, err := os.OpenFile(ftpath, os.O_WRONLY|os.O_APPEND|os.O_CREATE|os.O_TRUNC, 0600)
   if err != nil {
      return err
   }

   // update writer and save the previous crc
   w.f = ft
  //计算旧wal文件的crc
   prevCrc := w.encoder.crc.Sum32()
   w.encoder = newEncoder(w.f, prevCrc)
   if err = w.saveCrc(prevCrc); err != nil {
      return err
   }
   if err = w.encoder.encode(&walpb.Record{Type: metadataType, Data: w.metadata}); err != nil {
      return err
   }
   if err = w.saveState(&w.state); err != nil {
      return err
   }
   // close temp wal file
   if err = w.sync(); err != nil {
      return err
   }
   if err = w.f.Close(); err != nil {
      return err
   }

   // atomically move temp wal file to wal file
   if err = os.Rename(ftpath, fpath); err != nil {
      return err
   }

   // open the wal file and update writer again
   f, err := os.OpenFile(fpath, os.O_WRONLY|os.O_APPEND, 0600)
   if err != nil {
      return err
   }
   if err = fileutil.Preallocate(f, segmentSizeBytes); err != nil {
      plog.Errorf("failed to allocate space when creating new wal file (%v)", err)
      return err
   }

   w.f = f
   prevCrc = w.encoder.crc.Sum32()
   w.encoder = newEncoder(w.f, prevCrc)

   // lock the new wal file
   l, err := fileutil.NewLock(f.Name())
   if err != nil {
      return err
   }

   if err := l.Lock(); err != nil {
      return err
   }
   w.locks = append(w.locks, l)

   // increase the wal seq
   w.seq++

   plog.Infof("segmented wal file %v is created", fpath)
   return nil
}
```



## 快照管理

### 生成快照

etcd/etcdserver/server.go:

```go
func (s *EtcdServer) triggerSnapshot(ep *etcdProgress) {//在应用状态机时调用此方法
   if ep.appliedi-ep.snapi <= s.snapCount { //默认10000
      return
   }
  //有快照正在发送，避免再次生成快照 
  if atomic.LoadInt64(&s.inflightSnapshots) != 0 {
      return
   }

   plog.Infof("start to snapshot (applied: %d, lastsnap: %d)", ep.appliedi, ep.snapi)
  //开始生成快照
   s.snapshot(ep.appliedi, ep.confState)
   ep.snapi = ep.appliedi
}
```



```go
func (s *EtcdServer) snapshot(snapi uint64, confState raftpb.ConfState) {
   clone := s.store.Clone()

   s.wg.Add(1)
   go func() {
      defer s.wg.Done()

      d, err := clone.SaveNoCopy() //副本
      if err != nil {
         plog.Panicf("store save should never fail: %v", err)
      }
      snap, err := s.r.raftStorage.CreateSnapshot(snapi, &confState, d)
      if err != nil {
         if err == raft.ErrSnapOutOfDate {
            return
         }
         plog.Panicf("unexpected create snapshot error %v", err)
      }
      if s.cfg.V3demo {
         s.getKV().Commit()
      }
     //保存快照
      if err = s.r.storage.SaveSnap(snap); err != nil {
         plog.Fatalf("save snapshot error: %v", err)
      }
      plog.Infof("saved snapshot at index %d", snap.Metadata.Index)

      compacti := uint64(1)
      if snapi > numberOfCatchUpEntries { //默认5000，将多余的从内存移除
         compacti = snapi - numberOfCatchUpEntries
      }
      err = s.r.raftStorage.Compact(compacti)
      if err != nil {
         // the compaction was done asynchronously with the progress of raft.
         // raft log might already been compact.
         if err == raft.ErrCompacted {
            return
         }
         plog.Panicf("unexpected compaction error %v", err)
      }
      plog.Infof("compacted raft log at %d", compacti)
   }()
}
```



```go
func (ms *MemoryStorage) Compact(compactIndex uint64) error {
  //快照之后的日志回收
   ms.Lock()
   defer ms.Unlock()
   offset := ms.ents[0].Index
   if compactIndex <= offset {
      return ErrCompacted
   }
   if compactIndex > ms.lastIndex() {
      raftLogger.Panicf("compact %d is out of bound lastindex(%d)", compactIndex, ms.lastIndex())
   }
		//将为之前的数据移除，默认只保留5000
   i := compactIndex - offset
   ents := make([]pb.Entry, 1, 1+uint64(len(ms.ents))-i)
   ents[0].Index = ms.ents[i].Index
   ents[0].Term = ms.ents[i].Term
   ents = append(ents, ms.ents[i+1:]...)
   ms.ents = ents
   return nil
}
```

### 同步快照

```go
func (r *raft) sendAppend(to uint64) {
   pr := r.prs[to]
   if pr.isPaused() {
      return
   }
   m := pb.Message{}
   m.To = to

   term, errt := r.raftLog.term(pr.Next - 1)
   ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)

   if errt != nil || erre != nil { // 发送快照
      if !pr.RecentActive { //节点网络状态不好
         r.logger.Debugf("ignore sending snapshot to %x since it is not recently active", to)
         return
      }

      m.Type = pb.MsgSnap
      snapshot, err := r.raftLog.snapshot() //获取快照
      if err != nil {
         if err == ErrSnapshotTemporarilyUnavailable {
            r.logger.Debugf("%x failed to send snapshot to %x because snapshot is temporarily unavailable", r.id, to)
            return
         }
         panic(err) // TODO(bdarnell)
      }
      if IsEmptySnap(snapshot) {
         panic("need non-empty snapshot")
      }
      m.Snapshot = snapshot
     //快照的起始索引、term
      sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
      r.logger.Debugf("%x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
         r.id, r.raftLog.firstIndex(), r.raftLog.committed, sindex, sterm, to, pr)
      pr.becomeSnapshot(sindex)
      r.logger.Debugf("%x paused sending replication messages to %x [%s]", r.id, to, pr)
   }
   }
   r.send(m)
}
```



```go
func (pr *Progress) becomeSnapshot(snapshoti uint64) {
   pr.resetState(ProgressStateSnapshot) //同步快照
   pr.PendingSnapshot = snapshoti //正在发送快照的起始索引
}
```

### 接收快照

etcd/raft/raft.go

```go
func (r *raft) handleSnapshot(m pb.Message) {//Follower接收快照
   sindex, sterm := m.Snapshot.Metadata.Index, m.Snapshot.Metadata.Term
   if r.restore(m.Snapshot) {
      r.logger.Infof("%x [commit: %d] restored snapshot [index: %d, term: %d]",
         r.id, r.raftLog.committed, sindex, sterm)
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.lastIndex()})
   } else {
      r.logger.Infof("%x [commit: %d] ignored snapshot [index: %d, term: %d]",
         r.id, r.raftLog.committed, sindex, sterm)
      r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
   }
}
```

```go
func (r *raft) restore(s pb.Snapshot) bool {
   if s.Metadata.Index <= r.raftLog.committed { //起始索引小于提交索引
      return false
   }
   if r.raftLog.matchTerm(s.Metadata.Index, s.Metadata.Term) {//index、term是否匹配
      r.logger.Infof("%x [commit: %d, lastindex: %d, lastterm: %d] fast-forwarded commit to snapshot [index: %d, term: %d]",
         r.id, r.raftLog.committed, r.raftLog.lastIndex(), r.raftLog.lastTerm(), s.Metadata.Index, s.Metadata.Term)
      r.raftLog.commitTo(s.Metadata.Index)
      return false
   }

   r.logger.Infof("%x [commit: %d, lastindex: %d, lastterm: %d] starts to restore snapshot [index: %d, term: %d]",
      r.id, r.raftLog.committed, r.raftLog.lastIndex(), r.raftLog.lastTerm(), s.Metadata.Index, s.Metadata.Term)
	//
   r.raftLog.restore(s)
  //各个节点同步进度
   r.prs = make(map[uint64]*Progress)
   for _, n := range s.Metadata.ConfState.Nodes {
      match, next := uint64(0), uint64(r.raftLog.lastIndex())+1
      if n == r.id {
         match = next - 1
      } else {
         match = 0
      }
      r.setProgress(n, match, next)
      r.logger.Infof("%x restored progress of %x [%s]", r.id, n, r.prs[n])
   }
   return true
}
```

# 存储

## 创建backed实例

etcd/storage/backend/backend.go

```go
func New(path string, d time.Duration, limit int) Backend {
   return newBackend(path, d, limit)
}
```



```go
func NewDefaultBackend(path string) Backend {
  //事务提交间隔、批次(默认100毫秒、积攒10000)
   return newBackend(path, defaultBatchInterval, defaultBatchLimit)
}
```

```go
func newBackend(bcfg BackendConfig) *backend {
	bopts := &bolt.Options{}
	if boltOpenOptions != nil {
		*bopts = *boltOpenOptions
	}
	bopts.InitialMmapSize = bcfg.mmapSize()

	db, err := bolt.Open(bcfg.Path, 0600, bopts)
	if err != nil {
		plog.Panicf("cannot open database at %s (%v)", bcfg.Path, err)
	}

	// In future, may want to make buffering optional for low-concurrency systems
	// or dynamically swap between buffered/non-buffered depending on workload.
	b := &backend{
		db: db,

		batchInterval: bcfg.BatchInterval,
		batchLimit:    bcfg.BatchLimit,

		readTx: &readTx{buf: txReadBuffer{
			txBuffer: txBuffer{make(map[string]*bucketBuffer)}}, //创建readTx实例
		},

		stopc: make(chan struct{}),
		donec: make(chan struct{}),
	}
	b.batchTx = newBatchTxBuffered(b) //创建batchTxBuffered实例
	go b.run() //创建协程，提交事务
	return b
}
```



```go
func newBatchTxBuffered(backend *backend) *batchTxBuffered {
	tx := &batchTxBuffered{
		batchTx: batchTx{backend: backend},
		buf: txWriteBuffer{
			txBuffer: txBuffer{make(map[string]*bucketBuffer)},
			seq:      true,
		},
	}
	tx.Commit()
	return tx
}
```

## 定期提交事务

```go
func (b *backend) run() { //根据时间间隔提交事务
	defer close(b.donec)
	t := time.NewTimer(b.batchInterval) //创建定时器
	defer t.Stop()
	for {
		select {
		case <-t.C:
		case <-b.stopc:
			b.batchTx.CommitAndStop()
			return
		}
		b.batchTx.Commit() //提交事务
		t.Reset(b.batchInterval) //重置定时器
	}
}
```

## 生成快照

```go
func (b *backend) Snapshot() Snapshot {
	b.batchTx.Commit //提交事务

	b.mu.RLock()//加锁
	defer b.mu.RUnlock() //释放锁
	tx, err := b.db.Begin(false)//开启读事务
	if err != nil { 
		plog.Fatalf("cannot begin tx (%s)", err)
	}

	stopc, donec := make(chan struct{}), make(chan struct{})
	dbBytes := tx.Size() //获取boltdb中数据的大小
	go func() {
		defer close(donec)
		// 快照发送速度为100M/s
		var sendRateBytes int64 = 100 * 1024 * 1014
		warningTimeout := time.Duration(int64((float64(dbBytes) / float64(sendRateBytes)) * float64(time.Second)))
		if warningTimeout < minSnapshotWarningTimeout {
			warningTimeout = minSnapshotWarningTimeout
		}
		start := time.Now()
    //创建定时器
		ticker := time.NewTicker(warningTimeout)
		defer ticker.Stop() //方法结束时关闭定时器
		for {
			select {
			case <-ticker.C: //未发送完快照时，输出警告日志
				plog.Warningf("snapshotting is taking more than %v seconds to finish transferring %v MB [started at %v]", time.Since(start).Seconds(), float64(dbBytes)/float64(1024*1014), start)
			case <-stopc: //发送快照结束
				snapshotDurations.Observe(time.Since(start).Seconds())
				return
			}
		}
	}()

	return &snapshot{tx, stopc, donec} //创建快照实例
}
```

## 整理碎片

```go
func (b *backend) Defrag() error {
   return b.defrag()
}
```

```go
func (b *backend) defrag() error {
   now := time.Now()
	//首先获取batchTx/readTx/backend三把锁
   b.batchTx.Lock()
   defer b.batchTx.Unlock()

   b.mu.Lock()
   defer b.mu.Unlock()

   b.readTx.mu.Lock()
   defer b.readTx.mu.Unlock()

   b.batchTx.unsafeCommit(true) //提交事务，并不会创建新的写事务
   b.batchTx.tx = nil

   // 创建数据临时文件
   dir := filepath.Dir(b.db.Path())
   temp, err := ioutil.TempFile(dir, "db.tmp.*")
   if err != nil {
      return err
   }
   // bbolt v1.3.1-coreos.5 does not provide options.OpenFile so we
   // instead close the temp file and then let bolt reopen it.
   tdbp := temp.Name()
   if err := temp.Close(); err != nil {
      return err
   }
  //创建boltdb实例
   tmpdb, err := bolt.Open(tdbp, 0600, boltOpenOptions)
   if err != nil {
      return err
   }

   // 碎片整理
   err = defragdb(b.db, tmpdb, defragLimit)

   if err != nil {
      tmpdb.Close()
      if rmErr := os.RemoveAll(tmpdb.Path()); rmErr != nil {
         plog.Fatalf("failed to remove db.tmp after defragmentation completed: %v", rmErr)
      }
      return err
   }

   dbp := b.db.Path()

   err = b.db.Close()
   if err != nil {
      plog.Fatalf("cannot close database (%s)", err)
   }
   err = tmpdb.Close()
   if err != nil {
      plog.Fatalf("cannot close database (%s)", err)
   }
   // gofail: var defragBeforeRename struct{}
   err = os.Rename(tdbp, dbp)
   if err != nil {
      plog.Fatalf("cannot rename database (%s)", err)
   }
	//创建新的boltdb实例
   b.db, err = bolt.Open(dbp, 0600, boltOpenOptions)
   if err != nil {
      plog.Panicf("cannot open database at %s (%v)", dbp, err)
   }
  //创建写事务
   b.batchTx.tx, err = b.db.Begin(true)
   if err != nil {
      plog.Fatalf("cannot begin tx (%s)", err)
   }
	//创建读事务
   b.readTx.buf.reset()
   b.readTx.tx = b.unsafeBegin(false)

   size := b.readTx.tx.Size()
   db := b.db
   atomic.StoreInt64(&b.size, size)
   atomic.StoreInt64(&b.sizeInUse, size-(int64(db.Stats().FreePageN)*int64(db.Info().PageSize)))

   took := time.Since(now)
   defragDurations.Observe(took.Seconds())

   return nil
}
```

```go
func defragdb(odb, tmpdb *bolt.DB, limit int) error {
  //从旧数据库文件向新数据库文件复制键值对
   tmptx, err := tmpdb.Begin(true) //临时db开始写事务
   if err != nil {
      return err
   }

   tx, err := odb.Begin(false) //旧的db开始读事务
   if err != nil {
      return err
   }
   defer tx.Rollback()

   c := tx.Cursor()

   count := 0
   for next, _ := c.First(); next != nil; next, _ = c.Next() {
      b := tx.Bucket(next)
      if b == nil {
         return fmt.Errorf("backend: cannot defrag bucket %s", string(next))
      }

      tmpb, berr := tmptx.CreateBucketIfNotExists(next)
      if berr != nil {
         return berr
      }
      tmpb.FillPercent = 0.9 //提高利用率

      b.ForEach(func(k, v []byte) error {
         count++
         if count > limit {
            err = tmptx.Commit()
            if err != nil {
               return err
            }
            tmptx, err = tmpdb.Begin(true)
            if err != nil {
               return err
            }
            tmpb = tmptx.Bucket(next)
            tmpb.FillPercent = 0.9 // for seq write in for each

            count = 0
         }
         return tmpb.Put(k, v)
      })
   }

   return tmptx.Commit()
}
```

创建bucket

```go
func (t *batchTx) UnsafeCreateBucket(name []byte) {
   _, err := t.tx.CreateBucket(name)
   if err != nil && err != bolt.ErrBucketExists {
      log.Fatalf("storage: cannot create bucket %s (%v)", string(name), err)
   }
   t.pending++ //待提交的事务数+1
}
```

保存KeyValue

```go
func (t *batchTx) UnsafePut(bucketName []byte, key []byte, value []byte) {
   t.unsafePut(bucketName, key, value, false)
}
```



```go
func (t *batchTx) unsafePut(bucketName []byte, key []byte, value []byte, seq bool) {
   bucket := t.tx.Bucket(bucketName)
   if bucket == nil {
      log.Fatalf("storage: bucket %s does not exist", string(bucketName))
   }
   if seq {
      // 设置page的填充百分比，延迟页的分裂并减少空间的使用
      bucket.FillPercent = 0.9
   }
   if err := bucket.Put(key, value); err != nil {
      log.Fatalf("storage: cannot put key into bucket (%v)", err)
   }
   t.pending++
}
```

范围查找

```go
func (t *batchTx) UnsafeRange(bucketName []byte, key, endKey []byte, limit int64) (keys [][]byte, vs [][]byte) {
   bucket := t.tx.Bucket(bucketName)
   if bucket == nil {
      log.Fatalf("storage: bucket %s does not exist", string(bucketName))
   }

   if len(endKey) == 0 {
      if v := bucket.Get(key); v == nil {
         return keys, vs
      } else {
         return append(keys, key), append(vs, v)
      }
   }

   c := bucket.Cursor()
   for ck, cv := c.Seek(key); ck != nil && bytes.Compare(ck, endKey) < 0; ck, cv = c.Next() {
      vs = append(vs, cv)
      keys = append(keys, ck)
      if limit > 0 && limit == int64(len(keys)) {
         break
      }
   }

   return keys, vs
}
```

删除数据

```go
func (t *batchTx) UnsafeDelete(bucketName []byte, key []byte) {
   bucket := t.tx.Bucket(bucketName)
   if bucket == nil {
      log.Fatalf("storage: bucket %s does not exist", string(bucketName))
   }
   err := bucket.Delete(key)
   if err != nil {
      log.Fatalf("storage: cannot delete key from bucket (%v)", err)
   }
   t.pending++
}
```

提交事务

```go
func (t *batchTx) Commit() {//提交事务并开启新的事务
   t.Lock()
   defer t.Unlock()
   t.commit(false)//false：关闭boltdb
}
```

```go
func (t *batchTx) CommitAndStop() {//提交先前的事务，但是并不创建新的事务
   t.Lock()
   defer t.Unlock()
   t.commit(true)
}
```

```go
func (t *batchTx) Unlock() {
   if t.pending >= t.backend.batchLimit {//批量提交是我
      t.commit(false)
      t.pending = 0
   }
   t.Mutex.Unlock()
}
```

```go
func (t *batchTx) commit(stop bool) {
   var err error
   if t.tx != nil { //提交先前的事务
      if t.pending == 0 && !stop {
         return //没有待提交的事务并且boltdb没有关闭
      }
      err = t.tx.Commit() //提交事务
      atomic.AddInt64(&t.backend.commits, 1)

      t.pending = 0
      if err != nil {
         log.Fatalf("storage: cannot commit tx (%s)", err)
      }
   }

   if stop {
      return
   }

   t.backend.mu.RLock()
   defer t.backend.mu.RUnlock()
   // 创建新的事务
   t.tx, err = t.backend.db.Begin(true)
   if err != nil {
      log.Fatalf("storage: cannot begin tx (%s)", err)
   }
   atomic.StoreInt64(&t.backend.size, t.tx.Size()) //已经提交的数据大小
}
```

## ConsistentWatchableKV

```go
func New(b backend.Backend, le lease.Lessor, ig ConsistentIndexGetter) ConsistentWatchableKV {
   return newWatchableStore(b, le, ig)
}
```

```go
func newWatchableStore(b backend.Backend, le lease.Lessor, ig ConsistentIndexGetter) *watchableStore {
   s := &watchableStore{
      store:    NewStore(b, le, ig),
      victimc:  make(chan struct{}, 1),
      unsynced: newWatcherGroup(),
      synced:   newWatcherGroup(),
      stopc:    make(chan struct{}),
   }
  //实现并发读写
   s.store.ReadView = &readView{s}
   s.store.WriteView = &writeView{s}
   if s.le != nil {
      // use this store as the deleter so revokes trigger watch events
      s.le.SetRangeDeleter(func() lease.TxnDelete { return s.Write() })
   }
   s.wg.Add(2)
   go s.syncWatchersLoop() //推送历史版本
   go s.syncVictimsLoop()
   return s
}
```

## 上层存储-3.0

```go
func newWatchableStore(b backend.Backend, le lease.Lessor, ig ConsistentIndexGetter) *watchableStore {
   s := &watchableStore{
      store:    NewStore(b, le, ig), //创建store
      victimc:  make(chan struct{}, 1),
      unsynced: newWatcherGroup(),
      synced:   newWatcherGroup(),
      stopc:    make(chan struct{}),
   }
   if s.le != nil {
      // use this store as the deleter so revokes trigger watch events
      s.le.SetRangeDeleter(s)
   }
   s.wg.Add(2)
   go s.syncWatchersLoop()
   go s.syncVictimsLoop()
   return s
}
```

```go
func NewStore(b backend.Backend, le lease.Lessor) *store {
   s := &store{
      b:       b, //底层存储
      kvindex: newTreeIndex(),//内存索引，存放key以及对应的版本信息

      le: le,//租约

      currentRev:     revision{main: 1},//最新版本
      compactMainRev: -1, //合并版本

      fifoSched: schedule.NewFIFOScheduler(), //先见先出调度器

      stopc: make(chan struct{}),
   }

   if s.le != nil {
      s.le.SetRangeDeleter(s)
   }
	//创建数据桶、元数据桶
   tx := s.b.BatchTx()
   tx.Lock()
   tx.UnsafeCreateBucket(keyBucketName)
   tx.UnsafeCreateBucket(metaBucketName)
   tx.Unlock()
   s.b.ForceCommit() //提交事务

   if err := s.restore(); err != nil {
      // TODO: return the error instead of panic here?
      panic("failed to recover store from backend")
   }

   return s
}
```

```go
func (s *store) restore() error {
   min, max := newRevBytes(), newRevBytes()
   revToBytes(revision{main: 1}, min)
   revToBytes(revision{main: math.MaxInt64, sub: math.MaxInt64}, max)

   // restore index
   tx := s.b.BatchTx()
   tx.Lock()
  //获取合并版本
   _, finishedCompactBytes := tx.UnsafeRange(metaBucketName, finishedCompactKeyName, nil, 0)
   if len(finishedCompactBytes) != 0 {
      s.compactMainRev = bytesToRev(finishedCompactBytes[0]).main
      log.Printf("storage: restore compact to %d", s.compactMainRev)
   }

   // TODO: limit N to reduce max memory usage
   keys, vals := tx.UnsafeRange(keyBucketName, min, max, 0)
   for i, key := range keys {
      var kv storagepb.KeyValue
      if err := kv.Unmarshal(vals[i]); err != nil {
         log.Fatalf("storage: cannot unmarshal event: %v", err)
      }

      rev := bytesToRev(key[:revBytesLen])

      // restore index
      switch {
      case isTombstone(key): //已被删除
         s.kvindex.Tombstone(kv.Key, rev)
         if lease.LeaseID(kv.Lease) != lease.NoLease {
            err := s.le.Detach(lease.LeaseID(kv.Lease), []lease.LeaseItem{{Key: string(kv.Key)}}) //解绑，解除reaseId关联的key
            if err != nil && err != lease.ErrLeaseNotFound {
               log.Fatalf("storage: unexpected Detach error %v", err)
            }
         }
      default:
         s.kvindex.Restore(kv.Key, revision{kv.CreateRevision, 0}, rev, kv.Version)
         if lease.LeaseID(kv.Lease) != lease.NoLease {
            if s.le == nil {
               panic("no lessor to attach lease")
            }
           //关联reaseId和key
            err := s.le.Attach(lease.LeaseID(kv.Lease), []lease.LeaseItem{{Key: string(kv.Key)}})
            if err != nil && err != lease.ErrLeaseNotFound {
               panic("unexpected Attach error")
            }
         }
      }

      // update revision
      s.currentRev = rev
   }
	//判断是否重新调度合并
   _, scheduledCompactBytes := tx.UnsafeRange(metaBucketName, scheduledCompactKeyName, nil, 0)
   if len(scheduledCompactBytes) != 0 {
      scheduledCompact := bytesToRev(scheduledCompactBytes[0]).main
      if scheduledCompact > s.compactMainRev {
         log.Printf("storage: resume scheduled compaction at %d", scheduledCompact)
         go s.Compact(scheduledCompact) //重新调度合并
      }
   }

   tx.Unlock()

   return nil
}
```

### 写请求

```go
func (s *store) Put(key, value []byte, lease lease.LeaseID) int64 {
   id := s.TxnBegin() //开启事务
   s.put(key, value, lease)
   s.txnEnd(id) //结束事务

   putCounter.Inc()

   return int64(s.currentRev.main)
}
```

#### 开启事务

```go
func (s *store) TxnBegin() int64 {
   s.mu.Lock() //store独占锁
   s.currentRev.sub = 0
   s.tx = s.b.BatchTx()
   s.tx.Lock() //batchTx独占锁
	 s.saveIndex() //
   s.txnID = rand.Int63()
   return s.txnID
}
```

##### 幂等性

```go
func (s *store) saveIndex() {
   if s.ig == nil {
      return
   }
   tx := s.tx
   bs := s.bytesBuf8
 	//存储当前已经执行过的日志条目索引，实现幂等性
   binary.BigEndian.PutUint64(bs, s.ig.ConsistentIndex())
   // put the index into the underlying backend
   // tx has been locked in TxnBegin, so there is no need to lock it again
   tx.UnsafePut(metaBucketName, consistentIndexKeyName, bs)
}
```

#### 结束事务

```go
func (s *store) TxnEnd(txnID int64) error {
   err := s.txnEnd(txnID)
   if err != nil {
      return err
   }
   txnCounter.Inc()
   return nil
}
```

```go
func (s *store) txnEnd(txnID int64) error {
   if txnID != s.txnID {
      return ErrTxnIDMismatch
   }

   s.tx.Unlock() //释放batchTx独占锁
   if s.currentRev.sub != 0 {//sub不为0，说明是写事务
      s.currentRev.main += 1
   }
   s.currentRev.sub = 0

   dbTotalSize.Set(float64(s.b.Size()))
   s.mu.Unlock() //释放stroe独占锁
   return nil
}
```

### 读请求

```go
func (s *store) Range(key, end []byte, ro RangeOptions) (r *RangeResult, err error) {
	id := s.TxnBegin() //开启事务
	kvs, count, rev, err := s.rangeKeys(key, end, ro.Limit, ro.Rev, ro.Count)
	s.txnEnd(id) //结束事务

	rangeCounter.Inc()

	r = &RangeResult{
		KVs:   kvs,
		Count: count,
		Rev:   rev,
	}

	return r, err
}
```



```go
func NewStore(b backend.Backend, le lease.Lessor, ig ConsistentIndexGetter) *store {
   s := &store{
      b:       b, //boltdb
      ig:      ig, //幂等
      kvindex: newTreeIndex(), //B树内存索引

      le: le, //租约

      currentRev:     1, //最新版本
      compactMainRev: -1, //合并版本

      bytesBuf8: make([]byte, 8),
      fifoSched: schedule.NewFIFOScheduler(), //先进先出调度器

      stopc: make(chan struct{}),
   }
  //实现并发读写
   s.ReadView = &readView{s}
   s.WriteView = &writeView{s}
   if s.le != nil {
      s.le.SetRangeDeleter(func() lease.TxnDelete { return s.Write() })
   }
	//创建存放元数据、数据的bucket
   tx := s.b.BatchTx()
   tx.Lock()
   tx.UnsafeCreateBucket(keyBucketName)
   tx.UnsafeCreateBucket(metaBucketName)
   tx.Unlock()
   s.b.ForceCommit()

  //读取数据构建索引
   if err := s.restore(); err != nil {
      // TODO: return the error instead of panic here?
      panic("failed to recover store from backend")
   }

   return s
}
```

```go
func (s *store) restore() error {
   s.setupMetricsReporter()

   min, max := newRevBytes(), newRevBytes()
   revToBytes(revision{main: 1}, min)
   revToBytes(revision{main: math.MaxInt64, sub: math.MaxInt64}, max)

   keyToLease := make(map[string]lease.LeaseID)

   // restore index
   tx := s.b.BatchTx()
   tx.Lock()
	//获取合并版本号scheduledCompactKeyName、finishedCompactKeyName成对存在
   _, finishedCompactBytes := tx.UnsafeRange(metaBucketName, finishedCompactKeyName, nil, 0)
   if len(finishedCompactBytes) != 0 {
      s.compactMainRev = bytesToRev(finishedCompactBytes[0]).main
      plog.Printf("restore compact to %d", s.compactMainRev)
   }
   _, scheduledCompactBytes := tx.UnsafeRange(metaBucketName, scheduledCompactKeyName, nil, 0)
   scheduledCompact := int64(0)
   if len(scheduledCompactBytes) != 0 {
      scheduledCompact = bytesToRev(scheduledCompactBytes[0]).main
   }

   // index keys concurrently as they're loaded in from tx
   keysGauge.Set(0)
  //重建索引
   rkvc, revc := restoreIntoIndex(s.kvindex)
   for {
     //botldb读取数据
      keys, vals := tx.UnsafeRange(keyBucketName, min, max, int64(restoreChunkKeys))
      if len(keys) == 0 {
         break
      }
      // rkvc blocks if the total pending keys exceeds the restore
      // chunk size to keep keys from consuming too much memory.
      restoreChunk(rkvc, keys, vals, keyToLease)
      if len(keys) < restoreChunkKeys {
         // partial set implies final set
         break
      }
      // next set begins after where this one ended
      newMin := bytesToRev(keys[len(keys)-1][:revBytesLen])
      newMin.sub++
      revToBytes(newMin, min)
   }
   close(rkvc)
   s.currentRev = <-revc

   // keys in the range [compacted revision -N, compaction] might all be deleted due to compaction.
   // the correct revision should be set to compaction revision in the case, not the largest revision
   // we have seen.
   if s.currentRev < s.compactMainRev {//当前的版本小于合并版本
      s.currentRev = s.compactMainRev
   }
   if scheduledCompact <= s.compactMainRev { //最新启动的合并任务的版本号小于等于最后一次合并任务结束时的版本后
      scheduledCompact = 0
   }
		//租约
   for key, lid := range keyToLease {
      if s.le == nil {
         panic("no lessor to attach lease")
      }
      err := s.le.Attach(lid, []lease.LeaseItem{{Key: key}})
      if err != nil {
         plog.Errorf("unexpected Attach error: %v", err)
      }
   }

   tx.Unlock()

   if scheduledCompact != 0 { //之前的合并任务没有完成，调度合并任务，防止节点的数据不一致
      s.Compact(scheduledCompact)
      plog.Printf("resume scheduled compaction at %d", scheduledCompact)
   }

   return nil
}
```

# 网络层

## Transport接口

### start

```go
func (t *Transport) Start() error {
   var err error
  //创建stream通道,负责传输数据量小、发送比较频繁的消息，比如MsgApp、MsgHeartbeat
   t.streamRt, err = newStreamRoundTripper(t.TLSInfo, t.DialTimeout)
   if err != nil {
      return err
   }
  //创建pipeline通道，负责传输数据量大、发送频率低的消息，比如MsgSnap
   t.pipelineRt, err = NewRoundTripper(t.TLSInfo, t.DialTimeout)
   if err != nil {
      return err
   }
  //初始化remotes、peers
   t.remotes = make(map[types.ID]*remote)
   t.peers = make(map[types.ID]Peer)
  //探测pipeline、stream通道是否可用
   t.pipelineProber = probing.NewProber(t.pipelineRt)
   t.streamProber = probing.NewProber(t.streamRt)
   return nil
}
```

### AddPeer

```go
func (t *Transport) AddPeer(id types.ID, us []string) {
   t.mu.Lock()
   defer t.mu.Unlock()

   if t.peers == nil {
      panic("transport stopped")
   }
   if _, ok := t.peers[id]; ok {
      return
   }
   urls, err := types.NewURLs(us)
   if err != nil {
      plog.Panicf("newURLs %+v should never fail: %+v", us, err)
   }
   fs := t.LeaderStats.Follower(id.String())
  //创建peer实例
   t.peers[id] = startPeer(t, urls, id, fs)
  //探测节点的健康状况
   addPeerToProber(t.pipelineProber, id.String(), us, RoundTripperNameSnapshot, rtts)
   addPeerToProber(t.streamProber, id.String(), us, RoundTripperNameRaftMessage, rtts)
   plog.Infof("added peer %s", id)
}
```

### Send

```go
func (t *Transport) Send(msgs []raftpb.Message) {
   for _, m := range msgs {
      if m.To == 0 {
         // ignore intentionally dropped message
         continue
      }
      to := types.ID(m.To)

      t.mu.RLock() //加锁，获取peer、remote
      p, pok := t.peers[to]
      g, rok := t.remotes[to]
      t.mu.RUnlock() //释放锁

      if pok { //使用peer实例发送数据
         if m.Type == raftpb.MsgApp {
            t.ServerStats.SendAppendReq(m.Size())
         }
         p.send(m)
         continue
      }

      if rok { //使用remote实例发送数据
         g.send(m)
         continue
      }

      plog.Debugf("ignored message %s (sent to unknown peer %s)", m.Type, to)
   }
}
```

## Peer接口

```go
func startPeer(transport *Transport, urls types.URLs, peerID types.ID, fs *stats.FollowerStats) *peer {
   plog.Infof("starting peer %s...", peerID)
   defer plog.Infof("started peer %s", peerID)

   status := newPeerStatus(peerID)
  //根据节点的URLS创建urlPicker实例
   picker := newURLPicker(urls)
   errorc := transport.ErrorC
   r := transport.Raft
   pipeline := &pipeline{ //创建pipiline实例
      peerID:        peerID,
      tr:            transport,
      picker:        picker,
      status:        status,
      followerStats: fs,
      raft:          r,
      errorc:        errorc,
   }
   pipeline.start() //启动pipiline

   p := &peer{ //创建peer实例
      id:             peerID,
      r:              r,
      status:         status,
      picker:         picker,
      msgAppV2Writer: startStreamWriter(peerID, status, fs, r),//创建并启动streamWriter
      writer:         startStreamWriter(peerID, status, fs, r), 
      pipeline:       pipeline,
      snapSender:     newSnapshotSender(transport, picker, peerID, status),
      recvc:          make(chan raftpb.Message, recvBufSize),
      propc:          make(chan raftpb.Message, maxPendingProposals),
      stopc:          make(chan struct{}),
   }

   ctx, cancel := context.WithCancel(context.Background())
   p.cancel = cancel
   go func() {
      for {
         select {
         case mm := <-p.recvc: //获取连接上读取到的Message
            if err := r.Process(ctx, mm); err != nil { //交由raft处理
               plog.Warningf("failed to process raft message (%v)", err)
            }
         case <-p.stopc:
            return
         }
      }
   }()

   // r.Process might block for processing proposal when there is no leader.
   // Thus propc must be put into a separate routine with recvc to avoid blocking
   // processing other raft messages.
   go func() {
      for {
         select {
         case mm := <-p.propc: //处理msgProp类型的消息
            if err := r.Process(ctx, mm); err != nil {
               plog.Warningf("failed to process raft message (%v)", err)
            }
         case <-p.stopc:
            return
         }
      }
   }()
//创建streamReader，读取stream通道上的数据
   p.msgAppV2Reader = &streamReader{
      peerID: peerID,
      typ:    streamTypeMsgAppV2,
      tr:     transport,
      picker: picker,
      status: status,
      recvc:  p.recvc,
      propc:  p.propc,
   }
   p.msgAppReader = &streamReader{
      peerID: peerID,
      typ:    streamTypeMessage,
      tr:     transport,
      picker: picker,
      status: status,
      recvc:  p.recvc,
      propc:  p.propc,
   }
   p.msgAppV2Reader.start()
   p.msgAppReader.start()

   return p
}
```

### send

```go
func (p *peer) send(m raftpb.Message) {
   p.mu.Lock()
   paused := p.paused
   p.mu.Unlock()

   if paused {
      return
   }

   writec, name := p.pick(m) //挑选合适的通道
   select {
   case writec <- m: //将消息放入chan
   default://chan已满，报告底层的raft
      p.r.ReportUnreachable(m.To)
      if isMsgSnap(m) {
         p.r.ReportSnapshot(m.To, raft.SnapshotFailure)
      }
      if p.status.isActive() {
         plog.MergeWarningf("dropped internal raft message to %s since %s's sending buffer is full (bad/overloaded network)", p.id, name)
      }
      plog.Debugf("dropped %s to %s since %s's sending buffer is full", m.Type, p.id, name)
      sentFailures.WithLabelValues(types.ID(m.To).String()).Inc()
   }
}
```

```go
func (p *peer) pick(m raftpb.Message) (writec chan<- raftpb.Message, picked string) {
   var ok bool
  //考虑到MsgSnap可能有一个大的尺寸，例如1G，并且会长时间阻塞流
   if isMsgSnap(m) { //快照，优先使用pipeline发送
      return p.pipeline.msgc, pipelineMsg 
   } else if writec, ok = p.msgAppV2Writer.writec(); ok && isMsgApp(m) {
      return writec, streamAppV2
   } else if writec, ok = p.writer.writec(); ok {
      return writec, streamMsg
   }
   return p.pipeline.msgc, pipelineMsg
}
```

## Pipeline接口

### start

```go
func (p *pipeline) start() {
   p.stopc = make(chan struct{})
   p.msgc = make(chan raftpb.Message, pipelineBufSize) //默认缓存64
   p.wg.Add(connPerPipeline)
   //创建4个协程，用于发送消息
   for i := 0; i < connPerPipeline; i++ {
      go p.handle()
   }
   plog.Infof("started HTTP pipelining with peer %s", p.peerID)
}
```

### hanle

```go
func (p *pipeline) handle() { //发送消息
   defer p.wg.Done()

   for {
      select {
      case m := <-p.msgc:
         start := time.Now()
         err := p.post(pbutil.MustMarshal(&m)) //序列化，发送
         end := time.Now()

         if err != nil {
            p.status.deactivate(failureType{source: pipelineMsg, action: "write"}, err.Error())

            if m.Type == raftpb.MsgApp && p.followerStats != nil {
               p.followerStats.Fail()
            }
           //向raft报告节点不可达
            p.raft.ReportUnreachable(m.To)
            if isMsgSnap(m) {
               p.raft.ReportSnapshot(m.To, raft.SnapshotFailure)
            }
            sentFailures.WithLabelValues(types.ID(m.To).String()).Inc()
            continue
         }

         p.status.activate()
         if m.Type == raftpb.MsgApp && p.followerStats != nil {
            p.followerStats.Succ(end.Sub(start))
         }
         if isMsgSnap(m) {
           //向raft报告快照发送成功
            p.raft.ReportSnapshot(m.To, raft.SnapshotFinish)
         }
         sentBytes.WithLabelValues(types.ID(m.To).String()).Add(float64(m.Size()))
      case <-p.stopc:
         return
      }
   }
}
```

### post

```go
func (p *pipeline) post(data []byte) (err error) {//完成消息发送
   u := p.picker.pick() //获取对端包略的URL
   req := createPostRequest(u, RaftPrefix, bytes.NewBuffer(data), "application/protobuf", p.tr.URLs, p.tr.ID, p.tr.ClusterID) //创建请求

   done := make(chan struct{}, 1)
   ctx, cancel := context.WithCancel(context.Background())
   req = req.WithContext(ctx)
   go func() {
      select {
      case <-done:
      case <-p.stopc:
         waitSchedule()
         cancel()
      }
   }()
	//发送请求
   resp, err := p.tr.pipelineRt.RoundTrip(req) 
   done <- struct{}{} //请求发送完毕
   if err != nil {
      p.picker.unreachable(u)//将URL不可达
      return err
   }
   b, err := ioutil.ReadAll(resp.Body) //读取响应
   if err != nil {
      p.picker.unreachable(u) //将URL不可达
      return err
   }
   resp.Body.Close()

   err = checkPostResponse(resp, b, req, p.peerID) //检测响应内容
   if err != nil {
      p.picker.unreachable(u)
      if err == errMemberRemoved {
         reportCriticalError(err, p.errorc)
      }
      return err
   }

   return nil
}
```

## StreamWriter

## StreamReader

负责从stream通道读取数据

### start

```go
func (r *streamReader) start() {
   r.stopc = make(chan struct{})
   r.done = make(chan struct{})
   if r.errorc == nil {
      r.errorc = r.tr.ErrorC
   }

   go r.run()
}
```

```go
func (cr *streamReader) run() {
   t := cr.typ
   plog.Infof("started streaming with peer %s (%s reader)", cr.peerID, t)
   for {
      rc, err := cr.dial(t)
      if err != nil {
         if err != errUnsupportedStreamType {
            cr.status.deactivate(failureType{source: t.String(), action: "dial"}, err.Error())
         }
      } else {
         cr.status.activate()
         plog.Infof("established a TCP streaming connection with peer %s (%s reader)", cr.peerID, cr.typ)
         err := cr.decodeLoop(rc, t)
         plog.Warningf("lost the TCP streaming connection with peer %s (%s reader)", cr.peerID, cr.typ)
         switch {
         // all data is read out
         case err == io.EOF:
         // connection is closed by the remote
         case transport.IsClosedConnError(err):
         default:
            cr.status.deactivate(failureType{source: t.String(), action: "read"}, err.Error())
         }
      }
      select {
      // Wait 100ms to create a new stream, so it doesn't bring too much
      // overhead when retry.
      case <-time.After(100 * time.Millisecond):
      case <-cr.stopc:
         plog.Infof("stopped streaming with peer %s (%s reader)", cr.peerID, t)
         close(cr.done)
         return
      }
   }
}
```

## snapshotSender

```go
func (s *snapshotSender) send(merged snap.Message) {
   start := time.Now()

   m := merged.Message
   to := types.ID(m.To).String()

   body := createSnapBody(merged)//创建请求body
   defer body.Close()

   u := s.picker.pick() //挑选可用的URL
   req := createPostRequest(u, RaftSnapshotPrefix, body, "application/octet-stream", s.tr.URLs, s.from, s.cid) //创建请求

   plog.Infof("start to send database snapshot [index: %d, to %s]...", m.Snapshot.Metadata.Index, types.ID(m.To))

   err := s.post(req) //发送请求
   defer merged.CloseWithError(err)
   if err != nil { //出现异常
      plog.Warningf("database snapshot [index: %d, to: %s] failed to be sent out (%v)", m.Snapshot.Metadata.Index, types.ID(m.To), err)

      // errMemberRemoved is a critical error since a removed member should
      // always be stopped. So we use reportCriticalError to report it to errorc.
      if err == errMemberRemoved {
         reportCriticalError(err, s.errorc)
      }

      s.picker.unreachable(u) //标记RUL不可达
      s.status.deactivate(failureType{source: sendSnap, action: "post"}, err.Error())
      s.r.ReportUnreachable(m.To) //报告raft模块URL不可达
      // report SnapshotFailure to raft state machine. After raft state
      // machine knows about it, it would pause a while and retry sending
      // new snapshot message.
      s.r.ReportSnapshot(m.To, raft.SnapshotFailure) //上报快照发送失败
      sentFailures.WithLabelValues(to).Inc()
      snapshotSendFailures.WithLabelValues(to).Inc()
      return
   }
   s.status.activate()
   s.r.ReportSnapshot(m.To, raft.SnapshotFinish) //上报raft快照发送成功
   plog.Infof("database snapshot [index: %d, to: %s] sent out successfully", m.Snapshot.Metadata.Index, types.ID(m.To))

   sentBytes.WithLabelValues(to).Add(float64(merged.TotalSize))

   snapshotSend.WithLabelValues(to).Inc()
   snapshotSendSeconds.WithLabelValues(to).Observe(time.Since(start).Seconds())
}
```

# BoltDB

## Tag-data/v1

### 打开数据库

```go
func Open(path string, mode os.FileMode) (*DB, error) {
   var db = &DB{opened: true}

   db.path = path

   var err error
  //打开文件
   if db.file, err = os.OpenFile(db.path, os.O_RDWR|os.O_CREATE, mode); err != nil {
      _ = db.close()
      return nil, err
   }

   //获取文件锁，避免其他进程使用此文件
   if err := syscall.Flock(int(db.file.Fd()), syscall.LOCK_EX); err != nil {
      _ = db.close()
      return nil, err
   }

   //写数据的方法
   db.ops.writeAt = db.file.WriteAt

   if info, err := db.file.Stat(); err != nil { //获取文件信息
      return nil, fmt.Errorf("stat error: %s", err)
   } else if info.Size() == 0 { //初次打开数据数据
      if err := db.init(); err != nil { //初始化meta page
         return nil, err
      }
   } else { //非初次打开数据库
      // 读取第一个meta page，获取pageSize
      var buf [0x1000]byte
      if _, err := db.file.ReadAt(buf[:], 0); err == nil {
         m := db.pageInBuffer(buf[:], 0).meta()
         if err := m.validate(); err != nil {
            return nil, fmt.Errorf("meta0 error: %s", err)
         }
         db.pageSize = int(m.pageSize)
      }
   }

   // Memory map the data file.
   if err := db.mmap(0); err != nil {
      _ = db.close()
      return nil, err
   }

   db.freelist = &freelist{pending: make(map[txid][]pgid)}
  //从meta page获取freelist page
   db.freelist.read(db.page(db.meta().freelist))

   return db, nil
}
```

```go
func (db *DB) init() error {
  //前4个page（2个metapage、free list page、buckets page）
   // 设置pageSize
   db.pageSize = os.Getpagesize()

   // Create two meta pages on a buffer.
   buf := make([]byte, db.pageSize*4)
  //创建2个meta page,保证高可用性，一个metapage失效，可以用另一个来恢复
   for i := 0; i < 2; i++ {
      p := db.pageInBuffer(buf[:], pgid(i))
      p.id = pgid(i)
      p.flags = metaPageFlag

      // 初始化meta 
      m := p.meta()
      m.magic = magic
      m.version = version
      m.pageSize = uint32(db.pageSize)
      m.version = version
      m.freelist = 2
      m.buckets = 3
      m.pgid = 4 //下一个存储数据的page的id
      m.txid = txid(i)
   }

   // 分配free list page
   p := db.pageInBuffer(buf[:], pgid(2))
   p.id = pgid(2)
   p.flags = freelistPageFlag
   p.count = 0

   // 分配buckets page 
   p = db.pageInBuffer(buf[:], pgid(3))
   p.id = pgid(3)
   p.flags = bucketsPageFlag
   p.count = 0

   // 写入文件
   if _, err := db.ops.writeAt(buf, 0); err != nil {
      return err
   }
   if err := fdatasync(db.file); err != nil {
      return err
   }

   return nil
}
```

### 创建桶

```go
func (tx *Tx) CreateBucket(name string) error {
   if tx.db == nil {
      return ErrTxClosed
   } else if !tx.writable {
      return ErrTxNotWritable
   } else if b := tx.Bucket(name); b != nil {
      return ErrBucketExists
   } else if len(name) == 0 {
      return ErrBucketNameRequired
   } else if len(name) > MaxBucketNameSize {
      return ErrBucketNameTooLarge
   }

   p, err := tx.allocate(1) //从freelist分配page
   if err != nil {
      return err
   }
   p.flags = leafPageFlag //叶子节点

   tx.buckets.put(name, &bucket{root: p.id}) //添加到bucket map，name -> bucket

   return nil
}
```

#### 查找Bucket是否存在

```go
func (tx *Tx) Bucket(name string) *Bucket {//查找Bucket是否已经存在
   b := tx.buckets.get(name)
   if b == nil {
      return nil
   }

   return &Bucket{
      bucket: b,
      name:   name,
      tx:     tx,
   }
}
```

#### 分配page

```go
func (tx *Tx) allocate(count int) (*page, error) {
   p, err := tx.db.allocate(count) //分配page
   if err != nil {
      return nil, err
   }

   tx.pages[p.id] = p //缓存分配的page

   // 更新统计信息
   tx.stats.PageCount++ //page数
   tx.stats.PageAlloc += count * tx.db.pageSize //page大小
 
   return p, nil
}
```

```go
func (db *DB) allocate(count int) (*page, error) {
   //为page分配临时的buf
   buf := make([]byte, count*db.pageSize)
   p := (*page)(unsafe.Pointer(&buf[0]))
   p.overflow = uint32(count - 1)

   // 从freelist获取空闲页
   if p.id = db.freelist.allocate(count); p.id != 0 {
      return p, nil
   }
  
   //free list无空闲的page
   //设置pageid
   p.id = db.rwtx.meta.pgid 
   // 调整mmap
   var minsz = int((p.id+pgid(count))+1) * db.pageSize
   if minsz >= len(db.data) {
      if err := db.mmap(minsz); err != nil {
         return nil, fmt.Errorf("mmap allocate error: %s", err)
      }
   }

   // Move the page id high water mark.
   db.rwtx.meta.pgid += pgid(count)

   return p, nil
}
```

### 添加数据

```go
func (b *Bucket) Put(key []byte, value []byte) error {
  //事务验证
   if b.tx.db == nil {
      return ErrTxClosed
   } else if !b.Writable() {
      return ErrBucketNotWritable
   }

   //key、value验证
   if len(key) == 0 {
      return ErrKeyRequired
   } else if len(key) > MaxKeySize {
      return ErrKeyTooLarge
   } else if int64(len(value)) > MaxValueSize {
      return ErrValueTooLarge
   }

   // 游标移动到追加的位置
   c := b.Cursor()
   c.Seek(key)

   //插入key、value
   c.node(b.tx).put(key, key, value, 0)

   return nil
}
```

创建游标

```go
func (b *Bucket) Cursor() *Cursor {
   // Update transaction statistics.
   b.tx.stats.CursorCount++

   // Allocate and return a cursor.
   return &Cursor{
      tx:    b.tx,
      root:  b.root, //桶的根节点
      stack: make([]elemRef, 0),
   }
}
```

移动游标

```go
func (c *Cursor) Seek(seek []byte) (key []byte, value []byte) {
   _assert(c.tx.db != nil, "tx closed")
	//用栈存放查找过程中涉及的page、node
   c.stack = c.stack[:0]
   c.search(seek, c.root)//从桶的根节点查找key所在的位置
   ref := &c.stack[len(c.stack)-1]

   if ref.index >= ref.count() { //index越界，表明key不存在
      return nil, nil
   }

   return c.keyValue()
}
```

递归查找

```go
func (c *Cursor) search(key []byte, pgid pgid) {
   p, n := c.tx.pageNode(pgid) //根据pageId查找page或者node
   if p != nil { //page不为空，验证page的类型，必须为branch或者leaf
      _assert((p.flags&(branchPageFlag|leafPageFlag)) != 0, "invalid page type: "+p.typ())
   }
   //把查找过程中涉及的node、page封装成eleRef
   e := elemRef{page: p, node: n}
   c.stack = append(c.stack, e) //放入栈中

   if e.isLeaf() {//叶子节点
      c.nsearch(key) //从node或者page中查找key所在的位置
      return
   }

   if n != nil { //node不为空
      c.searchNode(key, n) //从node中查找
      return
   }
   c.searchPage(key, p) //从page中查找
}
```

查找page/node

```go
func (tx *Tx) pageNode(id pgid) (*page, *node) {
   if tx.nodes != nil { //写事务中的nodes-map不为空
      if n := tx.nodes[id]; n != nil {
         return nil, n //从事务中找到pageid对应的node
      }
   }
   return tx.page(id), nil //从事务中找pageid对应的page
}
```



```go
func (tx *Tx) page(id pgid) *page {
   // Check the dirty pages first.
   if tx.pages != nil { //首先检查事务中的脏页是否包含
      if p, ok := tx.pages[id]; ok {
         return p
      }
   }

   // 否则直接从db查找page
   return tx.db.page(id)
}
```



```go
func (db *DB) page(id pgid) *page {
   pos := id * pgid(db.pageSize)
   return (*page)(unsafe.Pointer(&db.data[pos]))
}
```

查找所在的位置

```go
func (c *Cursor) nsearch(key []byte) {//从叶子节点查找key的位置
   e := &c.stack[len(c.stack)-1]
   p, n := e.page, e.node

   // 首先从node查找
   if n != nil {
      index := sort.Search(len(n.inodes), func(i int) bool {
         return bytes.Compare(n.inodes[i].key, key) != -1 //返回大于等于key的位置
      })
      e.index = index
      return
   }

   // 从page中查找
   inodes := p.leafPageElements()
   index := sort.Search(int(p.count), func(i int) bool {
      return bytes.Compare(inodes[i].key(), key) != -1
   })
   e.index = index
}
```

从非leaf类型的node中查找

```go
func (c *Cursor) searchNode(key []byte, n *node) {
   var exact bool
   index := sort.Search(len(n.inodes), func(i int) bool {
      ret := bytes.Compare(n.inodes[i].key, key)
      if ret == 0 { //查找到此key所在的位置
         exact = true
      }
      return ret != -1
   })
   if !exact && index > 0 { //此node中不存在此key，index减1
      index--
   }
   c.stack[len(c.stack)-1].index = index

   c.search(key, n.inodes[index].pgid) //递归查找
}
```

从非leaf类型的page中查找

```go
func (c *Cursor) searchPage(key []byte, p *page) {
   inodes := p.branchPageElements()

   var exact bool
   index := sort.Search(int(p.count), func(i int) bool {
      ret := bytes.Compare(inodes[i].key(), key)
      if ret == 0 {
         exact = true
      }
      return ret != -1
   })
   if !exact && index > 0 {
      index--
   }
   c.stack[len(c.stack)-1].index = index

   c.search(key, inodes[index].pgid)
}
```

获取node

```go
func (c *Cursor) node(tx *Tx) *node {
   _assert(len(c.stack) > 0, "accessing a node with a zero-length cursor stack")
   if ref := &c.stack[len(c.stack)-1]; ref.node != nil && ref.isLeaf() {
      return ref.node //栈顶元素的node不为空并且类型为leaf
   }

   //从根开始并向下遍历
   var n = c.stack[0].node
   if n == nil { //创建node
      n = tx.node(c.stack[0].page.id, nil)
   }
   for _, ref := range c.stack[:len(c.stack)-1] {
      _assert(!n.isLeaf, "expected branch node")
      n = n.childAt(int(ref.index))
   }
   _assert(n.isLeaf, "expected leaf node")
   return n
}
```

创建node

```go
func (tx *Tx) node(pgid pgid, parent *node) *node {
   if tx.nodes == nil {
      return nil
   } else if n := tx.nodes[pgid]; n != nil { 
      return n //node已经存在
   }

   // 创建node，默认非leaf类型
   n := &node{tx: tx, parent: parent}
   if n.parent != nil {
      n.depth = n.parent.depth + 1
   }
   n.read(tx.page(pgid)) //获取page，读取page
   tx.nodes[pgid] = n //pageid -> node 
 
   tx.stats.NodeCount++ // node统计信息

   return n
}
```

读取page

```go
func (n *node) read(p *page) {
   n.pgid = p.id
   n.isLeaf = ((p.flags & leafPageFlag) != 0) //根据page类型决定node的类型
   n.inodes = make(inodes, int(p.count)) //创建inode数组

   for i := 0; i < int(p.count); i++ {
      inode := &n.inodes[i]
      if n.isLeaf { //叶子节点，存放key、value
         elem := p.leafPageElement(uint16(i))
         inode.key = elem.key()
         inode.value = elem.value()
      } else { //中间节点，存放pageid、key
         elem := p.branchPageElement(uint16(i))
         inode.pgid = elem.pgid
         inode.key = elem.key()
      }
   }

   // Save first key so we can find the node in the parent when we spill.
   if len(n.inodes) > 0 {
      n.key = n.inodes[0].key //在node中设置inode数组中的第一个key
   } else {
      n.key = nil
   }
}
```



```go
func (n *node) put(oldKey, newKey, value []byte, pgid pgid) {
   // 寻找插入的位置
   index := sort.Search(len(n.inodes), func(i int) bool { return bytes.Compare(n.inodes[i].key, oldKey) != -1 })

   exact := (len(n.inodes) > 0 && index < len(n.inodes) && bytes.Equal(n.inodes[index].key, oldKey))
   if !exact { //key不存在
      n.inodes = append(n.inodes, inode{})
      copy(n.inodes[index+1:], n.inodes[index:]) //将index及其之后的元素往后移动
   }
	//根据index获取inode，填充inode
   inode := &n.inodes[index]
   inode.key = newKey
   inode.value = value
   inode.pgid = pgid
}
```

### 读数据

```go
func (db *DB) View(fn func(*Tx) error) error {
   t, err := db.Begin(false)
   if err != nil {
      return err
   }

   // Mark as a managed tx so that the inner function cannot manually rollback.
   t.managed = true

   // If an error is returned from the function then pass it through.
   err = fn(t)
   t.managed = false
   if err != nil {
      _ = t.Rollback()
      return err
   }

   if err := t.Rollback(); err != nil {
      return err
   }

   return nil
}
```

```go
func (tx *Tx) Bucket(name string) *Bucket { //先获取Bucket
   b := tx.buckets.get(name)
   if b == nil {
      return nil
   }

   return &Bucket{
      bucket: b,
      name:   name,
      tx:     tx,
   }
}
```

```go
func (b *Bucket) Get(key []byte) []byte {//再从Bucket中查找数据
   c := b.Cursor() //创建游标
   k, v := c.Seek(key) //查询key是否存在

   // If our target node isn't the same key as what's passed in then return nil.
   if !bytes.Equal(key, k) {
      return nil
   }
   return v
}
```

### 事务

#### 读事务

```go
func (db *DB) beginTx() (*Tx, error) {
   db.metalock.Lock() //获取元数据独占锁
   defer db.metalock.Unlock()

   db.mmaplock.RLock() //mmap的读锁

   if !db.opened {
      return nil, ErrDatabaseNotOpen
   }

   // 创建事务
   t := &Tx{}
  //初始化事务
   t.init(db)

   //维护了读事务
   db.txs = append(db.txs, t)

   return t, nil
}
```

```go
func (tx *Tx) init(db *DB) {
   tx.db = db
   tx.pages = nil

   // 复制元数据信息
   tx.meta = &meta{}
   db.meta().copy(tx.meta)

   // 读取buckets信息
   tx.buckets = &buckets{}
   tx.buckets.read(tx.page(tx.meta.buckets))

   if tx.writable { //写事务，创建pages、nodes
      tx.pages = make(map[pgid]*page)
      tx.nodes = make(map[pgid]*node)

      // Increment the transaction id.
      tx.meta.txid += txid(1)
   }
}
```

#### 写事务

```go
func (db *DB) beginRWTx() (*Tx, error) {
   db.metalock.Lock() //获取元数据的独占锁
   defer db.metalock.Unlock()

   //获取独占锁
   db.rwlock.Lock()

   // Exit if the database is not open yet.
   if !db.opened {
      db.rwlock.Unlock()
      return nil, ErrDatabaseNotOpen
   }

   //创建事务
   t := &Tx{writable: true}
   t.init(db)
   db.rwtx = t

   //释放pages
   var minid txid = 0xFFFFFFFFFFFFFFFF
   for _, t := range db.txs {
      if t.id() < minid {
         minid = t.id()
      }
   }
   if minid > 0 {
      db.freelist.release(minid - 1)
   }

   return t, nil
}
```

#### 提交事务

```go
func (tx *Tx) Commit() error {
   if tx.managed {
      panic("managed tx commit not allowed")
   } else if tx.db == nil {
      return ErrTxClosed
   } else if !tx.writable {
      return ErrTxNotWritable
   }

   // TODO(benbjohnson): Use vectorized I/O to write out dirty pages.

   //删除后的重平衡，记录耗费的时间
   var startTime = time.Now()
   tx.rebalance()
   tx.stats.RebalanceTime += time.Since(startTime)

   // 分裂page
   startTime = time.Now()
   if err := tx.spill(); err != nil {
      tx.close()
      return err
   }
   tx.stats.SpillTime += time.Since(startTime)

   // 为buckets分配page
   p, err := tx.allocate((tx.buckets.size() / tx.db.pageSize) + 1)
   if err != nil {
      tx.close()
      return err
   }
   tx.buckets.write(p) //将buckets写入新的page

   // 释放先前的buckets所属的page
   tx.db.freelist.free(tx.id(), tx.page(tx.meta.buckets))
   tx.meta.buckets = p.id //指向新page

   // Free the freelist and allocate new pages for it. This will overestimate
   // the size of the freelist but not underestimate the size (which would be bad).
   tx.db.freelist.free(tx.id(), tx.page(tx.meta.freelist))
   p, err = tx.allocate((tx.db.freelist.size() / tx.db.pageSize) + 1)
   if err != nil {
      tx.close()
      return err
   }
  //持久化freelist
   tx.db.freelist.write(p)
   tx.meta.freelist = p.id

   // 持久化dirty page
   startTime = time.Now()
   if err := tx.write(); err != nil {
      tx.close()
      return err
   }

   // 持久化meta page
   if err := tx.writeMeta(); err != nil {
      tx.close()
      return err
   }
   tx.stats.WriteTime += time.Since(startTime)

   // 关闭事务
   tx.close()

   // 执行事务回调方法
   for _, fn := range tx.commitHandlers {
      fn()
   }

   return nil
}
```

#### 关闭事务

```go
func (tx *Tx) close() {
   if tx.writable { //写事务
      // 合并统计信息
      tx.db.metalock.Lock()
      tx.db.stats.TxStats.add(&tx.stats)
      tx.db.metalock.Unlock()

      //释放写锁
      tx.db.rwlock.Unlock()
   } else { //读事务
      tx.db.removeTx(tx)
   }
   tx.db = nil
}
```

```go
func (db *DB) removeTx(tx *Tx) {
   db.metalock.Lock()
   defer db.metalock.Unlock()

   //释放mmap的读锁 
   db.mmaplock.RUnlock()

   // 移除事务
   for i, t := range db.txs {
      if t == tx {
         db.txs = append(db.txs[:i], db.txs[i+1:]...)
         break
      }
   }

   // 合并统计信息
   db.stats.TxStats.add(&tx.stats)
}
```

# Alarm

etcd/etcdserver/server.go

```go
func (s *EtcdServer) restoreAlarms() error {
   s.applyV3 = s.newApplierV3()
   as, err := alarm.NewAlarmStore(s)
   if err != nil {
      return err
   }
   s.alarmStore = as
   if len(as.Get(pb.AlarmType_NOSPACE)) > 0 { //剩余空间不足
      s.applyV3 = newApplierV3Capped(s.applyV3)
   }
   return nil
}
```

```go
func (s *EtcdServer) newApplierV3() applierV3 {
   return newAuthApplierV3(
      s.AuthStore(),
      newQuotaApplierV3(s, &applierV3backend{s}),
   )
}
```

```go
func NewAlarmStore(bg BackendGetter) (*AlarmStore, error) {
   ret := &AlarmStore{types: make(map[pb.AlarmType]alarmSet), bg: bg}
   err := ret.restore()
   return ret, err
}
```

```go
func (a *AlarmStore) restore() error {
   b := a.bg.Backend()
   tx := b.BatchTx()

   tx.Lock()
   tx.UnsafeCreateBucket(alarmBucketName)
   err := tx.UnsafeForEach(alarmBucketName, func(k, v []byte) error {
      var m pb.AlarmMember
      if err := m.Unmarshal(k); err != nil {
         return err
      }
      a.addToMap(&m)
      return nil
   })
   tx.Unlock()

   b.ForceCommit()
   return err
}
```

```go
func (a *AlarmStore) addToMap(newAlarm *pb.AlarmMember) *pb.AlarmMember {
   t := a.types[newAlarm.Alarm]
   if t == nil {
      t = make(alarmSet)
      a.types[newAlarm.Alarm] = t
   }
   m := t[types.ID(newAlarm.MemberID)]
   if m != nil {
      return m
   }
   t[types.ID(newAlarm.MemberID)] = newAlarm
   return newAlarm
}
```

# MVCC多版本并发控制

## revision结构体

```go
type revision struct {
   // main is the main revision of a set of changes that happen atomically.
   main int64 //每个事务都有唯一的事务Id，全局递增

   // sub is the the sub revision of a change in a set of changes that happen
   // atomically. Each change has different increasing sub revision in that
   // set.
   sub int64 //一个事务内有多个操作，事务内sub从0开始递增
}
```

## keyIndex结构体

```go
type keyIndex struct {
   key         []byte //key
   modified    revision // 最后一次需改对应的revision信息
   generations []generation //历史修改信息，一个key从有到无对应一个generation
}
```

## generation结构体

```go
type generation struct {
   ver     int64 //从0开始递增，每次修改都会递增
   created revision // 创建时的revision信息
   revs    []revision//修改的历史revision信息
}
```

## 追加数据

```go
func (s *store) Put(key, value []byte, lease lease.LeaseID) int64 {
   id := s.TxnBegin() //开启事务
   s.put(key, value, lease)
   s.txnEnd(id) //结束事务

   putCounter.Inc()

   return int64(s.currentRev.main)
}
```



```go
func (s *store) TxnBegin() int64 {
   s.mu.Lock() //store的独占锁
   s.currentRev.sub = 0
   s.tx = s.b.BatchTx() 
   s.tx.Lock()//batchTx的独占锁

   s.txnID = rand.Int63() //事务Id
   return s.txnID
}
```

etcd/storage/kvstore.go

```go
func (s *store) put(key, value []byte, leaseID lease.LeaseID) {
   rev := s.currentRev.main + 1 //主版本信息
   c := rev
   oldLease := lease.NoLease

   // 查找key的最后一次修改的revision、创建时的revision、version
   grev, created, ver, err := s.kvindex.Get(key, rev)
   if err == nil {
      c = created.main
      ibytes := newRevBytes()
      revToBytes(grev, ibytes)
     //获取kv
      _, vs := s.tx.UnsafeRange(keyBucketName, ibytes, nil, 0)
      var kv storagepb.KeyValue
      if err = kv.Unmarshal(vs[0]); err != nil {
         log.Fatalf("storage: cannot unmarshal value: %v", err)
      }
      oldLease = lease.LeaseID(kv.Lease)
   }

   ibytes := newRevBytes()
   revToBytes(revision{main: rev, sub: s.currentRev.sub}, ibytes)

   ver = ver + 1
   kv := storagepb.KeyValue{
      Key:            key,
      Value:          value,
      CreateRevision: c,
      ModRevision:    rev,
      Version:        ver,
      Lease:          int64(leaseID),
   }

   d, err := kv.Marshal()
   if err != nil {
      log.Fatalf("storage: cannot marshal event: %v", err)
   }
  //写boltdb（main_sub -> keyvalue）
   s.tx.UnsafeSeqPut(keyBucketName, ibytes, d)
  //内存索引（key -> keyIndex）
   s.kvindex.Put(key, revision{main: rev, sub: s.currentRev.sub})
  //变更信息
   s.changes = append(s.changes, kv)
   s.currentRev.sub += 1

   if oldLease != lease.NoLease {
      if s.le == nil {
         panic("no lessor to detach lease")
      }
			//将key从lease中删除
      err = s.le.Detach(oldLease, []lease.LeaseItem{{Key: string(key)}})
      if err != nil {
         panic("unexpected error from lease detach")
      }
   }

   if leaseID != lease.NoLease {
      if s.le == nil {
         panic("no lessor to attach lease")
      }
			//将leaseId和key进行关联
      err = s.le.Attach(leaseID, []lease.LeaseItem{{Key: string(key)}})
      if err != nil {
         panic("unexpected error from lease Attach")
      }
   }
}
```

```go
func (s *store) txnEnd(txnID int64) error {
   if txnID != s.txnID {
      return ErrTxnIDMismatch
   }

   s.tx.Unlock() //释放batchTx的独占锁
   if s.currentRev.sub != 0 { //有事务进行
      s.currentRev.main += 1 //主版本+1
   }
   s.currentRev.sub = 0 //重置事务中的子操作版本为0

   dbTotalSize.Set(float64(s.b.Size())) //db占用空间的大小
   s.mu.Unlock() //释放store的独占锁
   return nil
}
```

## 范围查询

```go
func (s *store) Range(key, end []byte, limit, rangeRev int64) (kvs []storagepb.KeyValue, rev int64, err error) {
   id := s.TxnBegin() //加锁方式和追加数据一样。读写争抢同一把锁，写事务会因为 expensive read request 长时间阻塞，使延迟过大
   kvs, rev, err = s.rangeKeys(key, end, limit, rangeRev)
   s.txnEnd(id)
   rangeCounter.Inc()
   return kvs, rev, err
}
```

gSTM软件事务内存

## 串行读线性读

**串行读，适用于对数据敏感度较低,对时效性要求不高的场景**。直接从状态机读取数据，无需通过raft协议与集群节点交互，避免了磁盘写和网络通信

**线性读，适用于对数据敏感性高的场景。**一旦一个值更新成功，随后任何通过线性读的 client 都能及时访问到。

它需要经过 Raft 协议模块，因此在延时和吞吐量上相比串行读略差一点，适用于对数据一致性要求高的场景。

# Etcd-3.0

## AuthServer

鉴权服务

### 创建AuthServer

```go
func NewAuthServer(s *etcdserver.EtcdServer) *AuthServer {
   return &AuthServer{authenticator: s}
}
```

### 启用禁用鉴权

都会走raft

```go
func (as *AuthServer) AuthEnable(ctx context.Context, r *pb.AuthEnableRequest) (*pb.AuthEnableResponse, error) {//启用鉴权
   resp, err := as.authenticator.AuthEnable(ctx, r)
   if err != nil {
      return nil, togRPCError(err)
   }，
   return resp, nil
}

func (as *AuthServer) AuthDisable(ctx context.Context, r *pb.AuthDisableRequest) (*pb.AuthDisableResponse, error) {//禁用鉴权
   resp, err := as.authenticator.AuthDisable(ctx, r)
   if err != nil {
      return nil, togRPCError(err)
   }
   return resp, nil
}
```

### 鉴权

```go
func (s *EtcdServer) Authenticate(ctx context.Context, r *pb.AuthenticateRequest) (*pb.AuthenticateResponse, error) {
   st, err := s.AuthStore().GenSimpleToken()
   if err != nil {
      return nil, err
   }

   internalReq := &pb.InternalAuthenticateRequest{
      Name:        r.Name,
      Password:    r.Password,
      SimpleToken: st,
   }
	//用户名密码鉴权，走raft
   result, err := s.processInternalRaftRequest(ctx, pb.InternalRaftRequest{Authenticate: internalReq})
   if err != nil {
      return nil, err
   }
   if result.err != nil {
      return nil, result.err
   }
   return result.resp.(*pb.AuthenticateResponse), nil
}
```



```go
func (as *authStore) Authenticate(ctx context.Context, username, password string) (*pb.AuthenticateResponse, error) {
   // TODO(mitake): after adding jwt support, branching based on values of ctx is required
   index := ctx.Value("index").(uint64)
   simpleToken := ctx.Value("simpleToken").(string)

   tx := as.be.BatchTx()
   tx.Lock()
   defer tx.Unlock()

   user := getUser(tx, username)//查询用户是否存在
   if user == nil {
      return nil, ErrAuthFailed
   }
		//密码是否匹配
   if bcrypt.CompareHashAndPassword(user.Password, []byte(password)) != nil {
      plog.Noticef("authentication failed, invalid password for user %s", username)
      return &pb.AuthenticateResponse{}, ErrAuthFailed
   }
	//分配token
   token := fmt.Sprintf("%s.%d", simpleToken, index)
   as.assignSimpleTokenToUser(username, token)

   plog.Infof("authorized %s, token is %s", username, token)
   return &pb.AuthenticateResponse{Token: token}, nil
}
```

## quotaKVServer

在接收到事务类请求时，会判断是否足够的空间，默认最大空间8g。超过8g，向boltdb写入NOSPACE，

,此后集群拒绝事务类请求，只提供读服务。

当调大配额之后，仍然会拒绝写请求，此时需要发送一个取消告警（etcdctl alarm disarm）的命令，以消除空间不足的告警，即将NOSPACE从boltdb中删除

注意开启压缩配置是否开启、配置是否合理，否则db会一直膨胀

### 创建quotaKVServer

etcd/etcdserver/api/v3rpc/quota.go

```go
func NewQuotaKVServer(s *etcdserver.EtcdServer) pb.KVServer {
   return &quotaKVServer{
      NewKVServer(s),
      quotaAlarmer{etcdserver.NewBackendQuota(s), s, s.ID()},
   }
}
```

```go
func NewKVServer(s *etcdserver.EtcdServer) pb.KVServer {
   return &kvServer{hdr: newHeader(s), kv: s}
}
```



```go
func NewBackendQuota(s *EtcdServer) Quota {
   if s.Cfg.QuotaBackendBytes < 0 { .//不会检查空间是否充足
      plog.Warningf("disabling backend quota")
      return &passthroughQuota{}
   }
   if s.Cfg.QuotaBackendBytes == 0 { //默认2g
      return &backendQuota{s, backend.DefaultQuotaBytes}
   }
   if s.Cfg.QuotaBackendBytes > backend.MaxQuotaBytes { //超过8g，就设置8g
      plog.Warningf("backend quota %v exceeds maximum quota %v; using maximum", s.Cfg.QuotaBackendBytes, backend.MaxQuotaBytes)
      return &backendQuota{s, backend.MaxQuotaBytes}
   }
   return &backendQuota{s, s.Cfg.QuotaBackendBytes} //默认最大空间为8g
}
```

### 检测空间是否充足

```go
func (qa *quotaAlarmer) check(ctx context.Context, r interface{}) error {
  //检测空间是否充足
   if qa.q.Available(r) {
      return nil
   }
  //空间不足，发送AlarmRequest
   req := &pb.AlarmRequest{
      MemberID: uint64(qa.id),
      Action:   pb.AlarmRequest_ACTIVATE,
      Alarm:    pb.AlarmType_NOSPACE,
   }
   qa.a.Alarm(ctx, req)
   return rpctypes.ErrGRPCNoSpace
}
```



```go
func (b *backendQuota) Available(v interface{}) bool {
   // 已经占用的空间 + 请求数据占用的空间
   return b.s.Backend().Size()+int64(b.Cost(v)) < b.maxBackendBytes
}
```

### 接收Alarm请求

```go
func (ms *maintenanceServer) Alarm(ctx context.Context, ar *pb.AlarmRequest) (*pb.AlarmResponse, error) { //可能来自客户端请求或者本机空间不足调用此方法
   return ms.a.Alarm(ctx, ar)
}
```

etcd/etcdserver/v3_server.go:272

```go
func (s *EtcdServer) Alarm(ctx context.Context, r *pb.AlarmRequest) (*pb.AlarmResponse, error) {
  //走raft协议，并同步至其他节点，告知db无可用空间，日志也会进行持久化
   result, err := s.processInternalRaftRequest(ctx, pb.InternalRaftRequest{Alarm: r})
   if err != nil {
      return nil, err
   }
   if result.err != nil {
      return nil, result.err
   }
   return result.resp.(*pb.AlarmResponse), nil
}
```



### 处理Alarm请求

etcd/etcdserver/apply.go

```go
func (a *applierV3backend) Alarm(ar *pb.AlarmRequest) (*pb.AlarmResponse, error) {
   resp := &pb.AlarmResponse{}
   oldCount := len(a.s.alarmStore.Get(ar.Alarm))

   switch ar.Action {
   case pb.AlarmRequest_GET:
      resp.Alarms = a.s.alarmStore.Get(ar.Alarm)
   case pb.AlarmRequest_ACTIVATE: //激活alarm
      m := a.s.alarmStore.Activate(types.ID(ar.MemberID), ar.Alarm)//内存、磁盘存储alarm信息
      if m == nil {
         break
      }
      resp.Alarms = append(resp.Alarms, m)
      activated := oldCount == 0 && len(a.s.alarmStore.Get(m.Alarm)) == 1
      if !activated {
         break
      }

      switch m.Alarm {
      case pb.AlarmType_NOSPACE: //空间不足，替换applyv3为applierV3Capped
         plog.Warningf("alarm raised %+v", m)
         a.s.applyV3 = newApplierV3Capped(a)
      default:
         plog.Errorf("unimplemented alarm activation (%+v)", m)
      }
   case pb.AlarmRequest_DEACTIVATE: //撤销alarm
      m := a.s.alarmStore.Deactivate(types.ID(ar.MemberID), ar.Alarm)
      if m == nil {
         break
      }
      resp.Alarms = append(resp.Alarms, m)
      deactivated := oldCount > 0 && len(a.s.alarmStore.Get(ar.Alarm)) == 0
      if !deactivated {
         break
      }

      switch m.Alarm {
      case pb.AlarmType_NOSPACE:
         plog.Infof("alarm disarmed %+v", ar)
         a.s.applyV3 = a.s.newApplierV3()
      default:
         plog.Errorf("unimplemented alarm deactivation (%+v)", m)
      }
   default:
      return nil, nil
   }
   return resp, nil
}
```

#### 激活alarm

```go
func (a *AlarmStore) Activate(id types.ID, at pb.AlarmType) *pb.AlarmMember {
   a.mu.Lock()
   defer a.mu.Unlock()

   newAlarm := &pb.AlarmMember{MemberID: uint64(id), Alarm: at}
   if m := a.addToMap(newAlarm); m != newAlarm { //存放到内存map中
      return m
   }

   v, err := newAlarm.Marshal()
   if err != nil {
      plog.Panicf("failed to marshal alarm member")
   }
	//存放到boltdb
   b := a.bg.Backend()
   b.BatchTx().Lock()
   b.BatchTx().UnsafePut(alarmBucketName, v, nil)
   b.BatchTx().Unlock()

   return newAlarm
}
```

applierV3Capped结构体

```go
func newApplierV3Capped(base applierV3) applierV3 {  //拒绝事务类请求，返回ErrNoSpace
  	return &applierV3Capped{applierV3: base} 
}
```

```go
func (a *applierV3Capped) Put(txnID int64, p *pb.PutRequest) (*pb.PutResponse, error) {
   return nil, ErrNoSpace
}

func (a *applierV3Capped) Txn(r *pb.TxnRequest) (*pb.TxnResponse, error) {
   if a.q.Cost(r) > 0 {
      return nil, ErrNoSpace
   }
   return a.applierV3.Txn(r)
}
```

#### 撤销alarm

```go
func (a *AlarmStore) Deactivate(id types.ID, at pb.AlarmType) *pb.AlarmMember {
   a.mu.Lock()
   defer a.mu.Unlock()

   t := a.types[at]
   if t == nil {
      t = make(alarmSet)
      a.types[at] = t
   }
   m := t[id]
   if m == nil {
      return nil
   }

   delete(t, id)

   v, err := m.Marshal()
   if err != nil {
      plog.Panicf("failed to marshal alarm member")
   }

   b := a.bg.Backend()
   b.BatchTx().Lock()
   b.BatchTx().UnsafeDelete(alarmBucketName, v)
   b.BatchTx().Unlock()

   return m
}
```

### 启动时查看是否有NO_SPACE

etcd/etcdserver/server.go

```go
func (s *EtcdServer) restoreAlarms() error {
   s.applyV3 = s.newApplierV3()
   as, err := alarm.NewAlarmStore(s)
   if err != nil {
      return err
   }
   s.alarmStore = as
   if len(as.Get(pb.AlarmType_NOSPACE)) > 0 {//boltdb中写有NO_SPACE
      s.applyV3 = newApplierV3Capped(s.applyV3)//拒绝事务类请求
   }
   return nil
}
```

## 压缩机制

### 自动压缩

```go
//创建EtcdServer时，它表示启用时间周期性压缩，比如历史版本只会保留1个小时
if h := cfg.AutoCompactionRetention; h != 0 {
   srv.compactor = compactor.NewPeriodic(h, srv.kv, srv)
   srv.compactor.Run()
}
```

```go
func NewPeriodic(h int, rg RevGetter, c Compactable) *Periodic {
   return &Periodic{
      clock:        clockwork.NewRealClock(),
      periodInHour: h, //小时级别的压缩
      rg:           rg, 
      c:            c,
   }
}
```



```go
func (t *Periodic) Run() {
   t.ctx, t.cancel = context.WithCancel(context.Background())
   t.revs = make([]int64, 0)
   clock := t.clock

   go func() {
      last := clock.Now()
      for {
         t.revs = append(t.revs, t.rg.Rev()) //获取最新版本，放入revs数组
         select {
         case <-t.ctx.Done():
            return
         case <-clock.After(checkCompactionInterval): //每隔5分钟，获取一次版本号，放入revs数组
            t.mu.Lock()
            p := t.paused
            t.mu.Unlock()
            if p {
               continue
            }
         }
         if clock.Now().Sub(last) < time.Duration(t.periodInHour)*time.Hour {//小时级别的合并
            continue
         }
				//超过设置的保留时长，获取periodInHour之前的版本号
         rev := t.getRev(t.periodInHour)
         if rev < 0 {
            continue
         }

         plog.Noticef("Starting auto-compaction at revision %d", rev)
        //合并treeIndex、boltdb
         _, err := t.c.Compact(t.ctx, &pb.CompactionRequest{Revision: rev})
         if err == nil || err == mvcc.ErrCompacted {
            t.revs = make([]int64, 0)
            last = clock.Now()
            plog.Noticef("Finished auto-compaction at revision %d", rev)
         } else {
            plog.Noticef("Failed auto-compaction at revision %d (%v)", err, rev)
            plog.Noticef("Retry after %v", checkCompactionInterval)
         }
      }
   }()
}
```



```go
func (t *Periodic) getRev(h int) int64 { //获取版本，默认获取数组首元素
   i := len(t.revs) - int(time.Duration(h)*time.Hour/checkCompactionInterval)
   if i < 0 {
      return -1
   }
   return t.revs[i]
}
```

etcd/etcdserver/v3_server.go

```go
func (s *EtcdServer) Compact(ctx context.Context, r *pb.CompactionRequest) (*pb.CompactionResponse, error) {
  //提交合并请求，走raft协议，各个节点执行合并任务
   result, err := s.processInternalRaftRequest(ctx, pb.InternalRaftRequest{Compaction: r})
   if r.Physical && result != nil && result.physc != nil {
      <-result.physc
      s.be.ForceCommit()
   }
   if err != nil {
      return nil, err
   }
   if result.err != nil {
      return nil, result.err
   }
   resp := result.resp.(*pb.CompactionResponse)
   if resp == nil {
      resp = &pb.CompactionResponse{}
   }
   if resp.Header == nil {
      resp.Header = &pb.ResponseHeader{}
   }
   resp.Header.Revision = s.kv.Rev()
   return resp, nil
}
```

etcd/etcdserver/apply.go

```go
func (a *applierV3backend) Compaction(compaction *pb.CompactionRequest) (*pb.CompactionResponse, <-chan struct{}, error) {
   resp := &pb.CompactionResponse{}
   resp.Header = &pb.ResponseHeader{}
  //压缩
   ch, err := a.s.KV().Compact(compaction.Revision)
   if err != nil {
      return nil, ch, err
   }
   rr, _ := a.s.KV().Range([]byte("compaction"), nil, mvcc.RangeOptions{})
   resp.Header.Revision = rr.Rev
   return resp, ch, err
}
```



```go
func (s *store) Compact(rev int64) (<-chan struct{}, error) {
   s.mu.Lock()
   defer s.mu.Unlock()
   if rev <= s.compactMainRev { //小于等于之前的压缩版本
      ch := make(chan struct{})
      f := func(ctx context.Context) { s.compactBarrier(ctx, ch) }
      s.fifoSched.Schedule(f)
      return ch, ErrCompacted
   }
   if rev > s.currentRev.main { //大于当前的版本
      return nil, ErrFutureRev
   }

   start := time.Now()

   s.compactMainRev = rev

   rbytes := newRevBytes()
   revToBytes(revision{main: rev}, rbytes)

  //压缩开始时，保存压缩版本，防止压缩期间宕机或者出现异常，导致各节点的数据不一致
  //压缩完成时也会保存一个压缩结束时遍历的最后一个版本
  //启动时，会判断是否继续执行未完成的压缩任务
   tx := s.b.BatchTx()
   tx.Lock()
   tx.UnsafePut(metaBucketName, scheduledCompactKeyName, rbytes)
   tx.Unlock()
   // ensure that desired compaction is persisted
   s.b.ForceCommit()
	//压缩treeIndex中的key（保留最大版本号的数据即最新的数据、删除小于等于rev的key）
   keep := s.kvindex.Compact(rev)//返回有效的版本号
   ch := make(chan struct{})
  //根据有效的版本号删除废弃的历史版本数据
   var j = func(ctx context.Context) {
      if ctx.Err() != nil {
         s.compactBarrier(ctx, ch)
         return
      }
      if !s.scheduleCompaction(rev, keep) {
         s.compactBarrier(nil, ch)
         return
      }
      close(ch)
   }
   s.fifoSched.Schedule(j) //调度blotdb压缩任务

   indexCompactionPauseDurations.Observe(float64(time.Since(start) / time.Millisecond))
   return ch, nil
}
```

#### treeIndex压缩

etcd/mvcc/index.go

```go
func (ti *treeIndex) Compact(rev int64) map[revision]struct{} {
   available := make(map[revision]struct{})
   var emptyki []*keyIndex
   plog.Printf("store.index: compact %d", rev)
   // TODO: do not hold the lock for long time?
   // This is probably OK. Compacting 10M keys takes O(10ms).
   ti.Lock() //加锁
   defer ti.Unlock() //释放锁
   ti.tree.Ascend(compactIndex(rev, available, &emptyki))
   for _, ki := range emptyki {
      item := ti.tree.Delete(ki) //从内存中删除低版本的索引信息
      if item == nil {
         plog.Panic("store.index: unexpected delete failure during compaction")
      }
   }
   return available
}
```



```go
func compactIndex(rev int64, available map[revision]struct{}, emptyki *[]*keyIndex) func(i btree.Item) bool {
   return func(i btree.Item) bool {
      keyi := i.(*keyIndex)
      keyi.compact(rev, available)
      if keyi.isEmpty() {
         *emptyki = append(*emptyki, keyi)
      }
      return true
   }
}
```

#### boltdb压缩

压缩任务在遍历、删除 key 的过程中可能会对 boltdb 造成压力

为了不影响正常读写请求，在执行过程中会通过参数控制每次遍历删除的 key 数（默认为10000，每批间隔 10ms），分批完成key的删除

etcd/mvcc/kvstore_compaction.go

```go
func (s *store) scheduleCompaction(compactMainRev int64, keep map[revision]struct{}) bool {
   totalStart := time.Now()
   defer dbCompactionTotalDurations.Observe(float64(time.Since(totalStart) / time.Millisecond))

   end := make([]byte, 8)
   binary.BigEndian.PutUint64(end, uint64(compactMainRev+1))

   batchsize := int64(10000) //每次遍历默认获取10000条数据
   last := make([]byte, 8+1+8)
   for {
      var rev revision

      start := time.Now()
     //批次删除
      tx := s.b.BatchTx()
      tx.Lock()
			//范围查询10000条数据
      keys, _ := tx.UnsafeRange(keyBucketName, last, end, batchsize)
      for _, key := range keys {
         rev = bytesToRev(key)
         if _, ok := keep[rev]; !ok { //版本号是否有效
            tx.UnsafeDelete(keyBucketName, key) //无效直接删除
         }
      }

      if len(keys) < int(batchsize) { //小于10000，遍历结束
         rbytes := make([]byte, 8+1+8)
         revToBytes(revision{main: compactMainRev}, rbytes)
        //压缩任务完成，保存压缩版本
         tx.UnsafePut(metaBucketName, finishedCompactKeyName, rbytes)
         tx.Unlock()
         plog.Printf("finished scheduled compaction at %d (took %v)", compactMainRev, time.Since(totalStart))
         return true
      }

      // 更新last
      revToBytes(revision{main: rev.main, sub: rev.sub + 1}, last)
     //释放锁
      tx.Unlock()
      dbCompactionPauseDurations.Observe(float64(time.Since(start) / time.Millisecond))

      select {
      case <-time.After(100 * time.Millisecond): //休眠100毫秒
      case <-s.stopc:
         return false
      }
   }
}
```

## FIFOScheduler

etcd/pkg/schedule/schedule.go

```go
func NewFIFOScheduler() Scheduler { //先进先出调度器。主要用来调度合并任务和应用到状态机的任务
   f := &fifo{
      resume: make(chan struct{}, 1),
      donec:  make(chan struct{}, 1),
   }
   f.finishCond = sync.NewCond(&f.mu)
   f.ctx, f.cancel = context.WithCancel(context.Background())
   go f.run()
   return f
}
```

### 运行调度任务

```go
func (f *fifo) run() {
   // TODO: recover from job panic?
   defer func() {
      close(f.donec)
      close(f.resume)
   }()

   for {
      var todo Job
      f.mu.Lock()
     //获取首位的任务
      if len(f.pendings) != 0 {
         f.scheduled++
         todo = f.pendings[0]
      }
      f.mu.Unlock()
      if todo == nil { //暂无任务
         select {
         case <-f.resume: //等待外部提交任务
         case <-f.ctx.Done():
            f.mu.Lock()
            pendings := f.pendings
            f.pendings = nil
            f.mu.Unlock()
            // clean up pending jobs
            for _, todo := range pendings {
               todo(f.ctx)
            }
            return
         }
      } else { //有任务
         todo(f.ctx) //云心任务
         f.finishCond.L.Lock()
         f.finished++
         f.pendings = f.pendings[1:] //切片，移除完成的任务
         f.finishCond.Broadcast()
         f.finishCond.L.Unlock()
      }
   }
}
```

### 提交调度任务

```go
func (f *fifo) Schedule(j Job) { //将任务放入调度器
   f.mu.Lock()
   defer f.mu.Unlock()

   if f.cancel == nil {
      panic("schedule: schedule to stopped scheduler")
   }

   if len(f.pendings) == 0 {
      select {
      case f.resume <- struct{}{}: //唤醒
      default:
      }
   }
   f.pendings = append(f.pendings, j) //将任务放入数组

   return
}
```

## 串行读线性读

etcd/etcdserver/v3_server.go

```go
func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
   var result *applyResult
   var err error

   if r.Serializable {//串行读，直接从状态机读取数据
      var user string
      user, err = s.usernameFromCtx(ctx)
      if err != nil {
         return nil, err
      }
      result = s.applyV3.Apply( //状态机读取数据
         &pb.InternalRaftRequest{
            Header: &pb.RequestHeader{Username: user},
            Range:  r})
   } else { //线性读，走一遍raft协议涉及到磁盘持久化和网络传输
      result, err = s.processInternalRaftRequest(ctx, pb.InternalRaftRequest{Range: r})
   }
   if err != nil {
      return nil, err
   }
   if result.err != nil {
      return nil, result.err
   }
   return result.resp.(*pb.RangeResponse), nil
}
```

## 幂等性

```go
type consistentIndex uint64

func (i *consistentIndex) setConsistentIndex(v uint64) {
   atomic.StoreUint64((*uint64)(i), v)
}

func (i *consistentIndex) ConsistentIndex() uint64 {
   return atomic.LoadUint64((*uint64)(i))
}
```

```go
//创建EtcdServer时的部分代码
srv.kv = mvcc.New(srv.be, srv.lessor, &srv.consistIndex)
if beExist { //db文件存在
   kvindex := srv.kv.ConsistentIndex() //读取之前存储的consistent_index
   // 快照存在的情况下，consistent_index不可能小于snapshot.Metadata.Index
   if snapshot != nil && kvindex < snapshot.Metadata.Index {
      if kvindex != 0 {
         return nil, fmt.Errorf("database file (%v index %d) does not match with snapshot (index %d).", bepath, kvindex, snapshot.Metadata.Index)
      }
      plog.Warningf("consistent index never saved (snapshot index=%d)", snapshot.Metadata.Index)
   }
}
//设置consistIndex
srv.consistIndex.setConsistentIndex(srv.kv.ConsistentIndex())
```

```go
func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry) { //应用状态机时调用此方法
   shouldApplyV3 := false
  //日志索引大于ConsistentIndex，表示未执行，日志索引时单调递增的
  //ConsistentIndex存放在boltdb中，需要保证ConsistentIndex的保存和日志项保存的原子性
  //在开启事务时，写入ConsistentIndex
   if e.Index > s.consistIndex.ConsistentIndex() { 
      s.consistIndex.setConsistentIndex(e.Index)
      shouldApplyV3 = true //应用v3版本的状态机
   }

   // raft state machine may generate noop entry when leader confirmation.
   // skip it in advance to avoid some potential bug in the future
   if len(e.Data) == 0 {
      select {
      case s.forceVersionC <- struct{}{}:
      default:
      }
      return
   }

   var raftReq pb.InternalRaftRequest
   if !pbutil.MaybeUnmarshal(&raftReq, e.Data) { // backward compatible
      var r pb.Request
      pbutil.MustUnmarshal(&r, e.Data)
      s.w.Trigger(r.ID, s.applyV2Request(&r))
      return
   }
   if raftReq.V2 != nil {
      req := raftReq.V2
      s.w.Trigger(req.ID, s.applyV2Request(req))
      return
   }

   // 不应用V3版本的状态机
   if !shouldApplyV3 {
      return
   }

   id := raftReq.ID
   if id == 0 {
      id = raftReq.Header.ID
   }

   ar := s.applyV3.Apply(&raftReq)
   s.setAppliedIndex(e.Index)
   if ar.err != ErrNoSpace || len(s.alarmStore.Get(pb.AlarmType_NOSPACE)) > 0 {
      s.w.Trigger(id, ar)
      return
   }
	//空间不足
   plog.Errorf("applying raft message exceeded backend quota")
   go func() {
      a := &pb.AlarmRequest{
         MemberID: uint64(s.ID()),
         Action:   pb.AlarmRequest_ACTIVATE,
         Alarm:    pb.AlarmType_NOSPACE,
      }
      r := pb.InternalRaftRequest{Alarm: a}
      s.processInternalRaftRequest(context.TODO(), r)
      s.w.Trigger(id, ar)
   }()
}
```

```go
func (s *store) saveIndex() {//开启事务时调用此方法
   if s.ig == nil {
      return
   }
   tx := s.tx
   bs := s.bytesBuf8
   binary.BigEndian.PutUint64(bs, s.ig.ConsistentIndex())
   tx.UnsafePut(metaBucketName, consistentIndexKeyName, bs)//保存到boltdb
}
```

# ETCD-3.1

## 处理查询请求

```go
func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
   if s.ClusterVersion() == nil || s.ClusterVersion().LessThan(newRangeClusterVersion) {
     //3.1版本之前
      return s.legacyRange(ctx, r)
   }
   var resp *pb.RangeResponse
   var err error
   defer func(start time.Time) {
      warnOfExpensiveReadOnlyRangeRequest(start, r, resp, err)
   }(time.Now())

   if !r.Serializable { //线性读，直到committed之后的日志项应用到状态机
      err = s.linearizableReadNotify(ctx)
      if err != nil {
         return nil, err
      }
   }
   chk := func(ai *auth.AuthInfo) error {
      return s.authStore.IsRangePermitted(ai, r.Key, r.RangeEnd)
   }
   get := func() { resp, err = s.applyV3Base.Range(noTxn, r) }
  //串行读
   if serr := s.doSerialize(ctx, chk, get); serr != nil {
      err = serr
      return nil, err
   }
   return resp, err
}
```



```go
func (s *EtcdServer) linearizableReadNotify(ctx context.Context) error {
   s.readMu.RLock()
  //etcdServer初始化时会创建，每次接收到ReadIndex请求后也会创建
   nc := s.readNotifier //获取readNotifier，主要用于通信，告知readIndex请求处理完成
   s.readMu.RUnlock()

   select {
   case s.readwaitc <- struct{}{}: //表示提交一个ReadIndex请求，启动时会创建协程处理ReadIndex
   default:
   }

   select {
   case <-nc.c: //阻塞
      return nc.err
   case <-ctx.Done():
      return ctx.Err()
   case <-s.done:
      return ErrStopped
   }
}
```

## 线性读之ReadIndex

**无需再走一遍Raft协议**

```go
func (s *EtcdServer) Start() { //etcd启动
   s.start()
   s.goAttach(func() { s.adjustTicks() })
   s.goAttach(func() { s.publish(s.Cfg.ReqTimeout()) })
   s.goAttach(s.purgeFile)
   s.goAttach(func() { monitorFileDescriptor(s.stopping) })
   s.goAttach(s.monitorVersions)
   s.goAttach(s.linearizableReadLoop)  //线性读
}
```



```
ReadOnlySafe实现
Leader收到readindex请求之后，存放到readIndexQueue数组，然后发送心跳请求给集群内的节点，当收到过半节点的ack响应之后，将此readIndex请求之前的所有请求从readIndexQueue数组中移除，放入到raft模块的readStates通道中。ReadOnlySafe确保当前节点的角色仍然是Leader
```



```go
func (s *EtcdServer) linearizableReadLoop() {
   var rs raft.ReadState

   for {
      ctxToSend := make([]byte, 8)
      id1 := s.reqIDGen.Next()
      binary.BigEndian.PutUint64(ctxToSend, id1)

      select {
      case <-s.readwaitc://客户端提交了ReadIndex请求
      case <-s.stopping: //EtcdServer关闭
         return
      }

      nextnr := newNotifier() //创建新的Notifier

      s.readMu.Lock()
      nr := s.readNotifier
      s.readNotifier = nextnr
      s.readMu.Unlock()

      cctx, cancel := context.WithTimeout(context.Background(), s.Cfg.ReqTimeout())
      //处理ReadIndex请求
      if err := s.r.ReadIndex(cctx, ctxToSend); err != nil {
         cancel()
         if err == raft.ErrStopped {
            return
         }
         plog.Errorf("failed to get read index from raft: %v", err)
         readIndexFailed.Inc()
         nr.notify(err)
         continue
      }
      cancel()

      var (
         timeout bool
         done    bool
      )
      for !timeout && !done {
         select {
         case rs = <-s.r.readStateC: //readStateC通道中的ReadIndex请求都已经处理完成
            done = bytes.Equal(rs.RequestCtx, ctxToSend) //相等，结束循环
            if !done {
               id2 := uint64(0)
               if len(rs.RequestCtx) == 8 {
                  id2 = binary.BigEndian.Uint64(rs.RequestCtx)
               }
               slowReadIndex.Inc()
            }

         case <-time.After(s.Cfg.ReqTimeout()):
            nr.notify(ErrTimeout)
            timeout = true
            slowReadIndex.Inc()
         case <-s.stopping:
            return
         }
      }
      if !done {
         continue
      }
			//处理ReadIndex请求时，提交的日志项是否已经都应用到状态机
      if ai := s.getAppliedIndex(); ai < rs.Index {
         select {
         case <-s.applyWait.Wait(rs.Index): //尚未应用到状态机，进行等待
         case <-s.stopping:
            return
         }
      }
     //都已经应用到状态机，唤醒请求
      nr.notify(nil)
   }
}
```



```go
case pb.MsgReadIndex: //处理ReadIndex请求
   if r.quorum() > 1 {
      if r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(r.raftLog.committed)) != r.Term {
         return
      }

      switch r.readOnly.option {
      case ReadOnlySafe:
         r.readOnly.addRequest(r.raftLog.committed, m)
         r.bcastHeartbeatWithCtx(m.Entries[0].Data)
      case ReadOnlyLeaseBased:
         var ri uint64
         if r.checkQuorum {
            ri = r.raftLog.committed
         }
         if m.From == None || m.From == r.id { // from local member
            r.readStates = append(r.readStates, ReadState{Index: r.raftLog.committed, RequestCtx: m.Entries[0].Data})
         } else {
            r.send(pb.Message{To: m.From, Type: pb.MsgReadIndexResp, Index: ri, Entries: m.Entries})
         }
      }
   } else {
      r.readStates = append(r.readStates, ReadState{Index: r.raftLog.committed, RequestCtx: m.Entries[0].Data})
   }
```

etcd/raft/read_only.go:52

```go
fmyunc (ro *readOnly) addRequest(index uint64, m pb.Message) {//维护context
   ctx := string(m.Entries[0].Data)
   if _, ok := ro.pendingReadIndex[ctx]; ok {
      return
   }
  //readIndexQueue中存放接收的readindex请求，
   ro.pendingReadIndex[ctx] = &readIndexStatus{index: index, req: m, acks: make(map[uint64]struct{})}
   ro.readIndexQueue = append(ro.readIndexQueue, ctx)
}
```

etcd/raft/raft.go:466

```go
func (r *raft) bcastHeartbeatWithCtx(ctx []byte) { //发送心跳带有context
   for id := range r.prs {
      if id == r.id {
         continue
      }
      r.sendHeartbeat(id, ctx)
   }
}
```



```go
case pb.MsgHeartbeatResp: //处理心跳响应
   pr.RecentActive = true
   pr.resume()

   if pr.State == ProgressStateReplicate && pr.ins.full() {
      pr.ins.freeFirstOne()
   }
   if pr.Match < r.raftLog.lastIndex() {
      r.sendAppend(m.From)
   }

   if r.readOnly.option != ReadOnlySafe || len(m.Context) == 0 {
      return
   }
	
   ackCount := r.readOnly.recvAck(m)
   if ackCount < r.quorum() {
      return
   }
	//收到过半节点的ack
   rss := r.readOnly.advance(m)
   for _, rs := range rss {
      req := rs.req
      if req.From == None || req.From == r.id { // from local member
         r.readStates = append(r.readStates, ReadState{Index: rs.index, RequestCtx: req.Entries[0].Data})
      } else {
         r.send(pb.Message{To: req.From, Type: pb.MsgReadIndexResp, Index: rs.index, Entries: req.Entries})
      }
   }
```

```go
func (ro *readOnly) advance(m pb.Message) []*readIndexStatus {
   var (
      i     int
      found bool
   )

   ctx := string(m.Context)
   rss := []*readIndexStatus{}

   for _, okctx := range ro.readIndexQueue {
      i++
      rs, ok := ro.pendingReadIndex[okctx]
      if !ok {
         panic("cannot find corresponding read state from pending map")
      }
      rss = append(rss, rs) //获取ctx之前的readIndexStatus
      if okctx == ctx {
         found = true
         break
      }
   }

   if found {//找对了对应的readIndexStatus
      ro.readIndexQueue = ro.readIndexQueue[i:] //切片
      for _, rs := range rss {//从数组中移除
         delete(ro.pendingReadIndex, string(rs.req.Entries[0].Data))
      }
      return rss
   }

   return nil
}
```

## 预投票

避免无效的选举以及任期号term的不断递增

Follower 在转换成 Candidate 状态前，先进入 PreCandidate 状态，不自增任期号， 发起预投票。若获得集群多数节点认可，才能进入 Candidate 状态，发起选举流程。

etcd/raft/raft.go

```go
func (r *raft) tickElection() {
   r.electionElapsed++

   if r.promotable() && r.pastElectionTimeout() { //选举超时
      r.electionElapsed = 0
      r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
   }
}
```

```go
func (r *raft)  Step(m pb.Message) error {
   switch {
   case m.Term == 0://本地消息
      // local message
   case m.Term > r.Term:
      lead := m.From
      if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote {
         force := bytes.Equal(m.Context, []byte(campaignTransfer))
         inLease := r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout
         if !force && inLease {
            r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d, index: %d] at term %d: lease is not expired (remaining ticks: %d)",
               r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term, r.electionTimeout-r.electionElapsed)
            return nil
         }
         lead = None
      }
      switch {
      case m.Type == pb.MsgPreVote:
      case m.Type == pb.MsgPreVoteResp && !m.Reject:
      default:
         r.logger.Infof("%x [term: %d] received a %s message with higher term from %x [term: %d]",
            r.id, r.Term, m.Type, m.From, m.Term)
         r.becomeFollower(m.Term, lead)
      }

   case m.Term < r.Term:
      if r.checkQuorum && (m.Type == pb.MsgHeartbeat || m.Type == pb.MsgApp) {
         r.send(pb.Message{To: m.From, Type: pb.MsgAppResp})
      } else {
         r.logger.Infof("%x [term: %d] ignored a %s message with lower term from %x [term: %d]",
            r.id, r.Term, m.Type, m.From, m.Term)
      }
      return nil
   }

   switch m.Type {
   case pb.MsgHup:
      if r.state != StateLeader {
         ents, err := r.raftLog.slice(r.raftLog.applied+1, r.raftLog.committed+1, noLimit)
         if err != nil {
            r.logger.Panicf("unexpected error getting unapplied entries (%v)", err)
         }
         if n := numOfPendingConf(ents); n != 0 && r.raftLog.committed > r.raftLog.applied {
            r.logger.Warningf("%x cannot campaign at term %d since there are still %d pending configuration changes to apply", r.id, r.Term, n)
            return nil
         }

         r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
         if r.preVote { //开启预投票
            r.campaign(campaignPreElection)
         } else {
            r.campaign(campaignElection)
         }
      } else {
         r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
      }

   case pb.MsgVote, pb.MsgPreVote:
      // The m.Term > r.Term clause is for MsgPreVote. For MsgVote m.Term should
      // always equal r.Term.
      if (r.Vote == None || m.Term > r.Term || r.Vote == m.From) && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
         r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] cast %s for %x [logterm: %d, index: %d] at term %d",
            r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
         r.send(pb.Message{To: m.From, Type: voteRespMsgType(m.Type)})
         if m.Type == pb.MsgVote {
            // Only record real votes.
            r.electionElapsed = 0
            r.Vote = m.From
         }
      } else {
         r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected %s from %x [logterm: %d, index: %d] at term %d",
            r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
         r.send(pb.Message{To: m.From, Type: voteRespMsgType(m.Type), Reject: true})
      }

   default:
      r.step(r, m)
   }
   return nil
}
```



```go
func (r *raft) campaign(t CampaignType) {
   var term uint64
   var voteMsg pb.MessageType
   if t == campaignPreElection { //预选举
      r.becomePreCandidate()
      voteMsg = pb.MsgPreVote
      term = r.Term + 1 //term +1 
   } else {
      r.becomeCandidate()
      voteMsg = pb.MsgVote
      term = r.Term
   }
   if r.quorum() == r.poll(r.id, voteRespMsgType(voteMsg), true) {//得到过半节点的投票支持
      if t == campaignPreElection {
         r.campaign(campaignElection) //开始真正的投票阶段
      } else {
         r.becomeLeader() //直接成为leader
      }
      return
   }
   for id := range r.prs {
      if id == r.id {
         continue
      }
      r.logger.Infof("%x [logterm: %d, index: %d] sent %s request to %x at term %d",
         r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), voteMsg, id, r.Term)

      var ctx []byte
      if t == campaignTransfer {
         ctx = []byte(t)
      }
      r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx}) //发送投票信息
   }
}
```

# ETCD-3.2

```
Change in default `snapshot-count` value
'snapshot-count'的默认值[从10,000更改为100,000]较高的快照计数意味着在丢弃旧条目之前，Raft条目在内存中保存的时间更长。这是较不频繁的快照和[较高的内存使用]之间的权衡。更高的'snapshot-count'将表现为更高的内存使用量，而保留更多的Raft条目有助于提高慢速追随者的可用性:leader仍然能够将其日志复制给follower，而不是强迫follower从leader拉取快照保持数据同步。
```

## watch

mvcc/watchable_store.go

```go
func New(b backend.Backend, le lease.Lessor, ig ConsistentIndexGetter) ConsistentWatchableKV {
   return newWatchableStore(b, le, ig)
}
```

```go
func newWatchableStore(b backend.Backend, le lease.Lessor, ig ConsistentIndexGetter) *watchableStore {
   s := &watchableStore{
      store:    NewStore(b, le, ig),
      victimc:  make(chan struct{}, 1),
      unsynced: newWatcherGroup(),
      synced:   newWatcherGroup(),
      stopc:    make(chan struct{}),
   }
  //读写优化
   s.store.ReadView = &readView{s}
   s.store.WriteView = &writeView{s}
   if s.le != nil {
      // use this store as the deleter so revokes trigger watch events
      s.le.SetRangeDeleter(func() lease.TxnDelete { return s.Write() })
   }
   s.wg.Add(2)
   go s.syncWatchersLoop()
   go s.syncVictimsLoop() //异常场景重试机制
   return s
}
```

```go
func (s *watchableStore) syncVictimsLoop() {
   defer s.wg.Done()

   for {
      for s.moveVictims() != 0 {
         // try to update all victim watchers
      }
      s.mu.RLock()
      isEmpty := len(s.victims) == 0
      s.mu.RUnlock()

      var tickc <-chan time.Time
      if !isEmpty {
         tickc = time.After(10 * time.Millisecond)
      }

      select {
      case <-tickc:
      case <-s.victimc:
      case <-s.stopc:
         return
      }
   }
}
```

```go
func (s *watchableStore) moveVictims() (moved int) {
   s.mu.Lock()
   victims := s.victims
   s.victims = nil
   s.mu.Unlock()

   var newVictim watcherBatch
   for _, wb := range victims {
      // 再次尝试发送watch响应
      for w, eb := range wb {
         rev := w.minRev - 1
         if w.send(WatchResponse{WatchID: w.id, Events: eb.evs, Revision: rev}) {
            pendingEventsGauge.Add(float64(len(eb.evs)))
         } else { //通道已满
            if newVictim == nil {
               newVictim = make(watcherBatch)
            }
            newVictim[w] = eb
            continue
         }
         moved++
      }

      // assign completed victim watchers to unsync/sync
      s.mu.Lock()
      s.store.revMu.RLock()
      curRev := s.store.currentRev
      for w, eb := range wb {
         if newVictim != nil && newVictim[w] != nil {
            // couldn't send watch response; stays victim
            continue
         }
         w.victim = false
         if eb.moreRev != 0 {
            w.minRev = eb.moreRev
         }
         if w.minRev <= curRev {
            s.unsynced.add(w)
         } else {
            slowWatcherGauge.Dec()
            s.synced.add(w)
         }
      }
      s.store.revMu.RUnlock()
      s.mu.Unlock()
   }

   if len(newVictim) > 0 {
      s.mu.Lock()
      s.victims = append(s.victims, newVictim)
      s.mu.Unlock()
   }

   return moved
}
```

## 读优化

mvcc/kv_view.go

```go
func (rv *readView) Range(key, end []byte, ro RangeOptions) (r *RangeResult, err error) {
   tr := rv.kv.Read()
   defer tr.End()
   return tr.Range(key, end, ro)
}
```

### 事务开始

mvcc/kvstore_txn.go

```go
func (s *store) Read() TxnRead {
   s.mu.RLock() //获取store的读锁
   tx := s.b.ReadTx() //在创建backend时创建readTx
   s.revMu.RLock() //revMu主要用来读取当前的版本和合并版本
   tx.Lock() //获取readTx的读锁
   firstRev, rev := s.compactMainRev, s.currentRev
   s.revMu.RUnlock()  //释放revMu读锁
   return newMetricsTxnRead(&storeTxnRead{s, tx, firstRev, rev})
}
```

```go
b := &backend{
   db: db,

   batchInterval: bcfg.BatchInterval,
   batchLimit:    bcfg.BatchLimit,

   readTx: &readTx{buf: txReadBuffer{
      txBuffer: txBuffer{make(map[string]*bucketBuffer)}},
   }, //创建readTx

   stopc: make(chan struct{}),
   donec: make(chan struct{}),
}
b.batchTx = newBatchTxBuffered(b)
```



```go
func (rt *readTx) Lock()   { rt.mu.RLock() }//readTx加读锁
```

```go
func newMetricsTxnRead(tr TxnRead) TxnRead {
   return &metricsTxnWrite{&txnReadWrite{tr}, 0, 0, 0}
}
```

### 读释放锁

```go
func (tw *metricsTxnWrite) End() {
   defer tw.TxnWrite.End() //释放锁
   if sum := tw.ranges + tw.puts + tw.deletes; sum != 1 {
      if sum > 1 {
         txnCounter.Inc()
      }
      return
   }
   switch {
   case tw.ranges == 1:
      rangeCounter.Inc()
   case tw.puts == 1:
      putCounter.Inc()
   case tw.deletes == 1:
      deleteCounter.Inc()
   }
}
```



```go
func (tr *storeTxnRead) End() {
   tr.tx.Unlock() //释放readTx读锁
   tr.s.mu.RUnlock() //释放store的读锁
}
```

### 范围读

先从读缓冲区中查找数据，如果找到直接返回，否则从boltdb查找数据

```go
func (tw *metricsTxnWrite) Range(key, end []byte, ro RangeOptions) (*RangeResult, error) {
   tw.ranges++ //查询次数
   return tw.TxnWrite.Range(key, end, ro)
}
```

```go
func (tr *storeTxnRead) Range(key, end []byte, ro RangeOptions) (r *RangeResult, err error) {
   return tr.rangeKeys(key, end, tr.Rev(), ro)
}
```

```go
func (tr *storeTxnRead) rangeKeys(key, end []byte, curRev int64, ro RangeOptions) (*RangeResult, error) {
   rev := ro.Rev
   if rev > curRev { //请求中的版本号大于当前的版本号
      return &RangeResult{KVs: nil, Count: -1, Rev: curRev}, ErrFutureRev
   }
   if rev <= 0 { 
      rev = curRev
   }
   if rev < tr.s.compactMainRev {//小于合并版本号
      return &RangeResult{KVs: nil, Count: -1, Rev: 0}, ErrCompacted
   }
	//根据key的范围以及版本号从内存B+树查询keyIndex
   _, revpairs := tr.s.kvindex.Range(key, end, int64(rev))
   if len(revpairs) == 0 {
      return &RangeResult{KVs: nil, Count: 0, Rev: curRev}, nil
   }
   if ro.Count { //获取key的个数
      return &RangeResult{KVs: nil, Count: len(revpairs), Rev: curRev}, nil
   }

   var kvs []mvccpb.KeyValue
   for _, revpair := range revpairs {
      start, end := revBytesRange(revpair)
      _, vs := tr.tx.UnsafeRange(keyBucketName, start, end, 0)
      if len(vs) != 1 {
         plog.Fatalf("range cannot find rev (%d,%d)", revpair.main, revpair.sub)
      }
	
      var kv mvccpb.KeyValue
      if err := kv.Unmarshal(vs[0]); err != nil {
         plog.Fatalf("cannot unmarshal event: %v", err)
      }
      kvs = append(kvs, kv)
      if ro.Limit > 0 && len(kvs) >= int(ro.Limit) {
         break
      }
   }
   return &RangeResult{KVs: kvs, Count: len(revpairs), Rev: curRev}, nil
}
```

```go
func (rt *readTx) UnsafeRange(bucketName, key, endKey []byte, limit int64) ([][]byte, [][]byte) {
   if endKey == nil {
      // forbid duplicates for single keys
      limit = 1
   }
   if limit <= 0 {
      limit = math.MaxInt64
   }
   if limit > 1 && !bytes.Equal(bucketName, safeRangeBucket) {
      panic("do not use unsafeRange on non-keys bucket")
   }
  //内存中查找
   keys, vals := rt.buf.Range(bucketName, key, endKey, limit)
   if int64(len(keys)) == limit {
      return keys, vals
   }
   rt.txmu.Lock()
   //boltdb查找
   k2, v2, _ := unsafeRange(rt.tx, bucketName, key, endKey, limit-int64(len(keys)))
   rt.txmu.Unlock()
   return append(k2, keys...), append(v2, vals...)
}
```

```go
func unsafeRange(tx *bolt.Tx, bucketName, key, endKey []byte, limit int64) (keys [][]byte, vs [][]byte, err error) {
  //boltdb查找数据
   bucket := tx.Bucket(bucketName)
   if bucket == nil {
      return nil, nil, fmt.Errorf("bucket %s does not exist", bucketName)
   }
   if len(endKey) == 0 {
      if v := bucket.Get(key); v != nil {
         return append(keys, key), append(vs, v), nil
      }
      return nil, nil, nil
   }
   if limit <= 0 {
      limit = math.MaxInt64
   }
   c := bucket.Cursor()
   for ck, cv := c.Seek(key); ck != nil && bytes.Compare(ck, endKey) < 0; ck, cv = c.Next() {
      vs = append(vs, cv)
      keys = append(keys, ck)
      if limit == int64(len(keys)) {
         break
      }
   }
   return keys, vs, nil
}
```

## 写优化

**事务锁由粗粒度的互斥锁，优化成读写锁，同时引入了 buffer 来提升吞吐量。**

**读事务会加读锁，写事务结束时要加写锁更新 buffer。expensive read 请求会长时间持有锁，会导致写请求超时**

mvcc/kv_view.go

```go
func (wv *writeView) Put(key, value []byte, lease lease.LeaseID) (rev int64) {
   tw := wv.kv.Write()
   defer tw.End()
   return tw.Put(key, value, lease)
}
```

### 事务开始

```go
func (s *store) Write() TxnWrite {
   s.mu.RLock() //store的读锁
   tx := s.b.BatchTx() //batchTxBuffered
   tx.Lock()//batchTx独占锁
   tw := &storeTxnWrite{
      storeTxnRead: &storeTxnRead{s, tx, 0, 0},
      tx:           tx,
      beginRev:     s.currentRev, //当前最新版本
      changes:      make([]mvccpb.KeyValue, 0, 4), //将写入的数据存放到changes
   }
   return newMetricsTxnWrite(tw)
}
```

```go
func (t *batchTx) Lock() { //batchTx独占锁
   t.Mutex.Lock()
}
```

### 处理写操作

```go
func (tw *metricsTxnWrite) Put(key, value []byte, lease lease.LeaseID) (rev int64) {
   tw.puts++
   return tw.TxnWrite.Put(key, value, lease)
}
```

```go
func (tw *storeTxnWrite) Put(key, value []byte, lease lease.LeaseID) int64 {
   tw.put(key, value, lease)
   return int64(tw.beginRev + 1)
}
```

```go
func (tw *storeTxnWrite) put(key, value []byte, leaseID lease.LeaseID) {
   rev := tw.beginRev + 1 //此key的创建版本或者修改版本
   c := rev
   oldLease := lease.NoLease

   _, created, ver, err := tw.s.kvindex.Get(key, rev)
   if err == nil { //之前就已经存在此key
      c = created.main //修改创建版本
      oldLease = tw.s.le.GetLease(lease.LeaseItem{Key: string(key)}) //获取之前的leaseId
   }

   ibytes := newRevBytes()
   idxRev := revision{main: rev, sub: int64(len(tw.changes))}
   revToBytes(idxRev, ibytes)

   ver = ver + 1
   kv := mvccpb.KeyValue{
      Key:            key,
      Value:          value,
      CreateRevision: c, //创建版本
      ModRevision:    rev, //修改版本
      Version:        ver, //此版本每次修改递增
      Lease:          int64(leaseID), //设置新的leaseId
   }

   d, err := kv.Marshal() //序列化KeyValue 
   if err != nil {
      plog.Fatalf("cannot marshal event: %v", err)
   }
	//1、写入boltdb 2、写入内存缓冲区
   tw.tx.UnsafeSeqPut(keyBucketName, ibytes, d) 
   tw.s.kvindex.Put(key, idxRev) //写key对应的历史版本
   tw.changes = append(tw.changes, kv)  //事务内变更的keyvalue

   if oldLease != lease.NoLease { //旧的和新的leaseId不相同
      if tw.s.le == nil {
         panic("no lessor to detach lease")
      }
      err = tw.s.le.Detach(oldLease, []lease.LeaseItem{{Key: string(key)}}) //取消关联
      if err != nil {
         plog.Errorf("unexpected error from lease detach: %v", err)
      }
   }
   if leaseID != lease.NoLease {
      if tw.s.le == nil {
         panic("no lessor to attach lease")
      }
     //将新的leaseId和key进行绑定
      err = tw.s.le.Attach(leaseID, []lease.LeaseItem{{Key: string(key)}})
      if err != nil {
         panic("unexpected error from lease Attach")
      }
   }
}
```

```go
func (t *batchTxBuffered) UnsafeSeqPut(bucketName []byte, key []byte, value []byte) {
   t.batchTx.UnsafeSeqPut(bucketName, key, value) //写入boltdb
  t.buf.putSeq(bucketName, key, value) //写入map，bucketName->[](key,value)
}
```

```go
func (txw *txWriteBuffer) putSeq(bucket, k, v []byte) {
   b, ok := txw.buckets[string(bucket)] //根据bucketName获取bucketBuffer
   if !ok {
      b = newBucketBuffer()
      txw.buckets[string(bucket)] = b
   }
   b.add(k, v)

```

```go
func (bb *bucketBuffer) add(k, v []byte) {
   bb.buf[bb.used].key, bb.buf[bb.used].val = k, v
   bb.used++ //数组小标
   if bb.used == len(bb.buf) {
      buf := make([]kv, (3*len(bb.buf))/2)
      copy(buf, bb.buf)
      bb.buf = buf
   }
}
```

### 事务结束

```go
func (tw *metricsTxnWrite) End() {
   defer tw.TxnWrite.End()
   if sum := tw.ranges + tw.puts + tw.deletes; sum != 1 {
      if sum > 1 {
         txnCounter.Inc()
      }
      return
   }
   switch {
   case tw.ranges == 1:
      rangeCounter.Inc()
   case tw.puts == 1:
      putCounter.Inc()
   case tw.deletes == 1:
      deleteCounter.Inc()
   }
}
```

etcd/mvcc/kvstore_txn.go

```go
func (tw *storeTxnWrite) End() {
   if len(tw.changes) != 0 { //有更新操作
      tw.s.saveIndex(tw.tx) //保证幂等
      tw.s.revMu.Lock() //上独占锁，保护最新版本
      tw.s.currentRev++
   }
   tw.tx.Unlock() //调用batchTxBuffered的Unlock
   if len(tw.changes) != 0 {
      tw.s.revMu.Unlock() //释放保护版本的独占锁
   }
   tw.s.mu.RUnlock() //释放store的读锁
}
```

#### batchTxBuffered释放锁

etcd/mvcc/backend/batch_tx.go

```go
func (t *batchTxBuffered) Unlock() {
   if t.pending != 0 { //执行了变更操作
      t.backend.readTx.mu.Lock()//获取readTx的写锁，此读锁在读数据时也会上锁
      t.buf.writeback(&t.backend.readTx.buf) //将操作的数据写入读缓冲区
      t.backend.readTx.mu.Unlock() //释放readTx的写锁
      if t.pending >= t.backend.batchLimit { //积攒的未提交的事务足够多，batchTxBuffered提交事务
         t.commit(false)
      }
   }
   t.batchTx.Unlock()//释放batchTx的锁
}
```

mvcc/backend/tx_buffer.go

```go
func (txw *txWriteBuffer) writeback(txr *txReadBuffer) {
   for k, wb := range txw.buckets {
      rb, ok := txr.buckets[k]
      if !ok { //txReadBuffer没有此bucket的数据
         delete(txw.buckets, k) //从txWriteBuffer中删除
         txr.buckets[k] = wb //将此bucket的数据加入到txReadBuffer中
         continue
      }
      if !txw.seq && wb.used > 1 { //bucket中数据多于1条时进行排序
         sort.Sort(wb)
      }
     //txReadBuffer和txWriteBuffer中的数据进行合并
      rb.merge(wb)
   }
   txw.reset()
}
```

#### batchTx释放锁

etcd/mvcc/backend/batch_tx.go

```go
func (t *batchTx) Unlock() {
   if t.pending >= t.backend.batchLimit {//积攒的未提交的事务足够多，提交事务
      t.commit(false)//boltdb提交事务
   }
   t.Mutex.Unlock()//释放batchTx的独占锁
}
```



# Etcd-3.4

etcd/etcdserver/api/v3rpc/quota.go:

```go
func (s *quotaKVServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
   if err := s.qa.check(ctx, r); err != nil { //配额检测，占用空间是否超过阈值，默认最大8g
      return nil, err
   }
   return s.KVServer.Put(ctx, r)
}
```

etcd/etcdserver/api/v3rpc/key.go

```go
func (s *kvServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
   if err := checkPutRequest(r); err != nil { //校验请求的合法性
      return nil, err
   }

   resp, err := s.kv.Put(ctx, r)
   if err != nil {
      return nil, togRPCError(err)
   }

   s.hdr.fill(resp.Header)
   return resp, nil
}
```

etcd/etcdserver/v3_server.go

```go
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
   ctx = context.WithValue(ctx, traceutil.StartTimeKey, time.Now())//记录开始执行时间
   resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
   if err != nil {
      return nil, err
   }
   return resp.(*pb.PutResponse), nil
}
```



```go
func (s *EtcdServer) raftRequest(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
   return s.raftRequestOnce(ctx, r)
}
```



```go
func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
   result, err := s.processInternalRaftRequestOnce(ctx, r)
   if err != nil {
      return nil, err
   }
   if result.err != nil {
      return nil, result.err
   }
   if startTime, ok := ctx.Value(traceutil.StartTimeKey).(time.Time); ok && result.trace != nil {
      applyStart := result.trace.GetStartTime()
      // The trace object is created in apply. Here reset the start time to trace
      // the raft request time by the difference between the request start time
      // and apply start time
      result.trace.SetStartTime(startTime)
      result.trace.InsertStep(0, applyStart, "process raft request")
      result.trace.LogIfLong(traceThreshold)
   }
   return result.resp, nil
}
```

```go
func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error) {
   ai := s.getAppliedIndex()
   ci := s.getCommittedIndex()
  //限速，已提交的日志索引（committed index）比已应用到状态机的日志索引（applied index）超过了5000，返回"etcdserver: too many requests"错误给
   if ci > ai+maxGapBetweenApplyAndCommitIndex { 
      return nil, ErrTooManyRequests
   }

   r.Header = &pb.RequestHeader{
      ID: s.reqIDGen.Next(), //请求的唯一标志
   }

   authInfo, err := s.AuthInfoFromCtx(ctx) //鉴权
   if err != nil {
      return nil, err
   }
   if authInfo != nil {
      r.Header.Username = authInfo.Username
      r.Header.AuthRevision = authInfo.Revision
   }

   data, err := r.Marshal()
   if err != nil {
      return nil, err
   }

   if len(data) > int(s.Cfg.MaxRequestBytes) {
      return nil, ErrRequestTooLarge
   }

   id := r.ID
   if id == 0 {
      id = r.Header.ID
   }
   ch := s.w.Register(id) //注册进行等待响应的请求

   cctx, cancel := context.WithTimeout(ctx, s.Cfg.ReqTimeout())
   defer cancel()

   start := time.Now()
   err = s.r.Propose(cctx, data) //走raft协议
   if err != nil {
      proposalsFailed.Inc()
      s.w.Trigger(id, nil) // GC wait
      return nil, err
   }
   proposalsPending.Inc()
   defer proposalsPending.Dec()

   select {
   case x := <-ch: //阻塞，直到获取到响应结果
      return x.(*applyResult), nil
   case <-cctx.Done():
      proposalsFailed.Inc()
      s.w.Trigger(id, nil) // GC wait
      return nil, s.parseProposeCtxErr(cctx.Err(), start)
   case <-s.done:
      return nil, ErrStopped
   }
}
```

## 读优化

**3.2版本，在读事务开启后一直持有readTx的读锁，读事务结束后才会释放readTx读锁，容易出现长时间占有锁，导致写阻塞**

**3.4版本创建读事务时全量拷贝buffer中的数据，拷贝完成就释放了readTx的读锁，持有锁的时间短，不会阻塞写事务**

mvcc/kv_view.go

```go
func (rv *readView) Range(key, end []byte, ro RangeOptions) (r *RangeResult, err error) {
   tr := rv.kv.Read(traceutil.TODO())
   defer tr.End()
   return tr.Range(key, end, ro)
}
```

mvcc/kvstore_txn.go

```go
func (s *store) Read(trace *traceutil.Trace) TxnRead {
   s.mu.RLock() //store的读锁
   s.revMu.RLock() //用于读取合并版本、当前最新版本
   // backend holds b.readTx.RLock() only when creating the concurrentReadTx. After
   // ConcurrentReadTx is created, it will not block write transaction.
   tx := s.b.ConcurrentReadTx()
   tx.RLock() // RLock is no-op. concurrentReadTx does not need to be locked after it is created.
   firstRev, rev := s.compactMainRev, s.currentRev
   s.revMu.RUnlock()
   return newMetricsTxnRead(&storeTxnRead{s, tx, firstRev, rev, trace})
}
```

mvcc/backend/backend.go

```go
func (b *backend) ConcurrentReadTx() ReadTx {
   b.readTx.RLock() //获取readTx读锁，
   defer b.readTx.RUnlock() //释放readTx读锁
   // prevent boltdb read Tx from been rolled back until store read Tx is done. Needs to be called when holding readTx.RLock().
   b.readTx.txWg.Add(1)
   // TODO: might want to copy the read buffer lazily - create copy when A) end of a write transaction B) end of a batch interval.
   return &concurrentReadTx{
      buf:     b.readTx.buf.unsafeCopy(),
      tx:      b.readTx.tx,
      txMu:    &b.readTx.txMu,
      buckets: b.readTx.buckets,
      txWg:    b.readTx.txWg,
   }
}
```

## watch

如果watcher中的chan满了，etcd 为了保证 Watch 事件不丢失，会将此 watcher 从 synced watcherGroup 中删除，然后将此 watcher 和事件列表移到watchableStore的 victims数组中

mvcc/watchable_store.go

```go
func (s *watchableStore) syncVictimsLoop() {
   defer s.wg.Done()

   for {
      for s.moveVictims() != 0 {
         // try to update all victim watchers
      }
      s.mu.RLock()
      isEmpty := len(s.victims) == 0
      s.mu.RUnlock()

      var tickc <-chan time.Time
      if !isEmpty {
         tickc = time.After(10 * time.Millisecond)
      }

      select {
      case <-tickc:
      case <-s.victimc:
      case <-s.stopc:
         return
      }
   }
}
```

```go
func (s *watchableStore) moveVictims() (moved int) {
   s.mu.Lock()
   victims := s.victims
   s.victims = nil
   s.mu.Unlock()

   var newVictim watcherBatch
   for _, wb := range victims {
      for w, eb := range wb {
         rev := w.minRev - 1
        //尝试将堆积的事件再次推送到watcher的chan中
         if w.send(WatchResponse{WatchID: w.id, Events: eb.evs, Revision: rev}) {
            pendingEventsGauge.Add(float64(len(eb.evs)))
         } else {//推送失败，则再次加入到victim 数组中，等待下次重试
            if newVictim == nil {
               newVictim = make(watcherBatch)
            }
            newVictim[w] = eb
            continue
         }
         moved++
      }

      // assign completed victim watchers to unsync/sync
      s.mu.Lock()
      s.store.revMu.RLock()
      curRev := s.store.currentRev
      for w, eb := range wb {
         if newVictim != nil && newVictim[w] != nil {
            // couldn't send watch response; stays victim
            continue
         }
         w.victim = false
         if eb.moreRev != 0 {
            w.minRev = eb.moreRev
         }
         if w.minRev <= curRev {
            s.unsynced.add(w)
         } else {
            slowWatcherGauge.Dec()
            s.synced.add(w)
         }
      }
      s.store.revMu.RUnlock()
      s.mu.Unlock()
   }

   if len(newVictim) > 0 {
      s.mu.Lock()
      s.victims = append(s.victims, newVictim)
      s.mu.Unlock()
   }

   return moved
}
```

## 压缩请求

压缩的本质是回收历史版本，不会删除最新版本数据

mvcc/kvstore.go

```go
func (s *store) Compact(trace *traceutil.Trace, rev int64) (<-chan struct{}, error) {
   s.mu.Lock()
   //判断压缩版本号是否合法，并将其写入boltdb
   ch, err := s.updateCompactRev(rev) 
   trace.Step("check and update compact revision")
   if err != nil {
      s.mu.Unlock()
      return ch, err
   }
   s.mu.Unlock()
	//执行压缩
   return s.compact(trace, rev)
}
```

```go
func (s *store) updateCompactRev(rev int64) (<-chan struct{}, error) {
   s.revMu.Lock()
   if rev <= s.compactMainRev {//请求的版本号小于等于当前etcd记录的压缩版本号
      ch := make(chan struct{})
      f := func(ctx context.Context) { s.compactBarrier(ctx, ch) }
      s.fifoSched.Schedule(f)
      s.revMu.Unlock()
      return ch, ErrCompacted
   }
   if rev > s.currentRev { //大于当前最新的版本号
      s.revMu.Unlock()
      return nil, ErrFutureRev
   }
	//修改压缩版本号
   s.compactMainRev = rev

   rbytes := newRevBytes()
   revToBytes(revision{main: rev}, rbytes)
	//将压缩版本号写入boltdb
  //如果不保存版本号， 在执行合并任务过程发生了宕机，重启后，各个节点数据就会不一致
   tx := s.b.BatchTx()
   tx.Lock()
   tx.UnsafePut(metaBucketName, scheduledCompactKeyName, rbytes)
   tx.Unlock()
   // ensure that desired compaction is persisted
   s.b.ForceCommit()

   s.revMu.Unlock()

   return nil, nil
}
```

```go
func (s *store) compact(trace *traceutil.Trace, rev int64) (<-chan struct{}, error) {
   start := time.Now()
   keep := s.kvindex.Compact(rev)
   trace.Step("compact in-memory index tree")
   ch := make(chan struct{})
   var j = func(ctx context.Context) {
      if ctx.Err() != nil {
         s.compactBarrier(ctx, ch)
         return
      }
      if !s.scheduleCompaction(rev, keep) {
         s.compactBarrier(nil, ch)
         return
      }
      close(ch)
   }

   s.fifoSched.Schedule(j)

   indexCompactionPauseMs.Observe(float64(time.Since(start) / time.Millisecond))
   trace.Step("schedule compaction")
   return ch, nil
}
```

# GrpcProxy

任何问题都可以通过加一个中间层解决

支持海量的客户端连接，方便扩展读、扩展watch、扩展Lease，对watch、Lease进行合并降低服务端的watch数、Lease数，降低服务端的负载

## 服务端启动

```go
func startGRPCProxy(cmd *cobra.Command, args []string) {
   checkArgs()

   tlsinfo := newTLS(grpcProxyListenCA, grpcProxyListenCert, grpcProxyListenKey)
   if tlsinfo != nil {
      plog.Infof("ServerTLS: %s", tlsinfo)
   }
   m := mustListenCMux(tlsinfo)

   grpcl := m.Match(cmux.HTTP2())
   defer func() {
      grpcl.Close()
      plog.Infof("stopping listening for grpc-proxy client requests on %s", grpcProxyListenAddr)
   }()
	//创建client实例，负责与etcd集群通信
   client := mustNewClient()

   srvhttp, httpl := mustHTTPListener(m, tlsinfo)
   errc := make(chan error)
  //创建grpcserver并启动
   go func() { errc <- newGRPCProxyServer(client).Serve(grpcl) }()
   go func() { errc <- srvhttp.Serve(httpl) }()
   go func() { errc <- m.Serve() }()
   if len(grpcProxyMetricsListenAddr) > 0 {
      mhttpl := mustMetricsListener(tlsinfo)
      go func() {
         mux := http.NewServeMux()
         mux.Handle("/metrics", prometheus.Handler())
         plog.Fatal(http.Serve(mhttpl, mux))
      }()
   }

   // grpc-proxy is initialized, ready to serve
   notifySystemd()

   fmt.Fprintln(os.Stderr, <-errc)
   os.Exit(1)
}
```

### 创建client实例

```go
func mustNewClient() *clientv3.Client {
  //发现集群信息
   srvs := discoverEndpoints(grpcProxyDNSCluster, grpcProxyCA, grpcProxyInsecureDiscovery)
   eps := srvs.Endpoints
   if len(eps) == 0 {
      eps = grpcProxyEndpoints
   }
   cfg, err := newClientCfg(eps)
   if err != nil {
      fmt.Fprintln(os.Stderr, err)
      os.Exit(1)
   }
   client, err := clientv3.New(*cfg)
   if err != nil {
      fmt.Fprintln(os.Stderr, err)
      os.Exit(1)
   }
   return client
}
```

### 创建ProxyGrpcServer

```go
func newGRPCProxyServer(client *clientv3.Client) *grpc.Server {
   if len(grpcProxyNamespace) > 0 {
      client.KV = namespace.NewKV(client.KV, grpcProxyNamespace)
      client.Watcher = namespace.NewWatcher(client.Watcher, grpcProxyNamespace)
      client.Lease = namespace.NewLease(client.Lease, grpcProxyNamespace)
   }

   kvp, _ := grpcproxy.NewKvProxy(client)
   watchp, _ := grpcproxy.NewWatchProxy(client)
   if grpcProxyResolverPrefix != "" {
      grpcproxy.Register(client, grpcProxyResolverPrefix, grpcProxyAdvertiseClientURL, grpcProxyResolverTTL)
   }
   clusterp, _ := grpcproxy.NewClusterProxy(client, grpcProxyAdvertiseClientURL, grpcProxyResolverPrefix)
   leasep, _ := grpcproxy.NewLeaseProxy(client)
   mainp := grpcproxy.NewMaintenanceProxy(client)
   authp := grpcproxy.NewAuthProxy(client)
   electionp := grpcproxy.NewElectionProxy(client)
   lockp := grpcproxy.NewLockProxy(client)

   server := grpc.NewServer(
      grpc.StreamInterceptor(grpc_prometheus.StreamServerInterceptor),
      grpc.UnaryInterceptor(grpc_prometheus.UnaryServerInterceptor),
      grpc.MaxConcurrentStreams(math.MaxUint32),
   ) //创建grpcServer，添加拦截器

   //注册处理客户端请求的handler
   pb.RegisterKVServer(server, kvp)
   pb.RegisterWatchServer(server, watchp)
   pb.RegisterClusterServer(server, clusterp)
   pb.RegisterLeaseServer(server, leasep)
   pb.RegisterMaintenanceServer(server, mainp)
   pb.RegisterAuthServer(server, authp)
   v3electionpb.RegisterElectionServer(server, electionp)
   v3lockpb.RegisterLockServer(server, lockp)
   return server
}
```

## kvProxy

```go
func NewKvProxy(c *clientv3.Client) (pb.KVServer, <-chan struct{}) {
   kv := &kvProxy{
      kv:    c.KV,//通信
      cache: cache.NewCache(cache.DefaultMaxEntries), //缓存
   }
   donec := make(chan struct{})
   close(donec)
   return kv, donec
}
```

## WatchPrxoxy

```go
func NewWatchProxy(c *clientv3.Client) (pb.WatchServer, <-chan struct{}) {
   cctx, cancel := context.WithCancel(c.Ctx())
   wp := &watchProxy{
      cw:     c.Watcher,
      ctx:    cctx,
      leader: newLeader(c.Ctx(), c.Watcher),
   }
   wp.ranges = newWatchRanges(wp) //客户端watch的key的范围
   ch := make(chan struct{})
   go func() {
      defer close(ch)
      <-wp.leader.stopNotify()
      wp.mu.Lock()
      select {
      case <-wp.ctx.Done():
      case <-wp.leader.disconnectNotify():
         cancel()
      }
      <-wp.ctx.Done()
      wp.mu.Unlock()
      wp.wg.Wait()
      wp.ranges.stop()
   }()
   return wp, ch
}
```

### 创建watchProxyStream

```go
func (wp *watchProxy) Watch(stream pb.Watch_WatchServer) (err error) {
   wp.mu.Lock()
   select {
   case <-wp.ctx.Done():
      wp.mu.Unlock()
      select {
      case <-wp.leader.disconnectNotify():
         return grpc.ErrClientConnClosing
      default:
         return wp.ctx.Err()
      }
   default:
      wp.wg.Add(1)
   }
   wp.mu.Unlock()

   ctx, cancel := context.WithCancel(stream.Context())
   wps := &watchProxyStream{ //创建watchProxyStream实例
      ranges:   wp.ranges,
      watchers: make(map[int64]*watcher),
      stream:   stream,
      watchCh:  make(chan *pb.WatchResponse, 1024), //缓存watch事件
      ctx:      ctx,
      cancel:   cancel,
   }

   var lostLeaderC <-chan struct{}
   if md, ok := metadata.FromOutgoingContext(stream.Context()); ok {
      v := md[rpctypes.MetadataRequireLeaderKey]
      if len(v) > 0 && v[0] == rpctypes.MetadataHasLeader {
         lostLeaderC = wp.leader.lostNotify()
         // if leader is known to be lost at creation time, avoid
         // letting events through at all
         select {
         case <-lostLeaderC:
            wp.wg.Done()
            return rpctypes.ErrNoLeader
         default:
         }
      }
   }

   // post to stopc => terminate server stream; can't use a waitgroup
   // since all goroutines will only terminate after Watch() exits.
   stopc := make(chan struct{}, 3)
   go func() {
      defer func() { stopc <- struct{}{} }()
      wps.recvLoop() //接收客户端请求
   }()
   go func() {
      defer func() { stopc <- struct{}{} }()
      wps.sendLoop() //给客户端发送响应信息
   }()
   // tear down watch if leader goes down or entire watch proxy is terminated
   go func() {
      defer func() { stopc <- struct{}{} }()
      select {
      case <-lostLeaderC:
      case <-ctx.Done():
      case <-wp.ctx.Done():
      }
   }()

   <-stopc
   cancel()

   // recv/send may only shutdown after function exits;
   // goroutine notifies proxy that stream is through
   go func() {
      <-stopc
      <-stopc
      wps.close()
      wp.wg.Done()
   }()

   select {
   case <-lostLeaderC:
      return rpctypes.ErrNoLeader
   case <-wp.leader.disconnectNotify():
      return grpc.ErrClientConnClosing
   default:
      return wps.ctx.Err()
   }
}
```

### 接收客户端请求

```go
func (wps *watchProxyStream) recvLoop() error {
   for {
      req, err := wps.stream.Recv()
      if err != nil {
         return err
      }
      switch uv := req.RequestUnion.(type) {
      case *pb.WatchRequest_CreateRequest: //创建watch请求
         cr := uv.CreateRequest
         w := &watcher{ //创建watcher实例
            wr:  watchRange{string(cr.Key), string(cr.RangeEnd)}, //watch的范围
            id:  wps.nextWatcherID,
            wps: wps,

            nextrev:  cr.StartRevision,
            progress: cr.ProgressNotify,
            prevKV:   cr.PrevKv,
            filters:  v3rpc.FiltersFromRequest(cr),
         }
         if !w.wr.valid() { //验证范围是否有效
           //范围无效，发送响应
            w.post(&pb.WatchResponse{WatchId: -1, Created: true, Canceled: true})
            continue
         }
         wps.nextWatcherID++ //watcher唯一标志
         w.nextrev = cr.StartRevision 
         wps.watchers[w.id] = w // watcherId -> watcher
         wps.ranges.add(w) //维护客户端的watcher
      case *pb.WatchRequest_CancelRequest: //删除watch请求
         wps.delete(uv.CancelRequest.WatchId)
      default:
         panic("not implemented")
      }
   }
}
```

#### 创建watch

```go
func (wrs *watchRanges) add(w *watcher) {
  //每个watchRange对应一个watchBroadcasts，watchBroadcasts维护了所有的watcher
   wrs.mu.Lock()
   defer wrs.mu.Unlock()
	//根据watch的范围获取watchBroadcasts
   if wbs := wrs.bcasts[w.wr]; wbs != nil {
      wbs.add(w) //将watcher加入到watchBroadcasts
      return
   }
  //创建新的watchBroadcasts
   wbs := newWatchBroadcasts(wrs.wp)
   wrs.bcasts[w.wr] = wbs
   wbs.add(w)//将watcher加入到watchBroadcasts
}
```

##### 添加watcher

```go
func (wbs *watchBroadcasts) add(w *watcher) {
   wbs.mu.Lock()
   defer wbs.mu.Unlock()
   // find fitting bcast
   for wb := range wbs.bcasts {
      if wb.add(w) {
         wbs.watchers[w] = wb
         return
      }
   }
   // no fit; create a bcast
   wb := newWatchBroadcast(wbs.wp, w, wbs.update)
   wbs.watchers[w] = wb
   wbs.bcasts[wb] = struct{}{}
}
```



```go
func (wbs *watchBroadcasts) add(w *watcher) {
   wbs.mu.Lock()
   defer wbs.mu.Unlock()
   // find fitting bcast
   for wb := range wbs.bcasts {
      if wb.add(w) { //将watcher加入到对应的broadcast，保证此watcher加入到此broacast不会丢失事件，否则为watcher创建单独的broadcast
         wbs.watchers[w] = wb
         return
      }
   }
   // 创建broadcast
   wb := newWatchBroadcast(wbs.wp, w, wbs.update)
   wbs.watchers[w] = wb
   wbs.bcasts[wb] = struct{}{}
}
```

##### 创建watchBroadcast

```go
func newWatchBroadcast(wp *watchProxy, w *watcher, update func(*watchBroadcast)) *watchBroadcast 
 
   cctx, cancel := context.WithCancel(wp.ctx)
//创建watchBroadcast实例，维护watcher，发送watch请求，发布watch响应
   wb := &watchBroadcast{
      cancel:    cancel,
      nextrev:   w.nextrev,
      receivers: make(map[*watcher]struct{}),
      donec:     make(chan struct{}),
   }
   wb.add(w) //注册watcher
   go func() {
      defer close(wb.donec)

      opts := []clientv3.OpOption{
         clientv3.WithRange(w.wr.end),
         clientv3.WithProgressNotify(),
         clientv3.WithRev(wb.nextrev),
         clientv3.WithPrevKV(),
         clientv3.WithCreatedNotify(),
      }

      wch := wp.cw.Watch(cctx, w.wr.key, opts...) //向etcd集群发起watch请求

      for wr := range wch {
         wb.bcast(wr) //给客户端发送watch响应
         update(wb)
      }
   }()
   return wb
}
```



```go
func (wb *watchBroadcast) bcast(wr clientv3.WatchResponse) {
   wb.mu.Lock()
   defer wb.mu.Unlock()
   // watchers start on the given revision, if any; ignore header rev on create
   if wb.responses > 0 || wb.nextrev == 0 {
      wb.nextrev = wr.Header.Revision + 1
   }
   wb.responses++
   for r := range wb.receivers {
      r.send(wr) //发送watchresponse
   }
   if len(wb.receivers) > 0 {
      eventsCoalescing.Add(float64(len(wb.receivers) - 1))
   }
}
```

##### 创建watchBroadcasts

```go
func newWatchBroadcasts(wp *watchProxy) *watchBroadcasts {
  //创建watchBroadcasts实例，负责维护某一个范围的watcher
  //维护了所有的watcher与watchBroadcast的对应关系
  //维护了所有的Broadcase实例
   wbs := &watchBroadcasts{
      wp:       wp,
      bcasts:   make(map[*watchBroadcast]struct{}),
      watchers: make(map[*watcher]*watchBroadcast),
      updatec:  make(chan *watchBroadcast, 1),
      donec:    make(chan struct{}),
   }
   go func() {
      defer close(wbs.donec)
      for wb := range wbs.updatec {
         wbs.coalesce(wb) //合并
      }
   }()
   return wbs
}
```

```go
func (wbs *watchBroadcasts) coalesce(wb *watchBroadcast) { //合并watcher
   if wb.size() >= maxCoalesceReceivers { //默认5
      return
   }
   wbs.mu.Lock()
   for wbswb := range wbs.bcasts { //bcasts为维护的所有Broadcase实例
      if wbswb == wb {
         continue
      }
      wb.mu.Lock()
      wbswb.mu.Lock()
      // 1. check if wbswb is behind wb so it won't skip any events in wb
      // 2. ensure wbswb started; nextrev == 0 may mean wbswb is waiting
      // for a current watcher and expects a create event from the server.
      if wb.nextrev >= wbswb.nextrev && wbswb.responses > 0 {
        //wbswb落后与wb，将wb中的watcher移到wbswb中
         for w := range wb.receivers {
            wbswb.receivers[w] = struct{}{}
            wbs.watchers[w] = wbswb
         }
         wb.receivers = nil
      }
      wbswb.mu.Unlock()
      wb.mu.Unlock()
      if wb.empty() {
         delete(wbs.bcasts, wb)
         wb.stop()
         break
      }
   }
   wbs.mu.Unlock()
}
```

#### 删除watch

```go
func (wps *watchProxyStream) delete(id int64) {
   wps.mu.Lock()
   defer wps.mu.Unlock()

   w, ok := wps.watchers[id] //获取watcher
   if !ok {
      return
   }
   wps.ranges.delete(w) //从ranges删除watcher
   delete(wps.watchers, id) //从map删除watcher
  //删除watcher的响应信息
   resp := &pb.WatchResponse{
      Header:   &w.lastHeader,
      WatchId:  id,
      Canceled: true,
   }
   wps.watchCh <- resp //将响应放入通道
}
```

```go
func (wrs *watchRanges) delete(w *watcher) {
   wrs.mu.Lock()
   defer wrs.mu.Unlock()
   wbs, ok := wrs.bcasts[w.wr] //根据范围获取watchBroadcasts
   if !ok {
      panic("deleting missing range")
   }
   if wbs.delete(w) == 0 { //此watchBroadcasts无watcher
      wbs.stop()
      delete(wrs.bcasts, w.wr)
   }
}
```

```go
func (wbs *watchBroadcasts) delete(w *watcher) int {
   wbs.mu.Lock()
   defer wbs.mu.Unlock()

   wb, ok := wbs.watchers[w] //根据watcher获取watchbroadcast
   if !ok {
      panic("deleting missing watcher from broadcasts")
   }
   delete(wbs.watchers, w) //从map删除watcher
   wb.delete(w) //从watchbroadcast删除watcher
   if wb.empty() { //wb中已无watcher
      delete(wbs.bcasts, wb) //从broadcasts删除watchbroadcast
      wb.stop() //关闭watchbroadcast
   }
   return len(wbs.bcasts)
}
```

```go
func (wb *watchBroadcast) delete(w *watcher) { //删除watcher
   wb.mu.Lock()
   defer wb.mu.Unlock()
   if _, ok := wb.receivers[w]; !ok {
      panic("deleting missing watcher from broadcast")
   }
   delete(wb.receivers, w) //删除
   if len(wb.receivers) > 0 {
      // do not dec the only left watcher for coalescing.
      watchersCoalescing.Dec()
   }
}
```

```go
func (wb *watchBroadcast) stop() { //关闭watchbroadcast
   if !wb.empty() {
      watchersCoalescing.Sub(float64(wb.size() - 1))
   }

   wb.cancel()
   <-wb.donec
}
```

### 发送响应

```go
func (wps *watchProxyStream) sendLoop() {
   for {
      select {
      case wresp, ok := <-wps.watchCh:
         if !ok {
            return
         }
         if err := wps.stream.Send(wresp); err != nil { //发送watchResponse
            return
         }
      case <-wps.ctx.Done():
         return
      }
   }
}
```

## LeaseProxy

```go
func NewLeaseProxy(c *clientv3.Client) (pb.LeaseServer, <-chan struct{}) {
   cctx, cancel := context.WithCancel(c.Ctx())
   lp := &leaseProxy{
      leaseClient: pb.NewLeaseClient(c.ActiveConnection()),
      lessor:      c.Lease,
      ctx:         cctx,
      leader:      newLeader(c.Ctx(), c.Watcher),
   }
   ch := make(chan struct{})
   go func() {
      defer close(ch)
      <-lp.leader.stopNotify()
      lp.mu.Lock()
      select {
      case <-lp.ctx.Done():
      case <-lp.leader.disconnectNotify():
         cancel()
      }
      <-lp.ctx.Done()
      lp.mu.Unlock()
      lp.wg.Wait()
   }()
   return lp, ch
}
```

#### 续约

```go
func (lp *leaseProxy) LeaseKeepAlive(stream pb.Lease_LeaseKeepAliveServer) error {
   lp.mu.Lock()
   select {
   case <-lp.ctx.Done():
      lp.mu.Unlock()
      return lp.ctx.Err()
   default:
      lp.wg.Add(1)
   }
   lp.mu.Unlock()

   ctx, cancel := context.WithCancel(stream.Context())
  //创建leaseProxyStream实例
   lps := leaseProxyStream{
      stream:          stream,
      lessor:          lp.lessor,
      keepAliveLeases: make(map[int64]*atomicCounter),
      respc:           make(chan *pb.LeaseKeepAliveResponse),
      ctx:             ctx,
      cancel:          cancel,
   }

   errc := make(chan error, 2)

   var lostLeaderC <-chan struct{}
   if md, ok := metadata.FromOutgoingContext(stream.Context()); ok {
      v := md[rpctypes.MetadataRequireLeaderKey]
      if len(v) > 0 && v[0] == rpctypes.MetadataHasLeader {
         lostLeaderC = lp.leader.lostNotify()
         // if leader is known to be lost at creation time, avoid
         // letting events through at all
         select {
         case <-lostLeaderC:
            lp.wg.Done()
            return rpctypes.ErrNoLeader
         default:
         }
      }
   }
   stopc := make(chan struct{}, 3)
   //负责接收客户端请求
   go func() {
      defer func() { stopc <- struct{}{} }()
      if err := lps.recvLoop(); err != nil {
         errc <- err
      }
   }()
		//负责返回响应
   go func() {
      defer func() { stopc <- struct{}{} }()
      if err := lps.sendLoop(); err != nil {
         errc <- err
      }
   }()

   // tears down LeaseKeepAlive stream if leader goes down or entire leaseProxy is terminated.
   go func() {
      defer func() { stopc <- struct{}{} }()
      select {
      case <-lostLeaderC:
      case <-ctx.Done():
      case <-lp.ctx.Done():
      }
   }()

   var err error
   select {
   case <-stopc:
      stopc <- struct{}{}
   case err = <-errc:
   }
   cancel()

   // recv/send may only shutdown after function exits;
   // this goroutine notifies lease proxy that the stream is through
   go func() {
      <-stopc
      <-stopc
      <-stopc
      lps.close()
      close(errc)
      lp.wg.Done()
   }()

   select {
   case <-lostLeaderC:
      return rpctypes.ErrNoLeader
   case <-lp.leader.disconnectNotify():
      return grpc.ErrClientConnClosing
   default:
      if err != nil {
         return err
      }
      return ctx.Err()
   }
}
```

#####  recvLoop

```go
func (lps *leaseProxyStream) recvLoop() error {
   for {
      rr, err := lps.stream.Recv()
      if err == io.EOF {
         return nil
      }
      if err != nil {
         return err
      }
      lps.mu.Lock()
      //leaseId -> atomicCounter
      neededResps, ok := lps.keepAliveLeases[rr.ID]
      if !ok {
         neededResps = &atomicCounter{} //创建atomicCounter
         lps.keepAliveLeases[rr.ID] = neededResps //维护映射关系
         lps.wg.Add(1)
         go func() { //续约
            defer lps.wg.Done()
            if err := lps.keepAliveLoop(rr.ID, neededResps); err != nil {
               lps.cancel()
            }
         }()
      }
      neededResps.add(1) //需要响应计数+1
      lps.mu.Unlock()
   }
}
```

```go
func (lps *leaseProxyStream) keepAliveLoop(leaseID int64, neededResps *atomicCounter) error {
   cctx, ccancel := context.WithCancel(lps.ctx)
   defer ccancel()
   //续约
   respc, err := lps.lessor.KeepAlive(cctx, clientv3.LeaseID(leaseID)) 
   if err != nil {
      return err
   }
   // ticker expires when loop hasn't received keepalive within TTL
   var ticker <-chan time.Time
   for {
      select {
      case <-ticker: //超时
         lps.mu.Lock()
         // if there are outstanding keepAlive reqs at the moment of ticker firing,
         // don't close keepAliveLoop(), let it continuing to process the KeepAlive reqs.
         if neededResps.get() > 0 { //仍然有需要等待响应的请求
            lps.mu.Unlock()
            ticker = nil
            continue
         }
        //无等待响应的请求，删除leaseId，关闭keepAliveLoop
         delete(lps.keepAliveLeases, leaseID)
         lps.mu.Unlock()
         return nil
      case rp, ok := <-respc: //续约响应
         if !ok { //续约失败
            lps.mu.Lock()
            delete(lps.keepAliveLeases, leaseID) //删除leaseId
            lps.mu.Unlock()
            if neededResps.get() == 0 {
               return nil //无需要响应的请求
            }
           //获取剩余时间
            ttlResp, err := lps.lessor.TimeToLive(cctx, clientv3.LeaseID(leaseID))
            if err != nil {
               return err
            }
           //创建响应实例
            r := &pb.LeaseKeepAliveResponse{
               Header: ttlResp.ResponseHeader,
               ID:     int64(ttlResp.ID),
               TTL:    ttlResp.TTL,
            }
            for neededResps.get() > 0 {
               select {
               case lps.respc <- r: //发布响应
                  neededResps.add(-1) //等待响应的请求数 -1
               case <-lps.ctx.Done():
                  return nil
               }
            }
            return nil
         }
        //续约成功
         if neededResps.get() == 0 {
            continue //无等待响应的请求
         }
        //设置超时
         ticker = time.After(time.Duration(rp.TTL) * time.Second)
        //返回响应
         r := &pb.LeaseKeepAliveResponse{
            Header: rp.ResponseHeader,
            ID:     int64(rp.ID),
            TTL:    rp.TTL,
         }
         lps.replyToClient(r, neededResps)
      }
   }
}
```

```go
func (lps *leaseProxyStream) replyToClient(r *pb.LeaseKeepAliveResponse, neededResps *atomicCounter) { //返回响应
   timer := time.After(500 * time.Millisecond)
   for neededResps.get() > 0 {
      select {
      case lps.respc <- r: //将响应放入通道，sendLoop方法会将其返回给客户端
         neededResps.add(-1)
      case <-timer:
         return
      case <-lps.ctx.Done():
         return
      }
   }
}
```

##### sendLoop

```go
func (lps *leaseProxyStream) sendLoop() error { //返回响应
   for {
      select {
      case lrp, ok := <-lps.respc:
         if !ok {
            return nil
         }
         if err := lps.stream.Send(lrp); err != nil {
            return err
         }
      case <-lps.ctx.Done():
         return lps.ctx.Err()
      }
   }
}
```

# clientV3

```go
func newClient(cfg *Config) (*Client, error) {
   if cfg == nil {
      cfg = &Config{}
   }
   var creds *credentials.TransportCredentials
   if cfg.TLS != nil {
      c := credentials.NewTLS(cfg.TLS)
      creds = &c
   }

   // use a temporary skeleton client to bootstrap first connection
   baseCtx := context.TODO()
   if cfg.Context != nil {
      baseCtx = cfg.Context
   }

   ctx, cancel := context.WithCancel(baseCtx)
   client := &Client{ //创建client实例
      conn:     nil,
      dialerrc: make(chan error, 1),
      cfg:      *cfg,
      creds:    creds,
      ctx:      ctx,
      cancel:   cancel,
      mu:       new(sync.Mutex),
      callOpts: defaultCallOpts,
   }
   if cfg.Username != "" && cfg.Password != "" { //用户名、密码
      client.Username = cfg.Username
      client.Password = cfg.Password
   }
   if cfg.MaxCallSendMsgSize > 0 || cfg.MaxCallRecvMsgSize > 0 {
      if cfg.MaxCallRecvMsgSize > 0 && cfg.MaxCallSendMsgSize > cfg.MaxCallRecvMsgSize {
         return nil, fmt.Errorf("gRPC message recv limit (%d bytes) must be greater than send limit (%d bytes)", cfg.MaxCallRecvMsgSize, cfg.MaxCallSendMsgSize)
      }
     //grpc配置
      callOpts := []grpc.CallOption{
         defaultFailFast,
         defaultMaxCallSendMsgSize,
         defaultMaxCallRecvMsgSize,
      }
      if cfg.MaxCallSendMsgSize > 0 {
         callOpts[1] = grpc.MaxCallSendMsgSize(cfg.MaxCallSendMsgSize)
      }
      if cfg.MaxCallRecvMsgSize > 0 {
         callOpts[2] = grpc.MaxCallRecvMsgSize(cfg.MaxCallRecvMsgSize)
      }
      client.callOpts = callOpts
   }
	//创建healthBalancer实例
   client.balancer = newHealthBalancer(cfg.Endpoints, cfg.DialTimeout, func(ep string) (bool, error) {
      return grpcHealthCheck(client, ep)
   })

   // 建立连接
   conn, err := client.dial(cfg.Endpoints[0], grpc.WithBalancer(client.balancer))
   if err != nil {
      client.cancel()
      client.balancer.Close()
      return nil, err
   }
   client.conn = conn

   if cfg.DialTimeout > 0 { //阻塞等待连接建立完成
      hasConn := false
      waitc := time.After(cfg.DialTimeout)
      select {
      case <-client.balancer.ready():
         hasConn = true
      case <-ctx.Done():
      case <-waitc:
      }
      if !hasConn {
         err := context.DeadlineExceeded
         select {
         case err = <-client.dialerrc:
         default:
         }
         client.cancel()
         client.balancer.Close()
         conn.Close()
         return nil, err
      }
   }
	//创建对应的重试client
   client.Cluster = NewCluster(client)
   client.KV = NewKV(client)
   client.Lease = NewLease(client)
   client.Watcher = NewWatcher(client)
   client.Auth = NewAuth(client)
   client.Maintenance = NewMaintenance(client)

   if cfg.RejectOldCluster {
      if err := client.checkVersion(); err != nil {
         client.Close()
         return nil, err
      }
   }

   go client.autoSync()
   return client, nil
}
```

建立连接

```go
func (c *Client) dial(endpoint string, dopts ...grpc.DialOption) (*grpc.ClientConn, error) {
   opts := c.dialSetupOpts(endpoint, dopts...)
   host := getHost(endpoint)
   if c.Username != "" && c.Password != "" {
      c.tokenCred = &authTokenCredential{
         tokenMu: &sync.RWMutex{},
      }

      ctx := c.ctx
      if c.cfg.DialTimeout > 0 {
         cctx, cancel := context.WithTimeout(ctx, c.cfg.DialTimeout)
         defer cancel()
         ctx = cctx
      }

      err := c.getToken(ctx)
      if err != nil {
         if toErr(ctx, err) != rpctypes.ErrAuthNotEnabled {
            if err == ctx.Err() && ctx.Err() != c.ctx.Err() {
               err = context.DeadlineExceeded
            }
            return nil, err
         }
      } else {
         opts = append(opts, grpc.WithPerRPCCredentials(c.tokenCred))
      }
   }

   opts = append(opts, c.cfg.DialOptions...)

   conn, err := grpc.DialContext(c.ctx, host, opts...)
   if err != nil {
      return nil, err
   }
   return conn, nil
}
```

同步集群节点信息

```go
func (c *Client) autoSync() { 
   if c.cfg.AutoSyncInterval == time.Duration(0) {
      return
   }

   for {
      select {
      case <-c.ctx.Done():
         return
      case <-time.After(c.cfg.AutoSyncInterval):
         ctx, cancel := context.WithTimeout(c.ctx, 5*time.Second)
         err := c.Sync(ctx)
         cancel()
         if err != nil && err != c.ctx.Err() {
            logger.Println("Auto sync endpoints failed:", err)
         }
      }
   }
}
```



```go
func (c *Client) Sync(ctx context.Context) error {
   mresp, err := c.MemberList(ctx)
   if err != nil {
      return err
   }
   var eps []string
   for _, m := range mresp.Members {
      eps = append(eps, m.ClientURLs...)
   }
   c.SetEndpoints(eps...)
   return nil
}
```

创建KV

```go
func NewKV(c *Client) KV {
   api := &kv{remote: RetryKVClient(c)}
   if c != nil {
      api.callOpts = c.callOpts
   }
   return api
}
```

```go
func RetryKVClient(c *Client) pb.KVClient {
   repeatableRetry := c.newRetryWrapper(isRepeatableStopError)
   nonRepeatableRetry := c.newRetryWrapper(isNonRepeatableStopError)
   conn := pb.NewKVClient(c.conn)
   retryBasic := &retryKVClient{&nonRepeatableKVClient{conn, nonRepeatableRetry}, repeatableRetry}
   retryAuthWrapper := c.newAuthRetryWrapper()
   return &retryKVClient{
      &nonRepeatableKVClient{retryBasic, retryAuthWrapper},
      retryAuthWrapper}
}
```

```go
func (c *Client) newRetryWrapper(isStop retryStopErrFunc) retryRPCFunc {
   return func(rpcCtx context.Context, f rpcFunc) error {
      for {
         if err := readyWait(rpcCtx, c.ctx, c.balancer.ConnectNotify()); err != nil {
            return err
         }
         pinned := c.balancer.pinned()
         err := f(rpcCtx)
         if err == nil {
            return nil
         }
         if logger.V(4) {
            logger.Infof("clientv3/retry: error %q on pinned endpoint %q", err.Error(), pinned)
         }

         if s, ok := status.FromError(err); ok && (s.Code() == codes.Unavailable || s.Code() == codes.DeadlineExceeded || s.Code() == codes.Internal) {
            // mark this before endpoint switch is triggered
            c.balancer.hostPortError(pinned, err)
            c.balancer.next()
            if logger.V(4) {
               logger.Infof("clientv3/retry: switching from %q due to error %q", pinned, err.Error())
            }
         }

         if isStop(err) {
            return err
         }
      }
   }
}
```

常见操作

```go
func (kv *kv) Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error) {
   r, err := kv.Do(ctx, OpPut(key, val, opts...))
   return r.put, toErr(ctx, err)
}

func (kv *kv) Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error) {
   r, err := kv.Do(ctx, OpGet(key, opts...))
   return r.get, toErr(ctx, err)
}

func (kv *kv) Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error) {
   r, err := kv.Do(ctx, OpDelete(key, opts...))
   return r.del, toErr(ctx, err)
}
```

```go
func (kv *kv) Do(ctx context.Context, op Op) (OpResponse, error) {
   var err error
   switch op.t {
   case tRange: //查询
      var resp *pb.RangeResponse 
      resp, err = kv.remote.Range(ctx, op.toRangeRequest(), kv.callOpts...)
      if err == nil {
         return OpResponse{get: (*GetResponse)(resp)}, nil
      }
   case tPut: //写入
      var resp *pb.PutResponse
      r := &pb.PutRequest{Key: op.key, Value: op.val, Lease: int64(op.leaseID), PrevKv: op.prevKV, IgnoreValue: op.ignoreValue, IgnoreLease: op.ignoreLease}
      resp, err = kv.remote.Put(ctx, r, kv.callOpts...)
      if err == nil {
         return OpResponse{put: (*PutResponse)(resp)}, nil
      } 
   case tDeleteRange: //删除
      var resp *pb.DeleteRangeResponse
      r := &pb.DeleteRangeRequest{Key: op.key, RangeEnd: op.end, PrevKv: op.prevKV}
      resp, err = kv.remote.DeleteRange(ctx, r, kv.callOpts...)
      if err == nil {
         return OpResponse{del: (*DeleteResponse)(resp)}, nil
      }
   case tTxn: //事务
      var resp *pb.TxnResponse
      resp, err = kv.remote.Txn(ctx, op.toTxnRequest(), kv.callOpts...)
      if err == nil {
         return OpResponse{txn: (*TxnResponse)(resp)}, nil
      }
   default:
      panic("Unknown op")
   }
   return OpResponse{}, toErr(ctx, err)
}
```

# 读写流程

```go
func Server(s *etcdserver.EtcdServer, tls *tls.Config, gopts ...grpc.ServerOption) *grpc.Server {
	//配置信息
   var opts []grpc.ServerOption
   opts = append(opts, grpc.CustomCodec(&codec{}))
   if tls != nil {
      opts = append(opts, grpc.Creds(credentials.NewTLS(tls)))
   }
   opts = append(opts, grpc.UnaryInterceptor(newUnaryInterceptor(s)))
   opts = append(opts, grpc.StreamInterceptor(newStreamInterceptor(s)))
   opts = append(opts, grpc.MaxRecvMsgSize(int(s.Cfg.MaxRequestBytes+grpcOverheadBytes)))
   opts = append(opts, grpc.MaxSendMsgSize(maxSendBytes))
   opts = append(opts, grpc.MaxConcurrentStreams(maxStreams))
    //创建grpcServer实例
   grpcServer := grpc.NewServer(append(opts, gopts...)...)
	//注册服务（kv、watch、lease、cluster、auth）
   pb.RegisterKVServer(grpcServer, NewQuotaKVServer(s))
   pb.RegisterWatchServer(grpcServer, NewWatchServer(s))
   pb.RegisterLeaseServer(grpcServer, NewQuotaLeaseServer(s))
   pb.RegisterClusterServer(grpcServer, NewClusterServer(s))
   pb.RegisterAuthServer(grpcServer, NewAuthServer(s))
   pb.RegisterMaintenanceServer(grpcServer, NewMaintenanceServer(s))

   grpclogOnce.Do(func() {
      if s.Cfg.Debug {
         grpc.EnableTracing = true
         // enable info, warning, error
         grpclog.SetLoggerV2(grpclog.NewLoggerV2(os.Stderr, os.Stderr, os.Stderr))
      } else {
         // only discard info
         grpclog.SetLoggerV2(grpclog.NewLoggerV2(ioutil.Discard, os.Stderr, os.Stderr))
      }
   })

   // to display all registered metrics with zero values
   grpc_prometheus.Register(grpcServer)

   return grpcServer
}
```

etcd/etcdserver/etcdserverpb/rpc.pb.go

```go
func RegisterKVServer(s *grpc.Server, srv KVServer) { //注册kvServer
   s.RegisterService(&_KV_serviceDesc, srv)
}
```

```go
var _KV_serviceDesc = grpc.ServiceDesc{
   ServiceName: "etcdserverpb.KV", //服务名称
   HandlerType: (*KVServer)(nil),
   Methods: []grpc.MethodDesc{  //暴露的方法集合
      { 
         MethodName: "Range", //方法名称
         Handler:    _KV_Range_Handler, //执行方法的入口handler
      },
      {
         MethodName: "Put",
         Handler:    _KV_Put_Handler,
      },
      {
         MethodName: "DeleteRange",
         Handler:    _KV_DeleteRange_Handler,
      },
      {
         MethodName: "Txn",
         Handler:    _KV_Txn_Handler,
      },
      {
         MethodName: "Compact",
         Handler:    _KV_Compact_Handler,
      },
   },
   Streams:  []grpc.StreamDesc{},
   Metadata: "rpc.proto",
}
```

## 读流程

接收Range请求

grpc拦截

KVServer线性读、串行读

获取最新的已经提交的日志索引

等待提交的日志应用到状态机

从MVCC的treeIndex模块读取版本号

将版本号作为key从boltdb查询value

### grpc拦截

```go
func _KV_Range_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
   in := new(RangeRequest)
   if err := dec(in); err != nil {
      return nil, err
   }
   if interceptor == nil { //拦截器为空，直接执行range方法
      return srv.(KVServer).Range(ctx, in)
   }
   info := &grpc.UnaryServerInfo{
      Server:     srv,
      FullMethod: "/etcdserverpb.KV/Range",
   }
   handler := func(ctx context.Context, req interface{}) (interface{}, error) {
      return srv.(KVServer).Range(ctx, req.(*RangeRequest))
   }
  //执行拦截器，在执行range方法
   return interceptor(ctx, in, info, handler)
}
```

### 线性读/串行读

etcd/etcdserver/v3_server.go

```go
func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
   var resp *pb.RangeResponse
   var err error
   defer func(start time.Time) {
      warnOfExpensiveReadOnlyRangeRequest(start, r, resp, err)
   }(time.Now())

   if !r.Serializable { //线性读,等待提交的日志都应用到状态机，然后再执行串行读，保证读到写入的最新数据
      err = s.linearizableReadNotify(ctx)
      if err != nil {
         return nil, err
      }
   }
  //串行读，直接从状态机查询数据即从boltdb查询数据
   chk := func(ai *auth.AuthInfo) error {
      return s.authStore.IsRangePermitted(ai, r.Key, r.RangeEnd)
   }

   get := func() { resp, err = s.applyV3Base.Range(nil, r) }
   if serr := s.doSerialize(ctx, chk, get); serr != nil {
      err = serr
      return nil, err
   }
   return resp, err
}
```

#### 线性读

```go
func (s *EtcdServer) linearizableReadNotify(ctx context.Context) error {
   s.readMu.RLock()
   nc := s.readNotifier
   s.readMu.RUnlock()

   // signal linearizable loop for current notify if it hasn't been already
   select {
   case s.readwaitc <- struct{}{}:
   default:
   }

   // wait for read state notification
   select {
   case <-nc.c:
      return nc.err
   case <-ctx.Done():
      return ctx.Err()
   case <-s.done:
      return ErrStopped
   }
}
```

#### 串行读

```go
func (s *EtcdServer) doSerialize(ctx context.Context, chk func(*auth.AuthInfo) error, get func()) error {
   for {
      ai, err := s.AuthInfoFromCtx(ctx) //认证
      if err != nil {
         return err
      }
      if ai == nil {
         // chk expects non-nil AuthInfo; use empty credentials
         ai = &auth.AuthInfo{}
      }
      if err = chk(ai); err != nil { //鉴权
         if err == auth.ErrAuthOldRevision {
            continue
         }
         return err
      }
      // fetch response for serialized request
      get()
      //  empty credentials or current auth info means no need to retry
      if ai.Revision == 0 || ai.Revision == s.authStore.Revision() {
         return nil
      }
      // avoid TOCTOU error, retry of the request is required.
   }
}
```

### 状态机读取数据

etcd/etcdserver/apply.go

```go
func (a *applierV3backend) Range(txn mvcc.TxnRead, r *pb.RangeRequest) (*pb.RangeResponse, error) {
   resp := &pb.RangeResponse{}
   resp.Header = &pb.ResponseHeader{}

   if txn == nil { //开启读事务
      txn = a.s.kv.Read()
      defer txn.End() //关闭事务
   }

   if isGteRange(r.RangeEnd) {
      r.RangeEnd = []byte{}
   }

   limit := r.Limit
   if r.SortOrder != pb.RangeRequest_NONE ||
      r.MinModRevision != 0 || r.MaxModRevision != 0 ||
      r.MinCreateRevision != 0 || r.MaxCreateRevision != 0 {
      // fetch everything; sort and truncate afterwards
      limit = 0
   }
   if limit > 0 {
      // fetch one extra for 'more' flag
      limit = limit + 1
   }

   ro := mvcc.RangeOptions{
      Limit: limit,
      Rev:   r.Revision,
      Count: r.CountOnly, //是否是返回key的个数
   }
	//查询数据，接下的流程可以参考ETCD-3.2中的readView
   rr, err := txn.Range(r.Key, r.RangeEnd, ro)
   if err != nil {
      return nil, err
   }

   if r.MaxModRevision != 0 {
      f := func(kv *mvccpb.KeyValue) bool { return kv.ModRevision > r.MaxModRevision }
      pruneKVs(rr, f)
   }
   if r.MinModRevision != 0 {
      f := func(kv *mvccpb.KeyValue) bool { return kv.ModRevision < r.MinModRevision }
      pruneKVs(rr, f)
   }
   if r.MaxCreateRevision != 0 {
      f := func(kv *mvccpb.KeyValue) bool { return kv.CreateRevision > r.MaxCreateRevision }
      pruneKVs(rr, f)
   }
   if r.MinCreateRevision != 0 {
      f := func(kv *mvccpb.KeyValue) bool { return kv.CreateRevision < r.MinCreateRevision }
      pruneKVs(rr, f)
   }

   sortOrder := r.SortOrder
   if r.SortTarget != pb.RangeRequest_KEY && sortOrder == pb.RangeRequest_NONE {
      // Since current mvcc.Range implementation returns results
      // sorted by keys in lexiographically ascending order,
      // sort ASCEND by default only when target is not 'KEY'
      sortOrder = pb.RangeRequest_ASCEND
   }
   if sortOrder != pb.RangeRequest_NONE {
      var sorter sort.Interface
      switch {
      case r.SortTarget == pb.RangeRequest_KEY:
         sorter = &kvSortByKey{&kvSort{rr.KVs}}
      case r.SortTarget == pb.RangeRequest_VERSION:
         sorter = &kvSortByVersion{&kvSort{rr.KVs}}
      case r.SortTarget == pb.RangeRequest_CREATE:
         sorter = &kvSortByCreate{&kvSort{rr.KVs}}
      case r.SortTarget == pb.RangeRequest_MOD:
         sorter = &kvSortByMod{&kvSort{rr.KVs}}
      case r.SortTarget == pb.RangeRequest_VALUE:
         sorter = &kvSortByValue{&kvSort{rr.KVs}}
      }
      switch {
      case sortOrder == pb.RangeRequest_ASCEND:
         sort.Sort(sorter)
      case sortOrder == pb.RangeRequest_DESCEND:
         sort.Sort(sort.Reverse(sorter))
      }
   }

   if r.Limit > 0 && len(rr.KVs) > int(r.Limit) {
      rr.KVs = rr.KVs[:r.Limit] //截取limit
      resp.More = true //表明还有数据
   } 

   resp.Header.Revision = rr.Rev
   resp.Count = int64(rr.Count) //返回的条数
   for i := range rr.KVs {
      if r.KeysOnly { //只返回key，不需要value
         rr.KVs[i].Value = nil
      }
      resp.Kvs = append(resp.Kvs, &rr.KVs[i])
   }
   return resp, nil
}
```

## 写流程

grpc拦截

Quota检查配额

KVServer模块发起提案

wal写入日志、同步日志

应用状态机（treeIndex建立索引、boltdb写数据）

### grpc拦截

```go
func _KV_Put_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
   in := new(PutRequest)
   if err := dec(in); err != nil {
      return nil, err
   }
   if interceptor == nil {
      return srv.(KVServer).Put(ctx, in)
   }
   info := &grpc.UnaryServerInfo{
      Server:     srv,
      FullMethod: "/etcdserverpb.KV/Put",
   }
   handler := func(ctx context.Context, req interface{}) (interface{}, error) {
      return srv.(KVServer).Put(ctx, req.(*PutRequest))
   }
   return interceptor(ctx, in, info, handler)
}
```

### Quota模块

etcdserver/api/v3rpc/grpc.go

```go
pb.RegisterKVServer(grpcServer, NewQuotaKVServer(s)) //grpc启动时注册KVServer
```

etcdserver/api/v3rpc/quota.go

```go
func NewQuotaKVServer(s *etcdserver.EtcdServer) pb.KVServer {
   return &quotaKVServer{ //包裹EtcdServer
      NewKVServer(s), //pb.KVServer
      quotaAlarmer{etcdserver.NewBackendQuota(s, "kv"), s, s.ID()}, //警报：配额已满，当配额已满时，会将配额已满的信息通过raft协议复制各个节点
   }
}
```

```go
func (s *quotaKVServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
 //检查当前db大小加上请求的key-value大小之和是否超过了配额
   if err := s.qa.check(ctx, r); err != nil {
      return nil, err
   }
   //处理写请求
   return s.KVServer.Put(ctx, r) 
}
```

### 处理写请求

etcd/etcdserver/v3_server.go

```go
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
   resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
   if err != nil {
      return nil, err
   }
   return resp.(*pb.PutResponse), nil
}
```

```go
func (s *EtcdServer) raftRequest(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
   for {
      resp, err := s.raftRequestOnce(ctx, r)
      if err != auth.ErrAuthOldRevision {
         return resp, err
      }
   }
}
```

```go
func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
   result, err := s.processInternalRaftRequestOnce(ctx, r)
   if err != nil {
      return nil, err
   }
   if result.err != nil {
      return nil, result.err
   }
   return result.resp, nil
}
```



```go
func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*applyResult, error) {
   ai := s.getAppliedIndex() //应用到状态机的日志索引
   ci := s.getCommittedIndex() //已经提交的日志索引，即已经复制到超过半数的节点
   if ci > ai+maxGapBetweenApplyAndCommitIndex {
     //默认未应用到状态机的日志项超过5000，返回ErrTooManyRequests，起到限流的作用
     //导致此异常的原因可能是状态机流程中出现的expensive read request导致写事务阻塞
      return nil, ErrTooManyRequests
   }

   r.Header = &pb.RequestHeader{
      ID: s.reqIDGen.Next(), //生成请求唯一标志
   }

   authInfo, err := s.AuthInfoFromCtx(ctx) //认证
   if err != nil {
      return nil, err
   }
   if authInfo != nil {
      r.Header.Username = authInfo.Username
      r.Header.AuthRevision = authInfo.Revision
   }

   data, err := r.Marshal() //序列化
   if err != nil {
      return nil, err
   }

   if len(data) > int(s.Cfg.MaxRequestBytes) { //请求体不能超过1.5M
      return nil, ErrRequestTooLarge
   }

   id := r.ID //请求自带的id
   if id == 0 {
      id = r.Header.ID //设置为服务生成的id
   }
   ch := s.w.Register(id)  //注册（id->chan），返回chan

   cctx, cancel := context.WithTimeout(ctx, s.Cfg.ReqTimeout())//设置请求超时时间
   defer cancel()

   start := time.Now()
  //发起提案（如果当前节点是Follower，转发给Leader，只有Leader处理写请求）
   s.r.Propose(cctx, data) 
   proposalsPending.Inc() //处理中的提案数+1
   defer proposalsPending.Dec() //处理完-1

   select {
   case x := <-ch: //等待请求处理完成
      return x.(*applyResult), nil
   case <-cctx.Done(): //请求处理超时
      proposalsFailed.Inc() //失败提案数+1
      s.w.Trigger(id, nil) //删除此请求对应的chan
      return nil, s.parseProposeCtxErr(cctx.Err(), start)
   case <-s.done: //服务端关闭
      return nil, ErrStopped
   }
}
```

### 设置超时时间

```go
func (c *ServerConfig) ReqTimeout() time.Duration {
   // 5s for queue waiting, computation and disk IO delay
   // + 2 * election timeout for possible leader election
  //队列等待、计算、磁盘延迟+选举时间
   return 5*time.Second + 2*time.Duration(c.ElectionTicks)*time.Duration(c.TickMs)*time.Millisecond
}
```

### 提案

```go
func (n *node) Propose(ctx context.Context, data []byte) error {
   return n.step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}
```

```go
func (n *node) step(ctx context.Context, m pb.Message) error {
   ch := n.recvc
   if m.Type == pb.MsgProp { //提案类型
      ch = n.propc
   }

   select {
   case ch <- m: //将请求放入node的propc通道，node启动时会调用run方法，run方法会对propc进行处理
      return nil
   case <-ctx.Done():
      return ctx.Err()
   case <-n.done:
      return ErrStopped
   }
}
```

#### leader处理提案

raft/raft.go

```go
func stepLeader(r *raft, m pb.Message) {
   // These message types do not require any progress for m.From.
   switch m.Type {
		...
   case pb.MsgProp: //事务类请求
      if len(m.Entries) == 0 {
         r.logger.Panicf("%x stepped empty MsgProp", r.id)
      }
      if _, ok := r.prs[r.id]; !ok {
         return
      }
      if r.leadTransferee != None { //leader正在转让
         r.logger.Debugf("%x [term %d] transfer leadership to %x is in progress; dropping proposal", r.id, r.Term, r.leadTransferee)
         return
      }
			//忽略配置变更如果有配置变更尚未应用到状态机*
      for i, e := range m.Entries {
         if e.Type == pb.EntryConfChange {
            if r.pendingConf {
               r.logger.Infof("propose conf %s ignored since pending unapplied configuration", e.String())
               m.Entries[i] = pb.Entry{Type: pb.EntryNormal}
            }
            r.pendingConf = true
         }
      }
      r.appendEntry(m.Entries...) //追加数据存在内存中
      r.bcastAppend() // 向其他的节点同步数据
      return
  		...
}
```

### 应用状态机

etcd/etcdserver/server.go

```go
func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply) {
   s.applySnapshot(ep, apply)
   s.applyEntries(ep, apply)//写入到boltdb

   proposalsApplied.Set(float64(ep.appliedi))
   s.applyWait.Trigger(ep.appliedi) // 唤醒阻塞的readindex请求
   // wait for the raft routine to finish the disk writes before triggering a
   // snapshot. or applied index might be greater than the last index in raft
   // storage, since the raft routine might be slower than apply routine.
   <-apply.notifyc

   s.triggerSnapshot(ep) //触发快照
   select {
   // snapshot requested via send()
   case m := <-s.r.msgSnapC:
      merged := s.createMergedSnapshotMessage(m, ep.appliedt, ep.appliedi, ep.confState)
      s.sendMergedSnap(merged)
   default:
   }
}
```

```go
func (s *EtcdServer) applyEntries(ep *etcdProgress, apply *apply) {
   if len(apply.entries) == 0 {
      return
   }
   firsti := apply.entries[0].Index
   if firsti > ep.appliedi+1 {
      plog.Panicf("first index of committed entry[%d] should <= appliedi[%d] + 1", firsti, ep.appliedi)
   }
   var ents []raftpb.Entry
   if ep.appliedi+1-firsti < uint64(len(apply.entries)) {
      ents = apply.entries[ep.appliedi+1-firsti:]
   }
   if len(ents) == 0 {
      return
   }
   var shouldstop bool
   if ep.appliedt, ep.appliedi, shouldstop = s.apply(ents, &ep.confState); shouldstop {
      go s.stopWithDelay(10*100*time.Millisecond, fmt.Errorf("the member has been permanently removed from the cluster"))
   }
}
```

```go
func (s *EtcdServer) apply(es []raftpb.Entry, confState *raftpb.ConfState) (appliedt uint64, appliedi uint64, shouldStop bool) {
   for i := range es {
      e := es[i]
      switch e.Type {
      case raftpb.EntryNormal:
         s.applyEntryNormal(&e) //写数据
      case raftpb.EntryConfChange:
         // set the consistent index of current executing entry
         if e.Index > s.consistIndex.ConsistentIndex() {
            s.consistIndex.setConsistentIndex(e.Index)
         }
         var cc raftpb.ConfChange
         pbutil.MustUnmarshal(&cc, e.Data)
         removedSelf, err := s.applyConfChange(cc, confState)
         s.setAppliedIndex(e.Index)
         shouldStop = shouldStop || removedSelf
         s.w.Trigger(cc.ID, &confChangeResponse{s.cluster.Members(), err})
      default:
         plog.Panicf("entry type should be either EntryNormal or EntryConfChange")
      }
      atomic.StoreUint64(&s.r.index, e.Index)
      atomic.StoreUint64(&s.r.term, e.Term)
      appliedt = e.Term
      appliedi = e.Index
   }
   return appliedt, appliedi, shouldStop
}
```

#### 写数据

```go
func (s *EtcdServer) applyEntryNormal(e *raftpb.Entry) {
   shouldApplyV3 := false
   if e.Index > s.consistIndex.ConsistentIndex() {
      // 保证幂等，保证不会重复提交；提交事务时会将ConsistentIndex写入boltdb
      s.consistIndex.setConsistentIndex(e.Index)
      shouldApplyV3 = true
   }
   defer s.setAppliedIndex(e.Index) //修改已提交到状态机的索引

   // raft state machine may generate noop entry when leader confirmation.
   // skip it in advance to avoid some potential bug in the future
   if len(e.Data) == 0 {
      select {
      case s.forceVersionC <- struct{}{}:
      default:
      }
      // promote lessor when the local member is leader and finished
      // applying all entries from the last term.
      if s.isLeader() {
         s.lessor.Promote(s.Cfg.electionTimeout())
      }
      return
   }

   var raftReq pb.InternalRaftRequest
   if !pbutil.MaybeUnmarshal(&raftReq, e.Data) { //向后兼容v2版本
      var r pb.Request
      pbutil.MustUnmarshal(&r, e.Data)
      s.w.Trigger(r.ID, s.applyV2Request(&r)) //存储、唤醒阻塞的请求
      return
   }
   if raftReq.V2 != nil {
      req := raftReq.V2
      s.w.Trigger(req.ID, s.applyV2Request(req))
      return
   }

   // do not re-apply applied entries.
   if !shouldApplyV3 {
      return
   }
	//v3版本
   id := raftReq.ID
   if id == 0 {
      id = raftReq.Header.ID
   }
	//此请求是否已经注册过
   var ar *applyResult
   needResult := s.w.IsRegistered(id)
   if needResult || !noSideEffect(&raftReq) {
      if !needResult && raftReq.Txn != nil {
         removeNeedlessRangeReqs(raftReq.Txn)
      }
     //AuthApplierV3:认证、配额检查 或者 applierV3Capped:不可写入
      ar = s.applyV3.Apply(&raftReq)
   }

   if ar == nil {
      return
   }
   if ar.err != ErrNoSpace || len(s.alarmStore.Get(pb.AlarmType_NOSPACE)) > 0 {
     //非空间不足的异常或者boltdb已经写入AlarmRequest_nospace
      s.w.Trigger(id, ar) //唤醒阻塞的请求
      return
   }
	//空间不足，提交AlarmRequest，告知其他节点，禁止写入
   plog.Errorf("applying raft message exceeded backend quota")
   s.goAttach(func() {
      a := &pb.AlarmRequest{
         MemberID: uint64(s.ID()),
         Action:   pb.AlarmRequest_ACTIVATE,
         Alarm:    pb.AlarmType_NOSPACE,
      }
      s.raftRequest(s.ctx, pb.InternalRaftRequest{Alarm: a})//发起raft提案
      s.w.Trigger(id, ar) //唤醒阻塞的请求
   })
}
```

etcd/etcdserver/apply.go

```go
func (s *EtcdServer) newApplierV3() applierV3 {
   return newAuthApplierV3(
      s.AuthStore(), //负责认证、鉴权
      newQuotaApplierV3( //配额检查
        s, &applierV3backend{s} //负责存储
      ), 
   )
}
```

```go
func (aa *authApplierV3) Apply(r *pb.InternalRaftRequest) *applyResult {
   aa.mu.Lock()
   defer aa.mu.Unlock()
   if r.Header != nil {
      // backward-compatible with pre-3.0 releases when internalRaftRequest
      // does not have header field
      aa.authInfo.Username = r.Header.Username
      aa.authInfo.Revision = r.Header.AuthRevision
   }
  //权限认证
   if needAdminPermission(r) {
      if err := aa.as.IsAdminPermitted(&aa.authInfo); err != nil {
         aa.authInfo.Username = ""
         aa.authInfo.Revision = 0
         return &applyResult{err: err}
      }
   }
   ret := aa.applierV3.Apply(r)
   aa.authInfo.Username = ""
   aa.authInfo.Revision = 0
   return ret
}
```



```go
func (a *quotaApplierV3) Put(txn mvcc.TxnWrite, p *pb.PutRequest) (*pb.PutResponse, error) {
   ok := a.q.Available(p) //配额检查
   resp, err := a.applierV3.Put(txn, p) //后续流程参考ETCD-3.2 writeView
   if err == nil && !ok {
      err = ErrNoSpace
   }
   return resp, err
}
```



```go
func (b *backendQuota) Available(v interface{}) bool {
   // TODO: maybe optimize backend.Size()
   return b.s.Backend().Size()+int64(b.Cost(v)) < b.maxBackendBytes
}
```

etcd/etcdserver/apply.go

```go
func newApplierV3Capped(base applierV3) applierV3 { 
  return &applierV3Capped{applierV3: base}  //对authApplierV3进行装饰，主要禁止写操作
}
```

#### 配置变更

etcd/etcdserver/server.go

```go
func (s *EtcdServer) applyConfChange(cc raftpb.ConfChange, confState *raftpb.ConfState) (bool, error) {
   if err := s.cluster.ValidateConfigurationChange(cc); err != nil {
      cc.NodeID = raft.None
      s.r.ApplyConfChange(cc)
      return false, err
   }
   *confState = *s.r.ApplyConfChange(cc)
   switch cc.Type {
   case raftpb.ConfChangeAddNode: //添加节点
      m := new(membership.Member)
      if err := json.Unmarshal(cc.Context, m); err != nil {
         plog.Panicf("unmarshal member should never fail: %v", err)
      }
      if cc.NodeID != uint64(m.ID) {
         plog.Panicf("nodeID should always be equal to member ID")
      }
      s.cluster.AddMember(m)
      if m.ID != s.id {
         s.r.transport.AddPeer(m.ID, m.PeerURLs)
      }
   case raftpb.ConfChangeRemoveNode: //移除节点
      id := types.ID(cc.NodeID)
      s.cluster.RemoveMember(id)
      if id == s.id {
         return true, nil
      }
      s.r.transport.RemovePeer(id)
   case raftpb.ConfChangeUpdateNode: //修改配置
      m := new(membership.Member)
      if err := json.Unmarshal(cc.Context, m); err != nil {
         plog.Panicf("unmarshal member should never fail: %v", err)
      }
      if cc.NodeID != uint64(m.ID) {
         plog.Panicf("nodeID should always be equal to member ID")
      }
      s.cluster.UpdateRaftAttributes(m.ID, m.RaftAttributes)
      if m.ID != s.id {
         s.r.transport.UpdatePeer(m.ID, m.PeerURLs)
      }
   }
   return false, nil
}
```

# 多副本复制

## 全同步复制

指主收到一个写请求后，必须等待全部从节点确认返回后，才能返回给客户端成功。因此如果一个从节点故障，整个系统就会不可用。这种方案为了保证多副本的一致性，而牺牲了可用性，一般使用不多。

## 异步复制

指主收到一个写请求后，可及时返回给 client，异步将请求转发给各个副本，若还未将请求转发到副本前就故障了，则可能导致数据丢失，但是可用性是最高的

## 半同步复制

介于全同步复制、异步复制之间，它是指主收到一个写请求后，至少有一个副本接收数据后，就可以返回给客户端成功，在数据一致性、可用性上实现了平衡和取舍。

## 去中心化复制

它是指在一个 n 副本节点集群中，任意节点都可接受写请求，但一个成功的写入需要 w 个节点确认，读取也必须查询至少 r 个节点。但是会导致各种写入冲突，业务需要关注冲突处理。

# FAQ

```
What does "mvcc: database space exceeded" mean and how do I fix it?
etcd中的mvcc机制保留了key的历史版本，如果不定期压缩历史记录(通过设置'——auto-compaction')，Etcd最终将耗尽其存储空间。如果etcd的存储空间不足，它会发出空间配额警报，以保护集群不被进一步写入。只要告警被触发，etcd就会以错误'mvcc: database space exceeded'来响应写请求。
从低空间配额告警中恢复：
1、合并历史版本
2、碎片整理
3、解除告警
```

```
What does the etcd warning "apply entries took too long" mean?
在大多数etcd成员同意提交请求后，每个etcd服务器将请求应用于状态机，并将结果持久化到磁盘。即使使用慢速的机械磁盘或虚拟网络磁盘，如Amazon的EBS或谷歌的PD，通常情况下，应用一个请求所需的时间应该少于50毫秒。如果平均应用持续时间超过100毫秒，etcd将警告条目应用时间过长
通常这个问题是由慢盘引起的。该磁盘可能正在etcd和其他应用程序之间发生争用，或者磁盘太慢。要排除导致此警告的慢盘，请使用monitor [backend_commit_duration_seconds][backend_commit_metrics](p99持续时间应该少于25ms)来确认磁盘是相当快的。如果磁盘太慢，分配一个专用的磁盘或使用更快的磁盘通常可以解决这个问题

第二个最常见的原因是CPU不足。如果监控到机器的CPU使用率很高，etcd可能没有足够的计算容量。将etcd移至专用机器，增加进程资源隔离组，或者将etcd服务器进程重新设置为更高的优先级

昂贵的用户请求访问太多的键(例如获取整个键空间)也会导致很长的应用延迟
```

```
Do clients have to send requests to the etcd leader?
Leader处理所有需要集群协商一致的客户请求。但是，客户端并不需要知道哪个节点是Leader。任何发送给Follower的需要协商一致的请求都会自动转发给Leader。不需要协商一致的请求(例如，序列化读取)可以由任何集群成员处理。
```

```
What is maximum cluster size?
理论上，没有硬性限制。然而，etcd集群可能不应该有超过7个节点。一般建议运行5个节点。5个成员的etcd集群可以容忍两个成员的故障，这在大多数情况下已经足够了。尽管更大的集群提供了更好的容错性，但由于数据必须被复制，随着机器的增多写性能会受到影响
```

```
Request size limit
Etcd被设计用来处理元数据中典型的小键值对。更大的请求仍然会被处理，但可能会增加其他请求的延迟。目前，etcd保证支持带有最多1MB数据的RPC请求。在未来，尺寸限制可能会被放宽或变为可配置的。
Storage size limit
默认存储大小限制为2GB，可通过'——quota-backend-bytes '标志进行配置;支持高达8GB。
```

```
What does the etcd warning "snapshotting is taking more than x seconds to finish ..." mean?
etcd sends a snapshot of its complete key-value store to refresh slow followers and for [backups][backup].
 Slow snapshot transfer times increase MTTR; if the cluster is ingesting data with high throughput,
  slow followers may livelock by needing a new snapshot before finishing receiving a snapshot. 
  To catch slow snapshot performance, etcd warns when sending a snapshot takes more than thirty seconds and exceeds 
  the expected transfer time for a 1Gbps connection.
```

# 性能优化

## 客户端负载均衡

在 etcd 3.4 以前，client 只会选择一个 sever 节点 ，与其建立一个长连接。这可能会导致对应的 server 节点过载，其他节点却是低负载。3.4之后引入round-robin负载均衡算法，可以使集群的负载更加均衡

## 选择合适的鉴权

避免使用密码鉴权，而是使用证书鉴权。尽量复用token减少Authenticate RPC调用

## 选择合适的读模式

etcd 提供了串行读和线性读两种读模式。前者因为不经过 ReadIndex 模块，具有低延时、高吞吐量的特点；而后者在牺牲一点延时和吞吐量的基础上，实现了数据的强一致性读。

## 配额

默认 db quota 大小是 2G，最大为8G，超过就只读无法写入。需要根据业务场景，适当调整 db quota 大小，并配置的合适的压缩策略

## 限速

请求在提交到 Raft 模块前，会进行限速判断。

如果 Raft 模块已提交的日志索引（committed index）比已应用到状态机的日志索引（applied index）超过了5000，那么它就返回一个"etcdserver: too many requests"错误给 client。

## 心跳选举参数优化

通过日志和metrics来判断Leader是否稳定

Leader 节点会根据 heartbeart-interval 参数默认100ms定时向Follower 节点发送心跳。如果两次发送心跳间隔超过 2*heartbeart-interval，就会打印警告日志

超过 election timeout（默认 1000ms），Follower 节点就会发起新一轮的Leader 选举。

根据实际部署环境、业务场景，将心跳间隔时间调整到 100ms 到 400ms 左右，选举超时时间要求至少是心跳间隔的 10倍

## 磁盘和网络延迟

如果服务对性能、稳定性有较大要求，建议你使用 SSD 盘存放wal文件。

网络模块中会通过流式发送和 pipeline 等技术优化来降低延时、提高网络性能。

## 快照参数优化

Follower落后数据太多，Leader内存中的数据已经被合并，Leader会发送快照帮助Follower尽快同步数据

快照较大的时候会消耗cpu、磁盘、网络

snapshot-count参数是指收到多少个写请求后就触发生成一次快照

etcd 在生成快照时同时会进行压缩日志，保证在内存中保存 5000 条日志

避免频繁修改键值对数据

对于相同TTL失效的键值对绑定到相同的Lease

尽量避免大key的查询操作，在启动时查询一次，之后通过watch机制实时获取数据变化 或者将大key拆分成多个小key

# 请求延迟的原因

网络质量，节点之间的RTT延时、网卡带宽打满、丢包

**磁盘抖动导致wal日志持久化、boltdb事务提交出现抖动**

etcd_disk_wal_fsync_duration_seconds(disk_wal_fsync )表示 WAL 日志持久化的 fsync 系统调用延时数据。一般本地 SSD 盘 P99 延时在 10ms 内

etcd_disk_backend_commit_duration_seconds( disk_backend_commit )表示后端 boltdb 事务提交的延时，一般 P99 在 120ms 内。

etcd 还提供了 disk_backend_commit_rebalance_duration 和disk_backend_commit_spill_duration 两个 metrics，分别表示事务提交过程中 B+ tree的重平衡和分裂操作耗时分布区间。

expensive request

比如大包请求、涉及到大量 key 遍历、Authenticate 密码鉴权等操作

线性读高延迟，检查否存在大量写请求。线性读需确保本节点数据与 Leader 数据一样新，若本节点的数据与 Leader 差异较大，本节点追赶 Leader 数据过程会花费一定时间，最终导致高延时的线性读请求产生。

容量瓶颈，太多写请求导致线性读请求性能下降等

节点配置，CPU 繁忙导致请求处理延时、内存不够导致 swap

最后通过 ps/top/mpstat/perf 等 CPU、Memory 性能分析工具，检查 etcd 节点是否存在 CPU、Memory 瓶颈。通过开启 debug 模式，通过 pprof 分析 CPU 和内存瓶颈点。