## NewRaft

```go
func NewRaft(conf *Config, fsm FSM, logs LogStore, stable StableStore, snaps SnapshotStore, trans Transport) (*Raft, error) {
   // Validate the configuration.
   if err := ValidateConfig(conf); err != nil {
      return nil, err
   }

   // Ensure we have a LogOutput.
   var logger *log.Logger
   if conf.Logger != nil {
      logger = conf.Logger
   } else {
      if conf.LogOutput == nil {
         conf.LogOutput = os.Stderr
      }
      logger = log.New(conf.LogOutput, "", log.LstdFlags)
   }

   // Try to restore the current term.
   currentTerm, err := stable.GetUint64(keyCurrentTerm)
   if err != nil && err.Error() != "not found" {
      return nil, fmt.Errorf("failed to load current term: %v", err)
   }

   // Read the index of the last log entry.
   lastIndex, err := logs.LastIndex()
   if err != nil {
      return nil, fmt.Errorf("failed to find last log: %v", err)
   }

   // Get the last log entry.
   var lastLog Log
   if lastIndex > 0 {
      if err = logs.GetLog(lastIndex, &lastLog); err != nil {
         return nil, fmt.Errorf("failed to get last log at index %d: %v", lastIndex, err)
      }
   }

   // Make sure we have a valid server address and ID.
   protocolVersion := conf.ProtocolVersion
   localAddr := ServerAddress(trans.LocalAddr())
   localID := conf.LocalID

   // TODO (slackpad) - When we deprecate protocol version 2, remove this
   // along with the AddPeer() and RemovePeer() APIs.
   if protocolVersion < 3 && string(localID) != string(localAddr) {
      return nil, fmt.Errorf("when running with ProtocolVersion < 3, LocalID must be set to the network address")
   }

   // Create Raft struct.
   r := &Raft{
      protocolVersion: protocolVersion,
      applyCh:         make(chan *logFuture),
      conf:            *conf,
      fsm:             fsm,
      fsmMutateCh:     make(chan interface{}, 128),
      fsmSnapshotCh:   make(chan *reqSnapshotFuture),
      leaderCh:        make(chan bool),
      localID:         localID,
      localAddr:       localAddr,
      logger:          logger,
      logs:            logs,
      configurationChangeCh: make(chan *configurationChangeFuture),
      configurations:        configurations{},
      rpcCh:                 trans.Consumer(),
      snapshots:             snaps,
      userSnapshotCh:        make(chan *userSnapshotFuture),
      userRestoreCh:         make(chan *userRestoreFuture),
      shutdownCh:            make(chan struct{}),
      stable:                stable,
      trans:                 trans,
      verifyCh:              make(chan *verifyFuture, 64),
      configurationsCh:      make(chan *configurationsFuture, 8),
      bootstrapCh:           make(chan *bootstrapFuture),
      observers:             make(map[uint64]*Observer),
   }

   // 初始状态为follower.
   r.setState(Follower)

   // Start as leader if specified. This should only be used
   // for testing purposes.
   if conf.StartAsLeader { //是否自动设置为Leader
      r.setState(Leader)
      r.setLeader(r.localAddr)
   }

   // 最新的term
   r.setCurrentTerm(currentTerm)
  // 最新的日志（index、term）
   r.setLastLog(lastLog.Index, lastLog.Term)

   // 加载快照
   if err := r.restoreSnapshot(); err != nil {
      return nil, err
   }

   //获取快照中最新日志的index
   snapshotIndex, _ := r.getLastSnapshot()
  	//从文件加载日志
   for index := snapshotIndex + 1; index <= lastLog.Index; index++ {
      var entry Log
      if err := r.logs.GetLog(index, &entry); err != nil {
         r.logger.Printf("[ERR] raft: Failed to get log at %d: %v", index, err)
         panic(err)
      }
     	//处理配置日志
      r.processConfigurationLogEntry(&entry)
   }

   r.logger.Printf("[INFO] raft: Initial configuration (index=%d): %+v",
      r.configurations.latestIndex, r.configurations.latest.Servers)

   //设置处理心跳请求的处理器
   trans.SetHeartbeatHandler(r.processHeartbeat)

   // 启动后台工作
   r.goFunc(r.run) //监控状态的变更
   r.goFunc(r.runFSM) //应用已提交日志到状态机
   r.goFunc(r.runSnapshots) //生成快照
   return r, nil
}
```

```go
func (r *raftState) goFunc(f func()) {
   r.routinesGroup.Add(1)	//开启的协程数加1
   go func() {
      defer r.routinesGroup.Done() //协程数减1
      f()
   }()
}
```

```go
func (r *Raft) run() { //状态的变更
   for {
     
      select {
      case <-r.shutdownCh:	//接收到关闭的标志
         r.setLeader("")
         return
      default:
      }

      // Enter into a sub-FSM
      switch r.getState() {
      case Follower: //初始状态
         r.runFollower()
      case Candidate:
         r.runCandidate()
      case Leader:
         r.runLeader()
      }
   }
}
```

## runFollower

```go
func (r *Raft) runFollower() {
   didWarn := false
   r.logger.Info("entering followe    r state", "follower", r, "leader", r.Leader())
   metrics.IncrCounter([]string{"raft", "state", "follower"}, 1)
   //心跳时间，超过此时间出发新一轮的选举。防止同时多个Follower同时选举，一般会在HeartbeatTimeout的基础上加上随机值
   heartbeatTimer := randomTimeout(r.conf.HeartbeatTimeout)

   for r.getState() == Follower {
      select {
      case rpc := <-r.rpcCh: //客户请求
         r.processRPC(rpc) //处理rpc请求

      case c := <-r.configurationChangeCh: //配置变更
         // 非Leader拒绝此操作
         c.respond(ErrNotLeader)	

      case a := <-r.applyCh:
         // Reject any operations since we are not the leader
         a.respond(ErrNotLeader)

      case v := <-r.verifyCh:
         // Reject any operations since we are not the leader
         v.respond(ErrNotLeader)

      case r := <-r.userRestoreCh:
         // Reject any restores since we are not the leader
         r.respond(ErrNotLeader)

      case r := <-r.leadershipTransferCh:
         // 非Leader不处理Leader转让
         r.respond(ErrNotLeader)

      case c := <-r.configurationsCh:
         c.configurations = r.configurations.Clone()
         c.respond(nil)

      case b := <-r.bootstrapCh:
         b.respond(r.liveBootstrap(b.configuration))

      case <-heartbeatTimer: //触发心跳
         // Restart the heartbeat timer
         heartbeatTimer = randomTimeout(r.conf.HeartbeatTimeout)

         // Check if we have had a successful contact
         lastContact := r.LastContact()
         if time.Now().Sub(lastContact) < r.conf.HeartbeatTimeout {
            continue
         }

         //心跳失败
         lastLeader := r.Leader()
         r.setLeader("")

         if r.configurations.latestIndex == 0 {
            if !didWarn {
               r.logger.Warn("no known peers, aborting election")
               didWarn = true
            }
         } else if r.configurations.latestIndex == r.configurations.committedIndex &&
            !hasVote(r.configurations.latest, r.localID) {
            if !didWarn {
               r.logger.Warn("not part of stable configuration, aborting election")
               didWarn = true
            }
         } else {
            r.logger.Warn("heartbeat timeout reached, starting election", "last-leader", lastLeader)
            metrics.IncrCounter([]string{"raft", "transition", "heartbeat_timeout"}, 1)
            //转换为candidate状态
            r.setState(Candidate)
            return
         }

      case <-r.shutdownCh:
         return
      }
   }
}
```

