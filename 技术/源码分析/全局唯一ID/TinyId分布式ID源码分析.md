# TinyId源码分析

* [概述](#概述)
* [服务端批量生成Id](#服务端批量生成id)
  * [IdGenerator](#idgenerator)
    * [创建CachedIdGenerator](#创建cachedidgenerator)
    * [加载SegmentId](#加载segmentid)
  * [生成批量Id](#生成批量id)
    * [递增生成Id](#递增生成id)
    * [SegmentId初始化](#segmentid初始化)
    * [异步填充nextSegmentId](#异步填充nextsegmentid)
* [客户端本地生成Id](#客户端本地生成id)
* [服务端多数据源配置](#服务端多数据源配置)


# 概述

1、通过双Segment的方式提供请求批量Id，支持在服务端生成和客户端批量生成Id。
2、只支持号段模式（当前号段和备用号段，当前号段使用一定比例后，异步填充备用号段。使用号段，减轻了数据库压力）
3、支持多数据源，保证高可用

4、客户端请求服务端返回号段，在本地生成Id，减轻服务端的负载

# 服务端批量生成Id

com.xiaoju.uemc.tinyid.server.controller.IdContronller#nextId

```java
public Response<List<Long>> nextId(String bizType, Integer batchSize, String token) {//返回批量Id
    Response<List<Long>> response = new Response<>();
    Integer newBatchSize = checkBatchSize(batchSize); //默认最多返回10000个Id
    if (!tinyIdTokenService.canVisit(bizType, token)) {
        response.setCode(ErrorCode.TOKEN_ERR.getCode());
        response.setMessage(ErrorCode.TOKEN_ERR.getMessage());
        return response;
    }
    try {
      	//根据业务类型，获取对应的IdGenerator
        IdGenerator idGenerator = idGeneratorFactoryServer.getIdGenerator(bizType);
      	//服务端生成批量Id
        List<Long> ids = idGenerator.nextId(newBatchSize);
        response.setData(ids);
    } catch (Exception e) {
        response.setCode(ErrorCode.SYS_ERR.getCode());
        response.setMessage(e.getMessage());
        logger.error("nextId error", e);
    }
    return response;
}
```

## IdGenerator

**根据业务类型，获取对应的IdGenerator**

com.xiaoju.uemc.tinyid.base.factory.AbstractIdGeneratorFactory#getIdGenerator

```java
public IdGenerator getIdGenerator(String bizType) {
    if (generators.containsKey(bizType)) {
        return generators.get(bizType);
    }
    synchronized (this) {
        if (generators.containsKey(bizType)) {
            return generators.get(bizType);
        }
        IdGenerator idGenerator = createIdGenerator(bizType);
        generators.put(bizType, idGenerator);
        return idGenerator;
    }
}
```

### 创建CachedIdGenerator

com.xiaoju.uemc.tinyid.server.factory.impl.IdGeneratorFactoryServer#createIdGenerator

```java
public IdGenerator createIdGenerator(String bizType) {//创建IdGenerator
    logger.info("createIdGenerator :{}", bizType);
    return new CachedIdGenerator(bizType, tinyIdService);
}
```

com.xiaoju.uemc.tinyid.base.generator.impl.CachedIdGenerator#CachedIdGenerator

```java
public CachedIdGenerator(String bizType, SegmentIdService segmentIdService) {
    this.bizType = bizType;
    this.segmentIdService = segmentIdService;
    loadCurrent();//加载SegmentId
}
```

### 加载SegmentId


com.xiaoju.uemc.tinyid.base.generator.impl.CachedIdGenerator#loadCurrent

```java
public synchronized void loadCurrent() {
    if (current == null || !current.useful()) {
        if (next == null) {
            SegmentId segmentId = querySegmentId();
            this.current = segmentId;
        } else {
            current = next;
            next = null;
        }
    }
}
```

com.xiaoju.uemc.tinyid.base.generator.impl.CachedIdGenerator#querySegmentId

```java
private SegmentId querySegmentId() {
    String message = null;
    try {
        SegmentId segmentId = segmentIdService.getNextSegmentId(bizType);
        if (segmentId != null) {
            return segmentId;
        }
    } catch (Exception e) {
        message = e.getMessage();
    }
    throw new TinyIdSysException("error query segmentId: " + message);
}
```

com.xiaoju.uemc.tinyid.server.service.impl.DbSegmentIdServiceImpl#getNextSegmentId

```java
public SegmentId getNextSegmentId(String bizType) {
    // 获取nextTinyId的时候，有可能存在version冲突，需要重试
    for (int i = 0; i < Constants.RETRY; i++) {
        TinyIdInfo tinyIdInfo = tinyIdInfoDAO.queryByBizType(bizType);
        if (tinyIdInfo == null) {
            throw new TinyIdSysException("can not find biztype:" + bizType);
        }
        Long newMaxId = tinyIdInfo.getMaxId() + tinyIdInfo.getStep();
        Long oldMaxId = tinyIdInfo.getMaxId();
        int row = tinyIdInfoDAO.updateMaxId(tinyIdInfo.getId(), newMaxId, oldMaxId, tinyIdInfo.getVersion(),
                tinyIdInfo.getBizType());
        if (row == 1) { //乐观锁修改成功
            tinyIdInfo.setMaxId(newMaxId);
            SegmentId segmentId = convert(tinyIdInfo);
            logger.info("getNextSegmentId success tinyIdInfo:{} current:{}", tinyIdInfo, segmentId);
            return segmentId;
        } else {
            logger.info("getNextSegmentId conflict tinyIdInfo:{}", tinyIdInfo);
        }
    }
    throw new TinyIdSysException("get next segmentId conflict");
}
```

## 生成批量Id

com.xiaoju.uemc.tinyid.base.generator.impl.CachedIdGenerator#nextId(java.lang.Integer)

```java
public List<Long> nextId(Integer batchSize) {
    List<Long> ids = new ArrayList<>();
    for (int i = 0; i < batchSize; i++) {
        Long id = nextId();
        ids.add(id);
    }
    return ids;
}
```

com.xiaoju.uemc.tinyid.base.generator.impl.CachedIdGenerator#nextId()

```java
public Long nextId() {
    while (true) {
        if (current == null) {
            //从数据库加载currentSegment或者切换nextSegment为currentSegment
            loadCurrent();
            continue;
        }
        //获取可用的Id，按照增量递增
        Result result = current.nextId();
        //currentSegmentId已经耗尽，如果nextSegmentId不为空，将其切换为currentSegmentId，否则从数据加载
        if (result.getCode() == ResultCode.OVER) {
            loadCurrent();
        } else {
            //currentSegmentId使用量超过一定阈值，从数据库加载填充nextSegmentId
            if (result.getCode() == ResultCode.LOADING) {
                loadNext();
            }
            return result.getId();
        }
    }
}
```

### 递增生成Id

com.xiaoju.uemc.tinyid.base.entity.SegmentId#nextId

```java
public Result nextId() {
    init();
    long id = currentId.addAndGet(delta);//增量递增
    if (id > maxId) { //耗尽
        return new Result(ResultCode.OVER, id);
    }
    if (id >= loadingId) { //使用超过阈值，返回需要填充nextSegmentId的标志
        return new Result(ResultCode.LOADING, id);
    }
    return new Result(ResultCode.NORMAL, id); //正常返回
}
```

### SegmentId初始化

com.xiaoju.uemc.tinyid.base.entity.SegmentId#init

```java
 		/**官网描述
     * 这个方法主要为了1,4,7,10...这种序列准备的
     * 设置好初始值之后，会以delta的方式递增，保证无论开始id是多少都能生成正确的序列
     * 如当前是号段是(1000,2000]，delta=3, remainder=0，则经过这个方法后，currentId会先递增到1002,之后每次增加delta
     * 因为currentId会先递增，所以会浪费一个id，所以做了一次减delta的操作，实际currentId会从999开始增，第一个id还是1002
     */
public void init() {
    if (isInit) {
        return;
    }
    synchronized (this) {
        if (isInit) {
            return;
        }
        long id = currentId.get();
        if (id % delta == remainder) {
            isInit = true;
            return;
        }
        for (int i = 0; i <= delta; i++) {
            id = currentId.incrementAndGet();
            if (id % delta == remainder) {
                // 避免浪费 减掉系统自己占用的一个id
                currentId.addAndGet(0 - delta);
                isInit = true;
                return;
            }
        }
    }
}
```

### 异步填充nextSegmentId

com.xiaoju.uemc.tinyid.base.generator.impl.CachedIdGenerator#loadNext

```java
public void loadNext() {
    //currentSegmentId使用量超过阈值，从数据库加载填充nextSegmentId；只有当nextSegmentId为空并且当前没有其他线程正在加载
    if (next == null && !isLoadingNext) {
        synchronized (lock) {
            if (next == null && !isLoadingNext) {
                isLoadingNext = true; //标识开始加载
                executorService.submit(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            // 无论获取下个segmentId成功与否，都要将isLoadingNext赋值为false
                            next = querySegmentId();
                        } finally {
                            isLoadingNext = false; //标志加载结束
                        }
                    }
                });
            }
        }
    }
}
```

# 客户端本地生成Id

客户端复用了服务端生成Id的代码。客户端发送HTTP请求到服务端请求SegmentId，客户端本地本地生成Id,减轻了服务端的负载压力

com.xiaoju.uemc.tinyid.server.controller.IdContronller#nextSegmentId

```java
public Response<SegmentId> nextSegmentId(String bizType, String token) {//返回SegmentId
    Response<SegmentId> response = new Response<>();
    if (!tinyIdTokenService.canVisit(bizType, token)) {
        response.setCode(ErrorCode.TOKEN_ERR.getCode());
        response.setMessage(ErrorCode.TOKEN_ERR.getMessage());
        return response;
    }
    try {
      	//数据库中查询SegmentId
        SegmentId segmentId = segmentIdService.getNextSegmentId(bizType);
        response.setData(segmentId);
    } catch (Exception e) {
        response.setCode(ErrorCode.SYS_ERR.getCode());
        response.setMessage(e.getMessage());
        logger.error("nextSegmentId error", e);
    }
    return response;
}
```

# 服务端多数据源配置

## 配置多数据源

com.xiaoju.uemc.tinyid.server.config.DataSourceConfig#getDynamicDataSource

```java
public DataSource getDynamicDataSource() {
    DynamicDataSource routingDataSource = new DynamicDataSource();
    List<String> dataSourceKeys = new ArrayList<>();
    RelaxedPropertyResolver propertyResolver = new RelaxedPropertyResolver(environment, "datasource.tinyid.");
    //datasource.tinyid.names=primary,secondary
    String names = propertyResolver.getProperty("names");
    String dataSourceType = propertyResolver.getProperty("type");

    Map<Object, Object> targetDataSources = new HashMap<>(4);
    routingDataSource.setTargetDataSources(targetDataSources);
    routingDataSource.setDataSourceKeys(dataSourceKeys);
    // 多个数据源(primary->DataSource,secondary->DataSource)
    for (String name : names.split(SEP)) {
        Map<String, Object> dsMap = propertyResolver.getSubProperties(name + ".");
        DataSource dataSource = buildDataSource(dataSourceType, dsMap);
        buildDataSourceProperties(dataSource, dsMap);
        targetDataSources.put(name, dataSource);
        dataSourceKeys.add(name);
    }
    return routingDataSource;
}
```

## 随机选择数据源

com.xiaoju.uemc.tinyid.server.config.DynamicDataSource

```java
protected Object determineCurrentLookupKey () {
    if(dataSourceKeys.size() == 1) {
        return dataSourceKeys.get(0);
    }
    Random r = new Random();
    return dataSourceKeys.get(r.nextInt(dataSourceKeys.size()));
}
```

# References

https://github.com/didi/tinyid/wiki
