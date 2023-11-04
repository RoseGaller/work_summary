# OCTO-RPC

# ServiceConfig初始化

```java
//服务名
protected String serviceName;
// 服务接口, 调用端必配, 服务端若没有配置则取实现类的第一个接口
protected Class<?> serviceInterface;
// 接口实现 必配项
private T serviceImpl;
// 业务线程池配置2: 线程池对象
private ExecutorService bizWorkerExecutor;
// 业务线程池配置3: 方法粒度线程池对象
private Map<String, ExecutorService> methodWorkerExecutors;
```

 ProviderConfig成员属性

```java
protected String appkey;
private String registry;
private int port = Constants.DEFAULT_SERVER_PORT;
 // 单端口单服务
private ServiceConfig serviceConfig;
//单端口多服务
private List<ServiceConfig> serviceConfigList = new ArrayList<>();
// 权重
private Integer weight = Constants.DEFAULT_WEIGHT;
// 预热时间秒
private Integer warmup = 0;
// 分阶段耗时统计
private boolean timelineTrace;
// 兼容bean配置, 也可以SPI配置
private List<Filter> filters = Collections.emptyList();
```

```java
public void init() {
  //参数检查
    check();
    //SPI加载ZookeeperRegistryFactory
    if (StringUtils.isBlank(registry)) {
        RegistryFactory registryFactory = ExtensionLoader.getExtension(RegistryFactory.class);
        registry = registryFactory.getName();
    }
    //暴露的服务
    if (serviceConfig != null) {
        serviceConfigList.add(serviceConfig);
    }
    for (ServiceConfig serviceConfig : serviceConfigList) {
        serviceConfig.check();
        serviceConfig.configTrace(appkey);
    }
    addShutDownHook();
    //发布服务
    ServicePublisher.publishService(this);
}
```

## 发布服务

```java
public static void publishService(ProviderConfig config) {
    //初始化Http服务，实现服务信息、服务方法信息调用
    initHttpServer(RpcRole.PROVIDER);
    //初始化appKey
    initAppkey(config.getAppkey());
    //默认加载NettyServerFactory
    ServerFactory serverFactory = ExtensionLoader.getExtension(ServerFactory.class);
    //默认创建NettyServer
    Server server = serverFactory.buildServer(config);
    //注册服务到注册中心
    registerService(config);
    //维护提供的服务信息
    ProviderInfoRepository.addProviderInfo(config, server);
    LOGGER.info("Dorado service published: {}", getServiceNameList(config));
}
```

## 注册服务

```java
private static void registerService(ProviderConfig config) {
    //解析注册中心的名称、地址
    Map<String, String> registryInfo = parseRegistryCfg(config.getRegistry());
    RegistryFactory registryFactory;
    //SPI加载RegistryFactory
    if (registryInfo.isEmpty()) {
        registryFactory = ExtensionLoader.getExtension(RegistryFactory.class);
    } else {
        registryFactory = ExtensionLoader.getExtensionWithName(RegistryFactory.class, registryInfo.get(Constants.REGISTRY_WAY_KEY));
    }
    //获取注册服务，支持重试策略
    RegistryPolicy registry = registryFactory.getRegistryPolicy(registryInfo, Registry.OperType.REGISTRY);
    registry.register(convertProviderCfg2RegistryInfo(config, registry.getAttachments()));
    registryPolicyMap.put(config.getPort(), registry);
}
```

## 处理请求

com.meituan.dorado.transport.support.ProviderChannelHandler#received

