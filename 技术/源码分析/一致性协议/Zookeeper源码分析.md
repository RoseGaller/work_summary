# Zookeeper源码分析

# 客户端

## ZooKeeper

org.apache.zookeeper.ZooKeeper#ZooKeeper(java.lang.String, int, org.apache.zookeeper.Watcher, boolean)

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,  boolean canBeReadOnly) //连接地址、会话超时时间、默认Watcher、是否只读	
    throws IOException
{
    LOG.info("Initiating client connection, connectString=" + connectString
            + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);
		//设置默认watcher
    watchManager.defaultWatcher = watcher;
	  //解析服务端地址
    ConnectStringParser connectStringParser = new ConnectStringParser(
            connectString);
    //负责挑选连接的节点（初始化时打散集群地址，避免每个客户端挑选相同的集群节点，导致某个节点负载过重）
    HostProvider hostProvider = new StaticHostProvider(
            connectStringParser.getServerAddresses());
   //ZK是异步与服务端建立连接的，当连接尚未建立就发送请求时会抛出CONNECTIONLOSS异常
   //为了防止异常的发生可以通过注册Watcher，当监听到连接成功的事件时，再发送请求
   //创建ClientCnxn,管理客户端的网络IO
    cnxn = createConnection(connectStringParser.getChrootPath(),
            hostProvider, sessionTimeout, this, watchManager,
            getClientCnxnSocket(), canBeReadOnly);
  	//启动发送、接收线程
    cnxn.start();
}
```

## StaticHostProvider

```java
public StaticHostProvider(Collection<InetSocketAddress> serverAddresses) {//初始化
    this.resolver = new Resolver() {
        @Override
        public InetAddress[] getAllByName(String name) throws UnknownHostException {
            return InetAddress.getAllByName(name);
        }
    };
    init(serverAddresses);
}
```



```java
private void init(Collection<InetSocketAddress> serverAddresses) {
    if (serverAddresses.isEmpty()) {
        throw new IllegalArgrumentException(
                "A HostProvider may not be empty!");
    }

    this.serverAddresses.addAll(serverAddresses);
    //为了服务端的负载均衡，打散集群地址
    Collections.shuffle(this.serverAddresses); 
}
```

## ClientCnxn

```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket, boolean canBeReadOnly)
        throws IOException {
    this(chrootPath, hostProvider, sessionTimeout, zooKeeper, watcher,
         clientCnxnSocket, 0, new byte[16], canBeReadOnly);
}
```

```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId; //默认为0
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;

    connectTimeout = sessionTimeout / hostProvider.size(); //连接超时
    readTimeout = sessionTimeout * 2 / 3; //读取超时
    readOnly = canBeReadOnly;

    sendThread = new SendThread(clientCnxnSocket); //管理客户端与服务端的网络IO
    eventThread = new EventThread(); //处理客户端的watcher事件、回调方法

}
```

```java
public void start() {
    sendThread.start();
    eventThread.start();
}
```

## ClientCnxnSocketNIO

使用JDK原生的NIO进行网络通信

```java
private static ClientCnxnSocket getClientCnxnSocket() throws IOException {
  	//创建ClientCnxnSocketNIO
    String clientCnxnSocketName = System
            .getProperty(ZOOKEEPER_CLIENT_CNXN_SOCKET);
    if (clientCnxnSocketName == null) {
        clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
    }
    try {
        return (ClientCnxnSocket) Class.forName(clientCnxnSocketName).getDeclaredConstructor()
                .newInstance();
    } catch (Exception e) {
        IOException ioe = new IOException("Couldn't instantiate "
                + clientCnxnSocketName);
        ioe.initCause(e);
        throw ioe;
    }
}
```

## ClientCnxnSocketNetty

### connect

org.apache.zookeeper.ClientCnxnSocketNetty#connect

```java
void connect(InetSocketAddress addr) throws IOException {
    this.firstConnect = new CountDownLatch(1);
    Bootstrap bootstrap = (Bootstrap)((Bootstrap)((Bootstrap)((Bootstrap)((Bootstrap)(new Bootstrap()).group(this.eventLoopGroup)).channel(NettyUtils.nioOrEpollSocketChannel())).option(ChannelOption.SO_LINGER, -1)).option(ChannelOption.TCP_NODELAY, true)).handler(new ClientCnxnSocketNetty.ZKClientPipelineFactory(addr.getHostString(), addr.getPort()));
  
    bootstrap = this.configureBootstrapAllocator(bootstrap);
    bootstrap.validate();
    this.connectLock.lock();

    try {
        this.connectFuture = bootstrap.connect(addr);
        this.connectFuture.addListener(new ChannelFutureListener() {
            public void operationComplete(ChannelFuture channelFuture) throws Exception {
                boolean connected = false;
                ClientCnxnSocketNetty.this.connectLock.lock();

                try {
                    if (channelFuture.isSuccess()) {
                        if (ClientCnxnSocketNetty.this.connectFuture == null) {
                            ClientCnxnSocketNetty.LOG.info("connect attempt cancelled");
                            channelFuture.channel().close();
                            return;
                        }

                        ClientCnxnSocketNetty.this.channel = channelFuture.channel();
                        ClientCnxnSocketNetty.this.disconnected.set(false);
                        ClientCnxnSocketNetty.this.initialized = false;
                        ClientCnxnSocketNetty.this.lenBuffer.clear();
                        ClientCnxnSocketNetty.this.incomingBuffer = ClientCnxnSocketNetty.this.lenBuffer;
                        ClientCnxnSocketNetty.this.sendThread.primeConnection();
                        ClientCnxnSocketNetty.this.updateNow();
                        ClientCnxnSocketNetty.this.updateLastSendAndHeard();
                        if (ClientCnxnSocketNetty.this.sendThread.tunnelAuthInProgress()) {
                            ClientCnxnSocketNetty.this.waitSasl.drainPermits();
                            ClientCnxnSocketNetty.this.needSasl.set(true);
                            ClientCnxnSocketNetty.this.sendPrimePacket();
                        } else {
                            ClientCnxnSocketNetty.this.needSasl.set(false);
                        }

                        connected = true;
                        return;
                    }

                    ClientCnxnSocketNetty.LOG.info("future isn't success, cause:", channelFuture.cause());
                } finally {
                    ClientCnxnSocketNetty.this.connectFuture = null;
                    ClientCnxnSocketNetty.this.connectLock.unlock();
                    if (connected) {
                        ClientCnxnSocketNetty.LOG.info("channel is connected: {}", channelFuture.channel());
                    }

                    ClientCnxnSocketNetty.this.wakeupCnxn();
                    ClientCnxnSocketNetty.this.firstConnect.countDown();
                }

            }
        });
    } finally {
        this.connectLock.unlock();
    }

}
```

### read

org.apache.zookeeper.ClientCnxnSocketNetty.ZKClientHandler#channelRead0

```java
protected void channelRead0(ChannelHandlerContext ctx, ByteBuf buf) throws Exception {
    ClientCnxnSocketNetty.this.updateNow();

    while(buf.isReadable()) {
        if (ClientCnxnSocketNetty.this.incomingBuffer.remaining() > buf.readableBytes()) {
            int newLimit = ClientCnxnSocketNetty.this.incomingBuffer.position() + buf.readableBytes();
            ClientCnxnSocketNetty.this.incomingBuffer.limit(newLimit);
        }

        buf.readBytes(ClientCnxnSocketNetty.this.incomingBuffer);
        ClientCnxnSocketNetty.this.incomingBuffer.limit(ClientCnxnSocketNetty.this.incomingBuffer.capacity());
        if (!ClientCnxnSocketNetty.this.incomingBuffer.hasRemaining()) {
            ClientCnxnSocketNetty.this.incomingBuffer.flip();
            if (ClientCnxnSocketNetty.this.incomingBuffer == ClientCnxnSocketNetty.this.lenBuffer) {
                ClientCnxnSocketNetty.this.recvCount.getAndIncrement();
                ClientCnxnSocketNetty.this.readLength();
            } else if (!ClientCnxnSocketNetty.this.initialized) {
                ClientCnxnSocketNetty.this.readConnectResult();
                ClientCnxnSocketNetty.this.lenBuffer.clear();
                ClientCnxnSocketNetty.this.incomingBuffer = ClientCnxnSocketNetty.this.lenBuffer;
                ClientCnxnSocketNetty.this.initialized = true;
                ClientCnxnSocketNetty.this.updateLastHeard();
            } else {
                ClientCnxnSocketNetty.this.sendThread.readResponse(ClientCnxnSocketNetty.this.incomingBuffer);
                ClientCnxnSocketNetty.this.lenBuffer.clear();
                ClientCnxnSocketNetty.this.incomingBuffer = ClientCnxnSocketNetty.this.lenBuffer;
                ClientCnxnSocketNetty.this.updateLastHeard();
            }
        }
    }

    ClientCnxnSocketNetty.this.wakeupCnxn();
}
```

### write

```java
void doTransport(int waitTimeOut, List<Packet> pendingQueue, ClientCnxn cnxn) throws IOException, InterruptedException {
    try {
        if (this.firstConnect.await((long)waitTimeOut, TimeUnit.MILLISECONDS)) {
            Packet head = null;
            if (this.needSasl.get()) {
                if (!this.waitSasl.tryAcquire((long)waitTimeOut, TimeUnit.MILLISECONDS)) {
                    return;
                }
            } else {
                head = (Packet)this.outgoingQueue.poll((long)waitTimeOut, TimeUnit.MILLISECONDS);
            }

            if (!this.sendThread.getZkState().isAlive()) {
                this.addBack(head);
                return;
            }

            if (this.disconnected.get()) {
                this.addBack(head);
                throw new EndOfStreamException("channel for sessionid 0x" + Long.toHexString(this.sessionId) + " is lost");
            }

            if (head != null) {//发送请求
                this.doWrite(pendingQueue, head, cnxn);
            }

            return;
        }
    } finally {
        this.updateNow();
    }

}
```



```java
private void doWrite(List<Packet> pendingQueue, Packet p, ClientCnxn cnxn) {
    this.updateNow();
    boolean anyPacketsSent = false;

    while(true) {
        if (p != ClientCnxnSocketNetty.WakeupPacket.getInstance()) {
            if (p.requestHeader != null && p.requestHeader.getType() != 11 && p.requestHeader.getType() != 100) {
                p.requestHeader.setXid(cnxn.getXid());
                synchronized(pendingQueue) {
                    pendingQueue.add(p);
                }
            }

            this.sendPktOnly(p); //不调用flush
            anyPacketsSent = true;
        }

        if (this.outgoingQueue.isEmpty()) { //没有未发送的请求
            if (anyPacketsSent) {
                this.channel.flush(); //系统调用、发送数据
            }

            return;
        }

        p = (Packet)this.outgoingQueue.remove();
    }
}
```

```java
private ChannelFuture sendPkt(Packet p, boolean doFlush) {
    p.createBB();
    this.updateLastSend();
    ByteBuf writeBuffer = Unpooled.wrappedBuffer(p.bb);
    ChannelFuture result = doFlush ? this.channel.writeAndFlush(writeBuffer) : this.channel.write(writeBuffer);
    result.addListener(this.onSendPktDoneListener);
    return result;
}
```

```java
protected ClientCnxn createConnection(String chrootPath,
        HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        boolean canBeReadOnly) throws IOException {
    return new ClientCnxn(chrootPath, hostProvider, sessionTimeout, this,
            watchManager, clientCnxnSocket, canBeReadOnly);
}
```

```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;

    connectTimeout = sessionTimeout / hostProvider.size();
    readTimeout = sessionTimeout * 2 / 3;
    readOnly = canBeReadOnly;

    sendThread = new SendThread(clientCnxnSocket);
    eventThread = new EventThread();
}
```

启动SendThread、EventThread

org.apache.zookeeper.ClientCnxn#start

```java
public void start() {
    sendThread.start();
    eventThread.start();
}
```

## SendThread

org.apache.zookeeper.ClientCnxn.SendThread#SendThread

```java
SendThread(ClientCnxnSocket clientCnxnSocket) {
    super(makeThreadName("-SendThread()"));
    state = States.CONNECTING; //客户端网络连接状态：正在连接中
    this.clientCnxnSocket = clientCnxnSocket;
    setDaemon(true);
}
```

org.apache.zookeeper.ClientCnxn.SendThread#run

```java
public void run() {
    clientCnxnSocket.introduce(this,sessionId);
    clientCnxnSocket.updateNow();
    clientCnxnSocket.updateLastSendAndHeard();
    int to;
    long lastPingRwServer = Time.currentElapsedTime();
    final int MAX_SEND_PING_INTERVAL = 10000; //10 seconds
    InetSocketAddress serverAddress = null;
    while (state.isAlive()) {
        try {
            if (!clientCnxnSocket.isConnected()) { //尚未连接
                if(!isFirstConnect){ //不是第一次连接，进行休眠
                    try {
                        Thread.sleep(r.nextInt(1000));
                    } catch (InterruptedException e) {
                        LOG.warn("Unexpected exception", e);
                    }
                }
              
                // 会话已经关闭
                if (closing || !state.isAlive()) {
                    break;
                }
                //挑选连接地址，hostProvider会将地址进行打散，避免每次都连接同一个节点
                if (rwServerAddress != null) {
                    serverAddress = rwServerAddress;
                    rwServerAddress = null;
                } else {
                    serverAddress = hostProvider.next(1000);
                }
                //开始连接
                startConnect(serverAddress);
                clientCnxnSocket.updateLastSendAndHeard();
            }
						//已经连接成功，发送了连接请求并收到了响应
            if (state.isConnected()) { 
                // determine whether we need to send an AuthFailed event.
                if (zooKeeperSaslClient != null) {
                    boolean sendAuthEvent = false;
                    if (zooKeeperSaslClient.getSaslState() == ZooKeeperSaslClient.SaslState.INITIAL) {
                        try {
                            zooKeeperSaslClient.initialize(ClientCnxn.this);
                        } catch (SaslException e) {
                           LOG.error("SASL authentication with Zookeeper Quorum member failed: " + e);
                            state = States.AUTH_FAILED;
                            sendAuthEvent = true;
                        }
                    }
                    KeeperState authState = zooKeeperSaslClient.getKeeperState();
                    if (authState != null) {
                        if (authState == KeeperState.AuthFailed) {
                            // An authentication error occurred during authentication with the Zookeeper Server.
                            state = States.AUTH_FAILED;
                            sendAuthEvent = true;
                        } else {
                            if (authState == KeeperState.SaslAuthenticated) {
                                sendAuthEvent = true;
                            }
                        }
                    }

                    if (sendAuthEvent == true) {
                        eventThread.queueEvent(new WatchedEvent(
                              Watcher.Event.EventType.None,
                              authState,null));
                    }
                }
                to = readTimeout - clientCnxnSocket.getIdleRecv();
            } else {
                to = connectTimeout - clientCnxnSocket.getIdleRecv();
            }
            
            if (to <= 0) {
                String warnInfo;
                warnInfo = "Client session timed out, have not heard from server in "
                    + clientCnxnSocket.getIdleRecv()
                    + "ms"
                    + " for sessionid 0x"
                    + Long.toHexString(sessionId);
                LOG.warn(warnInfo);
                throw new SessionTimeoutException(warnInfo);
            }
            if (state.isConnected()) {
               //1000(1 second) is to prevent race condition missing to send the second ping
               //also make sure not to send too many pings when readTimeout is small 
                int timeToNextPing = readTimeout / 2 - clientCnxnSocket.getIdleSend() - 
                      ((clientCnxnSocket.getIdleSend() > 1000) ? 1000 : 0);
                //send a ping request either time is due or no packet sent out within MAX_SEND_PING_INTERVAL
                if (timeToNextPing <= 0 || clientCnxnSocket.getIdleSend() > MAX_SEND_PING_INTERVAL) {
                    sendPing(); //发送ping请求
                    clientCnxnSocket.updateLastSend();
                } else {
                    if (timeToNextPing < to) {
                        to = timeToNextPing;
                    }
                }
            }

            // If we are in read-only mode, seek for read/write server
            if (state == States.CONNECTEDREADONLY) {
                long now = Time.currentElapsedTime();
                int idlePingRwServer = (int) (now - lastPingRwServer);
                if (idlePingRwServer >= pingRwTimeout) {
                    lastPingRwServer = now;
                    idlePingRwServer = 0;
                    pingRwTimeout =
                        Math.min(2*pingRwTimeout, maxPingRwTimeout);
                    pingRwServer();
                }
                to = Math.min(to, pingRwTimeout - idlePingRwServer);
            }

            clientCnxnSocket.doTransport(to, pendingQueue, outgoingQueue, ClientCnxn.this);
        } catch (Throwable e) {
            if (closing) {
                if (LOG.isDebugEnabled()) {
                    // closing so this is expected
                    LOG.debug("An exception was thrown while closing send thread for session 0x"
                            + Long.toHexString(getSessionId())
                            + " : " + e.getMessage());
                }
                break;
            } else {
                // this is ugly, you have a better way speak up
                if (e instanceof SessionExpiredException) {
                    LOG.info(e.getMessage() + ", closing socket connection");
                } else if (e instanceof SessionTimeoutException) {
                    LOG.info(e.getMessage() + RETRY_CONN_MSG);
                } else if (e instanceof EndOfStreamException) {
                    LOG.info(e.getMessage() + RETRY_CONN_MSG);
                } else if (e instanceof RWServerFoundException) {
                    LOG.info(e.getMessage());
                } else if (e instanceof SocketException) {
                    LOG.info("Socket error occurred: {}: {}", serverAddress, e.getMessage());
                } else {
                    LOG.warn("Session 0x{} for server {}, unexpected error{}",
                                    Long.toHexString(getSessionId()),
                                    serverAddress,
                                    RETRY_CONN_MSG,
                                    e);
                }
                cleanup();
                if (state.isAlive()) {
                    eventThread.queueEvent(new WatchedEvent(
                            Event.EventType.None,
                            Event.KeeperState.Disconnected,
                            null));
                }
                clientCnxnSocket.updateNow();
                clientCnxnSocket.updateLastSendAndHeard();
            }
        }
    }
    cleanup();
    clientCnxnSocket.close();
    if (state.isAlive()) {
        eventThread.queueEvent(new WatchedEvent(Event.EventType.None,
                Event.KeeperState.Disconnected, null));
    }
    ZooTrace.logTraceMessage(LOG, ZooTrace.getTextTraceLevel(),
            "SendThread exited loop for session: 0x"
                   + Long.toHexString(getSessionId()));
}
```

### 与服务端建立连接

org.apache.zookeeper.ClientCnxn.SendThread#startConnect

```java
private void startConnect(InetSocketAddress addr) throws IOException {
    // initializing it for new connection
    saslLoginFailed = false;
    state = States.CONNECTING;

    setName(getName().replaceAll("\\(.*\\)",
            "(" + addr.getHostName() + ":" + addr.getPort() + ")"));
    if (ZooKeeperSaslClient.isEnabled()) {
        try {
            zooKeeperSaslClient = new ZooKeeperSaslClient(SaslServerPrincipal.getServerPrincipal(addr));
        } catch (LoginException e) {
            eventThread.queueEvent(new WatchedEvent(
              Watcher.Event.EventType.None,
              Watcher.Event.KeeperState.AuthFailed, null));
            saslLoginFailed = true;
        }
    }
    logStartConnect(addr);
    clientCnxnSocket.connect(addr);
}
```

#### connect

```java
void connect(InetSocketAddress addr) throws IOException { //发起连接
    SocketChannel sock = createSock(); //创建socketChannel
    try {
       registerAndConnect(sock, addr);
    } catch (IOException e) {
        LOG.error("Unable to open socket to " + addr);
        sock.close();
        throw e;
    }
    initialized = false;
    lenBuffer.clear(); //重置ByteBuffer
    incomingBuffer = lenBuffer;
}
```



```java
SocketChannel createSock() throws IOException {//创建socketChannel
    SocketChannel sock;
    sock = SocketChannel.open();
    sock.configureBlocking(false); //非阻塞
    sock.socket().setSoLinger(false, -1);
    sock.socket().setTcpNoDelay(true); //禁用nagel算法	
    return sock;
}
```



```java
void registerAndConnect(SocketChannel sock, InetSocketAddress addr) 
throws IOException {
  	//注册连接事件
    sockKey = sock.register(selector, SelectionKey.OP_CONNECT);
    //发起连接
    boolean immediateConnect = sock.connect(addr);
    //连接成功
    if (immediateConnect) {
        sendThread.primeConnection();
    }
}
```

#### primeConnection

org.apache.zookeeper.ClientCnxn.SendThread#primeConnection

```java
void primeConnection() throws IOException {
   //发送连接请求、认证请求、注册watcher
    LOG.info("Socket connection established to "
             + clientCnxnSocket.getRemoteSocketAddress()
             + ", initiating session");
    isFirstConnect = false;
    long sessId = (seenRwServerBefore) ? sessionId : 0;
    //发送连接请求
    ConnectRequest conReq = new ConnectRequest(0, lastZxid,
            sessionTimeout, sessId, sessionPasswd);
    synchronized (outgoingQueue) {
        // We add backwards since we are pushing into the front
        // Only send if there's a pending watch
        // TODO: here we have the only remaining use of zooKeeper in
        // this class. It's to be eliminated!
        if (!disableAutoWatchReset) {
            List<String> dataWatches = zooKeeper.getDataWatches();
            List<String> existWatches = zooKeeper.getExistWatches();
            List<String> childWatches = zooKeeper.getChildWatches();
            if (!dataWatches.isEmpty()
                        || !existWatches.isEmpty() || !childWatches.isEmpty()) {

                Iterator<String> dataWatchesIter = prependChroot(dataWatches).iterator();
                Iterator<String> existWatchesIter = prependChroot(existWatches).iterator();
                Iterator<String> childWatchesIter = prependChroot(childWatches).iterator();
                long setWatchesLastZxid = lastZxid;

                while (dataWatchesIter.hasNext()
                               || existWatchesIter.hasNext() || childWatchesIter.hasNext()) {
                    List<String> dataWatchesBatch = new ArrayList<String>();
                    List<String> existWatchesBatch = new ArrayList<String>();
                    List<String> childWatchesBatch = new ArrayList<String>();
                    int batchLength = 0;

                    // Note, we may exceed our max length by a bit when we add the last
                    // watch in the batch. This isn't ideal, but it makes the code simpler.
                    while (batchLength < SET_WATCHES_MAX_LENGTH) {
                        final String watch;
                        if (dataWatchesIter.hasNext()) {
                            watch = dataWatchesIter.next();
                            dataWatchesBatch.add(watch);
                        } else if (existWatchesIter.hasNext()) {
                            watch = existWatchesIter.next();
                            existWatchesBatch.add(watch);
                        } else if (childWatchesIter.hasNext()) {
                            watch = childWatchesIter.next();
                            childWatchesBatch.add(watch);
                        } else {
                            break;
                        }
                        batchLength += watch.length();
                    }
										//注册Watcher
                    SetWatches sw = new SetWatches(setWatchesLastZxid,
                            dataWatchesBatch,
                            existWatchesBatch,
                            childWatchesBatch);
                    RequestHeader h = new RequestHeader();
                    h.setType(ZooDefs.OpCode.setWatches);
                    h.setXid(-8);
                    Packet packet = new Packet(h, new ReplyHeader(), sw, null, null);
                    outgoingQueue.addFirst(packet);
                }
            }
        }
				//认证请求
        for (AuthData id : authInfo) {
            outgoingQueue.addFirst(new Packet(new RequestHeader(-4,
                    OpCode.auth), null, new AuthPacket(0, id.scheme,
                    id.data), null, null));
        }
        //连接请求
        outgoingQueue.addFirst(new Packet(null, null, conReq,
                    null, null, readOnly));
    }
    //注册写事件
    clientCnxnSocket.enableReadWriteOnly();
    if (LOG.isDebugEnabled()) {
        LOG.debug("Session establishment request sent on "
                + clientCnxnSocket.getRemoteSocketAddress());
    }
}
```

### 处理网络IO

org.apache.zookeeper.ClientCnxnSocketNIO#doTransport

```java
void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
        throws IOException, InterruptedException {
  	//阻塞，直到超时或有事件发生
    selector.select(waitTimeOut);
    Set<SelectionKey> selected;
    synchronized (this) {
        selected = selector.selectedKeys();
    }
    updateNow();
    for (SelectionKey k : selected) {
        SocketChannel sc = ((SocketChannel) k.channel());
        if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) { //连接事件
            if (sc.finishConnect()) {//连接成功
                updateLastSendAndHeard();
                sendThread.primeConnection(); //发送连接请求
            }
        } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
            doIO(pendingQueue, outgoingQueue, cnxn);//读写事件
        }
    }
    if (sendThread.getZkState().isConnected()) {
        synchronized(outgoingQueue) {
            if (findSendablePacket(outgoingQueue,
                    cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                enableWrite();
            }
        }
    }
    selected.clear();
}
```

1、读取数据交由SendThread的readResponse处理

2、发送请求时，将请求放入outgoingQueue中，请求写到网络中之后，会将其从outgoingQueue移除，加入到pendingQueue中，表示已经发送但是未接收到响应

org.apache.zookeeper.ClientCnxnSocketNIO#doIO

```java
void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
  throws InterruptedException, IOException {
    SocketChannel sock = (SocketChannel) sockKey.channel();
    if (sock == null) {
        throw new IOException("Socket is null!");
    }
    if (sockKey.isReadable()) { //读事件
        int rc = sock.read(incomingBuffer); //读数据
        if (rc < 0) {
            throw new EndOfStreamException(
                    "Unable to read additional data from server sessionid 0x"
                            + Long.toHexString(sessionId)
                            + ", likely server has closed socket");
        }
        //incomingBuffer已经被填满，说明已读取完整的响应或者响应的长度
        if (!incomingBuffer.hasRemaining()) {
            incomingBuffer.flip();
            //初次读取相等或者读取完成后设置incomingBuffer=lenBuffer
            if (incomingBuffer == lenBuffer) { 
                recvCount++;
                //根据响应的长度，创建incomingBuffer
                readLength();
            } else if (!initialized) { //尚未初始化
                readConnectResult(); //处理连接请求
                enableRead(); //注册读事件
                if (findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    // Since SASL authentication has completed (if client is configured to do so),
                    // outgoing packets waiting in the outgoingQueue can now be sent.
                    enableWrite(); //outgogingQueu不为空并且SASL禁用时，注册写事件
                }
                lenBuffer.clear(); //清空
                incomingBuffer = lenBuffer; 
                updateLastHeard();
                initialized = true; //初始化完成
            } else {
              //处理非连接请求的响应	
                sendThread.readResponse(incomingBuffer);
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                updateLastHeard();
            }
        }
    }
    if (sockKey.isWritable()) { //写事件
        synchronized(outgoingQueue) {
            Packet p = findSendablePacket(outgoingQueue,
                    cnxn.sendThread.clientTunneledAuthenticationInProgress());

            if (p != null) {
             	   updateLastSend();
                // If we already started writing p, p.bb will already exist
                if (p.bb == null) {
                    if ((p.requestHeader != null) &&
                            (p.requestHeader.getType() != OpCode.ping) &&
                            (p.requestHeader.getType() != OpCode.auth)) {
                        p.requestHeader.setXid(cnxn.getXid());
                    }
                    p.createBB();
                }
                sock.write(p.bb); //发送数据
                if (!p.bb.hasRemaining()) {//都已发送出去
                    sentCount++;
                    //移除
                    outgoingQueue.removeFirstOccurrence(p);
                    if (p.requestHeader != null
                            && p.requestHeader.getType() != OpCode.ping
                            && p.requestHeader.getType() != OpCode.auth) {
                        synchronized (pendingQueue) {
                          //加入pendingQueue，已经发送但未收到响应
                            pendingQueue.add(p);
                        }
                    }
                }
            }
            if (outgoingQueue.isEmpty()) {
                // No more packets to send: turn off write interest flag.
                // Will be turned on later by a later call to enableWrite(),
                // from within ZooKeeperSaslClient (if client is configured
                // to attempt SASL authentication), or in either doIO() or
                // in doTransport() if not.
                disableWrite();
            } else if (!initialized && p != null && !p.bb.hasRemaining()) {
                // On initial connection, write the complete connect request
                // packet, but then disable further writes until after
                // receiving a successful connection response.  If the
                // session is expired, then the server sends the expiration
                // response and immediately closes its end of the socket.  If
                // the client is simultaneously writing on its end, then the
                // TCP stack may choose to abort with RST, in which case the
                // client would never receive the session expired event.  See
                // http://docs.oracle.com/javase/6/docs/technotes/guides/net/articles/connection_release.html
                disableWrite();
            } else {
                // Just in case
                enableWrite();
            }
        }
    }
}
```

#### 连接响应

org.apache.zookeeper.ClientCnxnSocket#readConnectResult

```java
void readConnectResult() throws IOException {
    if (LOG.isTraceEnabled()) {
        StringBuilder buf = new StringBuilder("0x[");
        for (byte b : incomingBuffer.array()) {
            buf.append(Integer.toHexString(b) + ",");
        }
        buf.append("]");
        LOG.trace("readConnectResult " + incomingBuffer.remaining() + " "
                + buf.toString());
    }
    ByteBufferInputStream bbis = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
  	//读取incomingBuffer，填充ConnectResponse
    ConnectResponse conRsp = new ConnectResponse();
    conRsp.deserialize(bbia, "connect");

    // read "is read-only" flag
    boolean isRO = false;
    try {
        isRO = bbia.readBool("readOnly");
    } catch (IOException e) {
        LOG.warn("Connected to an old server; r-o mode will be unavailable");
    }
		//设置sessionId
    this.sessionId = conRsp.getSessionId();
    sendThread.onConnected(conRsp.getTimeOut(), this.sessionId,
            conRsp.getPasswd(), isRO);
}
```

org.apache.zookeeper.ClientCnxn.SendThread#onConnected

```java
void onConnected(int _negotiatedSessionTimeout, long _sessionId,
        byte[] _sessionPasswd, boolean isRO) throws IOException {
  	//服务端协商的会话过期时间
    negotiatedSessionTimeout = _negotiatedSessionTimeout;
    if (negotiatedSessionTimeout <= 0) {
        state = States.CLOSED;

        eventThread.queueEvent(new WatchedEvent(
                Watcher.Event.EventType.None,
                Watcher.Event.KeeperState.Expired, null));
        eventThread.queueEventOfDeath();

        String warnInfo;
        warnInfo = "Unable to reconnect to ZooKeeper service, session 0x"
            + Long.toHexString(sessionId) + " has expired";
        LOG.warn(warnInfo);
        throw new SessionExpiredException(warnInfo);
    }
    if (!readOnly && isRO) {
        LOG.error("Read/write client got connected to read-only server");
    }
  	//读超时
    readTimeout = negotiatedSessionTimeout * 2 / 3;
    //连接超时
    connectTimeout = negotiatedSessionTimeout / hostProvider.size();
    hostProvider.onConnected();
    sessionId = _sessionId;
    sessionPasswd = _sessionPasswd;
    //设置连接状态
    state = (isRO) ?
            States.CONNECTEDREADONLY : States.CONNECTED;
    seenRwServerBefore |= !isRO;
    LOG.info("Session establishment complete on server "
            + clientCnxnSocket.getRemoteSocketAddress()
            + ", sessionid = 0x" + Long.toHexString(sessionId)
            + ", negotiated timeout = " + negotiatedSessionTimeout
            + (isRO ? " (READ-ONLY mode)" : ""));
    //连接状态
    KeeperState eventState = (isRO) ?
            KeeperState.ConnectedReadOnly : KeeperState.SyncConnected;
 	 //触发Watcher通知、连接成功
    eventThread.queueEvent(new WatchedEvent(
            Watcher.Event.EventType.None,
            eventState, null));
}
```

#### 处理非连接响应

org.apache.zookeeper.ClientCnxn.SendThread#readResponse

```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {
  	//填充ReplyHeader
    ByteBufferInputStream bbis = new ByteBufferInputStream(
            incomingBuffer);
    BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
    ReplyHeader replyHdr = new ReplyHeader();
    replyHdr.deserialize(bbia, "header");
  
    if (replyHdr.getXid() == -2) { //ping响应
        // -2 is the xid for pings
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got ping response for sessionid: 0x"
                    + Long.toHexString(sessionId)
                    + " after "
                    + ((System.nanoTime() - lastPingSentNs) / 1000000)
                    + "ms");
        }
        return;
    }
    if (replyHdr.getXid() == -4) { //认证
        // -4 is the xid for AuthPacket               
        if(replyHdr.getErr() == KeeperException.Code.AUTHFAILED.intValue()) {
            state = States.AUTH_FAILED;                    
            eventThread.queueEvent( new WatchedEvent(Watcher.Event.EventType.None, 
                    Watcher.Event.KeeperState.AuthFailed, null) );                                  
        }
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got auth sessionid:0x"
                    + Long.toHexString(sessionId));
        }
        return;
    }
     //通知，创建WatchedEvent，交由eventThread处理
    if (replyHdr.getXid() == -1) { 
        // -1 means notification
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got notification sessionid:0x"
                + Long.toHexString(sessionId));
        }
        WatcherEvent event = new WatcherEvent();
        event.deserialize(bbia, "response");

        // convert from a server path to a client path
        if (chrootPath != null) {
            String serverPath = event.getPath();
            if(serverPath.compareTo(chrootPath)==0)
                event.setPath("/");
            else if (serverPath.length() > chrootPath.length())
                event.setPath(serverPath.substring(chrootPath.length()));
            else {
               LOG.warn("Got server path " + event.getPath()
                     + " which is too short for chroot path "
                     + chrootPath);
            }
        }

        WatchedEvent we = new WatchedEvent(event);
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got " + we + " for sessionid 0x"
                    + Long.toHexString(sessionId));
        }

        eventThread.queueEvent( we );
        return;
    }

    // If SASL authentication is currently in progress, construct and
    // send a response packet immediately, rather than queuing a
    // response as with other packets.
    if (clientTunneledAuthenticationInProgress()) {
        GetSASLRequest request = new GetSASLRequest();
        request.deserialize(bbia,"token");
        zooKeeperSaslClient.respondToServer(request.getToken(),
          ClientCnxn.this);
        return;
    }
		//从pendingQueue移除
    Packet packet;
    synchronized (pendingQueue) {
        if (pendingQueue.size() == 0) {
            throw new IOException("Nothing in the queue, but got "
                    + replyHdr.getXid());
        }
        packet = pendingQueue.remove();
    }
    /*
     * Since requests are processed in order, we better get a response
     * to the first request!
     请求顺序处理，返回响应对应的请求跟pendingQueue中的第一个请求匹配
     */
    try {
      	//请求的xid与响应的xid不匹配
        if (packet.requestHeader.getXid() != replyHdr.getXid()) {
            packet.replyHeader.setErr(
                    KeeperException.Code.CONNECTIONLOSS.intValue());
            throw new IOException("Xid out of order. Got Xid "
                    + replyHdr.getXid() + " with err " +
                    + replyHdr.getErr() +
                    " expected Xid "
                    + packet.requestHeader.getXid()
                    + " for a packet with details: "
                    + packet );
        }

        packet.replyHeader.setXid(replyHdr.getXid());
        packet.replyHeader.setErr(replyHdr.getErr());
        packet.replyHeader.setZxid(replyHdr.getZxid());
        if (replyHdr.getZxid() > 0) {
            lastZxid = replyHdr.getZxid();
        }
        if (packet.response != null && replyHdr.getErr() == 0) {
            packet.response.deserialize(bbia, "response");
        }

        if (LOG.isDebugEnabled()) {
            LOG.debug("Reading reply sessionid:0x"
                    + Long.toHexString(sessionId) + ", packet:: " + packet);
        }
    } finally {
        finishPacket(packet);//唤醒同步请求或者交由eventThread处理异步请求的回调方法
    }
}
```

org.apache.zookeeper.ClientCnxn#finishPacket

```java
private void finishPacket(Packet p) {
    if (p.watchRegistration != null) {
        p.watchRegistration.register(p.replyHeader.getErr());
    }

    if (p.cb == null) {//没有回调方法，同步发送
        synchronized (p) {
            p.finished = true;
            p.notifyAll(); //唤醒同步请求
        }
    } else { //异步发送，交由eventThread处理
        p.finished = true;
        eventThread.queuePacket(p);
    }
}
```

### 发送Ping请求

org.apache.zookeeper.ClientCnxn.SendThread#sendPing

```java
private void sendPing() {
    lastPingSentNs = System.nanoTime();
    RequestHeader h = new RequestHeader(-2, OpCode.ping);
    queuePacket(h, null, null, null, null, null, null, null, null);
}
```

org.apache.zookeeper.ClientCnxn#queuePacket

```java
Packet queuePacket(RequestHeader h, ReplyHeader r, Record request,
        Record response, AsyncCallback cb, String clientPath,
        String serverPath, Object ctx, WatchRegistration watchRegistration)
{
    Packet packet = null;

    // Note that we do not generate the Xid for the packet yet. It is
    // generated later at send-time, by an implementation of ClientCnxnSocket::doIO(),
    // where the packet is actually sent.
    synchronized (outgoingQueue) {
        packet = new Packet(h, r, request, response, watchRegistration);
        packet.cb = cb;
        packet.ctx = ctx;
        packet.clientPath = clientPath;
        packet.serverPath = serverPath;
        if (!state.isAlive() || closing) { //客户端已关闭
            conLossPacket(packet);
        } else {
            // If the client is asking to close the session then
            // mark as closing
            if (h.getType() == OpCode.closeSession) {
                closing = true;
            }
            outgoingQueue.add(packet); //放入发送列表
        }
    }
    sendThread.getClientCnxnSocket().wakeupCnxn();
    return packet;
}
```

## EventThread

org.apache.zookeeper.ClientCnxn.EventThread#run

```java
public void run() {
   try {
      isRunning = true;
      while (true) {
         Object event = waitingEvents.take();
         if (event == eventOfDeath) {//循环结束标志
            wasKilled = true;
         } else {
            processEvent(event); //处理Watcher事件、服务端返回的响应
         }
         if (wasKilled) //等待事件处理完成，再退出循环
            synchronized (waitingEvents) {
               if (waitingEvents.isEmpty()) {
                  isRunning = false;
                  break;
               }
            }
      }
   } catch (InterruptedException e) {
      LOG.error("Event thread exiting due to interruption", e);
   }

    LOG.info("EventThread shut down for session: 0x{}",
             Long.toHexString(getSessionId()));
}
```

### 生产Watch事件

org.apache.zookeeper.ClientCnxn.EventThread#queueEvent

```java
public void queueEvent(WatchedEvent event) {
    if (event.getType() == EventType.None
            && sessionState == event.getState()) {
        return;
    }
    sessionState = event.getState();

    // materialize the watchers based on the event
    WatcherSetEventPair pair = new WatcherSetEventPair(
            watcher.materialize(event.getState(), event.getType(),
                    event.getPath()),
                    event);
    // queue the pair (watch set & event) for later processing
    waitingEvents.add(pair);
}
```

### 生产响应事件

```java
public void queuePacket(Packet packet) {
   if (wasKilled) {
      synchronized (waitingEvents) {
         if (isRunning) waitingEvents.add(packet);
         else processEvent(packet);
      }
   } else {
      waitingEvents.add(packet);
   }
}
```

### 处理事件

处理Watch事件和回调方法

org.apache.zookeeper.ClientCnxn.EventThread#processEvent

```java
 private void processEvent(Object event) {
      try {
          if (event instanceof WatcherSetEventPair) { //通知
              // each watcher will process the event
              WatcherSetEventPair pair = (WatcherSetEventPair) event;
              for (Watcher watcher : pair.watchers) {
                  try {
                      watcher.process(pair.event); //执行Watcher
                  } catch (Throwable t) {
                      LOG.error("Error while calling watcher ", t);
                  }
              }
          } else { //其他响应,调用自定义的回调函数，处理的都是异步请求
              Packet p = (Packet) event; 
              int rc = 0;
              String clientPath = p.clientPath;
              if (p.replyHeader.getErr() != 0) {
                  rc = p.replyHeader.getErr();
              }
              if (p.cb == null) {
                  LOG.warn("Somehow a null cb got to EventThread!");
              } else if (p.response instanceof ExistsResponse
                      || p.response instanceof SetDataResponse
                      || p.response instanceof SetACLResponse) {
                  StatCallback cb = (StatCallback) p.cb;
                  if (rc == 0) {
                      if (p.response instanceof ExistsResponse) {
                          cb.processResult(rc, clientPath, p.ctx,
                                  ((ExistsResponse) p.response)
                                          .getStat());
                      } else if (p.response instanceof SetDataResponse) {
                          cb.processResult(rc, clientPath, p.ctx,
                                  ((SetDataResponse) p.response)
                                          .getStat());
                      } else if (p.response instanceof SetACLResponse) {
                          cb.processResult(rc, clientPath, p.ctx,
                                  ((SetACLResponse) p.response)
                                          .getStat());
                      }
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null);
                  }
              } else if (p.response instanceof GetDataResponse) {
                  DataCallback cb = (DataCallback) p.cb;
                  GetDataResponse rsp = (GetDataResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getData(), rsp.getStat());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null,
                              null);
                  }
              } else if (p.response instanceof GetACLResponse) {
                  ACLCallback cb = (ACLCallback) p.cb;
                  GetACLResponse rsp = (GetACLResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getAcl(), rsp.getStat());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null,
                              null);
                  }
              } else if (p.response instanceof GetChildrenResponse) {
                  ChildrenCallback cb = (ChildrenCallback) p.cb;
                  GetChildrenResponse rsp = (GetChildrenResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getChildren());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null);
                  }
              } else if (p.response instanceof GetChildren2Response) {
                  Children2Callback cb = (Children2Callback) p.cb;
                  GetChildren2Response rsp = (GetChildren2Response) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getChildren(), rsp.getStat());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null, null);
                  }
              } else if (p.response instanceof CreateResponse) {
                  StringCallback cb = (StringCallback) p.cb;
                  CreateResponse rsp = (CreateResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx,
                              (chrootPath == null
                                      ? rsp.getPath()
                                      : rsp.getPath()
                                .substring(chrootPath.length())));
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null);
                  }
              } else if (p.response instanceof MultiResponse) {
                      MultiCallback cb = (MultiCallback) p.cb;
                      MultiResponse rsp = (MultiResponse) p.response;
                      if (rc == 0) {
                              List<OpResult> results = rsp.getResultList();
                              int newRc = rc;
                              for (OpResult result : results) {
                                      if (result instanceof ErrorResult
                                          && KeeperException.Code.OK.intValue()
                                              != (newRc = ((ErrorResult) result).getErr())) {
                                              break;
                                      }
                              }
                              cb.processResult(newRc, clientPath, p.ctx, results);
                      } else {
                              cb.processResult(rc, clientPath, p.ctx, null);
                      }
              }  else if (p.cb instanceof VoidCallback) {
                  VoidCallback cb = (VoidCallback) p.cb;
                  cb.processResult(rc, clientPath, p.ctx);
              }
          }
      } catch (Throwable t) {
          LOG.error("Caught unexpected throwable", t);
      }
   }
}
```

## 创建节点

### 同步创建

org.apache.zookeeper.ZooKeeper#create(java.lang.String, byte[], java.util.List<ACL>, org.apache.zookeeper.CreateMode)

```java
public String create(final String path, byte data[], List<ACL> acl,
        CreateMode createMode)
    throws KeeperException, InterruptedException
{
    final String clientPath = path;
    PathUtils.validatePath(clientPath, createMode.isSequential());
		//客户端指定命名空间，创建的节点都在此目录下
    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(ZooDefs.OpCode.create); //请求类型
    CreateRequest request = new CreateRequest();
    CreateResponse response = new CreateResponse();
    request.setData(data); //请求数据
    request.setFlags(createMode.toFlag()); //永久、临时、顺序
    request.setPath(serverPath);
    if (acl != null && acl.size() == 0) {
        throw new KeeperException.InvalidACLException();
    }
    request.setAcl(acl);
  	//提交请求，放入队列，等待服务端返回响应
    ReplyHeader r = cnxn.submitRequest(h, request, response, null);
    if (r.getErr() != 0) {
        throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                clientPath);
    }
    if (cnxn.chrootPath == null) {
        return response.getPath();
    } else {
        return response.getPath().substring(cnxn.chrootPath.length());
    }
}
```

org.apache.zookeeper.ClientCnxn#submitRequest

```java
public ReplyHeader submitRequest(RequestHeader h, Record request,
        Record response, WatchRegistration watchRegistration)
        throws InterruptedException {
    ReplyHeader r = new ReplyHeader();
  //放入outgoingqueue列表
    Packet packet = queuePacket(h, r, request, response, null, null, null,
                null, watchRegistration);
  	//阻塞直到收到服务端的响应
    synchronized (packet) {
        while (!packet.finished) {
            packet.wait();
        }
    }
    return r;
}
```

```java
Packet queuePacket(RequestHeader h, ReplyHeader r, Record request,
        Record response, AsyncCallback cb, String clientPath,
        String serverPath, Object ctx, WatchRegistration watchRegistration)
{
    Packet packet = null;

    // Note that we do not generate the Xid for the packet yet. It is
    // generated later at send-time, by an implementation of ClientCnxnSocket::doIO(),
    // where the packet is actually sent.
    synchronized (outgoingQueue) {
        packet = new Packet(h, r, request, response, watchRegistration);
        packet.cb = cb;
        packet.ctx = ctx;
        packet.clientPath = clientPath;
        packet.serverPath = serverPath;
        if (!state.isAlive() || closing) { //连接异常
            conLossPacket(packet);
        } else {
            // 客户端发送close请求
            if (h.getType() == OpCode.closeSession) {
                closing = true;
            }
           //请求入队列
            outgoingQueue.add(packet);
        }
    }
  	//唤醒selector
    sendThread.getClientCnxnSocket().wakeupCnxn();
    return packet;
}
```



```java
private void conLossPacket(Packet p) { //连接异常
    if (p.replyHeader == null) {
        return;
    }
    switch (state) {
    case AUTH_FAILED:
        p.replyHeader.setErr(KeeperException.Code.AUTHFAILED.intValue());
        break;
    case CLOSED:
    p.replyHeader.setErr(KeeperException.Code.SESSIONEXPIRED.intValue());
        break;
    default:
        p.replyHeader.setErr(KeeperException.Code.CONNECTIONLOSS.intValue());
    }
    finishPacket(p);
}
```

```java
private void finishPacket(Packet p) {
    if (p.watchRegistration != null) {
        p.watchRegistration.register(p.replyHeader.getErr());
    }

    if (p.cb == null) { //未设置回调方法
        synchronized (p) {
            p.finished = true; //请求结束
            p.notifyAll(); //唤醒阻塞的请求
        }
    } else {
        p.finished = true;
        eventThread.queuePacket(p); //设置了回调方法，交由eventThread处理
    }
}
```

### 异步创建

```java
public void create(final String path, byte data[], List<ACL> acl,
        CreateMode createMode,  StringCallback cb, Object ctx)
{		//异步需要指定自定义的StringCallback，接收到响应信息后执行回调方法
    final String clientPath = path;
    PathUtils.validatePath(clientPath, createMode.isSequential());
	  
    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(ZooDefs.OpCode.create);
    CreateRequest request = new CreateRequest();
    CreateResponse response = new CreateResponse();
    ReplyHeader r = new ReplyHeader();
    request.setData(data);
    request.setFlags(createMode.toFlag());
    request.setPath(serverPath);
    request.setAcl(acl);
    cnxn.queuePacket(h, r, request, response, cb, clientPath,
            serverPath, ctx, null);
}
```

# 单机模式

## 启动

org.apache.zookeeper.server.ZooKeeperServerMain#main

```java
public void runFromConfig(ServerConfig config) throws IOException {
        LOG.info("Starting server");
        FileTxnSnapLog txnLog = null;
        try {
            final ZooKeeperServer zkServer = new ZooKeeperServer();
            // Registers shutdown handler which will be used to know the
            // server error or shutdown state changes.
            final CountDownLatch shutdownLatch = new CountDownLatch(1);
            zkServer.registerServerShutdownHandler(
                    new ZooKeeperServerShutdownHandler(shutdownLatch));
						// config.dataDir：快照目录
       			//config.dataLogDir：事务日志目录
        		//FileTxnSnapLog管理zk的数据存储
            txnLog = new FileTxnSnapLog(new File(config.dataLogDir), new File(
                    config.dataDir));
            txnLog.setServerStats(zkServer.serverStats());
          
            zkServer.setTxnLogFactory(txnLog);
            zkServer.setTickTime(config.tickTime);
            //根据以下两个参数调整客户端的会话超时时间
            zkServer.setMinSessionTimeout(config.minSessionTimeout);
            zkServer.setMaxSessionTimeout(config.maxSessionTimeout);
            //服务端通信工厂NettyServerCnxnFactory
            cnxnFactory = ServerCnxnFactory.createFactory();
           //设置端口、最大连接数
            cnxnFactory.configure(config.getClientPortAddress(),
                    config.getMaxClientCnxns());
            //绑定端口、加载快照日志到内存、开启会话追踪、设置请求处理器
            cnxnFactory.startup(zkServer);
            //阻塞，直到虚拟机退出
            shutdownLatch.await();
            shutdown();
            cnxnFactory.join();
            if (zkServer.canShutdown()) {
                zkServer.shutdown(true);
            }
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Server interrupted", e);
        } finally {
            if (txnLog != null) {
                txnLog.close();
            }
        }
    }
