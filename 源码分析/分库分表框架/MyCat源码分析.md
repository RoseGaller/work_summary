# MyCat源码分析

* [启动入口](#启动入口)
  * [初始化MycatServer](#初始化mycatserver)
  * [启动MycatServer](#启动mycatserver)
* [NIOAcceptor](#nioacceptor)
* [NIOReactor](#nioreactor)
  * [绑定连接](#绑定连接)
  * [获取读写事件](#获取读写事件)
  * [新建连接注册读事件](#新建连接注册读事件)
* [SQL执行](#sql执行)
  * [计算路由](#计算路由)
    * [获取路由策略](#获取路由策略)
    * [获取路由结果](#获取路由结果)
  * [执行SQL](#执行sql)


# 启动入口

org.opencloudb.MycatStartup#main

```java
public static void main(String[] args) {
   try {
      String home = SystemConfig.getHomePath();
      if (home == null) {
         System.out.println(SystemConfig.SYS_HOME + "  is not set.");
         System.exit(-1);
      }
      // init
      MycatServer server = MycatServer.getInstance();
      server.beforeStart();

      // startup
      server.startup();
      System.out.println("MyCAT Server startup successfully. see logs in logs/mycat.log");
      while (true) {
         Thread.sleep(300 * 1000);
      }
   } catch (Exception e) {
      SimpleDateFormat sdf = new SimpleDateFormat(dateFormat);
      LogLog.error(sdf.format(new Date()) + " startup error", e);
      System.exit(-1);
   }
}
```

## 初始化MycatServer

org.opencloudb.MycatServer#MycatServer

```java
public MycatServer() {
   this.config = new MycatConfig();
   this.timer = new Timer(NAME + "Timer", true);
   this.sqlRecorder = new SQLRecorder(config.getSystem()
         .getSqlRecordCount());
   this.isOnline = new AtomicBoolean(true);
   cacheService = new CacheService();
   //路由服务
   routerService = new RouteService(cacheService);
   // load datanode active index from properties
   dnIndexProperties = loadDnIndexProps();
   try {
      //SQL拦截器
      sqlInterceptor = (SQLInterceptor) Class.forName(
            config.getSystem().getSqlInterceptor()).newInstance();
   } catch (Exception e) {
      throw new RuntimeException(e);
   }
   catletClassLoader = new DynaClassLoader(SystemConfig.getHomePath()
         + File.separator + "catlet", config.getSystem()
         .getCatletClassCheckSeconds());
   this.startupTime = TimeUtil.currentTimeMillis();
}
```

## 启动MycatServer

org.opencloudb.MycatServer#startup

```java
public void startup() throws IOException {

   SystemConfig system = config.getSystem();
   int processorCount = system.getProcessors();

   // server startup
   LOGGER.info("===============================================");
   LOGGER.info(NAME + " is ready to startup ...");
   String inf = "Startup processors ...,total processors:"
         + system.getProcessors() + ",aio thread pool size:"
         + system.getProcessorExecutor()
         + "    \r\n each process allocated socket buffer pool "
         + " bytes ,buffer chunk size:"
         + system.getProcessorBufferChunk()
         + "  buffer pool's capacity(buferPool/bufferChunk) is:"
         + system.getProcessorBufferPool()
         / system.getProcessorBufferChunk();
   LOGGER.info(inf);
   LOGGER.info("sysconfig params:" + system.toString());

   //为来自管理系统的连接请求创建ManagerConnection
   ManagerConnectionFactory mf = new ManagerConnectionFactory();
   //为来自业务系统的连接请求创建ServerConnection
   ServerConnectionFactory sf = new ServerConnectionFactory();
   SocketAcceptor manager = null;
   SocketAcceptor server = null;
   aio = (system.getUsingAIO() == 1); //默认NIO

   // startup processors
   int threadPoolSize = system.getProcessorExecutor();
   processors = new NIOProcessor[processorCount];
   long processBuferPool = system.getProcessorBufferPool(); // 4096 * processors * 1000;
   int processBufferChunk = system.getProcessorBufferChunk(); //4096
   int socketBufferLocalPercent = system.getProcessorBufferLocalPercent();
   //DirectByteBuffer缓存池
   bufferPool = new BufferPool(processBuferPool, processBufferChunk,
         socketBufferLocalPercent / processorCount);
   businessExecutor = ExecutorUtil.create("BusinessExecutor",
         threadPoolSize);
   timerExecutor = ExecutorUtil.create("Timer", system.getTimerExecutor());
   listeningExecutorService = MoreExecutors.listeningDecorator(businessExecutor);

   for (int i = 0; i < processors.length; i++) {
      processors[i] = new NIOProcessor("Processor" + i, bufferPool,
            businessExecutor);
   }

   if (aio) {//默认false
      LOGGER.info("using aio network handler ");
      asyncChannelGroups = new AsynchronousChannelGroup[processorCount];
      // startup connector
      connector = new AIOConnector();
      for (int i = 0; i < processors.length; i++) {
         asyncChannelGroups[i] = AsynchronousChannelGroup
               .withFixedThreadPool(processorCount,
                     new ThreadFactory() {
                        private int inx = 1;

                        @Override
                        public Thread newThread(Runnable r) {
                           Thread th = new Thread(r);
                           th.setName(BufferPool.LOCAL_BUF_THREAD_PREX
                                 + "AIO" + (inx++));
                           LOGGER.info("created new AIO thread "
                                 + th.getName());
                           return th;
                        }
                     });

      }
      manager = new AIOAcceptor(NAME + "Manager", system.getBindIp(),
            system.getManagerPort(), mf, this.asyncChannelGroups[0]);

      // startup server

      server = new AIOAcceptor(NAME + "Server", system.getBindIp(),
            system.getServerPort(), sf, this.asyncChannelGroups[0]);

   } else { //NIO
      LOGGER.info("using nio network handler ");
      //读写事件分离器
      NIOReactorPool reactorPool = new NIOReactorPool(
            BufferPool.LOCAL_BUF_THREAD_PREX + "NIOREACTOR",
            processors.length);
      //负责作为客户端连接MySQL的主动连接事件
      connector = new NIOConnector(BufferPool.LOCAL_BUF_THREAD_PREX
            + "NIOConnector", reactorPool);
      ((NIOConnector) connector).start();
      //监听管理的连接请求
      manager = new NIOAcceptor(BufferPool.LOCAL_BUF_THREAD_PREX + NAME
            + "Manager", system.getBindIp(), system.getManagerPort(),
            mf, reactorPool);
      //监听应用程序的连接请求
      server = new NIOAcceptor(BufferPool.LOCAL_BUF_THREAD_PREX + NAME
            + "Server", system.getBindIp(), system.getServerPort(), sf,
            reactorPool);
   }
   // manager start
   manager.start();
   LOGGER.info(manager.getName() + " is started and listening on "
         + manager.getPort());
   //启动，监听应用程序的连接
   server.start();
   // server started
   LOGGER.info(server.getName() + " is started and listening on "
         + server.getPort());
   LOGGER.info("===============================================");
   // init datahost
   Map<String, PhysicalDBPool> dataHosts = config.getDataHosts();
   LOGGER.info("Initialize dataHost ...");
   for (PhysicalDBPool node : dataHosts.values()) {
      String index = dnIndexProperties.getProperty(node.getHostName(),
            "0");
      if (!"0".equals(index)) {
         LOGGER.info("init datahost: " + node.getHostName()
               + "  to use datasource index:" + index);
      }
      node.init(Integer.valueOf(index));
      node.startHeartbeat();
   }
   long dataNodeIldeCheckPeriod = system.getDataNodeIdleCheckPeriod();
   timer.schedule(updateTime(), 0L, TIME_UPDATE_PERIOD);
   timer.schedule(processorCheck(), 0L, system.getProcessorCheckPeriod());
   timer.schedule(dataNodeConHeartBeatCheck(dataNodeIldeCheckPeriod), 0L,
         dataNodeIldeCheckPeriod);
   timer.schedule(dataNodeHeartbeat(), 0L,
         system.getDataNodeHeartbeatPeriod());
   timer.schedule(catletClassClear(), 30000);

}
```

# NIOAcceptor

监听应用程序的连接请求

org.opencloudb.net.NIOAcceptor#run

```java
public void run() {
   final Selector tSelector = this.selector;
   for (;;) {
      ++acceptCount;
      try {
          tSelector.select(1000L);
         Set<SelectionKey> keys = tSelector.selectedKeys();
         try {
            for (SelectionKey key : keys) {
               if (key.isValid() && key.isAcceptable()) {
                  accept();//创建连接
               } else {
                  key.cancel();
               }
            }
         } finally {
            keys.clear();
         }
      } catch (Exception e) {
         LOGGER.warn(getName(), e);
      }
   }
}
```

org.opencloudb.net.NIOAcceptor#accept

```java
private void accept() {//创建客户端连接
   SocketChannel channel = null;
   try {
      channel = serverChannel.accept();
      channel.configureBlocking(false); //非阻塞
			//创建ServerConnection
      FrontendConnection c = factory.make(channel);
      c.setAccepted(true);
      c.setId(ID_GENERATOR.getId());
      //选择Processor
      NIOProcessor processor = (NIOProcessor) MycatServer.getInstance()
            .nextProcessor();
     //绑定Processor
      c.setProcessor(processor);
     
      //选择Reactor进行绑定
      NIOReactor reactor = reactorPool.getNextReactor();
     	//绑定Reactor，此客户端的IO读写由此NIOReactor负责
      reactor.postRegister(c);

   } catch (Exception e) {
        LOGGER.warn(getName(), e);
      closeChannel(channel);
   }
}
```

# NIOReactor

## 绑定连接

org.opencloudb.net.NIOReactor#postRegister

```java
final void postRegister(AbstractConnection c) {
   reactorR.registerQueue.offer(c); //存放新建的连接
   reactorR.selector.wakeup(); //唤醒selector
}
```

## 获取读写事件

org.opencloudb.net.NIOReactor.RW#run

```java
public void run() {
   final Selector selector = this.selector;
   Set<SelectionKey> keys = null;
   for (;;) {
      ++reactCount;
      try {
         selector.select(500L);
         register(selector); //为新建的连接注册读事件
         keys = selector.selectedKeys();
         for (SelectionKey key : keys) {
            AbstractConnection con = null;
            try {
               Object att = key.attachment();
               if (att != null) {
                  con = (AbstractConnection) att;
                  if (key.isValid() && key.isReadable()) { //读
                     try {
                        con.asynRead();
                     } catch (IOException e) {
                                      con.close("program err:" + e.toString());
                        continue;
                     } catch (Exception e) {
                        LOGGER.debug("caught err:", e);
                        con.close("program err:" + e.toString());
                        continue;
                     }
                  }
                  if (key.isValid() && key.isWritable()) { //写
                     con.doNextWriteCheck();
                  }
               } else {
                  key.cancel();
               }
                      } catch (CancelledKeyException e) {
                          if (LOGGER.isDebugEnabled()) {
                              LOGGER.debug(con + " socket key canceled");
                          }
                      } catch (Exception e) {
                          LOGGER.warn(con + " " + e);
                      }
         }
      } catch (Exception e) {
         LOGGER.warn(name, e);
      } finally {
         if (keys != null) {
            keys.clear();
         }

      }
   }
}
```

## 新建连接注册读事件

org.opencloudb.net.NIOReactor.RW#register

```java
private void register(Selector selector) {
   AbstractConnection c = null;
   if (registerQueue.isEmpty()) {
      return;
   }
   while ((c = registerQueue.poll()) != null) {
      try {
         //注册读事件
         ((NIOSocketWR) c.getSocketWR()).register(selector);
         c.register();
      } catch (Exception e) {
         c.close("register err" + e.toString());
      }
   }
}
```

# SQL执行

org.opencloudb.server.ServerConnection#execute

```java
public void execute(String sql, int type) {
   if (this.isClosed()) {
      LOGGER.warn("ignore execute ,server connection is closed " + this);
      return;
   }
   // 状态检查
   if (txInterrupted) {
      writeErrMessage(ErrorCode.ER_YES,
            "Transaction error, need to rollback." + txInterrputMsg);
      return;
   }

   // 检查当前使用的DB
   String db = this.schema;
   if (db == null) {
      writeErrMessage(ErrorCode.ERR_BAD_LOGICDB,
            "No MyCAT Database selected");
      return;
   }
   SchemaConfig schema = MycatServer.getInstance().getConfig()
         .getSchemas().get(db);
   if (schema == null) {
      writeErrMessage(ErrorCode.ERR_BAD_LOGICDB,
            "Unknown MyCAT Database '" + db + "'");
      return;
   }
	 //路由并执行SQL
   routeEndExecuteSQL(sql, type, schema);

}
```

org.opencloudb.server.ServerConnection#routeEndExecuteSQL

```java
public void routeEndExecuteSQL(String sql, int type, SchemaConfig schema) {
   // 路由计算
   RouteResultset rrs = null;
   try {
      rrs = MycatServer
            .getInstance()
            .getRouterservice()
            .route(MycatServer.getInstance().getConfig().getSystem(),
                  schema, type, sql, this.charset, this);

   } catch (Exception e) {
      StringBuilder s = new StringBuilder();
      LOGGER.warn(s.append(this).append(sql).toString() + " err:" + e.toString(),e);
      String msg = e.getMessage();
      writeErrMessage(ErrorCode.ER_PARSE_ERROR, msg == null ? e.getClass().getSimpleName() : msg);
      return;
   }
  
    // session执行
   if (rrs != null) {
      session.execute(rrs, type);
   }
}
```

## 计算路由

org.opencloudb.route.RouteService#route

```java
public RouteResultset route(SystemConfig sysconf, SchemaConfig schema,
      int sqlType, String stmt, String charset, ServerConnection sc)
      throws SQLNonTransientException {
   RouteResultset rrs = null;
   String cacheKey = null;

   if (sqlType == ServerParse.SELECT) {//SELECT类型的SQL
      //缓存key
      cacheKey = schema.getName() + stmt;
      //从缓存中获取路由结果
      rrs = (RouteResultset) sqlRouteCache.get(cacheKey);
      if (rrs != null) {
         return rrs;
      }
   }
  
   /*!mycat: sql = select name from aa */
       /*!mycat: schema = test */
       boolean isMatchOldHint = stmt.startsWith(OLD_MYCAT_HINT);
       boolean isMatchNewHint = stmt.startsWith(NEW_MYCAT_HINT);
   if (isMatchOldHint || isMatchNewHint ) { //有注解
      int endPos = stmt.indexOf("*/");
      if (endPos > 0) {           
         // 用!mycat:内部的语句来做路由分析
         int hintLength = isMatchOldHint ? OLD_MYCAT_HINT.length() : NEW_MYCAT_HINT.length();            
         String hint = stmt.substring(hintLength, endPos).trim();   
               int firstSplitPos = hint.indexOf(HINT_SPLIT);
               if(firstSplitPos > 0 ){
                   String hintType = hint.substring(0,firstSplitPos).trim().toLowerCase(Locale.US);
                   String hintValue = hint.substring(firstSplitPos + HINT_SPLIT.length()).trim();
                   if(hintValue.length()==0){
                       LOGGER.warn("comment int sql must meet :/*!mycat:type=value*/ or /*#mycat:type=value*/: "+stmt);
                       throw new SQLSyntaxErrorException("comment int sql must meet :/*!mycat:type=value*/ or /*#mycat:type=value*/: "+stmt);
                   }
                   String realSQL = stmt.substring(endPos + "*/".length()).trim();

                   HintHandler hintHandler = HintHandlerFactory.getHintHandler(hintType);
                   if(hintHandler != null){
                       rrs = hintHandler.route(sysconf,schema,sqlType,realSQL,charset,sc,tableId2DataNodeCache,hintValue);
                   }else{
                       LOGGER.warn("TODO , support hint sql type : " + hintType);
                   }
                   
               }else{//fixed by runfriends@126.com
                   LOGGER.warn("comment in sql must meet :/*!mycat:type=value*/ or /*#mycat:type=value*/: "+stmt);
                   throw new SQLSyntaxErrorException("comment in sql must meet :/*!mcat:type=value*/ or /*#mycat:type=value*/: "+stmt);
               }
      }
   } else { //无注解
      stmt = stmt.trim();
      rrs = RouteStrategyFactory.getRouteStrategy().route(sysconf, schema, sqlType, stmt,
            charset, sc, tableId2DataNodeCache);
   }
   //select类型的SQL会对路由结果缓存
   if (rrs!=null && sqlType == ServerParse.SELECT && rrs.isCacheAble()) {
      sqlRouteCache.putIfAbsent(cacheKey, rrs);
   }
   return rrs;
}
```

### 获取路由策略

org.opencloudb.route.factory.RouteStrategyFactory#getRouteStrategy()

```java
public static RouteStrategy getRouteStrategy() {
   if(!isInit) {
      init();
      isInit = true;
   }
   return defaultStrategy;
}
```

```java
private static void init() {
   String defaultSqlParser = MycatServer.getInstance().getConfig().getSystem().getDefaultSqlParser();
   defaultSqlParser = defaultSqlParser == null ? "" : defaultSqlParser;
   //修改为ConcurrentHashMap，避免并发问题
   strategyMap.putIfAbsent("druidparser", new DruidMycatRouteStrategy());
   
   defaultStrategy = strategyMap.get(defaultSqlParser);
   if(defaultStrategy == null) {
      defaultStrategy = strategyMap.get("druidparser");
   }
}
```

### 获取路由结果

org.opencloudb.route.impl.AbstractRouteStrategy#route

```java
public RouteResultset route(SystemConfig sysConfig, SchemaConfig schema,int sqlType, String origSQL,
      String charset, ServerConnection sc, LayerCachePool cachePool) throws SQLNonTransientException {

    //处理一些路由之前的逻辑
   if ( beforeRouteProcess(schema, sqlType, origSQL, sc) )
      return null;

    //SQL 语句拦截
   String stmt = MycatServer.getInstance().getSqlInterceptor().interceptSQL(origSQL, sqlType);
   if (origSQL != stmt && LOGGER.isDebugEnabled()) {
      LOGGER.debug("sql intercepted to " + stmt + " from " + origSQL);
   }
   
   if (schema.isCheckSQLSchema()) {
      stmt = RouterUtil.removeSchema(stmt, schema.getName());
   }
	
   RouteResultset rrs = new RouteResultset(stmt, sqlType);

   /**
    * 优化debug loaddata输出cache的日志会极大降低性能
    */
   if (LOGGER.isDebugEnabled() && origSQL.startsWith(LoadData.loadDataHint)) {
      rrs.setCacheAble(false);
   }

  		/**
        * rrs携带ServerConnection的autocommit状态用于在sql解析的时候遇到
        * select ... for update的时候动态设定RouteResultsetNode的canRunInReadDB属性
        */
   if (sc != null ) {
      rrs.setAutocommit(sc.isAutocommit());
   }

    //DDL 语句的路由
   if (ServerParse.DDL == sqlType) {
      return RouterUtil.routeToDDLNode(rrs, sqlType, stmt, schema);
   }
   
   //检查是否有分片
   if (schema.isNoSharding() && ServerParse.SHOW != sqlType) {
      rrs = RouterUtil.routeToSingleNode(rrs, schema.getDataNode(), stmt); //单节点路由
   } else {
      RouteResultset returnedSet = routeSystemInfo(schema, sqlType, stmt, rrs);
      if (returnedSet == null) {
         rrs = routeNormalSqlWithAST(schema, stmt, rrs, charset, cachePool);
      }
   }

   return rrs;
}
```

## 执行SQL

org.opencloudb.server.NonBlockingSession#execute

```java
public void execute(RouteResultset rrs, int type) {
   // clear prev execute resources
   clearHandlesResources();
   if (LOGGER.isDebugEnabled()) {
      StringBuilder s = new StringBuilder();
      LOGGER.debug(s.append(source).append(rrs).toString() + " rrs ");
   }

   // 检查路由结果是否为空
   RouteResultsetNode[] nodes = rrs.getNodes();
   if (nodes == null || nodes.length == 0 || nodes[0].getName() == null
         || nodes[0].getName().equals("")) {
      source.writeErrMessage(ErrorCode.ER_NO_DB_ERROR,
            "No dataNode found ,please check tables defined in schema:"
                  + source.getSchema());
      return;
   }
   if (nodes.length == 1) { //单节点
      singleNodeHandler = new SingleNodeHandler(rrs, this);
      try {
         singleNodeHandler.execute();
      } catch (Exception e) {
         LOGGER.warn(new StringBuilder().append(source).append(rrs), e);
         source.writeErrMessage(ErrorCode.ERR_HANDLE_DATA, e.toString());
      }
   } else { //多节点
      boolean autocommit = source.isAutocommit();
      SystemConfig sysConfig = MycatServer.getInstance().getConfig()
            .getSystem();
      int mutiNodeLimitType = sysConfig.getMutiNodeLimitType();
      multiNodeHandler = new MultiNodeQueryHandler(type, rrs, autocommit,
            this);

      try {
         multiNodeHandler.execute();
      } catch (Exception e) {
         LOGGER.warn(new StringBuilder().append(source).append(rrs), e);
         source.writeErrMessage(ErrorCode.ERR_HANDLE_DATA, e.toString());
      }
   }
}
```

