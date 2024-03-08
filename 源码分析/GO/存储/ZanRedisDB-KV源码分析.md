# PlaceDriver

## 启动流程

 apps/placedriver/main.go:

```go
func (p *program) Start() error {
   glog.InitWithFlag(flagSet)

   flagSet.Parse(os.Args[1:])
   fmt.Println(common.VerString("placedriver"))
   if *showVersion {
      os.Exit(0)
   }

   var cfg map[string]interface{}
   if *config != "" {
      _, err := toml.DecodeFile(*config, &cfg)
      if err != nil {
         log.Fatalf("ERROR: failed to load config file %s - %s", *config, err.Error())
      }
   }

   opts := pdserver.NewServerConfig()
   options.Resolve(opts, flagSet, cfg)
   if opts.LogDir != "" {
      glog.SetGLogDir(opts.LogDir)
   }
   glog.StartWorker(time.Second * 2)
   daemon := pdserver.NewServer(opts) //创建server

   daemon.Start() //启动server
   p.placedriver = daemon
   return nil
}
```

### 创建ServerConfig

pdserver/config.go:

```go
func NewServerConfig() *ServerConfig {
   hostname, err := os.Hostname()
   if err != nil {
      log.Fatal(err)
   }

   return &ServerConfig{
      HTTPAddress:        "0.0.0.0:18001",
      BroadcastAddr:      hostname,
      BroadcastInterface: "eth0",
      ProfilePort:        "7667",

      ClusterLeadershipAddresses: "",
      ClusterID:                  "",

      LogLevel: 1,
      LogDir:   "",
   }
}
```

### 创建PDServer

pdserver/server.go

```go
func NewServer(conf *ServerConfig) *Server {
   hname, err := os.Hostname()
   if err != nil {
      sLog.Fatal(err)
   }

   myNode := &cluster.NodeInfo{
      NodeIP:   conf.BroadcastAddr,
      Hostname: hname,
      Version:  common.VerBinary,
   }

   if conf.ClusterID == "" {
      sLog.Fatalf("cluster id can not be empty")
   }
   if conf.BroadcastInterface != "" {
      myNode.NodeIP = common.GetIPv4ForInterfaceName(conf.BroadcastInterface)
   }
   if myNode.NodeIP == "" {
      myNode.NodeIP = conf.BroadcastAddr
   } else {
      conf.BroadcastAddr = myNode.NodeIP
   }
   if myNode.NodeIP == "0.0.0.0" || myNode.NodeIP == "" {
      sLog.Errorf("can not decide the broadcast ip: %v", myNode.NodeIP)
      os.Exit(1)
   }
   _, myNode.HttpPort, _ = net.SplitHostPort(conf.HTTPAddress)
   if conf.ReverseProxyPort != "" {
      myNode.HttpPort = conf.ReverseProxyPort
   }

   sLog.Infof("Start with broadcast ip:%s", myNode.NodeIP)
   myNode.ID = cluster.GenNodeID(myNode, "pd")

   clusterOpts := &cluster.Options{}
   clusterOpts.AutoBalanceAndMigrate = conf.AutoBalanceAndMigrate
   if len(conf.BalanceInterval) == 2 {
      clusterOpts.BalanceStart, err = strconv.Atoi(conf.BalanceInterval[0])
      if err != nil {
         sLog.Errorf("invalid balance interval: %v", err)
         os.Exit(1)
      }
      clusterOpts.BalanceEnd, err = strconv.Atoi(conf.BalanceInterval[1])
      if err != nil {
         sLog.Errorf("invalid balance interval: %v", err)
         os.Exit(1)
      }
   }
   s := &Server{
      conf:             conf,
      stopC:            make(chan struct{}),
      pdCoord:          pdnode_coord.NewPDCoordinator(conf.ClusterID, myNode, clusterOpts),
      tombstonePDNodes: make(map[string]bool),
   }

   r := cluster.NewPDEtcdRegister(conf.ClusterLeadershipAddresses)
   s.pdCoord.SetRegister(r)

   return s
}
```

#### 创建PDCoordinator

cluster/pdnode_coord/pd_coordinator.go

```go
func NewPDCoordinator(clusterID string, n *cluster.NodeInfo, opts *cluster.Options) *PDCoordinator {
   coord := &PDCoordinator{
      clusterKey:             clusterID,
      myNode:                 *n,
      register:               nil,
      dataNodes:              make(map[string]cluster.NodeInfo),
      removingNodes:          make(map[string]string),
      checkNamespaceFailChan: make(chan cluster.NamespaceNameInfo, 3),
      stopChan:               make(chan struct{}),
      monitorChan:            make(chan struct{}),
      autoBalance:            opts.AutoBalanceAndMigrate,
   }
   coord.dpm = NewDataPlacement(coord)
   if opts != nil {
      coord.dpm.SetBalanceInterval(opts.BalanceStart, opts.BalanceEnd)
   }
   return coord
}
```

#### 创建DataPlacement

cluster/pdnode_coord/place_driver.go

```go
func NewDataPlacement(coord *PDCoordinator) *DataPlacement {
   return &DataPlacement{
      pdCoord:         coord,
      balanceInterval: [2]int{2, 4},
   }
}
```

#### 创建PDEtcdRegister

cluster/register_etcd.go

```go
func NewPDEtcdRegister(host string) *PDEtcdRegister {
   return &PDEtcdRegister{
      EtcdRegister:     NewEtcdRegister(host),
      watchNodesStopCh: make(chan bool, 1),
      refreshStopCh:    make(chan bool, 1),
   }
}
```

cluster/pdnode_coord/pd_coordinator.go

```go
func (pdCoord *PDCoordinator) SetRegister(r cluster.PDRegister) {
   pdCoord.register = r
}
```