```

## 初始化

org.apache.zookeeper.server.NettyServerCnxnFactory#startup

```java
public void startup(ZooKeeperServer zks) throws IOException,
        InterruptedException {
    start(); //绑定端口
    setZooKeeperServer(zks);//设置ZooKeeperServer
    zks.startdata(); //加载快照、事务日志到内存
    zks.startup(); //会话追踪、设置请求处理器
}
```

### startdata

从磁盘加载数据，包括事务日志和会话

org.apache.zookeeper.server.ZooKeeperServer#startdata

```java
public void startdata() throws IOException, InterruptedException {
    //内存中维护sessions、dataTree
    if (zkDb == null) {
        zkDb = new ZKDatabase(this.txnLogFactory);
    }  
    //尚未初始化
    if (!zkDb.isInitialized()) { 
      	//加载数据、sessions
        loadData();
    }
}
```



```java
public void loadData() throws IOException, InterruptedException {
    //已经初始化完成
    if(zkDb.isInitialized()){
        setZxid(zkDb.getDataTreeLastProcessedZxid());
    }
    else { //尚未初始化，加载数据
        setZxid(zkDb.loadDataBase());
    }
    
    // 清除过期的session
    LinkedList<Long> deadSessions = new LinkedList<Long>();
    for (Long session : zkDb.getSessions()) { //临时节点关联的session
        if (zkDb.getSessionWithTimeOuts().get(session) == null) {
            deadSessions.add(session);
        }
    }
    //设置初始化完成的标志
    zkDb.setDataTreeInit(true);
    for (long session : deadSessions) { //关闭session
        // XXX: Is lastProcessedZxid really the best thing to use?
        killSession(session, zkDb.getDataTreeLastProcessedZxid());
    }
}
```

#### loadDataBase

org.apache.zookeeper.server.ZKDatabase#loadDataBase

```java
public long loadDataBase() throws IOException {	//加载快照和日志文件
    long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);
    initialized = true;
    return zxid;
}
```

org.apache.zookeeper.server.persistence.FileTxnSnapLog#restore

```java
public long restore(DataTree dt, Map<Long, Integer> sessions, 
        PlayBackListener listener) throws IOException {
  	//加载快照文件
    snapLog.deserialize(dt, sessions); 
  	//加载日志文件
    return fastForwardFromEdits(dt, sessions, listener);
}
```

org.apache.zookeeper.server.persistence.FileSnap#deserialize(org.apache.zookeeper.server.DataTree, java.util.Map<java.lang.Long,java.lang.Integer>)

```java
public long deserialize(DataTree dt, Map<Long, Integer> sessions)
        throws IOException {//加载快照
    // 查找最新的100个快照文件，按照由新到旧排序，
    List<File> snapList = findNValidSnapshots(100);
    if (snapList.size() == 0) {
        return -1L;
    }
    File snap = null;
    boolean foundValid = false;
    //只要找到一个完好无损的快照文件就退出循环
    for (int i = 0; i < snapList.size(); i++) {
        snap = snapList.get(i);
        InputStream snapIS = null;
        CheckedInputStream crcIn = null;
        try {
            LOG.info("Reading snapshot " + snap);
            snapIS = new BufferedInputStream(new FileInputStream(snap));
            crcIn = new CheckedInputStream(snapIS, new Adler32());
            InputArchive ia = BinaryInputArchive.getArchive(crcIn);
            deserialize(dt,sessions, ia);
            long checkSum = crcIn.getChecksum().getValue();
            long val = ia.readLong("val");
            if (val != checkSum) {
                throw new IOException("CRC corruption in snapshot :  " + snap);
            }
            foundValid = true;
            break;
        } catch(IOException e) {
            LOG.warn("problem reading snap file " + snap, e);
        } finally {
            if (snapIS != null) 
                snapIS.close();
            if (crcIn != null) 
                crcIn.close();
        } 
    }
    if (!foundValid) {
        throw new IOException("Not able to find valid snapshots in " + snapDir);
    }
    //快照文件最大的zxId
    dt.lastProcessedZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
    return dt.lastProcessedZxid;
}
```

org.apache.zookeeper.server.persistence.FileTxnSnapLog#fastForwardFromEdits

```java
public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions,
                                 PlayBackListener listener) throws IOException { 
  	//dataDir：日志文件目录
    FileTxnLog txnLog = new FileTxnLog(dataDir);
    //加载快照生成之后，新增的日志文件
    TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);
  	//快照中的最大zxId
    long highestZxid = dt.lastProcessedZxid;
    TxnHeader hdr;
    try {
        while (true) {
            // iterator points to 
            // the first valid txn when initialized
            hdr = itr.getHeader();
            if (hdr == null) {
                //empty logs 
                return dt.lastProcessedZxid;
            }
            if (hdr.getZxid() < highestZxid && highestZxid != 0) {
                LOG.error("{}(higestZxid) > {}(next log) for type {}",
                        new Object[] { highestZxid, hdr.getZxid(),
                                hdr.getType() });
            } else {
                highestZxid = hdr.getZxid();
            }
            try {
                processTransaction(hdr,dt,sessions, itr.getTxn());
            } catch(KeeperException.NoNodeException e) {
               throw new IOException("Failed to process transaction type: " +
                     hdr.getType() + " error: " + e.getMessage(), e);
            }
            //将被提交的日志写入内存列表，方便数据的同步
            listener.onTxnLoaded(hdr, itr.getTxn());
            if (!itr.next()) 
                break;
        }
    } finally {
        if (itr != null) {
            itr.close();
        }
    }
    return highestZxid;
}
```

#### killSession

org.apache.zookeeper.server.ZooKeeperServer#killSession

```java
protected void killSession(long sessionId, long zxid) { //关闭过期的会话
   //移除会话关联的临时节点
    zkDb.killSession(sessionId, zxid); 
    if (LOG.isTraceEnabled()) {
        ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                                     "ZooKeeperServer --- killSession: 0x"
                + Long.toHexString(sessionId));
    }
    //移除会话
    if (sessionTracker != null) {
        sessionTracker.removeSession(sessionId);
    }
}
```

### startup

org.apache.zookeeper.server.ZooKeeperServer#startup

```java
public synchronized void startup() {
    if (sessionTracker == null) {
        createSessionTracker();
    }
    startSessionTracker();
    setupRequestProcessors();
    registerJMX();
    setState(State.RUNNING);
    notifyAll();
}
```

#### createSessionTracker

```java
protected void createSessionTracker() {//创建会话追踪
    sessionTracker = new SessionTrackerImpl(this, zkDb.getSessionWithTimeOuts(),
            tickTime, 1, getZooKeeperServerListener());
}
```

```java
protected void startSessionTracker() {//启动会话追踪，清除过期会话
    ((SessionTrackerImpl)sessionTracker).start();
}
```

#### setupRequestProcessors

责任链模式（PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor）

```java
protected void setupRequestProcessors() {//设置请求处理器
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor syncProcessor = new SyncRequestProcessor(this,
            finalProcessor);
    ((SyncRequestProcessor)syncProcessor).start();
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}
```

## 处理连接事件

org.apache.zookeeper.server.NettyServerCnxnFactory.CnxnChannelHandler#channelConnected

```java
public void channelConnected(ChannelHandlerContext ctx,
        ChannelStateEvent e) throws Exception
{
    if (LOG.isTraceEnabled()) {
        LOG.trace("Channel connected " + e);
    }
  	//管理所有的channel
    allChannels.add(ctx.getChannel());
  	//为每个连接创建NettyServerCnxn
    NettyServerCnxn cnxn = new NettyServerCnxn(ctx.getChannel(),
            zkServer, NettyServerCnxnFactory.this);
    ctx.setAttachment(cnxn);
  	//remoteAddress -> Set<NettyServerCnxn>
    addCnxn(cnxn);
}
```

```java
private void addCnxn(NettyServerCnxn cnxn) {
    synchronized (cnxns) {
        cnxns.add(cnxn);
        synchronized (ipMap){
            InetAddress addr =
                ((InetSocketAddress)cnxn.channel.getRemoteAddress())
                    .getAddress();
            Set<NettyServerCnxn> s = ipMap.get(addr);
            if (s == null) {
                s = new HashSet<NettyServerCnxn>();
            }
            s.add(cnxn);
            ipMap.put(addr,s);
        }
    }
}
```

## 接收请求

org.apache.zookeeper.server.NettyServerCnxnFactory.CnxnChannelHandler#messageReceived

```java
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e)
    throws Exception
{
    if (LOG.isTraceEnabled()) {
        LOG.trace("message received called " + e.getMessage());
    }
    try {
        if (LOG.isDebugEnabled()) {
            LOG.debug("New message " + e.toString()
                    + " from " + ctx.getChannel());
        }
        NettyServerCnxn cnxn = (NettyServerCnxn)ctx.getAttachment();
        synchronized(cnxn) {
            processMessage(e, cnxn);
        }
    } catch(Exception ex) {
        LOG.error("Unexpected exception in receive", ex);
        throw ex;
    }
}
```

## 请求处理

org.apache.zookeeper.server.NettyServerCnxn#receiveMessage

```java
if (initialized) { //已经初始化，处理非连接请求
    zks.processPacket(this, bb);
    if (zks.shouldThrottle(outstandingCount.incrementAndGet())) {//限流，默认1000
        disableRecvNoWait(); //禁止读
    }
} else { //尚未初始化，处理连接请求
    LOG.debug("got conn req request from "
            + getRemoteSocketAddress());
    zks.processConnectRequest(this, bb);
    initialized = true;
}
```

### 处理连接请求

org.apache.zookeeper.server.ZooKeeperServer#processConnectRequest

```java
public void processConnectRequest(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    BinaryInputArchive bia = BinaryInputArchive.getArchive(new ByteBufferInputStream(incomingBuffer));
    ConnectRequest connReq = new ConnectRequest();
    connReq.deserialize(bia, "connect");
    if (LOG.isDebugEnabled()) {
        LOG.debug("Session establishment request from client "
                + cnxn.getRemoteSocketAddress()
                + " client's lastZxid is 0x"
                + Long.toHexString(connReq.getLastZxidSeen()));
    }
    boolean readOnly = false;
    try {
        readOnly = bia.readBool("readOnly");
        cnxn.isOldClient = false;
    } catch (IOException e) {
        // this is ok -- just a packet from an old client which
        // doesn't contain readOnly field
        LOG.warn("Connection request from old client "
                + cnxn.getRemoteSocketAddress()
                + "; will be dropped if server is in r-o mode");
    }
    if (readOnly == false && this instanceof ReadOnlyZooKeeperServer) {
        String msg = "Refusing session request for not-read-only client "
            + cnxn.getRemoteSocketAddress();
        LOG.info(msg);
        throw new CloseRequestException(msg);
    }
    if (connReq.getLastZxidSeen() > zkDb.dataTree.lastProcessedZxid) {
        String msg = "Refusing session request for client "
            + cnxn.getRemoteSocketAddress()
            + " as it has seen zxid 0x"
            + Long.toHexString(connReq.getLastZxidSeen())
            + " our last zxid is 0x"
            + Long.toHexString(getZKDatabase().getDataTreeLastProcessedZxid())
            + " client must try another server";
        LOG.info(msg);
        throw new CloseRequestException(msg);
    }
    //获取客户端会话超时时间
    int sessionTimeout = connReq.getTimeOut();
    byte passwd[] = connReq.getPasswd();
    //服务端最小超时时间
    int minSessionTimeout = getMinSessionTimeout();
    if (sessionTimeout < minSessionTimeout) {
        sessionTimeout = minSessionTimeout;
    }
    //服务端最大超时时间
    int maxSessionTimeout = getMaxSessionTimeout();
    if (sessionTimeout > maxSessionTimeout) {
        sessionTimeout = maxSessionTimeout;
    }
    //重新设置超时时间（minSessionTimeou<=sessionTimeout<=maxSessionTimeout）
    cnxn.setSessionTimeout(sessionTimeout);
    // We don't want to receive any packets until we are sure that the
    // session is setup
    cnxn.disableRecv();
    long sessionId = connReq.getSessionId();
    if (sessionId != 0) { //非首次，断开重连
        long clientSessionId = connReq.getSessionId();
        LOG.info("Client attempting to renew session 0x"
                + Long.toHexString(clientSessionId)
                + " at " + cnxn.getRemoteSocketAddress());
        //关闭旧的会话
        serverCnxnFactory.closeSession(sessionId);
        cnxn.setSessionId(sessionId);
        //重新打开会话
        reopenSession(cnxn, sessionId, passwd, sessionTimeout);
    } else { //首次，创建会话
        LOG.info("Client attempting to establish new session at "
                + cnxn.getRemoteSocketAddress());
        createSession(cnxn, passwd, sessionTimeout);
    }
}
```

#### 首次创建会话

```java
long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {
    //创建session，进行session追踪
    long sessionId = sessionTracker.createSession(timeout);
    Random r = new Random(sessionId ^ superSecret);
    r.nextBytes(passwd);
  	//过期时间
    ByteBuffer to = ByteBuffer.allocate(4);
    to.putInt(timeout);
    cnxn.setSessionId(sessionId);
    //提交创建session的请求
    submitRequest(cnxn, sessionId, OpCode.createSession, 0, to, null);
    return sessionId;
}
```

#### 提交创建会话的请求

```java
private void submitRequest(ServerCnxn cnxn, long sessionId, int type,
        int xid, ByteBuffer bb, List<Id> authInfo) {
    Request si = new Request(cnxn, sessionId, xid, type, bb, authInfo);
    submitRequest(si);
}
```



```java
public void submitRequest(Request si) {
    //等待ZookeeperServer初始化完成
    if (firstProcessor == null) {
        synchronized (this) {
            try {
                // Since all requests are passed to the request
                // processor it should wait for setting up the request
                // processor chain. The state will be updated to RUNNING
                // after the setup.
                while (state == State.INITIAL) {
                    wait(1000);
                }
            } catch (InterruptedException e) {
                LOG.warn("Unexpected interruption", e);
            }
            if (firstProcessor == null || state != State.RUNNING) {
                throw new RuntimeException("Not started");
            }
        }
    }
    try {
        //对会话进行续约
        touch(si.cnxn);
        //判断请求的合法性
        boolean validpacket = Request.isValid(si.type);
        if (validpacket) {
            //将请求放入PrepRequestProcessor的队列中
            firstProcessor.processRequest(si);
            if (si.cnxn != null) {
                incInProcess(); //处理中的请求数+1；在FinalRequestProcessor中处理结束，请求数-1，用于实现限流
            }
        } else { //请求不合法
            LOG.warn("Received packet at server of unknown type " + si.type);
            new UnimplementedRequestProcessor().processRequest(si);
        }
    } catch (MissingSessionException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Dropping request: " + e.getMessage());
        }
    } catch (RequestProcessorException e) {
        LOG.error("Unable to process request:" + e.getMessage(), e);
    }
}
```

#### 创建会话

org.apache.zookeeper.server.PrepRequestProcessor#pRequest2Txn

```java
case OpCode.createSession: //处理创建会话
    request.request.rewind();
    int to = request.request.getInt();
    request.txn = new CreateSessionTxn(to);
    request.request.rewind();
		//对会话进行管理
    zks.sessionTracker.addSession(request.sessionId, to);
    zks.setOwner(request.sessionId, request.getOwner());
    break;
