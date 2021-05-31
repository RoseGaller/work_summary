# Nacos源码分析

* [AP存储](#ap存储)
  * [存储到内存](#存储到内存)
  * [数据同步](#数据同步)
    * [同步变更数据](#同步变更数据)
      * [选择任务调度器](#选择任务调度器)
      * [封装同步任务](#封装同步任务)
      * [提交同步任务](#提交同步任务)
    * [定时同步数据](#定时同步数据)
      * [同步数据校验和](#同步数据校验和)
      * [处理校验和请求](#处理校验和请求)
* [CP存储（Raft）](#cp存储raft)
  * [发布变更事件](#发布变更事件)
  * [执行推送任务](#执行推送任务)
  * [UDP推送数据](#udp推送数据)
  * [接收推送响应](#接收推送响应)
* [服务实例健康检测](#服务实例健康检测)
  * [处理实例的心跳请求](#处理实例的心跳请求)
  * [定时检测清除僵尸实例](#定时检测清除僵尸实例)


# AP存储

1、将接收到的变更数据实时的同步到各节点

2、开启定时任务，计算校验和，将校验和发送至其他节点，如果校验和不一致，向源节点发起获取数据的请求，以达到最终一致性

com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl#put

```java
public void put(String key, Record value) throws NacosException {
  onPut(key, value);//存储到内存
  taskDispatcher.addTask(key); //任务分发
}
```

## 存储到内存

com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl#onPut

```java
if (KeyBuilder.matchEphemeralInstanceListKey(key)) {
   Datum<Instances> datum = new Datum<>();
   datum.value = (Instances) value;
   datum.key = key;
   datum.timestamp.incrementAndGet();
   dataStore.put(key, datum); //存储
}
if (!listeners.containsKey(key)) {
   return;
}
notifier.addTask(key, ApplyAction.CHANGE);
```

## 数据同步

### 同步变更数据

com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher#addTask

```java
 public void addTask(String key) {//选择TaskScheduler，添加同步的key
     taskSchedulerList.get(mapTask(key))//1.计算所选的TaskScheduler
       .addTask(key); //2
}
```

#### 选择任务调度器

```java
public int mapTask(String key) {//初始化时默认创建10个TaskScheduler
    return Math.abs(key.hashCode()) % partitionConfig.getTaskDispatchThreadCount(); 
}
```

```java
public void addTask(String key) {//存储需要同步的key
    queue.offer(key);
}
```

#### 封装同步任务

com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher.TaskScheduler#run

```java
public void run() {//分批同步，累积要同步的key
    List<String> keys = new ArrayList<>();
    while (true) {
        try {
            String key = queue.poll(partitionConfig.getTaskDispatchPeriod(),
                TimeUnit.MILLISECONDS);
            if (Loggers.EPHEMERAL.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                Loggers.EPHEMERAL.debug("got key: {}", key);
            }
            if (dataSyncer.getServers() == null || dataSyncer.getServers().isEmpty()) {
                continue;
            }
            if (dataSize == 0) {
                keys = new ArrayList<>();
            }
            if (StringUtils.isNotBlank(key)) {
                keys.add(key);
                dataSize++;
            }
            //累积1000或者距离上次同步的时间间隔大约200ms
            if (dataSize == partitionConfig.getBatchSyncKeyCount() ||
                (System.currentTimeMillis() - lastDispatchTime) > partitionConfig.getTaskDispatchPeriod()) {
                for (Server member : dataSyncer.getServers()) {
                    if (NetUtils.localServer().equals(member.getKey())) {//排除本机
                        continue;
                    }
                    //向其他机器同步数据
                    SyncTask syncTask = new SyncTask();
                    syncTask.setKeys(keys); //同步的key集合
                    syncTask.setTargetServer(member.getKey()); //同步的机器地址
                    if (Loggers.EPHEMERAL.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                        Loggers.EPHEMERAL.debug("add sync task: {}", JSON.toJSONString(syncTask));
                    }
                    dataSyncer.submit(syncTask, 0);
                }
                lastDispatchTime = System.currentTimeMillis();
                dataSize = 0;
            }
        } catch (Exception e) {
            Loggers.EPHEMERAL.error("dispatch sync task failed.", e);
        }
    }
}
```

#### 提交同步任务

com.alibaba.nacos.naming.consistency.ephemeral.distro.DataSyncer#submit

```java
public void submit(SyncTask task, long delay) {
    // If it's a new task:
    if (task.getRetryCount() == 0) { //此任务第一次执行
        Iterator<String> iterator = task.getKeys().iterator();
        while (iterator.hasNext()) {
            String key = iterator.next();
            if (StringUtils.isNotBlank(taskMap.putIfAbsent(buildKey(key, task.getTargetServer()), key))) {
                // associated key already exist:
                if (Loggers.EPHEMERAL.isDebugEnabled()) {
                    Loggers.EPHEMERAL.debug("sync already in process, key: {}", key);
                }
                iterator.remove(); //将key从此次任务中移除
            }
        }
    }
    if (task.getKeys().isEmpty()) {
        return;
    }
    GlobalExecutor.submitDataSync(new Runnable() {
        @Override
        public void run() {
            try {
                if (servers == null || servers.isEmpty()) {
                    Loggers.SRV_LOG.warn("try to sync data but server list is empty.");
                    return;
                }
                List<String> keys = task.getKeys();
                if (Loggers.EPHEMERAL.isDebugEnabled()) {
                    Loggers.EPHEMERAL.debug("sync keys: {}", keys);
                }
                //获取集合key对应的数据
                Map<String, Datum> datumMap = dataStore.batchGet(keys);
                if (datumMap == null || datumMap.isEmpty()) {
                    // clear all flags of this task:
                    for (String key : task.getKeys()) {
                        taskMap.remove(buildKey(key, task.getTargetServer()));
                    }
                    return;
                }
                //序列化发送的数据
                byte[] data = serializer.serialize(datumMap);
                long timestamp = System.currentTimeMillis();
                //同步数据
                boolean success = NamingProxy.syncData(data, task.getTargetServer());
                if (!success) { //同步失败
                    SyncTask syncTask = new SyncTask();
                    syncTask.setKeys(task.getKeys());
                    syncTask.setRetryCount(task.getRetryCount() + 1);
                    syncTask.setLastExecuteTime(timestamp);
                    syncTask.setTargetServer(task.getTargetServer());
                    retrySync(syncTask); //重试
                } else { //同步成功，从taskMap中移除同步任务
                    for (String key : task.getKeys()) {
                        taskMap.remove(buildKey(key, task.getTargetServer()));
                    }
                }
            } catch (Exception e) {
                Loggers.EPHEMERAL.error("sync data failed.", e);
            }
        }
    }, delay);
}
```

### 定时同步数据

com.alibaba.nacos.naming.consistency.ephemeral.distro.DataSyncer#init

```java
@PostConstruct
public void init() {
    serverListManager.listen(this); //监听集群中节点的变化
    startTimedSync();//启动定时同步
}
```

#### 同步数据校验和

com.alibaba.nacos.naming.consistency.ephemeral.distro.DataSyncer.TimedSync#run

```java
public void run() {
    try {
        // send local timestamps to other servers:
        Map<String, String> keyChecksums = new HashMap<>(64);
        for (String key : dataStore.keys()) {
            //判断本机是否需要负责存储此key
            if (!distroMapper.responsible(KeyBuilder.getServiceName(key))) {
                continue;
            }
            //将此key以及数据的校验和同步到其他机器
            keyChecksums.put(key, dataStore.get(key).value.getChecksum());
        }
        if (keyChecksums.isEmpty()) {
            return;
        }
        if (Loggers.EPHEMERAL.isDebugEnabled()) {
            Loggers.EPHEMERAL.debug("sync checksums: {}", keyChecksums);
        }
        for (Server member : servers) {
            if (NetUtils.localServer().equals(member.getKey())) { //排除本机
                continue;
            }
            //向其他机器发送校验和
            NamingProxy.syncChecksums(keyChecksums, member.getKey());
        }
    } catch (Exception e) {
        Loggers.EPHEMERAL.error("timed sync task failed.", e);
    }
}
```

com.alibaba.nacos.naming.core.DistroMapper#responsible(java.lang.String)

```java
public boolean responsible(String serviceName) { //判断本机是否负责存储此service相关的数据
    if (!switchDomain.isDistroEnabled() || SystemUtils.STANDALONE_MODE) {
        return true;
    }
    if (CollectionUtils.isEmpty(healthyList)) {
        // means distro config is not ready yet
        return false;
    }
    int index = healthyList.indexOf(NetUtils.localServer());
    int lastIndex = healthyList.lastIndexOf(NetUtils.localServer());
    if (lastIndex < 0 || index < 0) {
        return true;
    }
    int target = distroHash(serviceName) % healthyList.size();
    return target >= index && target <= lastIndex;
}
```

```java
public int distroHash(String serviceName) { //计算分布式hash
    return Math.abs(serviceName.hashCode() % Integer.MAX_VALUE);
}
```

#### 处理校验和请求

com.alibaba.nacos.naming.controllers.DistroController#syncChecksum

```java
public String syncChecksum(HttpServletRequest request, HttpServletResponse response) throws Exception {
    String source = WebUtils.required(request, "source");
    String entity = IOUtils.toString(request.getInputStream(), "UTF-8");
    Map<String, String> dataMap =
        serializer.deserialize(entity.getBytes(), new TypeReference<Map<String, String>>() {
    });
    consistencyService.onReceiveChecksums(dataMap, source);
    return "ok";
}
```

com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl#onReceiveChecksums

```java
public void onReceiveChecksums(Map<String, String> checksumMap, String server) {
    List<String> toUpdateKeys = new ArrayList<>();
    List<String> toRemoveKeys = new ArrayList<>();
    for (Map.Entry<String, String> entry : checksumMap.entrySet()) {
        if (distroMapper.responsible(KeyBuilder.getServiceName(entry.getKey()))) {
            Loggers.EPHEMERAL.error("receive responsible key timestamp of " + entry.getKey() + " from " + server);
            return;
        }
        if (!dataStore.contains(entry.getKey()) ||
            dataStore.get(entry.getKey()).value == null ||
            !dataStore.get(entry.getKey()).value.getChecksum().equals(entry.getValue())) {
            toUpdateKeys.add(entry.getKey()); //需要更新的key
        }
    }
    for (String key : dataStore.keys()) {
        if (!server.equals(distroMapper.mapSrv(KeyBuilder.getServiceName(key)))) {
            continue;
        }
        if (!checksumMap.containsKey(key)) {
            toRemoveKeys.add(key);
        }
    }
    Loggers.EPHEMERAL.info("to remove keys: {}, to update keys: {}, source: {}", toRemoveKeys, toUpdateKeys, server);
    for (String key : toRemoveKeys) {
        onRemove(key);
    }
    if (toUpdateKeys.isEmpty()) {
        return;
    }
    try {
        //获取数据
        byte[] result = NamingProxy.getData(toUpdateKeys, server);
        processData(result);
    } catch (Exception e) {
        Loggers.EPHEMERAL.error("get data from " + server + " failed!", e);
    }
}
```





com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher

```
private List<TaskScheduler> taskSchedulerList = new ArrayList<>();//存放TaskScheduler，调度任务
```



```
private DataSyncer dataSyncer; //数据同步
```

com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher#init

```java
for (int i = 0; i < cpuCoreCount; i++) {
    TaskScheduler taskScheduler = new TaskScheduler(i);
    taskSchedulerList.add(taskScheduler);
    GlobalExecutor.submitTaskDispatch(taskScheduler);
}
```

com.alibaba.nacos.naming.consistency.ephemeral.distro.TaskDispatcher.TaskScheduler

批量异步的向集群其他的服务器同步数据

```java
    List<String> keys = new ArrayList<>();
    while (true) {
        try {

            String key = queue.poll(partitionConfig.getTaskDispatchPeriod(),
                TimeUnit.MILLISECONDS);
            if (Loggers.DISTRO.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                Loggers.DISTRO.debug("got key: {}", key);
            }
            if (dataSyncer.getServers() == null || dataSyncer.getServers().isEmpty()) {
                continue;
            }
            if (StringUtils.isBlank(key)) {
                continue;
            }
            if (dataSize == 0) {
                keys = new ArrayList<>();
            }
            keys.add(key);
            dataSize++;
            if (dataSize == partitionConfig.getBatchSyncKeyCount() ||
                (System.currentTimeMillis() - lastDispatchTime) > partitionConfig.getTaskDispatchPeriod()) {
                for (Member member : dataSyncer.getServers()) {
                    if (NetUtils.localServer().equals(member.getAddress())) {
                        continue;
                    }
                    SyncTask syncTask = new SyncTask();
                    syncTask.setKeys(keys);
                    syncTask.setTargetServer(member.getAddress());

                    if (Loggers.DISTRO.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                        Loggers.DISTRO.debug("add sync task: {}", JacksonUtils.toJson(syncTask));
                    }
                    dataSyncer.submit(syncTask, 0);
                }
                lastDispatchTime = System.currentTimeMillis();
                dataSize = 0;
            }

        } catch (Exception e) {
            Loggers.DISTRO.error("dispatch sync task failed.", e);
        }
    }
}
```

# CP存储（Raft）

1、本地节点存储、发布变更事件

2、将变更数据同步至集群各节点，阻塞直到收到过半数节点的响应

com.alibaba.nacos.naming.consistency.persistent.raft.RaftConsistencyServiceImpl#put

```java
public void put(String key, Record value) throws NacosException {
    try {
        raftCore.signalPublish(key, value);
    } catch (Exception e) {
        Loggers.RAFT.error("Raft put failed.", e);
        throw new NacosException(NacosException.SERVER_ERROR, "Raft put failed, key:" + key + ", value:" + value);
    }
}
```

com.alibaba.nacos.naming.consistency.persistent.raft.RaftCore#signalPublish

```java
public void signalPublish(String key, Record value) throws Exception {
    if (!isLeader()) { //不是leader
        JSONObject params = new JSONObject();
        params.put("key", key);
        params.put("value", value);
        Map<String, String> parameters = new HashMap<>(1);
        parameters.put("key", key);
        //将请求转发到leader节点
        raftProxy.proxyPostLarge(getLeader().ip, API_PUB, params.toJSONString(), parameters);
        return;
    }
    try {
        OPERATE_LOCK.lock();
        long start = System.currentTimeMillis();
        final Datum datum = new Datum();
        datum.key = key;
        datum.value = value;
        if (getDatum(key) == null) {
            datum.timestamp.set(1L);
        } else {
            datum.timestamp.set(getDatum(key).timestamp.incrementAndGet());
        }
        JSONObject json = new JSONObject();
        json.put("datum", datum);
        json.put("source", peers.local());
        //发布变更事件
        onPublish(datum, peers.local());
        final String content = JSON.toJSONString(json);
        //等待集群中过半节点返回成功响应
        final CountDownLatch latch = new CountDownLatch(peers.majorityCount());
        for (final String server : peers.allServersIncludeMyself()) {
            if (isLeader(server)) {
                latch.countDown();
                continue;
            }
            //异步发送数据
            final String url = buildURL(server, API_ON_PUB);
            HttpClient.asyncHttpPostLarge(url, Arrays.asList("key=" + key), content, new AsyncCompletionHandler<Integer>() {
                @Override
                public Integer onCompleted(Response response) throws Exception {
                    if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                        Loggers.RAFT.warn("[RAFT] failed to publish data to peer, datumId={}, peer={}, http code={}",
                            datum.key, server, response.getStatusCode());
                        return 1;
                    }
                    latch.countDown();
                    return 0;
                }
                @Override
                public STATE onContentWriteCompleted() {
                    return STATE.CONTINUE;
                }
            });
        }
        //阻塞，直到超时
        if (!latch.await(UtilsAndCommons.RAFT_PUBLISH_TIMEOUT, TimeUnit.MILLISECONDS)) { //超时抛异常
            // only majority servers return success can we consider this update success
            Loggers.RAFT.info("data publish failed, caused failed to notify majority, key={}", key);
            throw new IllegalStateException("data publish failed, caused failed to notify majority, key=" + key);
        }
        long end = System.currentTimeMillis();
        Loggers.RAFT.info("signalPublish cost {} ms, key: {}", (end - start), key);
    } finally {
        OPERATE_LOCK.unlock();
    }
}
```

## 发布变更事件

com.alibaba.nacos.naming.consistency.persistent.raft.RaftCore#onPublish

```java
public void onPublish(Datum datum, RaftPeer source) throws Exception {
    RaftPeer local = peers.local();
    if (datum.value == null) {  //value为空
        Loggers.RAFT.warn("received empty datum");
        throw new IllegalStateException("received empty datum");
    }

    if (!peers.isLeader(source.ip)) { //不再是Leader
        Loggers.RAFT.warn("peer {} tried to publish data but wasn't leader, leader: {}",
            JSON.toJSONString(source), JSON.toJSONString(getLeader()));
        throw new IllegalStateException("peer(" + source.ip + ") tried to publish " +
            "data but wasn't leader");
    }
    if (source.term.get() < local.term.get()) {//之前的任期小于当前任期
        Loggers.RAFT.warn("out of date publish, pub-term: {}, cur-term: {}",
            JSON.toJSONString(source), JSON.toJSONString(local));
        throw new IllegalStateException("out of date publish, pub-term:"
            + source.term.get() + ", cur-term: " + local.term.get());
    }
    local.resetLeaderDue();
    if (KeyBuilder.matchPersistentKey(datum.key)) {
        raftStore.write(datum); //持久化存储
    }
    //放入内存Map
    datums.put(datum.key, datum);
    if (isLeader()) {
        local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
    } else {
        if (local.term.get() + PUBLISH_TERM_INCREASE_COUNT > source.term.get()) {
            //set leader term:
            getLeader().term.set(source.term.get());
            local.term.set(getLeader().term.get());
        } else {
            local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
        }
    }
    raftStore.updateTerm(local.term.get());
    //添加同步任务
    notifier.addTask(datum, ApplyAction.CHANGE);
    Loggers.RAFT.info("data added/updated, key={}, term={}", datum.key, local.term);
}
```

```java
public void addTask(Datum datum, ApplyAction action) {
    if (services.containsKey(datum.key) && action == ApplyAction.CHANGE) { //已经包含此key
        return;
    }
    if (action == ApplyAction.CHANGE) {
        services.put(datum.key, StringUtils.EMPTY);
    }
    tasks.add(Pair.with(datum, action));
}
```

推送变更数据

com.alibaba.nacos.naming.consistency.persistent.raft.RaftCore.Notifier#run

```java
public void run() {
        Loggers.RAFT.info("raft notifier started");
        while (true) {
            try {
                Pair pair = tasks.take();
                if (pair == null) {
                    continue;
                }
                Datum datum = (Datum) pair.getValue0();
                ApplyAction action = (ApplyAction) pair.getValue1();
                services.remove(datum.key);
                int count = 0;
                if (listeners.containsKey(KeyBuilder.SERVICE_META_KEY_PREFIX)) {
                    if (KeyBuilder.matchServiceMetaKey(datum.key) && !KeyBuilder.matchSwitchKey(datum.key)) {
                        for (RecordListener listener : listeners.get(KeyBuilder.SERVICE_META_KEY_PREFIX)) {
                            try {
                                if (action == ApplyAction.CHANGE) {
                                    listener.onChange(datum.key, getDatum(datum.key).value);
                                }
                                if (action == ApplyAction.DELETE) {
                                    listener.onDelete(datum.key);
                                }
                            } catch (Throwable e) {
                                Loggers.RAFT.error("[NACOS-RAFT] error while notifying listener of key: {} {}", datum.key, e);
                            }
                        }
                    }
                }
                if (!listeners.containsKey(datum.key)) {
                    continue;
                }
                //订阅该key的监听器
                for (RecordListener listener : listeners.get(datum.key)) {
                    count++;
                    try {
                        if (action == ApplyAction.CHANGE) { //数据变更
                            listener.onChange(datum.key, getDatum(datum.key).value);
                            continue;
                        }
                        if (action == ApplyAction.DELETE) { //数据删除
                            listener.onDelete(datum.key);
                            continue;
                        }
                    } catch (Throwable e) {
                        Loggers.RAFT.error("[NACOS-RAFT] error while notifying listener of key: {} {}", datum.key, e);
                    }
                }
                if (Loggers.RAFT.isDebugEnabled()) {
                    Loggers.RAFT.debug("[NACOS-RAFT] datum change notified, key: {}, listener count: {}", datum.key, count);
                }
            } catch (Throwable e) {
                Loggers.RAFT.error("[NACOS-RAFT] Error while handling notifying task", e);
            }
        }
    }
}
```

## 执行推送任务

1、获取订阅此服务的客户端，并过滤掉僵尸客户端

2、判断是否压缩推送数据，通过udp发送数据

com.alibaba.nacos.naming.push.PushService#serviceChanged

```java
public void serviceChanged(final String namespaceId, final String serviceName) {
    // 合并事件减少推送的频率
    if (futureMap.containsKey(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName))) {
        return;
    }
    //异步通过udp推送变更数据
    Future future = udpSender.schedule(new Runnable() {
        @Override
        public void run() {
            try {
                Loggers.PUSH.info(serviceName + " is changed, add it to push queue.");
                //获取订阅此服务的客户端
                ConcurrentMap<String, PushClient> clients = clientMap.get(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName));
                if (MapUtils.isEmpty(clients)) {
                    return;
                }
                Map<String, Object> cache = new HashMap<>(16);
                long lastRefTime = System.nanoTime();
                for (PushClient client : clients.values()) {
                    if (client.zombie()) { //僵尸客户端，
                        Loggers.PUSH.debug("client is zombie: " + client.toString());
                        clients.remove(client.toString());
                        Loggers.PUSH.debug("client is zombie: " + client.toString());
                        continue;
                    }
                    Receiver.AckEntry ackEntry;
                    Loggers.PUSH.debug("push serviceName: {} to client: {}", serviceName, client.toString());
                    String key = getPushCacheKey(serviceName, client.getIp(), client.getAgent());
                    byte[] compressData = null;
                    Map<String, Object> data = null;
                    if (switchDomain.getDefaultPushCacheMillis() >= 20000 && cache.containsKey(key)) {
                        org.javatuples.Pair pair = (org.javatuples.Pair) cache.get(key);
                        compressData = (byte[]) (pair.getValue0());
                        data = (Map<String, Object>) pair.getValue1();
                        Loggers.PUSH.debug("[PUSH-CACHE] cache hit: {}:{}", serviceName, client.getAddrStr());
                    }
                    if (compressData != null) {
                      	//创建AckEntry
                        ackEntry = prepareAckEntry(client, compressData, data, lastRefTime);
                    } else {
                        ackEntry = prepareAckEntry(client, prepareHostsData(client), lastRefTime);
                        if (ackEntry != null) {
                            cache.put(key, new org.javatuples.Pair<>(ackEntry.origin.getData(), ackEntry.data));
                        }
                    }
                    Loggers.PUSH.info("serviceName: {} changed, schedule push for: {}, agent: {}, key: {}",
                        client.getServiceName(), client.getAddrStr(), client.getAgent(),  (ackEntry == null ? null : ackEntry.key));
									//udp推送数据
                    udpPush(ackEntry);
                }
            } catch (Exception e) {
                Loggers.PUSH.error("[NACOS-PUSH] failed to push serviceName: {} to client, error: {}", serviceName, e);
            } finally {
                futureMap.remove(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName));
            }
        }
    }, 1000, TimeUnit.MILLISECONDS);
    futureMap.put(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName), future);
}
```

## UDP推送数据

com.alibaba.nacos.naming.push.PushService#udpPush

```java
private static Receiver.AckEntry udpPush(Receiver.AckEntry ackEntry) {
    if (ackEntry == null) {
        Loggers.PUSH.error("[NACOS-PUSH] ackEntry is null.");
        return null;
    }
    if (ackEntry.getRetryTimes() > MAX_RETRY_TIMES) { //重试次数达到上限，默认为1
        Loggers.PUSH.warn("max re-push times reached, retry times {}, key: {}", ackEntry.retryTimes, ackEntry.key);
        ackMap.remove(ackEntry.key);
        udpSendTimeMap.remove(ackEntry.key);
        failedPush += 1;
        return ackEntry;
    }
    try {
        if (!ackMap.containsKey(ackEntry.key)) {
            totalPush++;
        }
        ackMap.put(ackEntry.key, ackEntry);
        udpSendTimeMap.put(ackEntry.key, System.currentTimeMillis());

        Loggers.PUSH.info("send udp packet: " + ackEntry.key);
        //发送
        udpSocket.send(ackEntry.origin);

        ackEntry.increaseRetryTime();
        //超时，未收到ack，重新发送
        executorService.schedule(new Retransmitter(ackEntry), TimeUnit.NANOSECONDS.toMillis(ACK_TIMEOUT_NANOS),
                TimeUnit.MILLISECONDS);
        return ackEntry;
    } catch (Exception e) {
        Loggers.PUSH.error("[NACOS-PUSH] failed to push data: {} to client: {}, error: {}",
            ackEntry.data, ackEntry.origin.getAddress().getHostAddress(), e);
        ackMap.remove(ackEntry.key);
        udpSendTimeMap.remove(ackEntry.key);
        failedPush += 1;

        return null;
    }
}
```

## 接收推送响应

com.alibaba.nacos.naming.push.PushService.Receiver#run

```java
public void run() {
    while (true) {
        byte[] buffer = new byte[1024 * 64];
        DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
        try {
            //接收响应数据
            udpSocket.receive(packet);

            String json = new String(packet.getData(), 0, packet.getLength(), Charset.forName("UTF-8")).trim();
            AckPacket ackPacket = JSON.parseObject(json, AckPacket.class);

            InetSocketAddress socketAddress = (InetSocketAddress) packet.getSocketAddress();
            String ip = socketAddress.getAddress().getHostAddress();
            int port = socketAddress.getPort();

            if (System.nanoTime() - ackPacket.lastRefTime > ACK_TIMEOUT_NANOS) { //响应超时
                Loggers.PUSH.warn("ack takes too long from {} ack json: {}",packet.getSocketAddress(), json);
            }
            String ackKey = getACKKey(ip, port, ackPacket.lastRefTime);
            AckEntry ackEntry = ackMap.remove(ackKey);
            if (ackEntry == null) {
                throw new IllegalStateException("unable to find ackEntry for key: " + ackKey
                        + ", ack json: " + json);
            }
            //推送耗时
            long pushCost = System.currentTimeMillis() - udpSendTimeMap.get(ackKey);
            Loggers.PUSH.info("received ack: {} from: {}:, cost: {} ms, unacked: {}, total push: {}",
                json, ip, port, pushCost, ackMap.size(), totalPush);
            pushCostMap.put(ackKey, pushCost);
            udpSendTimeMap.remove(ackKey);

        } catch (Throwable e) {
            Loggers.PUSH.error("[NACOS-PUSH] error while receiving ack data", e);
        }
    }
}
```

# 服务实例健康检测

## 处理实例的心跳请求

com.alibaba.nacos.naming.healthcheck.ClientBeatProcessor#run

```java
public void run() {
    Service service = this.service;
    if (Loggers.EVT_LOG.isDebugEnabled()) {
        Loggers.EVT_LOG.debug("[CLIENT-BEAT] processing beat: {}", rsInfo.toString());
    }
    String ip = rsInfo.getIp(); //源ip地址
    String clusterName = rsInfo.getCluster(); //集群
    int port = rsInfo.getPort();
    Cluster cluster = service.getClusterMap().get(clusterName);
    List<Instance> instances = cluster.allIPs(true);
    for (Instance instance : instances) {
        if (instance.getIp().equals(ip) && instance.getPort() == port) { //获取实例
            if (Loggers.EVT_LOG.isDebugEnabled()) {
                Loggers.EVT_LOG.debug("[CLIENT-BEAT] refresh beat: {}", rsInfo.toString());
            }
            instance.setLastBeat(System.currentTimeMillis()); //修改心跳时间
            if (!instance.isMarked()) {
                if (!instance.isHealthy()) {//之前下线的实例标记为健康状态
                    instance.setHealthy(true);
                    Loggers.EVT_LOG.info("service: {} {POS} {IP-ENABLED} valid: {}:{}@{}, region: {}, msg: client beat ok",
                        cluster.getService().getName(), ip, port, cluster.getName(), UtilsAndCommons.LOCALHOST_SITE);
                    getPushService().serviceChanged(service.getNamespaceId(), this.service.getName()); //推送上线的实例
                }
            }
        }
    }
}
```

## 定时检测清除僵尸实例

com.alibaba.nacos.naming.healthcheck.ClientBeatCheckTask#run

```java
public void run() {
    try {
        if (!getDistroMapper().responsible(service.getName())) {
            return;
        }

        List<Instance> instances = service.allIPs(true);

        // first set health status of instances:
        for (Instance instance : instances) {
            //默认超过15s没有收到心跳，将此实例标记为下线状态
            if (System.currentTimeMillis() - instance.getLastBeat() > ClientBeatProcessor.CLIENT_BEAT_TIMEOUT) {
                if (!instance.isMarked()) {
                    if (instance.isHealthy()) {
                        instance.setHealthy(false);
                        Loggers.EVT_LOG.info("{POS} {IP-DISABLED} valid: {}:{}@{}, region: {}, msg: client timeout after {}, last beat: {}",
                            instance.getIp(), instance.getPort(), instance.getClusterName(),
                            UtilsAndCommons.LOCALHOST_SITE, ClientBeatProcessor.CLIENT_BEAT_TIMEOUT, instance.getLastBeat());
                        getPushService().serviceChanged(service.getNamespaceId(), service.getName()); //告知订阅此服务的客户端，此实例已下线
                    }
                }
            }
        }
        //默认心跳超时30s
        // then remove obsolete instances:
        for (Instance instance : instances) {
            if (System.currentTimeMillis() - instance.getLastBeat() > service.getIpDeleteTimeout()) {
                // delete instance
                Loggers.SRV_LOG.info("[AUTO-DELETE-IP] service: {}, ip: {}", service.getName(), JSON.toJSONString(instance));
                deleteIP(instance);
            }
        }
    } catch (Exception e) {
        Loggers.SRV_LOG.warn("Exception while processing client beat time out.", e);
    }
}
```

```java
private void deleteIP(Instance instance) {

    try {
        NamingProxy.Request request = NamingProxy.Request.newRequest();
        request.appendParam("ip", instance.getIp())
            .appendParam("port", String.valueOf(instance.getPort()))
            .appendParam("ephemeral", "true")
            .appendParam("clusterName", instance.getClusterName())
            .appendParam("serviceName", service.getName())
            .appendParam("namespaceId", service.getNamespaceId());
        String url = "http://127.0.0.1:" + RunningConfig.getServerPort() + RunningConfig.getContextPath()
            + UtilsAndCommons.NACOS_NAMING_CONTEXT + "/instance?" + request.toUrl();
        // 异步删除实例
        HttpClient.asyncHttpDelete(url, null, null, new AsyncCompletionHandler() {
            @Override
            public Object onCompleted(Response response) throws Exception {
                if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                    Loggers.SRV_LOG.error("[IP-DEAD] failed to delete ip automatically, ip: {}, caused {}, resp code: {}",
                        instance.toJSON(), response.getResponseBody(), response.getStatusCode());
                }
                return null;
            }
        });

    } catch (Exception e) {
        Loggers.SRV_LOG.error("[IP-DEAD] failed to delete ip automatically, ip: {}, error: {}", instance.toJSON(), e);
    }
}
```






