# 美团Dorado

```java
public class ServiceBean extends ProviderConfig implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception { //初始化
        init();
    }
}
```

# ServiceConfig

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

 ProviderConfig

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
    //JDK SPI加载ZookeeperRegistryFactory
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
  	//注册钩子函数
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
    //JDKSPI 默认加载NettyServerFactory
    ServerFactory serverFactory = ExtensionLoader.getExtension(ServerFactory.class);
    //创建NettyServer
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

服务端通信

1、暴露端口，接收请求

2、解码器，读取请求

3、处理器，处理请求

com.meituan.dorado.bootstrap.provider.ServicePublisher#publishService

```java
//默认加载NettyServerFactory
ServerFactory serverFactory = ExtensionLoader.getExtension(ServerFactory.class);
//默认创建NettyServer
Server server = serverFactory.buildServer(config);
```

```java
public class NettyServerFactory implements ServerFactory {
    @Override
    public Server buildServer(ProviderConfig serviceConfig) {
        return new NettyServer(serviceConfig);
    }
}
```

com.meituan.dorado.transport.AbstractServer#AbstractServer

```java
public AbstractServer(ProviderConfig providerConfig) throws TransportException {
    this.localAddress = new InetSocketAddress(providerConfig.getPort());
  	//DoradoCodec 默认thrif协议
    this.codec = ExtensionLoader.getExtension(Codec.class); 
    this.providerConfig = providerConfig;
    this.channelHandler = new ProviderChannelHandler(providerConfig); //处理请求
    start();
}
```

```java
protected void doStart() {
 	 //是否启用epoll
  //创建bossgroup，默认1个线程
  //创建workergroup，io线程数默认Runtime.getRuntime().availableProcessors() * 2
    if (Epoll.isAvailable()) {
        logger.info("NettyServer use EpollEventLoopGroup!");
        bossGroup = new EpollEventLoopGroup(Constants.NIO_CONN_THREADS, new DefaultThreadFactory("DoradoServerNettyBossGroup"));
        workerGroup = new EpollEventLoopGroup(providerConfig.getIoWorkerThreadCount(),
                new DefaultThreadFactory("DoradoServerNettyWorkerGroup"));
    } else {
        bossGroup = new NioEventLoopGroup(Constants.NIO_CONN_THREADS, new DefaultThreadFactory("DoradoServerNettyBossGroup"));
        workerGroup = new NioEventLoopGroup(providerConfig.getIoWorkerThreadCount(),
                new DefaultThreadFactory("DoradoServerNettyWorkerGroup"));
    }
		//处理器
    final NettyServerHandler serverHandler = new NettyServerHandler(getChannelHandler());
    clientChannels = serverHandler.getChannels();
  	//服务端附带参数
    final Map<String, Object> attachments = new HashMap<>();
    attachments.put(Constants.SERVICE_IFACE, getDefaultServiceIface(providerConfig.getServiceConfigList()));
  	//分阶段耗时统计
    attachments.put(Constants.TRACE_IS_RECORD_TIMELINE, providerConfig.isTimelineTrace());
    bootstrap = new ServerBootstrap();
    bootstrap.group(bossGroup, workerGroup)
            .channel(workerGroup instanceof EpollEventLoopGroup ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            .option(ChannelOption.SO_REUSEADDR, true)
            .option(ChannelOption.SO_BACKLOG, 1024)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childOption(ChannelOption.TCP_NODELAY, true)
      //内存分配器directBuffer
            .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                  //添加编解码、处理器
                  //编解码是私有的，不可共享
                  //serverHandler处理器是线程安全的，可共享
                    NettyCodec nettyCodec = new NettyCodec(getCodec(), attachments);
                    ch.pipeline()
                            .addLast("codec", nettyCodec)
                            .addLast("handler", serverHandler);
                }
            });
    // 绑定端口
    ChannelFuture channelFuture = bootstrap.bind(getLocalAddress());
  	//阻塞直到绑定成功
    channelFuture.syncUninterruptibly();
  	//ServerChannel
    serverChannel = (ServerChannel) channelFuture.channel();
}
```

解码

