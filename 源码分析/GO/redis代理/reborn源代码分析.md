# reborn源代码分析

/cmd/proxy/main.go

```go
	http.HandleFunc("/setloglevel", handleSetLogLevel) //修改日志级别
	go http.ListenAndServe(httpAddr, nil)

	conf, err := router.LoadConf(configFile) //加载配置信息
	if err != nil {
		log.Fatal(err)
	}
	//填充配置信息
	conf.Addr = addr
	conf.HTTPAddr = httpAddr
	conf.ProxyID = proxyID
	conf.PidFile = pidfile
	conf.NetTimeout = netTimeout
	conf.Proto = proto
	conf.ProxyAuth = proxyAuth	
	s := router.NewServer(conf) // 创建Server
	s.Run() // 启动Server
```

## 创建Server 

pkg/proxy/router/router.go

```go
func NewServer(conf *Conf) *Server {
   log.Infof("start with configuration: %+v", conf)
   f := func(addr string) (*redisconn.Conn, error) { //  redis连接池，创建redis连接
      return newRedisConn(addr, conf.NetTimeout, RedisConnReaderSize, RedisConnWiterSize, conf.StoreAuth)
   }
   s := &Server{
      conf:          conf,
      evtbus:        make(chan interface{}, EventBusNum),
      top:           topo.NewTopo(conf.ProductName, conf.CoordinatorAddr, conf.f, conf.Coordinator),
      counter:       stats.NewCounters("router"),
      lastActionSeq: -1,
      startAt:       time.Now(),
      moper:         newMultiOperator(conf.Addr, conf.ProxyAuth),
      reqCh:         make(chan *PipelineRequest, PipelineRequestNum),
      pools:         redisconn.NewPools(PoolCapability, f), //用于数据迁移
      pipeConns:     make(map[string]*taskRunner),
      bufferedReq:   list.New(),
   }
   s.pi.ID = conf.ProxyID
   s.pi.State = models.PROXY_STATE_OFFLINE
   addr := conf.Addr
   addrs := strings.Split(addr, ":")
   if len(addrs) != 2 {
      log.Fatalf("bad addr %s", addr)
   }
   hname, err := os.Hostname()
   if err != nil {
      log.Fatal("get host name failed", err)
   }
   s.pi.Addr = hname + ":" + addrs[1]
   debugVarAddr := conf.HTTPAddr
   debugVarAddrs := strings.Split(debugVarAddr, ":")
   if len(debugVarAddrs) != 2 {
      log.Fatalf("bad debugVarAddr %s", debugVarAddr)
   }
   s.pi.DebugVarAddr = hname + ":" + debugVarAddrs[1]
   s.pi.Pid = os.Getpid()
   s.pi.StartAt = time.Now().String()
   log.Infof("proxy_info:%+v", s.pi)
   stats.Publish("evtbus", stats.StringFunc(func() string {
      return strconv.Itoa(len(s.evtbus))
   }))
   stats.Publish("startAt", stats.StringFunc(func() string {
      return s.startAt.String()
   }))
   s.RegisterAndWait(true) //注册ProxyInfo到zookeeper
   s.registerSignal() //注册监听关闭的信号
   _, err = s.top.WatchChildren(models.GetWatchActionPath(conf.ProductName), s.evtbus) //注册watch
   if err != nil {
      log.Fatal(errors.ErrorStack(err))
   }
   s.FillSlots() // 1.1，从zk获取Slot信息，填充Slot，默认1024个slot
   // start event handler
   go s.handleTopoEvent() // 1.2处理客户端请求、eventbus
   go s.dumpCounter() // 定时打印统计信息
   log.Info("proxy start ok")
   return s
}
```

### 注册ProxyInfo到ZK

pkg/proxy/router/router.go

```go
func (s *Server) RegisterAndWait(wait bool) {
   _, err := s.top.CreateProxyInfo(&s.pi)
   if err != nil {
      log.Fatal(errors.ErrorStack(err))
   }

   _, err = s.top.CreateProxyFenceNode(&s.pi)
   if err != nil {
      log.Warning(errors.ErrorStack(err))
   }

   if wait { //默认true
      s.waitOnline()
   }
}
```

### 监听关闭的信号