```

#### 管理会话

org.apache.zookeeper.server.SessionTrackerImpl#addSession

```java
synchronized public void addSession(long id, int sessionTimeout) {//维护会话
  	//sessionID -> sessionTimeout
    sessionsWithTimeout.put(id, sessionTimeout);
    if (sessionsById.get(id) == null) { //首次创建
        SessionImpl s = new SessionImpl(id, sessionTimeout, 0);
      	//sessionID -> SessionImpl
        sessionsById.put(id, s);
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                    "SessionTrackerImpl --- Adding session 0x"
                    + Long.toHexString(id) + " " + sessionTimeout);
        }
    } else {
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                    "SessionTrackerImpl --- Existing session 0x"
                    + Long.toHexString(id) + " " + sessionTimeout);
        }
    }
    touchSession(id, sessionTimeout);
}
```



```java
synchronized public boolean touchSession(long sessionId, int timeout) {
    if (LOG.isTraceEnabled()) {
        ZooTrace.logTraceMessage(LOG,
                                 ZooTrace.CLIENT_PING_TRACE_MASK,
                                 "SessionTrackerImpl --- Touch session: 0x"
                + Long.toHexString(sessionId) + " with timeout " + timeout);
    }
  	//获取SessionImpl
    SessionImpl s = sessionsById.get(sessionId);
    // Return false, if the session doesn't exists or marked as closing
    if (s == null || s.isClosing()) {
        return false;
    }
    //计算超时时间，对超时时间进行规整，为expirationInterval的整数倍，便于分桶管理会话
    long expireTime = roundToInterval(Time.currentElapsedTime() + timeout);
    if (s.tickTime >= expireTime) { //首次创建时，tickTime为0
        // Nothing needs to be done 
        return true;
    }
  	//维护了某个时间点超时的会话集合
    // HashMap<Long, SessionSet> sessionSets = new HashMap<Long, SessionSet>();
    SessionSet set = sessionSets.get(s.tickTime);
    if (set != null) {
        set.sessions.remove(s);
    }
  	//设置规整的超时时间
    s.tickTime = expireTime;
    set = sessionSets.get(s.tickTime);
  	//对会话进行维护
    if (set == null) {
        set = new SessionSet();
        sessionSets.put(expireTime, set);
    }
    set.sessions.add(s);
    return true;
}
```

```java
private long roundToInterval(long time) {
    // We give a one interval grace period
    return (time / expirationInterval + 1) * expirationInterval;
}
```

###  处理非连接请求

org.apache.zookeeper.server.ZooKeeperServer#processPacket

```java
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    // We have the request, now process and setup for next
    InputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    h.deserialize(bia, "header");
    // Through the magic of byte buffers, txn will not be
    // pointing
    // to the start of the txn
    incomingBuffer = incomingBuffer.slice();
    if (h.getType() == OpCode.auth) { //认证
        LOG.info("got auth packet " + cnxn.getRemoteSocketAddress());
        AuthPacket authPacket = new AuthPacket();
        ByteBufferInputStream.byteBuffer2Record(incomingBuffer, authPacket);
        String scheme = authPacket.getScheme();
        AuthenticationProvider ap = ProviderRegistry.getProvider(scheme);
        Code authReturn = KeeperException.Code.AUTHFAILED;
        if(ap != null) {
            try {
                authReturn = ap.handleAuthentication(cnxn, authPacket.getAuth());
            } catch(RuntimeException e) {
                LOG.warn("Caught runtime exception from AuthenticationProvider: " + scheme + " due to " + e);
                authReturn = KeeperException.Code.AUTHFAILED;                   
            }
        }
        if (authReturn!= KeeperException.Code.OK) {
            if (ap == null) {
                LOG.warn("No authentication provider for scheme: "
                        + scheme + " has "
                        + ProviderRegistry.listProviders());
            } else {
                LOG.warn("Authentication failed for scheme: " + scheme);
            }
            // send a response...
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
                    KeeperException.Code.AUTHFAILED.intValue());
            cnxn.sendResponse(rh, null, null);
            // ... and close connection
            cnxn.sendBuffer(ServerCnxnFactory.closeConn);
            cnxn.disableRecv();
        } else {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Authentication succeeded for scheme: "
                          + scheme);
            }
            LOG.info("auth success " + cnxn.getRemoteSocketAddress());
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
                    KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh, null, null);
        }
        return;
    } else {
        if (h.getType() == OpCode.sasl) {
            Record rsp = processSasl(incomingBuffer,cnxn);
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh,rsp, "response"); // not sure about 3rd arg..what is it?
            return;
        }
        else {//非认证请求
            Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(),
              h.getType(), incomingBuffer, cnxn.getAuthInfo());
            si.setOwner(ServerCnxn.me);
            submitRequest(si);
        }
    }
    cnxn.incrOutstandingRequests(h);
}
```

PrepRequestProcessor -> SyncRequestProcessor ->FinalRequestProcessor

PrepRequestProcessor

请求预处理器，把请求区分为非事务请求和事务类请求（改变服务器状态的请求），对事务类请求进行一系列的预处理。

SyncRequestProcessor

事务日志处理器，将事务请求记录到事务日志文件中，同时还会触发Zookeeper异步进行数据快照。

FinalRequestProcessor

最后一个处理器，主要负责进行客户端请求返回之前的收尾工作，创建请求的响应、将事务类请求应用到内存

org.apache.zookeeper.server.ZooKeeperServer#submitRequest(org.apache.zookeeper.server.Request)

```java
public void submitRequest(Request si) {
    //等待ZookeeperServer初始化完成
    if (firstProcessor == null) {
        synchronized (this) {
            try {
                // Since all requests are passed to the request
                // processor it should wait for setting up the request
                // processor chain. The state will be updated to RUNNING
                // after the setup.
                while (state == State.INITIAL) {
                    wait(1000);
                }
            } catch (InterruptedException e) {
                LOG.warn("Unexpected interruption", e);
            }
            if (firstProcessor == null || state != State.RUNNING) {
                throw new RuntimeException("Not started");
            }
        }
    }
    try {
        //对会话进行续约
        touch(si.cnxn);
        boolean validpacket = Request.isValid(si.type);
        if (validpacket) {
            //PrepRequestProcessor:存放到队列中
            firstProcessor.processRequest(si);
            if (si.cnxn != null) {
                incInProcess(); //处理中的请求数+1；在FinalRequestProcessor中处理结束，请求数-1
            }
        } else {
            LOG.warn("Received packet at server of unknown type " + si.type);
            new UnimplementedRequestProcessor().processRequest(si);
        }
    } catch (MissingSessionException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Dropping request: " + e.getMessage());
        }
    } catch (RequestProcessorException e) {
        LOG.error("Unable to process request:" + e.getMessage(), e);
    }
}
```

会话续约

org.apache.zookeeper.server.ZooKeeperServer#touch

```java
void touch(ServerCnxn cnxn) throws MissingSessionException {
    if (cnxn == null) {
        return;
    }
    long id = cnxn.getSessionId();
    int to = cnxn.getSessionTimeout();
    if (!sessionTracker.touchSession(id, to)) { //续约
        throw new MissingSessionException(
                "No session with sessionid 0x" + Long.toHexString(id)
                + " exists, probably expired and removed");
    }
}
```

#### PrepRequestProcessor

org.apache.zookeeper.server.PrepRequestProcessor#run

```java
public void run() {
    try {
        while (true) {
            Request request = submittedRequests.take(); //获取请求
            long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
            if (request.type == OpCode.ping) {
                traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
            }
            if (LOG.isTraceEnabled()) {
                ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
            }
            if (Request.requestOfDeath == request) {
                break;
            }
            pRequest(request); //预处理请求
        }
    } catch (RequestProcessorException e) {
        if (e.getCause() instanceof XidRolloverException) {
            LOG.info(e.getCause().getMessage());
        }
        handleException(this.getName(), e);
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
    LOG.info("PrepRequestProcessor exited loop!");
}
```



```java
protected void pRequest(Request request) throws RequestProcessorException {
    // LOG.info("Prep>>> cxid = " + request.cxid + " type = " +
    // request.type + " id = 0x" + Long.toHexString(request.sessionId));
    request.hdr = null;
    request.txn = null;
    
    try {
      //事务类请求，根据请求类型创建对应的Record，创建TxnHeader，区分事务类或非事务类请求、校验session有效性、其他的校验工作
        switch (request.type) {
            case OpCode.create:
            CreateRequest createRequest = new CreateRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, createRequest, true);
            break;
        case OpCode.delete:
            DeleteRequest deleteRequest = new DeleteRequest();               
            pRequest2Txn(request.type, zks.getNextZxid(), request, deleteRequest, true);
            break;
        case OpCode.setData:
            SetDataRequest setDataRequest = new SetDataRequest();                
            pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
            break;
        case OpCode.setACL:
            SetACLRequest setAclRequest = new SetACLRequest();                
            pRequest2Txn(request.type, zks.getNextZxid(), request, setAclRequest, true);
            break;
        case OpCode.check:
            CheckVersionRequest checkRequest = new CheckVersionRequest();              
            pRequest2Txn(request.type, zks.getNextZxid(), request, checkRequest, true);
            break;
        case OpCode.multi:
            MultiTransactionRecord multiRequest = new MultiTransactionRecord();
            try {
                ByteBufferInputStream.byteBuffer2Record(request.request, multiRequest);
            } catch(IOException e) {
                request.hdr =  new TxnHeader(request.sessionId, request.cxid, zks.getNextZxid(),
                        Time.currentWallTime(), OpCode.multi);
                throw e;
            }
            List<Txn> txns = new ArrayList<Txn>();
            //Each op in a multi-op must have the same zxid!
            long zxid = zks.getNextZxid();
            KeeperException ke = null;

            //Store off current pending change records in case we need to rollback
            HashMap<String, ChangeRecord> pendingChanges = getPendingChanges(multiRequest);

            int index = 0;
            for(Op op: multiRequest) {
                Record subrequest = op.toRequestRecord() ;

                /* If we've already failed one of the ops, don't bother
                 * trying the rest as we know it's going to fail and it
                 * would be confusing in the logfiles.
                 */
                if (ke != null) {
                    request.hdr.setType(OpCode.error);
                    request.txn = new ErrorTxn(Code.RUNTIMEINCONSISTENCY.intValue());
                } 
                
                /* Prep the request and convert to a Txn */
                else {
                    try {
                        pRequest2Txn(op.getType(), zxid, request, subrequest, false);
                    } catch (KeeperException e) {
                        ke = e;
                        request.hdr.setType(OpCode.error);
                        request.txn = new ErrorTxn(e.code().intValue());
                        LOG.info("Got user-level KeeperException when processing "
                              + request.toString() + " aborting remaining multi ops."
                              + " Error Path:" + e.getPath()
                              + " Error:" + e.getMessage());

                        request.setException(e);

                        /* Rollback change records from failed multi-op */
                        rollbackPendingChanges(zxid, pendingChanges);
                    }
                }

                //FIXME: I don't want to have to serialize it here and then
                //       immediately deserialize in next processor. But I'm 
                //       not sure how else to get the txn stored into our list.
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                request.txn.serialize(boa, "request") ;
                ByteBuffer bb = ByteBuffer.wrap(baos.toByteArray());

                txns.add(new Txn(request.hdr.getType(), bb.array()));
                index++;
            }

            request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid,
                    Time.currentWallTime(), request.type);
            request.txn = new MultiTxn(txns);
            
            break;

        //create/close session don't require request record
        case OpCode.createSession:
        case OpCode.closeSession:
            pRequest2Txn(request.type, zks.getNextZxid(), request, null, true);
            break;
				//非事务操作，不需要写入日志文件，只需验证会话的有效性
        //All the rest don't need to create a Txn - just verify session
        case OpCode.sync:
        case OpCode.exists:
        case OpCode.getData:
        case OpCode.getACL:
        case OpCode.getChildren:
        case OpCode.getChildren2:
        case OpCode.ping:
        case OpCode.setWatches:
            zks.sessionTracker.checkSession(request.sessionId,
                    request.getOwner());
            break;
        default:
            LOG.warn("unknown type " + request.type);
            break;
        }
    } catch (KeeperException e) {
        if (request.hdr != null) {
            request.hdr.setType(OpCode.error);
            request.txn = new ErrorTxn(e.code().intValue());
        }
        LOG.info("Got user-level KeeperException when processing "
                + request.toString()
                + " Error Path:" + e.getPath()
                + " Error:" + e.getMessage());
        request.setException(e);
    } catch (Exception e) {
        // log at error level as we are returning a marshalling
        // error to the user
        LOG.error("Failed to process " + request, e);

        StringBuilder sb = new StringBuilder();
        ByteBuffer bb = request.request;
        if(bb != null){
            bb.rewind();
            while (bb.hasRemaining()) {
                sb.append(Integer.toHexString(bb.get() & 0xff));
            }
        } else {
            sb.append("request buffer is null");
        }

        LOG.error("Dumping request buffer: 0x" + sb.toString());
        if (request.hdr != null) {
            request.hdr.setType(OpCode.error);
            request.txn = new ErrorTxn(Code.MARSHALLINGERROR.intValue());
        }
    }
  	//当前的事务ID
    request.zxid = zks.getZxid();
  	//调用SyncRequestProcessor处理请求
    nextProcessor.processRequest(request);
}
```

org.apache.zookeeper.server.PrepRequestProcessor#pRequest2Txn

```java
 protected void pRequest2Txn(int type, long zxid, Request request, Record record, boolean deserialize)  throws KeeperException, IOException, RequestProcessorException
{//处理事务操作，填充事务请求头
   //是否写入日志的标志，事务操作和会话操作需要写入日志
    request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid,
                                Time.currentWallTime(), type);

    switch (type) {
        case OpCode.create:        
        	 //检查会话的有效性
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            CreateRequest createRequest = (CreateRequest)record;   
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, createRequest);
            String path = createRequest.getPath();
            int lastSlash = path.lastIndexOf('/');
            if (lastSlash == -1 || path.indexOf('\0') != -1 || failCreate) {
                LOG.info("Invalid path " + path + " with session 0x" +
                        Long.toHexString(request.sessionId));
                throw new KeeperException.BadArgumentsException(path);
            }
            //
            List<ACL> listACL = removeDuplicates(createRequest.getAcl());
            if (!fixupACL(request.authInfo, listACL)) {
                throw new KeeperException.InvalidACLException(path);
            }
            String parentPath = path.substring(0, lastSlash);
            ChangeRecord parentRecord = getRecordForPath(parentPath);

            checkACL(zks, parentRecord.acl, ZooDefs.Perms.CREATE,
                    request.authInfo);
            int parentCVersion = parentRecord.stat.getCversion();
            CreateMode createMode =
                CreateMode.fromFlag(createRequest.getFlags());
            if (createMode.isSequential()) {
                path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
            }
            validatePath(path, request.sessionId);
            try {
                if (getRecordForPath(path) != null) {
                    throw new KeeperException.NodeExistsException(path);
                }
            } catch (KeeperException.NoNodeException e) {
                // ignore this one
            }
            boolean ephemeralParent = parentRecord.stat.getEphemeralOwner() != 0;
            if (ephemeralParent) {
                throw new KeeperException.NoChildrenForEphemeralsException(path);
            }
            int newCversion = parentRecord.stat.getCversion()+1;
            request.txn = new CreateTxn(path, createRequest.getData(),
                    listACL,
                    createMode.isEphemeral(), newCversion);
            StatPersisted s = new StatPersisted();
            if (createMode.isEphemeral()) {
                s.setEphemeralOwner(request.sessionId);
            }
            parentRecord = parentRecord.duplicate(request.hdr.getZxid());
            parentRecord.childCount++;
            parentRecord.stat.setCversion(newCversion);
            addChangeRecord(parentRecord);
            addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path, s,
                    0, listACL));
            break;
        case OpCode.delete:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            DeleteRequest deleteRequest = (DeleteRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, deleteRequest);
            path = deleteRequest.getPath();
            lastSlash = path.lastIndexOf('/');
            if (lastSlash == -1 || path.indexOf('\0') != -1
                    || zks.getZKDatabase().isSpecialPath(path)) {
                throw new KeeperException.BadArgumentsException(path);
            }
            parentPath = path.substring(0, lastSlash);
            parentRecord = getRecordForPath(parentPath);
            ChangeRecord nodeRecord = getRecordForPath(path);
            checkACL(zks, parentRecord.acl, ZooDefs.Perms.DELETE,
                    request.authInfo);
            int version = deleteRequest.getVersion();
            if (version != -1 && nodeRecord.stat.getVersion() != version) {
                throw new KeeperException.BadVersionException(path);
            }
            if (nodeRecord.childCount > 0) {
                throw new KeeperException.NotEmptyException(path);
            }
            request.txn = new DeleteTxn(path);
            parentRecord = parentRecord.duplicate(request.hdr.getZxid());
            parentRecord.childCount--;
            addChangeRecord(parentRecord);
            addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path,
                    null, -1, null));
            break;
        case OpCode.setData:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            SetDataRequest setDataRequest = (SetDataRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, setDataRequest);
            path = setDataRequest.getPath();
            validatePath(path, request.sessionId);
            nodeRecord = getRecordForPath(path);
            checkACL(zks, nodeRecord.acl, ZooDefs.Perms.WRITE,
                    request.authInfo);
            version = setDataRequest.getVersion();
            int currentVersion = nodeRecord.stat.getVersion();
            if (version != -1 && version != currentVersion) {
                throw new KeeperException.BadVersionException(path);
            }
            version = currentVersion + 1;
            request.txn = new SetDataTxn(path, setDataRequest.getData(), version);
            nodeRecord = nodeRecord.duplicate(request.hdr.getZxid());
            nodeRecord.stat.setVersion(version);
            addChangeRecord(nodeRecord);
            break;
        case OpCode.setACL:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            SetACLRequest setAclRequest = (SetACLRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, setAclRequest);
            path = setAclRequest.getPath();
            validatePath(path, request.sessionId);
            listACL = removeDuplicates(setAclRequest.getAcl());
            if (!fixupACL(request.authInfo, listACL)) {
                throw new KeeperException.InvalidACLException(path);
            }
            nodeRecord = getRecordForPath(path);
            checkACL(zks, nodeRecord.acl, ZooDefs.Perms.ADMIN,
                    request.authInfo);
            version = setAclRequest.getVersion();
            currentVersion = nodeRecord.stat.getAversion();
            if (version != -1 && version != currentVersion) {
                throw new KeeperException.BadVersionException(path);
            }
            version = currentVersion + 1;
            request.txn = new SetACLTxn(path, listACL, version);
            nodeRecord = nodeRecord.duplicate(request.hdr.getZxid());
            nodeRecord.stat.setAversion(version);
            addChangeRecord(nodeRecord);
            break;
        case OpCode.createSession:
            request.request.rewind();
            int to = request.request.getInt();
            request.txn = new CreateSessionTxn(to);
            request.request.rewind();
            zks.sessionTracker.addSession(request.sessionId, to);
            zks.setOwner(request.sessionId, request.getOwner());
            break;
        case OpCode.closeSession:
            // We don't want to do this check since the session expiration thread
            // queues up this operation without being the session owner.
            // this request is the last of the session so it should be ok
            //zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            HashSet<String> es = zks.getZKDatabase()
                    .getEphemerals(request.sessionId);
            synchronized (zks.outstandingChanges) {
                for (ChangeRecord c : zks.outstandingChanges) {
                    if (c.stat == null) {
                        // Doing a delete
                        es.remove(c.path);
                    } else if (c.stat.getEphemeralOwner() == request.sessionId) {
                        es.add(c.path);
                    }
                }
                for (String path2Delete : es) {
                    addChangeRecord(new ChangeRecord(request.hdr.getZxid(),
                            path2Delete, null, 0, null));
                }

                zks.sessionTracker.setSessionClosing(request.sessionId);
            }

            LOG.info("Processed session termination for sessionid: 0x"
                    + Long.toHexString(request.sessionId));
            break;
        case OpCode.check:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            CheckVersionRequest checkVersionRequest = (CheckVersionRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, checkVersionRequest);
            path = checkVersionRequest.getPath();
            validatePath(path, request.sessionId);
            nodeRecord = getRecordForPath(path);
            checkACL(zks, nodeRecord.acl, ZooDefs.Perms.READ,
                    request.authInfo);
            version = checkVersionRequest.getVersion();
            currentVersion = nodeRecord.stat.getVersion();
            if (version != -1 && version != currentVersion) {
                throw new KeeperException.BadVersionException(path);
            }
            version = currentVersion + 1;
            request.txn = new CheckVersionTxn(path, version);
            break;
        default:
            LOG.error("Invalid OpCode: {} received by PrepRequestProcessor", type);
    }
}
```

#### SyncRequestProcessor

org.apache.zookeeper.server.SyncRequestProcessor#run

```java
public void run() {
    try {
        int logCount = 0;
        //随机化做快照的间隔（做快照之前积压的日志数量），避免全部节点同时做快照
        setRandRoll(r.nextInt(snapCount/2));
        while (true) {
            Request si = null;
            //无刷新请求时，阻塞获取
            if (toFlush.isEmpty()) {
                si = queuedRequests.take();//阻塞获取
            } else {
                si = queuedRequests.poll(); //无阻塞获取
                if (si == null) { //无请求入队
                    flush(toFlush); //刷新请求，刷盘、调用nextProcessor
                    continue;
                }
            }
            if (si == requestOfDeath) {
                break;
            }
            if (si != null) {
                //写事务日志,非事务操作返回false
                if (zks.getZKDatabase().append(si)) {
                    logCount++;
                    //生成快照
                    if (logCount > (snapCount / 2 + randRoll)) {
                      //随机化做快照的间隔
                        setRandRoll(r.nextInt(snapCount/2));
                        // 关闭正在写的日志文件
                        zks.getZKDatabase().rollLog();
                      	//之前的快照任务尚未完成
                        if (snapInProcess != null && snapInProcess.isAlive()) {
                            LOG.warn("Too busy to snap, skipping");
                        } else {
                            //异步做快照
                            snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                    public void run() {
                                        try {
                                            zks.takeSnapshot();
                                        } catch(Exception e) {
                                            LOG.warn("Unexpected exception", e);
                                        }
                                    }
                                };
                            snapInProcess.start();
                        }
                        logCount = 0;
                    }
                } else if (toFlush.isEmpty()) { //读操作，非事务操作
                    //优化读请求（当没有待刷新的写请求时，直接将请求传递给后续的processor）
                    if (nextProcessor != null) {
                     //同步调用nextProcessor（如果nextProcessor处理耗时，导致syncRequestProcess阻塞）
                        nextProcessor.processRequest(si); 
                        if (nextProcessor instanceof Flushable) {
                            ((Flushable)nextProcessor).flush();
                        }
                    }
                    continue;
                }
                //累积1000次（既包含写事务也包含读事务），做一次刷盘操作
                toFlush.add(si);
                if (toFlush.size() > 1000) {
                    flush(toFlush);
                }
            }
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
        running = false;
    }
    LOG.info("SyncRequestProcessor exited!");
}
```



```java
private void flush(LinkedList<Request> toFlush)
    throws IOException, RequestProcessorException
{
    if (toFlush.isEmpty())
        return;
    //执行日志文件刷盘
    zks.getZKDatabase().commit();
    while (!toFlush.isEmpty()) {
        Request i = toFlush.remove();
        if (nextProcessor != null) {
            //同步调用nextProcessor
            nextProcessor.processRequest(i);
        }
    }
    if (nextProcessor != null && nextProcessor instanceof Flushable) {
        ((Flushable)nextProcessor).flush();
    }
}
```

#### FinalRequestProcessor

```java
public void processRequest(Request request) {
    if (LOG.isDebugEnabled()) {
        LOG.debug("Processing request:: " + request);
    }
    // request.addRQRec(">final");
    long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
    if (request.type == OpCode.ping) {
        traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
    }
    if (LOG.isTraceEnabled()) {
        ZooTrace.logRequest(LOG, traceMask, 'E', request, "");
    }
    ProcessTxnResult rc = null;
    synchronized (zks.outstandingChanges) {
        while (!zks.outstandingChanges.isEmpty()
                && zks.outstandingChanges.get(0).zxid <= request.zxid) {
            ChangeRecord cr = zks.outstandingChanges.remove(0);
            if (cr.zxid < request.zxid) {
                LOG.warn("Zxid outstanding "
                        + cr.zxid
                        + " is less than current " + request.zxid);
            }
            if (zks.outstandingChangesForPath.get(cr.path) == cr) {
                zks.outstandingChangesForPath.remove(cr.path);
            }
        }
        if (request.hdr != null) { //处理事务类请求
           TxnHeader hdr = request.hdr;
           Record txn = request.txn;
					 //写入内存树、执行watcher（如果watcher太多，会导致前一个Processor的处理请求的效率降低）
           rc = zks.processTxn(hdr, txn);
        }
        if (Request.isQuorum(request.type)) { //事务类请求
           //将新提交的事务请求加入committedLogList，在内存中存放，方便follower快速同步
            zks.getZKDatabase().addCommittedProposal(request); 
        }
    }

    if (request.hdr != null && request.hdr.getType() == OpCode.closeSession) { //关闭会话
        ServerCnxnFactory scxn = zks.getServerCnxnFactory();
        // this might be possible since
        // we might just be playing diffs from the leader
        if (scxn != null && request.cnxn == null) {
            // calling this if we have the cnxn results in the client's
            // close session response being lost - we've already closed
            // the session/socket here before we can send the closeSession
            // in the switch block below
            scxn.closeSession(request.sessionId);
            return;
        }
    }

    if (request.cnxn == null) {
        return;
    }
    ServerCnxn cnxn = request.cnxn;

    String lastOp = "NA";
    zks.decInProcess();
    Code err = Code.OK;
    Record rsp = null;
    boolean closeSession = false;
    try {
        if (request.hdr != null && request.hdr.getType() == OpCode.error) {
            throw KeeperException.create(KeeperException.Code.get((
                    (ErrorTxn) request.txn).getErr()));
        }

        KeeperException ke = request.getException();
        if (ke != null && request.type != OpCode.multi) {
            throw ke;
        }

        if (LOG.isDebugEnabled()) {
            LOG.debug("{}",request);
        }
        switch (request.type) {
        case OpCode.ping: { //处理ping请求
            zks.serverStats().updateLatency(request.createTime);

            lastOp = "PING";
            cnxn.updateStatsForResponse(request.cxid, request.zxid, lastOp,
                    request.createTime, Time.currentElapsedTime());

            cnxn.sendResponse(new ReplyHeader(-2,
                    zks.getZKDatabase().getDataTreeLastProcessedZxid(), 0), null, "response");
            return;
        }
        case OpCode.createSession: {
            zks.serverStats().updateLatency(request.createTime);

            lastOp = "SESS";
            cnxn.updateStatsForResponse(request.cxid, request.zxid, lastOp,
                    request.createTime, Time.currentElapsedTime());

            zks.finishSessionInit(request.cnxn, true);
            return;
        }
        case OpCode.multi: {
            lastOp = "MULT";
            rsp = new MultiResponse() ;

            for (ProcessTxnResult subTxnResult : rc.multiResult) {

                OpResult subResult ;

                switch (subTxnResult.type) {
                    case OpCode.check:
                        subResult = new CheckResult();
                        break;
                    case OpCode.create:
                        subResult = new CreateResult(subTxnResult.path);
                        break;
                    case OpCode.delete:
                        subResult = new DeleteResult();
                        break;
                    case OpCode.setData:
                        subResult = new SetDataResult(subTxnResult.stat);
                        break;
                    case OpCode.error:
                        subResult = new ErrorResult(subTxnResult.err) ;
                        break;
                    default:
                        throw new IOException("Invalid type of op");
                }

                ((MultiResponse)rsp).add(subResult);
            }

            break;
        }
        case OpCode.create: {
            lastOp = "CREA";
            rsp = new CreateResponse(rc.path);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.delete: {
            lastOp = "DELE";
            err = Code.get(rc.err);
            break;
        }
        case OpCode.setData: {
            lastOp = "SETD";
            rsp = new SetDataResponse(rc.stat);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.setACL: {
            lastOp = "SETA";
            rsp = new SetACLResponse(rc.stat);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.closeSession: {
            lastOp = "CLOS";
            closeSession = true;
            err = Code.get(rc.err);
            break;
        }
        case OpCode.sync: {
            lastOp = "SYNC";
            SyncRequest syncRequest = new SyncRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    syncRequest);
            rsp = new SyncResponse(syncRequest.getPath());
            break;
        }
        case OpCode.check: {
            lastOp = "CHEC";
            rsp = new SetDataResponse(rc.stat);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.exists: {
            lastOp = "EXIS";
            // TODO we need to figure out the security requirement for this!
            ExistsRequest existsRequest = new ExistsRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    existsRequest);
            String path = existsRequest.getPath();
            if (path.indexOf('\0') != -1) {
                throw new KeeperException.BadArgumentsException();
            }
            Stat stat = zks.getZKDatabase().statNode(path, existsRequest
                    .getWatch() ? cnxn : null);
            rsp = new ExistsResponse(stat);
            break;
        }
        case OpCode.getData: {
            lastOp = "GETD";
            GetDataRequest getDataRequest = new GetDataRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getDataRequest);
            DataNode n = zks.getZKDatabase().getNode(getDataRequest.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ,
                    request.authInfo);
            Stat stat = new Stat();
            byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat,
                    getDataRequest.getWatch() ? cnxn : null);
            rsp = new GetDataResponse(b, stat);
            break;
        }
        case OpCode.setWatches: {
            lastOp = "SETW";
            SetWatches setWatches = new SetWatches();
            // XXX We really should NOT need this!!!!
            request.request.rewind();
            ByteBufferInputStream.byteBuffer2Record(request.request, setWatches);
            long relativeZxid = setWatches.getRelativeZxid();
            zks.getZKDatabase().setWatches(relativeZxid, 
                    setWatches.getDataWatches(), 
                    setWatches.getExistWatches(),
                    setWatches.getChildWatches(), cnxn);
            break;
        }
        case OpCode.getACL: {
            lastOp = "GETA";
            GetACLRequest getACLRequest = new GetACLRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getACLRequest);
            DataNode n = zks.getZKDatabase().getNode(getACLRequest.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ | ZooDefs.Perms.ADMIN,
                    request.authInfo);

            Stat stat = new Stat();
            List<ACL> acl =
                    zks.getZKDatabase().getACL(getACLRequest.getPath(), stat);
            try {
                PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                        ZooDefs.Perms.ADMIN,
                        request.authInfo);
                rsp = new GetACLResponse(acl, stat);
            } catch (KeeperException.NoAuthException e) {
                List<ACL> acl1 = new ArrayList<ACL>(acl.size());
                for (ACL a : acl) {
                    if ("digest".equals(a.getId().getScheme())) {
                        Id id = a.getId();
                        Id id1 = new Id(id.getScheme(), id.getId().replaceAll(":.*", ":x"));
                        acl1.add(new ACL(a.getPerms(), id1));
                    } else {
                        acl1.add(a);
                    }
                }
                rsp = new GetACLResponse(acl1, stat);
            }
            break;
        }
        case OpCode.getChildren: {
            lastOp = "GETC";
            GetChildrenRequest getChildrenRequest = new GetChildrenRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getChildrenRequest);
            DataNode n = zks.getZKDatabase().getNode(getChildrenRequest.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ,
                    request.authInfo);
            List<String> children = zks.getZKDatabase().getChildren(
                    getChildrenRequest.getPath(), null, getChildrenRequest
                            .getWatch() ? cnxn : null);
            rsp = new GetChildrenResponse(children);
            break;
        }
        case OpCode.getChildren2: {
            lastOp = "GETC";
            GetChildren2Request getChildren2Request = new GetChildren2Request();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getChildren2Request);
            Stat stat = new Stat();
            DataNode n = zks.getZKDatabase().getNode(getChildren2Request.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ,
                    request.authInfo);
            List<String> children = zks.getZKDatabase().getChildren(
                    getChildren2Request.getPath(), stat, getChildren2Request
                            .getWatch() ? cnxn : null);
            rsp = new GetChildren2Response(children, stat);
            break;
        }
        }
    } catch (SessionMovedException e) {
        // session moved is a connection level error, we need to tear
        // down the connection otw ZOOKEEPER-710 might happen
        // ie client on slow follower starts to renew session, fails
        // before this completes, then tries the fast follower (leader)
        // and is successful, however the initial renew is then 
        // successfully fwd/processed by the leader and as a result
        // the client and leader disagree on where the client is most
        // recently attached (and therefore invalid SESSION MOVED generated)
        cnxn.sendCloseSession();
        return;
    } catch (KeeperException e) {
        err = e.code();
    } catch (Exception e) {
        // log at error level as we are returning a marshalling
        // error to the user
        LOG.error("Failed to process " + request, e);
        StringBuilder sb = new StringBuilder();
        ByteBuffer bb = request.request;
        bb.rewind();
        while (bb.hasRemaining()) {
            sb.append(Integer.toHexString(bb.get() & 0xff));
        }
        LOG.error("Dumping request buffer: 0x" + sb.toString());
        err = Code.MARSHALLINGERROR;
    }

    long lastZxid = zks.getZKDatabase().getDataTreeLastProcessedZxid();
    ReplyHeader hdr =
        new ReplyHeader(request.cxid, lastZxid, err.intValue());
		//统计总的延迟时间、最大、最小延迟	时间
    zks.serverStats().updateLatency(request.createTime);
    cnxn.updateStatsForResponse(request.cxid, lastZxid, lastOp,
                request.createTime, Time.currentElapsedTime());

    try {
        cnxn.sendResponse(hdr, rsp, "response"); //返回响应信息
        if (closeSession) {
            cnxn.sendCloseSession();
        }
    } catch (IOException e) {
        LOG.error("FIXMSG",e);
    }
}
```



```java
public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
    ProcessTxnResult rc;
    int opCode = hdr.getType(); //请求类型
    long sessionId = hdr.getClientId();
    rc = getZKDatabase().processTxn(hdr, txn);
    if (opCode == OpCode.createSession) {//创建session
        if (txn instanceof CreateSessionTxn) {
            CreateSessionTxn cst = (CreateSessionTxn) txn;
            sessionTracker.addSession(sessionId, cst
                    .getTimeOut());
        } else {
            LOG.warn("*****>>>>> Got "
                    + txn.getClass() + " "
                    + txn.toString());
        }
    } else if (opCode == OpCode.closeSession) { //关闭session
        sessionTracker.removeSession(sessionId);
    }
    return rc;
}
```

org.apache.zookeeper.server.ZKDatabase#processTxn

```java
public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {//应用到ZKDatabase
    return dataTree.processTxn(hdr, txn);
}
```

```java
public ProcessTxnResult processTxn(TxnHeader header, Record txn)
{
    ProcessTxnResult rc = new ProcessTxnResult();

    try {
        rc.clientId = header.getClientId();
        rc.cxid = header.getCxid();
        rc.zxid = header.getZxid();
        rc.type = header.getType();
        rc.err = 0;
        rc.multiResult = null;
        switch (header.getType()) {
            case OpCode.create:
                CreateTxn createTxn = (CreateTxn) txn;
                rc.path = createTxn.getPath();
                createNode(
                        createTxn.getPath(),
                        createTxn.getData(),
                        createTxn.getAcl(),
                        createTxn.getEphemeral() ? header.getClientId() : 0,
                        createTxn.getParentCVersion(),
                        header.getZxid(), header.getTime());
                break;
            case OpCode.delete:
                DeleteTxn deleteTxn = (DeleteTxn) txn;
                rc.path = deleteTxn.getPath();
                deleteNode(deleteTxn.getPath(), header.getZxid());
                break;
            case OpCode.setData:
                SetDataTxn setDataTxn = (SetDataTxn) txn;
                rc.path = setDataTxn.getPath();
                rc.stat = setData(setDataTxn.getPath(), setDataTxn
                        .getData(), setDataTxn.getVersion(), header
                        .getZxid(), header.getTime());
                break;
            case OpCode.setACL:
                SetACLTxn setACLTxn = (SetACLTxn) txn;
                rc.path = setACLTxn.getPath();
                rc.stat = setACL(setACLTxn.getPath(), setACLTxn.getAcl(),
                        setACLTxn.getVersion());
                break;
            case OpCode.closeSession:
                killSession(header.getClientId(), header.getZxid());
                break;
            case OpCode.error:
                ErrorTxn errTxn = (ErrorTxn) txn;
                rc.err = errTxn.getErr();
                break;
            case OpCode.check:
                CheckVersionTxn checkTxn = (CheckVersionTxn) txn;
                rc.path = checkTxn.getPath();
                break;
            case OpCode.multi:
                MultiTxn multiTxn = (MultiTxn) txn ;
                List<Txn> txns = multiTxn.getTxns();
                rc.multiResult = new ArrayList<ProcessTxnResult>();
                boolean failed = false;
                for (Txn subtxn : txns) {
                    if (subtxn.getType() == OpCode.error) {
                        failed = true;
                        break;
                    }
                }

                boolean post_failed = false;
                for (Txn subtxn : txns) {
                    ByteBuffer bb = ByteBuffer.wrap(subtxn.getData());
                    Record record = null;
                    switch (subtxn.getType()) {
                        case OpCode.create:
                            record = new CreateTxn();
                            break;
                        case OpCode.delete:
                            record = new DeleteTxn();
                            break;
                        case OpCode.setData:
                            record = new SetDataTxn();
                            break;
                        case OpCode.error:
                            record = new ErrorTxn();
                            post_failed = true;
                            break;
                        case OpCode.check:
                            record = new CheckVersionTxn();
                            break;
                        default:
                            throw new IOException("Invalid type of op: " + subtxn.getType());
                    }
                    assert(record != null);

                    ByteBufferInputStream.byteBuffer2Record(bb, record);
                   
                    if (failed && subtxn.getType() != OpCode.error){
                        int ec = post_failed ? Code.RUNTIMEINCONSISTENCY.intValue() 
                                             : Code.OK.intValue();

                        subtxn.setType(OpCode.error);
                        record = new ErrorTxn(ec);
                    }

                    if (failed) {
                        assert(subtxn.getType() == OpCode.error) ;
                    }

                    TxnHeader subHdr = new TxnHeader(header.getClientId(), header.getCxid(),
                                                     header.getZxid(), header.getTime(), 
                                                     subtxn.getType());
                    ProcessTxnResult subRc = processTxn(subHdr, record);
                    rc.multiResult.add(subRc);
                    if (subRc.err != 0 && rc.err == 0) {
                        rc.err = subRc.err ;
                    }
                }
                break;
        }
    } catch (KeeperException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Failed: " + header + ":" + txn, e);
        }
        rc.err = e.code().intValue();
    } catch (IOException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Failed: " + header + ":" + txn, e);
        }
    }
    /*
     * A snapshot might be in progress while we are modifying the data
     * tree. If we set lastProcessedZxid prior to making corresponding
     * change to the tree, then the zxid associated with the snapshot
     * file will be ahead of its contents. Thus, while restoring from
     * the snapshot, the restore method will not apply the transaction
     * for zxid associated with the snapshot file, since the restore
     * method assumes that transaction to be present in the snapshot.
     *
     * To avoid this, we first apply the transaction and then modify
     * lastProcessedZxid.  During restore, we correctly handle the
     * case where the snapshot contains data ahead of the zxid associated
     * with the file.
     */
    if (rc.zxid > lastProcessedZxid) {
       lastProcessedZxid = rc.zxid;
    }

    /*
     * Snapshots are taken lazily. It can happen that the child
     * znodes of a parent are created after the parent
     * is serialized. Therefore, while replaying logs during restore, a
     * create might fail because the node was already
     * created.
     *
     * After seeing this failure, we should increment
     * the cversion of the parent znode since the parent was serialized
     * before its children.
     *
     * Note, such failures on DT should be seen only during
     * restore.
     */
    if (header.getType() == OpCode.create &&
            rc.err == Code.NODEEXISTS.intValue()) {
        LOG.debug("Adjusting parent cversion for Txn: " + header.getType() +
                " path:" + rc.path + " err: " + rc.err);
        int lastSlash = rc.path.lastIndexOf('/');
        String parentName = rc.path.substring(0, lastSlash);
        CreateTxn cTxn = (CreateTxn)txn;
        try {
            setCversionPzxid(parentName, cTxn.getParentCVersion(),
                    header.getZxid());
        } catch (KeeperException.NoNodeException e) {
            LOG.error("Failed to set parent cversion for: " +
                  parentName, e);
            rc.err = e.code().intValue();
        }
    } else if (rc.err != Code.OK.intValue()) {
        LOG.debug("Ignoring processTxn failure hdr: " + header.getType() +
              " : error: " + rc.err);
    }
    return rc;
}
```

### 会话管理

#### 创建会话

org.apache.zookeeper.server.ZooKeeperServer#createSession

```java
long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {
   //处理连接事件时，创建session
    long sessionId = sessionTracker.createSession(timeout);
    Random r = new Random(sessionId ^ superSecret);
    r.nextBytes(passwd);
    ByteBuffer to = ByteBuffer.allocate(4);
    to.putInt(timeout);
    cnxn.setSessionId(sessionId);
    submitRequest(cnxn, sessionId, OpCode.createSession, 0, to, null);
    return sessionId;
}
```

```java
synchronized public long createSession(int sessionTimeout) {
    addSession(nextSessionId, sessionTimeout);
    return nextSessionId++;
}
```

```java
synchronized public void addSession(long id, int sessionTimeout) {
    sessionsWithTimeout.put(id, sessionTimeout);
    if (sessionsById.get(id) == null) {
        SessionImpl s = new SessionImpl(id, sessionTimeout, 0);
        sessionsById.put(id, s);
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                    "SessionTrackerImpl --- Adding session 0x"
                    + Long.toHexString(id) + " " + sessionTimeout);
        }
    } else {
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                    "SessionTrackerImpl --- Existing session 0x"
                    + Long.toHexString(id) + " " + sessionTimeout);
        }
    }
    touchSession(id, sessionTimeout);
}
```

```java
synchronized public boolean touchSession(long sessionId, int timeout) {
    if (LOG.isTraceEnabled()) {
        ZooTrace.logTraceMessage(LOG,
                                 ZooTrace.CLIENT_PING_TRACE_MASK,
                                 "SessionTrackerImpl --- Touch session: 0x"
                + Long.toHexString(sessionId) + " with timeout " + timeout);
    }
    SessionImpl s = sessionsById.get(sessionId);
    if (s == null || s.isClosing()) {
        return false;
    }
   //规范化过期时间
    long expireTime = roundToInterval(Time.currentElapsedTime() + timeout);
    if (s.tickTime >= expireTime) {
        // Nothing needs to be done
        return true;
    }
    SessionSet set = sessionSets.get(s.tickTime);
    if (set != null) {
        set.sessions.remove(s);
    }
    s.tickTime = expireTime;
    set = sessionSets.get(s.tickTime);
   //将session放入会话桶中，默认3秒
    if (set == null) {
        set = new SessionSet();
        sessionSets.put(expireTime, set);
    }
    set.sessions.add(s);
    return true;
}
```

#### 检查会话

org.apache.zookeeper.server.SessionTrackerImpl#checkSession

```java
synchronized public void checkSession(long sessionId, Object owner) throws KeeperException.SessionExpiredException, KeeperException.SessionMovedException {
    SessionImpl session = sessionsById.get(sessionId);
  	//会话过期
    if (session == null || session.isClosing()) { 
        throw new KeeperException.SessionExpiredException();
    }
    
    if (session.owner == null) {
        session.owner = owner;//设置会话的拥有者
    } else if (session.owner != owner) { //拥有者发生变更表明会话发生了转移
        throw new KeeperException.SessionMovedException();
    }
}
```

#### 续约

org.apache.zookeeper.server.ZooKeeperServer#touch

```java
void touch(ServerCnxn cnxn) throws MissingSessionException {
    if (cnxn == null) {
        return;
    }
    long id = cnxn.getSessionId();
    int to = cnxn.getSessionTimeout();
    if (!sessionTracker.touchSession(id, to)) {
        throw new MissingSessionException(
                "No session with sessionid 0x" + Long.toHexString(id)
                + " exists, probably expired and removed");
    }
}
```



### 创建节点	

#### 写入文件

org.apache.zookeeper.server.ZKDatabase#append

```java
public boolean append(Request si) throws IOException { //写入事务日志，SyncRequestProcessor中调用此方法
    return this.snapLog.append(si);
}
```

org.apache.zookeeper.server.persistence.FileTxnSnapLog#append

```java
public boolean append(Request si) throws IOException {
    return txnLog.append(si.hdr, si.txn);
}
```

org.apache.zookeeper.server.persistence.FileTxnLog#append

```java
public synchronized boolean append(TxnHeader hdr, Record txn)
    throws IOException
{
    //表明是非事务
    if (hdr == null) { 
        return false;
    }
		//校验事务标识zxID
    if (hdr.getZxid() <= lastZxidSeen) {
        LOG.warn("Current zxid " + hdr.getZxid()
                + " is <= " + lastZxidSeen + " for "
                + hdr.getType());
    } else {
        lastZxidSeen = hdr.getZxid();
    }

    if (logStream==null) {
      //新建事务日志
       if(LOG.isInfoEnabled()){
            LOG.info("Creating new log file: " + Util.makeLogName(hdr.getZxid()));
       }

       logFileWrite = new File(logDir, Util.makeLogName(hdr.getZxid()));
       fos = new FileOutputStream(logFileWrite);
       logStream=new BufferedOutputStream(fos);
       oa = BinaryOutputArchive.getArchive(logStream);
       FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
       fhdr.serialize(oa, "fileheader");
       // Make sure that the magic number is written before padding.
       logStream.flush();
       filePadding.setCurrentSize(fos.getChannel().position());
       streamsToFlush.add(fos);
    }
    filePadding.padFile(fos.getChannel());
    byte[] buf = Util.marshallTxnEntry(hdr, txn);
    if (buf == null || buf.length == 0) {
        throw new IOException("Faulty serialization for header " +
                "and txn");
    }
    Checksum crc = makeChecksumAlgorithm();
    crc.update(buf, 0, buf.length);
    oa.writeLong(crc.getValue(), "txnEntryCRC");
    Util.writeTxnBytes(oa, buf);

    return true;
}
```

#### 存放到内存树

org.apache.zookeeper.server.ZooKeeperServer#processTxn

```java
public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
    ProcessTxnResult rc;
    int opCode = hdr.getType();
    long sessionId = hdr.getClientId();
  	//处理事务除了会话
    rc = getZKDatabase().processTxn(hdr, txn);
  	//管理会话
    if (opCode == OpCode.createSession) {
        if (txn instanceof CreateSessionTxn) { //创建session
            CreateSessionTxn cst = (CreateSessionTxn) txn;
            sessionTracker.addSession(sessionId, cst
                    .getTimeOut());
        } else {
            LOG.warn("*****>>>>> Got "
                    + txn.getClass() + " "
                    + txn.toString());
        }
    } else if (opCode == OpCode.closeSession) {
        sessionTracker.removeSession(sessionId); //移除session
    }
    return rc;
}
```

org.apache.zookeeper.server.ZKDatabase#processTxn

```java
public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
    return dataTree.processTxn(hdr, txn);
}
```

org.apache.zookeeper.server.DataTree#processTxn

```java
public ProcessTxnResult processTxn(TxnHeader header, Record txn)
{
    ProcessTxnResult rc = new ProcessTxnResult();

    try {
        rc.clientId = header.getClientId();
        rc.cxid = header.getCxid();
        rc.zxid = header.getZxid();
        rc.type = header.getType();
        rc.err = 0;
        rc.multiResult = null;
        switch (header.getType()) {
            case OpCode.create:
                CreateTxn createTxn = (CreateTxn) txn;
                rc.path = createTxn.getPath();
                createNode(
                        createTxn.getPath(),
                        createTxn.getData(),
                        createTxn.getAcl(),
                        createTxn.getEphemeral() ? header.getClientId() : 0,
                        createTxn.getParentCVersion(),
                        header.getZxid(), header.getTime());
                break;
            case OpCode.delete:
                DeleteTxn deleteTxn = (DeleteTxn) txn;
                rc.path = deleteTxn.getPath();
                deleteNode(deleteTxn.getPath(), header.getZxid());
                break;
            case OpCode.setData:
                SetDataTxn setDataTxn = (SetDataTxn) txn;
                rc.path = setDataTxn.getPath();
                rc.stat = setData(setDataTxn.getPath(), setDataTxn
                        .getData(), setDataTxn.getVersion(), header
                        .getZxid(), header.getTime());
                break;
            case OpCode.setACL:
                SetACLTxn setACLTxn = (SetACLTxn) txn;
                rc.path = setACLTxn.getPath();
                rc.stat = setACL(setACLTxn.getPath(), setACLTxn.getAcl(),
                        setACLTxn.getVersion());
                break;
            case OpCode.closeSession:
                killSession(header.getClientId(), header.getZxid());
                break;
            case OpCode.error:
                ErrorTxn errTxn = (ErrorTxn) txn;
                rc.err = errTxn.getErr();
                break;
            case OpCode.check:
                CheckVersionTxn checkTxn = (CheckVersionTxn) txn;
                rc.path = checkTxn.getPath();
                break;
            case OpCode.multi:
                MultiTxn multiTxn = (MultiTxn) txn ;
                List<Txn> txns = multiTxn.getTxns();
                rc.multiResult = new ArrayList<ProcessTxnResult>();
                boolean failed = false;
                for (Txn subtxn : txns) {
                    if (subtxn.getType() == OpCode.error) {
                        failed = true;
                        break;
                    }
                }

                boolean post_failed = false;
                for (Txn subtxn : txns) {
                    ByteBuffer bb = ByteBuffer.wrap(subtxn.getData());
                    Record record = null;
                    switch (subtxn.getType()) {
                        case OpCode.create:
                            record = new CreateTxn();
                            break;
                        case OpCode.delete:
                            record = new DeleteTxn();
                            break;
                        case OpCode.setData:
                            record = new SetDataTxn();
                            break;
                        case OpCode.error:
                            record = new ErrorTxn();
                            post_failed = true;
                            break;
                        case OpCode.check:
                            record = new CheckVersionTxn();
                            break;
                        default:
                            throw new IOException("Invalid type of op: " + subtxn.getType());
                    }
                    assert(record != null);

                    ByteBufferInputStream.byteBuffer2Record(bb, record);
                   
                    if (failed && subtxn.getType() != OpCode.error){
                        int ec = post_failed ? Code.RUNTIMEINCONSISTENCY.intValue() 
                                             : Code.OK.intValue();

                        subtxn.setType(OpCode.error);
                        record = new ErrorTxn(ec);
                    }

                    if (failed) {
                        assert(subtxn.getType() == OpCode.error) ;
                    }

                    TxnHeader subHdr = new TxnHeader(header.getClientId(), header.getCxid(),
                                                     header.getZxid(), header.getTime(), 
                                                     subtxn.getType());
                    ProcessTxnResult subRc = processTxn(subHdr, record);
                    rc.multiResult.add(subRc);
                    if (subRc.err != 0 && rc.err == 0) {
                        rc.err = subRc.err ;
                    }
                }
                break;
        }
    } catch (KeeperException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Failed: " + header + ":" + txn, e);
        }
        rc.err = e.code().intValue();
    } catch (IOException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Failed: " + header + ":" + txn, e);
        }
    }
    if (rc.zxid > lastProcessedZxid) {
       lastProcessedZxid = rc.zxid;
    }

    if (header.getType() == OpCode.create &&
            rc.err == Code.NODEEXISTS.intValue()) {
        LOG.debug("Adjusting parent cversion for Txn: " + header.getType() +
                " path:" + rc.path + " err: " + rc.err);
        int lastSlash = rc.path.lastIndexOf('/');
        String parentName = rc.path.substring(0, lastSlash);
        CreateTxn cTxn = (CreateTxn)txn;
        try {
            setCversionPzxid(parentName, cTxn.getParentCVersion(),
                    header.getZxid());
        } catch (KeeperException.NoNodeException e) {
            LOG.error("Failed to set parent cversion for: " +
                  parentName, e);
            rc.err = e.code().intValue();
        }
    } else if (rc.err != Code.OK.intValue()) {
        LOG.debug("Ignoring processTxn failure hdr: " + header.getType() +
              " : error: " + rc.err);
    }
    return rc;
}
```

org.apache.zookeeper.server.DataTree#createNode

```java
public String createNode(String path, byte data[], List<ACL> acl,
        long ephemeralOwner, int parentCVersion, long zxid, long time)
        throws KeeperException.NoNodeException,
        KeeperException.NodeExistsException { //创建DataNode放到内存中
    int lastSlash = path.lastIndexOf('/');
    String parentName = path.substring(0, lastSlash); //父节点名称
    String childName = path.substring(lastSlash + 1); //子节点名称
    StatPersisted stat = new StatPersisted();
    stat.setCtime(time);
    stat.setMtime(time);
    stat.setCzxid(zxid);
    stat.setMzxid(zxid);
    stat.setPzxid(zxid);
    stat.setVersion(0);
    stat.setAversion(0);
    stat.setEphemeralOwner(ephemeralOwner); //永久节点为0，临时节点：客户端标志
    DataNode parent = nodes.get(parentName); //获取父节点
    if (parent == null) {
        throw new KeeperException.NoNodeException();
    }
    synchronized (parent) {
        Set<String> children = parent.getChildren(); //获取父节点的子节点列表
      	//节点已经存在
        if (children.contains(childName)) {
            throw new KeeperException.NodeExistsException();
        }
        
        if (parentCVersion == -1) {
            parentCVersion = parent.stat.getCversion();
            parentCVersion++;
        }    
        parent.stat.setCversion(parentCVersion);
        parent.stat.setPzxid(zxid);
        Long longval = aclCache.convertAcls(acl);
      //创建节点
        DataNode child = new DataNode(parent, data, longval, stat);
        parent.addChild(childName);
      //放入内存树ConcurrentHashMap中
        nodes.put(path, child);
        if (ephemeralOwner != 0) { //单独的ConcurrentHashMap存放临时节点
            HashSet<String> list = ephemerals.get(ephemeralOwner);
            if (list == null) {
                list = new HashSet<String>();
                ephemerals.put(ephemeralOwner, list);
            }
            synchronized (list) {
                list.add(path);
            }
        }
    }
    // now check if its one of the zookeeper node child
    if (parentName.startsWith(quotaZookeeper)) {
        // now check if its the limit node
        if (Quotas.limitNode.equals(childName)) {
            // this is the limit node
            // get the parent and add it to the trie
            pTrie.addPath(parentName.substring(quotaZookeeper.length()));
        }
        if (Quotas.statNode.equals(childName)) {
            updateQuotaForPath(parentName
                    .substring(quotaZookeeper.length()));
        }
    }
    // also check to update the quotas for this node
    String lastPrefix;
    if((lastPrefix = getMaxPrefixWithQuota(path)) != null) {
        // ok we have some match and need to update
        updateCount(lastPrefix, 1);
        updateBytes(lastPrefix, data == null ? 0 : data.length);
    }
    //给客户端发送watch通知
    dataWatches.triggerWatch(path, Event.EventType.NodeCreated);
    childWatches.triggerWatch(parentName.equals("") ? "/" : parentName,
            Event.EventType.NodeChildrenChanged);
    return path;
}
```

### WatchManager

WatchManager负责对客户端的Watcher进行注册、通知

Watcher只是保存在内存中，所以存在watcher丢失的情况

####  注册watcher

org.apache.zookeeper.server.DataTree#getData 

```java
public byte[] getData(String path, Stat stat, Watcher watcher)
        throws KeeperException.NoNodeException {
    DataNode n = nodes.get(path); //获取数据
    if (n == null) {
        throw new KeeperException.NoNodeException();
    }
    synchronized (n) {
        n.copyStat(stat);
        if (watcher != null) {
            dataWatches.addWatch(path, watcher);
        }
        return n.data;
    }
}
```



org.apache.zookeeper.server.WatchManager#addWatch

```java
public synchronized void addWatch(String path, Watcher watcher) {
    HashSet<Watcher> list = watchTable.get(path);
    if (list == null) {
        // don't waste memory if there are few watches on a node
        // rehash when the 4th entry is added, doubling size thereafter
        // seems like a good compromise
        list = new HashSet<Watcher>(4);
        watchTable.put(path, list);
    }
    list.add(watcher);

    HashSet<String> paths = watch2Paths.get(watcher);
    if (paths == null) {
        // cnxns typically have many watches, so use default cap here
        paths = new HashSet<String>();
        watch2Paths.put(watcher, paths);
    }
    paths.add(path);
}
```

#### 触发watch

org.apache.zookeeper.server.WatchManager#triggerWatch

```java
public Set<Watcher> triggerWatch(String path, EventType type) {
    return triggerWatch(path, type, null);
}
```



```java
public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type,
            KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
      //watchTable：path -> Set<Watcher>
        watchers = watchTable.remove(path); //根据path获取注册的watcher，从map中移除
        if (watchers == null || watchers.isEmpty()) {
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG,
                        ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                        "No watchers for " + path);
            }
            return null;
        }
      //watch2Paths : watcher -> Hashset<Path>
        for (Watcher w : watchers) {
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path); //移除path
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        w.process(e); //通知客户端发生事件
    }
    return watchers;
}
```

# 集群模式

## 启动入口

org.apache.zookeeper.server.quorum.QuorumPeerMain#main

```java
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        main.initializeAndRun(args);
    } catch (IllegalArgumentException e) {
        LOG.error("Invalid arguments, exiting abnormally", e);
        LOG.info(USAGE);
        System.err.println(USAGE);
        System.exit(2);
    } catch (ConfigException e) {
        LOG.error("Invalid config, exiting abnormally", e);
        System.err.println("Invalid config, exiting abnormally");
        System.exit(2);
    } catch (Exception e) {
        LOG.error("Unexpected exception, exiting abnormally", e);
        System.exit(1);
    }
    LOG.info("Exiting normally");
    System.exit(0);
}
```

```java
protected void initializeAndRun(String[] args)
    throws ConfigException, IOException
{
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }

    //清理磁盘（日志文件和快照文件）
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
            .getDataDir(), config.getDataLogDir(), config
            .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.servers.size() > 0) {
        runFromConfig(config); //集群模式
    } else {
        LOG.warn("Either no config or no quorum defined in config, running "
                + " in standalone mode");
        // standalone模式
        ZooKeeperServerMain.main(args);
    }
}
```

### 创建QuorumPeer

```java
public void runFromConfig(QuorumPeerConfig config) throws IOException {
  try {
      ManagedUtil.registerLog4jMBeans();
  } catch (JMException e) {
      LOG.warn("Unable to register log4j JMX control", e);
  }
  LOG.info("Starting quorum peer");
  try {
      //默认NettyServerCnxnFactory
      ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
      cnxnFactory.configure(config.getClientPortAddress(),
                            config.getMaxClientCnxns());
      quorumPeer = getQuorumPeer();
      //集群节点信息
      quorumPeer.setQuorumPeers(config.getServers());
      //管理事务日志、快照
      quorumPeer.setTxnFactory(new FileTxnSnapLog(
              new File(config.getDataLogDir()),
              new File(config.getDataDir())));
      //选举算法
      quorumPeer.setElectionType(config.getElectionAlg());
      //myId，服务实例标志
      quorumPeer.setMyid(config.getServerId());
      quorumPeer.setTickTime(config.getTickTime());
      quorumPeer.setInitLimit(config.getInitLimit());
      quorumPeer.setSyncLimit(config.getSyncLimit());
      quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
      quorumPeer.setCnxnFactory(cnxnFactory);
      quorumPeer.setQuorumVerifier(config.getQuorumVerifier());
      quorumPeer.setClientPortAddress(config.getClientPortAddress());
     //会话最小、最大超时时间
      quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
      quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
      quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
      //PARTICIPANT, OBSERVER
      quorumPeer.setLearnerType(config.getPeerType());
      //默认true
      quorumPeer.setSyncEnabled(config.getSyncEnabled());
      // sets quorum sasl authentication configurations
      quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
      if(quorumPeer.isQuorumSaslAuthEnabled()){
          quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
          quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
          quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
          quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
          quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
      }
      quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
      quorumPeer.initialize();
      //启动
      quorumPeer.start();
      quorumPeer.join();
  } catch (InterruptedException e) {
      // warn, but generally this is ok
      LOG.warn("Quorum Peer interrupted", e);
  }
}
```

### 启动QuorumPeer

```java
public synchronized void start() {
    //加载快照、事务日志文件
    loadDataBase();
    //绑定端口
    cnxnFactory.start();
    //开始Leader选举
    startLeaderElection();
    super.start();
}
```

```java
synchronized public void startLeaderElection() {
   try {
      currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
   } catch(IOException e) {
      RuntimeException re = new RuntimeException(e.getMessage());
      re.setStackTrace(e.getStackTrace());
      throw re;
   }
    for (QuorumServer p : getView().values()) {
        if (p.id == myid) {
            myQuorumAddr = p.addr;
            break;
        }
    }
    if (myQuorumAddr == null) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
  	//默认为3
    if (electionType == 0) {
        try {
            udpSocket = new DatagramSocket(myQuorumAddr.getPort());
            responder = new ResponderThread();
            responder.start();
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
    }
    this.electionAlg = createElectionAlgorithm(electionType);
}
```

#### 创建选举算法

org.apache.zookeeper.server.quorum.QuorumPeer#createElectionAlgorithm

```java
protected Election createElectionAlgorithm(int electionAlgorithm){
    Election le=null;
    //TODO: use a factory rather than a switch
    switch (electionAlgorithm) {
    case 0:
        le = new LeaderElection(this);
        break;
    case 1:
        le = new AuthFastLeaderElection(this);
        break;
    case 2:
        le = new AuthFastLeaderElection(this, true);
        break;
    case 3:
        //监听leader选举的端口
        qcm = createCnxnManager();
        QuorumCnxManager.Listener listener = qcm.listener;
        if(listener != null){
            //监听其他节点发送的投票请求
            listener.start();
            //选举算法
            le = new FastLeaderElection(this, qcm);
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        break;
    default:
        assert false;
    }
    return le;
}
```



```java
public QuorumCnxManager createCnxnManager() {
    return new QuorumCnxManager(this.getId(),
                                this.getView(),
                                this.authServer,
                                this.authLearner,
                                this.tickTime * this.syncLimit,
                                this.getQuorumListenOnAllIPs(),
                                this.quorumCnxnThreadsSize,
                                this.isQuorumSaslAuthEnabled());
}
```

### 初始化选举

org.apache.zookeeper.server.quorum.FastLeaderElection#FastLeaderElection

```java
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
    this.stop = false;
    this.manager = manager;
    starter(self, manager);
}
```



```java
private void starter(QuorumPeer self, QuorumCnxManager manager) {
    this.self = self;
    proposedLeader = -1;
    proposedZxid = -1;

    sendqueue = new LinkedBlockingQueue<ToSend>();
    recvqueue = new LinkedBlockingQueue<Notification>();
    this.messenger = new Messenger(manager);
}
```

org.apache.zookeeper.server.quorum.FastLeaderElection.Messenger#Messenger

```java
Messenger(QuorumCnxManager manager) {
		//负责发送投票信息
    this.ws = new WorkerSender(manager);
    Thread t = new Thread(this.ws,
            "WorkerSender[myid=" + self.getId() + "]");
    t.setDaemon(true);
    t.start();
		
    //负责接收投票的响应信息
    this.wr = new WorkerReceiver(manager);
    t = new Thread(this.wr,
            "WorkerReceiver[myid=" + self.getId() + "]");
    t.setDaemon(true);
    t.start();
}
```

#### 发送投票信息

org.apache.zookeeper.server.quorum.FastLeaderElection.Messenger.WorkerSender#run

```java
public void run() {
    while (!stop) {
        try {
            ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
            if(m == null) continue;

            process(m);
        } catch (InterruptedException e) {
            break;
        }
    }
    LOG.info("WorkerSender is down");
}
```



```java
void process(ToSend m) {
    ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), 
                                            m.leader,
                                            m.zxid, 
                                            m.electionEpoch, 
                                            m.peerEpoch);
    manager.toSend(m.sid, requestBuffer);
}
```



```java
public void toSend(Long sid, ByteBuffer b) {
    if (this.mySid == sid) {//发送给本节点
         b.position(0);
         addToRecvQueue(new Message(b.duplicate(), sid)); //放入recvQueue
    } else {
      	//发送给其他节点
        //将发送的请求放入节点对应的ArrayBlockingQueue
         ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
      //queueSendMap:sid -> ArrayBlockingQueue 
         ArrayBlockingQueue<ByteBuffer> bqExisting = queueSendMap.putIfAbsent(sid, bq);
         if (bqExisting != null) {
             addToSendQueue(bqExisting, b);
         } else {
             addToSendQueue(bq, b);
         }
         connectOne(sid); //建立连接
            
    }
}
```

```java
synchronized public void connectOne(long sid){
    if (!connectedToPeer(sid)){ //尚未建立连接
        InetSocketAddress electionAddr;
        if (view.containsKey(sid)) { //验证sid的有效性
            electionAddr = view.get(sid).electionAddr;
        } else {
            LOG.warn("Invalid server id: " + sid);
            return;
        }
        try {

            LOG.debug("Opening channel to server " + sid);
            Socket sock = new Socket();
            setSockOpts(sock);
            sock.connect(view.get(sid).electionAddr, cnxTO); //根据选举地址进行连接
            LOG.debug("Connected to server " + sid);

            if (quorumSaslAuthEnabled) {
                initiateConnectionAsync(sock, sid);
            } else {
                initiateConnection(sock, sid);
            }
        } catch (UnresolvedAddressException e) {
            // Sun doesn't include the address that causes this
            // exception to be thrown, also UAE cannot be wrapped cleanly
            // so we log the exception in order to capture this critical
            // detail.
            LOG.warn("Cannot open channel to " + sid
                    + " at election address " + electionAddr, e);
            // Resolve hostname for this server in case the
            // underlying ip address has changed.
            if (view.containsKey(sid)) {
                view.get(sid).recreateSocketAddresses();
            }
            throw e;
        } catch (IOException e) {
            LOG.warn("Cannot open channel to " + sid
                    + " at election address " + electionAddr,
                    e);
            // We can't really tell if the server is actually down or it failed
            // to connect to the server because the underlying IP address
            // changed. Resolve the hostname again just in case.
            if (view.containsKey(sid)) {
                view.get(sid).recreateSocketAddresses();
            }
        }
    } else {
        LOG.debug("There is a connection already for server " + sid);
    }
}
```

```java
public void initiateConnection(final Socket sock, final Long sid) {
    try {
        startConnection(sock, sid);
    } catch (IOException e) {
        LOG.error("Exception while connecting, id: {}, addr: {}, closing learner connection",
                 new Object[] { sid, sock.getRemoteSocketAddress() }, e);
        closeSocket(sock);
        return;
    }
}
```

```java
private boolean startConnection(Socket sock, Long sid)
        throws IOException {
    DataOutputStream dout = null;
    DataInputStream din = null;
    try {
        // Sending id and challenge
        dout = new DataOutputStream(sock.getOutputStream());
        dout.writeLong(this.mySid);
        dout.flush();

        din = new DataInputStream(
                new BufferedInputStream(sock.getInputStream()));
    } catch (IOException e) {
        LOG.warn("Ignoring exception reading or writing challenge: ", e);
        closeSocket(sock);
        return false;
    }

    // authenticate learner
    authLearner.authenticate(sock, view.get(sid).hostname);

    // If lost the challenge, then drop the new connection
    if (sid > this.mySid) {
        LOG.info("Have smaller server identifier, so dropping the " +
                 "connection: (" + sid + ", " + this.mySid + ")");
        closeSocket(sock);
        // Otherwise proceed with the connection
    } else {
      //负责发送消息
        SendWorker sw = new SendWorker(sock, sid);
      //负责接收消息
        RecvWorker rw = new RecvWorker(sock, din, sid, sw);
        sw.setRecv(rw);

        SendWorker vsw = senderWorkerMap.get(sid);
        
        if(vsw != null)
            vsw.finish();
        
        senderWorkerMap.put(sid, sw);
        queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
        
        sw.start(); //启动发送消息，从queueSendMap获取消息发送给远端
        rw.start(); //启动接收消息，将接收的响应放入recvQueue
        
        return true;    
        
    }
    return false;
}
```

#### 接收响应信息

```java
public void run() {

    Message response;
    while (!stop) {
        // Sleeps on receive
        try{
            response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
            if(response == null) continue;

            /*
             * If it is from an observer, respond right away.
             * Note that the following predicate assumes that
             * if a server is not a follower, then it must be
             * an observer. If we ever have any other type of
             * learner in the future, we'll have to change the
             * way we check for observers.
             */
            if(!validVoter(response.sid)){ //来自observer的响应
                Vote current = self.getCurrentVote();
                ToSend notmsg = new ToSend(ToSend.mType.notification,
                        current.getId(),
                        current.getZxid(),
                        logicalclock.get(),
                        self.getPeerState(),
                        response.sid,
                        current.getPeerEpoch());

                sendqueue.offer(notmsg);
            } else {
                // 来自Follower的响应
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Receive new notification message. My id = "
                            + self.getId());
                }

                /*
                 * We check for 28 bytes for backward compatibility
                 */
                if (response.buffer.capacity() < 28) {
                    LOG.error("Got a short response: "
                            + response.buffer.capacity());
                    continue;
                }
                boolean backCompatibility = (response.buffer.capacity() == 28);
                response.buffer.clear();

                // Instantiate Notification and set its attributes
                Notification n = new Notification();
                
                // State of peer that sent this message
                QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                switch (response.buffer.getInt()) {
                case 0:
                    ackstate = QuorumPeer.ServerState.LOOKING;
                    break;
                case 1:
                    ackstate = QuorumPeer.ServerState.FOLLOWING;
                    break;
                case 2:
                    ackstate = QuorumPeer.ServerState.LEADING;
                    break;
                case 3:
                    ackstate = QuorumPeer.ServerState.OBSERVING;
                    break;
                default:
                    continue;
                }
                
                n.leader = response.buffer.getLong();
                n.zxid = response.buffer.getLong();
                n.electionEpoch = response.buffer.getLong();
                n.state = ackstate;
                n.sid = response.sid;
                if(!backCompatibility){
                    n.peerEpoch = response.buffer.getLong();
                } else {
                    if(LOG.isInfoEnabled()){
                        LOG.info("Backward compatibility mode, server id=" + n.sid);
                    }
                    n.peerEpoch = ZxidUtils.getEpochFromZxid(n.zxid);
                }


                n.version = (response.buffer.remaining() >= 4) ? 
                             response.buffer.getInt() : 0x0;

                if(LOG.isInfoEnabled()){
                    printNotification(n);
                }

                /*
                 * If this server is looking, then send proposed leader
                 */

                if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){
                    recvqueue.offer(n);

                    /*
                     * Send a notification back if the peer that sent this
                     * message is also looking and its logical clock is
                     * lagging behind.
                     */
                    if((ackstate == QuorumPeer.ServerState.LOOKING)
                            && (n.electionEpoch < logicalclock.get())){
                        Vote v = getVote();
                        ToSend notmsg = new ToSend(ToSend.mType.notification,
                                v.getId(),
                                v.getZxid(),
                                logicalclock.get(),
                                self.getPeerState(),
                                response.sid,
                                v.getPeerEpoch());
                        sendqueue.offer(notmsg);
                    }
                } else {
                    /*
                     * If this server is not looking, but the one that sent the ack
                     * is looking, then send back what it believes to be the leader.
                     */
                    Vote current = self.getCurrentVote();
                    if(ackstate == QuorumPeer.ServerState.LOOKING){
                        if(LOG.isDebugEnabled()){
                            LOG.debug("Sending new notification. My id =  " +
                                    self.getId() + " recipient=" +
                                    response.sid + " zxid=0x" +
                                    Long.toHexString(current.getZxid()) +
                                    " leader=" + current.getId());
                        }
                        
                        ToSend notmsg;
                        if(n.version > 0x0) {
                            notmsg = new ToSend(
                                    ToSend.mType.notification,
                                    current.getId(),
                                    current.getZxid(),
                                    current.getElectionEpoch(),
                                    self.getPeerState(),
                                    response.sid,
                                    current.getPeerEpoch());
                            
                        } else {
                            Vote bcVote = self.getBCVote();
                            notmsg = new ToSend(
                                    ToSend.mType.notification,
                                    bcVote.getId(),
                                    bcVote.getZxid(),
                                    bcVote.getElectionEpoch(),
                                    self.getPeerState(),
                                    response.sid,
                                    bcVote.getPeerEpoch());
                        }
                        sendqueue.offer(notmsg);
                    }
                }
            }
        } catch (InterruptedException e) {
            System.out.println("Interrupted Exception while waiting for new message" +
                    e.toString());
        }
    }
    LOG.info("WorkerReceiver is down");
}
```

### 监控节点的状态

org.apache.zookeeper.server.quorum.QuorumPeer#run

```java
public void run() {
    setName("QuorumPeer" + "[myid=" + getId() + "]" +
            cnxnFactory.getLocalAddress());

    LOG.debug("Starting quorum peer");
    try {
        jmxQuorumBean = new QuorumBean(this);
        MBeanRegistry.getInstance().register(jmxQuorumBean, null);
        for(QuorumServer s: getView().values()){
            ZKMBeanInfo p;
            if (getId() == s.id) {
                p = jmxLocalPeerBean = new LocalPeerBean(this);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                    jmxLocalPeerBean = null;
                }
            } else {
                p = new RemotePeerBean(s);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                }
            }
        }
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        jmxQuorumBean = null;
    }

    try {
        /*
         * Main loop
         */
        while (running) {
            switch (getPeerState()) {
            case LOOKING: //初始状态
                LOG.info("LOOKING");
                if (Boolean.getBoolean("readonlymode.enabled")) {
                    LOG.info("Attempting to start ReadOnlyZooKeeperServer");
                    // Create read-only server but don't start it immediately
                    final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(
                            logFactory, this,
                            new ZooKeeperServer.BasicDataTreeBuilder(),
                            this.zkDb);
                    Thread roZkMgr = new Thread() {
                        public void run() {
                            try {
                                // lower-bound grace period to 2 secs
                                sleep(Math.max(2000, tickTime));
                                if (ServerState.LOOKING.equals(getPeerState())) {
                                    roZk.startup();
                                }
                            } catch (InterruptedException e) {
                                LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                            } catch (Exception e) {
                                LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                            }
                        }
                    };
                    try {
                        roZkMgr.start();
                        setBCVote(null);
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                        setPeerState(ServerState.LOOKING);
                    } finally {
                        // If the thread is in the the grace period, interrupt
                        // to come out of waiting.
                        roZkMgr.interrupt();
                        roZk.shutdown();
                    }
                } else {
                    try {
                        setBCVote(null);
                        //开始选举
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                        setPeerState(ServerState.LOOKING);
                    }
                }
                break;
            case OBSERVING:
                try {
                    LOG.info("OBSERVING");
                    setObserver(makeObserver(logFactory));
                    observer.observeLeader();
                } catch (Exception e) {
                    LOG.warn("Unexpected exception",e );                        
                } finally {
                    observer.shutdown();
                    setObserver(null);
                    setPeerState(ServerState.LOOKING);
                }
                break;
            case FOLLOWING:
                try {
                    LOG.info("FOLLOWING");
                    setFollower(makeFollower(logFactory));
                    follower.followLeader();
                } catch (Exception e) {
                    LOG.warn("Unexpected exception",e);
                } finally {
                    follower.shutdown();
                    setFollower(null);
                    setPeerState(ServerState.LOOKING);
                }
                break;
            case LEADING:
                LOG.info("LEADING");
                try {
                    //创建Leader
                    setLeader(makeLeader(logFactory));
                  	//死循环，发送心跳，过半数节点超时，退出循环，开启新一轮的选举
                    leader.lead();
                    setLeader(null);
                } catch (Exception e) {
                    LOG.warn("Unexpected exception",e);
                } finally {
                    if (leader != null) {
                        leader.shutdown("Forcing shutdown");
                        setLeader(null);
                    }
                    setPeerState(ServerState.LOOKING);
                }
                break;
            }
        }
    } finally {
        LOG.warn("QuorumPeer main thread exited");
        try {
            MBeanRegistry.getInstance().unregisterAll();
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        jmxQuorumBean = null;
        jmxLocalPeerBean = null;
    }
}
```

### 选举领导者

org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader

```java
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(
                self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        self.jmxLeaderElectionBean = null;
    }
    if (self.start_fle == 0) {
       self.start_fle = Time.currentElapsedTime();
    }
    try {
      	//接受的集群中各节点的投票信息
        HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

        HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = finalizeWait;

        synchronized(this){
            //选举前，增加逻辑时钟
            logicalclock.incrementAndGet();
            //更新本本节点的投票信息（机器id、最新的事务Id、选举轮次）
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info("New election. My id =  " + self.getId() +
                ", proposed zxid=0x" + Long.toHexString(proposedZxid));
        //将本节点的有票信息发送给集群中的PARTICIPANT类型的节点
        sendNotifications();

        /*
         * Loop in which we exchange notifications until we find a leader
         */

        while ((self.getPeerState() == ServerState.LOOKING) &&
                (!stop)){
            //recvqueue存放其他节点的投票信息
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time
             */
            Notification n = recvqueue.poll(notTimeout,
                    TimeUnit.MILLISECONDS);
            //没有接收到投票信息
            if(n == null){
                if(manager.haveDelivered()){
                    sendNotifications();//再次发送本节点的投票信息
                } else {
                    manager.connectAll();//连接所有节点
                }
                /*
                 * Exponential backoff
                 */
                int tmpTimeOut = notTimeout*2;
                notTimeout = (tmpTimeOut < maxNotificationInterval?
                        tmpTimeOut : maxNotificationInterval);
                LOG.info("Notification time out: " + notTimeout);
            }
            else if(validVoter(n.sid) && validVoter(n.leader)) { //验证投票信息是否合法，只接受PARTICIPANT节点的投票。是否接受其他节点的投票，取决于逻辑时钟、选举轮次、事务Id、机器Id
                /*
                 * Only proceed if the vote comes from a replica in the
                 * voting view for a replica in the voting view.
                 */
                switch (n.state) {
                case LOOKING:
                    // 其他节点的逻辑时钟大于本节点的逻辑时钟
                    if (n.electionEpoch > logicalclock.get()) {
                        logicalclock.set(n.electionEpoch);//修改本地逻辑时钟
                        recvset.clear(); //清空投票信息
                       
                        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            //接受其他节点的投票信息，更改本机的投票信息
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else {
                            updateProposal(getInitId(),
                                    getInitLastLoggedZxid(),
                                    getPeerEpoch());
                        }
                        //将更改之后的投票信息发送给集群节点
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) { //其他节点的逻辑时钟小于本节点的，不接受其投票信息
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                            proposedLeader, proposedZxid, proposedEpoch)) { //选举任期相同
                        //接受其他节点的投票信息，更改本机的投票信息
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        //将更改之后的投票信息发送给集群节点
                        sendNotifications();
                    }

                    //存放每个节点的投票信息
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    //集群中是否有半数节点将投票信息投给了同一个节点
                    if (termPredicate(recvset,
                            new Vote(proposedLeader, proposedZxid,
                                    logicalclock.get(), proposedEpoch))) {
                        // 选举出leader后等待200毫秒
                        while((n = recvqueue.poll(finalizeWait,
                                TimeUnit.MILLISECONDS)) != null){
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    proposedLeader, proposedZxid, proposedEpoch)){
                              	//如果仍然接收到了其他节点的投票信息，外面的循环还会进行一次
                                recvqueue.put(n);
                                break;
                            }
                        }
                        if (n == null) {
                            //设置节点角色
                            self.setPeerState((proposedLeader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                            //最终的投票信息
                            Vote endVote = new Vote(proposedLeader,
                                                    proposedZxid,
                                                    logicalclock.get(),
                                                    proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    break;
                case FOLLOWING:
                case LEADING:
                    if(n.electionEpoch == logicalclock.get()){
                        recvset.put(n.sid, new Vote(n.leader,
                                                      n.zxid,
                                                      n.electionEpoch,
                                                      n.peerEpoch));
                       
                        if(ooePredicate(recvset, outofelection, n)) {
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());

                            Vote endVote = new Vote(n.leader, 
                                    n.zxid, 
                                    n.electionEpoch, 
                                    n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    outofelection.put(n.sid, new Vote(n.version,
                                                        n.leader,
                                                        n.zxid,
                                                        n.electionEpoch,
                                                        n.peerEpoch,
                                                        n.state));
       
                    if(ooePredicate(outofelection, outofelection, n)) {
                        synchronized(this){
                            logicalclock.set(n.electionEpoch);
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                        }
                        Vote endVote = new Vote(n.leader,
                                                n.zxid,
                                                n.electionEpoch,
                                                n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)",
                            n.state, n.sid);
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if(self.jmxLeaderElectionBean != null){
                MBeanRegistry.getInstance().unregister(
                        self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
        LOG.debug("Number of connection processing threads: {}",
                manager.getConnectionThreadCount());
    }
}
```

org.apache.zookeeper.server.quorum.FastLeaderElection#sendNotifications

```java
private void sendNotifications() {
  	//首先获取参与选举投票的节点
    for (QuorumServer server : self.getVotingView().values()) {
        long sid = server.id;

        ToSend notmsg = new ToSend(ToSend.mType.notification,
                proposedLeader, //机器id
                proposedZxid, //最新的事务ID
                logicalclock.get(), //逻辑时钟，机器启动之后，经历过的选举次数
                QuorumPeer.ServerState.LOOKING, //当前的状态
                sid, //投票节点的机器Id
                proposedEpoch); //选举轮次，
        if(LOG.isDebugEnabled()){
            LOG.debug("Sending Notification: " + proposedLeader + " (n.leader), 0x"  +
                  Long.toHexString(proposedZxid) + " (n.zxid), 0x" + Long.toHexString(logicalclock.get())  +
                  " (n.round), " + sid + " (recipient), " + self.getId() +
                  " (myid), 0x" + Long.toHexString(proposedEpoch) + " (n.peerEpoch)");
        }
        sendqueue.offer(notmsg); //放入sendqueue
    }
}
```

org.apache.zookeeper.server.quorum.FastLeaderElection#learningState

```java
private ServerState learningState(){
    if(self.getLearnerType() == LearnerType.PARTICIPANT){
        LOG.debug("I'm a participant: " + self.getId());
        return ServerState.FOLLOWING;
    }
    else{
        LOG.debug("I'm an observer: " + self.getId());
        return ServerState.OBSERVING;
    }
}
```

## Observer

接收来自 leader 的 inform 消息，更新自己的本地存储，不参与提交和选举的投票过程。因此可以通过往集群里面添加 Observer节点来提高整个集群的读性能

### makeObserver

org.apache.zookeeper.server.quorum.QuorumPeer#makeObserver

```java
protected Observer makeObserver(FileTxnSnapLog logFactory) throws IOException {
    return new Observer(this, new ObserverZooKeeperServer(logFactory,
            this, new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
}
```

### observeLeader

org.apache.zookeeper.server.quorum.Observer#observeLeader

```java
void observeLeader() throws InterruptedException {
    zk.registerJMX(new ObserverBean(this, zk), self.jmxLocalPeerBean);
    try {
        //寻找leader
        QuorumServer leaderServer = findLeader();
        LOG.info("Observing " + leaderServer.addr);
        try {
            //与leader建立连接
            connectToLeader(leaderServer.addr, leaderServer.hostname);
            //OBSERVERINFO发送leader，leader返回LEADERINFO，Observer返回ACKEPOCH
            long newLeaderZxid = registerWithLeader(Leader.OBSERVERINFO);
            //同步
            syncWithLeader(newLeaderZxid);
            QuorumPacket qp = new QuorumPacket();
            while (this.isRunning()) {
                readPacket(qp);
                processPacket(qp);                   
            }
        } catch (Exception e) {
            LOG.warn("Exception when observing the leader", e);
            try {
                sock.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
            pendingRevalidations.clear();
        }
    } finally {
        zk.unregisterJMX(this);
    }
}
```

#### 寻找Leader

```java
protected QuorumServer findLeader() {
    QuorumServer leaderServer = null;
    //寻找leader id
    Vote current = self.getCurrentVote();
    for (QuorumServer s : self.getView().values()) {
        if (s.id == current.getId()) {
            s.recreateSocketAddresses();
            leaderServer = s;
            break;
        }
    }
    if (leaderServer == null) {
        LOG.warn("Couldn't find the leader with id = "
                + current.getId());
    }
    return leaderServer;
}
```

#### 连接Leader

```java
protected void connectToLeader(InetSocketAddress addr, String hostname)
        throws IOException, ConnectException, InterruptedException {
    sock = new Socket();        
    sock.setSoTimeout(self.tickTime * self.initLimit);
    for (int tries = 0; tries < 5; tries++) {
        try {
            sock.connect(addr, self.tickTime * self.syncLimit); //建立连接
            sock.setTcpNoDelay(nodelay);
            break;
        } catch (IOException e) {
            if (tries == 4) {
                LOG.error("Unexpected exception",e);
                throw e;
            } else {
                LOG.warn("Unexpected exception, tries="+tries+
                        ", connecting to " + addr,e);
                sock = new Socket();
                sock.setSoTimeout(self.tickTime * self.initLimit);
            }
        }
        Thread.sleep(1000);
    }
    self.authLearner.authenticate(sock, hostname);
  //创建leader的输入、
    leaderIs = BinaryInputArchive.getArchive(new BufferedInputStream(
            sock.getInputStream()));
    bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
    leaderOs = BinaryOutputArchive.getArchive(bufferedOutput);
}   
```

#### 向Leader注册

```java
  protected long registerWithLeader(int pktType) throws IOException{
      //发送myId、zxId	
   		long lastLoggedZxid = self.getLastLoggedZxid(); //最新的zxId
      QuorumPacket qp = new QuorumPacket();
      // OBSERVERINFO
      qp.setType(pktType);
      qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
    
      LearnerInfo li = new LearnerInfo(self.getId(), 0x10000);
      ByteArrayOutputStream bsid = new ByteArrayOutputStream();
      BinaryOutputArchive boa = BinaryOutputArchive.getArchive(bsid);
      boa.writeRecord(li, "LearnerInfo");
      qp.setData(bsid.toByteArray());
      //发送数据包
      writePacket(qp, true);
      //接收数据包
      readPacket(qp);
      final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
      //Leader返回LEADERINFO类型的响应信息
if (qp.getType() == Leader.LEADERINFO) {
       leaderProtocolVersion = ByteBuffer.wrap(qp.getData()).getInt();
       byte epochBytes[] = new byte[4];
       final ByteBuffer wrappedEpochBytes = ByteBuffer.wrap(epochBytes);
       if (newEpoch > self.getAcceptedEpoch()) {
          wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
          self.setAcceptedEpoch(newEpoch);
       } else if (newEpoch == self.getAcceptedEpoch()) {
              wrappedEpochBytes.putInt(-1);
       } else {
          throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
       }
       //向Leader发送ACKEPOCH类型的数据包
       QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
       writePacket(ackNewEpoch, true);
          return ZxidUtils.makeZxid(newEpoch, 0);
      } else {
       if (newEpoch > self.getAcceptedEpoch()) {
          self.setAcceptedEpoch(newEpoch);
       }
          if (qp.getType() != Leader.NEWLEADER) {
              LOG.error("First packet should have been NEWLEADER");
              throw new IOException("First packet should have been NEWLEADER");
          }
          return qp.getZxid();
      }
  } 
```

#### 数据同步

```java
protected void syncWithLeader(long newLeaderZxid) throws IOException, InterruptedException{
    QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
  
    QuorumPacket qp = new QuorumPacket();
    long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);
    boolean snapshotNeeded = true;
  	//读取diff、snap、trunc
    readPacket(qp);
    LinkedList<Long> packetsCommitted = new LinkedList<Long>();
    LinkedList<PacketInFlight> packetsNotCommitted = new LinkedList<PacketInFlight>();
    synchronized (zk) {
        if (qp.getType() == Leader.DIFF) { //增量同步，后续会接收到Leader提议缓存队列中的提议，包括Proposal内容数据包和Commit指令数据包
            LOG.info("Getting a diff from the leader 0x{}", Long.toHexString(qp.getZxid()));
            snapshotNeeded = false;
        }
        else if (qp.getType() == Leader.SNAP) { //快照同步，后续接收Leader节点的全量内存数据
            LOG.info("Getting a snapshot from leader 0x" + Long.toHexString(qp.getZxid()));
            //清空内存数据
            zk.getZKDatabase().clear();
          	//读取快照
            zk.getZKDatabase().deserializeSnapshot(leaderIs);
            String signature = leaderIs.readString("signature");
            if (!signature.equals("BenWasHere")) {
                LOG.error("Missing signature. Got " + signature);
                throw new IOException("Missing signature");                   
            }
            zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
        } else if (qp.getType() == Leader.TRUNC) { //先截断再DIFF同步或仅截断同
            LOG.warn("Truncating log to get in sync with the leader 0x"
                    + Long.toHexString(qp.getZxid()));
            boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
            if (!truncated) {
              	//发生异常，中断虚拟机
                LOG.error("Not able to truncate the log "
                        + Long.toHexString(qp.getZxid()));
                System.exit(13);
            }
            //修改lastProcessedZxId
            zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
        }
        else {
            LOG.error("Got unexpected packet from leader "
                    + qp.getType() + " exiting ... " );
            System.exit(13);
        }
        zk.createSessionTracker();
        long lastQueued = 0;
        boolean isPreZAB1_0 = true;
        boolean writeToTxnLog = !snapshotNeeded; //true
        outerLoop:
        while (self.isRunning()) { //DIFF差异化同步
            readPacket(qp);
            switch(qp.getType()) {
            case Leader.PROPOSAL:
                PacketInFlight pif = new PacketInFlight();
                pif.hdr = new TxnHeader();
                pif.rec = SerializeUtils.deserializeTxn(qp.getData(), pif.hdr);
                if (pif.hdr.getZxid() != lastQueued + 1) {
                LOG.warn("Got zxid 0x"
                        + Long.toHexString(pif.hdr.getZxid())
                        + " expected 0x"
                        + Long.toHexString(lastQueued + 1));
                }
                lastQueued = pif.hdr.getZxid();
                packetsNotCommitted.add(pif);
                break;
            case Leader.COMMIT:
                if (!writeToTxnLog) {
                    pif = packetsNotCommitted.peekFirst();
                    if (pif.hdr.getZxid() != qp.getZxid()) {
                        LOG.warn("Committing " + qp.getZxid() + ", but next proposal is " + pif.hdr.getZxid());
                    } else {
                        zk.processTxn(pif.hdr, pif.rec);
                        packetsNotCommitted.remove();
                    }
                } else {
                    packetsCommitted.add(qp.getZxid());
                }
                break;
            case Leader.INFORM:
                PacketInFlight packet = new PacketInFlight();
                packet.hdr = new TxnHeader();
                packet.rec = SerializeUtils.deserializeTxn(qp.getData(), packet.hdr);
                // Log warning message if txn comes out-of-order
                if (packet.hdr.getZxid() != lastQueued + 1) {
                    LOG.warn("Got zxid 0x"
                            + Long.toHexString(packet.hdr.getZxid())
                            + " expected 0x"
                            + Long.toHexString(lastQueued + 1));
                }
                lastQueued = packet.hdr.getZxid();
                if (!writeToTxnLog) {
                    zk.processTxn(packet.hdr, packet.rec);
                } else {
                    packetsNotCommitted.add(packet);
                    packetsCommitted.add(qp.getZxid());
                }
                break;
            case Leader.UPTODATE: //leader初始化完成之后会发送UPTODATE指令
                if (isPreZAB1_0) {
                    zk.takeSnapshot();
                    self.setCurrentEpoch(newEpoch);
                }
                self.cnxnFactory.setZooKeeperServer(zk);                
                break outerLoop;
            case Leader.NEWLEADER:  //数据同步完成，会接收NEWLEADER指令
                File updating = new File(self.getTxnFactory().getSnapDir(),
                                    QuorumPeer.UPDATING_EPOCH_FILENAME);
                if (!updating.exists() && !updating.createNewFile()) {
                    throw new IOException("Failed to create " +
                                          updating.toString());
                }
                if (snapshotNeeded) {
                    zk.takeSnapshot();
                }
                self.setCurrentEpoch(newEpoch);
                if (!updating.delete()) {
                    throw new IOException("Failed to delete " +
                                          updating.toString());
                }
                writeToTxnLog = true;
                isPreZAB1_0 = false;
                //发送ACK到Leader节点，表明已接收到NEWLEADER命令，完成数据同步
                writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
                break;
            }
        }
    }
    //响应Leader发送的UPTODATE指令，集群具备对外提供服务的能力
    ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
    writePacket(ack, true);

    sock.setSoTimeout(self.tickTime * self.syncLimit);
    zk.startup();
    self.updateElectionVote(newEpoch);
    if (zk instanceof FollowerZooKeeperServer) {//Follower
        FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
        //写入事务日志，返回ack
        for(PacketInFlight p: packetsNotCommitted) {
            fzk.logRequest(p.hdr, p.rec);
        }
        for(Long zxid: packetsCommitted) {
            fzk.commit(zxid);
        }
    } else if (zk instanceof ObserverZooKeeperServer) { //Observer
        ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
         //写入事务日志
        for (PacketInFlight p : packetsNotCommitted) {
            Long zxid = packetsCommitted.peekFirst();
            if (p.hdr.getZxid() != zxid) {
                LOG.warn("Committing " + Long.toHexString(zxid)
                        + ", but next proposal is "
                        + Long.toHexString(p.hdr.getZxid()));
                continue;
            }
            packetsCommitted.remove();
            Request request = new Request(null, p.hdr.getClientId(),
                    p.hdr.getCxid(), p.hdr.getType(), null, null);
            request.txn = p.rec;
            request.hdr = p.hdr;
            //写事务日志、写内存
            ozk.commitRequest(request);
        }
    } else {
        throw new UnsupportedOperationException("Unknown server type");
    }
}
```

#### 处理Leader发送的请求

```java
protected void processPacket(QuorumPacket qp) throws IOException{
    switch (qp.getType()) {
    case Leader.PING: //leader定时发送ping
        ping(qp);
        break;
    case Leader.PROPOSAL:
        LOG.warn("Ignoring proposal");
        break;
    case Leader.COMMIT:
        LOG.warn("Ignoring commit");            
        break;            
    case Leader.UPTODATE:
        LOG.error("Received an UPTODATE message after Observer started");
        break;
    case Leader.REVALIDATE:
        revalidate(qp);
        break;
    case Leader.SYNC:
        ((ObserverZooKeeperServer)zk).sync();
        break;
     //只接受INFORM类型的信息，已经被提交处理过的Propsal请求
    case Leader.INFORM:            
        TxnHeader hdr = new TxnHeader();
        Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
        Request request = new Request (null, hdr.getClientId(), 
                                       hdr.getCxid(),
                                       hdr.getType(), null, null);
        request.txn = txn;
        request.hdr = hdr;
        ObserverZooKeeperServer obs = (ObserverZooKeeperServer)zk;
        //写事务日志、做快照，然后将请求存放到commitProcessor的committedRequests队列中
        obs.commitRequest(request);            
        break;
    default:
        LOG.error("Invalid packet type: {} received by Observer", qp.getType());
    }
}
```

#### 处理器

org.apache.zookeeper.server.quorum.ObserverZooKeeperServer#setupRequestProcessors

```java
protected void setupRequestProcessors() {      
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    commitProcessor = new CommitProcessor(finalProcessor,
            Long.toString(getServerId()), true,
            getZooKeeperServerListener());
    commitProcessor.start();
    firstProcessor = new ObserverRequestProcessor(this, commitProcessor);
    ((ObserverRequestProcessor) firstProcessor).start();
		
    //写入事务日志、做快照，当接收到Leader的INFORM请求时调用此处理进行处理请求
    if (syncRequestProcessorEnabled) {//默认为true，无需设置成员变量nextProcessor
        syncProcessor = new SyncRequestProcessor(this, null);
        syncProcessor.start();
    }
}
```

##### ObserverRequestProcessor

第一个处理器，下一处理器为CommitProcessor

org.apache.zookeeper.server.quorum.ObserverRequestProcessor#run

```java
public void run() {
    try {
        while (!finished) {
            Request request = queuedRequests.take();
            if (LOG.isTraceEnabled()) {
                ZooTrace.logRequest(LOG, ZooTrace.CLIENT_REQUEST_TRACE_MASK,
                        'F', request, "");
            }
            if (request == Request.requestOfDeath) {
                break;
            }
            // 将客户端请求放入CommitProcessor的queuedRequests队列中
            //事务请求一直阻塞，直到接收到Leader发送的INFORM请求
            nextProcessor.processRequest(request);
						//转发事务请求到Leader            
            switch (request.type) {
            case OpCode.sync:
                zks.pendingSyncs.add(request);
                zks.getObserver().request(request);
                break;
            case OpCode.create:
            case OpCode.delete:
            case OpCode.setData:
            case OpCode.setACL:
            case OpCode.createSession:
            case OpCode.closeSession:
            case OpCode.multi:
                zks.getObserver().request(request);
                break;
            }
        }
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
    LOG.info("ObserverRequestProcessor exited loop!");
}
```

```java
synchronized public void processRequest(Request request) {
    // request.addRQRec(">commit");
    if (LOG.isDebugEnabled()) {
        LOG.debug("Processing request:: " + request);
    }
    
    if (!finished) { //未关闭
        queuedRequests.add(request); //queuedRequests存放客户端请求
        notifyAll();
    }
}
```

##### SyncRequestProcessor

无后继处理器，处理Leader发送的事务类请求（已被同步至过半节点）

##### CommitProcessor

后继处理器为FinalRequestProcessor;详情见Leader

org.apache.zookeeper.server.quorum.ObserverZooKeeperServer#commitRequest

```java
public void commitRequest(Request request) {   
   //Observer接收到Leader的INFORM消息
    if (syncRequestProcessorEnabled) {//默认为true
        //写入日志文件并且周期性做快照
        syncProcessor.processRequest(request);
    }
    //放入committedRequests列表 
    commitProcessor.commit(request);        
}
```

##### FinalRequestProcessor

详情见Leader

## Follower

### makeFollower

org.apache.zookeeper.server.quorum.QuorumPeer#makeFollower

```java
protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {
    return new Follower(this, new FollowerZooKeeperServer(logFactory, 
            this,new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
}
```

### followLeader

org.apache.zookeeper.server.quorum.Follower#followLeader

```java
void followLeader() throws InterruptedException {
    self.end_fle = Time.currentElapsedTime();
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken);
    LOG.info("FOLLOWING - LEADER ELECTION TOOK - {}", electionTimeTaken);
    self.start_fle = 0;
    self.end_fle = 0;
    fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
    try {
        //寻找Leader
        QuorumServer leaderServer = findLeader();            
        try {
            //与Leader建立连接
            connectToLeader(leaderServer.addr, leaderServer.hostname);
            //注册FollowerInfo
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
            long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
            if (newEpoch < self.getAcceptedEpoch()) {
                LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                        + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                throw new IOException("Error: Epoch of leader is lower");
            }
            //与Leader进行数据同步
            syncWithLeader(newEpochZxid);                
            QuorumPacket qp = new QuorumPacket();
            while (this.isRunning()) {
                readPacket(qp);
                //处理与Leader同步的数据
                processPacket(qp);
            }
        } catch (Exception e) {
            LOG.warn("Exception when following the leader", e);
            try {
                sock.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
            pendingRevalidations.clear();
        }
    } finally {
        zk.unregisterJMX((Learner)this);
    }
}
```

org.apache.zookeeper.server.quorum.Follower#processPacket

```java
protected void processPacket(QuorumPacket qp) throws IOException{
    switch (qp.getType()) {
    case Leader.PING:            
        ping(qp);            
        break;
    case Leader.PROPOSAL://内容数据包            
        TxnHeader hdr = new TxnHeader();
        Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
        if (hdr.getZxid() != lastQueued + 1) {
            LOG.warn("Got zxid 0x"
                    + Long.toHexString(hdr.getZxid())
                    + " expected 0x"
                    + Long.toHexString(lastQueued + 1));
        }
        lastQueued = hdr.getZxid();
        //将事务请求写入事务日志、返回ACK
        fzk.logRequest(hdr, txn);
        break;
    case Leader.COMMIT:
        //此事务已被过半节点接收，由commitProcessor交给FinalRequestProcessor处理，写入内存树中
        fzk.commit(qp.getZxid());
        break;
    case Leader.UPTODATE:
        LOG.error("Received an UPTODATE message after Follower started");
        break;
    case Leader.REVALIDATE:
        revalidate(qp);
        break;
    case Leader.SYNC:
        fzk.sync();
        break;
    default:
        LOG.error("Invalid packet type: {} received by Observer", qp.getType());
    }
}
```

#### 处理Proposal指令

1、记录事务日志 2、发送ACK

org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#logRequest

```java
public void logRequest(TxnHeader hdr, Record txn) { //接收到Leader的Proposal请求调用此方法
    Request request = new Request(null, hdr.getClientId(), hdr.getCxid(),
            hdr.getType(), null, null);
    request.hdr = hdr;
    request.txn = txn;
    request.zxid = hdr.getZxid();
    if ((request.zxid & 0xffffffffL) != 0) {
        pendingTxns.add(request); //处理中的事务
    }
  	//SyncRequestProcessor -> SendAckRequestProcessor
    syncProcessor.processRequest(request);//写入日志文件、返回ack给Leader
}
```

org.apache.zookeeper.server.SyncRequestProcessor#processRequest

```java
public void processRequest(Request request) {
    queuedRequests.add(request);
}
```

写日志、做快照、组提交

org.apache.zookeeper.server.SyncRequestProcessor#run

```java
public void run() {
    try {
        int logCount = 0;
        //避免全部节点同时做快照
        setRandRoll(r.nextInt(snapCount/2));
        while (true) {
            Request si = null;
            if (toFlush.isEmpty()) {
                si = queuedRequests.take();//无刷新请求时，阻塞获取
            } else {
                //有刷新请求但是队列中无请求
                si = queuedRequests.poll(); 
                if (si == null) { //无请求入队
                    flush(toFlush); //刷新请求，刷盘、调用nextProcessor
                    continue;
                }
            }
            if (si == requestOfDeath) {
                break;
            }
            if (si != null) {
                //写事务日志
                if (zks.getZKDatabase().append(si)) {
                    logCount++;
                    if (logCount > (snapCount / 2 + randRoll)) {
                        setRandRoll(r.nextInt(snapCount/2));
                        // roll the log
                        zks.getZKDatabase().rollLog();
                        if (snapInProcess != null && snapInProcess.isAlive()) {
                            LOG.warn("Too busy to snap, skipping");
                        } else {
                            //异步做快照
                            snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                    public void run() {
                                        try {
                                            zks.takeSnapshot();
                                        } catch(Exception e) {
                                            LOG.warn("Unexpected exception", e);
                                        }
                                    }
                                };
                            snapInProcess.start();
                        }
                        logCount = 0;
                    }
                } else if (toFlush.isEmpty()) {
                    // optimization for read heavy workloads
                    // iff this is a read, and there are no pending
                    // flushes (writes), then just pass this to the next
                    // processor
                    if (nextProcessor != null) {
                        nextProcessor.processRequest(si);
                        if (nextProcessor instanceof Flushable) {
                            ((Flushable)nextProcessor).flush();
                        }
                    }
                    continue;
                }
                toFlush.add(si);
                if (toFlush.size() > 1000) {//累积1000此，做一次刷盘操作
                    flush(toFlush);
                }
            }
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
        running = false;
    }
    LOG.info("SyncRequestProcessor exited!");
}
```

刷盘、调用SendAckRequestProcessor

```java
private void flush(LinkedList<Request> toFlush)
    throws IOException, RequestProcessorException
{
    if (toFlush.isEmpty())
        return;
    //刷盘
    zks.getZKDatabase().commit();
    while (!toFlush.isEmpty()) {
        Request i = toFlush.remove();
        if (nextProcessor != null) {
            //调用nextProcessor
            nextProcessor.processRequest(i);
        }
    }
    if (nextProcessor != null && nextProcessor instanceof Flushable) {
        ((Flushable)nextProcessor).flush();
    }
}
```

org.apache.zookeeper.server.quorum.SendAckRequestProcessor#processRequest

```java
public void processRequest(Request si) {
    if(si.type != OpCode.sync){
        QuorumPacket qp = new QuorumPacket(Leader.ACK, si.hdr.getZxid(), null,
            null);
        try {
            //向Leader返回ACK数据包
            learner.writePacket(qp, false);
        } catch (IOException e) {
            LOG.warn("Closing connection to leader, exception during packet send", e);
            try {
                if (!learner.sock.isClosed()) {
                    learner.sock.close();
                }
            } catch (IOException e1) {
                LOG.debug("Ignoring error closing the connection", e1);
            }
        }
    }
}
```

#### 处理Commit指令

org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#commit

```java
public void commit(long zxid) { //接收Leadr的COMMIT请求
    if (pendingTxns.size() == 0) {
        LOG.warn("Committing " + Long.toHexString(zxid)
                + " without seeing txn");
        return;
    }
  	//获取之前接收的Proposal请求
    long firstElementZxid = pendingTxns.element().zxid;
  	//pendingTxnds头部节点的zxid应该与zxid相等
    if (firstElementZxid != zxid) {
        LOG.error("Committing zxid 0x" + Long.toHexString(zxid)
                + " but next pending txn 0x"
                + Long.toHexString(firstElementZxid));
        System.exit(12);
    }
    Request request = pendingTxns.remove();
  //CommitProcessor -> FinalRequestProcessor
  //只有commitRequest队列中有请求时，直接交给后继处理器处理，写入内存
    commitProcessor.commit(request);
}
```

org.apache.zookeeper.server.quorum.CommitProcessor#commit

```java
synchronized public void commit(Request request) {
    if (!finished) {
        if (request == null) {
            LOG.warn("Committed a null!",
                     new Exception("committing a null! "));
            return;
        }
        if (LOG.isDebugEnabled()) {
            LOG.debug("Committing request:: " + request);
        }
        committedRequests.add(request);
        notifyAll();
    }
}
```

### 请求处理器

org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#setupRequestProcessors

```java
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    commitProcessor = new CommitProcessor(finalProcessor,
            Long.toString(getServerId()), true,
            getZooKeeperServerListener());
    commitProcessor.start();
    firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
    ((FollowerRequestProcessor) firstProcessor).start();
    //同步中接收到Leader的PROPOSAL请求时，调用SyncRequestProcessor，写日志、返回ack
    syncProcessor = new SyncRequestProcessor(this,
            new SendAckRequestProcessor((Learner)getFollower()));
    syncProcessor.start();
}
```

FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor

#### FollowerRequestProcessor

org.apache.zookeeper.server.quorum.FollowerRequestProcessor#run

```java
public void run() {
    try {
        while (!finished) {
            Request request = queuedRequests.take();
            if (LOG.isTraceEnabled()) {
                ZooTrace.logRequest(LOG, ZooTrace.CLIENT_REQUEST_TRACE_MASK,
                        'F', request, "");
            }
            if (request == Request.requestOfDeath) {
                break;
            }
            // 将客户端请求放入CommitProcessor的queuedRequests队列中
            //一直阻塞直到接收到Leader发送的Commit请求
            nextProcessor.processRequest(request);
						//将事务请求转发给leader            
            switch (request.type) {
            case OpCode.sync:
                zks.pendingSyncs.add(request);
                zks.getFollower().request(request);
                break;
            case OpCode.create:
            case OpCode.delete:
            case OpCode.setData:
            case OpCode.setACL:
            case OpCode.createSession:
            case OpCode.closeSession:
            case OpCode.multi:
                zks.getFollower().request(request);
                break;
            }
        }
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
    LOG.info("FollowerRequestProcessor exited loop!");
}
```

org.apache.zookeeper.server.quorum.Learner#request

```java
void request(Request request) throws IOException {
  	//封装Follower、Observer转发给Leader的事务请求
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    DataOutputStream oa = new DataOutputStream(baos);
    oa.writeLong(request.sessionId);
    oa.writeInt(request.cxid);
    oa.writeInt(request.type);
    if (request.request != null) {
        request.request.rewind();
        int len = request.request.remaining();
        byte b[] = new byte[len];
        request.request.get(b);
        request.request.rewind();
        oa.write(b);
    }
    oa.close();
    QuorumPacket qp = new QuorumPacket(Leader.REQUEST, -1, baos
            .toByteArray(), request.authInfo);
    writePacket(qp, true);
}
```

#### CommitProcessor

详情见Leader

```java
synchronized public void commit(Request request) { //接收到Leadre的COMMIT类型请求，执行此方法
    if (!finished) {
        if (request == null) {
            LOG.warn("Committed a null!",
                     new Exception("committing a null! "));
            return;
        }
        if (LOG.isDebugEnabled()) {
            LOG.debug("Committing request:: " + request);
        }
        committedRequests.add(request);
        notifyAll();
    }
}
```

#### SyncRequestProcessor

详情见Leader

#### SendAckRequestProcessor

```java
public void processRequest(Request si) {
    if(si.type != OpCode.sync){
        QuorumPacket qp = new QuorumPacket(Leader.ACK, si.hdr.getZxid(), null,
            null);
        try {
            learner.writePacket(qp, false); //发送ack给leader
        } catch (IOException e) {
            LOG.warn("Closing connection to leader, exception during packet send", e);
            try {
                if (!learner.sock.isClosed()) {
                    learner.sock.close();
                }
            } catch (IOException e1) {
                // Nothing to do, we are shutting things down, so an exception here is irrelevant
                LOG.debug("Ignoring error closing the connection", e1);
            }
        }
    }
}
```

```java
void writePacket(QuorumPacket pp, boolean flush) throws IOException {
    synchronized (leaderOs) {
        if (pp != null) {
            leaderOs.writeRecord(pp, "packet");
        }
        if (flush) {
            bufferedOutput.flush();
        }
    }
}
```

####  FinalRequestProcessor

详情见Leader

## Leader

### 创建Leader

org.apache.zookeeper.server.quorum.QuorumPeer#makeLeader

```java
protected Leader makeLeader(FileTxnSnapLog logFactory) throws IOException {
    return new Leader(this, new LeaderZooKeeperServer(logFactory,
            this,new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
}
```

1、创建LearnerCnxAcceptor并启动

  1.1、监听Follower、Observer节点的连接。

  1.2、接收Follower节点发送的FollowerInfo、Observer发送的ObserverInfo，返回LeaderInfo

  1.3、接收Follower、Observer发送的ACKEPOCH

  1.4、阻塞等待，直到接收到超过半数节点的ACKEPOCH 

  1.5、根据Follower、Observer传递的lastZxId，决定同步策略，DIFF、TRUNC、SNAP

  1.6、发送同步策略给Follower、Observer

  1.7、开启线程同步数据、发送NEWLEADER类型的数据包

  1.8、阻塞，直到接收到超过半数节点的ACK，表明数据同步完成

  1.9、发送UPTODATE，通知Follower、Observer节点，已经完成数据同步，可以对外提供服务

   1.10、开启循环，针对Leader发出的请求，读取Follower、Observer的响应信息，例如ACK、PING

2、Leader阻塞等待，直到接收过半节点的ACKEPOCH

3、Leader阻塞等待，直到半数节点完成数据同步

4、Leader初始化，对外提供服务

5、定时发送PING探测节点的健康

org.apache.zookeeper.server.quorum.Leader#lead

```java
void lead() throws IOException, InterruptedException {
    self.end_fle = Time.currentElapsedTime();
  	//选举花费时间
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken);
    LOG.info("LEADING - LEADER ELECTION TOOK - {}", electionTimeTaken);
    self.start_fle = 0;
    self.end_fle = 0;
    zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);
    try {
        self.tick.set(0);
       	//加载磁盘中的数据，快照和事务日志
        zk.loadData();
        leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());
        //接收Follower、Observer节点的请求
        cnxAcceptor = new LearnerCnxAcceptor();
        cnxAcceptor.start();
        readyToStart = true;
        long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());
        //设置选举之后，初始的事务Id
        zk.setZxid(ZxidUtils.makeZxid(epoch, 0));
        synchronized(this){
            lastProposed = zk.getZxid();
        }
        newLeaderProposal.packet = new QuorumPacket(NEWLEADER, zk.getZxid(),
                null, null);
        if ((newLeaderProposal.packet.getZxid() & 0xffffffffL) != 0) {
            LOG.info("NEWLEADER proposal has Zxid of "
                    + Long.toHexString(newLeaderProposal.packet.getZxid()));
        }
        //阻塞等待握手完成
        waitForEpochAck(self.getId(), leaderStateSummary);
        self.setCurrentEpoch(epoch);
        try {
            //阻塞等待，直到半数节点完成数据同步
            waitForNewLeaderAck(self.getId(), zk.getZxid());
        } catch (InterruptedException e) {
            shutdown("Waiting for a quorum of followers, only synced with sids: [ "
                    + getSidSetString(newLeaderProposal.ackSet) + " ]");
            HashSet<Long> followerSet = new HashSet<Long>();
            for (LearnerHandler f : learners)
                followerSet.add(f.getSid());
                
            if (self.getQuorumVerifier().containsQuorum(followerSet)) {
                LOG.warn("Enough followers present. "
                        + "Perhaps the initTicks need to be increased.");
            }
            Thread.sleep(self.tickTime);
            self.tick.incrementAndGet();
            return;
        }
        //Leader初始化，对外提供服务
        startZkServer();
        String initialZxid = System.getProperty("zookeeper.testingonly.initialZxid");
        if (initialZxid != null) {
            long zxid = Long.parseLong(initialZxid);
            zk.setZxid((zk.getZxid() & 0xffffffff00000000L) | zxid);
        }
        if (!System.getProperty("zookeeper.leaderServes", "yes").equals("no")) {
            self.cnxnFactory.setZooKeeperServer(zk);
        }
        //间隔=self.tickTime / 2，向集群节点发送Ping消息，
        boolean tickSkip = true;
        while (true) {
            Thread.sleep(self.tickTime / 2);
            if (!tickSkip) {
                self.tick.incrementAndGet();
            }
            HashSet<Long> syncedSet = new HashSet<Long>();
            syncedSet.add(self.getId());
            for (LearnerHandler f : getLearners()) {
              	 //参与选举的节点，心跳都没有超时，加入到syncedSet
                if (f.synced() && f.getLearnerType() == LearnerType.PARTICIPANT) {
                    syncedSet.add(f.getSid());
                }
              	//发送Ping请求
                f.ping();
            }
            // check leader running status
            if (!this.isRunning()) {
                shutdown("Unexpected internal error");
                return;
            }
            //超过半数的Follower节点心跳超时
          if (!tickSkip && !self.getQuorumVerifier().containsQuorum(syncedSet)) {
            //if (!tickSkip && syncedCount < self.quorumPeers.size() / 2) {
                // Lost quorum, shutdown
                shutdown("Not sufficient followers synced, only synced with sids: [ "
                        + getSidSetString(syncedSet) + " ]");
                // make sure the order is the same!
                // the leader goes to looking
                return;
          } 
          tickSkip = !tickSkip;
        }
    } finally {
        zk.unregisterJMX(this);
    }
}
```

### LearnerCnxAcceptor

处理Follower、Observer连接

org.apache.zookeeper.server.quorum.Leader.LearnerCnxAcceptor#run

```java
public void run() {
    try {
        while (!stop) {
            try{
                Socket s = ss.accept();
                s.setSoTimeout(self.tickTime * self.initLimit);
                s.setTcpNoDelay(nodelay);

                BufferedInputStream is = new BufferedInputStream(
                        s.getInputStream());
                LearnerHandler fh = new LearnerHandler(s, is, Leader.this);
                fh.start();
            } catch (SocketException e) {
                if (stop) {
                    LOG.info("exception while shutting down acceptor: "
                            + e);
                    stop = true;
                } else {
                    throw e;
                }
            } catch (SaslException e){
                LOG.error("Exception while connecting to quorum learner", e);
            }
        }
    } catch (Exception e) {
        LOG.warn("Exception while accepting follower", e);
    }
}
```

#### LearnerHandler

管理Leader与Follower、Observer节点的数据同步

org.apache.zookeeper.server.quorum.LearnerHandler#run

```java
public void run() {
    try {
      	//Leader维护LearnerHandler
        leader.addLearnerHandler(this);
        tickOfNextAckDeadline = leader.self.tick.get()
                + leader.self.initLimit + leader.self.syncLimit;
        ia = BinaryInputArchive.getArchive(bufferedInput);
        bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
        oa = BinaryOutputArchive.getArchive(bufferedOutput);

        QuorumPacket qp = new QuorumPacket();
        ia.readRecord(qp, "packet");
        //Leader启动后，接收从节点发送的FOLLOWERINFO、OBSERVERINFO，
        //包含Follower、Observer服务器的SID、已经处理过的最新的ZXID
        if(qp.getType() != Leader.FOLLOWERINFO && qp.getType() != Leader.OBSERVERINFO){
           LOG.error("First packet " + qp.toString()
                    + " is not FOLLOWERINFO or OBSERVERINFO!");
            return;
        }
        byte learnerInfoData[] = qp.getData();
        if (learnerInfoData != null) {
           if (learnerInfoData.length == 8) {
              ByteBuffer bbsid = ByteBuffer.wrap(learnerInfoData);
              this.sid = bbsid.getLong();
           } else {
              LearnerInfo li = new LearnerInfo();
              ByteBufferInputStream.byteBuffer2Record(ByteBuffer.wrap(learnerInfoData), li);
              this.sid = li.getServerid();
              this.version = li.getProtocolVersion();
           }
        } else {
           this.sid = leader.followerCounter.getAndDecrement();
        }
        LOG.info("Follower sid: " + sid + " : info : "
                + leader.self.quorumPeers.get(sid));
        if (qp.getType() == Leader.OBSERVERINFO) {
              learnerType = LearnerType.OBSERVER;
        }            
        long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
        long peerLastZxid;
        StateSummary ss = null;
        long zxid = qp.getZxid();
        long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
        if (this.getVersion() < 0x10000) {
            // we are going to have to extrapolate the epoch information
            long epoch = ZxidUtils.getEpochFromZxid(zxid);
            ss = new StateSummary(epoch, zxid);
            // fake the message
            leader.waitForEpochAck(this.getSid(), ss);
        } else {
            //返回LEADERINFO给Follower、Observer
            byte ver[] = new byte[4];
            ByteBuffer.wrap(ver).putInt(0x10000);
            QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, ZxidUtils.makeZxid(newEpoch, 0), ver, null);
            oa.writeRecord(newEpochPacket, "packet");
            bufferedOutput.flush();
            //Follower、Observer收到LEADERINFO后，返回ACKEPOCH消息给Leader
            QuorumPacket ackEpochPacket = new QuorumPacket();
            ia.readRecord(ackEpochPacket, "packet");
            if (ackEpochPacket.getType() != Leader.ACKEPOCH) {
                LOG.error(ackEpochPacket.toString()
                        + " is not ACKEPOCH");
                return;
}
            //Leader接收到ACKEPOCH，握手结束
            ByteBuffer bbepoch = ByteBuffer.wrap(ackEpochPacket.getData());
            ss = new StateSummary(bbepoch.getInt(), ackEpochPacket.getZxid());
            //阻塞，等待过半Participant类型的节点返回ACKEPOCH
            leader.waitForEpochAck(this.getSid(), ss);
        }
        peerLastZxid = ss.getLastZxid();
      	//默认是发送快照进行同步数据
        int packetToSend = Leader.SNAP;
        long zxidToSend = 0;
        long leaderLastZxid = 0;
        long updates = peerLastZxid;
        ReentrantReadWriteLock lock = leader.zk.getZKDatabase().getLogLock();
        ReadLock rl = lock.readLock();
        //根据Follower、Observer传递的lastZxId，决定同步策略，DIFF、TRUNC、SNAP
        try {
            rl.lock();        
            final long maxCommittedLog = leader.zk.getZKDatabase().getmaxCommittedLog();
            final long minCommittedLog = leader.zk.getZKDatabase().getminCommittedLog();
            LOG.info("Synchronizing with Follower sid: " + sid
                    +" maxCommittedLog=0x"+Long.toHexString(maxCommittedLog)
                    +" minCommittedLog=0x"+Long.toHexString(minCommittedLog)
                    +" peerLastZxid=0x"+Long.toHexString(peerLastZxid));

            LinkedList<Proposal> proposals = leader.zk.getZKDatabase().getCommittedLog();

            if (peerLastZxid == leader.zk.getZKDatabase().getDataTreeLastProcessedZxid()) { 
                // Follower与Leader的数据已经保持一致
                LOG.info("leader and follower are in sync, zxid=0x{}",
                        Long.toHexString(peerLastZxid));
                packetToSend = Leader.DIFF;
                zxidToSend = peerLastZxid;
            } else if (proposals.size() != 0) { 
              //内存中保存一部分最新的已提交的事务请求，判断Follower缺失的数据是否在Leader的commitLog中
                LOG.debug("proposal size is {}", proposals.size());
                if ((maxCommittedLog >= peerLastZxid)
                        && (minCommittedLog <= peerLastZxid)) {//
                    LOG.debug("Sending proposals to follower");

                    long prevProposalZxid = minCommittedLog;

                    boolean firstPacket=true;

                    packetToSend = Leader.DIFF;
                    zxidToSend = maxCommittedLog;

                    for (Proposal propose: proposals) {
                        // 跳过已经同步的事务
                        if (propose.packet.getZxid() <= peerLastZxid) {
                            prevProposalZxid = propose.packet.getZxid();
                            continue;
                        } else {
                            if (firstPacket) {
                                firstPacket = false;
                                //含有leader节点没有的事务请求，进行截断
                                if (prevProposalZxid < peerLastZxid) {
                                    // send a trunc message before sending the diff
                                    packetToSend = Leader.TRUNC;                                        
                                    zxidToSend = prevProposalZxid;
                                    updates = zxidToSend;
                                }
                            }
                          	//将缺失的事务放在队列中，稍后发送给其他节点
                            queuePacket(propose.packet);
                            QuorumPacket qcommit = new QuorumPacket(Leader.COMMIT, propose.packet.getZxid(),
                                    null, null);
                            queuePacket(qcommit);
                        }
                    }
                } else if (peerLastZxid > maxCommittedLog) { //非Leader节点需要截断事务日志
                    LOG.debug("Sending TRUNC to follower zxidToSend=0x{} updates=0x{}",
                            Long.toHexString(maxCommittedLog),
                            Long.toHexString(updates));
                  //比leader含有多余的数据，将非Leader节点maxCommittedLog之后的日志删除
                    packetToSend = Leader.TRUNC;
                    zxidToSend = maxCommittedLog;
                    updates = zxidToSend;
                } else { //落后于leader太多数据，落后的数据在committedLog不存在，进行快照同步
                    LOG.warn("Unhandled proposal scenario");
                }
            } else {
                // just let the state transfer happen
                LOG.debug("proposals is empty");
            }               
            LOG.info("Sending " + Leader.getPacketType(packetToSend));
            leaderLastZxid = leader.startForwarding(this, updates);

        } finally {
            rl.unlock();
        }
      	//缺失的数据都已放到queuedPackets中，newLeaderQP也放入到此队列，告知其他节点数据已同步完成
         QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
                ZxidUtils.makeZxid(newEpoch, 0), null, null);
         if (getVersion() < 0x10000) {
            oa.writeRecord(newLeaderQP, "packet");
        } else {
            queuedPackets.add(newLeaderQP);
        }
        bufferedOutput.flush();
        //Need to set the zxidToSend to the latest zxid
        if (packetToSend == Leader.SNAP) {
          	//当前最新的事务Id
            zxidToSend = leader.zk.getZKDatabase().getDataTreeLastProcessedZxid();
        }
      	//发送同步请求(DIFF/SNAP/TRUNC)
        oa.writeRecord(new QuorumPacket(packetToSend, zxidToSend, null, null), "packet");
        bufferedOutput.flush();
        /* if we are not truncating or sending a diff just send a snapshot */
        if (packetToSend == Leader.SNAP) {
            LOG.info("Sending snapshot last zxid of peer is 0x"
                    + Long.toHexString(peerLastZxid) + " " 
                    + " zxid of leader is 0x"
                    + Long.toHexString(leaderLastZxid)
                    + "sent zxid of db as 0x" 
                    + Long.toHexString(zxidToSend));
            // 将内存中的数据序列化成快照发送给其他节点
            leader.zk.getZKDatabase().serializeSnapshot(oa);
            oa.writeString("BenWasHere", "signature");
        }
        bufferedOutput.flush();
        //开启线程同步数据、发送NEWLEADER类型的数据包
        new Thread() {
            public void run() {
                Thread.currentThread().setName(
                        "Sender-" + sock.getRemoteSocketAddress());
                try {
                  	//发送内存中的最新的已经提交的事务
                    sendPackets();
                } catch (InterruptedException e) {
                    LOG.warn("Unexpected interruption",e);
                }
            }
        }.start();
        qp = new QuorumPacket();
        ia.readRecord(qp, "packet");
        if(qp.getType() != Leader.ACK){
            LOG.error("Next packet was supposed to be an ACK");
            return;
        }
        LOG.info("Received NEWLEADER-ACK message from " + getSid());
        //阻塞，直到接收到超过半数节点的ACK，表明数据同步完成
        leader.waitForNewLeaderAck(getSid(), qp.getZxid());
        syncLimitCheck.start();
        sock.setSoTimeout(leader.self.tickTime * leader.self.syncLimit);
        //等待Leader初始化完成
        synchronized(leader.zk){
            while(!leader.zk.isRunning() && !this.isInterrupted()){
                leader.zk.wait(20);
            }
        }
        //发送UPTODATE，通知Follower、Observer节点，已经完成数据同步，可以对外提供服务
        queuedPackets.add(new QuorumPacket(Leader.UPTODATE, -1, null, null));
        //开启循环，针对Leader发出的请求，读取Follower、Observer的请求信息，例如ACK、PING。还会接收转发的事务类请求
        while (true) {
            qp = new QuorumPacket();
            ia.readRecord(qp, "packet");
            long traceMask = ZooTrace.SERVER_PACKET_TRACE_MASK;
            if (qp.getType() == Leader.PING) {
                traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
            }
            if (LOG.isTraceEnabled()) {
                ZooTrace.logQuorumPacket(LOG, traceMask, 'i', qp);
            }
            tickOfNextAckDeadline = leader.self.tick.get() + leader.self.syncLimit;
            ByteBuffer bb;
            long sessionId;
            int cxid;
            int type;
            switch (qp.getType()) {
            case Leader.ACK: //Follower节点同步完事务请求会返回ACK
                if (this.learnerType == LearnerType.OBSERVER) {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Received ACK from Observer  " + this.sid);
                    }
                }
                syncLimitCheck.updateAck(qp.getZxid());
                leader.processAck(this.sid, qp.getZxid(), sock.getLocalSocketAddress());
                break;
            case Leader.PING: //续约
                // Process the touches
                ByteArrayInputStream bis = new ByteArrayInputStream(qp
                        .getData());
                DataInputStream dis = new DataInputStream(bis);
                while (dis.available() > 0) {
                    long sess = dis.readLong();
                    int to = dis.readInt();
                    //心跳续约
                    leader.zk.touch(sess, to);
                }
                break;
            case Leader.REVALIDATE://校验会话是否有效
                bis = new ByteArrayInputStream(qp.getData());
                dis = new DataInputStream(bis);
                long id = dis.readLong();
                int to = dis.readInt();
                ByteArrayOutputStream bos = new ByteArrayOutputStream();
                DataOutputStream dos = new DataOutputStream(bos);
                dos.writeLong(id);
                //
                boolean valid = leader.zk.touch(id, to);
                if (valid) {
                    try {
                        //set the session owner
                        // as the follower that
                        // owns the session
                        leader.zk.setOwner(id, this);
                    } catch (SessionExpiredException e) {
                        LOG.error("Somehow session " + Long.toHexString(id) + " expired right after being renewed! (impossible)", e);
                    }
                }
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logTraceMessage(LOG,
                                             ZooTrace.SESSION_TRACE_MASK,
                                             "Session 0x" + Long.toHexString(id)
                                             + " is valid: "+ valid);
                }
                dos.writeBoolean(valid);
                qp.setData(bos.toByteArray());
                queuedPackets.add(qp);
                break;
            case Leader.REQUEST: //接收转发的事务类请求
                bb = ByteBuffer.wrap(qp.getData());
                sessionId = bb.getLong();
                cxid = bb.getInt();
                type = bb.getInt();
                bb = bb.slice();
                Request si;
                if(type == OpCode.sync){
                    si = new LearnerSyncRequest(this, sessionId, cxid, type, bb, qp.getAuthinfo());
                } else {
                    si = new Request(null, sessionId, cxid, type, bb, qp.getAuthinfo());
                }
                si.setOwner(this);
                leader.zk.submitRequest(si);
                break;
            default:
                LOG.warn("unexpected quorum packet, type: {}", packetToString(qp));
                break;
            }
        }
    } catch (IOException e) {
        if (sock != null && !sock.isClosed()) {
            LOG.error("Unexpected exception causing shutdown while sock "
                    + "still open", e);
           //close the socket to make sure the 
           //other side can see it being close
           try {
              sock.close();
           } catch(IOException ie) {
              // do nothing
           }
        }
    } catch (InterruptedException e) {
        LOG.error("Unexpected exception causing shutdown", e);
    } finally {
        LOG.warn("******* GOODBYE " 
                + (sock != null ? sock.getRemoteSocketAddress() : "<null>")
                + " ********");
        shutdown();
    }
}
```

### 对外提供服务

org.apache.zookeeper.server.quorum.Leader#startZkServer

```java
private synchronized void startZkServer() {
    // Update lastCommitted and Db's zxid to a value representing the new epoch
    lastCommitted = zk.getZxid();
    LOG.info("Have quorum of supporters, sids: [ "
            + getSidSetString(newLeaderProposal.ackSet)
            + " ]; starting up and setting last processed zxid: 0x{}",
            Long.toHexString(zk.getZxid()));
    zk.startup();
  
    self.updateElectionVote(getEpoch());

    zk.getZKDatabase().setlastProcessedZxid(zk.getZxid());
}
```

org.apache.zookeeper.server.ZooKeeperServer#startup

```java
public synchronized void startup() {
  	//创建会话管理
    if (sessionTracker == null) {
        createSessionTracker();
    }
   //启动会话管理
    startSessionTracker();
  	//设置集群模式下的请求处理器
    setupRequestProcessors();
    registerJMX();
    setState(State.RUNNING);
    notifyAll();
}
```

**PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor**

org.apache.zookeeper.server.quorum.LeaderZooKeeperServer#setupRequestProcessors

```java
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(
            finalProcessor, getLeader().toBeApplied);
    commitProcessor = new CommitProcessor(toBeAppliedProcessor,
            Long.toString(getServerId()), false,
            getZooKeeperServerListener());
    commitProcessor.start();
    ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
            commitProcessor);
    proposalProcessor.initialize();
    firstProcessor = new PrepRequestProcessor(this, proposalProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}
```

#### PrepRequestProcessor

预处理所有类型的请求。对于非事务请求，只会验证会话的有效性，会话未过期，将请求交由后继Processor处理。

对于事务请求，除了验证会话的有效性，还会创建TxnHeader。非事务请求TxnHeader为空，事务请求TxnHeader不为空

org.apache.zookeeper.server.PrepRequestProcessor#run

```java
public void run() {
    try {
        while (true) {
            Request request = submittedRequests.take();
            long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
            if (request.type == OpCode.ping) {
                traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
            }
            if (LOG.isTraceEnabled()) {
                ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
            }
            if (Request.requestOfDeath == request) {
                break;
            }
            pRequest(request); //处理客户端请求或者来自Follower、Observer转发的事务请求
        }
    } catch (RequestProcessorException e) {
        if (e.getCause() instanceof XidRolloverException) {
            LOG.info(e.getCause().getMessage());
        }
        handleException(this.getName(), e);
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
    LOG.info("PrepRequestProcessor exited loop!");
}
```

```java
protected void pRequest(Request request) throws RequestProcessorException {
    // LOG.info("Prep>>> cxid = " + request.cxid + " type = " +
    // request.type + " id = 0x" + Long.toHexString(request.sessionId));
    request.hdr = null;
    request.txn = null;
    
    try {
        switch (request.type) {
            case OpCode.create:
            CreateRequest createRequest = new CreateRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, createRequest, true);
            break;
        case OpCode.delete:
            DeleteRequest deleteRequest = new DeleteRequest();               
            pRequest2Txn(request.type, zks.getNextZxid(), request, deleteRequest, true);
            break;
        case OpCode.setData:
            SetDataRequest setDataRequest = new SetDataRequest();                
            pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
            break;
        case OpCode.setACL:
            SetACLRequest setAclRequest = new SetACLRequest();                
            pRequest2Txn(request.type, zks.getNextZxid(), request, setAclRequest, true);
            break;
        case OpCode.check:
            CheckVersionRequest checkRequest = new CheckVersionRequest();              
            pRequest2Txn(request.type, zks.getNextZxid(), request, checkRequest, true);
            break;
        case OpCode.multi:
            MultiTransactionRecord multiRequest = new MultiTransactionRecord();
            try {
                ByteBufferInputStream.byteBuffer2Record(request.request, multiRequest);
            } catch(IOException e) {
                request.hdr =  new TxnHeader(request.sessionId, request.cxid, zks.getNextZxid(),
                        Time.currentWallTime(), OpCode.multi);
                throw e;
            }
            List<Txn> txns = new ArrayList<Txn>();
            //Each op in a multi-op must have the same zxid!
            long zxid = zks.getNextZxid();
            KeeperException ke = null;

            //Store off current pending change records in case we need to rollback
            HashMap<String, ChangeRecord> pendingChanges = getPendingChanges(multiRequest);

            int index = 0;
            for(Op op: multiRequest) {
                Record subrequest = op.toRequestRecord() ;

                /* If we've already failed one of the ops, don't bother
                 * trying the rest as we know it's going to fail and it
                 * would be confusing in the logfiles.
                 */
                if (ke != null) {
                    request.hdr.setType(OpCode.error);
                    request.txn = new ErrorTxn(Code.RUNTIMEINCONSISTENCY.intValue());
                } 
                
                /* Prep the request and convert to a Txn */
                else {
                    try {
                        pRequest2Txn(op.getType(), zxid, request, subrequest, false);
                    } catch (KeeperException e) {
                        ke = e;
                        request.hdr.setType(OpCode.error);
                        request.txn = new ErrorTxn(e.code().intValue());
                        LOG.info("Got user-level KeeperException when processing "
                              + request.toString() + " aborting remaining multi ops."
                              + " Error Path:" + e.getPath()
                              + " Error:" + e.getMessage());

                        request.setException(e);

                        /* Rollback change records from failed multi-op */
                        rollbackPendingChanges(zxid, pendingChanges);
                    }
                }

                //FIXME: I don't want to have to serialize it here and then
                //       immediately deserialize in next processor. But I'm 
                //       not sure how else to get the txn stored into our list.
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                request.txn.serialize(boa, "request") ;
                ByteBuffer bb = ByteBuffer.wrap(baos.toByteArray());

                txns.add(new Txn(request.hdr.getType(), bb.array()));
                index++;
            }

            request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid,
                    Time.currentWallTime(), request.type);
            request.txn = new MultiTxn(txns);
            
            break;

        //create/close session don't require request record
        case OpCode.createSession:
        case OpCode.closeSession:
            pRequest2Txn(request.type, zks.getNextZxid(), request, null, true);
            break;

        //All the rest don't need to create a Txn - just verify session
        case OpCode.sync:
        case OpCode.exists:
        case OpCode.getData:
        case OpCode.getACL:
        case OpCode.getChildren:
        case OpCode.getChildren2:
        case OpCode.ping:
        case OpCode.setWatches:
            zks.sessionTracker.checkSession(request.sessionId,
                    request.getOwner());
            break;
        default:
            LOG.warn("unknown type " + request.type);
            break;
        }
    } catch (KeeperException e) {
        if (request.hdr != null) {
            request.hdr.setType(OpCode.error);
            request.txn = new ErrorTxn(e.code().intValue());
        }
        LOG.info("Got user-level KeeperException when processing "
                + request.toString()
                + " Error Path:" + e.getPath()
                + " Error:" + e.getMessage());
        request.setException(e);
    } catch (Exception e) {
        // log at error level as we are returning a marshalling
        // error to the user
        LOG.error("Failed to process " + request, e);

        StringBuilder sb = new StringBuilder();
        ByteBuffer bb = request.request;
        if(bb != null){
            bb.rewind();
            while (bb.hasRemaining()) {
                sb.append(Integer.toHexString(bb.get() & 0xff));
            }
        } else {
            sb.append("request buffer is null");
        }

        LOG.error("Dumping request buffer: 0x" + sb.toString());
        if (request.hdr != null) {
            request.hdr.setType(OpCode.error);
            request.txn = new ErrorTxn(Code.MARSHALLINGERROR.intValue());
        }
    }
    request.zxid = zks.getZxid();
    nextProcessor.processRequest(request);
}
```

```java
 protected void pRequest2Txn(int type, long zxid, Request request, Record record, boolean deserialize)
    throws KeeperException, IOException, RequestProcessorException
{//预处理事务请求
   	//创建事务请求头
    request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid,
                                Time.currentWallTime(), type);

    switch (type) {
        case OpCode.create:                
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            CreateRequest createRequest = (CreateRequest)record;   
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, createRequest);
            String path = createRequest.getPath();
            int lastSlash = path.lastIndexOf('/');
            if (lastSlash == -1 || path.indexOf('\0') != -1 || failCreate) {
                LOG.info("Invalid path " + path + " with session 0x" +
                        Long.toHexString(request.sessionId));
                throw new KeeperException.BadArgumentsException(path);
            }
            List<ACL> listACL = removeDuplicates(createRequest.getAcl());
            if (!fixupACL(request.authInfo, listACL)) {
                throw new KeeperException.InvalidACLException(path);
            }
            String parentPath = path.substring(0, lastSlash);
            ChangeRecord parentRecord = getRecordForPath(parentPath);

            checkACL(zks, parentRecord.acl, ZooDefs.Perms.CREATE,
                    request.authInfo);
            int parentCVersion = parentRecord.stat.getCversion();
            CreateMode createMode =
                CreateMode.fromFlag(createRequest.getFlags());
            if (createMode.isSequential()) {
                path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
            }
            validatePath(path, request.sessionId);
            try {
                if (getRecordForPath(path) != null) {
                    throw new KeeperException.NodeExistsException(path);
                }
            } catch (KeeperException.NoNodeException e) {
                // ignore this one
            }
            boolean ephemeralParent = parentRecord.stat.getEphemeralOwner() != 0;
            if (ephemeralParent) {
                throw new KeeperException.NoChildrenForEphemeralsException(path);
            }
            int newCversion = parentRecord.stat.getCversion()+1;
            request.txn = new CreateTxn(path, createRequest.getData(),
                    listACL,
                    createMode.isEphemeral(), newCversion);
            StatPersisted s = new StatPersisted();
            if (createMode.isEphemeral()) {
                s.setEphemeralOwner(request.sessionId);
            }
            parentRecord = parentRecord.duplicate(request.hdr.getZxid());
            parentRecord.childCount++;
            parentRecord.stat.setCversion(newCversion);
            addChangeRecord(parentRecord);
            addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path, s,
                    0, listACL));
            break;
        case OpCode.delete:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            DeleteRequest deleteRequest = (DeleteRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, deleteRequest);
            path = deleteRequest.getPath();
            lastSlash = path.lastIndexOf('/');
            if (lastSlash == -1 || path.indexOf('\0') != -1
                    || zks.getZKDatabase().isSpecialPath(path)) {
                throw new KeeperException.BadArgumentsException(path);
            }
            parentPath = path.substring(0, lastSlash);
            parentRecord = getRecordForPath(parentPath);
            ChangeRecord nodeRecord = getRecordForPath(path);
            checkACL(zks, parentRecord.acl, ZooDefs.Perms.DELETE,
                    request.authInfo);
            int version = deleteRequest.getVersion();
            if (version != -1 && nodeRecord.stat.getVersion() != version) {
                throw new KeeperException.BadVersionException(path);
            }
            if (nodeRecord.childCount > 0) {
                throw new KeeperException.NotEmptyException(path);
            }
            request.txn = new DeleteTxn(path);
            parentRecord = parentRecord.duplicate(request.hdr.getZxid());
            parentRecord.childCount--;
            addChangeRecord(parentRecord);
            addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path,
                    null, -1, null));
            break;
        case OpCode.setData:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            SetDataRequest setDataRequest = (SetDataRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, setDataRequest);
            path = setDataRequest.getPath();
            validatePath(path, request.sessionId);
            nodeRecord = getRecordForPath(path);
            checkACL(zks, nodeRecord.acl, ZooDefs.Perms.WRITE,
                    request.authInfo);
            version = setDataRequest.getVersion();
            int currentVersion = nodeRecord.stat.getVersion();
            if (version != -1 && version != currentVersion) {
                throw new KeeperException.BadVersionException(path);
            }
            version = currentVersion + 1;
            request.txn = new SetDataTxn(path, setDataRequest.getData(), version);
            nodeRecord = nodeRecord.duplicate(request.hdr.getZxid());
            nodeRecord.stat.setVersion(version);
            addChangeRecord(nodeRecord);
            break;
        case OpCode.setACL:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            SetACLRequest setAclRequest = (SetACLRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, setAclRequest);
            path = setAclRequest.getPath();
            validatePath(path, request.sessionId);
            listACL = removeDuplicates(setAclRequest.getAcl());
            if (!fixupACL(request.authInfo, listACL)) {
                throw new KeeperException.InvalidACLException(path);
            }
            nodeRecord = getRecordForPath(path);
            checkACL(zks, nodeRecord.acl, ZooDefs.Perms.ADMIN,
                    request.authInfo);
            version = setAclRequest.getVersion();
            currentVersion = nodeRecord.stat.getAversion();
            if (version != -1 && version != currentVersion) {
                throw new KeeperException.BadVersionException(path);
            }
            version = currentVersion + 1;
            request.txn = new SetACLTxn(path, listACL, version);
            nodeRecord = nodeRecord.duplicate(request.hdr.getZxid());
            nodeRecord.stat.setAversion(version);
            addChangeRecord(nodeRecord);
            break;
        case OpCode.createSession:
            request.request.rewind();
            int to = request.request.getInt();
            request.txn = new CreateSessionTxn(to);
            request.request.rewind();
            zks.sessionTracker.addSession(request.sessionId, to);
            zks.setOwner(request.sessionId, request.getOwner());
            break;
        case OpCode.closeSession:
            // We don't want to do this check since the session expiration thread
            // queues up this operation without being the session owner.
            // this request is the last of the session so it should be ok
            //zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            HashSet<String> es = zks.getZKDatabase()
                    .getEphemerals(request.sessionId);
            synchronized (zks.outstandingChanges) {
                for (ChangeRecord c : zks.outstandingChanges) {
                    if (c.stat == null) {
                        // Doing a delete
                        es.remove(c.path);
                    } else if (c.stat.getEphemeralOwner() == request.sessionId) {
                        es.add(c.path);
                    }
                }
                for (String path2Delete : es) {
                    addChangeRecord(new ChangeRecord(request.hdr.getZxid(),
                            path2Delete, null, 0, null));
                }

                zks.sessionTracker.setSessionClosing(request.sessionId);
            }

            LOG.info("Processed session termination for sessionid: 0x"
                    + Long.toHexString(request.sessionId));
            break;
        case OpCode.check:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            CheckVersionRequest checkVersionRequest = (CheckVersionRequest)record;
            if(deserialize)
                ByteBufferInputStream.byteBuffer2Record(request.request, checkVersionRequest);
            path = checkVersionRequest.getPath();
            validatePath(path, request.sessionId);
            nodeRecord = getRecordForPath(path);
            checkACL(zks, nodeRecord.acl, ZooDefs.Perms.READ,
                    request.authInfo);
            version = checkVersionRequest.getVersion();
            currentVersion = nodeRecord.stat.getVersion();
            if (version != -1 && version != currentVersion) {
                throw new KeeperException.BadVersionException(path);
            }
            version = currentVersion + 1;
            request.txn = new CheckVersionTxn(path, version);
            break;
        default:
            LOG.error("Invalid OpCode: {} received by PrepRequestProcessor", type);
    }
}
```

#### ProposalRequestProcessor

后继处理器为CommitProcessor

将事务请求发送给Follower

将事务请求写入本地事务日志文件

等待过半Follower节点的ACK

​	发送COMMIT请求给Follower

​    发送INFORM给Observer

##### 初始化

org.apache.zookeeper.server.quorum.ProposalRequestProcessor#ProposalRequestProcessor

```java
public ProposalRequestProcessor(LeaderZooKeeperServer zks,
        RequestProcessor nextProcessor) {
    this.zks = zks;
    this.nextProcessor = nextProcessor;
    AckRequestProcessor ackProcessor = new AckRequestProcessor(zks.getLeader());
  //设置nextProcessor为AckRequestProcessor，写入日志成功后返回ack
    syncProcessor = new SyncRequestProcessor(zks, ackProcessor);
}
```

##### 启动SyncProcessor

org.apache.zookeeper.server.quorum.ProposalRequestProcessor#initialize

```java
public void initialize() {
    syncProcessor.start();
}
```

##### 处理请求

org.apache.zookeeper.server.quorum.ProposalRequestProcessor#processRequest

```java
public void processRequest(Request request) throws RequestProcessorException {
    
    if(request instanceof LearnerSyncRequest){
        zks.getLeader().processSync((LearnerSyncRequest)request);
    } else {
      	//交由后继Processor处理
         nextProcessor.processRequest(request);
        if (request.hdr != null) { //hdr不为空表明为事务操作
            try {
                zks.getLeader().propose(request); //向其他Follower发送事务操作
            } catch (XidRolloverException e) {
                throw new RequestProcessorException(e.getMessage(), e);
            }
            syncProcessor.processRequest(request); //写入日志文件
        }
    }
}
```

###### 后继处理器处理请求

org.apache.zookeeper.server.quorum.CommitProcessor#processRequest

```java
synchronized public void processRequest(Request request) {
    // request.addRQRec(">commit");
    if (LOG.isDebugEnabled()) {
        LOG.debug("Processing request:: " + request);
    }
    
    if (!finished) {
        queuedRequests.add(request);//queuedRequests存放客户端或者非leader节点转发的请求
        notifyAll();
    } 
}
```

###### 发送事务日志

org.apache.zookeeper.server.quorum.Leader#propose

```java
public Proposal propose(Request request) throws XidRolloverException {
    /**
     * Address the rollover issue. All lower 32bits set indicate a new leader
     * election. Force a re-election instead. See ZOOKEEPER-1277
     */
    if ((request.zxid & 0xffffffffL) == 0xffffffffL) {
        String msg =
                "zxid lower 32 bits have rolled over, forcing re-election, and therefore new epoch start";
        shutdown(msg);
        throw new XidRolloverException(msg);
    }
    byte[] data = SerializeUtils.serializeRequest(request);
    proposalStats.setLastProposalSize(data.length);
    QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data, null);
    
    Proposal p = new Proposal();
    p.packet = pp;
    p.request = request;
    synchronized (this) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Proposing:: " + request);
        }

        lastProposed = p.packet.getZxid();
        outstandingProposals.put(lastProposed, p); //维护zxId ——> Proposal
        sendPacket(pp);  //向Follower节点发送事务请求
    }
    return p;
}
```



```java
void sendPacket(QuorumPacket qp) {
    synchronized (forwardingFollowers) {
        for (LearnerHandler f : forwardingFollowers) {                
            f.queuePacket(qp);
        }
    }
}
```

###### 本机写入事务日志

org.apache.zookeeper.server.SyncRequestProcessor#processRequest

```java
public void processRequest(Request request) {
    queuedRequests.add(request);
}
```

#### CommitProcessor

包含两个列表committedRequests和queuedRequests

```
queuedRequests：存放尚未提交的事务请求和非事务请求（客户端请求或非leader转发的请求）
committedRequests：存放已经提交的事务请求（已提交说明事务请求已经同步到过半的Follower节点并且返回了ACK）
对于非事务请求可以直接交由后继Processor处理
对于事务请求必须等到此事务请求已提交，即queuedRequests中第一个请求与committedRequests中的第一个请求相匹配，才能交由后继ToBeAppliedRequestProcessor处理
此处理器保证了在处理一个事务请求时，不能处理其他任何请求
```

##### 初始化

org.apache.zookeeper.server.quorum.CommitProcessor#CommitProcessor

```java
public CommitProcessor(RequestProcessor nextProcessor, String id,
        boolean matchSyncs, ZooKeeperServerListener listener) {
    super("CommitProcessor:" + id, listener);
    this.nextProcessor = nextProcessor;
    this.matchSyncs = matchSyncs;
}
```

##### 处理请求

```java
public void run() {
    try {
        Request nextPending = null;  
        while (!finished) {
            int len = toProcess.size();
            for (int i = 0; i < len; i++) {
                nextProcessor.processRequest(toProcess.get(i));//交由后继处理器
            }
            toProcess.clear();
            synchronized (this) {
                if ((queuedRequests.size() == 0 || nextPending != null)
                        && committedRequests.size() == 0) {//事务请求尚未得到过半节点的ack，进行阻塞
                    wait();
                    continue;
                }
                // First check and see if the commi	t came in for the pending
                // request
                if ((queuedRequests.size() == 0 || nextPending != null)
                        && committedRequests.size() > 0) {
                  	//nextPending是事务请求，并且与committedRequests中的第一个请求相匹配，已经收到集群过半节点的ack，此请求可以交由后继处理器进行处理
                    Request r = committedRequests.remove();
                    if (nextPending != null
                            && nextPending.sessionId == r.sessionId
                            && nextPending.cxid == r.cxid) {
                        nextPending.hdr = r.hdr;
                        nextPending.txn = r.txn;
                        nextPending.zxid = r.zxid;
                      	//加入toProcess队列
                        toProcess.add(nextPending);
                        nextPending = null;
                    } else {
                        // 交由后继处理器处理
                        toProcess.add(r);
                    }
                }
            }

            // 转发给leader的事务类请求尚未返回响应，只要leader还未返回响应，就会阻塞后续请求的执行
            if (nextPending != null) {
                continue;
            }

            synchronized (this) {
                // Process the next requests in the queuedRequests
                while (nextPending == null && queuedRequests.size() > 0) {
                    Request request = queuedRequests.remove();
                  //事务类请求
                    switch (request.type) {
                    case OpCode.create:
                    case OpCode.delete:
                    case OpCode.setData:
                    case OpCode.multi:
                    case OpCode.setACL:
                    case OpCode.createSession:
                    case OpCode.closeSession:
                        nextPending = request; //发现事务类请求，退出循环
                        break;
                    case OpCode.sync:
                        if (matchSyncs) {
                            nextPending = request;
                        } else {
                            toProcess.add(request);
                        }
                        break;
                    default: //非事务请求
                        toProcess.add(request);
                    }
                }
            }
        }
    } catch (InterruptedException e) {
        LOG.warn("Interrupted exception while waiting", e);
    } catch (Throwable e) {
        LOG.error("Unexpected exception causing CommitProcessor to exit", e);
    }
    LOG.info("CommitProcessor exited loop!");
}
```

org.apache.zookeeper.server.quorum.CommitProcessor#commit

```java
synchronized public void commit(Request request) {
  	//此请求已经发送到过半节点并收到了ack响应
    if (!finished) {
        if (request == null) {
            LOG.warn("Committed a null!",
                     new Exception("committing a null! "));
            return;
        }
        if (LOG.isDebugEnabled()) {
            LOG.debug("Committing request:: " + request);
        }
        committedRequests.add(request);
        notifyAll();
    }
}
```

org.apache.zookeeper.server.quorum.CommitProcessor#processRequest

```java
synchronized public void processRequest(Request request) {
    // 来自ProposalRequestProcessor处理过的请求
    if (LOG.isDebugEnabled()) {
        LOG.debug("Processing request:: " + request);
    }
    
    if (!finished) {
        queuedRequests.add(request);
        notifyAll();
    }
}
```

#### SyncRequestProcessor

ProposalRequestProcessor在初始化时创建此处理器

负责将事务日志写入磁盘文件、异步做快照，将做快照的时机随机化

刷新磁盘后调用AckRequestProcessor的processRequest方法

```
queuedRequests：存放ProposalRequestProcessor传递的事务请求
toFlush：存放待将数据刷新到磁盘的事务请求，类似组提交
```

org.apache.zookeeper.server.SyncRequestProcessor#SyncRequestProcessor

```java
public SyncRequestProcessor(ZooKeeperServer zks,
        RequestProcessor nextProcessor) {
    super("SyncThread:" + zks.getServerId(), zks
            .getZooKeeperServerListener());
    this.zks = zks;
   //在ProposalRequestProcessor初始化时，将nextProcessor设置为AckRequestProcessor
    this.nextProcessor = nextProcessor;
    running = true;
}
```

org.apache.zookeeper.server.SyncRequestProcessor#run

```java
public void run() {
    try {
        int logCount = 0;
        //避免全部节点同时做快照
        setRandRoll(r.nextInt(snapCount/2));
        while (true) {
            Request si = null;

            if (toFlush.isEmpty()) {
                si = queuedRequests.take();//阻塞获取
            } else {
                si = queuedRequests.poll(); //无阻塞获取
                if (si == null) { //无请求入队
                    flush(toFlush); //刷新请求，刷盘、调用nextProcessor
                    continue;
                }
            }
            if (si == requestOfDeath) {
                break;
            }
            if (si != null) {
                //写事务日志
                if (zks.getZKDatabase().append(si)) {
                    logCount++;
                    if (logCount > (snapCount / 2 + randRoll)) {
                        setRandRoll(r.nextInt(snapCount/2));
                        // roll the log
                        zks.getZKDatabase().rollLog();
                        if (snapInProcess != null && snapInProcess.isAlive()) {
                            LOG.warn("Too busy to snap, skipping");
                        } else {
                            //异步做快照,同时可以处理客户端请求
                            snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                    public void run() {
                                        try {
                                            zks.takeSnapshot();
                                        } catch(Exception e) {
                                            LOG.warn("Unexpected exception", e);
                                        }
                                    }
                                };
                            snapInProcess.start();
                        }
                        logCount = 0;
                    }
                } else if (toFlush.isEmpty()) {
                    // optimization for read heavy workloads
                    // iff this is a read, and there are no pending
                    // flushes (writes), then just pass this to the next
                    // processor
                    if (nextProcessor != null) {
                        nextProcessor.processRequest(si);
                        if (nextProcessor instanceof Flushable) {
                            ((Flushable)nextProcessor).flush();
                        }
                    }
                    continue;
                }
                toFlush.add(si);
                if (toFlush.size() > 1000) {//累积1000此，做一次刷盘操作
                    flush(toFlush);
                }
            }
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
        running = false;
    }
    LOG.info("SyncRequestProcessor exited!");
}
```

写入事务日志后调用AckRequestProcessor的processRequest

#### AckRequestProcessor

AckRequestProcessor的processRequest方法在两个地方会被调用，一个地方就是领导者写入日志之后调用此方法，另一处就是领导者接收到从节点的ACK响应调用此方法

org.apache.zookeeper.server.quorum.AckRequestProcessor#processRequest

```java
public void processRequest(Request request) {
    QuorumPeer self = leader.self;
    if(self != null)
        leader.processAck(self.getId(), request.zxid, null);
    else
        LOG.error("Null QuorumPeer");
}
```

org.apache.zookeeper.server.quorum.Leader#processAck

```java
synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {
    if (LOG.isTraceEnabled()) {
        LOG.trace("Ack zxid: 0x{}", Long.toHexString(zxid));
        for (Proposal p : outstandingProposals.values()) {
            long packetZxid = p.packet.getZxid();
            LOG.trace("outstanding proposal: 0x{}",
                    Long.toHexString(packetZxid));
        }
        LOG.trace("outstanding proposals all");
    }

    if ((zxid & 0xffffffffL) == 0) {
        return;
    }
		//outstandingProposals存放尚未收到过半节点ack响应的事务请求
    if (outstandingProposals.size() == 0) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("outstanding is 0");	
        }
        return;
    }
  	//此事务请求已经被提交
    if (lastCommitted >= zxid) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("proposal has already been committed, pzxid: 0x{} zxid: 0x{}",
                    Long.toHexString(lastCommitted), Long.toHexString(zxid));
        }
        // The proposal has already been committed
        return;
    }
  	//outstandingProposals没有此事务请求，不合法(ProposalRequestProcessor的propose方法将Proposal放入outstandingProposals中)
    Proposal p = outstandingProposals.get(zxid);
    if (p == null) {
        LOG.warn("Trying to commit future proposal: zxid 0x{} from {}",
                Long.toHexString(zxid), followerAddr);
        return;
    }
    //sid可能是本节点的myId也可能是其他节点的myId
    p.ackSet.add(sid);
    if (LOG.isDebugEnabled()) {
        LOG.debug("Count for zxid: 0x{} is {}",
                Long.toHexString(zxid), p.ackSet.size());
    }
    //收到超过半数的ack
    if (self.getQuorumVerifier().containsQuorum(p.ackSet)){             
        if (zxid != lastCommitted+1) {
            LOG.warn("Commiting zxid 0x{} from {} not first!",
                    Long.toHexString(zxid), followerAddr);
            LOG.warn("First is 0x{}", Long.toHexString(lastCommitted + 1));
        }
        //移除已经提交的事务
        outstandingProposals.remove(zxid);
        if (p.request != null) {
            toBeApplied.add(p);
        }

        if (p.request == null) {
            LOG.warn("Going to commmit null request for proposal: {}", p);
        }
        //修改已经提交的事务Id，并发送至Follower节点
        commit(zxid);
        //给observers同步INFORM类型数据
        inform(p);
        //此事务请求已经被提交，交由后继处理器进行处理
        zk.commitProcessor.commit(p.request);
        if(pendingSyncs.containsKey(zxid)){
            for(LearnerSyncRequest r: pendingSyncs.remove(zxid)) {
                sendSync(r);
            }	
        }
    }
}
```

```java
public void commit(long zxid) {
    synchronized(this){
        lastCommitted = zxid;
    }
    QuorumPacket qp = new QuorumPacket(Leader.COMMIT, zxid, null, null);
    sendPacket(qp);
}
```

```java
public void inform(Proposal proposal) {   
    QuorumPacket qp = new QuorumPacket(Leader.INFORM, proposal.request.zxid, 
                                        proposal.packet.getData(), null);
    sendObserverPacket(qp);
}
```

#### ToBeAppliedRequestProcessor

org.apache.zookeeper.server.quorum.Leader.ToBeAppliedRequestProcessor#ToBeAppliedRequestProcessor

```java
ToBeAppliedRequestProcessor(RequestProcessor next,
        ConcurrentLinkedQueue<Proposal> toBeApplied) {
    if (!(next instanceof FinalRequestProcessor)) {
        throw new RuntimeException(ToBeAppliedRequestProcessor.class
                .getName()
                + " must be connected to "
                + FinalRequestProcessor.class.getName()
                + " not "
                + next.getClass().getName());
    }
  	//已经提交的事务请求(过半节点返回ack)
    this.toBeApplied = toBeApplied; //出入的为Leader的成员变量toBeApplied
    s//FinalRequestProcessor
    this.next = next;
}
```

```java
public void processRequest(Request request) throws RequestProcessorException {
    // request.addRQRec(">tobe");
    next.processRequest(request);
    Proposal p = toBeApplied.peek();
    if (p != null && p.request != null
            && p.request.zxid == request.zxid) {
        toBeApplied.remove();
    }
}
```

####  FinalRequestProcessor

主要负责将事务请求写入内存树中、发送响应给客户端

org.apache.zookeeper.server.FinalRequestProcessor#processRequest

```java
public void processRequest(Request request) {
    if (LOG.isDebugEnabled()) {
        LOG.debug("Processing request:: " + request);
    }
    // request.addRQRec(">final");
    long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
    if (request.type == OpCode.ping) {
        traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
    }
    if (LOG.isTraceEnabled()) {
        ZooTrace.logRequest(LOG, traceMask, 'E', request, "");
    }
    ProcessTxnResult rc = null;
    synchronized (zks.outstandingChanges) {
        while (!zks.outstandingChanges.isEmpty()
                && zks.outstandingChanges.get(0).zxid <= request.zxid) {
            ChangeRecord cr = zks.outstandingChanges.remove(0);
            if (cr.zxid < request.zxid) {
                LOG.warn("Zxid outstanding "
                        + cr.zxid
                        + " is less than current " + request.zxid);
            }
            if (zks.outstandingChangesForPath.get(cr.path) == cr) {
                zks.outstandingChangesForPath.remove(cr.path);
            }
        }
        if (request.hdr != null) { //事务请求
           TxnHeader hdr = request.hdr;
           Record txn = request.txn;

           rc = zks.processTxn(hdr, txn); //ZKDatabase处理事务请求，应用到内存树中
        }
        // do not add non quorum packets to the queue.
        if (Request.isQuorum(request.type)) {
            zks.getZKDatabase().addCommittedProposal(request);
        }
    }

    if (request.hdr != null && request.hdr.getType() == OpCode.closeSession) {
        ServerCnxnFactory scxn = zks.getServerCnxnFactory();
        // this might be possible since
        // we might just be playing diffs from the leader
        if (scxn != null && request.cnxn == null) {
            // calling this if we have the cnxn results in the client's
            // close session response being lost - we've already closed
            // the session/socket here before we can send the closeSession
            // in the switch block below
            scxn.closeSession(request.sessionId);
            return;
        }
    }

    if (request.cnxn == null) {
        return;
    }
    ServerCnxn cnxn = request.cnxn;

    String lastOp = "NA";
    zks.decInProcess();
    Code err = Code.OK;
    Record rsp = null;
    boolean closeSession = false;
    try {
        if (request.hdr != null && request.hdr.getType() == OpCode.error) {
            throw KeeperException.create(KeeperException.Code.get((
                    (ErrorTxn) request.txn).getErr()));
        }

        KeeperException ke = request.getException();
        if (ke != null && request.type != OpCode.multi) {
            throw ke;
        }

        if (LOG.isDebugEnabled()) {
            LOG.debug("{}",request);
        }
        switch (request.type) {
        case OpCode.ping: {
            zks.serverStats().updateLatency(request.createTime);

            lastOp = "PING";
            cnxn.updateStatsForResponse(request.cxid, request.zxid, lastOp,
                    request.createTime, Time.currentElapsedTime());

            cnxn.sendResponse(new ReplyHeader(-2,
                    zks.getZKDatabase().getDataTreeLastProcessedZxid(), 0), null, "response");
            return;
        }
        case OpCode.createSession: {
            zks.serverStats().updateLatency(request.createTime);

            lastOp = "SESS";
            cnxn.updateStatsForResponse(request.cxid, request.zxid, lastOp,
                    request.createTime, Time.currentElapsedTime());

            zks.finishSessionInit(request.cnxn, true);
            return;
        }
        case OpCode.multi: {
            lastOp = "MULT";
            rsp = new MultiResponse() ;

            for (ProcessTxnResult subTxnResult : rc.multiResult) {

                OpResult subResult ;

                switch (subTxnResult.type) {
                    case OpCode.check:
                        subResult = new CheckResult();
                        break;
                    case OpCode.create:
                        subResult = new CreateResult(subTxnResult.path);
                        break;
                    case OpCode.delete:
                        subResult = new DeleteResult();
                        break;
                    case OpCode.setData:
                        subResult = new SetDataResult(subTxnResult.stat);
                        break;
                    case OpCode.error:
                        subResult = new ErrorResult(subTxnResult.err) ;
                        break;
                    default:
                        throw new IOException("Invalid type of op");
                }

                ((MultiResponse)rsp).add(subResult);
            }

            break;
        }
        case OpCode.create: {
            lastOp = "CREA";
            rsp = new CreateResponse(rc.path);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.delete: {
            lastOp = "DELE";
            err = Code.get(rc.err);
            break;
        }
        case OpCode.setData: {
            lastOp = "SETD";
            rsp = new SetDataResponse(rc.stat);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.setACL: {
            lastOp = "SETA";
            rsp = new SetACLResponse(rc.stat);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.closeSession: {
            lastOp = "CLOS";
            closeSession = true;
            err = Code.get(rc.err);
            break;
        }
        case OpCode.sync: {
            lastOp = "SYNC";
            SyncRequest syncRequest = new SyncRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    syncRequest);
            rsp = new SyncResponse(syncRequest.getPath());
            break;
        }
        case OpCode.check: {
            lastOp = "CHEC";
            rsp = new SetDataResponse(rc.stat);
            err = Code.get(rc.err);
            break;
        }
        case OpCode.exists: {
            lastOp = "EXIS";
            // TODO we need to figure out the security requirement for this!
            ExistsRequest existsRequest = new ExistsRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    existsRequest);
            String path = existsRequest.getPath();
            if (path.indexOf('\0') != -1) {
                throw new KeeperException.BadArgumentsException();
            }
            Stat stat = zks.getZKDatabase().statNode(path, existsRequest
                    .getWatch() ? cnxn : null);
            rsp = new ExistsResponse(stat);
            break;
        }
        case OpCode.getData: {//查询数据
            lastOp = "GETD";
            GetDataRequest getDataRequest = new GetDataRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getDataRequest);
          //从内存树中查询DataNode
            DataNode n = zks.getZKDatabase().getNode(getDataRequest.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
          //权限认证
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ,
                    request.authInfo);
            Stat stat = new Stat();
          	//查询数据，
            byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat,
                    getDataRequest.getWatch() ? cnxn : null);
            rsp = new GetDataResponse(b, stat);
            break;
        }
        case OpCode.setWatches: {
            lastOp = "SETW";
            SetWatches setWatches = new SetWatches();
            // XXX We really should NOT need this!!!!
            request.request.rewind();
            ByteBufferInputStream.byteBuffer2Record(request.request, setWatches);
            long relativeZxid = setWatches.getRelativeZxid();
            zks.getZKDatabase().setWatches(relativeZxid, 
                    setWatches.getDataWatches(), 
                    setWatches.getExistWatches(),
                    setWatches.getChildWatches(), cnxn);
            break;
        }
        case OpCode.getACL: {
            lastOp = "GETA";
            GetACLRequest getACLRequest = new GetACLRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getACLRequest);
            DataNode n = zks.getZKDatabase().getNode(getACLRequest.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ | ZooDefs.Perms.ADMIN,
                    request.authInfo);

            Stat stat = new Stat();
            List<ACL> acl =
                    zks.getZKDatabase().getACL(getACLRequest.getPath(), stat);
            try {
                PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                        ZooDefs.Perms.ADMIN,
                        request.authInfo);
                rsp = new GetACLResponse(acl, stat);
            } catch (KeeperException.NoAuthException e) {
                List<ACL> acl1 = new ArrayList<ACL>(acl.size());
                for (ACL a : acl) {
                    if ("digest".equals(a.getId().getScheme())) {
                        Id id = a.getId();
                        Id id1 = new Id(id.getScheme(), id.getId().replaceAll(":.*", ":x"));
                        acl1.add(new ACL(a.getPerms(), id1));
                    } else {
                        acl1.add(a);
                    }
                }
                rsp = new GetACLResponse(acl1, stat);
            }
            break;
        }
        case OpCode.getChildren: {
            lastOp = "GETC";
            GetChildrenRequest getChildrenRequest = new GetChildrenRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getChildrenRequest);
            DataNode n = zks.getZKDatabase().getNode(getChildrenRequest.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ,
                    request.authInfo);
            List<String> children = zks.getZKDatabase().getChildren(
                    getChildrenRequest.getPath(), null, getChildrenRequest
                            .getWatch() ? cnxn : null);
            rsp = new GetChildrenResponse(children);
            break;
        }
        case OpCode.getChildren2: {
            lastOp = "GETC";
            GetChildren2Request getChildren2Request = new GetChildren2Request();
            ByteBufferInputStream.byteBuffer2Record(request.request,
                    getChildren2Request);
            Stat stat = new Stat();
            DataNode n = zks.getZKDatabase().getNode(getChildren2Request.getPath());
            if (n == null) {
                throw new KeeperException.NoNodeException();
            }
            PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                    ZooDefs.Perms.READ,
                    request.authInfo);
            List<String> children = zks.getZKDatabase().getChildren(
                    getChildren2Request.getPath(), stat, getChildren2Request
                            .getWatch() ? cnxn : null);
            rsp = new GetChildren2Response(children, stat);
            break;
        }
        }
    } catch (SessionMovedException e) {
        // session moved is a connection level error, we need to tear
        // down the connection otw ZOOKEEPER-710 might happen
        // ie client on slow follower starts to renew session, fails
        // before this completes, then tries the fast follower (leader)
        // and is successful, however the initial renew is then 
        // successfully fwd/processed by the leader and as a result
        // the client and leader disagree on where the client is most
        // recently attached (and therefore invalid SESSION MOVED generated)
        cnxn.sendCloseSession();
        return;
    } catch (KeeperException e) {
        err = e.code();
    } catch (Exception e) {
        // log at error level as we are returning a marshalling
        // error to the user
        LOG.error("Failed to process " + request, e);
        StringBuilder sb = new StringBuilder();
        ByteBuffer bb = request.request;
        bb.rewind();
        while (bb.hasRemaining()) {
            sb.append(Integer.toHexString(bb.get() & 0xff));
        }
        LOG.error("Dumping request buffer: 0x" + sb.toString());
        err = Code.MARSHALLINGERROR;
    }
		//最新的事务Id
    long lastZxid = zks.getZKDatabase().getDataTreeLastProcessedZxid();
  	//构建响应头
    ReplyHeader hdr =
        new ReplyHeader(request.cxid, lastZxid, err.intValue());

    zks.serverStats().updateLatency(request.createTime);
    cnxn.updateStatsForResponse(request.cxid, lastZxid, lastOp,
                request.createTime, Time.currentElapsedTime());

    try {
      //返回响应
        cnxn.sendResponse(hdr, rsp, "response");
        if (closeSession) {
            cnxn.sendCloseSession();
        }
    } catch (IOException e) {
        LOG.error("FIXMSG",e);
    }
}
```

### 会话管理

#### 客户端

为了维持客户端到ZooKeeper节点的session，如果在一段时间内客户端不需要向服务器端发送请求，客户端需要向服务器发送心跳消息PING。

org.apache.zookeeper.ClientCnxn.SendThread#sendPing

```java
private void sendPing() {
    lastPingSentNs = System.nanoTime();
    RequestHeader h = new RequestHeader(-2, OpCode.ping);
    queuePacket(h, null, null, null, null, null, null, null, null);
}
```

#### 服务端

##### 创建会话

org.apache.zookeeper.server.SessionTrackerImpl#addSession

```java
synchronized public void addSession(long id, int sessionTimeout) {
    sessionsWithTimeout.put(id, sessionTimeout);
    if (sessionsById.get(id) == null) {
      	//每个会话对应一个SessionImpl
        SessionImpl s = new SessionImpl(id, sessionTimeout, 0);
        sessionsById.put(id, s);
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                    "SessionTrackerImpl --- Adding session 0x"
                    + Long.toHexString(id) + " " + sessionTimeout);
        }
    } else {
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                    "SessionTrackerImpl --- Existing session 0x"
                    + Long.toHexString(id) + " " + sessionTimeout);
        }
    }
    touchSession(id, sessionTimeout);
}
```

##### 续约会话

```java
synchronized public boolean touchSession(long sessionId, int timeout) {
  	//首次创建会话或者客户端发送Ping请求时会调用此方法	
    if (LOG.isTraceEnabled()) {
        ZooTrace.logTraceMessage(LOG,
                                 ZooTrace.CLIENT_PING_TRACE_MASK,
                                 "SessionTrackerImpl --- Touch session: 0x"
                + Long.toHexString(sessionId) + " with timeout " + timeout);
    }
  	//获取SessionImpl
    SessionImpl s = sessionsById.get(sessionId);
    // Return false, if the session doesn't exists or marked as closing
    if (s == null || s.isClosing()) {
        return false;
    }
    //计算超时时间，对超时时间进行规整，为expirationInterval的整数倍，便于分桶管理会话
    long expireTime = roundToInterval(Time.currentElapsedTime() + timeout);
    if (s.tickTime >= expireTime) { //首次创建时，tickTime为0
        // Nothing needs to be done 
        return true;
    }
  	//维护了某个时间点超时的会话集合
    // HashMap<Long, SessionSet> sessionSets = new HashMap<Long, SessionSet>();
    SessionSet set = sessionSets.get(s.tickTime);
    if (set != null) {
        set.sessions.remove(s);
    }
  	//设置规整的超时时间
    s.tickTime = expireTime;
    set = sessionSets.get(s.tickTime);
  	//对会话进行维护
    if (set == null) {
        set = new SessionSet();
        sessionSets.put(expireTime, set);
    }
    set.sessions.add(s);
    return true;
}
```

##### 会话移除

org.apache.zookeeper.server.ZooKeeperServer#startSessionTracker

```java
protected void startSessionTracker() {
    ((SessionTrackerImpl)sessionTracker).start();
}
```



```java
synchronized public void run() {
    try {
        while (running) {
          	//每隔nextExpirationTime执行一次
            currentTime = Time.currentElapsedTime();
            if (nextExpirationTime > currentTime) {
                this.wait(nextExpirationTime - currentTime);//阻塞
                continue;
            }
            SessionSet set;
          	//移除过期的会话集合	
            set = sessionSets.remove(nextExpirationTime);
            if (set != null) {
                for (SessionImpl s : set.sessions) {
                    setSessionClosing(s.sessionId);
                    expirer.expire(s);
                }
            }
            nextExpirationTime += expirationInterval;
        }
    } catch (InterruptedException e) {
        handleException(this.getName(), e);
    }
    LOG.info("SessionTrackerImpl exited loop!");
}
```



```java
synchronized public void setSessionClosing(long sessionId) {
    if (LOG.isTraceEnabled()) {
        LOG.info("Session closing: 0x" + Long.toHexString(sessionId));
    }
    SessionImpl s = sessionsById.get(sessionId);
    if (s == null) {
        return;
    }
    s.isClosing = true; //会话关闭	
}
```

org.apache.zookeeper.server.ZooKeeperServer#expire

```java
public void expire(Session session) {
    long sessionId = session.getSessionId();
    LOG.info("Expiring session 0x" + Long.toHexString(sessionId)
            + ", timeout of " + session.getTimeout() + "ms exceeded");
    close(sessionId);
}
```



```java
private void close(long sessionId) {
  	//提交会话关闭的请求
    submitRequest(null, sessionId, OpCode.closeSession, 0, null, null);
}
```

### 快照

org.apache.zookeeper.server.SyncRequestProcessor#run

```java
zks.getZKDatabase().rollLog(); //生成新的日志文件
if (snapInProcess != null && snapInProcess.isAlive()) { //是否正在做快照
    LOG.warn("Too busy to snap, skipping");
} else {
    //异步做快照，同时可以处理客户端请求
    snapInProcess = new ZooKeeperThread("Snapshot Thread") {
            public void run() {
                try {
                    zks.takeSnapshot();
                } catch(Exception e) {
                    LOG.warn("Unexpected exception", e);
                }
            }
        };
    snapInProcess.start();
}
```

org.apache.zookeeper.server.ZooKeeperServer#takeSnapshot

```java
public void takeSnapshot(){
		//内存树
    //会话集合（sessionId->timeout）
    try {
        txnLogFactory.save(zkDb.getDataTree(), zkDb.getSessionWithTimeOuts());
    } catch (IOException e) {
        LOG.error("Severe unrecoverable error, exiting", e);
        // This is a severe error that we cannot recover from,
        // so we need to exit
        System.exit(10);
    }
}
```

org.apache.zookeeper.server.persistence.FileTxnSnapLog#save

```java
public void save(DataTree dataTree,
        ConcurrentHashMap<Long, Integer> sessionsWithTimeouts)
    throws IOException {
  	//应用到内存树的最新zxId
    long lastZxid = dataTree.lastProcessedZxid;
  	//创建快照文件
    File snapshotFile = new File(snapDir, Util.makeSnapshotName(lastZxid));
    LOG.info("Snapshotting: 0x{} to {}", Long.toHexString(lastZxid),
            snapshotFile);
    snapLog.serialize(dataTree, sessionsWithTimeouts, snapshotFile);
    
}
```

org.apache.zookeeper.server.persistence.FileSnap#serialize(org.apache.zookeeper.server.DataTree, java.util.Map<java.lang.Long,java.lang.Integer>, java.io.File)

```java
public synchronized void serialize(DataTree dt, Map<Long, Integer> sessions, File snapShot)  throws IOException {
  	//序列化dataTree和sessions到快照文件
    if (!close) {
        OutputStream sessOS = new BufferedOutputStream(new FileOutputStream(snapShot));
        CheckedOutputStream crcOut = new CheckedOutputStream(sessOS, new Adler32());
        //CheckedOutputStream cout = new CheckedOutputStream()
        OutputArchive oa = BinaryOutputArchive.getArchive(crcOut);
      //创建文件头
        FileHeader header = new FileHeader(SNAP_MAGIC, VERSION, dbId);
       //序列化
       	serialize(dt,sessions,oa, header);
        long val = crcOut.getChecksum().getValue();
        oa.writeLong(val, "val");
        oa.writeString("/", "path");
        sessOS.flush();
        crcOut.close();
        sessOS.close();
    }
}
```



```java
protected void serialize(DataTree dt,Map<Long, Integer> sessions,
        OutputArchive oa, FileHeader header) throws IOException {
    // this is really a programmatic error and not something that can
    // happen at runtime
    if(header==null)
        throw new IllegalStateException(
                "Snapshot's not open for writing: uninitialized header");
    header.serialize(oa, "fileheader");
    SerializeUtils.serializeSnapshot(dt,oa,sessions);
}
```

org.apache.zookeeper.server.util.SerializeUtils#serializeSnapshot

```java
public static void serializeSnapshot(DataTree dt,OutputArchive oa,
        Map<Long, Integer> sessions) throws IOException {
    HashMap<Long, Integer> sessSnap = new HashMap<Long, Integer>(sessions);
  	//写入会话个数
    oa.writeInt(sessSnap.size(), "count");
    for (Entry<Long, Integer> entry : sessSnap.entrySet()) {
      	//会话标志
        oa.writeLong(entry.getKey().longValue(), "id");
      	//未修整的超时时间
        oa.writeInt(entry.getValue().intValue(), "timeout");
    }
    //序列化内存树
    dt.serialize(oa, "tree");
}
```

org.apache.zookeeper.server.DataTree#serialize

```java
public void serialize(OutputArchive oa, String tag) throws IOException {
    scount = 0;
    aclCache.serialize(oa);
    serializeNode(oa, new StringBuilder(""));
    // / marks end of stream
    // we need to check if clear had been called in between the snapshot.
    if (root != null) {
        oa.writeString("/", "path");
    }
}
```

```java
void serializeNode(OutputArchive oa, StringBuilder path) throws IOException {
    String pathString = path.toString();
    DataNode node = getNode(pathString);
    if (node == null) {
        return;
    }
    String children[] = null;
    DataNode nodeCopy;
    synchronized (node) {
        scount++;
        StatPersisted statCopy = new StatPersisted();
        copyStatPersisted(node.stat, statCopy);
        //we do not need to make a copy of node.data because the contents
        //are never changed
        nodeCopy = new DataNode(node.parent, node.data, node.acl, statCopy);
        Set<String> childs = node.getChildren();
        children = childs.toArray(new String[childs.size()]);
    }
    oa.writeString(pathString, "path");
    oa.writeRecord(nodeCopy, "node");
    path.append('/');
    int off = path.length();
    for (String child : children) {
        // since this is single buffer being resused
        // we need
        // to truncate the previous bytes of string.
        path.delete(off, Integer.MAX_VALUE);
        path.append(child);
        serializeNode(oa, path);
    }
}
```

## 幽冥复现

何为幽冥复现：

在一些极端异常比如网络隔离，机器故障等情况下，Leader可能会经过多次切换和数据恢复，比如集群中有A、B、C三个节点：

Round1：A为Leader，A有部分日志尚未同步至到B、C节点就宕机了。

Round2：B当选Leader，在B当选Leader期间没有接收事务请求。

Round3：B宕机之后A重启，由于A节点上的事务zxid大于C节点，所以A当选Leader，将数据同步给C，就会发生幽灵复现。(由于Round2期间没有接收数据，A节点将Round1期间的数据同步到了C节点)

改进：

Zookeeper在每次选举出Leader之后，都会将EpochId（选举的轮次）存储到文件中。

在选举时，从文件读取EpochId并加入到选票信息中，发送给其他候选人，候选人如果发现epochId比自己的小就会忽略此投票，否则就会选择此投票，然后接着比较zxId、myId。

因此，对于此问题，Round 1中，比如A,B,C的CurrentEpoch为2；Round 2，A的CurrentEpoch为2，B,C的CurrentEpoch为3；Round 3，由于B,C的CurrentEpoch比A的大，所以A无法成为leader

### 选举之初

org.apache.zookeeper.server.quorum.QuorumPeer#startLeaderElection

```java
synchronized public void startLeaderElection() {
   try {
      currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
   } catch(IOException e) {
      RuntimeException re = new RuntimeException(e.getMessage());
      re.setStackTrace(e.getStackTrace());
      throw re;
   }
    for (QuorumServer p : getView().values()) {
        if (p.id == myid) {
            myQuorumAddr = p.addr;
            break;
        }
    }
    if (myQuorumAddr == null) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    if (electionType == 0) {
        try {
            udpSocket = new DatagramSocket(myQuorumAddr.getPort());
            responder = new ResponderThread();
            responder.start();
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
    }
    this.electionAlg = createElectionAlgorithm(electionType);
}
```

```java
public long getLastLoggedZxid() {
    if (!zkDb.isInitialized()) {//尚未初始化
       loadDataBase();
    }
   //获取最新日志的zxId
    return zkDb.getDataTreeLastProcessedZxid();
}
```

org.apache.zookeeper.server.quorum.QuorumPeer#getCurrentEpoch

```java
  public long getCurrentEpoch() throws IOException {//从磁盘读取currentEpoch文件
   if (currentEpoch == -1) {
      currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
   }
   return currentEpoch;
}
```

org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader

```java
synchronized(this){
    //选举前，增加逻辑时钟(系统每次启动都会从0开始)
    logicalclock.incrementAndGet();
    //读取currentEpoch，更新本本节点的投票信息
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}
```

### Leader保存选举轮次

org.apache.zookeeper.server.quorum.Leader#lead

```java
//当选leader之后调用lead方法
//阻塞直到过半的Participant类型的节点返回ACKEPOCH
waitForEpochAck(self.getId(), leaderStateSummary);
//保存当前的选举轮次到磁盘文件
self.setCurrentEpoch(epoch);
```

org.apache.zookeeper.server.quorum.QuorumPeer#setCurrentEpoch

```java
public void setCurrentEpoch(long e) throws IOException {
   currentEpoch = e;
   writeLongToFile(CURRENT_EPOCH_FILENAME, e);
}
```

### Follower保存选举轮次

org.apache.zookeeper.server.quorum.Learner#syncWithLeader

```java
case Leader.NEWLEADER:  //Leader发送NEWLEADER指令，说明过半的节点完成与Leade的数据同步
    File updating = new File(self.getTxnFactory().getSnapDir(),
                        QuorumPeer.UPDATING_EPOCH_FILENAME);
    if (!updating.exists() && !updating.createNewFile()) {
        throw new IOException("Failed to create " +
                              updating.toString());
    }
    if (snapshotNeeded) {
        zk.takeSnapshot();
    }
		//保存选举轮次到磁盘文件
    self.setCurrentEpoch(newEpoch);
    if (!updating.delete()) {
        throw new IOException("Failed to delete " +
                              updating.toString());
    }
    writeToTxnLog = true;
    isPreZAB1_0 = false;
    //发送ACK到Leader节点，表明完成数据同步
    writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
    break;
}
```

# 羊群效应

在分布式锁场景中，注册临时顺序节点，后缀数字小的锁请求者先获取锁。如果所有的锁请求者都 watch 锁持有者，当代表锁请求者的 znode 被删除以后，所有的锁请求者都会通知到，但是只有一个锁请 求者能拿到锁。这就是羊群效应。

为了避免羊群效应，每个锁请求者 watch 它前面的锁请求者。每次锁被释放，只会有一个锁请求者会被通知到。这样做还让锁的分配具有公平性，锁的分配遵循先到先得的原则。

# 磁盘管理

**使用单独磁盘目录作为事务日志的输出目录。事务日志的写入是顺序写入文件的过程，应该避免与其他应用程序产生对磁盘的竞争。**

**将事务日志和快照数据分别挂载到独立的磁盘**

**尽量避免内存和磁盘空间之间的交换**

# 磁盘清理

清理历史快照数据和事务日志文件

org.apache.zookeeper.server.quorum.QuorumPeerMain#initializeAndRun

```java
DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
        .getDataDir(), config.getDataLogDir(), config
        .getSnapRetainCount(), config.getPurgeInterval());
