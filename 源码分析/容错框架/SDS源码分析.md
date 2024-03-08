# 概述

轻量级、简单、易用的限流、熔断、降级系统，能让Java应用做到自动限流、熔断和快速恢复，
提升应用整体的“弹性”

# 如何使用？

```yaml
 #Sds服务端地址列表，可以填写多个服务端地址，为了HA和Load Balance，多个地址用英文逗号分隔
serverAddrList:
#应用组名称
appGroupName:
#应用名称
appName:
```

```java
//不从sds-admin获取策略配置，完全使用本地手动设置
CycleDataService.setPullPointStrategySwitch(false);
//也不向sds-admin上报统计数据
CycleDataService.setUploadDataSwitch(false);

//设置本地降级策略
SdsStrategy createOrderSdsStrategy = new SdsStrategy();
createOrderSdsStrategy.setPoint("降级点名称");
// 使用访问量降级，这里意味着在10s的滑动窗口中最多只能访问1000次业务方法
createOrderSdsStrategy.setVisitThreshold(1000L);
// 如果超过设定的阈值，降级比例是100%
createOrderSdsStrategy.setDowngradeRate(100);
SdsStrategyService.getInstance().resetOne("降级点名称", reateOrderSdsStrategy);
```

```java
//单例，线程安全
private static SdsClient sdsClient = SdsClientFactory.getOrCreateSdsClient("appGroupName", "appName", serverAddrList);
```

```java
try {
  
  if (sdsClient.shouldDowngrade("降级点名称")) {
    return false;
  }

  // 模拟正常业务逻辑耗时
  sleepMilliseconds(getTakeTime());

} catch (Exception e) {
  
  // 这里用于统计异常量
  sdsClient.exceptionSign(getPoint(), e);
  // 记得捕获完还得抛出去
  throw e;

} finally {
  
  // 回收资源，一般在finally代码中调用
  sdsClient.downgradeFinally(getPoint());
}
```

com.didiglobal.sds.easy.SdsEasyUtil#invokerMethod

```java
public static <T> T invokerMethod(String point, T downgradeValue, BizFunction<T> bizFunction) {
  	//bizFunction:业务方法
  	//downgradeValue：降级后返回的值
    if (bizFunction == null) {
        return downgradeValue;
    }

    /**
     * 如果降级点为空，那么就相当于没有降级功能
     */
    if (StringUtils.isBlank(point)) {
        return bizFunction.invokeBizMethod();
    }

    SdsClient sdsClient = SdsClientFactory.getSdsClient();

    // 如果客户端没初始化，那么直接执行业务方法
    if (sdsClient == null) {
        return bizFunction.invokeBizMethod();
    }

    try {
        if (sdsClient.shouldDowngrade(point)) {
            return downgradeValue;
        }

        return bizFunction.invokeBizMethod();

    } catch (Throwable e) {
        sdsClient.exceptionSign(point, e);
        throw e;

    } finally {
        sdsClient.downgradeFinally(point);
    }
}
```

```java
@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SdsDowngradeMethod {

    /**
     * 降级点
     */
    String point();

    /**
     * 异常类型
     */
    Class<?> exceptionClass() default Exception.class;

    /**
     * 出现异常后的降级方法名称，和使用注解标注的方法在同一个类中，注意这里有降级的优先级顺序
     * fallback 指定的方法 > sds-admin 配置的策略 > 默认（抛出异常）
     *
     * @return 方法名称
     */
    String fallback() default "";
}
```

```java
@Pointcut("@annotation(com.didiglobal.sds.client.annotation.SdsDowngradeMethod)")
public void pointcut() {
}
```

com.didiglobal.sds.aspectj.SdsPointAspect#invoke

