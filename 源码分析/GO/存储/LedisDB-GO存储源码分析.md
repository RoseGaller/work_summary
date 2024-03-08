# 入口

```go
var configFile = flag.String("config", "/etc/ledis.json", "ledisdb config file")

func main() {
   runtime.GOMAXPROCS(runtime.NumCPU())
   flag.Parse()
   if len(*configFile) == 0 {
      println("must use a config file")
      return
   }
   //读取配置文件，创建Config
   cfg, err := server.NewConfigWithFile(*configFile)
   if err != nil {
      println(err.Error())
      return
   }
	 //根据Config创建app
   var app *server.App
   app, err = server.NewApp(cfg)
   if err != nil {
      println(err.Error())
      return
   }
	 //存放关闭信号
   sc := make(chan os.Signal, 1)
   signal.Notify(sc,
      syscall.SIGHUP,
      syscall.SIGINT,
      syscall.SIGTERM,
      syscall.SIGQUIT)

   go func() {
      <-sc //接收关闭信号
      app.Close() //关闭app
   }()
	
   app.Run() //启动app
}
```

# 创建Config

server/config.go

```go
func NewConfigWithFile(fileName string) (*Config, error) {
   data, err := ioutil.ReadFile(fileName) //读取文件
   if err != nil {
      return nil, err
   }

   return NewConfig(data)
}
```

```go
func NewConfig(data json.RawMessage) (*Config, error) {
   c := new(Config)
	 //填充config
   err := json.Unmarshal(data, c)
   if err != nil {
      return nil, err
   }
   return c, nil
}
```

# 创建App

server/app.go

```go
func NewApp(cfg *Config) (*App, error) {
   if len(cfg.DataDir) == 0 {
      return nil, fmt.Errorf("must set data_dir first")
   }

   app := new(App)

   app.quit = make(chan struct{})

   app.closed = false

   app.cfg = cfg

   var err error
	//创建listener
   if strings.Contains(cfg.Addr, "/") {
      app.listener, err = net.Listen("unix", cfg.Addr)
   } else {
      app.listener, err = net.Listen("tcp", cfg.Addr)
   }

   if err != nil {
      return nil, err
   }
	//日志
   if len(cfg.AccessLog) > 0 {
      if path.Dir(cfg.AccessLog) == "." {
         app.access, err = newAcessLog(path.Join(cfg.DataDir, cfg.AccessLog))
      } else {
         app.access, err = newAcessLog(cfg.AccessLog)
      }
      if err != nil {
         return nil, err
      }
   }	
	//打开DB
   if app.ldb, err = ledis.Open(cfg.NewLedisConfig()); err != nil {
      return nil, err
   }
	//创建Master
   app.m = newMaster(app)

   return app, nil
}
```

server/replication.go

```go
func newMaster(app *App) *master {
   m := new(master)
   m.app = app

   m.infoName = path.Join(m.app.cfg.DataDir, "master.info")

   m.quit = make(chan struct{}, 1)

   m.compressBuf = make([]byte, 256)

   m.info = new(MasterInfo)

   //从磁盘加载masterInfo
   m.loadInfo()

   return m
}
```

## 打开DB

ledis/ledis.go

```go
func Open(cfg *Config) (*Ledis, error) {
   if len(cfg.DataDir) == 0 {
      return nil, fmt.Errorf("must set correct data_dir")
   }
	//调用c
   ldb, err := leveldb.Open(cfg.NewDBConfig())
   if err != nil {
      return nil, err
   }

   l := new(Ledis)

   l.quit = make(chan struct{})
   l.jobs = new(sync.WaitGroup)

   l.ldb = ldb

   if cfg.BinLog.Use { //启动binlog
      println("binlog will be refactored later, use your own risk!!!")
      l.binlog, err = NewBinLog(cfg.NewBinLogConfig()) //创建binglog
      if err != nil {
         return nil, err
      }
   } else {
      l.binlog = nil
   }
	//创建DB
   for i := uint8(0); i < MaxDBNumber; i++ {
      l.dbs[i] = newDB(l, i)
   }
	//删除过期数据
   l.activeExpireCycle()

   return l, nil
}
```

## 创建DB

```go
func newDB(l *Ledis, index uint8) *DB {
   d := new(DB)

   d.l = l

   d.db = l.ldb

   d.index = index

   d.kvTx = newTx(l)
   d.listTx = newTx(l)
   d.hashTx = newTx(l)
   d.zsetTx = newTx(l)
   d.binTx = newTx(l)

   return d
}
```