```java
private final ByteToMessageDecoder decoder = new ByteToMessageDecoder() {
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        ByteToMessageCodec.this.decode(ctx, in, out);
    }
    @Override
    protected void decodeLast(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        ByteToMessageCodec.this.decodeLast(ctx, in, out);
    }
};
```

io.netty.handler.codec.ByteToMessageCodec#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    decoder.channelRead(ctx, msg);
}
```

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
      //FastThreadLocal存放CodecOutputList，IO线程可以复用
      //CodecOutputList存放解析出的完整的请求
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            if (first) { //第一次读取
                cumulation = data;
            } else { //非第一次读取,可能出发分配内存，存放msg
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
          //调用解码器
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            if (cumulation != null && !cumulation.isReadable()) {
                numReads = 0;
                cumulation.release();
                cumulation = null;
            } else if (++ numReads >= discardAfterReads) {
                // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                // See https://github.com/netty/netty/issues/4275
                numReads = 0;
                discardSomeReadBytes();
            }

            int size = out.size();
            decodeWasNull = !out.insertSinceRecycled();
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```

```java
static CodecOutputList newInstance() {
    return CODEC_OUTPUT_LISTS_POOL.get().getOrCreate();
}
```

```java
public CodecOutputList getOrCreate() {
    if (count == 0) {
        return new CodecOutputList(NOOP_RECYCLER, 4);
    }
    --count;
    int idx = (currentIdx - 1) & mask;
    CodecOutputList list = elements[idx];
    currentIdx = idx;
    return list;
}
```