purgeMgr.start();
```

org.apache.zookeeper.server.DatadirCleanupManager#start

```java
public void start() {
    if (PurgeTaskStatus.STARTED == purgeTaskStatus) {//清理任务正在运行
        LOG.warn("Purge task is already running.");
        return;
    }
    // Don't schedule the purge task with zero or negative purge interval.
    if (purgeInterval <= 0) {
        LOG.info("Purge task is not scheduled.");
        return;
    }

    timer = new Timer("PurgeTask", true);
  	//默认保留3个快照数据文件
    TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);
    timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));

    purgeTaskStatus = PurgeTaskStatus.STARTED;
}
```

org.apache.zookeeper.server.DatadirCleanupManager.PurgeTask#run

```java
public void run() {
    LOG.info("Purge task started.");
    try {
        PurgeTxnLog.purge(new File(logsDir), new File(snapsDir), snapRetainCount);
    } catch (Exception e) {
        LOG.error("Error occurred while purging.", e);
    }
    LOG.info("Purge task completed.");
}
```

org.apache.zookeeper.server.PurgeTxnLog#purge

```java
public static void purge(File dataDir, File snapDir, int num) throws IOException {
    if (num < 3) { //默认保留3个快照文件
        throw new IllegalArgumentException(COUNT_ERR_MSG);
    }

    FileTxnSnapLog txnLog = new FileTxnSnapLog(dataDir, snapDir);
		//查找最新的3个快照文件
    List<File> snaps = txnLog.findNRecentSnapshots(num);
    int numSnaps = snaps.size();
    if (numSnaps > 0) {
        purgeOlderSnapshots(txnLog, snaps.get(numSnaps - 1));
    }
}
```



```java
static void purgeOlderSnapshots(FileTxnSnapLog txnLog, File snapShot) {
    final long leastZxidToBeRetain = Util.getZxidFromName(
            snapShot.getName(), PREFIX_SNAPSHOT);
	
    final Set<File> retainedTxnLogs = new HashSet<File>();
  
  	retainedTxnLogs.addAll(
        Arrays.asList(txnLog.getSnapshotLogs(leastZxidToBeRetain)));
  
    class MyFileFilter implements FileFilter{
        private final String prefix;
        MyFileFilter(String prefix){
            this.prefix=prefix;
        }
        public boolean accept(File f){
            if(!f.getName().startsWith(prefix + "."))
                return false;
            if (retainedTxnLogs.contains(f)) {
                return false;
            }
            long fZxid = Util.getZxidFromName(f.getName(), prefix);
            if (fZxid >= leastZxidToBeRetain) {
                return false;
            }
            return true;
        }
    }
    // 查找要删除的事务日志文件
    List<File> files = new ArrayList<File>();
    File[] fileArray = txnLog.getDataDir().listFiles(new MyFileFilter(PREFIX_LOG));
    if (fileArray != null) {
        files.addAll(Arrays.asList(fileArray));
    }
		//查找要删除的快照文件
    fileArray = txnLog.getSnapDir().listFiles(new MyFileFilter(PREFIX_SNAPSHOT));
    if (fileArray != null) {
        files.addAll(Arrays.asList(fileArray));
    }

    // 删除文件
    for(File f: files)
    {
        final String msg = "Removing file: "+
            DateFormat.getDateTimeInstance().format(f.lastModified())+
            "\t"+f.getPath();
        LOG.info(msg);
        System.out.println(msg);
        if(!f.delete()){
            System.err.println("Failed to remove "+f.getPath());
        }
    }

}
```

# Curator

## 入门实例

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
// 使用Factory方法
CuratorFramework client = CuratorFrameworkFactory.newClient(connectString, retryPolicy);
// Fluent风格
CuratorFramework client = CuratorFrameworkFactory.buidler()
  																						.connectString(connectString) 																															.retryPolicy(retryPolicy)
  																						 .build()
client.start();
// 同步版本
client.create().withMode(CreateMode.PERSISTENT).forPath(path, data);
```

