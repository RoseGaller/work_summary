# 概述

使用 Go 开发的，工作在客户端模式的透明代理，代理redis集群，实现了热点key的实时统计

# 启动入口

cmd/samaritan/samaritan.go

```go
func main() {
   parseFlag() //创建config
   initStats()

   initInstance() //创建instance
   writePid()

   handleSignal()
}
```

## initConfig

```go
func parseFlag() {
   if flag.ShowVersionOnly {
      showSamVersion()
      os.Exit(0)
   }

   initConfig()
   initLogger()

   logger.Info("Using CPUs: ", runtime.GOMAXPROCS(-1))
   logger.Info("Version: ", consts.Version)
   if len(consts.Build) != 0 {
      logger.Info("Build: ", consts.Build)
   }
}
```

## initInstance

```go
func initInstance() {
  //创建admin
   server := admin.New(c)
   //传入config的evtCh，存放发布的事件；创建controller
   ctl, err := controller.New(c.Subscribe())
   if err != nil {
      logger.Fatalf("Error while initializing controller: %v", err)
   }

   inst = &instance{
      id:       os.Getpid(),
      parentID: parentPid,
      c:        c,
      admin:    server,
      ctl:      ctl,
   }
  //创建
   r, err := hotrestart.New(inst)
   if err != nil {
      logger.Fatalf("Error while initializing restarter: %v", err)
   }
   inst.Restarter = r
   inst.Run()
}
```

## runInstance

```go
func (inst *instance) Run() {
   if err := inst.ctl.Start(); err != nil { //启动controller
      logger.Fatalf("Error while starting broker: %v", err)
   }

   inst.ShutdownParentAdmin()
   if err := inst.admin.Start(inst.parentID > 0); err != nil { //启动admin
      logger.Fatalf("Error while starting admin api: %s", err)
   }

   inst.DrainParentListeners()
   if inst.parentID > 0 {
      go func() {
         d, err := time.ParseDuration(os.Getenv(consts.ParentTerminateTimeEnv))
         if err != nil {
            d = consts.DefaultParentTerminateTime
         }
         logger.Infof("Will terminate parent %d after %s", inst.parentID, d)
         time.Sleep(d)
         inst.TerminateParent()
      }()
   }
}
```

# Config

```go
func initConfig() { //创建conig的入口
   if flag.ConfigFile == "" {
      log.Fatal("Config file cannot be empty")
   }

   logger.Info("Config file: ", flag.ConfigFile)
  //创建bootstrap
   b, err := config.LoadBootstrap(flag.ConfigFile)
   if err != nil {
      log.Fatalf("Failed to load bootstrap: %v", err)
   }

   // handle the metadata of instance.
   if b.Instance == nil {
      b.Instance = &common.Instance{}
   }
   b.Instance.Version = consts.Version
   b.Instance.Id = utils.PickFirstNonEmptyStr(
      flag.InstanceID,
      b.Instance.Id,
      genDefaultInstanceID(b.Admin.Bind.Port),
   )
   b.Instance.Belong = utils.PickFirstNonEmptyStr(
      flag.BelongService,
      b.Instance.Belong,
   )
	//根据bootstrap创建config
   c, err = config.New(b)
   if err != nil {
      logger.Fatalf("Failed to init config: %v", err)
   }
}
```

## 创建Bootstrap

config/bootstrap.go

```go
func LoadBootstrap(file string) (*bootstrap.Bootstrap, error) {
  //读取配置文件，创建bootstrap
   b, err := ioutil.ReadFile(file)
   if err != nil {
      return nil, err
   }
   return newBootstrapFromYAML(b)
}
```

```go
func newBootstrapFromYAML(b []byte) (*bootstrap.Bootstrap, error) {
   jsonBytes, err := yaml.YAMLToJSON(b)
   if err != nil {
      return nil, err
   }
   return newBootstrapFromJSON(jsonBytes)
}
```

```go
func newBootstrapFromJSON(b []byte) (*bootstrap.Bootstrap, error) {
   cfg := new(bootstrap.Bootstrap)
   if err := json.Unmarshal(b, cfg); err != nil {
      return nil, err
   }
   return cfg, nil
}
```

## 创建config

config/config.go

```go
func New(b *bootstrap.Bootstrap) (*Config, error) {
   if err := b.Validate(); err != nil {
      return nil, err
   }
		
   c := &Config{
      Bootstrap: b,
      sws:       make(map[string]*serviceWrapper),
   }
   if err := c.init(); err != nil {
      return nil, err
   }
   return c, nil
}
```

## 初始化config

config/config.go

```go
func (c *Config) init() error {

   evtLen := 32
   if len(c.StaticServices) > evtLen {
      evtLen = len(c.StaticServices)
   }
   c.evtCh = make(chan Event, evtLen)

   c.initStaticSvcs() //初始化静态服务
   c.initDynamic() //监听配置的变更
   return nil
}
```

### initStaticSvcs

```go
func (c *Config) initStaticSvcs() {//初始化服务
   for _, svc := range c.StaticServices {
      if svc == nil {
         continue
      }

      sw := &serviceWrapper{
         Service:   &service.Service{Name: svc.Name},
         Config:    svc.Config,
         Endpoints: svc.Endpoints,
      }
      c.sws[svc.Name] = sw
      c.emitSvcAddEvent(sw) //发布添加服务事件
   }
}
```

```go
func (c *Config) emitSvcAddEvent(sw *serviceWrapper) {//发布事件
   evt := &SvcAddEvent{ //服务添加事件
      Name:      sw.Service.Name,
      Config:    sw.Config,
      Endpoints: sw.Endpoints,
   }
   c.evtCh <- evt //发布事件
}
```

### initDynamic

```go
func (c *Config) initDynamic() error {
   if c.DynamicSourceConfig == nil {
      logger.Debugf("The config of dynamic source is empty, dynamic source will be disabled")
      return nil
   }

   if c.Instance.Belong == "" {
      logger.Warnf("The service which instance belongs to is empty, dynamic source will be disabled")
      return nil
   }
	//启用dynamic source，通过grpc通信
   d, err := dynamicSourceFactory(c.Bootstrap)
   if err != nil {
      return err
   }
  //注册钩子函数监听变更
	//called when a dependency is added or removed
   d.SetDependencyHook(c.handleDependencyUpdate)
  // called when the proxy config of subscribed service update
   d.SetSvcConfigHook(c.handleSvcConfigUpdate)
	// called when the endpoints of subscribed service updated
   d.SetSvcEndpointHook(c.handleSvcEndpointUpdate)
   c.d = d
   go c.d.Serve()
   return nil
}
```

