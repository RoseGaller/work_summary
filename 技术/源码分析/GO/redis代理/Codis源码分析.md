# Codis源码分析


# 组件

Codis-proxy

客户端连接的 Redis 代理服务, 实现了 Redis 协议,codis-proxy 本身是无状态的.灵活方便的进行扩缩容

Codis-Config

Codis 的管理工具, 支持包括, 添加/删除 Redis 节点, 添加/删除 Proxy 节点, 发起数据迁移等操作.

Codis-Dashboard

直接在浏览器上观察 Codis 集群的运行状态，页面中对集

Codis-Server

 Codis 项目维护的一个 Redis 分支, 基于 2.8.21 开发, 加入了 slot 的支持和原子的数据迁移指令.

Zookeeper/Etcd

放数据路由表和 codis-proxy 节点的元信息, codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy.

# 启动入口

codis/cmd/proxy/main.go:

## 初始化日志级别

```go
func init() { //init函数在main函数之前自动执行
   log.SetLevel(log.LEVEL_INFO)
}
```

```go
//全局变量
var (
   cpus       = 2
   addr       = ":9000" //处理外部请求的端口号
   httpAddr   = ":9001" //对外暴露http协议修改log level
   configFile = "config.ini"
)
```

## 动态修改日志级别

```go
//修改log level
http.HandleFunc("/setloglevel", handleSetLogLevel)
go func() {
   err := http.ListenAndServe(httpAddr, nil)
   log.PanicError(err, "http debug server quit")
}()
```

```go
func handleSetLogLevel(w http.ResponseWriter, r *http.Request) {
   r.ParseForm()
   setLogLevel(r.Form.Get("level"))
}
```

```go
func setLogLevel(level string) {
   level = strings.ToLower(level)
   var l = log.LEVEL_INFO
   switch level {
   case "error":
      l = log.LEVEL_ERROR
   case "warn", "warning":
      l = log.LEVEL_WARN
   case "debug":
      l = log.LEVEL_DEBUG
   case "info":
      fallthrough
   default:
      level = "info"
      l = log.LEVEL_INFO
   }
   log.SetLevel(l)
   log.Infof("set log level to <%s>", level)
}
```



```go
//读取配置文件config.ini
conf, err := proxy.LoadConf(configFile)
if err != nil {
   log.PanicErrorf(err, "load config failed")
}
```



```go
c := make(chan os.Signal, 1) //存放关闭事件
signal.Notify(c, os.Interrupt, syscall.SIGTERM, os.Kill)
//创建server
s := proxy.New(addr, httpAddr, conf)
defer s.Close() //关闭服务，释放资源（defer相当于Java中的Finally）

//启动协程，监听关闭事件
go func() {
   <-c	//阻塞，获取关闭事件
   log.Info("ctrl-c or SIGTERM found, bye bye...")
   s.Close() //关闭服务
}()
//请求dashboard注册proxy
time.Sleep(time.Second)
if err := s.SetMyselfOnline(); err != nil { //向dashboard发送上线请求
   log.WarnError(err, "mark myself online fail, you need mark online manually by dashboard")
}
//阻塞，直到server关闭
s.Join() //s.wait.Wait() （sync.WaitGroup类似Java中的CountDownLatch）
```

## 读取配置信息

```go
func LoadConf(configFile string) (*Config, error) {
   c := cfg.NewCfg(configFile)
   if err := c.Load(); err != nil {
      log.PanicErrorf(err, "load config '%s' failed", configFile)
   }

   conf := &Config{} // 等价于conf = new(Config)
   conf.productName, _ = c.ReadString("product", "test")
   if len(conf.productName) == 0 {
      log.Panicf("invalid config: product entry is missing in %s", configFile)
   }
   conf.dashboardAddr, _ = c.ReadString("dashboard_addr", "")
   if conf.dashboardAddr == "" {
      log.Panicf("invalid config: dashboard_addr is missing in %s", configFile)
   }
   conf.zkAddr, _ = c.ReadString("zk", "")
   if len(conf.zkAddr) == 0 {
      log.Panicf("invalid config: need zk entry is missing in %s", configFile)
   }
   conf.zkAddr = strings.TrimSpace(conf.zkAddr)
   conf.passwd, _ = c.ReadString("password", "")

   conf.proxyId, _ = c.ReadString("proxy_id", "")
   if len(conf.proxyId) == 0 {
      log.Panicf("invalid config: need proxy_id entry is missing in %s", configFile)
   }

   conf.proto, _ = c.ReadString("proto", "tcp")
   conf.provider, _ = c.ReadString("coordinator", "zookeeper")

   loadConfInt := func(entry string, defval int) int {
      v, _ := c.ReadInt(entry, defval)
      if v < 0 {
         log.Panicf("invalid config: read %s = %d", entry, v)
      }
      return v
   }

   conf.pingPeriod = loadConfInt("backend_ping_period", 5)
   conf.maxTimeout = loadConfInt("session_max_timeout", 1800)
   conf.maxBufSize = loadConfInt("session_max_bufsize", 131072)
   conf.maxPipeline = loadConfInt("session_max_pipeline", 1024)
   conf.zkSessionTimeout = loadConfInt("zk_session_timeout", 30000)
   if conf.zkSessionTimeout <= 100 {
      conf.zkSessionTimeout *= 1000
      log.Warn("zkSessionTimeout is to small, it is ms not second")
   }
   return conf, nil
}
```