### 启动PDServer

 pdserver/server.go:105

```go
func (s *Server) Start() {
   err := s.pdCoord.Start() //启动PD
   if err != nil {
      sLog.Errorf("FATAL: start coordinator failed - %s", err)
      os.Exit(1)
   }

   s.wg.Add(1)
   go func() {
      defer s.wg.Done()
      s.serveHttpAPI(s.conf.HTTPAddress, s.stopC) //启动Http
   }()
}
```

#### 启动PD

 cluster/pdnode_coord/pd_coordinator.go:79

```go
func (pdCoord *PDCoordinator) Start() error {
   if pdCoord.register != nil { //默认PDEtcdRegister
      pdCoord.register.InitClusterID(pdCoord.clusterKey)
     //注册节点信息到etcd
      err := pdCoord.register.Register(&pdCoord.myNode)
      if err != nil {
         cluster.CoordLog().Warningf("failed to register pd coordinator: %v", err)
         return err
      }
   }
   pdCoord.wg.Add(1)
   go pdCoord.handleLeadership()
   return nil
}
```

```go
func (etcdReg *EtcdRegister) InitClusterID(id string) {
   etcdReg.clusterID = id
   etcdReg.namespaceRoot = etcdReg.getNamespaceRootPath()
   etcdReg.clusterPath = etcdReg.getClusterPath()
   etcdReg.pdNodeRootPath = etcdReg.getPDNodeRootPath()
   go etcdReg.watchNamespaces() //监测namespace的变化
   go etcdReg.refreshNamespaces()//刷新namespace
}
```

```go
func (pdCoord *PDCoordinator) handleLeadership() {
   defer pdCoord.wg.Done()
   leaderChan := make(chan *cluster.NodeInfo)
   if pdCoord.register != nil {
     //选举leader
      go pdCoord.register.AcquireAndWatchLeader(leaderChan, pdCoord.stopChan)
   }
   defer func() {
      if e := recover(); e != nil {
         buf := make([]byte, 4096)
         n := runtime.Stack(buf, false)
         buf = buf[0:n]
         cluster.CoordLog().Errorf("panic %s:%v", buf, e)
      }

      cluster.CoordLog().Warningf("leadership watch exit.")
      if pdCoord.monitorChan != nil {
         close(pdCoord.monitorChan)
         pdCoord.monitorChan = nil
      }
   }()
   for {
      select {
      case l, ok := <-leaderChan:
         if !ok {
            cluster.CoordLog().Warningf("leader chan closed.")
            return
         }
         if l == nil {
            atomic.StoreInt32(&pdCoord.isClusterUnstable, 1)
            cluster.CoordLog().Warningf("leader is lost.")
            continue
         }
         if l.GetID() != pdCoord.leaderNode.GetID() { //leader发生变更
            cluster.CoordLog().Infof("leader changed from %v to %v", pdCoord.leaderNode, *l)
            pdCoord.leaderNode = *l
            if pdCoord.leaderNode.GetID() != pdCoord.myNode.GetID() { //leader不是本节点
               // remove watchers.
               if pdCoord.monitorChan != nil {
                  close(pdCoord.monitorChan)
               }
               pdCoord.monitorChan = make(chan struct{})
            }
            pdCoord.notifyLeaderChanged(pdCoord.monitorChan) //通知leader发生变化
         }
         if pdCoord.leaderNode.GetID() == "" {
            cluster.CoordLog().Warningf("leader is missing.")
         }
      }
   }
}
```

```go
func (pdCoord *PDCoordinator) notifyLeaderChanged(monitorChan chan struct{}) {
   if pdCoord.leaderNode.GetID() != pdCoord.myNode.GetID() { //leader不是本节点
      cluster.CoordLog().Infof("I am slave (%v). Leader is: %v", pdCoord.myNode, pdCoord.leaderNode)
      pdCoord.nodesMutex.Lock()
      pdCoord.removingNodes = make(map[string]string)
      pdCoord.nodesMutex.Unlock()

      pdCoord.wg.Add(1)
      go func() {
         defer pdCoord.wg.Done()
         pdCoord.handleDataNodes(monitorChan, false)
      }()

      return
   }
  //当选leader
   cluster.CoordLog().Infof("I am master now.")
   if pdCoord.register != nil {
      newNamespaces, _, err := pdCoord.register.GetAllNamespaces()
      if err != nil {
         cluster.CoordLog().Errorf("load namespace info failed: %v", err)
      } else {
         cluster.CoordLog().Infof("namespace loaded : %v", len(newNamespaces))
      }
   }

   pdCoord.wg.Add(1)
   go func() {
      defer pdCoord.wg.Done()
      pdCoord.handleDataNodes(monitorChan, true) 
   }()
   pdCoord.wg.Add(1)
   go func() {
      defer pdCoord.wg.Done()
      pdCoord.checkNamespaces(monitorChan)
   }()
   pdCoord.wg.Add(1)
   go func() {
      defer pdCoord.wg.Done()
      pdCoord.dpm.DoBalance(monitorChan)
   }()
   pdCoord.wg.Add(1)
   go func() {
      defer pdCoord.wg.Done()
      pdCoord.handleRemovingNodes(monitorChan)
   }()
}
```

### handleDataNodes