```java
public void received(final Channel channel, final Object message) {
    if (message instanceof Request) {
        final int messageType = ((Request) message).getMessageType();
        final InvokeHandler handler = handlerFactory.getInvocationHandler(messageType, role);
        try {
            prepareReqContext(channel, (Request) message);
           //获取运行时的线程池（默认线程池、业务线程池、方法线程池）
            ExecutorService executor = getExecutor(messageType, message);
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    Response response = null;
                    try {
                        if (Constants.MESSAGE_TYPE_SERVICE == messageType) { // 0:正常请求
                            RpcInvocation invocation = ((Request) message).getData();
                            Request request = (Request) message;
                            Object serviceImpl = serviceImplMap.get(request.getServiceName()); //服务名称(默认就是接口名称),获取对应的实例
                            if (serviceImpl == null) {
                                throw new ServiceException("Not find serviceImpl by serviceName=" + request.getServiceName());
                            }
                            request.setServiceImpl(serviceImpl);
                            invocation.putAttachment(Constants.RPC_REQUEST, message);

                            RpcResult rpcResult = filterHandler.handle(invocation);
                            response = handler.buildResponse((Request) message);
                            response.setResult(rpcResult);
                        } else {
                            response = handler.handle(((Request) message));
                        }
                        send(channel, response);
                    } catch (Throwable e) {
                        if (Constants.MESSAGE_TYPE_SERVICE == messageType) {
                            logger.error("Provider do method invoke failed, {}:{}", e.getClass().getSimpleName(), e.getMessage(), e);
                            AbstractInvokeTrace.reportServerTraceInfoIfNeeded((Request) message, response);
                        } else {
                            logger.warn("Message handle failed, {}:{} ", e.getClass().getSimpleName(), e.getMessage());
                        }
                        boolean isSendFailedResponse = channel.isConnected() &&
                                !(e.getCause() != null && e.getCause() instanceof ClosedChannelException);
                        if (isSendFailedResponse) {
                            sendFailedResponse(channel, handler, message, e);
                        }
                    }
                }
            });
        } catch (RejectedExecutionException e) {
            sendFailedResponse(channel, handler, message, e);
            logger.error("Worker task is rejected", e);
        }
    } else {
        logger.warn("Should not reach here, it means your message({}) is ignored", message);
    }
}
```

# ReferenceConfig初始化

com.meituan.dorado.config.service.ReferenceConfig#init

```java
public synchronized void init() {
    check(); //参数检测
    configRouter(); //设置路由策略
    configTrace(appkey);//链路追踪
    //订阅服务(直接连接服务提供者、连接注册中心)
    clusterHandler = ServiceSubscriber.subscribeService(this);
    //创建代理对象
    ProxyFactory proxyFactory = ExtensionLoader.getExtension(ProxyFactory.class);
    proxyObj = (T) proxyFactory.getProxy(clusterHandler); //获取代理对象
    addShutDownHook(); //添加ShutDownHook
}
```

## 配置路由策略

```java
private void configRouter() {
    RouterFactory.setRouter(serviceName, routerPolicy);
}
```

## 订阅服务

1. 直连连接服务提供方
2. 连接注册中心，获取服务提供方节点，与其建立连接

com.meituan.dorado.bootstrap.invoker.ServiceSubscriber#subscribeService

```java
public static ClusterHandler subscribeService(ReferenceConfig config) {
    initHttpServer(RpcRole.INVOKER); //初始化Http服务
    ClientConfig clientConfig = genClientConfig(config); //获取消费方配置
    InvokerRepository repository;
    if (isDirectConn(config)) { //直连
        repository = new InvokerRepository(clientConfig, true);
    } else { //监听Provider的变化，建立连接
        repository = new InvokerRepository(clientConfig);
        doSubscribeService(config, repository); //注册中心订阅服务
    }
    //获取ClusterHandler
    ClusterHandler clusterHandler = getClusterHandler(config.getClusterPolicy(), repository);
    return clusterHandler;
}
```

## 配置集群策略

1、失败自动恢复，后台记录失败请求重选节点定时重发，通常用于消息通知操作

2、快速失败，只发起一次调用，失败立即报错，通常用于非幂等性的写操作，即重复执行会出现不同结果的操作。

3、失败转移，当出现失败，重试其它节点，通常用于读操作，写建议重试为0或使用failfast