# Controller

## 创建Controller

controller/controller.go

```go
func New(evtC <-chan config.Event) (*Controller, error) {
   ctl := &Controller{
      evtC:  evtC, //config发布事件
      procs: make(map[string]proc.Proc), //维护代理的服务
      quit:  make(chan struct{}),
      done:  make(chan struct{}),
   }
   return ctl, nil
}
```

## 启动Controller

```go
func (c *Controller) Start() error {//监听config发布的事件
   c.startOnce.Do(func() { //启动一次
      go func() {
         defer close(c.done)
         for {
            select {
            case <-c.quit: //关闭
               c.stopAllProcs() //关闭所有的服务
               return
            case evt := <-c.evtC: //config发布事件
               c.handleEvent(evt)
            }
         }
      }()
   })
   return nil
}
```

## 处理Event

```go
func (c *Controller) handleEvent(evt config.Event) {//config发布事件，Controller处理事件
   switch evt := evt.(type) {
   case *config.SvcAddEvent: //服务添加
      c.handleSvcAdd(evt.Name, evt.Config, evt.Endpoints)
   case *config.SvcRemoveEvent:
      c.handleSvcDel(evt.Name)
   case *config.SvcConfigEvent:
      c.handleSvcConfigUpdate(evt.Name, evt.Config)
   case *config.SvcEndpointEvent:
      c.handleSvcEndpointsAdd(evt.Name, evt.Added)
      c.handleSvcEndpointsRemove(evt.Name, evt.Removed)
   default:
      logger.Warnf("unkown event: %v", evt)
   }

}
```

### handleSvcAdd

```go
func (c *Controller) handleSvcAdd(svcName string, cfg *service.Config, endpoints []*service.Endpoint) {
   if _, ok := c.getProc(svcName); ok { //已经处理过此类事件
      return
   }
   c.tryEnsureProc(svcName, cfg, endpointsToHosts(endpoints))
}
```



```go
func endpointsToHosts(endpoints []*service.Endpoint) []*host.Host {
  	//将Endpoint转换成Host
   hosts := make([]*host.Host, 0, len(endpoints))
   for _, endpoint := range endpoints {
      typ := host.TypeMain
      if endpoint.Type == service.Endpoint_BACKUP {
         typ = host.TypeBackup
      }
      addr := fmt.Sprintf("%s:%d", endpoint.Address.Ip, endpoint.Address.Port)
      hosts = append(hosts, host.NewWithType(addr, typ))
   }
   return hosts
}
```



```go
func (c *Controller) tryEnsureProc(svcName string, cfg *service.Config, hosts []*host.Host) (p proc.Proc) {
   if svcName == "" { 
      logger.Debugf("empty service name")
      return
   }
   if err := cfg.Validate(); cfg == nil || err != nil { //验证服务的配置合法性
      logger.Debugf("invalid config")
      return
   }

   proc, err := newProc(svcName, cfg, hosts) //创建proc
   if err != nil {
      logger.Warnf("Create processor %s failed: %v", svcName, err)
      return
   }
   if err := proc.Start(); err != nil { //启动proc
      logger.Warnf("Start processor %s failed: %v", svcName, err)
      return
   }
   c.addProc(proc)
   return proc
}
```

# Proc

Proc用于处理和转发网络流量，比如接收redis请求，转发给后端的redis集群

## 创建Proc

维护代理的服务，比如redis、tcp

### 代理redis配置

```yaml
static_services:
  - name: redis-demo
    config:
      listener:
        address:
          ip: 0.0.0.0
          port: 6379
      protocol: Redis
    endpoints:
        - address:
            ip: 176.17.0.2
            port: 7000
        - address:
            ip: 176.17.0.3
            port: 7000
        - address:
            ip: 176.17.0.4
            port: 7000
```

```go
var (
   builderRegistry = make(map[protocol.Protocol]Builder) //全局变量，维护所有的Builder
)
```

proc/redis/redis.go

```go
func init() {
  //注册redis
   proc.RegisterBuilder(protocol.Redis, new(builder)) //main函数运行前，执行init
}
```

proc/proc.go

```go
func New(name string, cfg *service.Config, hosts []*host.Host) (p Proc, err error) {
   b, ok := getBuilder(cfg.Protocol) //根据协议获取Builder
   if !ok {
      return nil, ErrUnsupportedProtocol
   }
   scopeName := fmt.Sprintf("service.%s.", strings.Replace(name, ".", "_", -1))
   s := NewStats(stats.CreateScope(scopeName))
   setDefaultValue(cfg)
   proc, err := b.Build(BuildParams{
      name,
      cfg,
      hosts,
      s,
      log.New(fmt.Sprintf("[%s]", name)),
   })//创建proc
   if err != nil {
      return nil, err
   }
   return newWrapppedProc(proc), nil
}
```



```go
func (*builder) Build(params proc.BuildParams) (proc.Proc, error) { //创建redisProc
   return newRedisProc(params.Name, params.Cfg, params.Hosts,
      params.Stats, params.Logger)
}
```



```go
func newRedisProc(svcName string, svcCfg *service.Config, svcHosts []*host.Host,
   stats *proc.Stats, logger log.Logger) (*redisProc, error) {
   p := &redisProc{
      name:     svcName,
      cfg:      newConfig(svcCfg),
      stats:    stats,
      logger:   logger,
      cmdHdlrs: make(map[string]*commandHandler), //维护所有命令对应的handler
   }
	//创建listener
   l, err := proc.NewListener(p.cfg.Listener, p.stats.Downstream, logger, p.handleConn)
   if err != nil {
      return nil, err
   }

   p.l = l
  //创建upstream
   p.u = newUpstream(p.cfg, svcHosts, p.logger, p.stats.Upstream)
  //初始化命令及对应的handler
   p.initCommandHandlers()
   return p, nil
}
```

### handleConnFunc

proc/redis/redis.go

```go
func (p *redisProc) handleConn(conn net.Conn) { //处理客户端连接
   s := newSession(p, conn)
   s.Serve()
}
```

