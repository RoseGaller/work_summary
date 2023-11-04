# Seata源码分析

* [AT模式](#at模式)
  * [流程](#流程)
    * [一阶段](#一阶段)
    * [二阶段提交](#二阶段提交)
    * [二阶段回滚](#二阶段回滚)
  * [实现](#实现)
    * [自动代理](#自动代理)
    * [直接注入代理数据源](#直接注入代理数据源)
    * [全局事务](#全局事务)
      * [注册全局事务](#注册全局事务)
      * [AT事务执行](#at事务执行)
      * [AT事务提交](#at事务提交)
        * [注册分支事务](#注册分支事务)
        * [识别锁冲突异常,稍后重试](#识别锁冲突异常稍后重试)
        * [上报分支事务执行结果](#上报分支事务执行结果)
      * [全局事务提交](#全局事务提交)
      * [全局事务回滚](#全局事务回滚)
    * [接收TC分支事务提交请求](#接收tc分支事务提交请求)
    * [接收TC分支事务回滚请求](#接收tc分支事务回滚请求)
    * [undolog过期删除](#undolog过期删除)
* [TCC模式](#tcc模式)
  * [流程](#流程-1)
    * [一阶段](#一阶段-1)
    * [二阶段提交](#二阶段提交-1)
    * [二阶段回滚](#二阶段回滚-1)
  * [问题](#问题)
    * [空回滚](#空回滚)
    * [悬挂控制](#悬挂控制)
    * [幂等控制](#幂等控制)
  * [优化](#优化)
    * [异步执行](#异步执行)
    * [同库模式](#同库模式)
* [SAGA模式](#saga模式)



# AT模式

AT模式是一种对业务无任何侵入的分布式事务解决方案。用户只需编写“业务 SQL”，便能轻松接入分布式事务

## 流程

### 一阶段

在一阶段，Seata 会拦截“业务 SQL”，先解析 SQL 语义，找到“业务 SQL”要更新的业务数据，在业务数据被更新前，将其保存成“before image”，然后执行“业务 SQL”更新业务数据，在业务数据更新之后，再将其保存成“after image”，最后生成行锁。

### 二阶段提交

二阶段如果是提交的话，因为“业务 SQL”在一阶段已经提交至数据库， 所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可

### 二阶段回滚

二阶段如果是回滚的话，Seata 就需要回滚一阶段已经执行的“业务 SQL”，还原业务数据。回滚方式便是用“before image”还原业务数据；但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。

## 实现

### 自动代理

```java
client.support.spring.datasource.autoproxy=true //自动对DataSource进行代理
```

io.seata.spring.annotation.GlobalTransactionScanner#postProcessAfterInitialization

```java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {//创建对DataSource的代理
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

### 直接注入代理数据源

### 全局事务

GlobalTransactional注解标志全局事务的入口。GlobalTransactionScanner对标有GlobalTransactional注解的方法的类进行代理。

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

事务执行

1. 获取或创建全局事务
2. 向TC注册（事务名称、超时时间），获取全局事务Id
3. 执行业务逻辑
4. 出现异常，回滚全局事务
5. 提交全局事务
6. 执行TransactionHook的afterCompletion方法

io.seata.tm.api.TransactionalTemplate#execute

```java
public Object execute(TransactionalExecutor business) throws Throwable {
    // 1. 获取或创建全局事务
    GlobalTransaction tx = GlobalTransactionContext.getCurrentOrCreate();
    //  获取事务信息（事务名称、超时时间、回滚策略）
    TransactionInfo txInfo = business.getTransactionInfo();
    if (txInfo == null) {
        throw new ShouldNeverHappenException("transactionInfo does not exist");
    }
    try {
        // 2、向TC注册（事务名称、超时时间），获取全局事务Id
        beginTransaction(txInfo, tx);
        Object rs = null;
        try {
            //3.执行业务逻辑
            rs = business.execute();
        } catch (Throwable ex) {
            // 4.回滚事务
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

#### 注册全局事务

io.seata.tm.api.TransactionalTemplate#beginTransaction

```java
private void beginTransaction(TransactionInfo txInfo, GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
    try {
        triggerBeforeBegin();
        tx.begin(txInfo.getTimeOut(), txInfo.getName());
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
  	//注册，返回全局事务Id
    xid = transactionManager.begin(null, null, name, timeout);
    status = GlobalStatus.Begin;
    RootContext.bind(xid);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Begin new global transaction [{}]", xid);
    }
}
```

#### AT事务执行

识别sql语句，创建修改前镜像数据、执行sql语句、创建修改后镜像数据、创建undolog、lockkeys、绑定事务信息

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
	   //和普通的statement执行sql语句一样	
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
    //绑定全局事务Id
    if (RootContext.inGlobalTransaction()) {
        String xid = RootContext.getXID();
        statementProxy.getConnectionProxy().bind(xid);
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

#### AT事务提交

注册分支事务，包括lockkey，返回分支事务Id

识别锁冲突异常，稍后重试

保存undolog到数据库

本地事务提交

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

##### 注册分支事务

```java
private void register() throws TransactionException {
    Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, getDataSourceProxy().getResourceId(),
        null, context.getXid(), null, context.buildLockKeys()); //包括锁记录
    context.setBranchId(branchId);
}
```

##### 识别锁冲突异常,稍后重试

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

##### 上报分支事务执行结果

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

#### 全局事务提交

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

#### 全局事务回滚

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

### 接收TC分支事务提交请求

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

### 接收TC分支事务回滚请求

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

### undolog过期删除

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

### 同库模式

调用方切面不再向 TC 注册了，而是直接往业务的数据库里插入一条事务记录。

# SAGA模式

在 Saga 模式下，分布式事务内有多个子事务，每一个子事务都有一个补偿服务。分布式事务执行过程中，依次执行子事务，如果所有子事务均执行成功，那么分布式事务提交。如果任何一个子事务操作执行失败，那么分布式事务会退回去执行前面各个子事务的补偿操作，回滚已提交的子事务。
