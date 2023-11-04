# 启动入口

redis.c

```c
int main(int argc, char **argv) {
​    struct timeval tv;
​    // 初始化库
    #ifdef INIT_SETPROCTITLE_REPLACEMENT
        spt_init(argc, argv);
    #endif
​    setlocale(LC_COLLATE,"");
​    zmalloc_enable_thread_safeness();
​    zmalloc_set_oom_handler(redisOutOfMemoryHandler);
​    srand(time(NULL)^getpid());
​    gettimeofday(&tv,NULL);
​    dictSetHashFunctionSeed(tv.tv_sec^tv.tv_usec^getpid());
​    // 检查服务器是否以 Sentinel 模式启动
​    server.sentinel_mode = checkForSentinelMode(argc,argv);
​    // 初始化服务器
​    initServerConfig();
​    // 如果服务器以 Sentinel 模式启动，那么进行 Sentinel 功能相关的初始化
​    // 并为要监视的主服务器创建一些相应的数据结构
​    if (server.sentinel_mode) {
​        initSentinelConfig();
​        initSentinel();
​    }
​    // 检查用户是否指定了配置文件，或者配置选项
​    if (argc >= 2) {
​        int j = 1; /* First option to parse in argv[] */
​        sds options = sdsempty();
​        char *configfile = NULL;
​        // 处理特殊选项 -h 、-v 和 --test-memory
​        if (strcmp(argv[1], "-v") == 0 ||
​            strcmp(argv[1], "--version") == 0) version();
​        if (strcmp(argv[1], "--help") == 0 ||
​            strcmp(argv[1], "-h") == 0) usage();
​        if (strcmp(argv[1], "--test-memory") == 0) {
​            if (argc == 3) {
​                memtest(atoi(argv[2]),50);
​                exit(0);
​            } else {
​                fprintf(stderr,"Please specify the amount of memory to test in megabytes.\n");
​                fprintf(stderr,"Example: ./redis-server --test-memory 4096\n\n");
​                exit(1);
​            }
​        }
​        /* First argument is the config file name? */
​        // 如果第一个参数（argv[1]）不是以 "--" 开头
​        // 那么它应该是一个配置文件
​        if (argv[j][0] != '-' || argv[j][1] != '-')
​            configfile = argv[j++];
​        // 对用户给定的其余选项进行分析，并将分析所得的字符串追加稍后载入的配置文件的内容之后
​        // 比如 --port 6380 会被分析为 "port 6380\n"
​        while(j != argc) {
​            if (argv[j][0] == '-' && argv[j][1] == '-') {
​                /* Option name */
​                if (sdslen(options)) options = sdscat(options,"\n");
​                options = sdscat(options,argv[j]+2);
​                options = sdscat(options," ");
​            } else {
​                options = sdscatrepr(options,argv[j],strlen(argv[j]));
​                options = sdscat(options," ");
​            }
​            j++;
​        }
​        if (configfile) server.configfile = getAbsolutePath(configfile);
​        // 重置保存条件
​        resetServerSaveParams();
​        // 载入配置文件， options 是前面分析出的给定选项
​        loadServerConfig(configfile,options);
​        sdsfree(options);
​        // 获取配置文件的绝对路径
​        if (configfile) server.configfile = getAbsolutePath(configfile);
​    } else {
​        redisLog(REDIS_WARNING, "Warning: no config file specified, using the default config. In order to specify a config file use %s /path/to/%s.conf", argv[0], server.sentinel_mode ? "sentinel" : "redis");
​    }
​    // 将服务器设置为守护进程
​    if (server.daemonize) daemonize();
​    // 创建并初始化服务器数据结构
​    initServer();
​    // 如果服务器是守护进程，那么创建 PID 文件
​    if (server.daemonize) createPidFile();
​    // 为服务器进程设置名字
​    redisSetProcTitle(argv[0]);
​    // 打印 ASCII LOGO
​    redisAsciiArt();
​    // 如果服务器不是运行在 SENTINEL 模式，那么执行以下代码
​    if (!server.sentinel_mode) {
​        // 打印问候语
​        redisLog(REDIS_WARNING,"Server started, Redis version " REDIS_VERSION);
​    #ifdef __linux__
​        // 打印内存警告
​        linuxOvercommitMemoryWarning();
​    #endif
​        // 从 AOF 文件或者 RDB 文件中载入数据
​        loadDataFromDisk();
​        // 启动集群？
​        if (server.cluster_enabled) {
​            if (verifyClusterConfigWithData() == REDIS_ERR) {
​                redisLog(REDIS_WARNING,
​                    "You can't have keys in a DB different than DB 0 when in "
​                    "Cluster mode. Exiting.");
​                exit(1);
​            }
​        }
​        // 打印 TCP 端口
​        if (server.ipfd_count > 0)
​            redisLog(REDIS_NOTICE,"The server is now ready to accept connections on port %d", server.port);
​        // 打印本地套接字端口
​        if (server.sofd > 0)
​            redisLog(REDIS_NOTICE,"The server is now ready to accept connections at %s", server.unixsocket);
​    } else {
​        sentinelIsRunning();
​    }
​    // 检查不正常的 maxmemory 配置
​    if (server.maxmemory > 0 && server.maxmemory < 1024*1024) {
​        redisLog(REDIS_WARNING,"WARNING: You specified a maxmemory value that is less than 1MB (current value is %llu bytes). Are you sure this is what you really want?", server.maxmemory);
​    }
​    // 运行事件处理器，一直到服务器关闭为止
​    aeSetBeforeSleepProc(server.el,beforeSleep);
​    aeMain(server.el);
​    // 服务器关闭，停止事件循环
​    aeDeleteEventLoop(server.el);
​    return 0;
}
```

# 初始化服务器

```c
void initServer() {

​    int j;
​    // 设置信号处理函数
​    signal(SIGHUP, SIG_IGN);
​    signal(SIGPIPE, SIG_IGN);
​    setupSignalHandlers();

​    // 设置 syslog
​    if (server.syslog_enabled) {
​        openlog(server.syslog_ident, LOG_PID | LOG_NDELAY | LOG_NOWAIT,
​            server.syslog_facility);
​    }

​    // 初始化并创建数据结构
​    server.current_client = NULL;
​    server.clients = listCreate();
​    server.clients_to_close = listCreate();
​    server.slaves = listCreate();
​    server.monitors = listCreate();
​    server.slaveseldb = -1; /* Force to emit the first SELECT command. */
​    server.unblocked_clients = listCreate();
​    server.ready_keys = listCreate();
​    server.clients_waiting_acks = listCreate();
​    server.get_ack_from_slaves = 0;
​    server.clients_paused = 0;

​    // 创建共享对象
​    createSharedObjects();
​    adjustOpenFilesLimit();
  	//创建EventLoop
​    server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);
     //创建DB
​    server.db = zmalloc(sizeof(redisDb)*server.dbnum);

​    // 打开 TCP 监听端口，用于等待客户端的命令请求
​    if (server.port != 0 &&
​        listenToPort(server.port,server.ipfd,&server.ipfd_count) == REDIS_ERR)
​        exit(1);
​   
  	// 打开 UNIX 本地端口
​    if (server.unixsocket != NULL) {
​        unlink(server.unixsocket); /* don't care if this fails */
​        server.sofd = anetUnixServer(server.neterr,server.unixsocket,
​            server.unixsocketperm, server.tcp_backlog);
​        if (server.sofd == ANET_ERR) {
​            redisLog(REDIS_WARNING, "Opening socket: %s", server.neterr);
​            exit(1);
​        }
​        anetNonBlock(NULL,server.sofd);
​    }
  
​    if (server.ipfd_count == 0 && server.sofd < 0) {
​        redisLog(REDIS_WARNING, "Configured to not listen anywhere, exiting.");
​        exit(1);
​    }

​    // 创建并初始化数据库结构
​    for (j = 0; j < server.dbnum; j++) {
​        server.db[j].dict = dictCreate(&dbDictType,NULL);
​        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
​        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
​        server.db[j].ready_keys = dictCreate(&setDictType,NULL);
​        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
​        server.db[j].eviction_pool = evictionPoolAlloc();
​        server.db[j].id = j;
​        server.db[j].avg_ttl = 0;
​    }

​    // 创建 PUBSUB 相关结构
​    server.pubsub_channels = dictCreate(&keylistDictType,NULL);
​    server.pubsub_patterns = listCreate();
​    listSetFreeMethod(server.pubsub_patterns,freePubsubPattern);
​    listSetMatchMethod(server.pubsub_patterns,listMatchPubsubPattern);

​    server.cronloops = 0;
​    server.rdb_child_pid = -1;
​    server.aof_child_pid = -1;
​    aofRewriteBufferReset();
​    server.aof_buf = sdsempty();
​    server.lastsave = time(NULL); /* At startup we consider the DB saved. */
​    server.lastbgsave_try = 0;    /* At startup we never tried to BGSAVE. */
​    server.rdb_save_time_last = -1;
​    server.rdb_save_time_start = -1;
​    server.dirty = 0;
​    resetServerStats();

​    server.stat_starttime = time(NULL);
​    server.stat_peak_memory = 0;
​    server.resident_set_size = 0;
​    server.lastbgsave_status = REDIS_OK;
​    server.aof_last_write_status = REDIS_OK;
​    server.aof_last_write_errno = 0;
​    server.repl_good_slaves_count = 0;
​    updateCachedTime();


​    //创建时间事件
​    if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
​        redisPanic("Can't create the serverCron time event.");
​        exit(1);
​    }

​    // 为 TCP 连接关联连接应答（accept）处理器
​    // acceptTcpHandler用于处理客户端连接
​    for (j = 0; j < server.ipfd_count; j++) {
​        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
​            acceptTcpHandler,NULL) == AE_ERR)
​            {
​                redisPanic(
​                    "Unrecoverable error creating server.ipfd file event.");
​            }
​    }

​    // 为本地套接字关联应答处理器
​    if (server.sofd > 0 && aeCreateFileEvent(server.el,server.sofd,AE_READABLE,
​        acceptUnixHandler,NULL) == AE_ERR)
  				redisPanic("Unrecoverable error creating server.sofd file event.");

​    // 如果AOF持久化功能已经打开，那么打开或创建一个 AOF 文件
​    if (server.aof_state == REDIS_AOF_ON) {
​        server.aof_fd = open(server.aof_filename,
​                               O_WRONLY|O_APPEND|O_CREAT,0644);
​        if (server.aof_fd == -1) {
​            redisLog(REDIS_WARNING, "Can't open the append-only file: %s",
​                strerror(errno));
​            exit(1);
​        }
​    }

​    // 对于 32 位实例来说，默认将最大可用内存限制在 3 GB
​    if (server.arch_bits == 32 && server.maxmemory == 0) {
​        redisLog(REDIS_WARNING,"Warning: 32 bit instance detected but no memory limit set. Setting 3 GB maxmemory limit with 'noeviction' policy now.");
​        server.maxmemory = 3072LL*(1024*1024); /* 3 GB */
​        server.maxmemory_policy = REDIS_MAXMEMORY_NO_EVICTION;
​    }

​    // 如果服务器以 cluster 模式打开，那么初始化 cluster
​    if (server.cluster_enabled) clusterInit();
  
​    // 初始化复制功能有关的脚本缓存
​    replicationScriptCacheInit();
  
​    // 初始化脚本系统
​    scriptingInit();

​    // 初始化慢查询功能，在内存中保存slow log
​    slowlogInit();

​    // 初始化 BIO 系统
​    bioInit();

}
```



# 加载AOF或者RDB文件

```c
void loadDataFromDisk(void) {
​    // 记录开始时间
​    long long start = ustime();
​    //是否开启AOF持久化
​    if (server.aof_state == REDIS_AOF_ON) {
​        // 尝试载入AOF文件
​        if (loadAppendOnlyFile(server.aof_filename) == REDIS_OK)
​            // 打印载入信息，并计算载入耗时
​            redisLog(REDIS_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
​    } else {//AOF持久化未开启
​        // 加载RDB文件
​        if (rdbLoad(server.rdb_filename) == REDIS_OK) {
​            // 打印载入信息，并计算载入耗时长度
​            redisLog(REDIS_NOTICE,"DB loaded from disk: %.3f seconds",
​                (float)(ustime()-start)/1000000);
​        } else if (errno != ENOENT) {
​            redisLog(REDIS_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
​            exit(1);
​        }
​    }
}
```

# 文件时间事件处理

```c
void aeMain(aeEventLoop *eventLoop) {
​    eventLoop->stop = 0;
​    while (!eventLoop->stop) {
​        // 如果有需要在事件处理前执行的函数，那么运行它
​        if (eventLoop->beforesleep != NULL)
​            eventLoop->beforesleep(eventLoop);
​        // 开始处理事件
​        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
​    }
}
```

## 文件事件

### 处理客户端连接

Networoking.c

```c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {

​    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
​    char cip[REDIS_IP_STR_LEN];
​    REDIS_NOTUSED(el);
​    REDIS_NOTUSED(mask);
​    REDIS_NOTUSED(privdata);

​    while(max--) {
​        // accept客户端连接
​        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
​        if (cfd == ANET_ERR) {
​            if (errno != EWOULDBLOCK)
​                redisLog(REDIS_WARNING,
​                    "Accepting client connection: %s", server.neterr);
​            return;
​        }
​        redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
​        // 为客户端创建客户端状态（redisClient）
​        acceptCommonHandler(cfd,0);
​    }
}
```



```c
static void acceptCommonHandler(int fd, int flags) {

​    // 创建客户端
​    redisClient *c;
​    if ((c = createClient(fd)) == NULL) {
​        redisLog(REDIS_WARNING,
​            "Error registering fd event for the new client: %s (fd=%d)",
​            strerror(errno),fd);
​        close(fd); /* May be already closed, just ignore errors */
​        return;
​    }

​    // 如果新添加的客户端令服务器的最大客户端数量达到了
​    // 那么向新客户端写入错误信息，并关闭新客户端
​    // 先创建客户端，再进行数量检查是为了方便地进行错误信息写入
​    if (listLength(server.clients) > server.maxclients) {
​        char *err = "-ERR max number of clients reached\r\n";
​        if (write(c->fd,err,strlen(err)) == -1) {
​        }
​        // 更新拒绝连接数
​        server.stat_rejected_conn++;
​        freeClient(c); //关闭释放连接
​        return;
​    }

​    // 更新连接次数
​    server.stat_numconnections++;
​    // 设置 FLAG
​    c->flags |= flags;

}
```

```c
redisClient *createClient(int fd) {

​    // 分配空间
​    redisClient *c = zmalloc(sizeof(redisClient));

​    // 当 fd 不为 -1 时，创建带网络连接的客户端
​    // 如果 fd 为 -1 ，那么创建无网络连接的伪客户端
​    // 因为 Redis 的命令必须在客户端的上下文中使用，所以在执行 Lua 环境中的命令时
​    // 需要用到这种伪终端
​    if (fd != -1) {
​        // 非阻塞
​        anetNonBlock(NULL,fd);
​        // 禁用 Nagle 算法
​        anetEnableTcpNoDelay(NULL,fd);
​        // 设置 keep alive
​        if (server.tcpkeepalive)
​            anetKeepAlive(NULL,fd,server.tcpkeepalive);
​        // 绑定读事件到事件 loop （开始接收命令请求）
​        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
​            readQueryFromClient, c) == AE_ERR)
​        {
​            close(fd);
​            zfree(c);
​            return NULL;
​        }
​    }

​    // 初始化各个属性
​    selectDb(c,0); //默认选择0号数据库
​    // 套接字
​    c->fd = fd;
​    // 名字
​    c->name = NULL;
​    // 回复缓冲区的偏移量
​    c->bufpos = 0;
​    // 查询缓冲区
​    c->querybuf = sdsempty();
​    // 查询缓冲区峰值
​    c->querybuf_peak = 0;
​    // 命令请求的类型
​    c->reqtype = 0;
​    // 命令参数数量
​    c->argc = 0;
​    // 命令参数
​    c->argv = NULL;
​    // 当前执行的命令和最近一次执行的命令
​    c->cmd = c->lastcmd = NULL;
​    // 查询缓冲区中未读入的命令内容数量
​    c->multibulklen = 0;
​    // 读入的参数的长度
​    c->bulklen = -1;
​    // 已发送字节数
​    c->sentlen = 0;
​    // 状态 FLAG
​    c->flags = 0;
​    // 创建时间和最后一次互动时间
​    c->ctime = c->lastinteraction = server.unixtime;
​    // 认证状态
​    c->authenticated = 0;
​    // 复制状态
​    c->replstate = REDIS_REPL_NONE;
​    // 复制偏移量
​    c->reploff = 0;
​    // 通过 ACK 命令接收到的偏移量
​    c->repl_ack_off = 0;
​    // 通过 AKC 命令接收到偏移量的时间
​    c->repl_ack_time = 0;
​    // 客户端为从服务器时使用，记录了从服务器所使用的端口号
​    c->slave_listening_port = 0;
​    // 回复链表
​    c->reply = listCreate();
​    // 回复链表的字节量
​    c->reply_bytes = 0;
​    // 回复缓冲区大小达到软限制的时间
​    c->obuf_soft_limit_reached_time = 0;
​    // 回复链表的释放和复制函数
​    listSetFreeMethod(c->reply,decrRefCountVoid);
​    listSetDupMethod(c->reply,dupClientReplyValue);
​    // 阻塞类型
​    c->btype = REDIS_BLOCKED_NONE;
​    // 阻塞超时
​    c->bpop.timeout = 0;
​    // 造成客户端阻塞的列表键
​    c->bpop.keys = dictCreate(&setDictType,NULL);
​    // 在解除阻塞时将元素推入到 target 指定的键中
​    // BRPOPLPUSH 命令时使用
​    c->bpop.target = NULL;
​    c->bpop.numreplicas = 0;
​    c->bpop.reploffset = 0;
​    c->woff = 0;
​    // 进行事务时监视的键
​    c->watched_keys = listCreate();
​    // 订阅的频道和模式
​    c->pubsub_channels = dictCreate(&setDictType,NULL);
​    c->pubsub_patterns = listCreate();
​    c->peerid = NULL;
​    listSetFreeMethod(c->pubsub_patterns,decrRefCountVoid);
​    listSetMatchMethod(c->pubsub_patterns,listMatchObjects);
​    // 如果不是伪客户端，那么添加到服务器的客户端链表中
​    if (fd != -1) listAddNodeTail(server.clients,c);
​    // 初始化客户端的事务状态
​    initClientMultiState(c);
​    // 返回客户端
​    return c;
}
```



### 处理请求

```c
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {

​    redisClient *c = (redisClient*) privdata;
​    int nread, readlen;
​    size_t qblen;
​    REDIS_NOTUSED(el);
​    REDIS_NOTUSED(mask);
​    // 设置服务器的当前客户端
​    server.current_client = c;
​    
​    // 读入长度（默认为 16 MB）
​    readlen = REDIS_IOBUF_LEN;

​    if (c->reqtype == REDIS_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
​        && c->bulklen >= REDIS_MBULK_BIG_ARG)
​    {
​        int remaining = (unsigned)(c->bulklen+2)-sdslen(c->querybuf);
​        if (remaining < readlen) readlen = remaining;
​    }

​    // 获取查询缓冲区当前内容的长度
​    // 如果读取出现 short read ，那么可能会有内容滞留在读取缓冲区里面
​    // 这些滞留内容也许不能完整构成一个符合协议的命令，
​    qblen = sdslen(c->querybuf);
​    // 如果有需要，更新缓冲区内容长度的峰值（peak）
​    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
​    // 为查询缓冲区分配空间
​    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
​    // 读入内容到查询缓存
​    nread = read(fd, c->querybuf+qblen, readlen);
​    // 读入出错
​    if (nread == -1) {
​        if (errno == EAGAIN) {
​            nread = 0;
​        } else {
​            redisLog(REDIS_VERBOSE, "Reading from client: %s",strerror(errno));
​            freeClient(c);
​            return;
​        }
​    // 遇到 EOF
​    } else if (nread == 0) {
​        redisLog(REDIS_VERBOSE, "Client closed connection");
​        freeClient(c);
​        return;
​    }

​    if (nread) {
​        // 根据内容，更新查询缓冲区（SDS） free 和 len 属性
​        // 并将 '\0' 正确地放到内容的最后
​        sdsIncrLen(c->querybuf,nread);
​        // 记录服务器和客户端最后一次互动的时间
​        c->lastinteraction = server.unixtime;
​        // 如果客户端是 master 的话，更新它的复制偏移量
​        if (c->flags & REDIS_MASTER) c->reploff += nread;
​    } else {
​        // 在 nread == -1 且 errno == EAGAIN 时运行
​        server.current_client = NULL;
​        return;
​    }
​    // 查询缓冲区长度超出服务器最大缓冲区长度
​    // 清空缓冲区并释放客户端
​    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
​        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();
​        bytes = sdscatrepr(bytes,c->querybuf,64);
​        redisLog(REDIS_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
​        sdsfree(ci);
​        sdsfree(bytes);
​        freeClient(c);
​        return;
​    }

​    // 从查询缓存重读取内容，创建参数，并执行命令
​    // 函数会执行到缓存中的所有内容都被处理完为止
​    processInputBuffer(c);
​    server.current_client = NULL;
}
```