## runCandidate

```go
func (r *Raft) runCandidate() {
   r.logger.Info("entering candidate state", "node", r, "term", r.getCurrentTerm()+1)
   metrics.IncrCounter([]string{"raft", "state", "candidate"}, 1)

   // 投票
   voteCh := r.electSelf()

   defer func() { r.candidateFromLeadershipTransfer = false }()
	
  //选举超时
   electionTimer := randomTimeout(r.conf.ElectionTimeout)

   // Tally the votes, need a simple majority
   grantedVotes := 0
   //集群中过半的节点数
   votesNeeded := r.quorumSize()
   r.logger.Debug("votes", "needed", votesNeeded)

   for r.getState() == Candidate {
      select {
      case rpc := <-r.rpcCh:
         r.processRPC(rpc) //处理RPC请求
      case vote := <-voteCh: //投票的响应结果
         // term大于本机节点，修改本机term，状态设置为Follower
         if vote.Term > r.getCurrentTerm() {
            r.logger.Debug("newer term discovered, fallback to follower")
            r.setState(Follower)
            r.setCurrentTerm(vote.Term)
            return
         }

         // 自己投自己的投票信息得到其他的节点的认可
         if vote.Granted {
            grantedVotes++ 
            r.logger.Debug("vote granted", "from", vote.voterID, "term", vote.Term, "tally", grantedVotes)
         }
	
         //收到集群内半数节点的投票支持
         if grantedVotes >= votesNeeded {
            r.logger.Info("election won", "tally", grantedVotes)
            r.setState(Leader) //状态修改为Leader
            r.setLeader(r.localAddr)
            return
         }

      case c := <-r.configurationChangeCh:
         // Reject any operations since we are not the leader
         c.respond(ErrNotLeader)

      case a := <-r.applyCh:
         // Reject any operations since we are not the leader
         a.respond(ErrNotLeader)

      case v := <-r.verifyCh:
         // Reject any operations since we are not the leader
         v.respond(ErrNotLeader)

      case r := <-r.userRestoreCh:
         // Reject any restores since we are not the leader
         r.respond(ErrNotLeader)

      case c := <-r.configurationsCh:
         c.configurations = r.configurations.Clone()
         c.respond(nil)

      case b := <-r.bootstrapCh:
         b.respond(ErrCantBootstrap)

      case <-electionTimer: //选举超时，重新运行runCandidate
         r.logger.Warn("Election timeout reached, restarting election")
         return

      case <-r.shutdownCh:
         return
      }
   }
}
```

### electSelf

```go
func (r *Raft) electSelf() <-chan *voteResult {
   // 创建chan，存放投票的响应结果
   respCh := make(chan *voteResult, len(r.configurations.latest.Servers))

   // 增加term
   r.setCurrentTerm(r.getCurrentTerm() + 1)

   // 最新日志的index，term
   lastIdx, lastTerm := r.getLastEntry()
  	//创建投票请求
   req := &RequestVoteRequest{
      RPCHeader:          r.getRPCHeader(),
      Term:               r.getCurrentTerm(),
      Candidate:          r.trans.EncodePeer(r.localID, r.localAddr),
      LastLogIndex:       lastIdx,
      LastLogTerm:        lastTerm,
      LeadershipTransfer: r.candidateFromLeadershipTransfer,
   }

   // 发送投票的请求函数
   askPeer := func(peer Server) {
     	//开启协程发送请求
      r.goFunc(func() {
         defer metrics.MeasureSince([]string{"raft", "candidate", "electSelf"}, time.Now())
         resp := &voteResult{voterID: peer.ID}
         err := r.trans.RequestVote(peer.ID, peer.Address, req, &resp.RequestVoteResponse)
         if err != nil {
            r.logger.Error("failed to make requestVote RPC",
               "target", peer,
               "error", err)
            resp.Term = req.Term
            resp.Granted = false
         }
        //响应结果放入chan	
         respCh <- resp
      })
   }

   // 对于集群中的每个节点发送投票请求
   for _, server := range r.configurations.latest.Servers {
      if server.Suffrage == Voter {
         if server.ID == r.localID { //本机节点
            //投票给自己
            if err := r.persistVote(req.Term, req.Candidate); err != nil 							{
               r.logger.Error("failed to persist vote", "error", err)
               return nil
            }
            // 存放自己的投票结果到chan
            respCh <- &voteResult{
               RequestVoteResponse: RequestVoteResponse{
                  RPCHeader: r.getRPCHeader(),
                  Term:      req.Term,
                  Granted:   true,
               },
               voterID: r.localID,
            }
         } else {//其他节点
            askPeer(server)
         }
      }
   }

   return respCh
}
```

## runLeader

```go
func (r *Raft) runLeader() {
   r.logger.Info("entering leader state", "leader", r)
   metrics.IncrCounter([]string{"raft", "state", "leader"}, 1)

   // Notify that we are the leader
   overrideNotifyBool(r.leaderCh, true)

   // Push to the notify channel if given
   if notify := r.conf.NotifyCh; notify != nil {
      select {
      case notify <- true:
      case <-r.shutdownCh:
      }
   }

   // setup leader state. This is only supposed to be accessed within the
   // leaderloop.
   r.setupLeaderState()

   // Cleanup state on step down
   defer func() {
      // Since we were the leader previously, we update our
      // last contact time when we step down, so that we are not
      // reporting a last contact time from before we were the
      // leader. Otherwise, to a client it would seem our data
      // is extremely stale.
      r.setLastContact()

      // Stop replication
      for _, p := range r.leaderState.replState {
         close(p.stopCh)
      }

      // Respond to all inflight operations
      for e := r.leaderState.inflight.Front(); e != nil; e = e.Next() {
         e.Value.(*logFuture).respond(ErrLeadershipLost)
      }

      // Respond to any pending verify requests
      for future := range r.leaderState.notify {
         future.respond(ErrLeadershipLost)
      }

      // Clear all the state
      r.leaderState.commitCh = nil
      r.leaderState.commitment = nil
      r.leaderState.inflight = nil
      r.leaderState.replState = nil
      r.leaderState.notify = nil
      r.leaderState.stepDown = nil

      // If we are stepping down for some reason, no known leader.
      // We may have stepped down due to an RPC call, which would
      // provide the leader, so we cannot always blank this out.
      r.leaderLock.Lock()
      if r.leader == r.localAddr {
         r.leader = ""
      }
      r.leaderLock.Unlock()

      // Notify that we are not the leader
      overrideNotifyBool(r.leaderCh, false)

      // Push to the notify channel if given
      if notify := r.conf.NotifyCh; notify != nil {
         select {
         case notify <- false:
         case <-r.shutdownCh:
            // On shutdown, make a best effort but do not block
            select {
            case notify <- false:
            default:
            }
         }
      }
   }()

   //开启数据同步
   r.startStopReplication()

   // 追加LogNoop类型的日志
   noop := &logFuture{
      log: Log{
         Type: LogNoop,
      },
   }
   r.dispatchLogs([]*logFuture{noop})

   // Sit in the leader loop until we step down
   r.leaderLoop()
}
```

