# 三大组件

​	1、TM：全局事务管理器，在开启分布式事务的服务端开启，并将全局事务发送到TC

​	2、TC：事务控制中心，控制全局事务的提交或者回滚。需要独立部署维护

​	3、RM：资源管理器，主要负责向TC上报分支事务，还有本地事务的管理。

​	RM如果执行事务出现异常，只会抛出异常，不会进行全局事务的回滚。全局事务捕获到分支事务的异常后，才会向TM发起事务的回滚。如果由RM发起全局事务的回滚，会缩短响应时间

# 如何集成？

## GlobalTransactionScanner

继承AbstractAutoProxyCreator，实现InitializingBean

### 初始化客户端

```java
@Override
public void afterPropertiesSet() {
    if (disableGlobalTransaction) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Global transaction is disabled.");
        }
        return;
    }
    //初始化客户端TM Client、RM Client 与TC进行通信
    initClient();
}
```

```java
private void initClient() {
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Initializing Global Transaction Clients ... ");
    }
    if (StringUtils.isNullOrEmpty(applicationId) || StringUtils.isNullOrEmpty(txServiceGroup)) {
        throw new IllegalArgumentException(
            "applicationId: " + applicationId + ", txServiceGroup: " + txServiceGroup);
    }
    //init TM
    TMClient.init(applicationId, txServiceGroup);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info(
            "Transaction Manager Client is initialized. applicationId[" + applicationId + "] txServiceGroup["
                + txServiceGroup + "]");
    }
    //init RM
    RMClient.init(applicationId, txServiceGroup);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info(
            "Resource Manager is initialized. applicationId[" + applicationId + "] txServiceGroup[" + txServiceGroup
                + "]");
    }

    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Global Transaction Clients are initialized. ");
    }
    registerSpringShutdownHook();
}
```

### 创建全局事务代理

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (disableGlobalTransaction) {
        return bean;
    }
    try {
        synchronized (PROXYED_SET) {
            //之前已经创建过
            if (PROXYED_SET.contains(beanName)) {
                return bean;
            }
            interceptor = null;
            //check TCC proxy
            if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, applicationContext)) {
                //TCC interceptor, proxy bean of sofa:reference/dubbo:reference, and LocalTCC
                interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
            } else {
                Class<?> serviceInterface = SpringProxyUtils.findTargetClass(bean);
                Class<?>[] interfacesIfJdk = SpringProxyUtils.findInterfaces(bean);

                if (!existsAnnotation(new Class[]{serviceInterface})
                    && !existsAnnotation(interfacesIfJdk)) {
                    return bean;
                }

                if (interceptor == null) {
                    interceptor = new GlobalTransactionalInterceptor(failureHandlerHook);
                    ConfigurationFactory.getInstance().addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,(ConfigurationChangeListener)interceptor);
                }
            }

            LOGGER.info(
                "Bean[" + bean.getClass().getName() + "] with name [" + beanName + "] would use interceptor ["
                    + interceptor.getClass().getName() + "]");
            if (!AopUtils.isAopProxy(bean)) {
                bean = super.wrapIfNecessary(bean, beanName, cacheKey);
            } else {
                AdvisedSupport advised = SpringProxyUtils.getAdvisedSupport(bean);
                Advisor[] advisor = buildAdvisors(beanName, getAdvicesAndAdvisorsForBean(null, null, null));
                for (Advisor avr : advisor) {
                    advised.addAdvisor(0, avr);
                }
            }
            PROXYED_SET.add(beanName);
            return bean;
        }
    } catch (Exception exx) {
        throw new RuntimeException(exx);
    }
}
```

### 数据源代理

```java
client.support.spring.datasource.autoproxy=true //自动对DataSource进行代理
```

io.seata.spring.annotation.GlobalTransactionScanner#postProcessAfterInitialization

```java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
  	//创建对DataSource的代理
    if (bean instanceof DataSource && !(bean instanceof DataSourceProxy) && ConfigurationFactory.getInstance().getBoolean(DATASOURCE_AUTOPROXY, false)) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Auto proxy of [{}]", beanName);
        }
        //生成DataSourceProxy
        DataSourceProxy dataSourceProxy = DataSourceProxyHolder.get().putDataSource((DataSource) bean);
        Class<?>[] interfaces = SpringProxyUtils.getAllInterfaces(bean);
      //代理类
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
              //调用的方法，在DataSourceProxy中不存在时，直接调用Datasource中相对应的方法
                Method m = BeanUtils.findDeclaredMethod(DataSourceProxy.class, method.getName(), method.getParameterTypes());
                if (null != m) {
                    return m.invoke(dataSourceProxy, args);
                } else {
                    boolean oldAccessible = method.isAccessible();
                    try {
                        method.setAccessible(true);
                        return method.invoke(bean, args);
                    } finally {
                        //recover the original accessible for security reason
                        method.setAccessible(oldAccessible);
                    }
                }
            }
        });
    }
    return super.postProcessAfterInitialization(bean, beanName);
}
```

## GlobalTransactionalInterceptor

@GlobalTransactional注解标志全局事务的入口

```java
public Object invoke(final MethodInvocation methodInvocation) throws Throwable {
    Class<?> targetClass = methodInvocation.getThis() != null ? AopUtils.getTargetClass(methodInvocation.getThis())
        : null;
    Method specificMethod = ClassUtils.getMostSpecificMethod(methodInvocation.getMethod(), targetClass);
    final Method method = BridgeMethodResolver.findBridgedMethod(specificMethod);

    final GlobalTransactional globalTransactionalAnnotation = getAnnotation(method, GlobalTransactional.class);
    final GlobalLock globalLockAnnotation = getAnnotation(method, GlobalLock.class);
    if (!disable && globalTransactionalAnnotation != null) {
        return handleGlobalTransaction(methodInvocation, globalTransactionalAnnotation); //对全局事务拦截
    } else if (!disable && globalLockAnnotation != null) {
        return handleGlobalLock(methodInvocation);
    } else {
        return methodInvocation.proceed();
    }
}
```

io.seata.spring.annotation.GlobalTransactionalInterceptor#handleGlobalTransaction

```java
private Object  handleGlobalTransaction(final MethodInvocation methodInvocation,
                                       final GlobalTransactional globalTrxAnno) throws Throwable {
    try {
        return transactionalTemplate.execute(new TransactionalExecutor() {
            @Override
            public Object execute() throws Throwable {
                return methodInvocation.proceed(); //执行业务逻辑
            }

            public String name() {
                String name = globalTrxAnno.name();
                if (!StringUtils.isNullOrEmpty(name)) {
                    return name;
                }
                return formatMethod(methodInvocation.getMethod());
            }

            @Override
            public TransactionInfo getTransactionInfo() { //事务信息
                TransactionInfo transactionInfo = new TransactionInfo();
                transactionInfo.setTimeOut(globalTrxAnno.timeoutMills());
                transactionInfo.setName(name());
                Set<RollbackRule> rollbackRules = new LinkedHashSet<>();
                for (Class<?> rbRule : globalTrxAnno.rollbackFor()) {
                    rollbackRules.add(new RollbackRule(rbRule));
                }
                for (String rbRule : globalTrxAnno.rollbackForClassName()) {
                    rollbackRules.add(new RollbackRule(rbRule));
                }
                for (Class<?> rbRule : globalTrxAnno.noRollbackFor()) {
                    rollbackRules.add(new NoRollbackRule(rbRule));
                }
                for (String rbRule : globalTrxAnno.noRollbackForClassName()) {
                    rollbackRules.add(new NoRollbackRule(rbRule));
                }
                transactionInfo.setRollbackRules(rollbackRules);
                return transactionInfo;
            }
        });
    } catch (TransactionalExecutor.ExecutionException e) {
        TransactionalExecutor.Code code = e.getCode();
        switch (code) {
            case RollbackDone:
                throw e.getOriginalException();
            case BeginFailure:
                failureHandler.onBeginFailure(e.getTransaction(), e.getCause());
                throw e.getCause();
            case CommitFailure:
                failureHandler.onCommitFailure(e.getTransaction(), e.getCause());
                throw e.getCause();
            case RollbackFailure:
                failureHandler.onRollbackFailure(e.getTransaction(), e.getCause());
                throw e.getCause();
            default:
                throw new ShouldNeverHappenException("Unknown TransactionalExecutor.Code: " + code);

        }
    }
}
```

## TransactionalTemplate

io.seata.tm.api.TransactionalTemplate#execute

```java
//分布式事务执行模板
public Object execute(TransactionalExecutor business) throws Throwable {
    // 1.获取或创建全局事务(DefaultGlobalTransaction)
    GlobalTransaction tx = GlobalTransactionContext.getCurrentOrCreate();
    //  获取事务信息（事务名称、超时时间、回滚策略）
    TransactionInfo txInfo = business.getTransactionInfo();
    if (txInfo == null) {
        throw new ShouldNeverHappenException("transactionInfo does not exist");
    }
    try {
        // 2.向TC注册（事务名称、超时时间），获取全局事务Id
      	//放在try外面，出现异常不会进行重试
        beginTransaction(txInfo, tx);
        Object rs = null;
        try {
            //3.执行业务逻辑
            rs = business.execute();
        } catch (Throwable ex) {
            // 4.回滚全局事务
            completeTransactionAfterThrowing(txInfo,tx,ex);
            throw ex;
        }
        // 5.提交全局事务
        commitTransaction(tx);
        return rs;
    } finally {
        //6. clear
        triggerAfterCompletion();
        cleanUp();
    }
}
```

## DefaultGlobalTransaction

io.seata.tm.api.DefaultGlobalTransaction#begin()

### 开启全局事务

```java
public void begin() throws TransactionException {
  	//默认60000
    begin(DEFAULT_GLOBAL_TX_TIMEOUT);
}