## 启动

org.apache.curator.framework.CuratorFrameworkFactory#newClient(java.lang.String, org.apache.curator.RetryPolicy)

```java
public static CuratorFramework newClient(String connectString, RetryPolicy retryPolicy) {
    return newClient(connectString, DEFAULT_SESSION_TIMEOUT_MS, DEFAULT_CONNECTION_TIMEOUT_MS, retryPolicy);
}
```

```java
public static CuratorFramework newClient(String connectString, int sessionTimeoutMs, int connectionTimeoutMs, RetryPolicy retryPolicy) { //参数众多，使用Builder模式创建对象
    return builder().connectString(connectString).sessionTimeoutMs(sessionTimeoutMs).connectionTimeoutMs(connectionTimeoutMs).retryPolicy(retryPolicy).build();
}
```

```java
public CuratorFramework build() {
    return new CuratorFrameworkImpl(this);
}
```

org.apache.curator.framework.imps.CuratorFrameworkImpl#start

```java
public void start() {
    this.log.info("Starting");
    if (!this.state.compareAndSet(CuratorFrameworkState.LATENT, CuratorFrameworkState.STARTED)) {
        throw new IllegalStateException("Cannot be started more than once");
    } else {
        try {
            this.connectionStateManager.start();
            ConnectionStateListener listener = new ConnectionStateListener() {
                public void stateChanged(CuratorFramework client, ConnectionState newState) {
                    if (ConnectionState.CONNECTED == newState || ConnectionState.RECONNECTED == newState) {
                        CuratorFrameworkImpl.this.logAsErrorConnectionErrors.set(true);
                    }

                }
            };
            this.getConnectionStateListenable().addListener(listener);
            this.client.start();
            this.executorService = Executors.newSingleThreadScheduledExecutor(this.threadFactory);
            this.executorService.submit(new Callable<Object>() {
                public Object call() throws Exception {
                    CuratorFrameworkImpl.this.backgroundOperationsLoop();
                    return null;
                }
            });
        } catch (Exception var2) {
            ThreadUtils.checkInterrupted(var2);
            this.handleBackgroundOperationException((OperationAndData)null, var2);
        }

    }
}
```

