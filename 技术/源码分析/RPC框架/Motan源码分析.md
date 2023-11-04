# Motan源码分析 

* [服务发布](#服务发布)
  * [入口](#入口)
  * [暴露服务](#暴露服务)
  * [装饰过滤器](#装饰过滤器)
  * [开始暴露服务](#开始暴露服务)
  * [创建Exporter](#创建exporter)
  * [初始化网络监听](#初始化网络监听)
  * [注册服务入口](#注册服务入口)
  * [创建Registry](#创建registry)
  * [注册服务](#注册服务)
  * [注销服务](#注销服务)
  * [StandardThreadExecutor](#standardthreadexecutor)
* [消费端](#消费端)
  * [入口](#入口-1)
    * [生成代理](#生成代理)
  * [调用流程](#调用流程)
    * [调用入口](#调用入口)
    * [获取方法返回类型](#获取方法返回类型)
    * [执行请求](#执行请求)
    * [集群策略](#集群策略)
    * [发起远端请求](#发起远端请求)
* [总结](#总结)


# 服务发布

## 入口

com.weibo.api.motan.config.ServiceConfig#export

```java
public synchronized void export() {
    if (exported.get()) {
        LoggerUtil.warn(String.format("%s has already been expoted, so ignore the export request!", interfaceClass.getName()));
        return;
    }
    //验证接口、方法
    checkInterfaceAndMethods(interfaceClass, methods);
    //加载注册中心地址
    List<URL> registryUrls = loadRegistryUrls();
    if (registryUrls == null || registryUrls.size() == 0) {
        throw new IllegalStateException("Should set registry config for service:" + interfaceClass.getName());
    }
    //协议->端口号，不同的协议对应不同的端口号
    Map<String, Integer> protocolPorts = getProtocolAndPort();
    for (ProtocolConfig protocolConfig : protocols) {
        Integer port = protocolPorts.get(protocolConfig.getId());
        if (port == null) {
            throw new MotanServiceException(String.format("Unknow port in service:%s, protocol:%s", interfaceClass.getName(),
                    protocolConfig.getId()));
        }
        //暴露服务
        doExport(protocolConfig, port, registryUrls);
    }
    afterExport();
}
```

## 暴露服务

```java
private void doExport(ProtocolConfig protocolConfig, int port, List<URL> registryURLs) {
    String protocolName = protocolConfig.getName();
    if (protocolName == null || protocolName.length() == 0) {
        protocolName = URLParamType.protocol.getValue();
    }
    String hostAddress = host;
    if (StringUtils.isBlank(hostAddress) && basicService != null) {
        hostAddress = basicService.getHost();
    }
    if (NetUtils.isInvalidLocalHost(hostAddress)) {
        hostAddress = getLocalHostAddress(registryURLs);
    }
    Map<String, String> map = new HashMap<String, String>();
    map.put(URLParamType.nodeType.getName(), MotanConstants.NODE_TYPE_SERVICE);
    map.put(URLParamType.refreshTimestamp.getName(), String.valueOf(System.currentTimeMillis()));
    collectConfigParams(map, protocolConfig, basicService, extConfig, this);
    collectMethodConfigParams(map, this.getMethods());
    URL serviceUrl = new URL(protocolName, hostAddress, port, interfaceClass.getName(), map);
    if (serviceExists(serviceUrl)) {
        LoggerUtil.warn(String.format("%s configService is malformed, for same service (%s) already exists ", interfaceClass.getName(),
                serviceUrl.getIdentity()));
        throw new MotanFrameworkException(String.format("%s configService is malformed, for same service (%s) already exists ",
                interfaceClass.getName(), serviceUrl.getIdentity()), MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    }
    List<URL> urls = new ArrayList<URL>();
    // injvm 协议只支持注册到本地，其他协议可以注册到local、remote
    if (MotanConstants.PROTOCOL_INJVM.equals(protocolConfig.getId())) {
        URL localRegistryUrl = null;
        for (URL ru : registryURLs) {
            if (MotanConstants.REGISTRY_PROTOCOL_LOCAL.equals(ru.getProtocol())) {
                localRegistryUrl = ru.createCopy();
                break;
            }
        }
        if (localRegistryUrl == null) {
            localRegistryUrl =
                    new URL(MotanConstants.REGISTRY_PROTOCOL_LOCAL, hostAddress, MotanConstants.DEFAULT_INT_VALUE,
                            RegistryService.class.getName());
        }
        urls.add(localRegistryUrl);
    } else {
        for (URL ru : registryURLs) {
            urls.add(ru.createCopy());
        }
    }
    for (URL u : urls) {
        u.addParameter(URLParamType.embed.getName(), StringTools.urlEncode(serviceUrl.toFullStr()));
        registereUrls.add(u.createCopy());
    }
 		 //SimpleConfigHandler
    ConfigHandler configHandler = ExtensionLoader.getExtensionLoader(ConfigHandler.class).getExtension(MotanConstants.DEFAULT_VALUE);
    exporters.add(configHandler.export(interfaceClass, ref, urls));
}
```

com.weibo.api.motan.config.handler.SimpleConfigHandler#export

```java
public <T> Exporter<T> export(Class<T> interfaceClass, T ref, List<URL> registryUrls) {
    String serviceStr = StringTools.urlDecode(registryUrls.get(0).getParameter(URLParamType.embed.getName()));
    URL serviceUrl = URL.valueOf(serviceStr);
    // 利用protocol decorator来增加filter特性
    String protocolName = serviceUrl.getParameter(URLParamType.protocol.getName(), URLParamType.protocol.getValue());
    //DefaultRpcProtocol
    Protocol orgProtocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(protocolName);
    //DefaultProvider
    Provider<T> provider = getProvider(orgProtocol, ref, serviceUrl, interfaceClass);
    //装饰协议，添加过滤器
    Protocol protocol = new ProtocolFilterDecorator(orgProtocol);
    Exporter<T> exporter = protocol.export(provider, serviceUrl);
    //注册服务到注册中心
    register(registryUrls, serviceUrl);
    return exporter;
}
```

com.weibo.api.motan.protocol.support.ProtocolFilterDecorator#export

```java
public <T> Exporter<T> export(Provider<T> provider, URL url) {
    return protocol.export(decorateWithFilter(provider, url), url);
}
```

## 装饰过滤器

com.weibo.api.motan.protocol.support.ProtocolFilterDecorator#decorateWithFilter(com.weibo.api.motan.rpc.Provider<T>, com.weibo.api.motan.rpc.URL)

```java
private <T> Provider<T> decorateWithFilter(final Provider<T> provider, URL url) {
    List<Filter> filters = getFilters(url, MotanConstants.NODE_TYPE_SERVICE);
    if (filters == null || filters.size() == 0) {
        return provider;
    }
    Provider<T> lastProvider = provider;
    for (Filter filter : filters) {
        final Filter f = filter;
        if (f instanceof InitializableFilter) {
            ((InitializableFilter) f).init(lastProvider);
        }
        final Provider<T> lp = lastProvider;
        lastProvider = new Provider<T>() {
            @Override
            public Response call(Request request) {
                return f.filter(lp, request);
            }

            @Override
            public String desc() {
                return lp.desc();
            }

            @Override
            public void destroy() {
                lp.destroy();
            }

            @Override
            public Class<T> getInterface() {
                return lp.getInterface();
            }

            @Override
            public Method lookupMethod(String methodName, String methodDesc) {
                return lp.lookupMethod(methodName, methodDesc);
            }

            @Override
            public URL getUrl() {
                return lp.getUrl();
            }

            @Override
            public void init() {
                lp.init();
            }

            @Override
            public boolean isAvailable() {
                return lp.isAvailable();
            }
          @Override
          public T getImpl() {
            return provider.getImpl();
          }
        };
    }
    return lastProvider;
}
```

## 开始暴露服务

com.weibo.api.motan.protocol.AbstractProtocol#export

```java
public <T> Exporter<T> export(Provider<T> provider, URL url) {
    if (url == null) {
        throw new MotanFrameworkException(this.getClass().getSimpleName() + " export Error: url is null",
                MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    }
    if (provider == null) {
        throw new MotanFrameworkException(this.getClass().getSimpleName() + " export Error: provider is null, url=" + url,
                MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    }
    //protocol key: protocol://host:port/group/interface/version
    String protocolKey = MotanFrameworkUtil.getProtocolKey(url);
    synchronized (exporterMap) {
        Exporter<T> exporter = (Exporter<T>) exporterMap.get(protocolKey);

        if (exporter != null) {
            throw new MotanFrameworkException(this.getClass().getSimpleName() + " export Error: service already exist, url=" + url,
                    MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
        }
        //创建Exporter
        exporter = createExporter(provider, url);
        //初始化
        exporter.init();
        exporterMap.put(protocolKey, exporter);
        LoggerUtil.info(this.getClass().getSimpleName() + " export Success: url=" + url);
        return exporter;
    }
```

## 创建Exporter

com.weibo.api.motan.protocol.rpc.DefaultRpcProtocol#createExporter

```java
protected <T> Exporter<T> createExporter(Provider<T> provider, URL url) {
    return new DefaultRpcExporter<T>(provider, url, this.ipPort2RequestRouter, this.exporterMap);
}
```

## 初始化网络监听

com.weibo.api.motan.rpc.AbstractNode#init

```java
public synchronized void init() {
    if (init) {
        LoggerUtil.warn(this.getClass().getSimpleName() + " node already init: " + desc());
        return;
    }
    boolean result = doInit();
    if (!result) {
        LoggerUtil.error(this.getClass().getSimpleName() + " node init Error: " + desc());
        throw new MotanFrameworkException(this.getClass().getSimpleName() + " node init Error: " + desc(),
                MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    } else {
        LoggerUtil.info(this.getClass().getSimpleName() + " node init Success: " + desc());
        init = true;
        available = true;
    }
}
```

```java
protected boolean doInit() {
    return server.open();
}
```

com.weibo.api.motan.transport.netty4.NettyServer#open

```java
public boolean open() {
    if (isAvailable()) {
        LoggerUtil.warn("NettyServer ServerChannel already Open: url=" + url);
        return state.isAliveState();
    }
    if (bossGroup == null) {
        bossGroup = new NioEventLoopGroup(1);
        workerGroup = new NioEventLoopGroup();
    }

    LoggerUtil.info("NettyServer ServerChannel start Open: url=" + url);
    boolean shareChannel = url.getBooleanParameter(URLParamType.shareChannel.getName(), URLParamType.shareChannel.getBooleanValue());
    final int maxContentLength = url.getIntParameter(URLParamType.maxContentLength.getName(), URLParamType.maxContentLength.getIntValue());
    int maxServerConnection = url.getIntParameter(URLParamType.maxServerConnection.getName(), URLParamType.maxServerConnection.getIntValue());
    int workerQueueSize = url.getIntParameter(URLParamType.workerQueueSize.getName(), URLParamType.workerQueueSize.getIntValue());

    int minWorkerThread, maxWorkerThread;

    if (shareChannel) {
        minWorkerThread = url.getIntParameter(URLParamType.minWorkerThread.getName(), MotanConstants.NETTY_SHARECHANNEL_MIN_WORKDER);
        maxWorkerThread = url.getIntParameter(URLParamType.maxWorkerThread.getName(), MotanConstants.NETTY_SHARECHANNEL_MAX_WORKDER);
    } else {
        minWorkerThread = url.getIntParameter(URLParamType.minWorkerThread.getName(), MotanConstants.NETTY_NOT_SHARECHANNEL_MIN_WORKDER);
        maxWorkerThread = url.getIntParameter(URLParamType.maxWorkerThread.getName(), MotanConstants.NETTY_NOT_SHARECHANNEL_MAX_WORKDER);
    }
    //创建标准线程池
    standardThreadExecutor = (standardThreadExecutor != null && !standardThreadExecutor.isShutdown()) ? standardThreadExecutor
            : new StandardThreadExecutor(minWorkerThread, maxWorkerThread, workerQueueSize, new DefaultThreadFactory("NettyServer-" + url.getServerPortStr(), true));
    //预启动核心线程
    standardThreadExecutor.prestartAllCoreThreads();
    //管理连接
    channelManage = new NettyServerChannelManage(maxServerConnection);
    //服务端网络配置
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    pipeline.addLast("channel_manage", channelManage);
                    pipeline.addLast("decoder", new NettyDecoder(codec, NettyServer.this, maxContentLength));
                    pipeline.addLast("encoder", new NettyEncoder());
                    //处理请求、响应
                    NettyChannelHandler handler = new NettyChannelHandler(NettyServer.this, messageHandler, standardThreadExecutor);
                    pipeline.addLast("handler", handler);
                }
            });
    serverBootstrap.childOption(ChannelOption.TCP_NODELAY, true);
    serverBootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
    //绑定端口
    ChannelFuture channelFuture = serverBootstrap.bind(new InetSocketAddress(url.getPort()));
    channelFuture.syncUninterruptibly();
    serverChannel = channelFuture.channel();
    state = ChannelState.ALIVE;
    StatsUtil.registryStatisticCallback(this);
    LoggerUtil.info("NettyServer ServerChannel finish Open: url=" + url);
    return state.isAliveState();
}
```

## 注册服务入口

com.weibo.api.motan.config.handler.SimpleConfigHandler#register

```java
private void register(List<URL> registryUrls, URL serviceUrl) {

    for (URL url : registryUrls) {
        // 根据check参数的设置，register失败可能会抛异常，上层应该知晓
        RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getExtension(url.getProtocol());
        if (registryFactory == null) {
            throw new MotanFrameworkException(new MotanErrorMsg(500, MotanErrorMsgConstant.FRAMEWORK_REGISTER_ERROR_CODE,
                    "register error! Could not find extension for registry protocol:" + url.getProtocol()
                            + ", make sure registry module for " + url.getProtocol() + " is in classpath!"));
        }
        Registry registry = registryFactory.getRegistry(url);
        registry.register(serviceUrl);
    }
}
```

## 创建Registry

com.weibo.api.motan.registry.zookeeper.ZookeeperRegistryFactory#createRegistry

```java
protected Registry createRegistry(URL registryUrl) {
    try {
        int timeout = registryUrl.getIntParameter(URLParamType.connectTimeout.getName(), URLParamType.connectTimeout.getIntValue());
        int sessionTimeout = registryUrl.getIntParameter(URLParamType.registrySessionTimeout.getName(), URLParamType.registrySessionTimeout.getIntValue());
        ZkClient zkClient = createInnerZkClient(registryUrl.getParameter("address"), sessionTimeout, timeout);
        return new ZookeeperRegistry(registryUrl, zkClient);
    } catch (ZkException e) {
        LoggerUtil.error("[ZookeeperRegistry] fail to connect zookeeper, cause: " + e.getMessage());
        throw e;
    }
}
```

## 注册服务

com.weibo.api.motan.registry.support.AbstractRegistry#register

```java
public void register(URL url) {
    if (url == null) {
        LoggerUtil.warn("[{}] register with malformed param, url is null", registryClassName);
        return;
    }
    LoggerUtil.info("[{}] Url ({}) will register to Registry [{}]", registryClassName, url, registryUrl.getIdentity());
    doRegister(removeUnnecessaryParmas(url.createCopy()));
    registeredServiceUrls.add(url);
    // available if heartbeat switcher already open
    if (MotanSwitcherUtil.isOpen(MotanConstants.REGISTRY_HEARTBEAT_SWITCHER)) {
        available(url);
    }
}
```

## 注销服务

com.weibo.api.motan.registry.zookeeper.ZookeeperRegistry#doRegister

```java
protected void doRegister(URL url) {
    try {
        serverLock.lock();
        // 防止旧节点未正常注销
        removeNode(url, ZkNodeType.AVAILABLE_SERVER);
        removeNode(url, ZkNodeType.UNAVAILABLE_SERVER);
        createNode(url, ZkNodeType.UNAVAILABLE_SERVER);
    } catch (Throwable e) {
        throw new MotanFrameworkException(String.format("Failed to register %s to zookeeper(%s), cause: %s", url, getUrl(), e.getMessage()), e);
    } finally {
        serverLock.unlock();
    }
}
```

## StandardThreadExecutor

```
 java.util.concurrent.threadPoolExecutor execute执行策略：        
				优先offer到queue，queue满后再扩充线程到maxThread，如果已经到了maxThread就reject.比较适合于CPU密集型应用（比如runnable内部执行的操作都在JVM内部，memory copy, or compute等等）
StandardThreadExecutor execute执行策略：	优先扩充线程到maxThread，再offer到queue，如果满了就reject. 
比较适合于业务处理需要远程资源的场景
```

com.weibo.api.motan.core.StandardThreadExecutor#StandardThreadExecutor(int, int, long, java.util.concurrent.TimeUnit, int, java.util.concurrent.ThreadFactory, java.util.concurrent.RejectedExecutionHandler)

```java
public StandardThreadExecutor(int coreThreads, int maxThreads, long keepAliveTime, TimeUnit unit,
      int queueCapacity, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
   super(coreThreads, maxThreads, keepAliveTime, unit, new ExecutorQueue(), threadFactory, handler);
   ((ExecutorQueue) getQueue()).setStandardThreadExecutor(this);
   submittedTasksCount = new AtomicInteger(0);
   // 最大并发任务限制： 队列buffer数 + 最大线程数 
   maxSubmittedTaskCount = queueCapacity + maxThreads; 
}
```

com.weibo.api.motan.core.StandardThreadExecutor#execute

```java
public void execute(Runnable command) {
   //任务数+1
   int count = submittedTasksCount.incrementAndGet();
   // 超过最大的并发任务限制，进行 reject
   // 依赖的LinkedTransferQueue没有长度限制，因此这里进行控制 
   if (count > maxSubmittedTaskCount) {
      submittedTasksCount.decrementAndGet();
     //执行拒绝策略。默认AbortPolicy抛出异常
      getRejectedExecutionHandler().rejectedExecution(command, this);
   }
   try {
      super.execute(command);
   } catch (RejectedExecutionException rx) {
      if (!((ExecutorQueue) getQueue()).force(command)) {
         submittedTasksCount.decrementAndGet();
         getRejectedExecutionHandler().rejectedExecution(command, this);
      }
   }
}
```

ExecutorQueue

继承自LinkedTransferQueue，相比与LinkedBlockingQueue有明显提升 

com.weibo.api.motan.core.ExecutorQueue#offer

```java
public boolean offer(Runnable o) {
    //当前线程池的线程数量
   int poolSize = threadPoolExecutor.getPoolSize();
	 //如果线程池线程数已经最大，直接将任务加入队列
   if (poolSize == threadPoolExecutor.getMaximumPoolSize()) {
      return super.offer(o);
   }
  //有空闲线程，将任务加入队列
   if (threadPoolExecutor.getSubmittedTasksCount() <= poolSize) {
      return super.offer(o);
   }
  //如果当前的线程数量小于最大线程数，返回false，强制创建线程，处理任务
   if (poolSize < threadPoolExecutor.getMaximumPoolSize()) {
      return false;
   }
   //直接放入队列
   return super.offer(o);
}
```

# 消费端

## 入口

com.weibo.api.motan.config.RefererConfig#getRef

```java
public T getRef() {
    if (ref == null) {
        initRef();
    }
    return ref;
}
```

```java
public synchronized void initRef() {
    if (initialized.get()) {
        return;
    }

    try {
        interfaceClass = (Class) Class.forName(interfaceClass.getName(), true, Thread.currentThread().getContextClassLoader());
    } catch (ClassNotFoundException e) {
        throw new MotanFrameworkException("ReferereConfig initRef Error: Class not found " + interfaceClass.getName(), e,
                MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    }

    if (CollectionUtil.isEmpty(protocols)) {
        throw new MotanFrameworkException(String.format("%s RefererConfig is malformed, for protocol not set correctly!",
                interfaceClass.getName()));
    }

    checkInterfaceAndMethods(interfaceClass, methods);

    clusterSupports = new ArrayList<>(protocols.size());
    List<Cluster<T>> clusters = new ArrayList<>(protocols.size());
    String proxy = null;

    ConfigHandler configHandler = ExtensionLoader.getExtensionLoader(ConfigHandler.class).getExtension(MotanConstants.DEFAULT_VALUE);

    List<URL> registryUrls = loadRegistryUrls();
    String localIp = getLocalHostAddress(registryUrls);
    for (ProtocolConfig protocol : protocols) {
        Map<String, String> params = new HashMap<>();
        params.put(URLParamType.nodeType.getName(), MotanConstants.NODE_TYPE_REFERER);
        params.put(URLParamType.version.getName(), URLParamType.version.getValue());
        params.put(URLParamType.refreshTimestamp.getName(), String.valueOf(System.currentTimeMillis()));

        collectConfigParams(params, protocol, basicReferer, extConfig, this);
        collectMethodConfigParams(params, this.getMethods());

        String path = StringUtils.isBlank(serviceInterface) ? interfaceClass.getName() : serviceInterface;
        URL refUrl = new URL(protocol.getName(), localIp, MotanConstants.DEFAULT_INT_VALUE, path, params);
        ClusterSupport<T> clusterSupport = createClusterSupport(refUrl, configHandler, registryUrls);

        clusterSupports.add(clusterSupport);
        clusters.add(clusterSupport.getCluster());
        if (proxy == null) {
          //默认JDK
            String defaultValue = StringUtils.isBlank(serviceInterface) ? URLParamType.proxy.getValue() : MotanConstants.PROXY_COMMON;
            proxy = refUrl.getParameter(URLParamType.proxy.getName(), defaultValue);
        }
    }
		//生成代理
    ref = configHandler.refer(interfaceClass, clusters, proxy);
    //初始化完成
    initialized.set(true);
}
```

### 生成代理

com.weibo.api.motan.config.handler.SimpleConfigHandler#refer

```java
public <T> T refer(Class<T> interfaceClass, List<Cluster<T>> clusters, String proxyType) {
    ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(proxyType); //默认JDK代理
    return proxyFactory.getProxy(interfaceClass, clusters);
}
```

```java
public <T> T getProxy(Class<T> clz, List<Cluster<T>> clusters) {
    return (T) Proxy.newProxyInstance(clz.getClassLoader(), new Class[]{clz}, new RefererInvocationHandler<>(clz, clusters));
}
```

## 调用流程

### 调用入口

com.weibo.api.motan.proxy.RefererInvocationHandler#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (isLocalMethod(method)) { //Object方法
        if ("toString".equals(method.getName())) {
            return clustersToString();
        }
        if ("equals".equals(method.getName())) {
            return proxyEquals(args[0]);
        }
        if ("hashCode".equals(method.getName())) {
            return this.clusters == null ? 0 : this.clusters.hashCode();
        }
        throw new MotanServiceException("can not invoke local method:" + method.getName());
    }
    //远程请求
    DefaultRequest request = new DefaultRequest();
    request.setRequestId(RequestIdGenerator.getRequestId()); //生成请求唯一标志
    request.setArguments(args);
    String methodName = method.getName();
    boolean async = false;
    //判断是否异步执行，方法名以async结尾并且返回类型为ResponseFuture
    if (methodName.endsWith(MotanConstants.ASYNC_SUFFIX) && method.getReturnType().equals(ResponseFuture.class)) {
        methodName = MotanFrameworkUtil.removeAsyncSuffix(methodName);//去除末尾的async
        async = true;
    }
    request.setMethodName(methodName);
    request.setParamtersDesc(ReflectUtil.getMethodParamDesc(method));
    request.setInterfaceName(interfaceName);

    return invokeRequest(request, getRealReturnType(async, this.clz, method, methodName), async);
}
```

### 获取方法返回类型

```java
private Class<?> getRealReturnType(boolean asyncCall, Class<?> clazz, Method method, String methodName) {
    if (asyncCall) {
        try {
            Method m = clazz.getMethod(methodName, method.getParameterTypes());
            return m.getReturnType();
        } catch (Exception e) {
            LoggerUtil.warn("RefererInvocationHandler get real return type fail. err:" + e.getMessage());
            return method.getReturnType();
        }
    } else {
        return method.getReturnType();
    }
}
```

### 执行请求

区分异步调用开始同步调用

com.weibo.api.motan.proxy.AbstractRefererHandler#invokeRequest

```java
Object invokeRequest(Request request, Class returnType, boolean async) throws Throwable {
    RpcContext curContext = RpcContext.getContext();
    curContext.putAttribute(MotanConstants.ASYNC_SUFFIX, async); //异步属性

    // set rpc context attachments to request
    Map<String, String> attachments = curContext.getRpcAttachments();
    if (!attachments.isEmpty()) {
        for (Map.Entry<String, String> entry : attachments.entrySet()) {
            request.setAttachment(entry.getKey(), entry.getValue());
        }
    }

    // add to attachment if client request id is set
    if (StringUtils.isNotBlank(curContext.getClientRequestId())) {
        request.setAttachment(URLParamType.requestIdFromClient.getName(), curContext.getClientRequestId());
    }

    // 当 referer配置多个protocol的时候，比如A,B,C，
    // 那么正常情况下只会使用A，如果A被开关降级，那么就会使用B，B也被降级，那么会使用C
    for (Cluster<T> cluster : clusters) {
        String protocolSwitcher = MotanConstants.PROTOCOL_SWITCHER_PREFIX + cluster.getUrl().getProtocol();

        Switcher switcher = switcherService.getSwitcher(protocolSwitcher);

        if (switcher != null && !switcher.isOn()) { //此协议不可用
            continue;
        }
        //version、group
        request.setAttachment(URLParamType.version.getName(), cluster.getUrl().getVersion());
        request.setAttachment(URLParamType.clientGroup.getName(), cluster.getUrl().getGroup());
        // 带上client的application和module
        request.setAttachment(URLParamType.application.getName(), cluster.getUrl().getApplication());
        request.setAttachment(URLParamType.module.getName(), cluster.getUrl().getModule());

        Response response = null;
        boolean throwException = Boolean.parseBoolean(cluster.getUrl().getParameter(URLParamType.throwException.getName(), URLParamType.throwException.getValue()));
        try {
            MotanFrameworkUtil.logEvent(request, MotanConstants.TRACE_INVOKE);
            response = cluster.call(request);
            if (async) { //异步
                if (response instanceof ResponseFuture) {
                    ((ResponseFuture) response).setReturnType(returnType);
                    return response;
                } else {
                    ResponseFuture responseFuture = new DefaultResponseFuture(request, 0, cluster.getUrl());
                    if (response.getException() != null) {
                        responseFuture.onFailure(response);
                    } else {
                        responseFuture.onSuccess(response);
                    }
                    responseFuture.setReturnType(returnType);
                    return responseFuture;
                }
            } else {//同步
                Object value = response.getValue();
                if (value != null && value instanceof DeserializableObject) {
                    try {
                        value = ((DeserializableObject) value).deserialize(returnType);
                    } catch (IOException e) {
                        LoggerUtil.error("deserialize response value fail! deserialize type:" + returnType, e);
                        throw new MotanFrameworkException("deserialize return value fail! deserialize type:" + returnType, e);
                    }
                }
                return value;
            }
        } catch (RuntimeException e) {
            if (ExceptionUtil.isBizException(e)) {
                Throwable t = e.getCause();
                // 只抛出Exception，防止抛出远程的Error
                if (t != null && t instanceof Exception) {
                    throw t;
                } else {
                    String msg = t == null ? "biz exception cause is null. origin error msg : " + e.getMessage() : ("biz exception cause is throwable error:" + t.getClass() + ", errmsg:" + t.getMessage());
                    throw new MotanServiceException(msg);
                }
            } else if (!throwException) {
                LoggerUtil.warn("RefererInvocationHandler invoke false, so return default value: uri=" + cluster.getUrl().getUri() + " " + MotanFrameworkUtil.toString(request), e);
                return getDefaultReturnValue(returnType);
            } else {
                LoggerUtil.error("RefererInvocationHandler invoke Error: uri=" + cluster.getUrl().getUri() + " " + MotanFrameworkUtil.toString(request), e);
                throw e;
            }
        }
    }
    throw new MotanServiceException("Referer call Error: cluster not exist, interface=" + interfaceName + " " + MotanFrameworkUtil.toString(request), MotanErrorMsgConstant.SERVICE_UNFOUND, false);
}
```

com.weibo.api.motan.cluster.support.ClusterSpi#call

```java
public Response call(Request request) {
    if (available.get()) {
        try {
            return haStrategy.call(request, loadBalance); //集群策略
        } catch (Exception e) {
            return callFalse(request, e);
        }
    }
    throw new MotanServiceException(String.format("ClusterSpi Call false for request: %s, ClusterSpi not created or destroyed", request),
            MotanErrorMsgConstant.SERVICE_UNFOUND, false);
}
```

### 集群策略

com.weibo.api.motan.cluster.ha.FailfastHaStrategy#call

```java
public Response call(Request request, LoadBalance<T> loadBalance) { //快速失败
    Referer<T> refer = loadBalance.select(request); //负载均衡
    return refer.call(request);
}
```

#### 负载均衡

com.weibo.api.motan.cluster.loadbalance.RoundRobinLoadBalance#doSelect

```java
protected Referer<T> doSelect(Request request) { //轮询策略
    List<Referer<T>> referers = getReferers();

    int index = getNextNonNegative();
    for (int i = 0; i < referers.size(); i++) {
        Referer<T> ref = referers.get((i + index) % referers.size());
        if (ref.isAvailable()) { //判断连接是否可用
            return ref;
        }
        idx.incrementAndGet();
    }
    return null;
}
```

#### 连接是否可用

com.weibo.api.motan.protocol.rpc.DefaultRpcReferer#isAvailable

```java
public boolean isAvailable() {
    return client.isAvailable();
}
```

com.weibo.api.motan.transport.netty4.NettyClient#isAvailable

```java
public boolean isAvailable() {
    return state.isAliveState();
}
```

com.weibo.api.motan.rpc.AbstractReferer#call

```java
public Response call(Request request) {
    if (!isAvailable()) {
        throw new MotanFrameworkException(this.getClass().getSimpleName() + " call Error: node is not available, url=" + url.getUri()
                + " " + MotanFrameworkUtil.toString(request));
    }
    incrActiveCount(request);
    Response response = null;
    try {
        response = doCall(request);
        return response;
    } finally {
        decrActiveCount(request, response);
    }
}
```

### 发起远端请求

com.weibo.api.motan.protocol.rpc.DefaultRpcReferer#doCall

```java
protected Response doCall(Request request) {
    try {
        // 为了能够实现跨group请求，需要使用server端的group。
        request.setAttachment(URLParamType.group.getName(), serviceUrl.getGroup());
        return client.request(request);
    } catch (TransportException exception) {
        throw new MotanServiceException("DefaultRpcReferer call Error: url=" + url.getUri(), exception);
    }
}
```

com.weibo.api.motan.transport.netty4.NettyClient#request(com.weibo.api.motan.rpc.Request)

```java
public Response request(Request request) throws TransportException {
    if (!isAvailable()) {
        throw new MotanServiceException("NettyChannel is unavailable: url=" + url.getUri() + MotanFrameworkUtil.toString(request));
    }
    boolean isAsync = false;
  	//获取异步属性
    Object async = RpcContext.getContext().getAttribute(MotanConstants.ASYNC_SUFFIX);
    if (async != null && async instanceof Boolean) {
        isAsync = (Boolean) async;
    }
    return request(request, isAsync);
}
```

com.weibo.api.motan.transport.netty4.NettyClient#request(com.weibo.api.motan.rpc.Request, boolean)

```java
private Response request(Request request, boolean async) throws TransportException {
    Channel channel;
    Response response;
    try {
        // 挑选发送请求的channel
        channel = getChannel();
        MotanFrameworkUtil.logEvent(request, MotanConstants.TRACE_CONNECTION);
        if (channel == null) {
            LoggerUtil.error("NettyClient borrowObject null: url=" + url.getUri() + " " + MotanFrameworkUtil.toString(request));
            return null;
        }
        // 异步发送请求，返回DefaultResponseFuture
        response = channel.request(request);
    } catch (Exception e) {
        LoggerUtil.error("NettyClient request Error: url=" + url.getUri() + " " + MotanFrameworkUtil.toString(request) + ", " + e.getMessage());

        if (e instanceof MotanAbstractException) {
            throw (MotanAbstractException) e;
        } else {
            throw new MotanServiceException("NettyClient request Error: url=" + url.getUri() + " " + MotanFrameworkUtil.toString(request), e);
        }
    }
    // 异步或者同步返回的响应
    response = asyncResponse(response, async);
    return response;
}
```

```java
private Response asyncResponse(Response response, boolean async) {
    if (async || !(response instanceof ResponseFuture)) { //异步直接返回
        return response;
    }
    //传入的response类型为DfaultResponseFuture，在构造函数中调用getValue进行阻塞
    return new DefaultResponse(response);
}
```

## 健康检测

```
HeartbeatClientEndpointManager
维护消费端服务的健康检测
```

### 维护每个服务的连接

com.weibo.api.motan.transport.support.HeartbeatClientEndpointManager#addEndpoint

```java
public void addEndpoint(Endpoint endpoint) { //从注册中心拉取服务之后会调用此方法
    if (!(endpoint instanceof Client)) {
        throw new MotanFrameworkException("HeartbeatClientEndpointManager addEndpoint Error: class not support " + endpoint.getClass());
    }

    Client client = (Client) endpoint;

    URL url = endpoint.getUrl();

    String heartbeatFactoryName = url.getParameter(URLParamType.heartbeatFactory.getName(), URLParamType.heartbeatFactory.getValue());

    HeartbeatFactory heartbeatFactory = ExtensionLoader.getExtensionLoader(HeartbeatFactory.class).getExtension(heartbeatFactoryName);

    if (heartbeatFactory == null) {
        throw new MotanFrameworkException("HeartbeatFactory not exist: " + heartbeatFactoryName);
    }

    endpoints.put(client, heartbeatFactory);
}
```

### 启动健康检测

com.weibo.api.motan.transport.support.HeartbeatClientEndpointManager#init

```java
public void init() { 
    executorService = Executors.newScheduledThreadPool(1);
    executorService.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {

            for (Map.Entry<Client, HeartbeatFactory> entry : endpoints.entrySet()) {
                Client endpoint = entry.getKey();

                try {
                    // 如果节点是存活状态，那么没必要走心跳
                    if (endpoint.isAvailable()) {
                        continue;
                    }

                    HeartbeatFactory factory = entry.getValue();
                    endpoint.heartbeat(factory.createRequest()); //发送心跳请求
                } catch (Exception e) {
                    LoggerUtil.error("HeartbeatEndpointManager send heartbeat Error: url=" + endpoint.getUrl().getUri() + ", " + e.getMessage());
                }
            }

        }
    }, MotanConstants.HEARTBEAT_PERIOD, MotanConstants.HEARTBEAT_PERIOD, TimeUnit.MILLISECONDS); //默认500ms
    ShutDownHook.registerShutdownHook(new Closable() {
        @Override
        public void close() {
            if (!executorService.isShutdown()) {
                executorService.shutdown();
            }
        }
    });
}
```

### 发送心跳

com.weibo.api.motan.transport.netty4.NettyClient#heartbeat

```java
public void heartbeat(Request request) {
    // 如果节点还没有初始化或者节点已经被close掉了，那么heartbeat也不需要进行了
    if (state.isUnInitState() || state.isCloseState()) {
        LoggerUtil.warn("NettyClient heartbeat Error: state={} url={}", state.name(), url.getUri());
        return;
    }

    LoggerUtil.info("NettyClient heartbeat request: url={}", url.getUri());

    try {
        // async request后，如果service is
        // available，那么将会自动把该client设置成可用
        request(request, true);
    } catch (Exception e) {
        LoggerUtil.error("NettyClient heartbeat Error: url={}, {}", url.getUri(), e.getMessage());
    }
}
```



```java
private Response request(Request request, boolean async) throws TransportException {
    Channel channel;
    Response response;
    try {
        // 挑选发送请求的channel
        channel = getChannel();
        MotanFrameworkUtil.logEvent(request, MotanConstants.TRACE_CONNECTION);
        if (channel == null) {
            LoggerUtil.error("NettyClient borrowObject null: url=" + url.getUri() + " " + MotanFrameworkUtil.toString(request));
            return null;
        }
        // 异步发送请求，返回DefaultResponseFuture
        response = channel.request(request);
    } catch (Exception e) {
        LoggerUtil.error("NettyClient request Error: url=" + url.getUri() + " " + MotanFrameworkUtil.toString(request) + ", " + e.getMessage());

        if (e instanceof MotanAbstractException) {
            throw (MotanAbstractException) e;
        } else {
            throw new MotanServiceException("NettyClient request Error: url=" + url.getUri() + " " + MotanFrameworkUtil.toString(request), e);
        }
    }

    // 异步或者同步返回的响应
    response = asyncResponse(response, async);

    return response;
}
```

com.weibo.api.motan.transport.netty4.NettyChannel#request

```java
public Response request(Request request) throws TransportException {
    int timeout = nettyClient.getUrl().getMethodParameter(request.getMethodName(), request.getParamtersDesc(), URLParamType.requestTimeout.getName(), URLParamType.requestTimeout.getIntValue());
    if (timeout <= 0) {
        throw new MotanFrameworkException("NettyClient init Error: timeout(" + timeout + ") <= 0 is forbid.", MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    }
    ResponseFuture response = new DefaultResponseFuture(request, timeout, this.nettyClient.getUrl());
  //维护requestId -> responseFuture的映射关系
    this.nettyClient.registerCallback(request.getRequestId(), response);
    byte[] msg = CodecUtil.encodeObjectToBytes(this, codec, request);
    ChannelFuture writeFuture = this.channel.writeAndFlush(msg);
    boolean result = writeFuture.awaitUninterruptibly(timeout, TimeUnit.MILLISECONDS);

    if (result && writeFuture.isSuccess()) { //请求发送成功
        MotanFrameworkUtil.logEvent(request, MotanConstants.TRACE_CSEND, System.currentTimeMillis());
      //注册监听器
        response.addListener(new FutureListener() {
            @Override
            public void operationComplete(Future future) throws Exception {
                if (future.isSuccess() || (future.isDone() && ExceptionUtil.isBizException(future.getException()))) {
                    // 成功的调用
                    nettyClient.resetErrorCount();
                } else {
                    // 失败的调用
                    nettyClient.incrErrorCount();
                }
            }
        });
        return response;
    }
		//发送异常
    writeFuture.cancel(true);
  //移除请求对应的DefaultResponseFuture
    response = this.nettyClient.removeCallback(request.getRequestId());
    if (response != null) {
        response.cancel();
    }
    // 失败的调用
    nettyClient.incrErrorCount();

    if (writeFuture.cause() != null) {
        throw new MotanServiceException("NettyChannel send request to server Error: url="
                + nettyClient.getUrl().getUri() + " local=" + localAddress + " "
                + MotanFrameworkUtil.toString(request), writeFuture.cause());
    } else {
        throw new MotanServiceException("NettyChannel send request to server Timeout: url="
                + nettyClient.getUrl().getUri() + " local=" + localAddress + " "
                + MotanFrameworkUtil.toString(request), false);
    }
}
```

com.weibo.api.motan.transport.netty4.NettyClient#registerCallback

```java
public void registerCallback(long requestId, ResponseFuture responseFuture) {
    //默认20000.缓存未响应的请求，超过上限抛出异常，避免内存溢出
    if (this.callbackMap.size() >= MotanConstants.NETTY_CLIENT_MAX_REQUEST) { 
        // reject request, prevent from OutOfMemoryError
        throw new MotanServiceException("NettyClient over of max concurrent request, drop request, url: "
                + url.getUri() + " requestId=" + requestId, MotanErrorMsgConstant.SERVICE_REJECT, false);
    }

    this.callbackMap.put(requestId, responseFuture);
}
```

com.weibo.api.motan.transport.netty4.NettyClient#resetErrorCount

```java
void resetErrorCount() {
    errorCount.set(0); //失败次数清零
    if (state.isAliveState()) {
        return;
    }
    synchronized (this) {
        if (state.isAliveState()) {
            return;
        }
        // 如果节点是unalive才进行设置，而如果是 close 或者 uninit，那么直接忽略
        if (state.isUnAliveState()) {
            long count = errorCount.longValue();
            // 过程中有其他并发更新errorCount的，因此这里需要进行一次判断
            if (count < fusingThreshold) {
                state = ChannelState.ALIVE;
                LoggerUtil.info("NettyClient recover available: url=" + url.getIdentity() + " "                        + url.getServerPortStr());
            }
        }
    }
}
```

com.weibo.api.motan.transport.netty4.NettyClient#incrErrorCount

```go
void incrErrorCount() { //增加调用失败的次数
    long count = errorCount.incrementAndGet();

    // 如果节点是可用状态，同时当前连续失败的次数超过熔断阈值，那么把该节点标示为不可用
    if (count >= fusingThreshold && state.isAliveState()) {
        synchronized (this) {
            count = errorCount.longValue();

            if (count >= fusingThreshold && state.isAliveState()) {
                LoggerUtil.error("NettyClient unavailable Error: url=" + url.getIdentity() + " "  + url.getServerPortStr());
                state = ChannelState.UNALIVE; //标示为不可用
            }
        }
    }
}
```

## 清除超时请求

com.weibo.api.motan.transport.netty4.NettyClient.TimeoutMonitor#run

```java
public void run() {
    long currentTime = System.currentTimeMillis();
    for (Map.Entry<Long, ResponseFuture> entry : callbackMap.entrySet()) {
        try {
            ResponseFuture future = entry.getValue();

            if (future.getCreateTime() + future.getTimeout() < currentTime) {
                removeCallback(entry.getKey());//移除map
                future.cancel(); //唤醒请求
            }
        } catch (Exception e) {
            LoggerUtil.error(name + " clear timeout future Error: uri=" + url.getUri() + " requestId=" + entry.getKey(), e);
        }
    }
}
```

## 异步调用

```
在声明的service上增加@MotanAsync注解
在项目pom.xml中增加build-helper-maven-plugin，用来把自动生成类的目录设置为source path
```

com.weibo.api.motan.transport.async.MotanAsyncProcessor#process

```go
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    if (roundEnv.processingOver()) {
        return true;
    }
  //为标有MotanAsync注解的接口生成新的接口，新的接口名称以及方法都已Async结尾
    for (Element elem : roundEnv.getElementsAnnotatedWith(MotanAsync.class)) {
        processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "MotanAsyncProcessor will process " + elem.toString() + ", generate class path:" + TARGET_DIR);
        try {
            writeAsyncClass(elem);
            processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "MotanAsyncProcessor done for " + elem.toString());
        } catch (Exception e) {
            processingEnv.getMessager().printMessage(Diagnostic.Kind.WARNING,
                    "MotanAsyncProcessor process " + elem.toString() + " fail. exception:" + e.getMessage());
            e.printStackTrace();
        }
    }
    return true;
}
```

# 总结

1、消费端支持异步调用

2、对JDK原生的线程池进行改造，发送的请求尽早的由更多的线程处理，当线程达到最大时，再放到队列中，降低延迟