### newListerner

proc/listener.go

```go
func NewListener(cfg *service.Listener, stats *DownstreamStats, logger log.Logger, connHandleFn ConnHandlerFunc) (Listener, error) {
   if err := cfg.Validate(); err != nil {
      return nil, err
   }
   if cfg.Address.GetIp() == "" {
      return nil, ErrIPNotSet
   }

   l := &listener{
      cfg:          cfg,
      Logger:       logger,
      stats:        stats,
      conns:        make(map[net.Conn]struct{}),
      connHandleFn: connHandleFn,
      drain:        make(chan struct{}),
      quit:         make(chan struct{}),
      done:         make(chan struct{}),
   }
   return l, nil
}
```

### newUpstream

proc/redis/upstream.go

```go
func newUpstream(cfg *config, hosts []*host.Host, logger log.Logger, stats *proc.UpstreamStats) *upstream {
   u := &upstream{
      cfg:            cfg,
      hosts:          host.NewSet(hosts...),
      logger:         logger,
      stats:          stats,
      hkc:            hotkey.NewCollector(50),
      slotsRefreshCh: make(chan struct{}, 1),
      quit:           make(chan struct{}),
      done:           make(chan struct{}),
   }
   u.clients.Store(make(map[string]*client))
   return u
}
```

### 初始化handler

proc/redis/redis.go

```go
func (p *redisProc) initCommandHandlers() {
   scope := p.stats.NewChild("redis")
   for _, cmd := range simpleCommands { 
      p.addHandler(scope, cmd, handleSimpleCommand)
   }

   for _, cmd := range sumResultCommands {
      p.addHandler(scope, cmd, handleSumResultCommand)
   }

   p.addHandler(scope, "eval", handleEval)
   p.addHandler(scope, "mset", handleMSet)
   p.addHandler(scope, "mget", handleMGet)
   p.addHandler(scope, "scan", handleScan)
   p.addHandler(scope, "hotkey", handleHotKey)

   p.addHandler(scope, "ping", handlePing)
   p.addHandler(scope, "quit", handleQuit)
   p.addHandler(scope, "info", handleInfo)
   p.addHandler(scope, "time", handleTime)
   p.addHandler(scope, "select", handleSelect)
}
```

## 启动Proc

proc/redis/redis.go

```go
func (p *redisProc) Start() error {
   p.wg.Add(1)
   go func() {
      p.u.Serve() //upstream serveForServer
      p.wg.Done()
   }()
   p.wg.Add(1)
   go func() {
      p.l.Serve() //处理客户端连接、请求
      p.wg.Done()
   }()
   return nil
}
```

## 处理请求

```go
func (p *redisProc) handleRequest(req *rawRequest) {
   // rawRequest metrics
   p.stats.Downstream.RqTotal.Inc()
   req.RegisterHook(func(req *rawRequest) {
      switch req.Response().Type {
      case Error:
         p.stats.Downstream.RqFailureTotal.Inc()
      default:
         p.stats.Downstream.RqSuccessTotal.Inc()
      }
      p.stats.Downstream.RqDurationMs.Record(uint64(req.Duration() / time.Millisecond))
   })

   // check
   if !req.IsValid() {
      req.SetResponse(newError(invalidRequest))
      return
   }

   // find corresponding _handler
   cmd := string(req.Body().Array[0].Text)
   hdlr, ok := p.findHandler(cmd)
   if !ok {
      // unsupported command
      req.SetResponse(newError(fmt.Sprintf("ERR unsupported command '%s'", cmd)))
      return
   }

   cmdStats := hdlr.stats
   cmdStats.Total.Inc()
   req.RegisterHook(func(req *rawRequest) {
      switch req.Response().Type {
      case Error:
         cmdStats.Error.Inc()
      default:
         cmdStats.Success.Inc()
      }
      latency := uint64(req.Duration() / time.Microsecond)
      cmdStats.LatencyMicros.Record(latency)
      if latency > p.cfg.slowReqThresholdInMicros {
         p.stats.Counter("rq_slow_total").Inc()
      }
   })
   hdlr.handle(p.u, req)
}
```

### 获取handler

#### handleSimpleCommand

```go
func handleSimpleCommand(u *upstream, req *rawRequest) {
   body := req.Body()
   if len(body.Array) < 2 {
      req.SetResponse(newError(invalidRequest))
      return
   }

   simpleReq := newSimpleRequest(body)
   simpleReq.RegisterHook(func(simpleReq *simpleRequest) {
      req.SetResponse(simpleReq.Response())
   })
   key := body.Array[1].Text
   u.MakeRequest(key, simpleReq)
}
```

#### handleSumResultCommand

```go
func handleSumResultCommand(u *upstream, req *rawRequest) {
   sumResultReq, err := newSumResultRequest(req)
   if err != nil {
      req.SetResponse(newError(err.Error()))
      return
   }

   simpleReqs := sumResultReq.Split()
   for i := 0; i < len(simpleReqs); i++ {
      simpleReq := simpleReqs[i]
      key := simpleReq.Body().Array[1].Text
      u.MakeRequest(key, simpleReq)
   }
}
```

### 发起请求

```go
func (u *upstream) MakeRequest(routingKey []byte, req *simpleRequest) {
   addr, err := u.chooseHost(routingKey, req) //选择redis实例地址
   if err != nil {
      req.SetResponse(newError(err.Error()))
      return
   }
   u.MakeRequestToHost(addr, req) //发送请求
}
```

#### 选择redis实例

```go
func (u *upstream) chooseHost(routingKey []byte, req *simpleRequest) (string, error) {
   hash := crc16(hashtag(routingKey))//计算hash
   inst := u.slots[hash&(slotNum-1)] //计算slot

   // There is no redis instance which is responsible for the specified slot.
   // The possible reasons are as follows:
   // 1) The proxy has just started, and has not yet pulled routing information.
   // 2) The redis cluster has some temporary problems, and will recover automatically later.
   // To avoid these problems, could randomly choose one from the provided seed hosts.
   if inst == nil {
      return u.randomHost()
   }

   if !req.IsReadOnly() {
      return inst.Addr, nil
   }

   // read-only requests
   var candidates []string
   readStrategy := redis.ReadStrategy_MASTER
   if option := u.cfg.GetRedisOption(); option != nil {
      readStrategy = option.ReadStrategy
   }
   switch readStrategy {
   case redis.ReadStrategy_MASTER:
      candidates = append(candidates, inst.Addr)
   case redis.ReadStrategy_BOTH:
      candidates = append(candidates, inst.Addr)
      fallthrough
   case redis.ReadStrategy_REPLICA:
      for _, replica := range inst.Replicas {
         candidates = append(candidates, replica.Addr)
      }
   }

   if len(candidates) == 0 {
      candidates = append(candidates, inst.Addr)
   }
   i := 0
   l := len(candidates)
   if l > 1 {
      // The concurrency-safe rand source does not scale well to multiple cores,
      // so we use currrent unix time in nanoseconds to avoid it. As a result these
      // generated values are sequential rather than random, but are acceptable.
      i = int(time.Now().UnixNano()) % l
   }
   return candidates[i], nil
}
```