org.apache.curator.CuratorZookeeperClient#start

```java
public void start() throws Exception {
    this.log.debug("Starting");
    if (!this.started.compareAndSet(false, true)) {
        IllegalStateException ise = new IllegalStateException("Already started");
        throw ise;
    } else {
        this.state.start();
    }
}
```

```java
void start() throws Exception {
    log.debug("Starting");
    this.ensembleProvider.start();
    this.reset();
}
```

```java
private synchronized void reset() throws Exception {
    log.debug("reset");
    this.instanceIndex.incrementAndGet();
    this.isConnected.set(false);
    this.connectionStartMs = System.currentTimeMillis();
    this.zooKeeper.closeAndReset();
    this.zooKeeper.getZooKeeper();
}
```

```java
void closeAndReset() throws Exception {
    this.internalClose();
    this.helper = new HandleHolder.Helper() {
        private volatile ZooKeeper zooKeeperHandle = null;
        private volatile String connectionString = null;

        public ZooKeeper getZooKeeper() throws Exception {
            synchronized(this) {
                if (this.zooKeeperHandle == null) {
                    this.connectionString = HandleHolder.this.ensembleProvider.getConnectionString();
                    this.zooKeeperHandle = HandleHolder.this.zookeeperFactory.newZooKeeper(this.connectionString, HandleHolder.this.sessionTimeout, HandleHolder.this.watcher, HandleHolder.this.canBeReadOnly);
                }

                HandleHolder.this.helper = new HandleHolder.Helper() {
                    public ZooKeeper getZooKeeper() throws Exception {
                        return zooKeeperHandle;
                    }

                    public String getConnectionString() {
                        return connectionString;
                    }
                };
                return this.zooKeeperHandle;
            }
        }

        public String getConnectionString() {
            return this.connectionString;
        }
    };
}
```