## 清理数据

ledis/ledis.go

```go
func (l *Ledis) activeExpireCycle() {
   var executors []*elimination = make([]*elimination, len(l.dbs))
   for i, db := range l.dbs {
      executors[i] = db.newEliminator()
   }

   l.jobs.Add(1)
   go func() {
      tick := time.NewTicker(1 * time.Second)
      end := false
      done := make(chan struct{})
      for !end {
         select {
         case <-tick.C:
            go func() {
               for _, eli := range executors {
                  eli.active()
               }
               done <- struct{}{}
            }()
            <-done
         case <-l.quit:
            end = true
            break
         }
      }

      tick.Stop()
      l.jobs.Done()
   }()
}
```

# 启动App

server/app.go

```go
func (app *App) Run() {
   if len(app.cfg.SlaveOf) > 0 {
      app.slaveof(app.cfg.SlaveOf) //启用了主从复制
   }

   for !app.closed {
     //监听客户端连接
      conn, err := app.listener.Accept()
      if err != nil {
         continue
      }
			//处理连接
      newClient(conn, app)
   }
}
```

## 主从复制

server/replication.go

```go
func (app *App) slaveof(masterAddr string) error {
   app.m.Lock()
   defer app.m.Unlock()

   if len(masterAddr) == 0 {
      return app.m.stopReplication() //停止复制
   } else {
      return app.m.startReplication(masterAddr) //启动复制
   }

   return nil
}
```

## 创建Client

server/client.go

```go
func newClient(c net.Conn, app *App) { //为每条连接创建Client
   co := new(client)

   co.app = app
   co.ldb = app.ldb
   //use default db
   co.db, _ = app.ldb.Select(0)
   co.c = c
	
  //创建带缓冲区的reader、writer
   co.rb = bufio.NewReaderSize(c, 256)
   co.wb = bufio.NewWriterSize(c, 256)

   co.reqC = make(chan error, 1)

   co.compressBuf = make([]byte, 256)

  //接收、处理客户端请求
   go co.run()
}
```

server/client.go

```go
func (c *client) run() {//接收处理请求	
   defer func() {
      if e := recover(); e != nil {
         buf := make([]byte, 4096)
         n := runtime.Stack(buf, false)
         buf = buf[0:n]

         log.Fatal("client run panic %s:%v", buf, e)
      }

      c.c.Close()
   }()

   for {
      req, err := c.readRequest() //读取请求
      if err != nil {
         return
      }

      c.handleRequest(req) //处理请求
   }
}
```

### 读取请求

```go
func (c *client) readRequest() ([][]byte, error) {
   l, err := c.readLine()
   if err != nil {
      return nil, err
   } else if len(l) == 0 || l[0] != '*' {
      return nil, errReadRequest
   }

   var nparams int
   if nparams, err = strconv.Atoi(ledis.String(l[1:])); err != nil {
      return nil, err
   } else if nparams <= 0 {
      return nil, errReadRequest
   }

   req := make([][]byte, 0, nparams)
   var n int
   for i := 0; i < nparams; i++ {
      if l, err = c.readLine(); err != nil {
         return nil, err
      }

      if len(l) == 0 {
         return nil, errReadRequest
      } else if l[0] == '$' {
         //handle resp string
         if n, err = strconv.Atoi(ledis.String(l[1:])); err != nil {
            return nil, err
         } else if n == -1 {
            req = append(req, nil)
         } else {
            buf := make([]byte, n)
            if _, err = io.ReadFull(c.rb, buf); err != nil {
               return nil, err
            }

            if l, err = c.readLine(); err != nil {
               return nil, err
            } else if len(l) != 0 {
               return nil, errors.New("bad bulk string format")
            }

            req = append(req, buf)

         }

      } else {
         return nil, errReadRequest
      }
   }

   return req, nil
}
```

### 处理请求