@Override
public void begin(int timeout) throws TransactionException {
    begin(timeout, DEFAULT_GLOBAL_TX_NAME);
}

@Override
public void begin(int timeout, String name) throws TransactionException {
    if (role != GlobalTransactionRole.Launcher) {
        check();
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Begin(): just involved in global transaction [" + xid + "]");
        }
        return;
    }
    if (xid != null) {
        throw new IllegalStateException();
    }
    if (RootContext.getXID() != null) {
        throw new IllegalStateException();
    }
  	//获取全局事务Id
    xid = transactionManager.begin(null, null, name, timeout);
    status = GlobalStatus.Begin;
  	//线程绑定全局事务ID，可以实现全局事务id的传递
    RootContext.bind(xid);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Begin new global transaction [" + xid + "]");
    }

}
```

### 提交全局事务

```java
@Override
public void commit() throws TransactionException {
    if (role == GlobalTransactionRole.Participant) {
        // Participant has no responsibility of committing
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Commit(): just involved in global transaction [" + xid + "]");
        }
        return;
    }
    if (xid == null) {
        throw new IllegalStateException();
    }
  	//提交失败，进行重试
    int retry = COMMIT_RETRY_COUNT;
    try {
        while (retry > 0) {
            try {
              //向TC发送GlobalCommitRequest
                status = transactionManager.commit(xid);
                break;
            } catch (Throwable ex) {
                LOGGER.error("Failed to report global commit [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
                retry--;
                //重试失败，抛出异常
                if (retry == 0) {
                    throw new TransactionException("Failed to report global commit", ex);
                }
            }
        }
    } finally {
        if (RootContext.getXID() != null) {
            if (xid.equals(RootContext.getXID())) {
                RootContext.unbind();
            }
        }
    }
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("[" + xid + "] commit status:" + status);
    }

}
```

### 回滚全局事务

```java
@Override
public void rollback() throws TransactionException {
    if (role == GlobalTransactionRole.Participant) {
        // Participant has no responsibility of committing
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Rollback(): just involved in global transaction [" + xid + "]");
        }
        return;
    }
    if (xid == null) {
        throw new IllegalStateException();
    }

  	//回滚失败，重试
    int retry = ROLLBACK_RETRY_COUNT;
    try {
        while (retry > 0) {
            try {
              //向TC发送GlobalRollbackRequest
                status = transactionManager.rollback(xid);
                break;
            } catch (Throwable ex) {
                LOGGER.error("Failed to report global rollback [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
                retry--;
                if (retry == 0) {
                    throw new TransactionException("Failed to report global rollback", ex);
                }
            }
        }
    } finally {
        if (RootContext.getXID() != null) {
            if (xid.equals(RootContext.getXID())) {
                RootContext.unbind();
            }
        }
    }
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("[" + xid + "] rollback status:" + status);
    }
}
```

## DefaultTransactionManager

事务管理器，向TC发送请求

```java
@Override
public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
    throws TransactionException {
    GlobalBeginRequest request = new GlobalBeginRequest();
    request.setTransactionName(name);
    request.setTimeout(timeout);
    GlobalBeginResponse response = (GlobalBeginResponse)syncCall(request);
    if (response.getResultCode() == ResultCode.Failed) {
        throw new TmTransactionException(TransactionExceptionCode.BeginFailed, response.getMsg());
    }
    return response.getXid();
}

@Override
public GlobalStatus commit(String xid) throws TransactionException {
        GlobalCommitRequest globalCommit = new GlobalCommitRequest();
    globalCommit.setXid(xid);
    GlobalCommitResponse response = (GlobalCommitResponse)syncCall(globalCommit);
    return response.getGlobalStatus();
}

@Override
public GlobalStatus rollback(String xid) throws TransactionException {
    GlobalRollbackRequest globalRollback = new GlobalRollbackRequest();
    globalRollback.setXid(xid);
    GlobalRollbackResponse response = (GlobalRollbackResponse)syncCall(globalRollback);
    return response.getGlobalStatus();
}
```

```java
private AbstractTransactionResponse syncCall(AbstractTransactionRequest request) throws TransactionException {
    try {
        return (AbstractTransactionResponse)TmRpcClient.getInstance().sendMsgWithResponse(request);
    } catch (TimeoutException toe) {
        throw new TmTransactionException(TransactionExceptionCode.IO, "RPC timeout", toe);
    }
}
```

## DefaultResourceManager

分支事务管理器，向TC发送请求，分支事务的注册、提交、回滚

```java
protected void initResourceManagers() {
  //初始化资源管理器（AT->DataSourceManager,SAGA->SagaResourceManager,
  //TCC->TCCResourceManager）
  List<ResourceManager> allResourceManagers = EnhancedServiceLoader.loadAll(ResourceManager.class);
  if (CollectionUtils.isNotEmpty(allResourceManagers)) {
    for (ResourceManager rm : allResourceManagers) {
      resourceManagers.put(rm.getBranchType(), rm);
    }
  }
}
@Override
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId,
                                 String resourceId, String applicationData)
    throws TransactionException {
  //根据分支类型获取对应的资源管理器
    return getResourceManager(branchType).branchCommit(branchType, xid, branchId, resourceId, applicationData);
}

@Override
public BranchStatus branchRollback(BranchType branchType, String xid, long branchId,
                                   String resourceId, String applicationData)
    throws TransactionException {
    return getResourceManager(branchType).branchRollback(branchType, xid, branchId, resourceId, applicationData);
}

