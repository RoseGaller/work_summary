# Leaf源码分析

* [概述](#概述)
* [snowflake模式](#snowflake模式)
  * [配置](#配置)
  * [初始化](#初始化)
  * [获取Id](#获取id)
* [号段模式](#号段模式)
  * 初始化
  * 获取Id


# 概述

Leaf 提供两种生成的ID的方式（号段模式和snowflake模式)

既支持以HTTP形式对外提供服务，也支持以RPC的形式提供服务

# snowflake模式

**1、弱依赖Zookeeper，除了每次会去ZK拿数据以外，也会在本机缓存一个workerID文件，当Zookeeper出现问题时，从本地文件加载WorkerID**

**2、对sequence的初始值进行随机化,避免极端情况下的数据倾斜**

## 配置

```
leaf.snowflake.enable=true #启动snowflake模式
leaf.snowflake.zk.address=127.0.0.1:2181 #zk地址
```

## 初始化

com.sankuai.inf.leaf.snowflake.SnowflakeIDGenImpl#SnowflakeIDGenImpl(java.lang.String, int, long)

```java
public SnowflakeIDGenImpl(String zkAddress, int port, long twepoch) {
    //起始的时间戳
    this.twepoch = twepoch; 
    Preconditions.checkArgument(timeGen() > twepoch, "Snowflake not support twepoch gt currentTime");
    final String ip = Utils.getIp();
    SnowflakeZookeeperHolder holder = new SnowflakeZookeeperHolder(ip, String.valueOf(port), zkAddress);
    LOGGER.info("twepoch:{} ,ip:{} ,zkAddress:{} port:{}", twepoch, ip, zkAddress, port);
    //创建永久顺序节点，获取workerId
    boolean initFlag = holder.init();
    if (initFlag) {
        workerId = holder.getWorkerID();
        LOGGER.info("START SUCCESS USE ZK WORKERID-{}", workerId);
    } else {
        Preconditions.checkArgument(initFlag, "Snowflake Id Gen is not init ok");
    }
    Preconditions.checkArgument(workerId >= 0 && workerId <= maxWorkerId, "workerID must gte 0 and lte 1023");
}
```

com.sankuai.inf.leaf.snowflake.SnowflakeZookeeperHolder#init

```java
public boolean init() {//创建永久顺序节点，从zk或者本地文件获取workerId
    try {
        CuratorFramework curator = createWithOptions(connectionString, new RetryUntilElapsed(1000, 4), 10000, 6000);
        curator.start();
        //节点是否存在
        Stat stat = curator.checkExists().forPath(PATH_FOREVER);
        if (stat == null) {
            //不存在根节点,机器第一次启动,创建/snowflake/ip:port-000000000,并上传数据
            zk_AddressNode = createNode(curator);
            //worker id 默认是0
            updateLocalWorkerID(workerID);
            //定时上报本机时间给forever节点
            ScheduledUploadData(curator, zk_AddressNode);
            return true;
        } else {//不是第一次启动
            Map<String, Integer> nodeMap = Maps.newHashMap();//ip:port->00001
            Map<String, String> realNode = Maps.newHashMap();//ip:port->(ipport-000001)
            //存在根节点,先检查是否有属于自己的根节点
            ///snowflake/ip:port-000000000
            List<String> keys = curator.getChildren().forPath(PATH_FOREVER);
            for (String key : keys) {
                String[] nodeKey = key.split("-");
                realNode.put(nodeKey[0], key);
                nodeMap.put(nodeKey[0], Integer.parseInt(nodeKey[1]));
            }
            Integer workerid = nodeMap.get(listenAddress);
            if (workerid != null) {
                //有自己的节点,zk_AddressNode=ip:port
                zk_AddressNode = PATH_FOREVER + "/" + realNode.get(listenAddress);
                workerID = workerid;//启动worder时使用会使用
                if (!checkInitTimeStamp(curator, zk_AddressNode)) {
                    throw new CheckLastTimeException("init timestamp check error,forever node timestamp gt this node time");
                }
                //准备创建临时节点
                doService(curator);
                //在节点文件系统上缓存一个workid值,zk失效,机器重启时保证能够正常启动
                updateLocalWorkerID(workerID);
                LOGGER.info("[Old NODE]find forever node have this endpoint ip-{} port-{} workid-{} childnode and start SUCCESS", ip, port, workerID);
            } else {
                //表示新启动的节点,创建持久节点 ,不用check时间
                String newNode = createNode(curator);
                zk_AddressNode = newNode;
                String[] nodeKey = newNode.split("-");
                workerID = Integer.parseInt(nodeKey[1]);
                doService(curator);
              	//在节点文件系统上缓存一个workid值,zk失效,机器重启时保证能够正常启动
                updateLocalWorkerID(workerID);
                LOGGER.info("[New NODE]can not find node on forever node that endpoint ip-{} port-{} workid-{},create own node on forever node and start SUCCESS ", ip, port, workerID);
            }
        }
    } catch (Exception e) { //弱依赖Zookeeper，一定程度上提高了SLA
        LOGGER.error("Start node ERROR {}", e);
        try {
          //从本地文件加载workerId
            Properties properties = new Properties();
            properties.load(new FileInputStream(new File(PROP_PATH.replace("{port}", port + ""))));
            workerID = Integer.valueOf(properties.getProperty("workerID"));
            LOGGER.warn("START FAILED ,use local node file properties workerID-{}", workerID);
        } catch (Exception e1) {
            LOGGER.error("Read file error ", e1);
            return false;
        }
    }
    return true;
}
```

## 获取Id

com.sankuai.inf.leaf.snowflake.SnowflakeIDGenImpl#get

```java
public synchronized Result get(String key) {//考虑时间回退、毫秒的初始sequence取随机值
    long timestamp = timeGen();
    if (timestamp < lastTimestamp) { //时间发生回退
        long offset = lastTimestamp - timestamp;
        if (offset <= 5) { //回退时间比较少
            try {
                wait(offset << 1); //先等待
                timestamp = timeGen(); //再次获取时间
                if (timestamp < lastTimestamp) { //如果回退还发生，则返回异常
                    return new Result(-1, Status.EXCEPTION);
                }
            } catch (InterruptedException e) {
                LOGGER.error("wait interrupted");
                return new Result(-2, Status.EXCEPTION);
            }
        } else {//回退时间比较大，直接返回异常信息
            return new Result(-3, Status.EXCEPTION);
        }
    }
    if (lastTimestamp == timestamp) {
        sequence = (sequence + 1) & sequenceMask;
        //sequence为0的时候表示当前毫秒内生成的序列号已达到上限
        if (sequence == 0) {
            //sequence取随机值，避免极端情况下，每一毫秒只有一个或者少量请求，避免数据倾斜
            sequence = RANDOM.nextInt(100);
            timestamp = tilNextMillis(lastTimestamp);
        }
    } else {
        sequence = RANDOM.nextInt(100);
    }
    lastTimestamp = timestamp;
    //计算当前时间的ID
    long id = ((timestamp - twepoch) << timestampLeftShift) | (workerId << workerIdShift) | sequence;
    return new Result(id, Status.SUCCESS);
}
```

# 号段模式

1、根据两个Segmen填充的间隔，动态调整step

2、默认情况下，当前号段已经使用10%，则开始下载下一号段

## 初始化

com.sankuai.inf.leaf.segment.SegmentIDGenImpl#init

```java
public boolean init() {
    logger.info("Init ...");
    // 确保加载到kv后才初始化成功
    updateCacheFromDb();
    initOK = true;
    updateCacheFromDbAtEveryMinute();
    return initOK;
}

private void updateCacheFromDbAtEveryMinute() {
    ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor(new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("check-idCache-thread");
            t.setDaemon(true);
            return t;
        }
    });
    service.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            updateCacheFromDb();
        }
    }, 60, 60, TimeUnit.SECONDS);
}
```

```java
private void updateCacheFromDb() {
  //扫描数据库，获取新增的服务，创建对应的SegmentBuffer
    logger.info("update cache from db");
    StopWatch sw = new Slf4JStopWatch();
    try {
        List<String> dbTags = dao.getAllTags(); //获取所有的服务标志
        if (dbTags == null || dbTags.isEmpty()) {
            return;
        }
        List<String> cacheTags = new ArrayList<String>(cache.keySet());
        Set<String> insertTagsSet = new HashSet<>(dbTags);
        Set<String> removeTagsSet = new HashSet<>(cacheTags);
        //db中新加的tags灌进cache
        for(int i = 0; i < cacheTags.size(); i++){
            String tmp = cacheTags.get(i);
            if(insertTagsSet.contains(tmp)){
                insertTagsSet.remove(tmp); //移除已经缓存的，保留新增的
            }
        }
        for (String tag : insertTagsSet) {
            SegmentBuffer buffer = new SegmentBuffer();
            buffer.setKey(tag);
            Segment segment = buffer.getCurrent();
            segment.setValue(new AtomicLong(0));
            segment.setMax(0);
            segment.setStep(0);
            cache.put(tag, buffer);
            logger.info("Add tag {} from db to IdCache, SegmentBuffer {}", tag, buffer);
        }
        //cache中已失效的tags从cache删除
        for(int i = 0; i < dbTags.size(); i++){
            String tmp = dbTags.get(i);
            if(removeTagsSet.contains(tmp)){
                removeTagsSet.remove(tmp);
            }
        }
        for (String tag : removeTagsSet) {
            cache.remove(tag);
            logger.info("Remove tag {} from IdCache", tag);
        }
    } catch (Exception e) {
        logger.warn("update cache from db exception", e);
    } finally {
        sw.stop("updateCacheFromDb");
    }
}
```

## 获取Id

com.sankuai.inf.leaf.segment.SegmentIDGenImpl#get

```java
public Result get(final String key) { //key为业务标志
    if (!initOK) { //尚未初始化完成
        return new Result(EXCEPTION_ID_IDCACHE_INIT_FALSE, Status.EXCEPTION);
    }
    if (cache.containsKey(key)) { //判断服务标志的合法性
        SegmentBuffer buffer = cache.get(key);
        if (!buffer.isInitOk()) { //尚未初始化
            synchronized (buffer) {
                if (!buffer.isInitOk()) {//双重检测，尚未初始化
                    try {
                        updateSegmentFromDb(key, buffer.getCurrent());//从数据库加载Segment
                        logger.info("Init buffer. Update leafkey {} {} from db", key, buffer.getCurrent());
                        buffer.setInitOk(true);
                    } catch (Exception e) {
                        logger.warn("Init buffer {} exception", buffer.getCurrent(), e);
                    }
                }
            }
        }
         //初始化完成
        return getIdFromSegmentBuffer(cache.get(key));
    }
    return new Result(EXCEPTION_ID_KEY_NOT_EXISTS, Status.EXCEPTION);

```

com.sankuai.inf.leaf.segment.SegmentIDGenImpl#updateSegmentFromDb

```java
public void updateSegmentFromDb(String key, Segment segment) {
    StopWatch sw = new Slf4JStopWatch();
    SegmentBuffer buffer = segment.getBuffer();
    LeafAlloc leafAlloc;
    if (!buffer.isInitOk()) { //尚未初始化
        leafAlloc = dao.updateMaxIdAndGetLeafAlloc(key);
        buffer.setStep(leafAlloc.getStep());
        buffer.setMinStep(leafAlloc.getStep());//leafAlloc中的step为DB中的step
    } else if (buffer.getUpdateTimestamp() == 0) {////第二次加载，不同的地方就是要设置加载更新的时间，之后根据两次加载之间的间隔动态调整step
        leafAlloc = dao.updateMaxIdAndGetLeafAlloc(key);
        buffer.setUpdateTimestamp(System.currentTimeMillis());
        buffer.setStep(leafAlloc.getStep());
        buffer.setMinStep(leafAlloc.getStep());//leafAlloc中的step为DB中的step
    } else {
      //动态修改步长
        long duration = System.currentTimeMillis() - buffer.getUpdateTimestamp();
        int nextStep = buffer.getStep();
        //距离上次加载号段时间间隔小于15分钟；高并发下号段消耗的太快，增大步长
        if (duration < SEGMENT_DURATION) { 
            if (nextStep * 2 > MAX_STEP) { //加载的步长不能大于最大值100，0000
                //do nothing
            } else {
                nextStep = nextStep * 2; //修改步长为原来的两倍
            }
        } else if (duration < SEGMENT_DURATION * 2) { //间隔在15-30分钟之间
            //do nothing with nextStep
        } else { //间隔大于30分钟，缩小步长
            nextStep = nextStep / 2 >= buffer.getMinStep() ? nextStep / 2 : nextStep;
        }
        logger.info("leafKey[{}], step[{}], duration[{}mins], nextStep[{}]", key, buffer.getStep(), String.format("%.2f",((double)duration / (1000 * 60))), nextStep);
        LeafAlloc temp = new LeafAlloc();
        temp.setKey(key);
        temp.setStep(nextStep);
        //修改步长且获取号段
        leafAlloc = dao.updateMaxIdByCustomStepAndGetLeafAlloc(temp);
        buffer.setUpdateTimestamp(System.currentTimeMillis());
        buffer.setStep(nextStep);
        buffer.setMinStep(leafAlloc.getStep());//leafAlloc的step为DB中的step
    }
    // must set value before set max
    long value = leafAlloc.getMaxId() - buffer.getStep();
    segment.getValue().set(value); //设置初始的value
    segment.setMax(leafAlloc.getMaxId());//设置最大Id
    segment.setStep(buffer.getStep()); //设置步长step
    sw.stop("updateSegmentFromDb", key + " " + segment);
}
```

com.sankuai.inf.leaf.segment.dao.impl.IDAllocDaoImpl#updateMaxIdAndGetLeafAlloc

```java
public LeafAlloc updateMaxIdAndGetLeafAlloc(String tag) {
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
        sqlSession.update("com.sankuai.inf.leaf.segment.dao.IDAllocMapper.updateMaxId", tag);
        LeafAlloc result = sqlSession.selectOne("com.sankuai.inf.leaf.segment.dao.IDAllocMapper.getLeafAlloc", tag);
        sqlSession.commit();
        return result;
    } finally {
        sqlSession.close();
    }
}
```

com.sankuai.inf.leaf.segment.dao.IDAllocMapper#updateMaxId

```java
@Update("UPDATE leaf_alloc SET max_id = max_id + step WHERE biz_tag = #{tag}")
void updateMaxId(@Param("tag") String tag);
```

com.sankuai.inf.leaf.segment.dao.IDAllocMapper#getLeafAlloc

```java
@Select("SELECT biz_tag, max_id, step FROM leaf_alloc WHERE biz_tag = #{tag}")
@Results(value = {
        @Result(column = "biz_tag", property = "key"),
        @Result(column = "max_id", property = "maxId"),
        @Result(column = "step", property = "step")
})
LeafAlloc getLeafAlloc(@Param("tag") String tag);
```

com.sankuai.inf.leaf.segment.SegmentIDGenImpl#getIdFromSegmentBuffer

```java
public Result getIdFromSegmentBuffer(final SegmentBuffer buffer) {//初始化已经完成的情况下
    while (true) {
        buffer.rLock().lock();
        try {
          //获取当前使用的Segment
            final Segment segment = buffer.getCurrent();
            //如果下一个segment处于不可切换状态并且当前segment的号段已经使用了10%，则尝试加载新的号段
            if (!buffer.isNextReady() && (segment.getIdle() < 0.9 * segment.getStep()) && buffer.getThreadRunning().compareAndSet(false, true)) {
              //开启线程，加载新的segment
                service.execute(new Runnable() {
                    @Override
                    public void run() {
                        //填充下一个Segment
                        Segment next = buffer.getSegments()[buffer.nextPos()];
                        boolean updateOk = false;
                        try {
                            updateSegmentFromDb(buffer.getKey(), next);
                            updateOk = true;
                            logger.info("update segment {} from db {}", buffer.getKey(), next);
                        } catch (Exception e) {
                            logger.warn(buffer.getKey() + " updateSegmentFromDb exception", e);
                        } finally {
                            if (updateOk) {
                                buffer.wLock().lock();
                                buffer.setNextReady(true); //加载完毕
                                buffer.getThreadRunning().set(false); //标志无线程正在运行加载新的号段
                                buffer.wLock().unlock();
                            } else {
                                buffer.getThreadRunning().set(false);
                            }
                        }
                    }
                });
            }
            long value = segment.getValue().getAndIncrement();
            if (value < segment.getMax()) {
                return new Result(value, Status.SUCCESS);
            }
        } finally {
            buffer.rLock().unlock();
        }
       //等待segment加载完成
        waitAndSleep(buffer);
        buffer.wLock().lock();
        try {
            final Segment segment = buffer.getCurrent();
            long value = segment.getValue().getAndIncrement();
            if (value < segment.getMax()) {
                return new Result(value, Status.SUCCESS);
            }
            //当前号段已耗尽判断下一号段是否加载完毕
            if (buffer.isNextReady()) {
                //切换，将下一号段切换为当前号段
                buffer.switchPos();
                buffer.setNextReady(false);
            } else { //两个segment都不可用，抛出异常
                logger.error("Both two segments in {} are not ready!", buffer);
                return new Result(EXCEPTION_ID_TWO_SEGMENTS_ARE_NULL, Status.EXCEPTION);
            }
        } finally {
            buffer.wLock().unlock();
        }
    }
}
```

# References

https://github.com/Meituan-Dianping/Leaf

https://tech.meituan.com/2017/04/21/mt-leaf.html

https://tech.meituan.com/2019/03/07/open-source-project-leaf.html