#### processInputBuffer

```c
void processInputBuffer(redisClient *c) {

​    while(sdslen(c->querybuf)) {

​        // 如果客户端正处于暂停状态，那么直接返回
​        if (!(c->flags & REDIS_SLAVE) && clientsArePaused()) return;

​        // REDIS_BLOCKED 状态表示客户端正在被阻塞
​        if (c->flags & REDIS_BLOCKED) return;

​        // 客户端已经设置了关闭 FLAG ，没有必要处理命令了
​        if (c->flags & REDIS_CLOSE_AFTER_REPLY) return;


​        // 判断请求的类型
​        // http://redis.readthedocs.org/en/latest/topic/protocol.html
​        if (!c->reqtype) {
​            if (c->querybuf[0] == '*') {
​                // 多条查询
​                c->reqtype = REDIS_REQ_MULTIBULK;
​            } else {
​                // 内联查询
​                c->reqtype = REDIS_REQ_INLINE;
​            }
​        }

​        // 将缓冲区中的内容转换成命令，以及命令参数
​        if (c->reqtype == REDIS_REQ_INLINE) {
​            if (processInlineBuffer(c) != REDIS_OK) break;
​        } else if (c->reqtype == REDIS_REQ_MULTIBULK) {
​            if (processMultibulkBuffer(c) != REDIS_OK) break;
​        } else {
​            redisPanic("Unknown request type");
​        }

​        if (c->argc == 0) {
​            resetClient(c);//并重置客户端
​        } else {
​            // 执行命令，并重置客户端
​            if (processCommand(c) == REDIS_OK)
​                resetClient(c);
​        }
​    }
}
```

#### processCommand

```c
int processCommand(redisClient *c) {


​    // 特别处理 quit 命令
​    if (!strcasecmp(c->argv[0]->ptr,"quit")) {
​        addReply(c,shared.ok);
​        c->flags |= REDIS_CLOSE_AFTER_REPLY;
​        return REDIS_ERR;
​    }


​    // 查找命令，并进行命令合法性检查，以及命令参数个数检查
​    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
​    if (!c->cmd) {
​        // 没找到指定的命令
​        flagTransaction(c);
​        addReplyErrorFormat(c,"unknown command '%s'",
​            (char*)c->argv[0]->ptr);
​        return REDIS_OK;
​    } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
​               (c->argc < -c->cmd->arity)) {
​        // 参数个数错误
​        flagTransaction(c);
​        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",
​            c->cmd->name);
​        return REDIS_OK;
​    }

​    // 检查认证信息
​    if (server.requirepass && !c->authenticated && c->cmd->proc != authCommand)
​    {
​        flagTransaction(c);
​        addReply(c,shared.noautherr);
​        return REDIS_OK;
​    }

​    if (server.cluster_enabled &&
​        !(c->flags & REDIS_MASTER) &&
​        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0))
​    {
​        int hashslot;
​        // 集群已下线
​        if (server.cluster->state != REDIS_CLUSTER_OK) {
​            flagTransaction(c);
​            addReplySds(c,sdsnew("-CLUSTERDOWN The cluster is down. Use CLUSTER INFO for more information\r\n"));
​            return REDIS_OK;
​        // 集群运作正常
​        } else {
​            int error_code;
​            clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,&hashslot,&error_code);
​            // 不能执行多键处理命令
​            if (n == NULL) {
​                flagTransaction(c);
​                if (error_code == REDIS_CLUSTER_REDIR_CROSS_SLOT) {
​                    addReplySds(c,sdsnew("-CROSSSLOT Keys in request don't hash to the same slot\r\n"));
​                } else if (error_code == REDIS_CLUSTER_REDIR_UNSTABLE) {
​                    addReplySds(c,sdsnew("-TRYAGAIN Multiple keys request during rehashing of slot\r\n"));
​                } else {
​                    redisPanic("getNodeByQuery() unknown error.");
​                }
​                return REDIS_OK;

​            // 命令针对的槽和键不是本节点处理的，进行转向
​            } else if (n != server.cluster->myself) {
​                flagTransaction(c);
​                // -<ASK or MOVED> <slot> <ip>:<port>
​                // 例如 -ASK 10086 127.0.0.1:12345
​                addReplySds(c,sdscatprintf(sdsempty(),
​                    "-%s %d %s:%d\r\n",
​                    (error_code == REDIS_CLUSTER_REDIR_ASK) ? "ASK" : "MOVED",
​                    hashslot,n->ip,n->port));
​                return REDIS_OK;
​            }
​            // 如果执行到这里，说明键 key 所在的槽由本节点处理
​            // 或者客户端执行的是无参数命令
​        }
​    }

​    // 如果设置了最大内存，那么检查内存是否超过限制，并做相应的操作
​    if (server.maxmemory) {
​        // 如果内存已超过限制，那么尝试通过删除过期键来释放内存
​        int retval = freeMemoryIfNeeded();
​        // 如果即将要执行的命令可能占用大量内存（REDIS_CMD_DENYOOM）
​        // 并且前面的内存释放失败的话
​        // 那么向客户端返回内存错误
​        if ((c->cmd->flags & REDIS_CMD_DENYOOM) && retval == REDIS_ERR) {
​            flagTransaction(c);
​            addReply(c, shared.oomerr);
​            return REDIS_OK;
​        }
​    }
​    // 如果这是一个主服务器，并且这个服务器之前执行 BGSAVE 时发生了错误
​    // 那么不执行写命令
​    if (((server.stop_writes_on_bgsave_err &&
​          server.saveparamslen > 0 &&
​          server.lastbgsave_status == REDIS_ERR) ||
​          server.aof_last_write_status == REDIS_ERR) &&
​        server.masterhost == NULL &&
​        (c->cmd->flags & REDIS_CMD_WRITE ||
​         c->cmd->proc == pingCommand))
​    {
​        flagTransaction(c);

​        if (server.aof_last_write_status == REDIS_OK)

​            addReply(c, shared.bgsaveerr);

​        else

​            addReplySds(c,

​                sdscatprintf(sdsempty(),

​                "-MISCONF Errors writing to the AOF file: %s\r\n",

​                strerror(server.aof_last_write_errno)));

​        return REDIS_OK;

​    }




​    // 如果没有足够多的状态良好的从服务器与主服务器连接，禁止写入
​    if (server.repl_min_slaves_to_write &&
​        server.repl_min_slaves_max_lag &&
​        c->cmd->flags & REDIS_CMD_WRITE &&
​        server.repl_good_slaves_count < server.repl_min_slaves_to_write)
​    {
​        flagTransaction(c);
​        addReply(c, shared.noreplicaserr);
​        return REDIS_OK;
​    }

​    // 如果这个服务器是一个只读 slave 的话，那么拒绝执行写命令
​    if (server.masterhost && server.repl_slave_ro &&
​        !(c->flags & REDIS_MASTER) &&
​        c->cmd->flags & REDIS_CMD_WRITE)
​    {
​        addReply(c, shared.roslaveerr);
​        return REDIS_OK;
​    }

​    // 在订阅于发布模式的上下文中，只能执行订阅和退订相关的命令
​    if ((dictSize(c->pubsub_channels) > 0 || listLength(c->pubsub_patterns) > 0)
​        &&
​        c->cmd->proc != subscribeCommand &&
​        c->cmd->proc != unsubscribeCommand &&
​        c->cmd->proc != psubscribeCommand &&
​        c->cmd->proc != punsubscribeCommand) {
​        addReplyError(c,"only (P)SUBSCRIBE / (P)UNSUBSCRIBE / QUIT allowed in this context");
​        return REDIS_OK;
​    } 

​    if (server.masterhost && server.repl_state != REDIS_REPL_CONNECTED &&
​        server.repl_serve_stale_data == 0 &&
​        !(c->cmd->flags & REDIS_CMD_STALE))
​    {
​        flagTransaction(c);
​        addReply(c, shared.masterdownerr);
​        return REDIS_OK;
​    }

​    // 如果服务器正在载入数据到数据库，那么只执行带有 REDIS_CMD_LOADING
​    // 标识的命令，否则将出错
​    if (server.loading && !(c->cmd->flags & REDIS_CMD_LOADING)) {
​        addReply(c, shared.loadingerr);
​        return REDIS_OK;
​    }




​    // Lua 脚本超时，只允许执行限定的操作，比如 SHUTDOWN 和 SCRIPT KILL
​    if (server.lua_timedout &&
​          c->cmd->proc != authCommand &&
​          c->cmd->proc != replconfCommand &&
​        !(c->cmd->proc == shutdownCommand &&
​          c->argc == 2 &&
​          tolower(((char*)c->argv[1]->ptr)[0]) == 'n') &&
​        !(c->cmd->proc == scriptCommand &&
​          c->argc == 2 &&
​          tolower(((char*)c->argv[1]->ptr)[0]) == 'k'))
​    {
​        flagTransaction(c);
​        addReply(c, shared.slowscripterr);
​        return REDIS_OK;
​    }
​    /* Exec the command */
​    if (c->flags & REDIS_MULTI &&
​        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
​        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
​    {
​        // 在事务上下文中
​        // 除 EXEC 、 DISCARD 、 MULTI 和 WATCH 命令之外
​        // 其他所有命令都会被入队到事务队列中
​        queueMultiCommand(c);
​        addReply(c,shared.queued);
​    } else {
​        // 执行命令
​        call(c,REDIS_CALL_FULL);
​        c->woff = server.master_repl_offset;
​        // 处理那些解除了阻塞的键
​        if (listLength(server.ready_keys))
​            handleClientsBlockedOnLists();
​    }
​    return REDIS_OK;
}
```

##### 释放内存

```c
int freeMemoryIfNeeded(void) {
    size_t mem_used, mem_tofree, mem_freed;
    int slaves = listLength(server.slaves);

    mem_used = zmalloc_used_memory(); //目前占用的内存总数
    if (slaves) { //减去从服务器的输出缓冲区的内存
        listIter li;
        listNode *ln;

        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            redisClient *slave = listNodeValue(ln);
            unsigned long obuf_bytes = getClientOutputBufferMemoryUsage(slave);
            if (obuf_bytes > mem_used)
                mem_used = 0;
            else
                mem_used -= obuf_bytes;
        }
    }
    if (server.aof_state != REDIS_AOF_OFF) { //减去AOF缓冲区的内存
        mem_used -= sdslen(server.aof_buf);
        mem_used -= aofRewriteBufferSize();
    }

    if (mem_used <= server.maxmemory) return REDIS_OK; //空间充足

    if (server.maxmemory_policy == REDIS_MAXMEMORY_NO_EVICTION) //空间不足且不允许淘汰
        return REDIS_ERR; 

    mem_tofree = mem_used - server.maxmemory; //计算释放的内存
    mem_freed = 0;
    while (mem_freed < mem_tofree) { //释放内存
        int j, k, keys_freed = 0;
        // 遍历所有字典
        for (j = 0; j < server.dbnum; j++) {
            long bestval = 0; /* just to prevent warning */
            sds bestkey = NULL;
            dictEntry *de;
            redisDb *db = server.db+j;
            dict *dict;

            if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_LRU ||
                server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_RANDOM)
            {
                // 如果策略是 allkeys-lru 或者 allkeys-random 
                // 那么淘汰的目标为所有数据库键
                dict = server.db[j].dict;
            } else {
                // 如果策略是 volatile-lru 、 volatile-random 或者 volatile-ttl 
                // 那么淘汰的目标为带过期时间的数据库键
                dict = server.db[j].expires;
            }

            // 跳过空字典
            if (dictSize(dict) == 0) continue;

            
            if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_RANDOM ||
                server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_RANDOM)
            {// 从目标字典中随机选择
                de = dictGetRandomKey(dict);
                bestkey = dictGetKey(de);
            }

            /* volatile-lru and allkeys-lru policy */
            else if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_LRU ||
                server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
            {
                struct evictionPoolEntry *pool = db->eviction_pool;

                while(bestkey == NULL) {
                  //挑选key，填充db->eviction_pool
                    evictionPoolPopulate(dict, db->dict, db->eviction_pool);
                    /* Go backward from best to worst element to evict. */
                    for (k = REDIS_EVICTION_POOL_SIZE-1; k >= 0; k--) {
                        if (pool[k].key == NULL) continue;
                        de = dictFind(dict,pool[k].key);

                        /* Remove the entry from the pool. */
                        sdsfree(pool[k].key);
                        /* Shift all elements on its right to left. */
                        memmove(pool+k,pool+k+1,
                            sizeof(pool[0])*(REDIS_EVICTION_POOL_SIZE-k-1));
                        /* Clear the element on the right which is empty
                         * since we shifted one position to the left.  */
                        pool[REDIS_EVICTION_POOL_SIZE-1].key = NULL;
                        pool[REDIS_EVICTION_POOL_SIZE-1].idle = 0;

                        /* If the key exists, is our pick. Otherwise it is
                         * a ghost and we need to try the next element. */
                        if (de) {
                            bestkey = dictGetKey(de);
                            break;
                        } else {
                            /* Ghost... */
                            continue;
                        }
                    }
                }
            }

            else if (server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_TTL) {
              //从dict中随机挑选5个key，淘汰过期时间距离当前时间最接近的key
                for (k = 0; k < server.maxmemory_samples; k++) {
                    sds thiskey;
                    long thisval;

                    de = dictGetRandomKey(dict);
                    thiskey = dictGetKey(de);
                    thisval = (long) dictGetVal(de);

                    /* Expire sooner (minor expire unix timestamp) is better
                     * candidate for deletion */
                    if (bestkey == NULL || thisval < bestval) {
                        bestkey = thiskey;
                        bestval = thisval;
                    }
                }
            }

            // 删除被选中的键
            if (bestkey) {
                long long delta;

                robj *keyobj = createStringObject(bestkey,sdslen(bestkey));
                propagateExpire(db,keyobj);
                /* We compute the amount of memory freed by dbDelete() alone.
                 * It is possible that actually the memory needed to propagate
                 * the DEL in AOF and replication link is greater than the one
                 * we are freeing removing the key, but we can't account for
                 * that otherwise we would never exit the loop.
                 *
                 * AOF and Output buffer memory will be freed eventually so
                 * we only care about memory used by the key space. */
                // 计算删除键所释放的内存数量
                delta = (long long) zmalloc_used_memory();
                dbDelete(db,keyobj);
                delta -= (long long) zmalloc_used_memory();
                mem_freed += delta;
                
                // 对淘汰键的计数器增一
                server.stat_evictedkeys++;

                notifyKeyspaceEvent(REDIS_NOTIFY_EVICTED, "evicted",
                    keyobj, db->id);
                decrRefCount(keyobj);
                keys_freed++;

                /* When the memory to free starts to be big enough, we may
                 * start spending so much time here that is impossible to
                 * deliver data to the slaves fast enough, so we force the
                 * transmission here inside the loop. */
                if (slaves) flushSlavesOutputBuffers();
            }
        }

        if (!keys_freed) return REDIS_ERR; /* nothing to free... */
    }

    return REDIS_OK;
}
```



#### call

触发monitor监视器

执行命令

根据执行时间是否写入slow log

写入aof文件

同步数据至salve

```c
void call(redisClient *c, int flags) {//对命令的执行进行了封装（统计、aof、传播）

​    // start 记录命令开始执行的时间
​    long long dirty, start, duration;
​    // 记录命令开始执行前的 FLAG
​    int client_old_flags = c->flags;


​    //MONITOR监视器
​    if (listLength(server.monitors) &&
​        !server.loading &&
​        !(c->cmd->flags & REDIS_CMD_SKIP_MONITOR))
​    {
​        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
​    }

​    c->flags &= ~(REDIS_FORCE_AOF|REDIS_FORCE_REPL);
​    redisOpArrayInit(&server.also_propagate);
​    // 保留旧 dirty 计数器值
​    dirty = server.dirty;
​    // 计算命令开始执行的时间
​    start = ustime();
​    // 执行实现函数
​    c->cmd->proc(c);
​    // 计算命令执行耗费的时间
​    duration = ustime()-start;
​    // 计算命令执行之后的 dirty 值
​    dirty = server.dirty-dirty;


​    // 不将从 Lua 中发出的命令放入 SLOWLOG ，也不进行统计
​    if (server.loading && c->flags & REDIS_LUA_CLIENT)
​        flags &= ~(REDIS_CALL_SLOWLOG | REDIS_CALL_STATS);


​    // 如果调用者是 Lua ，那么根据命令 FLAG 和客户端 FLAG
​    // 打开传播（propagate)标志
​    if (c->flags & REDIS_LUA_CLIENT && server.lua_caller) {
​        if (c->flags & REDIS_FORCE_REPL)
​            server.lua_caller->flags |= REDIS_FORCE_REPL;
​        if (c->flags & REDIS_FORCE_AOF)
​            server.lua_caller->flags |= REDIS_FORCE_AOF;
​    }

​    // 如果有需要，将命令放到 SLOWLOG 里面
​    if (flags & REDIS_CALL_SLOWLOG && c->cmd->proc != execCommand)
​        slowlogPushEntryIfNeeded(c->argv,c->argc,duration);
  
​    // 更新命令的统计信息
​    if (flags & REDIS_CALL_STATS) {
​        c->cmd->microseconds += duration;
​        c->cmd->calls++;
​    }

​    // 将命令复制到 AOF 和 slave 节点
​    if (flags & REDIS_CALL_PROPAGATE) {
​        int flags = REDIS_PROPAGATE_NONE;
​        // 强制 REPL 传播
​        if (c->flags & REDIS_FORCE_REPL) flags |= REDIS_PROPAGATE_REPL;
​        // 强制 AOF 传播
​        if (c->flags & REDIS_FORCE_AOF) flags |= REDIS_PROPAGATE_AOF;
​        // 如果数据库有被修改，那么启用 REPL 和 AOF 传播
​        if (dirty)
​            flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);
​        if (flags != REDIS_PROPAGATE_NONE)
​            propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
​    }
  	
​    c->flags &= ~(REDIS_FORCE_AOF|REDIS_FORCE_REPL);
​    c->flags |= client_old_flags & (REDIS_FORCE_AOF|REDIS_FORCE_REPL);
​    // 传播额外的命令
​    if (server.also_propagate.numops) {
​        int j;
​        redisOp *rop;
​        for (j = 0; j < server.also_propagate.numops; j++) {
​            rop = &server.also_propagate.ops[j];
​            propagate(rop->cmd, rop->dbid, rop->argv, rop->argc, rop->target);
​        }
​        redisOpArrayFree(&server.also_propagate);
​    }
​    server.stat_numcommands++;
}
```

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
​               int flags){

​    // 传播到 AOF
​    if (server.aof_state != REDIS_AOF_OFF && flags & REDIS_PROPAGATE_AOF)
​        feedAppendOnlyFile(cmd,dbid,argv,argc);

​    // 传播到 slave
​    if (flags & REDIS_PROPAGATE_REPL)
​        replicationFeedSlaves(server.slaves,dbid,argv,argc);

}
```

##### 写AOF

```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
		//写aof文件