### dispatchLogs

```go
func (r *Raft) dispatchLogs(applyLogs []*logFuture) {
  //被Leader调用，写日志到磁盘、标记为正在发送中、同步到其他节点
   now := time.Now()
   defer metrics.MeasureSince([]string{"raft", "leader", "dispatchLog"}, now)
	//最新的term、index
   term := r.getCurrentTerm()
   lastIndex := r.getLastIndex()

   n := len(applyLogs)
   logs := make([]*Log, n)
   metrics.SetGauge([]string{"raft", "leader", "dispatchNumLogs"}, float32(n))
		//封装成Log
   for idx, applyLog := range applyLogs {
      applyLog.dispatch = now
      lastIndex++
      applyLog.log.Index = lastIndex
      applyLog.log.Term = term
      logs[idx] = &applyLog.log
     	//需要同步的日志放入inflight
      r.leaderState.inflight.PushBack(applyLog)
   }

   // 写日志本地磁盘文件
   if err := r.logs.StoreLogs(logs); err != nil {
      r.logger.Error("failed to commit logs", "error", err)
      for _, applyLog := range applyLogs {
         applyLog.respond(err)
      }
      r.setState(Follower)
      return
   }
   r.leaderState.commitment.match(r.localID, lastIndex)

   //写入磁盘成功，更改最新的index、term
   r.setLastLog(lastIndex, term)

   //触发同步操作	
   for _, f := range r.leaderState.replState {
      asyncNotifyCh(f.triggerCh)
   }
}
```

```go
func (c *commitment) match(server ServerID, matchIndex uint64) {
  //更新每个节点同步的进度
   c.Lock()
   defer c.Unlock()
   if prev, hasVote := c.matchIndexes[server]; hasVote && matchIndex > prev {
      c.matchIndexes[server] = matchIndex //更新为最新的index
      c.recalculate()
   }
}
```

```go
func (c *commitment) recalculate() {
  //计算最新提交的index
   if len(c.matchIndexes) == 0 {
      return
   }
   //  存放各节点同步的index
   matched := make([]uint64, 0, len(c.matchIndexes))
   for _, idx := range c.matchIndexes {
      matched = append(matched, idx)
   }
  //排序
   sort.Sort(uint64Slice(matched))
  //获取列表中间位置存放的index
   quorumMatchIndex := matched[(len(matched)-1)/2]
	 //修改提交的index
   if quorumMatchIndex > c.commitIndex && quorumMatchIndex >= c.startIndex {
      c.commitIndex = quorumMatchIndex
      asyncNotifyCh(c.commitCh)//触发提交
   }
}
```

### leaderLoop