#### 发送请求

```go
func (u *upstream) MakeRequestToHost(addr string, req *simpleRequest) {
   // request metrics
   u.stats.RqTotal.Inc()
   req.RegisterHook(func(req *simpleRequest) {
      if req.Response().Type == Error {
         u.stats.RqFailureTotal.Inc()
      } else {
         u.stats.RqSuccessTotal.Inc()
      }
      u.stats.RqDurationMs.Record(uint64(req.Duration() / time.Millisecond))
   })

   select {
   case <-u.quit:
      req.SetResponse(newError(upstreamExited))
      return
   default:
   }

   c, err := u.getClient(addr) //获取client实例
   if err != nil {
      req.SetResponse(newError(err.Error()))
      return
   }
   // TODO: detect client status
   c.Send(req) //向redis发送请求
}
```

# UpStream

proc/redis/upstream.go

```go
func (u *upstream) Serve() { 
   var wg sync.WaitGroup
   wg.Add(2)
   go func() {
      defer wg.Done()
      u.loopRefreshSlots() //刷新slot，维护和redis实例的连接
   }()
   go func() {
      defer wg.Done()
      u.hkc.Run(u.quit) //处理hot key
   }()
  //阻塞直到协程结束
   wg.Wait()

   //关闭所有客户端连接
   u.clientsMu.Lock()
   clients := u.loadClients()
   for _, c := range clients {
      c.Stop()
   }
   u.clientsMu.Unlock()
   close(u.done)
}
```

```
func handleSimpleCommand(u *upstream, req *rawRequest) {
   body := req.Body()
   if len(body.Array) < 2 {
      req.SetResponse(newError(invalidRequest))
      return
   }

   simpleReq := newSimpleRequest(body)
   simpleReq.RegisterHook(func(simpleReq *simpleRequest) {
      req.SetResponse(simpleReq.Response())
   })
   key := body.Array[1].Text
   u.MakeRequest(key, simpleReq)
}
```

## 刷新SLOT

```go
func (u *upstream) loopRefreshSlots() {
   u.triggerSlotsRefresh() //启动时立即触发slot刷新 
   for {
      select {
      case <-u.quit: //关闭
         return
      case <-time.After(slotsRefFreq): //定时触发，2min
      case <-u.slotsRefreshCh: //其他触发，比如发布添加服务事件后，立即触发
      }
			
     //刷新slot
      u.refreshSlots() 

      t := time.NewTimer(slotsRefMinRate)
      select {
      case <-t.C:
      case <-u.quit:
         t.Stop()
         return
      }
   }
}
```



```go
func (u *upstream) triggerSlotsRefresh() { //触发slot刷新
   select {
   case u.slotsRefreshCh <- struct{}{}:
   default:
   }
   if u.slotsRefTriggerHook != nil {
      u.slotsRefTriggerHook() //执行钩子函数
   }
}
```

```go
func (u *upstream) refreshSlots() { //刷新redis cluster中的slot信息
   scope := u.stats.NewChild("slots_refresh")
   scope.Counter("total").Inc() //请求计数
   err := u.doSlotsRefresh()//刷新slot
   if err == nil { //响应成功
      u.logger.Debugf("refresh slots success")
      scope.Counter("success_total").Inc()
      u.slotsLastUpdateTime = time.Now()
      return
   }

   scope.Counter("failure_total").Inc() //响应失败计数
   u.logger.Warnf("fail to refresh slots: %v, will retry...", err)
   // 重试，立即触发 
   u.triggerSlotsRefresh()
   return
}
```

```go
func (u *upstream) doSlotsRefresh() error {
   v := newArray(
      *newBulkString("cluster"),
      *newBulkString("nodes"),
   )
  //构建请求
   req := newSimpleRequest(v)
	//随机选择redis实例
   addr, err := u.randomHost()
   if err != nil {
      return err
   }
  //发送请求
   u.MakeRequestToHost(addr, req)

   // 等待响应结果
   req.Wait()
   resp := req.Response()
   if resp.Type == Error {
      return errors.New(string(resp.Text))
   }
   if resp.Type != BulkString {
      return errInvalidClusterNodes
   }
  //解析响应结果
   insts, err := parseClusterNodes(string(resp.Text))
   if err != nil {
      return err
   }

   // 更新slot
   for _, inst := range insts {
      for _, slot := range inst.Slots {
         if slot < 0 || slot >= slotNum {
            continue
         }
         // NOTE: it's safe in x86-64 platform.
         u.slots[slot] = inst
      }
   }
   return nil
}
```

### newSimpleRequest

```go
func newSimpleRequest(v *RespValue) *simpleRequest {//构建请求
   return &simpleRequest{
      createdAt: time.Now(),
      body:      v,
      hooks:     make([]func(*simpleRequest), 0, 4), // pre-alloc
      done:      make(chan struct{}),
   }
}
```

### randomHost

```go
func (u *upstream) randomHost() (string, error) {//随机选择redis实例
   h := u.hosts.Random()
   if h == nil {
      return "", errors.New("no available host")
   }
   return h.Addr, nil
}
```

### MakeRequestToHost

