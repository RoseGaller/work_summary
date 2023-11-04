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

# 插件

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

解析注解Intercepts

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

方法执行

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
