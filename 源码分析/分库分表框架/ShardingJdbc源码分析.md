# ShardingJdbc源码分析

* [ShardingDataSource](#shardingdatasource)
* [ShardingConnection](#shardingconnection)
* [ShardingPreparedStatement](#shardingpreparedstatement)
* [执行sql](#执行sql)
  * [计算路由](#计算路由)
  * [初始化PreparedStatementExecutor](#初始化preparedstatementexecutor)
  * [executeQuery](#executequery)
    * [同步执行](#同步执行)
    * [异步执行](#异步执行)
  * [归并](#归并)
    * [遍历归并](#遍历归并)
    * [排序归并](#排序归并)


# ShardingDataSource

org.apache.shardingsphere.shardingjdbc.api.ShardingDataSourceFactory#createDataSource

```java
public static DataSource createDataSource(
        final Map<String, DataSource> dataSourceMap, final ShardingRuleConfiguration shardingRuleConfig, final Properties props) throws SQLException { //创建数据源
    return new ShardingDataSource(dataSourceMap, new ShardingRule(shardingRuleConfig, dataSourceMap.keySet()), props);
}
```

# ShardingConnection

org.apache.shardingsphere.shardingjdbc.jdbc.core.datasource.ShardingDataSource#getConnection

```java
public final ShardingConnection getConnection() { //获取连接
    return new ShardingConnection(getDataSourceMap(), runtimeContext, TransactionTypeHolder.get());
}
```

# ShardingPreparedStatement

org.apache.shardingsphere.shardingjdbc.jdbc.core.connection.ShardingConnection#prepareStatement(java.lang.String)

```java
public PreparedStatement prepareStatement(final String sql) throws SQLException {
    return new ShardingPreparedStatement(this, sql);
}
```

# 执行sql

org.apache.shardingsphere.shardingjdbc.jdbc.core.statement.ShardingPreparedStatement#executeQuery

```java
public ResultSet executeQuery() throws SQLException {
    ResultSet result;
    try {
        clearPrevious();//清除PreparedStatementExecutor的变量
        prepare(); // 计算路由
        initPreparedStatementExecutor(); //初始化PreparedStatementExecutor
        //执行sql，合并结果
        MergedResult mergedResult = mergeQuery(preparedStatementExecutor.executeQuery());
        result = new ShardingResultSet(preparedStatementExecutor.getResultSets(), mergedResult, this, executionContext);
    } finally {
        clearBatch();
    }
    currentResultSet = result;
    return result;
}
```

## 计算路由

org.apache.shardingsphere.shardingjdbc.jdbc.core.statement.ShardingPreparedStatement#prepare

```java
private void prepare() {
    executionContext = prepareEngine.prepare(sql, getParameters());
    findGeneratedKey().ifPresent(generatedKey -> generatedValues.add(generatedKey.getGeneratedValues().getLast()));
}
```

## 初始化PreparedStatementExecutor

org.apache.shardingsphere.shardingjdbc.jdbc.core.statement.ShardingPreparedStatement#initPreparedStatementExecutor

```java
private void initPreparedStatementExecutor() throws SQLException {
    preparedStatementExecutor.init(executionContext); //执行sql前的准备工作，根据路由结果确定连接模式，为每个sql建立连接、创建Statement
    setParametersForStatements(); //设置sql参数
    replayMethodForStatements(); //设置statement参数
}
```

org.apache.shardingsphere.shardingjdbc.executor.PreparedStatementExecutor#init

```java
public void init(final ExecutionContext executionContext) throws SQLException {
    setSqlStatementContext(executionContext.getSqlStatementContext());
    getInputGroups().addAll(obtainExecuteGroups(executionContext.getExecutionUnits()));
    cacheStatements();
}
```

org.apache.shardingsphere.shardingjdbc.executor.PreparedStatementExecutor#obtainExecuteGroups

```java
   private Collection<InputGroup<StatementExecuteUnit>> obtainExecuteGroups(final Collection<ExecutionUnit> executionUnits) throws SQLException {
        return getSqlExecutePrepareTemplate().getExecuteUnitGroups(executionUnits, new SQLExecutePrepareCallback() {
            
            @Override
            public List<Connection> getConnections(final ConnectionMode connectionMode, final String dataSourceName, final int connectionSize) throws SQLException {
                return PreparedStatementExecutor.super.getConnection().getConnections(connectionMode, dataSourceName, connectionSize);
            }
            
            @Override
            public StatementExecuteUnit createStatementExecuteUnit(final Connection connection, final ExecutionUnit executionUnit, final ConnectionMode connectionMode) throws SQLException {
                return new StatementExecuteUnit(executionUnit, createPreparedStatement(connection, executionUnit.getSqlUnit().getSql()), connectionMode);
            }
        });
    }
```

org.apache.shardingsphere.sharding.execute.sql.prepare.SQLExecutePrepareTemplate#getExecuteUnitGroups

```java
public Collection<InputGroup<StatementExecuteUnit>> getExecuteUnitGroups(final Collection<ExecutionUnit> executionUnits, final SQLExecutePrepareCallback callback) throws SQLException {
    return getSynchronizedExecuteUnitGroups(executionUnits, callback);
}
```

org.apache.shardingsphere.sharding.execute.sql.prepare.SQLExecutePrepareTemplate#getSynchronizedExecuteUnitGroups

```java
private Collection<InputGroup<StatementExecuteUnit>> getSynchronizedExecuteUnitGroups(
        final Collection<ExecutionUnit> executionUnits, final SQLExecutePrepareCallback callback) throws SQLException {
   //根据数据源对executionUnits进行分组，datasource -> List<SQLUnit>
    Map<String, List<SQLUnit>> sqlUnitGroups = getSQLUnitGroups(executionUnits);
    Collection<InputGroup<StatementExecuteUnit>> result = new LinkedList<>();
    for (Entry<String, List<SQLUnit>> entry : sqlUnitGroups.entrySet()) {
        result.addAll(getSQLExecuteGroups(entry.getKey(), entry.getValue(), callback));
    }
    return result;
}
```

org.apache.shardingsphere.sharding.execute.sql.prepare.SQLExecutePrepareTemplate#getSQLExecuteGroups

```java
private List<InputGroup<StatementExecuteUnit>> getSQLExecuteGroups(final String dataSourceName,
                                                                   final List<SQLUnit> sqlUnits, final SQLExecutePrepareCallback callback) throws SQLException {
    List<InputGroup<StatementExecuteUnit>> result = new LinkedList<>();
    //对同一数据源的sql根据desiredPartitionSize进行分组，每组对应一个数据库连接
    int desiredPartitionSize = Math.max(0 == sqlUnits.size() % maxConnectionsSizePerQuery ? sqlUnits.size() / maxConnectionsSizePerQuery : sqlUnits.size() / maxConnectionsSizePerQuery + 1, 1);
    List<List<SQLUnit>> sqlUnitPartitions = Lists.partition(sqlUnits, desiredPartitionSize);
  
    //连接模式
    ConnectionMode connectionMode = maxConnectionsSizePerQuery < sqlUnits.size() ? ConnectionMode.CONNECTION_STRICTLY : ConnectionMode.MEMORY_STRICTLY;
  
    //获取连接
    List<Connection> connections = callback.getConnections(connectionMode, dataSourceName, sqlUnitPartitions.size());
    int count = 0;
  	
    for (List<SQLUnit> each : sqlUnitPartitions) {
        result.add(getSQLExecuteGroup(connectionMode, connections.get(count++), dataSourceName, each, callback));//创建InputGroup<StatementExecuteUnit>
    }
    return result;
}
```

org.apache.shardingsphere.sharding.execute.sql.prepare.SQLExecutePrepareTemplate#getSQLExecuteGroup

```java
private InputGroup<StatementExecuteUnit> getSQLExecuteGroup(final ConnectionMode connectionMode, final Connection connection,
                                                            final String dataSourceName, final List<SQLUnit> sqlUnitGroup, final SQLExecutePrepareCallback callback) throws SQLException {
    List<StatementExecuteUnit> result = new LinkedList<>();
    for (SQLUnit each : sqlUnitGroup) {
        result.add(callback.createStatementExecuteUnit(connection, new ExecutionUnit(dataSourceName, each), connectionMode));
    }
    return new InputGroup<>(result);
}
```

org.apache.shardingsphere.sharding.execute.sql.prepare.SQLExecutePrepareCallback#createStatementExecuteUnit

```java
public StatementExecuteUnit createStatementExecuteUnit(final Connection connection, final ExecutionUnit executionUnit, final ConnectionMode connectionMode) throws SQLException {
    return new StatementExecuteUnit(executionUnit, 	
                                    connection.createStatement(getResultSetType(), getResultSetConcurrency(), getResultSetHoldability()),//创建Statement
                                    connectionMode);
}
```

## executeQuery

org.apache.shardingsphere.shardingjdbc.executor.PreparedStatementExecutor#executeQuery

```java
public List<QueryResult> executeQuery() throws SQLException {
    final boolean isExceptionThrown = ExecutorExceptionHandler.isExceptionThrown();
    SQLExecuteCallback<QueryResult> executeCallback = new SQLExecuteCallback<QueryResult>(getDatabaseType(), isExceptionThrown) {
        
        @Override
        protected QueryResult executeSQL(final String sql, final Statement statement, final ConnectionMode connectionMode) throws SQLException {
            return getQueryResult(statement, connectionMode);
        }
    };
    return executeCallback(executeCallback);
}
```

org.apache.shardingsphere.shardingjdbc.executor.AbstractStatementExecutor#executeCallback

```java
protected final <T> List<T> executeCallback(final SQLExecuteCallback<T> executeCallback) throws SQLException {
    List<T> result = sqlExecuteTemplate.execute((Collection) inputGroups, executeCallback);
    refreshMetaDataIfNeeded(connection.getRuntimeContext(), sqlStatementContext);
    return result;
}
```

org.apache.shardingsphere.sharding.execute.sql.execute.SQLExecuteTemplate#execute

```java
public <T> List<T> execute(final Collection<InputGroup<? extends StatementExecuteUnit>> inputGroups,
                           final SQLExecuteCallback<T> firstCallback, final SQLExecuteCallback<T> callback) throws SQLException {
    try {
        return executorEngine.execute((Collection) inputGroups, firstCallback, callback, serial);
    } catch (final SQLException ex) {
        ExecutorExceptionHandler.handleException(ex);
        return Collections.emptyList();
    }
}
```

org.apache.shardingsphere.underlying.executor.engine.ExecutorEngine#execute

```java
public <I, O> List<O> execute(final Collection<InputGroup<I>> inputGroups, 
                              final GroupedCallback<I, O> firstCallback, final GroupedCallback<I, O> callback, final boolean serial) throws SQLException {
    if (inputGroups.isEmpty()) {
        return Collections.emptyList();
    }
    return serial ? serialExecute(inputGroups, firstCallback, callback) : parallelExecute(inputGroups, firstCallback, callback);
}
```

### 同步执行

org.apache.shardingsphere.underlying.executor.engine.ExecutorEngine#serialExecute

```java
private <I, O> List<O> serialExecute(final Collection<InputGroup<I>> inputGroups, final GroupedCallback<I, O> firstCallback, final GroupedCallback<I, O> callback) throws SQLException { //同步执行
    Iterator<InputGroup<I>> inputGroupsIterator = inputGroups.iterator();
    InputGroup<I> firstInputs = inputGroupsIterator.next();
    List<O> result = new LinkedList<>(syncExecute(firstInputs, null == firstCallback ? callback : firstCallback));
    for (InputGroup<I> each : Lists.newArrayList(inputGroupsIterator)) {
        result.addAll(syncExecute(each, callback));
    }
    return result;
}
```

### 异步执行

org.apache.shardingsphere.underlying.executor.engine.ExecutorEngine#syncExecute

```java
private <I, O> Collection<O> syncExecute(final InputGroup<I> inputGroup, final GroupedCallback<I, O> callback) throws SQLException {
    return callback.execute(inputGroup.getInputs(), true, ExecutorDataMap.getValue());
}
```

org.apache.shardingsphere.sharding.execute.sql.execute.SQLExecuteCallback#execute

```java
public final Collection<T> execute(final Collection<StatementExecuteUnit> statementExecuteUnits, 
                                   final boolean isTrunkThread, final Map<String, Object> dataMap) throws SQLException {
    Collection<T> result = new LinkedList<>();
    for (StatementExecuteUnit each : statementExecuteUnits) {//ExecutionUnit、Statement、ConnectionMode
        result.add(execute0(each, isTrunkThread, dataMap));
    }
    return result;
}
```

org.apache.shardingsphere.sharding.execute.sql.execute.SQLExecuteCallback#execute0

```java
private T execute0(final StatementExecuteUnit statementExecuteUnit, final boolean isTrunkThread, final Map<String, Object> dataMap) throws SQLException {
    ExecutorExceptionHandler.setExceptionThrown(isExceptionThrown);
  
    DataSourceMetaData dataSourceMetaData = getDataSourceMetaData(statementExecuteUnit.getStatement().getConnection().getMetaData());
  	//执行钩子
    SQLExecutionHook sqlExecutionHook = new SPISQLExecutionHook();
    try {
      	//执行单元，封装了数据源、sql语句、sql参数
        ExecutionUnit executionUnit = statementExecuteUnit.getExecutionUnit();
        sqlExecutionHook.start(executionUnit.getDataSourceName(), executionUnit.getSqlUnit().getSql(), executionUnit.getSqlUnit().getParameters(), dataSourceMetaData, isTrunkThread, dataMap);
      
        T result = executeSQL(executionUnit.getSqlUnit().getSql(), statementExecuteUnit.getStatement(), statementExecuteUnit.getConnectionMode());
      
        sqlExecutionHook.finishSuccess();
        return result;
    } catch (final SQLException ex) {
        sqlExecutionHook.finishFailure(ex);
        ExecutorExceptionHandler.handleException(ex);
        return null;
    }
}
```

org.apache.shardingsphere.sharding.execute.sql.execute.SQLExecuteCallback#executeSQL

```java
protected QueryResult executeSQL(final String sql, final Statement statement, final ConnectionMode connectionMode) throws SQLException {
    return getQueryResult(statement, connectionMode);
}
```

org.apache.shardingsphere.shardingjdbc.executor.PreparedStatementExecutor#getQueryResult

```java
private QueryResult getQueryResult(final Statement statement, final ConnectionMode connectionMode) throws SQLException {
    PreparedStatement preparedStatement = (PreparedStatement) statement;
    ResultSet resultSet = preparedStatement.executeQuery(); //执行jdbc
    getResultSets().add(resultSet); //添加结果集
    return ConnectionMode.MEMORY_STRICTLY == connectionMode ? new StreamQueryResult(resultSet) //内存限制模式，多条连接并发执行sql，返回流式结果集
      : new MemoryQueryResult(resultSet);//连接限制模式，串行执行，一条连接串行执行多条语句返回内存结果集
}
```

org.apache.shardingsphere.underlying.executor.engine.ExecutorEngine#parallelExecute

```java
private <I, O> List<O> parallelExecute(final Collection<InputGroup<I>> inputGroups, final GroupedCallback<I, O> firstCallback, final GroupedCallback<I, O> callback) throws SQLException { //异步执行
    Iterator<InputGroup<I>> inputGroupsIterator = inputGroups.iterator();
    InputGroup<I> firstInputs = inputGroupsIterator.next();
    Collection<ListenableFuture<Collection<O>>> restResultFutures = asyncExecute(Lists.newArrayList(inputGroupsIterator), callback);
    return getGroupResults(syncExecute(firstInputs, null == firstCallback ? callback : firstCallback), restResultFutures);
}
```

## 归并

org.apache.shardingsphere.shardingjdbc.jdbc.core.statement.ShardingPreparedStatement#mergeQuery

```java
private MergedResult mergeQuery(final List<QueryResult> queryResults) throws SQLException {
    ShardingRuntimeContext runtimeContext = connection.getRuntimeContext();
  	//创建MergeEngine
    MergeEngine mergeEngine = new MergeEngine(runtimeContext.getRule().toRules(), runtimeContext.getProperties(), runtimeContext.getDatabaseType(), runtimeContext.getMetaData().getSchema());
  	//合并
    return mergeEngine.merge(queryResults, executionContext.getSqlStatementContext());
}
```

org.apache.shardingsphere.underlying.pluggble.merge.MergeEngine#merge

```java
public MergedResult merge(final List<QueryResult> queryResults, final SQLStatementContext sqlStatementContext) throws SQLException {
    registerMergeDecorator();
    return merger.process(queryResults, sqlStatementContext);
}
```

org.apache.shardingsphere.underlying.pluggble.merge.MergeEngine#registerMergeDecorator

```java
private void registerMergeDecorator() { //注册结果处理引擎
    for (ResultProcessEngine each : OrderedSPIRegistry.getRegisteredServices(ResultProcessEngine.class)) {
        Class<?> ruleClass = (Class<?>) each.getType();
        // FIXME rule.getClass().getSuperclass() == ruleClass for orchestration, should decouple extend between orchestration rule and sharding rule
        rules.stream().filter(rule -> rule.getClass() == ruleClass || rule.getClass().getSuperclass() == ruleClass).collect(Collectors.toList())
                .forEach(rule -> merger.registerProcessEngine(rule, each));
    }
}
```

org.apache.shardingsphere.underlying.merge.MergeEntry#process

```java
public MergedResult process(final List<QueryResult> queryResults, final SQLStatementContext sqlStatementContext) throws SQLException {
    Optional<MergedResult> mergedResult = merge(queryResults, sqlStatementContext);
    Optional<MergedResult> result = mergedResult.isPresent() ? Optional.of(decorate(mergedResult.get(), sqlStatementContext)) : decorate(queryResults.get(0), sqlStatementContext);
    return result.orElseGet(() -> new TransparentMergedResult(queryResults.get(0)));
}
```

org.apache.shardingsphere.underlying.merge.MergeEntry#merge

```java
private Optional<MergedResult> merge(final List<QueryResult> queryResults, final SQLStatementContext sqlStatementContext) throws SQLException {
    for (Entry<BaseRule, ResultProcessEngine> entry : engines.entrySet()) {
        if (entry.getValue() instanceof ResultMergerEngine) {
            ResultMerger resultMerger = ((ResultMergerEngine) entry.getValue()).newInstance(databaseType, entry.getKey(), properties, sqlStatementContext); //创建ResultMerger
            return Optional.of(resultMerger.merge(queryResults, sqlStatementContext, schemaMetaData)); //合并
        }
    }
    return Optional.empty();
}
```

org.apache.shardingsphere.sharding.merge.ShardingResultMergerEngine#newInstance

```java
public ResultMerger newInstance(final DatabaseType databaseType, final ShardingRule shardingRule, final ConfigurationProperties properties, final SQLStatementContext sqlStatementContext) {//创建ResultMerger
    if (sqlStatementContext instanceof SelectStatementContext) { //查询语句
        return new ShardingDQLResultMerger(databaseType);
    } 
    if (sqlStatementContext.getSqlStatement() instanceof DALStatement) {
        return new ShardingDALResultMerger(shardingRule);
    }
    return new TransparentResultMerger();
}
```

org.apache.shardingsphere.sharding.merge.dql.ShardingDQLResultMerger#merge

```java
public MergedResult merge(final List<QueryResult> queryResults, final SQLStatementContext sqlStatementContext, final SchemaMetaData schemaMetaData) throws SQLException {
    if (1 == queryResults.size()) {//只有一个结果集
        return new IteratorStreamMergedResult(queryResults);//遍历归并
    }
  	//过个结果集
    Map<String, Integer> columnLabelIndexMap = getColumnLabelIndexMap(queryResults.get(0));
    SelectStatementContext selectStatementContext = (SelectStatementContext) sqlStatementContext;
    selectStatementContext.setIndexes(columnLabelIndexMap);
    MergedResult mergedResult = build(queryResults, selectStatementContext, columnLabelIndexMap, schemaMetaData); //根据查询条件确定使用哪种归并
    return decorate(queryResults, selectStatementContext, mergedResult);
}
```

org.apache.shardingsphere.sharding.merge.dql.ShardingDQLResultMerger#build

```java
private MergedResult build(final List<QueryResult> queryResults, final SelectStatementContext selectStatementContext,
                           final Map<String, Integer> columnLabelIndexMap, final SchemaMetaData schemaMetaData) throws SQLException {
    //有分组语句或者聚合函数，则执行分组归并
    if (isNeedProcessGroupBy(selectStatementContext)) {
        return getGroupByMergedResult(queryResults, selectStatementContext, columnLabelIndexMap, schemaMetaData);
    }
  	//有Distinct去重
    if (isNeedProcessDistinctRow(selectStatementContext)) {
        setGroupByForDistinctRow(selectStatementContext);
        return getGroupByMergedResult(queryResults, selectStatementContext, columnLabelIndexMap, schemaMetaData);
    }
  	//有排序
    if (isNeedProcessOrderBy(selectStatementContext)) {
        return new OrderByStreamMergedResult(queryResults, selectStatementContext, schemaMetaData);
    }
  	//遍历归并
    return new IteratorStreamMergedResult(queryResults);
}
```

org.apache.shardingsphere.sharding.merge.dql.ShardingDQLResultMerger#getGroupByMergedResult

```java
private MergedResult getGroupByMergedResult(final List<QueryResult> queryResults, final SelectStatementContext selectStatementContext,
                                            final Map<String, Integer> columnLabelIndexMap, final SchemaMetaData schemaMetaData) throws SQLException {
    return selectStatementContext.isSameGroupByAndOrderByItems()//groupby和orderby字段是否相同
            ? new GroupByStreamMergedResult(columnLabelIndexMap, queryResults, selectStatementContext, schemaMetaData) //相同采用流式归并
            : new GroupByMemoryMergedResult(queryResults, selectStatementContext, schemaMetaData); //不同采用内存归并
}
```

org.apache.shardingsphere.sharding.merge.dql.ShardingDQLResultMerger#decorate

```java
private MergedResult decorate(final List<QueryResult> queryResults, final SelectStatementContext selectStatementContext, final MergedResult mergedResult) throws SQLException {
    PaginationContext paginationContext = selectStatementContext.getPaginationContext();
    if (!paginationContext.isHasPagination() || 1 == queryResults.size()) {
        return mergedResult;
    }
    String trunkDatabaseName = DatabaseTypes.getTrunkDatabaseType(databaseType.getName()).getName();
    if ("MySQL".equals(trunkDatabaseName) || "PostgreSQL".equals(trunkDatabaseName)) {
        return new LimitDecoratorMergedResult(mergedResult, paginationContext);
    }
    if ("Oracle".equals(trunkDatabaseName)) {
        return new RowNumberDecoratorMergedResult(mergedResult, paginationContext);
    }
    if ("SQLServer".equals(trunkDatabaseName)) {
        return new TopAndRowNumberDecoratorMergedResult(mergedResult, paginationContext);
    }
    return mergedResult;
}
```

### 遍历归并

org.apache.shardingsphere.sharding.merge.dql.iterator.IteratorStreamMergedResult#IteratorStreamMergedResult

```java
public IteratorStreamMergedResult(final List<QueryResult> queryResults) {
    this.queryResults = queryResults.iterator(); //结果集迭代器
    setCurrentQueryResult(this.queryResults.next()); //设置当前迭代的结果集
}
```

org.apache.shardingsphere.sharding.merge.dql.iterator.IteratorStreamMergedResult#next

```java
public boolean next() throws SQLException {
    if (getCurrentQueryResult().next()) { //当前结果集是否还有数据
        return true;
    }
    if (!queryResults.hasNext()) { //结果集迭代器是否还有未曾遍历的结果集
        return false;
    }
    setCurrentQueryResult(queryResults.next()); //设置最新的结果集
    boolean hasNext = getCurrentQueryResult().next(); //结果集是否有数据
    if (hasNext) { //有数据，返回true
        return true;
    }
  	//结果集可能无数据，循环找到有数据的结果集
    while (!hasNext && queryResults.hasNext()) { //当前结果集无数据但是还有未曾迭代的结果集
        setCurrentQueryResult(queryResults.next());
        hasNext = getCurrentQueryResult().next();
    }
    return hasNext;
}
```

获取具体的值

org.apache.shardingsphere.underlying.merge.result.impl.stream.StreamMergedResult#getValue

```java
public Object getValue(final int columnIndex, final Class<?> type) throws SQLException {
    Object result = getCurrentQueryResult().getValue(columnIndex, type);
    wasNull = getCurrentQueryResult().wasNull();
    return result;
}
```

### 排序归并

org.apache.shardingsphere.sharding.merge.dql.groupby.GroupByStreamMergedResult#GroupByStreamMergedResult

```java
public GroupByStreamMergedResult(final Map<String, Integer> labelAndIndexMap, final List<QueryResult> queryResults,
                                 final SelectStatementContext selectStatementContext, final SchemaMetaData schemaMetaData) throws SQLException {
    super(queryResults, selectStatementContext, schemaMetaData);
    this.selectStatementContext = selectStatementContext;
    currentRow = new ArrayList<>(labelAndIndexMap.size());
    currentGroupByValues = getOrderByValuesQueue().isEmpty()
            ? Collections.emptyList() : new GroupByValue(getCurrentQueryResult(), selectStatementContext.getGroupByContext().getItems()).getGroupValues();
}
```

org.apache.shardingsphere.sharding.merge.dql.orderby.OrderByStreamMergedResult#OrderByStreamMergedResult

```java
public OrderByStreamMergedResult(final List<QueryResult> queryResults, final SelectStatementContext selectStatementContext, final SchemaMetaData schemaMetaData) throws SQLException {
    this.orderByItems = selectStatementContext.getOrderByContext().getItems();
    this.orderByValuesQueue = new PriorityQueue<>(queryResults.size()); //创建优先队列
    orderResultSetsToQueue(queryResults, selectStatementContext, schemaMetaData);
    isFirstNext = true;
}
```

org.apache.shardingsphere.sharding.merge.dql.orderby.OrderByStreamMergedResult#orderResultSetsToQueue

```java
private void orderResultSetsToQueue(final List<QueryResult> queryResults, final SelectStatementContext selectStatementContext, final SchemaMetaData schemaMetaData) throws SQLException {
    for (QueryResult each : queryResults) {
        OrderByValue orderByValue = new OrderByValue(each, orderByItems, selectStatementContext, schemaMetaData);
        if (orderByValue.next()) {
            orderByValuesQueue.offer(orderByValue); //将结果集放入优先队列
        }
    }
    setCurrentQueryResult(orderByValuesQueue.isEmpty() ? queryResults.get(0) : orderByValuesQueue.peek().getQueryResult()); //获取堆顶的结果集
}
```

org.apache.shardingsphere.sharding.merge.dql.groupby.GroupByStreamMergedResult#next

```java
public boolean next() throws SQLException {
    currentRow.clear();
    if (getOrderByValuesQueue().isEmpty()) {
        return false;
    }
    if (isFirstNext()) { //第一次获取
        super.next();
    }
    if (aggregateCurrentGroupByRowAndNext()) {
        currentGroupByValues = new GroupByValue(getCurrentQueryResult(), selectStatementContext.getGroupByContext().getItems()).getGroupValues();
    }
    return true;
}
```

org.apache.shardingsphere.sharding.merge.dql.orderby.OrderByStreamMergedResult#next

```java
public boolean next() throws SQLException {
    if (orderByValuesQueue.isEmpty()) { //优先队列为空，返回false
        return false;
    }
    if (isFirstNext) { //第一次遍历，返回true
        isFirstNext = false;
        return true;
    }
    OrderByValue firstOrderByValue = orderByValuesQueue.poll();//取出堆顶结果集
    if (firstOrderByValue.next()) { //结果集还有数据，游标后移，再次放入队列
        orderByValuesQueue.offer(firstOrderByValue);
    }
    if (orderByValuesQueue.isEmpty()) {//优先队列为空，返回false
        return false;
    }
  	//获取堆顶结果集
    setCurrentQueryResult(orderByValuesQueue.peek().getQueryResult());
    return true;
}
```

org.apache.shardingsphere.sharding.merge.dql.groupby.GroupByStreamMergedResult#aggregateCurrentGroupByRowAndNext

```java
private boolean aggregateCurrentGroupByRowAndNext() throws SQLException {
    boolean result = false;
    Map<AggregationProjection, AggregationUnit> aggregationUnitMap = Maps.toMap(
            selectStatementContext.getProjectionsContext().getAggregationProjections(), input -> AggregationUnitFactory.create(input.getType(), input instanceof AggregationDistinctProjection));
 	 //合并分组条件相同的记录
    while (currentGroupByValues.equals(new GroupByValue(getCurrentQueryResult(), selectStatementContext.getGroupByContext().getItems()).getGroupValues())) {
        aggregate(aggregationUnitMap); //聚和
        cacheCurrdentRow(); //设置当前记录到currentRow
       //获取下一条记录，调用父类中的next方法从而使得currentResultSet指向下一个元素，从其他resultSet查找分组条件相同的记录
        result = super.next();
        if (!result) {
            break;
        }
    }
  	//设置聚和结果到currentRow
    setAggregationValueToCurrentRow(aggregationUnitMap);
    return result;
}
```

创建 AggregationUnit

org.apache.shardingsphere.sharding.merge.dql.groupby.aggregation.AggregationUnitFactory#create

```java
public static AggregationUnit create(final AggregationType type, final boolean isDistinct) {
    switch (type) {
        case MAX:
            return new ComparableAggregationUnit(false);
        case MIN:
            return new ComparableAggregationUnit(true);
        case SUM:
            return isDistinct ? new DistinctSumAggregationUnit() : new AccumulationAggregationUnit();
        case COUNT:
            return isDistinct ? new DistinctCountAggregationUnit() : new AccumulationAggregationUnit();
        case AVG:
            return isDistinct ? new DistinctAverageAggregationUnit() : new AverageAggregationUnit();
        default:
            throw new UnsupportedOperationException(type.name());
    }
}
```