```go
func (u *upstream) MakeRequestToHost(addr string, req *simpleRequest) {//发送请求
   // 请求度量信息
   u.stats.RqTotal.Inc()
   req.RegisterHook(func(req *simpleRequest) { ///注册钩子函数
      if req.Response().Type == Error {
         u.stats.RqFailureTotal.Inc()
      } else {
         u.stats.RqSuccessTotal.Inc()
      }
      u.stats.RqDurationMs.Record(uint64(req.Duration() / time.Millisecond))
   })

   select {
   case <-u.quit: //关闭
      req.SetResponse(newError(upstreamExited))
      return
   default:
   }
	//获取client连接
   c, err := u.getClient(addr)
   if err != nil {
      req.SetResponse(newError(err.Error()))
      return
   }
   //发送请求
   c.Send(req)
}
```

## 处理热点KEY

proc/redis/hotkey/collector.go

```go
func (c *Collector) Run(stop <-chan struct{}) { 
   collectTicker := time.NewTicker(c.collectInterval) //默认10s
   evictTicker := time.NewTicker(c.evictInterval) //默认1min
   defer func() {
      collectTicker.Stop()
      evictTicker.Stop()
   }()

   for {
      select {
      case <-stop: //upstream已关闭
         return
      case <-collectTicker.C:
         c.collect() //收集hot key
      case <-evictTicker.C:
         c.evictStale() // hot key进行衰减
      } 
   }
}
```

### CollectHotKey

```go
func (c *Collector) collect() {//收集hot key
   // get all keys and acutal visits in the current period.
   c.rwmu.RLock()
   accessedKeyNames := make(map[string]uint64)
  // counters:client -> counter
   for _, counter := range c.counters {//每个client对应一个counter
     for key, hitCount := range counter.Latch() { //key -> freq(读取频次，累加所得)
         accessedKeyNames[key] += hitCount
      }
   }
   c.rwmu.RUnlock()
   if len(accessedKeyNames) == 0 {
      return
   }

   //key -> logrithmCounter
   c.rwmu.RLock()
   curHotKeys := make(map[string]*logrithmCounter, len(c.keys))
   for _, key := range c.keys {
      curHotKeys[key.Name] = key.Counter
   }
   c.rwmu.RUnlock()

   // 维护新的hot key
   res := newSortedHotKeys(c.capacity) //默认50
   for keyName, counter := range curHotKeys { //当前维护的hot key
      visits := accessedKeyNames[keyName] //最近被访问的次数
      counter.ReaptIncr(visits) //概率性累加
      key := HotKey{Name: keyName, Counter: counter}
      res.Insert(key) 
      delete(accessedKeyNames, keyName)
   }

   for keyName, hitCount := range accessedKeyNames { //在之前的hot key不存在的key
      counter := new(logrithmCounter)
      counter.ReaptIncr(hitCount) //概率性计数
      key := HotKey{Name: keyName, Counter: counter}
      res.Insert(key)
   }

   // update the cached hot keys
   c.rwmu.Lock()
   c.keys = res.Data()
   c.rwmu.Unlock()
}
```



```go
func (c *logrithmCounter) ReaptIncr(times uint64) {
   // lazy init rand.
   if c.rnd == nil {
      c.rnd = rand.New(rand.NewSource(time.Now().UnixNano()))
   }

   for i := uint64(0); i < times; i++ {
      if c.val == 255 { //最大计数到255
         break
      }
      r := c.rnd.Float64() //生成随机数
      //当前的计数值乘以defaultLogFactor，再加1，然后取倒数
      p := 1.0 / float64(float64(c.val)*defaultLogFactor+1)
      if r < p { //大于随机数，计数值加1
         c.val++
      }
   }
   c.lut = nowInMinute() //更新的时间
}
```

### evictStale

```go
func (c *Collector) evictStale() {//驱逐陈旧的hot key
   c.rwmu.Lock()
   defer c.rwmu.Unlock()

   //超过1分钟未读取，将计数值减半
   curTimeInMinute := nowInMinute()
   for _, key := range c.keys {
      counter := key.Counter
      if curTimeInMinute > counter.LastUpdateTimeInMinute() {
         counter.Halve()
      }
   }

   // 清除计数值为0的key
   keys := make([]HotKey, 0, len(c.keys))
   for _, key := range c.keys {
      if key.Counter.Value() != 0 {
         keys = append(keys, key)
      }
   }
   c.keys = keys
}
```

# Session

## 创建会话

```go
func newSession(p *redisProc, conn net.Conn) *session {
   return &session{ //客户端与代理之间的会话
      p:              p,
      conn:           conn,
      enc:            newEncoder(conn, 8192),
      dec:            newDecoder(conn, 4096),
      processingReqs: make(chan *rawRequest, 32),
      quit:           make(chan struct{}),
      done:           make(chan struct{}),
   }
}
```

## 处理会话请求

```go
func (s *session) Serve() {
   writeDone := make(chan struct{})
   go func() {
      s.loopWrite() //返回响应
      s.conn.Close() //关闭连接
      s.doQuit() // 通知loopRead协程，退出循环
      close(writeDone)  //关闭通道
   }()

   s.loopRead() //读取客户端请求
   s.conn.Close() //关闭连接
   s.doQuit() //通知loopWrite退出循环
   <-writeDone //等待通道关闭
   close(s.done)
}
```

### 读取请求

```go
func (s *session) loopRead() {
   for {
      v, err := s.dec.Decode() //解码
      if err != nil {
         if err != io.EOF {
            s.p.logger.Warnf("loop read exit: %v", err)
         }
         return
      }
      req := newRawRequest(v) //封装请求
      s.p.handleRequest(req) //redisProc处理请求
      select {
      case s.processingReqs <- req: //存放处理中的请求
      case <-s.quit:
         return
      }
   }
}
```

### 返回响应

```go
func (s *session) loopWrite() {
   var (
      req *rawRequest
      err error
   )
   for {
      select {
      case <-s.quit:
         return
      case req = <-s.processingReqs: //获取处理中的请求
      }
      req.Wait() //等待请求处理完成
      resp := req.Response() //获取响应
      if err = s.enc.Encode(resp); err != nil { //编码
         goto FAIL
      }
      //如果还有未处理的请求，则延迟调用flush，减少系统调用次数
      if len(s.processingReqs) != 0 { 
         continue
      }
      // flush the buffer data to the underlying connection.
      if err = s.enc.Flush(); err != nil {
         goto FAIL
      }
   }
FAIL:
   s.p.logger.Warnf("loop write exit: %v", err)
}
```

# Client

## 创建获取client