```go
func (c *client) handleRequest(req [][]byte) {
   var err error

   start := time.Now()

   if len(req) == 0 {
      err = ErrEmptyCommand
   } else {
     	//获取命令
      c.cmd = strings.ToLower(ledis.String(req[0]))
     //获取参数
      c.args = req[1:]
			//根据Command方法
      f, ok := regCmds[c.cmd]
      if !ok {
         err = ErrNotFound
      } else {
         go func() {
            c.reqC <- f(c) //执行请求，返回nil或者error
         }()
         err = <-c.reqC //等待执行完成
      }
   }
	//执行耗时
   duration := time.Since(start)

   if c.app.access != nil {
      c.logBuf.Reset()
      for i, r := range req {
         left := 256 - c.logBuf.Len()
         if left <= 0 {
            break
         } else if len(r) <= left {
            c.logBuf.Write(r)
            if i != len(req)-1 {
               c.logBuf.WriteByte(' ')
            }
         } else {
            c.logBuf.Write(r[0:left])
         }
      }

      c.app.access.Log(c.c.RemoteAddr().String(), duration.Nanoseconds()/1000000, c.logBuf.Bytes(), err)
   }

   if err != nil {
      c.writeError(err) //写异常信息到写缓冲区
   }
	//发送缓冲区内的响应
   c.wb.Flush()
}
```

# KV写入

ledis/t_kv.go

```go
func (db *DB) Set(key []byte, value []byte) error {
  //检测key、value的大小
   if err := checkKeySize(key); err != nil {
      return err
   } else if err := checkValueSize(value); err != nil {
      return err
   }

   var err error
   key = db.encodeKVKey(key)

   t := db.kvTx

   t.Lock()
   defer t.Unlock()

   t.Put(key, value)

   //todo, binlog

   err = t.Commit()

   return err
}
```

```go
func (db *DB) encodeKVKey(key []byte) []byte { //对key进行编码
   ek := make([]byte, len(key)+2)
   ek[0] = db.index //所属的数据库编号
   ek[1] = KVType //命令类型（kV、hash、list、zset）
   copy(ek[2:], key)
   return ek
}
```

ledis/tx.go

```go
func (t *tx) Put(key []byte, value []byte) {
   t.wb.Put(key, value) //写入writebatch

   if t.binlog != nil { //记录binlog
      buf := encodeBinLogPut(key, value)
      t.batch = append(t.batch, buf)
   }
}
```

```go
func (t *tx) Commit() error { //提交
   var err error
   if t.binlog != nil {
      t.l.Lock()
      err = t.wb.Commit()
      if err != nil {
         t.l.Unlock()
         return err
      }

      err = t.binlog.Log(t.batch...) //写入binlog文件

      t.l.Unlock()
   } else {
      t.l.Lock()
      err = t.wb.Commit()
      t.l.Unlock()
   }
   return err
}
```

ledis/binlog.go

```go
func (l *BinLog) Log(args ...[]byte) error { //写binlog
   var err error

   if l.logFile == nil {
      if err = l.openNewLogFile(); err != nil { //创建binlog文件
         return err
      }
   }

   //we treat log many args as a batch, so use same createTime
   createTime := uint32(time.Now().Unix())

   for _, data := range args {
      payLoadLen := uint32(len(data)) //长度
		
     	//创建时间
      if err := binary.Write(l.logWb, binary.BigEndian, createTime); err != nil {
         return err
      }
			//长度
      if err := binary.Write(l.logWb, binary.BigEndian, payLoadLen); err != nil {
         return err
      }
			//数据
      if _, err := l.logWb.Write(data); err != nil {
         return err
      }
   }
	//写入文件
   if err = l.logWb.Flush(); err != nil {
      log.Error("write log error %s", err.Error())
      return err
   }

   l.checkLogFileSize() //检测文件大小

   return nil
}
```

```go
func (l *BinLog) openNewLogFile() error {//打开新的binlog文件
   var err error
   lastName := l.getLogFile() //新的binlog文件的名称

   logPath := path.Join(l.cfg.Path, lastName) //文件路径
   if l.logFile, err = os.OpenFile(logPath, os.O_CREATE|os.O_WRONLY, 0666); err != nil { //不存在就创建文件，可写入
      log.Error("open new logfile error %s", err.Error())
      return err
   }

   if l.cfg.MaxFileNum > 0 && len(l.logNames) == l.cfg.MaxFileNum { //超过最大文件数
      l.purge(1) //删除一个binlog文件
   }

   l.logNames = append(l.logNames, lastName)

   //创建writer
   if l.logWb == nil {
      l.logWb = bufio.NewWriterSize(l.logFile, 1024) 
   } else {
      l.logWb.Reset(l.logFile)
   }

   if err = l.flushIndex(); err != nil {
      return err
   }

   return nil
}
```