@Override
public Long branchRegister(BranchType branchType, String resourceId,
                           String clientId, String xid, String applicationData, String lockKeys)
    throws TransactionException {
    return getResourceManager(branchType).branchRegister(branchType, resourceId, clientId, xid, applicationData,
        lockKeys);
}
```

# 流程

1. 获取或创建全局事务
2. 向TC注册（事务名称、超时时间），获取全局事务Id
3. 执行业务逻辑
4. 出现异常，回滚全局事务
5. 提交全局事务
6. 执行TransactionHook的afterCompletion方法

## 注册全局事务

io.seata.tm.api.TransactionalTemplate#beginTransaction

```java
private void beginTransaction(TransactionInfo txInfo, GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
    try {
        triggerBeforeBegin();
        tx.begin(txInfo.getTimeOut(), txInfo.getName()); //远端获取全局事务Id
        triggerAfterBegin();
    } catch (TransactionException txe) {
        throw new TransactionalExecutor.ExecutionException(tx, txe,
            TransactionalExecutor.Code.BeginFailure);
    }
}
```

```java
public void begin(int timeout, String name) throws TransactionException {
    if (role != GlobalTransactionRole.Launcher) {
        check();
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Begin(): just involved in global transaction [{}]", xid);
        }
        return;
    }
    if (xid != null) {
        throw new IllegalStateException();
    }
    if (RootContext.getXID() != null) {
        throw new IllegalStateException();
    }
  	//注册，返回全局事务Id,全局事务负责管理分支事务
    xid = transactionManager.begin(null, null, name, timeout);
    status = GlobalStatus.Begin;
    RootContext.bind(xid);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Begin new global transaction [{}]", xid);
    }
}
```

## 业务执行

识别sql语句，创建修改前镜像数据、执行sql语句、创建修改后镜像数据

创建undolog、lockkeys、绑定事务信息

### 获取执行器

io.seata.rm.datasource.PreparedStatementProxy#executeUpdate

```java
public int executeUpdate() throws SQLException {
    return ExecuteTemplate.execute(this, new StatementCallback<Integer, PreparedStatement>() {
        @Override
        public Integer execute(PreparedStatement statement, Object... args) throws SQLException {
            return statement.executeUpdate();
        }
    });
}
```

```java
public static <T, S extends Statement> T execute(SQLRecognizer sqlRecognizer,
                                                 StatementProxy<S> statementProxy,
                                                 StatementCallback<T, S> statementCallback,
                                                 Object... args) throws SQLException {
	   //和普通的statement执行sql语句一样	：没有全局事务、没有全局锁
    if (!RootContext.inGlobalTransaction() && !RootContext.requireGlobalLock()) {
        // Just work as original statement
        return statementCallback.execute(statementProxy.getTargetStatement(), args);
    }
		//sql识别器
    if (sqlRecognizer == null) {
        sqlRecognizer = SQLVisitorFactory.get(
                statementProxy.getTargetSQL(),
                statementProxy.getConnectionProxy().getDbType());
    }
   //针对不同的sql类型创建不同的Executor
    Executor<T> executor = null;
    if (sqlRecognizer == null) {
        executor = new PlainExecutor<T, S>(statementProxy, statementCallback);
    } else {
        switch (sqlRecognizer.getSQLType()) {
            case INSERT:
                executor = new InsertExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            case UPDATE:
                executor = new UpdateExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            case DELETE:
                executor = new DeleteExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            case SELECT_FOR_UPDATE:
                executor = new SelectForUpdateExecutor<T, S>(statementProxy, statementCallback, sqlRecognizer);
                break;
            default:
                executor = new PlainExecutor<T, S>(statementProxy, statementCallback);
                break;
        }
    }
    T rs = null;
    try {
        rs = executor.execute(args);
    } catch (Throwable ex) {
        if (!(ex instanceof SQLException)) {
            // Turn other exception into SQLException
            ex = new SQLException(ex);
        }
        throw (SQLException)ex;
    }
    return rs;
}
```

io.seata.rm.datasource.exec.BaseTransactionalExecutor#execute

```java
public Object execute(Object... args) throws Throwable {
    
    if (RootContext.inGlobalTransaction()) {
        String xid = RootContext.getXID();
        statementProxy.getConnectionProxy().bind(xid);//绑定全局事务Id
    }
    if (RootContext.requireGlobalLock()) {
        statementProxy.getConnectionProxy().setGlobalLockRequire(true);
    } else {
        statementProxy.getConnectionProxy().setGlobalLockRequire(false);
    }
    return doExecute(args);
}
```

io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#doExecute

```java
public T doExecute(Object... args) throws Throwable {
    AbstractConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
    if (connectionProxy.getAutoCommit()) {
        return executeAutoCommitTrue(args);
    } else {
        return executeAutoCommitFalse(args);
    }
}
```

io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#executeAutoCommitTrue

```java
protected T executeAutoCommitTrue(Object[] args) throws Throwable {
    AbstractConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
    try {
      	//手动提交
        connectionProxy.setAutoCommit(false);
        //锁冲突，重试策略
        return new LockRetryPolicy(connectionProxy.getTargetConnection()).execute(() -> {
            T result = executeAutoCommitFalse(args);
            connectionProxy.commit();
            return result;
        });
    } catch (Exception e) {
        // when exception occur in finally,this exception will lost, so just print it here
        LOGGER.error("execute executeAutoCommitTrue error:{}", e.getMessage(), e);
        if (!LockRetryPolicy.isLockRetryPolicyBranchRollbackOnConflict()) {
            connectionProxy.getTargetConnection().rollback();
        }
        throw e;
    } finally {
        ((ConnectionProxy) connectionProxy).getContext().reset();
        connectionProxy.setAutoCommit(true);
    }
}
```

### 执行语句

io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#executeAutoCommitFalse

```java
protected T executeAutoCommitFalse(Object[] args) throws Exception {
    //修改前镜像
    TableRecords beforeImage = beforeImage();
    //执行sql语句
    T result = statementCallback.execute(statementProxy.getTargetStatement(), args);
    //修改后镜像
    TableRecords afterImage = afterImage(beforeImage);
    //创建lockKey、sqlUndoLog，绑定到当前的connectionProxy
    prepareUndoLog(beforeImage, afterImage);
    return result;
}
```

### 本地事务提交

注册分支事务，包括lockkey，返回分支事务Id

识别锁冲突异常，稍后重试

保存undolog到数据库

本地事务提交，释放本地锁

事务提交成功，上报PhaseOne_Done

事务提交失败，上报PhaseOne_Failed

io.seata.rm.datasource.ConnectionProxy#commit

```java
public void commit() throws SQLException {
    try {
        LOCK_RETRY_POLICY.execute(() -> {
            doCommit();
            return null;
        });
    } catch (SQLException e) {
       throw e;
    } catch (Exception e) {
        throw new SQLException(e);
    }
}
```

```java
private void doCommit() throws SQLException {
    if (context.inGlobalTransaction()) {
        processGlobalTransactionCommit();
    } else if (context.isGlobalLockRequire()) {
        processLocalCommitWithGlobalLocks();
    } else {
        targetConnection.commit();
    }
}
```

```java
private void processGlobalTransactionCommit() throws SQLException {
    try {
        //注册分支事务
        register();
    } catch (TransactionException e) {
        //识别锁冲突异常
        recognizeLockKeyConflictException(e, context.buildLockKeys());
    }
    try {
        //前后镜像的日志保存到数据库
        if (context.hasUndoLog()) {
        UndoLogManagerFactory.getUndoLogManager(this.getDbType()).flushUndoLogs(this);
        }
        //jdbc提交事务
        targetConnection.commit();
    } catch (Throwable ex) {
        LOGGER.error("process connectionProxy commit error: {}", ex.getMessage(), ex);
        //上报分支事务一阶段执行失败
        report(false);
        throw new SQLException(ex);
    }
    //上报分支事务一阶段执行成功
    if (IS_REPORT_SUCCESS_ENABLE) {
        report(true);
    }
    //清空绑定信息
    context.reset();
}
```

#### 注册分支事务

```java
private void register() throws TransactionException {
    Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, getDataSourceProxy().getResourceId(),
        null, context.getXid(), null, context.buildLockKeys()); //包括锁记录
    context.setBranchId(branchId);
}
```

#### 识别锁冲突异常,稍后重试

```java
private void recognizeLockKeyConflictException(TransactionException te, String lockKeys) throws SQLException {
    if (te.getCode() == TransactionExceptionCode.LockKeyConflict) {
        StringBuilder reasonBuilder = new StringBuilder("get global lock fail, xid:" + context.getXid());
        if (StringUtils.isNotBlank(lockKeys)) {
            reasonBuilder.append(", lockKeys:" + lockKeys);
        }
        throw new LockConflictException(reasonBuilder.toString());
    } else {
        throw new SQLException(te);
    }
}
```

#### 上报分支事务执行结果

```java
private void report(boolean commitDone) throws SQLException {
    //默认重试5次
    int retry = REPORT_RETRY_COUNT;
    while (retry > 0) {
        try {
            DefaultResourceManager.get().branchReport(BranchType.AT, context.getXid(), context.getBranchId(),
                commitDone ? BranchStatus.PhaseOne_Done : BranchStatus.PhaseOne_Failed, null);
            return;
        } catch (Throwable ex) {
            LOGGER.error("Failed to report [" + context.getBranchId() + "/" + context.getXid() + "] commit done ["
                + commitDone + "] Retry Countdown: " + retry);
            retry--;
            if (retry == 0) {
                throw new SQLException("Failed to report branch status " + commitDone, ex);
            }
        }
    }
}
```

## 全局事务

### 提交

io.seata.tm.api.DefaultGlobalTransaction#commit

```java
public void commit() throws TransactionException {
    if (role == GlobalTransactionRole.Participant) {
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Commit(): just involved in global transaction [{}]", xid);
        }
        return;
    }
    if (xid == null) {
        throw new IllegalStateException();
    }
    int retry = COMMIT_RETRY_COUNT;
    try {
        while (retry > 0) {
            try {
              	//向tc发送GlobalCommitRequest
              	//tc再向rm发送提交
                status = transactionManager.commit(xid);
                break;
            } catch (Throwable ex) {
                LOGGER.error("Failed to report global commit [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
                retry--;
                if (retry == 0) {
                    throw new TransactionException("Failed to report global commit", ex);
                }
            }
        }
    } finally {
        if (RootContext.getXID() != null) {
            if (xid.equals(RootContext.getXID())) {
                RootContext.unbind();
            }
        }
    }
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("[{}] commit status: {}", xid, status);
    }
}
```

### 回滚

io.seata.tm.api.DefaultGlobalTransaction#rollback

```java
public void rollback() throws TransactionException {
    if (role == GlobalTransactionRole.Participant) {
        // Participant has no responsibility of rollback
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Rollback(): just involved in global transaction [{}]", xid);
        }
        return;
    }
    if (xid == null) {
        throw new IllegalStateException();
    }
    int retry = ROLLBACK_RETRY_COUNT;
    try {
        while (retry > 0) {
            try {
              	//向tc发送GlobalRollbackRequest
              	//tc再向rm发送回滚
                status = transactionManager.rollback(xid);
                break;
            } catch (Throwable ex) {
                LOGGER.error("Failed to report global rollback [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
                retry--;
                if (retry == 0) {
                    throw new TransactionException("Failed to report global rollback", ex);
                }
            }
        }
    } finally {
        if (RootContext.getXID() != null) {
            if (xid.equals(RootContext.getXID())) {
                RootContext.unbind();
            }
        }
    }
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("[{}] rollback status: {}", xid, status);
    }
}
```

## 分支事务

### 提交

BranchCommitReques， TC发送的二阶段提交请求，AT模式只需将UndoLog删除。将接收的请求放入AsyncWorker的有界队列ASYNC_COMMIT_BUFFER，如果队列已满，打印日志。AsyncWorker初始化时，创建定时器，定时将undolog批量删除。

io.seata.rm.AbstractRMHandler#handle(io.seata.core.protocol.transaction.BranchCommitRequest)

```java
public BranchCommitResponse handle(BranchCommitRequest request) {
    BranchCommitResponse response = new BranchCommitResponse();
    exceptionHandleTemplate(new AbstractCallback<BranchCommitRequest, BranchCommitResponse>() {
        @Override
        public void execute(BranchCommitRequest request, BranchCommitResponse response)
            throws TransactionException {
            doBranchCommit(request, response);
        }
    }, request, response);
    return response;
}
```

io.seata.rm.AbstractRMHandler#doBranchCommit

```java
protected void doBranchCommit(BranchCommitRequest request, BranchCommitResponse response)
    throws TransactionException {
    String xid = request.getXid();
    long branchId = request.getBranchId();
    String resourceId = request.getResourceId();
    String applicationData = request.getApplicationData();
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Branch committing: " + xid + " " + branchId + " " + resourceId + " " + applicationData);
    }
    BranchStatus status = getResourceManager().branchCommit(request.getBranchType(), xid, branchId, resourceId,
        applicationData);
    response.setXid(xid);
    response.setBranchId(branchId);
    response.setBranchStatus(status);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Branch commit result: " + status);
    }
}
```

io.seata.rm.datasource.DataSourceManager#branchCommit

```java
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId,
                                 String applicationData) throws TransactionException {
    return asyncWorker.branchCommit(branchType, xid, branchId, resourceId, applicationData);
}
```

```java
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId,
                                 String applicationData) throws TransactionException {
    if (!ASYNC_COMMIT_BUFFER.offer(new Phase2Context(branchType, xid, branchId, resourceId, applicationData))) {
        LOGGER.warn("Async commit buffer is FULL. Rejected branch [" + branchId + "/" + xid
            + "] will be handled by housekeeping later.");
    }
    return BranchStatus.PhaseTwo_Committed;
}
```

### 回滚

io.seata.rm.AbstractRMHandler#handle(io.seata.core.protocol.transaction.BranchRollbackRequest)

```java
public BranchRollbackResponse handle(BranchRollbackRequest request) {
    BranchRollbackResponse response = new BranchRollbackResponse();
    exceptionHandleTemplate(new AbstractCallback<BranchRollbackRequest, BranchRollbackResponse>() {
        @Override
        public void execute(BranchRollbackRequest request, BranchRollbackResponse response)
            throws TransactionException {
            doBranchRollback(request, response);
        }
    }, request, response);
    return response;
}
```

```java
public void undo(DataSourceProxy dataSourceProxy, String xid, long branchId) throws TransactionException {
    Connection conn = null;
    ResultSet rs = null;
    PreparedStatement selectPST = null;
    boolean originalAutoCommit = true;

    for (; ; ) {
        try {
            conn = dataSourceProxy.getPlainConnection();

            // The entire undo process should run in a local transaction.
            if (originalAutoCommit = conn.getAutoCommit()) {
                conn.setAutoCommit(false);
            }
            // 查询undolog，并加上排他锁
            selectPST = conn.prepareStatement(SELECT_UNDO_LOG_SQL);
            selectPST.setLong(1, branchId);
            selectPST.setString(2, xid);
            rs = selectPST.executeQuery();
            boolean exists = false;
            while (rs.next()) {
                exists = true;
                //TC可能会给多个进程重复发送回滚请求，保证只有正常状态的undolog才能被处理
                int state = rs.getInt(ClientTableColumnsName.UNDO_LOG_LOG_STATUS);
                if (!canUndo(state)) {
                    if (LOGGER.isInfoEnabled()) {
                        LOGGER.info("xid {} branch {}, ignore {} undo_log", xid, branchId, state);
                    }
                    return;
                }
                String contextString = rs.getString(ClientTableColumnsName.UNDO_LOG_CONTEXT);
                Map<String, String> context = parseContext(contextString);
                Blob b = rs.getBlob(ClientTableColumnsName.UNDO_LOG_ROLLBACK_INFO);
                byte[] rollbackInfo = BlobUtils.blob2Bytes(b);
                String serializer = context == null ? null : context.get(UndoLogConstants.SERIALIZER_KEY);
                UndoLogParser parser = serializer == null ? UndoLogParserFactory.getInstance()
                    : UndoLogParserFactory.getInstance(serializer);
                BranchUndoLog branchUndoLog = parser.decode(rollbackInfo);
                try {
                    // put serializer name to local
                    setCurrentSerializer(parser.getName());
                    List<SQLUndoLog> sqlUndoLogs = branchUndoLog.getSqlUndoLogs();
                    if (sqlUndoLogs.size() > 1) {
                        Collections.reverse(sqlUndoLogs);
                    }
                    for (SQLUndoLog sqlUndoLog : sqlUndoLogs) {
                        TableMeta tableMeta = TableMetaCacheFactory.getTableMetaCache(dataSourceProxy).getTableMeta(
                            conn, sqlUndoLog.getTableName(),dataSourceProxy.getResourceId());
                        sqlUndoLog.setTableMeta(tableMeta);
                        //
                        AbstractUndoExecutor undoExecutor = UndoExecutorFactory.getUndoExecutor(
                            dataSourceProxy.getDbType(), sqlUndoLog);
                        undoExecutor.executeOn(conn);
                    }
                } finally {
                    // remove serializer name
                    removeCurrentSerializer();
                }
            }
            if (exists) { //undolog存在，说明第一阶段分支事务已经完成，可以直接回滚，清理undolog
                deleteUndoLog(xid, branchId, conn);
                conn.commit();
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("xid {} branch {}, undo_log deleted with {}", xid, branchId,
                        State.GlobalFinished.name());
                }
            } else { //undolog不存在，分支事务出现异常。例如业务逻辑处理超时，尚未写入undolog，全局事务触发事务回滚，会发现undolog不存在。
                //为了保证一致性，插入状态为GlobalFinished的undolog，防止其他程序中的第一阶段的本地事务正常提交。
                insertUndoLogWithGlobalFinished(xid, branchId, UndoLogParserFactory.getInstance(), conn);
                conn.commit();
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("xid {} branch {}, undo_log added with {}", xid, branchId,
                        State.GlobalFinished.name());
                }
            }
            return;
        } catch (SQLIntegrityConstraintViolationException e) {
            // Possible undo_log has been inserted into the database by other processes, retrying rollback undo_log
            if (LOGGER.isInfoEnabled()) {
                LOGGER.info("xid {} branch {}, undo_log inserted, retry rollback", xid, branchId);
            }
        } catch (Throwable e) {
            if (conn != null) {
                try {
                    conn.rollback();
                } catch (SQLException rollbackEx) {
                    LOGGER.warn("Failed to close JDBC resource while undo ... ", rollbackEx);
                }
            }
            throw new BranchTransactionException(BranchRollbackFailed_Retriable, String
                .format("Branch session rollback failed and try again later xid = %s branchId = %s %s", xid,
                    branchId, e.getMessage()), e);

        } finally {
            try {
                if (rs != null) {
                    rs.close();
                }
                if (selectPST != null) {
                    selectPST.close();
                }
                if (conn != null) {
                    if (originalAutoCommit) {
                        conn.setAutoCommit(true);
                    }
                    conn.close();
                }
            } catch (SQLException closeEx) {
                LOGGER.warn("Failed to close JDBC resource while undo ... ", closeEx);
            }
        }
    }
}
```

事务回滚，判断修改前的和当期的数据是否相同，如果相同进行事务回滚，不同抛出异常，稍后重试

zio.seata.rm.datasource.undo.AbstractUndoExecutor#executeOn

```java
public void executeOn(Connection conn) throws SQLException {
    //默认IS_UNDO_DATA_VALIDATION_ENABLE=true，校验undolog
    if (IS_UNDO_DATA_VALIDATION_ENABLE && !dataValidationAndGoOn(conn)) {
        return;
    }
    //回滚
    try {
        String undoSQL = buildUndoSQL();
        PreparedStatement undoPST = conn.prepareStatement(undoSQL);
        TableRecords undoRows = getUndoRows();
        for (Row undoRow : undoRows.getRows()) {
            ArrayList<Field> undoValues = new ArrayList<>();
            Field pkValue = null;
            for (Field field : undoRow.getFields()) {
                if (field.getKeyType() == KeyType.PrimaryKey) {
                    pkValue = field;
                } else {
                    undoValues.add(field);
                }
            }
            undoPrepare(undoPST, undoValues, pkValue);
            undoPST.executeUpdate();
        }
    } catch (Exception ex) {
        if (ex instanceof SQLException) {
            throw (SQLException) ex;
        } else {
            throw new SQLException(ex);
        }
    }
}
```

数据验证

```java
protected boolean dataValidationAndGoOn(Connection conn) throws SQLException {
    //获取前后镜像
    TableRecords beforeRecords = sqlUndoLog.getBeforeImage();
    TableRecords afterRecords = sqlUndoLog.getAfterImage();
    Result<Boolean> beforeEqualsAfterResult = DataCompareUtils.isRecordsEquals(beforeRecords, afterRecords);
    //修改前、后的数据相同，无需回滚
    if (beforeEqualsAfterResult.getResult()) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Stop rollback because there is no data change " +
                "between the before data snapshot and the after data snapshot.");
        }
        return false;
    }
    TableRecords currentRecords = queryCurrentRecords(conn);
    Result<Boolean> afterEqualsCurrentResult = DataCompareUtils.isRecordsEquals(afterRecords, currentRecords);
    //修改后的数据和当前的数据不同
    if (!afterEqualsCurrentResult.getResult()) {
        //修改前的数据和当前的数据相同，无需回归
        Result<Boolean> beforeEqualsCurrentResult = DataCompareUtils.isRecordsEquals(beforeRecords, currentRecords);
        if (beforeEqualsCurrentResult.getResult()) {
            if (LOGGER.isInfoEnabled()) {
                LOGGER.info("Stop rollback because there is no data change " +
                    "between the before data snapshot and the current data snapshot.");
            }
            return false;
        } else { //修改前和当前的数据不同，修改后的数据和当前的数据不同，说明数据已经被其他事物修改，抛出异常，需要人工介入处理
            if (LOGGER.isInfoEnabled()) {
                if (StringUtils.isNotBlank(afterEqualsCurrentResult.getErrMsg())) {
                    LOGGER.info(afterEqualsCurrentResult.getErrMsg(), afterEqualsCurrentResult.getErrMsgParams());
                }
            }
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("check dirty datas failed, old and new data are not equal," +
                    "tableName:[" + sqlUndoLog.getTableName() + "]," +
                    "oldRows:[" + JSON.toJSONString(afterRecords.getRows()) + "]," +
                    "newRows:[" + JSON.toJSONString(currentRecords.getRows()) + "].");
            }
            throw new SQLException("Has dirty records when undo.");
        }
    }
    return true;
}
```

## undolog过期删除

undolog默认最多存在7天

io.seata.rm.RMHandlerAT#handle

```java
public void handle(UndoLogDeleteRequest request) {
    DataSourceManager dataSourceManager = (DataSourceManager)getResourceManager();
    DataSourceProxy dataSourceProxy = dataSourceManager.get(request.getResourceId());
    if (dataSourceProxy == null) {
        LOGGER.warn("Failed to get dataSourceProxy for delete undolog on " + request.getResourceId());
        return;
    }
    Date logCreatedSave = getLogCreated(request.getSaveDays());
    Connection conn = null;
    try {
        conn = dataSourceProxy.getPlainConnection();
        int deleteRows = 0;
        do {
            try {
                deleteRows = UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType())
                        .deleteUndoLogByLogCreated(logCreatedSave, LIMIT_ROWS, conn);
                if (deleteRows > 0 && !conn.getAutoCommit()) {
                    conn.commit();
                }
            } catch (SQLException exx) {
                if (deleteRows > 0 && !conn.getAutoCommit()) {
                    conn.rollback();
                }
                throw exx;
            }
        } while (deleteRows == LIMIT_ROWS);
    } catch (Exception e) {
        LOGGER.error("Failed to delete expired undo_log, error:{}", e.getMessage(), e);
    } finally {
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException closeEx) {
                LOGGER.warn("Failed to close JDBC resource while deleting undo_log ", closeEx);
            }
        }
    }
}
```

# 如何可观测？

监控指标Metrics、结构化的日志格式Logging、分布式事务全链路的可追溯Tracing

利用SPI机制，根据配置加载Exporter和Registry的实现类

基于消息订阅与通知机制，监听所有全局事务的状态变更事件，并publish到EventBus

事件订阅者消费事件，并将生成的metrics写入Registry

监控系统（如prometheus）从Exporter中拉取数据

io.seata.server.Server#main

```java
public static void main(String[] args) throws IOException {
		...
    //initialize the metrics
    MetricsManager.get().init();
  	...
}

