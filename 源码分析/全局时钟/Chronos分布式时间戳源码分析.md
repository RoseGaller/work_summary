# Chronos源码分析

* [概述](#概述)
* [启动入口](#启动入口)
  * [创建ChronosServerWatcher](#创建chronosserverwatcher)
  * [创建ChronosServer](#创建chronosserver)
  * [选举主节点](#选举主节点)
  * [监听主节点的创建与删除](#监听主节点的创建与删除)
  * [监听连接事件](#监听连接事件)
  * [选举主节点成功](#选举主节点成功)
    * [初始化时间戳](#初始化时间戳)
    * [启动服务](#启动服务)
* [服务端请求处理](#服务端请求处理)
  * [获取单个时间戳](#获取单个时间戳)
  * [获取范围时间戳](#获取范围时间戳)


# 概述

Chronos，是实现高可用、高性能、提供全局唯一而且严格单调递增timestamp的服务。
Chronos采用主备架构，依赖Zookeeper，主服务器挂了以后备服务器迅速感知并接替服务，从而实现系统的高可用。
服务端使用[Thrift](http://thrift.apache.org/)框架，
全局只有唯一的ChronosServer提供服务，分配的timestamp保证严格单调递增。
并且将已分配的值持久化到ZooKeeper上，即使发生failover也能保证服务的正确性。

不同的业务使用不同的Chronos集群，支持海量的并发请求

# 启动入口

com.xiaomi.infra.chronos.ChronosServer#main

```java
public static void main(String[] args) {
  Properties properties = new Properties();
  LOG.info("Load chronos.cfg configuration from class path");
  try {
    //加载配置文件
   properties.load(ChronosServer.class.getClassLoader().getResourceAsStream("chronos.cfg"));
    //获取集群名称，设置baseZnode的值
    properties.setProperty(FailoverServer.BASE_ZNODE,
      "/chronos" + "/" + properties.getProperty(CLUSTER_NAME));
  } catch (IOException e) {
    LOG.fatal("Error to load chronos.cfg configuration, exit immediately", e);
    System.exit(0);
  }

  ChronosServer chronosServer = null;
  LOG.info("Init chronos server and connect ZooKeeper");
  try {
    //创建ChronosServer
    chronosServer = new ChronosServer(properties);
  } catch (Exception e) {
    LOG.fatal("Exception to init chronos server, exit immediately", e);
    System.exit(0);
  }
  //启动服务
  chronosServer.run();
}
```

## 创建ChronosServerWatcher

com.xiaomi.infra.chronos.ChronosServerWatcher#ChronosServerWatcher(java.util.Properties, boolean)

```java
public ChronosServerWatcher(Properties properties, boolean canInitZnode) throws IOException {
  super(properties, canInitZnode);
  //持久化时间戳
  persistentTimestampZnode = baseZnode + "/persistent-timestamp";
}
```

```java
public FailoverWatcher(Properties properties, boolean canInitZnode) throws IOException {
  this.properties = properties;

  baseZnode = properties.getProperty(FailoverServer.BASE_ZNODE, "/failover-server");
  masterZnode = baseZnode + "/master";
  backupServersZnode = baseZnode + "/backup-servers";
  //Zookeeper集群信息
  zkQuorum = properties.getProperty(FailoverServer.ZK_QUORUM, "127.0.0.1:2181");
  sessionTimeout = Integer.parseInt(properties
      .getProperty(FailoverServer.SESSION_TIMEOUT, "5000"));
  connectRetryTimes = Integer.parseInt(properties.getProperty(FailoverServer.CONNECT_RETRY_TIMES,
    "10"));
  isZkSecure = Boolean.parseBoolean(properties.getProperty(FailoverServer.ZK_SECURE, "false"));
  zkAdmin = properties.getProperty(FailoverServer.ZK_ADMIN, "h_chronos_admin");
  jaasFile = properties.getProperty(FailoverServer.JAAS_FILE, "../conf/jaas.conf");
  krb5File = properties.getProperty(FailoverServer.KRB5_FILE, "/etc/krb5.conf");
  String serverHost = properties.getProperty(FailoverServer.SERVER_HOST, "127.0.0.1");
  int serverPort = Integer.parseInt(properties.getProperty(FailoverServer.SERVER_PORT, "10086"));
  hostPort = new HostPort(serverHost, serverPort);

  if (isZkSecure) {
    LOG.info("Connect with secure ZooKeeper cluster, use " + jaasFile + " and " + krb5File);
    System.setProperty("java.security.auth.login.config", jaasFile);
    System.setProperty("java.security.krb5.conf", krb5File);
  }
  //与Zookeeper建立连接
  connectZooKeeper();
  if (canInitZnode) { //默认true
    initZnode(); //创建持久书类型的节点
  }
}
```

## 创建ChronosServer

com.xiaomi.infra.chronos.ChronosServer#ChronosServer(com.xiaomi.infra.chronos.ChronosServerWatcher, java.util.Properties)

```java
public ChronosServer(final ChronosServerWatcher chronosServerWatcher, Properties properties)
    throws TTransportException, ChronosException {
  super(chronosServerWatcher);
  this.chronosServerWatcher = chronosServerWatcher;
  this.properties = properties;
  //初始化ThriftServer
  LOG.info("Init thrift server in " + chronosServerWatcher.getHostPort().getHostPort());
  initThriftServer();
}
```

```java
private void initThriftServer() throws TTransportException, FatalChronosException,
    ChronosException {
  //工作线程数
  int maxThread = Integer.parseInt(properties.getProperty(MAX_THREAD,
    String.valueOf(Integer.MAX_VALUE)));
  String serverHost = properties.getProperty(FailoverServer.SERVER_HOST);
  int serverPort = Integer.parseInt(properties.getProperty(FailoverServer.SERVER_PORT));
  int socketTimeout = Integer
      .parseInt(properties.getProperty(SOCKET_TIMEOUT, String.valueOf(3000L)));
  TServerSocket serverTransport =
      new TServerSocket(new InetSocketAddress(serverHost, serverPort), socketTimeout);
  // Default message length limit is 10KB
  int messageLengthLimit =
      Integer.parseInt(properties.getProperty(MESSAGE_LENGTH_LIMIT, String.valueOf(10240)));
  Factory proFactory = new TBinaryProtocol.Factory(true, false, messageLengthLimit, -1L);
  //处理客户端获取时间戳请求
  chronosImplement = new ChronosImplement(properties, chronosServerWatcher);
  TProcessor processor = new ChronosService.Processor(chronosImplement);
  Args rpcArgs = new Args(serverTransport);
  rpcArgs.processor(processor);
  rpcArgs.protocolFactory(proFactory);
  rpcArgs.maxWorkerThreads(maxThread);
  thriftServer = new TThreadPoolServer(rpcArgs);
}
```

com.xiaomi.infra.chronos.zookeeper.FailoverServer#run

```java
public void run() {
  LOG.info("The server runs and prepares for leader election");
  if (failoverWatcher.blockUntilActive()) {//阻塞，直到集群中有leader选举成功
    LOG.info("The server becomes active master and prepare to do business logic");
    doAsActiveServer();//选举成功的节点执行此方法
  }
  failoverWatcher.close();
  LOG.info("The server exits after running business logic");
}
```

## 选举主节点

com.xiaomi.infra.chronos.zookeeper.FailoverWatcher#blockUntilActive

```java
public boolean blockUntilActive() { //选举成为leader的节点退出循环
  while (true) {
    try {
      if (ZooKeeperUtil.createEphemeralNodeAndWatch(this, masterZnode, hostPort.getHostPort().getBytes())) { //创建集群下的主节点
        LOG.info("Deleting ZNode for " + backupServersZnode + "/" + hostPort.getHostPort()
            + " from backup master directory");
        //从zk删除备份节点
        ZooKeeperUtil.deleteNodeFailSilent(this,
          backupServersZnode + "/" + hostPort.getHostPort());
        //已经有leader
        hasActiveServer.set(true);
        LOG.info("Become active master in " + hostPort.getHostPort());
        return true;
      }
      //已经有leader
      hasActiveServer.set(true);
      //创建备份节点，设置节点地址，注册Watcher
      ZooKeeperUtil.createEphemeralNodeAndWatch(this,
        backupServersZnode + "/" + hostPort.getHostPort(), hostPort.getHostPort().getBytes());
      //获取masternode的信息，并关注masternode节点的变化，注册watcher
      String msg;
      byte[] bytes = ZooKeeperUtil.getDataAndWatch(this, masterZnode);
      if (bytes == null) {
        msg = ("A master was detected, but went down before its address "
            + "could be read.  Attempting to become the next active master");
      } else {
        if (hostPort.getHostPort().equals(new String(bytes))) {
          msg = ("Current master has this master's address, " + hostPort.getHostPort() + "; master was restarted? Deleting node.");
          // Hurry along the expiration of the znode.
          ZooKeeperUtil.deleteNode(this, masterZnode);
        } else {
          msg = "Another master " + new String(bytes) + " is the active master, "
              + hostPort.getHostPort() + "; waiting to become the next active master";
        }
      }
      LOG.info(msg);
    } catch (KeeperException ke) {
      LOG.error("Received an unexpected KeeperException when block to become active, aborting",
        ke);
      return false;
    }
    //阻塞，直到下一次触发leader选举
    synchronized (hasActiveServer) {
      while (hasActiveServer.get()) {
        try {
          hasActiveServer.wait();
        } catch (InterruptedException e) {
          // We expect to be interrupted when a master dies, will fall out if so
          if (LOG.isDebugEnabled()) {
            LOG.debug("Interrupted while waiting to be master");
          }
          return false;
        }
      }
    }
  }
}
```

## 监听主节点的创建与删除

com.xiaomi.infra.chronos.zookeeper.FailoverWatcher#handleMasterNodeChange

```java
private void handleMasterNodeChange() {
  try {
    synchronized (hasActiveServer) {
      if (ZooKeeperUtil.watchAndCheckExists(this, masterZnode)) {//master节点存在
        if (LOG.isDebugEnabled()) {
          LOG.debug("A master is now available");
        }
        hasActiveServer.set(true);
      } else { //master节点不存在
        if (LOG.isDebugEnabled()) {
          LOG.debug("No master available. Notifying waiting threads");
        }
        //触发新一轮的leader选举
        hasActiveServer.set(false);
        hasActiveServer.notifyAll();
      }
    }
  } catch (KeeperException ke) {
    LOG.error("Received an unexpected KeeperException, aborting", ke);
  }
}
```

## 监听连接事件

com.xiaomi.infra.chronos.zookeeper.FailoverWatcher#processConnection

```java
protected void processConnection(WatchedEvent event) {
  switch (event.getState()) {
  case SyncConnected:
    LOG.info(hostPort.getHostPort() + " sync connect from ZooKeeper");
    try {
      waitToInitZooKeeper(2000); // init zookeeper in another thread, wait for a while
    } catch (Exception e) {
      LOG.fatal("Error to init ZooKeeper object after sleeping 2000 ms, exit immediately");
      System.exit(0);
    } 
    break;
  case Disconnected: // be triggered when kill the server or the leader of zk cluster 
    LOG.warn(hostPort.getHostPort() + " received disconnected from ZooKeeper");
    break;
  case AuthFailed://认证失败
    LOG.fatal(hostPort.getHostPort() + " auth fail, exit immediately");
    System.exit(0);
  case Expired://会话过期
    LOG.fatal(hostPort.getHostPort() + " received expired from ZooKeeper, exit immediately");
    System.exit(0); ////退出应用程序
    break;
  default:
    break;
  }
}
```

## 选举主节点成功

com.xiaomi.infra.chronos.ChronosServer#doAsActiveServer

```java
public void doAsActiveServer() {
  try {
    LOG.info("As active master, init timestamp from ZooKeeper");
    //初始化时间戳
    chronosImplement.initTimestamp();
  } catch (Exception e) {
    LOG.fatal("Exception to init timestamp from ZooKeeper, exit immediately");
    System.exit(0);
  }
  //当前节点为主节点
  chronosServerWatcher.setBeenActiveMaster(true);
  //启动ThriftServer，接收外部请求
  LOG.info("Start to accept thrift request");
  startThriftServer();
}
```

### 初始化时间戳

com.xiaomi.infra.chronos.ChronosImplement#initTimestamp

```java
public void initTimestamp() throws ChronosException, FatalChronosException {
  //从zk获取之前持久化的时间戳
  maxAssignedTimestamp = chronosServerWatcher.getPersistentTimestamp();
  //新的持久化时间戳，此后分配的时间戳都小于此值
  long newPersistentTimestamp = maxAssignedTimestamp + zkAdvanceTimestamp;
  //同步持久化到zk
  chronosServerWatcher.setPersistentTimestamp(newPersistentTimestamp);
  LOG.info("Get persistent timestamp " + maxAssignedTimestamp + " and set "
      + newPersistentTimestamp + " in ZooKeeper");
}
```

### 启动服务

com.xiaomi.infra.chronos.ChronosServer#startThriftServer

```java
public void startThriftServer() {
  if (thriftServer != null) {
    thriftServer.serve(); //thrift协议，接收外部请求
  }
}
```

# 服务端请求处理

## 获取单个时间戳

com.xiaomi.infra.chronos.ChronosImplement#getTimestamp

```java
public long getTimestamp() throws TException {
  return getTimestamps(1);
}
```

## 获取范围时间戳

```
客户端可以使用的范围[timestamp, timestamp + range)，避免多次网络通信
```

com.xiaomi.infra.chronos.ChronosImplement#getTimestamps

```java
public long getTimestamps(int range) throws TException {

  // can get 2^18(262144) times for each millisecond 
  long currentTime = System.currentTimeMillis() << 18;
  synchronized (this) { //同步代码块
    // maxAssignedTimestamp is assigned last time, can't return currentTime when it's less or equal
    if (currentTime > maxAssignedTimestamp) {//maxAssignedTimestamp，服务启动时从zk获取，之后为上次分配的时间戳。
      maxAssignedTimestamp = currentTime + range - 1; //当前时间戳大于上次分配的时间戳或者zk存储的时间戳
    } else { //当前时间戳小于上次分配的时间戳
      maxAssignedTimestamp += range;
    }
    //返回给客户端的时间戳范围
    // now [maxAssignedTimestamp - range + 1, maxAssignedTimestamp] will be returned

    // for correctness, compare with persistent timestamp and set it if necessary
    if (maxAssignedTimestamp >= chronosServerWatcher.getCachedPersistentTimestamp()) {//有可能range过大，导致maxAssignedTimestamp大于zk存储的时间戳
      // wait for the result of asyn set
      sleepUntilAsyncSet();//如果有线程异步持久化时间戳等待持久化成功
      // sync set persistent timestamp if necessary
      if (maxAssignedTimestamp >= chronosServerWatcher.getCachedPersistentTimestamp()) {//二次判断，分配的最大时间戳是否大于zk存储的时间戳
        //zkAdvanceTimestamp，预分配的时间戳范围
        //生成新的时间戳
        long newPersistentTimestamp = maxAssignedTimestamp + zkAdvanceTimestamp;
        if (LOG.isDebugEnabled()) {
          LOG.debug("Try to sync set persistent timestamp " + newPersistentTimestamp);
        }
        try {
          //同步持久化到zk，保证zk存储的时间戳大于当前分配的时间戳，哪怕leader宕机，保证分配的时间戳不会重复，不会小于之前分配的时间戳
          chronosServerWatcher.setPersistentTimestamp(newPersistentTimestamp);
        } catch (ChronosException e) {
          LOG.fatal("Error to set persistent timestamp, exit immediately");
          System.exit(0);
        }
      }
    }
    //基于性能考虑，避免在每次分配了时间戳都持久化到zk，可以在分配的时间戳大于等于持久化的时间戳之前，异步持久化
    // for performance, async set persistent timestamp before reaching persistent timestamp
    if (!isAsyncSetPersistentTimestamp
        && maxAssignedTimestamp >= chronosServerWatcher.getCachedPersistentTimestamp()
            - zkAdvanceTimestamp * 0.5) {
      long newPersistentTimestamp = chronosServerWatcher.getCachedPersistentTimestamp()
          + zkAdvanceTimestamp;
      if (LOG.isDebugEnabled()) {
        LOG.debug("Try to async set persistent timestamp " + newPersistentTimestamp);
      }
      //正在异步持久化时间戳的标志
      isAsyncSetPersistentTimestamp = true;
      //异步持久化时间戳
      asyncSetPersistentTimestamp(newPersistentTimestamp);
    }
    return maxAssignedTimestamp - range + 1;
  }
}
```

# 总结

1、ZK不适用频繁修改的场景。为了保证生成的时间戳单调递增，每次都需要保存到ZK，但是为了避免频繁的网络通信以及ZK节点间的数据同步，采用预分配时间戳的方式，需要配置预分配的时间范围，把已分配的最大时间戳和预分配的时间范围得相加到一个新的最大时间戳，保存至ZK。此后分配的时间戳都小于ZK存储的时间戳。当分配的时间戳快要接近ZK保存的时间戳时，通过异步的方式将ZK的时间戳更新为当前最大的时间戳加上预分配时间范围。

2、为了降低客户端与服务端的通信开销，一次请求返回多个时间戳。