## 实例化server

codis/pkg/proxy/proxy.go

```go
func New(addr string, debugVarAddr string, conf *Config) *Server {
   log.Infof("create proxy with config: %+v", conf)
   proxyHost := strings.Split(addr, ":")[0]
   debugHost := strings.Split(debugVarAddr, ":")[0]
   hostname, err := os.Hostname()
   if err != nil {
      log.PanicErrorf(err, "get host name failed")
   }
   if proxyHost == "0.0.0.0" || strings.HasPrefix(proxyHost, "127.0.0.") || proxyHost == "" {
      proxyHost = hostname
   }
   if debugHost == "0.0.0.0" || strings.HasPrefix(debugHost, "127.0.0.") || debugHost == "" {
      debugHost = hostname
   }
   //创建Server
   s := &Server{conf: conf, lastActionSeq: -1, groups: make(map[int]int)}
   //与zookeepe建立连接
   s.topo = NewTopo(conf.productName, conf.zkAddr, conf.fact, conf.provider, conf.zkSessionTimeout)
   s.info.Id = conf.proxyId
   s.info.State = models.PROXY_STATE_OFFLINE //初始状态
   s.info.Addr = proxyHost + ":" + strings.Split(addr, ":")[1]
   s.info.DebugVarAddr = debugHost + ":" + strings.Split(debugVarAddr, ":")[1]
   s.info.Pid = os.Getpid()
   s.info.StartAt = time.Now().String()
   s.kill = make(chan interface{})
   log.Infof("proxy info = %+v", s.info)
   //监听端口
   if l, err := net.Listen(conf.proto, addr); err != nil {
      log.PanicErrorf(err, "open listener failed")
   } else {
      s.listener = l
   }
   //创建router
   s.router = router.NewWithAuth(conf.passwd)
   //存放事件
   s.evtbus = make(chan interface{}, 1024) 
   //注册到zookeeper
   s.register()
   s.wait.Add(1)
   go func() {
      defer s.wait.Done() //协程退出前执行
      s.serve()  //对外提供服务
   }()
   return s
}
```

pkg/proxy/router/router.go

```java
const MaxSlotNum = models.DEFAULT_SLOT_NUM //声明常量，默认slot数量1024
```

```go
func New() *Router {
   return NewWithAuth("")
}
```

```go
func NewWithAuth(auth string) *Router {
   s := &Router{
      auth: auth,
      pool: make(map[string]*SharedBackendConn), //维护与每个redis实例的连接
   }
   for i := 0; i < len(s.slots); i++ {
      s.slots[i] = &Slot{id: i} //初始化slot
   }
   return s
}
```

# 对外提供服务

pkg/proxy/proxy.go

```go
func (s *Server) serve() {
   defer s.close()
   if !s.waitOnline() { //阻塞直到proxy注册zk成功
      return
   }
   //注册watch事件
   s.rewatchNodes()
   //填充1024slot
   for i := 0; i < router.MaxSlotNum; i++ {
      s.fillSlot(i)
   }
   log.Info("proxy is serving")
   go func() {
      defer s.close()
      s.handleConns() //处理客户端请求
   }()
   //处理服务端各种事件
   s.loopEvents()
}
```

## 阻塞直到上线

pkg/proxy/proxy.go

```go
func (s *Server) waitOnline() bool {
   for {
      //从zk获取proxy信息
      info, err := s.topo.GetProxyInfo(s.info.Id) 
      if err != nil {
         log.PanicErrorf(err, "get proxy info failed: %s", s.info.Id)
      }
      switch info.State {
      case models.PROXY_STATE_MARK_OFFLINE: //下线
         log.Infof("mark offline, proxy got offline event: %s", s.info.Id)
         s.markOffline() //处理下线
         return false
      case models.PROXY_STATE_ONLINE: //上线
         s.info.State = info.State
         log.Infof("we are online: %s", s.info.Id)
         s.rewatchProxy() //注册watcher
         return true
      }
      select {
      case <-s.kill: //关闭事件
         log.Infof("mark offline, proxy is killed: %s", s.info.Id)
         s.markOffline() //标记下线
         return false
      default:
      }
      log.Infof("wait to be online: %s", s.info.Id)
      time.Sleep(3 * time.Second) //休眠3秒
   }
}
```

