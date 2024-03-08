# Mybatis源码分析

* [核心组件](#核心组件)
  * [SqlSessionFactoryBuilder](#sqlsessionfactorybuilder)
  * [Configuration](#configuration)
  * [SqlSessionFactory](#sqlsessionfactory)
  * [SqlSession](#sqlsession)
  * [Executor](#executor)
  * [InterceptorChain](#interceptorchain)
* [查询流程](#查询流程)
  * [直接SQL语句查询](#直接sql语句查询)
    * [MappedStatement](#mappedstatement)
    * [StatementHandler](#statementhandler)
    * [Statement](#statement)
    * [DefaultParameterHandler](#defaultparameterhandler)
    * [ResultHandler](#resulthandler)
  * [接口查询](#接口查询)
* [插件](#插件)

配置文件解析

mybatis-config.xml和XxxMapper.xml

# 核心组件

## SqlSessionFactoryBuilder

org.apache.ibatis.session.SqlSessionFactoryBuilder#build(java.io.Reader)

```java
public SqlSessionFactory build(Reader reader) { //创建SqlSessionFactory
  return build(reader, null, null);
}
```

org.apache.ibatis.session.SqlSessionFactoryBuilder#build(java.io.Reader, java.lang.String, java.util.Properties)

```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
  try {
    XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
    return build(parser.parse()); //解析xml文件，封装成Configuration
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      reader.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```

## Configuration

org.apache.ibatis.builder.xml.XMLConfigBuilder#parse

```java
public Configuration parse() { //封装xml信息
  if (parsed) {
    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
  }
  parsed = true;
  parseConfiguration(parser.evalNode("/configuration")); //解析XML文件，创建Configuration
  return configuration;
}
```

## SqlSessionFactory

org.apache.ibatis.session.SqlSessionFactoryBuilder#build(org.apache.ibatis.session.Configuration)

```java
public SqlSessionFactory build(Configuration config) {//负责创建SqlSession
  return new DefaultSqlSessionFactory(config);//根据配置信息创建DefaultSqlSessionFactory
}
```

## SqlSession

org.apache.ibatis.session.defaults.DefaultSqlSessionFactory#openSession()

```java
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}
```

org.apache.ibatis.session.defaults.DefaultSqlSessionFactory#openSessionFromDataSource

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    final Executor executor = configuration.newExecutor(tx, execType); //创建Executor
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

## Executor

org.apache.ibatis.session.Configuration#newExecutor(org.apache.ibatis.transaction.Transaction, org.apache.ibatis.session.ExecutorType)

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction); //批量操作
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);//对JDBC中的Statement对象做了缓存，当执行相同的SQL语句时，直接从缓存中取出Statement对象进行复用，避免了频繁创建和销毁Statement对象，从而提升系统性能，这是享元思想的应用
  } else {
    executor = new SimpleExecutor(this, transaction); //基础的Executor，能够完成基本的增删改查操作
  }
  if (cacheEnabled) { //启用二级缓存
    executor = new CachingExecutor(executor); //使用装饰模式，为BatchExecutor、ReuseExecutor、SimpleExecutor增加了缓存功能
  }
  executor = (Executor) interceptorChain.pluginAll(executor); //拦截器链
  return executor;
}
```

## InterceptorChain

为Executor、ParameterHandler、ResultSetHandler、StatementHandler创建代理对象

org.apache.ibatis.plugin.InterceptorChain#pluginAll

```java
public Object pluginAll(Object target) {
  for (Interceptor interceptor : interceptors) {
    target = interceptor.plugin(target);
  }
  return target;
}
```

# 查询流程

## 直接SQL语句查询

org.apache.ibatis.session.defaults.DefaultSqlSession#selectList(java.lang.String)

```java
public <E> List<E> selectList(String statement) {
  return this.selectList(statement, null);
}
```

org.apache.ibatis.session.defaults.DefaultSqlSession#selectList(java.lang.String, java.lang.Object)

```java
public <E> List<E> selectList(String statement, Object parameter) {
  return this.selectList(statement, parameter, RowBounds.DEFAULT);
}
```

### MappedStatement

org.apache.ibatis.session.defaults.DefaultSqlSession#selectList

```java
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    //根据MappedStatement
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

org.apache.ibatis.executor.BaseExecutor#query

```java
 public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
   BoundSql boundSql = ms.getBoundSql(parameter);
   CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
   return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

org.apache.ibatis.executor.BaseExecutor#query

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // issue #601
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // issue #482
      clearLocalCache();
    }
  }
  return list;
}
```

org.apache.ibatis.executor.BaseExecutor#queryFromDatabase

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    localCache.removeObject(key);
  }
  localCache.putObject(key, list);
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

org.apache.ibatis.executor.SimpleExecutor#doQuery

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    //创建StatementHandler
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    //创建Statement
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.<E>query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

### StatementHandler

org.apache.ibatis.session.Configuration#newStatementHandler

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  //封装了对JDBC Statement的操作
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  //对StatementHandler增强
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
```

org.apache.ibatis.executor.statement.RoutingStatementHandler#RoutingStatementHandler

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

  switch (ms.getStatementType()) {
    case STATEMENT:
      delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case PREPARED: //默认StatementHandler
      delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case CALLABLE:
      delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    default:
      throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
  }

}
```

### Statement

org.apache.ibatis.executor.SimpleExecutor#prepareStatement

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection, transaction.getTimeout());//创建Statement
  handler.parameterize(stmt); //设置参数
  return stmt;
}
```

org.apache.ibatis.executor.statement.PreparedStatementHandler#parameterize

```java
public void parameterize(Statement statement) throws SQLException {
  parameterHandler.setParameters((PreparedStatement) statement);//设置参数
}
```

### DefaultParameterHandler

org.apache.ibatis.scripting.defaults.DefaultParameterHandler#setParameters

```java
public void setParameters(PreparedStatement ps) {
  ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings != null) {
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        TypeHandler typeHandler = parameterMapping.getTypeHandler();
        JdbcType jdbcType = parameterMapping.getJdbcType();
        if (value == null && jdbcType == null) {
          jdbcType = configuration.getJdbcTypeForNull();
        }
        try {
          typeHandler.setParameter(ps, i + 1, value, jdbcType);
        } catch (TypeException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        } catch (SQLException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        }
      }
    }
  }
}
```

org.apache.ibatis.executor.statement.PreparedStatementHandler#query

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute(); //执行Sql语句
  return resultSetHandler.<E> handleResultSets(ps); //处理结果集
}
```

### ResultHandler

处理结果集

org.apache.ibatis.executor.resultset.DefaultResultSetHandler#handleResultSets

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

  final List<Object> multipleResults = new ArrayList<Object>();

  int resultSetCount = 0;
  ResultSetWrapper rsw = getFirstResultSet(stmt);

  List<ResultMap> resultMaps = mappedStatement.getResultMaps();
  int resultMapCount = resultMaps.size();
  validateResultMapsCount(rsw, resultMapCount);
  while (rsw != null && resultMapCount > resultSetCount) {
    ResultMap resultMap = resultMaps.get(resultSetCount);
    //处理结果集
    handleResultSet(rsw, resultMap, multipleResults, null);
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
  }
	//存储过程
  String[] resultSets = mappedStatement.getResultSets();
  if (resultSets != null) {
    while (rsw != null && resultSetCount < resultSets.length) {
      ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
      if (parentMapping != null) {
        String nestedResultMapId = parentMapping.getNestedResultMapId();
        ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
        handleResultSet(rsw, resultMap, null, parentMapping);
      }
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
  }
  return collapseSingleResultList(multipleResults);
}
```

org.apache.ibatis.executor.resultset.DefaultResultSetHandler#handleResultSet

```java
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
  try {
    if (parentMapping != null) {
      handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
    } else {
      if (resultHandler == null) {
        DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
        handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
        multipleResults.add(defaultResultHandler.getResultList());
      } else {
        handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
      }
    }
  } finally {
    // issue #228 (close resultsets)
    closeResultSet(rsw.getResultSet());
  }
}
```

## Mapper代理

创建代理对象，内部还是通过SqlSession来实现

org.apache.ibatis.session.defaults.DefaultSqlSession#getMapper

```java
public <T> T getMapper(Class<T> type) {
  return configuration.<T>getMapper(type, this);
}
```

org.apache.ibatis.session.Configuration#getMapper

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

org.apache.ibatis.binding.MapperRegistry#getMapper

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  //type -> MapperProxyFactory
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  //解析配置文件时，已放入map
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);//创建代理对象
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

org.apache.ibatis.binding.MapperProxyFactory#newInstance(org.apache.ibatis.session.SqlSession)

```java
public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

org.apache.ibatis.binding.MapperProxyFactory#newInstance(org.apache.ibatis.binding.MapperProxy<T>)

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```

org.apache.ibatis.binding.MapperProxy#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) { //Object方法
      return method.invoke(this, args); 
    } else if (isDefaultMethod(method)) { //默认方法
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  return mapperMethod.execute(sqlSession, args);
}
```

org.apache.ibatis.binding.MapperProxy#cachedMapperMethod

```java
private MapperMethod cachedMapperMethod(Method method) {//封装成MapperMethod
  MapperMethod mapperMethod = methodCache.get(method);
  if (mapperMethod == null) { 
    mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
    methodCache.put(method, mapperMethod);
  }
  return mapperMethod;
}
```

org.apache.ibatis.binding.MapperMethod#execute

```java
public Object execute(SqlSession sqlSession, Object[] args) {//解析参数，sqlsession执行sql
  Object result;
  switch (command.getType()) {
    case INSERT: {
      //解析Param注解的参数
   		Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName() 
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```

# 如何使用自增主键？

org.apache.ibatis.executor.statement.SimpleStatementHandler#update

```java
@Override
public int update(Statement statement) throws SQLException {
  String sql = boundSql.getSql();
  Object parameterObject = boundSql.getParameterObject();
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  int rows;
  if (keyGenerator instanceof Jdbc3KeyGenerator) {
    statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
    rows = statement.getUpdateCount();
    keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
  } else if (keyGenerator instanceof SelectKeyGenerator) {
    statement.execute(sql);
    rows = statement.getUpdateCount();
    keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
  } else {
    statement.execute(sql);
    rows = statement.getUpdateCount();
  }
  return rows;
}
```

org.apache.ibatis.executor.keygen.Jdbc3KeyGenerator#processAfter

```java
public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
  processBatch(ms, stmt, getParameters(parameter));
}

public void processBatch(MappedStatement ms, Statement stmt, Collection<Object> parameters) {
  ResultSet rs = null;
  try {
    rs = stmt.getGeneratedKeys();
    final Configuration configuration = ms.getConfiguration();
    final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    final String[] keyProperties = ms.getKeyProperties();
    final ResultSetMetaData rsmd = rs.getMetaData();
    TypeHandler<?>[] typeHandlers = null;
    if (keyProperties != null && rsmd.getColumnCount() >= keyProperties.length) {
      for (Object parameter : parameters) {
        // there should be one row for each statement (also one for each parameter)
        if (!rs.next()) {
          break;
        }
        final MetaObject metaParam = configuration.newMetaObject(parameter);
        if (typeHandlers == null) {
          typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
        }
        populateKeys(rs, metaParam, keyProperties, typeHandlers);
      }
    }
  } catch (Exception e) {
    throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
  } finally {
    if (rs != null) {
      try {
        rs.close();
      } catch (Exception e) {
        // ignore
      }
    }
  }
}
```

# 如何使用插件？

```xml
<!--配置自定义的插件-->
<plugins>
    <plugin interceptor="com.lct.plugin.ExamplePlugin">
        <property name="test" value="test"/>
    </plugin>
</plugins>
```

```java
@Intercepts({@Signature(type= StatementHandler.class,method = "prepare",args = {Connection.class,Integer.class})})
public class ExamplePlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("对方法进行了增强....");
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        System.out.println("将要包装的目标对象:"+target);
        return Plugin.wrap(target,this);
    }

    @Override
    public void setProperties(Properties properties) {
        System.out.println("插件配置的初始化参数:"+properties );
    }
}
```

```
StatementHandler/Executor/ParameterHandler/ResultSetHandler
每个创建出来的对象不是直接返回的，而是interceptorChain.pluginAll(parameterHandler)
用到了代理模式和责任链模式
```

org.apache.ibatis.session.Configuration

```java
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
  ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
  parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
  return parameterHandler;
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
    ResultHandler resultHandler, BoundSql boundSql) {
  ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
  resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
  return resultSetHandler;
}

public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}

public Executor newExecutor(Transaction transaction) {
  return newExecutor(transaction, defaultExecutorType);
}

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

org.apache.ibatis.plugin.InterceptorChain#pluginAll

```java
public Object pluginAll(Object target) {
  for (Interceptor interceptor : interceptors) {
    target = interceptor.plugin(target);
  }
  return target;
}
```

org.apache.ibatis.plugin.Plugin#wrap

```java
public static Object wrap(Object target, Interceptor interceptor) {//包装目标类
  //解析注解Intercepts
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor); 
  Class<?> type = target.getClass();
  //获取代理接口
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  if (interfaces.length > 0) { //创建代理对象
    return Proxy.newProxyInstance(
        type.getClassLoader(),
        interfaces,
        new Plugin(target, interceptor, signatureMap));
  }
  return target;
}
```

org.apache.ibatis.plugin.Plugin#getSignatureMap 

```java
private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
  Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
  // issue #251
  if (interceptsAnnotation == null) { //无注解，抛出异常
    throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
  }
  Signature[] sigs = interceptsAnnotation.value();
  //为接口的多个方法进行代理
  Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
  for (Signature sig : sigs) {
    Set<Method> methods = signatureMap.get(sig.type());
    if (methods == null) {
      methods = new HashSet<Method>();
      signatureMap.put(sig.type(), methods);
    }
    try {
      //sig.type() 获取代理接口
      //sig.method() 代理的方法
      //sig.args() 代理的参数
      Method method = sig.type().getMethod(sig.method(), sig.args());
      methods.add(method);
    } catch (NoSuchMethodException e) {
      throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
    }
  }
  return signatureMap;
}
```

org.apache.ibatis.plugin.Plugin#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    Set<Method> methods = signatureMap.get(method.getDeclaringClass()); //代理的方法
    if (methods != null && methods.contains(method)) { //执行的方法需要代理
      return interceptor.intercept(new Invocation(target, method, args));
    }
    return method.invoke(target, args); //直接执行目标方法
  } catch (Exception e) {
    throw ExceptionUtil.unwrapThrowable(e);
  }
}
```

# 如何对事务管理？

```xml
<!--当前的事务事务管理器是JDBC，如果不配置默认是ManagedTransactionFactory-->
<transactionManager type="JDBC"></transactionManager>
```

```
ManagedTransactionFactory、JdbcTransactionFactory都实现TransactionFactory接口
工厂模式
```

org.apache.ibatis.transaction.jdbc.JdbcTransactionFactory#newTransaction(javax.sql.DataSource, org.apache.ibatis.session.TransactionIsolationLevel, boolean)

```java
public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
  	//autoCommit=false，默认情况下，事务不会自动提交
    return new JdbcTransaction(ds, level, autoCommit);
}
```

```java
public Connection getConnection() throws SQLException {
    if (this.connection == null) {
        this.openConnection();
    }

    return this.connection;
}

public void commit() throws SQLException {
    if (this.connection != null && !this.connection.getAutoCommit()) {
        if (log.isDebugEnabled()) {
            log.debug("Committing JDBC Connection [" + this.connection + "]");
        }

        this.connection.commit();
    }

}
```

```java
public void rollback() throws SQLException {
    if (this.connection != null && !this.connection.getAutoCommit()) {
        if (log.isDebugEnabled()) {
            log.debug("Rolling back JDBC Connection [" + this.connection + "]");
        }

        this.connection.rollback();
    }

}

public void close() throws SQLException {
    if (this.connection != null) {
        this.resetAutoCommit();
        if (log.isDebugEnabled()) {
            log.debug("Closing JDBC Connection [" + this.connection + "]");
        }

        this.connection.close();
    }

}
```

```java
public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
  	//commit()、rollback() 方法都是空实现，事务的提交和回滚都是依靠容器管理的
    return new ManagedTransaction(ds, level, this.closeConnection);
}
```

```java
public Connection getConnection() throws SQLException {
    if (this.connection == null) {
        this.openConnection();
    }

    return this.connection;
}

public void commit() throws SQLException {
}

public void rollback() throws SQLException {
}

public void close() throws SQLException {
    if (this.closeConnection && this.connection != null) {
        if (log.isDebugEnabled()) {
            log.debug("Closing JDBC Connection [" + this.connection + "]");
        }

        this.connection.close();
    }

}

protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
        log.debug("Opening JDBC Connection");
    }

    this.connection = this.dataSource.getConnection();
    if (this.level != null) {
        this.connection.setTransactionIsolation(this.level.getLevel());
    }

}
```

# 如何整合第三方日志

/Users/chengtaoli/.m2/repository/org/mybatis/mybatis/3.5.1/mybatis3.5.1.jar!/org/apache/ibatis/logging/LogFactory.class:92

```java
//加载LogFactory时，执行static代码块
static {
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
}
```

```java
public static synchronized void useLog4J2Logging() {
    setImplementation(Log4j2Impl.class);
}
```

```java
private static void tryImplementation(Runnable runnable) {
    if (logConstructor == null) {//构造器为空，运行runnable的run方法
        try {
            runnable.run();
        } catch (Throwable var2) {
        }
    }
}
```

```java
private static void setImplementation(Class<? extends Log> implClass) {
    try {
        Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
        Log log = (Log)candidate.newInstance(LogFactory.class.getName());
        if (log.isDebugEnabled()) {
            log.debug("Logging initialized using '" + implClass + "' adapter.");
        }
     	 	//通过构造器创建对象成功
        logConstructor = candidate;
    } catch (Throwable var3) {
        throw new LogException("Error setting Log implementation.  Cause: " + var3, var3);
    }
}
```

# 设计模式

Builder模式：SqlSessionFactoryBuild

⼯⼚⽅法模式:SqlSessionFactory、TransactionFactory、LogFactory

单例模式:ErrorContext、LogFactory

代理模式：MapperProxy、插件

迭代器模式：PropertyTokenizer

装饰者模式:Cache包中的cache.decorators⼦包中等各个装饰者的实现

适配器模式:logging包下的提供Log接口，通过接口实现适配

模板⽅法模式：例如 BaseExecutor 和 SimpleExecutor，还有 BaseTypeHandler 和所有的⼦类例如IntegerTypeHandler

组合模式：例如SqlNode和各个⼦类ChooseSqlNode等

# 如何集成Spring？

```java
//该注解类似于spring配置文件
@Configuration
//扫描mapper
@MapperScan(basePackages="com.lct.study.dao")
public class MyBatisConfig {
  
   @Autowired
   private Environment env;

   //1、根据配置文件，注入数据源
   @Bean(name="dataSource")
   @ConfigurationProperties(prefix="datasource")
   public DataSource getDataSource() throws Exception{
        return new DruidDataSource();
   }
   
     //2、根据数据源创建SqlSessionFactory
   @Bean(name="sqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception{
       //其实就是封装了mybatis初始化的信息
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        //指定数据源(这个必须有，否则报错)
        sqlSessionFactoryBean.setDataSource(dataSource);
        //下边两句仅仅用于*.xml文件，如果整个持久层操作不需要使用到xml文件的话（只用注解就可以搞定），则不加
        String basePackage = env.getProperty("mybatis.typeAliasesPackage");
        String xmlPackage = env.getProperty("mybatis.mapperLocations");

        //在Mapper配置文件中在parameterType的值就不用写成全路径名了
        sqlSessionFactoryBean.setTypeAliasesPackage(basePackage);
        //指定xml文件位置
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(xmlPackage));
        return sqlSessionFactoryBean.getObject();
    }

    //3、根据SqlSessionFactory创建SqlSessionTemplate
   @Bean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    //4、创建事务管理器
    @Bean
    public PlatformTransactionManager annotationDrivenTransactionManager() throws Exception{
        return new DataSourceTransactionManager(getDataSource());
    }
    
}
```

## MapperScannerRegistrar

扫描指定包下的mapper接口，封装BeanDefinition，修改beanclass，注册到BeanDefinitionRegistry，每个mapper接口的实例最终通过调用MapperFactoryBean的getObject方法获得。MapperFactoryBean实现接口FactoryBean，还需注入SqlSessionFactory

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({MapperScannerRegistrar.class})
public @interface MapperScan {
}
```

org.mybatis.spring.annotation.MapperScannerRegistrar#registerBeanDefinitions、

```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    if (this.resourceLoader != null) {
        scanner.setResourceLoader(this.resourceLoader);
    }

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
        scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
        scanner.setMarkerInterface(markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
        scanner.setBeanNameGenerator((BeanNameGenerator)BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
        scanner.setMapperFactoryBean((MapperFactoryBean)BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }

    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));
    List<String> basePackages = new ArrayList();
    String[] var10 = annoAttrs.getStringArray("value");
    int var11 = var10.length;

    int var12;
    String pkg;
    for(var12 = 0; var12 < var11; ++var12) {
        pkg = var10[var12];
        if (StringUtils.hasText(pkg)) {
            basePackages.add(pkg);
        }
    }

  	//获取mapper接口所在的包
    var10 = annoAttrs.getStringArray("basePackages");
    var11 = var10.length;

    for(var12 = 0; var12 < var11; ++var12) {
        pkg = var10[var12];
        if (StringUtils.hasText(pkg)) {
            basePackages.add(pkg);
        }
    }

    Class[] var14 = annoAttrs.getClassArray("basePackageClasses");
    var11 = var14.length;

    for(var12 = 0; var12 < var11; ++var12) {
        Class<?> clazz = var14[var12];
        basePackages.add(ClassUtils.getPackageName(clazz));
    }

    scanner.registerFilters();
 	 //扫描mapper接口，注册GenericBeanDefinition到BeanDefinitionRegistry
    scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
  	//扫描mapper接口
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
    if (beanDefinitions.isEmpty()) {
        this.logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      	//修改mapper接口的beanDefinition，修改beanclass为mapperFactoryBean
        this.processBeanDefinitions(beanDefinitions);
    }
    return beanDefinitions;
}
```

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    Iterator var3 = beanDefinitions.iterator();

    while(var3.hasNext()) {
        BeanDefinitionHolder holder = (BeanDefinitionHolder)var3.next();
        GenericBeanDefinition definition = (GenericBeanDefinition)holder.getBeanDefinition();
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + definition.getBeanClassName() + "' mapperInterface");
        }

//构造器参数，即public MapperFactoryBean(Class<T> mapperInterface)
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName());
      //修改beanclass为MapperFactoryBean，通过getObject获取具体的bean
        definition.setBeanClass(this.mapperFactoryBean.getClass());
       //根据注解中的信息设置属性
        definition.getPropertyValues().add("addToConfig", this.addToConfig);
        boolean explicitFactoryUsed = false;
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
            definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
            definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
            explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
            if (explicitFactoryUsed) {
                this.logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
            }

            definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
            if (explicitFactoryUsed) {
                this.logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
            }

            definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
            explicitFactoryUsed = true;
        }

        if (!explicitFactoryUsed) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
            }

            definition.setAutowireMode(2);
        }
    }

}
```

org.mybatis.spring.mapper.MapperFactoryBean#getObject

```java
public T getObject() throws Exception {
  	//调用SqlSessionTemplate的getMapper方法
    return this.getSqlSession().getMapper(this.mapperInterface);
}
```

## SqlSessionFactoryBean

负责mybatis的一些初始化操作，解析配置文件、mapper文件，创建SqlSessionFactory

org.mybatis.spring.SqlSessionFactoryBean#afterPropertiesSet

```java
public void afterPropertiesSet() throws Exception {
  notNull(dataSource, "Property 'dataSource' is required");
  notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
  state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
            "Property 'configuration' and 'configLocation' can not specified with together");

  this.sqlSessionFactory = buildSqlSessionFactory();
}
```

```java
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

  Configuration configuration;

  XMLConfigBuilder xmlConfigBuilder = null;
  if (this.configuration != null) {
    configuration = this.configuration;
    if (configuration.getVariables() == null) {
      configuration.setVariables(this.configurationProperties);
    } else if (this.configurationProperties != null) {
      configuration.getVariables().putAll(this.configurationProperties);
    }
  } else if (this.configLocation != null) {
    xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
    configuration = xmlConfigBuilder.getConfiguration();
  } else {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Property `configuration` or 'configLocation' not specified, using default MyBatis Configuration");
    }
    configuration = new Configuration();
    configuration.setVariables(this.configurationProperties);
  }

  if (this.objectFactory != null) {
    configuration.setObjectFactory(this.objectFactory);
  }

  if (this.objectWrapperFactory != null) {
    configuration.setObjectWrapperFactory(this.objectWrapperFactory);
  }

  if (this.vfs != null) {
    configuration.setVfsImpl(this.vfs);
  }

  if (hasLength(this.typeAliasesPackage)) {
    String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
        ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
    for (String packageToScan : typeAliasPackageArray) {
      configuration.getTypeAliasRegistry().registerAliases(packageToScan,
              typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Scanned package: '" + packageToScan + "' for aliases");
      }
    }
  }

  if (!isEmpty(this.typeAliases)) {
    for (Class<?> typeAlias : this.typeAliases) {
      configuration.getTypeAliasRegistry().registerAlias(typeAlias);
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Registered type alias: '" + typeAlias + "'");
      }
    }
  }

  if (!isEmpty(this.plugins)) {
    for (Interceptor plugin : this.plugins) {
      configuration.addInterceptor(plugin);
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Registered plugin: '" + plugin + "'");
      }
    }
  }

  if (hasLength(this.typeHandlersPackage)) {
    String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
        ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
    for (String packageToScan : typeHandlersPackageArray) {
      configuration.getTypeHandlerRegistry().register(packageToScan);
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Scanned package: '" + packageToScan + "' for type handlers");
      }
    }
  }

  if (!isEmpty(this.typeHandlers)) {
    for (TypeHandler<?> typeHandler : this.typeHandlers) {
      configuration.getTypeHandlerRegistry().register(typeHandler);
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Registered type handler: '" + typeHandler + "'");
      }
    }
  }

  if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
    try {
      configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
    } catch (SQLException e) {
      throw new NestedIOException("Failed getting a databaseId", e);
    }
  }

  if (this.cache != null) {
    configuration.addCache(this.cache);
  }

  if (xmlConfigBuilder != null) {
    try {
      xmlConfigBuilder.parse();

      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Parsed configuration file: '" + this.configLocation + "'");
      }
    } catch (Exception ex) {
      throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  if (this.transactionFactory == null) {
    this.transactionFactory = new SpringManagedTransactionFactory();
  }

  configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

  if (!isEmpty(this.mapperLocations)) {
    for (Resource mapperLocation : this.mapperLocations) {
      if (mapperLocation == null) {
        continue;
      }

      try {
        XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
            configuration, mapperLocation.toString(), configuration.getSqlFragments());
        xmlMapperBuilder.parse();
      } catch (Exception e) {
        throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
      } finally {
        ErrorContext.instance().reset();
      }

      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
      }
    }
  } else {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
    }
  }

  return this.sqlSessionFactoryBuilder.build(configuration);
}
```

## SqlSessionTemplate

org.mybatis.spring.SqlSessionTemplate#SqlSessionTemplate(org.apache.ibatis.session.SqlSessionFactory, org.apache.ibatis.session.ExecutorType, org.springframework.dao.support.PersistenceExceptionTranslator)

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
    PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
  notNull(executorType, "Property 'executorType' is required");

  this.sqlSessionFactory = sqlSessionFactory;
  this.executorType = executorType;
  this.exceptionTranslator = exceptionTranslator;
  //创建SqlSession的代理类
  this.sqlSessionProxy = (SqlSession) newProxyInstance(
      SqlSessionFactory.class.getClassLoader(),
      new Class[] { SqlSession.class },
      new SqlSessionInterceptor());
}
```

```java
public <T> T getMapper(Class<T> type) { 
  //获取mapper接口的bean实例
  //MapperFactoryBean#getObject会调用此方法
  return getConfiguration().getMapper(type, this);
}
```

org.mybatis.spring.SqlSessionTemplate.SqlSessionInterceptor#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  //创建原生的SqlSession
  SqlSession sqlSession = getSqlSession(
      SqlSessionTemplate.this.sqlSessionFactory,
      SqlSessionTemplate.this.executorType,
      SqlSessionTemplate.this.exceptionTranslator);
  try {
    //方法执行
    Object result = method.invoke(sqlSession, args);
    //当事务不由Spring事务管理器管理的时候，会立即提交事务
    if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) { 
      sqlSession.commit(true);
    }
    return result;
  } catch (Throwable t) {
    Throwable unwrapped = unwrapThrowable(t);
    if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
      closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
      sqlSession = null;
      Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
      if (translated != null) {
        unwrapped = translated;
      }
    }
    throw unwrapped;
  } finally {
    if (sqlSession != null) {
      closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
    }
  }
}
```

org.mybatis.spring.SqlSessionUtils#getSqlSession(org.apache.ibatis.session.SqlSessionFactory, org.apache.ibatis.session.ExecutorType, org.springframework.dao.support.PersistenceExceptionTranslator)

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
  notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

  //没有整合spring事务管理器时，为空
  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

  SqlSession session = sessionHolder(executorType, holder);
  if (session != null) {
    return session;
  }

  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug("Creating a new SqlSession");
  }

  session = sessionFactory.openSession(executorType);

 	//没有整合spring事务管理器时，不会注册SessionHolder
  registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

  return session;
}
```

# MyBatis-Plus

一个 MyBatis的增强工具，在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生

设计模式中的装饰者模式可以实现功能的增强

```java
 InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