```go
func (r *Raft) leaderLoop() {
   // stepDown is used to track if there is an inflight log that
   // would cause us to lose leadership (specifically a RemovePeer of
   // ourselves). If this is the case, we must not allow any logs to
   // be processed in parallel, otherwise we are basing commit on
   // only a single peer (ourself) and replicating to an undefined set
   // of peers.
   stepDown := false
   lease := time.After(r.conf.LeaderLeaseTimeout)

   for r.getState() == Leader {
      select {
      case rpc := <-r.rpcCh:
         r.processRPC(rpc) //处理RPC请求

      case <-r.leaderState.stepDown:
         r.setState(Follower)

      case future := <-r.leadershipTransferCh://Leader转让
         if r.getLeadershipTransferInProgress() { //正在转让中
            r.logger.Debug(ErrLeadershipTransferInProgress.Error())
            future.respond(ErrLeadershipTransferInProgress)
            continue
         }

         r.logger.Debug("starting leadership transfer", "id", future.ID, "address", future.Address)

         // When we are leaving leaderLoop, we are no longer
         // leader, so we should stop transferring.
         leftLeaderLoop := make(chan struct{})
         defer func() { close(leftLeaderLoop) }()

         stopCh := make(chan struct{})
         doneCh := make(chan error, 1)

         // This is intentionally being setup outside of the
         // leadershipTransfer function. Because the TimeoutNow
         // call is blocking and there is no way to abort that
         // in case eg the timer expires.
         // The leadershipTransfer function is controlled with
         // the stopCh and doneCh.
         go func() {
            select {
            case <-time.After(r.conf.ElectionTimeout):
               close(stopCh)
               err := fmt.Errorf("leadership transfer timeout")
               r.logger.Debug(err.Error())
               future.respond(err)
               <-doneCh
            case <-leftLeaderLoop:
               close(stopCh)
               err := fmt.Errorf("lost leadership during transfer (expected)")
               r.logger.Debug(err.Error())
               future.respond(nil)
               <-doneCh
            case err := <-doneCh:
               if err != nil {
                  r.logger.Debug(err.Error())
               }
               future.respond(err)
            }
         }()

         // leaderState.replState is accessed here before
         // starting leadership transfer asynchronously because
         // leaderState is only supposed to be accessed in the
         // leaderloop.
         id := future.ID
         address := future.Address
         if id == nil { //转让的节点无效
            s := r.pickServer() //挑选节点
            if s != nil {
               id = &s.ID
               address = &s.Address
            } else {
               doneCh <- fmt.Errorf("cannot find peer")
               continue
            }
         }
        //获取转让节点的复制状态
         state, ok := r.leaderState.replState[*id]
         if !ok {
            doneCh <- fmt.Errorf("cannot find replication state for %v", id)
            continue
         }

         go r.leadershipTransfer(*id, *address, state, stopCh, doneCh)

      case <-r.leaderState.commitCh: //日志提交
         // Process the newly committed entries
         oldCommitIndex := r.getCommitIndex()
         commitIndex := r.leaderState.commitment.getCommitIndex()
         r.setCommitIndex(commitIndex)

         // New configration has been committed, set it as the committed
         // value.
         if r.configurations.latestIndex > oldCommitIndex &&
            r.configurations.latestIndex <= commitIndex {
            r.setCommittedConfiguration(r.configurations.latest, r.configurations.latestIndex)
            if !hasVote(r.configurations.committed, r.localID) {
               stepDown = true
            }
         }

         start := time.Now()
         var groupReady []*list.Element
         var groupFutures = make(map[uint64]*logFuture)
         var lastIdxInGroup uint64

         // 已提交的日志从inflight中移除
         for e := r.leaderState.inflight.Front(); e != nil; e = e.Next() 					{
            commitLog := e.Value.(*logFuture)
            idx := commitLog.log.Index
            if idx > commitIndex {
               // Don't go past the committed index
               break
            }

            // Measure the commit time
            metrics.MeasureSince([]string{"raft", "commitTime"}, commitLog.dispatch)
            groupReady = append(groupReady, e)
            groupFutures[idx] = commitLog
            lastIdxInGroup = idx
         }

         // Process the group
         if len(groupReady) != 0 {
           	//应用到状态机
            r.processLogs(lastIdxInGroup, groupFutures)
						//从inflight中移除
            for _, e := range groupReady {
               r.leaderState.inflight.Remove(e)
            }
         }

         // Measure the time to enqueue batch of logs for FSM to apply
         metrics.MeasureSince([]string{"raft", "fsm", "enqueue"}, start)

         // Count the number of logs enqueued
         metrics.SetGauge([]string{"raft", "commitNumLogs"}, float32(len(groupReady)))

         if stepDown {
            if r.conf.ShutdownOnRemove {
               r.logger.Info("removed ourself, shutting down")
               r.Shutdown()
            } else {
               r.logger.Info("removed ourself, transitioning to follower")
               r.setState(Follower)
            }
         }

      case v := <-r.verifyCh:
         if v.quorumSize == 0 {
            // Just dispatched, start the verification
            r.verifyLeader(v)

         } else if v.votes < v.quorumSize {
            // Early return, means there must be a new leader
            r.logger.Warn("new leader elected, stepping down")
            r.setState(Follower)
            delete(r.leaderState.notify, v)
            for _, repl := range r.leaderState.replState {
               repl.cleanNotify(v)
            }
            v.respond(ErrNotLeader)

         } else {
            // Quorum of members agree, we are still leader
            delete(r.leaderState.notify, v)
            for _, repl := range r.leaderState.replState {
               repl.cleanNotify(v)
            }
            v.respond(nil)
         }

      case future := <-r.userRestoreCh:
         if r.getLeadershipTransferInProgress() {
            r.logger.Debug(ErrLeadershipTransferInProgress.Error())
            future.respond(ErrLeadershipTransferInProgress)
            continue
         }
         err := r.restoreUserSnapshot(future.meta, future.reader)
         future.respond(err)

      case future := <-r.configurationsCh:
         if r.getLeadershipTransferInProgress() {
            r.logger.Debug(ErrLeadershipTransferInProgress.Error())
            future.respond(ErrLeadershipTransferInProgress)
            continue
         }
         future.configurations = r.configurations.Clone()
         future.respond(nil)

      case future := <-r.configurationChangeChIfStable():
         if r.getLeadershipTransferInProgress() {
            r.logger.Debug(ErrLeadershipTransferInProgress.Error())
            future.respond(ErrLeadershipTransferInProgress)
            continue
         }
         r.appendConfigurationEntry(future)

      case b := <-r.bootstrapCh:
         b.respond(ErrCantBootstrap)

      case newLog := <-r.applyCh:
         if r.getLeadershipTransferInProgress() {
            r.logger.Debug(ErrLeadershipTransferInProgress.Error())
            newLog.respond(ErrLeadershipTransferInProgress)
            continue
         }
         // Group commit, gather all the ready commits
         ready := []*logFuture{newLog}
      GROUP_COMMIT_LOOP:
         for i := 0; i < r.conf.MaxAppendEntries; i++ {
            select {
            case newLog := <-r.applyCh:
               ready = append(ready, newLog)
            default:
               break GROUP_COMMIT_LOOP
            }
         }

         // Dispatch the logs
         if stepDown {
            // we're in the process of stepping down as leader, don't process anything new
            for i := range ready {
               ready[i].respond(ErrNotLeader)
            }
         } else {
            r.dispatchLogs(ready)
         }

      case <-lease:
       //LeaderLeaseTimeout,Leader当选的时长，当超过此时长并且与过半的节点没有连通，状态修改为Follower
         maxDiff := r.checkLeaderLease()
				//调整检测网络连通性的间隔
         // contacted, without going negative
         checkInterval := r.conf.LeaderLeaseTimeout - maxDiff
         if checkInterval < minCheckInterval {
            checkInterval = minCheckInterval
         }

         // Renew the lease timer
         lease = time.After(checkInterval)

      case <-r.shutdownCh: //关闭
         return
      }
   }
}
```

#### leadershipTransfer

```go
func (r *Raft) leadershipTransfer(id ServerID, address ServerAddress, repl *followerReplication, stopCh chan struct{}, doneCh chan error) {

   // make sure we are not already stopped
   select {
   case <-stopCh:
      doneCh <- nil
      return
   default:
   }

   // Step 1: set this field which stops this leader from responding to any client requests.
   r.setLeadershipTransferInProgress(true)
   defer func() { r.setLeadershipTransferInProgress(false) }()

   for atomic.LoadUint64(&repl.nextIndex) <= r.getLastIndex() {
      err := &deferError{}
      err.init()
      repl.triggerDeferErrorCh <- err
      select {
      case err := <-err.errCh:
         if err != nil {
            doneCh <- err
            return
         }
      case <-stopCh:
         doneCh <- nil
         return
      }
   }

   // Step ?: the thesis describes in chap 6.4.1: Using clocks to reduce
   // messaging for read-only queries. If this is implemented, the lease
   // has to be reset as well, in case leadership is transferred. This
   // implementation also has a lease, but it serves another purpose and
   // doesn't need to be reset. The lease mechanism in our raft lib, is
   // setup in a similar way to the one in the thesis, but in practice
   // it's a timer that just tells the leader how often to check
   // heartbeats are still coming in.

   // Step 3: send TimeoutNow message to target server.
   err := r.trans.TimeoutNow(id, address, &TimeoutNowRequest{RPCHeader: r.getRPCHeader()}, &TimeoutNowResponse{})
   if err != nil {
      err = fmt.Errorf("failed to make TimeoutNow RPC to %v: %v", id, err)
   }
   doneCh <- err
}
```

#### checkLeaderLease

```go
func (r *Raft) checkLeaderLease() time.Duration {
   // 可以与Leader连通的节点数
   contacted := 0

   // 最大的连通间隔
   var maxDiff time.Duration
   now := time.Now()
   for _, server := range r.configurations.latest.Servers {
      if server.Suffrage == Voter {
         if server.ID == r.localID { //本机与自己一定连通
            contacted++
            continue
         }
         f := r.leaderState.replState[server.ID]
         //计算时间间隔
         diff := now.Sub(f.LastContact())
         if diff <= r.conf.LeaderLeaseTimeout {
           	//节点连通性良好
            contacted++
            if diff > maxDiff {
               maxDiff = diff
            }
         } else {//连通性不好
            if diff <= 3*r.conf.LeaderLeaseTimeout {
               r.logger.Warn("failed to contact", "server-id", server.ID, "time", diff)
            } else {
               r.logger.Debug("failed to contact", "server-id", server.ID, "time", diff)
            }
         }
         metrics.AddSample([]string{"raft", "leader", "lastContact"}, float32(diff/time.Millisecond))
      }
   }

   // Verify we can contact a quorum
   quorum := r.quorumSize()
   if contacted < quorum { //连通良好的节点数小于过半的节点数
      r.logger.Warn("failed to contact quorum of nodes, stepping down")
      r.setState(Follower)//转为Follower状态
      metrics.IncrCounter([]string{"raft", "transition", "leader_lease_timeout"}, 1)
   }
   return maxDiff
}
```