```

io.seata.server.metrics.MetricsManager#init

```java
public void init() {
    boolean enabled = ConfigurationFactory.getInstance().getBoolean(
        ConfigurationKeys.METRICS_PREFIX + ConfigurationKeys.METRICS_ENABLED, false);
    if (enabled) {
        registry = RegistryFactory.getInstance();
        if (registry != null) {
            List<Exporter> exporters = ExporterFactory.getInstanceList();
            //only at least one metrics exporter implement had imported in pom then need register MetricsSubscriber
            if (exporters.size() != 0) {
                exporters.forEach(exporter -> exporter.setRegistry(registry));
                EventBusManager.get().register(new MetricsSubscriber(registry));
            }
        }
    }
}
```

io.seata.server.metrics.MetricsSubscriber#MetricsSubscriber

```java
private final Registry registry;

//维护各种事件对应的监听器
private final Map<GlobalStatus, Consumer<GlobalTransactionEvent>> consumers;
```

```java
public MetricsSubscriber(Registry registry) {
    this.registry = registry;
    consumers = new HashMap<>();
    consumers.put(GlobalStatus.Begin, this::processGlobalStatusBegin);
    consumers.put(GlobalStatus.Committed, this::processGlobalStatusCommitted);
    consumers.put(GlobalStatus.Rollbacked, this::processGlobalStatusRollbacked);

    consumers.put(GlobalStatus.CommitFailed, this::processGlobalStatusCommitFailed);
    consumers.put(GlobalStatus.RollbackFailed, this::processGlobalStatusRollbackFailed);
    consumers.put(GlobalStatus.TimeoutRollbacked, this::processGlobalStatusTimeoutRollbacked);
    consumers.put(GlobalStatus.TimeoutRollbackFailed, this::processGlobalStatusTimeoutRollbackFailed);
}
```

```java
public class EventBusManager {
    private static class SingletonHolder {
        private static EventBus INSTANCE = new GuavaEventBus("tc");
    }