## 注册watch事件

pkg/proxy/proxy.go

```go
func (s *Server) rewatchNodes() []string {
   nodes, err := s.topo.WatchChildren(models.GetWatchActionPath(s.topo.ProductName), s.evtbus) //将触发的事件放入evtbus
   if err != nil {
      log.PanicErrorf(err, "watch children failed")
   }
   return nodes
}
```

## 填充槽位

slot信息通过dashboard注册到zk

1、从zookeeper获取slot的信息以及组信息（主、副本）

2、维护redis连接

pkg/proxy/proxy.go

```go
func (s *Server) fillSlot(i int) {
   //获取槽位对应的redis实例、group信息
   slotInfo, slotGroup, err := s.topo.GetSlotByIndex(i)
   if err != nil {
      log.PanicErrorf(err, "get slot by index failed", i)
   }
   var from string
   //获取该槽位的的master地址
   var addr = groupMaster(*slotGroup)
   if slotInfo.State.Status == models.SLOT_STATUS_MIGRATE { //正在数据迁移
      fromGroup, err := s.topo.GetGroup(slotInfo.State.MigrateStatus.From) //源组
      if err != nil {
         log.PanicErrorf(err, "get migrate from failed")
      }
      //获取源组的master
      from = groupMaster(*fromGroup) 
      if from == addr {
         log.Panicf("set slot %04d migrate from %s to %s", i, from, addr)
      }
   }
  //slot->groupId
   s.groups[i] = slotInfo.GroupId
  //router填充slot
   s.router.FillSlot(i, addr, from,
      slotInfo.State.Status == models.SLOT_STATUS_PRE_MIGRATE)
}
```

pkg/proxy/router/router.go

```go
func (s *Router) FillSlot(i int, addr, from string, lock bool) error {
   s.mu.Lock() //获取锁
   defer s.mu.Unlock() //执行完方法释放锁
   if s.closed {
      return errClosedRouter
   }
   s.fillSlot(i, addr, from, lock)
   return nil
}
```

```go
func (s *Router) fillSlot(i int, addr, from string, lock bool) {
   if !s.isValidSlot(i) { //验证槽位的合法性 i >= 0 && i < len(s.slots)
      return
   }
   slot := s.slots[i]
   slot.blockAndWait()
  
	//连接不为空，并且已经关闭，从router的连接池中移除
   s.putBackendConn(slot.backend.bc)
   s.putBackendConn(slot.migrate.bc)
   //清空此槽位的信息
   slot.reset()

   if len(addr) != 0 {
      xx := strings.Split(addr, ":")
      if len(xx) >= 1 {
         slot.backend.host = []byte(xx[0]) //master地址
      }
      if len(xx) >= 2 {
         slot.backend.port = []byte(xx[1]) //master端口
      }
      slot.backend.addr = addr
     //创建SharedBackendConn，存放到连接池，并启动协程处理redis网络IO
      slot.backend.bc = s.getBackendConn(addr)
   }
   if len(from) != 0 { //槽位发生了迁移，需要迁移数据
      slot.migrate.from = from
      slot.migrate.bc = s.getBackendConn(from)
   }

   if !lock {
      slot.unblock()
   }

   if slot.migrate.bc != nil {
      log.Infof("fill slot %04d, backend.addr = %s, migrate.from = %s",
         i, slot.backend.addr, slot.migrate.from)
   } else {
      log.Infof("fill slot %04d, backend.addr = %s",
         i, slot.backend.addr)
   }
}
```

### 移除连接池中的连接

pkg/proxy/router/router.go

```go
func (s *Router) putBackendConn(bc *SharedBackendConn) {
  //连接不为空，并且已经关闭，从router的连接池中移除
   if bc != nil && bc.Close() {
      delete(s.pool, bc.Addr())
   }
}
```

```go
func (s *SharedBackendConn) Close() bool {
   s.mu.Lock()
   defer s.mu.Unlock()
   if s.refcnt <= 0 { //已经关闭
      log.Panicf("shared backend conn has been closed, close too many times")
   }
   if s.refcnt == 1 { //
      s.BackendConn.Close()
   }
   s.refcnt-- //引用计数
   return s.refcnt == 0 //为0，表明没有被引用
}
```

### 创建SharedBackendConn

CodisLabs/codis/pkg/proxy/router/router.go

