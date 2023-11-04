# SDS源码分析

* [概述](#概述)
* [组件](#组件)
  * [SdsClientFactory](#sdsclientfactory)
  * [CommonSdsClient](#commonsdsclient)
    * [初始化策略链条](#初始化策略链条)
      * [VisitStrategyExecutor](#visitstrategyexecutor)
      * [TokenBucketStrategyExecutor](#tokenbucketstrategyexecutor)
      * [ConcurrentStrategyExecutor](#concurrentstrategyexecutor)
      * [TimeoutStrategyExecutor](#timeoutstrategyexecutor)
      * [ExceptionStrategyExecutor](#exceptionstrategyexecutor)
      * [ExceptionRateStrategyExecutor](#exceptionratestrategyexecutor)
    * [初始化心跳服务](#初始化心跳服务)
      * [拉取降级点策略配置](#拉取降级点策略配置)
  * [SdsPowerfulCounterService](#sdspowerfulcounterservice)
  * [SlidingWindowData](#slidingwindowdata)
    * [初始化](#初始化)
* [降级入口](#降级入口)
  * [执行策略链](#执行策略链)
  * [降级逻辑](#降级逻辑)
* [降级出口](#降级出口)


# 概述

轻量级、简单、易用的限流、熔断、降级系统，能让Java应用做到自动限流、熔断和快速恢复，
提升应用整体的“弹性”

# 组件

## SdsClientFactory

com.didiglobal.sds.client.SdsClientFactory

```java
static {
    String appGroupName = System.getProperty(APP_GROUP_NAME);
    String appName = System.getProperty(APP_NAME);
    String serverAddrList = System.getProperty(SERVER_ADDR_LIST);

    if (StringUtils.isBlank(appGroupName) || StringUtils.isBlank(appName) || StringUtils.isBlank(serverAddrList)) {
        logger.info("SdsClientFactory#static 系统参数" + APP_GROUP_NAME + ", " + APP_NAME + ", " + SERVER_ADDR_LIST +
                "没配置全，不通过系统参数初始化SdsClient。");

    } else { //饿汉单例模式，利用类加载机制，创建AbstractSdsClient
        instance = buildSdsClient(appGroupName, appName, serverAddrList);
    }
}
```

## CommonSdsClient

创建调用链条，创建心跳服务，拉取策略信息；创建定时任务，定期拉取策略、发送心跳并上报统计信息

com.didiglobal.sds.client.SdsClientFactory#buildSdsClient

```java
private static AbstractSdsClient buildSdsClient(String appGroupName, String appName, String serverAddrList) {//创建CommonSdsClient
    AssertUtil.notBlack(appGroupName, "appGroupName can not black!");
    AssertUtil.notBlack(appName, "appName can not black!");
    AssertUtil.notBlack(serverAddrList, "serverAddrList can not black!");

    AbstractSdsClient sdsClient = new CommonSdsClient(appGroupName, appName, serverAddrList);
    logger.info("SdsClientFactory#buildSdsClient 创建SdsClient成功，sdsClient：" + sdsClient);

    return sdsClient;
}
```

com.didiglobal.sds.client.CommonSdsClient#CommonSdsClient

```java
protected CommonSdsClient(String appGroupName, String appName, String serverAddrList) {
    super(appGroupName, appName, serverAddrList);

    // 初始化策略链条
    initStrategyChain();

    // 初始化心跳线程
    initHeartBeat(appGroupName, appName, serverAddrList);
}
```

### 初始化策略链条

com.didiglobal.sds.client.CommonSdsClient#initStrategyChain

```java
private void initStrategyChain() {
    // 默认使用用户自定义的
    String customStrategyExecutorBuilderName = "";
    try {
      //JDK自带的SPI机制加载实现StrategyExecutorBuilder接口的类
        ServiceLoader<StrategyExecutorBuilder> strategyExecutorBuilders = ServiceLoader.load(StrategyExecutorBuilder.class);
        Iterator<StrategyExecutorBuilder> strategyExecutorBuilderIterator = strategyExecutorBuilders.iterator();
        // 注意这里用的是 if, 即只会找到第一个
        if (strategyExecutorBuilderIterator.hasNext()) {
            StrategyExecutorBuilder next = strategyExecutorBuilderIterator.next();
            customStrategyExecutorBuilderName = next.getClass().getName();
            //策略链条
            strategyExecutorChain = next.build();
        }
    } catch (Exception e) {
        // 如果构建策略链出错了, 打印 warn 日志提醒, 在下面重新构建策略链
        logger.warn("CommonSdsClient initStrategyChain occur error, it will use default strategy executor chain", e);
    }
    // 构建默认策略链
    if (strategyExecutorChain == null) {
        logger.info("CommonSdsClient use default strategy executor chain");
        strategyExecutorChain = new DefaultStrategyExecutorBuilder().build();
    } else {
        logger.info("CommonSdsClient use custom strategy executor chain: {}", customStrategyExecutorBuilderName);
    }
}
```

com.didiglobal.sds.client.strategy.DefaultStrategyExecutorBuilder#build

```java
public AbstractStrategyExecutor build() {//调用链模式
    return new VisitStrategyExecutor(
            new TokenBucketStrategyExecutor(
                    new ConcurrentStrategyExecutor(
                            new TimeoutStrategyExecutor(
                                    new ExceptionStrategyExecutor(
                                            new ExceptionRateStrategyExecutor(null)
                                    )))));
}
```

#### VisitStrategyExecutor

com.didiglobal.sds.client.strategy.executor.VisitStrategyExecutor#strategyCheck

```java
public boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {  //模板方法

    if (strategy.getVisitThreshold() == null || strategy.getVisitThreshold() < 0) {
        return true;
    }

    // 注意，这里一定要将降级量减去，否则一旦流量大于访问量阈值，将一直被降级下去
    return (checkData.getVisitCount() - checkData.getDowngradeCount()) <= strategy.getVisitThreshold();
}
```

####  TokenBucketStrategyExecutor

令牌桶

com.didiglobal.sds.client.strategy.executor.TokenBucketStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
    if (strategy.getTokenBucketGeneratedTokensInSecond() == null || strategy.
            getTokenBucketGeneratedTokensInSecond() < 0) {
        return true;
    }

    // 如果当前桶的令牌还没用完，那么直接返回
    if (checkData.getTakeTokenBucketNum() <= strategy.getTokenBucketGeneratedTokensInSecond()) {
        return true;
    }

    // 如果当前桶的令牌已经用完，但又没额外设置桶容量（即默认桶容量和桶每秒生成的令牌数相同），那就直接拒绝
    if (strategy.getTokenBucketSize() <= strategy.getTokenBucketGeneratedTokensInSecond()) {
        return false;
    }

    // 每个桶可用的令牌数不能超过桶容量
    if (checkData.getTakeTokenBucketNum() > strategy.getTokenBucketSize()) {
        return false;
    }

    // 如果当前秒的令牌已经不够用，那么就看历史桶中是否有剩余令牌能匀一下
    return CYCLE_BUCKET_NUM * BUCKET_TIME * strategy.getTokenBucketGeneratedTokensInSecond() - checkData.getTakeTokenBucketNum()
            + checkData.getDowngradeCount() > 0;
}
```

#### ConcurrentStrategyExecutor

com.didiglobal.sds.client.strategy.executor.ConcurrentStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {

    if(checkData.getConcurrentAcquire() == null) {
        return true;
    }

    /**
     * 由于并发策略使用了信号量，所以在统计的同时已经起到了策略判断的功能，所以这里直接返回判断结果
     */
    return checkData.getConcurrentAcquire();
}
```

####  TimeoutStrategyExecutor

超时请求数

com.didiglobal.sds.client.strategy.executor.TimeoutStrategyExecutor#strategyCheck

```java
public boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
    if (strategy.getTimeoutCountThreshold() == null || strategy.getTimeoutCountThreshold() < 0) {
        return true;
    }

    return checkData.getTimeoutCount() < strategy.getTimeoutCountThreshold();
}
```

#### ExceptionStrategyExecutor

异常请求数

com.didiglobal.sds.client.strategy.executor.ExceptionStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
    if (strategy.getExceptionThreshold() == null || strategy.getExceptionThreshold() < 0) {
        return true;
    }

    return checkData.getExceptionCount() < strategy.getExceptionThreshold();
}
```

#### ExceptionRateStrategyExecutor

异常请求占比

com.didiglobal.sds.client.strategy.executor.ExceptionRateStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
    if (strategy.getExceptionRateStart() == null || strategy.getExceptionRateStart() < 0 ||
            strategy.getExceptionRateThreshold() == null || strategy.getExceptionRateThreshold() < 0 ||
            strategy.getExceptionRateThreshold() > 100 ||
            strategy.getExceptionRateStart() >= checkData.getVisitCount()) {
        return true;
    }

    // 注意，在计算异常率时，分母不能直接使用访问量，需要把降级数量移除，因为降级后实际没有真正访问业务方法，不把降级数量移除将导致计算的异常率偏小
    long inputTotal = checkData.getVisitCount() - checkData.getDowngradeCount();

    return inputTotal == 0 || (checkData.getExceptionCount() + 0.0) * 100 / inputTotal < strategy.getExceptionRateThreshold();
}
```

### 初始化心跳服务

com.didiglobal.sds.client.CommonSdsClient#initHeartBeat

```java
private void initHeartBeat(String appGroupName, String appName, String serverAddrList) {
    SdsHeartBeatService.createOnlyOne(appGroupName, appName, serverAddrList);
}
```

com.didiglobal.sds.client.service.SdsHeartBeatService#createOnlyOne

```java
public static void createOnlyOne(String appGroupName, String appName, String serverAddrList) {
    if (instance == null) {
        synchronized (SdsHeartBeatService.class) {
            if (instance == null) {
                instance = new SdsHeartBeatService(appGroupName, appName, serverAddrList);

                instance.updatePointStrategyFromWebServer(); //启动之初，拉取降级点配置
            }
        }
    }
}
```

#### 拉取降级点策略配置

com.didiglobal.sds.client.service.SdsHeartBeatService#updatePointStrategyFromWebServer

```java
public void updatePointStrategyFromWebServer() {
		//请求参数
    Map<String, Object> param = new HashMap<>();
    HeartbeatRequest clientRequest = new HeartbeatRequest();
    clientRequest.setAppGroupName(appGroupName);
    clientRequest.setAppName(appName);
    clientRequest.setIp(IpUtils.getIp());
    clientRequest.setHostname(IpUtils.getHostname());
    clientRequest.setVersion(version);

    //获取客户端用到的所有降级点
    Enumeration<String> keys = SdsPowerfulCounterService.getInstance().getPointCounterMap().keys();
    List<String> pointList = new ArrayList<>();
    while(keys.hasMoreElements()) {
        pointList.add(keys.nextElement());
    }
    clientRequest.setPointList(pointList);

    param.put("client", JSON.toJSONString(clientRequest)); //客户端用到的降级点

    HeartBeatResponse response = null;
    String body = null;
    try {
        body = HttpUtils.post(getCurPullUrl(), param);
        logger.info("SdsHeartBeatService#updatePointStrategyFromWebServer 服务端应答：" + body);

        if (StringUtils.isNotBlank(body)) {
            response = JSON.parseObject(body, HeartBeatResponse.class);

        } else {
            logger.warn("SdsHeartBeatService#updatePointStrategyFromWebServer 服务端应答为空，请求参数："
                    + JSON.toJSONString(param));
            return;
        }

    } catch (Exception e) {
        /**
         * 抛异常则下次选取另一个地址
         */
        String curUrl = getCurPullUrl();
        String nextUrl = getNextPullUrl();
        logger.warn("SdsHeartBeatService#updatePointStrategyFromWebServer 请求服务端（" + curUrl + "）异常：" +
                body + ", 切换服务端地址为：" + nextUrl, e);
    }

    if (response == null) {
        logger.warn("SdsHeartBeatService#updatePointStrategyFromWebServer 服务端应答转JSON为null，请求参数："
                + JSON.toJSONString(param));
        return;
    }

    if (StringUtils.isNotBlank(response.getErrorMsg())) {
        logger.warn("SdsHeartBeatService#updatePointStrategyFromWebServer 服务端有错误信息：" + response.getErrorMsg());
        return;
    }

    // 如果没有更新，直接返回
    if (response.isChanged() == null || !response.isChanged()) {
        logger.info("SdsHeartBeatService#updatePointStrategyFromWebServer 版本号没变，无需更新");
        return;
    }

    String sdsSchemeName = response.getSdsSchemeName();
    version = response.getVersion();

    ConcurrentHashMap<String, SdsStrategy> strategies = new ConcurrentHashMap<>();

    if (response.getStrategies() != null && response.getStrategies().size() > 0) {
        for (SdsStrategy strategy : response.getStrategies()) {
            strategies.put(strategy.getPoint(), strategy);
        }
    }

    /**
     * 更新降级点信息
     */
    resetPointInfo(strategies);

    logger.info("SdsHeartBeatService#updatePointStrategyFromWebServer 重设降级点参数成功，当前降级预案：" + sdsSchemeName);
}
```

发送心跳请求

com.didiglobal.sds.client.service.SdsHeartBeatService#uploadHeartbeatData

```java
public void uploadHeartbeatData() {
    try {
        Date now = new Date();
				//本机统计信息
        Map<String, SdsCycleInfo> pointInfoMap = buildPointCycleInfo(now.getTime());
				//本机信息
        HeartbeatRequest clientRequest = new HeartbeatRequest();
        Map<String, Object> param = new HashMap<>();
        clientRequest.setAppGroupName(appGroupName);
        clientRequest.setAppName(appName);
        clientRequest.setIp(IpUtils.getIp());
        clientRequest.setHostname(IpUtils.getHostname());
        clientRequest.setStatisticsCycleTime(new Date(DateUtils.getLastCycleEndTime(CYCLE_BUCKET_NUM * BUCKET_TIME,
                now.getTime())));
        clientRequest.setPointInfoMap(pointInfoMap);
        param.put("client", JSON.toJSONString(clientRequest));

        logger.info("SdsHeartBeatService#uploadHeartbeatData 客户端请求参数：" + JSON.toJSONString(param));

        String body = null;
        try {
            body = HttpUtils.post(getCurUploadUrl(), param);
            logger.info("SdsHeartBeatService#uploadHeartbeatData 服务端应答：" + body);

        } catch (Exception e) {
            /**
             * 抛异常则下次选取另一个地址
             */
            String curUrl = getCurUploadUrl();
            String nextUrl = getNextUploadUrl();
            logger.warn("SdsHeartBeatService#uploadHeartbeatData 请求服务端（" + curUrl + "）异常：" +
                    body + ", 切换服务端地址为：" + nextUrl, e);
        }

    } catch (Throwable t) {
        logger.warn("SdsHeartBeatService#uploadHeartbeatData 心跳任务异常，可能无法及时上传数据", t);
    }
}
```

## SdsPowerfulCounterService

维护降级点的统计计数器

```java
//计数器Map，key-降级点名称，value-访问计数器对象
private final static ConcurrentHashMap<String, PowerfulCycleTimeCounter> pointCounterMap =
        new ConcurrentHashMap<>();
//饿汉单例模式
private final static SdsPowerfulCounterService counterService = new SdsPowerfulCounterService();
```

```java
public long getLastSecondVisitBucketValue(String point, long time) {
    PowerfulCycleTimeCounter counter = getOrCreatePoint(point);
    return counter.getLastSecondVisitBucketValue(time);
}
```

```java
public boolean concurrentAcquire(String point, long time) {
    PowerfulCycleTimeCounter counter = getOrCreatePoint(point);

    return counter.concurrentAcquire(time);
}
```

## SlidingWindowData

采用滑动窗口来进行数据统计

### 初始化

com.didiglobal.sds.client.counter.SlidingWindowData#SlidingWindowData()

```java
public SlidingWindowData() {
    this(BizConstant.CYCLE_NUM, BizConstant.CYCLE_BUCKET_NUM, 1);
}
```

```java
public SlidingWindowData(Integer cycleNum, Integer cycleBucketNum, Integer bucketTimeSecond) {
    this.cycleNum = cycleNum; //默认3个周期,30个桶
    this.cycleBucketNum = cycleBucketNum; //默认10，每个周期10s，一个周期10个桶
    this.bucketTimeSecond = bucketTimeSecond; //默认1s

    this.bucketSize = cycleNum * cycleBucketNum; //默认30
    this.bucketArray = new AtomicLongArray(bucketSize); //创建桶数组
}
```

com.didiglobal.sds.client.counter.SlidingWindowData#incrementAndGet

```java
public VisitWrapperValue incrementAndGet(long time) { //增加数据并获取
    int bucketIndex = getBucketIndexByTime(time); //获取所属的桶
    //当前秒内的数据
    long curSecondValue = bucketArray.incrementAndGet(bucketIndex);
    // 需要将该周期的所有秒数据统计出来再返回
    long slidingCycleValue = curSecondValue;
    for (int i = bucketIndex - cycleBucketNum + 1; i < bucketIndex; i++) {
        slidingCycleValue += bucketArray.get(switchIndex(i));
    }
    //当前秒内的数据、过去10秒内的数据
    return new VisitWrapperValue(curSecondValue, slidingCycleValue);
}
```

```java
private int getBucketIndexByTime(long time) {//获取当前桶（秒）所在的数组索引
    return (int) ((DateUtils.getSecond(time) / bucketTimeSecond) % bucketSize);
}
```



# 降级入口

com.didiglobal.sds.client.CommonSdsClient#shouldDowngrade

```java
public boolean shouldDowngrade(String point) {
    AssertUtil.notBlack(point, "降级点不能为空！");

    try {
        long now = System.currentTimeMillis();
        //记录开始时间
        TimeStatisticsUtil.startTime();
				//设置降级开始时间
        setDowngradeStartTime(now);
				//采用饿汉单例模式，维护了各降级点对应的计数器
        SdsPowerfulCounterService sdsPowerfulCounterService = SdsPowerfulCounterService.getInstance();
				//默认获取当前秒内的访问量、过去10秒内的访问量
        VisitWrapperValue visitWrapperValue = sdsPowerfulCounterService.visitInvokeAddAndGet(point, now);
        // 获取上一秒的访问量
        long lastSecondVisitCount = sdsPowerfulCounterService.getLastSecondVisitBucketValue(point, now);
        boolean concurrentAcquire = sdsPowerfulCounterService.concurrentAcquire(point, now);
        long exceptionCount = sdsPowerfulCounterService.getExceptionInvoke(point, now);
        long timeoutCount = sdsPowerfulCounterService.getTimeoutInvoke(point, now);
        long takeTokenBucketNum = sdsPowerfulCounterService.tokenBucketInvokeAddAndGet(point, now);
        long downgradeCount = sdsPowerfulCounterService.getCurCycleDowngrade(point, now);

        return judge(point, visitWrapperValue.getSlidingCycleValue(), visitWrapperValue.getBucketValue(),
                lastSecondVisitCount, concurrentAcquire, exceptionCount, timeoutCount, takeTokenBucketNum,
                downgradeCount, now);

    } catch (Exception e) {
        logger.warn("CommonSdsClient#downgradeEntrance " + point + " 异常", e);
        return false;
    }
}
```

com.didiglobal.sds.client.CommonSdsClient#judge

```java
private boolean judge(String point, long visitCount, long curSecondVisitCount, long lastSecondVisitCount,
                      boolean concurrentAcquire, long exceptionCount, long timeoutCount, long takeTokenBucketNum,
                      long downgradeCount, long time) {

    // 获取配置的策略：规则和触犯规则后的动作
    SdsStrategy strategy = SdsStrategyService.getInstance().getStrategy(point);
    /**
     * 没做策略或者做了策略但是降级比例小于等于0表示不需要降级
     */
    if (null == strategy || strategy.getDowngradeRate() <= 0) {
        return false;
    }

    DowngradeActionType downgradeActionType = null;
    //没有降级或者已经超过了降级时间
    if (!SdsDowngradeDelayService.getInstance().isDowngradeDelay(point, time)) {

        /**
         * 构建执行策略检查的参数
         */
        CheckData checkData = new CheckData();
        checkData.setPoint(point);
        checkData.setTime(time);
        checkData.setVisitCount(visitCount);
        checkData.setCurSecondVisitCount(curSecondVisitCount);
        checkData.setLastSecondVisitCount(lastSecondVisitCount);
        checkData.setConcurrentAcquire(concurrentAcquire);
        checkData.setExceptionCount(exceptionCount);
        checkData.setTimeoutCount(timeoutCount);
        checkData.setTakeTokenBucketNum(takeTokenBucketNum);
        checkData.setDowngradeCount(downgradeCount);

        /**
         * 执行策略链，如果通过了策略中所有的规则，则不需要降级
         */
        if ((downgradeActionType = strategyExecutorChain.execute(checkData, strategy)) == null) {
            return false;
        }

        /**
         * 访问量第一次超过阈值时，需要重设延迟日期
         * 注意：这里会有一个小问题，当降级延迟结束时，如果此时的调用量已经超过阈值，将不会立马出发新一次的降级延迟，不
         * 过即使没有出发新的降级延迟，接下来的请求由于超过阈值，也会处于降级状态中，所以影响不大
         */
        //            if (reqCount == threshhold + 1) {
        //                logger.debug("SdsClient 降级点重置延迟日期:" + point);
        // 如果访问量超过阈值，则需要去看该降级点是否有降级延迟，如果有配置降级延迟，则降级延迟开始
        SdsDowngradeDelayService.getInstance().resetExpireTime(point, time);

    } else {
        if (SdsDowngradeDelayService.getInstance().retryChoice(point, time)) { //处于降级中
            logger.info("CommonSdsClient 本次请求是降级延迟的重试请求：" + point);
            return false;
        }
    }

    boolean needDowngrade;
    /**
     * 大于等于100说明百分百降级
     */
    if (strategy.getDowngradeRate() >= 100) {
        needDowngrade = true;
    } else {
        needDowngrade = ThreadLocalRandom.current().nextInt(100) < strategy.getDowngradeRate();
    }
    if (needDowngrade) { //需要降级
        // 降级计数器+1
        addDowngradeCount(point, time, downgradeActionType);
    }
    return needDowngrade;
}
```

## 执行策略链

com.didiglobal.sds.client.strategy.executor.AbstractStrategyExecutor#execute

```java
public DowngradeActionType execute(CheckData checkData, SdsStrategy strategy) {
    boolean checkSuccess = strategyCheck(checkData, strategy);

    if (checkSuccess && next != null) {
        return next.execute(checkData, strategy);
    }

    if (!checkSuccess) {
        return getStrategyType();
    }

    return null;
}
```

## 降级逻辑

com.didiglobal.sds.client.CommonSdsClient#addDowngradeCount

```java
private void addDowngradeCount(String point, long time, DowngradeActionType downgradeActionType) {
    SdsPowerfulCounterService.getInstance().downgradeAddAndGet(point, time);//降级次数+1
    //通知降级后的自定义处理逻辑
    SdsDowngradeActionNotify.notify(point, downgradeActionType, new Date(time));
}
```

# 降级出口

统计业务耗时、并发数减1、超时更新统计信息

com.didiglobal.sds.client.CommonSdsClient#downgradeFinally

```java
public void downgradeFinally(String point) {
    try {
        // （非降级）业务处理耗时
        long consumeTime = TimeStatisticsUtil.getConsumeTime();

        // 并发数减1
        SdsPowerfulCounterService.getInstance().concurrentRelease(point);

        SdsStrategy strategy = SdsStrategyService.getInstance().getStrategy(point);
        if (strategy == null) {
            return;
        }

        Long timeoutThreshold = strategy.getTimeoutThreshold(); //超时时间
         //如果耗时超过设定的超时时间则超时计数器+1
        if (timeoutThreshold != null && timeoutThreshold > 0 && consumeTime > timeoutThreshold) {
            Long downgradeStartTimeValue = getDowngradeStartTime();
            if (downgradeStartTimeValue == null) {
                logger.warn("CommonSdsClient downgradeExit: 调用 downgradeFinally 之前请先调用 shouldDowngrade ！");
                return;
            }
						//增加降级量
            SdsPowerfulCounterService.getInstance().timeoutInvokeAddAndGet(point, downgradeStartTimeValue);
        }

    } finally {
        setDowngradeStartTime(null);
    }
}
```