```java
public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
    @Override
    public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
        final ByteBuf buffer;
        if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
            // Expand cumulation (by replace it) when either there is not more room in the buffer
            // or if the refCnt is greater then 1 which may happen when the user use slice().retain() or
            // duplicate().retain() or if its read-only.
            //
            // See:
            // - https://github.com/netty/netty/issues/2327
            // - https://github.com/netty/netty/issues/1764
            buffer = expandCumulation(alloc, cumulation, in.readableBytes());
        } else {
            buffer = cumulation;
        }
        buffer.writeBytes(in);
        in.release();
        return buffer;
    }
};
```

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            int outSize = out.size();	
						//已经有读取完整的数据
            if (outSize > 0) {
               //传递给处理器进行处理
                fireChannelRead(ctx, out, outSize);
                //清空out
                out.clear();
                //判断handler是否删除
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }
						//可以读取的字节数
            int oldInputLength = in.readableBytes();
        	  //解码
            decodeRemovalReentryProtection(ctx, in, out);
					  //判断handler是否删除
            if (ctx.isRemoved()) {
                break;
            }
						//没有解析出完整的数据
            if (outSize == out.size()) {
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }
						//没有数据可读，却解析出完整的数据，抛出异常
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```

com.meituan.dorado.transport.netty.NettyCodec#decode

```java
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List out) throws Exception {
  //先将Netty中的ByteBuf转换成JDK中的ByteBuffer
    int totalLength = lengthDecoder.decodeLength(in.nioBuffer());
  //上面的转换操作，不会改变ByteBuf中的readerIndex
  //可以读取的字节数
    int readableBytes = in.readableBytes();
    if (totalLength < 0) { //没有读取到完整的数据
        logger.debug("Not getting enough bytes to get totalLength.");
        return;
    }
    if (readableBytes < totalLength) { //没有读取到完整的数据
        logger.debug("Not getting enough bytes, need {} bytes but got {} bytes", totalLength,   readableBytes);
        return;
    }
		//读取到完整数据
    NettyChannel nettyChannel = ChannelManager.getOrAddChannel(ctx.channel());
 		//将ByteBuf中读取到buffer数组中，readerIndex会变化
    byte[] buffer = new byte[totalLength];
    in.readBytes(buffer);
		//序列化，在放到out列表
    out.add(codec.decode(nettyChannel, buffer, attachments));
}
```

```java
public ByteBuffer nioBuffer() {
    return nioBuffer(readerIndex, readableBytes());
}
```

```java
public int readableBytes() {
    return writerIndex - readerIndex;
}
```

```java
public ByteBuffer nioBuffer(int index, int length) {
    checkIndex(index, length);
    index = idx(index);
    return ((ByteBuffer) memory.duplicate().position(index).limit(index + length)).slice();
}
```

com.meituan.dorado.transport.DoradoLengthDecoder#decodeLength

```java
public int decodeLength(ByteBuffer in) { // 4（magic） + 4 （长度）+ 数据
  	//remaining：可以读取的元素个数
    if (in.remaining() < LEAST_BYTE_SIZE) { //4个字节
        return -1;
    }

    byte[] first4Bytes = new byte[4];
    in.get(first4Bytes); //读取到字节数组，ByteBuffer中的position会增加

    if (first4Bytes[0] == MAGIC[0] && first4Bytes[1] == MAGIC[1]) {
        if (in.remaining() < LEAST_BYTE_SIZE) { //剩余的可以读取的元素个数
            return -1;
        } else {
            return in.getInt() + 8;
        }
    } else {
        int originalThriftSize = BytesUtil.bytes2int(first4Bytes, 0);
        if (originalThriftSize > 0) {
            return originalThriftSize + 4;
        } else {
            throw new ProtocolException("Receiving data not octo protocol or original thrift protocol!");
        }
    }
}
```

com.meituan.dorado.codec.octo.OctoCodec#decode

```java
public Object decode(Channel channel, byte[] buffer, Map<String, Object> attachments) throws ProtocolException {
  //反序列化header、body、解压缩都在IO线程执行
    if (buffer.length < PACKAGE_HEAD_LENGTH) {
        throw new ProtocolException("Message length less than header length");
    }
    if (!isOctoProtocol(buffer)) {
        // 非Octo协议
        return decodeThrift(buffer, attachments);
    }
    TraceTimeline timeline = TraceTimeline.newRecord(CommonUtil.objectToBool(attachments.get(Constants.TRACE_IS_RECORD_TIMELINE), false),
            TraceTimeline.DECODE_START_TS);

    Map<String, Object> attachInfo = new HashMap<String, Object>();
    byte[] headerBodyBytes = getHeaderBodyBuff(buffer, attachInfo);
    int headerBodyLength = headerBodyBytes.length;
    short headerLength = BytesUtil.bytes2short(buffer, PACKAGE_HEAD_INFO_LENGTH + TOTAL_LEN_FIELD_LENGTH);

    byte[] headerBytes = new byte[headerLength];
    System.arraycopy(headerBodyBytes, 0, headerBytes, 0, headerLength);
    byte[] bodyBytes = new byte[headerBodyLength - headerLength];
    System.arraycopy(headerBodyBytes, headerLength, bodyBytes, 0, headerBodyLength - headerLength);

    byte serialize = (byte) attachInfo.get(ATTACH_INFO_SERIALIZE_CODE);
    Header header = null;
    try {
        header = decodeOctoHeader(headerBytes);
    } catch (TException e) {
        throw new ProtocolException("Deserialize Octo protocol header failed, cause " + e.getMessage(), e);
    }

    if (header.isSetRequestInfo()) {
        DefaultRequest request = MetaUtil.convertHeaderToRequest(header);
        request.setSerialize(serialize);
        request.setCompressType((CompressType) attachInfo.get(ATTACH_INFO_COMPRESS_TYPE));
        request.setDoChecksum((Boolean) attachInfo.get(ATTACH_INFO_IS_DO_CHECK));
        if (request.getMessageType() != MessageType.Normal.getValue()) {
            if (request.getMessageType() == MessageType.NormalHeartbeat.getValue() ||
                    request.getMessageType() == MessageType.ScannerHeartbeat.getValue()) {
                request.setHeartbeat(true);
            }
            return request;
        }

        timeline.record(TraceTimeline.DECODE_BODY_START_TS);
        RpcInvocation bodyObj = decodeReqBody(serialize, bodyBytes, request);
        request.setData(bodyObj);

        bodyObj.putAttachment(Constants.TRACE_TIMELINE, timeline);
        MetaUtil.recordTraceInfoInDecode(buffer, request);
        return request;
    } else if (header.isSetResponseInfo()) {
        DefaultResponse response = MetaUtil.convertHeaderToResponse(header);
        try {
            if (response.getMessageType() != MessageType.Normal.getValue()) {
                if (response.getMessageType() == MessageType.NormalHeartbeat.getValue() ||
                        response.getMessageType() == MessageType.ScannerHeartbeat.getValue()) {
                    response.setHeartbeat(true);
                }
                return response;
            }
            timeline.record(TraceTimeline.DECODE_BODY_START_TS);
            RpcResult bodyObj = decodeRspBody(serialize, bodyBytes, response);
            response.setResult(bodyObj);
        } catch (Exception e) {
            if (e instanceof TimeoutException) {
                throw e;
            }
            response.setException(e);
        }
        if (response.getRequest() != null && response.getRequest().getData() != null) {
            TraceTimeline.copyRecord(timeline, response.getRequest().getData());
        }
        MetaUtil.recordTraceInfoInDecode(buffer, response);
        return response;
    } else {
        throw new ProtocolException("Deserialize header lack request or response info.");
    }
}
```

com.meituan.dorado.transport.netty.NettyServerHandler#channelRead

com.meituan.dorado.transport.AbstractServer#AbstractServer

```java
this.channelHandler = new ProviderChannelHandler(providerConfig); //处理请求
```

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    try {
        NettyChannel nettyChannel = ChannelManager.getOrAddChannel(ctx.channel());
        handler.received(nettyChannel, msg);
    } finally {
        ChannelManager.removeIfDisconnected(ctx.channel());
    }
}
```