```go
func (s *Server) registerSignal() {
   c := make(chan os.Signal, 1)
   signal.Notify(c, os.Interrupt, syscall.SIGTERM, os.Kill)
   go func() {
      <-c
      log.Info("ctrl-c or SIGTERM found, mark offline server")
      done := make(chan error)
      s.evtbus <- &killEvent{done: done}
      <-done
   }()
}
```

### 填充槽位信息

pkg/proxy/router/router.go

```go
func (s *Server) FillSlots() {
   for i := 0; i < models.DEFAULT_SLOT_NUM; i++ {//默认1024
      s.fillSlot(i, false) 
   }
}
```

```go
func (s *Server) fillSlot(i int, force bool) {
   if !validSlot(i) {
      return
   }
   if !force && s.slots[i] != nil { //检测槽位是否已经设置
      log.Fatalf("slot %d already filled, slot: %+v", i, s.slots[i])
   }
   s.clearSlot(i) //清除之前填充的信息
   slotInfo, groupInfo, err := s.top.GetSlotByIndex(i) //从zk获取slotInfo、groupInfo
   if err != nil {
      log.Fatal(errors.ErrorStack(err))
   }
  //所属的group，group下的主redis、从redis服务器信息
   slot := &Slot{
      slotInfo:  slotInfo,
      dst:       group.NewGroup(*groupInfo), //slot所属的master、salve机器信息
      groupInfo: groupInfo,
   }
   log.Infof("fill slot %d, force %v, %+v", i, force, slot.dst)
   
   if slot.slotInfo.State.Status == models.SLOT_STATUS_MIGRATE { //slot正在迁移
      // 获取迁移的原地址
      from, err := s.top.GetGroup(slot.slotInfo.State.MigrateStatus.From)
      if err != nil { // TODO: retry ?
         log.Fatal(err)
      }
      //设置迁移的原地址
      slot.migrateFrom = group.NewGroup(*from) 
   }
   s.slots[i] = slot
   s.counter.Add("FillSlot", 1)
}
```

```go
func NewGroup(groupInfo models.ServerGroup) *Group {
   g := &Group{
      redisServers: make(map[string]*models.Server), //slot所属的机器实例
   }

   for _, server := range groupInfo.Servers {
      if server.Type == models.SERVER_TYPE_MASTER { //master实例
         if len(g.master) > 0 {
            log.Fatalf("two masters are not allowed: %+v", groupInfo)
         }
         g.master = server.Addr
      }
      g.redisServers[server.Addr] = server
   }

   if len(g.master) == 0 { //缺失master实例
      log.Fatalf("master not found: %+v", groupInfo)
   }

   return g
}
```

### 处理客户端请求、事件

reborn/pkg/proxy/router/router.go

```go
func (s *Server) handleTopoEvent() {
	for {
		select {
		case r := <-s.reqCh: //接收到的客户端请求都放入此channel
			if s.slots[r.slotIdx].slotInfo.State.Status == models.SLOT_STATUS_PRE_MIGRATE { //正在迁移slot
        //如果slot正在迁移中，将请求暂存到bufferedReq，待此slot迁移完成再处理
				s.bufferedReq.PushBack(r)
				continue
			}
      //优先处理暂存的请求
			for e := s.bufferedReq.Front(); e != nil; {
				next := e.Next()
				if s.dispatch(e.Value.(*PipelineRequest)) { //将客户端请求发送到具体的redis
					s.bufferedReq.Remove(e) //处理完从list中移除
				}
				e = next
			}
			if !s.dispatch(r) { // 将客户端请求发送到具体的redis
				log.Fatalf("should never happend, %+v, %+v", r, s.slots[r.slotIdx].slotInfo)
			}
		case e := <-s.evtbus: //处理事件
			switch e.(type) {
			case *killEvent: //关闭事件
				s.handleMarkOffline()
				e.(*killEvent).done <- nil
			default: //用户的操作事件
				if s.top.IsSessionExpiredEvent(e) {
					log.Fatalf("session expired: %+v", e)
				}
				evtPath := GetEventPath(e)
				log.Infof("got event %s, %v, lastActionSeq %d", s.pi.ID, e, s.lastActionSeq)
				if strings.Index(evtPath, models.GetActionResponsePath(s.conf.ProductName)) == 0 {
					seq, err := strconv.Atoi(path.Base(evtPath))
					if err != nil {
						log.Warning(err)
					} else {
						if seq < s.lastActionSeq {
							log.Infof("ignore, lastActionSeq %d, seq %d", s.lastActionSeq, seq)
							continue
						}
					}
				}
				s.processAction(e)
			}
		}
	}
}
```