​    sds buf = sdsempty();
​    robj *tmpargv[3];

​     //使用 SELECT 命令，显式设置数据库，确保之后的命令被设置到正确的数据库
​    if (dictid != server.aof_selected_db) {
​        char seldb[64];
​        snprintf(seldb,sizeof(seldb),"%d",dictid);
​        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
​            (unsigned long)strlen(seldb),seldb);
​        server.aof_selected_db = dictid;
​    }
​    // EXPIRE 、 PEXPIRE 和 EXPIREAT 命令
​    if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
​        cmd->proc == expireatCommand) {
​        ///Translate EXPIRE/PEXPIRE/EXPIREAT into PEXPIREAT 
​         // 将 EXPIRE 、 PEXPIRE 和 EXPIREAT 都翻译成 PEXPIREAT
​        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
​    // SETEX 和 PSETEX 命令
​    } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
​        //Translate SETEX/PSETEX to SET and PEXPIREAT 
​         //将两个命令都翻译成 SET 和 PEXPIREAT
​        tmpargv[0] = createStringObject("SET",3);
​        tmpargv[1] = argv[1];
​        tmpargv[2] = argv[3];
​        buf = catAppendOnlyGenericCommand(buf,3,tmpargv);
​        // PEXPIREAT
​        decrRefCount(tmpargv[0]);
​        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
​    // 其他命令
​    } else {
​        buf = catAppendOnlyGenericCommand(buf,argc,argv);
​    }

​     //将命令追加到AOF缓存中，
​    if (server.aof_state == REDIS_AOF_ON)
​        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

​     //如果BGREWRITEAOF正在进行， 那么我们还需要将命令追加到重写缓存中
      //避免了对AOF文件的征用
​    if (server.aof_child_pid != -1)
​        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

  ​   // 释放
​    sdsfree(buf);

}
```

##### 同步数据

```c
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) {
		//同步数据
​    listNode *ln;
​    listIter li;
​    int j, len;
​    char llstr[REDIS_LONGSTR_SIZE];

​    // backlog 为空，且没有从服务器，直接返回
​    if (server.repl_backlog == NULL && listLength(slaves) == 0) return;

​    redisAssert(!(listLength(slaves) != 0 && server.repl_backlog == NULL));

​    // 如果有需要的话，发送 SELECT 命令，指定数据库
​    if (server.slaveseldb != dictid) {
​        robj *selectcmd;
​        if (dictid >= 0 && dictid < REDIS_SHARED_SELECT_CMDS) {
​            selectcmd = shared.select[dictid];
​        } else {
​            int dictid_len;
​            dictid_len = ll2string(llstr,sizeof(llstr),dictid);
​            selectcmd = createObject(REDIS_STRING,
​                sdscatprintf(sdsempty(),
​                "*2\r\n$6\r\nSELECT\r\n$%d\r\n%s\r\n",
​                dictid_len, llstr));
​        }

​        // 将 SELECT 命令添加到 backlog
​        if (server.repl_backlog) feedReplicationBacklogWithObject(selectcmd);


​        // 发送给所有从服务器
​        listRewind(slaves,&li);
​        while((ln = listNext(&li))) {
​            redisClient *slave = ln->value;
​            addReply(slave,selectcmd);
​        }

​        if (dictid < 0 || dictid >= REDIS_SHARED_SELECT_CMDS)
​            decrRefCount(selectcmd);
​    }

​    server.slaveseldb = dictid;

​    // 将命令写入到backlog
​    if (server.repl_backlog) {
​        char aux[REDIS_LONGSTR_SIZE+3];
​        /* Add the multi bulk reply length. */
​        aux[0] = '*';
​        len = ll2string(aux+1,sizeof(aux)-1,argc);
​        aux[len+1] = '\r';
​        aux[len+2] = '\n';
​        feedReplicationBacklog(aux,len+3);
​        for (j = 0; j < argc; j++) {
​            long objlen = stringObjectLen(argv[j]);
​            // 将参数从对象转换成协议格式
​            aux[0] = '$';
​            len = ll2string(aux+1,sizeof(aux)-1,objlen);
​            aux[len+1] = '\r';
​            aux[len+2] = '\n';
​            feedReplicationBacklog(aux,len+3);
​            feedReplicationBacklogWithObject(argv[j]);
​            feedReplicationBacklog(aux+len+1,2);
​        }
​    }
​    /* Write the command to every slave. */
​    listRewind(slaves,&li);
​    while((ln = listNext(&li))) {
​        // 指向从服务器
​        redisClient *slave = ln->value;
​        // 不要给正在等待 BGSAVE 开始的从服务器发送命令
​        if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START) continue;

​        // 向已经接收完和正在接收 RDB 文件的从服务器发送命令
​        // 如果从服务器正在接收主服务器发送的 RDB 文件，
​        // 那么在初次 SYNC 完成之前，主服务器发送的内容会被放进一个缓冲区里面
​        addReplyMultiBulkLen(slave,argc);
​        for (j = 0; j < argc; j++)
​            addReplyBulk(slave,argv[j]);
​    }
}
```



### 回复响应

```c
void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) {

​    redisClient *c = privdata;
​    int nwritten = 0, totwritten = 0, objlen;
​    size_t objmem;
​    robj *o;
​    REDIS_NOTUSED(el);
​    REDIS_NOTUSED(mask);
​    // 一直循环，直到回复缓冲区为空
​    // 或者指定条件满足为止
​    while(c->bufpos > 0 || listLength(c->reply)) {
​        if (c->bufpos > 0) { //从缓冲区读响应信息
​            nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen);
​            // 出错则跳出
​            if (nwritten <= 0) break;
​            // 成功写入则更新写入计数器变量
​            c->sentlen += nwritten;
​            totwritten += nwritten;
​            // 如果缓冲区中的内容已经全部写入完毕
​            // 那么清空客户端的两个计数器变量
​            if (c->sentlen == c->bufpos) {
​                c->bufpos = 0;
​                c->sentlen = 0;
​            }
​        } else { //从reply列表获取响应
​            // listLength(c->reply) != 0
​            // 取出位于链表最前面的对象
​            o = listNodeValue(listFirst(c->reply));
​            objlen = sdslen(o->ptr);
​            objmem = getStringObjectSdsUsedMemory(o);
​            if (objlen == 0) {
​                listDelNode(c->reply,listFirst(c->reply));
​                c->reply_bytes -= objmem;
​                continue;
​            }
​            // 写入内容到套接字
​            nwritten = write(fd, ((char*)o->ptr)+c->sentlen,objlen-c->sentlen);
​            // 写入出错则跳出
​            if (nwritten <= 0) break;
​            // 成功写入则更新写入计数器变量
​            c->sentlen += nwritten;
​            totwritten += nwritten;
​            // 如果缓冲区内容全部写入完毕，那么删除已写入完毕的节点
​            if (c->sentlen == objlen) {
​                listDelNode(c->reply,listFirst(c->reply));
​                c->sentlen = 0;
​                c->reply_bytes -= objmem;
​            }
​        }
				
  				//为了避免一个非常大的回复独占服务器，当写入的总数量超过64kb ，中断写入，将处理时间让给其他客户端，剩余的内容等下次写入
​        if (totwritten > REDIS_MAX_WRITE_PER_EVENT &&
​            (server.maxmemory == 0 ||
​             zmalloc_used_memory() < server.maxmemory)) break;
​    }

​    // 写入出错检查
​    if (nwritten == -1) {
​        if (errno == EAGAIN) {
​            nwritten = 0;
​        } else {
​            redisLog(REDIS_VERBOSE,
​                "Error writing to client: %s", strerror(errno));
​            freeClient(c);
​            return;
​        }
​    }

​    if (totwritten > 0) {
​        if (!(c->flags & REDIS_MASTER)) c->lastinteraction = server.unixtime;
​    }

  	//输出缓冲区为空并且响应信息的长度为0
​    if (c->bufpos == 0 && listLength(c->reply) == 0) {
​        c->sentlen = 0;
​        // 删除write handler
​        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);
​        // 如果指定了写入之后关闭客户端 FLAG ，那么关闭客户端
​        if (c->flags & REDIS_CLOSE_AFTER_REPLY) 
  						freeClient(c);
​    }
}
```

## 时间事件

定时事件和周期事件

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {

​    int j;

​    REDIS_NOTUSED(eventLoop);

​    REDIS_NOTUSED(id);

​    REDIS_NOTUSED(clientData);

​    if (server.watchdog_period) watchdogScheduleSignal(server.watchdog_period);

​    updateCachedTime();

​    // 记录服务器执行命令的次数
​    run_with_period(100) trackOperationsPerSecond();


​    server.lruclock = getLRUClock();

​    // 记录服务器的内存峰值
​    if (zmalloc_used_memory() > server.stat_peak_memory)
​        server.stat_peak_memory = zmalloc_used_memory();
​    server.resident_set_size = zmalloc_get_rss();


​    // 服务器进程收到 SIGTERM 信号，关闭服务器
​    if (server.shutdown_asap) {
​        // 尝试关闭服务器
​        if (prepareForShutdown(0) == REDIS_OK) exit(0);
​        // 如果关闭失败，那么打印 LOG ，并移除关闭标识
​        redisLog(REDIS_WARNING,"SIGTERM received but errors trying to shut down the server, check the logs for more information");
​        server.shutdown_asap = 0;
​    }

​    // 打印数据库的键值对信息
​    run_with_period(5000) {
​        for (j = 0; j < server.dbnum; j++) {
​            long long size, used, vkeys;
​            // 可用键值对的数量
​            size = dictSlots(server.db[j].dict);
​            // 已用键值对的数量
​            used = dictSize(server.db[j].dict);
​            // 带有过期时间的键值对数量
​            vkeys = dictSize(server.db[j].expires);
​            // 用 LOG 打印数量
​            if (used || vkeys) {
​                redisLog(REDIS_VERBOSE,"DB %d: %lld keys (%lld volatile) in %lld slots HT.",j,used,vkeys,size);
​                /* dictPrintStats(server.dict); */
​            }
​        }
​    }

​    // 如果服务器没有运行在 SENTINEL 模式下，那么打印客户端的连接信息
​    if (!server.sentinel_mode) {
​        run_with_period(5000) {
​            redisLog(REDIS_VERBOSE,
​                "%lu clients connected (%lu slaves), %zu bytes in use",
​                listLength(server.clients)-listLength(server.slaves),
​                listLength(server.slaves),
​                zmalloc_used_memory());
​        }
​    }

​    // 检查客户端，关闭超时客户端，并释放客户端多余的缓冲区
​    clientsCron();

​    // 对数据库执行各种操作（清除过期key、渐进式rehash）
​    databasesCron();

​    // 如果 BGSAVE 和 BGREWRITEAOF 都没有在执行
​    // 并且有一个BGREWRITEAOF在等待，那么执行 BGREWRITEAOF
​    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
​        server.aof_rewrite_scheduled)
​    {
​        rewriteAppendOnlyFileBackground(); //重写AOF文件
​    }

​    // 检查 BGSAVE 或者 BGREWRITEAOF 是否已经执行完毕
​    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1) {
​        int statloc;
​        pid_t pid;
​        // 接收子进程发来的信号，非阻塞
​        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
​            int exitcode = WEXITSTATUS(statloc);
​            int bysignal = 0;
​            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);
​            // BGSAVE 执行完毕
​            if (pid == server.rdb_child_pid) {
​                backgroundSaveDoneHandler(exitcode,bysignal);
​            // BGREWRITEAOF 执行完毕
​            } else if (pid == server.aof_child_pid) {
​                backgroundRewriteDoneHandler(exitcode,bysignal);
​            } else {
​                redisLog(REDIS_WARNING,
​                    "Warning, detected child with unmatched pid: %ld",
​                    (long)pid);
​            }
​            updateDictResizePolicy();
​        }
​    } else {
​        // 既然没有 BGSAVE 或者 BGREWRITEAOF 在执行，那么检查是否需要执行它们
​        // 遍历所有保存条件，看是否需要执行 BGSAVE 命令
​         for (j = 0; j < server.saveparamslen; j++) {
​            struct saveparam *sp = server.saveparams+j;
​            // 检查是否有某个保存条件已经满足了
​            if (server.dirty >= sp->changes &&
​                server.unixtime-server.lastsave > sp->seconds &&
​                (server.unixtime-server.lastbgsave_try >
​                 REDIS_BGSAVE_RETRY_DELAY ||
​                 server.lastbgsave_status == REDIS_OK))
​            {
​                redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
​                    sp->changes, (int)sp->seconds);
​                // 执行 BGSAVE
​                rdbSaveBackground(server.rdb_filename);
​                break;
​            }
​         }
​        // 触发重写AOF
​         if (server.rdb_child_pid == -1 &&
​             server.aof_child_pid == -1 &&
​             server.aof_rewrite_perc &&
​             // AOF 文件的当前大小大于执行 BGREWRITEAOF 所需的最小大小
​             server.aof_current_size > server.aof_rewrite_min_size)
​         {
​            // 上一次完成 AOF 写入之后，AOF 文件的大小
​            long long base = server.aof_rewrite_base_size ?
​                            server.aof_rewrite_base_size : 1;
​            // AOF 文件当前的体积相对于 base 的体积的百分比
​            long long growth = (server.aof_current_size*100/base) - 100;
​            // 如果增长体积的百分比超过了 growth ，那么执行 BGREWRITEAOF
​            if (growth >= server.aof_rewrite_perc) {
​                redisLog(REDIS_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
​                // 执行 BGREWRITEAOF
​                rewriteAppendOnlyFileBackground();
​            }
​         }
​    }

​    // 根据 AOF 政策，
​    // 考虑是否需要将 AOF 缓冲区中的内容写入到 AOF 文件中
​    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

​    run_with_period(1000) {
​        if (server.aof_last_write_status == REDIS_ERR)
​            flushAppendOnlyFile(0);
​    }

​    // 关闭那些需要异步关闭的客户端
​    freeClientsInAsyncFreeQueue();

​    clientsArePaused(); /* Don't check return value, just use the side effect. */

​    // 重连接主服务器、向主服务器发送 ACK 、判断数据发送失败情况、断开本服务器超时的从服务器，等等
​    run_with_period(1000) replicationCron();

​    // 如果服务器运行在集群模式下，那么执行集群操作
​    run_with_period(100) {
​        if (server.cluster_enabled) clusterCron();
​    }

​    // 如果服务器运行在 sentinel 模式下，那么执行 SENTINEL 的主函数
​    run_with_period(100) {
​        if (server.sentinel_mode) sentinelTimer();
​    }

​    // 集群
​    run_with_period(1000) {
​        migrateCloseTimedoutSockets();
​    }
​    server.cronloops++;

    return 1000/server.hz;
}
```

### databasesCron

1、清除过期key

2、对哈希表进行渐进式的rehash

```c
void databasesCron(void) {

​    // 如果服务器不是从服务器，那么执行主动过期键清除(随机删除过期key)
​    if (server.active_expire_enabled && server.masterhost == NULL)
​        // 清除模式为CYCLE_SLOW
​        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);


​    // 在没有 BGSAVE 或者 BGREWRITEAOF 执行时，对哈希表进行 rehash
​    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
​        static unsigned int resize_db = 0;
​        static unsigned int rehash_db = 0;
​        unsigned int dbs_per_call = REDIS_DBCRON_DBS_PER_CALL;
​        unsigned int j;

​        // 设定要测试的数据库数量
​        if (dbs_per_call > server.dbnum) dbs_per_call = server.dbnum;

​        // 调整字典的大小
​        for (j = 0; j < dbs_per_call; j++) {
​            tryResizeHashTables(resize_db % server.dbnum);
​            resize_db++;
​        }

​        // 对字典进行渐进式 rehash
​        if (server.activerehashing) {
​            for (j = 0; j < dbs_per_call; j++) {
​                int work_done = incrementallyRehash(rehash_db % server.dbnum);
​                rehash_db++;
​                if (work_done) {
​                    break;
​                }
​            }
​        }
​    }
}
```

#### 清除过期key

限制清除key这个操作的执行时间

每此会挑选固定数量的数据库进行删除，下次执行的时候会从下一个数据库开始执行

随机挑选过期的key，如果过期的key过多，会一直随机挑选然后删除，直到过期key的低于设置的阈值或者执行时间达到上限

```c
void activeExpireCycle(int type) {

​    // 静态变量，用来累积函数连续执行时的数据
​    static unsigned int current_db = 0; /* Last DB tested. */
​    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
​    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

​    unsigned int j, iteration = 0;
  
​    // 默认每次处理的数据库数量
​    unsigned int dbs_per_call = REDIS_DBCRON_DBS_PER_CALL;

​    // 函数开始的时间
​    long long start = ustime(), timelimit;


​    // 快速模式
​    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {

​        // 如果上次函数没有触发 timelimit_exit ，那么不执行处理
​        if (!timelimit_exit) return;

​        // 如果距离上次执行未够一定时间，那么不执行处理
​        if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;

​        // 运行到这里，说明执行快速处理，记录当前时间
​        last_fast_cycle = start;

​    }


​    if (dbs_per_call > server.dbnum || timelimit_exit)
​        dbs_per_call = server.dbnum;



​    // 函数处理的微秒时间上限
​    timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
​    timelimit_exit = 0;
​    if (timelimit <= 0) timelimit = 1;


​    // 如果是运行在快速模式之下,最多运行1000（微秒）
​    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
​        timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* in microseconds. */

​    // 遍历数据库
​    for (j = 0; j < dbs_per_call; j++) {
​        int expired;
​        // 指向要处理的数据库
​        redisDb *db = server.db+(current_db % server.dbnum);


​        // 为 DB 计数器加一，如果进入 do 循环之后因为超时而跳出
​        // 那么下次会直接从下个 DB 开始处理
​        current_db++;

​        do {
​            unsigned long num, slots;
​            long long now, ttl_sum;
​            int ttl_samples;

​            // 获取数据库中带过期时间的键的数量
​            // 如果该数量为 0 ，直接跳过这个数据库
​            if ((num = dictSize(db->expires)) == 0) {
​                db->avg_ttl = 0;
​                break;
​            }
​            // 获取数据库中键值对的数量
​            slots = dictSlots(db->expires);
​            // 当前时间
​            now = mstime();

​            // 这个数据库的使用率低于 1% ，扫描起来太费力了（大部分都会 MISS）
​            // 跳过，等待字典收缩程序运行
​            if (num && slots > DICT_HT_INITIAL_SIZE &&
​                (num*100/slots < 1)) break;

​            // 已处理过期键计数器
​            expired = 0;
​            // 键的总 TTL 计数器
​            ttl_sum = 0;
​            // 总共处理的键计数器
​            ttl_samples = 0;

​            // 每次最多只能检查20个键
​            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
​                num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;
  
​            // 开始遍历数据库
​            while (num--) {
​                dictEntry *de;
​                long long ttl;
​                // 从 expires 中随机取出一个带过期时间的键
​                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
​                // 计算 TTL
​                ttl = dictGetSignedIntegerVal(de)-now;
​                // 如果键已经过期，那么删除它，并将 expired 计数器增一
​                if (activeExpireCycleTryExpire(db,de,now)) expired++;
​                if (ttl < 0) ttl = 0;
​                // 累积键的 TTL
​                ttl_sum += ttl;
​                // 累积处理键的个数
​                ttl_samples++;
​            }

​            // 为这个数据库更新平均 TTL 统计数据
​            if (ttl_samples) {
​                // 计算当前平均值
​                long long avg_ttl = ttl_sum/ttl_samples;
​                // 如果这是第一次设置数据库平均 TTL ，那么进行初始化
​                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
​                // 取数据库的上次平均 TTL 和今次平均 TTL 的平均值
​                db->avg_ttl = (db->avg_ttl+avg_ttl)/2;
​            }


​            // 我们不能用太长时间处理过期键，
​            // 所以这个函数执行一定时间之后就要返回
​            // 更新遍历次数
​            iteration++;
​            // 每遍历 16 次执行一次
​            if ((iteration & 0xf) == 0 && /* check once every 16 iterations. */
​                (ustime()-start) > timelimit)
​            {
​                // 如果遍历次数正好是 16 的倍数
​                // 并且遍历的时间超过了 timelimit
​                // 那么断开 timelimit_exit
​                timelimit_exit = 1;
​            }
​            // 已经超时了，返回
​            if (timelimit_exit) return;

​            // 如果随机选择的20个key中，过期的key没有超过25%，则不再进行遍历
​        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
​    }
}
```