com.meituan.dorado.transport.support.ProviderChannelHandler#received

```java
public void received(final Channel channel, final Object message) {
  	//先判断反序列化出的对象类型
    if (message instanceof Request) {
        final int messageType = ((Request) message).getMessageType();
        //根据请求类型获取InvokeHandler
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
                            //根据服务名称(默认就是接口名称),获取对应的实例
                            Object serviceImpl = serviceImplMap.get(request.getServiceName());
                            if (serviceImpl == null) {
                                throw new ServiceException("Not find serviceImpl by serviceName=" + request.getServiceName());
                            }
                            request.setServiceImpl(serviceImpl);
                            invocation.putAttachment(Constants.RPC_REQUEST, message);
                            //处理请求
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

com.meituan.dorado.transport.support.ProviderChannelHandler#send

```java
public void send(Channel channel, Object message) {
    channel.send(message);
}
```

```java
public void send(Object message) throws TransportException {
    String messageName = message == null ? "" : ClazzUtil.getClazzSimpleName(message.getClass());
    if (isClosed()) {
        logger.warn("Channel{} closed, won't send {}", this.toString(), messageName);
        return;
    }
    if (!isConnected()) {
        throw new TransportException(
                "Failed to send message " + messageName + ", cause: Channel" + this + " is not connected.");

    }
    doSend(message);
}
```

```java
public void doSend(final Object message) {
    try {
        ioChannel.writeAndFlush(message).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) {
                if (!f.isSuccess()) {
                    logger.error("Failed to send message[{}] to {}.", message.toString(), getRemoteAddress(), f.cause());
                    if (message instanceof Request) {
                        final Request request = (Request) message;
                        ResponseFuture future = (ResponseFuture) request.getAttachment(Constants.RESPONSE_FUTURE);
                        if (future != null) {
                            future.setCause(new TransportException("Failed to send message " + message +
                                    " to " + getRemoteAddress(), f.cause()));
                        }
                    }
                }
            }
        });
    } catch (Throwable e) {
        throw new TransportException("Failed to send message " + message + " to " + getRemoteAddress() + ", caused by " + e.getMessage(), e);
    }
}
```

# ReferenceConfig

# 初始化

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

批量处理，请求先写⼊入RingBuffer；
• 优化线程模型，将序列化与反序列化这种耗时的操作从Netty的IO
线程中挪到⽤用户线程池中；
• 启⽤用压缩以应对⼤大数据量的请求，默认snappy压缩算法；
• 定制msgpack序列化，序列化模版，同时还⽀支持fast json、hessian
等多种序列化协议；