## QuickStartServer

com.alipay.sofa.rpc.quickstart.QuickStartServer

### ServerConfig 

```java
ServerConfig serverConfig = new ServerConfig()
    .setProtocol("bolt") // 设置一个协议，默认bolt
    .setPort(12200) // 设置一个端口，默认12200
    .setDaemon(false); // 非守护线程
```

### ProviderConfig

```java
ProviderConfig<HelloService> providerConfig = new ProviderConfig<HelloService>()
    .setInterfaceId(HelloService.class.getName()) // 指定接口
    .setRef(new HelloServiceImpl()) // 指定实现
    .setServer(serverConfig); // 指定服务端
```

### Export

com.alipay.sofa.rpc.config.ProviderConfig#export

```java
public synchronized void export() { //发布服务
    if (providerBootstrap == null) {
      	//默认DefaultProviderBootstrap
        providerBootstrap = Bootstraps.from(this);
    }
    providerBootstrap.export();
}
```

#### 创建Bootstraps

com.alipay.sofa.rpc.bootstrap.Bootstraps#from(com.alipay.sofa.rpc.config.ProviderConfig<T>)

```java
public static <T> ProviderBootstrap<T> from(ProviderConfig<T> providerConfig) {
    ProviderBootstrap bootstrap = ExtensionLoaderFactory.getExtensionLoader(ProviderBootstrap.class)
        .getExtension(providerConfig.getBootstrap(),
            new Class[] { ProviderConfig.class },
            new Object[] { providerConfig });
    return (ProviderBootstrap<T>) bootstrap;
}
```

#### 发布服务

```java
public void export() {
    if (providerConfig.getDelay() > 0) { // 延迟加载,单位毫秒
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(providerConfig.getDelay());
                } catch (Throwable ignore) { // NOPMD
                }
                doExport();
            }
        });
        thread.setDaemon(true);
        thread.setName("DelayExportThread");
        thread.start();
    } else {
        doExport(); //立即发布
    }
}
```

```java
private void doExport() {
    if (exported) {//避免重复发布
        return;
    }
  	//interfaceId + ":" + uniqueId;
    String key = providerConfig.buildKey();
    String appName = providerConfig.getAppName();
    // 检查参数
    checkParameters();
    if (LOGGER.isInfoEnabled(appName)) {
        LOGGER.infoWithApp(appName, "Export provider config : {} with bean id {}", key, providerConfig.getId());
    }

    // 注意同一interface，同一uniqleId，不同server情况
    AtomicInteger cnt = EXPORTED_KEYS.get(key); // 计数器
    if (cnt == null) { // 没有发布过
        cnt = CommonUtils.putToConcurrentMap(EXPORTED_KEYS, key, new AtomicInteger(0));
    }
    int c = cnt.incrementAndGet();
    int maxProxyCount = providerConfig.getRepeatedExportLimit();
    if (maxProxyCount > 0) {
        if (c > maxProxyCount) {
            cnt.decrementAndGet();
            // 超过最大数量，直接抛出异常
            throw new SofaRpcRuntimeException("Duplicate provider config with key " + key
                + " has been exported more than " + maxProxyCount + " times!"
                + " Maybe it's wrong config, please check it."
                + " Ignore this if you did that on purpose!");
        } else if (c > 1) {
            if (LOGGER.isInfoEnabled(appName)) {
                LOGGER.infoWithApp(appName, "Duplicate provider config with key {} has been exported!"
                    + " Maybe it's wrong config, please check it."
                    + " Ignore this if you did that on purpose!", key);
            }
        }
    }

    try {
        // 构造请求调用器，处理消费方的请求
        providerProxyInvoker = new ProviderProxyInvoker(providerConfig);
        // 1.初始化注册中心
        if (providerConfig.isRegister()) {
            List<RegistryConfig> registryConfigs = providerConfig.getRegistry();
            if (CommonUtils.isNotEmpty(registryConfigs)) {
                for (RegistryConfig registryConfig : registryConfigs) {
                    RegistryFactory.getRegistry(registryConfig); // 提前初始化Registry
                }
            }
        }
        // 2.将处理器注册到server,同一个服务可能保留不同的协议
        List<ServerConfig> serverConfigs = providerConfig.getServer();
        for (ServerConfig serverConfig : serverConfigs) {
            try {
              	//创建Server
                Server server = serverConfig.buildIfAbsent();
                // 注册服务信息、请求处理器
                server.registerProcessor(providerConfig, providerProxyInvoker);
                if (serverConfig.isAutoStart()) { //自启动
                    server.start();
                }
            } catch (SofaRpcRuntimeException e) {
                throw e;
            } catch (Exception e) {
                LOGGER.errorWithApp(appName, "Catch exception when register processor to server: "
                    + serverConfig.getId(), e);
            }
        }	
	
        // 3.注册到注册中心
        providerConfig.setConfigListener(new ProviderAttributeListener());
        register();
    } catch (Exception e) {
        cnt.decrementAndGet();
        if (e instanceof SofaRpcRuntimeException) {
            throw (SofaRpcRuntimeException) e;
        } else {
            throw new SofaRpcRuntimeException("Build provider proxy error!", e);
        }
    }

    // 记录一些缓存数据
    RpcRuntimeContext.cacheProviderConfig(this);
    exported = true;
}
```

com.alipay.sofa.rpc.config.ServerConfig#buildIfAbsent

```java
public synchronized Server buildIfAbsent() {
    if (server != null) {
        return server;
    }
    // 提前检查协议+序列化方式
    // ConfigValueHelper.check(ProtocolType.valueOf(getProtocol()),
    //                SerializationType.valueOf(getSerialization()));

    server = ServerFactory.getServer(this);
    return server;
}
```

com.alipay.sofa.rpc.server.ServerFactory#getServer

```java
public synchronized static Server getServer(ServerConfig serverConfig) {
    try {
        Server server = SERVER_MAP.get(Integer.toString(serverConfig.getPort()));
        if (server == null) {
            // 算下网卡和端口
            resolveServerConfig(serverConfig);

            ExtensionClass<Server> ext = ExtensionLoaderFactory.getExtensionLoader(Server.class)
                .getExtensionClass(serverConfig.getProtocol());
            if (ext == null) {
                throw ExceptionUtils.buildRuntime("server.protocol", serverConfig.getProtocol(),
                    "Unsupported protocol of server!");
            }
            server = ext.getExtInstance();
          	//初始化Server
            server.init(serverConfig);
            SERVER_MAP.put(serverConfig.getPort() + "", server);
        }
        return server;
    } catch (SofaRpcRuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new SofaRpcRuntimeException(e.getMessage(), e);
    }
}
```

### ServerInit

com.alipay.sofa.rpc.server.bolt.BoltServer#init

```java
public void init(ServerConfig serverConfig) {
    this.serverConfig = serverConfig;
    // 启动线程池
    bizThreadPool = initThreadPool(serverConfig);
    //处理消费端请求的入口类
    boltServerProcessor = new BoltServerProcessor(this);
}
```

```java
protected ThreadPoolExecutor initThreadPool(ServerConfig serverConfig) {
  	//初始化线程池	
    ThreadPoolExecutor threadPool = BusinessPool.initPool(serverConfig);
    threadPool.setThreadFactory(new NamedThreadFactory(
        "SofaBizProcessor-" + serverConfig.getPort(), serverConfig.isDaemon()));
    threadPool.setRejectedExecutionHandler(new SofaRejectedExecutionHandler());
    if (serverConfig.isPreStartCore()) { // 初始化核心线程池
        threadPool.prestartAllCoreThreads();
    }
    return threadPool;
}
```

```java
public BoltServerProcessor(BoltServer boltServer) {
    this.boltServer = boltServer;
    // 支持自定义业务线程池，扩展自Bolt框架
    this.executorSelector = new UserThreadPoolSelector(); 
}
```

com.alipay.sofa.rpc.server.bolt.BoltServerProcessor.UserThreadPoolSelector#select

```java
public Executor select(String requestClass, Object requestHeader) {
  	//线程池选择器，根据服务的不同选择不同的线程池处理请求
    if (SofaRequest.class.getName().equals(requestClass)
        && requestHeader != null) {
        Map<String, String> headerMap = (Map<String, String>) requestHeader;
        try {
            String service = headerMap.get(RemotingConstants.HEAD_SERVICE);
            if (service == null) {
                service = headerMap.get(RemotingConstants.HEAD_TARGET_SERVICE);
            }
            if (service != null) {
                UserThreadPool threadPool = UserThreadPoolManager.getUserThread(service);
                if (threadPool != null) {
                    Executor executor = threadPool.getExecutor();
                    if (executor != null) {
                        // 存在自定义线程池，且不为空
                        return executor;
                    }
                }
            }
        } catch (Exception e) {
            if (LOGGER.isWarnEnabled()) {
                LOGGER.warn(LogCodes.getLog(LogCodes.WARN_DESERIALIZE_HEADER_ERROR), e);
            }
        }
    }
    return getExecutor(); //使用默认的线程池
}
```

com.alipay.sofa.rpc.server.bolt.BoltServer#registerProcessor