```go
func (s *Router) getBackendConn(addr string) *SharedBackendConn {
   bc := s.pool[addr] //根据地址获取SharedBackendConn
   if bc != nil {
      bc.IncrRefcnt() //引用+1
   } else {
     //创建SharedBackendConn
      bc = NewSharedBackendConn(addr, s.auth)
     //放入连接池
      s.pool[addr] = bc
   }
   return bc
}
```



```go
func NewSharedBackendConn(addr, auth string) *SharedBackendConn {
  //创建时，自动引用+1
   return &SharedBackendConn{BackendConn: NewBackendConn(addr, auth), refcnt: 1}
}
```



```go
func NewBackendConn(addr, auth string) *BackendConn {
   bc := &BackendConn{
      addr: addr, auth: auth,
      input: make(chan *Request, 1024), //存放redis请求
   }
   //启动协程，向redis发送请求、接收响应
   go bc.Run()
   return bc
}
```

### 发送请求到redis

pkg/proxy/router/backend.go

```go
func (bc *BackendConn) Run() {
   log.Infof("backend conn [%p] to %s, start service", bc, bc.addr)
   for k := 0; ; k++ {
      err := bc.loopWriter()
      if err == nil {
         break
      } else {//处理网络IO出现异常，将积压请求的响应设置为失败
         for i := len(bc.input); i != 0; i-- {
            r := <-bc.input
            bc.setResponse(r, nil, err)
         }
      }
      //开启新一轮的循环
      log.WarnErrorf(err, "backend conn [%p] to %s, restart [%d]", bc, bc.addr, k)
      //休眠
      time.Sleep(time.Millisecond * 50)
   }
   log.Infof("backend conn [%p] to %s, stop and exit", bc, bc.addr)
}
```

pkg/proxy/router/backend.go

```java
func (bc *BackendConn) loopWriter() error {
   r, ok := <-bc.input //获取客户端请求
   if ok {
      c, tasks, err := bc.newBackendReader() //创建连接、tasks用于存放已经发送给redis的请求
      if err != nil {
         return bc.setResponse(r, nil, err)
      }
      defer close(tasks)
			//刷新策略
      p := &FlushPolicy{
         Encoder:     c.Writer,
         MaxBuffered: 64, //累积数据
         MaxInterval: 300, //时间
      }
      for ok {
         var flush = len(bc.input) == 0 //当前暂存的客户端请求为0，可以将socket缓冲区的数据强制一次性发送出去，避免多次系统调用
         if bc.canForward(r) { //可以直接发放
            if err := p.Encode(r.Resp, flush); err != nil { //编码
               return bc.setResponse(r, nil, err) //编码发送失败，返回error
            }
            tasks <- r  //发送成功的请求放入tasks中
         } else {
            if err := p.Flush(flush); err != nil { //socket缓冲区的数据发送出去
               return bc.setResponse(r, nil, err)
            }
            bc.setResponse(r, nil, ErrFailedRequest) //标记当前请求失败
         } 

         r, ok = <-bc.input //获取请求
      }
   }
   return nil
}
```

#### 与redis建立连接

pkg/proxy/router/backend.go

```go
func (bc *BackendConn) newBackendReader() (*redis.Conn, chan<- *Request, error) {
   c, err := redis.DialTimeout(bc.addr, 1024*512, time.Second) //建立连接
   if err != nil {
      return nil, nil, err
   }
   //设置读写超时时间
   c.ReaderTimeout = time.Minute
   c.WriterTimeout = time.Minute

   if err := bc.verifyAuth(c); err != nil { //认证
      c.Close()
      return nil, nil, err
   }
   //存放等待redis响应的客户端请求
   tasks := make(chan *Request, 4096)
   go func() {
      defer c.Close()
      for r := range tasks {
         resp, err := c.Reader.Decode() //读取redis响应结果
         bc.setResponse(r, resp, err) //设置请求的响应结果
         if err != nil {
            // close tcp to tell writer we are failed and should quit
            c.Close()
         }
      }
   }()
   return c, tasks, nil
}
```

#### 设置响应结果，唤醒阻塞请求

pkg/proxy/router/backend.go

```go
func (bc *BackendConn) setResponse(r *Request, resp *redis.Resp, err error) error {
   r.Response.Resp, r.Response.Err = resp, err
   if err != nil && r.Failed != nil {
      r.Failed.Set(true)
   }
   if r.Wait != nil {
      r.Wait.Done() //唤醒阻塞的请求
   }
   if r.slot != nil {
      r.slot.Done() 
   }
   return err
}
```

## 处理客户端连接

pkg/proxy/proxy.go

