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
    //装饰协议，添加过滤器，增强orgProtocol的功能
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

# 总结

1、消费端支持异步调用

2、对JDK原生的线程池进行改造，发送的请求尽早的由更多的线程处理，当线程达到最大时，再放到队列中，降低延迟