//使用MyBatis-Plus中的MybatisSqlSessionFactoryBuilder加载配置文件
 SqlSessionFactory sqlSessionFactory = new MybatisSqlSessionFactoryBuilder()
   																											.build(inputStream);
```

com.baomidou.mybatisplus.core.MybatisSqlSessionFactoryBuilder#build(java.io.InputStream, java.lang.String, java.util.Properties)

```java
//继承SqlSessionFactoryBuilder，重写build
public class MybatisSqlSessionFactoryBuilder extends SqlSessionFactoryBuilder {
  
  @Override
  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
      try {
          //TODO 这里换成 MybatisXMLConfigBuilder 而不是 XMLConfigBuilder
          MybatisXMLConfigBuilder parser = new MybatisXMLConfigBuilder(inputStream, environment, properties);
          return build(parser.parse());
      } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error building SqlSession.", e);
      } finally {
          ErrorContext.instance().reset();
          try {
              inputStream.close();
          } catch (IOException e) {
              // Intentionally ignore. Prefer previous error.
          }
      }
  }
    @Override
    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        try {
            //TODO 这里换成 MybatisXMLConfigBuilder 而不是 XMLConfigBuilder
            MybatisXMLConfigBuilder parser = new MybatisXMLConfigBuilder(inputStream, environment, properties);
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                inputStream.close();
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }
  
    // TODO 使用自己的逻辑,注入必须组件
    @Override
    public SqlSessionFactory build(Configuration config) {
        MybatisConfiguration configuration = (MybatisConfiguration) config;
        GlobalConfig globalConfig = configuration.getGlobalConfig();
        // 初始化 Sequence
        if (null != globalConfig.getWorkerId()
            && null != globalConfig.getDatacenterId()) {
            IdWorker.initSequence(globalConfig.getWorkerId(), globalConfig.getDatacenterId());
        }
        if (globalConfig.isEnableSqlRunner()) {
            new SqlRunnerInjector().inject(configuration);
        }

        SqlSessionFactory sqlSessionFactory = super.build(configuration);

        // 设置全局参数属性 以及 缓存 sqlSessionFactory
        globalConfig.signGlobalConfig(sqlSessionFactory);

        return sqlSessionFactory;
    }
}
```

```
MybatisSqlSessionFactoryBuilder、MybatisXMLConfigBuilder、MybatisConfiguration各自继承mybatis中的SqlSessionFactoryBuilder、BaseBuilder、Configuration，重写其中的方法
```

com.baomidou.mybatisplus.core.MybatisXMLConfigBuilder#MybatisXMLConfigBuilder(org.apache.ibatis.parsing.XPathParser, java.lang.String, java.util.Properties)

```java
private MybatisXMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    // TODO 使用 MybatisConfiguration 而不是 Configuration
    super(new MybatisConfiguration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
}
```

com.baomidou.mybatisplus.core.MybatisXMLConfigBuilder#mapperElement

```java
private void mapperElement(XNode parent) throws Exception {
    /*
     * 定义集合 用来分类放置mybatis的Mapper与XML 按顺序依次遍历
     */
    if (parent != null) {
        //指定在classpath中的mapper文件
        Set<String> resources = new HashSet<>();
        //指向一个mapper接口
        Set<Class<?>> mapperClasses = new HashSet<>();
        setResource(parent, resources, mapperClasses);
        // 依次遍历 首先 resource 然后 mapper
        for (String resource : resources) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            //TODO
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,
                configuration.getSqlFragments());
            mapperParser.parse();
        }
        for (Class<?> mapper : mapperClasses) {
            //TODO
            configuration.addMapper(mapper);
        }
    }
}
```

org.apache.ibatis.builder.xml.XMLMapperBuilder#parse

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    bindMapperForNamespace();
  }

  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
```

