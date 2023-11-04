# Elastic-Job分布式任务调度源码分析

* [启动](#启动)
  * [创建JobScheduler](#创建jobscheduler)
  * [初始化作业](#初始化作业)
    * [创建调度器](#创建调度器)
    * [创建JobDetail](#创建jobdetail)
    * [调度作业](#调度作业)
* [执行作业](#执行作业)
  * [作业开始执行的入口](#作业开始执行的入口)
  * [作业执行的模板方法](#作业执行的模板方法)
    * [获取执行分片](#获取执行分片)
    * [等待Leader分配分片](#等待leader分配分片)
    * [正常运行分片](#正常运行分片)
    * [错过任务重执行](#错过任务重执行)
    * [失效转移](#失效转移)
  * [](#)


# 启动

## 创建JobScheduler

io.elasticjob.lite.api.JobScheduler#JobScheduler(io.elasticjob.lite.reg.base.CoordinatorRegistryCenter, io.elasticjob.lite.config.LiteJobConfiguration, io.elasticjob.lite.event.JobEventBus, io.elasticjob.lite.api.listener.ElasticJobListener...)

```java
   private JobScheduler(final CoordinatorRegistryCenter regCenter, final LiteJobConfiguration liteJobConfig, final JobEventBus jobEventBus, final ElasticJobListener... elasticJobListeners) {
        JobRegistry.getInstance().addJobInstance(liteJobConfig.getJobName(), new JobInstance()); //本地注册JobInstance（jobName->JobInstance）
        this.liteJobConfig = liteJobConfig; //作业的配置信息
        this.regCenter = regCenter; //zk通信，负责与zk相关的操作
        List<ElasticJobListener> elasticJobListenerList = Arrays.asList(elasticJobListeners);//自定义的监听器，添加作业执行前的执行的方法、作业执行后的执行的方法
        setGuaranteeServiceForElasticJobListeners(regCenter, elasticJobListenerList);
        schedulerFacade = new SchedulerFacade(regCenter, liteJobConfig.getJobName(), elasticJobListenerList);
        jobFacade = new LiteJobFacade(regCenter, liteJobConfig.getJobName(), Arrays.asList(elasticJobListeners), jobEventBus);//为作业提供内部服务的门面类
    }
```

## 初始化作业

io.elasticjob.lite.api.JobScheduler#init

```java
public void init() {
  LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig); //更新job的配置信息到zk
  //设置作业的分片数
 JobRegistry.getInstance().setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(), liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
  //作业控制器（创建调度器、创建JobDetail）
  JobScheduleController jobScheduleController = new JobScheduleController(
    createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
  //jobName-> jobScheduleController,jobName ->regCenter
  JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
  schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
  //开始作业调度
 jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
}
```

### 创建调度器

io.elasticjob.lite.api.JobScheduler#createScheduler

```java
private Scheduler createScheduler() { //创建调度器
    Scheduler result;
    try {
        StdSchedulerFactory factory = new StdSchedulerFactory();
        factory.initialize(getBaseQuartzProperties());
        result = factory.getScheduler();
     		result.getListenerManager()
           				.addTriggerListener(schedulerFacade.newJobTriggerListener());
    } catch (final SchedulerException ex) {
        throw new JobSystemException(ex);
    }
    return result;
}
```

### 创建JobDetail

io.elasticjob.lite.api.JobScheduler#createJobDetail

```java
private JobDetail createJobDetail(final String jobClass) {   //创建JobDetail
    //LiteJob作业执行的模板，
    JobDetail result = JobBuilder.newJob(LiteJob.class).withIdentity(liteJobConfig.getJobName()).build();
    result.getJobDataMap().put(JOB_FACADE_DATA_MAP_KEY, jobFacade); //注入到LiteJob
    Optional<ElasticJob> elasticJobInstance = createElasticJobInstance();
    if (elasticJobInstance.isPresent()) {
        result.getJobDataMap().put(ELASTIC_JOB_DATA_MAP_KEY, elasticJobInstance.get());
    } else if (!jobClass.equals(ScriptJob.class.getCanonicalName())) {
        try {
            result.getJobDataMap().put(ELASTIC_JOB_DATA_MAP_KEY, Class.forName(jobClass).newInstance()); //创建自定义的ElasticJob实例，注入到LiteJob
        } catch (final ReflectiveOperationException ex) {
            throw new JobConfigurationException("Elastic-Job: Job class '%s' can not initialize.", jobClass);
        }
    }
    return result;
}
```

### 调度作业

io.elasticjob.lite.internal.schedule.JobScheduleController#scheduleJob

```java
public void scheduleJob(final String cron) { //调度作业
    try {
        if (!scheduler.checkExists(jobDetail.getKey())) {
            scheduler.scheduleJob(jobDetail, createTrigger(cron));
        }
        scheduler.start();
    } catch (final SchedulerException ex) {
        throw new JobSystemException(ex);
    }
}
```

# 执行作业

## 作业开始执行的入口

io.elasticjob.lite.internal.schedule.LiteJob#execute

```java
@Override
public void execute(final JobExecutionContext context) throws JobExecutionException {
    JobExecutorFactory.getJobExecutor(elasticJob, jobFacade).execute();
}
```

## 作业执行的模板方法

io.elasticjob.lite.executor.AbstractElasticJobExecutor#execute()

```java
public final void execute() {
    try {
        //检查本机与注册中心的时间误差秒数是否在允许范围.
        jobFacade.checkJobExecutionEnvironment();
    } catch (final JobExecutionEnvironmentException cause) {
        jobExceptionHandler.handleException(jobName, cause);
    }
    //从zk获取分片信息
    ShardingContexts shardingContexts = jobFacade.getShardingContexts();
    //发送TASK_STAGING类型的事件
    if (shardingContexts.isAllowSendJobEvent()) {
        jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_STAGING, String.format("Job '%s' execute begin.", jobName));
    }
    if (jobFacade.misfireIfRunning(shardingContexts.getShardingItemParameters().keySet())) {
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_FINISHED, String.format(
                    "Previous job '%s' - shardingItems '%s' is still running, misfired job will start after previous job completed.", jobName, 
                    shardingContexts.getShardingItemParameters().keySet()));
        }
        return;
    }
    try {
        //分片运行前之执行的操作
        jobFacade.beforeJobExecuted(shardingContexts);
        //CHECKSTYLE:OFF
    } catch (final Throwable cause) {
        //CHECKSTYLE:ON
        jobExceptionHandler.handleException(jobName, cause);
    }
    //正常执行分片
    execute(shardingContexts, JobExecutionEvent.ExecutionSource.NORMAL_TRIGGER);
    //查看是否有错失执行的任务
    while (jobFacade.isExecuteMisfired(shardingContexts.getShardingItemParameters().keySet())) {
        ///移除错失执行的分片
        jobFacade.clearMisfire(shardingContexts.getShardingItemParameters().keySet());
        //执行错失执行的分片
        execute(shardingContexts, JobExecutionEvent.ExecutionSource.MISFIRE);
    }
    //如开启了失败转移，执行失败的分片将再次被执行，主要针对节点失败
    jobFacade.failoverIfNecessary();
    try {
        //分片运行结束后执行的操作
        jobFacade.afterJobExecuted(shardingContexts);
        //CHECKSTYLE:OFF
    } catch (final Throwable cause) {
        //CHECKSTYLE:ON
        jobExceptionHandler.handleException(jobName, cause);
    }
}
```

### 获取执行分片

io.elasticjob.lite.internal.schedule.LiteJobFacade#getShardingContexts

```java
public ShardingContexts getShardingContexts() {
    //是否配置了失效转移
    boolean isFailover = configService.load(true).isFailover();
    if (isFailover) {
        //获取本机执行失败的分片
        List<Integer> failoverShardingItems = failoverService.getLocalFailoverItems();
        if (!failoverShardingItems.isEmpty()) {
            return executionContextService.getJobShardingContext(failoverShardingItems); //创建ShardingContexts
        }
    }
    //执行前由主节点进行的分片的分配，从节点等待作业主节点分片完成
    shardingService.shardingIfNecessary();
    //获取分配给本机的分片
    List<Integer> shardingItems = shardingService.getLocalShardingItems();
    if (isFailover) {
        shardingItems.removeAll(failoverService.getLocalTakeOffItems());
    }
    //删除禁用的分片项
    shardingItems.removeAll(executionService.getDisabledItems(shardingItems));
    //创建ShardingContexts
    return executionContextService.getJobShardingContext(shardingItems);
}
```

### 等待Leader分配分片

io.elasticjob.lite.internal.sharding.ShardingService#shardingIfNecessary

```java
public void shardingIfNecessary() {
    List<JobInstance> availableJobInstances = instanceService.getAvailableJobInstances();
    if (!isNeedSharding() || availableJobInstances.isEmpty()) {
        return;
    }
     //选举leader负责分片的分配，其他节点阻塞直到分片分配完成
    if (!leaderService.isLeaderUntilBlock()) { 
        blockUntilShardingCompleted(); 
        return;
    }
    //leader节点往下执行
    //等待正在运行的作业执行结束
    waitingOtherShardingItemCompleted();
    //加载配置信息
    LiteJobConfiguration liteJobConfig = configService.load(false);
    //分片数
    int shardingTotalCount = liteJobConfig.getTypeConfig().getCoreConfig().getShardingTotalCount();
    log.debug("Job '{}' sharding begin.", jobName);
    //创建作业正在运行的临时节点
    jobNodeStorage.fillEphemeralJobNode(ShardingNode.PROCESSING, "");
    //重置分片
    resetShardingInfo(shardingTotalCount);
    //分片策略
    JobShardingStrategy jobShardingStrategy = JobShardingStrategyFactory.getStrategy(liteJobConfig.getJobShardingStrategyClass());
    //将分片结果写到zk
    jobNodeStorage.executeInTransaction(new PersistShardingInfoTransactionExecutionCallback(jobShardingStrategy.sharding(availableJobInstances, jobName, shardingTotalCount)));
    log.debug("Job '{}' sharding complete.", jobName);
}
```

### 正常运行分片

io.elasticjob.lite.executor.AbstractElasticJobExecutor#execute

```java
private void execute(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource executionSource) {
    if (shardingContexts.getShardingItemParameters().isEmpty()) {
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_FINISHED, String.format("Sharding item for job '%s' is empty.", jobName));
        }
        return;
    }
    //任务分片已经开始运行
    jobFacade.registerJobBegin(shardingContexts);
    String taskId = shardingContexts.getTaskId();
    if (shardingContexts.isAllowSendJobEvent()) {
        jobFacade.postJobStatusTraceEvent(taskId, State.TASK_RUNNING, "");
    }
    try {
        process(shardingContexts, executionSource);
    } finally {
        // TODO 考虑增加作业失败的状态，并且考虑如何处理作业失败的整体回路
        //作业运行完成，删除分片运行节点
        jobFacade.registerJobCompleted(shardingContexts);
        if (itemErrorMessages.isEmpty()) { //执行成功
            if (shardingContexts.isAllowSendJobEvent()) {
                jobFacade.postJobStatusTraceEvent(taskId, State.TASK_FINISHED, "");
            }
        } else { //执行出现异常
            if (shardingContexts.isAllowSendJobEvent()) {
                jobFacade.postJobStatusTraceEvent(taskId, State.TASK_ERROR, itemErrorMessages.toString());
            }
        }
    }
}
```



io.elasticjob.lite.executor.AbstractElasticJobExecutor#process

```java
private void process(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource executionSource) {//多个分片交由线程池处理，单个分片直接运行
    Collection<Integer> items = shardingContexts.getShardingItemParameters().keySet();
    //只有一个分片
    if (1 == items.size()) {
        int item = shardingContexts.getShardingItemParameters().keySet().iterator().next();
        JobExecutionEvent jobExecutionEvent = new JobExecutionEvent(shardingContexts.getTaskId(), jobName, executionSource, item);
        process(shardingContexts, item, jobExecutionEvent);
        return;
    }
    //多个分片交由线程池执行
    final CountDownLatch latch = new CountDownLatch(items.size());
    for (final int each : items) {
        //JobExecutionEvent分片的运行信息
        final JobExecutionEvent jobExecutionEvent = new JobExecutionEvent(shardingContexts.getTaskId(), jobName, executionSource, each);
        if (executorService.isShutdown()) {
            return;
        }
        executorService.submit(new Runnable() {           
            @Override
            public void run() {
                try {
                    process(shardingContexts, each, jobExecutionEvent);
                } finally {
                    latch.countDown();
                }
            }
        });
    }
    try {
        //等待所有分片执行完成
        latch.await();
    } catch (final InterruptedException ex) {
        Thread.currentThread().interrupt();
    }
}
```

io.elasticjob.lite.executor.AbstractElasticJobExecutor#process

```java
private void process(final ShardingContexts shardingContexts, final int item, final JobExecutionEvent startEvent) {
    if (shardingContexts.isAllowSendJobEvent()) {
        jobFacade.postJobExecutionEvent(startEvent);
    }
    log.trace("Job '{}' executing, item is: '{}'.", jobName, item);
    JobExecutionEvent completeEvent;
    try {
        //执行自定义的业务逻辑
        process(new ShardingContext(shardingContexts, item));
        //作业执行成功
        completeEvent = startEvent.executionSuccess();
        log.trace("Job '{}' executed, item is: '{}'.", jobName, item);
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobExecutionEvent(completeEvent);
        }
        // CHECKSTYLE:OFF
    } catch (final Throwable cause) {
        // CHECKSTYLE:ON 作业执行失败
        completeEvent = startEvent.executionFailure(cause);
        jobFacade.postJobExecutionEvent(completeEvent);
        //分片执行失败的原因
        itemErrorMessages.put(item, ExceptionUtil.transform(cause));
        //打印作业执行的异常信息
        jobExceptionHandler.handleException(jobName, cause);
    }
}
```

### 错过任务重执行

io.elasticjob.lite.internal.schedule.JobTriggerListener#triggerMisfired

```java
public void triggerMisfired(final Trigger trigger) {//监控错过执行的任务
    if (null != trigger.getPreviousFireTime()) {
        executionService.setMisfire(shardingService.getLocalShardingItems());
    }
}
```

```java
public void setMisfire(final Collection<Integer> items) {//设置任务被错过执行的标记
    for (int each : items) {
        jobNodeStorage.createJobNodeIfNeeded(ShardingNode.getMisfireNode(each));
    }
}
```

在开启错过任务重执行功能之后，ElasticJob 将会在上次作业执行完毕后，立刻触发执行错过的作业

```java
while (jobFacade.isExecuteMisfired(shardingContexts.getShardingItemParameters().keySet())) {
    ///移除错失执行的分片
    jobFacade.clearMisfire(shardingContexts.getShardingItemParameters().keySet());
    //执行错失执行的分片
    execute(shardingContexts, JobExecutionEvent.ExecutionSource.MISFIRE);
}
```

### 失效转移

在开启失效转移功能之后，ElasticJob 将会在上次作业执行完毕后，会判断是否有需要失效转移的任务

```java
public void failoverIfNecessary() {
    if (configService.load(true).isFailover()) { //开启失效转移
        failoverService.failoverIfNecessary();
    }
}
```

```java
public void failoverIfNecessary() {
    if (needFailover()) {
        jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback()); //竞争leader，执行失效转移
    }
}
```

```java
private boolean needFailover() { //zk是否存在failover节点
    return jobNodeStorage.isJobNodeExisted(FailoverNode.ITEMS_ROOT) && !jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).isEmpty()
            && !JobRegistry.getInstance().isJobRunning(jobName);
}
```

io.elasticjob.lite.internal.failover.FailoverService.FailoverLeaderExecutionCallback#execute

```java
public void execute() {
    if (JobRegistry.getInstance().isShutdown(jobName) || !needFailover()) {
        return;
    }
   //获取failover下的第一个分片项进行处理
    int crashedItem = Integer.parseInt(jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).get(0));
    log.debug("Failover job '{}' begin, crashed item '{}'", jobName, crashedItem);
    jobNodeStorage.fillEphemeralJobNode(FailoverNode.getExecutionFailoverNode(crashedItem), JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
  //删除失效转移的分片项
    jobNodeStorage.removeJobNodeIfExisted(FailoverNode.getItemsNode(crashedItem));
    // TODO 不应使用triggerJob, 而是使用executor统一调度
    JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
     //立刻启动作业
    if (null != jobScheduleController) {
        jobScheduleController.triggerJob();
    }
}
```

io.elasticjob.lite.internal.failover.FailoverListenerManager.JobCrashedJobListener#dataChanged

```java
protected void dataChanged(final String path, final Type eventType, final String data) {//监听作业执行节点宕机
    if (isFailoverEnabled() && Type.NODE_REMOVED == eventType && instanceNode.isInstancePath(path)) {
        String jobInstanceId = path.substring(instanceNode.getInstanceFullPath().length() + 1);
        if (jobInstanceId.equals(JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId())) {
            return;
        }
        //获取失效转移的分片项
        List<Integer> failoverItems = failoverService.getFailoverItems(jobInstanceId);
        if (!failoverItems.isEmpty()) {
            for (int each : failoverItems) {
                //设置失效的分片项标记.
                failoverService.setCrashedFailoverFlag(each);
                //如果开启了失效转移, 则执行作业失效转移.
                failoverService.failoverIfNecessary();
            }
        } else {
            for (int each : shardingService.getShardingItems(jobInstanceId)) {
                failoverService.setCrashedFailoverFlag(each);
                failoverService.failoverIfNecessary();
            }
        }
    }
}
```

## 