```go
func (u *upstream) getClient(addr string) (*client, error) {//获取client，用于向redis发送请求
   c, ok := u.loadClients()[addr] //从map获取client;upstream维护与redis实例的连接
   if ok {
      return c, nil
   }

   // NOTE: fail fast when the addr is unreachable
   v, loaded := u.createClientCalls.LoadOrStore(addr, &createClientCall{
      done: make(chan struct{}),
   })
   call := v.(*createClientCall)
   if loaded {//多线程环境下，
     //线程A从clients和createClientCalls中没有获取到，开始创建createaclientcall
     //B线程从clients未获取到，但是从createClientCalls中获取到了，等待线程A创建client完成
      <-call.done
      return call.res, call.err
   }
  //创建client
   c, err := u.createClient(addr)
   call.res, call.err = c, err
  //创建client完成
   close(call.done) 
   return c, err
}
```



```go
func (u *upstream) createClient(addr string) (*client, error) { //创建client
   u.clientsMu.Lock()
   defer u.clientsMu.Unlock()

   select {
   case <-u.quit:
      return nil, errors.New(upstreamExited)
   default:
   }
   c, ok := u.loadClients()[addr]
   if ok {
      return c, nil
   }
		//建立连接
   conn, err := netutil.Dial("tcp", addr, *u.cfg.ConnectTimeout)
   if err != nil {
      return nil, err
   }
   options := []clientOption{
      withKeyCounter(u.hkc.AllocCounter(addr)), //每个redis实例的连接都创建counter
      withRedirectionCb(u.handleRedirection), //返回了ask、moved
      withClusterDownCb(u.handleClusterDown), //集群宕机
   }
   c, err = newClient(conn, u.cfg, u.logger, options...)
   if err != nil {
      return nil, err
   }

   // 启动client
   go func() {
      c.Start()
      u.removeClient(addr)
   }()
   u.addClientLocked(addr, c)
   return c, nil
}
```

### 初始化过滤器

```go
func (c *client) initFilters() error {
   filters := []Filter{
      newHotKeyFilter(c.keyCounter), //统计热点key
      newCompressFilter(c.cfg), //压缩
   }

   chain := newRequestFilterChain()
   for _, filter := range filters {
      chain.AddFilter(filter)
   }
   c.filter = chain
   return nil
}
```

### hotKeyFilter

```go
func (f *hotKeyFilter) Do(cmd string, req *simpleRequest) FilterStatus {
   key := f.extractKey(cmd, req.Body())
   if len(key) > 0 && f.counter != nil {
      f.counter.Incr(key)
   }
   return Continue
}
```

#### compressFilter

```go
func (f *compressFilter) Do(cmd string, req *simpleRequest) FilterStatus {
   // skip if compression config is null
   if f.cfg == nil ||
      f.cfg.GetRedisOption() == nil ||
      f.cfg.GetRedisOption().GetCompression() == nil {
      return Continue
   }

   // register decompression hook if needed.
   if _, ok := wkSkipCheckCmdsInDecps[cmd]; !ok {
      req.RegisterHook(func(request *simpleRequest) {
         f.Decompress(request.resp)
      })
   }

   // skip if the compression is not enabled.
   cfg := f.cfg.GetRedisOption().GetCompression()
   if !cfg.Enable {
      return Continue
   }
   // reject the banned commands.
   if _, ok := bannedCmdsInCps[cmd]; ok {
      errStr := fmt.Sprintf("ERR command '%s' is disabled in compress mode", cmd)
      req.SetResponse(newError(errStr))
      return Stop
   }
   f.Compress(cfg, cmd, req.body)
   return Continue
}
```

## 启动client

proc/redis/upstream.go

```go
func (c *client) Start() { //启动client
   writeDone := make(chan struct{})
   go func() {
      c.loopWrite()
      c.conn.Close()
      close(writeDone)
   }()

   c.loopRead()
   c.conn.Close()
   c.quitOnce.Do(func() {
      close(c.quit)
   })
   <-writeDone
   c.drainRequests()
   close(c.done)
}
```

### loopWrite

```go
func (c *client) loopWrite() { //将客户端的redis请求发送到redis实例
   var (
      req *simpleRequest
      err error
   )
   for {
      select {
      case <-c.quit://关闭
         return
      case req = <-c.pendingReqs:
      }

      switch c.filter.Do(req) { //执行过滤器
      case Continue:
      case Stop:
         continue
      }

      err = c.enc.Encode(req.Body()) //请求编码
      if err != nil {
         goto FAIL
      }

      if len(c.pendingReqs) == 0 {
         if err = c.enc.Flush(); err != nil { //发送
            goto FAIL
         }
      }

      select {
      case <-c.quit:
         return
      case c.processingReqs <- req: //正在处理中
      }
   }

FAIL:
   // req and error must not be nil
   req.SetResponse(newError(err.Error()))
   c.logger.Warnf("loop write exit: %v", err)
}
```

### loopRead

```go
func (c *client) loopRead() { //读取redis的响应结果
   for {
      resp, err := c.dec.Decode() //解码响应
      if err != nil {
         if err != io.EOF && !strings.Contains(err.Error(), "use of closed network connection") {
            c.logger.Warnf("loop read exit: %v", err)
         }
         return
      }

      req := <-c.processingReqs //获取对应的请求
      c.handleResp(req, resp)
   }
}
```



```go
func (c *client) handleResp(req *simpleRequest, v *RespValue) {
   if v.Type != Error {
      req.SetResponse(v)
      return
   }

   i := bytes.Index(v.Text, []byte(" "))
   var errPrefix []byte
   if i != -1 {
      errPrefix = v.Text[:i]
   }
   switch {
   case bytes.EqualFold(errPrefix, []byte(MOVED)),
      bytes.EqualFold(errPrefix, []byte(ASK)): //slot发生了迁移 
      if c.onRedirection != nil {
         c.onRedirection(req, v)
         return
      }
   case bytes.EqualFold(errPrefix, []byte(CLUSTERDOWN)): //slot所在的实例宕机
      if c.onClusterDown != nil {
         c.onClusterDown(req, v)
         return
      }
   }

   // 设置响应
   req.SetResponse(v)
}
```

proc/redis/request.go

```go
func (r *simpleRequest) SetResponse(resp *RespValue) {
   r.finishedAt = time.Now()
   r.resp = resp
   // call hook by order, LIFO
   for i := len(r.hooks) - 1; i >= 0; i-- { //执行注册的钩子函数
      hook := r.hooks[i]
      hook(r)
   }
   close(r.done) //关闭chan（读取完chan中的数据后，chan关闭，再读取返回0和false），唤醒请求
}
```