```java
private void bindMapperForNamespace() {
  String namespace = builderAssistant.getCurrentNamespace();
  if (namespace != null) {
    Class<?> boundType = null;
    try {
      boundType = Resources.classForName(namespace);
    } catch (ClassNotFoundException e) {
      //ignore, bound type is not required
    }
    if (boundType != null) {
      if (!configuration.hasMapper(boundType)) {
        // Spring may not know the real resource name so we set a flag
        // to prevent loading again this resource from the mapper interface
        // look at MapperAnnotationBuilder#loadXmlResource
        configuration.addLoadedResource("namespace:" + namespace);
        configuration.addMapper(boundType); //添加到configuration
      }
    }
  }
}
```

com.baomidou.mybatisplus.core.MybatisConfiguration#addMapper

```java
@Override
public <T> void addMapper(Class<T> type) {
    mybatisMapperRegistry.addMapper(type);
}
```

com.baomidou.mybatisplus.core.MybatisMapperRegistry#addMapper

```java
    @Override
    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (hasMapper(type)) {
                // TODO 如果之前注入 直接返回
                return;
                // TODO 这里就不抛异常了
//                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                // TODO 这里也换成 MybatisMapperProxyFactory 而不是 MapperProxyFactory
                knownMappers.put(type, new MybatisMapperProxyFactory<>(type));
                // It's important that the type is added before the parser is run
                // otherwise the binding may automatically be attempted by the
                // mapper parser. If the type is already known, it won't try.
                // TODO 这里也换成 MybatisMapperAnnotationBuilder 而不是 MapperAnnotationBuilder
                MybatisMapperAnnotationBuilder parser = new MybatisMapperAnnotationBuilder(config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
```