#### 调整字典大小

```c
void tryResizeHashTables(int dbid) {

​    if (htNeedsResize(server.db[dbid].dict))
​        dictResize(server.db[dbid].dict);

​    if (htNeedsResize(server.db[dbid].expires))
​        dictResize(server.db[dbid].expires);

}
```



```c
int htNeedsResize(dict *dict) { //是否需要调整字典的容量

​    long long size, used;
​    size = dictSlots(dict);
​    used = dictSize(dict);
​    return (size && used && size > DICT_HT_INITIAL_SIZE &&
​            (used*100/size < REDIS_HT_MINFILL));

}
```



```c
int dictResize(dict *d) //调整字典容量

{

​    int minimal;

​    // 不能在关闭 rehash 或者正在 rehash 的时候调用
​    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;

​    minimal = d->ht[0].used;
​    if (minimal < DICT_HT_INITIAL_SIZE)
​        minimal = DICT_HT_INITIAL_SIZE;

​    // 调整字典的大小
​    return dictExpand(d, minimal);

}
```



```c
int dictExpand(dict *d, unsigned long size)//调整字典的大小
{

​    // 新哈希表
​    dictht n; /* the new hash table */

​   //计算第一个大于等于size的2的N次方，用作哈希表的值
​    unsigned long realsize = _dictNextPower(size);


​    // 不能在字典正在 rehash 时进行
​    // size 的值也不能小于0号哈希表的当前已使用节点
​    if (dictIsRehashing(d) || d->ht[0].used > size)
​        return DICT_ERR;

​    // 为哈希表分配空间，并将所有指针指向 NULL
​    n.size = realsize;
​    n.sizemask = realsize-1;
​    n.table = zcalloc(realsize*sizeof(dictEntry*)); //分配内存
​    n.used = 0;



​    // 如果 0 号哈希表为空，那么这是一次初始化：
​    // 程序将新哈希表赋给 0 号哈希表的指针，然后字典就可以开始处理键值对了。
​    if (d->ht[0].table == NULL) {
​        d->ht[0] = n;
​        return DICT_OK;
​    }


​    // 如果0号哈希表非空，那么这是一次rehash ：
​    // 将新哈希表设置为1号哈希表，
​    d->ht[1] = n;
​    d->rehashidx = 0; //进行rehash的标志
​    return DICT_OK;
}
```

#### 主动rehash

```c
int incrementallyRehash(int dbid) {

​    /* Keys dictionary */
​    if (dictIsRehashing(server.db[dbid].dict)) { //正在进行rehash，标志符为1
​        dictRehashMilliseconds(server.db[dbid].dict,1);//限制rehash的时间为1毫秒
​        return 1;
​    }
  
​    /* Expires */
​    if (dictIsRehashing(server.db[dbid].expires)) {
​        dictRehashMilliseconds(server.db[dbid].expires,1);
​        return 1; 
​    }

​    return 0;
}
```

```go
int dictRehashMilliseconds(dict *d, int ms) {
​    // 记录开始时间
​    long long start = timeInMilliseconds();
​    int rehashes = 0;
​    while(dictRehash(d,100)) { //默认每迁移100个桶，查看执行的时间是否超过1毫秒
​        rehashes += 100;
​        if (timeInMilliseconds()-start > ms) break;
​    }
​    return rehashes;

}
```

```c
int dictRehash(dict *d, int n) {

​    // 只可以在 rehash 进行中时执行
​    if (!dictIsRehashing(d)) return 0;
​    // 进行 N 步迁移
​    while(n--) {
​        dictEntry *de, *nextde;
​        /* Check if we already rehashed the whole table... */
​        // 如果 0 号哈希表为空，那么表示 rehash 执行完毕
​        // T = O(1)
​        if (d->ht[0].used == 0) {
​            // 释放 0 号哈希表
​            zfree(d->ht[0].table);
​            // 将原来的 1 号哈希表设置为新的 0 号哈希表
​            d->ht[0] = d->ht[1];
​            // 重置旧的 1 号哈希表
​            _dictReset(&d->ht[1]);
​            // 关闭 rehash 标识
​            d->rehashidx = -1;
​            // 返回0表示 rehash 已经完成
​            return 0;
​        }


​        // 确保 rehashidx 没有越界
​        assert(d->ht[0].size > (unsigned)d->rehashidx);

​        // 略过数组中为空的索引，找到下一个非空索引
​        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;

​        // 指向该索引的链表表头节点
​        de = d->ht[0].table[d->rehashidx];
  
​        // 将链表中的所有节点迁移到新哈希表
​        while(de) {
​            unsigned int h;
​            // 保存下个节点的指针
​            nextde = de->next;
​            // 计算新哈希表的哈希值，以及节点插入的索引位置
​            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
​            // 插入节点到新哈希表
​            de->next = d->ht[1].table[h];
​            d->ht[1].table[h] = de;
  
​            // 更新计数器
​            d->ht[0].used--;
​            d->ht[1].used++;
​            // 继续处理下个节点
​            de = nextde;
​        }
​        // 将刚迁移完的哈希表索引的指针设为空
​        d->ht[0].table[d->rehashidx] = NULL;
​        // 更新 rehash 索引
​        d->rehashidx++;
​    }
​    return 1;
}
```

# 处理Set命令

```c
void setCommand(redisClient *c) {

​    int j;

​    robj *expire = NULL;readQueryFromClient

​    int unit = UNIT_SECONDS;

​    int flags = REDIS_SET_NO_FLAGS;



​    // 设置选项参数

​    for (j = 3; j < c->argc; j++) {

​        char *a = c->argv[j]->ptr;

​        robj *next = (j == c->argc-1) ? NULL : c->argv[j+1];



​        if ((a[0] == 'n' || a[0] == 'N') &&

​            (a[1] == 'x' || a[1] == 'X') && a[2] == '\0') {

​            flags |= REDIS_SET_NX;

​        } else if ((a[0] == 'x' || a[0] == 'X') &&

​                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0') {

​            flags |= REDIS_SET_XX;

​        } else if ((a[0] == 'e' || a[0] == 'E') &&

​                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' && next) {

​            unit = UNIT_SECONDS;

​            expire = next;

​            j++;

​        } else if ((a[0] == 'p' || a[0] == 'P') &&

​                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' && next) {

​            unit = UNIT_MILLISECONDS;

​            expire = next;

​            j++;

​        } else {

​            addReply(c,shared.syntaxerr);

​            return;

​        }

​    }



​    // 尝试对值对象进行编码
​    c->argv[2] = tryObjectEncoding(c->argv[2]);

​    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}
```

```c
void setGenericCommand(redisClient *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {

  	//过期时间
​    long long milliseconds = 0; 
​    if (expire) {
​        // 取出过期时间
​        if (getLongLongFromObjectOrReply(c, expire, &milliseconds, NULL) != REDIS_OK)
​            return;
​        // expire 参数的值不正确时报错
​        if (milliseconds <= 0) {
​            addReplyError(c,"invalid expire time in SETEX");
​            return;
​        }
​        // Redis以毫秒的形式保存过期时间
​        if (unit == UNIT_SECONDS) milliseconds *= 1000;

​    }



​    // 如果设置了 NX 或者 XX 参数，那么检查条件是否不符合这两个设置
​    // 在条件不符合时报错，报错的内容由 abort_reply 参数决定
​    if ((flags & REDIS_SET_NX && lookupKeyWrite(c->db,key) != NULL) ||
​        (flags & REDIS_SET_XX && lookupKeyWrite(c->db,key) == NULL))
​    {
​        addReply(c, abort_reply ? abort_reply : shared.nullbulk);
​        return;
​    }

​    // 将键值关联到数据库
​    setKey(c->db,key,val);

​    // 将数据库设为脏
​    server.dirty++;

​    // 为键设置过期时间
​    if (expire) setExpire(c->db,key,mstime()+milliseconds);

​    // 发送事件通知
​    notifyKeyspaceEvent(REDIS_NOTIFY_STRING,"set",key,c->db->id);


​    // 发送事件通知
​    if (expire) notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,
​        "expire",key,c->db->id);

​    // 设置成功，向客户端发送回复
​    // 回复的内容由 ok_reply 决定
​    addReply(c, ok_reply ? ok_reply : shared.ok);

}
```

## setKey

```c
void setKey(redisDb *db, robj *key, robj *val) {

​    // 添加或覆写数据库中的键值对
​    if (lookupKeyWrite(db,key) == NULL) {
​        dbAdd(db,key,val);//添加
​    } else { 
​        dbOverwrite(db,key,val); //覆盖
​    }

​    incrRefCount(val); //增加被引用的次数

  ​    // 移除键的过期时间
​    removeExpire(db,key);

​    // 发送键修改通知
​    signalModifiedKey(db,key);

}
```

### dictAdd

```c
int dictAdd(dict *d, void *key, void *val){ //新建	

​    // 尝试添加键到字典，并返回包含了这个键的新哈希节点
​    dictEntry *entry = dictAddRaw(d,key);

​    // 键已存在，添加失败
​    if (!entry) return DICT_ERR;

​    // 键不存在，设置节点的值
​    dictSetVal(d, entry, val);

​    // 添加成功
​    return DICT_OK;

}
```



```c
dictEntry *dictAddRaw(dict *d, void *key)

{

​    int index;
​    dictEntry *entry;
​    dictht *ht;

​    //进行单步rehash
​    if (dictIsRehashing(d)) _dictRehashStep(d);

​    // 计算键在哈希表中的索引值，如果值为 -1 ，那么表示键已经存在
​    if ((index = _dictKeyIndex(d, key)) == -1)
​        return NULL;

​    // 如果字典正在 rehash ，那么将新键添加到 1 号哈希表
​    // 否则，将新键添加到 0 号哈希表
​    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
​    // 为新节点分配空间
​    entry = zmalloc(sizeof(*entry));
​    // 将新节点插入到链表表头
​    entry->next = ht->table[index];
​    ht->table[index] = entry;
​    // 更新哈希表已使用节点数量
​    ht->used++;

​    // 设置新节点的键
​    dictSetKey(d, entry, key);
​    return entry;

}
```

### dbOverwrite

```c
void dbOverwrite(redisDb *db, robj *key, robj *val) { //覆盖写
 
​    dictEntry *de = dictFind(db->dict,key->ptr); //根据key查找旧值

​    redisAssertWithInfo(NULL,key,de != NULL);// 必须存在，否则中止
​   
​    dictReplace(db->dict, key->ptr, val); // 覆写旧值

}
```



```c
int dictReplace(dict *d, void *key, void *val){

​    dictEntry *entry, auxentry;

​    // 尝试直接将键值对添加到字典
​    // 如果键key不存在的话，添加会成功
​    if (dictAdd(d, key, val) == DICT_OK)
​        return 1;


​    // 运行到这里，说明键 key 已经存在，那么找出包含这个 key 的节点
​    entry = dictFind(d, key);
​    // 先保存原有的值的指针
​    auxentry = *entry;
​    // 然后设置新的值
​    dictSetVal(d, entry, val);
​    // 然后释放旧值
​    dictFreeVal(d, &auxentry);

​    return 0;

}
```

## setExpire

```c
void setExpire(redisDb *db, robj *key, long long when) {

​    dictEntry *kde, *de;

​    // 从数据字典获取
​    kde = dictFind(db->dict,key->ptr);

​    redisAssertWithInfo(NULL,key,kde != NULL); //该key必须存在

​    // 从过期字典获取
​    de = dictReplaceRaw(db->expires,dictGetKey(kde));

​    // 设置键的过期时间
​    dictSetSignedIntegerVal(de,when);

}
```

# get命令

```c
void getCommand(redisClient *c) {

​    getGenericCommand(c);

}
```



```c
int getGenericCommand(redisClient *c) {

​    robj *o;

​    // 尝试从数据库中取出键 c->argv[1] 对应的值对象
​    // 如果键不存在时，向客户端发送回复信息，并返回 NULL
​    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL)
​        return REDIS_OK;

​    // 值对象存在，检查它的类型
​    if (o->type != REDIS_STRING) {
​        // 类型错误
​        addReply(c,shared.wrongtypeerr);
​        return REDIS_ERR;
​    } else {
​        // 类型正确，向客户端返回对象的值
​        addReplyBulk(c,o);
​        return REDIS_OK;
​    }

}
```

```c
robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply) {

​    // 查找
​    robj *o = lookupKeyRead(c->db, key);

​    // 决定是否发送信息
​    if (!o) addReply(c,reply);

​    return o;

}
```



```c
robj *lookupKeyRead(redisDb *db, robj *key) {

​    robj *val;
​    // 检查 key 释放已经过期
​    expireIfNeeded(db,key);

​    // 从数据库中取出键的值
​    val = lookupKey(db,key);

​    // 更新命中/不命中信息
​    if (val == NULL)
​        server.stat_keyspace_misses++;
​    else
​        server.stat_keyspace_hits++;
​    // 返回值
​    return val;

}
```

## expireIfNeeded

```c
int expireIfNeeded(redisDb *db, robj *key) {

​    // 取出键的过期时间
​    mstime_t when = getExpire(db,key);
​    mstime_t now;

​    // 没有过期时间
​    if (when < 0) return 0; 



​    // 如果服务器正在进行载入，那么不进行任何过期检查
​    if (server.loading) return 0;

​    now = server.lua_caller ? server.lua_time_start : mstime();


​    // 当服务器运行在replication模式时
​    // 附属节点并不主动删除 key
​    // 它只返回一个逻辑上正确的返回值
​    // 真正的删除操作要等待主节点发来删除命令时才执行
​    // 从而保证数据的同步
​    if (server.masterhost != NULL) return now > when;

​    // 运行到这里，表示键带有过期时间，并且服务器为主节点
​    // 如果未过期，返回 0
​    if (now <= when) return 0;
  
​    server.stat_expiredkeys++;
​    // 向 AOF 文件和附属节点传播过期信息
​    propagateExpire(db,key);

​    // 发送事件通知
​    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
​        "expired",key,db->id);

​    // 将过期键从数据库中删除
​    return dbDelete(db,key);

}
```

## lookupKey

```c
robj *lookupKey(redisDb *db, robj *key) { //查找key对应的value

​    // 从数据字典查找
​    dictEntry *de = dictFind(db->dict,key->ptr);
​    
     // 节点存在
​    if (de) {
​        // 取出值
​        robj *val = dictGetVal(de);

​        // 更新时间信息（只在不存在子进程时执行，防止破坏 copy-on-write 机制）
​        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
​            val->lru = LRU_CLOCK();
​        // 返回值
​        return val;
​    } else {
​        // 节点不存在
​        return NULL;
​    }
}
```

# AOF