# Listener

监听客户端连接并处理其请求

proc/listener.go

```go
func (l *listener) Serve() error {
   ip := l.cfg.GetAddress().GetIp()
   port := l.cfg.GetAddress().GetPort()
   address := fmt.Sprintf("%s:%d", ip, port)

   var ln net.Listener
   for {
      select {
      case <-l.quit:
         return nil
      case <-l.drain:
         return nil
      default:
      }

      var err error
      ln, err = defaultListenFunc("tcp", address)
      if err == nil {
         break
      }

      l.Warnf("listen on %s failed: %v, will keep trying...", address, err)
      // TODO: use backoff algorithm to calculate sleep time.
      t := time.NewTimer(time.Millisecond * 500)
      select {
      case <-t.C:
      case <-l.drain:
         return nil
      case <-l.quit:
         return nil
      }
   }

   l.ln = ln
   l.Infof("start serving at %s", ln.Addr().String())
   l.serve()
   l.Infof("stop serving at %s, waiting all conns done", ln.Addr().String())

   l.connsWg.Wait()
   l.Infof("all conns done")
   close(l.done)
   return nil
}
```



```go
func (l *listener) serve() {
   var tempDelay time.Duration
   for {
      conn, err := l.ln.Accept() //客户端连接
      if err != nil {
         if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
            if tempDelay == 0 {
               tempDelay = 5 * time.Millisecond
            } else {
               tempDelay *= 2
            }
            if max := 1 * time.Second; tempDelay > max {
               tempDelay = max
            }
            l.Warnf("accept failed: %v; retrying in %s", err, tempDelay)
            timer := time.NewTimer(tempDelay)
            select {
            case <-timer.C:
            case <-l.quit:
               timer.Stop()
               return
            }
            continue
         }

         select {
         case <-l.drain:
            return
         case <-l.quit:
            return
         default:
         }
         l.Warnf("done serving; accept failed: %v", err)
         return
      }

      l.connsWg.Add(1)
      go func(conn net.Conn) {
         l.handleRawConn(conn)
         l.connsWg.Done()
      }(conn)
   }
}
```

## handleRawConn

```go
func (l *listener) handleRawConn(rawConn net.Conn) {
   conn := l.wrapRawConn(rawConn) //包装con
   connCreatedAt := time.Now() 

   if !l.addConn(conn) { //维护连接
      conn.Close() //超过连接数，关闭连接
      return
   }

   l.Debugf("%s -> %s created", conn.RemoteAddr(), l.ln.Addr().String())
   defer func() {
      conn.Close()
      l.removeConn(conn)
      l.Debugf("%s -> %s finished, duration: %s", conn.RemoteAddr(), l.ln.Addr().String(), time.Since(connCreatedAt).String())
   }()

   if l.connHandleFn == nil {
      l.Warnf("conn handle fn is nil, will close conn immediately")
      return
   }
   l.connHandleFn(conn)  //处理连接上的读写,不同服务对应不同的connHanleFn
}
```



```go
func (l *listener) wrapRawConn(rawConn net.Conn) net.Conn {
   c := netutil.New(rawConn)
   // TODO(kirk91): use hook to replace it.
   stats := &netutil.Stats{
      WriteTotal: l.stats.CxTxBytesTotal,
      ReadTotal:  l.stats.CxRxBytesTotal,
      Duration:   l.stats.CxLengthSec,
   }
   c.SetStats(stats)
   return c
}
```



```go
func (l *listener) addConn(conn net.Conn) bool {
   l.mu.Lock()
   defer l.mu.Unlock()
   if l.conns == nil {
      return false
   }
   if l.connsLimit() { //限制连接数
      l.stats.CxRestricted.Inc()
      l.Warnf("connections limit, %s -> %s, will close", conn.RemoteAddr().String(), l.ln.Addr().String())
      return false
   }
   l.conns[conn] = struct{}{}
   l.stats.CxTotal.Inc()
   l.stats.CxActive.Inc()
   return true
}
```

## handleConn

proc/redis/redis.go

```go
func (p *redisProc) handleConn(conn net.Conn) {//处理redis请求
   s := newSession(p, conn) //创建session
   s.Serve() //处理读写
}
```

proc/redis/session.go

```go
func newSession(p *redisProc, conn net.Conn) *session {
   return &session{
      p:              p,
      conn:           conn,
      enc:            newEncoder(conn, 8192),
      dec:            newDecoder(conn, 4096),
      processingReqs: make(chan *rawRequest, 32),
      quit:           make(chan struct{}),
      done:           make(chan struct{}),
   }
}
```



```go
func (s *session) Serve() {
   writeDone := make(chan struct{})
   go func() {
      s.loopWrite()
      s.conn.Close()
      s.doQuit()
      close(writeDone)
   }()

   s.loopRead()
   s.conn.Close()
   s.doQuit()
   <-writeDone
   close(s.done)
}
```

### loopRead

```go
func (s *session) loopRead() {//读取客户端请求
   for {
      v, err := s.dec.Decode() //解码请求
      if err != nil {
         if err != io.EOF {
            s.p.logger.Warnf("loop read exit: %v", err)
         }
         return
      }

      req := newRawRequest(v) //创建rawRequest
      s.p.handleRequest(req) //处理请求

      select {
      case s.processingReqs <- req: //正在处理中，等待响应
      case <-s.quit:
         return
      }
   }
}
```

#### handleRequest

proc/redis/redis.go