#### 将请求分发至Redis

（reborn/pkg/proxy/router/router.go:550）

```go
func (s *Server) dispatch(r *PipelineRequest) (success bool) {
   slotStatus := s.slots[r.slotIdx].slotInfo.State.Status
   if slotStatus != models.SLOT_STATUS_ONLINE && slotStatus != models.SLOT_STATUS_MIGRATE {
      return false
   }
  // 如果此slot正在迁移，则将请求的key逐一迁移；迁移完成再处理请求
   if err := s.handleMigrateState(r.slotIdx, r.keys...); err != nil {
      r.backQ <- &PipelineResponse{ctx: r, resp: nil, err: err}
      return true
   }
  //获取此slot的master对应的taskRunner；对于redis只会建立一条连接，taskrunner封装了对redis的读写请求
   tr, ok := s.pipeConns[s.slots[r.slotIdx].dst.Master()]
   if !ok {
      // try recreate taskrunner  1.2.1.2 创建taskrunner
      if err := s.createTaskRunner(s.slots[r.slotIdx]); err != nil {
         r.backQ <- &PipelineResponse{ctx: r, resp: nil, err: err}
         return true
      }
      tr = s.pipeConns[s.slots[r.slotIdx].dst.Master()]
   }
   //将PipelineRequest放入taskRunner的channel中
   tr.in <- r 
   return true
}
```

迁移请求的key

pkg/proxy/router/router.go

```go
func (s *Server) handleMigrateState(slotIndex int, keys ...[]byte) error {
   shd := s.slots[slotIndex]
   if shd.slotInfo.State.Status != models.SLOT_STATUS_MIGRATE {
      return nil
   }
   if shd.migrateFrom == nil {
      log.Fatalf("migrateFrom not exist %+v", shd)
   }
   if shd.dst.Master() == shd.migrateFrom.Master() {
      log.Fatalf("the same migrate src and dst, %+v", shd)
   }
   //从连接池中获取连接
   redisConn, err := s.pools.GetConn(shd.migrateFrom.Master())
   if err != nil {
      return errors.Trace(err)
   }
	//将连接返还给连接池
   defer s.pools.PutConn(redisConn)
	 //给slotFrom发送请求，将key迁移到slotDst
   err = writeMigrateKeyCmd(redisConn, shd.dst.Master(), MigrateKeyTimeoutMs, keys...)
   if err != nil {
      redisConn.Close()
      log.Errorf("migrate key %s error, from %s to %s, err:%v",
         string(keys[0]), shd.migrateFrom.Master(), shd.dst.Master(), err)
      return errors.Trace(err)
   }
   redisReader := redisConn.BufioReader()
   //处理迁移的结果
   for i := 0; i < len(keys); i++ {
      resp, err := parser.Parse(redisReader)
      if err != nil {
         log.Errorf("migrate key %s error, from %s to %s, err:%v",
            string(keys[i]), shd.migrateFrom.Master(), shd.dst.Master(), err)
         redisConn.Close()
         return errors.Trace(err)
      }
      result, err := resp.Bytes()
      log.Debug("migrate", string(keys[0]), "from", shd.migrateFrom.Master(), "to", shd.dst.Master(),
         string(result))
      if resp.Type == parser.ErrorResp {
         redisConn.Close()
         log.Error(string(keys[0]), string(resp.Raw), "migrateFrom", shd.migrateFrom.Master())
         return errors.New(string(resp.Raw))
      }
   }
   s.counter.Add("Migrate", int64(len(keys)))
   return nil
}
```

####  创建TaskRunner 

pkg/proxy/router/router.go