```java
//降级方法和注解标注的方法要在同一个类上
//方法名称必须是SdsDowngradeMethod中的 fallback() 指定的
//降级方法的入参类型必须和限流方法的保持一致或者可以多添加一个参数（类型必须为SdsException，且必须放在最后一个位置）
@Around("pointcut()")
public Object invoke(ProceedingJoinPoint pjp) throws Throwable {

    MethodSignature signature = (MethodSignature) pjp.getSignature();
    Class<?> targetClass = pjp.getTarget().getClass();

    Method originMethod = getDeclaredMethod(targetClass, signature.getName(),
            signature.getMethod().getParameterTypes());
    if (originMethod == null) {
        throw new IllegalStateException(
                "获取类方法失败，class：" + targetClass.getName() + ", method:" + signature.getMethod().getName());
    }

    SdsDowngradeMethod sdsDowngradeMethodAnnotation = originMethod.getAnnotation(SdsDowngradeMethod.class);
    if (sdsDowngradeMethodAnnotation == null) {
        throw new IllegalStateException("这不应该发生，请联系管理员！！");
    }

    SdsClient sdsClient = SdsClientFactory.getSdsClient();
    if (sdsClient == null) {
        return pjp.proceed();
    }

    String point = sdsDowngradeMethodAnnotation.point();
    try {
        // 如果需要被降级，那么直接抛异常，不执行业务方法
        if (sdsClient.shouldDowngrade(point)) {
            throw new SdsException(point);
        }

        return pjp.proceed();

    } catch (SdsException ex) {
        Object downgradeReturnValue = null;
        // 优先判断是否被 fallback 处理
        if (!StringUtils.isBlank(sdsDowngradeMethodAnnotation.fallback())) {
            downgradeReturnValue = this.handleFallback(pjp, ex);
            if (downgradeReturnValue != null) {
                return downgradeReturnValue;
            }
        }
        // 再判断是否被 sds-admin 降级处理
        downgradeReturnValue = SdsDowngradeReturnValueService.getInstance().getDowngradeReturnValue(point, originMethod.getGenericReturnType());
        if (downgradeReturnValue != null) {
            return downgradeReturnValue;
        }
        // 如果都没有处理，使用默认策略抛出异常
        ex.setPoint(point);
        throw ex;

    } catch (Throwable throwable) {
        // 这里用于统计异常量
        sdsClient.exceptionSign(point, throwable);

        throw throwable;

    } finally {
        sdsClient.downgradeFinally(point);
    }
}
```

```java
private Object handleFallback(ProceedingJoinPoint pjp, SdsException ex) {
    MethodSignature signature = (MethodSignature) pjp.getSignature();
    Class<?> targetClass = pjp.getTarget().getClass();
    Class<?>[] parameterTypes = signature.getMethod().getParameterTypes();
    Object[] args = pjp.getArgs();
    Method originMethod = this.getDeclaredMethod(targetClass, signature.getName(), parameterTypes);
    // 上面已经判断过，不会发生 NPE
    SdsDowngradeMethod sdsDowngradeMethodAnnotation = originMethod.getAnnotation(SdsDowngradeMethod.class);
    // 获取降级方法名称
    String fallbackMethodName = sdsDowngradeMethodAnnotation.fallback();
    // 优先查找带异常的方法
    Method fallbackMethod = null;
    Class<?>[] newParameterTypes = Arrays.copyOf(parameterTypes, parameterTypes.length + 1);
    newParameterTypes[newParameterTypes.length - 1] = ex.getClass();
    Object[] newArgs = Arrays.copyOf(args, args.length + 1);
    newArgs[newArgs.length - 1] = ex;
    fallbackMethod = this.getDeclaredMethod(targetClass, fallbackMethodName, newParameterTypes);
    if (fallbackMethod == null) {
        // 如果没有查找到，则查找不带异常的方法，注意这里要把参数变回来，即去掉 SdsException
        newArgs = args;
        fallbackMethod = this.getDeclaredMethod(targetClass, fallbackMethodName, parameterTypes);
    }
    if (fallbackMethod == null) {
        logger.warn("找不到 {} 对应的降级方法，请检查您的注解配置", fallbackMethodName);
        return null;
    }
    try {
        return fallbackMethod.invoke(pjp.getThis(), newArgs);
    } catch (Exception e) {
        logger.error("fallbackMethod 处理失败", e);
    }
    return null;
}
```

# 整合springboot

引入sds-spring-boot-starter,自动注入SdsProperties和SdsClient

```java
package com.didiglobal.sds.extension.spring.boot.autoconfigure;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * 对应properties文件的：
 * sds.app.group.name, sds.app.name, sds.server.addr.list
 */
@ConfigurationProperties("sds")
public class SdsProperties {

    private String appGroupName;

    private String appName;

    private String serverAddrList;

    public String getAppGroupName() {
        return appGroupName;
    }

    public void setAppGroupName(String appGroupName) {
        this.appGroupName = appGroupName;
    }

    public String getAppName() {
        return appName;
    }

    public void setAppName(String appName) {
        this.appName = appName;
    }

    public String getServerAddrList() {
        return serverAddrList;
    }

    public void setServerAddrList(String serverAddrList) {
        this.serverAddrList = serverAddrList;
    }
}
```