```java
public void registerProcessor(ProviderConfig providerConfig, Invoker instance) {
    // 缓存Invoker对象
    String key = ConfigUniqueNameGenerator.getUniqueName(providerConfig);
  	//维护uniqueName -> Invoker,处理消费端请求时，从此map获取对应的invoker
    invokerMap.put(key, instance);
    // 缓存接口的方法
    ReflectCache.putServiceMethodCache(key, providerConfig.getProxyClass());
}
```

com.alipay.sofa.rpc.config.ConfigUniqueNameGenerator#getUniqueName

```java
public static String getUniqueName(AbstractInterfaceConfig interfaceConfig) {
    // 加上 1.0 是为了兼容之前的版本
    String uniqueId = interfaceConfig.getUniqueId();
    return interfaceConfig.getInterfaceId() + ":" + interfaceConfig.getVersion()
        + (StringUtils.isEmpty(uniqueId) ? "" : ":" + uniqueId);
}
```

### handleRequest

com.alipay.sofa.rpc.server.bolt.BoltServerProcessor#handleRequest

```java
public void handleRequest(BizContext bizCtx, AsyncContext asyncCtx, SofaRequest request) {
    // RPC内置上下文
    RpcInternalContext context = RpcInternalContext.getContext();
    context.setProviderSide(true);

    String appName = request.getTargetAppName();
    if (appName == null) {
        // 默认全局appName
        appName = (String) RpcRuntimeContext.get(RpcRuntimeContext.KEY_APPNAME);
    }

    // 是否链路异步化中
    boolean isAsyncChain = false;
    try { // 这个 try-finally 为了保证Context一定被清理
        processingCount.incrementAndGet(); // 正在处理的请求统计值加1

        context.setRemoteAddress(bizCtx.getRemoteHost(), bizCtx.getRemotePort()); // 远程地址
        context.setAttachment(RpcConstants.HIDDEN_KEY_ASYNC_CONTEXT, asyncCtx); // 远程返回的通道

        if (RpcInternalContext.isAttachmentEnable()) {
            InvokeContext boltInvokeCtx = bizCtx.getInvokeContext();
            if (boltInvokeCtx != null) {
                putToContextIfNotNull(boltInvokeCtx, InvokeContext.BOLT_PROCESS_WAIT_TIME,
                    context, RpcConstants.INTERNAL_KEY_PROCESS_WAIT_TIME); // rpc线程池等待时间 Long
            }
        }
        if (EventBus.isEnable(ServerReceiveEvent.class)) {
            EventBus.post(new ServerReceiveEvent(request));
        }

        // 开始处理
        SofaResponse response = null; // 响应，用于返回
        Throwable throwable = null; // 异常，用于记录
        ProviderConfig providerConfig = null;
        String serviceName = request.getTargetServiceUniqueName();

        try { // 这个try-catch 保证一定有Response
            invoke:
            {
                if (!boltServer.isStarted()) { // 服务端已关闭
                    throwable = new SofaRpcException(RpcErrorType.SERVER_CLOSED, LogCodes.getLog(
                        LogCodes.WARN_PROVIDER_STOPPED, SystemInfo.getLocalHost() + ":" +
                            boltServer.serverConfig.getPort()));
                    response = MessageBuilder.buildSofaErrorResponse(throwable.getMessage());
                    break invoke;
                }
                if (bizCtx.isRequestTimeout()) { // 加上丢弃超时的请求的逻辑
                    throwable = clientTimeoutWhenReceiveRequest(appName, serviceName, bizCtx.getRemoteAddress());
                    break invoke;
                }
                // 查找服务
                Invoker invoker = boltServer.findInvoker(serviceName);
                if (invoker == null) {
                    throwable = cannotFoundService(appName, serviceName);
                    response = MessageBuilder.buildSofaErrorResponse(throwable.getMessage());
                    break invoke;
                }
                if (invoker instanceof ProviderProxyInvoker) {
                    providerConfig = ((ProviderProxyInvoker) invoker).getProviderConfig();
                    context.setInterfaceConfig(providerConfig);
                    // 找到服务后，打印服务的appName
                    appName = providerConfig != null ? providerConfig.getAppName() : null;
                }
                // 查找方法
                String methodName = request.getMethodName();
                Method serviceMethod = ReflectCache.getServiceMethod(serviceName, methodName,
                    request.getMethodArgSigs());
                if (serviceMethod == null) {
                    throwable = cannotFoundServiceMethod(appName, methodName, serviceName);
                    response = MessageBuilder.buildSofaErrorResponse(throwable.getMessage());
                    break invoke;
                } else {
                    request.setMethod(serviceMethod);
                }

                // 真正调用
                response = doInvoke(serviceName, invoker, request);

                if (bizCtx.isRequestTimeout()) { // 加上丢弃超时的响应的逻辑
                    throwable = clientTimeoutWhenSendResponse(appName, serviceName, bizCtx.getRemoteAddress());
                    break invoke;
                }
            }
        } catch (Exception e) {
            // 服务端异常，不管是啥异常
            LOGGER.errorWithApp(appName, "Server Processor Error!", e);
            throwable = e;
            response = MessageBuilder.buildSofaErrorResponse(e.getMessage());
        }

        // Response不为空，代表需要返回给客户端
        if (response != null) {
            RpcInvokeContext invokeContext = RpcInvokeContext.peekContext();
            isAsyncChain = CommonUtils.isTrue(invokeContext != null ?
                (Boolean) invokeContext.remove(RemotingConstants.INVOKE_CTX_IS_ASYNC_CHAIN) : null);
            // 如果是服务端异步代理模式，特殊处理，因为该模式是在业务代码自主异步返回的
            if (!isAsyncChain) {
                // 其它正常请求
                try { // 这个try-catch 保证一定要记录tracer
                    asyncCtx.sendResponse(response);
                } finally {
                    if (EventBus.isEnable(ServerSendEvent.class)) {
                        EventBus.post(new ServerSendEvent(request, response, throwable));
                    }
                }
            }
        }
    } catch (Throwable e) {
        // 可能有返回时的异常
        if (LOGGER.isErrorEnabled(appName)) {
            LOGGER.errorWithApp(appName, e.getMessage(), e);
        }
    } finally {
        processingCount.decrementAndGet();
        if (!isAsyncChain) {
            if (EventBus.isEnable(ServerEndHandleEvent.class)) {
                EventBus.post(new ServerEndHandleEvent());
            }
        }
        RpcInvokeContext.removeContext();
        RpcInternalContext.removeAllContext();
    }
}
```

```java
private SofaResponse doInvoke(String serviceName, Invoker invoker, SofaRequest request) throws SofaRpcException {
    // 开始调用，先记下当前的ClassLoader
    ClassLoader rpcCl = Thread.currentThread().getContextClassLoader();
    try {
        // 切换线程的ClassLoader到 服务 自己的ClassLoader
        ClassLoader serviceCl = ReflectCache.getServiceClassLoader(serviceName);
        Thread.currentThread().setContextClassLoader(serviceCl);
        return invoker.invoke(request);
    } finally {
        Thread.currentThread().setContextClassLoader(rpcCl);
    }
}
```

com.alipay.sofa.rpc.filter.ProviderInvoker#invoke

```java
public SofaResponse invoke(SofaRequest request) throws SofaRpcException {

    /*// 将接口的<sofa:param />的配置复制到RpcInternalContext TODO
    RpcInternalContext context = RpcInternalContext.getContext();
    Map params = providerConfig.getParameters();
    if (params != null) {
       context.setAttachments(params);
    }
    // 将方法的<sofa:param />的配置复制到invocation
    String methodName = request.getMethodName();
    params = (Map) providerConfig.getMethodConfigValue(methodName, SofaConstants.CONFIG_KEY_PARAMS);
    if (params != null) {
       context.setAttachments(params);
    }*/

    SofaResponse sofaResponse = new SofaResponse();
    long startTime = RpcRuntimeContext.now();
    try {
        // 反射 真正调用业务代码
        Method method = request.getMethod();
        if (method == null) {
            throw new SofaRpcException(RpcErrorType.SERVER_FILTER, "Need decode method first!");
        }
        Object result = method.invoke(providerConfig.getRef(), request.getMethodArgs());

        sofaResponse.setAppResponse(result);
    } catch (IllegalArgumentException e) { // 非法参数，可能是实现类和接口类不对应) 
        sofaResponse.setErrorMsg(e.getMessage());
    } catch (IllegalAccessException e) { // 如果此 Method 对象强制执行 Java 语言访问控制，并且底层方法是不可访问的
        sofaResponse.setErrorMsg(e.getMessage());
        //        } catch (NoSuchMethodException e) { // 如果找不到匹配的方法
        //            sofaResponse.setErrorMsg(e.getMessage());
        //        } catch (ClassNotFoundException e) { // 如果指定的类加载器无法定位该类
        //            sofaResponse.setErrorMsg(e.getMessage());
    } catch (InvocationTargetException e) { // 业务代码抛出异常
        sofaResponse.setAppResponse(e.getCause());
    } finally {
        if (RpcInternalContext.isAttachmentEnable()) {
            long endTime = RpcRuntimeContext.now();
            RpcInternalContext.getContext().setAttachment(RpcConstants.INTERNAL_KEY_IMPL_ELAPSE,
                endTime - startTime);
        }
    }

    return sofaResponse;
}
```

## QuickStartClient

### ConsumerConfig

com.alipay.sofa.rpc.quickstart.QuickStartClient