    public static final EventBus get() {
        return SingletonHolder.INSTANCE;
    }
}
```



io.seata.core.rpc.netty.AbstractRpcRemotingClient#init

```java
public void init() {
    clientBootstrap.start();
    timerExecutor.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            clientChannelManager.reconnect(getTransactionServiceGroup());
        }
    }, SCHEDULE_INTERVAL_MILLS, SCHEDULE_INTERVAL_MILLS, TimeUnit.SECONDS);
    mergeSendExecutorService = new ThreadPoolExecutor(MAX_MERGE_SEND_THREAD,
        MAX_MERGE_SEND_THREAD,
        KEEP_ALIVE_TIME, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<>(),
        new NamedThreadFactory(getThreadPrefix(), MAX_MERGE_SEND_THREAD));
    mergeSendExecutorService.submit(new MergedSendRunnable());
    super.init();
}
```

```java
void reconnect(String transactionServiceGroup) {
    List<String> availList = null;
    try {
     		 //获取TC地址
      	 //1、先根据transactionServiceGroup，从配置中心获取TC的集群名
     		 //2、根据TC的集群名，从注册中心获取TC地址
        availList = getAvailServerList(transactionServiceGroup);
    } catch (Exception exx) {
        LOGGER.error("Failed to get available servers: {}", exx.getMessage());
    }
    if (CollectionUtils.isEmpty(availList)) {
        LOGGER.error("no available server to connect.");
        return;
    }
  	//和每个TC建立连接
    for (String serverAddress : availList) {
        try {
            acquireChannel(serverAddress);
        } catch (Exception e) {
            LOGGER.error("{} can not connect to {} cause:{}",FrameworkErrorCode.NetConnect.getErrCode(), serverAddress, e.getMessage(), e);
        }
    }
}
```

```java
private List<String> getAvailServerList(String transactionServiceGroup) throws Exception {
    List<String> availList = new ArrayList<>();
  //从注册中心拉取TC地址
    List<InetSocketAddress> availInetSocketAddressList = RegistryFactory.getInstance().lookup(
        transactionServiceGroup);
    if (!CollectionUtils.isEmpty(availInetSocketAddressList)) {
        for (InetSocketAddress address : availInetSocketAddressList) {
            availList.add(NetUtil.toStringAddress(address));
        }
    }
    return availList;
}
```

```java
public static RegistryService getInstance() {
    RegistryType registryType;
 	 //registry.type
    String registryTypeName = ConfigurationFactory.CURRENT_FILE_INSTANCE.getConfig(
        ConfigurationKeys.FILE_ROOT_REGISTRY + ConfigurationKeys.FILE_CONFIG_SPLIT_CHAR
            + ConfigurationKeys.FILE_ROOT_TYPE);
    try {
        registryType = RegistryType.getType(registryTypeName);
    } catch (Exception exx) {
        throw new NotSupportYetException("not support registry type: " + registryTypeName);
    }
    if (RegistryType.File == registryType) {
        return FileRegistryServiceImpl.getInstance();
    } else {
      //获取注册服务
        return EnhancedServiceLoader.load(RegistryProvider.class, Objects.requireNonNull(registryType).name()).provide();
    }
}
```

```java
public List<InetSocketAddress> lookup(String key) throws Exception {
    String clusterName = getServiceGroup(key);

    if (null == clusterName) {
        return null;
    }

    return doLookup(clusterName);
}
```

```java
private String getServiceGroup(String key) {
  //从配置中心，根据transactionServiceGroup获取该应用侧对应的TC集群名
    Configuration configuration = ConfigurationFactory.getInstance();
    String clusterNameKey = PREFIX_SERVICE_ROOT + CONFIG_SPLIT_CHAR + PREFIX_SERVICE_MAPPING + key;
    return configuration.getConfig(clusterNameKey);
}
```

io.seata.core.rpc.netty.AbstractRpcRemotingClient#loadBalance

```java
//负载均衡
private String loadBalance(String transactionServiceGroup) {
    InetSocketAddress address = null;
    try {
        List<InetSocketAddress> inetSocketAddressList = RegistryFactory.getInstance().lookup(transactionServiceGroup);
        address = LoadBalanceFactory.getInstance().select(inetSocketAddressList);
    } catch (Exception ex) {
        LOGGER.error(ex.getMessage());
    }
    if (address == null) {
        throw new FrameworkException(NoAvailableService);
    }
    return NetUtil.toStringAddress(address);
}
```

# AT模式

​	AT模式是一种对业务无任何侵入的分布式事务解决方案

​	用户只需编写“业务 SQL”，便能轻松接入分布式事务

​	AT模式适用于数据库

​	由于本地事务提交前会获取全局锁，并且在全局事务提交时才会释放全局锁

## 流程

### 一阶段

​	在一阶段，Seata 会拦截“业务 SQL”先解析 SQL 语义，找到“业务 SQL”要更新的业务数据

​	在业务数据被更新前，将其保存成“before image”

​	然后执行“业务 SQL”更新业务数据

​	在业务数据更新之后，再将其保存成“after image”

​	和本地事务一块提交到数据库，提交前需要获取全局锁

### 二阶段提交

​	一阶段分支事务全部成功，二阶段提交全局事务

​	二阶段主要是删除快照数据，释放全局锁

​	二阶段可以异步执行，提高吞吐量

### 二阶段回滚

​	Seata需要回滚一阶段已经执行的“业务 SQL”，还原业务数据。回滚方式便是用“before image”还原业务数据；但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。

### 写隔离

一阶段本地事务提交前，需要确保先拿到该行的全局锁

拿不到全局锁 ，不能提交本地事务。

拿全局锁 的尝试被限制在一定范围内，超出范围将放弃，并回滚本地事务，释放本地锁

全局锁在分布式事务结束前一直被某个事务持有，不会发生脏写的问题

### 读隔离

Seata（AT 模式）的默认全局隔离级别是 读未提交（Read Uncommitted）

如果应用在特定场景下，必须要求全局的读已提交 ，Seata 通过对 SELECT FOR UPDATE 语句的代理来实现，获取全局锁来实现	

创建 UNDO_LOG 表、事务入口标注@GlobalTransactional、并对DataSource进行代理

# TCC模式

相对于 AT 模式，TCC 模式对业务代码有一定的侵入性，但是 TCC 模式无 AT 模式的全局行锁，TCC 性能会比 AT 模式高很多

## 流程

### 一阶段

Try：资源的检测和预留；

### 二阶段提交

Confirm：执行的业务操作提交；要求 Try 成功 Confirm 一定要能成功

### 二阶段回滚

Cancel：预留资源释放

## 问题

### 空回滚

Cancel 接口设计时需要允许空回滚。在 Try 接口因为丢包时没有收到，事务管理器会触发回滚，这时会触发 Cancel 接口，这时 Cancel 执行时发现没有对应的事务 xid 或主键时，需要返回回滚成功。让事务服务管理器认为已回滚，否则会不断重试，而 Cancel 又没有对应的业务数据可以进行回滚

### 悬挂控制

Cancel 比 Try 接口先执行。出现的原因是 Try 由于网络拥堵而超时，事务管理器生成回滚，触发 Cancel 接口，而最终又收到了 Try 接口调用，但是 Cancel 比 Try 先到。按照前面允许空回滚的逻辑，回滚会返回成功，事务管理器认为事务已回滚成功，则此时的 Try 接口不应该执行，否则会产生数据不一致，所以我们在 Cancel 空回滚返回成功之前先记录该条事务 xid 或业务主键，标识这条记录已经回滚过，Try 接口先检查这条事务xid或业务主键如果已经标记为回滚成功过，则不执行 Try 的业务操作。

### 幂等控制

Try、Comfirm、Cancel3个方法需保证幂等性。因为网络抖动或拥堵可能会超时，事务管理器会对资源进行重试操作，所以很可能一个业务操作会被重复调用，为了不因为重复调用而多次占用资源，需要对服务设计时进行幂等控制，通常我们可以用事务 xid 或业务主键判重来控制。

## 优化

### 异步执行

TCC 模型会对资源业务进行锁定，锁定的资源业务既不会阻塞其他事务在第一阶段对于相同资源的继续使用，也不会影响本事务第二阶段的正确执行。从理论上来说，只要业务允许，事务的第二阶段什么时候执行都可以。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD })
@Inherited
public @interface TwoPhaseBusinessAction {

    /**
     * TCC bean name, must be unique
     */
    String name() ;

    /**
     * commit methed name
     */
    String commitMethod() default "commit";

    /**
     * rollback method name
     */
    String rollbackMethod() default "rollback";

}
```