```go
func (pdCoord *PDCoordinator) handleDataNodes(monitorChan chan struct{}, isMaster bool) {
  //监测集群中节点的上下线 
  nodesChan := make(chan []cluster.NodeInfo)
   if pdCoord.register != nil {
      go pdCoord.register.WatchDataNodes(nodesChan, monitorChan)
   }
   cluster.CoordLog().Debugf("start watch the nodes.")
   defer func() {
      cluster.CoordLog().Infof("stop watch the nodes.")
   }()
   for {
      select {
      case nodes, ok := <-nodesChan:
         if !ok {
            return
         }
         // check if any node changed.
         cluster.CoordLog().Infof("Current data nodes: %v", len(nodes))
         oldNodes := pdCoord.dataNodes
         newNodes := make(map[string]cluster.NodeInfo)
         for _, v := range nodes {
            newNodes[v.GetID()] = v
         }
         pdCoord.nodesMutex.Lock()
         pdCoord.dataNodes = newNodes
         check := false
         for oldID, oldNode := range oldNodes {
            if _, ok := newNodes[oldID]; !ok { //节点下线
               cluster.CoordLog().Warningf("node failed: %v, %v", oldID, oldNode)
               // if node is missing we need check election immediately.
               check = true
            }
         }
         // failed need be protected by lock so we can avoid contention.
         if check {
            atomic.AddInt64(&pdCoord.nodesEpoch, 1)
         }
         if int32(len(pdCoord.dataNodes)) > atomic.LoadInt32(&pdCoord.stableNodeNum) {
            oldNum := atomic.LoadInt32(&pdCoord.stableNodeNum)
            atomic.StoreInt32(&pdCoord.stableNodeNum, int32(len(pdCoord.dataNodes)))
            cluster.CoordLog().Warningf("stable cluster node number changed from %v to: %v", oldNum, atomic.LoadInt32(&pdCoord.stableNodeNum))
         }
         pdCoord.nodesMutex.Unlock()

         if pdCoord.register == nil {
            continue
         }
         for newID, newNode := range newNodes {
            if _, ok := oldNodes[newID]; !ok { //新节点上线
               cluster.CoordLog().Infof("new node joined: %v, %v", newID, newNode)
               check = true
            }
         }
         if check && isMaster { //当前节点时leader并且集群中有节点上下线
            atomic.AddInt64(&pdCoord.nodesEpoch, 1)
            atomic.StoreInt32(&pdCoord.isClusterUnstable, 1)
            pdCoord.triggerCheckNamespaces("", 0, time.Millisecond*10)
         }
      }
   }
}
```

```go
func (pdCoord *PDCoordinator) triggerCheckNamespaces(namespace string, part int, delay time.Duration) {
   time.Sleep(delay)

   select {
   case pdCoord.checkNamespaceFailChan <- cluster.NamespaceNameInfo{NamespaceName: namespace, NamespacePartition: part}:
   case <-pdCoord.stopChan:
      return
   case <-time.After(time.Second):
      return
   }
}
```

### checkNamespaces

```go
func (pdCoord *PDCoordinator) checkNamespaces(monitorChan chan struct{}) {
  // check if partition is enough,
	// check if replication is enough
	// check any unexpected state.
   ticker := time.NewTicker(time.Second * 60)
   waitingMigrateNamespace := make(map[string]map[int]time.Time)
   defer func() {
      ticker.Stop()
      cluster.CoordLog().Infof("check namespaces quit.")
   }()

   if pdCoord.register == nil {
      return
   }
   for {
      select {
      case <-monitorChan:
         return
      case <-ticker.C:
         pdCoord.doCheckNamespaces(monitorChan, nil, waitingMigrateNamespace, true)
      case failedInfo := <-pdCoord.checkNamespaceFailChan:
         pdCoord.doCheckNamespaces(monitorChan, &failedInfo, waitingMigrateNamespace, failedInfo.NamespaceName == "")
      }
   }
}
```

### DoBalance

```go
func (dp *DataPlacement) DoBalance(monitorChan chan struct{}) {
   //check period for the data balance.
   ticker := time.NewTicker(time.Minute * 10)
   defer func() {
      ticker.Stop()
      cluster.CoordLog().Infof("balance check exit.")
   }()
   for {
      select {
      case <-monitorChan:
         return
      case <-ticker.C:
         // 只在指定的时间段内进行重平衡
         if time.Now().Hour() > dp.balanceInterval[1] || time.Now().Hour() < dp.balanceInterval[0] {
            continue
         }
         if !dp.pdCoord.IsMineLeader() { //非leader
            cluster.CoordLog().Infof("not leader while checking balance")
            continue
         }
         if !dp.pdCoord.IsClusterStable() {
            cluster.CoordLog().Infof("no balance since cluster is not stable while checking balance")
            continue
         }
         if !dp.pdCoord.autoBalance { //未开启自动平衡
            continue
         }
         cluster.CoordLog().Infof("begin checking balance of namespace data...")
         currentNodes := dp.pdCoord.getCurrentNodes(nil)
         validNum := len(currentNodes)
         if validNum < 2 {
            continue
         }
         dp.rebalanceNamespace(monitorChan)
      }
   }
}
```

### handleRemovingNodes