```java
ZooKeeper getZooKeeper() throws Exception {
    return this.helper != null ? this.helper.getZooKeeper() : null;
}
```

## 创建节点

org.apache.curator.framework.imps.CuratorFrameworkImpl#create

```java
public CreateBuilder create() {
    Preconditions.checkState(this.getState() == CuratorFrameworkState.STARTED, "instance must be started before calling this method");
    return new CreateBuilderImpl(this);
}
```

```java
public ACLBackgroundPathAndBytesable<String> withMode(CreateMode mode) {
    this.createMode = mode; //顺序、临时
    return this; 
}
```

```java
public String forPath(String path) throws Exception {
    return this.forPath(path, this.client.getDefaultData());
}
```

```java
public String forPath(String givenPath, byte[] data) throws Exception {
    if (this.compress) { //对数据进行压缩
        data = this.client.getCompressionProvider().compress(givenPath, data);
    }

    String adjustedPath = this.adjustPath(this.client.fixForNamespace(givenPath, this.createMode.isSequential()));
    String returnPath = null;
    if (this.backgrounding.inBackground()) { //异步调用
        this.pathInBackground(adjustedPath, data, givenPath);
    } else { //同步调用
        String path = this.protectedPathInForeground(adjustedPath, data);
        returnPath = this.client.unfixForNamespace(path);
    }

    return returnPath;
}
```