### TccActionInterceptor

io.seata.spring.tcc.TccActionInterceptor#invoke

```java
public Object invoke(final MethodInvocation invocation) throws Throwable {
    if(!RootContext.inGlobalTransaction()){
        //not in transaction
        return invocation.proceed();
    }
    Method method = getActionInterfaceMethod(invocation);
    TwoPhaseBusinessAction businessAction = method.getAnnotation(TwoPhaseBusinessAction.class);
    //try method
    if (businessAction != null) {
        //save the xid
        String xid = RootContext.getXID();
        //clear the context
        RootContext.unbind();
        try {
            Object[] methodArgs = invocation.getArguments();
            //Handler the TCC Aspect
            Map<String, Object> ret = actionInterceptorHandler.proceed(method, methodArgs, xid, businessAction,
                    new Callback<Object>() {
                        @Override
                        public Object execute() throws Throwable {
                            return invocation.proceed();
                        }
                    });
            //return the final result
            return ret.get(Constants.TCC_METHOD_RESULT);
        } finally {
            //recovery the context
            RootContext.bind(xid);
        }
    }
    return invocation.proceed();
}
```

```java
public Map<String, Object> proceed(Method method, Object[] arguments, String xid, TwoPhaseBusinessAction businessAction,
                                   Callback<Object> targetCallback) throws Throwable {
    Map<String, Object> ret = new HashMap<String, Object>(16);

    //TCC name
    String actionName = businessAction.name();
    BusinessActionContext actionContext = new BusinessActionContext();
    actionContext.setXid(xid);
    //set action anme
    actionContext.setActionName(actionName);

    //注册分支事务，包括了方法参数、commitMethod、rollbackMethod
    String branchId = doTccActionLogStore(method, arguments, businessAction, actionContext);
    actionContext.setBranchId(branchId);

    //set the parameter whose type is BusinessActionContext
    Class<?>[] types = method.getParameterTypes();
    int argIndex = 0;
    for (Class<?> cls : types) {
        if (cls.getName().equals(BusinessActionContext.class.getName())) {
            arguments[argIndex] = actionContext;
            break;
        }
        argIndex++;
    }
    //the final parameters of the try method
    ret.put(Constants.TCC_METHOD_ARGUMENTS, arguments);
    //targetCallback.execute() 执行业务逻辑
    ret.put(Constants.TCC_METHOD_RESULT, targetCallback.execute());
    return ret;
}
```