#### processRPC

```go
func (r *Raft) processRPC(rpc RPC) {
  	//检测请求头
   if err := r.checkRPCHeader(rpc); err != nil {
      rpc.Respond(nil, err)
      return
   }

   switch cmd := rpc.Command.(type) {
   case *AppendEntriesRequest:
      r.appendEntries(rpc, cmd)
   case *RequestVoteRequest:
      r.requestVote(rpc, cmd)
   case *InstallSnapshotRequest:
      r.installSnapshot(rpc, cmd)
   case *TimeoutNowRequest:
      r.timeoutNow(rpc, cmd)
   default:
      r.logger.Error("got unexpected command",
         "command", hclog.Fmt("%#v", rpc.Command))
      rpc.Respond(nil, fmt.Errorf("unexpected command"))
   }
}
```

##### appendEntries

```go
func (r *Raft) appendEntries(rpc RPC, a *AppendEntriesRequest) {
   defer metrics.MeasureSince([]string{"raft", "rpc", "appendEntries"}, time.Now())
   // 创建追加日志的响应
   resp := &AppendEntriesResponse{
      RPCHeader:      r.getRPCHeader(),
      Term:           r.getCurrentTerm(),
      LastLog:        r.getLastIndex(),
      Success:        false,
      NoRetryBackoff: false,
   }
   var rpcErr error
   //将请求的响应放入response channel
   defer func() {
      rpc.Respond(resp, rpcErr)
   }()
   // 小于当前节点的term，说明Leader发生了转让，Follower尚未知晓
   if a.Term < r.getCurrentTerm() {
      return
   }
  
   if a.Term > r.getCurrentTerm() || r.getState() != Follower {
      // Ensure transition to follower
      r.setState(Follower)
      r.setCurrentTerm(a.Term)
      resp.Term = a.Term
   }
   //设置leader
   r.setLeader(ServerAddress(r.trans.DecodePeer(a.Leader)))

   // 校验之前的日志是否相同
   if a.PrevLogEntry > 0 {
      lastIdx, lastTerm := r.getLastEntry()
      var prevLogTerm uint64
      if a.PrevLogEntry == lastIdx {
         prevLogTerm = lastTerm
      } else {
         var prevLog Log
         //根据index从文件总获取Log
         if err := r.logs.GetLog(a.PrevLogEntry, &prevLog); err != nil {
            r.logger.Warn("failed to get previous log",
               "previous-index", a.PrevLogEntry,
               "last-index", lastIdx,
               "error", err)
            resp.NoRetryBackoff = true
            return
         }
         prevLogTerm = prevLog.Term
      }
      //主从节点的日志项不匹配
      if a.PrevLogTerm != prevLogTerm {
         r.logger.Warn("previous log term mis-match",
            "ours", prevLogTerm,
            "remote", a.PrevLogTerm)
         resp.NoRetryBackoff = true
         return
      }
   }
   // 处理接收到的日志项
   if len(a.Entries) > 0 {
      start := time.Now()
      // Delete any conflicting entries, skip any duplicates
      lastLogIdx, _ := r.getLastLog()
      var newEntries []*Log
      for i, entry := range a.Entries {
         if entry.Index > lastLogIdx {
            newEntries = a.Entries[i:]
            break
         }
         //获取Log
         var storeEntry Log
         if err := r.logs.GetLog(entry.Index, &storeEntry); err != nil {
            r.logger.Warn("failed to get log entry",
               "index", entry.Index,
               "error", err)
            return
         }
         //term不匹配
         if entry.Term != storeEntry.Term {
            r.logger.Warn("clearing log suffix",
               "from", entry.Index,
               "to", lastLogIdx)
            //范围删除[index,lastLogIndex]
            if err := r.logs.DeleteRange(entry.Index, lastLogIdx); err != nil {
               r.logger.Error("failed to clear log suffix", "error", err)
               return
            }
            if entry.Index <= r.configurations.latestIndex {
               r.setLatestConfiguration(r.configurations.committed, r.configurations.committedIndex)
            }
            newEntries = a.Entries[i:]
            break
         }
      }

      if n := len(newEntries); n > 0 {
         // 追加新的日志项到日志文件
         if err := r.logs.StoreLogs(newEntries); err != nil {
            r.logger.Error("failed to append to logs", "error", err)
            // TODO: leaving r.getLastLog() in the wrong
            // state if there was a truncation above
            return
         }
         // Handle any new configuration changes
         for _, newEntry := range newEntries {
            r.processConfigurationLogEntry(newEntry)
         }
         //获取最后的日志项
         last := newEntries[n-1]
         r.setLastLog(last.Index, last.Term)
      }
      metrics.MeasureSince([]string{"raft", "rpc", "appendEntries", "storeLogs"}, start)
   }

  // 修改commit index(接收的是Leader的同步请求)
   if a.LeaderCommitIndex > 0 && a.LeaderCommitIndex > r.getCommitIndex() {
      start := time.Now()
      idx := min(a.LeaderCommitIndex, r.getLastIndex())
      r.setCommitIndex(idx)
      if r.configurations.latestIndex <= idx {
         r.setCommittedConfiguration(r.configurations.latest, r.configurations.latestIndex)
      }
      r.processLogs(idx, nil)
      metrics.MeasureSince([]string{"raft", "rpc", "appendEntries", "processLogs"}, start)
   }

   // Everything went well, set success
   resp.Success = true
   r.setLastContact()
   return
}
```

##### requestVote

