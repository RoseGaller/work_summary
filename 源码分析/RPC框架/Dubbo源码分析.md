# Dubbo源码分析

* [扩展点机制](#扩展点机制)
* [消费方异步调用](#消费方异步调用)
  * [使用 CompletableFuture 签名的接口](#使用-completablefuture-签名的接口)
  * [使用 RpcContext](#使用-rpccontext)
* [线程模型](#线程模型)
  * [AllDispatcher](#alldispatcher)
  * [DirectDispatcher](#directdispatcher)
  * [MessageOnlyDispatcher](#messageonlydispatcher)
  * [ExecutionDispatcher](#executiondispatcher)
  * [ConnectionOrderedDispatcher](#connectionordereddispatcher)
* [线程池](#线程池)
  * [LimitedThreadPool](#limitedthreadpool)
  * [EagerThreadPool](#eagerthreadpool)
* [服务自省](#服务自省)
  * [DynamicConfigurationServiceNameMapping](#dynamicconfigurationservicenamemapping)
    * [提供方发布](#提供方发布)
    * [消费方订阅](#消费方订阅)
  * [RemoteWritableMetadataService](#remotewritablemetadataservice)


#  扩展点机制

JDK SPI

在查找实现类的过程中，需要遍历SPI配置文件中定义的索所有实现类，该过程中会将所有的实现类进行实例化

Dubbo SPI

按需实例化，对SPI配置文件进行了修改和扩展

将配置文件改成了KV格式

dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol

使用方式

```java
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("dubbo")
```

com.alibaba.dubbo.common.extension.ExtensionLoader#getExtensionLoader

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type +
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }

    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

com.alibaba.dubbo.common.extension.ExtensionLoader#getExtension

```java
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    //是否有缓存的实例
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) { //双重检测
                instance = createExtension(name); //创建实例
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

com.alibaba.dubbo.common.extension.ExtensionLoader#createExtension

```java
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name); //获取实现类class
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz); //获取实例
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance()); //存放创建的实例
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance); //注入
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```

com.alibaba.dubbo.common.extension.ExtensionLoader#getExtensionClasses

```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses(); //加载class
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

com.alibaba.dubbo.common.extension.ExtensionLoader#loadExtensionClasses

```java
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) { //默认的实现
        String value = defaultAnnotation.value();
        if (value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY); //存放dubbo内部使用的SPI配置文件
    loadFile(extensionClasses, DUBBO_DIRECTORY); //存放用户自定义的SPI配置文件
    loadFile(extensionClasses, SERVICES_DIRECTORY); //兼容JDK SPI
    return extensionClasses;
}
```

com.alibaba.dubbo.common.extension.ExtensionLoader#injectExtension

```java
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                 //方法以set开头、修饰符时public、只有一个参数
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0]; //参数类型
                    try {
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //从objectFactory中获取对应的实例（dubbo spi获取或者spring容器获取）
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object); //注入到instance中
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

# 消费方异步调用

## 使用 CompletableFuture 签名的接口

```java
public interface AsyncService {
    CompletableFuture<String> sayHello(String name);
}
```

## 使用 RpcContext

在 consumer.xml 中配置：

```xml
<dubbo:reference id="asyncService" interface="org.apache.dubbo.samples.governance.api.AsyncService">
      <dubbo:method name="sayHello" async="true" />
</dubbo:reference>
```

调用代码:

```java
// 此调用会立即返回null
asyncService.sayHello("world");
// 拿到调用的Future引用，当结果返回后，会被通知和设置到此Future
CompletableFuture<String> helloFuture = RpcContext.getContext().getCompletableFuture();
// 为Future添加回调
helloFuture.whenComplete((retValue, exception) -> {
    if (exception == null) {
        System.out.println(retValue);
    } else {
        exception.printStackTrace();
    }
});
```

org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke

```java
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    inv.setAttachment(Constants.VERSION_KEY, version);

    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation); //是否异步发送
        //返回值是否是CompletableFuture
        boolean isAsyncFuture = RpcUtils.isReturnTypeFuture(inv);
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);//是否关注响应信息
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (isOneway) { //无需关注响应结果
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) { //异步发送
            ResponseFuture future = currentClient.request(inv, timeout);
            // For compatibility
            FutureAdapter<Object> futureAdapter = new FutureAdapter<>(future);
            RpcContext.getContext().setFuture(futureAdapter); //当前线程绑定FutureAdapter

            Result result;
            if (isAsyncFuture) { //方法返回值类型为CompletableFuture
                // register resultCallback, sometimes we need the async result being processed by the filter chain.
                result = new AsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
            } else {
                result = new SimpleAsyncRpcResult(futureAdapter, futureAdapter.getResultFuture(), false);
            }
            return result;
        } else { //同步发送
            RpcContext.getContext().setFuture(null);
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

# 线程模型

## AllDispatcher

默认实现，所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。

```java
public class AllDispatcher implements Dispatcher {

    public static final String NAME = "all";

    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new AllChannelHandler(handler, url); //继承WrappedChannelHandler，重写所有方法，先从WrappedChannelHandler获取线程池对象executor，再将执行的任务封装，交由线程池执行
    }

}
```

## DirectDispatcher

所有消息都不派发到线程池，全部在 IO 线程上直接执行。

```java
public class DirectDispatcher implements Dispatcher {

    public static final String NAME = "direct";

    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return handler; //WrappedChannelHandle
    }

}
```

## MessageOnlyDispatcher

只有**请求、响应消**息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。

```java
public class MessageOnlyDispatcher implements Dispatcher {

    public static final String NAME = "message";

    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new MessageOnlyChannelHandler(handler, url); //与AllDispatcher类似，只重写received方法
    }

}
```

## ExecutionDispatcher

只有**请求消息**派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行

```java
public class ExecutionDispatcher implements Dispatcher {

    public static final String NAME = "execution";

    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new ExecutionChannelHandler(handler, url);//与MessageOnlyDispatcher类似，只不过在receive方法中区分了请求和响应
    }

}
```

## ConnectionOrderedDispatcher

将连接断开事件由单独的线程池有序逐个执行，其它消息派发到线程池。

```java
public class ConnectionOrderedDispatcher implements Dispatcher {

    public static final String NAME = "connection";

    @Override
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {
        return new ConnectionOrderedChannelHandler(handler, url);
    }

}
```

# 线程池

## LimitedThreadPool

池中的线程数只会增长不会收缩,避免收缩时突然来了大流量引起的性能问题

```java
public class LimitedThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(Constants.THREAD_NAME_KEY, Constants.DEFAULT_THREAD_NAME);
        int cores = url.getParameter(Constants.CORE_THREADS_KEY, Constants.DEFAULT_CORE_THREADS);
        int threads = url.getParameter(Constants.THREADS_KEY, Constants.DEFAULT_THREADS);
        int queues = url.getParameter(Constants.QUEUES_KEY, Constants.DEFAULT_QUEUES);
        return new ThreadPoolExecutor(cores, threads, Long.MAX_VALUE, TimeUnit.MILLISECONDS,
                queues == 0 ? new SynchronousQueue<Runnable>() :
                        (queues < 0 ? new LinkedBlockingQueue<Runnable>()
                                : new LinkedBlockingQueue<Runnable>(queues)),
                new NamedInternalThreadFactory(name, true), new AbortPolicyWithReport(name, url));
    }

}
```

## EagerThreadPool

优先创建`Worker`线程池，在达到最大线程数时，将任务放入阻塞队列中。优先创建线程处理任务，避免在队列中的排队等待，降低了延迟

```java
public class EagerThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(Constants.THREAD_NAME_KEY, Constants.DEFAULT_THREAD_NAME);
        int cores = url.getParameter(Constants.CORE_THREADS_KEY, Constants.DEFAULT_CORE_THREADS);
        int threads = url.getParameter(Constants.THREADS_KEY, Integer.MAX_VALUE);
        int queues = url.getParameter(Constants.QUEUES_KEY, Constants.DEFAULT_QUEUES);
        int alive = url.getParameter(Constants.ALIVE_KEY, Constants.DEFAULT_ALIVE);

        // init queue and executor
        TaskQueue<Runnable> taskQueue = new TaskQueue<Runnable>(queues <= 0 ? 1 : queues);
        EagerThreadPoolExecutor executor = new EagerThreadPoolExecutor(cores,
                threads,
                alive,
                TimeUnit.MILLISECONDS,
                taskQueue,
                new NamedInternalThreadFactory(name, true),
                new AbortPolicyWithReport(name, url));
        taskQueue.setExecutor(executor);
        return executor;
    }
}
```

重写execute方法

org.apache.dubbo.common.threadpool.support.eager.EagerThreadPoolExecutor#execute

```java
public void execute(Runnable command) {
    if (command == null) {
        throw new NullPointerException();
    }
    // do not increment in method beforeExecute!
    submittedTaskCount.incrementAndGet();
    try {
        super.execute(command);
    } catch (RejectedExecutionException rx) {
        // retry to offer the task into queue.
        final TaskQueue queue = (TaskQueue) super.getQueue();
        try {
            if (!queue.retryOffer(command, 0, TimeUnit.MILLISECONDS)) {
                submittedTaskCount.decrementAndGet();
                throw new RejectedExecutionException("Queue capacity is full.", rx);
            }
        } catch (InterruptedException x) {
            submittedTaskCount.decrementAndGet();
            throw new RejectedExecutionException(x);
        }
    } catch (Throwable t) {
        // decrease any way
        submittedTaskCount.decrementAndGet();
        throw t;
    }
}
```

org.apache.dubbo.common.threadpool.support.eager.TaskQueue#offer

```java
public boolean offer(Runnable runnable) {
    if (executor == null) {
        throw new RejectedExecutionException("The task queue does not have executor!");
    }
		//当前线程数
    int currentPoolThreadSize = executor.getPoolSize();
    //有空闲的work线程，放入队列由空闲的work线程处理
    if (executor.getSubmittedTaskCount() < currentPoolThreadSize) {
        return super.offer(runnable);
    }
		//当前线程数小于最大线程数，并且没有空闲woker线程，则创建新的work线程
    // return false to let executor create new worker.
    if (currentPoolThreadSize < executor.getMaximumPoolSize()) {
        return false; //表明创建新的线程
    }

    // currentPoolThreadSize >= max
    return super.offer(runnable); //放入阻塞队列
}
```

# 服务自省

在微服务架构中，服务是基本单位，而 Dubbo 架构中服务的基本单位是 Java 接口

一个 Dubbo Provider 节点可以注册多个服务接口。随着业务发展，服务接口的数量会越来越多，就导致注册中心的内存压力越来越大。

从 2.7.5 版本开始，Dubbo 引入了服务自省架构。以应用维度进行注册，而不是接口维度进行注册。减轻注册中心的压力

1、Provider启动，发布业务服务同时发布元数据服务

2、向注册中心注册（应用名 -> ip地址）

3、Consumer启动，根据应用名向注册中心拉取ip列表

4、Consumer根据IP地址向Provider发送元数据请求服务，获取接口级别的元数据信息

5、Consumer根据返回的元数据向对应的Provider发送调用业务服务的请求

## 提供方发布

org.apache.dubbo.config.event.listener.ServiceNameMappingListener#onEvent

```java
public void onEvent(ServiceConfigExportedEvent event) {//提供方启动之后发布事件
    ServiceConfig serviceConfig = event.getServiceConfig();
    List<URL> exportedURLs = serviceConfig.getExportedUrls();
    exportedURLs.forEach(url -> {
        String serviceInterface = url.getServiceInterface();
        String group = url.getParameter(GROUP_KEY);
        String version = url.getParameter(VERSION_KEY);
        String protocol = url.getProtocol();
        serviceNameMapping.map(serviceInterface, group, version, protocol);//向配置中心注册（应用服务名称->接口名称）
    });
}
```

org.apache.dubbo.metadata.DynamicConfigurationServiceNameMapping#map

```java
public void map(String serviceInterface, String group, String version, String protocol) {

    if (IGNORED_SERVICE_INTERFACES.contains(serviceInterface)) {
        return;
    }
		//获取配置中心
    DynamicConfiguration dynamicConfiguration = DynamicConfiguration.getDynamicConfiguration();

    // the Dubbo Service Key as group
    // the service(application) name as key
    // It does matter whatever the content is, we just need a record
    String key = getName(); //应用服务名称
    String content = valueOf(System.currentTimeMillis());
    execute(() -> {
        dynamicConfiguration.publishConfig(key, buildGroup(serviceInterface, group, version, protocol), content);
        if (logger.isInfoEnabled()) {
            logger.info(String.format("Dubbo service[%s] mapped to interface name[%s].",
                    group, serviceInterface, group));
        }
    });
}
```

org.apache.dubbo.metadata.DynamicConfigurationServiceNameMapping#buildGroup

```java
protected static String buildGroup(String serviceInterface, String group, String version, String protocol) { //发布的value
    //        the issue : https://github.com/apache/dubbo/issues/4671
    //        StringBuilder groupBuilder = new StringBuilder(serviceInterface)
    //                .append(KEY_SEPARATOR).append(defaultString(group))
    //                .append(KEY_SEPARATOR).append(defaultString(version))
    //                .append(KEY_SEPARATOR).append(defaultString(protocol));
    //        return groupBuilder.toString();
    return DEFAULT_MAPPING_GROUP + SLASH + serviceInterface;
}
```

## 消费方订阅

```java
protected void subscribeURLs(URL url, NotifyListener listener) {

    writableMetadataService.subscribeURL(url);

 	 //根据url获取servciceNames
    Set<String> serviceNames = getServices(url); 
    if (CollectionUtils.isEmpty(serviceNames)) {
        throw new IllegalStateException("Should has at least one way to know which services this interface belongs to, subscription url: " + url);
    }
		//根据serviceNames从注册中心获取对应的ip地址
    serviceNames.forEach(serviceName -> subscribeURLs(url, listener, serviceName));

}
```

org.apache.dubbo.metadata.DynamicConfigurationServiceNameMapping#get

```java
public Set<String> get(String serviceInterface, String group, String version, String protocol) { 
  		//根据接口名称从配置中心拉取应用服务名称
    DynamicConfiguration dynamicConfiguration = DynamicConfiguration.getDynamicConfiguration();

    Set<String> serviceNames = new LinkedHashSet<>();
    execute(() -> {
        Set<String> keys = dynamicConfiguration.getConfigKeys(buildGroup(serviceInterface, group, version, protocol)); //以接口、分组、版本号、协议为key，从配置中心获取对应的应用名称
        serviceNames.addAll(keys);
    });
    return Collections.unmodifiableSet(serviceNames);
}
```

```java
protected void subscribeURLs(URL url, NotifyListener listener, String serviceName) {
	//根据serviceName从注册中获取ip地址
    List<ServiceInstance> serviceInstances = serviceDiscovery.getInstances(serviceName);

    subscribeURLs(url, listener, serviceName, serviceInstances);

    // register ServiceInstancesChangedListener
    registerServiceInstancesChangedListener(url, new ServiceInstancesChangedListener(serviceName) {

        @Override
        public void onEvent(ServiceInstancesChangedEvent event) {
            subscribeURLs(url, listener, event.getServiceName(), new ArrayList<>(event.getServiceInstances()));
        }
    });
}
```

org.apache.dubbo.metadata.store.RemoteWritableMetadataServiceDelegate#publishServiceDefinition

```java
public void publishServiceDefinition(URL providerUrl) {
    defaultWritableMetadataService.publishServiceDefinition(providerUrl); //本地存储
    remoteWritableMetadataService.publishServiceDefinition(providerUrl); //远端存储
}
```

org.apache.dubbo.metadata.store.InMemoryWritableMetadataService#publishServiceDefinition

```java
public void publishServiceDefinition(URL providerUrl) {
    try {
        String interfaceName = providerUrl.getParameter(INTERFACE_KEY);
        if (StringUtils.isNotEmpty(interfaceName)
                && !ProtocolUtils.isGeneric(providerUrl.getParameter(GENERIC_KEY))) {
            Class interfaceClass = Class.forName(interfaceName);
            ServiceDefinition serviceDefinition = ServiceDefinitionBuilder.build(interfaceClass);
            Gson gson = new Gson();
            String data = gson.toJson(serviceDefinition);
            //
            serviceDefinitions.put(providerUrl.getServiceKey(), data);
            return;
        }
        logger.error("publishProvider interfaceName is empty . providerUrl: " + providerUrl.toFullString());
    } catch (ClassNotFoundException e) {
        //ignore error
        logger.error("publishProvider getServiceDescriptor error. providerUrl: " + providerUrl.toFullString(), e);
    }
}
```

## RemoteWritableMetadataService

org.apache.dubbo.metadata.store.RemoteWritableMetadataService#publishServiceDefinition

```java
public void publishServiceDefinition(URL providerUrl) { //服务提供方启动时，将服务元数据信息注册到配置中心
    try {
        String interfaceName = providerUrl.getParameter(INTERFACE_KEY);
        if (StringUtils.isNotEmpty(interfaceName)
                && !ProtocolUtils.isGeneric(providerUrl.getParameter(GENERIC_KEY))) {
            Class interfaceClass = Class.forName(interfaceName);
            ServiceDefinition serviceDefinition = ServiceDefinitionBuilder.build(interfaceClass);
            getMetadataReport().storeProviderMetadata(new MetadataIdentifier(providerUrl.getServiceInterface(),
                    providerUrl.getParameter(VERSION_KEY), providerUrl.getParameter(GROUP_KEY),
                    null, null), serviceDefinition);
            return;
        }
        logger.error("publishProvider interfaceName is empty . providerUrl: " + providerUrl.toFullString());
    } catch (ClassNotFoundException e) {
        //ignore error
        logger.error("publishProvider getServiceDescriptor error. providerUrl: " + providerUrl.toFullString(), e);
    }

    // backward compatibility
    publishProvider(providerUrl);
}
```

org.apache.dubbo.metadata.report.support.AbstractMetadataReport#storeProviderMetadata

```java
public void storeProviderMetadata(MetadataIdentifier providerMetadataIdentifier, ServiceDefinition serviceDefinition) {
    if (syncReport) {
        storeProviderMetadataTask(providerMetadataIdentifier, serviceDefinition);
    } else {
        reportCacheExecutor.execute(() -> storeProviderMetadataTask(providerMetadataIdentifier, serviceDefinition));
    }
}
```

org.apache.dubbo.metadata.report.support.AbstractMetadataReport#storeProviderMetadataTask

```java
private void storeProviderMetadataTask(MetadataIdentifier providerMetadataIdentifier, ServiceDefinition serviceDefinition) {
    try {
        if (logger.isInfoEnabled()) {
            logger.info("store provider metadata. Identifier : " + providerMetadataIdentifier + "; definition: " + serviceDefinition);
        }
        allMetadataReports.put(providerMetadataIdentifier, serviceDefinition);
        failedReports.remove(providerMetadataIdentifier);
        Gson gson = new Gson();
        String data = gson.toJson(serviceDefinition);
      	//远端存储
        doStoreProviderMetadata(providerMetadataIdentifier, data);
        saveProperties(providerMetadataIdentifier, data, true, !syncReport);
    } catch (Exception e) {
        // retry again. If failed again, throw exception.
        failedReports.put(providerMetadataIdentifier, serviceDefinition);
        metadataReportRetry.startRetryTask();
        logger.error("Failed to put provider metadata " + providerMetadataIdentifier + " in  " + serviceDefinition + ", cause: " + e.getMessage(), e);
    }
}
```

