# Netty源码分析

* [服务端初始化](#服务端初始化)
  * [服务端例子](#服务端例子)
  * [NioEventLoopGroup初始化](#nioeventloopgroup初始化)
    * [创建NioEventLoop](#创建nioeventloop)
    * [创建NioEventLoop选择器](#创建nioeventloop选择器)
  * [绑定bossGroup、workGroup](#绑定bossgroupworkgroup)
  * [创建不同类型的Channel](#创建不同类型的channel)
  * [绑定端口](#绑定端口)
    * [初始化注册](#初始化注册)
    * [创建NioServerSocketChannel](#创建nioserversocketchannel)
    * [初始化](#初始化)
    * [注册Channel](#注册channel)
    * [注册OP_ACCEPT](#注册op_accept)
    * [绑定](#绑定)
* [处理客户端连接](#处理客户端连接)
  * [创建NioSocketChannel](#创建niosocketchannel)
  * [注册Channel](#注册channel-1)
  * [调用自定义的ChannelInitializer](#调用自定义的channelinitializer)
* [读数据](#读数据)
  * [自适应数据大小的分配器](#自适应数据大小的分配器)
  * [创建Handle](#创建handle)
  * [分配bytebuf](#分配bytebuf)
  * [读取数据](#读取数据)
  * [记录读取字节数](#记录读取字节数)
  * [触发ChannelRead](#触发channelread)
  * [判断是否可以继续读](#判断是否可以继续读)
  * [触发ChannelReadComplete](#触发channelreadcomplete)
* [写数据](#写数据)
  * [写数据三种方式](#写数据三种方式)
  * [写数据入口](#写数据入口)
  * [写数据到buffer](#写数据到buffer)
    * [将数据转成DirectBuffer](#将数据转成directbuffer)
    * [计算占用的字节](#计算占用的字节)
    * [存放到缓冲区](#存放到缓冲区)
    * [计算累积的数据量](#计算累积的数据量)
    * [超过高水位线设置不可写](#超过高水位线设置不可写)
  * [Flush](#flush)
* [NioEventLoop](#nioeventloop)
  * [优化SelectorImpl](#优化selectorimpl)
  * [selector自动重建](#selector自动重建)
  * [创建任务队列](#创建任务队列)
  * [ioRatio](#ioratio)
  * [事件处理](#事件处理)
    * [计算策略](#计算策略)
    * [处理IO事件](#处理io事件)
    * [处理非IO任务](#处理非io任务)
* [FlushConsolidationHandler](#flushconsolidationhandler)
* [流量整形](#流量整形)
  * [Channel级别](#channel级别)
    * [消息读取的流量整形](#消息读取的流量整形)
    * [消息发送流量整形](#消息发送流量整形)
  * [全局流量整形](#全局流量整形)
* [FastThreadLocal](#fastthreadlocal)
  * [初始化](#初始化-1)
  * [设置值](#设置值)
    * [获取线程私有的本地数组](#获取线程私有的本地数组)
    * [创建线程私有的本地数组](#创建线程私有的本地数组)
    * [赋值](#赋值)
  * [获取值](#获取值)
* [HashedWheelTimer](#hashedwheeltimer)
  * [初始化](#初始化-2)
  * [添加任务](#添加任务)
  * [删除任务](#删除任务)
  * [启动工作线程](#启动工作线程)
  * [工作线程运行流程](#工作线程运行流程)
    * [移除被删除的任务](#移除被删除的任务)
    * [添加任务到时间轮](#添加任务到时间轮)
    * [执行过期任务](#执行过期任务)
* [Recycler](#recycler)
  * [对象的获取](#对象的获取)
  * [对象的回收](#对象的回收)
    * [同线程回收](#同线程回收)
    * [跨线程回收](#跨线程回收)
* [Mpsc Queue](#mpsc-queue)
  * [多生产者单消费的无界队列](#多生产者单消费的无界队列)
  * [多生产者单消费的有界队列](#多生产者单消费的有界队列)
* [HTTP2](#http2)
  * [协议升级](#协议升级)
    * [客户端](#客户端)
    * [服务端](#服务端)
  * [解码](#解码)
* [解码器](#解码器)
  * [FixedLengthFrameDecoder](#fixedlengthframedecoder)
  * [DelimiterBasedFrameDecoder](#delimiterbasedframedecoder)
* [总结](#总结)


# 服务端初始化

## 服务端例子

```java
public final class EchoServer {
    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));
    public static void main(String[] args) throws Exception {
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }
        ByteBuf src = PooledByteBufAllocator.DEFAULT.directBuffer(512);
        System.out.println( 0xFFFFFE00);
      
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        //自定义的业务处理器
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class) //服务端channel类型
             .option(ChannelOption.SO_BACKLOG, 100) //参数
             .handler(new LoggingHandler(LogLevel.INFO)) //server
             .childHandler(new ChannelInitializer<SocketChannel>() { //client
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });
            //绑定端口
            ChannelFuture f = b.bind(PORT).sync();
            // 阻塞直到server socket被关闭
            f.channel().closeFuture().sync();
        } finally {
            // 关闭IO线程池
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

```java
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
   //读
   @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);
    }
   //读完成
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
    //异常
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

## NioEventLoopGroup初始化

bossGroup:
NioEventLoop中的selector轮询创建连接事件(OP_ACCEPT):
创建socketchannel
初始化socketchannel并从workergroup中选择一个NioEventLoop
workerGroup:
将socketchannel注册到选择的NioEventLoop的selector 
注册读事件(OP_READ)到selector上

 io.netty.util.concurrent.MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, java.util.concurrent.Executor, io.netty.util.concurrent.EventExecutorChooserFactory, java.lang.Object...)

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }
    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    //创建NioEventLoop
    children = new EventExecutor[nThreads];
    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }
                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }
    //根据NioeventLoop数量创建选择器
    //DefaultEventExecutorChooserFactory
    chooser = chooserFactory.newChooser(children);
    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };
    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }
    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

### 创建NioEventLoop

io.netty.channel.nio.NioEventLoopGroup#newChild

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
```

### 创建NioEventLoop选择器

io.netty.util.concurrent.DefaultEventExecutorChooserFactory#newChooser

```java
public EventExecutorChooser newChooser(EventExecutor[] executors) {
     //根据NioEventLoop的数量决定使用哪个选择器
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

## 绑定bossGroup、workGroup

io.netty.bootstrap.ServerBootstrap#group(io.netty.channel.EventLoopGroup, io.netty.channel.EventLoopGroup)

```java
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    ObjectUtil.checkNotNull(childGroup, "childGroup");
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup; //workGroup
    return this;
}
```

```java
public B group(EventLoopGroup group) {
    ObjectUtil.checkNotNull(group, "group");
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group; //bossGroup
    return self();
}
```

## 创建不同类型的Channel

io.netty.bootstrap.AbstractBootstrap#channel

```java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
            ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}
```

```java
public B channelFactory(io.netty.channel.ChannelFactory<? extends C> channelFactory) {
    return channelFactory((ChannelFactory<C>) channelFactory);
}
```

```java
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    ObjectUtil.checkNotNull(channelFactory, "channelFactory");
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return self();
}
```

io.netty.channel.ReflectiveChannelFactory#newChannel

```java
public T newChannel() {//泛型T代表不同的Channel
    try {
        //反射创建channel
        return constructor.newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
    }
}
```

## 绑定端口

io.netty.bootstrap.AbstractBootstrap#doBind

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }
    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        //等着register完成来通知再执行bind
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

### 初始化注册

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
      	//创建channel
        channel = channelFactory.newChannel();
        //初始化
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            channel.unsafe().closeForcibly();
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    //开始register
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

### 创建NioServerSocketChannel

io.netty.channel.socket.nio.NioServerSocketChannel#NioServerSocketChannel(java.nio.channels.ServerSocketChannel)

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT); //监听事件
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());//创建Serversocket
}
```

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        //非阻塞模式
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            logger.warn(
                        "Failed to close a partially initialized socket.", e2);
        }
        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId(); //创建channelId
    unsafe = newUnsafe(); //创建NioMessageUnsafe
    pipeline = newChannelPipeline(); //创建DefaultChannelPipeline
}
```

### 初始化

io.netty.bootstrap.ServerBootstrap#init

```java
void init(Channel channel) {
    setChannelOptions(channel, options0().entrySet().toArray(newOptionArray(0)), logger);
    setAttributes(channel, attrs0().entrySet().toArray(newAttrArray(0)));
    ChannelPipeline p = channel.pipeline();
    final EventLoopGroup currentChildGroup = childGroup; // workGroup
    final ChannelHandler currentChildHandler = childHandler; 
    final Entry<ChannelOption<?>, Object>[] currentChildOptions =
            childOptions.entrySet().toArray(newOptionArray(0));
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    //ChannelInitializer一次性初始化handler:
    //负责添加一个ServerBootstrapAcceptor handler，添加完后，自己就移除了:
    //ServerBootstrapAcceptor handler： 负责接收客户端连接创建连接后，对连接的初始化工作。
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

### 注册Channel

io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)

```java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```



```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

io.netty.channel.AbstractChannel.AbstractUnsafe#register

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```



```java
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
         //server socket的注册不会走进下面if,server socket接受连接创建的socket可以走进去。因为accept后就active了。
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

io.netty.channel.nio.AbstractNioChannel#doRegister

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            logger.info("initial register： " + 0);
            //注册0到NioEventLoop的selector上，不是OP_ACCEPT(16)
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```

### 注册OP_ACCEPT

io.netty.channel.DefaultChannelPipeline#fireChannelActive

```java
public final ChannelPipeline fireChannelActive() {
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}
```

io.netty.channel.DefaultChannelPipeline.HeadContext#channelActive

```java
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();
    //注册读事件：读包括：创建连接/读数据）
    readIfIsAutoRead();
}
```

```java
private void readIfIsAutoRead() {
    if (channel.config().isAutoRead()) {
        channel.read();
    }
}
```

io.netty.channel.AbstractChannel#read

```java
public Channel read() {
    pipeline.read();
    return this;
}
```

io.netty.channel.DefaultChannelPipeline#read

```java
public final ChannelPipeline read() {
    tail.read();
    return this;
}
```

io.netty.channel.DefaultChannelPipeline.HeadContext#read

```java
public void read(ChannelHandlerContext ctx) {
    //实际上就是注册OP_ACCEPT/OP_READ事件:创建连接或者读事件
    unsafe.beginRead();
}
```

```java
public final void beginRead() {
    assertEventLoop();
    if (!isActive()) {
        return;
    }
    try {
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}
```

io.netty.channel.nio.AbstractNioChannel#doBeginRead

```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }
    readPending = true;
    final int interestOps = selectionKey.interestOps();
    //注册OP_ACCEPT
    if ((interestOps & readInterestOp) == 0) {
        logger.info("interest ops： " + readInterestOp);
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

### 绑定

io.netty.bootstrap.AbstractBootstrap#doBind0

```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

io.netty.channel.DefaultChannelPipeline.HeadContext#bind

```java
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
}
```

io.netty.channel.AbstractChannel.AbstractUnsafe#bind

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
    //绑定后，才开始激活
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```

io.netty.channel.socket.nio.NioServerSocketChannel#doBind

```java
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

# 客户端如何建立连接？

io.netty.bootstrap.Bootstrap#connect()

```java
public ChannelFuture connect() {
    validate();
    SocketAddress remoteAddress = this.remoteAddress;
    if (remoteAddress == null) {
        throw new IllegalStateException("remoteAddress not set");
    }

    return doResolveAndConnect(remoteAddress, config.localAddress());
}
```

```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    if (regFuture.isDone()) { //绑定到NioEventLoop完成
        if (!regFuture.isSuccess()) { //绑定失败
            return regFuture;
        }
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else { //绑定尚未完成
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();
                    doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel(); //反射
        init(channel); //初始化channel
    } catch (Throwable t) {
        if (channel != null) {
            channel.unsafe().closeForcibly();
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
		//将channel绑定到NioEventLoop
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

```java
void init(Channel channel) {
    ChannelPipeline p = channel.pipeline();
    p.addLast(config.handler());

    setChannelOptions(channel, options0().entrySet().toArray(newOptionArray(0)), logger);
    setAttributes(channel, attrs0().entrySet().toArray(newAttrArray(0)));
}
```

io.netty.bootstrap.Bootstrap#doConnect

```java
private static void doConnect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {

    final Channel channel = connectPromise.channel();
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (localAddress == null) {
                channel.connect(remoteAddress, connectPromise);
            } else {
                channel.connect(remoteAddress, localAddress, connectPromise);
            }
            connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        }
    });
}
```

io.netty.channel.DefaultChannelPipeline#connect(java.net.SocketAddress, io.netty.channel.ChannelPromise)

```java
public final ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return tail.connect(remoteAddress, promise);
}
```

io.netty.channel.AbstractChannelHandlerContext#connect(java.net.SocketAddress, java.net.SocketAddress, io.netty.channel.ChannelPromise)

```java
public ChannelFuture connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    }
    if (isNotValidPromise(promise, false)) {
        // cancelled
        return promise;
    }

    final AbstractChannelHandlerContext next = findContextOutbound(MASK_CONNECT);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeConnect(remoteAddress, localAddress, promise);
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeConnect(remoteAddress, localAddress, promise);
            }
        }, promise, null);
    }
    return promise;
}
```

最终调用HeaderContext的connect方法

io.netty.channel.DefaultChannelPipeline.HeadContext#connect

```java
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```

io.netty.channel.nio.AbstractNioChannel.AbstractNioUnsafe#connect

```java
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    try {
        if (connectPromise != null) { //之前的连接尚未完成
            throw new ConnectionPendingException();
        }
        boolean wasActive = isActive();
        if (doConnect(remoteAddress, localAddress)) { //立刻连接成功
            fulfillConnectPromise(promise, wasActive); //唤醒promise
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // 调度连接超时
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
              	//调度延迟事件，交由eventLoop执行
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                                new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }
          
						//connectPromise添加监听器
            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) { //true：表明在连接创建成功之前，已经被删除了
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise()); //关闭
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```

io.netty.channel.socket.nio.NioSocketChannel#doConnect

```java
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        doBind0(localAddress);
    }

    boolean success = false;
    try {
      	//调用SocketChannel的connect方法
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if (!connected) {//没有立刻连接成功
            selectionKey().interestOps(SelectionKey.OP_CONNECT);//注册连接事件
        }
        success = true;
        return connected;
    } finally {
        if (!success) { //没有连接成功，关闭channel
            doClose();
        }
    }
}
```

io.netty.channel.nio.NioEventLoop#processSelectedKey(java.nio.channels.SelectionKey, io.netty.channel.nio.AbstractNioChannel)

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) { //连接部分源码
  if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
    int ops = k.interestOps();
    ops &= ~SelectionKey.OP_CONNECT;
    k.interestOps(ops);

    unsafe.finishConnect();
  }
}
```

```java
public final void finishConnect() {
    // Note this method is invoked by the event loop only if the connection attempt was
    // neither cancelled nor timed out.

    assert eventLoop().inEventLoop();

    try {
        boolean wasActive = isActive();
        doFinishConnect();
        fulfillConnectPromise(connectPromise, wasActive); //唤醒connectPromise
    } catch (Throwable t) {
        fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
    } finally {
        if (connectTimeoutFuture != null) { //取消事件，并不会从任务队列删除
            connectTimeoutFuture.cancel(false);
        }
        connectPromise = null;
    }
}
```

# 服务端如何处理连接？

服务端初始化时，会调用一下方法

```java
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) {
        final ChannelPipeline pipeline = ch.pipeline();//此处channel为NioSocketChannel
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() { //添加ServerBootstrapAcceptor到服务端的pipeline中
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});
```

加入到pipeline后，会调用handlerAdded方法

io.netty.channel.ChannelInitializer#handlerAdded

```java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        if (initChannel(ctx)) { //执行服务端的pipeline中的ChannelInitializer
            removeState(ctx);
        }
    }
}
```

io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read

```java
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                //连接数
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading());
        } catch (Throwable t) {
            exception = t;
        }

        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            pipeline.fireChannelRead(readBuf.get(i));
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (exception != null) {
            closed = closeOnReadError(exception);

            pipeline.fireExceptionCaught(exception);
        }

        if (closed) {
            inputShutdown = true;
            if (isOpen()) {
                close(voidPromise());
            }
        }
    } finally {
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```

## 创建NioSocketChannel

io.netty.channel.socket.nio.NioServerSocketChannel#doReadMessages

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    //接受新连接创建SocketChannel，调用ServerSocketChannel的accept
    SocketChannel ch = SocketUtils.accept(javaChannel());
    try {
        if (ch != null) { //创建NioSocketChannel，绑定ServerSocketChannel,并注册读事件
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);
        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }
    return 0;
}
```

io.netty.bootstrap.ServerBootstrap.ServerBootstrapAcceptor#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg; //channel为NioSocketChannel
    child.pipeline().addLast(childHandler); //childHandler为处理客户端的ChannelInitializer
    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);
    try {
      //将NioSocketChannel和NioEventLoop绑定
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

## 注册Channel

io.netty.channel.MultithreadEventLoopGroup#register(io.netty.channel.Channel)

```java
public ChannelFuture register(Channel channel) {
    //将NioSocketChannel注册到NioEventLoop中的Selector上
    return next().register(channel);
}
```

io.netty.channel.AbstractChannel.AbstractUnsafe#register0

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
      	//注册到selector
        doRegister();
        neverRegistered = false;
        registered = true;

        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
      	//注册自定义的ChannelInitializer到NioSocketChannel
        pipeline.fireChannelRegistered();
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive(); //连接建立成功
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
          //注册到selector上
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```

## 调用自定义的ChannelInitializer

将自定义的ChannelHandler加入到SocketChannel的ChannelPipeline

io.netty.channel.ChannelInitializer#channelRegistered

```java
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    if (initChannel(ctx)) {
        ctx.pipeline().fireChannelRegistered();
        removeState(ctx);
    } else {
        ctx.fireChannelRegistered();
    }
}
```

```java
public void initChannel(SocketChannel ch) throws Exception {
    ChannelPipeline p = ch.pipeline(); //SocketChannel的ChannelPipeline
    if (sslCtx != null) {
        p.addLast(sslCtx.newHandler(ch.alloc()));
    }
    p.addLast(serverHandler);
}
```

# 如何读取数据?

为什么不是一次性把通道中的数据全部读完，而是要循环读取？

为了避免单个通道占用太多时间，导致其他连接没有机会去读取数据

所以Netty 会限制在一次读事件处理过程中调用底层读取 API 的次数，这个次数默认为 16 次

io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read

```java
public final void read() {
        final ChannelConfig config = config();
        if (shouldBreakReadReady(config)) {
            clearReadPending();
            return;
        }
        final ChannelPipeline pipeline = pipeline();
        //ByteBuf分配器，一般是池化的分配器
        final ByteBufAllocator allocator = config.getAllocator();
        //默认AdaptiveRecvByteBufAllocator
        final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
        allocHandle.reset(config);

        ByteBuf byteBuf = null;
        boolean close = false;
        try {
            do {
                //尽可能分配合适的大小：guess
                byteBuf = allocHandle.allocate(allocator);
                //读并且记录读了多少，如果读满了，下次continue的话就直接扩容。
                allocHandle.lastBytesRead(doReadBytes(byteBuf));
                if (allocHandle.lastBytesRead() <= 0) {
                    // nothing was read. release the buffer.
                    byteBuf.release();
                    byteBuf = null;
                    close = allocHandle.lastBytesRead() < 0; //表明发生了IO异常，关闭连接 
                    if (close) {
                        // There is nothing left to read as we received an EOF.
                        readPending = false;
                    }
                    break;
                }
								//增加读取的次数，默认16
                allocHandle.incMessagesRead(1);
                readPending = false;
                //pipeline上执行，解码、业务逻辑的处理就在这个地方
                pipeline.fireChannelRead(byteBuf);
                byteBuf = null;
            } while (allocHandle.continueReading());

            //记录这次读事件总共读了多少数据，计算下次分配大小。
            allocHandle.readComplete();
            //相当于完成本次读事件的处理
            pipeline.fireChannelReadComplete();
            if (close) {
                closeOnRead(pipeline);
            }
        } catch (Throwable t) {
            handleReadException(pipeline, byteBuf, t, close, allocHandle);
        } finally {
            if (!readPending && !config.isAutoRead()) {
               removeReadOp();
            }
        }
    }
}
```

ByteBufAllocator ：分配器

RecvByteBufAllocator：数据接收分配器默认AdaptiveRecvByteBufAllocator

## 自适应数据大小的分配器

io.netty.channel.AdaptiveRecvByteBufAllocator

初始化

```java
//最小分配
static final int DEFAULT_MINIMUM = 64;
//初始分配
static final int DEFAULT_INITIAL = 1024;
//最大分配
static final int DEFAULT_MAXIMUM = 65536;
//增长步长
private static final int INDEX_INCREMENT = 4;
//缩减步长
private static final int INDEX_DECREMENT = 1;
private static final int[] SIZE_TABLE;
static {
    List<Integer> sizeTable = new ArrayList<Integer>();
    //16、32、48、64、80...496:  + 16
    //小于512时，增加64，减小16
    for (int i = 16; i < 512; i += 16) {
        sizeTable.add(i);
    }
    //512、512*2、512*4、512*8、512*16直到整形最大值,
    //大于512时，16倍增大，1倍减小
    //后面会判断最大，最小限制，默认在64和65536之间。
    for (int i = 512; i > 0; i <<= 1) {
        sizeTable.add(i);
    }
    SIZE_TABLE = new int[sizeTable.size()];
    for (int i = 0; i < SIZE_TABLE.length; i ++) {
        SIZE_TABLE[i] = sizeTable.get(i);
    }
}
```

```java
public AdaptiveRecvByteBufAllocator(int minimum, int initial, int maximum) {
    checkPositive(minimum, "minimum");
    if (initial < minimum) {
        throw new IllegalArgumentException("initial: " + initial);
    }
    if (maximum < initial) {
        throw new IllegalArgumentException("maximum: " + maximum);
    }
    //控制size table区间最小值minIndex
    int minIndex = getSizeTableIndex(minimum);
    if (SIZE_TABLE[minIndex] < minimum) {
        this.minIndex = minIndex + 1;
    } else {
        this.minIndex = minIndex;
    }
    //控制size table区间最大值maxIndex
    int maxIndex = getSizeTableIndex(maximum);
    if (SIZE_TABLE[maxIndex] > maximum) {
        this.maxIndex = maxIndex - 1;
    } else {
        this.maxIndex = maxIndex;
    }
    this.initial = initial;
}
```

## 创建Handle

根据读取的字节数，计算下次分配的大小

io.netty.channel.AbstractChannel.AbstractUnsafe#recvBufAllocHandle

```java
public RecvByteBufAllocator.Handle recvBufAllocHandle() {
    if (recvHandle == null) {
        recvHandle = config().getRecvByteBufAllocator().newHandle();
    }
    return recvHandle;
}
```

```java
public Handle newHandle() {
    return new HandleImpl(minIndex, maxIndex, initial);
}
```

```java
HandleImpl(int minIndex, int maxIndex, int initial) {
    this.minIndex = minIndex;
    this.maxIndex = maxIndex;

    index = getSizeTableIndex(initial);
    //初始值
    nextReceiveBufferSize = SIZE_TABLE[index];
}
```

## 分配bytebuf

io.netty.channel.DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle#allocate

```java
public ByteBuf allocate(ByteBufAllocator alloc) {
    return alloc.ioBuffer(guess());
}
```

```java
public int guess() {
     return nextReceiveBufferSize; //初始1024
}
```

## 读取数据

io.netty.channel.socket.nio.NioSocketChannel#doReadBytes

```java
protected int doReadBytes(ByteBuf byteBuf) throws Exception {
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
```

## 记录读取字节数

io.netty.channel.AdaptiveRecvByteBufAllocator.HandleImpl#lastBytesRead

```java
public void lastBytesRead(int bytes) {
    //判断是否预估的空间都被“读”到的数据填满了，如果填满了，尝试扩容再试试。
    if (bytes == attemptedBytesRead()) {
        record(bytes);
    }
    super.lastBytesRead(bytes);
}
```

```
接受数据buffer的容量会尽可能的足够大以接受数据,也尽可能的小以不会浪费它的空间
```

```java
private void record(int actualReadBytes) {
    //尝试是否可以减小分配的空间仍然能满足需求：
    //尝试方法：当前实际读取的size是否小于或等于打算缩小的尺寸
    if (actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT - 1)]) {
        //decreaseNow: 连续2次尝试减小都可以
        if (decreaseNow) {
            //减小
            index = max(index - INDEX_DECREMENT, minIndex);
            nextReceiveBufferSize = SIZE_TABLE[index];
            decreaseNow = false;
        } else {
            decreaseNow = true;
        }
    //判断是否实际读取的数据大于等于预估的，如果是，尝试扩容
    } else if (actualReadBytes >= nextReceiveBufferSize) {
        index = min(index + INDEX_INCREMENT, maxIndex);
        nextReceiveBufferSize = SIZE_TABLE[index];
        decreaseNow = false;
    }
}
```

```java
public void lastBytesRead(int bytes) {
    lastBytesRead = bytes; //上次读取字节数
    if (bytes > 0) {
        totalBytesRead += bytes; //总共读取的字节数
    }
}
```

## 触发ChannelRead

自定义的Handler读取数据，执行业务逻辑

## 判断是否可以继续读

```java
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead() &&
            //respectMaybeMoreData,默认为true，会判断有更多数据可以读取
            (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
           totalMessages < maxMessagePerRead 
      		 && totalBytesRead > 0;
}
```

## 触发ChannelReadComplete

io.netty.channel.DefaultChannelPipeline.HeadContext#channelReadComplete

```java
public void channelReadComplete(ChannelHandlerContext ctx) {
    ctx.fireChannelReadComplete();
    readIfIsAutoRead(); 
}
private void readIfIsAutoRead() {
    if (channel.config().isAutoRead()) {
        channel.read(); ////该方法最终会调用Channel的read方法，注册读事件
    }
}
```

# 如何发送数据？

## 写数据三种方式

write：写到ChannelOutboundBuffer

flush：将ChannelOutboundBuffer中的数据发送出去

writeAndFlush：写到ChannelOutboundBuffer，立即发送

**channelHandlerContext.channel().write() :从 TailContext 开始执行;**

**channelHandlerContext.write() : 从当前的 Context 开始。**

## 写数据入口

io.netty.channel.AbstractChannelHandlerContext#write(java.lang.Object)

```java
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}
```

```java
public ChannelPromise newPromise() {
    return new DefaultChannelPromise(channel(), executor());
}
```

```java
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
    write(msg, false, promise);
    return promise;
}
```

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ObjectUtil.checkNotNull(msg, "msg");
    try {
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }
    final AbstractChannelHandlerContext next = findContextOutbound(flush ?
            (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
    //引用计数用的，用来检测内存泄漏
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        final AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        if (!safeExecute(executor, task, promise, m)) {
            task.cancel();
        }
    }
}
```

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}
```

## 写数据到buffer

io.netty.channel.AbstractChannel.AbstractUnsafe#write

```java
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {//channel已经关闭了
        safeSetFailure(promise, newClosedChannelException(initialCloseCause));
        ReferenceCountUtil.release(msg); //释放资源，防止资源泄露
        return;
    }
    int size;
    try {
        //将msg不是Direct类型，就转换
        msg = filterOutboundMessage(msg);
        //计算数据占用的字节
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }
    //存放到outboundBuffer
    outboundBuffer.addMessage(msg, size, promise);
}
```

### 将数据转成DirectBuffer

io.netty.channel.nio.AbstractNioByteChannel#filterOutboundMessage

```java
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }
        return newDirectBuffer(buf);
    }
    if (msg instanceof FileRegion) {
        return msg;
    }
    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```

```java
protected final ByteBuf newDirectBuffer(ByteBuf buf) {
    final int readableBytes = buf.readableBytes();
    //无数据
    if (readableBytes == 0) {
        ReferenceCountUtil.safeRelease(buf);
        return Unpooled.EMPTY_BUFFER;
    }
    //由ByteBuf分配器分配ByteBuf
    final ByteBufAllocator alloc = alloc();
    if (alloc.isDirectBufferPooled()) { //池化的DirectBuffer
        ByteBuf directBuf = alloc.directBuffer(readableBytes);
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);//释放原来的bytebuf
        return directBuf;
    }
    //从Recycler对象池中获取ByteBuf
    final ByteBuf directBuf = ByteBufUtil.threadLocalDirectBuffer();
    if (directBuf != null) {
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);//释放原来的bytebuf
        return directBuf;
    }
    // 分配和释放一个没有池化的直接缓冲区是非常昂贵的，直接返回原Bytebuf
    return buf;
}
```

### 计算占用的字节

io.netty.channel.DefaultMessageSizeEstimator.HandleImpl#size

```java
public int size(Object msg) {
    if (msg instanceof ByteBuf) {
        return ((ByteBuf) msg).readableBytes();
    }
    if (msg instanceof ByteBufHolder) {
        return ((ByteBufHolder) msg).content().readableBytes();
    }
    if (msg instanceof FileRegion) {
        return 0;
    }
    return unknownSize;
}
```

### 存放到缓冲区

io.netty.channel.ChannelOutboundBuffer#addMessage

```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    //创建Entry，从RECYCLER中获取
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    //追加到末尾
    tailEntry = entry;
    if (unflushedEntry == null) {//未刷新
        unflushedEntry = entry;
    }
    //增加未发送的字节数
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```

### 计算累积的数据量

```java
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }
    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
    //判断待发送的数据的size是否高于高水位线
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        //设置为不可写
        setUnwritable(invokeLater);
    }
}
```

### 超过高水位线设置不可写

```java
private void setUnwritable(boolean invokeLater) {
    for (;;) {
        final int oldValue = unwritable;
        final int newValue = oldValue | 1;
        //设置unwritable
        if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
            if (oldValue == 0 && newValue != 0) {
                fireChannelWritabilityChanged(invokeLater);
            }
            break;
        }
    }
}
```

## Flush

io.netty.channel.AbstractChannel.AbstractUnsafe#flush

```java
public final void flush() {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    //表明channel关闭了。
    if (outboundBuffer == null) {
        return;
    }
    outboundBuffer.addFlush();
    flush0();
}
```

```java
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) { //有未刷新的数据
        if (flushedEntry == null) { //还未刷新
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) { //promise已经被删除
                //释放msg占用的空间、减少累积的bytes，设置channel立即可写
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);
        unflushedEntry = null;
    }
}
```

```java
protected void flush0() {
    if (inFlush0) { //正在刷新中
        return;
    }
    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }
    inFlush0 = true; //标志正在刷新中
    if (!isActive()) { //连接已经关闭
        try {
            if (isOpen()) {
                outboundBuffer.failFlushed(new NotYetConnectedException(), true);
            } else {
                outboundBuffer.failFlushed(newClosedChannelException(initialCloseCause), false);
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }
    try {
        //将缓冲的数据发送出去
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        if (t instanceof IOException && config().isAutoClose()) {
            initialCloseCause = t;
            close(voidPromise(), t, newClosedChannelException(t), false);
        } else {
            try {
                shutdownOutput(voidPromise(), t);
            } catch (Throwable t2) {
                initialCloseCause = t;
                close(voidPromise(), t2, newClosedChannelException(t), false);
            }
        }
    } finally {
        inFlush0 = false;
    }
}
```



io.netty.channel.nio.AbstractNioByteChannel#doWrite

```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    //默认写16
    int writeSpinCount = config().getWriteSpinCount();
    do {
        Object msg = in.current();
        if (msg == null) {   //写完所有的数据
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }
        writeSpinCount -= doWriteInternal(in, msg);
    } while (writeSpinCount > 0);
    incompleteWrite(writeSpinCount < 0);
}
```

io.netty.channel.nio.AbstractNioByteChannel#doWriteInternal

```java
private int doWriteInternal(ChannelOutboundBuffer in, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (!buf.isReadable()) {
            in.remove();
            return 0;
        }
        final int localFlushedAmount = doWriteBytes(buf);
        if (localFlushedAmount > 0) {
            in.progress(localFlushedAmount);
            if (!buf.isReadable()) {
                in.remove(); //移除发送的数据
            }
            return 1;
        }
    } else if (msg instanceof FileRegion) {
        FileRegion region = (FileRegion) msg;
        if (region.transferred() >= region.count()) {
            in.remove();
            return 0;
        }
        long localFlushedAmount = doWriteFileRegion(region);
        if (localFlushedAmount > 0) {
            in.progress(localFlushedAmount);
            if (region.transferred() >= region.count()) {
                in.remove();
            }
            return 1;
        }
    } else {
        throw new Error();
    }
    return WRITE_STATUS_SNDBUF_FULL;
}
```

io.netty.channel.ChannelOutboundBuffer#remove()

```java
public boolean remove() {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;
    ChannelPromise promise = e.promise;
    int size = e.pendingSize;
    //修改flushedEntry链表，将entry.next赋值给flushedEntry
    removeEntry(e);
    if (!e.cancelled) {
        // 释放空间.
        ReferenceCountUtil.safeRelease(msg);
        //设置数据发送成功
        safeSuccess(promise);
        //减少积压的bytes
        decrementPendingOutboundBytes(size, false, true);
    }
    //放回对象池
    e.recycle();
    return true;
}
```

# 如何优化SelectorImpl？

```java
//是否对SelectorImpl的属性selectedKeys、publicKeys使用的数据结构进行优化，默认false进行优化
private static final boolean DISABLE_KEY_SET_OPTIMIZATION =
        SystemPropertyUtil.getBoolean("io.netty.noKeySetOptimization", false);
```

io.netty.channel.nio.NioEventLoop#openSelector

```java
private SelectorTuple openSelector() {
    final Selector unwrappedSelector;
    try {
        unwrappedSelector = provider.openSelector();
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }

    if (DISABLE_KEY_SET_OPTIMIZATION) { //不进行优化
        return new SelectorTuple(unwrappedSelector);
    }
		//获取SelectorImpl对象
    Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
            try {
                return Class.forName(
                        "sun.nio.ch.SelectorImpl",
                        false,
                        PlatformDependent.getSystemClassLoader());
            } catch (Throwable cause) {
                return cause;
            }
        }
    });
	
   
    if (!(maybeSelectorImplClass instanceof Class) ||
        // ensure the current selector implementation is what we can instrument.
        !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
        if (maybeSelectorImplClass instanceof Throwable) {
            Throwable t = (Throwable) maybeSelectorImplClass;
            logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
        }
        return new SelectorTuple(unwrappedSelector);
    }

    final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
  
  	//替换的实现类，内部通过数组存储SelectionKey对象
    final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

    Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
            try {
              	//通过反射获取属性，类型都是Set，底层通过HashMap实现
                Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
              
                Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                    // Let us try to use sun.misc.Unsafe to replace the SelectionKeySet.
                    // This allows us to also do this in Java9+ without any extra flags.
                  	//通过unsafe替换属性的实现，获取属性的偏移量
                    long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                    long publicSelectedKeysFieldOffset =
                            PlatformDependent.objectFieldOffset(publicSelectedKeysField);
										//selectedKeys、publicKeys都能找到
                    if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {										//开始替换
                        PlatformDependent.putObject(
                                unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                        PlatformDependent.putObject(
                                unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                        return null;
                    }
                    // We could not retrieve the offset, lets try reflection as last-resort.
                }
								//通过反射替换成新的实现
                Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                if (cause != null) {
                    return cause;
                }
                cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                if (cause != null) {
                    return cause;
                }

                selectedKeysField.set(unwrappedSelector, selectedKeySet);
                publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                return null;
            } catch (NoSuchFieldException e) {
                return e;
            } catch (IllegalAccessException e) {
                return e;
            }
        }
    });

    if (maybeException instanceof Exception) {
        selectedKeys = null;
        Exception e = (Exception) maybeException;
        logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
        return new SelectorTuple(unwrappedSelector);
    }
    selectedKeys = selectedKeySet;
    logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
    return new SelectorTuple(unwrappedSelector,
                             new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
}
```



# 如何自动重建selector？

```java
private static final int MIN_PREMATURE_SELECTOR_RETURNS = 3; 
private static final int SELECTOR_AUTO_REBUILD_THRESHOLD;
```

```java
int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512); //默认512，空轮询超过此值，自动重建Selector
if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
    selectorAutoRebuildThreshold = 0;
}

SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;
```

io.netty.channel.nio.NioEventLoop#selectRebuildSelector

```java
private Selector selectRebuildSelector(int selectCnt) throws IOException {//开始重建
    logger.warn(
            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
            selectCnt, selector);

    rebuildSelector(); //重建
    Selector selector = this.selector;

    // Select again to populate selectedKeys.
    selector.selectNow();
    return selector;
}
```

io.netty.channel.nio.NioEventLoop#rebuildSelector

```java
public void rebuildSelector() {
    if (!inEventLoop()) { //如果是其他线程发起的rebuildSelector，将rebuildSelector封装成Task，放到taskQueue中
        execute(new Runnable() {
            @Override
            public void run() {
                rebuildSelector0();
            }
        });
        return;
    }
    rebuildSelector0();
}
```

```java
private void rebuildSelector0() {
    final Selector oldSelector = selector;
    final SelectorTuple newSelectorTuple;

    if (oldSelector == null) {
        return;
    }

    try {
        //创建新的Seclector
        newSelectorTuple = openSelector(); 
    } catch (Exception e) {
        logger.warn("Failed to create a new Selector.", e);
        return;
    }

  	//注册所有的channnel到新的Selector
    // Register all channels to the new Selector.
    int nChannels = 0;
    for (SelectionKey key: oldSelector.keys()) {
        Object a = key.attachment();
        try {
            if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                continue;
            }

            int interestOps = key.interestOps();
            key.cancel();
            SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
            if (a instanceof AbstractNioChannel) {
                // Update SelectionKey
                ((AbstractNioChannel) a).selectionKey = newKey;
            }
            nChannels ++;
        } catch (Exception e) {
            logger.warn("Failed to re-register a Channel to the new Selector.", e);
            if (a instanceof AbstractNioChannel) {
                AbstractNioChannel ch = (AbstractNioChannel) a;
                ch.unsafe().close(ch.unsafe().voidPromise());
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                invokeChannelUnregistered(task, key, e);
            }
        }
    }

    selector = newSelectorTuple.selector;
    unwrappedSelector = newSelectorTuple.unwrappedSelector;

    try {
        oldSelector.close(); //关闭旧的Selector
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("Failed to close the old Selector.", t);
        }
    }

    if (logger.isInfoEnabled()) {
        logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
    }
}
```



# 如何处理事件？

iorate参数用来平衡NioEventLoop处理IO任务和非IO任务的时间。范围1-100,默认50.说明在处理IO上和处理非IO任务花费的时间相同。

io.netty.channel.nio.NioEventLoop#run

```java
protected void run() {
    for (;;) {
        try {
            try {
              	//selectStrategy：控制select循环
                //如果有事件需要立即处理，select可能会延迟也可能会跳过
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:

                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                }
            } catch (IOException e) {
                // 重建selector
                rebuildSelector0();
                handleLoopException(e);
                continue;
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) { //不会权衡io和非io任务的处理时间
                try {
                    processSelectedKeys(); //先处理IO任务
                } finally {
                    runAllTasks(); //后处理非io任务
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();//先处理IO任务
                } finally {
                    //根据处理io任务的时间，计算处理非io任务的时间
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

## 计算策略

io.netty.channel.DefaultSelectStrategy#calculateStrategy

```java
public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {		//如果有任务，执行非阻塞的select，否则执行阻塞的select
    return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT; 
}
```

```java
private final IntSupplier selectNowSupplier = new IntSupplier() {
    @Override
    public int get() throws Exception {
        return selectNow();
    }
};
```

io.netty.channel.nio.NioEventLoop#selectNow

```java
int selectNow() throws IOException {
    try {
        return selector.selectNow();
    } finally {
        if (wakenUp.get()) {
            selector.wakeup();
        }
    }
}
```

io.netty.channel.nio.NioEventLoop#select

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
      	//统计select触发次数
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        //计算定时任务的触发时间
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        long normalizedDeadlineNanos = selectDeadLineNanos - initialNanoTime();
        if (nextWakeupTime != normalizedDeadlineNanos) {
            nextWakeupTime = normalizedDeadlineNanos;
        }

        for (;;) {
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) { //定时任务超时，直接退出循环
                if (selectCnt == 0) { //尚未执行一次select操作，则执行非阻塞的select
                    //非阻塞，没有数据返回0
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;
            }
          
            //如果没有定时任务执行阻塞1s
            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                break;
            }
            if (Thread.interrupted()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) { //触发空轮询次数的阈值
                selector = selectRebuildSelector(selectCnt); //重新构建selector
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                        selectCnt - 1, selector);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                    selector, e);
        }
    }
}
```

## 处理IO事件

io.netty.channel.nio.NioEventLoop#processSelectedKeys

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {  //不用JDK的selector.selectedKeys()
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

io.netty.channel.nio.NioEventLoop#processSelectedKeysOptimized

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
      
        selectedKeys.keys[i] = null;

        //selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) { //需要再次执行select
            selectedKeys.reset(i + 1);
            selectAgain();
            i = -1;
        }
    }
}
```

io.netty.channel.nio.NioEventLoop#processSelectedKey(java.nio.channels.SelectionKey, io.netty.channel.nio.AbstractNioChannel)

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) { //SelectionKey无效
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        unsafe.close(unsafe.voidPromise()); //关闭channel
        return;
    }

    try {
        int readyOps = k.readyOps();
       
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) { //连接
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            unsafe.finishConnect();
        }

        if ((readyOps & SelectionKey.OP_WRITE) != 0) { //写
            ch.unsafe().forceFlush();
        }

        //处理读请求（断开连接）或接入连接
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

## 处理非IO任务

io.netty.util.concurrent.SingleThreadEventExecutor#runAllTasks(long)

```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();//将超时的定时任务放入任务队列
    Runnable task = pollTask(); //从任务队列获取任务
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }
    //计算非io任务执行结束的时间
    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task); //执行任务

        runTasks ++;

        if ((runTasks & 0x3F) == 0) { //每执行64次任务，判断是否结束执行非io任务
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask(); //获取任务
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

io.netty.util.concurrent.SingleThreadEventExecutor#fetchFromScheduledTaskQueue

```java
private boolean fetchFromScheduledTaskQueue() {
    if (scheduledTaskQueue == null || scheduledTaskQueue.isEmpty()) {
        return true;
    }
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    for (;;) {
      	//从定时任务队列获取等待超时的任务
        Runnable scheduledTask = pollScheduledTask(nanoTime);
        if (scheduledTask == null) {
            return true;
        }
        //将定时任务放入任务队列
        if (!taskQueue.offer(scheduledTask)) {
            scheduledTaskQueue.add((ScheduledFutureTask<?>) scheduledTask);//放回定时任务队列
            return false;
        }
    }
}
```





# DefaultPromise

```java
private static final AtomicReferenceFieldUpdater<DefaultPromise, Object> RESULT_UPDATER =
        AtomicReferenceFieldUpdater.newUpdater(DefaultPromise.class, Object.class, "result"); //并发下修改result
private volatile Object result;
```

io.netty.util.concurrent.DefaultPromise#setSuccess

```java
public Promise<V> setSuccess(V result) {
    if (setSuccess0(result)) {
        return this;
    }
    throw new IllegalStateException("complete already: " + this);
}
```

```java
private boolean setSuccess0(V result) {
    return setValue0(result == null ? SUCCESS : result);
}
```

```java
private boolean setValue0(Object objResult) {
    if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
        RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {//CAS修改结果
        if (checkNotifyWaiters()) { 
            notifyListeners(); //触发监听器
        }
        return true;
    }
    return false;
}
```

```java
private synchronized boolean checkNotifyWaiters() {
    if (waiters > 0) {
        notifyAll();
    }
    return listeners != null; //设置的监听器不为空
}
```

io.netty.util.concurrent.DefaultPromise#notifyListeners

```java
private void notifyListeners() {
    EventExecutor executor = executor();
    if (executor.inEventLoop()) {//在NioEventLoop中
        final InternalThreadLocalMap threadLocals = InternalThreadLocalMap.get();
        final int stackDepth = threadLocals.futureListenerStackDepth();
        if (stackDepth < MAX_LISTENER_STACK_DEPTH) {
            threadLocals.setFutureListenerStackDepth(stackDepth + 1);
            try {
                notifyListenersNow();
            } finally {
                threadLocals.setFutureListenerStackDepth(stackDepth);
            }
            return;
        }
    }

    safeExecute(executor, new Runnable() {
        @Override
        public void run() {
            notifyListenersNow();
        }
    });
}
```

io.netty.util.concurrent.DefaultPromise#notifyListenersNow

```java
private void notifyListenersNow() {
    Object listeners;
    synchronized (this) {
        if (notifyingListeners || this.listeners == null) { //已经触发或者尚未注册监听器
            return;
        }
        notifyingListeners = true;  
        listeners = this.listeners;
        this.listeners = null;
    }
    for (;;) {
        if (listeners instanceof DefaultFutureListeners) {
            notifyListeners0((DefaultFutureListeners) listeners);
        } else {
            notifyListener0(this, (GenericFutureListener<?>) listeners);
        }
        synchronized (this) {
            if (this.listeners == null) {
                // Nothing can throw from within this method, so setting notifyingListeners back to false does not
                // need to be in a finally block.
                notifyingListeners = false;
                return;
            }
            listeners = this.listeners;
            this.listeners = null;
        }
    }
}
```

io.netty.util.concurrent.DefaultPromise#sync

```java
public Promise<V> sync() throws InterruptedException {
    await();
    rethrowIfFailed();
    return this;
}
```

```java
public Promise<V> await() throws InterruptedException {
    if (isDone()) { //是否已经完成
        return this;
    }

    if (Thread.interrupted()) { //线程被中断
        throw new InterruptedException(toString());
    }

    checkDeadLock(); //检查死锁

    synchronized (this) {
        while (!isDone()) { //循环判断，防止线程被意外唤醒导致的功能异常
            incWaiters(); //增加waiter，统计waiter数量，防止waiter数量过多
            try {
                wait();
            } finally {
                decWaiters(); //减少waiter
            }
        }
    }
    return this;
}
```

# 如何检测连接有效性？

当有业务消息时，无须心跳检测

当链路空闲即无数据读、写时，才会触发心跳检测

## 初始化

io.netty.handler.timeout.IdleStateHandler#IdleStateHandler(boolean, long, long, long, java.util.concurrent.TimeUnit)

```java
public IdleStateHandler(boolean observeOutput,
        long readerIdleTime, long writerIdleTime, long allIdleTime,
        TimeUnit unit) {
    ObjectUtil.checkNotNull(unit, "unit");

 		//写空闲触发时，是否考虑字节的消耗，默认false
    this.observeOutput = observeOutput; 

  	//计算读空闲时间
    if (readerIdleTime <= 0) {
        readerIdleTimeNanos = 0;
    } else {
        readerIdleTimeNanos = Math.max(unit.toNanos(readerIdleTime), MIN_TIMEOUT_NANOS);
    }
    
    //计算写空闲时间
    if (writerIdleTime <= 0) {
        writerIdleTimeNanos = 0;
    } else {
        writerIdleTimeNanos = Math.max(unit.toNanos(writerIdleTime), MIN_TIMEOUT_NANOS);
    }
  	
    //计算读写空闲时间
    if (allIdleTime <= 0) {
        allIdleTimeNanos = 0;
    } else {
        allIdleTimeNanos = Math.max(unit.toNanos(allIdleTime), MIN_TIMEOUT_NANOS);
    }
}
```

## handlerAdded

在执行pipeline.addLast方法时，调用此方法

io.netty.handler.timeout.IdleStateHandler#handlerAdded

```java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isActive() && ctx.channel().isRegistered()) {
        initialize(ctx);
    } else {
    }
}
```

```java
private void initialize(ChannelHandlerContext ctx) {
    switch (state) {
    case 1: //已经初始化
    case 2: //已经被销毁
        return;
    }

    state = 1;
    initOutputChanged(ctx);
  	//上次读写时间
    lastReadTime = lastWriteTime = ticksInNanos();
    //创建读写定时任务，定时任务由channel对应的NioEventLoop处理
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS); //读空闲
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS); //写空闲
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS); //读写空闲
    }
}
```

io.netty.handler.timeout.IdleStateHandler#initOutputChanged

```java
private void initOutputChanged(ChannelHandlerContext ctx) {
    if (observeOutput) { //默认false
        Channel channel = ctx.channel();
        Unsafe unsafe = channel.unsafe();
        ChannelOutboundBuffer buf = unsafe.outboundBuffer();

        if (buf != null) {
          	//计算当前正在写的数据的hashcode
            lastMessageHashCode = System.identityHashCode(buf.current());
            //尚未flush的字节数
            lastPendingWriteBytes = buf.totalPendingWriteBytes();
            //当前flush进度
            lastFlushProgress = buf.currentProgress();
        }
    }
}
```

## 空闲任务

### ReaderIdleTimeoutTask

io.netty.handler.timeout.IdleStateHandler.ReaderIdleTimeoutTask#run

```java
protected void run(ChannelHandlerContext ctx) { //读空闲
    long nextDelay = readerIdleTimeNanos;
    if (!reading) {
        nextDelay -= ticksInNanos() - lastReadTime;
    }

    if (nextDelay <= 0) { //触发读空闲
        //再次创建定时任务
        readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

        boolean first = firstReaderIdleEvent; //第一次触发为true
        firstReaderIdleEvent = false;

        try {
          	//创建空闲事件
            IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
          	//向后传递空闲事件
            channelIdle(ctx, event);
        } catch (Throwable t) {
            ctx.fireExceptionCaught(t);
        }
    } else {
        // 未触发读空闲。再次创建定时任务
        readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}
```

### WriterIdleTimeoutTask

io.netty.handler.timeout.IdleStateHandler.WriterIdleTimeoutTask#run

```java
protected void run(ChannelHandlerContext ctx) { //写空闲

    long lastWriteTime = IdleStateHandler.this.lastWriteTime;
    long nextDelay = writerIdleTimeNanos - (ticksInNanos() - lastWriteTime);
    if (nextDelay <= 0) { //触发写空闲
        //创建新的定时任务
        writerIdleTimeout = schedule(ctx, this, writerIdleTimeNanos, TimeUnit.NANOSECONDS);

        boolean first = firstWriterIdleEvent;
        firstWriterIdleEvent = false;

        try {
            if (hasOutputChanged(ctx, first)) { //ChannelOutboundBuffer有变化，返回true
                return;
            }
						//传递写空闲事件
            IdleStateEvent event = newIdleStateEvent(IdleState.WRITER_IDLE, first);
            channelIdle(ctx, event);
        } catch (Throwable t) {
            ctx.fireExceptionCaught(t);
        }
    } else {  //创建新的定时任务
        writerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}
```

io.netty.handler.timeout.IdleStateHandler#hasOutputChanged

```java
private boolean hasOutputChanged(ChannelHandlerContext ctx, boolean first) {
    if (observeOutput) { //默认false，ChannelOutboundBuffer有变化，返回true，不会触发写空闲事件

        if (lastChangeCheckTimeStamp != lastWriteTime) {
            lastChangeCheckTimeStamp = lastWriteTime;
          
            if (!first) {
                return true;
            }
        }

        Channel channel = ctx.channel();
        Unsafe unsafe = channel.unsafe();
        ChannelOutboundBuffer buf = unsafe.outboundBuffer();

        if (buf != null) {
            int messageHashCode = System.identityHashCode(buf.current());
            long pendingWriteBytes = buf.totalPendingWriteBytes();

            if (messageHashCode != lastMessageHashCode || pendingWriteBytes != lastPendingWriteBytes) {
                lastMessageHashCode = messageHashCode;
                lastPendingWriteBytes = pendingWriteBytes;

                if (!first) {
                    return true;
                }
            }

            long flushProgress = buf.currentProgress();
            if (flushProgress != lastFlushProgress) {
                lastFlushProgress = flushProgress;

                if (!first) {
                    return true;
                }
            }
        }
    }

    return false;
}
```

### AllIdleTimeoutTask

io.netty.handler.timeout.IdleStateHandler.AllIdleTimeoutTask#run

```java
protected void run(ChannelHandlerContext ctx) { //读写空闲

    long nextDelay = allIdleTimeNanos;
    if (!reading) {
        nextDelay -= ticksInNanos() - Math.max(lastReadTime, lastWriteTime);
    }
    if (nextDelay <= 0) {
        //创建新的定时任务
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
    } else { //创建新的定时任务
        allIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}
```

```java
protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    ctx.fireUserEventTriggered(evt);
}
```

# 如何增强写吞吐？

writeAndFlush是加急写，会触发系统调用，直接将数据发送给对端。writeAndFlus通常开销很大

因此,在大多数情况下(写延迟可以与吞吐量进行权衡)，尽量减少刷新操作。

多次write之后，再调用flush

FlushConsolidationHandler，继承自ChannelDuplexHandler，既是ChannelInboundHandler也是ChannelOutboundHandler

io.netty.handler.flush.FlushConsolidationHandler#channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {//读开始
    readInProgress = true; //标志正在读
    ctx.fireChannelRead(msg);
}
```

io.netty.handler.flush.FlushConsolidationHandler#channelReadComplete

```java
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {//读结束
    // This may be the last event in the read loop, so flush now!
  //这可能是在读循环中的最后一个事件，立刻调用flush操作
    resetReadAndFlushIfNeeded(ctx);
    ctx.fireChannelReadComplete();
}
```

```java
private void resetReadAndFlushIfNeeded(ChannelHandlerContext ctx) {
    readInProgress = false; //标志读结束
    flushIfNeeded(ctx);
}
```

io.netty.handler.flush.FlushConsolidationHandler#flushIfNeeded

```java
private void flushIfNeeded(ChannelHandlerContext ctx) {
    if (flushPendingCount > 0) { //有未刷新的写，立即刷新出去
        flushNow(ctx);
    }
}
```

io.netty.handler.flush.FlushConsolidationHandler#flush

```java
public void flush(ChannelHandlerContext ctx) throws Exception {
    //调用channelread时，设置为true
    //调用channelreadcomplete时，设置为false
    if (readInProgress) { //正在读
        if (++flushPendingCount == explicitFlushAfterFlushes) { //累积超过阈值，刷新
            flushNow(ctx);
        }
    } else if (consolidateWhenNoReadInProgress) { //当没有读的时候也需合并
        if (++flushPendingCount == explicitFlushAfterFlushes) {
            flushNow(ctx);//达到阈值立即刷新
        } else {
            scheduleFlush(ctx); //后台异步执行
        }
    } else {
        flushNow(ctx); //总是立即刷新
    }
}
```

# 如何实现流量整形？

## Channel级别

### 消息读取的流量整形

io.netty.handler.traffic.AbstractTrafficShapingHandler#channelRead

```java
public void channelRead(final ChannelHandlerContext ctx, final Object msg) throws Exception {
  	//计算读取的字节大小
    long size = calculateSize(msg);
    long now = TrafficCounter.milliSecondFromNano();
    if (size > 0) {
        //计算读取等待的时间（10ms-15s）
        long wait = trafficCounter.readTimeToWait(size, readLimit, maxTime, now);
        wait = checkWaitReadTime(ctx, wait, now);
        if (wait >= MINIMAL_WAIT) { // 最少等待10ms
            Channel channel = ctx.channel();
            ChannelConfig config = channel.config();
            if (logger.isDebugEnabled()) {
                logger.debug("Read suspend: " + wait + ':' + config.isAutoRead() + ':'
                        + isHandlerActive(ctx));
            }
            if (config.isAutoRead() && isHandlerActive(ctx)) {
                config.setAutoRead(false); //设置为不可读
                channel.attr(READ_SUSPENDED).set(true); //读取挂起
                Attribute<Runnable> attr = channel.attr(REOPEN_TASK);
                Runnable reopenTask = attr.get();
                if (reopenTask == null) {
                    reopenTask = new ReopenReadTimerTask(ctx);
                    attr.set(reopenTask);
                }
                ctx.executor().schedule(reopenTask, wait, TimeUnit.MILLISECONDS);
                if (logger.isDebugEnabled()) {
                    logger.debug("Suspend final status => " + config.isAutoRead() + ':'
                            + isHandlerActive(ctx) + " will reopened at: " + wait);
                }
            }
        }
    }
    informReadOperation(ctx, now);
    ctx.fireChannelRead(msg);
}
```

### 消息发送流量整形

io.netty.handler.traffic.AbstractTrafficShapingHandler#write

```java
public void write(final ChannelHandlerContext ctx, final Object msg, final ChannelPromise promise)
        throws Exception {
    //计算需要发送的bytebuf的大小
    long size = calculateSize(msg);
    long now = TrafficCounter.milliSecondFromNano();
    if (size > 0) {
        //计算等待时间
        long wait = trafficCounter.writeTimeToWait(size, writeLimit, maxTime, now);
        if (wait >= MINIMAL_WAIT) { //等待时间超过10ms，通过定时任务进行消息发送
            if (logger.isDebugEnabled()) {
                logger.debug("Write suspend: " + wait + ':' + ctx.channel().config().isAutoRead() + ':'
                        + isHandlerActive(ctx));
            }
            submitWrite(ctx, msg, size, wait, now, promise);
            return;
        }
    }
    //等待时间小于10ms，封装成Tosend任务，添加到NioEventLoop的task队列立即执行
    submitWrite(ctx, msg, size, 0, now, promise);
}
```

```java
void submitWrite(final ChannelHandlerContext ctx, final Object msg,
        final long size, final long delay, final long now,
        final ChannelPromise promise) {
    final ToSend newToSend;
    synchronized (this) {
        //立即发送
        if (delay == 0 && messagesQueue.isEmpty()) {
            trafficCounter.bytesRealWriteFlowControl(size);
            ctx.write(msg, promise);
            return;
        }
        //封装成ToSend，添加到messagesQueue队列
        newToSend = new ToSend(delay + now, msg, promise);
        messagesQueue.addLast(newToSend);
        queueSize += size;
        checkWriteSuspend(ctx, delay, queueSize);
    }
    //定时任务进行消息发送
    final long futureNow = newToSend.relativeTimeAction;
    ctx.executor().schedule(new Runnable() {
        @Override
        public void run() {
            sendAllValid(ctx, futureNow);
        }
    }, delay, TimeUnit.MILLISECONDS);
}
```

```java
private void sendAllValid(final ChannelHandlerContext ctx, final long now) {
    // write order control
    synchronized (this) {
        ToSend newToSend = messagesQueue.pollFirst();
        for (; newToSend != null; newToSend = messagesQueue.pollFirst()) {
            if (newToSend.relativeTimeAction <= now) {
                long size = calculateSize(newToSend.toSend);
                trafficCounter.bytesRealWriteFlowControl(size);
                queueSize -= size;
                ctx.write(newToSend.toSend, newToSend.promise);
            } else {//尚未到达执行时间
                messagesQueue.addFirst(newToSend);
                break;
            }
        }
        if (messagesQueue.isEmpty()) {
            releaseWriteSuspended(ctx);
        }
    }
    ctx.flush();
}
```

## 全局流量整形

```
GlobalTrafficShapingHandler全局共享，标有Sharable注解
```

# 如何实现更快的ThreadLocal？

Netty自定义了FastThreadLocalThread继承JDK中的Thread，用属性InternalThreadLocalMap来存放ThreadLocal，存放的结构是一个数组，每次创建ThreadLocal都会分配一个索引，当线程从ThreadLocal获取值时，先获取ThreadLocal的索引，再根据索引，从数组中获取数据

## 初始化

io.netty.util.concurrent.FastThreadLocal#FastThreadLocal

```java
public FastThreadLocal() {
    index = InternalThreadLocalMap.nextVariableIndex();//每个FastThreadLocal分配唯一标志
}
```

```java
public static int nextVariableIndex() {
    int index = nextIndex.getAndIncrement(); 
    if (index < 0) {
        nextIndex.decrementAndGet();
        throw new IllegalStateException("too many thread-local indexed variables");
    }
    return index;
}
```

## 设置值

io.netty.util.concurrent.FastThreadLocal#set(V)

```java
public final void set(V value) {
    if (value != InternalThreadLocalMap.UNSET) {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        setKnownNotUnset(threadLocalMap, value);
    } else {
        remove();
    }
}
```

```java
public static InternalThreadLocalMap get() {
    Thread thread = Thread.currentThread();
    //判断线程类型
    if (thread instanceof FastThreadLocalThread) {
        return fastGet((FastThreadLocalThread) thread);
    } else {
        return slowGet();
    }
}
```

### 获取线程私有的本地数组

```java
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if (threadLocalMap == null) {
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}
```

### 创建线程私有的本地数组

```java
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    //创建InternalThreadLocalMap
    if (threadLocalMap == null) {
      	//绑定线程
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}
```

### 赋值

```java
private void setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
    if (threadLocalMap.setIndexedVariable(index, value)) {
        addToVariablesToRemove(threadLocalMap, this);
    }
}
```

index为FastThreadLocal的唯一标志，每个线程的本地数组indexedVariables的下标对应FastThreadLocal的标志

```java
public boolean setIndexedVariable(int index, Object value) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object oldValue = lookup[index];
        lookup[index] = value; //赋值
        return oldValue == UNSET;
    } else { //扩容数组
        expandIndexedVariableTableAndSet(index, value);
        return true;
    }
}
```

## 获取值

io.netty.util.concurrent.FastThreadLocal#get()

```java
public final V get() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET) {
        return (V) v;
    }
    return initialize(threadLocalMap);
}
```

io.netty.util.internal.InternalThreadLocalMap#InternalThreadLocalMap

```java
private InternalThreadLocalMap() {
    super(newIndexedVariableTable());
}
```

```java
private static Object[] newIndexedVariableTable() {
    Object[] array = new Object[32];
    Arrays.fill(array, UNSET); //用UNSET填充数组
    return array;
}
```

```java
public Object indexedVariable(int index) { //根据FastThreadLocal标志获取对应的值
    Object[] lookup = indexedVariables;
    return index < lookup.length? lookup[index] : UNSET;
}
```

```java
private V initialize(InternalThreadLocalMap threadLocalMap) { //设置初始值
    V v = null;
    try {
        v = initialValue();
    } catch (Exception e) {
        PlatformDependent.throwException(e);
    }
    threadLocalMap.setIndexedVariable(index, v);
    addToVariablesToRemove(threadLocalMap, this);
    return v;
}
```

slowGet

```java
private static InternalThreadLocalMap slowGet() {
    ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
    InternalThreadLocalMap ret = slowThreadLocalMap.get(); //从JDK的ThreadLocal中获取
  	//InternalThreadLocalMap根据索引存放每个ThreadLocal的值
    if (ret == null) {
        ret = new InternalThreadLocalMap();
        slowThreadLocalMap.set(ret); //将InternalThreadLocalMap放到JDK的ThreadLocal中
    }
    return ret;
}
```

io.netty.util.internal.UnpaddedInternalThreadLocalMap

```java
static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<InternalThreadLocalMap>();
```



# 如何实现时间轮？

处理延迟任务的时间轮，延迟任务的新增和删除都是O(1)的复杂度，只需一个线程就可以驱动时间轮进行工作。对延迟任务的真正删除会延迟执行

## 初始化

io.netty.util.HashedWheelTimer#HashedWheelTimer(java.util.concurrent.ThreadFactory, long, java.util.concurrent.TimeUnit, int, boolean, long)

```java
public HashedWheelTimer(
        ThreadFactory threadFactory,
        long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection,
        long maxPendingTimeouts) {

    ObjectUtil.checkNotNull(threadFactory, "threadFactory");
    ObjectUtil.checkNotNull(unit, "unit");
    ObjectUtil.checkPositive(tickDuration, "tickDuration");
    ObjectUtil.checkPositive(ticksPerWheel, "ticksPerWheel");
    //默认512
    wheel = createWheel(ticksPerWheel);
    //用于快速取模
    mask = wheel.length - 1;
    //转换成功纳秒处理
    long duration = unit.toNanos(tickDuration);
    if (duration >= Long.MAX_VALUE / wheel.length) {
        throw new IllegalArgumentException(String.format(
                "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                tickDuration, Long.MAX_VALUE / wheel.length));
    }
    if (duration < MILLISECOND_NANOS) {
        logger.warn("Configured tickDuration {} smaller then {}, using 1ms.",
                    tickDuration, MILLISECOND_NANOS);
        this.tickDuration = MILLISECOND_NANOS;
    } else {
        this.tickDuration = duration;
    }
    //创建工作线程
    workerThread = threadFactory.newThread(worker);
    //内存泄露检测
    leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;
    //默认无限制
    this.maxPendingTimeouts = maxPendingTimeouts;
    //避免创建太多的HashedWheelTimer
    if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
        WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
        reportTooManyInstances();
    }
}
```

```java
private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
    if (ticksPerWheel <= 0) {
        throw new IllegalArgumentException(
                "ticksPerWheel must be greater than 0: " + ticksPerWheel);
    }
    if (ticksPerWheel > 1073741824) {
        throw new IllegalArgumentException(
                "ticksPerWheel may not be greater than 2^30: " + ticksPerWheel);
    }
    //数组长度为2的次方
    ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
    //创建时间轮数组
    HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
    for (int i = 0; i < wheel.length; i ++) {
        wheel[i] = new HashedWheelBucket();
    }
    return wheel;
}
```

## 添加任务

io.netty.util.HashedWheelTimer#newTimeout

```java
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {//时间复杂度O(1)
    ObjectUtil.checkNotNull(task, "task");
    ObjectUtil.checkNotNull(unit, "unit");
    //任务数
    long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
    if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
        pendingTimeouts.decrementAndGet();
        throw new RejectedExecutionException("Number of pending timeouts ("
            + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
            + "timeouts (" + maxPendingTimeouts + ")");
    }
    start();//如果work线程还没启动，需要启动
  	//startTime在worker线程启动时设置
    //计算任务的deadline
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    if (delay > 0 && deadline < 0) {
        deadline = Long.MAX_VALUE;
    }
    //创建HashedWheelTimeout，封装定时任务
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    //添加到MpscQueue（多生产者单消费者队列）
    timeouts.add(timeout);
    return timeout;
}
```

## 删除任务

io.netty.util.HashedWheelTimer.HashedWheelTimeout#cancel

```java
public boolean cancel() { //时间复杂度O(1)
    if (!compareAndSetState(ST_INIT, ST_CANCELLED)) {//修改任务的状态为删除状态,只修改一次
        return false;
    }
  	//实际业务中，会有多个线程往时间轮中添加、删除事件，但是只有一个线程负责执行事件
    //添加到删除队列（多生产者单消费者队列，尽可能减少锁的负载）
 	 	//延迟删除，至少等待一个执行周期。避免了多个线程同时进行删除时对锁的争夺
    timer.cancelledTimeouts.add(this);
    return true;
}
```

## 启动工作线程

io.netty.util.HashedWheelTimer#start

```java
public void start() {
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT: //初始状态
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                workerThread.start(); //启动线程
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }
    //阻塞直到开始时间被工作线程初始化
    while (startTime == 0) {
        try {
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
        }
    }
}
```

## 工作线程运行流程

io.netty.util.HashedWheelTimer.Worker#run

```java
public void run() {
    // 初始话开始时间
    startTime = System.nanoTime();
    if (startTime == 0) {
        startTime = 1;
    }
    //唤醒等待worker启动的线程
    startTimeInitialized.countDown();
    do {
        //计算下次tick的时间，并sleep到下次tick
        final long deadline = waitForNextTick();
        if (deadline > 0) {
            //获取当前tick在HashedWheelBucket数组中的对应的下标
            int idx = (int) (tick & mask);
            //移除被取消的任务
            processCancelledTasks();
            HashedWheelBucket bucket =
                    wheel[idx];
            //从MpscQueue中取出任务，加入到对应的slot
            transferTimeoutsToBuckets();
            //执行过期的任务
            bucket.expireTimeouts(deadline);
            tick++;
        }
    } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

    // Fill the unprocessedTimeouts so we can return them from stop() method.
    for (HashedWheelBucket bucket: wheel) {
        bucket.clearTimeouts(unprocessedTimeouts);
    }
    for (;;) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
            break;
        }
        if (!timeout.isCancelled()) {
            unprocessedTimeouts.add(timeout);
        }
    }
    processCancelledTasks();
}
```

### 移除被删除的任务

io.netty.util.HashedWheelTimer.Worker#processCancelledTasks

```java
private void processCancelledTasks() {
    for (;;) {
        HashedWheelTimeout timeout = cancelledTimeouts.poll();
        if (timeout == null) {
            break;
        }
        try {
            timeout.remove();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("An exception was thrown while process a cancellation task", t);
            }
        }
    }
}
```

### 添加任务到时间轮

io.netty.util.HashedWheelTimer.Worker#transferTimeoutsToBuckets

```java
private void transferTimeoutsToBuckets() {
    for (int i = 0; i < 100000; i++) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
            break;
        }
        if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) { //已经删除
            continue;
        }
        //计算任务需要经过多少个tick
        long calculated = timeout.deadline / tickDuration;
        //计算任务需要在时间轮中经历的圈数
        timeout.remainingRounds = (calculated - tick) / wheel.length;
        final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
        int stopIndex = (int) (ticks & mask); //计算wheel中的下标
        HashedWheelBucket bucket = wheel[stopIndex];
        bucket.addTimeout(timeout); //添加HashedWheelTimeout到对应的HashedWheelBucket
    }
}
```

### 执行过期任务

io.netty.util.HashedWheelTimer.HashedWheelBucket#expireTimeouts

```java
public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;
    while (timeout != null) {
        HashedWheelTimeout next = timeout.next;
        //经历的圈数已经耗尽
        if (timeout.remainingRounds <= 0) {
            next = remove(timeout); //从双向链表中移除
            if (timeout.deadline <= deadline) { //到期
                timeout.expire(); //执行到期事件
            } else {
                throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
            }
        } else if (timeout.isCancelled()) { //如果被删除，从双向链表中移除
            next = remove(timeout);
        } else { //圈数减1
            timeout.remainingRounds --;
        }
        timeout = next;
    }
}
```

# 如何实现对象的复用？

对象池，实现对象的复用，避免对象频繁的创建与销毁

## 对象的获取

当 Stack 中 elements 有数据时，直接从栈顶弹出

当 Stack 中 elements 没有数据时，尝试从 WeakOrderQueue 中回收一个 Link 包含的对象实例到 Stack 中，然后从栈顶弹出

io.netty.util.Recycler#get

```java
public final T get() {
    if (maxCapacityPerThread == 0) { //容量为0
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    Stack<T> stack = threadLocal.get();//获取当前线程缓存的Stack
    DefaultHandle<T> handle = stack.pop(); //从stack获取DefaultHandle
    if (handle == null) {//没有可用对象
        handle = stack.newHandle();//创建新的Handle
        handle.value = newObject(handle);//创建的对象，并和Handle绑定
    }
    return (T) handle.value;
}
```

io.netty.util.Recycler.Stack#pop

```java
DefaultHandle<T> pop() {
    int size = this.size;
    if (size == 0) { //stack无可用的对象
        if (!scavenge()) { //尝试从其他线程回收的对象中转移一些到此stack
            return null;
        }
        size = this.size;
        if (size <= 0) {
            // double check, avoid races
            return null;
        }
    }
    //stack有可用对象
    size --;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    this.size = size;
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    return ret;
}
```

io.netty.util.Recycler.Stack#scavenge

```java
private boolean scavenge() {
    if (scavengeSome()) {
        return true;
    }
    prev = null;
    cursor = head;
    return false;
}
```

```java
private boolean scavengeSome() {
    WeakOrderQueue prev;
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        prev = null;
        cursor = head;
        if (cursor == null) {
            return false;
        }
    } else {
        prev = this.prev;
    }
    boolean success = false;
    do {
        // 尝试迁移WeakOrderQueue中部分对象实例到 Stack 中
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
        WeakOrderQueue next = cursor.getNext();
        if (cursor.get() == null) { //关联此queue的线程已退出
            if (cursor.hasFinalData()) {
                for (;;) {
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }
            //将已退出的线程从WeakOrderQueue链表中移除
            if (prev != null) {
                cursor.reclaimAllSpaceAndUnlink();
                prev.setNext(next);
            }
        } else {
            prev = cursor;
        }
        cursor = next;
    } while (cursor != null && !success);
    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```

## 对象的回收

io.netty.util.Recycler.DefaultHandle#recycle

```java
public void recycle(Object object) { //回收对象
    if (object != value) { //回收的对象和之前绑定的对象不是同一个
        throw new IllegalArgumentException("object does not belong to handle");
    }
    Stack<?> stack = this.stack;
    if (lastRecycledId != recycleId || stack == null) {
        throw new IllegalStateException("recycled already");
    }
    stack.push(this); //重新放入栈中
}
```

io.netty.util.Recycler.Stack#push

```java
void push(DefaultHandle<?> item) { //压栈
    Thread currentThread = Thread.currentThread();
    if (threadRef.get() == currentThread) { //同线程回收
        pushNow(item);
    } else { //异线程回收
        pushLater(item, currentThread);
    }
}
```

### 同线程回收

```java
private void pushNow(DefaultHandle<?> item) {
    //防止多次回收
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;
    int size = this.size;
    //不能超出最大容量、控制回收速率
    if (size >= maxCapacity || dropHandle(item)) {
        return;
    }
    if (size == elements.length) { //扩容
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }
    elements[size] = item;
    this.size = size + 1;
}
```

控制回收速率

```java
boolean dropHandle(DefaultHandle<?> handle) {
    if (!handle.hasBeenRecycled) { //还未被回收
        if (handleRecycleCount < interval) {
            handleRecycleCount++;
            // Drop the object.
            return true;
        }
        //超过interval，会对hanle回收
        handleRecycleCount = 0;
        handle.hasBeenRecycled = true; //被回收
    }
    return false;
}
```

### 跨线程回收

io.netty.util.Recycler.Stack#pushLater

```java
private void pushLater(DefaultHandle<?> item, Thread thread) {
    if (maxDelayedQueues == 0) { //不支持跨线程回收，默认2倍CPU核数
        return;
    }
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    WeakOrderQueue queue = delayedRecycled.get(this);
    if (queue == null) {
        //最多回收2倍CPU核数
        if (delayedRecycled.size() >= maxDelayedQueues) {
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);  //无法帮助此stack回收对象
            return;
        }
        //创建WeakOrderQueue，检测是否达到为此stack回收对象的最大值，超过最大值，不再为stack回收对象
        if ((queue = newWeakOrderQueue(thread)) == null) {
            return;
        }
        delayedRecycled.put(this, queue);
    } else if (queue == WeakOrderQueue.DUMMY) { //不为此stack回收对象
        return;
    }
    //添加对象到WeakOrderQueue的Link链表中
    queue.add(item);
}
```

io.netty.util.Recycler.Stack#newWeakOrderQueue

```java
private WeakOrderQueue newWeakOrderQueue(Thread thread) {
    return WeakOrderQueue.newQueue(this, thread);
}
```

io.netty.util.Recycler.WeakOrderQueue#newQueue

```java
static WeakOrderQueue newQueue(Stack<?> stack, Thread thread) {
    if (!Head.reserveSpaceForLink(stack.availableSharedCapacity)) {
        return null;
    }
    final WeakOrderQueue queue = new WeakOrderQueue(stack, thread);
    //创建的WeakOrderQueue与stack绑定
    stack.setHead(queue);
    return queue;
}
```

io.netty.util.Recycler.Stack#setHead

```java
synchronized void setHead(WeakOrderQueue queue) {
    queue.setNext(head);
    head = queue;
}
```

```java
void add(DefaultHandle<?> handle) {
    handle.lastRecycledId = id;
    if (handleRecycleCount < interval) { //控制回收速率
        handleRecycleCount++;
        return;
    }
    handleRecycleCount = 0;
    Link tail = this.tail;
    int writeIndex;
    if ((writeIndex = tail.get()) == LINK_CAPACITY) { //链表尾部的Link写满
        Link link = head.newLink(); //创建Link，追加到链表尾部
        if (link == null) {
            return;
        }
        this.tail = tail = tail.next = link;
        writeIndex = tail.get();
    }
    tail.elements[writeIndex] = handle;
    handle.stack = null;
    tail.lazySet(writeIndex + 1);
}
```

# 如何实现更快的队列？

主要在NioEventLoop和HashedWheelTimer中使用到

通过大量填充long类型的变量解决伪共享的问题

数组的容量为2的次幂，可以通过位运算方便计算出数组的下标

大量使用CAS

入队操作中引入了producerLimit，减少了主动获取consumerIndex的次数，提升了性能

io.netty.channel.nio.NioEventLoop#newTaskQueue(io.netty.channel.EventLoopTaskQueueFactory)

```java
private static Queue<Runnable> newTaskQueue(
        EventLoopTaskQueueFactory queueFactory) {
  	//使用JCTools下的集合类，针对不同的生产消费场景进行了优化
    if (queueFactory == null) {
        return newTaskQueue0(DEFAULT_MAX_PENDING_TASKS);
    }
    return queueFactory.newTaskQueue(DEFAULT_MAX_PENDING_TASKS);
}
```

io.netty.channel.nio.NioEventLoop#newTaskQueue0

```java
private static Queue<Runnable> newTaskQueue0(int maxPendingTasks) {
  
    return maxPendingTasks == Integer.MAX_VALUE ? PlatformDependent.<Runnable>newMpscQueue()//无界
            : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks); //有界
}
```

## 多生产者单消费的无界队列

io.netty.util.internal.PlatformDependent#newMpscQueue()

```java
public static <T> Queue<T> newMpscQueue() { //无界
    return Mpsc.newMpscQueue();
}
```

```java
static <T> Queue<T> newMpscQueue() {
    return USE_MPSC_CHUNKED_ARRAY_QUEUE ? new MpscUnboundedArrayQueue<T>(MPSC_CHUNK_SIZE)
                                        : new MpscUnboundedAtomicArrayQueue<T>(MPSC_CHUNK_SIZE);
}
```

## 多生产者单消费的有界队列

io.netty.util.internal.PlatformDependent#newMpscQueue(int)

```java
public static <T> Queue<T> newMpscQueue(final int maxCapacity) { //有界
    return Mpsc.newMpscQueue(maxCapacity);
}
```

```java
static <T> Queue<T> newMpscQueue(final int maxCapacity) {
    // Calculate the max capacity which can not be bigger then MAX_ALLOWED_MPSC_CAPACITY.
    // This is forced by the MpscChunkedArrayQueue implementation as will try to round it
    // up to the next power of two and so will overflow otherwise.
    final int capacity = max(min(maxCapacity, MAX_ALLOWED_MPSC_CAPACITY), MIN_MAX_MPSC_CAPACITY);
    return USE_MPSC_CHUNKED_ARRAY_QUEUE ? new MpscChunkedArrayQueue<T>(MPSC_CHUNK_SIZE, capacity)
                                        : new MpscGrowableAtomicArrayQueue<T>(MPSC_CHUNK_SIZE, capacity);
}
```

# HTTP2

## 协议升级

### 客户端

HttpClientUpgradeHandler

io.netty.handler.codec.http.HttpClientUpgradeHandler#write

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)
        throws Exception { //首次写，请求头添加与http2相关的信息，比如upgrade
    if (!(msg instanceof HttpRequest)) {
        ctx.write(msg, promise);
        return;
    }


    if (upgradeRequested) {
        promise.setFailure(new IllegalStateException(
                "Attempting to write HTTP request with upgrade in progress"));
        return;
    }
		//默认false，第一次发送数据时，将器设置为true
    upgradeRequested = true; 
  	//设置升级协议所必须的请求头
    setUpgradeRequestHeaders(ctx, (HttpRequest) msg);

    // Continue writing the request.
    ctx.write(msg, promise);

    // Notify that the upgrade request was issued.
    ctx.fireUserEventTriggered(UpgradeEvent.UPGRADE_ISSUED);
    // Now we wait for the next HTTP response to see if we switch protocols.
}
```

io.netty.handler.codec.http.HttpClientUpgradeHandler#setUpgradeRequestHeaders

```java
private void setUpgradeRequestHeaders(ChannelHandlerContext ctx, HttpRequest request) {//设置升级请求头
    // 设置upgrade
    request.headers().set(HttpHeaderNames.UPGRADE, upgradeCodec.protocol());

    // Add all protocol-specific headers to the request.
    Set<CharSequence> connectionParts = new LinkedHashSet<CharSequence>(2);
    connectionParts.addAll(upgradeCodec.setUpgradeHeaders(ctx, request));

    // Set the CONNECTION header from the set of all protocol-specific headers that were added.
    StringBuilder builder = new StringBuilder();
    for (CharSequence part : connectionParts) {
        builder.append(part);
        builder.append(',');
    }
    builder.append(HttpHeaderValues.UPGRADE);
  	//设置CONNECTION
    request.headers().add(HttpHeaderNames.CONNECTION, builder.toString());
}
```

接收server返回的升级响应

io.netty.handler.codec.http.HttpClientUpgradeHandler#decode

```java
protected void decode(ChannelHandlerContext ctx, HttpObject msg, List<Object> out)
        throws Exception {
    FullHttpResponse response = null;
    try {
        if (!upgradeRequested) {
            throw new IllegalStateException("Read HTTP response without requesting protocol switch");
        }

        if (msg instanceof HttpResponse) {
            HttpResponse rep = (HttpResponse) msg;
            if (!SWITCHING_PROTOCOLS.equals(rep.status())) {
                // The server does not support the requested protocol, just remove this handler
                // and continue processing HTTP.
                // NOTE: not releasing the response since we're letting it propagate to the
                // next handler.
                ctx.fireUserEventTriggered(UpgradeEvent.UPGRADE_REJECTED);
                removeThisHandler(ctx);
                ctx.fireChannelRead(msg);
                return;
            }
        }

        if (msg instanceof FullHttpResponse) {
            response = (FullHttpResponse) msg;
            // Need to retain since the base class will release after returning from this method.
            response.retain();
            out.add(response);
        } else {
            // Call the base class to handle the aggregation of the full request.
            super.decode(ctx, msg, out);
            if (out.isEmpty()) {
                // The full request hasn't been created yet, still awaiting more data.
                return;
            }

            assert out.size() == 1;
            response = (FullHttpResponse) out.get(0);
        }
			  //响应信息的头部必须要有UPGRADE
        CharSequence upgradeHeader = response.headers().get(HttpHeaderNames.UPGRADE);
        if (upgradeHeader != null && !AsciiString.contentEqualsIgnoreCase(upgradeCodec.protocol(), upgradeHeader)) {
            throw new IllegalStateException(
                    "Switching Protocols response with unexpected UPGRADE protocol: " + upgradeHeader);
        }

        // Upgrade to the new protocol.
        sourceCodec.prepareUpgradeFrom(ctx); //准备从http升级到http2
        upgradeCodec.upgradeTo(ctx, response); //添加http2相关的handler

        // Notify that the upgrade to the new protocol completed successfully.
        ctx.fireUserEventTriggered(UpgradeEvent.UPGRADE_SUCCESSFUL);

        // We guarantee UPGRADE_SUCCESSFUL event will be arrived at the next handler
        // before http2 setting frame and http response.
        sourceCodec.upgradeFrom(ctx);

        // We switched protocols, so we're done with the upgrade response.
        // Release it and clear it from the output.
        response.release();
        out.clear();
        removeThisHandler(ctx);
    } catch (Throwable t) {
        release(response);
        ctx.fireExceptionCaught(t);
        removeThisHandler(ctx);
    }
}
```

io.netty.handler.codec.http2.Http2ClientUpgradeCodec#upgradeTo

```java
public void upgradeTo(ChannelHandlerContext ctx, FullHttpResponse upgradeResponse)
    throws Exception {
    try {
      
        //添加handler，触发handlerAdd方法,发送connection preface（连接前言）
        ctx.pipeline().addAfter(ctx.name(), handlerName, upgradeToHandler);

        if (http2MultiplexHandler != null) {
            final String name = ctx.pipeline().context(connectionHandler).name();
            ctx.pipeline().addAfter(name, null, http2MultiplexHandler);
        }

        // Reserve local stream 1 for the response.
        connectionHandler.onHttpClientUpgrade();
    } catch (Http2Exception e) {
        ctx.fireExceptionCaught(e);
        ctx.close();
    }
}
```

io.netty.handler.codec.http2.Http2ConnectionHandler.PrefaceDecoder#sendPreface

```java
private void sendPreface(ChannelHandlerContext ctx) throws Exception {
    if (prefaceSent || !ctx.channel().isActive()) {
        return;
    }

    prefaceSent = true;
		//必须是客户端
    final boolean isClient = !connection().isServer();
    if (isClient) { //发送连接前言
        // Clients must send the preface string as the first bytes on the connection.
      ctx.write(connectionPrefaceBuf()).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
    }

    // Both client and server must send their initial settings.
    encoder.writeSettings(ctx, initialSettings, ctx.newPromise()).addListener(
            ChannelFutureListener.CLOSE_ON_FAILURE);

    if (isClient) {
        userEventTriggered(ctx,Http2ConnectionPrefaceAndSettingsFrameWrittenEvent.INSTANCE);
    }
}
```

io.netty.handler.codec.http2.Http2ConnectionHandler#onHttpClientUpgrade

```java
public void onHttpClientUpgrade() throws Http2Exception {
    if (connection().isServer()) {
        throw connectionError(PROTOCOL_ERROR, "Client-side HTTP upgrade requested for a server");
    }
    if (!prefaceSent()) {
        // If the preface was not sent yet it most likely means the handler was not added to the pipeline before
        // calling this method.
        throw connectionError(INTERNAL_ERROR, "HTTP upgrade must occur after preface was sent");
    }
    if (decoder.prefaceReceived()) {
        throw connectionError(PROTOCOL_ERROR, "HTTP upgrade must occur before HTTP/2 preface is received");
    }

    // Create a local stream used for the HTTP cleartext upgrade.
    connection().local().createStream(HTTP_UPGRADE_STREAM_ID, true);
}
```

### 服务端

io.netty.handler.codec.http.HttpServerUpgradeHandler#decode

```java
protected void decode(ChannelHandlerContext ctx, HttpObject msg, List<Object> out)
        throws Exception {
    // Determine if we're already handling an upgrade request or just starting a new one.
    handlingUpgrade |= isUpgradeRequest(msg); //协议升级请求，请求头包含UPGRADE
    if (!handlingUpgrade) {
        // Not handling an upgrade request, just pass it to the next handler.
        ReferenceCountUtil.retain(msg);
        out.add(msg);
        return;
    }

    FullHttpRequest fullRequest;
    if (msg instanceof FullHttpRequest) {
        fullRequest = (FullHttpRequest) msg;
        ReferenceCountUtil.retain(msg);
        out.add(msg);
    } else {
        // Call the base class to handle the aggregation of the full request.
        super.decode(ctx, msg, out);
        if (out.isEmpty()) {
            // The full request hasn't been created yet, still awaiting more data.
            return;
        }

        // Finished aggregating the full request, get it from the output list.
        assert out.size() == 1;
        handlingUpgrade = false;
        fullRequest = (FullHttpRequest) out.get(0);
    }

    if (upgrade(ctx, fullRequest)) { //升级
        // The upgrade was successful, remove the message from the output list
        // so that it's not propagated to the next handler. This request will
        // be propagated as a user event instead.
        out.clear(); //清空
    }

    // The upgrade did not succeed, just allow the full request to propagate to the
    // next handler.
}

/**
```

io.netty.handler.codec.http.HttpServerUpgradeHandler#upgrade

```java
private boolean upgrade(final ChannelHandlerContext ctx, final FullHttpRequest request) {
    // Select the best protocol based on those requested in the UPGRADE header.
    final List<CharSequence> requestedProtocols = splitHeader(request.headers().get(HttpHeaderNames.UPGRADE));
    final int numRequestedProtocols = requestedProtocols.size();
    UpgradeCodec upgradeCodec = null;
    CharSequence upgradeProtocol = null;
    for (int i = 0; i < numRequestedProtocols; i ++) {
        final CharSequence p = requestedProtocols.get(i);
        final UpgradeCodec c = upgradeCodecFactory.newUpgradeCodec(p);
        if (c != null) {
            upgradeProtocol = p; //升级协议(明文的h2c，密文的h2)
            upgradeCodec = c;
            break;
        }
    }

    if (upgradeCodec == null) { // 没有获取到客户端指定的升级协议
        return false;
    }

    // Make sure the CONNECTION header is present.
    List<String> connectionHeaderValues = request.headers().getAll(HttpHeaderNames.CONNECTION);
    if (connectionHeaderValues == null) {
        return false;
    }
    final StringBuilder concatenatedConnectionValue = new StringBuilder(connectionHeaderValues.size() * 10);
    for (CharSequence connectionHeaderValue : connectionHeaderValues) {
        concatenatedConnectionValue.append(connectionHeaderValue).append(COMMA);
    }
    concatenatedConnectionValue.setLength(concatenatedConnectionValue.length() - 1);
  
    // CONNECTION header包含UPGRADE和必要的header
    Collection<CharSequence> requiredHeaders = upgradeCodec.requiredUpgradeHeaders();
    List<CharSequence> values = splitHeader(concatenatedConnectionValue);
    if (!containsContentEqualsIgnoreCase(values, HttpHeaderNames.UPGRADE) ||
            !containsAllContentEqualsIgnoreCase(values, requiredHeaders)) {
        return false;
    }
  
    //请求头必须包含必要的header
    for (CharSequence requiredHeader : requiredHeaders) {
        if (!request.headers().contains(requiredHeader)) {
            return false;
        }
    }
   
		//创建协议升级的响应
    final FullHttpResponse upgradeResponse = createUpgradeResponse(upgradeProtocol); 
  
    //请求头必须有HTTP2-Settings
    if (!upgradeCodec.prepareUpgradeResponse(ctx, request, upgradeResponse.headers())) { 
        return false;
    }

     //创建协议升级事件
    final UpgradeEvent event = new UpgradeEvent(upgradeProtocol, request); 

    // After writing the upgrade response we immediately prepare the
    // pipeline for the next protocol to avoid a race between completion
    // of the write future and receiving data before the pipeline is
    // restructured.
    try {
      	//返回协议升级的响应
        final ChannelFuture writeComplete = ctx.writeAndFlush(upgradeResponse);
      
        sourceCodec.upgradeFrom(ctx); //移除handler
        upgradeCodec.upgradeTo(ctx, request); //添加HTTP2相关的handler

        //从Pipeline中移除HttpServerUpgradeHandler
        ctx.pipeline().remove(HttpServerUpgradeHandler.this); 

        //增加请求的引用计数，传递UpgradeEvent协议升级事件
        ctx.fireUserEventTriggered(event.retain()); 
      
				//为升级响应注册监听器
        writeComplete.addListener(ChannelFutureListener.CLOSE_ON_FAILURE); 
    } finally {
        // Release the event if the upgrade event wasn't fired.
        event.release(); 
    }
    return true;
}
```

io.netty.handler.codec.http2.Http2ServerUpgradeCodec#upgradeTo

```java
public void upgradeTo(final ChannelHandlerContext ctx, FullHttpRequest upgradeRequest) {
    try {
       
      	//添加HTTP/2 connection handler,触发handlerAdded
        ctx.pipeline().addAfter(ctx.name(), handlerName, connectionHandler);

        // Add also all extra handlers as these may handle events / messages produced by the connectionHandler.
        // See https://github.com/netty/netty/issues/9314
        if (handlers != null) {
            final String name = ctx.pipeline().context(connectionHandler).name();
            for (int i = handlers.length - 1; i >= 0; i--) {
                ctx.pipeline().addAfter(name, null, handlers[i]);
            }
        }
        connectionHandler.onHttpServerUpgrade(settings);
    } catch (Http2Exception e) {
        ctx.fireExceptionCaught(e);
        ctx.close();
    }
}
```

io.netty.handler.codec.http2.Http2ConnectionHandler#handlerAdded

```java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    // Initialize the encoder, decoder, flow controllers, and internal state.
    encoder.lifecycleManager(this);
    decoder.lifecycleManager(this);
    encoder.flowController().channelHandlerContext(ctx);
    decoder.flowController().channelHandlerContext(ctx);
    byteDecoder = new PrefaceDecoder(ctx); //创建连接前言解码器
}
```

io.netty.handler.codec.http2.Http2ConnectionHandler.PrefaceDecoder#PrefaceDecoder

```java
PrefaceDecoder(ChannelHandlerContext ctx) throws Exception {
    clientPrefaceString = clientPrefaceString(encoder.connection());
    // This handler was just added to the context. In case it was handled after
    // the connection became active, send the connection preface now.
    sendPreface(ctx); //发送连接前言
}
```

io.netty.handler.codec.http2.Http2ConnectionHandler.PrefaceDecoder#sendPreface

```java
private void sendPreface(ChannelHandlerContext ctx) throws Exception {
    if (prefaceSent || !ctx.channel().isActive()) { //尚未发送连接前言
        return;
    }
   
		//标志连接前言已经发送
    prefaceSent = true;
	
  	//只有client端才会发送连接前言
    final boolean isClient = !connection().isServer();
    if (isClient) { 
      ctx.write(connectionPrefaceBuf()).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
    }
  
		//客户端和服务端都会发送初始settings
    encoder.writeSettings(ctx, initialSettings, ctx.newPromise()).addListener(
            ChannelFutureListener.CLOSE_ON_FAILURE);

    if (isClient) {
        userEventTriggered(ctx,Http2ConnectionPrefaceAndSettingsFrameWrittenEvent.INSTANCE);
    }
}
```

## 解码

io.netty.handler.codec.http2.Http2ConnectionHandler.PrefaceDecoder#decode

```java
public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    try {
        if (ctx.channel().isActive() && readClientPrefaceString(in) && verifyFirstFrameIsSettings(in)) {
            // After the preface is read, it is time to hand over control to the post initialized decoder.
            byteDecoder = new FrameDecoder();
            byteDecoder.decode(ctx, in, out);
        }
    } catch (Throwable e) {
        onError(ctx, false, e);
    }
}
```

io.netty.handler.codec.http2.Http2ConnectionHandler.FrameDecoder#decode

```java
public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    try {
        decoder.decodeFrame(ctx, in, out);
    } catch (Throwable e) {
        onError(ctx, false, e);
    }
}
```

io.netty.handler.codec.http2.DefaultHttp2ConnectionDecoder#decodeFrame

```java
public void decodeFrame(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Http2Exception {
    frameReader.readFrame(ctx, in, internalFrameListener);
}
```

io.netty.handler.codec.http2.DefaultHttp2FrameReader#readFrame

```java
public void readFrame(ChannelHandlerContext ctx, ByteBuf input, Http2FrameListener listener)
        throws Http2Exception { //读取数据
    if (readError) {
        input.skipBytes(input.readableBytes());
        return;
    }
    try {
        do {
            if (readingHeaders) {
                processHeaderState(input);
                if (readingHeaders) { //frame header尚未读取完整
                    // Wait until the entire header has arrived.
                    return;
                }
            }

            // The header is complete, fall into the next case to process the payload.
            // This is to ensure the proper handling of zero-length payloads. In this
            // case, we don't want to loop around because there may be no more data
            // available, causing us to exit the loop. Instead, we just want to perform
            // the first pass at payload processing now.
            processPayloadState(ctx, input, listener); //读取frame payload
            if (!readingHeaders) {
                // Wait until the entire payload has arrived.
                return;
            }
        } while (input.isReadable());
    } catch (Http2Exception e) {
        readError = !Http2Exception.isStreamError(e);
        throw e;
    } catch (RuntimeException e) {
        readError = true;
        throw e;
    } catch (Throwable cause) {
        readError = true;
        PlatformDependent.throwException(cause);
    }
}
```

读取FrameHeader

io.netty.handler.codec.http2.DefaultHttp2FrameReader#processHeaderState

```java
private void processHeaderState(ByteBuf in) throws Http2Exception { 
    if (in.readableBytes() < FRAME_HEADER_LENGTH) { // Http2报文头9个字节
        // Wait until the entire frame header has been read.
        return;
    }

    // Read the header and prepare the unmarshaller to read the frame.
    payloadLength = in.readUnsignedMedium(); //3个字节的数据长度
    if (payloadLength > maxFrameSize) {
        throw connectionError(FRAME_SIZE_ERROR, "Frame length: %d exceeds maximum: %d", payloadLength,
                              maxFrameSize);
    }
    frameType = in.readByte(); //帧类型
    flags = new Http2Flags(in.readUnsignedByte()); //标志位（END_HEADERS:表示头数据结束,END_STREAM:表示单方向数据发送结束）
    streamId = readUnsignedInt(in); //流标志符

    // We have consumed the data, next time we read we will be expecting to read the frame payload.
    readingHeaders = false; //frame header已经成功读取9字节

    switch (frameType) { //验证合法性
        case DATA:
            verifyDataFrame();
            break;
        case HEADERS:
            verifyHeadersFrame();
            break;
        case PRIORITY:
            verifyPriorityFrame();
            break;
        case RST_STREAM:
            verifyRstStreamFrame();
            break;
        case SETTINGS:
            verifySettingsFrame();
            break;
        case PUSH_PROMISE:
            verifyPushPromiseFrame();
            break;
        case PING:
            verifyPingFrame();
            break;
        case GO_AWAY:
            verifyGoAwayFrame();
            break;
        case WINDOW_UPDATE:
            verifyWindowUpdateFrame();
            break;
        case CONTINUATION:
            verifyContinuationFrame();
            break;
        default:
            // Unknown frame type, could be an extension.
            verifyUnknownFrame();
            break;
    }
}
```

读取FramePayload

io.netty.handler.codec.http2.DefaultHttp2FrameReader#processPayloadState

```java
private void processPayloadState(ChannelHandlerContext ctx, ByteBuf in, Http2FrameListener listener)
                throws Http2Exception {
    if (in.readableBytes() < payloadLength) { //payload不完整
        return;
    }

  	//可以读取完整数据
    // Only process up to payloadLength bytes.
    int payloadEndIndex = in.readerIndex() + payloadLength;

    readingHeaders = true; //下次开始读取FrameHeader

    // Read the payload and fire the frame event to the listener.
    switch (frameType) {
        case DATA:
            readDataFrame(ctx, in, payloadEndIndex, listener);
            break;
        case HEADERS:
            readHeadersFrame(ctx, in, payloadEndIndex, listener);
            break;
        case PRIORITY:
            readPriorityFrame(ctx, in, listener);
            break;
        case RST_STREAM:
            readRstStreamFrame(ctx, in, listener);
            break;
        case SETTINGS:
            readSettingsFrame(ctx, in, listener);
            break;
        case PUSH_PROMISE:
            readPushPromiseFrame(ctx, in, payloadEndIndex, listener);
            break;
        case PING:
            readPingFrame(ctx, in.readLong(), listener);
            break;
        case GO_AWAY:
            readGoAwayFrame(ctx, in, payloadEndIndex, listener);
            break;
        case WINDOW_UPDATE:
            readWindowUpdateFrame(ctx, in, listener);
            break;
        case CONTINUATION:
            readContinuationFrame(in, payloadEndIndex, listener);
            break;
        default:
            readUnknownFrame(ctx, in, payloadEndIndex, listener);
            break;
    }
    in.readerIndex(payloadEndIndex);
}
```

# 如何实现解码？

## FixedLengthFrameDecoder

固定长度，实现简单，但是会有空间的浪费

io.netty.handler.codec.FixedLengthFrameDecoder#FixedLengthFrameDecoder

```java
public FixedLengthFrameDecoder(int frameLength) {
    checkPositive(frameLength, "frameLength");
    this.frameLength = frameLength; //固定长度
}
```

io.netty.handler.codec.FixedLengthFrameDecoder#decode

```java
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in); //解码
    if (decoded != null) {
        out.add(decoded);
    }
}
```

```java
protected Object decode(
        @SuppressWarnings("UnusedParameters") ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    if (in.readableBytes() < frameLength) { //可读取的字节数少于固定长度
        return null;
    } else {
        return in.readRetainedSlice(frameLength); //读取完整数据
    }
}
```

## DelimiterBasedFrameDecoder

分隔符解码器，实现简单，需要扫描内容查找分隔符

io.netty.handler.codec.DelimiterBasedFrameDecoder#decode

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    if (lineBasedDecoder != null) {
        return lineBasedDecoder.decode(ctx, buffer);
    }
    // Try all delimiters and choose the delimiter which yields the shortest frame.
    int minFrameLength = Integer.MAX_VALUE;
    ByteBuf minDelim = null;
    for (ByteBuf delim: delimiters) {
        int frameLength = indexOf(buffer, delim);
        if (frameLength >= 0 && frameLength < minFrameLength) {
            minFrameLength = frameLength;
            minDelim = delim;
        }
    }

    if (minDelim != null) {
        int minDelimLength = minDelim.capacity();
        ByteBuf frame;

        if (discardingTooLongFrame) {
            // We've just finished discarding a very large frame.
            // Go back to the initial state.
            discardingTooLongFrame = false;
            buffer.skipBytes(minFrameLength + minDelimLength);

            int tooLongFrameLength = this.tooLongFrameLength;
            this.tooLongFrameLength = 0;
            if (!failFast) {
                fail(tooLongFrameLength);
            }
            return null;
        }

        if (minFrameLength > maxFrameLength) {
            // Discard read frame.
            buffer.skipBytes(minFrameLength + minDelimLength);
            fail(minFrameLength);
            return null;
        }

        if (stripDelimiter) {
            frame = buffer.readRetainedSlice(minFrameLength);
            buffer.skipBytes(minDelimLength);
        } else {
            frame = buffer.readRetainedSlice(minFrameLength + minDelimLength);
        }

        return frame;
    } else {
        if (!discardingTooLongFrame) {
            if (buffer.readableBytes() > maxFrameLength) {
                // Discard the content of the buffer until a delimiter is found.
                tooLongFrameLength = buffer.readableBytes();
                buffer.skipBytes(buffer.readableBytes());
                discardingTooLongFrame = true;
                if (failFast) {
                    fail(tooLongFrameLength);
                }
            }
        } else {
            // Still discarding the buffer since a delimiter is not found.
            tooLongFrameLength += buffer.readableBytes();
            buffer.skipBytes(buffer.readableBytes());
        }
        return null;
    }
}
```

# 总结

1、SelectorImpl优化，将Set类型的属性替换为数组实现的

2、ioRatio 默认50  平衡在NIoEventLoop中执行IO和非IO操作的时间，如果为50，两种操作花费的时间同样多，如果设置为100，不会再去平衡IO操作和非IO操作

3、Netty 的那些“锁”事

- 在意锁的对象和范围 -> 减少粒度	Synchronized method -> Synchronized block
- 注意锁的对象本身大小 -> 减少空间占用 AtomicLong -> Volatile long + AtomicLongFieldUpdater
- 注意锁的速度 -> 提高并发性
- 不同场景选择不同的并发包 -> 因需而变  Jdk’s LinkedBlockingQueue (MPMC) -> jctools’ MPSC
- 衡量好锁的价值 -> 能不用则不用 避免用锁:用 ThreadLocal 来避免资源争用
- Netty 应用场景下:局部串行 + 整体并行 > 一个队列 + 多个线程模式

4、对象池、内存池的复用

5、RPS（Receive Packet Steering）主要是把软中断的负载均衡到各个cpu，该功能主要针对单队列网卡多CPU环境。网卡驱动对每个流生成一个hash标识，这个HASH值得计算可以通过四元组来计算（SIP，SPORT，DIP，DPORT），然后由中断处理的地方根据这个hash标识分配到相应的CPU上去，这样就可以比较充分的发挥多核的能力了

6、RFS（Receive flow steering）确保应用程序处理的cpu跟软中断处理的cpu是同一个，这样就充分利用cpu的cache

Netty如何对内存进行保护？

1、提供了内存池和对象池

2、消息发送队列积压保护

消息发送队列ChannelOutboundBuffer并没有容量上限，如果发送方发送速度过快或者一次批量发送数据过大，会导致ChannelOutboundBuffer的内存膨胀，可能会使系统的内存溢出

根据业务配置合适的高水位（WriteBufferHighWaterMark）对消息的发送速度进行控制

同时，在发送消息时，调用channel的isWritable方法判断是否可写

3、缓冲区溢出保护

在对消息进行解码使，会创建缓冲区

在创建ByteBuf时对它的容量上限进行保护性设置

在消息解码时，对消息长度进行判断，如果超过最大容量，则抛出解码异常

链路的有效性检测

1、TCP层面的心跳检测，比如TCP的keep-alive机制

2、协议层的心跳检测

3、应用层的心跳检测



Netty中IO读写操作默认使用内存池的堆外内存，如果需要额外使用ByteBuf，建议也采用内存池的方式。如果不涉及网络读写，可以使用堆内存池，这样内存的创建效率会高一些

如何让应用易诊断

完善“线程名”

完善 “Handler ”名称



整改线程模型

1、在 handler 内部使用 JDK Executors

2、在添加handler时，指定Executor

​	EventExecutorGroup eventExecutorGroup = new UnorderedThreadPoolEventExecutor(10); 	

   pipeline.addLast(eventExecutorGroup, serverHandler)   
 io.netty.channel.DefaultChannelPipeline#addLast(io.netty.util.concurrent.EventExecutorGroup, java.lang.String, io.netty.channel.ChannelHandler)

```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);
				//创建DefaultChannelHandlerContext
        newCtx = newContext(group, filterName(name, handler), handler);
        addLast0(newCtx);
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            callHandlerAddedInEventLoop(newCtx, executor);
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}
```

```java
private EventExecutor childExecutor(EventExecutorGroup group) {
    if (group == null) {
        return null;
    }
  //是否只使用EventExecutorGroup中的一个EventExecutor，默认null
    Boolean pinEventExecutor = channel.config().getOption(ChannelOption.SINGLE_EVENTEXECUTOR_PER_GROUP);
    if (pinEventExecutor != null && !pinEventExecutor) {
        return group.next();
    }
    Map<EventExecutorGroup, EventExecutor> childExecutors = this.childExecutors;
    if (childExecutors == null) {
        childExecutors = this.childExecutors = new IdentityHashMap<EventExecutorGroup, EventExecutor>(4);
    }
  
    EventExecutor childExecutor = childExecutors.get(group);
    if (childExecutor == null) {
        childExecutor = group.next();
        childExecutors.put(group, childExecutor);
    }
    return childExecutor;
}
```

io.netty.util.concurrent.UnorderedThreadPoolEventExecutor#next

```java
public EventExecutor next() {
    return this; //返回的是线程池
}
```

io.netty.util.concurrent.MultithreadEventExecutorGroup#next

```java
public EventExecutor next() {
    return chooser.next(); //只有一个线程，NioEventLoop
}
```

```java
public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
}
```

减少系统调用

```java
public class HttpPipeliningHandler extends SimpleChannelInboundHandler<HttpRequest> {   	    @Override  
  public void channelRead(ChannelHandlerContext ctx, HttpRequest req) {   
    ChannelFuture future = ctx.writeAnd(createResponse(req));    
    if (!isKeepAlive(req)) {     
      future.addListener(ChannelFutureListener.CLOSE);  
    }  
  }
  @Override 
  public void channelReadComplete(ChannelHandlerContext ctx) {   
    ctx.flush();  
  } 
}
```

Limit flushes as much as possible as syscalls are quite expensive

减少GC压力

```java
channel.write(msg, Channel.voidPromise())
```

设置高低水位线，每次写数据判断channel是否可写

Risk of *OutOfMemoryError* if writing too fast and having slow receiver!

通过channelpipeline传递自定义事件

```java
public enum CustomEvents {
  MyCustomEvent
}

public class CustomEventHandler extends ChannelInboundHandlerAdapter {
  @Override
  public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt == MyCustomEvent) { // do something}
  }
}

ChannelPipeline pipeline = channel.pipeline();
pipeline.fireUserEventTriggered(MyCustomEvent);
```

