# Sofa-Bolt源码分析

* [服务端初始化](#服务端初始化)
* [绑定端口号](#绑定端口号)
* [注册自定义的处理器](#注册自定义的处理器)
* [解码](#解码)
  * [创建解码器](#创建解码器)
  * [AbstractBatchDecoder](#abstractbatchdecoder)
  * [读取数据](#读取数据)
  * [解码](#解码-1)
* [RPC处理器处理请求](#rpc处理器处理请求)
* [发送响应信息](#发送响应信息)
* [编码](#编码)
* [支持的3种请求方式](#支持的3种请求方式)
  * [发送oneway请求，不关心返回结果](#发送oneway请求不关心返回结果)
  * [同步发送请求](#同步发送请求)
  * [异步执行，返回future，](#异步执行返回future)
* [总结](#总结)

# 服务端初始化

```java
private final EventLoopGroup   bossGroup     = NettyEventLoopUtil                                                                              .newEventLoopGroup(                                                                                 1,
                                                                                    new NamedThreadFactory(
                                                                                     "Rpc-netty-server-boss",
                                                                                        false));
```

com.alipay.remoting.rpc.RpcServer#doInit

```java
protected void doInit() {
    if (this.addressParser == null) {
        this.addressParser = new RpcAddressParser();
    }
    if (this.switches().isOn(GlobalSwitch.SERVER_MANAGE_CONNECTION_SWITCH)) { //是否管理客户端连接
        // in server side, do not care the connection service state, so use null instead of global switch
        ConnectionSelectStrategy connectionSelectStrategy = new RandomSelectStrategy(null);
        this.connectionManager = new DefaultServerConnectionManager(connectionSelectStrategy);
        this.connectionManager.startup();

        this.connectionEventHandler = new RpcConnectionEventHandler(switches());
        this.connectionEventHandler.setConnectionManager(this.connectionManager);
        this.connectionEventHandler.setConnectionEventListener(this.connectionEventListener);
    } else {
        this.connectionEventHandler = new ConnectionEventHandler(switches());
        this.connectionEventHandler.setConnectionEventListener(this.connectionEventListener);
    }


    initRpcRemoting();
    this.bootstrap = new ServerBootstrap();
    this.bootstrap.group(bossGroup, workerGroup)
        .channel(NettyEventLoopUtil.getServerSocketChannelClass())
        .option(ChannelOption.SO_BACKLOG, ConfigManager.tcp_so_backlog())
        .option(ChannelOption.SO_REUSEADDR, ConfigManager.tcp_so_reuseaddr())
        .childOption(ChannelOption.TCP_NODELAY, ConfigManager.tcp_nodelay())
        .childOption(ChannelOption.SO_KEEPALIVE, ConfigManager.tcp_so_keepalive());
    //设置高低水位线（channel级别）
    initWriteBufferWaterMark();
    //设置ByteBuffer分配器
    if (ConfigManager.netty_buffer_pooled()) { //启用池化的ByteBuffer分配器
        this.bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    } else {
        this.bootstrap.option(ChannelOption.ALLOCATOR, UnpooledByteBufAllocator.DEFAULT)
            .childOption(ChannelOption.ALLOCATOR, UnpooledByteBufAllocator.DEFAULT);
    }

    //  如果支持epoll的话，LEVEL_TRIGGERED
    NettyEventLoopUtil.enableTriggeredMode(bootstrap);

    final boolean idleSwitch = ConfigManager.tcp_idle_switch();
    //在ideleTime时间内，如果既没有读操作也没有写操作，发送IdleStateEvent，将此连接关闭
    final int idleTime = ConfigManager.tcp_server_idle();
    //监听读写空闲触发的IdleStateEvent事件
    final ChannelHandler serverIdleHandler = new ServerIdleHandler();
    //Rpc处理器，处理远端发送的请求；userProcessors维护用户自定义Processor，不同请求对应不同的Processor
    final RpcHandler rpcHandler = new RpcHandler(true, this.userProcessors);
    this.bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel channel) {
            ChannelPipeline pipeline = channel.pipeline();
            //设置解码器，非共享
            pipeline.addLast("decoder", codec.newDecoder());
            //设置编码器
            pipeline.addLast("encoder", codec.newEncoder());
            //是否启用读写空闲连接检测，移除空闲连接，释放资源
            if (idleSwitch) {
                pipeline.addLast("idleStateHandler", new IdleStateHandler(0, 0, idleTime,
                    TimeUnit.MILLISECONDS));
                pipeline.addLast("serverIdleHandler", serverIdleHandler);
            }
            //设置连接事件处理器
            pipeline.addLast("connectionEventHandler", connectionEventHandler);
            //设置用户请求处理器
            pipeline.addLast("handler", rpcHandler);
            //创建Connection
            createConnection(channel);
        }
      
        private void createConnection(SocketChannel channel) {
            Url url = addressParser.parse(RemotingUtil.parseRemoteAddress(channel));
            if (switches().isOn(GlobalSwitch.SERVER_MANAGE_CONNECTION_SWITCH)) {
                //管理创建的Channel
                connectionManager.add(new Connection(channel, url), url.getUniqueKey());
            } else {
                new Connection(channel, url);//创建connection并将其与channel绑定
            }
            //传递Channel连接事件
            channel.pipeline().fireUserEventTriggered(ConnectionEventType.CONNECT);
        }
    });
}
```

# 绑定端口号

com.alipay.remoting.rpc.RpcServer#doStart

```java
protected boolean doStart() throws InterruptedException {
    this.channelFuture = this.bootstrap.bind(new InetSocketAddress(ip(), port())).sync();
    return this.channelFuture.isSuccess();
}
```

# 注册自定义的处理器

com.alipay.remoting.rpc.RpcServer#registerUserProcessor

```java
public void registerUserProcessor(UserProcessor<?> processor) {
    UserProcessorRegisterHelper.registerUserProcessor(processor, this.userProcessors);
}
```

# ServerIdleHandler

服务端处理读写空闲事件

com.alipay.remoting.ServerIdleHandler#userEventTriggered

```java
public void userEventTriggered(final ChannelHandlerContext ctx, Object evt) throws Exception {
    //处理读写空闲事件
    if (evt instanceof IdleStateEvent) {
        try {
            logger.warn("Connection idle, close it from server side: {}",
                RemotingUtil.parseRemoteAddress(ctx.channel()));
            ctx.close();
        } catch (Exception e) {
            logger.warn("Exception caught when closing connection in ServerIdleHandler.", e);
        }
    } else {
        super.userEventTriggered(ctx, evt);
    }
}
```

# 解码

## 创建解码器

com.alipay.remoting.rpc.RpcCodec

```java
@Override
public ChannelHandler newDecoder() {//解码器
    return new RpcProtocolDecoder(RpcProtocolManager.DEFAULT_PROTOCOL_CODE_LENGTH);
}
```

根据读取数据封装成RequestCommand，尚未反序列化requestclass、header、body。在RpcHandler处理过程中根据实际需要进行反序列化，延迟反序列化或者避免无用的反序列化，降低资源消耗

com.alipay.remoting.codec.ProtocolCodeBasedDecoder#decode

```java
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception { //解码
    in.markReaderIndex();
    //解析ProtocolCode
    ProtocolCode protocolCode = decodeProtocolCode(in);
    if (null != protocolCode) {
        //解析ProtocolVersion
        byte protocolVersion = decodeProtocolVersion(in);
        if (ctx.channel().attr(Connection.PROTOCOL).get() == null) {
            ctx.channel().attr(Connection.PROTOCOL).set(protocolCode);
            if (DEFAULT_ILLEGAL_PROTOCOL_VERSION_LENGTH != protocolVersion) {
                ctx.channel().attr(Connection.VERSION).set(protocolVersion);
            }
        }
        //根据protocolCode获取对应的协议
        Protocol protocol = ProtocolManager.getProtocol(protocolCode);
        if (null != protocol) {
            //重置readerIndex
            in.resetReaderIndex();
            //解码封装成Command
            protocol.getDecoder().decode(ctx, in, out);
        } else {
            throw new CodecException("Unknown protocol code: [" + protocolCode
                                     + "] while decode in ProtocolDecoder.");
        }
    }
}
```

## AbstractBatchDecoder

批量解包、批量提交

```java
public static final Cumulator MERGE_CUMULATOR     = new Cumulator() { //累积读取到的字节	
                                                      @Override
                                                      public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
            ByteBuf buffer;
              if (cumulation.writerIndex() > cumulation.maxCapacity() -in.readableBytes()
                  || cumulation.refCnt() > 1) { //cumulation没有足够空间容纳新读取的数据，进行扩容
                  buffer = expandCumulation(alloc,cumulation, in.readableBytes());
             } else {
                  buffer = cumulation;
             }
             buffer.writeBytes(in); //累积新读取的数据
             in.release();//释放读取的数据占用的内存
             return buffer;
       }
     };
```

## 读取数据

com.alipay.remoting.codec.AbstractBatchDecoder#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        RecyclableArrayList out = RecyclableArrayList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            //第一次读取数据
            if (first) {
                cumulation = data;
            } else { //累积已经读取的数据
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            callDecode(ctx, cumulation, out); //解码
        } catch (DecoderException e) {
            throw e;
        } catch (Throwable t) {
            throw new DecoderException(t);
        } finally {
            if (cumulation != null && !cumulation.isReadable()) { //数据读取成功
                numReads = 0;
                cumulation.release(); //释放bytebuf
                cumulation = null;
            } else if (++numReads >= discardAfterReads) {//如果连续读取16次，为了避免发生OOM，将会丢弃数据
                // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                // See https://github.com/netty/netty/issues/4275
                numReads = 0; //重置读取次数
                discardSomeReadBytes(); //丢弃数据
            }
            int size = out.size();
            if (size == 0) {
                decodeWasNull = true;
            } else if (size == 1) { //读取到一条数据
                ctx.fireChannelRead(out.get(0));
            } else { //多条数据，封装成ArrayList
                ArrayList<Object> ret = new ArrayList<Object>(size);
                for (int i = 0; i < size; i++) {
                    ret.add(out.get(i));
                }
                //批量传递
                ctx.fireChannelRead(ret);
            }

            out.recycle(); //对象回收，重复使用
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```

## 解码

com.alipay.remoting.codec.AbstractBatchDecoder#callDecode

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            int outSize = out.size();
            int oldInputLength = in.readableBytes();
             //调用自定义的解码器，将解码成功的数据加入到out中
            decode(ctx, in, out);
            // Check if this handler was removed before continuing the loop.
            // If it was removed, it is not safe to continue to operate on the buffer.
            //
            // See https://github.com/netty/netty/issues/1664
            if (ctx.isRemoved()) { //判断此Handler是否已经移除，防止出现未知的错误
                break;
            }
            if (outSize == out.size()) { //接收的数据不完整
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }
            if (oldInputLength == in.readableBytes()) {//可读取的字节数没有发生变化，但是解析出了完整的数据
                throw new DecoderException(
                    StringUtil.simpleClassName(getClass())
                            + ".decode() did not read anything but decoded a message.");
            }
            if (isSingleDecode()) { //单条解码，默认false
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Throwable cause) {
        throw new DecoderException(cause);
    }
}
```

# RPC处理器处理请求

com.alipay.remoting.rpc.RpcHandler#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    //解码时设置ProtocolCode
    ProtocolCode protocolCode = ctx.channel().attr(Connection.PROTOCOL).get();
    //获取Protocol
    Protocol protocol = ProtocolManager.getProtocol(protocolCode);
    //获取CommandHandler
    protocol.getCommandHandler().handleCommand(
        new RemotingContext(ctx, new InvokeContext(), serverSide, userProcessors), msg);
}
```

处理请求

com.alipay.remoting.rpc.protocol.RpcCommandHandler#handleCommand

```java
public void handleCommand(RemotingContext ctx, Object msg) throws Exception {
    this.handle(ctx, msg);
}
```

com.alipay.remoting.rpc.protocol.RpcCommandHandler#handle

```java
private void handle(final RemotingContext ctx, final Object msg) {
    try {
        if (msg instanceof List) { //接收的的批量请求
            final Runnable handleTask = new Runnable() {
                @Override
                public void run() {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Batch message! size={}", ((List<?>) msg).size());
                    }
                    for (final Object m : (List<?>) msg) {
                        RpcCommandHandler.this.process(ctx, m);
                    }
                }
            };
            //是否由ProcessorManager默认的executor执行
            if (RpcConfigManager.dispatch_msg_list_in_default_executor()) {
                processorManager.getDefaultExecutor().execute(handleTask);
            } else {//直接运行
                handleTask.run();
            }
        } else { //接收单个请求
            process(ctx, msg);
        }
    } catch (final Throwable t) {
        processException(ctx, msg, t);
    }
}
```

com.alipay.remoting.rpc.protocol.RpcCommandHandler#process

```java
private void process(RemotingContext ctx, Object msg) {
    try {
        final RpcCommand cmd = (RpcCommand) msg;
        final RemotingProcessor processor = processorManager.getProcessor(cmd.getCmdCode());
        processor.process(ctx, cmd, processorManager.getDefaultExecutor());
    } catch (final Throwable t) {
        processException(ctx, msg, t);
    }
}
```

com.alipay.remoting.rpc.protocol.RpcRequestProcessor#process

```java
public void process(RemotingContext ctx, RpcRequestCommand cmd, ExecutorService defaultExecutor)  throws Exception {
    if (!deserializeRequestCommand(ctx, cmd, RpcDeserializeLevel.DESERIALIZE_CLAZZ)) {
        return;
    }
    //根据请求中的requestClass，获取自定义UserProcessor
    UserProcessor userProcessor = ctx.getUserProcessor(cmd.getRequestClass());
    if (userProcessor == null) {
        String errMsg = "No user processor found for request: " + cmd.getRequestClass();
        logger.error(errMsg);
        sendResponseIfNecessary(ctx, cmd.getType(), this.getCommandFactory()
            .createExceptionResponse(cmd.getId(), errMsg));
        return;// must end process
    }
    //超时，将请求丢弃,快速失败
    // set timeout check state from user's processor
    ctx.setTimeoutDiscard(userProcessor.timeoutDiscard());
    //是否在IO线程执行（NioEventLoop），默认false
    if (userProcessor.processInIOThread()) {
        if (!deserializeRequestCommand(ctx, cmd, RpcDeserializeLevel.DESERIALIZE_ALL)) {
            return;
        }
        // process in io thread
        new ProcessTask(ctx, cmd).run(); //IO线程中直接运行
        return;// end
    }
    Executor executor;
    //是否用线程池选择器获取线程池
    if (null == userProcessor.getExecutorSelector()) {
        executor = userProcessor.getExecutor();
    } else {
        // in case haven't deserialized in io thread
        // it need to deserialize clazz and header before using executor dispath strategy
        if (!deserializeRequestCommand(ctx, cmd, RpcDeserializeLevel.DESERIALIZE_HEADER)) { //反序列化，获取请求Header、请求Class
            return;
        }
        //try get executor with strategy
        executor = userProcessor.getExecutorSelector().select(cmd.getRequestClass(),
            cmd.getRequestHeader()); //根据getRequestClass、RequestHeader选择线程池
    }
    // Till now, if executor still null, then try default
    if (executor == null) {
        executor = (this.getExecutor() == null ? defaultExecutor : this.getExecutor()); //如果userProcessor中的executor为空，就用默认的executor
    }
    // use the final executor dispatch process task
    executor.execute(new ProcessTask(ctx, cmd)); //将ctx、cmd封装成ProcessTask，扔到executor执行
}
```

com.alipay.remoting.rpc.protocol.RpcRequestProcessor.ProcessTask#run

```java
public void run() {
    try {
        RpcRequestProcessor.this.doProcess(ctx, msg);
    } catch (Throwable e) {
        //protect the thread running this task
        String remotingAddress = RemotingUtil.parseRemoteAddress(ctx.getChannelContext()
            .channel());
        logger
            .error(
                "Exception caught when process rpc request command in RpcRequestProcessor, Id="
                        + msg.getId() + "! Invoke source address is [" + remotingAddress
                        + "].", e);
    }
}
```

超时检测、反序列化、将请求交由指定的UserPorcessor处理

com.alipay.remoting.rpc.protocol.RpcRequestProcessor#doProcess

```java
public void doProcess(final RemotingContext ctx, RpcRequestCommand cmd) throws Exception {
    long currentTimestamp = System.currentTimeMillis();
    preProcessRemotingContext(ctx, cmd, currentTimestamp);
    //判断是否超时,超时直接丢弃
    if (ctx.isTimeoutDiscard() && ctx.isRequestTimeout()) {
        //超时打印日志
        timeoutLog(cmd, currentTimestamp, ctx);// do some log
        return;// then, discard this request
    }
    debugLog(ctx, cmd, currentTimestamp);
    //反序列化出请求的所有信息
    if (!deserializeRequestCommand(ctx, cmd, RpcDeserializeLevel.DESERIALIZE_ALL)) {
        return;
    }
    //UserProcessor处理接收到的请求
    dispatchToUserProcessor(ctx, cmd);
}
```

填充运行信息

com.alipay.remoting.rpc.protocol.RpcRequestProcessor#preProcessRemotingContext

```java
private void preProcessRemotingContext(RemotingContext ctx, RpcRequestCommand cmd,
                                       long currentTimestamp) {
    ctx.setArriveTimestamp(cmd.getArriveTime()); //请求到达时间,解码成功之后设置
    ctx.setTimeout(cmd.getTimeout()); //请求超时时间
    ctx.setRpcCommandType(cmd.getType()); //请求类型
    ctx.getInvokeContext().putIfAbsent(InvokeContext.BOLT_PROCESS_WAIT_TIME,
        currentTimestamp - cmd.getArriveTime()); //此请求正式执行前等待的时间
}
```

自定义的UserProcessor处理请求

com.alipay.remoting.rpc.protocol.RpcRequestProcessor#dispatchToUserProcessor

```java
private void dispatchToUserProcessor(RemotingContext ctx, RpcRequestCommand cmd) {
    final int id = cmd.getId(); //请求Id
    final byte type = cmd.getType(); //请求类型
    // processor here must not be null, for it have been checked before
    UserProcessor processor = ctx.getUserProcessor(cmd.getRequestClass());
    if (processor instanceof AsyncUserProcessor) { //异步处理
        try {
            processor.handleRequest(processor.preHandleRequest(ctx, cmd.getRequestObject()),
                new RpcAsyncContext(ctx, cmd, this), cmd.getRequestObject());
        } catch (RejectedExecutionException e) {
            logger
                .warn("RejectedExecutionException occurred when do ASYNC process in RpcRequestProcessor");
            sendResponseIfNecessary(ctx, type, this.getCommandFactory()
                .createExceptionResponse(id, ResponseStatus.SERVER_THREADPOOL_BUSY));
        } catch (Throwable t) {
            String errMsg = "AYSNC process rpc request failed in RpcRequestProcessor, id=" + id;
            logger.error(errMsg, t);
            sendResponseIfNecessary(ctx, type, this.getCommandFactory()
                .createExceptionResponse(id, t, errMsg));
        }
    } else { //同步处理（先调用preHandleRequest，生成BizContext，将其和请求对象一起传递给handleRequest）
        try {
            Object responseObject = processor
                .handleRequest(processor.preHandleRequest(ctx, cmd.getRequestObject()),
                    cmd.getRequestObject());

            sendResponseIfNecessary(ctx, type,
                this.getCommandFactory().createResponse(responseObject, cmd));
        } catch (RejectedExecutionException e) {
            logger
                .warn("RejectedExecutionException occurred when do SYNC process in RpcRequestProcessor");
            sendResponseIfNecessary(ctx, type, this.getCommandFactory()
                .createExceptionResponse(id, ResponseStatus.SERVER_THREADPOOL_BUSY));
        } catch (Throwable t) {
            String errMsg = "SYNC process rpc request failed in RpcRequestProcessor, id=" + id;
            logger.error(errMsg, t);
            sendResponseIfNecessary(ctx, type, this.getCommandFactory()
                .createExceptionResponse(id, t, errMsg));
        }
    }
}
```

# 发送响应信息

com.alipay.remoting.rpc.protocol.RpcRequestProcessor#sendResponseIfNecessary

```java
public void sendResponseIfNecessary(final RemotingContext ctx, byte type,
                                    final RemotingCommand response) {
    final int id = response.getId();
    //请求类型REQUEST_ONEWAY，不需要发送响应
    if (type != RpcCommandType.REQUEST_ONEWAY) {
        RemotingCommand serializedResponse = response;
        try {
            //序列化（class）
            response.serialize();
        } catch (SerializationException e) {
            String errMsg = "SerializationException occurred when sendResponseIfNecessary in RpcRequestProcessor, id="
                            + id;
            logger.error(errMsg, e);
            serializedResponse = this.getCommandFactory().createExceptionResponse(id,
                ResponseStatus.SERVER_SERIAL_EXCEPTION, e);
            try {
                serializedResponse.serialize();// serialize again for exception response
            } catch (SerializationException e1) {
                // should not happen
                logger.error("serialize SerializationException response failed!");
            }
        } catch (Throwable t) {
            String errMsg = "Serialize RpcResponseCommand failed when sendResponseIfNecessary in RpcRequestProcessor, id="
                            + id;
            logger.error(errMsg, t);
            serializedResponse = this.getCommandFactory()
                .createExceptionResponse(id, t, errMsg);
        }
        ctx.writeAndFlush(serializedResponse).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (logger.isDebugEnabled()) {
                    logger.debug("Rpc response sent! requestId="
                                 + id
                                 + ". The address is "
                                 + RemotingUtil.parseRemoteAddress(ctx.getChannelContext()
                                     .channel()));
                }
                if (!future.isSuccess()) {
                    logger.error(
                        "Rpc response send failed! id="
                                + id
                                + ". The address is "
                                + RemotingUtil.parseRemoteAddress(ctx.getChannelContext()
                                    .channel()), future.cause());
                }
            }
        });
    } else {
        if (logger.isDebugEnabled()) {
            logger.debug("Oneway rpc request received, do not send response, id=" + id
                         + ", the address is "
                         + RemotingUtil.parseRemoteAddress(ctx.getChannelContext().channel()));
        }
    }
}
```

# 编码

io.netty.handler.codec.MessageToByteEncoder#write

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        if (acceptOutboundMessage(msg)) {
            I cast = (I) msg;
            //分配bytebuf，默认堆外内存
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
              	//调用自定义的编码器
                encode(ctx, cast, buf);
            } finally {
                ReferenceCountUtil.release(cast);
            }
            if (buf.isReadable()) {
                ctx.write(buf, promise);
            } else {
                buf.release();
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
            buf = null;
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) {
            buf.release();
        }
    }
}
```

com.alipay.remoting.codec.ProtocolCodeBasedEncoder#encode

```java
protected void encode(ChannelHandlerContext ctx, Serializable msg, ByteBuf out)
                                                                               throws Exception {
    //获取解码时，此channel使用的协议
    Attribute<ProtocolCode> att = ctx.channel().attr(Connection.PROTOCOL);
    ProtocolCode protocolCode;
    if (att == null || att.get() == null) { //默认的协议码
        protocolCode = this.defaultProtocolCode;
    } else {
        protocolCode = att.get();
    }
     //获取协议
    Protocol protocol = ProtocolManager.getProtocol(protocolCode);
    //获取协议的编码器，进行编码
    protocol.getEncoder().encode(ctx, msg, out);
}
```

# 支持的3种请求方式

## oneway请求

不关心返回结果，发送失败打印错误信息

com.alipay.remoting.BaseRemoting#oneway

```java
protected void oneway(final Connection conn, final RemotingCommand request) {
    try {
        conn.getChannel().writeAndFlush(request).addListener(new ChannelFutureListener() {

            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (!f.isSuccess()) { //发送失败
                    logger.error("Invoke send failed. The address is {}",
                        RemotingUtil.parseRemoteAddress(conn.getChannel()), f.cause());
                }
            }

        });
    } catch (Exception e) {
        if (null == conn) {
            logger.error("Conn is null");
        } else {
            logger.error("Exception caught when sending invocation. The address is {}",
                RemotingUtil.parseRemoteAddress(conn.getChannel()), e);
        }
    }
}
```

## 同步请求

阻塞，直到接收服务端的响应结果

com.alipay.remoting.BaseRemoting#invokeSync

```java
protected RemotingCommand invokeSync(final Connection conn, final RemotingCommand request,
                                     final int timeoutMillis) throws RemotingException,
                                                             InterruptedException {
    //创建InvokeFuture                                                           
    final InvokeFuture future = createInvokeFuture(request, request.getInvokeContext());
    conn.addInvokeFuture(future);
    final int requestId = request.getId();
    try {
        conn.getChannel().writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (!f.isSuccess()) {
                    conn.removeInvokeFuture(requestId);//移除future
                    future.putResponse(commandFactory.createSendFailedResponse(
                        conn.getRemoteAddress(), f.cause())); //唤醒
                    logger.error("Invoke send failed, id={}", requestId, f.cause());
                }
            }

        });
    } catch (Exception e) {
        conn.removeInvokeFuture(requestId);
        future.putResponse(commandFactory.createSendFailedResponse(conn.getRemoteAddress(), e));
        logger.error("Exception caught when sending invocation, id={}", requestId, e);
    }
    //阻塞timeoutMillis，如果没有响应，返回TIMEOUT Response
    RemotingCommand response = future.waitResponse(timeoutMillis);
    if (response == null) {
        conn.removeInvokeFuture(requestId);
        response = this.commandFactory.createTimeoutResponse(conn.getRemoteAddress());
        logger.warn("Wait response, request id={} timeout!", requestId);
    }
    return response;
}
```

## 异步执行

### 返回future

com.alipay.remoting.BaseRemoting#invokeWithFuture

```java
protected InvokeFuture invokeWithFuture(final Connection conn, final RemotingCommand request,final int timeoutMillis) {
    final InvokeFuture future = createInvokeFuture(request, request.getInvokeContext());
  	//Connection维护InvokeFuture
    conn.addInvokeFuture(future);
    final int requestId = request.getId();
    try {
        //HashedWheelTimer实现定时任务调度
        Timeout timeout = TimerHolder.getTimer().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                InvokeFuture future = conn.removeInvokeFuture(requestId);
                if (future != null) {
                    future.putResponse(commandFactory.createTimeoutResponse(conn
                        .getRemoteAddress()));
                }
            }

        }, timeoutMillis, TimeUnit.MILLISECONDS);
        future.addTimeout(timeout);

        conn.getChannel().writeAndFlush(request).addListener(new ChannelFutureListener() {

            @Override
            public void operationComplete(ChannelFuture cf) throws Exception {
                if (!cf.isSuccess()) { //发送失败
                    InvokeFuture f = conn.removeInvokeFuture(requestId);
                    if (f != null) {
                        f.cancelTimeout(); //删除定时任务
                        f.putResponse(commandFactory.createSendFailedResponse(
                            conn.getRemoteAddress(), cf.cause()));
                    }
                    logger.error("Invoke send failed. The address is {}",
                        RemotingUtil.parseRemoteAddress(conn.getChannel()), cf.cause());
                }
            }

        });
    } catch (Exception e) {
        //异常，移除Future
        InvokeFuture f = conn.removeInvokeFuture(requestId);
        if (f != null) {
            f.cancelTimeout(); //取消定时事件
            f.putResponse(commandFactory.createSendFailedResponse(conn.getRemoteAddress(), e));
        }
        logger.error("Exception caught when sending invocation. The address is {}",
            RemotingUtil.parseRemoteAddress(conn.getChannel()), e);
    }
    return future;
}
```

### 设置callback

com.alipay.remoting.BaseRemoting#invokeWithCallback

```java
protected void invokeWithCallback(final Connection conn, final RemotingCommand request,
                                  final InvokeCallback invokeCallback, final int timeoutMillis) {
    //创建DefaultInvokeFuture，封装callback，通过CountDownLatch进行阻塞
    final InvokeFuture future = createInvokeFuture(conn, request, request.getInvokeContext(),
        invokeCallback);
    conn.addInvokeFuture(future);
    final int requestId = request.getId();
    try {
        Timeout timeout = TimerHolder.getTimer().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                InvokeFuture future = conn.removeInvokeFuture(requestId);
                if (future != null) {
                    //创建超时响应
                    future.putResponse(commandFactory.createTimeoutResponse(conn
                        .getRemoteAddress()));
                    //尝试异步调用
                    future.tryAsyncExecuteInvokeCallbackAbnormally();
                }
            }

        }, timeoutMillis, TimeUnit.MILLISECONDS);
        future.addTimeout(timeout); //绑定Timeout

        //加急写，立即发送请求，不需要缓存到ChannelOutboundBuffer，但是会降低吞吐量
        conn.getChannel().writeAndFlush(request).addListener(new ChannelFutureListener() {

            @Override
            public void operationComplete(ChannelFuture cf) throws Exception {
                if (!cf.isSuccess()) { //发送失败
                    InvokeFuture f = conn.removeInvokeFuture(requestId);
                    if (f != null) {
                        f.cancelTimeout(); //删除Timeout
                        f.putResponse(commandFactory.createSendFailedResponse(
                            conn.getRemoteAddress(), cf.cause())); //创建发送失败的响应
                        f.tryAsyncExecuteInvokeCallbackAbnormally(); //异步执行异常的回调函数
                    }
                    logger.error("Invoke send failed. The address is {}",
                        RemotingUtil.parseRemoteAddress(conn.getChannel()), cf.cause());
                }
            }

        });
    } catch (Exception e) {
        InvokeFuture f = conn.removeInvokeFuture(requestId);
        if (f != null) {
            f.cancelTimeout();
            f.putResponse(commandFactory.createSendFailedResponse(conn.getRemoteAddress(), e));
            f.tryAsyncExecuteInvokeCallbackAbnormally();
        }
        logger.error("Exception caught when sending invocation. The address is {}",
            RemotingUtil.parseRemoteAddress(conn.getChannel()), e);
    }
}
```



# 处理接收的响应

com.alipay.remoting.rpc.protocol.RpcResponseProcessor#doProcess

```java
public void doProcess(RemotingContext ctx, RemotingCommand cmd) {
		//获取Channel的Connection对象
    Connection conn = ctx.getChannelContext().channel().attr(Connection.CONNECTION).get();
    //根据请求Id从Connection获取InvokeFuture，Connection对象维护了此channel已发送但未接收响应的请求
    InvokeFuture future = conn.removeInvokeFuture(cmd.getId());
    ClassLoader oldClassLoader = null;
    try {
        if (future != null) {
            if (future.getAppClassLoader() != null) {
                oldClassLoader = Thread.currentThread().getContextClassLoader();
                Thread.currentThread().setContextClassLoader(future.getAppClassLoader());
            }
           //填充响应
            future.putResponse(cmd);
          //从时间轮删除定时事件
            future.cancelTimeout();
            try {
                future.executeInvokeCallback(); //执行回调
            } catch (Exception e) {
                logger.error("Exception caught when executing invoke callback, id={}",
                    cmd.getId(), e);
            }
        } else {
            logger
                .warn("Cannot find InvokeFuture, maybe already timeout, id={}, from={} ",
                    cmd.getId(),
                    RemotingUtil.parseRemoteAddress(ctx.getChannelContext().channel()));
        }
    } finally {
        if (null != oldClassLoader) {
            Thread.currentThread().setContextClassLoader(oldClassLoader);
        }
    }
}
```

com.alipay.remoting.rpc.DefaultInvokeFuture#executeInvokeCallback

```java
public void executeInvokeCallback() {
    if (callbackListener != null) {
        if (this.executeCallbackOnlyOnce.compareAndSet(false, true)) {
            callbackListener.onResponse(this);
        }
    }
}
```

com.alipay.remoting.rpc.RpcInvokeCallbackListener#onResponse

```java
public void onResponse(InvokeFuture future) {
    InvokeCallback callback = future.getInvokeCallback();
    if (callback != null) { //回调函数不为空
        CallbackTask task = new CallbackTask(this.getRemoteAddress(), future);
        if (callback.getExecutor() != null) {
            // There is no need to switch classloader, because executor is provided by user.
            try {
                callback.getExecutor().execute(task);
            } catch (RejectedExecutionException e) {
                logger.warn("Callback thread pool busy.");
            }
        } else {
            task.run();
        }
    }
}
```

com.alipay.remoting.rpc.RpcInvokeCallbackListener.CallbackTask#run

```java
public void run() { //真正执行回调函数
        //回调函数
        InvokeCallback callback = future.getInvokeCallback();
        ResponseCommand response = null;

        try {
            response = (ResponseCommand) future.waitResponse(0);
        } catch (InterruptedException e) {
            String msg = "Exception caught when getting response from InvokeFuture. The address is "
                         + this.remoteAddress;
            logger.error(msg, e);
        }
        if (response == null || response.getResponseStatus() != ResponseStatus.SUCCESS) {
            try {
                //根据response是否为空以及response status，创建不同的异常
                Exception e;
                if (response == null) {
                    e = new InvokeException("Exception caught in invocation. The address is "
                                            + this.remoteAddress + " responseStatus:"
                                            + ResponseStatus.UNKNOWN, future.getCause());
                } else {
                    response.setInvokeContext(future.getInvokeContext());
                    switch (response.getResponseStatus()) {
                        case TIMEOUT:
                            e = new InvokeTimeoutException(
                                "Invoke timeout when invoke with callback.The address is "
                                        + this.remoteAddress);
                            break;
                        case CONNECTION_CLOSED:
                            e = new ConnectionClosedException(
                                "Connection closed when invoke with callback.The address is "
                                        + this.remoteAddress);
                            break;
                        case SERVER_THREADPOOL_BUSY:
                            e = new InvokeServerBusyException(
                                "Server thread pool busy when invoke with callback.The address is "
                                        + this.remoteAddress);
                            break;
                        case SERVER_EXCEPTION:
                            String msg = "Server exception when invoke with callback.Please check the server log! The address is "
                                         + this.remoteAddress;
                            RpcResponseCommand resp = (RpcResponseCommand) response;
                            resp.deserialize();
                            Object ex = resp.getResponseObject();
                            if (ex instanceof Throwable) {
                                e = new InvokeServerException(msg, (Throwable) ex);
                            } else {
                                e = new InvokeServerException(msg);
                            }
                            break;
                        default:
                            e = new InvokeException(
                                "Exception caught in invocation. The address is "
                                        + this.remoteAddress + " responseStatus:"
                                        + response.getResponseStatus(), future.getCause());

                    }
                }
                //执行回调的异常方法
                callback.onException(e);
            } catch (Throwable e) {
                logger
                    .error(
                        "Exception occurred in user defined InvokeCallback#onException() logic, The address is {}",
                        this.remoteAddress, e);
            }
        } else { // 响应成功
            ClassLoader oldClassLoader = null;
            try {
                if (future.getAppClassLoader() != null) {
                    oldClassLoader = Thread.currentThread().getContextClassLoader();
                    Thread.currentThread().setContextClassLoader(future.getAppClassLoader());
                }
                response.setInvokeContext(future.getInvokeContext());
                RpcResponseCommand rpcResponse = (RpcResponseCommand) response;
                response.deserialize();
                try {
                    callback.onResponse(rpcResponse.getResponseObject());
                } catch (Throwable e) {
                    logger
                        .error(
                            "Exception occurred in user defined InvokeCallback#onResponse() logic.",
                            e);
                }
            } catch (CodecException e) {
                logger
                    .error(
                        "CodecException caught on when deserialize response in RpcInvokeCallbackListener. The address is {}.",
                        this.remoteAddress, e);
            } catch (Throwable e) {
                logger.error(
                    "Exception caught in RpcInvokeCallbackListener. The address is {}",
                    this.remoteAddress, e);
            } finally {
                if (oldClassLoader != null) {
                    Thread.currentThread().setContextClassLoader(oldClassLoader);
                }
            }
        } // enf of else
    } // end of run
}
```

空闲事件

io.netty.handler.timeout.IdleStateHandler#initialize

```java
private void initialize(ChannelHandlerContext ctx) {
    // Avoid the case where destroy() is called before scheduling timeouts.
    // See: https://github.com/netty/netty/issues/143
    switch (state) {
    case 1:
    case 2:
        return;
    }

    state = 1;
    initOutputChanged(ctx);

    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
  //开启全空闲检测（读或者写空闲）
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}
```

io.netty.handler.timeout.IdleStateHandler.AllIdleTimeoutTask#run

```java
protected void run(ChannelHandlerContext ctx) {

    long nextDelay = allIdleTimeNanos;
    if (!reading) {
        nextDelay -= ticksInNanos() - Math.max(lastReadTime, lastWriteTime);
    }
    if (nextDelay <= 0) {
        allIdleTimeout = schedule(ctx, this, allIdleTimeNanos, TimeUnit.NANOSECONDS);

        boolean first = firstAllIdleEvent;
        firstAllIdleEvent = false;

        try {
            if (hasOutputChanged(ctx, first)) {
                return;
            }

          //传递空闲事件
            IdleStateEvent event = newIdleStateEvent(IdleState.ALL_IDLE, first);
            channelIdle(ctx, event);
        } catch (Throwable t) {
            ctx.fireExceptionCaught(t);
        }
    } else {
        // Either read or write occurred before the timeout - set a new
        // timeout with shorter delay.
        allIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}
```

```java
protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    ctx.fireUserEventTriggered(evt);
}
```

ServerIdleHandler

com.alipay.remoting.ServerIdleHandler#userEventTriggered

```java
public void userEventTriggered(final ChannelHandlerContext ctx, Object evt) throws Exception {
    //处理读写空闲事件
    if (evt instanceof IdleStateEvent) {
        try {
            logger.warn("Connection idle, close it from server side: {}",
                RemotingUtil.parseRemoteAddress(ctx.channel()));
            ctx.close(); //关闭连接
        } catch (Exception e) {
            logger.warn("Exception caught when closing connection in ServerIdleHandler.", e);
        }
    } else {
        super.userEventTriggered(ctx, evt);
    }
}
```

# 总结

通信协议的设计

- `ProtocolCode` ：如果一个端口，需要处理多种协议的请求，那么这个字段是必须的。因为需要根据 `ProtocolCode` 来进入不同的核心编解码器。比如在支付宝，因为曾经使用过基于mina开发的通信框架，当时设计了一版协议。因此，我们在设计新版协议时，需要预留该字段，来适配不同的协议类型。该字段可以在想换协议的时候，方便的进行更换。
- `ProtocolVersion` ：确定了某一种通信协议后，我们还需要考虑协议的微小调整需求，因此需要增加一个 `version` 的字段，方便在协议上追加新的字段
- `RequestType` ：请求类型， 比如`request` `response` `oneway`

- `CommandCode` ：请求命令类型，比如 `request` 可以分为：负载请求，或者心跳请求。`oneway` 之所以需要单独设置，是因为在处理响应时，需要做特殊判断，来控制响应是否回传。
- `CommandVersion` ：请求命令版本号。该字段用来区分请求命令的不同版本。如果修改 `Command` 版本，不修改协议，那么就是纯粹代码重构的需求；除此情况，`Command` 的版本升级，往往会同步做协议的升级。
- `RequestId` ：请求 ID，该字段主要用于异步请求时，保留请求存根使用，便于响应回来时触发回调。另外，在日志打印与问题调试时，也需要该字段。
- `Codec` ：序列化器。该字段用于保存在做业务的序列化时，使用的是哪种序列化器。通信框架不限定序列化方式，可以方便的扩展。
- `Switch` ：协议开关，用于一些协议级别的开关控制，比如 CRC 校验，安全校验等。
- `Timeout` ：超时字段，客户端发起请求时，所设置的超时时间。该字段非常有用，在后面会详细讲解用法。
- `ResponseStatus` ：响应码。从字段精简的角度，我们不可能每次响应都带上完整的异常栈给客户端排查问题，因此，我们会定义一些响应码，通过编号进行网络传输，方便客户端定位问题。
- `ClassLen` ：业务请求类名长度
- `HeaderLen` ：业务请求头长度
- `ContentLen` ：业务请求体长度
- `ClassName` ：业务请求类名。需要注意类名传输的时候，务必指定字符集，不要依赖系统的默认字符集。曾经线上的机器，因为运维误操作，默认的字符集被修改，导致字符的传输出现编解码问题。而我们的通信框架指定了默认字符集，因此躲过一劫。
- `HeaderContent` ：业务请求头
- `BodyContent` ：业务请求体
- `CRC32` ：CRC校验码，这也是通信场景里必不可少的一部分，，我们金融业务属性的特征，这个显得尤为重要。



灵活的反序列化时机控制

协议的基本字段所占用空间是比较小的，目前只有24个字节

协议上的主要负载就是 `ClassName` ，`HeaderContent` ， `BodyContent` 这三部分。这三部分的序列化和反序列化是整个请求响应里最耗时的部分

在请求发送阶段，在调用 Netty 的写接口之前，会在业务线程先做好序列化

而在请求接收阶段，反序列化的时机就需要考虑一下了。结合上面提到的最佳实践的网络 IO 模型，请求接收阶段，我们有 IO 线程，业务线程两种线程池



业务逻辑耗时时，IO线程只反序列化classname，业务线程反序列化HeaderContent和BodyContent和做业务逻辑

用户希望使用多个业务线程池时使用，线程池隔离场景，IO线程反序列化className和HeaderContent，根据Header的内容选择业务线程池

用户不希望切换线程，比如IO密集轻计算业务，IO线程反序列化className和HeaderContent与BodyContent和做业务处理逻辑

1、借助HashedWheelTimer实现超时机制

2、快速失败

 		传输协议中设有timeout这个字段，所设置的超时时间通过协议传到了 Server 端

​		服务端解码成功之后记录一个到达时间，当开始进行请求处理的时候，计算当前时间与到达时间的差值超过请求的超时时间，将此请求丢弃。

​		在RPC框架中，还会记录请求开始时间和请求执行完毕的时间，统计请求排队的时间和请求处理的时间。请求执行完毕，计算当前时间与到达时间的差值，如果超时，将此请求丢弃

3、批量优化

​      对Netty的解码器进行了优化，将解码出的业务对象，组装成List，批量提交给ChannelPipeline处理

4、灵活配置业务请求是在IO线程中处理还是由业务线程池处理

5、线程池选择器 `ExecutorSelector` 

​	用户可以提供多个业务线程池，根据ExecutorSelector` 来实现选择合适的线程池。

​	不同的请求使用不同的线程池，对请求进行隔离

​	ExecutorSelector和UserProcessor绑定

​	UserProcessor也可以配置自用的Executor

6、按需进行反序列化，降低资源消耗

用户请求处理器(UserProcessor)

保存业务传输对象的 `className` 与 `UserProcessor` 的对应关系