```go
func (r *Raft) requestVote(rpc RPC, req *RequestVoteRequest) {
   defer metrics.MeasureSince([]string{"raft", "rpc", "requestVote"}, time.Now())
   r.observe(*req)
   // Setup a response
   resp := &RequestVoteResponse{
      RPCHeader: r.getRPCHeader(),
      Term:      r.getCurrentTerm(),
      Granted:   false,//默认不承认此投票请求
   }
   var rpcErr error
   defer func() {
     	//返回响应
      rpc.Respond(resp, rpcErr)
   }()

   // Version 0 servers will panic unless the peers is present. It's only
   // used on them to produce a warning message.
   if r.protocolVersion < 2 {
      resp.Peers = encodePeers(r.configurations.latest, r.trans)
   }

	
   candidate := r.trans.DecodePeer(req.Candidate)
   if leader := r.Leader(); leader != "" && leader != candidate && !req.LeadershipTransfer { //已经有Leader
      r.logger.Warn("rejecting vote request since we have a leader",
         "from", candidate,
         "leader", leader)
      return
   }
   // 小于当前节点的term，不认可此投票请求
   if req.Term < r.getCurrentTerm() {
      return
   }
   // 大于当前节点的term
   if req.Term > r.getCurrentTerm() {
      // Ensure transition to follower
      r.logger.Debug("lost leadership because received a requestVote with a newer term")
      r.setState(Follower)
      r.setCurrentTerm(req.Term)
      resp.Term = req.Term
   }

   //上次投票的term
   lastVoteTerm, err := r.stable.GetUint64(keyLastVoteTerm)
   if err != nil && err.Error() != "not found" {
      r.logger.Error("failed to get last vote term", "error", err)
      return
   }
   //上次投票通过的节点信息
   lastVoteCandBytes, err := r.stable.Get(keyLastVoteCand)
   if err != nil && err.Error() != "not found" {
      r.logger.Error("failed to get last vote candidate", "error", err)
      return
   }
   // 已经投过票
   if lastVoteTerm == req.Term && lastVoteCandBytes != nil {
      r.logger.Info("duplicate requestVote for same term", "term", req.Term)
      if bytes.Compare(lastVoteCandBytes, req.Candidate) == 0 {
         r.logger.Warn("duplicate requestVote from", "candidate", req.Candidate)
         resp.Granted = true
      }
      return
   }

   //请求的term小
   lastIdx, lastTerm := r.getLastEntry()
   if lastTerm > req.LastLogTerm {
      r.logger.Warn("rejecting vote request since our last term is greater",
         "candidate", candidate,
         "last-term", lastTerm,
         "last-candidate-term", req.LastLogTerm)
      return
   }
   //term相同
   if lastTerm == req.LastLogTerm && lastIdx > req.LastLogIndex {
      r.logger.Warn("rejecting vote request since our last index is greater",
         "candidate", candidate,
         "last-index", lastIdx,
         "last-candidate-index", req.LastLogIndex)
      return
   }
   //存储投票信息
   // Persist a vote for safety
   if err := r.persistVote(req.Term, req.Candidate); err != nil {
      r.logger.Error("failed to persist vote", "error", err)
      return
   }
   //投票通过
   resp.Granted = true
   r.setLastContact()
   return
}
```

```go
func (r *Raft) runFSM() {
   var lastIndex, lastTerm uint64

   batchingFSM, batchingEnabled := r.fsm.(BatchingFSM)
   configStore, configStoreEnabled := r.fsm.(ConfigurationStore)

   commitSingle := func(req *commitTuple) {
      // Apply the log if a command or config change
      var resp interface{}
      // Make sure we send a response
      defer func() {
         // Invoke the future if given
         if req.future != nil {
            req.future.response = resp
            req.future.respond(nil)
         }
      }()

      switch req.log.Type {
      case LogCommand:
         start := time.Now()
         resp = r.fsm.Apply(req.log)
         metrics.MeasureSince([]string{"raft", "fsm", "apply"}, start)

      case LogConfiguration:
         if !configStoreEnabled {
            // Return early to avoid incrementing the index and term for
            // an unimplemented operation.
            return
         }

         start := time.Now()
         configStore.StoreConfiguration(req.log.Index, DecodeConfiguration(req.log.Data))
         metrics.MeasureSince([]string{"raft", "fsm", "store_config"}, start)
      }

      // Update the indexes
      lastIndex = req.log.Index
      lastTerm = req.log.Term
   }

   commitBatch := func(reqs []*commitTuple) {
      if !batchingEnabled {
         for _, ct := range reqs {
            commitSingle(ct)
         }
         return
      }

      // Only send LogCommand and LogConfiguration log types. LogBarrier types
      // will not be sent to the FSM.
      shouldSend := func(l *Log) bool {
         switch l.Type {
         case LogCommand, LogConfiguration:
            return true
         }
         return false
      }

      var lastBatchIndex, lastBatchTerm uint64
      sendLogs := make([]*Log, 0, len(reqs))
      for _, req := range reqs {
         if shouldSend(req.log) {
            sendLogs = append(sendLogs, req.log)
         }
         lastBatchIndex = req.log.Index
         lastBatchTerm = req.log.Term
      }

      var responses []interface{}
      if len(sendLogs) > 0 {
         start := time.Now()
         responses = batchingFSM.ApplyBatch(sendLogs)
         metrics.MeasureSince([]string{"raft", "fsm", "applyBatch"}, start)
         metrics.AddSample([]string{"raft", "fsm", "applyBatchNum"}, float32(len(reqs)))

         // Ensure we get the expected responses
         if len(sendLogs) != len(responses) {
            panic("invalid number of responses")
         }
      }

      // Update the indexes
      lastIndex = lastBatchIndex
      lastTerm = lastBatchTerm

      var i int
      for _, req := range reqs {
         var resp interface{}
         // If the log was sent to the FSM, retrieve the response.
         if shouldSend(req.log) {
            resp = responses[i]
            i++
         }

         if req.future != nil {
            req.future.response = resp
            req.future.respond(nil)
         }
      }
   }

   restore := func(req *restoreFuture) {
      // Open the snapshot
      meta, source, err := r.snapshots.Open(req.ID)
      if err != nil {
         req.respond(fmt.Errorf("failed to open snapshot %v: %v", req.ID, err))
         return
      }

      // Attempt to restore
      start := time.Now()
      if err := r.fsm.Restore(source); err != nil {
         req.respond(fmt.Errorf("failed to restore snapshot %v: %v", req.ID, err))
         source.Close()
         return
      }
      source.Close()
      metrics.MeasureSince([]string{"raft", "fsm", "restore"}, start)

      // Update the last index and term
      lastIndex = meta.Index
      lastTerm = meta.Term
      req.respond(nil)
   }

   snapshot := func(req *reqSnapshotFuture) {
      // Is there something to snapshot?
      if lastIndex == 0 {
         req.respond(ErrNothingNewToSnapshot)
         return
      }

      // Start a snapshot
      start := time.Now()
      snap, err := r.fsm.Snapshot()
      metrics.MeasureSince([]string{"raft", "fsm", "snapshot"}, start)

      // Respond to the request
      req.index = lastIndex
      req.term = lastTerm
      req.snapshot = snap
      req.respond(err)
   }

   for {
      select {
      case ptr := <-r.fsmMutateCh:
         switch req := ptr.(type) {
         case []*commitTuple:
            commitBatch(req)

         case *restoreFuture:
            restore(req)

         default:
            panic(fmt.Errorf("bad type passed to fsmMutateCh: %#v", ptr))
         }

      case req := <-r.fsmSnapshotCh:
         snapshot(req)

      case <-r.shutdownCh:
         return
      }
   }
}
```

```go
func (r *Raft) runSnapshots() {
   for {
      select {
      case <-randomTimeout(r.conf.SnapshotInterval): //触发生成快照的间隔
         // 检查是否需要生成快照
         if !r.shouldSnapshot() {
            continue
         }

         // 满足触生成快照的条件，生成快照
         if _, err := r.takeSnapshot(); err != nil {
            r.logger.Error("failed to take snapshot", "error", err)
         }

      case future := <-r.userSnapshotCh: //客户端发送的生成快照的请求
         // 用户触发，立即触发生成快照
         id, err := r.takeSnapshot()
         if err != nil {
            r.logger.Error("failed to take snapshot", "error", err)
         } else {
            future.opener = func() (*SnapshotMeta, io.ReadCloser, error) 							{
               return r.snapshots.Open(id)
            }
         }
         future.respond(err)

      case <-r.shutdownCh: //服务端关闭
         return
      }
   }
}
```