```java
private static ClusterHandler getClusterHandler(String clusterPolicy, InvokerRepository repository) {
    Cluster cluster = ExtensionLoader.getExtensionWithName(Cluster.class, clusterPolicy);
    if (cluster == null) {
        logger.warn("Not find cluster by policy {}, change to {}", clusterPolicy, Constants.DEFAULT_CLUSTER_POLICY);
        clusterPolicy = Constants.DEFAULT_CLUSTER_POLICY;
        cluster = ExtensionLoader.getExtensionWithName(Cluster.class, clusterPolicy);
    }
    if (cluster == null) {
        throw new RpcException("Not find cluster by policy " + clusterPolicy);
    }

    ClusterHandler clusterHandler = cluster.buildClusterHandler(repository);
    if (clusterHandler == null) {
        throw new RpcException("Not find cluster Handler by policy " + clusterPolicy);
    }
    return clusterHandler;
}
```

## 创建动态代理

com.meituan.dorado.rpc.proxy.jdk.JdkProxyFactory#getProxy

```java
public <T> T getProxy(ClusterHandler<T> handler) {
    Class<?>[] interfaces = new Class<?>[]{handler.getInterface()};
    return getProxyWithInterface(handler, interfaces);
}

public <T> T getProxyWithInterface(ClusterHandler<T> handler, Class<?>[] interfaces) {
    return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
            interfaces, new DefaultInvocationHandler(handler));
}
```

## 远程调用

com.meituan.dorado.cluster.ClusterHandler#handle

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    Class<?>[] parameterTypes = method.getParameterTypes();
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(this, args);
    }
    if ("toString".equals(methodName) && parameterTypes.length == 0) {
        return this.toString();
    }
    if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
        return this.hashCode();
    }
    if ("equals".equals(methodName) && parameterTypes.length == 1) {
        return this.equals(args[0]);
    }
    //接口名称、方法、参数、参数类型
    RpcInvocation invocation = new RpcInvocation(handler.getInterface(), method, args,
            parameterTypes);
    TraceTimeline timeline = TraceTimeline.newRecord(handler.getRepository().getClientConfig().isTimelineTrace(),
            TraceTimeline.INVOKE_START_TS);
    invocation.putAttachment(Constants.TRACE_TIMELINE, timeline);
    //ClusterHandler
    return handler.handle(invocation).getReturnVal();
}
```

## 挑选节点

1、应用集群策略

2、应用路由策略

3、负载均衡策略

com.meituan.dorado.cluster.ClusterHandler#handle6/

```java
public RpcResult handle(RpcInvocation invocation) throws Throwable {
    //获取路由策略
    Router router = RouterFactory.getRouter(repository.getClientConfig().getServiceName());
    //获取负载均衡策略
    loadBalance = LoadBalanceFactory.getLoadBalance(repository.getClientConfig().getServiceName());
    //获取所有的Provider
    List<Invoker<T>> invokerList = obtainInvokers();
    //获取可以路由的Provider节点
    List<Invoker<T>> invokersAfterRoute = router.route(invokerList);
    if (invokersAfterRoute == null || invokersAfterRoute.isEmpty()) {
        logger.error("Provider list is empty after route, router policy:{}, ignore router policy", router.getName());
    } else {
        invokerList = invokersAfterRoute;
    }
    //负载均衡器从invokersAfterRoute中挑选一个节点，发送请求
    return doInvoke(invocation, invokerList);
}
```

## 消费端发送请求

com.meituan.dorado.rpc.handler.invoker.AbstractInvokerInvokeHandler#handle1

```java
public Response handle(Request request) throws Throwable {
    DefaultFuture future = new DefaultFuture(request);
    request.putAttachment(Constants.RESPONSE_FUTURE, future);
    ServiceInvocationRepository.putRequestAndFuture(request, future);

    TraceTimeline.record(TraceTimeline.FILTER_FIRST_STAGE_END_TS, request.getData());
    try {
        request.getClient().request(request, request.getTimeout());
        //异步发送
        if (AsyncContext.isAsyncReq(request.getData())) {
            return handleAsync(request, future);
        } else {
            //同步发送
            return handleSync(request, future);
        }
    } finally {
        TraceTimeline.record(TraceTimeline.FILTER_SECOND_STAGE_START_TS, request.getData());
    }
}
```