```go
func (s *Server) handleConns() {
   ch := make(chan net.Conn, 4096) //设置初始容量，存放客户端连接
   defer close(ch) //关闭chan
   // 为客户端连接创建Session
   go func() {
      for c := range ch {
         x := router.NewSessionSize(c, s.conf.passwd, s.conf.maxBufSize, s.conf.maxTimeout)
         go x.Serve(s.router, s.conf.maxPipeline)
      }
   }()
  //监听客户端连接
   for {
      c, err := s.listener.Accept()  //监听连接
      if err != nil {
         if ne, ok := err.(net.Error); ok && ne.Temporary() {
            log.WarnErrorf(err, "[%p] proxy accept new connection failed, get temporary error", s)
            time.Sleep(time.Millisecond*10)
            continue
         }
         log.WarnErrorf(err, "[%p] proxy accept new connection failed, get non-temporary error, must shutdown", s)
         return
      } else {
         ch <- c //将连接放入channel
      }
   }
}
```

## 客户端编解码

pkg/proxy/router/session.go

```go
func NewSessionSize(c net.Conn, auth string, bufsize int, seconds int) *Session {
   s := &Session{CreateUnix: time.Now().Unix(), auth: auth}
   s.Conn = redis.NewConnSize(c, bufsize)
   s.Conn.ReaderTimeout = time.Second * time.Duration(seconds)
   s.Conn.WriterTimeout = time.Second * 30
   log.Infof("session [%p] create: %s", s, s)
   return s
}
```

```go
func NewConnSize(sock net.Conn, bufsize int) *Conn {
   conn := &Conn{Sock: sock}
   conn.Reader = NewDecoderSize(&connReader{Conn: conn}, bufsize) //负责解码客户端请求
   conn.Writer = NewEncoderSize(&connWriter{Conn: conn}, bufsize) //负责编码客户端响应
   return conn
}
```

```go
func NewDecoderSize(r io.Reader, size int) *Decoder {
   br, ok := r.(*bufio.Reader) //强转为bufio.Reader
   if !ok { //强转失败
      br = bufio.NewReaderSize(r, size) //创建bufio.Reader实例
   }
   return &Decoder{Reader: br}
}
```

```go
func NewEncoderSize(w io.Writer, size int) *Encoder {
   bw, ok := w.(*bufio.Writer)
   if !ok {
      bw = bufio.NewWriterSize(w, size)
   }
   return &Encoder{Writer: bw}
}
```

## 处理客户端读写

pkg/proxy/router/session.go

```go
func (s *Session) Serve(d Dispatcher, maxPipeline int) { //router实现了方法Dispatch
   var errlist errors.ErrorList
   defer func() {
      if err := errlist.First(); err != nil {
         log.Infof("session [%p] closed: %s, error = %s", s, s, err)
      } else {
         log.Infof("session [%p] closed: %s, quit", s, s)
      }
      s.Close()
   }()
  //最大1024，存放客户端请求及响应信息
   tasks := make(chan *Request, maxPipeline) 
   //将redis返回的响应信息返回给客户端
   go func() {
      defer func() {
         for _ = range tasks {
         }
      }()
      if err := s.loopWriter(tasks); err != nil {
         errlist.PushBack(err)
      }
      s.Close()
   }()
   defer close(tasks) 
   //读取客户端请求，并进行处理，将处理结果放入tasks
   if err := s.loopReader(tasks, d); err != nil {
     	//读取出现异常
      errlist.PushBack(err)
   }
}
```

### 客户端请求处理

pkg/proxy/router/session.go

```go
func (s *Session) loopReader(tasks chan<- *Request, d Dispatcher) error {
   if d == nil {
      return errors.New("nil dispatcher")
   }
   for !s.quit {
      resp, err := s.Reader.Decode() //读取客户端请求
      if err != nil {//出现异常直接退出
         return err
      }
      r, err := s.handleRequest(resp, d) //处理客户端请求
      if err != nil { //出现异常直接退出
         return err
      } else {
         tasks <- r //处理结果放入tasks
      }
   }
   return nil
}
```

#### 解码

```go
func (d *Decoder) Decode() (*Resp, error) {
   if d.Err != nil {
      return nil, errors.Trace(ErrFailedDecoder)
   }
   r, err := d.decodeResp(0)
   if err != nil {
      d.Err = err
   }
   return r, err
}
```

```go
func (d *Decoder) decodeResp(depth int) (*Resp, error) {
   b, err := d.ReadByte()//从客户端连接读取数据存放到缓冲区，返回一个字节
   if err != nil {
      return nil, errors.Trace(err)
   }
   switch t := RespType(b); t { //请求类型
   case TypeString, TypeError, TypeInt:
      r := &Resp{Type: t}
      r.Value, err = d.decodeTextBytes() //请求数据
      return r, err
   case TypeBulkBytes:
      r := &Resp{Type: t}
      r.Value, err = d.decodeBulkBytes()
      return r, err
   case TypeArray:
      r := &Resp{Type: t}
      r.Array, err = d.decodeArray(depth)
      return r, err
   default:
      if depth != 0 {
         return nil, errors.Errorf("bad resp type %s", t)
      }
      if err := d.UnreadByte(); err != nil {
         return nil, errors.Trace(err)
      }
      r := &Resp{Type: TypeArray}
      r.Array, err = d.decodeSingleLineBulkBytesArray()
      return r, err
   }
}
```