```go
func (s *Server) createTaskRunner(slot *Slot) error {
   dst := slot.dst.Master()
   if _, ok := s.pipeConns[dst]; !ok {
      // 1.2.1.2.1  新建TaskRunner
      tr, err := NewTaskRunner(dst, s.conf.NetTimeout, s.conf.StoreAuth) 
      if err != nil {
         return errors.Errorf("create task runner failed, %v,  %+v, %+v", err, slot.dst, slot.slotInfo)
      } else {
         s.pipeConns[dst] = tr
      }
   }
   return nil
}
```

1.2.1.2.1 新建TaskRunner（reborn/pkg/proxy/router/taskrunner.go:206）

```go
func NewTaskRunner(addr string, netTimeout int, auth string) (*taskRunner, error) {
   tr := &taskRunner{
      in:         make(chan interface{}, TaskRunnerInNum),
      out:        make(chan interface{}, TaskRunnerOutNum),
      redisAddr:  addr,
      tasks:      list.New(),
      netTimeout: netTimeout,
      auth:       auth,
   }
   //创建redis连接
   c, err := newRedisConn(addr, netTimeout, PipelineBufSize, PipelineBufSize, auth)
   if err != nil {
      return nil, errors.Trace(err)
   }
   tr.c = c
   //1.2.1.2.1.1将客户端的请求发送给redis
   go tr.writeloop()
   //1.2.1.2.1.2处理从redis返回的响应信息
   go tr.readloop()
   return tr, nil
}
```

1.2.1.2.1.1将客户端的请求发送给redis（reborn/pkg/proxy/router/taskrunner.go:175）

```go
func (tr *taskRunner) writeloop() {
   var err error
   tick := time.Tick(2 * time.Second)
   for {
      if tr.closed && tr.tasks.Len() == 0 {
         log.Warning("exit taskrunner", tr.redisAddr)
         tr.wgClose.Done()
         tr.c.Close()
         return
      }
			//出现异常，进行恢复
      if err != nil { //clean up
         err = tr.tryRecover(err)
         if err != nil {
            continue
         }
      }

      select {
      case t := <-tr.in: //发送请求给redis
         tr.processTask(t)
      case resp := <-tr.out: //out中包含redis返回的响应信息、读取数据时产生的error信息
         err = tr.handleResponse(resp)
      case <-tick:
         if tr.tasks.Len() > 0 && int(time.Since(tr.latest).Seconds()) > tr.netTimeout {
            tr.c.Close()
         }
      }
   }
}
```

1.2.1.2.1.2处理从redis返回的响应信息（reborn/pkg/proxy/router/taskrunner.go:34）

```go
func (tr *taskRunner) readloop() {
   for {
      resp, err := parser.Parse(tr.c.BufioReader())
      if err != nil {
         tr.out <- err
         return
      }

      tr.out <- resp
   }
}
```

## 启动Server

pkg/proxy/router/router.go

```go
func (s *Server) Run() {
   log.Infof("listening %s on %s", s.conf.Proto, s.conf.Addr)
   listener, err := net.Listen(s.conf.Proto, s.conf.Addr) //监听端口
   if err != nil {
      log.Fatal(err)
   }
   //接收客户端连接处理请求
   for {
      conn, err := listener.Accept()  //接收客户端连接
      if err != nil {
         log.Warning(errors.ErrorStack(err))
         continue
      }
      go s.handleConn(conn) // 每个连接创建单独的协程，负责数据读写
   }
}
```

```go
func (s *Server) handleConn(c net.Conn) {
   log.Info("new connection", c.RemoteAddr())

   s.counter.Add("connections", 1)
   //创建session实例
   client := &session{
      Conn:          c,
      r:             bufio.NewReaderSize(c, DefaultReaderSize),
      w:             bufio.NewWriterSize(c, DefaultWiterSize),
      CreateAt:      time.Now(),
      backQ:         make(chan *PipelineResponse, PipelineResponseNum),
      closeSignal:   &sync.WaitGroup{},
      authenticated: false,
   }
   client.closeSignal.Add(1)
   go client.WritingLoop() // 向客户端返回响应信息
   var err error //读取请求数据时产生的异常信息
   defer func() {
      client.closeSignal.Wait() //等待写协程结束
      if errors2.ErrorNotEqual(err, io.EOF) {
         log.Warningf("close connection %v, %v", client, errors.ErrorStack(err))
      } else {
         log.Infof("close connection %v", client)
      }
      s.counter.Add("connections", -1)
   }()
   for {
      err = s.redisTunnel(client) // 读取数据，封装请求
      if err != nil {
         close(client.backQ) //关闭channel
         return
      }
      client.Ops++
   }
}
```