### TCCResourceManager

```java
//维护了每个Prepare对应的Resource（commitMethod/rollbackMethod/targetBean）
private Map<String, Resource> tccResourceCache = new ConcurrentHashMap<String, Resource>();

```

```java
@Override
public void registerResource(Resource resource) {
    TCCResource tccResource = (TCCResource)resource;
    tccResourceCache.put(tccResource.getResourceId(), tccResource);
    synchronized (RESOURCE_LOCK) {
        RESOURCE_LOCK.notifyAll();
    }
    super.registerResource(tccResource);
}

@Override
public Map<String, Resource> getManagedResources() {
    return tccResourceCache;
}

//接收TC请求，执行commitMethod
@Override
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId,
                                 String applicationData) throws TransactionException {
    TCCResource tccResource = (TCCResource)tccResourceCache.get(resourceId);
    if (tccResource == null) {
        throw new ShouldNeverHappenException("TCC resource is not exist, resourceId:" + resourceId);
    }
  	//获取commitMethod对应的Bean
    Object targetTCCBean = tccResource.getTargetBean();
    Method commitMethod = tccResource.getCommitMethod();
    if (targetTCCBean == null || commitMethod == null) {
        throw new ShouldNeverHappenException("TCC resource is not available, resourceId:" + resourceId);
    }
    try {
        boolean result = false;
        //BusinessActionContext
        BusinessActionContext businessActionContext = getBusinessActionContext(xid, branchId, resourceId,
            applicationData);
      //执行方法
        Object ret = commitMethod.invoke(targetTCCBean, businessActionContext);
        LOGGER.info(
            "TCC resource commit result :" + ret + ", xid:" + xid + ", branchId:" + branchId + ", resourceId:"
                + resourceId);
        if (ret != null) {
            if (ret instanceof TwoPhaseResult) {
                result = ((TwoPhaseResult)ret).isSuccess();
            } else {
                result = (boolean)ret;
            }
        }
        return result ? BranchStatus.PhaseTwo_Committed : BranchStatus.PhaseTwo_CommitFailed_Retryable;
    } catch (Throwable t) {
        String msg = String.format("commit TCC resource error, resourceId: %s, xid: %s.", resourceId, xid);
        LOGGER.error(msg, t);
        throw new FrameworkException(t, msg);
    }
}

//接收TC请求，执行rollBack方法
@Override
public BranchStatus branchRollback(BranchType branchType, String xid, long branchId, String resourceId,
                                   String applicationData) throws TransactionException {
    TCCResource tccResource = (TCCResource)tccResourceCache.get(resourceId);
    if (tccResource == null) {
        throw new ShouldNeverHappenException("TCC resource is not exist, resourceId:" + resourceId);
    }
    Object targetTCCBean = tccResource.getTargetBean();
    Method rollbackMethod = tccResource.getRollbackMethod();
    if (targetTCCBean == null || rollbackMethod == null) {
        throw new ShouldNeverHappenException("TCC resource is not available, resourceId:" + resourceId);
    }
    try {
        boolean result = false;
        //BusinessActionContext
        BusinessActionContext businessActionContext = getBusinessActionContext(xid, branchId, resourceId,
            applicationData);
        Object ret = rollbackMethod.invoke(targetTCCBean, businessActionContext);
        LOGGER.info(
            "TCC resource rollback result :" + ret + ", xid:" + xid + ", branchId:" + branchId + ", resourceId:"
                + resourceId);
        if (ret != null) {
            if (ret instanceof TwoPhaseResult) {
                result = ((TwoPhaseResult)ret).isSuccess();
            } else {
                result = (boolean)ret;
            }
        }
        return result ? BranchStatus.PhaseTwo_Rollbacked : BranchStatus.PhaseTwo_RollbackFailed_Retryable;
    } catch (Throwable t) {
        String msg = String.format("rollback TCC resource error, resourceId: %s, xid: %s.", resourceId, xid);
        LOGGER.error(msg, t);
        throw new FrameworkException(t, msg);
    }
}
```

### 同库模式

调用方切面不再向 TC 注册了，而是直接往业务的数据库里插入一条事务记录。

# SAGA模式

在 Saga 模式下，分布式事务内有多个子事务，每一个子事务都有一个补偿服务。分布式事务执行过程中，依次执行子事务，如果所有子事务均执行成功，那么分布式事务提交。如果任何一个子事务操作执行失败，那么分布式事务会退回去执行前面各个子事务的补偿操作，回滚已提交的子事务。

Seata-tms

事务管理

io.seata.tm.DefaultTransactionManager 全局事务管理，事务开启、提交、回滚、远程调用

io.seata.tm.TMClient#init   初始化TmRpcClient