pkg/proxy/router/session.go

```go
var (
   blacklist = make(map[string]bool) //存放不支持的命令
)
func init() {
	for _, s := range []string{
		"KEYS", "MOVE", "OBJECT", "RENAME", "RENAMENX", "SCAN", "BITOP", "MSETNX", "MIGRATE", "RESTORE",
		"BLPOP", "BRPOP", "BRPOPLPUSH", "PSUBSCRIBE", "PUBLISH", "PUNSUBSCRIBE", "SUBSCRIBE", "RANDOMKEY",
		"UNSUBSCRIBE", "DISCARD", "EXEC", "MULTI", "UNWATCH", "WATCH", "SCRIPT",
		"BGREWRITEAOF", "BGSAVE", "CLIENT", "CONFIG", "DBSIZE", "DEBUG", "FLUSHALL", "FLUSHDB",
		"LASTSAVE", "MONITOR", "SAVE", "SHUTDOWN", "SLAVEOF", "SLOWLOG", "SYNC", "TIME",
		"SLOTSINFO", "SLOTSDEL", "SLOTSMGRTSLOT", "SLOTSMGRTONE", "SLOTSMGRTTAGSLOT", "SLOTSMGRTTAGONE", "SLOTSCHECK",
	} {
		blacklist[s] = true
	}
}
```

```go
func (s *Session) handleRequest(resp *redis.Resp, d Dispatcher) (*Request, error) {
   opstr, err := getOpStr(resp)
   if err != nil {
      return nil, err
   }
   //验证是否支持此命令
   if isNotAllowed(opstr) {
      return nil, errors.New(fmt.Sprintf("command <%s> is not allowed", opstr))
   }
   usnow := microseconds()
   s.LastOpUnix = usnow / 1e6
   s.Ops++
   //构建Request
   r := &Request{
      OpStr:  opstr,
      Start:  usnow,
      Resp:   resp,
      Wait:   &sync.WaitGroup{},
      Failed: &s.failed,
   }
   if opstr == "QUIT" { //关闭session连接，将quit设置为true
      return s.handleQuit(r)
   }
   if opstr == "AUTH" { //认证
      return s.handleAuth(r)
   }
   if !s.authorized {
      if s.auth != "" {
         r.Response.Resp = redis.NewError([]byte("NOAUTH Authentication required."))
         return r, nil
      }
      s.authorized = true
   }
   switch opstr {
   case "SELECT":
      return s.handleSelect(r)
   case "PING":
      return s.handlePing(r)
   case "MGET":
      return s.handleRequestMGet(r, d)
   case "MSET":
      return s.ha	ndleRequestMSet(r, d)
   case "DEL":
      return s.handleRequestMDel(r, d)
   }
   return r, d.Dispatch(r) //请求分发
}
```

#### 分发请求

pkg/proxy/router/router.go

```go
func (s *Router) Dispatch(r *Request) error {
   hkey := getHashKey(r.Resp, r.OpStr) //获取请求的key
   slot := s.slots[hashSlot(hkey)] //计算key(考虑hashtag)的哈希值，获取对应的slot槽位
   return slot.forward(r, hkey) //请求发送给redis实例
}
```

pkg/proxy/router/slots.go

```go
func (s *Slot) forward(r *Request, key []byte) error {
   s.lock.RLock()
   //slot是否已经初始化、是否在迁移，获取SharedBackendConn
   bc, err := s.prepare(r, key)
   s.lock.RUnlock()
   if err != nil {
      return err
   } else {
      bc.PushBack(r)//存放请求
      return nil
   }
}
```

#### 获取slot连接

pkg/proxy/router/slots.go

```go
func (s *Slot) prepare(r *Request, key []byte) (*SharedBackendConn, error) {
   if s.backend.bc == nil { //slot尚未初始化
      log.Infof("slot-%04d is not ready: key = %s", s.id, key)
      return nil, ErrSlotIsNotReady
   }
  //如果此slot正在迁移数据，发送迁移此key的请求，等待此key迁移完成
   if err := s.slotsmgrt(r, key); err != nil { 
      log.Warnf("slot-%04d migrate from = %s to %s failed: key = %s, error = %s",
         s.id, s.migrate.from, s.backend.addr, key, err)
      return nil, err
   } else {
      r.slot = &s.wait
      r.slot.Add(1)
      return s.backend.bc, nil
   }
}
```