### 同步调用

```java
private String protectedPathInForeground(String adjustedPath, byte[] data) throws Exception {
    try {
        return this.pathInForeground(adjustedPath, data);
    } catch (Exception var4) {
        ThreadUtils.checkInterrupted(var4);
        if ((var4 instanceof ConnectionLossException || !(var4 instanceof KeeperException)) && this.protectedId != null) {
            (new FindAndDeleteProtectedNodeInBackground(this.client, ZKPaths.getPathAndNode(adjustedPath).getPath(), this.protectedId)).execute();
            this.protectedId = UUID.randomUUID().toString();
        }

        throw var4;
    }
}
```

```java
private String pathInForeground(final String path, final byte[] data) throws Exception {
    TimeTrace trace = this.client.getZookeeperClient().startTracer("CreateBuilderImpl-Foreground");
    final AtomicBoolean firstTime = new AtomicBoolean(true);
    String returnPath = (String)RetryLoop.callWithRetry(this.client.getZookeeperClient(), new Callable<String>() {
        public String call() throws Exception {
            boolean localFirstTime = firstTime.getAndSet(false) && !CreateBuilderImpl.this.debugForceFindProtectedNode;
            String createdPath = null;
            if (!localFirstTime && CreateBuilderImpl.this.doProtected) {
                CreateBuilderImpl.this.debugForceFindProtectedNode = false;
                createdPath = CreateBuilderImpl.this.findProtectedNodeInForeground(path);
            }

            if (createdPath == null) {
                try {
                  //同步创建
                    createdPath = CreateBuilderImpl.this.client.getZooKeeper().create(path, data, CreateBuilderImpl.this.acling.getAclList(path), CreateBuilderImpl.this.createMode);
                } catch (NoNodeException var4) {
                    if (!CreateBuilderImpl.this.createParentsIfNeeded) {
                        throw var4;
                    }

                    ZKPaths.mkdirs(CreateBuilderImpl.this.client.getZooKeeper(), path, false, CreateBuilderImpl.this.client.getAclProvider(), CreateBuilderImpl.this.createParentsAsContainers);
                    createdPath = CreateBuilderImpl.this.client.getZooKeeper().create(path, data, CreateBuilderImpl.this.acling.getAclList(path), CreateBuilderImpl.this.createMode);
                }
            }

            if (CreateBuilderImpl.this.failNextCreateForTesting) {
                CreateBuilderImpl.this.failNextCreateForTesting = false;
                throw new ConnectionLossException();
            } else {
                return createdPath;
            }
        }
    });
    trace.commit();
    return returnPath;
}
```

```java
public static <T> T callWithRetry(CuratorZookeeperClient client, Callable<T> proc) throws Exception {
    T result = null;
    RetryLoop retryLoop = client.newRetryLoop();

    while(retryLoop.shouldContinue()) { //重试
        try {
            client.internalBlockUntilConnectedOrTimedOut();
            result = proc.call();
            retryLoop.markComplete();
        } catch (Exception var5) {
            ThreadUtils.checkInterrupted(var5);
            retryLoop.takeException(var5);
        }
    }

    return result;
}
```

# 总结

1、请求隔离：服务端将客户端请求、同步请求、选举请求分别使用不同的端口的处理，避免了不同类型请求之间的相互影响

2、线程隔离：不同的请求处理器使用独立的线程处理队列中请求

3、分桶策略：服务端对会话的超时进行规整，使用分桶策略管理大量的会话

4、快照的时机随机化：避免集群中所有的节点同时做快照

5、批量组提交：避免每次写入事务日志都直接写入磁盘，减少操作系统刷盘次数

6、磁盘空间预分配：事务日志文件的磁盘空间进行预分配，文件的追加写入会触发底层磁盘IO为文件开辟新的磁盘块，可以让文件尽可能的占用连续的磁盘扇区，减少后续写入和读取文件时的磁盘寻道开销，

7、快照可以避免日志文件过多，启动时间过长的问题，快照还可以减少存储空间的占用，只保留最新的数据改动

8、客户端将集群的地址进行打散，避免都连接到集群中的同一节点，导致其负载升高

9、使用单独的磁盘作为事务日志的输出目录，不要和其他应用程序共享一块磁盘

10、容灾：三机房部署和双机房部署。以7台机器为例，三机房部署有两种部署方案，分别为（3，1，3）和（3，2，2），这两种方案都可以实现在某一机房出现问题的情况，仍然可以满足’过半‘原则。双机房部署只有一种方案，（4，3），但是4台机器所在的机房出现问题，就不满足’过半‘原则。

# Reference

https://github.com/apache/zookeeper

https://github.com/alibaba/taokeeper