```go
func (l *BinLog) checkLogFileSize() bool {
   if l.logFile == nil {
      return false
   }

   st, _ := l.logFile.Stat()
   if st.Size() >= int64(l.cfg.MaxFileSize) { //超过文件最大值
      l.lastLogIndex++

      l.logFile.Close() //关闭binlog文件
      l.logFile = nil
      return true
   }

   return false
}
```

# 复制

server/replication.go

```go
func (m *master) startReplication(masterAddr string) error {
   //stop last replcation, if avaliable
   m.Close()

   if masterAddr != m.info.Addr {
      m.resetInfo(masterAddr) //重置masterinfo，将其写入文件
      if err := m.saveInfo(); err != nil {
         log.Error("save master info error %s", err.Error())
         return err
      }
   }

   m.quit = make(chan struct{}, 1)

   go m.runReplication()//开启复制
   return nil
}
```

```go
func (m *master) runReplication() {
   m.wg.Add(1) //复制中
   defer m.wg.Done() //复制结束

   for {
      select {
      case <-m.quit:
         return
      default:
         if err := m.connect(); err != nil { //与master建立连接
            log.Error("connect master %s error %s, try 2s later", m.info.Addr, err.Error())
            time.Sleep(2 * time.Second) //异常，休眠
            continue
         }
      }

      if m.info.LogFileIndex == 0 { //全量复制
         //开启全量复制
         if err := m.fullSync(); err != nil {
            log.Warn("full sync error %s", err.Error())
            return
         }

         if m.info.LogFileIndex == 0 {
            //master not support binlog, we cannot sync, so stop replication
            m.stopReplication()
            return
         }
      }
			//增量同步
      for {
         for {
           //binlog文件以及位置
            lastIndex := m.info.LogFileIndex
            lastPos := m.info.LogPos
            if err := m.sync(); err != nil {
               log.Warn("sync error %s", err.Error())
               return
            }

            if m.info.LogFileIndex == lastIndex && m.info.LogPos == lastPos {
               //sync no data, wait 1s and retry
               break
            }
         }

         select {
         case <-m.quit:
            return
         case <-time.After(1 * time.Second):
            break
         }
      }
   }

   return
}
```

## 全量复制

server/replication.go

```go
func (m *master) fullSync() error {
   //直接发送全量复制的命令
   if _, err := m.c.Write(fullSyncCmd); err != nil {
      return err
   }
	//创建文件
   dumpPath := path.Join(m.app.cfg.DataDir, "master.dump")
   f, err := os.OpenFile(dumpPath, os.O_CREATE|os.O_WRONLY, os.ModePerm)
   if err != nil {
      return err
   }

   defer os.Remove(dumpPath)
	//读取master全量数据
   err = ReadBulkTo(m.rb, f)
   f.Close()
   if err != nil {
      log.Error("read dump data error %s", err.Error())
      return err
   }
	//清空本地数据库
   if err = m.app.ldb.FlushAll(); err != nil {
      return err
   }

   var head *ledis.MasterInfo
  //加载全量数据到db，返回masterinfo
   head, err = m.app.ldb.LoadDumpFile(dumpPath)

   if err != nil {
      log.Error("load dump file error %s", err.Error())
      return err
   }

   m.info.LogFileIndex = head.LogFileIndex
   m.info.LogPos = head.LogPos

   return m.saveInfo() //复制信息写入文件
}
```

```go
func ReadBulkTo(rb *bufio.Reader, w io.Writer) error {
   l, err := ReadLine(rb) // $45\r\n,$后面表示长度
   if len(l) == 0 {
      return errBulkFormat
   } else if l[0] == '$' {
      var n int
      //handle resp string
      if n, err = strconv.Atoi(ledis.String(l[1:])); err != nil {//读取长度
         return err
      } else if n == -1 {
         return nil
      } else {
        //从rb中读取n个字节写入到w中
         if _, err = io.CopyN(w, rb, int64(n)); err != nil { 
            return err
         }

         if l, err = ReadLine(rb); err != nil { //读取\r\n
            return err
         } else if len(l) != 0 {
            return errBulkFormat
         }
      }
   } else {
      return errBulkFormat
   }

   return nil
}
```

## 增量同步

