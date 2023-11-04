# HotKey源码分析

* [概述](#概述)
* [Client启动](#client启动)
  * [与etcd建立连接](#与etcd建立连接)
  * [定时推送Key到Worker节点](#定时推送key到worker节点)
    * [获取推送数据](#获取推送数据)
    * [选择worker进行推送](#选择worker进行推送)
  * [定时对Worker连接重连](#定时对worker连接重连)
  * [注册时事件监听器](#注册时事件监听器)
    * [监听worker信息变动](#监听worker信息变动)
    * [监听热点key推送](#监听热点key推送)
  * [启动与etcd相关的监听](#启动与etcd相关的监听)
* [Worker启动](#worker启动)
  * [与Etcd建立连接](#与etcd建立连接-1)
  * [初始化Netty网络通信](#初始化netty网络通信)
  * [读取客户端消息](#读取客户端消息)
    * [处理上报appName](#处理上报appname)
    * [处理心跳包](#处理心跳包)
    * [处理热Key](#处理热key)
  * [消息的处理](#消息的处理)
    * [基于Queue的生产消费模型](#基于queue的生产消费模型)
      * [生产消息](#生产消息)
      * [消费消息](#消费消息)
        * [创建消费者](#创建消费者)
        * [开始消费](#开始消费)
    * [基于Disruptor生产消费模型](#基于disruptor生产消费模型)
      * [消费消息](#消费消息-1)
  * [热Key的判断](#热key的判断)


# 概述

探测热点Key，将其推送到各个服务端内存，降低了热数据对数据层的查询压力，提升了应用性能

# Client启动

com.jd.platform.hotkey.client.ClientStarter#startPipeline

```java
/**
 * 启动监听etcd
 */
public void startPipeline() {
    JdLogger.info(getClass(), "etcdServer:" + etcdServer);
    //设置caffeine的最大容量
    Context.CAFFEINE_SIZE = caffeineSize;
    //1.设置etcd地址，建立连接
    EtcdConfigFactory.buildConfigCenter(etcdServer);
    //2.开始定时推送，基于key，默认500ms
    PushSchedulerStarter.startPusher(pushPeriod);
    //推送热点数量统计，基于keyRule
    PushSchedulerStarter.startCountPusher(10);
    //3.开启worker重连器
    WorkerRetryConnector.retryConnectWorkers();
    //4.注册时事件监听器
    registEventBus();
    //5.启动与etcd相关的监听
    EtcdStarter starter = new EtcdStarter();
    starter.start();
}
```

## 与etcd建立连接 

com.jd.platform.hotkey.client.etcd.EtcdConfigFactory#buildConfigCenter

```java
public static void buildConfigCenter(String etcdServer) {
    //连接多个时，逗号分隔
    configCenter = JdEtcdBuilder.build(etcdServer);
}
```

## 定时推送Key到Worker节点

 com.jd.platform.hotkey.client.core.key.PushSchedulerStarter#startPusher

```java
public static void startPusher(Long period) {
    if (period == null || period <= 0) {
        period = 500L;
    }
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      	//TurnKeyCollector
        IKeyCollector<HotKeyModel, HotKeyModel> collectHK =KeyHandlerFactory.getCollector();
        KeyHandlerFactory.getPusher().send(Context.APP_NAME, collectHK.lockAndGetResult());
        collectHK.finishOnce();
    },0, period, TimeUnit.MILLISECONDS);
}
```

### 获取推送数据

com.jd.platform.hotkey.client.core.key.TurnKeyCollector#lockAndGetResult

```java
@Override
public List<HotKeyModel> lockAndGetResult() {//获取发送给worker的HotKeyModel，map0、map1同时只有其中之一可以使用.自增后，对应的map就会停止被写入，等待被读取
    atomicLong.addAndGet(1);
    List<HotKeyModel> list;
    if (atomicLong.get() % 2 == 0) {
        list = get(map1);
        map1.clear();
    } else {
        list = get(map0);
        map0.clear();
    }
    return list;
}
```

### 选择worker进行推送

com.jd.platform.hotkey.client.core.key.NettyKeyPusher#send

```java
public void send(String appName, List<HotKeyModel> list) {
    //积攒了半秒的key集合，按照hash分发到不同的worker
    long now = System.currentTimeMillis();

    Map<Channel, List<HotKeyModel>> map = new HashMap<>();
    for(HotKeyModel model : list) {
        model.setCreateTime(now);
      	//选择key需要推送到的worker
        Channel channel = WorkerInfoHolder.chooseChannel(model.getKey());
        if (channel == null) {
            continue;
        }
        List<HotKeyModel> newList = map.computeIfAbsent(channel, k -> new ArrayList<>());
        newList.add(model);
    }
		//推送HotKeyModel到worker
    for (Channel channel : map.keySet()) {
        try {
            List<HotKeyModel> batch = map.get(channel);
            channel.writeAndFlush(MsgBuilder.buildByteBuf(new HotKeyMsg(MessageType.REQUEST_NEW_KEY, FastJsonUtils.convertObjectToJSON(batch))));
        } catch (Exception e) {
            try {
                InetSocketAddress insocket = (InetSocketAddress) channel.remoteAddress();
                JdLogger.error(getClass(),"flush error " + insocket.getAddress().getHostAddress());
            } catch (Exception ex) {
                JdLogger.error(getClass(),"flush error");
            }

        }
    }

}
```

## 定时对Worker连接重连

com.jd.platform.hotkey.client.core.worker.WorkerRetryConnector#retryConnectWorkers

```java
public static void retryConnectWorkers() {
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    //开启拉取etcd的worker信息，如果拉取失败，则定时继续拉取
    scheduledExecutorService.scheduleAtFixedRate(WorkerRetryConnector::reConnectWorkers, 30, 30, TimeUnit.SECONDS);
}
```

```java
private static void reConnectWorkers() {
    List<String> nonList = WorkerInfoHolder.getNonConnectedWorkers();
    if (nonList.size() == 0) {
        return;
    }
    JdLogger.info(WorkerRetryConnector.class, "trying to reConnect to these workers :" + nonList);
    NettyClient.getInstance().connect(nonList);
}
```

```java
public static List<String> getNonConnectedWorkers() {
    List<String> list = new ArrayList<>();
    for (Server server : WORKER_HOLDER) {
        //如果没连上，或者连上连接异常的
        if (!channelIsOk(server.channel)) {
            list.add(server.address);
        }
    }
    return list;
}
```

```java
private static boolean channelIsOk(Channel channel) {//判断连接的状态
    return channel != null && channel.isActive();
}
```

## 注册时事件监听器

```java
private void registEventBus() {
    //关注WorkerInfoChangeEvent事件,worker的上线、下线
    EventBusCenter.register(new WorkerChangeSubscriber());
    //热key探测回调关注热key事件
    EventBusCenter.register(new ReceiveNewKeySubscribe());
    //关注Rule的变化的事件
    EventBusCenter.register(new KeyRuleHolder());
}
```

### 监听worker信息变动

com.jd.platform.hotkey.client.core.worker.WorkerChangeSubscriber 

```java
com.jd.platform.hotkey.client.core.worker.WorkerChangeSubscriber 
//上线
@Subscribe
public void connectAll(WorkerInfoChangeEvent event) {
    List<String> addresses = event.getAddresses();
    if (addresses == null) {
        addresses = new ArrayList<>();
    }

    WorkerInfoHolder.mergeAndConnectNew(addresses);
}
```

```java
//下线
@Subscribe
public void channelInactive(ChannelInactiveEvent inactiveEvent) {
    //获取断线的channel
    Channel channel = inactiveEvent.getChannel();
    InetSocketAddress socketAddress = (InetSocketAddress) channel.remoteAddress();
    String address = socketAddress.getHostName() + ":" + socketAddress.getPort();
    JdLogger.warn(getClass(), "this channel is inactive : " + socketAddress + " trying to remove this connection");

    WorkerInfoHolder.dealChannelInactive(address);
}
```

### 监听热点key推送

com.jd.platform.hotkey.client.callback.ReceiveNewKeySubscribe

```java
@Subscribe
public void newKeyComing(ReceiveNewKeyEvent event) {
    HotKeyModel hotKeyModel = event.getModel();
    if (hotKeyModel == null) {
        return;
    }
    if (receiveNewKeyListener != null) {
        receiveNewKeyListener.newKey(hotKeyModel);
    }
}
```

## 启动与etcd相关的监听

```java
    public void start() {
      	//拉取worker信息
        fetchWorkerInfo();
        //拉取全量的Rule信息
        fetchRule();
        //拉取已经存在的热Key
        fetchExistHotKey();
        //异步监听rule规则变化
        startWatchRule();
        //监听热key事件，只监听手工添加、删除的key
        startWatchHotKey();
    }
```

# Worker启动

## 与Etcd建立连接

com.jd.platform.hotkey.worker.config.EtcdConfig

```java
@Configuration
public class EtcdConfig {
    @Value("${etcd.server}")
    private String etcdServer;

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Bean
    public IConfigCenter client() {
        logger.info("etcd address : " + etcdServer);
        //连接多个时，逗号分隔
        return JdEtcdBuilder.build(etcdServer);
    }
}
```

## 初始化Netty网络通信

com.jd.platform.hotkey.worker.starters.NodesServerStarter#start

```java
@PostConstruct
public void start() {//创建NodesServer，并将其启动
    AsyncPool.asyncDo(() -> {
        logger.info("netty server is starting");
        NodesServer nodesServer = new NodesServer();
        //对客户端的管理，新来、断线的管理
        nodesServer.setClientChangeListener(iClientChangeListener);
        //对客户端发来的消息，进行过滤处理
        nodesServer.setMessageFilters(messageFilters);
        nodesServer.setCodec(codec);
        try {
          //启动ServerBootstrap
            nodesServer.startNettyServer(port);
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
}
```

com.jd.platform.hotkey.worker.netty.server.NodesServer#startNettyServer

```java
public void startNettyServer(int port) throws Exception { //启动ServerBootstrap 
    //boss单线程
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup(CpuNum.workerCount());
    try {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .handler(new LoggingHandler(LogLevel.INFO))
                .option(ChannelOption.SO_BACKLOG, 1024)
                //保持长连接
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                //设置处理器
                .childHandler(new ChildChannelHandler());
        //绑定端口，同步等待成功
        ChannelFuture future = bootstrap.bind(port).sync();
        //等待服务器监听端口关闭
        future.channel().closeFuture().sync();
    } catch (Exception e) {
        //do nothing
        System.out.println("netty stop");
    } finally {
        //优雅退出，释放线程池资源
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
```

## 读取客户端消息

com.jd.platform.hotkey.worker.netty.server.NodesServerHandler#channelRead0

```java
protected void channelRead0(ChannelHandlerContext ctx, String message) {
    if (StringUtils.isEmpty(message)) {
        return;
    }
    HotKeyMsg msg = FastJsonUtils.toBean(message, HotKeyMsg.class);
    //过滤客户端发来的消息
    for (INettyMsgFilter messageFilter : messageFilters) { 
        boolean doNext = messageFilter.chain(msg, ctx);
        if (!doNext) {
            return;
        }
    }
}
```

### 处理上报appName

com.jd.platform.hotkey.worker.netty.filter.AppNameFilter

```java
public boolean chain(HotKeyMsg message, ChannelHandlerContext ctx) {
    if (MessageType.APP_NAME == message.getMessageType()) {
        String appName = message.getBody();
        if (clientEventListener != null) {
            clientEventListener.newClient(appName, NettyIpUtil.clientIp(ctx), ctx);
        }
        return false;
    }
    return true;
}
```

### 处理心跳包

com.jd.platform.hotkey.worker.netty.filter.HeartBeatFilter#chain

```java
public boolean chain(HotKeyMsg message, ChannelHandlerContext ctx) {
    if (MessageType.PING == message.getMessageType()) {
        String hotMsg = FastJsonUtils.convertObjectToJSON(new HotKeyMsg(MessageType.PONG, PONG));
        FlushUtil.flush(ctx, MsgBuilder.buildByteBuf(hotMsg));
        return false;
    }
    return true;
}
```

### 处理热Key

com.jd.platform.hotkey.worker.netty.filter.HotKeyFilter#chain

```java
public boolean chain(HotKeyMsg message, ChannelHandlerContext ctx) {
    if (MessageType.REQUEST_NEW_KEY == message.getMessageType()) {
        totalReceiveKeyCount.incrementAndGet();
        publishMsg(message.getBody(), ctx);//发布接收到的热key消息
        return false;
    }
    return true;
}
```

com.jd.platform.hotkey.worker.netty.filter.HotKeyFilter#publishMsg

```java
private void publishMsg(String message, ChannelHandlerContext ctx) {
    //老版的用的单个HotKeyModel，新版用的数组
    List<HotKeyModel> models = FastJsonUtils.toList(message, HotKeyModel.class);
    for (HotKeyModel model : models) {
        //白名单key不处理
        if (WhiteListHolder.contains(model.getKey())) {
            continue;
        }
      	//延迟超过1s,打印log信息
        long timeOut = SystemClock.now() - model.getCreateTime();
        if (timeOut > 1000) {
            logger.info("key timeout " + timeOut + ", from ip : " + NettyIpUtil.clientIp(ctx));
        }
        keyProducer.push(model);
    }
}
```

## 消息的处理

### 基于Queue的生产消费模型

#### 生产消息

com.jd.platform.hotkey.worker.keydispatcher.KeyProducer#push

```java
public void push(HotKeyModel model) {
    if (model == null || model.getKey() == null) {
        return;
    }
    //5秒前的过时消息就不处理了
    if (SystemClock.now() - model.getCreateTime() > InitConstant.timeOut) {
        expireTotalCount.increment();
        return;
    }
    try {
       //存放到有界限的LinkedBlockingQueue中
        QUEUE.put(model);
        totalOfferCount.increment();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

#### 消费消息

##### 创建消费者

com.jd.platform.hotkey.worker.keydispatcher.DispatcherConfig#consumer

```java
public Consumer consumer() {
    int nowCount = CpuNum.workerCount();
    //将实际值赋给static变量
    if(threadCount != 0) {
        nowCount = threadCount;
    }
		
    List<KeyConsumer> consumerList = new ArrayList<>();
    for (int i = 0; i < nowCount; i++) {
       //KeyConsumer包含处理消息的逻辑
        KeyConsumer keyConsumer = new KeyConsumer();
        keyConsumer.setKeyListener(iKeyListener);
        consumerList.add(keyConsumer);
       //提交线程池，开始消息
        threadPoolExecutor.submit(keyConsumer::beginConsume);
    }
    return new Consumer(consumerList);
}
```

##### 开始消费

com.jd.platform.hotkey.worker.keydispatcher.KeyConsumer#beginConsume

```java
public void beginConsume() {
    while (true) {
        try {
            HotKeyModel model = QUEUE.take();
            if (model.isRemove()) {
                iKeyListener.removeKey(model, KeyEventOriginal.CLIENT);
            } else {
                iKeyListener.newKey(model, KeyEventOriginal.CLIENT);
            }
            //处理完毕，将数量加1
            totalDealCount.increment();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 基于Disruptor生产消费模型

创建MessageProducer

com.jd.platform.hotkey.worker.disruptor.ProducerFactory#createHotKeyProducer

```java
public static MessageProducer<HotKeyEvent> createHotKeyProducer(IKeyListener iKeyListener) {
        int threadCount = CpuNum.workerCount();
        //如果手工指定了线程数，就用手工指定的
        if (InitConstant.threadCount != 0) {
            threadCount = InitConstant.threadCount;
        } else {
            if (threadCount >= 8) {
                threadCount = threadCount / 2;
            }
        }
  			//创建实现了Disruptor中WorkHandler接口的消费者
        HotKeyEventConsumer[] array = new HotKeyEventConsumer[threadCount];
        for (int i = 0; i < threadCount; i++) {
            array[i] = new HotKeyEventConsumer(i);
            //IKeyListener，处理热Key
            array[i].setKeyListener(iKeyListener);
        }
        DisruptorBuilder<HotKeyEvent> builder = new DisruptorBuilder<>();
  			//创建Disruptor，并将其启动
        Disruptor<HotKeyEvent> disruptor = builder
                .setBufferSize(InitConstant.bufferSize * 1024 * 1024) //RingBuffer大小，一般都
                .setEventFactory(HotKeyEvent::new) //创建消息的工厂
                .setWorkerHandlers(array)    //不重复消费的
                .build();

        return new HotKeyEventProducer(disruptor);
    }
```

#### 消费消息

com.jd.platform.hotkey.worker.disruptor.hotkey.HotKeyEventConsumer#onNewEvent

```java
protected void onNewEvent(HotKeyEvent hotKeyEvent) {
    HotKeyModel model = hotKeyEvent.getModel();
    if (iKeyListener == null) {
        logger.warn("new key is coming, but no consumer deal this key!");
        return;
    }

    if (model.isRemove()) {
        iKeyListener.removeKey(model, KeyEventOriginal.CLIENT);
    } else {
        iKeyListener.newKey(model, KeyEventOriginal.CLIENT);
    }
}
```

## 热Key的判断

com.jd.platform.hotkey.worker.keylistener.KeyListener#newKey

```java
public void newKey(HotKeyModel hotKeyModel, KeyEventOriginal original) {
    //cache里的key
    String key = buildKey(hotKeyModel);
    //判断是不是刚热不久
    Object o = hotCache.getIfPresent(key);
    if (o != null) {
        return;
    }
    SlidingWindow slidingWindow = checkWindow(hotKeyModel, key);
    //加上客户端发来的count，看看hot没
    boolean hot = slidingWindow.addCount(hotKeyModel.getCount());
    //如果没hot，重新put，cache会自动刷新过期时间
    if (!hot) {
        CaffeineCacheHolder.getCache(hotKeyModel.getAppName()).put(key, slidingWindow);
    } else {
        hotCache.put(key, 1);
        //删掉该key
        CaffeineCacheHolder.getCache(hotKeyModel.getAppName()).invalidate(key);
        //开启推送
        hotKeyModel.setCreateTime(SystemClock.now());
        logger.info(NEW_KEY_EVENT + hotKeyModel.getKey());
        //分别推送到各client和etcd，双重保证，确保client接收到热点Key
        for (IPusher pusher : iPushers) {
            pusher.push(hotKeyModel);
        }
    }
}
```