```java
ConsumerConfig<HelloService> consumerConfig = new ConsumerConfig<HelloService>()
    .setInterfaceId(HelloService.class.getName()) // 指定接口
    .setProtocol("bolt") // 指定协议
    .setDirectUrl("bolt://127.0.0.1:12200") // 指定直连地址
    .setConnectTimeout(10 * 1000);
```

### Refer

```java
HelloService helloService = consumerConfig.refer();
```

com.alipay.sofa.rpc.config.ConsumerConfig#refer

```java
public T refer() {//引用服务
    if (consumerBootstrap == null) {
      	//创建DefaultConsumerBootstrap
        consumerBootstrap = Bootstraps.from(this);
    }
    return consumerBootstrap.refer();
}
```

com.alipay.sofa.rpc.bootstrap.DefaultConsumerBootstrap#refer

```java
public T refer() {
    if (proxyIns != null) { //已经创建了代理实现类
        return proxyIns;
    }
    synchronized (this) {
        if (proxyIns != null) { //双重检测
            return proxyIns;
        }
        String key = consumerConfig.buildKey();
        String appName = consumerConfig.getAppName();
        // 检查参数
        checkParameters();
        // 提前检查接口类
        if (LOGGER.isInfoEnabled(appName)) {
            LOGGER.infoWithApp(appName, "Refer consumer config : {} with bean id {}", key, consumerConfig.getId());
        }

        // 注意同一interface，同一tags，同一protocol情况
        AtomicInteger cnt = REFERRED_KEYS.get(key); // 计数器
        if (cnt == null) { // 没有发布过
            cnt = CommonUtils.putToConcurrentMap(REFERRED_KEYS, key, new AtomicInteger(0));
        }
        int c = cnt.incrementAndGet();
        int maxProxyCount = consumerConfig.getRepeatedReferLimit();
        if (maxProxyCount > 0) {
            if (c > maxProxyCount) {
                cnt.decrementAndGet();
                // 超过最大数量，直接抛出异常
                throw new SofaRpcRuntimeException("Duplicate consumer config with key " + key
                    + " has been referred more than " + maxProxyCount + " times!"
                    + " Maybe it's wrong config, please check it."
                    + " Ignore this if you did that on purpose!");
            } else if (c > 1) {
                if (LOGGER.isInfoEnabled(appName)) {
                    LOGGER.infoWithApp(appName, "Duplicate consumer config with key {} has been referred!"
                        + " Maybe it's wrong config, please check it."
                        + " Ignore this if you did that on purpose!", key);
                }
            }
        }

        try {
            // 创建Cluster
            cluster = ClusterFactory.getCluster(this);
            // 设置Listener，监听消费者配置变更
            consumerConfig.setConfigListener(buildConfigListener(this));
            consumerConfig.setProviderInfoListener(buildProviderInfoListener(this));
            // 初始化 cluster
            cluster.init();
            // 构造Invoker对象（执行链）	，发送远程请求
            proxyInvoker = buildClientProxyInvoker(this);
            // 创建代理类
            proxyIns = (T) ProxyFactory.buildProxy(consumerConfig.getProxy(), consumerConfig.getProxyClass(),
                proxyInvoker);
        } catch (Exception e) {
            if (cluster != null) {
                cluster.destroy();
                cluster = null;
            }
            consumerConfig.setConfigListener(null);
            consumerConfig.setProviderInfoListener(null);
            cnt.decrementAndGet(); // 发布失败不计数
            if (e instanceof SofaRpcRuntimeException) {
                throw (SofaRpcRuntimeException) e;
            } else {
                throw new SofaRpcRuntimeException("Build consumer proxy error!", e);
            }
        }
        if (consumerConfig.getOnAvailable() != null && cluster != null) {
            cluster.checkStateChange(false); // 状态变化通知监听器
        }
        RpcRuntimeContext.cacheConsumerConfig(this);
        return proxyIns;
    }
}
```

初始化集群

com.alipay.sofa.rpc.client.AbstractCluster#init

```java
public synchronized void init() {
    if (initialized) { // 已初始化
        return;
    }
    // 构造Router链，过滤出可以访问哪些服务提供方
    routerChain = RouterChain.buildConsumerChain(consumerBootstrap);
    // 负载均衡策略 ，选择一个服务提供方，发送请求
    loadBalancer = LoadBalancerFactory.getLoadBalancer(consumerBootstrap);
    // 地址管理器
    addressHolder = AddressHolderFactory.getAddressHolder(consumerBootstrap);
    // 连接管理器
    connectionHolder = ConnectionHolderFactory.getConnectionHolder(consumerBootstrap);
    // 构造Filter链,最底层是调用过滤器
    this.filter	Chain = FilterChain.buildConsumerChain(this.consumerConfig,
        new ConsumerInvoker(consumerBootstrap));

    if (consumerConfig.isLazy()) { // 延迟连接
        if (LOGGER.isInfoEnabled(consumerConfig.getAppName())) {
            LOGGER.infoWithApp(consumerConfig.getAppName(), "Connection will be initialized when first invoke.");
        }
    }

    // 启动重连线程
    connectionHolder.init();
    try {
        // 得到服务端列表
        List<ProviderGroup> all = consumerBootstrap.subscribe();
        if (CommonUtils.isNotEmpty(all)) {
            // 初始化服务端连接（建立长连接)
            updateAllProviders(all);
        }
    } catch (SofaRpcRuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new SofaRpcRuntimeException("Init provider's transport error!", e);
    }

    // 启动成功
    initialized = true;

    // 如果check=true表示强依赖
    if (consumerConfig.isCheck() && !isAvailable()) {
        throw new SofaRpcRuntimeException("The consumer is depend on alive provider " +
            "and there is no alive provider, you can ignore it " +
            "by ConsumerConfig.setCheck(boolean) (default is false)");
    }
}
```

com.alipay.sofa.rpc.bootstrap.DefaultConsumerBootstrap#buildClientProxyInvoker

```java
protected ClientProxyInvoker buildClientProxyInvoker(ConsumerBootstrap bootstrap) {
    return new DefaultClientProxyInvoker(bootstrap);
}
```

### remoteCall

#### JDKInvocationHandler

com.alipay.sofa.rpc.proxy.jdk.JDKInvocationHandler#invoke

```java
public Object invoke(Object proxy, Method method, Object[] paramValues)
    throws Throwable {
    String methodName = method.getName();
    Class[] paramTypes = method.getParameterTypes();
    if ("toString".equals(methodName) && paramTypes.length == 0) {
        return proxyInvoker.toString();
    } else if ("hashCode".equals(methodName) && paramTypes.length == 0) {
        return proxyInvoker.hashCode();
    } else if ("equals".equals(methodName) && paramTypes.length == 1) {
        Object another = paramValues[0];
        return proxy == another ||
            (proxy.getClass().isInstance(another) && proxyInvoker.equals(JDKProxy.parseInvoker(another)));
    }
  	//创建请求
    SofaRequest sofaRequest = MessageBuilder.buildSofaRequest(method.getDeclaringClass(),
        method, paramTypes, paramValues);
  	//DefaultClientProxyInvoker
    SofaResponse response = proxyInvoker.invoke(sofaRequest);
    if (response.isError()) {
        throw new SofaRpcException(RpcErrorType.SERVER_UNDECLARED_ERROR, response.getErrorMsg());
    }
    Object ret = response.getAppResponse();
    if (ret instanceof Throwable) {
        throw (Throwable) ret;
    } else {
        if (ret == null) {
            return ClassUtils.getDefaultArg(method.getReturnType());
        }
        return ret;
    }
}
```

com.alipay.sofa.rpc.message.MessageBuilder#buildSofaRequest(java.lang.Class<?>, java.lang.reflect.Method, java.lang.Class[], java.lang.Object[])

```java
public static SofaRequest buildSofaRequest(Class<?> clazz, Method method, Class[] argTypes, Object[] args) {
    SofaRequest request = new SofaRequest();
    request.setInterfaceName(clazz.getName());
    request.setMethodName(method.getName());
    request.setMethod(method);
    request.setMethodArgs(args == null ? new Object[0] : args);
    request.setMethodArgSigs(ClassTypeUtils.getTypeStrs(argTypes, true));
    return request;
}
```

#### ClientProxyInvoker

com.alipay.sofa.rpc.client.ClientProxyInvoker#invoke

```java
public SofaResponse invoke(SofaRequest request) throws SofaRpcException {
    SofaResponse response = null;
    Throwable throwable = null;
    try {
        RpcInternalContext.pushContext();
        RpcInternalContext context = RpcInternalContext.getContext();
        context.setProviderSide(false); //消费端
        // 包装请求
        decorateRequest(request);
        try {
            // 产生开始调用事件
            if (EventBus.isEnable(ClientStartInvokeEvent.class)) {
                EventBus.post(new ClientStartInvokeEvent(request));
            }
            // 得到结果
            response = cluster.invoke(request);
        } catch (SofaRpcException e) {
            throwable = e;
            throw e;
        } finally {
            // 产生调用结束事件
            if (EventBus.isEnable(ClientEndInvokeEvent.class)) {
                EventBus.post(new ClientEndInvokeEvent(request, response, throwable));
            }
        }
        // 包装响应
        decorateResponse(response);
        return response;
    } finally {
        RpcInternalContext.removeContext();
        RpcInternalContext.popContext();
    }
}
```