```go
func (m *master) sync() error {
  	//发送增量同步的命令
   logIndexStr := strconv.FormatInt(m.info.LogFileIndex, 10)
   logPosStr := strconv.FormatInt(m.info.LogPos, 10)
	// "*3\r\n$4\r\nsync\r\n$%d\r\n%s\r\n$%d\r\n%s\r\n"
   cmd := ledis.Slice(fmt.Sprintf(syncCmdFormat, len(logIndexStr),
      logIndexStr, len(logPosStr), logPosStr))
   if _, err := m.c.Write(cmd); err != nil {
      return err
   }
	//清空同步缓冲区
   m.syncBuf.Reset()
	//读取数据到缓冲区
   err := ReadBulkTo(m.rb, &m.syncBuf)
   if err != nil {
      return err
   }
	//解压缩
   var buf []byte
   buf, err = snappy.Decode(m.compressBuf, m.syncBuf.Bytes())
   if err != nil {
      return err
   } else if len(buf) > len(m.compressBuf) {
      m.compressBuf = buf
   }

   if len(buf) < 16 {
      return fmt.Errorf("invalid sync data len %d", len(buf))
   }

   m.info.LogFileIndex = int64(binary.BigEndian.Uint64(buf[0:8]))
   m.info.LogPos = int64(binary.BigEndian.Uint64(buf[8:16]))

   if m.info.LogFileIndex == 0 {
      //master now not support binlog, stop replication
      m.stopReplication()
      return nil
   } else if m.info.LogFileIndex == -1 {
      return m.fullSync()
   }
	//将同步的数据写入DB	
   err = m.app.ldb.ReplicateFromData(buf[16:])
   if err != nil {
      return err
   }

   return m.saveInfo() //保存同步的进度

}
```

ledis/replication.go

```go
func (l *Ledis) ReplicateFromData(data []byte) error {
   rb := bytes.NewReader(data)

   l.Lock()
   err := l.ReplicateFromReader(rb)
   l.Unlock()

   return err
}
```

```go
func (l *Ledis) ReplicateFromReader(rb io.Reader) error {
   //解析字节数组，应用到db
   f := func(createTime uint32, event []byte) error {
      err := l.ReplicateEvent(event)
      if err != nil {
         log.Fatal("replication error %s, skip to next", err.Error())
         return ErrSkipEvent
      }
      return nil
   }

   return ReadEventFromReader(rb, f)
}
```

```go
func ReadEventFromReader(rb io.Reader, f func(createTime uint32, event []byte) error) error {
   var createTime uint32
   var dataLen uint32
   var dataBuf bytes.Buffer
   var err error
//binlog格式：createTime、dataLeng、data（type、keyLen、key、value）
   for {
     	//读取创建事件
      if err = binary.Read(rb, binary.BigEndian, &createTime); err != nil {
         if err == io.EOF {
            break
         } else {
            return err
         }
      }
			//读取单条binlog的长度
      if err = binary.Read(rb, binary.BigEndian, &dataLen); err != nil {
         return err
      }
			//将binlog读取到dataBuf
      if _, err = io.CopyN(&dataBuf, rb, int64(dataLen)); err != nil {
         return err
      }
      //解析data
      err = f(createTime, dataBuf.Bytes())
      if err != nil && err != ErrSkipEvent {
         return err
      }

      dataBuf.Reset()
   }

   return nil
}
```

```go
func (l *Ledis) ReplicateEvent(event []byte) error {
   if len(event) == 0 {
      return errInvalidBinLogEvent
   }

   logType := uint8(event[0]) //类型（KV、Hash）
   switch logType {
   case BinLogTypePut:
      return l.replicatePutEvent(event)
   case BinLogTypeDeletion:
      return l.replicateDeleteEvent(event)
   case BinLogTypeCommand:
      return l.replicateCommandEvent(event)
   default:
      return errInvalidBinLogEvent
   }
}
```

```go
func (l *Ledis) replicatePutEvent(event []byte) error {
  //解析 key value
   key, value, err := decodeBinLogPut(event)
   if err != nil {
      return err
   }
		//应用到db
   if err = l.ldb.Put(key, value); err != nil {
      return err
   }
	
  //写binlog
   if l.binlog != nil {
      err = l.binlog.Log(event)
   }

   return err
}
```



```go
func decodeBinLogPut(sz []byte) ([]byte, []byte, error) {
   if len(sz) < 3 || sz[0] != BinLogTypePut { //type 1个字节 长度 2个字节
      return nil, nil, errBinLogPutType 
   }
	//读取2个字节
   keyLen := int(binary.BigEndian.Uint16(sz[1:]))
   if 3+keyLen > len(sz) {
      return nil, nil, errBinLogPutType
   }

   return sz[3 : 3+keyLen], sz[3+keyLen:], nil //返回key value
}
```

