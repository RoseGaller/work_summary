# Reborn源码分析

* [启动入口](#启动入口)
* [创建Server](#创建server)
  * [填充Slots](#填充slots)
  * [处理客户端请求、事件](#处理客户端请求事件)
    * [分发客户端请求](#分发客户端请求)
      * [处理迁移](#处理迁移)
      * [创建TaskRunner](#创建taskrunner)
        * [将客户端的请求发送给redis](#将客户端的请求发送给redis)
        * [处理从redis返回的响应信息](#处理从redis返回的响应信息)
* [启动Server](#启动server)
  * [监听端口，处理读写请求](#监听端口处理读写请求)
    * [处理连接事件](#处理连接事件)
    * [读取客户端请求](#读取客户端请求)
    * [返回客户端响应信息](#返回客户端响应信息)


# 启动入口

reborn/cmd/proxy/main.go

```go
s := router.NewServer(conf) //1、创建Server
s.Run() //2、启动Server
```

# 创建Server 

reborn/pkg/proxy/router/router.go

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
   s.FillSlots() // 从zk获取Slot信息，填充Slot，默认1024个slot
   // start event handler
   go s.handleTopoEvent() //处理客户端请求、watch回调
   go s.dumpCounter() // 定时打印统计信息
   log.Info("proxy start ok")
   return s
}
```

## 填充Slots

（ reborn/pkg/proxy/router/router.go

```go
func (s *Server) FillSlots() {
   for i := 0; i < models.DEFAULT_SLOT_NUM; i++ {//默认1024
      s.fillSlot(i, false) //填充slot 信息
   }
}
```

reborn/pkg/proxy/router/router.go

```go
func (s *Server) fillSlot(i int, force bool) {
   if !validSlot(i) {
      return
   }
   if !force && s.slots[i] != nil { //check
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
      dst:       group.NewGroup(*groupInfo),
      groupInfo: groupInfo,
   }
   log.Infof("fill slot %d, force %v, %+v", i, force, slot.dst)
   //slot正在迁移
   if slot.slotInfo.State.Status == models.SLOT_STATUS_MIGRATE { 
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

## 处理客户端请求、事件

reborn/pkg/proxy/router/router.go

```go
func (s *Server) handleTopoEvent() {
	for {
		select {
		case r := <-s.reqCh: //客户端请求都放入此channel
			if s.slots[r.slotIdx].slotInfo.State.Status == models.SLOT_STATUS_PRE_MIGRATE { //正在迁移slot
				s.bufferedReq.PushBack(r) //将迁移的slot加入到List，待此slot迁移完成再处理
				continue
			}
			for e := s.bufferedReq.Front(); e != nil; {
				next := e.Next()
				if s.dispatch(e.Value.(*PipelineRequest)) { //将客户端请求发送到具体的redis
					s.bufferedReq.Remove(e) //处理完从list中移除
				}
				e = next
			}
			if !s.dispatch(r) { // 1.2.1将客户端请求发送到具体的redis
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

### 分发客户端请求

reborn/pkg/proxy/router/router.go

```go
func (s *Server) dispatch(r *PipelineRequest) (success bool) {
   slotStatus := s.slots[r.slotIdx].slotInfo.State.Status
   if slotStatus != models.SLOT_STATUS_ONLINE && slotStatus != models.SLOT_STATUS_MIGRATE {
      return false
   }
  //如果此slot正在迁移，则将请求的key逐一迁移；迁移完成再处理请求
   if err := s.handleMigrateState(r.slotIdx, r.keys...); err != nil {
      r.backQ <- &PipelineResponse{ctx: r, resp: nil, err: err} //设置迁移出现异常响应
      return true
   }
  //获取此slot的master对应的taskRunner；对于redis只会建立一条连接，taskrunner封装了对redis的读写请求
   tr, ok := s.pipeConns[s.slots[r.slotIdx].dst.Master()]
   if !ok {
      //创建taskrunner
      if err := s.createTaskRunner(s.slots[r.slotIdx]); err != nil {
         r.backQ <- &PipelineResponse{ctx: r, resp: nil, err: err} //设置创建TaskRunner出现的异常
         return true
      }
      tr = s.pipeConns[s.slots[r.slotIdx].dst.Master()]// 获取此slot的master对应的taskRunner
   }
   //将PipelineRequest放入taskRunner的请求channel
   tr.in <- r 
   return true
}
```

#### 处理迁移

reborn/pkg/proxy/router/router.go

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
   //从迁移专有的连接池中获取连接
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

reborn/pkg/proxy/router/router.go

```go
func (s *Server) createTaskRunner(slot *Slot) error {
   dst := slot.dst.Master()
   if _, ok := s.pipeConns[dst]; !ok {
      //新建TaskRunner
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

reborn/pkg/proxy/router/taskrunner.go

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
   //将客户端的请求发送给redis
   go tr.writeloop()
   //处理从redis返回的响应信息
   go tr.readloop()
   return tr, nil
}
```

##### 将客户端的请求发送给redis

reborn/pkg/proxy/router/taskrunner.go

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

##### 处理从redis返回的响应信息

reborn/pkg/proxy/router/taskrunner.go

```go
func (tr *taskRunner) readloop() {
   for {
      resp, err := parser.Parse(tr.c.BufioReader()) //读取redis的响应信息
      if err != nil {
         tr.out <- err
         return
      }

      tr.out <- resp //out存放redis返回的响应信息
   }
}
```

# 启动Server

## 监听端口，处理读写请求

reborn/pkg/proxy/router/router.go

```go
func (s *Server) Run() {
   log.Infof("listening %s on %s", s.conf.Proto, s.conf.Addr)
   listener, err := net.Listen(s.conf.Proto, s.conf.Addr) //监听端口
   if err != nil {
      log.Fatal(err)
   }
   for {
      conn, err := listener.Accept()  //接收客户端连接
      if err != nil {
         log.Warning(errors.ErrorStack(err))
         continue
      }
      go s.handleConn(conn) //每个连接创建单独的协程，负责数据读写
   }
}
```

### 处理连接事件

reborn/pkg/proxy/router/router.go

```go
func (s *Server) handleConn(c net.Conn) {
   log.Info("new connection", c.RemoteAddr())

   s.counter.Add("connections", 1)
   client := &session{
      Conn:          c,
      r:             bufio.NewReaderSize(c, DefaultReaderSize),
      w:             bufio.NewWriterSize(c, DefaultWiterSize),
      CreateAt:      time.Now(),
      backQ:         make(chan *PipelineResponse, PipelineResponseNum),
      closeSignal:   &sync.WaitGroup{},
      authenticated: false,
   } //创建Session
   client.closeSignal.Add(1)
   go client.WritingLoop() //向客户端返回响应信息
   var err error
   defer func() {
      client.closeSignal.Wait() //waiting for writer goroutine
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
         close(client.backQ)
         return
      }
      client.Ops++
   }
}
```

### 读取客户端请求

reborn/pkg/proxy/router/router.go

```go
func (s *Server) redisTunnel(c *session) error {
   resp, op, keys, err := getRespOpKeys(c)
	if err != nil {
		return errors.Trace(err)
	}
	k := keys[0]

	opstr := strings.ToUpper(string(op)) //解析命令

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
		if !isTheSameSlot(keys) { //所有的key不是都属于同一个slot
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
	} //封装请求信息
	pr.wg.Add(1)
	s.reqCh <- pr //将请求放入到Server的reqCh中
	pr.wg.Wait() //等待此PipelineRequest处理完成
	return nil
}
```

### 返回客户端响应信息

reborn/pkg/proxy/router/session.go

```go
func (s *session) WritingLoop() {
   s.lastUnsentResponseSeq = 1
   for {
      select {
      case resp, ok := <-s.backQ: //s.backQ存放返回给客户端的响应信息
         if !ok {
            s.Close()
            s.closeSignal.Done()
            return
         }

         flush, err := s.handleResponse(resp)//唤醒阻塞的请求
         if err != nil {
            log.Warning(s.RemoteAddr(), resp.ctx, errors.ErrorStack(err))
            s.Close() //notify reader to exit
            continue
         }

         if flush && len(s.backQ) == 0 {
            err := s.w.Flush()
            if err != nil {
               s.Close() //notify reader to exit
               log.Warning(s.RemoteAddr(), resp.ctx, errors.ErrorStack(err))
               continue
            }
         }
      }
   }
}
```