```java
@Configuration
@EnableConfigurationProperties({SdsProperties.class})
public class SdsAutoConfiguration {

    private Logger logger = SdsLoggerFactory.getDefaultLogger();

    @Bean
    @ConditionalOnMissingBean
    public SdsClient sdsClient(SdsProperties sdsProperties) {
        SdsClient sdsClient = SdsClientFactory.getOrCreateSdsClient(sdsProperties.getAppGroupName(), sdsProperties.getAppName(),
                sdsProperties.getServerAddrList());
        logger.info("SdsAutoConfiguration create SdsClient with sdsProperties:{}", sdsProperties);
        return sdsClient;
    }
}
```

# 初始化

com.didiglobal.sds.client.SdsClientFactory

```java
// volatile 防止重排序
private volatile static AbstractSdsClient instance = null;
static {
  	//关键的配置信息
    String appGroupName = System.getProperty(APP_GROUP_NAME);
    String appName = System.getProperty(APP_NAME);
    String serverAddrList = System.getProperty(SERVER_ADDR_LIST);

    if (StringUtils.isBlank(appGroupName) || StringUtils.isBlank(appName) || StringUtils.isBlank(serverAddrList)) {
        logger.info("SdsClientFactory#static 系统参数" + APP_GROUP_NAME + ", " + APP_NAME + ", " + SERVER_ADDR_LIST +
                "没配置全，不通过系统参数初始化SdsClient。");

    } else { 
      	//饿汉单例模式，利用类加载机制，创建AbstractSdsClient
        instance = buildSdsClient(appGroupName, appName, serverAddrList);
    }
}
```

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
  //VisitStrategyExecutor最先执行
    return new VisitStrategyExecutor(
            new TokenBucketStrategyExecutor(
                    new ConcurrentStrategyExecutor(
                            new TimeoutStrategyExecutor(
                                    new ExceptionStrategyExecutor(
                                            new ExceptionRateStrategyExecutor(null)
                                    )))));
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
								//启动之初，拉取降级点配置
                instance.updatePointStrategyFromWebServer(); 
            }
        }
    }
}
```

com.didiglobal.sds.client.service.SdsHeartBeatService#SdsHeartBeatService

```java
private SdsHeartBeatService(String appGroupName, String appName, String serverAddrList) {
    AssertUtil.notBlack(appGroupName, "appGroupName不能为空！");
    AssertUtil.notBlack(appName, "appName不能为空！");
    AssertUtil.notBlack(serverAddrList, "serverAddrList不能为空！");

    this.appGroupName = appGroupName;
    this.appName = appName;

    buildServerUrl(serverAddrList);
}
```

```java
private void buildServerUrl(String serverAddrList) {
    String[] urlArray = serverAddrList.split(",");
    for (String url : urlArray) {
        if (StringUtils.isBlank(url)) {
            continue;
        }

        url = url.trim();

        SERVER_UPLOAD_URL_LIST.add(url.endsWith("/") ? url + UPLOAD_HEARTBEAT_PATH : url + "/" +
                UPLOAD_HEARTBEAT_PATH);
        SERVER_PULL_URL_LIST.add(url.endsWith("/") ? url + PULL_POINT_STRATEGY_PATH : url + "/" +
                PULL_POINT_STRATEGY_PATH);
    }

    if (SERVER_UPLOAD_URL_LIST.isEmpty()) {
        logger.error("SdsHeartBeatService 服务端地址列表为空，请设置服务端地址列表！");
        return;
    }

    //通过随机打乱的方式来实现LB
    Collections.shuffle(SERVER_UPLOAD_URL_LIST);
    Collections.shuffle(SERVER_PULL_URL_LIST);

    logger.info("SdsHeartBeatService 新增Upload服务端地址列表：" + JSON.toJSONString(SERVER_UPLOAD_URL_LIST));
    logger.info("SdsHeartBeatService 新增Pull服务端地址列表：" + JSON.toJSONString(SERVER_PULL_URL_LIST));
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

# 链路调用

com.didiglobal.sds.client.strategy.executor.AbstractStrategyExecutor#execute

```java
public abstract class AbstractStrategyExecutor {

  //模板方法
  public DowngradeActionType execute(CheckData checkData, SdsStrategy strategy) {
      //策略检查
      boolean checkSuccess = strategyCheck(checkData, strategy); //具体实现

      if (checkSuccess && next != null) {
          //执行一个链路
          return next.execute(checkData, strategy);
      }

      if (!checkSuccess) {
          //返回降级类型
          return getStrategyType();
      }

      //没有发生降级
      //感觉应该也设置枚举类型，表示未降级；DowngradeActionType.NONE(0，"未降级")
      return null; 
  }
}
```

## 访问量限流

com.didiglobal.sds.client.strategy.executor.VisitStrategyExecutor#strategyCheck

```java
//CheckData从滑动窗口获取
//限制访问量，基于滑动窗口的限流
public boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {  

    if (strategy.getVisitThreshold() == null || strategy.getVisitThreshold() < 0) {
        //未设置降级策略
      	return true; 
    }

    // 注意，这里一定要将降级量减去，否则一旦流量大于访问量阈值，将一直被降级下去
    return (checkData.getVisitCount() - checkData.getDowngradeCount()) <= strategy.getVisitThreshold();
}
```

## 令牌桶限流

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

## 并发限流

com.didiglobal.sds.client.strategy.executor.ConcurrentStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {

    if(checkData.getConcurrentAcquire() == null) {
        return true;
    }

    //由于并发策略使用了信号量，所以在统计的同时已经起到了策略判断的功能，所以这里直接返回判断结果
    return checkData.getConcurrentAcquire();
}
```

## 超时限流

com.didiglobal.sds.client.strategy.executor.TimeoutStrategyExecutor#strategyCheck

```java
public boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
    if (strategy.getTimeoutCountThreshold() == null || strategy.getTimeoutCountThreshold() < 0) {
        return true;
    }

    return checkData.getTimeoutCount() < strategy.getTimeoutCountThreshold();
}
```

## 异常限流

### 异常请求数限流

com.didiglobal.sds.client.strategy.executor.ExceptionStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
    if (strategy.getExceptionThreshold() == null || strategy.getExceptionThreshold() < 0) {
        return true;
    }

    return checkData.getExceptionCount() < strategy.getExceptionThreshold();
}
```