```java
@Override
public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
        loadXmlResource();
        configuration.addLoadedResource(resource);
        final String typeName = type.getName();
        assistant.setCurrentNamespace(typeName);
        parseCache();
        parseCacheRef();
        SqlParserHelper.initSqlParserInfoCache(type);
        Method[] methods = type.getMethods();
        for (Method method : methods) {
            try {
                // issue #237
                if (!method.isBridge()) {
                    parseStatement(method);
                    SqlParserHelper.initSqlParserInfoCache(typeName, method);
                }
            } catch (IncompleteElementException e) {
                // TODO 使用 MybatisMethodResolver 而不是 MethodResolver
                configuration.addIncompleteMethod(new MybatisMethodResolver(this, method));
            }
        }
      //判断是否继承BaseMapper
        // TODO 注入 CURD 动态 SQL , 放在在最后, because 可能会有人会用注解重写sql
        if (GlobalConfigUtils.isSupperMapperChildren(configuration, type)) {
            GlobalConfigUtils.getSqlInjector(configuration).inspectInject(assistant, type);
        }
    }
    parsePendingMethods();
}
```

com.baomidou.mybatisplus.core.injector.AbstractSqlInjector#inspectInject

```java
@Override
public void inspectInject(MapperBuilderAssistant builderAssistant, Class<?> mapperClass) {
    Class<?> modelClass = extractModelClass(mapperClass);
    if (modelClass != null) {
        String className = mapperClass.toString();
        Set<String> mapperRegistryCache = GlobalConfigUtils.getMapperRegistryCache(builderAssistant.getConfiguration());
        if (!mapperRegistryCache.contains(className)) {
            //17个方法
            List<AbstractMethod> methodList = this.getMethodList();
            if (CollectionUtils.isNotEmpty(methodList)) {
                TableInfo tableInfo = TableInfoHelper.initTableInfo(builderAssistant, modelClass);
                // 循环注入自定义方法
                methodList.forEach(m -> m.inject(builderAssistant, mapperClass, modelClass, tableInfo));
            } else {
                logger.debug(mapperClass.toString() + ", No effective injection method was found.");
            }
            mapperRegistryCache.add(className);
        }
    }
}
```