## 追加AOF

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc,
               int flags)
{
    // 传播到 AOF
    if (server.aof_state != REDIS_AOF_OFF && flags & REDIS_PROPAGATE_AOF)
        feedAppendOnlyFile(cmd,dbid,argv,argc);

    // 传播到 slave
    if (flags & REDIS_PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves,dbid,argv,argc);
}
```

```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
    sds buf = sdsempty();
    robj *tmpargv[3];
     //设置数据库
    if (dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);

        server.aof_selected_db = dictid;
    }

    // EXPIRE 、 PEXPIRE 和 EXPIREAT 命令
    if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
        cmd->proc == expireatCommand) {
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
    // SETEX 和 PSETEX 命令
    } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
        tmpargv[0] = createStringObject("SET",3);
        tmpargv[1] = argv[1];
        tmpargv[2] = argv[3];
        buf = catAppendOnlyGenericCommand(buf,3,tmpargv);
        decrRefCount(tmpargv[0]);
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
    // 其他命令
    } else {
        buf = catAppendOnlyGenericCommand(buf,argc,argv);
    }

     //将命令追加到 AOF 缓存中，
    if (server.aof_state == REDIS_AOF_ON)
        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

     //如果 BGREWRITEAOF 正在进行，那么我们还需要将命令追加到重写缓存中，
    if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

    // 释放
    sdsfree(buf);
}
```



## 刷盘AOF

```c
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    // 缓冲区中没有任何内容，直接返回
    if (sdslen(server.aof_buf) == 0) return;
    // 策略为每秒 FSYNC 
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        // 是否有 SYNC 正在后台进行
        sync_in_progress = bioPendingJobsOfType(REDIS_BIO_AOF_FSYNC) != 0;

    // 每秒 fsync ，并且强制写入为假
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {

         //当 fsync 策略为每秒钟一次时， fsync 在后台执行。
         //如果后台仍在执行 FSYNC ，那么我们可以延迟写操作一两秒
         //（如果强制执行 write 的话，服务器主线程将阻塞在 write 上面）
        if (sync_in_progress) { //sync正在进行
            if (server.aof_flush_postponed_start == 0) {//前面没有推迟过 write 操作
                //记录推迟写的时间
                server.aof_flush_postponed_start = server.unixtime;
                return;
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. 
                 *
                 * 如果之前已经因为 fsync 而推迟了 write 操作
                 * 但是推迟的时间不超过 2 秒，那么直接返回
                 * 不执行 write 或者 fsync
                 */
                return;

            }

            server.aof_delayed_fsync++; //记录推迟fsync的次数
            redisLog(REDIS_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }

    server.aof_flush_postponed_start = 0;
       //写入aof文件
    nwritten = write(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    if (nwritten != (signed)sdslen(server.aof_buf)) {

        static time_t last_write_error_log = 0;
        int can_log = 0;

        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        /* Lof the AOF write error and record the error code. */
        if (nwritten == -1) {
            if (can_log) {
                redisLog(REDIS_WARNING,"Error writing to the AOF file: %s",
                    strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            if (can_log) {
                redisLog(REDIS_WARNING,"Short write while writing to "
                                       "the AOF file: (nwritten=%lld, "
                                       "expected=%lld)",
                                       (long long)nwritten,
                                       (long long)sdslen(server.aof_buf));
            }

            // 尝试移除新追加的不完整内容
            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    redisLog(REDIS_WARNING, "Could not remove short write "
                             "from the append-only file.  Redis may refuse "
                             "to load the AOF the next time it starts.  "
                             "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftrunacate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        // 处理写入 AOF 文件时出现的错误
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synched on disk. */
            redisLog(REDIS_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = REDIS_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        // 写入成功，更新最后写入状态
        if (server.aof_last_write_status == REDIS_ERR) {
            redisLog(REDIS_WARNING,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = REDIS_OK;
        }
    }

    // 更新写入后的 AOF 文件大小
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). 
     *
     * 如果 AOF 缓存的大小足够小的话，那么重用这个缓存，
     * 否则的话，释放 AOF 缓存。
     */
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        // 清空缓存中的内容，等待重用
        sdsclear(server.aof_buf);
    } else {
        // 释放缓存
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. 
     *
     * 如果 no-appendfsync-on-rewrite 选项为开启状态，
     * 并且有 BGSAVE 或者 BGREWRITEAOF 正在进行的话，
     * 那么不执行 fsync 
     */
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
            return;

    // 总是执行 fsnyc
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* aof_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        aof_fsync(server.aof_fd); /* Let's try to get this data on the disk */

        // 更新最后一次执行 fsnyc 的时间
        server.aof_last_fsync = server.unixtime;

    // 策略为每秒 fsnyc ，并且距离上次 fsync 已经超过 1 秒
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        // 放到后台执行
        if (!sync_in_progress) aof_background_fsync(server.aof_fd);
        // 更新最后一次执行 fsync 的时间
        server.aof_last_fsync = server.unixtime;
    }
}
```

# 复制

如果在主从复制架构中出现宕机的情况，需要分情况看:

 1、 从 Redis 宕机

从库重新启动后会自动加入到主从架构中，自动完成同步数据;

 在从库在断开期间，如果主库的变化不大，从库再次启动后，做增量同步，否则全量同步
 2、 主 Redis 宕机
 第一步，在从数据库中执行 SLAVEOF NO ONE 命令，断开主从关系并且提升为主库继续服务; 

第二步，将主库重新启动后，执行 SLAVEOF 命令，将其设置为其他库的从库

哨兵(sentinel)自动完成上述两步，实现主从切换

## 执行salveof命令

```c
void slaveofCommand(redisClient *c) {

​    //集群模式中禁用salveof使用
​    if (server.cluster_enabled) {
​        addReplyError(c,"SLAVEOF not allowed in cluster mode.");
​        return;
​    }

​    // SLAVEOF NO ONE 让从服务器转为主服务器
​    if (!strcasecmp(c->argv[1]->ptr,"no") &&
​        !strcasecmp(c->argv[2]->ptr,"one")) {
​        if (server.masterhost) {
​            // 让服务器取消复制，成为主服务器
​            replicationUnsetMaster();
​            redisLog(REDIS_NOTICE,"MASTER MODE enabled (user request)");
​        }
​    } else {
​        long port;
​        // 获取端口参数
​        if ((getLongFromObjectOrReply(c, c->argv[2], &port, NULL) != REDIS_OK))
​            return;

​        // 检查输入的 host 和 port 是否服务器目前的主服务器
​        // 如果是的话，向客户端返回 +OK ，不做其他动作
​        if (server.masterhost && !strcasecmp(server.masterhost,c->argv[1]->ptr)
​            && server.masterport == port) {
​            redisLog(REDIS_NOTICE,"SLAVE OF would result into synchronization with the master we are already connected with. No operation performed.");
​            addReplySds(c,sdsnew("+OK Already connected to specified master\r\n"));
​            return;
​        }

​       // 将服务器设为指定地址的从服务器
​        replicationSetMaster(c->argv[1]->ptr, port);
​        redisLog(REDIS_NOTICE,"SLAVE OF %s:%d enabled (user request)",
​            server.masterhost, server.masterport);
​    }
​    addReply(c,shared.ok);
}
```

### 设置服务器为指定地址的从服务器

```c
void replicationSetMaster(char *ip, int port) {
    // 清除原有的主服务器地址（如果有的话）
    sdsfree(server.masterhost);
    // IP
    server.masterhost = sdsnew(ip);
    // 端口
    server.masterport = port;
    // 如果之前有其他地址，那么释放它
    if (server.master) freeClient(server.master);
    // 断开所有从服务器的连接，强制所有从服务器执行重同步
    disconnectSlaves(); /* Force our slaves to resync with us as well. */
    // 清空可能有的 master 缓存，因为已经不会执行 PSYNC 了
    replicationDiscardCachedMaster(); /* Don't try a PSYNC. */
    // 释放 backlog ，同理， PSYNC 目前已经不会执行了
    freeReplicationBacklog(); /* Don't allow our chained slaves to PSYNC. */
    // 取消之前的复制进程（如果有的话）
    cancelReplicationHandshake();
    // 进入连接状态
    server.repl_state = REDIS_REPL_CONNECT; 
    server.master_repl_offset = 0;
    server.repl_down_since = 0;
}
```

## 复制的时间事件

```c
void replicationCron(void) { //每秒调用一次

    // 尝试连接到主服务器，但超时
    if (server.masterhost &&
        (server.repl_state == REDIS_REPL_CONNECTING ||
         server.repl_state == REDIS_REPL_RECEIVE_PONG) &&
        (time(NULL)-server.repl_transfer_lastio) > server.repl_timeout)
    {
        redisLog(REDIS_WARNING,"Timeout connecting to the MASTER...");
        // 取消连接
        undoConnectWithMaster();
    }

    // RDB 文件的传送已超时
    if (server.masterhost && server.repl_state == REDIS_REPL_TRANSFER &&
        (time(NULL)-server.repl_transfer_lastio) > server.repl_timeout)
    {
        redisLog(REDIS_WARNING,"Timeout receiving bulk data from MASTER... If the problem persists try to set the 'repl-timeout' parameter in redis.conf to a larger value.");
        // 停止传送，并删除临时文件
        replicationAbortSyncTransfer();
    }

    // 从服务器曾经连接上主服务器，但现在超时
    if (server.masterhost && server.repl_state == REDIS_REPL_CONNECTED &&
        (time(NULL)-server.master->lastinteraction) > server.repl_timeout)
    {
        redisLog(REDIS_WARNING,"MASTER timeout: no data nor PING received...");
        // 释放主服务器
        freeClient(server.master);
    }

    // 尝试连接主服务器
    if (server.repl_state == REDIS_REPL_CONNECT) {
        redisLog(REDIS_NOTICE,"Connecting to MASTER %s:%d",
            server.masterhost, server.masterport);
        if (connectWithMaster() == REDIS_OK) {
            redisLog(REDIS_NOTICE,"MASTER <-> SLAVE sync started");
        }
    }

    // 向主服务器发送 ACK 命令
    if (server.masterhost && server.master &&
        !(server.master->flags & REDIS_PRE_PSYNC))
        replicationSendAck();
    
     // 如果服务器有从服务器，定时向它们发送 PING 。
     //这样从服务器就可以实现显式的 master 超时判断机制，
     //即使 TCP 连接未断开也是如此。
    if (!(server.cronloops % (server.repl_ping_slave_period * server.hz))) {
        listIter li;
        listNode *ln;
        robj *ping_argv[1];

        /* First, send PING */
        // 向所有已连接 slave （状态为 ONLINE）发送 PING
        ping_argv[0] = createStringObject("PING",4);
        replicationFeedSlaves(server.slaves, server.slaveseldb, ping_argv, 1);
        decrRefCount(ping_argv[0]);

        /* Second, send a newline to all the slaves in pre-synchronization
         * stage, that is, slaves waiting for the master to create the RDB file.
         *
         * 向那些正在等待 RDB 文件的从服务器（状态为 BGSAVE_START 或 BGSAVE_END）
         * 发送 "\n"
         *
         * The newline will be ignored by the slave but will refresh the
         * last-io timer preventing a timeout. 
         *
         * 这个 "\n" 会被从服务器忽略，
         * 它的作用就是用来防止主服务器因为长期不发送信息而被从服务器误判为超时
         */
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            redisClient *slave = ln->value;

            if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START ||
                slave->replstate == REDIS_REPL_WAIT_BGSAVE_END) {
                if (write(slave->fd, "\n", 1) == -1) {
                    /* Don't worry, it's just a ping. */
                }
            }
        }
    }

    // 断开超时从服务器
    if (listLength(server.slaves)) {
        listIter li;
        listNode *ln;
        // 遍历所有从服务器
        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            redisClient *slave = ln->value;
            // 略过未 ONLINE 的从服务器
            if (slave->replstate != REDIS_REPL_ONLINE) continue;
            if (slave->flags & REDIS_PRE_PSYNC) continue;
            // 释放超时从服务器
            if ((server.unixtime - slave->repl_ack_time) > server.repl_timeout)
            {
                char ip[REDIS_IP_STR_LEN];
                int port;
                if (anetPeerToString(slave->fd,ip,sizeof(ip),&port) != -1) {
                    redisLog(REDIS_WARNING,
                        "Disconnecting timedout slave: %s:%d",
                        ip, slave->slave_listening_port);
                }
                // 释放
                freeClient(slave);
            }
        }
    }

    if (listLength(server.slaves) == 0 && server.repl_backlog_time_limit &&
        server.repl_backlog)
    {
        time_t idle = server.unixtime - server.repl_no_slaves_since;

        if (idle > server.repl_backlog_time_limit) {
            // 释放
            freeReplicationBacklog();
            redisLog(REDIS_NOTICE,
                "Replication backlog freed after %d seconds "
                "without connected slaves.",
                (int) server.repl_backlog_time_limit);
        }
    }

    if (listLength(server.slaves) == 0 &&
        server.aof_state == REDIS_AOF_OFF &&
        listLength(server.repl_scriptcache_fifo) != 0)
    {
        replicationScriptCacheFlush();
    }

    // 更新同步状态良好的从服务器的数量
    refreshGoodSlavesCount();
}
```

### 更新同步状态良好的从服务器的数量

```c
void refreshGoodSlavesCount(void) {
    listIter li;
    listNode *ln;
    int good = 0;
		
    if (!server.repl_min_slaves_to_write || 
        !server.repl_min_slaves_max_lag) return;

    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        redisClient *slave = ln->value;
        // 计算延迟值
        time_t lag = server.unixtime - slave->repl_ack_time;
				//计算同步状态良好的从服务器数量
        if (slave->replstate == REDIS_REPL_ONLINE &&
            lag <= server.repl_min_slaves_max_lag) good++;
    }

    // 更新状态良好的从服务器数量，如果数量少，有可能主服务器发生了问题，禁止主服务器的写入操作
    server.repl_good_slaves_count = good;
}
```



##  syncCommand

```c
void syncCommand(redisClient *c) {//sync psync

			//如果是salve直接返回
​    if (c->flags & REDIS_SLAVE) return;
​    if (server.masterhost && server.repl_state != REDIS_REPL_CONNECTED) {
​        addReplyError(c,"Can't SYNC while not connected with my master");
​        return;
​    }

​    // 在客户端仍有输出数据等待输出，不能SYNC
​    if (listLength(c->reply) != 0 || c->bufpos != 0) {
​        addReplyError(c,"SYNC and PSYNC are invalid with pending output");
​        return;
​    }

​    redisLog(REDIS_NOTICE,"Slave asks for synchronization");

​     //+FULLRESYNC <runid> <offset>
​    if (!strcasecmp(c->argv[0]->ptr,"psync")) {
​        // 增量同步返回REDIS_OK
​        if (masterTryPartialResynchronization(c) == REDIS_OK) {//增量同步
​            // 可执行PSYNC
​            server.stat_sync_partial_ok++;
​            return;
​        } else { //全量同步
​            // 不可执行 PSYNC
​            char *master_runid = c->argv[1]->ptr;
​            if (master_runid[0] != '?') server.stat_sync_partial_err++;
​        }
​    } else {
​        c->flags |= REDIS_PRE_PSYNC;
​    }

​    server.stat_sync_full++; //全量同步计数

​    // 检查是否有 BGSAVE 在执行
​    if (server.rdb_child_pid != -1) {
​        redisClient *slave;
​        listNode *ln;
​        listIter li;
​        // 如果有至少一个 slave 在等待这个 BGSAVE 完成
​        // 那么说明正在进行的 BGSAVE 所产生的 RDB 也可以为其他 slave 所用
​        listRewind(server.slaves,&li);
​        while((ln = listNext(&li))) {
​            slave = ln->value;
​            if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_END) break;
​        }

​        if (ln) { //  可以使用目前 BGSAVE 所生成的 RDB
​            copyClientOutputBuffer(c,slave);
​            c->replstate = REDIS_REPL_WAIT_BGSAVE_END;
​            redisLog(REDIS_NOTICE,"Waiting for end of BGSAVE for SYNC");
​        } else { //等待bgsave开始
​            c->replstate = REDIS_REPL_WAIT_BGSAVE_START;
​            redisLog(REDIS_NOTICE,"Waiting for next BGSAVE for SYNC");
​        }

​    } else {
​        // 没有 BGSAVE 在进行，开始一个新的 BGSAVE
​        redisLog(REDIS_NOTICE,"Starting BGSAVE for SYNC");
​        if (rdbSaveBackground(server.rdb_filename) != REDIS_OK) {
​            redisLog(REDIS_NOTICE,"Replication failed, can't BGSAVE");
​            addReplyError(c,"Unable to perform background save");
​            return;
​        }
​        // 设置状态
​        c->replstate = REDIS_REPL_WAIT_BGSAVE_END;
​        // 因为新 slave 进入，刷新复制脚本缓存
​        replicationScriptCacheFlush();
​    }
​    if (server.repl_disable_tcp_nodelay)
​        anetDisableTcpNoDelay(NULL, c->fd); /* Non critical if it fails. */
​    c->repldbfd = -1;
​    c->flags |= REDIS_SLAVE;
​    server.slaveseldb = -1; /* Force to re-emit the SELECT command. */
​    // 添加到 slave 列表中
​    listAddNodeTail(server.slaves,c);
​    // 避免给每个从节点开辟一块缓冲区，造成内存资源浪费。
​    if (listLength(server.slaves) == 1 && server.repl_backlog == NULL)
​        createReplicationBacklog();
​    return;
}
```

### 复制缓冲区

#### 创建缓冲区

```c
void createReplicationBacklog(void) {
    redisAssert(server.repl_backlog == NULL);
    // repl_backlog_size，循环缓冲区本身的总长度
    server.repl_backlog = zmalloc(server.repl_backlog_size);
    // 循环缓冲区中目前累积的数据的长度
    server.repl_backlog_histlen = 0;
    //循环缓冲区的写指针
    server.repl_backlog_idx = 0;
    server.master_repl_offset++;
  //循环缓冲区中最早保存的数据的首字节，在全局范围内的偏移值
    server.repl_backlog_off = server.master_repl_offset+1;
}
```

#### 填充缓冲区

```c
void feedReplicationBacklog(void *ptr, size_t len) {
    unsigned char *p = ptr;
    //master全局offset
    server.master_repl_offset += len;
    while(len) {
        //缓冲区剩余可用空间
        size_t thislen = server.repl_backlog_size - server.repl_backlog_idx;
        if (thislen > len) thislen = len; //计算当前可写入的大小
        // 将p中的thislen内容复制到缓冲区
        memcpy(server.repl_backlog+server.repl_backlog_idx,p,thislen);
        //更新写位置
        server.repl_backlog_idx += thislen;
        if (server.repl_backlog_idx == server.repl_backlog_size)
            server.repl_backlog_idx = 0; //缓冲区填满
        len -= thislen;//减去已写入的字节数，得到未写入的内容
        p += thislen; // 未写入内容
        server.repl_backlog_histlen += thislen; //实际长度
    }
    if (server.repl_backlog_histlen > server.repl_backlog_size)
        server.repl_backlog_histlen = server.repl_backlog_size;
   //缓冲区的开始offset
    server.repl_backlog_off = server.master_repl_offset -
                              server.repl_backlog_histlen + 1;
}
```

#### 读取缓冲区

```c
long long addReplyReplicationBacklog(redisClient *c, long long offset) {
    long long j, skip, len;
    redisLog(REDIS_DEBUG, "[PSYNC] Slave request offset: %lld", offset);

    if (server.repl_backlog_histlen == 0) { //缓冲区无数据
        redisLog(REDIS_DEBUG, "[PSYNC] Backlog history len is zero");
        return 0;
    }

    redisLog(REDIS_DEBUG, "[PSYNC] Backlog size: %lld",
             server.repl_backlog_size);
    redisLog(REDIS_DEBUG, "[PSYNC] First byte: %lld",
             server.repl_backlog_off);
    redisLog(REDIS_DEBUG, "[PSYNC] History len: %lld",
             server.repl_backlog_histlen);
    redisLog(REDIS_DEBUG, "[PSYNC] Current index: %lld",
             server.repl_backlog_idx);

    skip = offset - server.repl_backlog_off; //跳过缓冲区的内容
    redisLog(REDIS_DEBUG, "[PSYNC] Skipping: %lld", skip);

    /* Point j to the oldest byte, that is actaully our
     * server.repl_backlog_off byte. */
    j = (server.repl_backlog_idx +
        (server.repl_backlog_size-server.repl_backlog_histlen)) %
        server.repl_backlog_size;
    redisLog(REDIS_DEBUG, "[PSYNC] Index of first byte: %lld", j);

    /* Discard the amount of data to seek to the specified 'offset'. */
    j = (j + skip) % server.repl_backlog_size;

    /* Feed slave with data. Since it is a circular buffer we have to
     * split the reply in two parts if we are cross-boundary. */
    len = server.repl_backlog_histlen - skip;
    redisLog(REDIS_DEBUG, "[PSYNC] Reply total length: %lld", len);
    while(len) {
        long long thislen =
            ((server.repl_backlog_size - j) < len) ?
            (server.repl_backlog_size - j) : len;

        redisLog(REDIS_DEBUG, "[PSYNC] addReply() length: %lld", thislen);
        addReplySds(c,sdsnewlen(server.repl_backlog + j, thislen));
        len -= thislen;
        j = 0;
    }
    return server.repl_backlog_histlen - skip;
}
```



## masterTryPartialResynchronization

```c
int masterTryPartialResynchronization(redisClient *c) {//尝试增量同步（runId、offset），将repl_backlog中的数据同步给从库

​    long long psync_offset, psync_len;
​    char *master_runid = c->argv[1]->ptr; //请求中的runId
​    char buf[128];
​    int buflen;


		//从服务器传来的runId为？或者runId与master的runId不同，则开启全量同步
​    if (strcasecmp(master_runid, server.runid)) {
​        if (master_runid[0] != '?') {
​            redisLog(REDIS_NOTICE,"Partial resynchronization not accepted: "
​                "Runid mismatch (Client asked for runid '%s', my runid is '%s')",
​                master_runid, server.runid);
​        } else {
​            redisLog(REDIS_NOTICE,"Full resync requested by slave.");
​        }
​        // 需要全量同步
​        goto need_full_resync;
​    }


​    // 取出 psync_offset 参数
​    if (getLongLongFromObjectOrReply(c,c->argv[2],&psync_offset,NULL) !=
​       REDIS_OK) goto need_full_resync;

​        
​    if (!server.repl_backlog ||// 如果没有 backlog
​        // 或者 psync_offset 小于 server.repl_backlog_off
​        // （想要恢复的那部分数据已经被覆盖）
​        psync_offset < server.repl_backlog_off ||
​        // psync offset 大于 backlog 所保存的数据的偏移量
​        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
​    {
​        // 执行 FULL RESYNC
​        redisLog(REDIS_NOTICE,
​            "Unable to partial resync with the slave for lack of backlog (Slave request was: %lld).", psync_offset);
​        if (psync_offset > server.master_repl_offset) {
​            redisLog(REDIS_WARNING,
​                "Warning: slave tried to PSYNC with an offset that is greater than the master replication offset.");
​        }
​        goto need_full_resync;
​    }

		//增量同步
​    c->flags |= REDIS_SLAVE;
​    c->replstate = REDIS_REPL_ONLINE;
​    c->repl_ack_time = server.unixtime;
​    listAddNodeTail(server.slaves,c);
​    // 向从服务器发送一个同步 +CONTINUE ，表示 PSYNC 可以执行
​    buflen = snprintf(buf,sizeof(buf),"+CONTINUE\r\n");
​    if (write(c->fd,buf,buflen) != buflen) {
​        freeClientAsync(c);
​        return REDIS_OK;
​    }
​    // 发送 backlog 中的内容（也即是从服务器缺失的那些内容）到从服务器
​    psync_len = addReplyReplicationBacklog(c,psync_offset);
​    redisLog(REDIS_NOTICE,
​        "Partial resynchronization request accepted. Sending %lld bytes of backlog starting from offset %lld.", psync_len, psync_offset);
​    // 刷新低延迟从服务器的数量
​    refreshGoodSlavesCount();
​    return REDIS_OK; /* The caller can return, no full resync needed. */


//需要全量同步
need_full_resync:
​    // 刷新 psync_offset
​    psync_offset = server.master_repl_offset;
​    // 刷新 psync_offset
​    if (server.repl_backlog == NULL) psync_offset++;
​    // 发送 +FULLRESYNC ，表示需要完整重同步
​    buflen = snprintf(buf,sizeof(buf),"+FULLRESYNC %s %lld\r\n",
​                      server.runid,psync_offset);
​    if (write(c->fd,buf,buflen) != buflen) {
​        freeClientAsync(c);
​        return REDIS_OK;
​    }
​    return REDIS_ERR;
}
```

# 哨兵机制

## 初始化

```c
 if (server.sentinel_mode) {
   		initSentinelConfig(); //配置
      initSentinel(); //Sentinel状态属性
   } 
```



```c
void sentinelIsRunning(void) {
  //打印哨兵Id，每次运行都是不同的
    redisLog(REDIS_WARNING,"Sentinel runid is %s", server.runid);

    // Sentinel 不能在没有配置文件的情况下执行
    if (server.configfile == NULL) {
        redisLog(REDIS_WARNING,
            "Sentinel started without a config file. Exiting...");
        exit(1);
    } else if (access(server.configfile,W_OK) == -1) {
        redisLog(REDIS_WARNING,
            "Sentinel config file %s is not writable: %s. Exiting...",
            server.configfile,strerror(errno));
        exit(1);
    }

    //对每个Master产生+monitor事件
    sentinelGenerateInitialMonitorEvents();
}
```



```c
void sentinelGenerateInitialMonitorEvents(void) {
    dictIterator *di;
    dictEntry *de;

    di = dictGetIterator(sentinel.masters);
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);
        sentinelEvent(REDIS_WARNING,"+monitor",ri,"%@ quorum %d",ri->quorum);
    }
    dictReleaseIterator(di);
}
```

## sentinel定时器

```c
void sentinelTimer(void) {

    //判断是否需要进入TITL模式
    sentinelCheckTiltCondition();

    // 执行定期操作
    // 比如 PING 实例、分析主服务器和从服务器的 INFO 命令
    // 向其他监视相同主服务器的 sentinel 发送问候信息
    // 并接收其他 sentinel 发来的问候信息
    // 执行故障转移操作，等等
    sentinelHandleDictOfRedisInstances(sentinel.masters);

    sentinelRunPendingScripts();

    sentinelCollectTerminatedScripts();
  
    sentinelKillTimedoutScripts();

    server.hz = REDIS_DEFAULT_HZ + rand() % REDIS_DEFAULT_HZ;
}
```

### 是否进入TILT模式

```c
void sentinelCheckTiltCondition(void) {
 		//TILT模式下的sentinel仍然会进行监控并收集信息，但是不执行故障转移、下线判断之类的操作
    //TILT模式主要是为了防止sentinel自身的原因导致出现对实例下线的误判
     mstime_t now = mstime();
    //计算上次运行sentinel和当前时间的差
    mstime_t delta = now - sentinel.previous_time;
    // 如果差为负数，或者大于 2 秒钟，那么进入 TILT 模式
    if (delta < 0 || delta > SENTINEL_TILT_TRIGGER) {
        // 打开标记
        sentinel.tilt = 1;
        // 记录进入 TILT 模式的开始时间
        sentinel.tilt_start_time = mstime();
        // 打印事件
        sentinelEvent(REDIS_WARNING,"+tilt",NULL,"#tilt mode entered");
    }
    // 更新最后一次 sentinel 运行时间
    sentinel.previous_time = mstime();
}
```

### 执行调度操作

```c
void sentinelHandleDictOfRedisInstances(dict *instances) {
    dictIterator *di;
    dictEntry *de;
    sentinelRedisInstance *switch_to_promoted = NULL;

    //遍历实例(主服务器、从服务器或者sentinel)
    di = dictGetIterator(instances);
    while((de = dictNext(di)) != NULL) {
        // 取出实例对应的实例结构
        sentinelRedisInstance *ri = dictGetVal(de);
        // 执行调度操作
        sentinelHandleRedisInstance(ri);
        // 如果被遍历的是主服务器，那么递归地遍历该主服务器的所有从服务器
        // 以及所有 sentinel
        if (ri->flags & SRI_MASTER) {
            // 所有从服务器
            sentinelHandleDictOfRedisInstances(ri->slaves);
            // 所有 sentinel
            sentinelHandleDictOfRedisInstances(ri->sentinels);
            // 对已下线主服务器（ri）的故障迁移已经完成
            // ri 的所有从服务器都已经同步到新主服务器
            if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {
                // 已选出新的主服务器
                switch_to_promoted = ri;
            }
        }
    }

    // 将原主服务器（已下线）从主服务器表格中移除，并使用新主服务器代替它
    if (switch_to_promoted)
        sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);

    dictReleaseIterator(di);
}
```

#### sentinelHandleRedisInstance

```c
void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {

    //创建连接
    sentinelReconnectInstance(ri);

    // 周期性向实例发送 PING、 INFO 或者 PUBLISH 命令
    sentinelSendPeriodicCommands(ri);

    // 如果 Sentinel 处于 TILT 模式，那么不执行故障检测。
    if (sentinel.tilt) {
        if (mstime()-sentinel.tilt_start_time < SENTINEL_TILT_PERIOD) 
          	return;
        // 时间已过，退出TILT模式
        sentinel.tilt = 0;
        sentinelEvent(REDIS_WARNING,"-tilt",NULL,"#tilt mode exited");
    }

    // 检查给定实例是否进入 SDOWN 状态
    sentinelCheckSubjectivelyDown(ri);

    if (ri->flags & (SRI_MASTER|SRI_SLAVE)) {
    }

    /* 对主服务器进行处理 */
    if (ri->flags & SRI_MASTER) {
        // 判断 master 是否进入 ODOWN 状态
        sentinelCheckObjectivelyDown(ri);
        // 如果主服务器进入了 ODOWN 状态，那么开始一次故障转移操作
        if (sentinelStartFailoverIfNeeded(ri))
            // 强制向其他 Sentinel 发送 SENTINEL is-master-down-by-addr 命令
            // 刷新其他 Sentinel 关于主服务器的状态
            sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);

        // 执行故障转移
        sentinelFailoverStateMachine(ri);

        // 如果有需要的话，向其他 Sentinel 发送 SENTINEL is-master-down-by-addr 命令
        // 刷新其他 Sentinel 关于主服务器的状态
        // 这一句是对那些没有进入 if(sentinelStartFailoverIfNeeded(ri)) { /* ... */ }
        // 语句的主服务器使用的
        sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
    }
}
```

### 当前哨兵检测实例是否下线

```c
void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {//SRI_S_DOWN状态

    mstime_t elapsed = 0;

    if (ri->last_ping_time)
        elapsed = mstime() - ri->last_ping_time;

     //如果检测到连接的活跃度很低，那么考虑重断开连接，并进行重连
    if (ri->cc &&
        (mstime() - ri->cc_conn_time) > SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
        ri->last_ping_time != 0 && /* Ther is a pending ping... */
        /* The pending ping is delayed, and we did not received
         * error replies as well. */
        (mstime() - ri->last_ping_time) > (ri->down_after_period/2) &&
        (mstime() - ri->last_pong_time) > (ri->down_after_period/2))
    {
        sentinelKillLink(ri,ri->cc);
    }

    // 考虑断开实例的 pc 连接
    if (ri->pc &&
        (mstime() - ri->pc_conn_time) > SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
        (mstime() - ri->pc_last_activity) > (SENTINEL_PUBLISH_PERIOD*3))
    {
        sentinelKillLink(ri,ri->pc);
    }

    /* 1) It is not replying.
     * 2) We believe it is a master, it reports to be a slave for enough time
     *    to meet the down_after_period, plus enough time to get two times
     *    INFO report from the instance. 
     */
    if (elapsed > ri->down_after_period ||
        (ri->flags & SRI_MASTER &&
         ri->role_reported == SRI_SLAVE &&
         mstime() - ri->role_reported_time >
          (ri->down_after_period+SENTINEL_INFO_PERIOD*2)))
    {
        if ((ri->flags & SRI_S_DOWN) == 0) {
            // 发送事件
            sentinelEvent(REDIS_WARNING,"+sdown",ri,"%@");
            // 记录进入 SDOWN 状态的时间
            ri->s_down_since_time = mstime();
            // 打开 SDOWN 标志
            ri->flags |= SRI_S_DOWN;
        }
    } else {
        // 移除（可能有的） SDOWN 状态
        if (ri->flags & SRI_S_DOWN) {
            // 发送事件
            sentinelEvent(REDIS_WARNING,"-sdown",ri,"%@");
            // 移除相关标志
            ri->flags &= ~(SRI_S_DOWN|SRI_SCRIPT_KILL_SENT);
        }
    }
}
```

### 其他哨兵是否任务实例下线

```c
void sentinelCheckObjectivelyDown(sentinelRedisInstance *master) {//SRI_O_DOWN状态
    dictIterator *di;
    dictEntry *de;
    int quorum = 0, odown = 0;

    // 如果当前 Sentinel 将主服务器判断为主观下线
    // 那么检查是否有其他 Sentinel 同意这一判断
    // 当同意的数量足够时，将主服务器判断为客观下线
    if (master->flags & SRI_S_DOWN) {

        // 统计同意的 Sentinel 数量（起始的1代表本 Sentinel）
        quorum = 1; /* the current sentinel. */

        // 统计其他认为 master 进入下线状态的 Sentinel 的数量
        di = dictGetIterator(master->sentinels);
        while((de = dictNext(di)) != NULL) {
            sentinelRedisInstance *ri = dictGetVal(de);
            // 该 SENTINEL 也认为 master 已下线
            if (ri->flags & SRI_MASTER_DOWN) quorum++;
        }
        dictReleaseIterator(di);
        // 如果投票得出的支持数目大于等于判断ODOWN所需的票数
        // 那么进入 ODOWN 状态
        if (quorum >= master->quorum) odown = 1;
    }

    if (odown) {
        // master 已 ODOWN
        if ((master->flags & SRI_O_DOWN) == 0) {
            // 发送事件
            sentinelEvent(REDIS_WARNING,"+odown",master,"%@ #quorum %d/%d",
                quorum, master->quorum);
            // 打开 ODOWN 标志
            master->flags |= SRI_O_DOWN;
            // 记录进入 ODOWN 的时间
            master->o_down_since_time = mstime();
        }
    } else {
        // 未进入 ODOWN
        if (master->flags & SRI_O_DOWN) {
            // 发送事件
            sentinelEvent(REDIS_WARNING,"-odown",master,"%@");
            // 移除 ODOWN 标志
            master->flags &= ~SRI_O_DOWN;
        }
    }
}
```

### 故障转移

```c
int sentinelStartFailoverIfNeeded(sentinelRedisInstance *master) {

    if (!(master->flags & SRI_O_DOWN)) return 0; //未处于O_DOWN状态

    if (master->flags & SRI_FAILOVER_IN_PROGRESS) return 0; //正处于转移中

    //避免两次执行故障转移的时间比较近
    if (mstime() - master->failover_start_time <
        master->failover_timeout*2)
    {
        if (master->failover_delay_logged != master->failover_start_time) {
            time_t clock = (master->failover_start_time +
                            master->failover_timeout*2) / 1000;
            char ctimebuf[26];
            ctime_r(&clock,ctimebuf);
            ctimebuf[24] = '\0'; /* Remove newline. */
            master->failover_delay_logged = master->failover_start_time;
            redisLog(REDIS_WARNING,
                "Next failover delay: I will not start a failover before %s",
                ctimebuf);
        }
        return 0;
    }
    // 开始一次故障转移
    sentinelStartFailover(master);
    return 1;
}
```

SENTINEL_FAILOVER_STATE_WAIT_START



```c
void sentinelStartFailover(sentinelRedisInstance *master) {
    redisAssert(master->flags & SRI_MASTER); //必须时master

    // 更新故障转移状态
    master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;

    // 更新主服务器状态
    master->flags |= SRI_FAILOVER_IN_PROGRESS;

    // 更新纪元,每次发生故障转移都会加1
    master->failover_epoch = ++sentinel.current_epoch;

   //发布事件
    sentinelEvent(REDIS_WARNING,"+new-epoch",master,"%llu",
        (unsigned long long) sentinel.current_epoch);
    sentinelEvent(REDIS_WARNING,"+try-failover",master,"%@");

    // 记录故障转移状态的变更时间
    master->failover_start_time = mstime()+rand()%SENTINEL_MAX_DESYNC;
    master->failover_state_change_time = mstime();
}
```

向其他哨兵发送命令

```c
void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
    dictIterator *di;
    dictEntry *de;

    // 遍历正在监视相同 master 的所有 sentinel
    // 向它们发送 SENTINEL is-master-down-by-addr 命令
    di = dictGetIterator(master->sentinels);
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);

        // 距离该 sentinel 最后一次回复 SENTINEL master-down-by-addr 命令已经过了多久
        mstime_t elapsed = mstime() - ri->last_master_down_reply_time;

        char port[32];
        int retval;

        /* If the master state from other sentinel is too old, we clear it. */
        if (elapsed > SENTINEL_ASK_PERIOD*5) {
            ri->flags &= ~SRI_MASTER_DOWN;
            sdsfree(ri->leader);
            ri->leader = NULL;
        }

        /* Only ask if master is down to other sentinels if:
         *
         *
         * 1) We believe it is down, or there is a failover in progress.
         * 2) Sentinel is connected.
         * 3) We did not received the info within SENTINEL_ASK_PERIOD ms. 
         */
        if ((master->flags & SRI_S_DOWN) == 0) continue;
        if (ri->flags & SRI_DISCONNECTED) continue;
        if (!(flags & SENTINEL_ASK_FORCED) &&
            mstime() - ri->last_master_down_reply_time < SENTINEL_ASK_PERIOD)
            continue;

        // 发送 SENTINEL is-master-down-by-addr 命令
        ll2string(port,sizeof(port),master->addr->port);
        retval = redisAsyncCommand(ri->cc,
                    sentinelReceiveIsMasterDownReply, NULL,
                    "SENTINEL is-master-down-by-addr %s %s %llu %s",
                    master->addr->ip, port,
                    sentinel.current_epoch,
                    // 如果本 Sentinel 已经检测到 master 进入 ODOWN 
                    // 并且要开始一次故障转移，那么向其他 Sentinel 发送自己的运行 ID
                    // 让对方将给自己投一票（如果对方在这个纪元内还没有投票的话）
                    (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?
                    server.runid : "*");//返回投票信息
        if (retval == REDIS_OK) ri->pending_commands++;
    }
    dictReleaseIterator(di);
}
```

### 处理IS-MASTER-DOWN-BY-ADDR请求

```c
 /* SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>*/
        sentinelRedisInstance *ri;
        long long req_epoch;
        uint64_t leader_epoch = 0;
        char *leader = NULL;
        long port;
        int isdown = 0;

        if (c->argc != 6) goto numargserr;
        if (getLongFromObjectOrReply(c,c->argv[3],&port,NULL) != REDIS_OK ||
            getLongLongFromObjectOrReply(c,c->argv[4],&req_epoch,NULL)
                                                              != REDIS_OK)
            return;
        ri = getSentinelRedisInstanceByAddrAndRunID(sentinel.masters,
            c->argv[2]->ptr,port,NULL);

         
        if (!sentinel.tilt && ri && (ri->flags & SRI_S_DOWN) &&
                                    (ri->flags & SRI_MASTER))
          //没有处于TILT模式
            isdown = 1;

        /* Vote for the master (or fetch the previous vote) if the request
         * includes a runid, otherwise the sender is not seeking for a vote. */
        if (ri && ri->flags & SRI_MASTER && strcasecmp(c->argv[5]->ptr,"*")) {
            leader = sentinelVoteLeader(ri,(uint64_t)req_epoch,
                                            c->argv[5]->ptr,
                                            &leader_epoch);
        }

        /* Reply with a three-elements multi-bulk reply:
         * down state, leader, vote epoch. */
        // 多条回复
        // 1) <down_state>    1 代表下线， 0 代表未下线
        // 2) <leader_runid>  Sentinel 选举作为领头 Sentinel 的运行 ID
        // 3) <leader_epoch>  领头 Sentinel 目前的配置纪元
        addReplyMultiBulkLen(c,3);
        addReply(c, isdown ? shared.cone : shared.czero);
        addReplyBulkCString(c, leader ? leader : "*");
        addReplyLongLong(c, (long long)leader_epoch);
        if (leader) sdsfree(leader);