### 异常请求占比限流

com.didiglobal.sds.client.strategy.executor.ExceptionRateStrategyExecutor#strategyCheck

```java
protected boolean strategyCheck(CheckData checkData, SdsStrategy strategy) {
  //exceptionRateStart:异常率计算的起始访问量;当请求数比较少时，不会计算异常率
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
				//增加访问量并返回当前的访问量
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

         //构建执行策略检查的参数
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

         //执行策略链，如果通过了策略中所有的规则，则不需要降级
         //null，表示不降级
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
     		 //处于降级中
        if (SdsDowngradeDelayService.getInstance().retryChoice(point, time)) { 
            logger.info("CommonSdsClient 本次请求是降级延迟的重试请求：" + point);
            return false;
        }
    }
		
    //执行策略，返回降级
  
    boolean needDowngrade;
  	//大于等于100说明百分百降级
    if (strategy.getDowngradeRate() >= 100) { 
        needDowngrade = true;
    } else {
        needDowngrade = ThreadLocalRandom.current().nextInt(100) < strategy.getDowngradeRate();
    }
  		//需要降级
    if (needDowngrade) { 
        // 降级计数器+1
        addDowngradeCount(point, time, downgradeActionType);
    }
    return needDowngrade;
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

统计业务耗时，超时更新统计信息

并发数减1

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
				//策略中的超时时间
        Long timeoutThreshold = strategy.getTimeoutThreshold(); 
        //如果耗时超过设定的超时时间，则超时计数器+1
        if (timeoutThreshold != null && timeoutThreshold > 0 && consumeTime > timeoutThreshold) {
            //执行的开始时间
            Long downgradeStartTimeValue = getDowngradeStartTime();
            if (downgradeStartTimeValue == null) {
                logger.warn("CommonSdsClient downgradeExit: 调用 downgradeFinally 之前请先调用 shouldDowngrade ！");
                return;
            }
						//增加超时计数
            SdsPowerfulCounterService.getInstance().timeoutInvokeAddAndGet(point, downgradeStartTimeValue);
        }

    } finally {
        setDowngradeStartTime(null);
    }
}
```

#  降级通知类

com.didiglobal.sds.client.config.SdsDowngradeActionNotify#addDowngradeActionListener

```java
//维护监听器的数据结构，CopyOnWriteArrayList适用于读多写少的场景
private final static CopyOnWriteArrayList<DowngradeActionListener> listeners = new CopyOnWriteArrayList<>(); 
```

