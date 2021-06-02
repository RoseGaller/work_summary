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
            //设置解码器
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
        /**
         * create connection operation<br>
         * <ul>
         * <li>If flag manageConnection be true, use {@link DefaultConnectionManager} to add a new connection, meanwhile bind it with the channel.</li>
         * <li>If flag manageConnection be false, just create a new connection and bind it with the channel.</li>
         * </ul>
         */
        private void createConnection(SocketChannel channel) {
            Url url = addressParser.parse(RemotingUtil.parseRemoteAddress(channel));
            if (switches().isOn(GlobalSwitch.SERVER_MANAGE_CONNECTION_SWITCH)) {
                //管理创建的Channel
                connectionManager.add(new Connection(channel, url), url.getUniqueKey());
            } else {
                new Connection(channel, url);
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

# 解码

## 创建解码器

com.alipay.remoting.rpc.RpcCodec

```java
@Override
public ChannelHandler newDecoder() {//解码器
    //默认协议的长度为1
    return new RpcProtocolDecoder(RpcProtocolManager.DEFAULT_PROTOCOL_CODE_LENGTH);
}
```

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
                                                      public ByteBuf cumulate(ByteBufAllocator alloc,
                                                                              ByteBuf cumulation,
                                                                              ByteBuf in) {
                                                          ByteBuf buffer;
                                                          if (cumulation.writerIndex() > cumulation
                                                              .maxCapacity()
                                                                                         - in.readableBytes()
                                                              || cumulation.refCnt() > 1) { //cumulation没有足够空间容纳新读取的数据，进行扩容
                                                              buffer = expandCumulation(alloc,
                                                                  cumulation,
                                                                  in.readableBytes());
                                                          } else {
                                                              buffer = cumulation;
                                                          }
                                                          buffer.writeBytes(in); //累积新读取的数据
                                                          in.release();//及时释放内存
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
    //解码时已设置ProtocolCode
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
            @SuppressWarnings("unchecked")
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

## 发送oneway请求，不关心返回结果

com.alipay.remoting.BaseRemoting#oneway

```java
protected void oneway(final Connection conn, final RemotingCommand request) {
    try {
        conn.getChannel().writeAndFlush(request).addListener(new ChannelFutureListener() {

            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (!f.isSuccess()) {
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

## 同步发送请求

com.alipay.remoting.BaseRemoting#invokeSync

```java
protected RemotingCommand invokeSync(final Connection conn, final RemotingCommand request,
                                     final int timeoutMillis) throws RemotingException,
                                                             InterruptedException {
    final InvokeFuture future = createInvokeFuture(request, request.getInvokeContext());
    conn.addInvokeFuture(future);
    final int requestId = request.getId();
    try {
        conn.getChannel().writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (!f.isSuccess()) {
                    conn.removeInvokeFuture(requestId);
                    future.putResponse(commandFactory.createSendFailedResponse(
                        conn.getRemoteAddress(), f.cause()));
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

## 异步执行，返回future，

```java
protected InvokeFuture invokeWithFuture(final Connection conn, final RemotingCommand request,
                                        final int timeoutMillis) {
    final InvokeFuture future = createInvokeFuture(request, request.getInvokeContext());
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
                if (!cf.isSuccess()) {
                    InvokeFuture f = conn.removeInvokeFuture(requestId);
                    if (f != null) {
                        f.cancelTimeout();
                        f.putResponse(commandFactory.createSendFailedResponse(
                            conn.getRemoteAddress(), cf.cause()));
                    }
                    logger.error("Invoke send failed. The address is {}",
                        RemotingUtil.parseRemoteAddress(conn.getChannel()), cf.cause());
                }
            }

        });
    } catch (Exception e) {
        //异常，日出Future
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

处理接收的响应

com.alipay.remoting.rpc.protocol.RpcResponseProcessor#doProcess

```java
public void doProcess(RemotingContext ctx, RemotingCommand cmd) {
		//获取对应的Connection
    Connection conn = ctx.getChannelContext().channel().attr(Connection.CONNECTION).get();
  //根据请求ID从Connection获取InvokeFuture
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
          //删除事件
            future.cancelTimeout();
            try {
                future.executeInvokeCallback();
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

# 总结

1、借助HashedWheelTimer实现超时机制

2、快速失败，服务端解码成功之后记录一个到达时间，当开始进行请求处理的时候，计算当前时间与到达时间的差值超过请求的超时时间，将此请求丢弃。在RPC框架中，还会记录请求开始时间和请求执行完毕的时间，统计请求排队的时间和请求处理的时间。请求执行完毕，计算当前时间与到达时间的差值，如果超过超时时间，将此请求丢弃

3、解码器对Netty的解码器进行了优化，将解码出的业务对象，组装成List，批量提交给ChannelPipiline处理

4、灵活配置业务请求是在IO线程中处理还是由业务线程池处理

5、线程池选择器 `ExecutorSelector` ：用户可以提供多个业务线程池，使用 `ExecutorSelector` 来实现选择合适的线程池