```

### 执行故障转移

SENTINEL_FAILOVER_STATE_WAIT_START -> 

SENTINEL_FAILOVER_STATE_SELECT_SLAVE ->

SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE->

SENTINEL_FAILOVER_STATE_WAIT_PROMOTION->

SENTINEL_FAILOVER_STATE_RECONF_SLAVES

```c
void sentinelFailoverStateMachine(sentinelRedisInstance *ri) {
    redisAssert(ri->flags & SRI_MASTER);

    // master 未进入故障转移状态，直接返回
    if (!(ri->flags & SRI_FAILOVER_IN_PROGRESS)) return;

    switch(ri->failover_state) {

        // 等待故障转移开始
        case SENTINEL_FAILOVER_STATE_WAIT_START:
            sentinelFailoverWaitStart(ri);
            break;

        // 选择从服务器
        case SENTINEL_FAILOVER_STATE_SELECT_SLAVE:
            sentinelFailoverSelectSlave(ri);
            break;
        
        // 升级被选中的从服务器为新主服务器
        case SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE:
            sentinelFailoverSendSlaveOfNoOne(ri);
            break;

        // 等待升级生效，如果升级超时，那么重新选择新主服务器
        case SENTINEL_FAILOVER_STATE_WAIT_PROMOTION:
            sentinelFailoverWaitPromotion(ri);
            break;

        // 向从服务器发送 SLAVEOF 命令，让它们同步新主服务器
        case SENTINEL_FAILOVER_STATE_RECONF_SLAVES:
            sentinelFailoverReconfNextSlave(ri);
            break;
    }
}
```

#### 等待转移开始

确保本机sentinel为Leader，修改状态为SENTINEL_FAILOVER_STATE_SELECT_SLAVE

```c
void sentinelFailoverWaitStart(sentinelRedisInstance *ri) {
    char *leader;
    int isleader;

    /* Check if we are the leader for the failover epoch. */
    leader = sentinelGetLeader(ri, ri->failover_epoch);
    isleader = leader && strcasecmp(leader,server.runid) == 0;
    sdsfree(leader);

    /* If I'm not the leader, and it is not a forced failover via
     * SENTINEL FAILOVER, then I can't continue with the failover. */
    if (!isleader && !(ri->flags & SRI_FORCE_FAILOVER)) {
        int election_timeout = SENTINEL_ELECTION_TIMEOUT;

        /* The election timeout is the MIN between SENTINEL_ELECTION_TIMEOUT
         * and the configured failover timeout. */
        if (election_timeout > ri->failover_timeout)
            election_timeout = ri->failover_timeout;

        /* Abort the failover if I'm not the leader after some time. */
        if (mstime() - ri->failover_start_time > election_timeout) {
            sentinelEvent(REDIS_WARNING,"-failover-abort-not-elected",ri,"%@");
            // 取消故障转移
            sentinelAbortFailover(ri);
        }
        return;
    }

    // 本 Sentinel 作为领头，开始执行故障迁移操作...
    sentinelEvent(REDIS_WARNING,"+elected-leader",ri,"%@");
    // 进入选择从服务器状态
    ri->failover_state = SENTINEL_FAILOVER_STATE_SELECT_SLAVE;
    ri->failover_state_change_time = mstime();
    sentinelEvent(REDIS_WARNING,"+failover-state-select-slave",ri,"%@");
}
```

#### 选择从服务器

```c
void sentinelFailoverSelectSlave(sentinelRedisInstance *ri) {

    // 在旧主服务器所属的从服务器中，选择新服务器
    sentinelRedisInstance *slave = sentinelSelectSlave(ri);

    // 没有合适的从服务器，终止故障转移操作
    if (slave == NULL) {
        // 没有可用的从服务器可以提升为新主服务器，故障转移操作无法执行
        sentinelEvent(REDIS_WARNING,"-failover-abort-no-good-slave",ri,"%@");
        // 中止故障转移
        sentinelAbortFailover(ri);
    } else {
        // 成功选定新主服务器
        // 发送事件
        sentinelEvent(REDIS_WARNING,"+selected-slave",slave,"%@");
        // 打开实例的升级标记
        slave->flags |= SRI_PROMOTED;
        // 记录被选中的从服务器
        ri->promoted_slave = slave;
        // 更新故障转移状态
        ri->failover_state = SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE;
        // 更新状态改变时间
        ri->failover_state_change_time = mstime();
        // 发送事件
        sentinelEvent(REDIS_NOTICE,"+failover-state-send-slaveof-noone",
            slave, "%@");
    }
}
```

```c
sentinelRedisInstance *sentinelSelectSlave(sentinelRedisInstance *master) {

    sentinelRedisInstance **instance =
        zmalloc(sizeof(instance[0])*dictSize(master->slaves));
    sentinelRedisInstance *selected = NULL;
    int instances = 0;
    dictIterator *di;
    dictEntry *de;
    mstime_t max_master_down_time = 0;	

    // 计算可以接收的，从服务器与主服务器之间的最大下线时间
    // 这个值可以保证被选中的从服务器的数据库不会太旧
    if (master->flags & SRI_S_DOWN)
        max_master_down_time += mstime() - master->s_down_since_time;
    max_master_down_time += master->down_after_period * 10;

    // 遍历所有从服务器,选择网络连接良好的从服务器
    di = dictGetIterator(master->slaves);
    while((de = dictNext(di)) != NULL) {
        // 从服务器实例
        sentinelRedisInstance *slave = dictGetVal(de);
        mstime_t info_validity_time;
        // 忽略所有 SDOWN 、ODOWN 或者已断线的从服务器
        if (slave->flags & (SRI_S_DOWN|SRI_O_DOWN|SRI_DISCONNECTED)) continue;
        //ping超时
        if (mstime() - slave->last_avail_time > SENTINEL_PING_PERIOD*5) continue;
        if (slave->slave_priority == 0) continue;

        /* If the master is in SDOWN state we get INFO for slaves every second.
         * Otherwise we get it with the usual period so we need to account for
         * a larger delay. */
        if (master->flags & SRI_S_DOWN)
            info_validity_time = SENTINEL_PING_PERIOD*5;
        else
            info_validity_time = SENTINEL_INFO_PERIOD*3;
        // INFO响应已过期
        if (mstime() - slave->info_refresh > info_validity_time) continue;
        if (slave->master_link_down_time > max_master_down_time) continue;
        // 将被选中的 slave 保存到数组中
        instance[instances++] = slave;
    }
    dictReleaseIterator(di);

    if (instances) {
        // 对被选中的从服务器进行排序（优先级、复制进度、runId）
        qsort(instance,instances,sizeof(sentinelRedisInstance*),
            compareSlavesForPromotion);
        // 分值最低的从服务器为被选中服务器
        selected = instance[0];
    }
  
    zfree(instance);
    // 返回被选中的从服务区
    return selected;
}
```

#### 升级从服务器为主

```c
void sentinelFailoverSendSlaveOfNoOne(sentinelRedisInstance *ri) {
    int retval;
    // 如果选中的从服务器断线了，那么在给定的时间内重试
    // 如果给定时间内选中的从服务器也没有上线，那么终止故障迁移操作
    if (ri->promoted_slave->flags & SRI_DISCONNECTED) {
        // 如果超过时限，就不再重试
        if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {
            sentinelEvent(REDIS_WARNING,"-failover-abort-slave-timeout",ri,"%@");
            sentinelAbortFailover(ri);
        }
        return;
    }

    //向被升级的从服务器发送 SLAVEOF NO ONE 命令，将它变为一个主服务器。
    retval = sentinelSendSlaveOf(ri->promoted_slave,NULL,0);
    if (retval != REDIS_OK) return;
    sentinelEvent(REDIS_NOTICE, "+failover-state-wait-promotion",
        ri->promoted_slave,"%@");
    // 更新状态
    // 这个状态会让 Sentinel 等待被选中的从服务器升级为主服务器
    ri->failover_state = SENTINEL_FAILOVER_STATE_WAIT_PROMOTION;
    // 更新状态改变的时间
    ri->failover_state_change_time = mstime();
}
```

#### 升级主是否超时

```c
void sentinelFailoverWaitPromotion(sentinelRedisInstance *ri) {
  //通过周期发送INFO给从服务器，判断升级主是否成功
    if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {
        sentinelEvent(REDIS_WARNING,"-failover-abort-slave-timeout",ri,"%@");
        sentinelAbortFailover(ri);
    }
}
```

#### 向其他的从服务器发送SlaveOf

```c
void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master) {
    dictIterator *di;
    dictEntry *de;
    int in_progress = 0;

    //获取从服务器
    di = dictGetIterator(master->slaves);
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *slave = dictGetVal(de);
        // SLAVEOF 命令已发送，或者同步正在进行
        if (slave->flags & (SRI_RECONF_SENT|SRI_RECONF_INPROG))
            in_progress++;
    }
    dictReleaseIterator(di);

    // 如果正在同步的从服务器的数量少于 parallel-syncs 选项的值
    // 那么继续遍历从服务器，并让从服务器对新主服务器进行同步
    di = dictGetIterator(master->slaves);
    while(in_progress < master->parallel_syncs &&
          (de = dictNext(di)) != NULL)
    {
        sentinelRedisInstance *slave = dictGetVal(de);
        int retval;

        // 跳过新主服务器，以及已经完成了同步的从服务器
        if (slave->flags & (SRI_PROMOTED|SRI_RECONF_DONE)) continue;

        if ((slave->flags & SRI_RECONF_SENT) &&
            (mstime() - slave->slave_reconf_sent_time) >
            SENTINEL_SLAVE_RECONF_TIMEOUT)
        {
            sentinelEvent(REDIS_NOTICE,"-slave-reconf-sent-timeout",slave,"%@");
            // 清除已发送 SLAVEOF 命令的标记
            slave->flags &= ~SRI_RECONF_SENT;
            slave->flags |= SRI_RECONF_DONE;
        }

        // 如果已向从服务器发送 SLAVEOF 命令，或者同步正在进行
        // 又或者从服务器已断线，那么略过该服务器
        if (slave->flags & (SRI_DISCONNECTED|SRI_RECONF_SENT|SRI_RECONF_INPROG))
            continue;

        // 向从服务器发送 SLAVEOF 命令，让它同步新主服务器
        retval = sentinelSendSlaveOf(slave,
                master->promoted_slave->addr->ip,
                master->promoted_slave->addr->port);
        if (retval == REDIS_OK) {
            // 将状态改为 SLAVEOF 命令已发送
            slave->flags |= SRI_RECONF_SENT;
            // 更新发送 SLAVEOF 命令的时间
            slave->slave_reconf_sent_time = mstime();
            sentinelEvent(REDIS_NOTICE,"+slave-reconf-sent",slave,"%@");
            // 增加当前正在同步的从服务器的数量
            in_progress++;
        }
    }
    dictReleaseIterator(di);

    // 判断是否所有从服务器的同步都已经完成
    sentinelFailoverDetectEnd(master);
}
```

# 集群

## 加入集群

```c
if (!strcasecmp(c->argv[1]->ptr,"meet") && c->argc == 4) {
  /* CLUSTER MEET <ip> <port> */
  // 将给定地址的节点添加到当前节点所处的集群里
  long long port;
  // 检查 port 参数的合法性
  if (getLongLongFromObject(c->argv[3], &port) != REDIS_OK) {
    addReplyErrorFormat(c,"Invalid TCP port specified: %s",
                        (char*)c->argv[3]->ptr);
    return;
  }
  // 尝试与给定地址的节点进行连接
  if (clusterStartHandshake(c->argv[2]->ptr,port) == 0 &&
      errno == EINVAL)
  {
    // 连接失败
    addReplyErrorFormat(c,"Invalid node address specified: %s:%s",
                        (char*)c->argv[2]->ptr, (char*)c->argv[3]->ptr);
  } else {
    // 连接成功
    addReply(c,shared.ok);
  }
```

```c
int clusterStartHandshake(char *ip, int port) {
    clusterNode *n;
    char norm_ip[REDIS_IP_STR_LEN];
    struct sockaddr_storage sa;

    /* IP sanity check */
    if (inet_pton(AF_INET,ip,
            &(((struct sockaddr_in *)&sa)->sin_addr)))
    {
        sa.ss_family = AF_INET;
    } else if (inet_pton(AF_INET6,ip,
            &(((struct sockaddr_in6 *)&sa)->sin6_addr)))
    {
        sa.ss_family = AF_INET6;
    } else {
        errno = EINVAL;
        return 0;
    }
    /* Port sanity check */
    if (port <= 0 || port > (65535-REDIS_CLUSTER_PORT_INCR)) {
        errno = EINVAL;
        return 0;
    }
    /* Set norm_ip as the normalized string representation of the node
     * IP address. */
    if (sa.ss_family == AF_INET)
        inet_ntop(AF_INET,
            (void*)&(((struct sockaddr_in *)&sa)->sin_addr),
            norm_ip,REDIS_IP_STR_LEN);
    else
        inet_ntop(AF_INET6,
            (void*)&(((struct sockaddr_in6 *)&sa)->sin6_addr),
            norm_ip,REDIS_IP_STR_LEN);

    //防止多次握手
    if (clusterHandshakeInProgress(norm_ip,port)) {
        errno = EAGAIN;
        return 0;
    }
    /* Add the node with a random address (NULL as first argument to
     * createClusterNode()). Everything will be fixed during the
     * handskake. */
    n = createClusterNode(NULL,REDIS_NODE_HANDSHAKE|REDIS_NODE_MEET);
    memcpy(n->ip,norm_ip,sizeof(n->ip));
    n->port = port;

    // 将节点添加到集群当中
    clusterAddNode(n);

    return 1;
}
```

## 槽指派

```c
if ((!strcasecmp(c->argv[1]->ptr,"addslots") ||
               !strcasecmp(c->argv[1]->ptr,"delslots")) && c->argc >= 3)
    {
        /* CLUSTER ADDSLOTS <slot> [slot] ... */
        /* CLUSTER DELSLOTS <slot> [slot] ... */
        int j, slot;
        // 一个数组，记录所有要添加或者删除的槽
        unsigned char *slots = zmalloc(REDIS_CLUSTER_SLOTS);
        // 检查这是 delslots 还是 addslots
        int del = !strcasecmp(c->argv[1]->ptr,"delslots");
        // 将 slots 数组的所有值设置为 0
        memset(slots,0,REDIS_CLUSTER_SLOTS);
       
        for (j = 2; j < c->argc; j++) { // 处理所有输入 slot 参数
            // 获取 slot 数字
            if ((slot = getSlotOrReply(c,c->argv[j])) == -1) {
                zfree(slots);
                return;
            }
            // 如果这是 delslots 命令，并且指定槽为未指定，那么返回一个错误
            if (del && server.cluster->slots[slot] == NULL) {
                addReplyErrorFormat(c,"Slot %d is already unassigned", slot);
                zfree(slots);
                return;
            // 如果这是 addslots 命令，并且槽已经有节点在负责，那么返回一个错误
            } else if (!del && server.cluster->slots[slot]) {
                addReplyErrorFormat(c,"Slot %d is already busy", slot);
                zfree(slots);
                return;
            }

            // 如果某个槽指定了一次以上，那么返回一个错误
            if (slots[slot]++ == 1) {
                addReplyErrorFormat(c,"Slot %d specified multiple times",
                    (int)slot);
                zfree(slots);
                return;
            }
        }
        // 处理所有输入 slot
        for (j = 0; j < REDIS_CLUSTER_SLOTS; j++) {
            if (slots[j]) { //slot需要增加或者删除
                int retval;

                /* If this slot was set as importing we can clear this 
                 * state as now we are the real owner of the slot. */
                if (server.cluster->importing_slots_from[j]) //迁入中
                    server.cluster->importing_slots_from[j] = NULL;

                // 添加或者删除指定 slot
                retval = del ? clusterDelSlot(j) :
                               clusterAddSlot(myself,j);
                redisAssertWithInfo(c,NULL,retval == REDIS_OK);
            }
        }
        zfree(slots);
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
}
```



## 槽迁移

### 迁出

```c
       
if (!strcasecmp(c->argv[3]->ptr,"migrating") && c->argc == 5) {
   // CLUSTER SETSLOT <slot> MIGRATING <node id>
   // 将本节点的槽 slot 迁移至 node id 所指定的节点
  // 被迁移的槽必须属于本节点
  if (server.cluster->slots[slot] != myself) {
    addReplyErrorFormat(c,"I'm not the owner of hash slot %u",slot);
    return;
  }

  // 迁移的目标节点必须是本节点已知的
  if ((n = clusterLookupNode(c->argv[4]->ptr)) == NULL) {
    addReplyErrorFormat(c,"I don't know about node %s",
                        (char*)c->argv[4]->ptr);
    return;
  }

  // 为槽设置迁移目标节点
  server.cluster->migrating_slots_to[slot] = n;
} 
```

### 迁入

```c
if (!strcasecmp(c->argv[3]->ptr,"importing") && c->argc == 5) {
// CLUSTER SETSLOT <slot> IMPORTING <node id>
  // 如果 slot 槽本身已经由本节点处理，那么无须进行导入
  if (server.cluster->slots[slot] == myself) {
  addReplyErrorFormat(c,
  "I'm already the owner of hash slot %u",slot);
  return;
  }

  // node id 指定的节点必须是本节点已知的，这样才能从目标节点导入槽
  if ((n = clusterLookupNode(c->argv[4]->ptr)) == NULL) {
  addReplyErrorFormat(c,"I don't know about node %s",
  (char*)c->argv[3]->ptr);
  return;
  }

  // 为槽设置导入目标节点
  server.cluster->importing_slots_from[slot] = n;

}
```

### 获取槽位的key

```c
if (!strcasecmp(c->argv[1]->ptr,"getkeysinslot") && c->argc == 4) {//根据slot获取对应的key
        /* CLUSTER GETKEYSINSLOT <slot> <count> */ 
        //count表明获取key的个数
        long long maxkeys, slot;
        unsigned int numkeys, j;
        robj **keys;

        // 取出 slot 参数
        if (getLongLongFromObjectOrReply(c,c->argv[2],&slot,NULL) != REDIS_OK)
            return;
        // 取出 count 参数
        if (getLongLongFromObjectOrReply(c,c->argv[3],&maxkeys,NULL)
            != REDIS_OK)
            return;
        // 检查参数的合法性
        if (slot < 0 || slot >= REDIS_CLUSTER_SLOTS || maxkeys < 0) {
            addReplyError(c,"Invalid slot or number of keys");
            return;
        }

        // 分配一个保存键的数组
        keys = zmalloc(sizeof(robj*)*maxkeys);
        // 将键记录到 keys 数组
        numkeys = getKeysInSlot(slot, keys, maxkeys);

        // 打印获得的键
        addReplyMultiBulkLen(c,numkeys);
        for (j = 0; j < numkeys; j++) addReplyBulk(c,keys[j]);
        zfree(keys); //释放内存

    } 
```

### 转移数据

```c
void migrateCommand(redisClient *c) {
    int fd, copy, replace, j;
    long timeout;
    long dbid;
    long long ttl, expireat;
    robj *o;
    rio cmd, payload;
    int retry_num = 0;

try_again:
    /* Initialization */
    copy = 0;
    replace = 0;
    ttl = 0;

    //解析参数
    for (j = 6; j < c->argc; j++) {
        if (!strcasecmp(c->argv[j]->ptr,"copy")) {
            copy = 1;
        } else if (!strcasecmp(c->argv[j]->ptr,"replace")) {
            replace = 1;
        } else {
            addReply(c,shared.syntaxerr);
            return;
        }
    }

    // 检查参数
    if (getLongFromObjectOrReply(c,c->argv[5],&timeout,NULL) != REDIS_OK)
        return;
    if (getLongFromObjectOrReply(c,c->argv[4],&dbid,NULL) != REDIS_OK)
        return;
    if (timeout <= 0) timeout = 1000;

    //获取value
    if ((o = lookupKeyRead(c->db,c->argv[3])) == NULL) {
        addReplySds(c,sdsnew("+NOKEY\r\n"));
        return;
    }
		//MIGRATE＜target_ip＞＜target_port＞＜key_name＞0＜timeout＞
    //目标节点建立连接
    fd = migrateGetSocket(c,c->argv[1],c->argv[2],timeout);
    if (fd == -1) return; /* error sent to the client by migrateGetSocket() */

    //选择数据库
    rioInitWithBuffer(&cmd,sdsempty());
    redisAssertWithInfo(c,NULL,rioWriteBulkCount(&cmd,'*',2));
    redisAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,"SELECT",6));
    redisAssertWithInfo(c,NULL,rioWriteBulkLongLong(&cmd,dbid));

    // 获取过期时间
    expireat = getExpire(c->db,c->argv[3]);
    if (expireat != -1) {
        ttl = expireat-mstime();
        if (ttl < 1) ttl = 1;
    }
    redisAssertWithInfo(c,NULL,rioWriteBulkCount(&cmd,'*',replace ? 5 : 4));

    if (server.cluster_enabled)
        redisAssertWithInfo(c,NULL,
            rioWriteBulkString(&cmd,"RESTORE-ASKING",14));
    else
        redisAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,"RESTORE",7));

    redisAssertWithInfo(c,NULL,sdsEncodedObject(c->argv[3]));
    redisAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,c->argv[3]->ptr,
            sdslen(c->argv[3]->ptr)));
    redisAssertWithInfo(c,NULL,rioWriteBulkLongLong(&cmd,ttl));

    /* Emit the payload argument, that is the serialized object using
     * the DUMP format. */
    createDumpPayload(&payload,o);
    redisAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,payload.io.buffer.ptr,
                                sdslen(payload.io.buffer.ptr)));
    sdsfree(payload.io.buffer.ptr);

    if (replace)
        redisAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,"REPLACE",7));

    // 以64kb为单位进行发送
    errno = 0;
    {
        sds buf = cmd.io.buffer.ptr;
        size_t pos = 0, towrite;
        int nwritten = 0;

        while ((towrite = sdslen(buf)-pos) > 0) {
            towrite = (towrite > (64*1024) ? (64*1024) : towrite);
            nwritten = syncWrite(fd,buf+pos,towrite,timeout); //同步写
            if (nwritten != (signed)towrite) goto socket_wr_err;
            pos += nwritten;
        }
    }

    //同步读写会阻塞源节点正常处理请求
    //在迁移数据时要控制迁移的 key 数量和 key 大小，避免一次性迁移过多的 key 或是过大的 key，而导致 Redis 阻塞。
    {
        char buf1[1024];
        char buf2[1024];
				//同步读取响应信息
        if (syncReadLine(fd, buf1, sizeof(buf1), timeout) <= 0)
            goto socket_rd_err;
        if (syncReadLine(fd, buf2, sizeof(buf2), timeout) <= 0)
            goto socket_rd_err;

        if (buf1[0] == '-' || buf2[0] == '-') { //失败
            addReplyErrorFormat(c,"Target instance replied with error: %s",
                (buf1[0] == '-') ? buf1+1 : buf2+1);
        } else { //成功
            robj *aux; 
            if (!copy) {
                dbDelete(c->db,c->argv[3]); //删除
                signalModifiedKey(c->db,c->argv[3]);
            }
            addReply(c,shared.ok);
            server.dirty++;

            aux = createStringObject("DEL",3);
            rewriteClientCommandVector(c,2,aux,c->argv[3]);
            decrRefCount(aux);
        }
    }

    sdsfree(cmd.io.buffer.ptr);
    return;