##### decorateRequest

com.alipay.sofa.rpc.bootstrap.DefaultClientProxyInvoker#decorateRequest

```java
protected void decorateRequest(SofaRequest request) {//包装请求
    // 公共的设置
    super.decorateRequest(request);

    // 缓存是为了加快速度
    request.setTargetServiceUniqueName(serviceName);
    request.setSerializeType(serializeType);

    if (!consumerConfig.isGeneric()) {
        // 找到调用类型， generic的时候类型在filter里进行判断
        request.setInvokeType(consumerConfig.getMethodInvokeType(request.getMethodName()));
    }

    RpcInvokeContext invokeCtx = RpcInvokeContext.peekContext();
    RpcInternalContext internalContext = RpcInternalContext.getContext();
    if (invokeCtx != null) {
        // 如果用户设置了调用级别回调函数
        SofaResponseCallback responseCallback = invokeCtx.getResponseCallback();
        if (responseCallback != null) {
            request.setSofaResponseCallback(responseCallback);
            invokeCtx.setResponsxueCallback(null); // 一次性用完
            invokeCtx.put(RemotingConstants.INVOKE_CTX_IS_ASYNC_CHAIN,
                isSendableResponseCallback(responseCallback));
        }
        // 如果用户设置了调用级别超时时间
        Integer timeout = invokeCtx.getTimeout();
        if (timeout != null) {
            request.setTimeout(timeout);
            invokeCtx.setTimeout(null);// 一次性用完
        }
        // 如果用户指定了调用的URL
        String targetURL = invokeCtx.getTargetURL();
        if (targetURL != null) {
            internalContext.setAttachment(HIDDEN_KEY_PINPOINT, targetURL);
            invokeCtx.setTargetURL(null);// 一次性用完
        }
        // 如果用户指定了透传数据
        if (RpcInvokeContext.isBaggageEnable()) {
            // 需要透传
            BaggageResolver.carryWithRequest(invokeCtx, request);
            internalContext.setAttachment(HIDDEN_KEY_INVOKE_CONTEXT, invokeCtx);
        }
    }
    if (RpcInternalContext.isAttachmentEnable()) {
        internalContext.setAttachment(INTERNAL_KEY_APP_NAME, consumerConfig.getAppName());
        internalContext.setAttachment(INTERNAL_KEY_PROTOCOL_NAME, consumerConfig.getProtocol());
    }

    // 额外属性通过HEAD传递给服务端
    request.addRequestProp(RemotingConstants.HEAD_APP_NAME, consumerConfig.getAppName());
    request.addRequestProp(RemotingConstants.HEAD_PROTOCOL, consumerConfig.getProtocol());
}
```

##### invoke

com.alipay.sofa.rpc.client.AbstractCluster#invoke

```java
public SofaResponse invoke(SofaRequest request) throws SofaRpcException {
    //开始发起远程调用
    SofaResponse response = null;
    try {
        // 做一些初始化检查，例如未连接可以连接
        checkClusterState();
        // 开始调用
        countOfInvoke.incrementAndGet(); // 计数+1         
        response = doInvoke(request);
        return response;
    } catch (SofaRpcException e) {
        // 客户端收到异常（客户端自己的异常）
        throw e;
    } finally {
        countOfInvoke.decrementAndGet(); // 计数-1
    }
}
```

FailFastCluster

com.alipay.sofa.rpc.client.FailFastCluster#doInvoke

```java
public SofaResponse doInvoke(SofaRequest request) throws SofaRpcException {
  	//快速失败
  	//负载均衡，选择一个服务方
    ProviderInfo providerInfo = select(request);
    try {
      	//调用链
        SofaResponse response = filterChain(providerInfo, request);
        if (response != null) {
            return response;
        } else {
            throw new SofaRpcException(RpcErrorType.CLIENT_UNDECLARED_ERROR,
                "Failed to call " + request.getInterfaceName() + "." + request.getMethodName()
                    + " on remote server " + providerInfo + ", return null");
        }
    } catch (Exception e) {
        throw new SofaRpcException(RpcErrorType.CLIENT_UNDECLARED_ERROR,
            "Failed to call " + request.getInterfaceName() + "." + request.getMethodName()
                + " on remote server: " + providerInfo + ", cause by: "
                + e.getClass().getName() + ", message is: " + e.getMessage(), e);
    }
}
```

FailoverCluster

com.alipay.sofa.rpc.client.FailoverCluster#doInvoke

```java
public SofaResponse doInvoke(SofaRequest request) throws SofaRpcException {		//故障转移
    String methodName = request.getMethodName();
    int retries = consumerConfig.getMethodRetries(methodName);
    int time = 0;
    SofaRpcException throwable = null;// 异常日志
    List<ProviderInfo> invokedProviderInfos = new ArrayList<ProviderInfo>(retries + 1);
    do {
      	//负载均衡，选择一个服务方
        ProviderInfo providerInfo = select(request, invokedProviderInfos);
        try {
          	//应用调用链
            SofaResponse response = filterChain(providerInfo, request);
            if (response != null) {
                if (throwable != null) {
                    if (LOGGER.isWarnEnabled(consumerConfig.getAppName())) {
                        LOGGER.warnWithApp(consumerConfig.getAppName(),
                            LogCodes.getLog(LogCodes.WARN_SUCCESS_BY_RETRY,
                                throwable.getClass() + ":" + throwable.getMessage(),
                                invokedProviderInfos));
                    }
                }
                return response;
            } else {
                throwable = new SofaRpcException(RpcErrorType.CLIENT_UNDECLARED_ERROR,
                    "Failed to call " + request.getInterfaceName() + "." + methodName
                        + " on remote server " + providerInfo + ", return null");
            }
        } catch (SofaRpcException e) { 
            //TODO 服务端繁忙、超时异常 才发起rpc异常重试，此处不方便扩展重试异常
            if (e.getErrorType() == RpcErrorType.SERVER_BUSY
                || e.getErrorType() == RpcErrorType.CLIENT_TIMEOUT) {
                throwable = e;
                time++;
            } else {
                throw e;
            }
        } catch (Exception e) { // 其它异常不重试
            throw new SofaRpcException(RpcErrorType.CLIENT_UNDECLARED_ERROR,
                "Failed to call " + request.getInterfaceName() + "." + request.getMethodName()
                    + " on remote server: " + providerInfo + ", cause by unknown exception: "
                    + e.getClass().getName() + ", message is: " + e.getMessage(), e);
        } finally {
            if (RpcInternalContext.isAttachmentEnable()) {
                RpcInternalContext.getContext().setAttachment(RpcConstants.INTERNAL_KEY_INVOKE_TIMES,
                    time + 1); // 重试次数
            }
        }
        invokedProviderInfos.add(providerInfo);
    } while (time <= retries);

    throw throwable;
}
```

com.alipay.sofa.rpc.client.AbstractCluster#filterChain

```java
protected SofaResponse filterChain(ProviderInfo providerInfo, SofaRequest request) throws SofaRpcException {
    RpcInternalContext context = RpcInternalContext.getContext();
    context.setInterfaceConfig(consumerConfig);
  	//设置此请求发送的服务提供方
    context.setProviderInfo(providerInfo);
  	//FilterChain
    return filterChain.invoke(request);
}
```

```java
public SofaResponse invoke(SofaRequest sofaRequest) throws SofaRpcException {
    return invokerChain.invoke(sofaRequest);
}
```

com.alipay.sofa.rpc.filter.ConsumerInvoker#invoke

```java
public SofaResponse invoke(SofaRequest sofaRequest) throws SofaRpcException {
    // 设置下服务器应用
    ProviderInfo providerInfo = RpcInternalContext.getContext().getProviderInfo();
    String appName = providerInfo.getStaticAttr(ProviderInfoAttrs.ATTR_APP_NAME);
    if (StringUtils.isNotEmpty(appName)) {
        sofaRequest.setTargetAppName(appName);
    }

    // 目前只是通过client发送给服务端
    return consumerBootstrap.getCluster().sendMsg(providerInfo, sofaRequest);
}
```

com.alipay.sofa.rpc.client.AbstractCluster#sendMsg

```java
public SofaResponse sendMsg(ProviderInfo providerInfo, SofaRequest request) throws SofaRpcException {
  	//获取连接
    ClientTransport clientTransport = connectionHolder.getAvailableClientTransport(providerInfo);
  	//连接可用
    if (clientTransport != null && clientTransport.isAvailable()) {
        return doSendMsg(providerInfo, clientTransport, request);
    } else {
        throw unavailableProviderException(request.getTargetServiceUniqueName(), providerInfo.getOriginUrl());
    }
}
```

com.alipay.sofa.rpc.client.AbstractCluster#doSendMsg