```go
func (pdCoord *PDCoordinator) handleRemovingNodes(monitorChan chan struct{}) {
   cluster.CoordLog().Debugf("start handle the removing nodes.")
   defer func() {
      cluster.CoordLog().Infof("stop handle the removing nodes.")
   }()
   ticker := time.NewTicker(time.Minute)
   defer ticker.Stop()
   for {
      select {
      case <-monitorChan:
         return
      case <-ticker.C:
         //
         anyStateChanged := false
         pdCoord.nodesMutex.RLock()
         removingNodes := make(map[string]string)
         for nid, removeState := range pdCoord.removingNodes {
            removingNodes[nid] = removeState
         }
         pdCoord.nodesMutex.RUnlock()
         // remove state: marked -> pending -> data_transfered -> done
         if len(removingNodes) == 0 {
            continue
         }
         currentNodes := pdCoord.getCurrentNodes(nil)
         nodeNameList := make([]string, 0, len(currentNodes))
         for _, s := range currentNodes {
            nodeNameList = append(nodeNameList, s.ID)
         }

         allNamespaces, _, err := pdCoord.register.GetAllNamespaces()
         if err != nil {
            continue
         }
         for nid, _ := range removingNodes {
            anyPending := false
            cluster.CoordLog().Infof("handle removing node %v ", nid)
            // only check the namespace with one replica left
            // because the doCheckNamespaces will check the others
            // we add a new replica for the removing node
            for _, namespacePartList := range allNamespaces {
               for _, namespaceInfo := range namespacePartList {
                  if cluster.FindSlice(namespaceInfo.RaftNodes, nid) == -1 {
                     continue
                  }
                  if len(namespaceInfo.GetISR()) <= namespaceInfo.Replica {
                     anyPending = true
                     // find new catchup and wait isr ready
                     removingNodes[nid] = "pending"
                     newInfo, err := pdCoord.dpm.addNodeToNamespaceAndWaitReady(monitorChan, &namespaceInfo,
                        nodeNameList)
                     if err != nil {
                        cluster.CoordLog().Infof("namespace %v data on node %v transfered failed, waiting next time", namespaceInfo.GetDesp(), nid)
                        continue
                     } else if newInfo != nil {
                        namespaceInfo = *newInfo
                     }
                     cluster.CoordLog().Infof("namespace %v data on node %v transfered success", namespaceInfo.GetDesp(), nid)
                     anyStateChanged = true
                  }
                  err := pdCoord.removeNamespaceFromNode(&namespaceInfo, nid)
                  if err != nil {
                     anyPending = true
                  }
               }
               if !anyPending {
                  anyStateChanged = true
                  cluster.CoordLog().Infof("node %v data has been transfered, it can be removed from cluster: state: %v", nid, removingNodes[nid])
                  if removingNodes[nid] != "data_transfered" && removingNodes[nid] != "done" {
                     removingNodes[nid] = "data_transfered"
                  } else {
                     if removingNodes[nid] == "data_transfered" {
                        removingNodes[nid] = "done"
                     } else if removingNodes[nid] == "done" {
                        pdCoord.nodesMutex.Lock()
                        _, ok := pdCoord.dataNodes[nid]
                        if !ok {
                           delete(removingNodes, nid)
                           cluster.CoordLog().Infof("the node %v is removed finally since not alive in cluster", nid)
                        }
                        pdCoord.nodesMutex.Unlock()
                     }
                  }
               }
            }
         }

         if anyStateChanged {
            pdCoord.nodesMutex.Lock()
            pdCoord.removingNodes = removingNodes
            pdCoord.nodesMutex.Unlock()
         }
      }
   }
}
```

#### 启动HTTP

 pdserver/http.go:626

```go
func (s *Server) serveHttpAPI(addr string, stopC <-chan struct{}) {
   if s.conf.ProfilePort != "" {
      go http.ListenAndServe(":"+s.conf.ProfilePort, nil)
   }
   s.initHttpHandler()
   srv := http.Server{
      Addr:    addr,
      Handler: s,
   }
   l, err := common.NewStoppableListener(srv.Addr, stopC)
   if err != nil {
      panic(err)
   }
   err = srv.Serve(l)
   sLog.Infof("http server stopped: %v", err)
}
```

## PdServer

### 创建Namespace

 pdserver/http.go:312

```go
func (s *Server) doCreateNamespace(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
   reqParams, err := url.ParseQuery(req.URL.RawQuery)
   if err != nil {
      return nil, common.HttpErr{Code: 400, Text: "INVALID_REQUEST"}
   }

   ns := reqParams.Get("namespace") //获取
   if ns == "" {
      return nil, common.HttpErr{Code: 400, Text: "MISSING_ARG_NAMESPACE"}
   }

   if !common.IsValidNamespaceName(ns) {
      return nil, common.HttpErr{Code: 400, Text: "INVALID_ARG_NAMESPACE"}
   }
   engType := reqParams.Get("engtype")
   if engType == "" {
      engType = "rockredis"
   }

   pnumStr := reqParams.Get("partition_num")
   if pnumStr == "" {
      return nil, common.HttpErr{Code: 400, Text: "MISSING_ARG_PARTITION_NUM"}
   }
   pnum, err := GetValidPartitionNum(pnumStr)
   if err != nil {
      return nil, common.HttpErr{Code: 400, Text: "INVALID_ARG_PARTITION_NUM"}
   }
   replicatorStr := reqParams.Get("replicator")
   if replicatorStr == "" {
      return nil, common.HttpErr{Code: 400, Text: "MISSING_ARG_REPLICATOR"}
   }
   replicator, err := GetValidReplicator(replicatorStr)
   if err != nil {
      return nil, common.HttpErr{Code: 400, Text: "INVALID_ARG_REPLICATOR"}
   }

   expPolicy := reqParams.Get("expiration_policy")
   if expPolicy == "" {
      expPolicy = common.DefaultExpirationPolicy
   } else if _, err := common.StringToExpirationPolicy(expPolicy); err != nil {
      return nil, common.HttpErr{Code: 400, Text: "INVALID_ARG_EXPIRATION_POLICY"}
   }

   tagStr := reqParams.Get("tags")
   var tagList []string
   if tagStr != "" {
      tagList = strings.Split(tagStr, ",")
   }

   if !s.pdCoord.IsMineLeader() {
      return nil, common.HttpErr{Code: 400, Text: cluster.ErrFailedOnNotLeader}
   }
   var meta cluster.NamespaceMetaInfo
   meta.PartitionNum = pnum
   meta.Replica = replicator
   meta.EngType = engType
   meta.ExpirationPolicy = expPolicy
   meta.Tags = make(map[string]bool)
   for _, tag := range tagList {
      if strings.TrimSpace(tag) != "" {
         meta.Tags[strings.TrimSpace(tag)] = true
      }
   }

   err = s.pdCoord.CreateNamespace(ns, meta)
   if err != nil {
      sLog.Infof("create namespace failed: %v, %v", ns, err)
      return nil, common.HttpErr{Code: 500, Text: err.Error()}
   }
   return nil, nil
}
```