socket_wr_err:
    sdsfree(cmd.io.buffer.ptr);
    migrateCloseSocket(c->argv[1],c->argv[2]);
    if (errno != ETIMEDOUT && retry_num++ == 0) goto try_again;
    addReplySds(c,
        sdsnew("-IOERR error or timeout writing to target instance\r\n"));
    return;

socket_rd_err:
    sdsfree(cmd.io.buffer.ptr);
    migrateCloseSocket(c->argv[1],c->argv[2]);
    if (errno != ETIMEDOUT && retry_num++ == 0) goto try_again;
    addReplySds(c,
        sdsnew("-IOERR error or timeout reading from target node\r\n"));
    return;
}
```



### 迁移完修改slot信息

```c
 if (!strcasecmp(c->argv[3]->ptr,"node") && c->argc == 5) {
     /* CLUSTER SETSLOT <SLOT> NODE <NODE ID> */
     // 将slot指派给指定的node
     clusterNode *n = clusterLookupNode(c->argv[4]->ptr); //查找目标节点
   
     if (!n) {  //目标节点不已存在
       addReplyErrorFormat(c,"Unknown node %s",
                           (char*)c->argv[4]->ptr);
       return;
     }

     /* If this hash slot was served by 'myself' before to switch
               * make sure there are no longer local keys for this hash slot. */
     if (server.cluster->slots[slot] == myself && n != myself) {
       if (countKeysInSlot(slot) != 0) { //保证slot中无键值对
         addReplyErrorFormat(c,
                             "Can't assign hashslot %d to a different node "
                             "while I still hold keys for this hash slot.", slot);
         return;
       }
     }
   
     if (countKeysInSlot(slot) == 0 &&
         server.cluster->migrating_slots_to[slot]) 
       server.cluster->migrating_slots_to[slot] = NULL; //清除迁出的状态

     if (n == myself &&
         server.cluster->importing_slots_from[slot])//清除迁入的状态
     {
       uint64_t maxEpoch = clusterGetMaxEpoch();

       if (myself->configEpoch == 0 ||
           myself->configEpoch != maxEpoch)
       {
         server.cluster->currentEpoch++;
         myself->configEpoch = server.cluster->currentEpoch;
         clusterDoBeforeSleep(CLUSTER_TODO_FSYNC_CONFIG);
         redisLog(REDIS_WARNING,
                  "configEpoch set to %llu after importing slot %d",
                  (unsigned long long) myself->configEpoch, slot);
       }
       server.cluster->importing_slots_from[slot] = NULL;
     }

     //清除槽位信息
     clusterDelSlot(slot);

     //填充槽位信息
     clusterAddSlot(n,slot);

   } else {
     addReplyError(c,
                   "Invalid CLUSTER SETSLOT action or number of arguments");
     return;
   }
  clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE);
  addReply(c,shared.ok);

  } 