```go
func (p *redisProc) handleRequest(req *rawRequest) {
   p.stats.Downstream.RqTotal.Inc() //请求计数
   req.RegisterHook(func(req *rawRequest) { //注册钩子函数，返回响应之后执行
      switch req.Response().Type {
      case Error:
         p.stats.Downstream.RqFailureTotal.Inc()
      default:
         p.stats.Downstream.RqSuccessTotal.Inc()
      }
      p.stats.Downstream.RqDurationMs.Record(uint64(req.Duration() / time.Millisecond))
   })

   // check
   if !req.IsValid() { //验证请求的合法性
      req.SetResponse(newError(invalidRequest))
      return
   }

   // 查找对应的handler
   cmd := string(req.Body().Array[0].Text)
   hdlr, ok := p.findHandler(cmd)
   if !ok {
      // unsupported command
      req.SetResponse(newError(fmt.Sprintf("ERR unsupported command '%s'", cmd)))
      return
   }

   cmdStats := hdlr.stats
   cmdStats.Total.Inc()
   req.RegisterHook(func(req *rawRequest) { //注册钩子函数，度量信息
      switch req.Response().Type {
      case Error:
         cmdStats.Error.Inc()
      default:
         cmdStats.Success.Inc()
      }
      latency := uint64(req.Duration() / time.Microsecond)
      cmdStats.LatencyMicros.Record(latency)
      if latency > p.cfg.slowReqThresholdInMicros {
         p.stats.Counter("rq_slow_total").Inc()
      }
   })
   hdlr.handle(p.u, req) //处理请求
}
```

#### handleSimpleCommand	

proc/redis/handler.go

```go
func handleSimpleCommand(u *upstream, req *rawRequest) { //处理简单命令
   body := req.Body()
   if len(body.Array) < 2 {
      req.SetResponse(newError(invalidRequest))
      return
   }

   simpleReq := newSimpleRequest(body)
   simpleReq.RegisterHook(func(simpleReq *simpleRequest) {
      req.SetResponse(simpleReq.Response())
   })
   key := body.Array[1].Text
   u.MakeRequest(key, simpleReq)
}
```

#### handleSumResultCommand

```go
func handleSumResultCommand(u *upstream, req *rawRequest) {
   sumResultReq, err := newSumResultRequest(req)
   if err != nil {
      req.SetResponse(newError(err.Error()))
      return
   }

   simpleReqs := sumResultReq.Split()
   for i := 0; i < len(simpleReqs); i++ {
      simpleReq := simpleReqs[i]
      key := simpleReq.Body().Array[1].Text
      u.MakeRequest(key, simpleReq)
   }
}
```

##### newSumResultRequest

```go
func newSumResultRequest(raw *rawRequest) (*sumResultRequest, error) {
   v := raw.Body().Array
   if len(v) < 2 {
      return nil, errors.New(invalidRequest)
   }
   r := &sumResultRequest{
      raw:       raw,
      childWait: atomic.NewInt32(0),
   }
   return r, nil
}
```

##### Split

proc/redis/request.go

```go
func (r *sumResultRequest) Split() []*simpleRequest {
   if r.children != nil {
      return r.children
   }

   //切分成简单请求
   v := r.raw.Body().Array
   sreqs := make([]*simpleRequest, 0, len(v)-1)
   for i := 1; i < len(v); i++ {
      sv := &RespValue{
         Type: Array,
         Array: []RespValue{
            v[0],
            v[i],
         },
      }
      sreq := newSimpleRequest(sv)
      sreq.RegisterHook(r.onChildDone) //注册钩子
      sreqs = append(sreqs, sreq)
   }

   r.children = sreqs
   r.childWait.Store(int32(len(sreqs)))//更新为切分之后简单请求的个数
   return sreqs
}
```

```go
func (r *sumResultRequest) onChildDone(simpleReq *simpleRequest) {
   wait := r.childWait.Dec()//简单请求返回响应减1
   if wait == 0 {//唤醒阻塞的请求
      r.setResponse()
   }
}
```

##### MakeRequest

proc/redis/upstream.go

```go
func (u *upstream) MakeRequest(routingKey []byte, req *simpleRequest) {
   addr, err := u.chooseHost(routingKey, req) //选择redis实例
   if err != nil {
      req.SetResponse(newError(err.Error()))
      return
   }
   u.MakeRequestToHost(addr, req)
}
```

```go
func (u *upstream) chooseHost(routingKey []byte, req *simpleRequest) (string, error) {
   hash := crc16(hashtag(routingKey)) //计算key或者hashtag的哈希吗
   inst := u.slots[hash&(slotNum-1)]

   if inst == nil {
     //prxoy刚刚启动，尚未拉取集群信息
     //redis集群出现了临时故障
      return u.randomHost() //随机选择一个redis实例
   }

   if !req.IsReadOnly() {
      return inst.Addr, nil
   }

   // read-only requests
   var candidates []string
   readStrategy := redis.ReadStrategy_MASTER
   if option := u.cfg.GetRedisOption(); option != nil {
      readStrategy = option.ReadStrategy
   }
   switch readStrategy {
   case redis.ReadStrategy_MASTER:
      candidates = append(candidates, inst.Addr)
   case redis.ReadStrategy_BOTH:
      candidates = append(candidates, inst.Addr)
      fallthrough
   case redis.ReadStrategy_REPLICA:
      for _, replica := range inst.Replicas {
         candidates = append(candidates, replica.Addr)
      }
   }

   if len(candidates) == 0 {
      candidates = append(candidates, inst.Addr)
   }
   i := 0
   l := len(candidates)
   if l > 1 {
      // The concurrency-safe rand source does not scale well to multiple cores,
      // so we use currrent unix time in nanoseconds to avoid it. As a result these
      // generated values are sequential rather than random, but are acceptable.
      i = int(time.Now().UnixNano()) % l
   }
   return candidates[i], nil
}
```

### loopWrite

```go
func (s *session) loopWrite() { //将响应发送给客户端
   var (
      req *rawRequest
      err error
   )
   for {
      select {
      case <-s.quit:
         return
      case req = <-s.processingReqs:
      }
			//等待响应
      req.Wait()
      resp := req.Response()
      if err = s.enc.Encode(resp); err != nil { //编码
         goto FAIL
      }

      if len(s.processingReqs) != 0 {//如果还有正在处理中的请求，延迟调用flush，减少系统调用的次数
         continue
      }

      if err = s.enc.Flush(); err != nil { //返回给客户端响应
         goto FAIL
      }
   }	

FAIL: //编码异常、flush异常
   s.p.logger.Warnf("loop write exit: %v", err)
}
```

# References

[官方文档](https://samaritan-proxy.github.io/docs/)

[详解 Samaritan——饿了么最新开源的透明代理](https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487045&amp;idx=1&amp;sn=846c3fd05a52378cb22f623cc05d564c&source=41)

[如何快速定位 Redis 热点 key](https://mp.weixin.qq.com/s/rZs-oWBGGYtNKLMpI0-tXw)