com.baomidou.mybatisplus.core.injector.DefaultSqlInjector#getMethodList

```java
@Override
public List<AbstractMethod> getMethodList() {
    return Stream.of(
        new Insert(),
        new Delete(),
        new DeleteByMap(),
        new DeleteById(),
        new DeleteBatchByIds(),
        new Update(),
        new UpdateById(),
        new SelectById(),
        new SelectBatchByIds(),
        new SelectByMap(),
        new SelectOne(),
        new SelectCount(),
        new SelectMaps(),
        new SelectMapsPage(),
        new SelectObjs(),
        new SelectList(),
        new SelectPage()
    ).collect(toList());
}
```

com.baomidou.mybatisplus.core.MybatisConfiguration#getMapper

```java
@Override
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mybatisMapperRegistry.getMapper(type, sqlSession);
}
```

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // TODO 这里换成 MybatisMapperProxyFactory 而不是 MapperProxyFactory
    final MybatisMapperProxyFactory<T> mapperProxyFactory = (MybatisMapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MybatisPlusMapperRegistry.");
    }
    try {
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```

## 如何实现自定义生成主键？

继承自DefaultParameterHandler，重写setParameters方法

com.baomidou.mybatisplus.core.MybatisDefaultParameterHandler#processParameter

```java
public MybatisDefaultParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    super(mappedStatement, processParameter(mappedStatement, parameterObject),//设置参数
          boundSql);
    this.mappedStatement = mappedStatement;
    this.configuration = mappedStatement.getConfiguration();
    this.typeHandlerRegistry = mappedStatement.getConfiguration().getTypeHandlerRegistry();
    this.parameterObject = parameterObject;
    this.boundSql = boundSql;
}
```

```java
protected static Object processParameter(MappedStatement ms, Object parameterObject) {
    /* 只处理插入或更新操作 */
    if (parameterObject != null
        && (SqlCommandType.INSERT == ms.getSqlCommandType() || SqlCommandType.UPDATE == ms.getSqlCommandType())) {
        //检查 parameterObject
        if (ReflectionKit.isPrimitiveOrWrapper(parameterObject.getClass())
            || parameterObject.getClass() == String.class) {
            return parameterObject;
        }
        Collection<Object> parameters = getParameters(parameterObject);
        if (null != parameters) {
            // 感觉这里可以稍微优化一下，理论上都是同一个.
            parameters.stream().filter(Objects::nonNull).forEach(obj -> process(ms, obj));
        } else {
            process(ms, parameterObject);
        }
    }
    return parameterObject;
}
```

```java
private static void process(MappedStatement ms, Object parameterObject) {
    TableInfo tableInfo = null;
    Object entity = parameterObject;
    if (parameterObject instanceof Map) {
        Map<?, ?> map = (Map<?, ?>) parameterObject;
        if (map.containsKey(Constants.ENTITY)) {
            Object et = map.get(Constants.ENTITY);
            if (et != null) {
                if (et instanceof Map) {
                    Map<?, ?> realEtMap = (Map<?, ?>) et;
                    if (realEtMap.containsKey(Constants.MP_OPTLOCK_ET_ORIGINAL)) {
                        entity = realEtMap.get(Constants.MP_OPTLOCK_ET_ORIGINAL);
                        tableInfo = TableInfoHelper.getTableInfo(entity.getClass());
                    }
                } else {
                    entity = et;
                    tableInfo = TableInfoHelper.getTableInfo(entity.getClass());
                }
            }
        }
    } else {
        tableInfo = TableInfoHelper.getTableInfo(parameterObject.getClass());
    }
    if (tableInfo != null) {
       //到这里就应该转换到实体参数对象了,因为填充和ID处理都是争对实体对象处理的,不用传递原参数对象下去.
        MetaObject metaObject = ms.getConfiguration().newMetaObject(entity);
        if (SqlCommandType.INSERT == ms.getSqlCommandType()) { 
          	//插入
           //1、生成自定义主键 2、自动填充属性值
            populateKeys(tableInfo, metaObject, entity);
            insertFill(metaObject, tableInfo);
        } else {
            updateFill(metaObject, tableInfo);
        }
    }
}
```

填充主键

```java
protected static void populateKeys(TableInfo tableInfo, MetaObject metaObject, Object entity) {
    final IdType idType = tableInfo.getIdType();
    final String keyProperty = tableInfo.getKeyProperty();
    if (StringUtils.isNotBlank(keyProperty) && null != idType && idType.getKey() >= 3) {
        final IdentifierGenerator identifierGenerator = GlobalConfigUtils.getGlobalConfig(tableInfo.getConfiguration()).getIdentifierGenerator();
        Object idValue = metaObject.getValue(keyProperty);
        if (StringUtils.checkValNull(idValue)) {
            if (idType.getKey() == IdType.ASSIGN_ID.getKey()) { //主键生成策略
                if (Number.class.isAssignableFrom(tableInfo.getKeyType())) { //数字
                    metaObject.setValue(keyProperty, identifierGenerator.nextId(entity));
                } else {//字符串
                    metaObject.setValue(keyProperty, identifierGenerator.nextId(entity).toString());
                }
            } else if (idType.getKey() == IdType.ASSIGN_UUID.getKey()) {
                metaObject.setValue(keyProperty, identifierGenerator.nextUUID(entity));
            }
        }
    }
}
```

公共字段自动写入

```java
protected static void insertFill(MetaObject metaObject, TableInfo tableInfo) {
    GlobalConfigUtils.getMetaObjectHandler(tableInfo.getConfiguration()).ifPresent(metaObjectHandler -> {
        if (metaObjectHandler.openInsertFill()) {
            if (tableInfo.isWithInsertFill()) {
              	//调用自定义的MetaObjectHandler，实现公共字段自动写入
                metaObjectHandler.insertFill(metaObject);
            } else {
                // 兼容旧操作 id类型为input或none的要用填充器处理一下
                if (metaObjectHandler.compatibleFillId()) {
                    String keyProperty = tableInfo.getKeyProperty();
                    if (StringUtils.isNotBlank(keyProperty)) {
                        Object value = metaObject.getValue(keyProperty);
                        if (value == null && (IdType.NONE == tableInfo.getIdType() || IdType.INPUT == tableInfo.getIdType())) {
                            metaObjectHandler.insertFill(metaObject);
                        }
                    }
                }
            }
        }
    });
}
```