```



## 集群时间事件

```c
void clusterCron(void) {
    dictIterator *di;
    dictEntry *de;
    int update_state = 0;
    int orphaned_masters; /* How many masters there are without ok slaves. */
    int max_slaves; /* Max number of ok slaves for a single master. */
    int this_slaves; /* Number of ok slaves for our master (if we are slave). */
    mstime_t min_pong = 0, now = mstime();
    clusterNode *min_pong_node = NULL;
    // 迭代计数器，一个静态变量
    static unsigned long long iteration = 0;
    mstime_t handshake_timeout;

    iteration++; /* Number of times this function was called so far. */

    handshake_timeout = server.cluster_node_timeout;
    if (handshake_timeout < 1000) handshake_timeout = 1000;

    di = dictGetSafeIterator(server.cluster->nodes); //获取集群节点
    while((de = dictNext(di)) != NULL) {
        clusterNode *node = dictGetVal(de);

        // 跳过当前节点或者没有指定地址的节点
        if (node->flags & (REDIS_NODE_MYSELF|REDIS_NODE_NOADDR)) continue;

        if (nodeInHandshake(node) && now - node->ctime > handshake_timeout) {//握手超时
            freeClusterNode(node);
            continue;
        }

        if (node->link == NULL) { //尚未创建连接
            int fd;
            mstime_t old_ping_sent;
            clusterLink *link;

            fd = anetTcpNonBlockBindConnect(server.neterr, node->ip,
                node->port+REDIS_CLUSTER_PORT_INCR,
                    server.bindaddr_count ? server.bindaddr[0] : NULL);
            if (fd == -1) {
                redisLog(REDIS_DEBUG, "Unable to connect to "
                    "Cluster Node [%s]:%d -> %s", node->ip,
                    node->port+REDIS_CLUSTER_PORT_INCR,
                    server.neterr);
                continue;
            }
            link = createClusterLink(node);
            link->fd = fd;
            node->link = link;
            aeCreateFileEvent(server.el,link->fd,AE_READABLE,
                              
            old_ping_sent = node->ping_sent;
            clusterSendPing(link, node->flags & REDIS_NODE_MEET ?
                    CLUSTERMSG_TYPE_MEET : CLUSTERMSG_TYPE_PING);//发送MEET命令或者PING命令

            if (old_ping_sent) { //之前发送过PING命令
                node->ping_sent = old_ping_sent;
            }

            node->flags &= ~REDIS_NODE_MEET; //清除MEET标志

            redisLog(REDIS_DEBUG,"Connecting with Node %.40s at %s:%d",
                    node->name, node->ip, node->port+REDIS_CLUSTER_PORT_INCR);
        }
    }
    dictReleaseIterator(di); //释放存放当前集群节点的dict

    // 每执行10次时间事件，随机选择一个节点发送PING信息
    if (!(iteration % 10)) {
        int j;
        for (j = 0; j < 5; j++) { //随机挑选5个节点，从中选择PONG回复时间最晚的节点
            // 随机在集群中挑选节点
            de = dictGetRandomKey(server.cluster->nodes);
            clusterNode *this = dictGetVal(de);

            if (this->link == NULL || this->ping_sent != 0) continue;

            if (this->flags & (REDIS_NODE_MYSELF|REDIS_NODE_HANDSHAKE))
                continue;

            if (min_pong_node == NULL || min_pong > this->pong_received) {
                min_pong_node = this;
                min_pong = this->pong_received;
            }
        }

        if (min_pong_node) { //发送PING信息
            redisLog(REDIS_DEBUG,"Pinging node %.40s", min_pong_node->name);
            clusterSendPing(min_pong_node->link, CLUSTERMSG_TYPE_PING);
        }
    }

    // 遍历所有节点，检查是否需要将某个节点标记为下线
    /* Iterate nodes to check if we need to flag something as failing.
     * This loop is also responsible to:
     * 1) Check if there are orphaned masters (masters without non failing
     *    slaves).
     * 2) Count the max number of non failing slaves for a single master.
     * 3) Count the number of slaves for our master, if we are a slave. */
    orphaned_masters = 0;
    max_slaves = 0;
    this_slaves = 0;
    di = dictGetSafeIterator(server.cluster->nodes);
    //遍历节点是否标记为下线状态
    while((de = dictNext(di)) != NULL) {
        clusterNode *node = dictGetVal(de);
        now = mstime(); /* Use an updated time at every iteration. */
        mstime_t delay;

      //跳过无效节点
        if (node->flags &
            (REDIS_NODE_MYSELF|REDIS_NODE_NOADDR|REDIS_NODE_HANDSHAKE))
                continue;

        if (nodeIsSlave(myself) && nodeIsMaster(node) && !nodeFailed(node)) {
            int okslaves = clusterCountNonFailingSlaves(node);

            if (okslaves == 0 && node->numslots > 0) orphaned_masters++;
            if (okslaves > max_slaves) max_slaves = okslaves;
            if (nodeIsSlave(myself) && myself->slaveof == node)
                this_slaves = okslaves;
        }

        /* If we are waiting for the PONG more than half the cluster
         * timeout, reconnect the link: maybe there is a connection
         * issue even if the node is alive. */
        if (node->link && /* is connected */
            now - node->link->ctime >
            server.cluster_node_timeout && /* was not already reconnected */
            node->ping_sent && /* we already sent a ping */
            node->pong_received < node->ping_sent && /* still waiting pong */
            /* and we are waiting for the pong more than timeout/2 */
            now - node->ping_sent > server.cluster_node_timeout/2)
        {
            freeClusterLink(node->link);
        }

        if (node->link &&
            node->ping_sent == 0 &&
            (now - node->pong_received) > server.cluster_node_timeout/2)
        {//发送PING信息
            clusterSendPing(node->link, CLUSTERMSG_TYPE_PING);
            continue;
        }

        /* If we are a master and one of the slaves requested a manual
         * failover, ping it continuously. */
        if (server.cluster->mf_end &&
            nodeIsMaster(myself) &&
            server.cluster->mf_slave == node &&
            node->link)
        {
            clusterSendPing(node->link, CLUSTERMSG_TYPE_PING);
            continue;
        }

        if (node->ping_sent == 0) continue;

        delay = now - node->ping_sent;
        if (delay > server.cluster_node_timeout) { //等待PONG超时
            if (!(node->flags & (REDIS_NODE_PFAIL|REDIS_NODE_FAIL))) {
                redisLog(REDIS_DEBUG,"*** NODE %.40s possibly failing",
                    node->name);
                node->flags |= REDIS_NODE_PFAIL; //标记下线
                update_state = 1;
            }
        }
    }
    dictReleaseIterator(di);

    /* If we are a slave node but the replication is still turned off,
     * enable it if we know the address of our master and it appears to
     * be up. */
    if (nodeIsSlave(myself) &&
        server.masterhost == NULL &&
        myself->slaveof &&
        nodeHasAddr(myself->slaveof))
    {
        replicationSetMaster(myself->slaveof->ip, myself->slaveof->port);
    }

    /* Abourt a manual failover if the timeout is reached. */
    manualFailoverCheckTimeout();

    if (nodeIsSlave(myself)) {
        clusterHandleManualFailover();
        clusterHandleSlaveFailover();
        /* If there are orphaned slaves, and we are a slave among the masters
         * with the max number of non-failing slaves, consider migrating to
         * the orphaned masters. Note that it does not make sense to try
         * a migration if there is no master with at least *two* working
         * slaves. */
        if (orphaned_masters && max_slaves >= 2 && this_slaves == max_slaves)
            clusterHandleSlaveMigration(max_slaves);
    }

    // 更新集群状态
    if (update_state || server.cluster->state == REDIS_CLUSTER_FAIL)
        clusterUpdateState();
}
```



## 执行命令

 MOVED错误

代表槽的负责权已经从一个节点转移到了另一个节点：在客户端收到关于槽i的MOVED错误之后，客户端每次遇到关于槽i的命令请求时，都可以直接将命令请求发送至MOVED错误所指向的节点，因为该节点就是目前负责槽i的节点

ASK错误

如果请求的key对应的槽位正在迁移中，则给客户端返回ASK错误。

当客户端接收到ASK错误并转向至正在导入槽的节点时，客户端会先向节点发送一个ASKING命令，然后才重新发送想要执行的命令，这是因为如果客户端不发送ASKING命令，而直接发送想要执行的命令的话，那么客户端发送的命令将被节点拒绝执行，并返回MOVED错误

注意，客户端的REDIS_ASKING标识是一个一次性标识，当节点执行了一个带有REDIS_ASKING标识的客户端发送的命令之后，客户端的REDIS_ASKING标识就会被移除。

```c
    if (server.cluster_enabled &&
        !(c->flags & REDIS_MASTER) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0))
    {
        int hashslot;
        if (server.cluster->state != REDIS_CLUSTER_OK) {     // 集群已下线
            flagTransaction(c);
            addReplySds(c,sdsnew("-CLUSTERDOWN The cluster is down. Use CLUSTER INFO for more information\r\n"));
            return REDIS_OK;
        } else {   // 集群正常
            int error_code;
            clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,&hashslot,&error_code);
            // 不能执行多键处理命令,命令中的key对应过个slot
            if (n == NULL) {
                flagTransaction(c);
                if (error_code == REDIS_CLUSTER_REDIR_CROSS_SLOT) {
                    addReplySds(c,sdsnew("-CROSSSLOT Keys in request don't hash to the same slot\r\n"));
                } else if (error_code == REDIS_CLUSTER_REDIR_UNSTABLE) {
                    /* The request spawns mutliple keys in the same slot,
                     * but the slot is not "stable" currently as there is
                     * a migration or import in progress. */
                    addReplySds(c,sdsnew("-TRYAGAIN Multiple keys request during rehashing of slot\r\n"));
                } else {
                    redisPanic("getNodeByQuery() unknown error.");
                }
                return REDIS_OK;

            } else if (n != server.cluster->myself) { //命令中的键不是由本节点处理
                flagTransaction(c);
                addReplySds(c,sdscatprintf(sdsempty(),
                    "-%s %d %s:%d\r\n",
                    (error_code == REDIS_CLUSTER_REDIR_ASK) ? "ASK" : "MOVED",
                    hashslot,n->ip,n->port));

                return REDIS_OK;
            }
        }
    }
```

## 维护slot对应的key

```c
void slotToKeyAdd(robj *key) { //添加key到slot中

    // 计算出键所属的槽
    unsigned int hashslot = keyHashSlot(key->ptr,sdslen(key->ptr));

    // 将槽 slot 作为分值，键作为成员，添加到 slots_to_keys 跳跃表里面
    zslInsert(server.cluster->slots_to_keys,hashslot,key);
    incrRefCount(key);
}
```

```c
void slotToKeyDel(robj *key) { //清除slot中key
    unsigned int hashslot = keyHashSlot(key->ptr,sdslen(key->ptr));

    zslDelete(server.cluster->slots_to_keys,hashslot,key);
}
```



## 消息

MEET消息

将节点加入集群

PING消息

检测节点是否在线、同步集群信息

PONG消息

FAIL消息

PUBLISH消息

# 内存友好的数据结构

## 创建共享对象

为了避免在内存中反复创建这些经常被访问的数据，Redis 就采用了共享对象的设计思想。把这些常用数据创建为共享对象，当上层应用需要访问它们时，直接读取就行。

```c
void createSharedObjects(void) {
    int j;

    // 常用回复
    shared.crlf = createObject(REDIS_STRING,sdsnew("\r\n"));
    shared.ok = createObject(REDIS_STRING,sdsnew("+OK\r\n"));
    shared.err = createObject(REDIS_STRING,sdsnew("-ERR\r\n"));
    shared.emptybulk = createObject(REDIS_STRING,sdsnew("$0\r\n\r\n"));
    shared.czero = createObject(REDIS_STRING,sdsnew(":0\r\n"));
    shared.cone = createObject(REDIS_STRING,sdsnew(":1\r\n"));
    shared.cnegone = createObject(REDIS_STRING,sdsnew(":-1\r\n"));
    shared.nullbulk = createObject(REDIS_STRING,sdsnew("$-1\r\n"));
    shared.nullmultibulk = createObject(REDIS_STRING,sdsnew("*-1\r\n"));
    shared.emptymultibulk = createObject(REDIS_STRING,sdsnew("*0\r\n"));
    shared.pong = createObject(REDIS_STRING,sdsnew("+PONG\r\n"));
    shared.queued = createObject(REDIS_STRING,sdsnew("+QUEUED\r\n"));
    shared.emptyscan = createObject(REDIS_STRING,sdsnew("*2\r\n$1\r\n0\r\n*0\r\n"));
    // 常用错误回复
    shared.wrongtypeerr = createObject(REDIS_STRING,sdsnew(
        "-WRONGTYPE Operation against a key holding the wrong kind of value\r\n"));
    shared.nokeyerr = createObject(REDIS_STRING,sdsnew(
        "-ERR no such key\r\n"));
    shared.syntaxerr = createObject(REDIS_STRING,sdsnew(
        "-ERR syntax error\r\n"));
    shared.sameobjecterr = createObject(REDIS_STRING,sdsnew(
        "-ERR source and destination objects are the same\r\n"));
    shared.outofrangeerr = createObject(REDIS_STRING,sdsnew(
        "-ERR index out of range\r\n"));
    shared.noscripterr = createObject(REDIS_STRING,sdsnew(
        "-NOSCRIPT No matching script. Please use EVAL.\r\n"));
    shared.loadingerr = createObject(REDIS_STRING,sdsnew(
        "-LOADING Redis is loading the dataset in memory\r\n"));
    shared.slowscripterr = createObject(REDIS_STRING,sdsnew(
        "-BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.\r\n"));
    shared.masterdownerr = createObject(REDIS_STRING,sdsnew(
        "-MASTERDOWN Link with MASTER is down and slave-serve-stale-data is set to 'no'.\r\n"));
    shared.bgsaveerr = createObject(REDIS_STRING,sdsnew(
        "-MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.\r\n"));
    shared.roslaveerr = createObject(REDIS_STRING,sdsnew(
        "-READONLY You can't write against a read only slave.\r\n"));
    shared.noautherr = createObject(REDIS_STRING,sdsnew(
        "-NOAUTH Authentication required.\r\n"));
    shared.oomerr = createObject(REDIS_STRING,sdsnew(
        "-OOM command not allowed when used memory > 'maxmemory'.\r\n"));
    shared.execaborterr = createObject(REDIS_STRING,sdsnew(
        "-EXECABORT Transaction discarded because of previous errors.\r\n"));
    shared.noreplicaserr = createObject(REDIS_STRING,sdsnew(
        "-NOREPLICAS Not enough good slaves to write.\r\n"));
    shared.busykeyerr = createObject(REDIS_STRING,sdsnew(
        "-BUSYKEY Target key name already exists.\r\n"));

    // 常用字符
    shared.space = createObject(REDIS_STRING,sdsnew(" "));
    shared.colon = createObject(REDIS_STRING,sdsnew(":"));
    shared.plus = createObject(REDIS_STRING,sdsnew("+"));

    // 常用 SELECT 命令
    for (j = 0; j < REDIS_SHARED_SELECT_CMDS; j++) {
        char dictid_str[64];
        int dictid_len;

        dictid_len = ll2string(dictid_str,sizeof(dictid_str),j);
        shared.select[j] = createObject(REDIS_STRING,
            sdscatprintf(sdsempty(),
                "*2\r\n$6\r\nSELECT\r\n$%d\r\n%s\r\n",
                dictid_len, dictid_str));
    }

    // 发布与订阅的有关回复
    shared.messagebulk = createStringObject("$7\r\nmessage\r\n",13);
    shared.pmessagebulk = createStringObject("$8\r\npmessage\r\n",14);
    shared.subscribebulk = createStringObject("$9\r\nsubscribe\r\n",15);
    shared.unsubscribebulk = createStringObject("$11\r\nunsubscribe\r\n",18);
    shared.psubscribebulk = createStringObject("$10\r\npsubscribe\r\n",17);
    shared.punsubscribebulk = createStringObject("$12\r\npunsubscribe\r\n",19);

    // 常用命令
    shared.del = createStringObject("DEL",3);
    shared.rpop = createStringObject("RPOP",4);
    shared.lpop = createStringObject("LPOP",4);
    shared.lpush = createStringObject("LPUSH",5);

    // 常用整数
    for (j = 0; j < REDIS_SHARED_INTEGERS; j++) {
        shared.integers[j] = createObject(REDIS_STRING,(void*)(long)j);
        shared.integers[j]->encoding = REDIS_ENCODING_INT;
    }

    // 常用长度 bulk 或者 multi bulk 回复
    for (j = 0; j < REDIS_SHARED_BULKHDR_LEN; j++) {
        shared.mbulkhdr[j] = createObject(REDIS_STRING,
            sdscatprintf(sdsempty(),"*%d\r\n",j));
        shared.bulkhdr[j] = createObject(REDIS_STRING,
            sdscatprintf(sdsempty(),"$%d\r\n",j));
    }
    /* The following two shared objects, minstring and maxstrings, are not
     * actually used for their value but as a special object meaning
     * respectively the minimum possible string and the maximum possible
     * string in string comparisons for the ZRANGEBYLEX command. */
    shared.minstring = createStringObject("minstring",9);
    shared.maxstring = createStringObject("maxstring",9);
}

```

## 整数集合

### 结构定义

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组，使用数组存储数据，内存连续，避免了内存碎片，提高内存使用率
    int8_t contents[];

} intset;
```

### 创建

```c
intset *intsetNew(void) {
    // 分配空间
    intset *is = zmalloc(sizeof(intset));
    // 设置初始编码
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    // 初始化元素数量
    is->length = 0;
    return is;
}
```

## 压缩列表

总字节数（4字节） + 最后一个元素的偏移量（4字节） + 元素数量（2字节）+ （元素内容）+ 末尾标志（1字节）

元素内容： 前一项的长度 + 当前项的编码方式 + 实际数据

```c
unsigned char *ziplistNew(void) {
    //初始大小
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;
    // 分配内存
    unsigned char *zl = zmalloc(bytes);
    // 初始化表属性
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    // 设置表末端
    zl[bytes-1] = ZIP_END;

    return zl;
}
```