## PDCoordinator

创建NameSp

cluster/pdnode_coord/pd_api.go:

```go
func (pdCoord *PDCoordinator) CreateNamespace(namespace string, meta cluster.NamespaceMetaInfo) error {
    //处理请求的PDServer为Leader
   if pdCoord.leaderNode.GetID() != pdCoord.myNode.GetID() { 
      cluster.CoordLog().Infof("not leader while create namespace")
      return ErrNotLeader
   }
	//namespace名称是否合法
   if !common.IsValidNamespaceName(namespace) {
      return errors.New("invalid namespace name")
   }
	//分区数不能太多
   if meta.PartitionNum >= common.MAX_PARTITION_NUM {
      return errors.New("max partition allowed exceed")
   }
  
  //获取当前节点
   currentNodes := pdCoord.getCurrentNodes(meta.Tags)
   if len(currentNodes) < meta.Replica { //节点数小于副本数
      cluster.CoordLog().Infof("nodes %v is less than replica %v", len(currentNodes), meta)
      return ErrNodeUnavailable.ToErrorType()
   }
   
   if ok, _ := pdCoord.register.IsExistNamespace(namespace); !ok {
     //namespace不存在
      meta.MagicCode = time.Now().UnixNano()
      var err error
      meta.MinGID, err = pdCoord.register.PrepareNamespaceMinGID()
      if err != nil {
         cluster.CoordLog().Infof("prepare namespace %v gid failed :%v", namespace, err)
         return err
      }
     //在etcd创建namespace目录；存储节点监听namespace目录的变化。当监听到有分区创建时，开始为分区选择Leaer
      err = pdCoord.register.CreateNamespace(namespace, &meta)
      if err != nil {
         cluster.CoordLog().Infof("create namespace key %v failed :%v", namespace, err)
         return err
      }
   } else {
      cluster.CoordLog().Warningf("namespace already exist :%v ", namespace)
      return ErrAlreadyExist
   }
   cluster.CoordLog().Infof("create namespace: %v, with meta: %v", namespace, meta)
  // 为每个分区分配节点
   return pdCoord.checkAndUpdateNamespacePartitions(currentNodes, namespace, meta)
}
```

```go
func (pdCoord *PDCoordinator) checkAndUpdateNamespacePartitions(currentNodes map[string]cluster.NodeInfo,
   namespace string, meta cluster.NamespaceMetaInfo) error {
   existPart := make(map[int]*cluster.PartitionMetaInfo)
   for i := 0; i < meta.PartitionNum; i++ {
      err := pdCoord.register.CreateNamespacePartition(namespace, i)
      if err != nil {
         cluster.CoordLog().Warningf("failed to create namespace %v-%v: %v", namespace, i, err)
         // handle already exist
         t, err := pdCoord.register.GetNamespacePartInfo(namespace, i)
         if err != nil {
            cluster.CoordLog().Warningf("exist namespace partition failed to get info: %v", err)
            if err != cluster.ErrKeyNotFound {
               return err
            }
         } else {
            cluster.CoordLog().Infof("create namespace partition already exist %v-%v", namespace, i)
            existPart[i] = t
         }
      }
   }
   partReplicaList, err := pdCoord.dpm.allocNamespaceRaftNodes(namespace, currentNodes, meta.Replica, meta.PartitionNum, existPart)
   if err != nil {
      cluster.CoordLog().Infof("failed to alloc nodes for namespace: %v", err)
      return err.ToErrorType()
   }
   if len(partReplicaList) != meta.PartitionNum {
      return ErrNodeUnavailable.ToErrorType()
   }
   for i := 0; i < meta.PartitionNum; i++ {
      if _, ok := existPart[i]; ok {
         continue
      }

      tmpReplicaInfo := partReplicaList[i]
      if len(tmpReplicaInfo.GetISR()) <= meta.Replica/2 {
         cluster.CoordLog().Infof("failed update info for namespace : %v-%v since not quorum", namespace, i, tmpReplicaInfo)
         continue
      }
      commonErr := pdCoord.register.UpdateNamespacePartReplicaInfo(namespace, i, &tmpReplicaInfo, tmpReplicaInfo.Epoch())
      if commonErr != nil {
         cluster.CoordLog().Infof("failed update info for namespace : %v-%v, %v", namespace, i, commonErr)
         continue
      }
   }
   pdCoord.triggerCheckNamespaces("", 0, time.Millisecond*500)
   return nil
}
```

## DataPlacement

```go
func (dp *DataPlacement) DoBalance(monitorChan chan struct{}) { //重平衡
   //check period for the data balance.
   ticker := time.NewTicker(time.Minute * 10)
   defer func() {
      ticker.Stop()
      cluster.CoordLog().Infof("balance check exit.")
   }()
   for {
      select {
      case <-monitorChan:
         return
      case <-ticker.C:
         // only balance at given interval
         if time.Now().Hour() > dp.balanceInterval[1] || time.Now().Hour() < dp.balanceInterval[0] {
            continue
         }
         if !dp.pdCoord.IsMineLeader() {
            cluster.CoordLog().Infof("not leader while checking balance")
            continue
         }
         if !dp.pdCoord.IsClusterStable() {
            cluster.CoordLog().Infof("no balance since cluster is not stable while checking balance")
            continue
         }
         if !dp.pdCoord.autoBalance {
            continue
         }
         cluster.CoordLog().Infof("begin checking balance of namespace data...")
         currentNodes := dp.pdCoord.getCurrentNodes(nil)
         validNum := len(currentNodes)
         if validNum < 2 {
            continue
         }
         dp.rebalanceNamespace(monitorChan)
      }
   }
}
```