## startStopReplication

raft.go/startStopReplication

```go
func (r *Raft) startStopReplication() {
   inConfig := make(map[ServerID]bool, len(r.configurations.latest.Servers))
   lastIdx := r.getLastIndex()

   // 开始同步
   for _, server := range r.configurations.latest.Servers {
      if server.ID == r.localID {
         continue
      }
      inConfig[server.ID] = true
     //创建followerReplication
      if _, ok := r.leaderState.replState[server.ID]; !ok {
         r.logger.Info("added peer, starting replication", "peer", server.ID)
         s := &followerReplication{
            peer:                server,
            commitment:          r.leaderState.commitment,
            stopCh:              make(chan uint64, 1),
            triggerCh:           make(chan struct{}, 1),
            triggerDeferErrorCh: make(chan *deferError, 1),
            currentTerm:         r.getCurrentTerm(),
            nextIndex:           lastIdx + 1,
            lastContact:         time.Now(),
            notify:              make(map[*verifyFuture]struct{}),
            notifyCh:            make(chan struct{}, 1),
            stepDown:            r.leaderState.stepDown,
         }
         r.leaderState.replState[server.ID] = s
         r.goFunc(func() { r.replicate(s) }) //开启协程同步
         asyncNotifyCh(s.triggerCh) //触发同步
         r.observe(PeerObservation{Peer: server, Removed: false})
      }
   }

   // 停止数据同步
   for serverID, repl := range r.leaderState.replState {
      if inConfig[serverID] {
         continue
      }
      // Replicate up to lastIdx and stop
      r.logger.Info("removed peer, stopping replication", "peer", serverID, "last-index", lastIdx)
      repl.stopCh <- lastIdx //发送停止同步的信号
      close(repl.stopCh)
      delete(r.leaderState.replState, serverID)
      r.observe(PeerObservation{Peer: repl.peer, Removed: true})
   }
}
```

replication.go/replicate

```go
func (r *Raft) replicate(s *followerReplication) {
   stopHeartbeat := make(chan struct{})
   defer close(stopHeartbeat)
   //发送心跳
   r.goFunc(func() { r.heartbeat(s, stopHeartbeat) })
	
RPC:
   shouldStop := false
   for !shouldStop {
      select {
      case maxIndex := <-s.stopCh: //停止同步
         // Make a best effort to replicate up to this index
         if maxIndex > 0 {
            r.replicateTo(s, maxIndex)
         }
         return
      case deferErr := <-s.triggerDeferErrorCh:
         lastLogIdx, _ := r.getLastLog()
         shouldStop = r.replicateTo(s, lastLogIdx)
         if !shouldStop {
            deferErr.respond(nil)
         } else {
            deferErr.respond(fmt.Errorf("replication failed"))
         } 
      case <-s.triggerCh: //触发同步（leader当选开始同步数据时或数据有新增时）
         lastLogIdx, _ := r.getLastLog() //获取最新的index
         shouldStop = r.replicateTo(s, lastLogIdx)
      case <-randomTimeout(r.conf.CommitTimeout):
         lastLogIdx, _ := r.getLastLog()
         shouldStop = r.replicateTo(s, lastLogIdx)
      }
      if !shouldStop && s.allowPipeline { //以pipeline方式同步数据，不需要一请求一响应
         goto PIPELINE
      }
   }
   return
	//以pipeline方式同步数据
PIPELINE:
   s.allowPipeline = false

  //使用pipeline方式进行复制可以获得更高的性能。这个方法不能优雅地从错误中恢复，所以我们后退返回标准模式
   if err := r.pipelineReplicate(s); err != nil {
      if err != ErrPipelineReplicationNotSupported {
         r.logger.Error("failed to start pipeline replication to", "peer", s.peer, "error", err)
      }
   }
   goto RPC
}
```

### RPC同步

replication.go

```go
func (r *Raft) replicateTo(s *followerReplication, lastIndex uint64) (shouldStop bool) {
   var req AppendEntriesRequest //同步数据请求
   var resp AppendEntriesResponse //同步数据响应
   var start time.Time
START:
   // Prevent an excessive retry rate on errors
   if s.failures > 0 {
      select {
      case <-time.After(backoff(failureWait, s.failures, maxFailureScale)):
      case <-r.shutdownCh:
      }
   }

   // 填充同步请求要同步的数据
   if err := r.setupAppendEntries(s, &req, atomic.LoadUint64(&s.nextIndex), lastIndex); err == ErrLogNotFound { 
      goto SEND_SNAP //如果根据index获取不到数据，
   } else if err != nil {
      return
   }

   //开始发送请求
   start = time.Now() //开始时间
   if err := r.trans.AppendEntries(s.peer.ID, s.peer.Address, &req, &resp); err != nil {
      r.logger.Error("failed to appendEntries to", "peer", s.peer, "error", err)
      s.failures++
      return
   }
   appendStats(string(s.peer.ID), start, float32(len(req.Entries)))

  
   if resp.Term > req.Term { //从节点的term大于leader节点的term
      r.handleStaleTerm(s)
      return true //停止同步
   }

   // Update the last contact
   s.setLastContact()


   if resp.Success { //响应成功
      // Update our replication state
      updateLastAppended(s, &req)

      // Clear any failures, allow pipelining
      s.failures = 0
      s.allowPipeline = true //允许以pipeline的方式同步数据
   } else {
      atomic.StoreUint64(&s.nextIndex, max(min(s.nextIndex-1, resp.LastLog+1), 1))
      if resp.NoRetryBackoff {
         s.failures = 0
      } else {
         s.failures++
      }
      r.logger.Warn("appendEntries rejected, sending older logs", "peer", s.peer, "next", atomic.LoadUint64(&s.nextIndex))
   }

CHECK_MORE:
   // Poll the stop channel here in case we are looping and have been asked
   // to stop, or have stepped down as leader. Even for the best effort case
   // where we are asked to replicate to a given index and then shutdown,
   // it's better to not loop in here to send lots of entries to a straggler
   // that's leaving the cluster anyways.
   select {
   case <-s.stopCh:
      return true
   default:
   }

   // Check if there are more logs to replicate
   if atomic.LoadUint64(&s.nextIndex) <= lastIndex {
      goto START
   }
   return

   // SEND_SNAP is used when we fail to get a log, usually because the follower
   // is too far behind, and we must ship a snapshot down instead
SEND_SNAP: //发送快照
   if stop, err := r.sendLatestSnapshot(s); stop {
      return true
   } else if err != nil {
      r.logger.Error("failed to send snapshot to", "peer", s.peer, "error", err)
      return
   }

   // Check if there is more to replicate
   goto CHECK_MORE
}
```

#### 填充同步请求

replication.go

```go
func (r *Raft) setupAppendEntries(s *followerReplication, req *AppendEntriesRequest, nextIndex, lastIndex uint64) error { //填充同步的数据
   req.RPCHeader = r.getRPCHeader()
   req.Term = s.currentTerm
   req.Leader = r.trans.EncodePeer(r.localID, r.localAddr)
   req.LeaderCommitIndex = r.getCommitIndex() 
   //前一条数据的index和term
   if err := r.setPreviousLog(req, nextIndex); err != nil {
      return err
   }
   //获取数据，填充请求
   if err := r.setNewLogs(req, nextIndex, lastIndex); err != nil {
      return err
   }
   return nil
}
```