#### 等待迁移完成

pkg/proxy/router/slots.go

```go
func (s *Slot) slotsmgrt(r *Request, key []byte) error {
   if len(key) == 0 || s.migrate.bc == nil { //没有在迁移数据
      return nil
   }
  //此slot正在迁移数据，构建迁移此key的请求
   m := &Request{
      Resp: redis.NewArray([]*redis.Resp{
         redis.NewBulkBytes([]byte("SLOTSMGRTTAGONE")),
         redis.NewBulkBytes(s.backend.host),
         redis.NewBulkBytes(s.backend.port),
         redis.NewBulkBytes([]byte("3000")),
         redis.NewBulkBytes(key),
      }),
      Wait: &sync.WaitGroup{},
   }
   s.migrate.bc.PushBack(m)
	//阻塞直到迁移完成
   m.Wait.Wait()
   resp, err := m.Response.Resp, m.Response.Err
   if err != nil {
      return err
   }
   if resp == nil {
      return ErrRespIsRequired
   }
   if resp.IsError() {
      return errors.New(fmt.Sprintf("error resp: %s", resp.Value))
   }
   if resp.IsInt() { //迁移成功
      log.Debugf("slot-%04d migrate from %s to %s: key = %s, resp = %s",
         s.id, s.migrate.from, s.backend.addr, key, resp.Value)
      return nil
   } else {
      return errors.New(fmt.Sprintf("error resp: should be integer, but got %s", resp.Type))
   }
}
```

#### 客户端请求存放到slot连接的请求队列

pkg/proxy/router/backend.go

```go
func (bc *BackendConn) PushBack(r *Request) {
   if r.Wait != nil {
      r.Wait.Add(1)
   }
   bc.input <- r
}
```

### 响应处理

pkg/proxy/router/session.go

```go
func (s *Session) loopWriter(tasks <-chan *Request) error {
  //发送策略
   p := &FlushPolicy{
      Encoder:     s.Writer,
      MaxBuffered: 32,
      MaxInterval: 300,
   }
   for r := range tasks { //tasks存放客户端请求
      //处理响应信息
      resp, err := s.handleResponse(r)
      if err != nil {
         return err
      }
      //编码发送，如果没有请求，立即将响应发送出去
      if err := p.Encode(resp, len(tasks) == 0); err != nil {
         return err
      }
   }
   return nil
}
```

#### 处理响应信息

pkg/proxy/router/session.go

```go
func (s *Session) handleResponse(r *Request) (*redis.Resp, error) {
   //等待请求处理完成
   r.Wait.Wait() 
   if r.Coalesce != nil {
      if err := r.Coalesce(); err != nil {
         return nil, err
      }
   }
   resp, err := r.Response.Resp, r.Response.Err
   if err != nil {
      return nil, err
   }
   if resp == nil {
      return nil, ErrRespIsRequired
   }
   incrOpStats(r.OpStr, microseconds()-r.Start)
   return resp, nil
}
```

#### 编码发送

pkg/proxy/router/backend.go

```go
func (p *FlushPolicy) Encode(resp *redis.Resp, force bool) error {//积压的数据超过阈值、刷新的间隔超过阈值、当前无请求，立即将响应发送出去
   if err := p.Encoder.Encode(resp, false); err != nil {
      return err
   } else {
      p.nbuffered++
      return p.Flush(force)
   }
}
```

pkg/proxy/redis/encoder.go

```go
func (e *Encoder) Encode(r *Resp, flush bool) error {
   if e.Err != nil {
      return e.Err
   }
  //编码响应信息
   err := e.encodeResp(r)
   if err == nil && flush {
      err = errors.Trace(e.Flush())
   }
   if err != nil {
      e.Err = err
   }
   return err
}
```

##### 响应格式

Simple strings简单字符串：以+开头，后跟字符串，遇到CRLF停止\r\n；示例："+OK\r\n“

Errors错误：以-开头，后跟字符串，遇到CRLF停止；-后面为ERR或WRONGTYPE

Integers整数：以：开头，后跟字符串表示的整数，遇到CRLF停止;示例：":1000\r\n"

Bulk Strings字符串块:以$开头，后跟字符串串长度，CRLF结尾;示例："$6\r\nfoobar\r\n”

Arrays数组类型：以*开头，后跟数组长度N，CRLF结尾；*

字串块数组示例："*2\r\n$3\r\nget\r\n$3\r\nkey\r\n”

整数数组实例："*3\r\n:1\r\n:2\r\n:3\r\n"