```go
func (dp *DataPlacement) rebalanceNamespace(monitorChan chan struct{}) (bool, bool) {
   moved := false
   isAllBalanced := false
  //上次的平衡还未处理完成
   if !atomic.CompareAndSwapInt32(&dp.pdCoord.balanceWaiting, 0, 1) {
      cluster.CoordLog().Infof("another balance is running, should wait")
      return moved, isAllBalanced
   }
   defer atomic.StoreInt32(&dp.pdCoord.balanceWaiting, 0)

  //从etcd获取所有的namespace
   allNamespaces, _, err := dp.pdCoord.register.GetAllNamespaces()
   if err != nil {
      cluster.CoordLog().Infof("scan namespaces error: %v", err)
      return moved, isAllBalanced
   }
   namespaceList := make([]cluster.PartitionMetaInfo, 0)
   for _, parts := range allNamespaces {
      for _, p := range parts { //PartitionMetaInfo
         namespaceList = append(namespaceList, p)
      }
   }
   nodeNameList := make([]string, 0)
   movedNamespace := ""
   isAllBalanced = true
   for _, namespaceInfo := range namespaceList { //PartitionMetaInfo
      select {
      case <-monitorChan:
         return moved, false
      default:
      }
      if !dp.pdCoord.IsClusterStable() {
         return moved, false
      }
      if !dp.pdCoord.IsMineLeader() {
         return moved, true
      }
      // balance only one namespace once
      if movedNamespace != "" && movedNamespace != namespaceInfo.Name {
         continue
      }
      if len(namespaceInfo.Removings) > 0 {
         continue
      }
      currentNodes := dp.pdCoord.getCurrentNodes(namespaceInfo.Tags)
      nodeNameList = nodeNameList[0:0]
      for _, n := range currentNodes {
         nodeNameList = append(nodeNameList, n.ID)
      }

      partitionNodes, err := dp.getRebalancedNamespacePartitions(
         namespaceInfo.Name,
         namespaceInfo.PartitionNum,
         namespaceInfo.Replica, currentNodes)
      if err != nil {
         isAllBalanced = false
         continue
      }
      moveNodes := make([]string, 0)
      for _, nid := range namespaceInfo.GetISR() {
         found := false
         for _, expectedNode := range partitionNodes[namespaceInfo.Partition] {
            if nid == expectedNode {
               found = true
               break
            }
         }
         if !found {
            moveNodes = append(moveNodes, nid)
         }
      }
      for _, nid := range moveNodes {
         movedNamespace = namespaceInfo.Name
         cluster.CoordLog().Infof("node %v need move for namespace %v since %v not in expected isr list: %v", nid,
            namespaceInfo.GetDesp(), namespaceInfo.RaftNodes, partitionNodes[namespaceInfo.Partition])
         var err error
         var newInfo *cluster.PartitionMetaInfo
         if len(namespaceInfo.GetISR()) <= namespaceInfo.Replica {
            newInfo, err = dp.addNodeToNamespaceAndWaitReady(monitorChan, &namespaceInfo,
               nodeNameList)
         }
         if err != nil {
            return moved, false
         }
         if newInfo != nil {
            namespaceInfo = *newInfo
         }
         coordErr := dp.pdCoord.removeNamespaceFromNode(&namespaceInfo, nid)
         moved = true
         if coordErr != nil {
            return moved, false
         }
      }
      expectLeader := partitionNodes[namespaceInfo.Partition][0]
      if _, ok := namespaceInfo.Removings[expectLeader]; ok {
         cluster.CoordLog().Infof("namespace %v expected leader: %v is marked as removing", namespaceInfo.GetDesp(),
            expectLeader)
      } else {
         isrList := namespaceInfo.GetISR()
         if len(moveNodes) == 0 && (len(isrList) >= namespaceInfo.Replica) &&
            (namespaceInfo.RaftNodes[0] != expectLeader) {
            for index, nid := range namespaceInfo.RaftNodes {
               if nid == expectLeader {
                  cluster.CoordLog().Infof("need move leader for namespace %v since %v not expected leader: %v",
                     namespaceInfo.GetDesp(), namespaceInfo.RaftNodes, expectLeader)
                  namespaceInfo.RaftNodes[0], namespaceInfo.RaftNodes[index] = namespaceInfo.RaftNodes[index], namespaceInfo.RaftNodes[0]
                  err := dp.pdCoord.register.UpdateNamespacePartReplicaInfo(namespaceInfo.Name, namespaceInfo.Partition,
                     &namespaceInfo.PartitionReplicaInfo, namespaceInfo.PartitionReplicaInfo.Epoch())
                  moved = true
                  if err != nil {
                     cluster.CoordLog().Infof("move leader for namespace %v failed: %v", namespaceInfo.GetDesp(), err)
                     return moved, false
                  }
               }
            }
            select {
            case <-monitorChan:
            case <-time.After(time.Second):
            }
         }
      }
      if len(moveNodes) > 0 || moved {
         isAllBalanced = false
      }
   }

   return moved, isAllBalanced
}
```



# KVServer

## 启动流程

 apps/zankv/main.go:

```go
func (p *program) Start() error {
   flagSet.Parse(os.Args[1:])

   fmt.Println(common.VerString("ZanRedisDB"))
   if *showVersion {
      os.Exit(0)
   }
  //读取配置文件，创建ConfigFile
   var configFile server.ConfigFile
   configDir := filepath.Dir(*configFilePath)
   if *configFilePath != "" {
      d, err := ioutil.ReadFile(*configFilePath)
      if err != nil {
         panic(err)
      }
      err = json.Unmarshal(d, &configFile)
      if err != nil {
         panic(err)
      }
   }
   if configFile.ServerConf.DataDir == "" {
      tmpDir, err := ioutil.TempDir("", fmt.Sprintf("rocksdb-test-%d", time.Now().UnixNano()))
      if err != nil {
         panic(err)
      }
      configFile.ServerConf.DataDir = tmpDir
   }

   serverConf := configFile.ServerConf

   loadConf, _ := json.MarshalIndent(configFile, "", " ")
   fmt.Printf("loading with conf:%v\n", string(loadConf))
   bip := server.GetIPv4ForInterfaceName(serverConf.BroadcastInterface)
   if bip == "" || bip == "0.0.0.0" {
      panic("broadcast ip can not be found")
   } else {
      serverConf.BroadcastAddr = bip
   }
   fmt.Printf("broadcast ip is :%v\n", bip)
  //创建KVserver
   app := server.NewServer(serverConf)
  //遍历namespace，创建并初始化namespace
   for _, nsNodeConf := range serverConf.Namespaces {
      nsFile := path.Join(configDir, nsNodeConf.Name)
      d, err := ioutil.ReadFile(nsFile)
      if err != nil {
         panic(err)
      }
      var nsConf node.NamespaceConfig
      err = json.Unmarshal(d, &nsConf)
      if err != nil {
         panic(err)
      }
      if nsConf.Name != nsNodeConf.Name {
         panic("namespace name not match the config file")
      }
      if nsConf.Replicator <= 0 {
         panic("namespace replicator should be set")
      }

      id := nsNodeConf.LocalReplicaID
      clusterNodes := make(map[uint64]node.ReplicaInfo)
      for _, v := range nsConf.RaftGroupConf.SeedNodes {
         clusterNodes[v.ReplicaID] = v
      }
     //初始化
      app.InitKVNamespace(id, &nsConf, false)
   }
   app.Start()
   p.server = app
   return nil
}
```

### 创建KVServer

 server/server.go

```go
func NewServer(conf ServerConfig) *Server {
   hname, err := os.Hostname()
   if err != nil {
      sLog.Fatal(err)
   }
   if conf.TickMs < 100 {
      conf.TickMs = 100
   }
   if conf.ElectionTick < 5 {
      conf.ElectionTick = 5
   }
   if conf.MaxScanJob <= 0 {
      conf.MaxScanJob = common.MAX_SCAN_JOB
   }
   if conf.ProfilePort == 0 {
      conf.ProfilePort = 7666
   }
   myNode := &cluster.NodeInfo{
      NodeIP:      conf.BroadcastAddr,
      Hostname:    hname,
      RedisPort:   strconv.Itoa(conf.RedisAPIPort),
      HttpPort:    strconv.Itoa(conf.HttpAPIPort),
      Version:     common.VerBinary,
      Tags:        make(map[string]bool),
      DataRoot:    conf.DataDir,
      RsyncModule: "zanredisdb",
   }
   if conf.DataRsyncModule != "" {
      myNode.RsyncModule = conf.DataRsyncModule
   }

   if conf.ClusterID == "" {
      sLog.Fatalf("cluster id can not be empty")
   }
   if conf.BroadcastInterface != "" {
      myNode.NodeIP = common.GetIPv4ForInterfaceName(conf.BroadcastInterface)
   }
   if myNode.NodeIP == "" {
      myNode.NodeIP = conf.BroadcastAddr
   } else {
      conf.BroadcastAddr = myNode.NodeIP
   }
   if myNode.NodeIP == "0.0.0.0" || myNode.NodeIP == "" {
      sLog.Fatalf("can not decide the broadcast ip: %v", myNode.NodeIP)
   }
   conf.LocalRaftAddr = strings.Replace(conf.LocalRaftAddr, "0.0.0.0", myNode.NodeIP, 1)
   myNode.RaftTransportAddr = conf.LocalRaftAddr
   for _, tag := range conf.Tags {
      myNode.Tags[tag] = true
   }
   os.MkdirAll(conf.DataDir, common.DIR_PERM)
	//创建server
   s := &Server{
      conf:          conf,
      stopC:         make(chan struct{}),
      raftHttpDoneC: make(chan struct{}),
      startTime:     time.Now(),
      maxScanJob:    conf.MaxScanJob,
   }

   ts := &stats.TransportStats{}
   ts.Initialize()
  //创建raft网络通信
   s.raftTransport = &rafthttp.Transport{
      DialTimeout: time.Second * 5,
      ClusterID:   conf.ClusterID,
      Raft:        s,
      Snapshotter: s,
      TrStats:     ts,
      PeersStats:  stats.NewPeersStats(),
      ErrorC:      nil,
   }
   mconf := &node.MachineConfig{
      BroadcastAddr: conf.BroadcastAddr,
      HttpAPIPort:   conf.HttpAPIPort,
      LocalRaftAddr: conf.LocalRaftAddr,
      DataRootDir:   conf.DataDir,
      TickMs:        conf.TickMs,
      ElectionTick:  conf.ElectionTick,
      RocksDBOpts:   conf.RocksDBOpts,
   }
  //创建NamespaceMgr
   s.nsMgr = node.NewNamespaceMgr(s.raftTransport, mconf)
   myNode.RegID = mconf.NodeID

   if conf.EtcdClusterAddresses != "" {
      r := cluster.NewDNEtcdRegister(conf.EtcdClusterAddresses)
      s.dataCoord = datanode_coord.NewDataCoordinator(conf.ClusterID, myNode, s.nsMgr)
      if err := s.dataCoord.SetRegister(r); err != nil {
         sLog.Fatalf("failed to init register for coordinator: %v", err)
      }
      s.raftTransport.ID = types.ID(s.dataCoord.GetMyRegID())
      s.nsMgr.SetClusterInfoInterface(s.dataCoord)
   } else {
      s.raftTransport.ID = types.ID(myNode.RegID)
   }

   return s
}
```