#### 根据索引获取数据

```go
func (r *Raft) setNewLogs(req *AppendEntriesRequest, nextIndex, lastIndex uint64) error {
   req.Entries = make([]*Log, 0, r.conf.MaxAppendEntries) //默认批量发送64条
   maxIndex := min(nextIndex+uint64(r.conf.MaxAppendEntries)-1, lastIndex)
   for i := nextIndex; i <= maxIndex; i++ {
      oldLog := new(Log)
      if err := r.logs.GetLog(i, oldLog); err != nil { //根据index获取数据
         r.logger.Error("failed to get log", "index", i, "error", err)
         return err
      }
      req.Entries = append(req.Entries, oldLog) //填充到请求中
   }
   return nil
}
```

#### 发送同步请求

net_transport.go

```go
func (n *NetworkTransport) AppendEntries(id ServerID, target ServerAddress, args *AppendEntriesRequest, resp *AppendEntriesResponse) error {
   return n.genericRPC(id, target, rpcAppendEntries, args, resp)
}
```

net_transport.go:398

```go
func (n *NetworkTransport) genericRPC(id ServerID, target ServerAddress, rpcType uint8, args interface{}, resp interface{}) error {
   // 获取连接
   conn, err := n.getConnFromAddressProvider(id, target)
   if err != nil {
      return err
   }
   // 设置超时时间
   if n.timeout > 0 {
      conn.conn.SetDeadline(time.Now().Add(n.timeout))
   }
   // 发送请求
   if err = sendRPC(conn, rpcType, args); err != nil {
      return err
   }
   // 解析响应
   canReturn, err := decodeResponse(conn, resp)
   if canReturn { //返还给连接池
      n.returnConn(conn)
   }
   return err
}
```

### Pipeline同步

replication.go

```go
func (r *Raft) pipelineReplicate(s *followerReplication) error {
   //创建AppendPipeline
   pipeline, err := r.trans.AppendEntriesPipeline(s.peer.ID, s.peer.Address)
   if err != nil {
      return err
   }
  //关闭
   defer pipeline.Close()

   // Log start and stop of pipeline
   r.logger.Info("pipelining replication", "peer", s.peer)
   defer r.logger.Info("aborting pipeline replication", "peer", s.peer)

   // Create a shutdown and finish channel
   stopCh := make(chan struct{})
   finishCh := make(chan struct{})

   // 启动协程，解析pipeline请求的响应
   r.goFunc(func() { r.pipelineDecode(s, pipeline, stopCh, finishCh) })

   // Start pipeline sends at the last good nextIndex
   nextIndex := atomic.LoadUint64(&s.nextIndex)

   shouldStop := false
SEND:
   for !shouldStop {
      select {
      case <-finishCh:
         break SEND
      case maxIndex := <-s.stopCh: //停止同步
         if maxIndex > 0 {
            r.pipelineSend(s, pipeline, &nextIndex, maxIndex)
         }
         break SEND
      case deferErr := <-s.triggerDeferErrorCh:
         lastLogIdx, _ := r.getLastLog()
         shouldStop = r.pipelineSend(s, pipeline, &nextIndex, lastLogIdx)
         if !shouldStop {
            deferErr.respond(nil)
         } else {
            deferErr.respond(fmt.Errorf("replication failed"))
         }
      case <-s.triggerCh: //触发同步
         lastLogIdx, _ := r.getLastLog()
         shouldStop = r.pipelineSend(s, pipeline, &nextIndex, lastLogIdx)//同步数据
      case <-randomTimeout(r.conf.CommitTimeout):
         lastLogIdx, _ := r.getLastLog()
         shouldStop = r.pipelineSend(s, pipeline, &nextIndex, lastLogIdx)
      }
   }

   // Stop our decoder, and wait for it to finish
   close(stopCh)
   select {
   case <-finishCh:
   case <-r.shutdownCh:
   }
   return nil
}
```

#### 解析pipeline请求的响应

replication.go:483

```go
func (r *Raft) pipelineDecode(s *followerReplication, p AppendPipeline, stopCh, finishCh chan struct{}) {
   defer close(finishCh)
   respCh := p.Consumer()
   for {
      select {
      case ready := <-respCh:
         req, resp := ready.Request(), ready.Response()
         appendStats(string(s.peer.ID), ready.Start(), float32(len(req.Entries)))

         if resp.Term > req.Term { //检测term，如果从节点的term大于leader节点，停止同步
            r.handleStaleTerm(s)
            return
         }

         s.setLastContact()

         // Abort pipeline if not successful
         if !resp.Success { //响应失败，退出循环
            return
         }
         updateLastAppended(s, req)//修改同步状态
      case <-stopCh:
         return
      }
   }
}
```

#### 发送pipeline请求

replication.go

```go
func (r *Raft) pipelineSend(s *followerReplication, p AppendPipeline, nextIdx *uint64, lastIndex uint64) (shouldStop bool) {
   // Create a new append request
   req := new(AppendEntriesRequest)
  //填充请求
   if err := r.setupAppendEntries(s, req, *nextIdx, lastIndex); err != nil {
      return true
   }

   // Pipeline the append entries
   if _, err := p.AppendEntries(req, new(AppendEntriesResponse)); err != nil {
      r.logger.Error("failed to pipeline appendEntries", "peer", s.peer, "error", err)
      return true
   }

   // Increase the next send log to avoid re-sending old logs
   if n := len(req.Entries); n > 0 {
      last := req.Entries[n-1]
      atomic.StoreUint64(nextIdx, last.Index+1)
   }
   return false
}
```

net_transport.go

```go
func (n *netPipeline) AppendEntries(args *AppendEntriesRequest, resp *AppendEntriesResponse) (AppendFuture, error) {
   // 创建appendFuture
   future := &appendFuture{
      start: time.Now(),
      args:  args,
      resp:  resp,
   }
   future.init()

   // Add a send timeout
   if timeout := n.trans.timeout; timeout > 0 {
      n.conn.conn.SetWriteDeadline(time.Now().Add(timeout))
   }

   //发送请求
   if err := sendRPC(n.conn, rpcAppendEntries, future.args); err != nil {
      return nil, err
   }

   select {
   case n.inprogressCh <- future://inprogressCh默认128，背压，防止堆积过多的未响应的请求
      return future, nil
   case <-n.shutdownCh:
      return nil, ErrPipelineShutdown
   }
}
```

```go
func sendRPC(conn *netConn, rpcType uint8, args interface{}) error {
   // 请求类型
   if err := conn.w.WriteByte(rpcType); err != nil {
      conn.Release()
      return err
   }

   // Send the request
   if err := conn.enc.Encode(args); err != nil {
      conn.Release()
      return err
   }

   // Flush
   if err := conn.w.Flush(); err != nil {
      conn.Release()
      return err
   }
   return nil
}
```

