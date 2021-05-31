# 数据同步

## 开始同步

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

## RPC同步

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

### 填充同步请求

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

### 根据索引获取数据

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

### 发送同步请求

net_transport.go:388

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

## Pipeline同步

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

### 解析pipeline请求的响应

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

### 发送pipeline请求

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