NewNamespaceMgr

 node/namespace.go

```go
func NewNamespaceMgr(transport *rafthttp.Transport, conf *MachineConfig) *NamespaceMgr {
   ns := &NamespaceMgr{
      kvNodes:       make(map[string]*NamespaceNode),
      groups:        make(map[uint64]string),
      nsMetas:       make(map[string]NamespaceMeta),
      raftTransport: transport,
      machineConf:   conf,
      newLeaderChan: make(chan string, 2048),
      stopC:         make(chan struct{}),
   }
   regID, err := ns.LoadMachineRegID()
   if err != nil {
      nodeLog.Infof("load my register node id failed: %v", err)
   } else if regID > 0 {
      ns.machineConf.NodeID = regID
   }
   return ns
}
```

### 初始化Namespace

```go
func (s *Server) InitKVNamespace(id uint64, conf *node.NamespaceConfig, join bool) (*node.NamespaceNode, error) {
   return s.nsMgr.InitNamespaceNode(conf, id, join)
}
```

### 启动KVServer

 server/server.go

```go
func (s *Server) Start() {
   s.raftTransport.Start()
   s.wg.Add(1)
   go func() {
      defer s.wg.Done()
      s.serveRaft()
   }()

   if s.dataCoord != nil {
      err := s.dataCoord.Start()
      if err != nil {
         sLog.Fatalf("data coordinator start failed: %v", err)
      }
   } else {
      s.nsMgr.Start()
   }

   // api server should disable the api request while starting until replay log finished and
   // also while we recovery we need to disable api.
   s.wg.Add(2)
   go func() {
      defer s.wg.Done()
      s.serveRedisAPI(s.conf.RedisAPIPort, s.stopC)
   }()
   go func() {
      defer s.wg.Done()
      s.serveHttpAPI(s.conf.HttpAPIPort, s.stopC)
   }()
}
```

## NamespaceMgr

负责管理所有的Namespace

 node/namespace.go

```go
func (nsm *NamespaceMgr) InitNamespaceNode(conf *NamespaceConfig, raftID uint64, join bool) (*NamespaceNode, error) {
   if atomic.LoadInt32(&nsm.stopping) == 1 {
      return nil, errStopping
   }

   expPolicy, err := common.StringToExpirationPolicy(conf.ExpirationPolicy)
   if err != nil {
      nodeLog.Infof("namespace %v invalid expire policy : %v", conf.Name, conf.ExpirationPolicy)
      return nil, err
   }

   nsm.mutex.Lock()
   defer nsm.mutex.Unlock()
   if n, ok := nsm.kvNodes[conf.Name]; ok {
      return n, ErrNamespaceAlreadyExist
   }

   kvOpts := &KVOptions{
      DataDir:          path.Join(nsm.machineConf.DataRootDir, conf.Name),
      EngType:          conf.EngType,
      RockOpts:         nsm.machineConf.RocksDBOpts,
      ExpirationPolicy: expPolicy,
   }
   rockredis.FillDefaultOptions(&kvOpts.RockOpts)

   if conf.PartitionNum <= 0 {
      return nil, errNamespaceConfInvalid
   }
   if conf.Replicator <= 0 {
      return nil, errNamespaceConfInvalid
   }
   clusterNodes := make(map[uint64]ReplicaInfo)
   for _, v := range conf.RaftGroupConf.SeedNodes {
      clusterNodes[v.ReplicaID] = v
   }
   _, ok := clusterNodes[uint64(raftID)]
   if !ok {
      join = true
   }

   d, _ := json.MarshalIndent(&conf, "", " ")
   nodeLog.Infof("namespace load config: %v", string(d))
   d, _ = json.MarshalIndent(&kvOpts, "", " ")
   nodeLog.Infof("namespace kv config: %v", string(d))
   nodeLog.Infof("local namespace node %v start with raft cluster: %v", raftID, clusterNodes)
   raftConf := &RaftConfig{
      GroupID:        conf.RaftGroupConf.GroupID,
      GroupName:      conf.Name,
      ID:             uint64(raftID),
      RaftAddr:       nsm.machineConf.LocalRaftAddr,
      DataDir:        kvOpts.DataDir,
      RaftPeers:      clusterNodes,
      SnapCount:      conf.SnapCount,
      SnapCatchup:    conf.SnapCatchup,
      Replicator:     conf.Replicator,
      OptimizedFsync: conf.OptimizedFsync,
   }
   kv, err := NewKVNode(kvOpts, nsm.machineConf, raftConf, nsm.raftTransport,
      join, nsm.onNamespaceDeleted(raftConf.GroupID, conf.Name),
      nsm.clusterInfo, nsm.newLeaderChan)
   if err != nil {
      return nil, err
   }
   if _, ok := nsm.nsMetas[conf.BaseName]; !ok {
      nsm.nsMetas[conf.BaseName] = NamespaceMeta{
         PartitionNum: conf.PartitionNum,
      }
   }

   n := &NamespaceNode{
      Node: kv,
      conf: conf,
   }

   nsm.kvNodes[conf.Name] = n
   nsm.groups[raftConf.GroupID] = conf.Name
   return n, nil
}
```

# References

https://tech.youzan.com/shi-yong-kai-yuan-ji-zhu-gou-jian-you-zan-fen-bu-shi-kvcun-chu-fu-wu/