```java
protected SofaResponse doSendMsg(ProviderInfo providerInfo, ClientTransport transport, SofaRequest request) throws SofaRpcException {
  	//发送远程请求
    RpcInternalContext context = RpcInternalContext.getContext();
    // 添加调用的服务端远程地址
 RpcInternalContext.getContext().setRemoteAddress(providerInfo.getHost(), providerInfo.getPort());
    try {
        checkProviderVersion(providerInfo, request); // 根据服务端版本特殊处理
        String invokeType = request.getInvokeType();
      	//决定超时时间
        int timeout = resolveTimeout(request, consumerConfig, providerInfo);

        SofaResponse response = null;
        // 同步调用
        if (RpcConstants.INVOKER_TYPE_SYNC.equals(invokeType)) {
            long start = RpcRuntimeContext.now();
            try {
              	//阻塞，直到接收到服务提供方的响应
                response = transport.syncSend(request, timeout);
            } finally {
                if (RpcInternalContext.isAttachmentEnable()) {
                    long elapsed = RpcRuntimeContext.now() - start;
                    context.setAttachment(RpcConstants.INTERNAL_KEY_CLIENT_ELAPSE, elapsed);
                }
            }
        }
        // 单向调用
        else if (RpcConstants.INVOKER_TYPE_ONEWAY.equals(invokeType)) {
            long start = RpcRuntimeContext.now();
            try {
                transport.oneWaySend(request, timeout);
                response = new SofaResponse();
            } finally {
                if (RpcInternalContext.isAttachmentEnable()) {
                    long elapsed = RpcRuntimeContext.now() - start;
                    context.setAttachment(RpcConstants.INTERNAL_KEY_CLIENT_ELAPSE, elapsed);
                }
            }
        }
        // Callback调用
        else if (RpcConstants.INVOKER_TYPE_CALLBACK.equals(invokeType)) {
            // 调用级别回调监听器
            SofaResponseCallback sofaResponseCallback = request.getSofaResponseCallback();
            if (sofaResponseCallback == null) {
                SofaResponseCallback methodResponseCallback = consumerConfig
                    .getMethodOnreturn(request.getMethodName());
                if (methodResponseCallback != null) { // 方法的Callback
                    request.setSofaResponseCallback(methodResponseCallback);
                }
            }
            transport.asyncSend(request, timeout);
            response = new SofaResponse();
        }
        // Future调用
        else if (RpcConstants.INVOKER_TYPE_FUTURE.equals(invokeType)) {
            // 开始调用
            ResponseFuture future = transport.asyncSend(request, timeout);
            // 放入线程上下文
            RpcInternalContext.getContext().setFuture(future);
            response = new SofaResponse();
        } else {
            throw new SofaRpcException(RpcErrorType.CLIENT_UNDECLARED_ERROR, "Unknown invoke type:" + invokeType);
        }
        return response;
    } catch (SofaRpcException e) {
        throw e;
    } catch (Throwable e) { // 客户端其它异常
        throw new SofaRpcException(RpcErrorType.CLIENT_UNDECLARED_ERROR, e);
    }
}
```

```java
private int resolveTimeout(SofaRequest request, ConsumerConfig consumerConfig, ProviderInfo providerInfo) {//决定超时的时间
    // 先去调用级别配置
    Integer timeout = request.getTimeout();
    if (timeout == null) {
        // 取客户端配置（先方法级别再接口级别）
        timeout = consumerConfig.getMethodTimeout(request.getMethodName());
        if (timeout == null || timeout < 0) {
            // 再取服务端配置
            timeout = (Integer) providerInfo.getDynamicAttr(ATTR_TIMEOUT);
            if (timeout == null) {
                // 取框架默认值
                timeout = getIntValue(CONSUMER_INVOKE_TIMEOUT);
            }
        }
    }
    return timeout;
}
```

com.alipay.sofa.rpc.transport.bolt.BoltClientTransport#syncSend

```java
public SofaResponse syncSend(SofaRequest request, int timeout) throws SofaRpcException {
    checkConnection();
    RpcInternalContext context = RpcInternalContext.getContext();
    InvokeContext boltInvokeContext = createInvokeContext(request);
    SofaResponse response = null;
    SofaRpcException throwable = null;
    try {
        beforeSend(context, request);
        response = doInvokeSync(request, boltInvokeContext, timeout);
        return response;
    } catch (Exception e) { // 其它异常
        throwable = convertToRpcException(e);
        throw throwable;
    } finally {
        afterSend(context, boltInvokeContext, request);
        if (EventBus.isEnable(ClientSyncReceiveEvent.class)) {//开启单机故障剔除功能
            EventBus.post(new ClientSyncReceiveEvent((ConsumerConfig) context.getInterfaceConfig(),
                context.getProviderInfo(), request, response, throwable)); //发布事件
        }
    }
}
```

##### decorateResponse

com.alipay.sofa.rpc.bootstrap.DefaultClientProxyInvoker#decorateResponse

```java
protected void decorateResponse(SofaResponse response) {
    // 公共的设置
    super.decorateResponse(response);
    // 上下文内转外
    RpcInternalContext context = RpcInternalContext.getContext();
    ResponseFuture future = context.getFuture();
    RpcInvokeContext invokeCtx = null;
    if (future != null) {
        invokeCtx = RpcInvokeContext.getContext();
        invokeCtx.setFuture(future);
    }
    if (RpcInvokeContext.isBaggageEnable()) {
        BaggageResolver.pickupFromResponse(invokeCtx, response, true);
    }
    // bad code
    if (RpcInternalContext.isAttachmentEnable()) {
        String resultCode = (String) context.getAttachment(INTERNAL_KEY_RESULT_CODE);
        if (resultCode != null) {
            if (invokeCtx == null) {
                invokeCtx = RpcInvokeContext.getContext();
            }
            invokeCtx.put(RemotingConstants.INVOKE_CTX_RPC_RESULT_CODE, resultCode);
        }
    }
}
```

## 扩展

相比JDK原生 SPI，实现了更强大的功能

com.alipay.sofa.rpc.ext.ExtensionLoaderFactory#getExtensionLoader(java.lang.Class<T>)

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> clazz) {
    return getExtensionLoader(clazz, null);
}
```

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> clazz, ExtensionLoaderListener<T> listener) {
  	//Class ->	ExtensionLoader
    ExtensionLoader<T> loader = LOADER_MAP.get(clazz);
    if (loader == null) {
        synchronized (ExtensionLoaderFactory.class) {
            loader = LOADER_MAP.get(clazz);
            if (loader == null) {
                loader = new ExtensionLoader<T>(clazz, listener);
                LOADER_MAP.put(clazz, loader);
            }
        }
    }
    return loader;
}
```

com.alipay.sofa.rpc.ext.ExtensionLoader#ExtensionLoader(java.lang.Class<T>, boolean, com.alipay.sofa.rpc.ext.ExtensionLoaderListener<T>)

```java
protected ExtensionLoader(Class<T> interfaceClass, boolean autoLoad, ExtensionLoaderListener<T> listener) {
    if (RpcRunningState.isShuttingDown()) {
        this.interfaceClass = null;
        this.interfaceName = null;
        this.listener = null;
        this.factory = null;
        this.extensible = null;
        this.all = null;
        return;
    }
    // 接口为空，既不是接口，也不是抽象类
    if (interfaceClass == null ||
        !(interfaceClass.isInterface() || Modifier.isAbstract(interfaceClass.getModifiers()))) {
        throw new IllegalArgumentException("Extensible class must be interface or abstract class!");
    }
    this.interfaceClass = interfaceClass;
    this.interfaceName = ClassTypeUtils.getTypeStr(interfaceClass);
    this.listener = listener;
    Extensible extensible = interfaceClass.getAnnotation(Extensible.class);
    if (extensible == null) {
        throw new IllegalArgumentException(
            "Error when load extensible interface " + interfaceName + ", must add annotation @Extensible.");
    } else {
        this.extensible = extensible;
    }

    this.factory = extensible.singleton() ? new ConcurrentHashMap<String, T>() : null;
    this.all = new ConcurrentHashMap<String, ExtensionClass<T>>();
    if (autoLoad) {
        List<String> paths = RpcConfigs.getListValue(RpcOptions.EXTENSION_LOAD_PATH);
        for (String path : paths) {
            loadFromFile(path);
        }
    }
}
```

```java
protected synchronized void loadFromFile(String path) {
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Loading extension of extensible {} from path: {}", interfaceName, path);
    }
    // 默认如果不指定文件名字，就是接口名
    String file = StringUtils.isBlank(extensible.file()) ? interfaceName : extensible.file().trim();
    String fullFileName = path + file;
    try {
        ClassLoader classLoader = ClassLoaderUtils.getClassLoader(getClass());
        loadFromClassLoader(classLoader, fullFileName);
    } catch (Throwable t) {
        if (LOGGER.isErrorEnabled()) {
            LOGGER.error("Failed to load extension of extensible " + interfaceName + " from path:" + fullFileName,
                t);
        }
    }
}
```

```java
protected void loadFromClassLoader(ClassLoader classLoader, String fullFileName) throws Throwable { //读取文件
    Enumeration<URL> urls = classLoader != null ? classLoader.getResources(fullFileName)
        : ClassLoader.getSystemResources(fullFileName);
    // 可能存在多个文件。
    if (urls != null) {
        while (urls.hasMoreElements()) {
            // 读取一个文件
            URL url = urls.nextElement();
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("Loading extension of extensible {} from classloader: {} and file: {}",
                    interfaceName, classLoader, url);
            }
            BufferedReader reader = null;
            try {
                reader = new BufferedReader(new InputStreamReader(url.openStream(), "UTF-8"));
                String line;
                while ((line = reader.readLine()) != null) {
                    readLine(url, line);
                }
            } catch (Throwable t) {
                if (LOGGER.isWarnEnabled()) {
                    LOGGER.warn("Failed to load extension of extensible " + interfaceName
                        + " from classloader: " + classLoader + " and file:" + url, t);
                }
            } finally {
                if (reader != null) {
                    reader.close();
                }
            }
        }
    }
}
```