```java
//添加监听器
public static boolean addDowngradeActionListener(DowngradeActionListener downgradeActionListener) {
  
    if (downgradeActionListener == null) {
        return false;
    }

    if (listeners.size() > 100) {
        logger.warn("SdsDowngradeActionNotify#addDowngradeActionListener 难道你在死循环调用这个方法？");
        return false;
    }

    return listeners.add(downgradeActionListener);
}
```

```java
//类加载时，初始化线程池调用监听器
private final static ExecutorService notifyPool = new ThreadPoolExecutor(10, 10, 1L,
        TimeUnit.MINUTES, new ArrayBlockingQueue<Runnable>(1000),
        new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r, "SdsDowngradeActionNotify");
                thread.setDaemon(true);
                return thread;
            }
        },
        new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                logger.warn("SdsDowngradeActionNotify notifyPool已经满了，只能将该降级Action时间丢弃。");
            }
        }
);
```

```java
//降级通知
public static void notify(final String point, final DowngradeActionType downgradeActionType, final Date time) {
    try {
        notifyPool.execute(new Runnable() {
            @Override
            public void run() {
                for (DowngradeActionListener listener : listeners) {
                    try {
                        listener.downgradeAction(point, downgradeActionType, time);
                    } catch (Exception e) {//监听器异常
                        logger.warn("SdsDowngradeActionNotify#notify 监听器执行时产生了异常，downgradeActionType:" +
                                downgradeActionType + ", time:" + time + ", point:" + point, e);
                    }
                }
            }
        });

    } catch (Exception e) { //提交任务异常
        logger.warn("SdsDowngradeActionNotify#notify 提交降级事件任务时产生了异常，downgradeActionType:" +
                downgradeActionType + ", time:" + time + ", point:" + point, e);
    }
}
```

# 滑动窗口

com.didiglobal.sds.client.counter.SlidingWindowData#SlidingWindowData(java.lang.Integer, java.lang.Integer, java.lang.Integer)

```java
public SlidingWindowData(Integer cycleNum, Integer cycleBucketNum, Integer bucketTimeSecond) {
    this.cycleNum = cycleNum; //默认3个周期
    this.cycleBucketNum = cycleBucketNum; //每个周期10个桶，每个周期10s
    this.bucketTimeSecond = bucketTimeSecond; //默认每个桶1s

    this.bucketSize = cycleNum * cycleBucketNum; //默认30个桶
    this.bucketArray = new AtomicLongArray(bucketSize); //创建桶数组
}
```

```java
public VisitWrapperValue incrementAndGet(long time) {
  	//获取所属的桶
    int bucketIndex = getBucketIndexByTime(time); 
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
//根据时间（秒）获取所属的桶的下标
private int getBucketIndexByTime(long time) {
    //默认，bucketTimeSecond=1，bucketSize=30
    return (int) ((DateUtils.getSecond(time) / bucketTimeSecond) % bucketSize);
}
```



```java
 //设置桶内数据
@Override
public void setBucketValue(long time, Long value) {
  	//根据time获取所属的桶
    int bucketIndex = getBucketIndexByTime(time);
    bucketArray.set(bucketIndex, value);
}
```

```java
//获取桶内数据
@Override
public long getBucketValue(long time) { 
    int bucketIndex = getBucketIndexByTime(time);
    return bucketArray.get(bucketIndex);
}
```

```java
//默认情况下，获取过去10s内的数据总和
@Override
public long getCurSlidingCycleValue(long time) {
    return getSlidingCycleBucketTotalValue(getBucketIndexByTime(time));
}
```



```java
private long getSlidingCycleBucketTotalValue(int bucketIndex) {
    long total = bucketArray.get(bucketIndex);

    for (int i = bucketIndex - cycleBucketNum + 1; i < bucketIndex; i++) {
        total += bucketArray.get(switchIndex(i));
    }

    return total;
}
```

```java
//由于需要把桶（秒）的数组打造成环形数组，所以这里需要对index做特殊处理
private int switchIndex(int index) {
  	//坏味道
    if (index >= 0 && index < bucketSize) {
        return index;

    } else if (index < 0) {
        return bucketSize + index;

    } else {
        return index - bucketSize;
    }
}
//好味道实现
private int switchIndexGoodTaste(int index) {
  	
    if (index >= 0 && index < bucketSize) {
        return index;
    } 
  	if (index < 0) {
        return bucketSize + index;
    } 
  	if (index > bucketSize){
        return index - bucketSize;
    }
}
```
