# Zookeeper源码分析

  * [客户端](#客户端)
    * [ZooKeeper](#zookeeper)
    * [ClientCnxnSocket](#clientcnxnsocket)
    * [SendThread](#sendthread)
      * [1.与服务端建立连接](#1与服务端建立连接)
      * [2.发送连接请求、认证请求](#2发送连接请求认证请求)
      * [3.读写IO处理](#3读写io处理)
      * [4.处理响应](#4处理响应)
      * [5.结束请求](#5结束请求)
    * [EventThread](#eventthread)
      * [1.生产Watch事件](#1生产watch事件)
      * [2.生产响应事件](#2生产响应事件)
      * [3.处理事件](#3处理事件)
  * [Standalone模式](#standalone模式)
    * [1.启动](#1启动)
    * [2.初始化](#2初始化)
    * [3.接收请求](#3接收请求)
    * [4.请求处理](#4请求处理)
      * [4.1处理连接请求](#41处理连接请求)
        * [4.1.1 创建会话](#411-创建会话)
      * [4.2 处理非连接请求](#42-处理非连接请求)
    * [5、Processor处理请求](#5processor处理请求)
  * [Quorum模式](#quorum模式)
    * [Observer](#observer)
      * [创建Observer](#创建observer)
      * [寻找Leader](#寻找leader)
      * [连接Leader](#连接leader)
      * [数据同步初始化](#数据同步初始化)
      * [处理PING/INFORM](#处理pinginform)
    * [Follower](#follower)
      * [创建Follower](#创建follower)
        * [处理Proposal指令](#处理proposal指令)
        * [处理Commit指令](#处理commit指令)
    * [Leader](#leader)
      * [创建Leader](#创建leader)
      * [LearnerCnxAcceptor](#learnercnxacceptor)
      * [LearnerHandler](#learnerhandler)


## 客户端

### ZooKeeper

org.apache.zookeeper.ZooKeeper#ZooKeeper(java.lang.String, int, org.apache.zookeeper.Watcher, boolean)

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
        boolean canBeReadOnly)
    throws IOException
{
    LOG.info("Initiating client connection, connectString=" + connectString
            + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

    watchManager.defaultWatcher = watcher;

    ConnectStringParser connectStringParser = new ConnectStringParser(
            connectString);
    HostProvider hostProvider = new StaticHostProvider(
            connectStringParser.getServerAddresses());
    cnxn = createConnection(connectStringParser.getChrootPath(),
            hostProvider, sessionTimeout, this, watchManager,
            getClientCnxnSocket(), canBeReadOnly);
    cnxn.start();
}
```

### ClientCnxnSocket

负责网络通信

```java
private static ClientCnxnSocket getClientCnxnSocket() throws IOException {
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

### SendThread

#### 1.与服务端建立连接

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
            // An authentication error occurred when the SASL client tried to initialize:
            // for Kerberos this means that the client failed to authenticate with the KDC.
            // This is different from an authentication error that occurs during communication
            // with the Zookeeper server, which is handled below.
            LOG.warn("SASL configuration failed: " + e + " Will continue connection to Zookeeper server without "
              + "SASL authentication, if Zookeeper server allows it.");
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

```java
void connect(InetSocketAddress addr) throws IOException {
    SocketChannel sock = createSock();
    try {
       registerAndConnect(sock, addr);
    } catch (IOException e) {
        LOG.error("Unable to open socket to " + addr);
        sock.close();
        throw e;
    }
    initialized = false;
    /*
     * Reset incomingBuffer
     */
    lenBuffer.clear();
    incomingBuffer = lenBuffer;
}
```

```java
void registerAndConnect(SocketChannel sock, InetSocketAddress addr) 
throws IOException {
    sockKey = sock.register(selector, SelectionKey.OP_CONNECT);
    boolean immediateConnect = sock.connect(addr);
    if (immediateConnect) {
        sendThread.primeConnection();
    }
}
```

#### 2.发送连接请求、认证请求

org.apache.zookeeper.ClientCnxn.SendThread#primeConnection

```java
void primeConnection() throws IOException {
    LOG.info("Socket connection established to "
             + clientCnxnSocket.getRemoteSocketAddress()
             + ", initiating session");
    isFirstConnect = false;
    long sessId = (seenRwServerBefore) ? sessionId : 0;
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

        for (AuthData id : authInfo) {
            outgoingQueue.addFirst(new Packet(new RequestHeader(-4,
                    OpCode.auth), null, new AuthPacket(0, id.scheme,
                    id.data), null, null));
        }
        outgoingQueue.addFirst(new Packet(null, null, conReq,
                    null, null, readOnly));
    }
    clientCnxnSocket.enableReadWriteOnly();
    if (LOG.isDebugEnabled()) {
        LOG.debug("Session establishment request sent on "
                + clientCnxnSocket.getRemoteSocketAddress());
    }
}
```

#### 3.读写IO处理

org.apache.zookeeper.ClientCnxnSocketNIO#doTransport

```java
void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue,
                 ClientCnxn cnxn)
        throws IOException, InterruptedException {
    selector.select(waitTimeOut);
    Set<SelectionKey> selected;
    synchronized (this) {
        selected = selector.selectedKeys();
    }
    // Everything below and until we get back to the select is
    // non blocking, so time is effectively a constant. That is
    // Why we just have to do this once, here
    updateNow();
    for (SelectionKey k : selected) {
        SocketChannel sc = ((SocketChannel) k.channel());
        if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
            if (sc.finishConnect()) {
                updateLastSendAndHeard();
                sendThread.primeConnection();
            }
        } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
            doIO(pendingQueue, outgoingQueue, cnxn);
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

2、发送请求时，将请求放入outgoingQueue中，请求写到网络中之后，会将其从outgoingQueue移除，加入到

pendingQueue中，表示已经发送但是为接收到响应

org.apache.zookeeper.ClientCnxnSocketNIO#doIO

```java
void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
  throws InterruptedException, IOException {
    SocketChannel sock = (SocketChannel) sockKey.channel();
    if (sock == null) {
        throw new IOException("Socket is null!");
    }
    if (sockKey.isReadable()) {
        int rc = sock.read(incomingBuffer);
        if (rc < 0) {
            throw new EndOfStreamException(
                    "Unable to read additional data from server sessionid 0x"
                            + Long.toHexString(sessionId)
                            + ", likely server has closed socket");
        }
        if (!incomingBuffer.hasRemaining()) {
            incomingBuffer.flip();
            if (incomingBuffer == lenBuffer) {
                recvCount++;
                readLength();
            } else if (!initialized) {
                readConnectResult();
                enableRead();
                if (findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    // Since SASL authentication has completed (if client is configured to do so),
                    // outgoing packets waiting in the outgoingQueue can now be sent.
                    enableWrite();
                }
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                updateLastHeard();
                initialized = true;
            } else {
              //处理读
                sendThread.readResponse(incomingBuffer);
                lenBuffer.clear();
                incomingBuffer = lenBuffer;
                updateLastHeard();
            }
        }
    }
    if (sockKey.isWritable()) {
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
                sock.write(p.bb);
                if (!p.bb.hasRemaining()) {
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



```java
public ReplyHeader submitRequest(RequestHeader h, Record request,
        Record response, WatchRegistration watchRegistration)
        throws InterruptedException {
    ReplyHeader r = new ReplyHeader();
  	//加入到outgoingQueue中
    Packet packet = queuePacket(h, r, request, response, null, null, null,
                null, watchRegistration);
    synchronized (packet) {
        while (!packet.finished) {
            packet.wait();//等待响应
        }
    }
    return r;
}
```

#### 4.处理响应

org.apache.zookeeper.ClientCnxn.SendThread#readResponse

```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    ByteBufferInputStream bbis = new ByteBufferInputStream(
            incomingBuffer);
    BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
    ReplyHeader replyHdr = new ReplyHeader();
    replyHdr.deserialize(bbia, "header");
    if (replyHdr.getXid() == -2) { //ping
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
    if (replyHdr.getXid() == -4) { //auth
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
    if (replyHdr.getXid() == -1) { //watch
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
        //将watch事件交由eventThread处理
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
		
    Packet packet;
    synchronized (pendingQueue) {
        if (pendingQueue.size() == 0) { 
            throw new IOException("Nothing in the queue, but got "
                    + replyHdr.getXid());
        }
        packet = pendingQueue.remove(); //移除
    }
    /*
     * Since requests are processed in order, we better get a response
     * to the first request!
     */
    try {
      //收到的响应应该对应pendingQueue中的第一个请求
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
        finishPacket(packet);//唤醒阻塞的请求
    }
}
```

#### 5.结束请求

org.apache.zookeeper.ClientCnxn#finishPacket

```java
if (p.watchRegistration != null) {
    p.watchRegistration.register(p.replyHeader.getErr());
}

if (p.cb == null) { //没有回调方法，同步发送
    synchronized (p) {
        p.finished = true;
        p.notifyAll(); //唤醒
    }
} else {
    p.finished = true;
    eventThread.queuePacket(p); //交由EventThread处理
}
```

### EventThread

#### 1.生产Watch事件

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

#### 2.生产响应事件

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

#### 3.处理事件

处理Watch事件和回调方法

org.apache.zookeeper.ClientCnxn.EventThread#processEvent

```java
 private void processEvent(Object event) {
      try {
          if (event instanceof WatcherSetEventPair) {
              // each watcher will process the event
              WatcherSetEventPair pair = (WatcherSetEventPair) event;
              for (Watcher watcher : pair.watchers) {
                  try {
                      watcher.process(pair.event);
                  } catch (Throwable t) {
                      LOG.error("Error while calling watcher ", t);
                  }
              }
          } else {
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

## Standalone模式

### 1.启动

org.apache.zookeeper.server.ZooKeeperServerMain#main

```java
public void runFromConfig(ServerConfig config) throws IOException {
        LOG.info("Starting server");
        FileTxnSnapLog txnLog = null;
        try {
            // Note that this thread isn't going to be doing anything else,
            // so rather than spawning another thread, we will just call
            // run() in this thread.
            // create a file logger url from the command line args
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
            zkServer.setMinSessionTimeout(config.minSessionTimeout);
            zkServer.setMaxSessionTimeout(config.maxSessionTimeout);
            //服务端通信工厂
            cnxnFactory = ServerCnxnFactory.createFactory();
           //设置端口、最大连接数
            cnxnFactory.configure(config.getClientPortAddress(),
                    config.getMaxClientCnxns());
            //绑定端口、加载快照日志到内存、开启会话追踪、设置请求处理器
            cnxnFactory.startup(zkServer);
            // Watch status of ZooKeeper server. It will do a graceful shutdown
            // if the server is not running or hits an internal error.
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

### 2.初始化

org.apache.zookeeper.server.NettyServerCnxnFactory#startup

```java
public void startup(ZooKeeperServer zks) throws IOException,
        InterruptedException {
    start(); //绑定端口
    setZooKeeperServer(zks);
    zks.startdata(); //加载快照、事务日志到内存
    zks.startup(); //会话追踪、设置请求处理器
}
```

### 3.接收请求

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

### 4.请求处理

org.apache.zookeeper.server.NettyServerCnxn#receiveMessage

```java
if (initialized) { //已经初始化，处理非连接请求
    zks.processPacket(this, bb);
    if (zks.shouldThrottle(outstandingCount.incrementAndGet())) {
        disableRecvNoWait();
    }
} else { //尚未初始化，处理连接请求
    LOG.debug("got conn req request from "
            + getRemoteSocketAddress());
    zks.processConnectRequest(this, bb);
    initialized = true;
}
```

#### 4.1处理连接请求

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

##### 4.1.1 创建会话

org.apache.zookeeper.server.ZooKeeperServer#createSession

```java
long createSession(ServerCnxn cnxn, byte passwd[], int timeout) {
    //创建session，进行session追踪
    long sessionId = sessionTracker.createSession(timeout);
    Random r = new Random(sessionId ^ superSecret);
    r.nextBytes(passwd);
    ByteBuffer to = ByteBuffer.allocate(4);
    to.putInt(timeout);
    cnxn.setSessionId(sessionId);
    //提交创建session的请求
    submitRequest(cnxn, sessionId, OpCode.createSession, 0, to, null);
    return sessionId;
}
```

#### 4.2 处理非连接请求

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
    if (h.getType() == OpCode.auth) {
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
        else {
            Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(),
              h.getType(), incomingBuffer, cnxn.getAuthInfo());
            si.setOwner(ServerCnxn.me);
            submitRequest(si);
        }
    }
    cnxn.incrOutstandingRequests(h);
}
```

### 5、Processor处理请求

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

```-
PrepRequestProcessor -> SyncRequestProcessor ->FinalRequestProcessor
```

PrepRequestProcessor

请求预处理器，把请求区分为非事务请求和事务类请求（改变服务器状态的请求），对与事务类请求进行一系列的预处理。

SyncRequestProcessor

事务日志处理器，将事务请求记录到事务日志文件中，同时还会触发Zookeeper尽进行数据快照。

FinalRequestProcessor

最后一个处理器，主要负责进行客户端请求返回之前的收尾工作，创建请求的响应、对与事务类请求将事务应用到内存

## Quorum模式

org.apache.zookeeper.server.quorum.QuorumPeerMain#runFromConfig

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
      //myId
      quorumPeer.setMyid(config.getServerId());
      quorumPeer.setTickTime(config.getTickTime());
      quorumPeer.setInitLimit(config.getInitLimit());
      quorumPeer.setSyncLimit(config.getSyncLimit());
      quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
      quorumPeer.setCnxnFactory(cnxnFactory);
      quorumPeer.setQuorumVerifier(config.getQuorumVerifier());
      quorumPeer.setClientPortAddress(config.getClientPortAddress());
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

启动

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

创建选举算法

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

选举

org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader













### Observer

#### 创建Observer

org.apache.zookeeper.server.quorum.QuorumPeer#makeObserver

```java
protected Observer makeObserver(FileTxnSnapLog logFactory) throws IOException {
    return new Observer(this, new ObserverZooKeeperServer(logFactory,
            this, new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
}
```

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
    // Find the leader by id
    Vote current = self.getCurrentVote();
    for (QuorumServer s : self.getView().values()) {
        if (s.id == current.getId()) {
            // Ensure we have the leader's correct IP address before
            // attempting to connect.
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
            sock.connect(addr, self.tickTime * self.syncLimit);
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
    leaderIs = BinaryInputArchive.getArchive(new BufferedInputStream(
            sock.getInputStream()));
    bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
    leaderOs = BinaryOutputArchive.getArchive(bufferedOutput);
}   
```

向Leader注册

```java
  protected long registerWithLeader(int pktType) throws IOException{
      //发送myId、zxId
   long lastLoggedZxid = self.getLastLoggedZxid();
      QuorumPacket qp = new QuorumPacket();
      // FOLLOWERINFO|OBSERVERINFO
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

#### 数据同步初始化

```java
protected void syncWithLeader(long newLeaderZxid) throws IOException, InterruptedException{
    QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
    QuorumPacket qp = new QuorumPacket();
    long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);
    // In the DIFF case we don't need to do a snapshot because the transactions will sync on top of any existing snapshot
    // For SNAP and TRUNC the snapshot is needed to save that history
    boolean snapshotNeeded = true;
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
            // The leader is going to dump the database
            // clear our own database and read
            zk.getZKDatabase().clear();
            zk.getZKDatabase().deserializeSnapshot(leaderIs);
            String signature = leaderIs.readString("signature");
            if (!signature.equals("BenWasHere")) {
                LOG.error("Missing signature. Got " + signature);
                throw new IOException("Missing signature");                   
            }
            zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
        } else if (qp.getType() == Leader.TRUNC) { //先截断再DiFF同步、仅截断同步
            LOG.warn("Truncating log to get in sync with the leader 0x"
                    + Long.toHexString(qp.getZxid()));
            boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
            if (!truncated) {
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
            case Leader.UPTODATE: //当集群中有半数节点响应Leader发送的NEWLEADER指令并返回ACK时，会接收UPTODATE指令
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
                //发送ACK到Leader节点，表明完成数据同步
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
        for(PacketInFlight p: packetsNotCommitted) {
            fzk.logRequest(p.hdr, p.rec);
        }
        for(Long zxid: packetsCommitted) {
            fzk.commit(zxid);
        }
    } else if (zk instanceof ObserverZooKeeperServer) { //Observer
        ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
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

#### 处理PING/INFORM

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
        //写事务日志、做快照、写内存
        obs.commitRequest(request);            
        break;
    default:
        LOG.error("Invalid packet type: {} received by Observer", qp.getType());
    }
}
```

### Follower

#### 创建Follower

org.apache.zookeeper.server.quorum.QuorumPeer#makeFollower

```java
protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {
    return new Follower(this, new FollowerZooKeeperServer(logFactory, 
            this,new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
}
```

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
    case Leader.COMMIT://提交指令数据包
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

##### 处理Proposal指令

1、记录事务日志 2、发送ACK

org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#logRequest

```java
public void logRequest(TxnHeader hdr, Record txn) {
    Request request = new Request(null, hdr.getClientId(), hdr.getCxid(),
            hdr.getType(), null, null);
    request.hdr = hdr;
    request.txn = txn;
    request.zxid = hdr.getZxid();
    if ((request.zxid & 0xffffffffL) != 0) {
        pendingTxns.add(request); //处理中的事务
    }
  	//SyncRequestProcessor -> SendAckRequestProcessor
    syncProcessor.processRequest(request);
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

##### 处理Commit指令

org.apache.zookeeper.server.quorum.FollowerZooKeeperServer#commit

```java
public void commit(long zxid) {
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
}j
```

### Leader

#### 创建Leader

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
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken);
    LOG.info("LEADING - LEADER ELECTION TOOK - {}", electionTimeTaken);
    self.start_fle = 0;
    self.end_fle = 0;
    zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);
    try {
        self.tick.set(0);
        zk.loadData();
        leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());
        // Start thread that waits for connection requests from
        // new followers.
        cnxAcceptor = new LearnerCnxAcceptor();
        cnxAcceptor.start();
        readyToStart = true;
        long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());
        
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
                if (f.synced() && f.getLearnerType() == LearnerType.PARTICIPANT) {
                    syncedSet.add(f.getSid());
                }
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

#### LearnerCnxAcceptor

监听Follower、Observer连接

org.apache.zookeeper.server.quorum.Leader.LearnerCnxAcceptor#run

```java
public void run() {
    try {
        while (!stop) {
            try{
                Socket s = ss.accept();
                // start with the initLimit, once the ack is processed
                // in LearnerHandler switch to the syncLimit
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

                    // When Leader.shutdown() calls ss.close(),
                    // the call to accept throws an exception.
                    // We catch and set stop to true.
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

管理Leader与Follower、Observer节点的通信

org.apache.zookeeper.server.quorum.LearnerHandler#run

```java
public void run() {
    try {
        leader.addLearnerHandler(this);
        tickOfNextAckDeadline = leader.self.tick.get()
                + leader.self.initLimit + leader.self.syncLimit;
        ia = BinaryInputArchive.getArchive(bufferedInput);
        bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
        oa = BinaryOutputArchive.getArchive(bufferedOutput);

        QuorumPacket qp = new QuorumPacket();
        ia.readRecord(qp, "packet");
        //Leader启动后，接收从节点发送额FOLLOWERINFO、OBSERVERINFO，
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

            if (peerLastZxid == leader.zk.getZKDatabase().getDataTreeLastProcessedZxid()) { //
                // Follower is already sync with us, send empty diff
                LOG.info("leader and follower are in sync, zxid=0x{}",
                        Long.toHexString(peerLastZxid));
                packetToSend = Leader.DIFF;
                zxidToSend = peerLastZxid;
            } else if (proposals.size() != 0) {
                LOG.debug("proposal size is {}", proposals.size());
                if ((maxCommittedLog >= peerLastZxid)
                        && (minCommittedLog <= peerLastZxid)) {
                    LOG.debug("Sending proposals to follower");

                    // as we look through proposals, this variable keeps track of previous
                    // proposal Id.
                    long prevProposalZxid = minCommittedLog;

                    // Keep track of whether we are about to send the first packet.
                    // Before sending the first packet, we have to tell the learner
                    // whether to expect a trunc or a diff
                    boolean firstPacket=true;

                    // If we are here, we can use committedLog to sync with
                    // follower. Then we only need to decide whether to
                    // send trunc or not
                    packetToSend = Leader.DIFF;
                    zxidToSend = maxCommittedLog;

                    for (Proposal propose: proposals) {
                        // skip the proposals the peer already has
                        if (propose.packet.getZxid() <= peerLastZxid) {
                            prevProposalZxid = propose.packet.getZxid();
                            continue;
                        } else {
                            // If we are sending the first packet, figure out whether to trunc
                            // in case the follower has some proposals that the leader doesn't
                            if (firstPacket) {
                                firstPacket = false;
                                // Does the peer have some proposals that the leader hasn't seen yet
                                if (prevProposalZxid < peerLastZxid) {
                                    // send a trunc message before sending the diff
                                    packetToSend = Leader.TRUNC;                                        
                                    zxidToSend = prevProposalZxid;
                                    updates = zxidToSend;
                                }
                            }
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
                    packetToSend = Leader.TRUNC;
                    zxidToSend = maxCommittedLog;
                    updates = zxidToSend;
                } else {
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
            zxidToSend = leader.zk.getZKDatabase().getDataTreeLastProcessedZxid();
        }
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
            // Dump data to peer
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
        //开启循环，针对Leader发出的请求，读取Follower、Observer的响应信息，例如ACK、PING。还会接收转发的事务类请求
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
            case Leader.ACK: //
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
            case Leader.REQUEST: //接收非leader节点转发的事务类请求
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