```java
protected void readLine(URL url, String line) throws Throwable {
    String[] aliasAndClassName = parseAliasAndClassName(line);
    if (aliasAndClassName == null || aliasAndClassName.length != 2) {
        return;
    }
    String alias = aliasAndClassName[0];
    String className = aliasAndClassName[1];
    // 读取配置的实现类
    Class tmp;
    try {
        tmp = ClassUtils.forName(className, false);
    } catch (Throwable e) {
        if (LOGGER.isWarnEnabled()) {
            LOGGER.warn("Extension {} of extensible {} is disabled, cause by: {}",
                className, interfaceName, ExceptionUtils.toShortString(e, 2));
        }
        return;
    }
    if (!interfaceClass.isAssignableFrom(tmp)) {
        throw new IllegalArgumentException("Error when load extension of extensible " + interfaceName +
            " from file:" + url + ", " + className + " is not subtype of interface.");
    }
    Class<? extends T> implClass = (Class<? extends T>) tmp;

    // 检查是否有可扩展标识
    Extension extension = implClass.getAnnotation(Extension.class);
    if (extension == null) {
        throw new IllegalArgumentException("Error when load extension of extensible " + interfaceName +
            " from file:" + url + ", " + className + " must add annotation @Extension.");
    } else {
        String aliasInCode = extension.value();
        if (StringUtils.isBlank(aliasInCode)) {
            // 扩展实现类未配置@Extension 标签
            throw new IllegalArgumentException("Error when load extension of extensible " + interfaceClass +
                " from file:" + url + ", " + className + "'s alias of @Extension is blank");
        }
        if (alias == null) {
            // spi文件里没配置，用代码里的
            alias = aliasInCode;
        } else {
            // spi文件里配置的和代码里的不一致
            if (!aliasInCode.equals(alias)) {
                throw new IllegalArgumentException("Error when load extension of extensible " + interfaceName +
                    " from file:" + url + ", aliases of " + className + " are " +
                    "not equal between " + aliasInCode + "(code) and " + alias + "(file).");
            }
        }
        // 接口需要编号，实现类没设置
        if (extensible.coded() && extension.code() < 0) {
            throw new IllegalArgumentException("Error when load extension of extensible " + interfaceName +
                " from file:" + url + ", code of @Extension must >=0 at " + className + ".");
        }
    }
    // 不可以是default和*
    if (StringUtils.DEFAULT.equals(alias) || StringUtils.ALL.equals(alias)) {
        throw new IllegalArgumentException("Error when load extension of extensible " + interfaceName +
            " from file:" + url + ", alias of @Extension must not \"default\" and \"*\" at " + className + ".");
    }
    // 检查是否有存在同名的
    ExtensionClass old = all.get(alias);
    ExtensionClass<T> extensionClass = null;
    if (old != null) {
        // 如果当前扩展可以覆盖其它同名扩展
        if (extension.override()) {
            // 如果优先级还没有旧的高，则忽略
            if (extension.order() < old.getOrder()) {
                if (LOGGER.isDebugEnabled()) {
                    LOGGER.debug("Extension of extensible {} with alias {} override from {} to {} failure, " +
                        "cause by: order of old extension is higher",
                        interfaceName, alias, old.getClazz(), implClass);
                }
            } else {
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("Extension of extensible {} with alias {}: {} has been override to {}",
                        interfaceName, alias, old.getClazz(), implClass);
                }
                // 如果当前扩展可以覆盖其它同名扩展
                extensionClass = buildClass(extension, implClass, alias);
            }
        }
        // 如果旧扩展是可覆盖的
        else {
            if (old.isOverride() && old.getOrder() >= extension.order()) {
                // 如果已加载覆盖扩展，再加载到原始扩展
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("Extension of extensible {} with alias {}: {} has been loaded, ignore origin {}",
                        interfaceName, alias, old.getClazz(), implClass);
                }
            } else {
                // 如果不能被覆盖，抛出已存在异常
                throw new IllegalStateException(
                    "Error when load extension of extensible " + interfaceClass + " from file:" + url +
                        ", Duplicate class with same alias: " + alias + ", " + old.getClazz() + " and " + implClass);
            }
        }
    } else {
        extensionClass = buildClass(extension, implClass, alias);
    }
    if (extensionClass != null) {
        // 检查是否有互斥的扩展点
        for (Map.Entry<String, ExtensionClass<T>> entry : all.entrySet()) {
            ExtensionClass existed = entry.getValue();
            if (extensionClass.getOrder() >= existed.getOrder()) {
                // 新的优先级 >= 老的优先级，检查新的扩展是否排除老的扩展
                String[] rejection = extensionClass.getRejection();
                if (CommonUtils.isNotEmpty(rejection)) {
                    for (String rej : rejection) {
                        ExtensionClass removed = all.remove(rej);
                        if (removed != null) {
                            if (LOGGER.isInfoEnabled()) {
                                LOGGER.info(
                                    "Extension of extensible {} with alias {}: {} has been reject by new {}",
                                    interfaceName, removed.getAlias(), removed.getClazz(), implClass);
                            }
                        }
                    }
                }
            } else {
                String[] rejection = existed.getRejection();
                if (CommonUtils.isNotEmpty(rejection)) {
                    for (String rej : rejection) {
                        if (rej.equals(extensionClass.getAlias())) {
                            // 被其它扩展排掉
                            if (LOGGER.isInfoEnabled()) {
                                LOGGER.info(
                                    "Extension of extensible {} with alias {}: {} has been reject by old {}",
                                    interfaceName, alias, implClass, existed.getClazz());
                                return;
                            }
                        }
                    }
                }
            }
        }

        loadSuccess(alias, extensionClass);
    }
}
```

com.alipay.sofa.rpc.ext.ExtensionLoader#getExtension(java.lang.String, java.lang.Class[], java.lang.Object[])

```java
public T getExtension(String alias, Class[] argTypes, Object[] args) {
  	//根据名字获取ExtentionClass
    ExtensionClass<T> extensionClass = getExtensionClass(alias);
    if (extensionClass == null) {
        throw new SofaRpcRuntimeException("Not found extension of " + interfaceName + " named: \"" + alias + "\"!");
    } else {
      	//单例，从ConcurrentHashMap中获取
        if (extensible.singleton() && factory != null) { 
            T t = factory.get(alias);
            if (t == null) {
                synchronized (this) {
                    t = factory.get(alias);
                    if (t == null) {
                        t = extensionClass.getExtInstance(argTypes, args);
                        factory.put(alias, t);
                    }
                }
            }
            return t;
        } else { //多例
            return extensionClass.getExtInstance(argTypes, args);
        }
    }
}
```

com.alipay.sofa.rpc.ext.ExtensionClass#getExtInstance(java.lang.Class[], java.lang.Object[])

```java
public T getExtInstance(Class[] argTypes, Object[] args) {
    if (clazz != null) {
        try {
            if (singleton) { // 如果是单例
                if (instance == null) {
                    synchronized (this) {
                        if (instance == null) {
                            instance = ClassUtils.newInstanceWithArgs(clazz, argTypes, args);
                        }
                    }
                }
                return instance; // 保留单例
            } else {
                return ClassUtils.newInstanceWithArgs(clazz, argTypes, args);
            }
        } catch (Exception e) {
            throw new SofaRpcRuntimeException("create " + clazz.getCanonicalName() + " instance error", e);
        }
    }
    throw new SofaRpcRuntimeException("Class of ExtensionClass is null");
}
```

com.alipay.sofa.rpc.common.utils.ClassUtils#newInstanceWithArgs

```java
public static <T> T newInstanceWithArgs(Class<T> clazz, Class<?>[] argTypes, Object[] args)throws SofaRpcRuntimeException {
  	//根据参数自动检测构造方法
    if (CommonUtils.isEmpty(argTypes)) {
        return newInstance(clazz);
    }
    try {
        if (!(clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers()))) {
            Constructor<T> constructor = clazz.getDeclaredConstructor(argTypes);
            constructor.setAccessible(true);
            return constructor.newInstance(args);
        } else {
            Constructor<T>[] constructors = (Constructor<T>[]) clazz.getDeclaredConstructors();
            if (constructors == null || constructors.length == 0) {
                throw new SofaRpcRuntimeException("The " + clazz.getCanonicalName()
                    + " has no constructor with argTypes :" + Arrays.toString(argTypes));
            }
            Constructor<T> constructor = null;
            for (Constructor<T> c : constructors) {
                Class[] ps = c.getParameterTypes();
                if (ps.length == argTypes.length + 1) { // 长度多一
                    boolean allMath = true;
                    for (int i = 1; i < ps.length; i++) { // 而且第二个开始的参数类型匹配
                        if (ps[i] != argTypes[i - 1]) {
                            allMath = false;
                            break;
                        }
                    }
                    if (allMath) {
                        constructor = c;
                        break;
                    }
                }
            }
            if (constructor == null) {
                throw new SofaRpcRuntimeException("The " + clazz.getCanonicalName()
                    + " has no constructor with argTypes :" + Arrays.toString(argTypes));
            } else {
                constructor.setAccessible(true);
                Object[] newArgs = new Object[args.length + 1];
                System.arraycopy(args, 0, newArgs, 1, args.length);
                return constructor.newInstance(newArgs);
            }
        }
    } catch (SofaRpcRuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new SofaRpcRuntimeException(e.getMessage(), e);
    }
}
```