```go
func (e *Encoder) encodeResp(r *Resp) error {
  // 写入+-*:$
   if err := e.WriteByte(byte(r.Type)); err != nil { //类型
      return errors.Trace(err)
   }
   switch r.Type {
   default:
      return errors.Errorf("bad resp type %s", r.Type)
   case TypeString, TypeError, TypeInt:
      return e.encodeTextBytes(r.Value) //数据
   case TypeBulkBytes:
      return e.encodeBulkBytes(r.Value)
   case TypeArray:
      return e.encodeArray(r.Array)
   }
}
```

pkg/proxy/redis/encoder.go

```go
func (e *Encoder) encodeTextBytes(b []byte) error {
  //写内容
   if _, err := e.Write(b); err != nil {
      return errors.Trace(err)
   }
  //写 \r\n
   if _, err := e.WriteString("\r\n"); err != nil {
      return errors.Trace(err)
   }
   return nil
}
```

```go
func (e *Encoder) encodeBulkBytes(b []byte) error {
   if b == nil {
      return e.encodeInt(-1)
   } else {
     	//写长度 \r\n
      if err := e.encodeInt(int64(len(b))); err != nil {
         return err
      }
     //写内容 \r\n
      return e.encodeTextBytes(b)
   }
}
```

```go
func (e *Encoder) encodeArray(a []*Resp) error {
   if a == nil {
      return e.encodeInt(-1)
   } else {
     //写长度 \r\n
      if err := e.encodeInt(int64(len(a))); err != nil {
         return err
      }
     	//递归写
      for _, r := range a {
         if err := e.encodeResp(r); err != nil {
            return err
         }
      }
      return nil
   }
}
```

#### 刷新

pkg/proxy/router/backend.go

```java
func (p *FlushPolicy) Flush(force bool) error {//当tasks为空时，force=true
   if force || p.needFlush() {
      if err := p.Encoder.Flush(); err != nil {
         return err
      }
      p.nbuffered = 0
      p.lastflush = microseconds()
   }
   return nil
}
```



```go
func (p *FlushPolicy) needFlush() bool { //是否系统调用
   if p.nbuffered != 0 {
      if p.nbuffered > p.MaxBuffered { //调用Encode的次数
         return true
      }
      if microseconds()-p.lastflush > p.MaxInterval { //刷新间隔
         return true
      }
   }
   return false
}
```

## 处理各种事件

pkg/proxy/proxy.go:442

```go
func (s *Server) loopEvents() {
   //定时器，发送ping
   ticker := time.NewTicker(time.Second)
   defer ticker.Stop()
   var tick int = 0
   for s.info.State == models.PROXY_STATE_ONLINE { //在线
      select {
      case <-s.kill: //关闭事件
         log.Infof("mark offline, proxy is killed: %s", s.info.Id)
         s.markOffline() //proxy下线，修改zk注册的信息为下线状态
      case e := <-s.evtbus: //注册的watch事件
         evtPath := getEventPath(e)
         log.Infof("got event %s, %v, lastActionSeq %d", s.info.Id, e, s.lastActionSeq)
         if strings.Index(evtPath, models.GetActionResponsePath(s.conf.productName)) == 0 {
            seq, err := strconv.Atoi(path.Base(evtPath))
            if err != nil {
               log.ErrorErrorf(err, "parse action seq failed")
            } else {
               if seq < s.lastActionSeq {
                  log.Infof("ignore seq = %d", seq)
                  continue
               }
            }
         }
         s.processAction(e) //处理上线、下线
      case <-ticker.C: 
        if maxTick := s.conf.pingPeriod; maxTick != 0 {  //默认5秒
          	//周期向redis发送ping(只有当redis连接空闲时，即无客户端请求时)
            if tick++; tick >= maxTick {
               s.router.KeepAlive()
               tick = 0
            }
         }
      }
   }
}
```

### 下线

```go
func (s *Server) markOffline() {
   s.topo.Close(s.info.Id) //删除zk节点、关闭连接
   s.info.State = models.PROXY_STATE_MARK_OFFLINE //修改为下线状态
}
```

### 心跳

pkg/proxy/router/router.go:

```go
func (s *Router) KeepAlive() error {
   s.mu.Lock()
   defer s.mu.Unlock()
  //已经关闭
   if s.closed {
      return ErrClosedRouter
   }
  //发送ping
   for _, bc := range s.pool {
      bc.KeepAlive()
   }
   return nil
}
```

pkg/proxy/router/backend.go

```go
func (bc *BackendConn) KeepAlive() bool {
  //有尚未发送的请求，直接返回
   if len(bc.input) != 0 { 
      return false
   }
  //发送ping，将ping请求放入input中
   bc.PushBack(&Request{
      Resp: redis.NewArray([]*redis.Resp{
         redis.NewBulkBytes([]byte("PING")),
      }),
   })
   return true
}
```

