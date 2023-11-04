# Servicecomb-pack源码分析

* [概述](#概述)
* [配置初始化](#配置初始化)
* [事务拦截初始化](#事务拦截初始化)
* [Saga事务](#saga事务)
  * [sagaStart总事务](#sagastart总事务)
  * [Saga子事务](#saga子事务)
  * [维护补偿方法](#维护补偿方法)
  * [Saga与Alpha的通信](#saga与alpha的通信)
    * [GrpcSagaClientMessageSender](#grpcsagaclientmessagesender)
    * [GrpcCompensateStreamObserver](#grpccompensatestreamobserver)
    * [SagaLoadBalanceSender](#sagaloadbalancesender)
      * [FastestSender](#fastestsender)
    * [执行补偿方法](#执行补偿方法)
* [TCC事务](#tcc事务)
  * [TccStart总事务](#tccstart总事务)
  * [TCC子事务](#tcc子事务)
  * [维护补偿方法](#维护补偿方法-1)
  * [TCC与Alpha通信](#tcc与alpha通信)
  * [执行补偿方法](#执行补偿方法-1)
    * [](#)


# 概述

提供了中心化的分布式事务解决方案，实现了SAGA和TCC

# 配置初始化

org.apache.servicecomb.pack.omega.spring.OmegaSpringConfig

```java
//唯一事务Id生成器
@ConditionalOnMissingBean
@Bean(name = {"omegaUniqueIdGenerator"})
IdGenerator<String> idGenerator() {
  return new UniqueIdGenerator();
}
```

```java
//管理全局事务Id、本地事务Id
@Bean
OmegaContext omegaContext(@Qualifier("omegaUniqueIdGenerator") IdGenerator<String> idGenerator, SagaMessageSender messageSender) {
  ServerMeta serverMeta = messageSender.onGetServerMeta();
  boolean akkaEnabeld = Boolean.parseBoolean(serverMeta.getMetaMap().get(AlphaMetaKeys.AkkaEnabled.name()));
  return new OmegaContext(idGenerator, AlphaMetas.builder().akkaEnabled(akkaEnabeld).build());
}
```

```java
//管理Saga模式下的所有补偿方法
@Bean(name = {"compensationContext"})
CallbackContext compensationContext(OmegaContext omegaContext, SagaMessageSender sender) {
  return new CallbackContext(omegaContext, sender);
}
```

```java
//管理TCC模式下的所有Confirm、Cancel方法
@Bean(name = {"coordinateContext"})
CallbackContext coordinateContext(OmegaContext omegaContext, SagaMessageSender sender) {
  return new CallbackContext(omegaContext, sender);
}
```

```java
//服务配置信息
@Bean
ServiceConfig serviceConfig(@Value("${spring.application.name}") String serviceName, @Value("${omega.instance.instanceId:#{null}}") String instanceId) {
  return new ServiceConfig(serviceName,instanceId);
}
```

```java
//Alpha集群发现
@Bean
@ConditionalOnProperty(name = "alpha.cluster.register.type", havingValue = "default", matchIfMissing = true)
AlphaClusterDiscovery alphaClusterAddress(@Value("${alpha.cluster.address:0.0.0.0:8080}") String[] addresses){
  return AlphaClusterDiscovery.builder().addresses(addresses).build();
}
```

```java
//Alpha集群配置信息
@Bean
AlphaClusterConfig alphaClusterConfig(
    @Value("${alpha.cluster.ssl.enable:false}") boolean enableSSL,
    @Value("${alpha.cluster.ssl.mutualAuth:false}") boolean mutualAuth,
    @Value("${alpha.cluster.ssl.cert:client.crt}") String cert,
    @Value("${alpha.cluster.ssl.key:client.pem}") String key,
    @Value("${alpha.cluster.ssl.certChain:ca.crt}") String certChain,
    @Lazy AlphaClusterDiscovery alphaClusterDiscovery,
    @Lazy MessageHandler handler,
    @Lazy TccMessageHandler tccMessageHandler) {

  LOG.info("Discovery alpha cluster address {} from {}",alphaClusterDiscovery.getAddresses() == null ? "" : String.join(",",alphaClusterDiscovery.getAddresses()), alphaClusterDiscovery.getDiscoveryType().name());
  MessageFormat messageFormat = new KryoMessageFormat();
  AlphaClusterConfig clusterConfig = AlphaClusterConfig.builder()
      .addresses(ImmutableList.copyOf(alphaClusterDiscovery.getAddresses()))
      .enableSSL(enableSSL)
      .enableMutualAuth(mutualAuth)
      .cert(cert)
      .key(key)
      .certChain(certChain)
      .messageDeserializer(messageFormat)
      .messageSerializer(messageFormat)
      .messageHandler(handler) //Saga模式下，接收Alpha请求处理补偿方法
      .tccMessageHandler(tccMessageHandler) //TCC模式下，接收Alpha请求处理补偿方法
      .build();
  return clusterConfig;
}
```

```java
//Saga模式下，与各个Alpha节点建立连接，创建GrpcSagaClientMessageSender对象
@Bean(name = "sagaLoadContext")
LoadBalanceContext sagaLoadBalanceSenderContext(
    AlphaClusterConfig alphaClusterConfig,
    ServiceConfig serviceConfig,
    @Value("${omega.connection.reconnectDelay:3000}") int reconnectDelay,
    @Value("${omega.connection.sending.timeout:8}") int timeoutSeconds) {
  LoadBalanceContext loadBalanceSenderContext = new LoadBalanceContextBuilder(
      TransactionType.SAGA,
      alphaClusterConfig,
      serviceConfig,
      reconnectDelay,
      timeoutSeconds).build();
  return loadBalanceSenderContext;
}
```

```java
//挑选SagaMessageSender，与Alpha通信
@Bean
SagaMessageSender sagaLoadBalanceSender(@Qualifier("sagaLoadContext") LoadBalanceContext loadBalanceSenderContext) {
  final SagaMessageSender sagaMessageSender = new SagaLoadBalanceSender(loadBalanceSenderContext, new FastestSender());
  //与Alpha建立连接
  sagaMessageSender.onConnected();
  Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
    @Override
    public void run() {
      sagaMessageSender.onDisconnected();
      sagaMessageSender.close();
    }
  }));
  return sagaMessageSender;
}
```

```java
//创建TCC模式下，与各个Alpha建立GrpcTccClientMessageSender
@Bean(name = "tccLoadContext")
LoadBalanceContext loadBalanceSenderContext(
    AlphaClusterConfig alphaClusterConfig,
    ServiceConfig serviceConfig,
    @Value("${omega.connection.reconnectDelay:3000}") int reconnectDelay,
    @Value("${omega.connection.sending.timeout:8}") int timeoutSeconds) {
  LoadBalanceContext loadBalanceSenderContext = new LoadBalanceContextBuilder(
      TransactionType.TCC,
      alphaClusterConfig,
      serviceConfig,
      reconnectDelay,
      timeoutSeconds).build();
  return loadBalanceSenderContext;
}
```

```java
//TCC模式下，挑选Alpha节点，发送请求
@Bean
TccMessageSender tccLoadBalanceSender(@Qualifier("tccLoadContext") LoadBalanceContext loadBalanceSenderContext) {
  final TccMessageSender tccMessageSender = new TccLoadBalanceSender(loadBalanceSenderContext, new FastestSender());
  tccMessageSender.onConnected();
  Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
    @Override
    public void run() {
      tccMessageSender.onDisconnected();
      tccMessageSender.close();
    }
  }));
  return tccMessageSender;
}
```

# 事务拦截初始化

org.apache.servicecomb.pack.omega.transaction.spring.TransactionAspectConfig

```java
//Saga模式 - 接收Alpha的补偿请求，执行事务补偿
@Bean
MessageHandler messageHandler(SagaMessageSender sender,
    @Qualifier("compensationContext") CallbackContext context, OmegaContext omegaContext) {
  return new CompensationMessageHandler(sender, context);
}
```

```java
//拦截SagaStart注解的方法，Saga事务的入口
@Bean
SagaStartAspect sagaStartAspect(SagaMessageSender sender, OmegaContext context) {
  return new SagaStartAspect(sender, context);
}
```

```java
//拦截Compensable注解的方法，Saga子事务
@Bean
TransactionAspect transactionAspect(SagaMessageSender sender, OmegaContext context) {
  return new TransactionAspect(sender, context);
}
```

```java
//实现BeanPostProcessor的postProcessAfterInitialization方法，
//维护标有Compensable注解的方法
//维护标有OmegaContextAware注解的字段，针对Executor类型的字段，创建JDK动态代理
@Bean
CompensableAnnotationProcessor compensableAnnotationProcessor(OmegaContext omegaContext,
    @Qualifier("compensationContext") CallbackContext compensationContext) {
  return new CompensableAnnotationProcessor(omegaContext, compensationContext);
}
```



```java
//TCC模式 - 接收Alpha的补偿请求，执行事务补偿
@Bean
TccMessageHandler coordinateMessageHandler(
    TccMessageSender tccMessageSender,
    @Qualifier("coordinateContext") CallbackContext coordinateContext,
    OmegaContext omegaContext,
    ParametersContext parametersContext) {
  return new CoordinateMessageHandler(tccMessageSender, coordinateContext, omegaContext, parametersContext);
}
```

```java
//拦截TccStart注解的方法，TCC事务的入口
@Bean
TccStartAspect tccStartAspect(
    TccMessageSender tccMessageSender,
    OmegaContext context) {
  return new TccStartAspect(tccMessageSender, context);
}
```

```java
//拦截Participate注解的方法，TCC的子事务
@Bean
TccParticipatorAspect tccParticipatorAspect(
    TccMessageSender tccMessageSender,
    OmegaContext context, ParametersContext parametersContext) {
  return new TccParticipatorAspect(tccMessageSender, context, parametersContext);
}
```

```java
//维护标有Participate注解的方法
//维护标有OmegaContextAware注解的字段，针对Executor类型的字段，创建JDK动态代理
@Bean
ParticipateAnnotationProcessor participateAnnotationProcessor(OmegaContext omegaContext,
    @Qualifier("coordinateContext") CallbackContext coordinateContext) {
  return new ParticipateAnnotationProcessor(omegaContext, coordinateContext);
}
```



# Saga事务

## sagaStart总事务

标有sagaStart注解的方法就是Saga事务的入口,SagaStartAspect对sagaStart注解进行代理拦截

org.apache.servicecomb.pack.omega.transaction.SagaStartAspect#advise

```java
@Around("execution(@org.apache.servicecomb.pack.omega.context.annotations.SagaStart * *(..)) && @annotation(sagaStart)")
Object advise(ProceedingJoinPoint joinPoint, SagaStart sagaStart) throws Throwable {
  initializeOmegaContext();
  if(context.getAlphaMetas().isAkkaEnabled() && sagaStart.timeout()>0){
    SagaStartAnnotationProcessorTimeoutWrapper wrapper = new SagaStartAnnotationProcessorTimeoutWrapper(this.sagaStartAnnotationProcessor);
    return wrapper.apply(joinPoint,sagaStart,context);
  }else{
    SagaStartAnnotationProcessorWrapper wrapper = new SagaStartAnnotationProcessorWrapper(this.sagaStartAnnotationProcessor);
    return wrapper.apply(joinPoint,sagaStart,context);
  }
}
```

org.apache.servicecomb.pack.omega.transaction.wrapper.SagaStartAnnotationProcessorWrapper#apply

```java
public Object apply(ProceedingJoinPoint joinPoint, SagaStart sagaStart, OmegaContext context)
    throws Throwable {
  Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
  //向Alpha注册Saga事务
  sagaStartAnnotationProcessor.preIntercept(sagaStart.timeout());
  LOG.debug("Initialized context {} before execution of method {}", context, method.toString());
  try {
    //执行具体的业务逻辑
    Object result = joinPoint.proceed();
    if (sagaStart.autoClose()) {
      //向Alpha发送Saga总事务执行成功的请求
      sagaStartAnnotationProcessor.postIntercept(context.globalTxId());
      LOG.debug("Transaction with context {} has finished.", context);
    } else {
      LOG.debug("Transaction with context {} is not finished in the SagaStarted annotated method.", context);
    }
    return result;
  } catch (Throwable throwable) {
    // We don't need to handle the OmegaException here
    if (!(throwable instanceof OmegaException)) {
      //发送Saga总事务异常的请求
      sagaStartAnnotationProcessor.onError(method.toString(), throwable);
      LOG.error("Transaction {} failed.", context.globalTxId());
    }
    throw throwable;
  } finally {
    context.clear();
  }
}
```

在saga事务的执行之前、执行成功、执行出现异常向Alpha发送不同的Event

Saga事务执行之前发送SagaStartedEvent

org.apache.servicecomb.pack.omega.transaction.SagaStartAnnotationProcessor#preIntercept

```java
public AlphaResponse preIntercept(int timeout) {
  try {
    return sender
        .send(new SagaStartedEvent(omegaContext.globalTxId(), omegaContext.localTxId(), timeout));
  } catch (OmegaException e) {
    throw new TransactionalException(e.getMessage(), e.getCause());
  }
}
```

Saga事务执行成功SagaEndedEvent

org.apache.servicecomb.pack.omega.transaction.SagaStartAnnotationProcessor#postIntercept

```java
public void postIntercept(String parentTxId) {
  AlphaResponse response = sender
      .send(new SagaEndedEvent(omegaContext.globalTxId(), omegaContext.localTxId()));
  //TODO we may know if the transaction is aborted from fsm alpha backend
  if (response.aborted()) {
    throw new OmegaException("transaction " + parentTxId + " is aborted");
  }
}
```

Saga事务出现异常TxAbortedEvent

org.apache.servicecomb.pack.omega.transaction.SagaStartAnnotationProcessor#onError

```java
public void onError(String compensationMethod, Throwable throwable) {
  String globalTxId = omegaContext.globalTxId();
  if(omegaContext.getAlphaMetas().isAkkaEnabled()){
    sender.send(
        new SagaAbortedEvent(globalTxId, omegaContext.localTxId(), null, compensationMethod,
            throwable));
  }else{
    sender.send(
        new TxAbortedEvent(globalTxId, omegaContext.localTxId(), null, compensationMethod,
            throwable));
  }
}
```

## Saga子事务

标有Compensable注解的方法是Saga的子事务，TransactionAspect会对其代理

对子事务代理

org.apache.servicecomb.pack.omega.transaction.TransactionAspect#advise

```java
@Around("execution(@org.apache.servicecomb.pack.omega.transaction.annotations.Compensable * *(..)) && @annotation(compensable)")
Object advise(ProceedingJoinPoint joinPoint, Compensable compensable) throws Throwable {
  Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
  // just check if we need to setup the transaction context information first
  TransactionContext transactionContext = extractTransactionContext(joinPoint.getArgs());
  if (transactionContext != null) {
    populateOmegaContext(context, transactionContext);
  }
  // SCB-1011 Need to check if the globalTxId transaction is null to avoid the message sending failure
  if (context.globalTxId() == null) {
    throw new OmegaException("Cannot find the globalTxId from OmegaContext. Please using @SagaStart to start a global transaction.");
  }
  String localTxId = context.localTxId();
  context.newLocalTxId();
  LOG.debug("Updated context {} for compensable method {} ", context, method.toString());
//获取恢复策略，默认DefaultRecovery
  int forwardRetries = compensable.forwardRetries();
  RecoveryPolicy recoveryPolicy = RecoveryPolicyFactory.getRecoveryPolicy(forwardRetries);
  try {
    return recoveryPolicy.apply(joinPoint, compensable, interceptor, context, localTxId, forwardRetries);
  } finally {
    context.setLocalTxId(localTxId);
    LOG.debug("Restored context back to {}", context);
  }
}
```

org.apache.servicecomb.pack.omega.transaction.AbstractRecoveryPolicy#apply

```java
public Object apply(ProceedingJoinPoint joinPoint, Compensable compensable,
    CompensableInterceptor interceptor, OmegaContext context, String parentTxId, int forwardRetries)
    throws Throwable {
  Object result;
  if(compensable.forwardTimeout()>0){
    RecoveryPolicyTimeoutWrapper wrapper = new RecoveryPolicyTimeoutWrapper(this);
    result = wrapper.applyTo(joinPoint, compensable, interceptor, context, parentTxId, forwardRetries);
  } else {
    result = this.applyTo(joinPoint, compensable, interceptor, context, parentTxId, forwardRetries);
  }
  return result;
}
```

向Alpha发送TxStartedEvent、TxEndedEvent、TxAbortedEvent事件

org.apache.servicecomb.pack.omega.transaction.DefaultRecovery#applyTo

```java
public Object applyTo(ProceedingJoinPoint joinPoint, Compensable compensable, CompensableInterceptor interceptor,
    OmegaContext context, String parentTxId, int forwardRetries) throws Throwable {
  Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
  LOG.debug("Intercepting compensable method {} with context {}", method.toString(), context);
  //补偿方法
  String compensationSignature =
      compensable.compensationMethod().isEmpty() ? "" : compensationMethodSignature(joinPoint, compensable, method);

  String retrySignature = (forwardRetries != 0 || compensationSignature.isEmpty()) ? method.toString() : "";
  //注册子事务
  AlphaResponse response = interceptor.preIntercept(parentTxId, compensationSignature, compensable.forwardTimeout(),
      retrySignature, forwardRetries, joinPoint.getArgs());
  //总事务已经被丢弃
  if (response.aborted()) {
    String abortedLocalTxId = context.localTxId();
    context.setLocalTxId(parentTxId);
    throw new InvalidTransactionException("Abort sub transaction " + abortedLocalTxId +
        " because global transaction " + context.globalTxId() + " has already aborted.");
  }
  try {
    //执行业务逻辑
    Object result = joinPoint.proceed();
    //saga子事务执行成功
    interceptor.postIntercept(parentTxId, compensationSignature);
    return result;
  } catch (Throwable throwable) {
    if (compensable.forwardRetries() == 0 || (compensable.forwardRetries() > 0
        && forwardRetries == 1)) {
      //子事务执行异常
      interceptor.onError(parentTxId, compensationSignature, throwable);
    }
    throw throwable;
  }
```

## 维护补偿方法

在Bean实例化之后，查找标有Compensable注解的方法，获取补偿方法，建立业务方法与补偿方法的映射关系保存到CallbackContext

org.apache.servicecomb.pack.omega.transaction.spring.CompensableAnnotationProcessor#checkMethod

```java
private void checkMethod(Object bean) {
  ReflectionUtils.doWithMethods(
      bean.getClass(),
      new CompensableMethodCheckingCallback(bean, compensationContext));
}
```

org.apache.servicecomb.pack.omega.transaction.spring.CompensableMethodCheckingCallback#doWith

```java
public void doWith(Method method) throws IllegalArgumentException {
  if (!method.isAnnotationPresent(Compensable.class)) {
    return;
  }
  //解析注解
  Compensable compensable = method.getAnnotation(Compensable.class);
  //补偿方法
  String compensationMethod = compensable.compensationMethod();
  // we don't support the retries number below -1.
  if (compensable.forwardRetries() < -1) {
    throw new IllegalArgumentException(String.format("Compensable %s of method %s, the forward retries should not below -1.", compensable, method.getName()));
  }
  loadMethodContext(method, compensationMethod);
}
```

org.apache.servicecomb.pack.omega.transaction.spring.MethodCheckingCallback#loadMethodContext

```java
protected void loadMethodContext(Method method, String ... candidates) {
  for (String each : candidates) {
    try {
      //获取补偿方法
      Method signature = bean.getClass().getDeclaredMethod(each, method.getParameterTypes());
      //补偿方法的全限定名称
      String key = getTargetBean(bean).getClass().getDeclaredMethod(each, method.getParameterTypes()).toString();
      //补偿方法的全限定名称->(Bean,补偿方法)
      callbackContext.addCallbackContext(key, signature, bean);
      LOG.debug("Found callback method [{}] in {}", each, bean.getClass().getCanonicalName());
    } catch (Exception ex) {
      throw new OmegaException(
          "No such " + callbackType + " method [" + each + "] found in " + bean.getClass().getCanonicalName(), ex);
    }
  }
}
```

2、在Bean实例化之后，查找标有OmegaContextAware注解的字段并且类型是Executor，创建代理类。目的是在线程池中执行事务方法可以获取到正确的globalTxId、localTxId，这两个信息都是保存在ThreadLocal中的，如果跨线程，是获取不到的。

org.apache.servicecomb.pack.omega.transaction.spring.CompensableAnnotationProcessor#checkFields

```java
private void checkFields(Object bean) {
  ReflectionUtils.doWithFields(bean.getClass(), new ExecutorFieldCallback(bean, omegaContext));
}
```

org.apache.servicecomb.pack.omega.transaction.spring.ExecutorFieldCallback#doWith

```java
public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
  if (!field.isAnnotationPresent(OmegaContextAware.class)) {
    return;
  }
  ReflectionUtils.makeAccessible(field);
  Class<?> generic = field.getType();
  //只对Executor类型拦截
  if (!Executor.class.isAssignableFrom(generic)) {
    throw new IllegalArgumentException(
        "Only Executor, ExecutorService, and ScheduledExecutorService are supported for @"
            + OmegaContextAware.class.getSimpleName());
  }
  field.set(bean, ExecutorProxy.newInstance(field.get(bean), field.getType(), omegaContext));
}
```

对Execuor进行代理，拦截execute方法

org.apache.servicecomb.pack.omega.transaction.spring.ExecutorFieldCallback.ExecutorProxy#newInstance

```java
private static Object newInstance(Object target, Class<?> targetClass, OmegaContext omegaContext) {
  Class<?>[] interfaces = targetClass.isInterface() ? new Class<?>[] {targetClass} : targetClass.getInterfaces();
  return Proxy.newProxyInstance(
      targetClass.getClassLoader(),
      interfaces,
      new ExecutorProxy(target, omegaContext));
}
```

执行execute方法时，需对Runnable、Callable进行代理

org.apache.servicecomb.pack.omega.transaction.spring.ExecutorFieldCallback.ExecutorProxy#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  return method.invoke(target, augmentRunnablesWithOmegaContext(args));
}
```

对Runnable或者Callable创建代理

org.apache.servicecomb.pack.omega.transaction.spring.ExecutorFieldCallback.ExecutorProxy#augmentRunnablesWithOmegaContext

```java
private Object[] augmentRunnablesWithOmegaContext(Object[] args) {
  Object[] augmentedArgs = new Object[args.length];
  for (int i = 0; i < args.length; i++) {
    Object arg = args[i];
    if (isExecutable(arg)) {
      augmentedArgs[i] = RunnableProxy.newInstance(arg, omegaContext);
    } else if (isCollectionOfExecutables(arg)) {
      List argList = new ArrayList();
      Collection argCollection = (Collection<?>) arg;
      for (Object a : argCollection) {
        argList.add(RunnableProxy.newInstance(a, omegaContext));
      }
      augmentedArgs[i] = argList;
    } else {
      augmentedArgs[i] = arg;
    }
  }
  return augmentedArgs;
}
```

在创建Runable或者Callable时，先获取到当前线程的事务信息

org.apache.servicecomb.pack.omega.transaction.spring.ExecutorFieldCallback.RunnableProxy#RunnableProxy

```java
private RunnableProxy(OmegaContext omegaContext, Object runnable) {
  this.omegaContext = omegaContext;
  this.globalTxId = omegaContext.globalTxId();
  this.localTxId = omegaContext.localTxId();
  this.runnable = runnable;
}
```

跨线程执行时，将实例化时获取到事务信息和被线程池调度的线程绑定

org.apache.servicecomb.pack.omega.transaction.spring.ExecutorFieldCallback.RunnableProxy#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    LOG.debug("Setting OmegaContext with globalTxId [{}] & localTxId [{}]",
        globalTxId,
        localTxId);
    omegaContext.setGlobalTxId(globalTxId);
    omegaContext.setLocalTxId(localTxId);
    return method.invoke(runnable, args);
  } finally {
    omegaContext.clear();
    LOG.debug("Cleared OmegaContext with globalTxId [{}] & localTxId [{}]",
        globalTxId,
        localTxId);
  }
}
```

## Saga与Alpha的通信

### GrpcSagaClientMessageSender

负责TxEvent的最终发送

org.apache.servicecomb.pack.omega.connector.grpc.saga.GrpcSagaClientMessageSender#GrpcSagaClientMessageSender

```java
public GrpcSagaClientMessageSender(
    String address,
    ManagedChannel channel,
    MessageSerializer serializer,
    MessageDeserializer deserializer,
    ServiceConfig serviceConfig,
    MessageHandler handler,
    LoadBalanceContext loadContext) {
  this.target = address;
  //异步
  this.asyncEventService = TxEventServiceGrpc.newStub(channel);
  //同步
  this.blockingEventService = TxEventServiceGrpc.newBlockingStub(channel);
  this.serializer = serializer;
  this.compensateStreamObserver =
      new GrpcCompensateStreamObserver(loadContext, this, handler, deserializer);
  this.serviceConfig = serviceConfig(serviceConfig.serviceName(), serviceConfig.instanceId());
}
```

向Alpha发送TxEvent

org.apache.servicecomb.pack.omega.connector.grpc.saga.GrpcSagaClientMessageSender#send

```java
public AlphaResponse send(TxEvent event) {
  GrpcAck grpcAck = blockingEventService.onTxEvent(convertEvent(event));
  return new AlphaResponse(grpcAck.getAborted());
}
```

### GrpcCompensateStreamObserver

接收Alpha推送的GrpcCompensateCommand

org.apache.servicecomb.pack.omega.connector.grpc.saga.GrpcCompensateStreamObserver#onNext

```java
public void onNext(GrpcCompensateCommand command) {
  LOG.info("Received compensate command, global tx id: {}, local tx id: {}, compensation method: {}",
      command.getGlobalTxId(), command.getLocalTxId(), command.getCompensationMethod());
  messageHandler.onReceive(
      command.getGlobalTxId(),
      command.getLocalTxId(),
      command.getParentTxId().isEmpty() ? null : command.getParentTxId(),
      command.getCompensationMethod(),
      deserializer.deserialize(command.getPayloads().toByteArray()));
}
```

### SagaLoadBalanceSender

Saga事务的发送均衡器

开始发送TxEvent前，选择要发送到哪个Alpha

org.apache.servicecomb.pack.omega.connector.grpc.saga.SagaLoadBalanceSender#send

```java
public AlphaResponse send(TxEvent event) {
  do {
    final SagaMessageSender messageSender = pickMessageSender();
    Optional<AlphaResponse> response = doGrpcSend(messageSender, event, new SenderExecutor<TxEvent>() {
      @Override
      public AlphaResponse apply(TxEvent event) {
        return messageSender.send(event);
      }
    });
    if (response.isPresent()) return response.get();
  } while (!Thread.currentThread().isInterrupted());

  throw new OmegaException("Failed to send event " + event + " due to interruption");
}
```

#### FastestSender

选择MessageSender，优先发送给响应时间快的MessageSender

org.apache.servicecomb.pack.omega.connector.grpc.core.FastestSender#pick

```java
public MessageSender pick(Map<? extends MessageSender, Long> messageSenders, Supplier<MessageSender> defaultSender) {
  Long min = Long.MAX_VALUE;
  MessageSender sender = null;
  for (Map.Entry<? extends MessageSender, Long> entry : messageSenders.entrySet()) {
    if (entry.getValue() != Long.MAX_VALUE && min > entry.getValue()) {
      min = entry.getValue();
      sender = entry.getKey();
    }
  }
  if (sender == null) {
    return defaultSender.get();
  } else {
    return sender;
  }
}
```

发送请求，修改messageSender对应的Alpha的响应时间

org.apache.servicecomb.pack.omega.connector.grpc.core.LoadBalanceSenderAdapter#doGrpcSend

```java
public <T> Optional<AlphaResponse> doGrpcSend(MessageSender messageSender, T event, SenderExecutor<T> executor) {
  AlphaResponse response = null;
  try {
    long startTime = System.nanoTime();
    response = executor.apply(event);
    //设置此MessageSender连接的alpha处理请求的时间，每次都挑选响应时间比较快的alpha进行通信
    loadContext.getSenders().put(messageSender, System.nanoTime() - startTime);
  } catch (OmegaException e) {
    throw e;
  } catch (Exception e) {
    LOG.error("Retry sending event {} due to failure", event, e);
    loadContext.getSenders().put(messageSender, Long.MAX_VALUE);
  }
  return Optional.fromNullable(response);
}
```

### 执行补偿方法

接收Alpha信息

org.apache.servicecomb.pack.omega.transaction.CompensationMessageHandler#onReceive

```java
public void onReceive(String globalTxId, String localTxId, String parentTxId, String compensationMethod,
    Object... payloads) {
  //回调补偿方法
  context.apply(globalTxId, localTxId, compensationMethod, payloads);
  //向Alpha发送TxCompensatedEvent
  sender.send(new TxCompensatedEvent(globalTxId, localTxId, parentTxId, compensationMethod));
}
```

org.apache.servicecomb.pack.omega.transaction.CallbackContext#apply

```java
public void apply(String globalTxId, String localTxId, String callbackMethod, Object... payloads) {
  CallbackContextInternal contextInternal = contexts.get(callbackMethod);
  String oldGlobalTxId = omegaContext.globalTxId();
  String oldLocalTxId = omegaContext.localTxId();
  try {
    //补偿方法所对应的事务信息保存到omegaContext的ThreadLocal中，防止在补偿方法通过omegaContext获取到的事务信息不正确
    omegaContext.setGlobalTxId(globalTxId);
    omegaContext.setLocalTxId(localTxId);
    //执行补偿方法
    contextInternal.callbackMethod.invoke(contextInternal.target, payloads);
    if (omegaContext.getAlphaMetas().isAkkaEnabled()) {
      sender.send(
          new TxCompensateAckSucceedEvent(omegaContext.globalTxId(), omegaContext.localTxId(),
              omegaContext.globalTxId()));
    }
    LOG.info("Callback transaction with global tx id [{}], local tx id [{}]", globalTxId, localTxId);
  } catch (IllegalAccessException | InvocationTargetException e) {
    if (omegaContext.getAlphaMetas().isAkkaEnabled()) {
      sender.send(
          new TxCompensateAckFailedEvent(omegaContext.globalTxId(), omegaContext.localTxId(),
              omegaContext.globalTxId()));
    }
    LOG.error(
        "Pre-checking for callback method " + contextInternal.callbackMethod.toString()
            + " was somehow skipped, did you forget to configure callback method checking on service startup?",
        e);
  } finally {
    omegaContext.setGlobalTxId(oldGlobalTxId);
    omegaContext.setLocalTxId(oldLocalTxId);
  }
}
```

# TCC事务

## TccStart总事务

对标有TccStart注解的方法进行代理，TccStartAnnotationProcessor在事务的执行前、执行成功、执行异常时向Alpha发送TccStartedEvent、TccEndedEvent（Succeed）、TccEndedEvent（Failed）

org.apache.servicecomb.pack.omega.transaction.tcc.TccStartAspect#advise

```java
@Around("execution(@org.apache.servicecomb.pack.omega.context.annotations.TccStart * *(..)) && @annotation(tccStart)")
Object advise(ProceedingJoinPoint joinPoint, TccStart tccStart) throws Throwable {
  initializeOmegaContext();
  Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
  tccStartAnnotationProcessor.preIntercept(context.globalTxId(), method.toString(), tccStart.timeout());
  LOG.debug("Initialized context {} before execution of method {}", context, method.toString());
  try {
    //业务逻辑
    Object result = joinPoint.proceed();
    tccStartAnnotationProcessor.postIntercept(context.globalTxId(), method.toString());
    LOG.debug("Transaction with context {} has finished.", context);
    return result;
  } catch (Throwable throwable) {
    // We don't need to handle the OmegaException here
    if (!(throwable instanceof OmegaException)) {
      tccStartAnnotationProcessor.onError(context.globalTxId(), method.toString(), throwable);
      LOG.error("Transaction {} failed.", context.globalTxId());
    }
    throw throwable;
  } finally {
    context.clear();
  }
}
```

## TCC子事务

对标有Participate注解的方法代理，向Alpha发送ParticipationStartedEvent、ParticipationEndedEvent（Succeed）、ParticipationEndedEvent（Failed），含有confirmMethod和cancelMethod信息

org.apache.servicecomb.pack.omega.transaction.tcc.TccParticipatorAspect#advise

```java
@Around("execution(@org.apache.servicecomb.pack.omega.transaction.annotations.Participate * *(..)) && @annotation(participate)")
Object advise(ProceedingJoinPoint joinPoint, Participate participate) throws Throwable {
  Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
  TransactionContext transactionContext = extractTransactionContext(joinPoint.getArgs());
  if (transactionContext != null) {
    populateOmegaContext(context, transactionContext);
  }
  //透传的parenId
  String localTxId = context.localTxId();
  //获取cancelMethod
  String cancelMethod = callbackMethodSignature(joinPoint, participate.cancelMethod(), method);
  //获取confirmMethod
  String confirmMethod = callbackMethodSignature(joinPoint, participate.confirmMethod(), method);
  //本地事务Id
  context.newLocalTxId();
  LOG.debug("Updated context {} for participate method {} ", context, method.toString());
  try {
    AlphaResponse response = tccMessageSender.participationStart(new ParticipationStartedEvent(context.globalTxId(), context.localTxId(), localTxId,
                                 confirmMethod, cancelMethod));
    if(response.aborted()){
      throw new OmegaException("transcation has aborted: " + context.globalTxId());
    }
    //执行业务逻辑，try方法
    Object result = joinPoint.proceed();
    // Send the participate message back
    tccMessageSender.participationEnd(new ParticipationEndedEvent(context.globalTxId(), context.localTxId(), localTxId,
        confirmMethod, cancelMethod, TransactionStatus.Succeed));
    //本地维护补偿方法的参数（localTxId -> args）
    parametersContext.putParameters(context.localTxId(), joinPoint.getArgs());
    LOG.debug("Participate Transaction with context {} has finished.", context);
    return result;
  } catch (Throwable throwable) {
    // Now we don't handle the error message
    if(!(throwable instanceof OmegaException)){
      tccMessageSender.participationEnd(new ParticipationEndedEvent(context.globalTxId(), context.localTxId(), localTxId,
              confirmMethod, cancelMethod, TransactionStatus.Failed));
    }
    LOG.error("Participate Transaction with context {} failed.", context, throwable);
    throw throwable;
  } finally {
    context.setLocalTxId(localTxId);
  }
}
```

## 维护补偿方法

查找标有Participate注解的方法和标有OmegaContextAware注解的字段，ParticipateAnnotationProcessor实现了BeanPostProcessor接口，实现postProcessAfterInitialization，在Bean实例化之后，进行查找

org.apache.servicecomb.pack.omega.transaction.spring.ParticipateMethodCheckingCallback#doWith

```java
public void doWith(Method method) throws IllegalArgumentException {
  if (!method.isAnnotationPresent(Participate.class)) {
    return;
  }
  String confirmMethod = method.getAnnotation(Participate.class).confirmMethod();
  String cancelMethod = method.getAnnotation(Participate.class).cancelMethod();
  loadMethodContext(method, confirmMethod, cancelMethod);
}
```

```java
protected void loadMethodContext(Method method, String ... candidates) {
  for (String each : candidates) {
    try {
      //获取补偿方法
      Method signature = bean.getClass().getDeclaredMethod(each, method.getParameterTypes());
      //补偿方法全限定名称
      String key = getTargetBean(bean).getClass().getDeclaredMethod(each, method.getParameterTypes()).toString();
      callbackContext.addCallbackContext(key, signature, bean);
      LOG.debug("Found callback method [{}] in {}", each, bean.getClass().getCanonicalName());
    } catch (Exception ex) {
      throw new OmegaException(
          "No such " + callbackType + " method [" + each + "] found in " + bean.getClass().getCanonicalName(), ex);
    }
  }
}
```

## TCC与Alpha通信

org.apache.servicecomb.pack.omega.connector.grpc.tcc.GrpcTccClientMessageSender#GrpcTccClientMessageSender

```java
public GrpcTccClientMessageSender(ServiceConfig serviceConfig,
    ManagedChannel channel,
    String address,
    TccMessageHandler handler,
    LoadBalanceContext loadContext) {
  this.target = address;
  tccBlockingEventService = TccEventServiceGrpc.newBlockingStub(channel);
  tccAsyncEventService = TccEventServiceGrpc.newStub(channel);
  this.serviceConfig = serviceConfig(serviceConfig.serviceName(), serviceConfig.instanceId());
  observer = new GrpcCoordinateStreamObserver(loadContext, this, handler);
}
```

接收Alpha推送消息

org.apache.servicecomb.pack.omega.connector.grpc.tcc.GrpcCoordinateStreamObserver#onNext

```java
public void onNext(GrpcTccCoordinateCommand command) {
  LOG.info("Received coordinate command, global tx id: {}, local tx id: {}, call method: {}",
      command.getGlobalTxId(), command.getLocalTxId(), command.getMethod());
  messageHandler.onReceive(command.getGlobalTxId(), command.getLocalTxId(), command.getParentTxId(), command.getMethod());
}
```

## 执行补偿方法

org.apache.servicecomb.pack.omega.transaction.tcc.CoordinateMessageHandler#onReceive

```java
public void onReceive(String globalTxId, String localTxId, String parentTxId, String methodName) {
  // 根据localTxId获取参数，执行补偿方法
  callbackContext.apply(globalTxId, localTxId, methodName, parametersContext.getParameters(localTxId));
  tccMessageSender.coordinate(new CoordinatedEvent(globalTxId, localTxId, parentTxId, methodName, TransactionStatus.Succeed));
  // 移除本地保存的参数
  parametersContext.removeParameter(localTxId);
}
```

### 