io.seata.tm.api.TransactionalTemplate 事务模板（获取事务信息、开启全局事务、执行业务逻辑、正常提交事务、异常回滚事务）

io.seata.tm.api.DefaultGlobalTransaction 默认全局事务 事务开启的入口

# TC

io.seata.server.Server#main

```java
public static void main(String[] args) throws IOException {
    
  //命令行参数解析
    ParameterParser parameterParser = new ParameterParser(args);

    //初始化 metrics
    MetricsManager.get().init();

  	//存储模式
    System.setProperty(ConfigurationKeys.STORE_MODE, parameterParser.getStoreMode());

    RpcServer rpcServer = new RpcServer(WORKING_THREADS);
    //server port
    rpcServer.setListenPort(parameterParser.getPort());
    UUIDGenerator.init(parameterParser.getServerNode());
    //初始化SessionManager
    SessionHolder.init(parameterParser.getStoreMode());

   //处理远端请求
    DefaultCoordinator coordinator = new DefaultCoordinator(rpcServer);
    coordinator.init();
    rpcServer.setHandler(coordinator);
    // register ShutdownHook
    ShutdownHook.getInstance().addDisposable(coordinator);

    //127.0.0.1 and 0.0.0.0 are not valid here.
    if (NetUtil.isValidIp(parameterParser.getHost(), false)) {
        XID.setIpAddress(parameterParser.getHost());
    } else {
        XID.setIpAddress(NetUtil.getLocalIp());
    }
    XID.setPort(rpcServer.getListenPort());

    try {
        //服务端启动
        rpcServer.init();
    } catch (Throwable e) {
        LOGGER.error("rpcServer init error:{}", e.getMessage(), e);
        System.exit(-1);
    }

    System.exit(0);
}
```

## DefaultCoordinator

```java
//初始化定时器
public void init() {
    retryRollbacking.scheduleAtFixedRate(() -> {
        try {
         		//处理回滚失败的事务
            handleRetryRollbacking();
        } catch (Exception e) {
            LOGGER.info("Exception retry rollbacking ... ", e);
        }
    }, 0, ROLLBACKING_RETRY_PERIOD, TimeUnit.MILLISECONDS);

    retryCommitting.scheduleAtFixedRate(() -> {
        try {
          		//处理提交失败的事务
            handleRetryCommitting();
        } catch (Exception e) {
            LOGGER.info("Exception retry committing ... ", e);
        }
    }, 0, COMMITTING_RETRY_PERIOD, TimeUnit.MILLISECONDS);

    asyncCommitting.scheduleAtFixedRate(() -> {
        try {
          	//处理异步提交的任务
            handleAsyncCommitting();
        } catch (Exception e) {
            LOGGER.info("Exception async committing ... ", e);
        }
    }, 0, ASYN_COMMITTING_RETRY_PERIOD, TimeUnit.MILLISECONDS);

    timeoutCheck.scheduleAtFixedRate(() -> {
        try {
          	//检测事务是否超时
            timeoutCheck();
        } catch (Exception e) {
            LOGGER.info("Exception timeout checking ... ", e);
        }
    }, 0, TIMEOUT_RETRY_PERIOD, TimeUnit.MILLISECONDS);

    undoLogDelete.scheduleAtFixedRate(() -> {
        try {
          	//删除undolog
            undoLogDelete();
        } catch (Exception e) {
            LOGGER.info("Exception undoLog deleting ... ", e);
        }
    }, UNDOLOG_DELAY_DELETE_PERIOD, UNDOLOG_DELETE_PERIOD, TimeUnit.MILLISECONDS);
}
```

io.seata.server.coordinator.DefaultCoordinator#doGlobalBegin

```java
protected void doGlobalBegin(GlobalBeginRequest request, GlobalBeginResponse response, RpcContext rpcContext)
    throws TransactionException {
    response.setXid(core.begin(rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(),
        request.getTransactionName(), request.getTimeout()));
}
```

io.seata.server.coordinator.DefaultCore#begin

```java
public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
    throws TransactionException {
  //创建GlobalSession，生成全局事务Id
    GlobalSession session = GlobalSession.createGlobalSession(applicationId, transactionServiceGroup, name,
        timeout);
   //添加SessionLifecycleListener
    session.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
	 //会话开始
    session.begin();

    //transaction start event
    eventBus.post(new GlobalTransactionEvent(session.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
        session.getTransactionName(), session.getBeginTime(), null, session.getStatus()));

    LOGGER.info("Successfully begin global transaction xid = {}", session.getXid());
    return session.getXid(); //返回全局事务Id
}
```

io.seata.server.session.GlobalSession#begin

```java
public void begin() throws TransactionException {
    this.status = GlobalStatus.Begin;//全局事务状态
    this.beginTime = System.currentTimeMillis(); //全局事务开始时间
    this.active = true;
    for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
        lifecycleListener.onBegin(this); // 将全局事务保存到数据库
    }
}
```

io.seata.server.session.AbstractSessionManager#onBegin

```java
@Override
public void onBegin(GlobalSession globalSession) throws TransactionException {
    addGlobalSession(globalSession);
}
```

```java
@Override
public void addGlobalSession(GlobalSession session) throws TransactionException {
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("MANAGER[" + name + "] SESSION[" + session + "] " + LogOperation.GLOBAL_ADD);
    }
    writeSession(LogOperation.GLOBAL_ADD, session);
}
```

```java
private void writeSession(LogOperation logOperation, SessionStorable sessionStorable) throws TransactionException {
    if (!transactionStoreManager.writeSession(logOperation, sessionStorable)) {
        if (LogOperation.GLOBAL_ADD.equals(logOperation)) {
            throw new GlobalTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Fail to store global session");
        } else if (LogOperation.GLOBAL_UPDATE.equals(logOperation)) {
            throw new GlobalTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Fail to update global session");
        } else if (LogOperation.GLOBAL_REMOVE.equals(logOperation)) {
            throw new GlobalTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Fail to remove global session");
        } else if (LogOperation.BRANCH_ADD.equals(logOperation)) {
            throw new BranchTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Fail to store branch session");
        } else if (LogOperation.BRANCH_UPDATE.equals(logOperation)) {
            throw new BranchTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Fail to update branch session");
        } else if (LogOperation.BRANCH_REMOVE.equals(logOperation)) {
            throw new BranchTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Fail to remove branch session");
        } else {
            throw new BranchTransactionException(TransactionExceptionCode.FailedWriteSession,
                "Unknown LogOperation:" + logOperation.name());
        }
    }
}
```

io.seata.server.store.db.DatabaseTransactionStoreManager#writeSession

```java
public boolean writeSession(LogOperation logOperation, SessionStorable session) {
    if (LogOperation.GLOBAL_ADD.equals(logOperation)) {
        return logStore.insertGlobalTransactionDO(convertGlobalTransactionDO(session));
    } else if (LogOperation.GLOBAL_UPDATE.equals(logOperation)) {
        return logStore.updateGlobalTransactionDO(convertGlobalTransactionDO(session));
    } else if (LogOperation.GLOBAL_REMOVE.equals(logOperation)) {
        return logStore.deleteGlobalTransactionDO(convertGlobalTransactionDO(session));
    } else if (LogOperation.BRANCH_ADD.equals(logOperation)) {
        return logStore.insertBranchTransactionDO(convertBranchTransactionDO(session));
    } else if (LogOperation.BRANCH_UPDATE.equals(logOperation)) {
        return logStore.updateBranchTransactionDO(convertBranchTransactionDO(session));
    } else if (LogOperation.BRANCH_REMOVE.equals(logOperation)) {
        return logStore.deleteBranchTransactionDO(convertBranchTransactionDO(session));
    } else {
        throw new StoreException("Unknown LogOperation:" + logOperation.name());
    }
}
```

io.seata.core.rpc.netty.RpcServer#init

```java
public void init() {
    super.init();
    setChannelHandlers(RpcServer.this);
    DefaultServerMessageListenerImpl defaultServerMessageListenerImpl = new DefaultServerMessageListenerImpl(
        transactionMessageHandler);
    defaultServerMessageListenerImpl.init();
    defaultServerMessageListenerImpl.setServerMessageSender(this);
    this.setServerMessageListener(defaultServerMessageListenerImpl);
    super.start();

}
```