## 单机故障剔除

### 收集信息

com.alipay.sofa.rpc.event.FaultToleranceSubscriber#onEvent

```java
public void onEvent(Event originEvent) { //收集同步、异步调用信息
    Class eventClass = originEvent.getClass();
		
    if (eventClass == ClientSyncReceiveEvent.class) { //同步结果事件
     	// 收集和统计 RPC 同步 调用次数和出现异常的次数
        if (!FaultToleranceConfigManager.isEnable()) {
            return;
        }
        // 同步结果
        ClientSyncReceiveEvent event = (ClientSyncReceiveEvent) originEvent;
        ConsumerConfig consumerConfig = event.getConsumerConfig();
        ProviderInfo providerInfo = event.getProviderInfo();
        InvocationStat result = InvocationStatFactory.getInvocationStat(consumerConfig, providerInfo);
        if (result != null) {
            result.invoke();	//增加调用次数
            Throwable t = event.getThrowable();
            if (t != null) {  // 执行结果异常
                result.catchException(t);
            }
        }
    } else if (eventClass == ClientAsyncReceiveEvent.class) {//异步结果事件
      //收集和统计 RPC 异步 调用次数和出现异常的次数  
      if (!FaultToleranceConfigManager.isEnable()) {
            return;
        }
        // 异步结果
        ClientAsyncReceiveEvent event = (ClientAsyncReceiveEvent) originEvent;
        ConsumerConfig consumerConfig = event.getConsumerConfig();
        ProviderInfo providerInfo = event.getProviderInfo();
        InvocationStat result = InvocationStatFactory.getInvocationStat(consumerConfig, providerInfo);
        if (result != null) {
            result.invoke();
            Throwable t = event.getThrowable();
            if (t != null) {
                result.catchException(t);
            }
        }
    } else if (eventClass == ProviderInfoRemoveEvent.class) {
        ProviderInfoRemoveEvent event = (ProviderInfoRemoveEvent) originEvent;
        ConsumerConfig consumerConfig = event.getConsumerConfig();
        ProviderGroup providerGroup = event.getProviderGroup();
        if (!ProviderHelper.isEmpty(providerGroup)) {
            for (ProviderInfo providerInfo : providerGroup.getProviderInfos()) {
                InvocationStatFactory.removeInvocationStat(consumerConfig, providerInfo);
            }
        }
    } else if (eventClass == ProviderInfoUpdateEvent.class) {
        ProviderInfoUpdateEvent event = (ProviderInfoUpdateEvent) originEvent;
        ConsumerConfig consumerConfig = event.getConsumerConfig();
        List<ProviderInfo> add = new ArrayList<ProviderInfo>();
        List<ProviderInfo> remove = new ArrayList<ProviderInfo>();
        ProviderHelper.compareGroup(event.getOldProviderGroup(), event.getNewProviderGroup(), add, remove);
        for (ProviderInfo providerInfo : remove) {
            InvocationStatFactory.removeInvocationStat(consumerConfig, providerInfo);
        }
    } else if (eventClass == ProviderInfoUpdateAllEvent.class) {
        ProviderInfoUpdateAllEvent event = (ProviderInfoUpdateAllEvent) originEvent;
        ConsumerConfig consumerConfig = event.getConsumerConfig();
        List<ProviderInfo> add = new ArrayList<ProviderInfo>();
        List<ProviderInfo> remove = new ArrayList<ProviderInfo>();
        ProviderHelper.compareGroups(event.getOldProviderGroups(), event.getNewProviderGroups(), add, remove);
        for (ProviderInfo providerInfo : remove) {
            InvocationStatFactory.removeInvocationStat(consumerConfig, providerInfo);
        }
    }
}
```

com.alipay.sofa.rpc.client.aft.InvocationStatFactory#getInvocationStat(com.alipay.sofa.rpc.config.ConsumerConfig, com.alipay.sofa.rpc.client.ProviderInfo)

```javascript
public static InvocationStat getInvocationStat(ConsumerConfig consumerConfig, ProviderInfo providerInfo) {
    String appName = consumerConfig.getAppName();
    if (appName == null) {
        return null;
    }
    //此应用是否开启单机故障摘除功能
    if (FaultToleranceConfigManager.isRegulationEffective(appName)) {
        return getInvocationStat(new InvocationStatDimension(providerInfo, consumerConfig));
    }
    return null;
}
```

```java
public static InvocationStat getInvocationStat(InvocationStatDimension statDimension) {
  	//ALL_STATS:InvocationStatDimension(消费端+服务端)->InvocationStat
    InvocationStat invocationStat = ALL_STATS.get(statDimension);
    if (invocationStat == null) {
      //创建InvocationStat
        invocationStat = new ServiceExceptionInvocationStat(statDimension);
      	//放入Map
        InvocationStat old = ALL_STATS.putIfAbsent(statDimension, invocationStat);
        if (old != null) {
            invocationStat = old;
        }
        for (InvocationStatListener listener : LISTENERS) {
            listener.onAddInvocationStat(invocationStat);
        }
    }
    return invocationStat;
}
```

com.alipay.sofa.rpc.client.aft.impl.TimeWindowRegulator.TimeWindowRegulatorListener#onAddInvocationStat	

```java
public void onAddInvocationStat(InvocationStat invocationStat) {
    if (measureStrategy != null) {
        MeasureModel measureModel = measureStrategy.buildMeasureModel(invocationStat);
        if (measureModel != null) {
            measureModels.add(measureModel);
          	//以appName+interface为单位，启动一个线程
            startRegulate();
        }
    }
}
```

### 构建测量模型

com.alipay.sofa.rpc.client.aft.impl.ServiceHorizontalMeasureStrategy#buildMeasureModel

```java
public MeasureModel buildMeasureModel(InvocationStat invocationStat) {
    InvocationStatDimension statDimension = invocationStat.getDimension();
  	//appname+interface
    String key = statDimension.getDimensionKey();
    //appServiceMeasureModels:（AppName+interface）-> MeasureModel（invocationStat），维护统计信息	（执行次数、异常次数、异常比例）
  	//measureModel维护所有的invocationStat
    MeasureModel measureModel = appServiceMeasureModels.get(key);
    if (measureModel == null) {
        measureModel = new MeasureModel(statDimension.getAppName(), statDimension.getService());
        MeasureModel oldMeasureModel = appServiceMeasureModels.putIfAbsent(key, measureModel);
        if (oldMeasureModel == null) {
            measureModel.addInvocationStat(invocationStat);
            return measureModel;
        } else {
            oldMeasureModel.addInvocationStat(invocationStat);
            return null;
        }
    } else {
        measureModel.addInvocationStat(invocationStat);
        return null;
    }
}
```

### 启动窗口计算

com.alipay.sofa.rpc.client.aft.impl.TimeWindowRegulator#startRegulate

```java
public void startRegulate() {
    if (measureStarted.compareAndSet(false, true)) {
        measureScheduler.start();//每隔1s执行一次
    }
}
```

com.alipay.sofa.rpc.client.aft.impl.TimeWindowRegulator.MeasureRunnable#run

```java
public void run() {
 		//每隔1s执行一次加1操作
    measureCounter.incrementAndGet();
    for (MeasureModel measureModel : measureModels) {
        try {
            if (isArriveTimeWindow(measureModel)) {
               //根据收集信息计算测量结果
                MeasureResult measureResult = measureStrategy.measure(measureModel);
              	//根据测量结果执行降级或者恢复策略
                regulationExecutor.submit(new RegulationRunnable(measureResult));
            }
        } catch (Exception e) {
            LOGGER.error	WithApp(measureModel.getAppName(), "Error when doMeasure: " + e.getMessage(), e);
        }
    }
}
```

#### 计算测量结果

com.alipay.sofa.rpc.client.aft.impl.ServiceHorizontalMeasureStrategy#measure

