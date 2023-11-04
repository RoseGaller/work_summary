# Janus网关源码分析

* [启动流程](#启动流程)
  * [初始化网关](#初始化网关)
    * [初始化路由](#初始化路由)
      * [加载路由配置](#加载路由配置)
      * [转换路由配置](#转换路由配置)
        * [合并Predicate](#合并predicate)
        * [合并Filter](#合并filter)
  * [启动HTTP容器](#启动http容器)
* [处理请求流程](#处理请求流程)
  * [获取客户端请求](#获取客户端请求)
  * [处理入口](#处理入口)
  * [获取路由配置](#获取路由配置)
  * [执行过滤器链](#执行过滤器链)
  * [过滤器类型](#过滤器类型)
    * [PRE_IN](#pre_in)
    * [INBOUND](#inbound)
    * [INVOKE](#invoke)
      * [HttpFilter](#httpfilter)
      * [HttpLbByHostsFilter](#httplbbyhostsfilter)
      * [LBHttpFilter](#lbhttpfilter)
    * [OUTBOUND](#outbound)
    * [AFTER_OUT](#after_out)
  * [请求后端服务](#请求后端服务)
  * [后端响应处理](#后端响应处理)


# 启动流程

org.xujin.janus.JanusServerApplication#main

```java
public static void main(String[] args) throws MalformedURLException {
    // 初始化网关Filter和配置
    JanusBootStrap.initGateway();
 
    // 启动HTTP容器,监听外部请求
     NettyServer nettyServer = new NettyServer();
    int httpPort = ApplicationConfig.getApplicationPort();
    logger.info("start netty  Server...,port:{}", httpPort);
    nettyServer.start(httpPort);

   //启动网关的Damon Server
    int daemonPort = httpPort + 1;
    logger.info("start daemon Server...,port:{}", daemonPort);
    JanusDaemonServer.start(daemonPort);

    //设置DaemonClient配置
    logger.info("start set daemonClient config...");
    JanusDaemonClient.setDaemonClient();
    logger.info("start First Janus Server Ping Admin...");
    JanusDaemonClient.pingJanusAdmin();
}
```

## 初始化网关

org.xujin.janus.startup.JanusBootStrap#initGateway

```java
public static void initGateway() {
    try {
        //I.初始化本地静态配置
        ApplicationConfig.init();
        //I.静态Filter 加载
        StaticFilterLoader.load();
        //I.静态Predicate 加载
        StaticPredicateLoader.load();
        //I.监听配置文件的变化
        ConfigChangeListener.listen();
        //I.监听route配置变化
        RouteChangeListener.listen();
        //I.监听plugin文件变化
        ClassFileChangeListener.listen();
        //I.初始化注册中心
        initRegisterCenter();
        //II.加载配置文件
        if (ApplicationConfig.getLocal()) {
            LocalConfigLoader.load();//从本地配置文
        } else {
            AdminConfigLoader.load(); //远程配置中心
        }
        //III.动态加载Filter Class、Predicate Class
        DynamicFileManger.startPoller();
        //III.初始化JanusMetrics
        JanusMetricsInitializer.init();
        //IV.初始化路由
        RouteRepo.init();
    } catch (Exception ex) {
        logger.error("init janus error,message:"+ex.getMessage(),ex);
        DynamicFileManger.stopPoller();
        RegistryServiceRepo.destroy();
        JanusServerApplication.fail(ex.getMessage());
    }
}
```

### 初始化路由

#### 加载路由配置

org.xujin.janus.core.route.RouteLoader#loadFromConfig

```java
public static void loadFromConfig() {
    RoutesConfig routeConfigList = ConfigRepo.getServerConfig().getRoutesConfig();
    if (routeConfigList == null
            || routeConfigList.getRoutes() == null || routeConfigList.getRoutes().size() <= 0) {
        logger.debug("no route config found ");
        return;
    }
    //get route form routeConfig
    List<Route> allRoutes = routeConfigList.getRoutes().stream()
            .filter(e -> e.getId() != null)
            .map(e -> convertToRoute(e)) //将RouteConfig转换为Route
            .collect(Collectors.toList());
    RouteRepo.add(allRoutes);
}
```

#### 转换路由配置

```java
public static Route convertToRoute(RouteConfig routeConfig) {
    if (routeConfig == null) {
        return null;
    }
  	//合并Predicate
    Predicate predicate = combinePredicate(routeConfig);
    //合并Filter
    List<Filter> filters = combineFilters(routeConfig);
  	//排序  
    filters = FilterSort.sort(filters);
    //获取负载均衡名称
    String loadBalancerName = routeConfig.getLoadBalancerName();
    //获取负载均衡器
    LoadBalancer loadBalancer = LoadBalancerFactory.create(loadBalancerName);
    //创建Route
    Route route = Route.builder(routeConfig)
            .predicate(predicate)
            .filters(filters)
            .loadBalancer(loadBalancer)
            .metadata(routeConfig.getMetadata())
            .build();
    return route;

}
```

##### 合并Predicate

```java
private static Predicate combinePredicate(RouteConfig routeConfig) {
    List<PredicatesConfig> predicatesConfigs = routeConfig.getPredicates();
    if (predicatesConfigs == null || predicatesConfigs.size() <= 0) {
        return null;
    }
    //get predicate from config
    Predicate predicateFirst = PredicateRepo.get(predicatesConfigs.get(0).getName());
    if (predicateFirst == null) {
        return null;
    }
    for (int i = 1; i < predicatesConfigs.size(); i++) {
        Predicate predicate = PredicateRepo.get(predicatesConfigs.get(i).getName());
        if (predicate != null) {
            predicateFirst = predicateFirst.and(predicate);
        }
    }
    return predicateFirst;
}
```

##### 合并Filter

```java
private static List<Filter> combineFilters(RouteConfig routeConfig) {
    List<Filter> resultFilter = new ArrayList<>();
  	//系统过滤器
    List<Filter> systemFilters = FilterRepo.getSystemFilters();
    resultFilter.addAll(systemFilters);
  	//全局过滤器
    List<Filter> globalFilters = getGlobalFilters();
    resultFilter.addAll(globalFilters);
    //路由配置中的过滤器配置
    List<FilterConfig> filterConfigs = routeConfig.getFilters();
    if (filterConfigs == null) {
        return resultFilter;
    }
    //过滤filter
    List<Filter> configFilter = filterConfigs.stream()
            .map(e -> FilterRepo.get(e.getName())) //根据过滤器名称从FilterRepo获取Filter
            .filter(e -> e != null)
            .collect(Collectors.toList());
    if (configFilter == null || configFilter.size() <= 0) {
        return resultFilter;
    }
    resultFilter.addAll(configFilter);
    return resultFilter;
}
```

## 启动HTTP容器

org.xujin.janus.core.netty.NettyServer#start

```java
public void start(int port) {
    //内存泄露检测
    ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.ADVANCED);
    //初始化NioEventLoopGroup线程池
    initEventPool();
    ServerBootstrap bootstrap = new ServerBootstrap();
    bootstrap.group(bossGroup, workGroup)
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_REUSEADDR, serverConfig.isReUseAddr())
            .option(ChannelOption.SO_BACKLOG, serverConfig.getBackLog())
            .childOption(ChannelOption.SO_RCVBUF, serverConfig.getRevBuf())
            .childOption(ChannelOption.SO_SNDBUF, serverConfig.getSndBuf())
            .childOption(ChannelOption.TCP_NODELAY, serverConfig.isTcpNoDelay())
            .childOption(ChannelOption.SO_KEEPALIVE, serverConfig.isKeepalive())
            .childHandler(initChildHandler());
    //绑定端口号，注册监听器，监听绑定成功事件
    bootstrap.bind(port).addListener((ChannelFutureListener) channelFuture -> {
        if (channelFuture.isSuccess()) { //成功绑定端口
            changeOnLine(true); //修改为上线状态
            log.info("服务端启动成功【" + IpUtils.getHost() + ":" + port + "】");
        } else {
            log.error("服务端启动失败【" + IpUtils.getHost() + ":" + port + "】,cause:"
                    + channelFuture.cause().getMessage());
            shutdown(port);
        }
    });
}
```

org.xujin.janus.core.netty.NettyServer#initChildHandler

```java
private ChannelInitializer initChildHandler() {
    return new ChannelInitializer<SocketChannel>() {
        //处理请求的线程池
        final ThreadPoolExecutor serverHandlerPool = ThreadPoolUtils.makeServerThreadPool(NettyServer.class.getSimpleName());
        //网关Server的Http请求入口Handler
        final NettyServerHandler nettyServerHandler =  new NettyServerHandler(serverHandlerPool);
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            //initHandler(ch.pipeline(), serverConfig);
            ch.pipeline().addLast(new IdleStateHandler(180, 0, 0));
            ch.pipeline().addLast("logging", new LoggingHandler(LogLevel.DEBUG));
            // We want to allow longer request lines, headers, and chunks
            // respectively.
            ch.pipeline().addLast("decoder", new HttpRequestDecoder(
                    serverConfig.getMaxInitialLineLength(),
                    serverConfig.getMaxHeaderSize(),
                    serverConfig.getMaxChunkSize()));
            //ch.pipeline().addLast("inflater", new HttpContentDecompressor());
            ch.pipeline().addLast("aggregator", new HttpObjectAggregator(serverConfig.getMaxHttpLength()));
            //响应解码器
            ch.pipeline().addLast("http-encoder", new HttpResponseEncoder());
            //目的是支持异步大文件传输（）
            ch.pipeline().addLast("http-chunked", new ChunkedWriteHandler());
            //网关Server的Netty HTTP连接处理类
            ch.pipeline().addLast("process", nettyServerHandler);
        }
    };
}
```

# 处理请求流程

## 获取客户端请求

org.xujin.janus.core.netty.server.handler.NettyServerHandler#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    log.info("<---1.janus server start processing request--->");
    if (msg instanceof FullHttpRequest) {

        FullHttpRequest fullHttpRequest = (FullHttpRequest) msg;
        log.info("FullHttpRequest is : " + fullHttpRequest);
				//获取DefaultJanusCtx
        DefaultJanusCtx currCtx = ctx.channel().attr(ctxAttributeKey).get();
        String cookieStr = fullHttpRequest.headers().get(HttpHeaderNames.COOKIE);
        Set<Cookie> cookies = null;
        if (cookieStr != null) {
            cookies = ServerCookieDecoder.STRICT.decode(cookieStr);
        }
        //请求信息填充到currCtx
        currCtx.setOriFullHttpRequest(fullHttpRequest);
        currCtx.setOriHttpResponse(null);
        currCtx.setIsKeepAlive(HttpUtil.isKeepAlive(fullHttpRequest));
        currCtx.setCookies(cookies);

        // 判断只有http协议解析成功，才会进行心跳检查，否则不进行心跳检查
        if (fullHttpRequest.decoderResult().isSuccess()) {
            if (isHeatBeatUrl(fullHttpRequest, currCtx)) { //处理心跳请求
                return;
            }
        }
        
        //执行Janus Filter逻辑入口
        FilterRunner.run(currCtx);
    } else {
        if (msg instanceof ReferenceCounted) {
            ReferenceCountUtil.release(msg);
        }
    }
}
```

## 处理入口

org.xujin.janus.core.FilterRunner#run

```java
public static void run(DefaultJanusCtx ctx) {
    FilterContext context = new FilterContext(ctx);
    Tags tags = Tags.of(MetricsConst.TAG_KEY_URL, context.getCtx().getOriFullHttpRequest().uri());
    JanusMetrics.counter(MetricsConst.METRIC_RECEIVE_REQUEST, tags);
    try {
        //根据请求获取匹配的路由配置
        Route route = matchRoute(context);
        long start = System.currentTimeMillis();
        //绑定开始处理时间、路由配置
        context.getSessionContext().put(SessionContextKey.FILTER_START_TIME, start);
        context.getSessionContext().put(SessionContextKey.FILTER_ROUTE, route);
        //start filter chain 开始执行
        FilterChain.start(context);

    } catch (CallNotPermittedException ex) {
        CircuitBreaker circuitBreaker = (CircuitBreaker) context.getSessionContext().get(SessionContextKey.CIRCUIT_BREAKER);
        logger.error("circuit breaker name:" + circuitBreaker.getName() + " is open", ex);
        //监控埋点
        JanusMetrics.counter(MetricsConst.METRIC_BREAKER_COUNT, tags);
        JanusMetrics.counter(MetricsConst.METRIC_ERROR_REQUEST, tags);
        context.setTextResponse("service unavailable,message:" + ex.getMessage(), HttpResponseStatus.INTERNAL_SERVER_ERROR);
        ctx.janusToOutsideClientAsyncSend(context.getCtx().getOriHttpResponse());
        JanusMetrics.counter(MetricsConst.METRIC_ERROR_REQUEST, tags);
    } catch (Exception ex) {
        logger.error("run filter chain error", ex);
        context.setTextResponse("run filter chain error,message:" + ex.getMessage(), HttpResponseStatus.BAD_GATEWAY);
        ctx.janusToOutsideClientAsyncSend(context.getCtx().getOriHttpResponse());
        JanusMetrics.counter(MetricsConst.METRIC_ERROR_REQUEST, tags);
    }
}
```

## 获取路由配置

org.xujin.janus.core.FilterRunner#matchRoute

```java
private static Route matchRoute(FilterContext context) {
    //获取所有的路由配置
    Set<Route> allRoutes = RouteRepo.getAllRoutes();
    //获取匹配的路由配置
    Route route = new RouteHandler(allRoutes).lookupRoute(context);
    if (route != null) {
        logger.info("route:" + route.getId() + " matched for " + context.toString());
        return route;
    } else {
        throw new RuntimeException("no route match for " + context.toString());
    }
}
```

```java
public Route lookupRoute(FilterContext filterContext) {
    if (this.allRoutes == null) {
        return null;
    }
    Route route = this.allRoutes.stream()
            .filter(e -> e.getPredicate() != null) //predicate不能为空
            .filter(e -> {
                filterContext.getSessionContext().put(SessionContextKey.PREDICATE_ROUTE, e);
                boolean result = e.getPredicate().test(filterContext);
                filterContext.getSessionContext().put(SessionContextKey.PREDICATE_ROUTE, null);
                return result;
            })
            .findFirst() //获取匹配的第一个路由配置
            .orElse(null);
    return route;
}
```

## 执行过滤器链

路由中包含了需要执行的过滤器

org.xujin.janus.core.filter.FilterChain#start

```java
public static void start(FilterContext context) {
    if (context == null) {
        throw new IllegalArgumentException("context cannot be null");
    }
    int filterStartIndex = -1;
    //存放执行的过滤器的索引
    context.getSessionContext().put(SessionContextKey.FILTER_INDEX, filterStartIndex);
    next(context);
}
```

```java
public static void next(FilterContext context) {
    int index = (int) context.getSessionContext().get(SessionContextKey.FILTER_INDEX);
    index++;
    run(context, index);
}
```

```java
private static void run(FilterContext context, int index) {
    //存放当前执行的过滤器的索引
    context.getSessionContext().put(SessionContextKey.FILTER_INDEX, index);
    //获取当前的路由配置
    Route route = (Route) context.getSessionContext().get(SessionContextKey.FILTER_ROUTE);
    if (route == null) {
        throw new RuntimeException("no route found in " + context.toString());
    }
    //此路由的所有过滤器
    List<Filter> routeFilters = route.getFilters();
    if (routeFilters == null || routeFilters.size() <= 0) {
        throw new RuntimeException("no filters found in " + route.getId());
    }
    if (index >= 0 && index < routeFilters.size()) {
        Filter filter = routeFilters.get(index); //获取当前需要执行的过滤器
        filter.filter(context);//执行过滤器
    }
}
```

模板方法

org.xujin.janus.core.filter.filters.AbstractSyncFilter#filter

```java
public void filter(FilterContext context) {
    if (!isEnable()) { //此过滤器禁用
        logger.info("filter:" + name() + " not execute cause disable,context:" + context.toString());
        FilterChain.next(context); //直接调用下一过滤器
        return;
    }
    logger.info("filter:" + name() + " execute,context:" + context.toString());
    doFilter(context); //执行当前过滤器，路由配置中包含的过滤器需要实现此方法
    FilterChain.next(context);//调用下一过滤器
}
```

## 过滤器类型

### PRE_IN

非功能性的前置过滤器

### INBOUND

功能性的前置过滤器

### INVOKE

负责向后端发起请求调用

#### HttpFilter

org.xujin.janus.core.filter.filters.HttpFilter#doFilter

```java
public void doFilter(FilterContext context) { //处理协议为HTTP或HTTPS的请求
    Route route = getRoute(context);
    String backendUriScheme = route.getProtocol();
    if (!Protocol.HTTP.getValue().equalsIgnoreCase(backendUriScheme)
            && !Protocol.HTTPS.toString().equalsIgnoreCase(backendUriScheme)) {
        skipInvoke(context);//跳过此过滤器
        return;
    }
   // protocol: http
   // hosts:
   //   - 127.0.0.1:8084
    List<String> backendHosts=route.getHosts(); //获取指定的后端地址
    if(null==backendHosts||backendHosts.size()==0){
        return;
    }
    //更改请求地址,选择第一个作为后端请求地址
    String host= backendHosts.get(0);
    String[] hotsStr= host.split(":");
    String requestUriStr = context.getCtx().getOriFullHttpRequest().uri();
    URI requestUri = URI.create(requestUriStr);
    URI newRequestUri = UriBuilder.from(requestUri)
            .replaceScheme(backendUriScheme)
            .replaceHost(String.valueOf(hotsStr[0]))
            .replacePort(Integer.valueOf(hotsStr[1]).intValue())
            .build();
    context.getCtx().getOriFullHttpRequest().setUri(newRequestUri.toString());
    invoke(context);
}
```

#### HttpLbByHostsFilter

org.xujin.janus.core.filter.filters.HttpLbByHostsFilter#doFilter

```java
public void doFilter(FilterContext context) {//从hosts中挑选一个执行
    Route route= getRoute(context);
    String protocol = getRoute(context).getProtocol();//获取协议；lb://hosts
    if (!Protocol.REST_LB_HOSTS.getValue().equalsIgnoreCase(protocol)) {
        skipInvoke(context);
        return;
    }
    LoadBalancer loadBalancer = route.getLoadBalancer(); //获取负载均衡器
    ServerNode serverNode = lookUpServer(loadBalancer, route, context); //挑选请求节点
    String requestUriStr = context.getCtx().getOriFullHttpRequest().uri();
    URI requestUri = URI.create(requestUriStr);
    URI newRequestUri = UriBuilder.from(requestUri)
            .replaceScheme(backendProtocol)
            .replaceHost(serverNode.getHost())
            .replacePort(serverNode.getPort())
            .build();
    context.getCtx().getOriFullHttpRequest().setUri(newRequestUri.toString());
    invoke(context);
}
```

```java
private ServerNode lookUpServer(LoadBalancer loadBalancer, Route route, FilterContext httpContext) {
    LoadBalancerParam loadBalancerParam = new LoadBalancerParam(httpContext.getCtx().getClientIp()
            , httpContext.getCtx().getClientPort());
    List<String> hosts=route.getHosts();
    List<ServerNode> serverNodes = new ArrayList<>();
    for(String host:hosts){ //将host封装成ServerNode
        String [] hostStr=host.split(":");
        ServerNode serverNode=new ServerNode(hostStr[0],Integer.valueOf(hostStr[1]).intValue());
        serverNodes.add(serverNode);
    }
  //负载均衡器挑选请求节点
    ServerNode serverNode = loadBalancer.select(serverNodes, loadBalancerParam);
    if (serverNode == null) {
        throw new RuntimeException("no server found for " + route.getServiceName());
    }
    return serverNode;
}
```

#### LBHttpFilter

org.xujin.janus.core.filter.filters.LBHttpFilter#doFilter

```java
public void doFilter(FilterContext context) {
    Route route=getRoute(context);
    String protocol = route.getProtocol();//lb://sc
    LoadBalancer loadBalancer = route.getLoadBalancer();
    if (!Protocol.REST_LB_SC.getValue().equalsIgnoreCase(protocol)) {
        skipInvoke(context);//跳过此过滤器
        return;
    }
    //挑选节点
    ServerNode serverNode = lookUpServer(loadBalancer, route, context);
    String requestUriStr = context.getCtx().getOriFullHttpRequest().uri();
    URI requestUri = URI.create(requestUriStr);
    URI newRequestUri = UriBuilder.from(requestUri)
            .replaceScheme(backendProtocol)
            .replaceHost(serverNode.getHost())
            .replacePort(serverNode.getPort())
            .build();
    context.getCtx().getOriFullHttpRequest().setUri(newRequestUri.toString());
    //请求后端
    invoke(context);
}
```

```java
private ServerNode lookUpServer(LoadBalancer loadBalancer, Route route, FilterContext httpContext) {
     //获取注册服务
    RegistryService serviceDiscovery = RegistryServiceRepo.getRegistryService();
    LoadBalancerParam loadBalancerParam = new LoadBalancerParam(httpContext.getCtx().getClientIp()
            , httpContext.getCtx().getClientPort());
    //从注册中心拉取服务
    List<ServerNode> serverNodes = serviceDiscovery.getServerNode(route.getServiceName());
  //挑选节点
    ServerNode serverNode = loadBalancer.select(serverNodes, loadBalancerParam);
    if (serverNode == null) {
        throw new RuntimeException("no server found for " + route.getServiceName());
    }
    return serverNode;
}
```

### OUTBOUND

后置过滤器，后端服务返回响应信息后执行

### AFTER_OUT

将响应信息返回给客户端

## 请求后端服务

org.xujin.janus.core.filter.filters.AbstractHttpInvokeFilter#invoke

```java
public void invoke(FilterContext context) {
    //use netty client send async http request
    if (context.getSessionContext().get(SessionContextKey.MOCK_RESPONSE) != null) {
        complete(context);
        return;
    }
    //异步发送
    AsyncHttpRequestHelper.asyncSendHttpRequest(context.getCtx(), httpResponse -> {
        context.getCtx().setOriHttpResponse(httpResponse);
        AsyncCircuitBreakerResult.onComplete(context);
        AsyncRetryResult.onComplete(context, t -> {
            FilterChain.previous(context);
            return null;
        });
        complete(context); //执行下一个过滤器
    }, //设置成功回调方法
    exception -> {
        context.setTextResponse(exception.getMessage(), HttpResponseStatus.GATEWAY_TIMEOUT);
        AsyncCircuitBreakerResult.onError(context, exception);
        AsyncRetryResult.onError(context, exception, t -> {
            FilterChain.previous(context);
            return null;
        });
        error(context, exception);
    }); //设置异常回调方法
}
```

org.xujin.janus.core.util.AsyncHttpRequestHelper#asyncSendHttpRequest

```java
public static void asyncSendHttpRequest(DefaultJanusCtx defaultJanusCtx, Consumer<HttpResponse> completeSupplier,
                                        Consumer<Throwable> errorSupplier) {
    try {
        //成功回调方法
        defaultJanusCtx.setExecResponseCallBack(ctx -> {
            completeSupplier.accept(ctx.getOriHttpResponse());
        });
        defaultJanusCtx.setInnerServerChannelInactiveCallback(ctx -> {
            //与Server的链接断开，正常断开还是异常断开？？
            logger.info("client close");
        });
        //异常回调方法
        defaultJanusCtx.setInnerServerExceptionCaughtCallback((ctx, throwable) -> {
            errorSupplier.accept(throwable);
        });
        //异步发送请求
        			   defaultJanusCtx.janusToInnerServerAsyncSend(defaultJanusCtx.getOriFullHttpRequest());

    } catch (Exception exception) {
        logger.error("execute asyncSendHttpRequest error", exception);
        errorSupplier.accept(exception);
    }
}
```

org.xujin.janus.core.netty.ctx.DefaultJanusCtx#janusToInnerServerAsyncSend

```java
public void janusToInnerServerAsyncSend(HttpRequest httpRequest) throws Exception {
    URI requestUri = URI.create(httpRequest.uri());
    String address = requestUri.getHost() + ":" + getPort(requestUri);
    ReferenceCountUtil.retain(httpRequest);
    httpRequest.headers().set(HttpHeaderNames.HOST,address);
   //发送请求
    new NettyClientSender().asyncSend(address, httpRequest, this);
}
```

获取连接，发送请求

org.xujin.janus.core.netty.client.connect.ConnectClientPool#asyncSend

```java
public void asyncSend(HttpRequest httpRequest, String address, DefaultJanusCtx ctx) throws Exception {
		//获取后端连接
    ConnectClient connectClient = this.getConnectClient(address);
    try {
      //发送请求
        connectClient.send(httpRequest, ctx);
    } catch (Exception e) {
        throw e;
    }
}
```

org.xujin.janus.core.netty.client.NettyConnectClient#send

```java
public void send(HttpRequest httpRequest, DefaultJanusCtx ctx) throws Exception {
    //发送请求通过AttributeKey标记Channel
    AttributeKey<DefaultJanusCtx> ctxAttributeKey = AttributeKey.valueOf(NettyConstants.CHANNEL_CTX_KEY);
    ctx.setInnerServerChannel(channel);
    this.channel.attr(ctxAttributeKey).set(ctx); //绑定channel和ctx
    this.channel.writeAndFlush(httpRequest).sync();//阻塞发送成功
}
```

## 后端响应处理

org.xujin.janus.core.netty.client.handler.NettyClientHandler#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
  log.info("<---3.Janus client Handler Process the returned response----->");
  if (msg instanceof HttpResponse) {
    HttpResponse httpResponse = (HttpResponse) msg;
    //根据key从Channel中获取返回结果,返回给客户端
    DefaultJanusCtx janusCtx = ctx.channel().attr(ctxAttributeKey).get();
    if (janusCtx !=  null) {
      //设置响应
      janusCtx.setOriHttpResponse(httpResponse);
      //执行回调（执行下一个过滤器）
      Optional.ofNullable(janusCtx.getExecResponseCallBack()).ifPresent(cb -> cb.runCallBack(janusCtx));
    }
  }
}
```