### 读取客户端请求

pkg/proxy/router/router.go:242）

```go
func (s *Server) redisTunnel(c *session) error {
   resp, op, keys, err := getRespOpKeys(c)
	if err != nil {
		return errors.Trace(err)
	}
	k := keys[0]

	opstr := strings.ToUpper(string(op))

	if opstr == "AUTH" {
		buf, err := s.handleAuthCommand(opstr, k)
		s.sendBack(c, op, keys, resp, buf)
		c.authenticated = (err == nil)
		return errors.Trace(err)
	} else if len(s.conf.ProxyAuth) > 0 && !c.authenticated {
		buf := []byte("-ERR NOAUTH Authentication required\r\n")
		s.sendBack(c, op, keys, resp, buf)
		return errors.Errorf("NOAUTH Authentication required")
	}

	buf, next, err := filter(opstr, keys, c, s.conf.NetTimeout)
	if err != nil {
		if len(buf) > 0 { //quit command or error message
			s.sendBack(c, op, keys, resp, buf)
		}
		return errors.Trace(err)
	}

	start := time.Now()
	defer func() {
		recordResponseTime(s.counter, time.Since(start)/1000/1000)
	}()

	s.counter.Add(opstr, 1)
	s.counter.Add("ops", 1)
	if !next {
		s.sendBack(c, op, keys, resp, buf)
		return nil
	}

	if isMulOp(opstr) { //MGET、MSET、DEL单独处理
		if !isTheSameSlot(keys) { // can not send to redis directly 所有的key是不是都属于同一个slot
			var result []byte
			err := s.moper.handleMultiOp(opstr, keys, &result)
			if err != nil {
				return errors.Trace(err)
			}
			s.sendBack(c, op, keys, resp, result)
			return nil
		}
	}

	i := mapKey2Slot(k) //计算此key属于哪个slot，还要考虑Hashtag
	c.pipelineSeq++
	pr := &PipelineRequest{
		slotIdx: i,
		op:      op,
		keys:    keys,
		seq:     c.pipelineSeq,
		backQ:   c.backQ,
		req:     resp,
		wg:      &sync.WaitGroup{},
	}
	pr.wg.Add(1)
	s.reqCh <- pr //将请求放入到Server的reqCh中
	pr.wg.Wait() //等待此PipelineRequest处理完成
	return nil
}
```

### 返回客户端响应信息

pkg/proxy/router/session.go

```go
func (s *session) WritingLoop() {
   s.lastUnsentResponseSeq = 1
   for {
      select {
      case resp, ok := <-s.backQ:
         if !ok { //channel已经关闭
            s.Close()
            s.closeSignal.Done()
            return
         }

         flush, err := s.handleResponse(resp)
         if err != nil {
            log.Warning(s.RemoteAddr(), resp.ctx, errors.ErrorStack(err))
            s.Close() //notify reader to exit
            continue
         }

         if flush && len(s.backQ) == 0 { //channel中以无信息可响应给客户端
            err := s.w.Flush() //调用flush操作，节省系统调用次数
            if err != nil {
               s.Close() //出现异常，关闭连接
               log.Warning(s.RemoteAddr(), resp.ctx, errors.ErrorStack(err))
               continue
            }
         }
      }
   }
}
```

pkg/proxy/router/session.go

```go
func (s *session) handleResponse(resp *PipelineResponse) (flush bool, err error) {
   if resp.ctx.seq != s.lastUnsentResponseSeq {
      log.Fatal("should never happend")
   }

   s.lastUnsentResponseSeq++
   if resp.ctx.wg != nil {
      resp.ctx.wg.Done() //唤醒阻塞的请求
   }

   if resp.err != nil { //notify close client connection
      return true, resp.err
   }

   if !s.closed {
      if err := s.writeResp(resp); err != nil {
         return false, errors.Trace(err)
      }

      flush = true
   }

   return
}
```