```java
public MeasureResult measure(MeasureModel measureModel) {

    MeasureResult measureResult = new MeasureResult();
    measureResult.setMeasureModel(measureModel);

    String appName = measureModel.getAppName();
    List<InvocationStat> stats = measureModel.getInvocationStats();
    if (!CommonUtils.isNotEmpty(stats)) {
        return measureResult;
    }

    //如果有被新剔除的InvocationStat，则不会存在于该次获取结果中。
    List<InvocationStat> invocationStats = getInvocationStatSnapshots(stats);

    long timeWindow = FaultToleranceConfigManager.getTimeWindow(appName);
    /* leastWindowCount在同一次度量中保持不变*/
    long leastWindowCount = FaultToleranceConfigManager.getLeastWindowCount(appName);
    leastWindowCount = leastWindowCount < LEGAL_LEAST_WINDOW_COUNT ? LEGAL_LEAST_WINDOW_COUNT
        : leastWindowCount;

    /* 计算平均异常率和度量单个ip的时候都需要使用到appWeight*/
    double averageExceptionRate = calculateAverageExceptionRate(invocationStats, leastWindowCount);

    double leastWindowExceptionRateMultiple = FaultToleranceConfigManager
        .getLeastWindowExceptionRateMultiple(appName);

    for (InvocationStat invocationStat : invocationStats) {
        MeasureResultDetail measureResultDetail = null;
        InvocationStatDimension statDimension = invocationStat.getDimension();

        long windowCount = invocationStat.getInvokeCount();
        long invocationLeastWindowCount = getInvocationLeastWindowCount(invocationStat,
            ProviderInfoWeightManager.getWeight(statDimension.getProviderInfo()),
            leastWindowCount);
      	//平均异常率为-1，忽略 
        if (averageExceptionRate == -1) {
            measureResultDetail = new MeasureResultDetail(statDimension, MeasureState.IGNORE);
        } else {
          	//窗口内的调用次数大于窗口内最小调用次数
            if (invocationLeastWindowCount != -1 && windowCount >= invocationLeastWindowCount) {
              	//计算窗口内的异常率
                double windowExceptionRate = invocationStat.getExceptionRate();
                if (averageExceptionRate == 0) {
                    measureResultDetail = new MeasureResultDetail(statDimension, MeasureState.HEALTH);
                } else {
                  	//计算异常率是平均异常率的倍数，默认大于等于6倍，异常，否则正常
                    double windowExceptionRateMultiple = CalculateUtils.divide(
                        windowExceptionRate, averageExceptionRate);
                    measureResultDetail = windowExceptionRateMultiple >= leastWindowExceptionRateMultiple ?
                        new MeasureResultDetail(statDimension, MeasureState.ABNORMAL) :
                        new MeasureResultDetail(statDimension, MeasureState.HEALTH);
                }
              	//设置窗口内的异常率
                measureResultDetail.setAbnormalRate(windowExceptionRate);
              	//设置平均异常率
                measureResultDetail.setAverageAbnormalRate(averageExceptionRate);
								                	     measureResultDetail.setLeastAbnormalRateMultiple(leastWindowExceptionRateMultiple);
            } else {
                measureResultDetail = new MeasureResultDetail(statDimension, MeasureState.IGNORE);
            }
        }

        measureResultDetail.setWindowCount(windowCount);
        measureResultDetail.setTimeWindow(timeWindow);
        measureResultDetail.setLeastWindowCount(invocationLeastWindowCount);
        measureResult.addMeasureDetail(measureResultDetail);
    }

    InvocationStatFactory.updateInvocationStats(invocationStats);
    return measureResult;
}
```

#### 执行降级恢复策略

com.alipay.sofa.rpc.client.aft.impl.TimeWindowRegulator.RegulationRunnable#run

```java
public void run() {//根据度量结果执行降级策略
    List<MeasureResultDetail> measureResultDetails = measureResult.getAllMeasureResultDetails();
    for (MeasureResultDetail measureResultDetail : measureResultDetails) {
        try {
            doRegulate(measureResultDetail); //执行降级、恢复
        } catch (Exception e) {
            LOGGER.errorWithApp(measureResult.getMeasureModel().getAppName(),
                "Error when doRegulate: " + e.getMessage(), e);
        }
    }
}
```

```java
void doRegulate(MeasureResultDetail measureResultDetail) {
    MeasureState measureState = measureResultDetail.getMeasureState();
    InvocationStatDimension statDimension = measureResultDetail.getInvocationStatDimension();
		//此app是否开启降级功能
    boolean isDegradeEffective = regulationStrategy.isDegradeEffective(measureResultDetail);
    if (isDegradeEffective) {
        if (measureState.equals(MeasureState.ABNORMAL)) {//节点异常
            boolean isReachMaxDegradeIpCount = regulationStrategy.isReachMaxDegradeIpCount(measureResultDetail);
            if (!isReachMaxDegradeIpCount) {
                weightDegradeStrategy.degrade(measureResultDetail);
            } else {
                String appName = measureResult.getMeasureModel().getAppName();
                if (LOGGER.isInfoEnabled(appName)) {
                    LOGGER.infoWithApp(appName, LogCodes.getLog(LogCodes.INFO_REGULATION_ABNORMAL_NOT_DEGRADE,
                        "Reach degrade number limit.", statDimension.getService(), statDimension.getIp(),
                        statDimension.getAppName()));
                }
            }
        } else if (measureState.equals(MeasureState.HEALTH)) {//正常
          	//之前是否异常
            boolean isExistDegradeList = regulationStrategy.isExistInTheDegradeList(measureResultDetail);
            if (isExistDegradeList) {
              	//恢复
                recoverStrategy.recover(measureResultDetail);
              	//从降级列表移除
                regulationStrategy.removeFromDegradeList(measureResultDetail);
            }
            //没有被降级过，因此不需要被恢复。
        }
    } else {
        if (measureState.equals(MeasureState.ABNORMAL)) { //异常
          	//执行日志降级策略，打印日志
            logDegradeStrategy.degrade(measureResultDetail);
            String appName = measureResult.getMeasureModel().getAppName();
            if (LOGGER.isInfoEnabled(appName)) {
                LOGGER.infoWithApp(appName, LogCodes.getLog(LogCodes.INFO_REGULATION_ABNORMAL_NOT_DEGRADE,
                    "Degrade switch is off", statDimension.getService(),
                    statDimension.getIp(), statDimension.getAppName()));
            }
        }
    }
}
```

##### WeightDegradeStrategy

com.alipay.sofa.rpc.client.aft.impl.WeightDegradeStrategy#degrade

```java
public void degrade(MeasureResultDetail measureResultDetail) {
    super.degrade(measureResultDetail);

    InvocationStatDimension statDimension = measureResultDetail.getInvocationStatDimension();
    String appName = statDimension.getAppName();

    ProviderInfo providerInfo = statDimension.getProviderInfo();
    // 正在预热
    if (providerInfo == null || providerInfo.getStatus() == ProviderStatus.WARMING_UP) {
        return;
    }
  	//当前权重
    int currentWeight = ProviderInfoWeightManager.getWeight(providerInfo);
    //应用的权重降级速率，默认为0.05s
    double weightDegradeRate = FaultToleranceConfigManager.getWeightDegradeRate(appName);
  	//降级的最小权重
    int degradeLeastWeight = FaultToleranceConfigManager.getDegradeLeastWeight(appName);
		//计算降级权重
    int degradeWeight = CalculateUtils.multiply(currentWeight, weightDegradeRate);
    degradeWeight = degradeWeight < degradeLeastWeight ? degradeLeastWeight : degradeWeight;

    // 设置消费端的服务提供方的权重为降级权重
    boolean success = ProviderInfoWeightManager.degradeWeight(providerInfo, degradeWeight);
    if (success && LOGGER.isInfoEnabled(appName)) {
        LOGGER.infoWithApp(appName, "the weight was degraded. serviceUniqueName:["
            + statDimension.getService() + "],ip:["
            + statDimension.getIp() + "],origin weight:["
            + currentWeight + "],degraded weight:["
            + degradeWeight + "].");
    }
}
```

##### WeightRecoverStrategy

com.alipay.sofa.rpc.client.aft.impl.WeightRecoverStrategy#recover

```java
public void recover(MeasureResultDetail measureResultDetail) {//恢复策略
    InvocationStatDimension statDimension = measureResultDetail.getInvocationStatDimension();
    ProviderInfo providerInfo = statDimension.getProviderInfo();
    // provider被移除 或者正在预热
    if (providerInfo == null || providerInfo.getStatus() == ProviderStatus.WARMING_UP) {
        return;
    }
  	//获取权重（正常的权重或者预热中的权重）
    Integer currentWeight = ProviderInfoWeightManager.getWeight(providerInfo);
    if (currentWeight == -1) {
        return;
    }	
		//获取应用名称
    String appName = statDimension.getAppName();
  	//权重恢复速率，默认为2
    double weightRecoverRate = FaultToleranceConfigManager.getWeightRecoverRate(appName);
		//计算恢复权重
    int recoverWeight = CalculateUtils.multiply(currentWeight, weightRecoverRate);
  	//原始权重
    int originWeight = statDimension.getOriginWeight();

    // recover weight of this provider info
    if (recoverWeight >= originWeight) { //大于原始权重
        measureResultDetail.setRecoveredOriginWeight(true); //恢复到原始权重
      	//设置消费端的提供方的权重为原始权重
        ProviderInfoWeightManager.recoverOriginWeight(providerInfo, originWeight);
        if (LOGGER.isInfoEnabled(appName)) {
            LOGGER.infoWithApp(appName, "the weight was recovered to origin value. serviceUniqueName:["
                + statDimension.getService() + "],ip:["
                + statDimension.getIp() + "],origin weight:["
                + currentWeight + "],recover weight:["
                + originWeight + "].");
        }
    } else {
        measureResultDetail.setRecoveredOriginWeight(false);//尚未恢复到原始权重
      	//设置消费端的提供方的恢复权重，提供方状态为RECOVERING
        boolean success = ProviderInfoWeightManager.recoverWeight(providerInfo, recoverWeight);
        if (success && LOGGER.isInfoEnabled(appName)) {
            LOGGER.infoWithApp(appName, "the weight was recovered. serviceUniqueName:["
                + statDimension.getService() + "],ip:["
                + statDimension.getIp() + "],origin weight:["
                + currentWeight + "],recover weight:["
                + recoverWeight + "].");
        }
    }
}
```