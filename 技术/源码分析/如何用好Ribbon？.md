# 如何启用？

使用时，只需在RestTemplate上添加对应@LoadBalanced注解即可，表明启用负载均衡功能

Ribbon的负载均衡是通过给标有@LoadBalanced注解的RestTemplate添加拦截器实现的，在拦截器内实现服务实例的负载均衡

```java
@Bean
@LoadBalanced // Ribbon负载均衡
public RestTemplate getRestTemplate() {
  return new RestTemplate();
}
```

```yml
#针对被调用的微服务，设置负载策略
调用方微服务名称:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
```

# 重要组件

## ServerList

存储服务实例。存储分为静态和动态两种方式

静态存储需要事先配置好固定的服务实例信息

动态存储需要从注册中心获取对应的服务实例信息

```java
public interface ServerList<T extends Server> {
    public List<T> getInitialListOfServers();
    /**
     * Return updated list of servers. This is called say every 30 secs
     */
    public List<T> getUpdatedListOfServers();   
}
```

## ServerListFilter

有了服务信息后，在某些场景下可能能需要过滤一部分服务实例

```java
public interface ServerListFilter<T extends Server> {
    public List<T> getFilteredListOfServers(List<T> servers);
}
```

## ServerListUpdater

用于服务实例更新操作

```java
public interface ServerListUpdater {

    /**
     * an interface for the updateAction that actually executes a server list update
     */
    public interface UpdateAction {
        void doUpdate();
    }
    /**
     * start the serverList updater with the given update action
     * This call should be idempotent.
     *
     * @param updateAction
     */
    void start(UpdateAction updateAction);

    /**
     * stop the serverList updater. This call should be idempotent
     */
    void stop();

    /**
     * @return the last update timestamp as a {@link java.util.Date} string
     */
    String getLastUpdate();

    /**
     * @return the number of ms that has elapsed since last update
     */
    long getDurationSinceLastUpdateMs();

    /**
     * @return the number of update cycles missed, if valid
     */
    int getNumberMissedCycles();

    /**
     * @return the number of threads used, if vaid
     */
    int getCoreThreads();
}
```

## IPing

来检测服务实例信息是否可用 

```java
public interface IPing {
    /**
     * Checks whether the given <code>Server</code> is "alive" i.e. should be considered a candidate while loadbalancing
     */
    public boolean isAlive(Server server);
}
```

## IRule 

提供了很多种算法策略来选择实例信息

```java
public interface IRule{
    /*
     * choose one alive server from lb.allServers or
     * lb.upServers according to key
     */
    public Server choose(Object key);
    public void setLoadBalancer(ILoadBalancer lb);
    public ILoadBalancer getLoadBalancer();    
}
```

## ILoadBalancer 

ILoadBalancer 中定义了软件负载均衡操作的接口，比如动态更新一组服务列表，根据指定算法从现有服务器列表中选择一个可用的服务等操作

```java
public interface ILoadBalancer {

   /**
    * Initial list of servers.
    * This API also serves to add additional ones at a later time
    * The same logical server (host:port) could essentially be added multiple times
    * (helpful in cases where you want to give more "weightage" perhaps ..)
    * 
    * @param newServers new servers to add
    */
   public void addServers(List<Server> newServers);
   
   /**
    * Choose a server from load balancer.
    * 
    * @param key An object that the load balancer may use to determine which server to return. null if 
    *         the load balancer does not use this parameter.
    * @return server chosen
    */
   public Server chooseServer(Object key);
   
   /**
    * To be called by the clients of the load balancer to notify that a Server is down
    * else, the LB will think its still Alive until the next Ping cycle - potentially
    * (assuming that the LB Impl does a ping)
    * 
    * @param server Server to mark as down
    */
   public void markServerDown(Server server);
   
   /**
    * @deprecated 2016-01-20 This method is deprecated in favor of the
    * cleaner {@link #getReachableServers} (equivalent to availableOnly=true)
    * and {@link #getAllServers} API (equivalent to availableOnly=false).
    *
    * Get the current list of servers.
    *
    * @param availableOnly if true, only live and available servers should be returned
    */
   @Deprecated
   public List<Server> getServerList(boolean availableOnly);

   /**
    * @return Only the servers that are up and reachable.
     */
    public List<Server> getReachableServers();

    /**
     * @return All known servers, both reachable and unreachable.
     */
   public List<Server> getAllServers();
}
```

# 自动配置

查找被@LoadBalanced标注的RestTemplate

查找增强器，其实就是对RestTemplate设置拦截器

注入SmartInitializingSingleton，负责执行增强

注入LoadBalancerInterceptor，负责服务实例的负载均衡

RibbonLoadBalancerClient

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
  //根据serviceId获取ILoadBalancer
  //根据ILoadBalancer获取ServiceInstance
   <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
  //执行具体请求
   <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
  //拼接请求方式 传统中是ip:port 现在是服务名称:port 形式
   URI reconstructURI(ServiceInstance instance, URI original);
}
```

## RibbonAutoConfiguration

org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration

```java
@Configuration
@ConditionalOnClass({IClient.class, RestTemplate.class, AsyncRestTemplate.class, Ribbon.class})
@RibbonClients
//先注入EurekaClientAutoConfiguration后注入RibbonAutoConfiguration内部的bean ，需要discoveryClient发现服务
@AutoConfigureAfter(
    name = {"org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration"}
)
//先注入RibbonAutoConfiguration内部的bean再注入LoadBalancerAutoConfiguration
@AutoConfigureBefore({LoadBalancerAutoConfiguration.class, AsyncLoadBalancerAutoConfiguration.class})
@EnableConfigurationProperties({RibbonEagerLoadProperties.class, ServerIntrospectorProperties.class})
public class RibbonAutoConfiguration {
  
  //在类上标注了RibbonClients、RibbonClient，集合中存放每个服务对应的配置信息
  	@Autowired(required = false)
	private List<RibbonClientSpecification> configurations = new ArrayList<>();

  //配置信息：是否要预先初始化、预先初始化的服务
	@Autowired
	private RibbonEagerLoadProperties ribbonEagerLoadProperties;

	@Bean
	public HasFeatures ribbonFeature() {
		return HasFeatures.namedFeature("Ribbon", Ribbon.class);
	}

  //子ApplicationContext,存放全局的配置信息以及每个服务独自的配置信息
	@Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
		factory.setConfigurations(this.configurations);
		return factory;
	}
	//RibbonLoadBalancerClient
	@Bean
	@ConditionalOnMissingBean(LoadBalancerClient.class)
	public LoadBalancerClient loadBalancerClient() {
		return new RibbonLoadBalancerClient(springClientFactory());
	}
  //重试策略
	@Bean
	@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
	@ConditionalOnMissingBean
	public LoadBalancedRetryFactory loadBalancedRetryPolicyFactory(final SpringClientFactory clientFactory) {
		return new RibbonLoadBalancedRetryFactory(clientFactory);
	}

  //用于获取配置文件中每个服务对应的ILoadBalancer、IPing、IRule、ServerListFilter、ServerList
  //服务名称.ribbon.NFLoadBalancerRuleClassName:class全限定名称
	@Bean
	@ConditionalOnMissingBean
	public PropertiesFactory propertiesFactory() {
		return new PropertiesFactory();
	}

  //提前初始化，监听ApplicationReadyEvent事件
	@Bean
	@ConditionalOnProperty(value = "ribbon.eager-load.enabled")
	public RibbonApplicationContextInitializer ribbonApplicationContextInitializer() {
		return new RibbonApplicationContextInitializer(springClientFactory(),
				ribbonEagerLoadProperties.getClients());
	}
}
```

### SpringClientFactory

org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration#springClientFactory

```java
@Bean
public SpringClientFactory springClientFactory() {
   SpringClientFactory factory = new SpringClientFactory();
   factory.setConfigurations(this.configurations);
   return factory;
}
```

```java
public SpringClientFactory() {
   //通过SpringClientFactory获取bean实例时，
   //注入RibbonClientConfiguration
   super(RibbonClientConfiguration.class, NAMESPACE, "ribbon.client.name");
}
```

#### RibbonClientConfiguration

```java
//默认配置

@Bean
@ConditionalOnMissingBean
public IClientConfig ribbonClientConfig() { //配置信息
   DefaultClientConfigImpl config = new DefaultClientConfigImpl();
   config.loadProperties(this.name);
  //连接、读取超时
   config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
   config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
   return config;
}
//自定义rule注入容器的两种方式
//1、注解注入,此种情况不会执行ribbonRule。RibbonClientConfiguration会延迟加载
//2、配置文件注入，会执行ribbonRule
@Bean
@ConditionalOnMissingBean
public IRule ribbonRule(IClientConfig config) { //负载策略
   if (this.propertiesFactory.isSet(IRule.class, name)) {
      return this.propertiesFactory.get(IRule.class, config, name);
   }
  //默认负载策略
   ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
   rule.initWithNiwsConfig(config);
   return rule;
}
@Bean
@ConditionalOnMissingBean
@SuppressWarnings("unchecked")
public ServerList<Server> ribbonServerList(IClientConfig config) {
  //配置文件中设置了Server地址
   if (this.propertiesFactory.isSet(ServerList.class, name)) {
      return this.propertiesFactory.get(ServerList.class, config, name);
   }
   ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
   serverList.initWithNiwsConfig(config);
   return serverList;
}
@Bean
@ConditionalOnMissingBean
public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
   return new PollingServerListUpdater(config);
}
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
      ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
      IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
   if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
      return this.propertiesFactory.get(ILoadBalancer.class, config, name);
   }
   return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
         serverListFilter, serverListUpdater);
}
```

## LoadBalancerAutoConfiguration

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

   //存放标有LoadBalanced注解的RestTemplate
   @LoadBalanced
   @Autowired(required = false)
   private List<RestTemplate> restTemplates = Collections.emptyList();

  //注入RestTemplateCustomizer，创建SmartInitializingSingleton，增强RestTemplate
  //其实就是加入一个拦截器LoadBalancerInterceptor
  //实现SmartInitializingSingleton接口的类，会在非延迟加载的Bean注入到容器之后
  //执行afterSingletonsInstantiated
   @Bean
   public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
         final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
      return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
   }

    //转换请求
   @Autowired(required = false)
   private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

  
   //注入LoadBalancerClient（在RibbonAutoConfiguration中生成）
   //LoadBalancerRequestFactory:创建LoadBalancerRequest
   @Bean
   @ConditionalOnMissingBean
   public LoadBalancerRequestFactory loadBalancerRequestFactory(
         LoadBalancerClient loadBalancerClient) {
      return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
   }

  //在缺失RetryTemplate的情况下才会进行实例化
   @Configuration
   @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
   static class LoadBalancerInterceptorConfig {
     
     //创建拦截器LoadBalancerInterceptor，负载请求
      @Bean
      public LoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient, //负责执行请求
            LoadBalancerRequestFactory requestFactory)// 负责创建请求
      {
         return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
      }

     //创建RestTemplateCustomizer，负责将拦截器注入到RestTemplate
      @Bean
      @ConditionalOnMissingBean
      public RestTemplateCustomizer restTemplateCustomizer(
            final LoadBalancerInterceptor loadBalancerInterceptor) {
         return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
      }
   }

  //以下与RetryTemplate相关
   @Configuration
   @ConditionalOnClass(RetryTemplate.class)
   public static class RetryAutoConfiguration {

      @Bean
      @ConditionalOnMissingBean
      public LoadBalancedRetryFactory loadBalancedRetryFactory() {
         return new LoadBalancedRetryFactory() {};
      }
   }

   @Configuration
   @ConditionalOnClass(RetryTemplate.class)
   public static class RetryInterceptorAutoConfiguration {
      @Bean
      @ConditionalOnMissingBean
      public RetryLoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient, LoadBalancerRetryProperties properties,
            LoadBalancerRequestFactory requestFactory,
            LoadBalancedRetryFactory loadBalancedRetryFactory) {
         return new RetryLoadBalancerInterceptor(loadBalancerClient, properties,
               requestFactory, loadBalancedRetryFactory);
      }

      @Bean
      @ConditionalOnMissingBean
      public RestTemplateCustomizer restTemplateCustomizer(
            final RetryLoadBalancerInterceptor loadBalancerInterceptor) {
         return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
      }
   }

```

### SmartInitializingSingleton

给集合中的每一个RestTemplate对象添加一个拦截器

```java
@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
      final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {//增强restTemplate的时机
   return () -> restTemplateCustomizers.ifAvailable(customizers -> {
           for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
               for (RestTemplateCustomizer customizer : customizers) {
                   customizer.customize(restTemplate);
               }
           }
       });
}
```

### LoadBalancerInterceptor

```java
@Bean
public LoadBalancerInterceptor ribbonInterceptor(
      LoadBalancerClient loadBalancerClient,
      LoadBalancerRequestFactory requestFactory) {
  //在背部实现负载均衡
   return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
}
```

### RestTemplateCustomizer

为RestTemplate添加负载均衡拦截器

```java
@Bean
@ConditionalOnMissingBean
public RestTemplateCustomizer restTemplateCustomizer(
      final LoadBalancerInterceptor loadBalancerInterceptor) {
   return restTemplate -> {
              List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                      restTemplate.getInterceptors());
              list.add(loadBalancerInterceptor);
              restTemplate.setInterceptors(list);
          };
}
```

### LoadBalancerRequestFactory

```java
@Bean
@ConditionalOnMissingBean
public LoadBalancerRequestFactory loadBalancerRequestFactory(
      LoadBalancerClient loadBalancerClient) {
  //创建LoadBalancerRequest
   return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
}

```

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(final HttpRequest request,
      final byte[] body, final ClientHttpRequestExecution execution) {
   return instance -> {
           HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
           if (transformers != null) {
               for (LoadBalancerRequestTransformer transformer : transformers) {
                   serviceRequest = transformer.transformRequest(serviceRequest, instance);
               }
           }
           return execution.execute(serviceRequest, body);
       };
}
```



# 执行流程

org.springframework.web.client.RestTemplate#doExecute

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
      @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

   Assert.notNull(url, "URI is required");
   Assert.notNull(method, "HttpMethod is required");
   ClientHttpResponse response = null;
   try {
     	//创建LoadBalancerRequest
      ClientHttpRequest request = createRequest(url, method);
      if (requestCallback != null) {
         requestCallback.doWithRequest(request);
      }
     //调用LoadBalancerRequest
      response = request.execute();
      handleResponse(url, method, response);
      return (responseExtractor != null ? responseExtractor.extractData(response) : null);
   }
   catch (IOException ex) {
      String resource = url.toString();
      String query = url.getRawQuery();
      resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
      throw new ResourceAccessException("I/O error on " + method.name() +
            " request for \"" + resource + "\": " + ex.getMessage(), ex);
   }
   finally {
      if (response != null) {
         response.close();
      }
   }
}
```

## 创建请求

```java
protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
  //最终返回InterceptingClientHttpRequest
   ClientHttpRequest request = getRequestFactory().createRequest(url, method);
   if (logger.isDebugEnabled()) {
      logger.debug("Created " + method.name() + " request for \"" + url + "\"");
   }
   return request;
}
```

```java
//获取请求工厂
public ClientHttpRequestFactory getRequestFactory() { //获取
   List<ClientHttpRequestInterceptor> interceptors = getInterceptors(); //获取注入的拦截器
   if (!CollectionUtils.isEmpty(interceptors)) {
      ClientHttpRequestFactory factory = this.interceptingRequestFactory;
      if (factory == null) { //创建InterceptingClientHttpRequestFactory
        //super.getRequestFactory() -> RibbonClientHttpRequestFactory
         factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), i，nterceptors);
         this.interceptingRequestFactory = factory;
      }
      return factory;
   }
   else {//没有拦截器
      return super.getRequestFactory();
   }
}
```

org.springframework.http.client.AbstractClientHttpRequestFactoryWrapper#createRequest(java.net.URI, org.springframework.http.HttpMethod)

```java
public final ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException {
    return this.createRequest(uri, httpMethod, this.requestFactory);
}
```

org.springframework.http.client.InterceptingClientHttpRequestFactory#createRequest

```java
protected ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) {
   return new InterceptingClientHttpRequest(requestFactory, this.interceptors, uri, httpMethod);
}
```

## 执行

org.springframework.http.client.InterceptingClientHttpRequest#executeInternal

```java
protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
   InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
   return requestExecution.execute(this, bufferedOutput);
}
```

org.springframework.http.client.InterceptingClientHttpRequest.InterceptingRequestExecution#execute

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
   if (this.iterator.hasNext()) { 
     	//执行拦截器
      ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
     //this代表InterceptingRequestExecution
      return nextInterceptor.intercept(request, body, this);
   }//无拦截器或者都已经都已经执行完毕
   else {
      HttpMethod method = request.getMethod();
      Assert.state(method != null, "No standard HTTP method");
     	//调用RibbonClientHttpRequestFactory返回RibbonHttpRequest
      ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
     //添加headers
      request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
     //添加请求体
      if (body.length > 0) {
         if (delegate instanceof StreamingHttpOutputMessage) {
            StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
            streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
         }
         else {
            StreamUtils.copy(body, delegate.getBody());
         }
      }
      return delegate.execute();
   }
}
```

### 执行拦截器

#### BasicAuthorizationInterceptor

org.springframework.http.client.support.BasicAuthorizationInterceptor#intercept

```java
public ClientHttpResponse intercept(HttpRequest request, byte[] body,
      ClientHttpRequestExecution execution) throws IOException {

   String token = Base64Utils.encodeToString((this.username + ":" + this.password).getBytes(UTF_8));
   request.getHeaders().add("Authorization", "Basic " + token);
  //继续调用InterceptingRequestExecution的方法
   return execution.execute(request, body); 
}
```

#### LoadBalancerInterceptor

org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor#intercept

```java
//负责挑选服务实例，挑选出实例，最后再调用InterceptingRequestExecution的execute方法
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
      final ClientHttpRequestExecution execution) throws IOException {
   final URI originalUri = request.getURI();//请求路径
   String serviceName = originalUri.getHost();//服务名称
   Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
  //loadBalancer -> RibbonLoadBalancerClient
   return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution)//创建LoadBalancerRequest请求
                                   );
}
```

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(final HttpRequest request,
      final byte[] body, final ClientHttpRequestExecution execution) {
   return instance -> {
           HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
           if (transformers != null) {
               for (LoadBalancerRequestTransformer transformer : transformers) {
                   serviceRequest = transformer.transformRequest(serviceRequest, instance);
               }
           }
           return execution.execute(serviceRequest, body);
       };
}
```

org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execute

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
   //获取负载均衡器
  	ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
   //通过负载均衡器选择发送请求的服务实例
    Server server = this.getServer(loadBalancer);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    } else {
        //创建RibbonServer
        RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
       //发送请求
        return this.execute(serviceId, ribbonServer, request);
    }
}
```

```java
public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
    Server server = null;
    if (serviceInstance instanceof RibbonLoadBalancerClient.RibbonServer) {
        server = ((RibbonLoadBalancerClient.RibbonServer)serviceInstance).getServer();
    }

    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    } else {
        RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
        RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

        try {
          //发送请求
            T returnVal = request.apply(serviceInstance);
            statsRecorder.recordStats(returnVal);
            return returnVal;
        } catch (IOException var8) {
            statsRecorder.recordStats(var8);
            throw var8;
        } catch (Exception var9) {
            statsRecorder.recordStats(var9);
            ReflectionUtils.rethrowRuntimeException(var9);
            return null;
        }
    }
}
```

```java
//执行apply
HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
if (transformers != null) {
    for (LoadBalancerRequestTransformer transformer : transformers) {
        serviceRequest = transformer.transformRequest(serviceRequest, instance);
    }
}
//execution -> InterceptingRequestExecution
return execution.execute(serviceRequest, body);
```

org.springframework.cloud.netflix.ribbon.RibbonHttpRequest#executeInternal

```java
@Override
protected ClientHttpResponse executeInternal(HttpHeaders headers)
      throws IOException {
   try {
      addHeaders(headers);
      if (outputStream != null) {
         outputStream.close();
         builder.entity(outputStream.toByteArray());
      }
      HttpRequest request = builder.build();
     	//RestClient
      HttpResponse response = client.executeWithLoadBalancer(request, config);
      return new RibbonHttpResponse(response);
   } catch (Exception e) {
      throw new IOException(e);
   }
}
```

# 如何获取ILoadBalancer？

org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#getLoadBalancer

```java
protected ILoadBalancer getLoadBalancer(String serviceId) {
  	//根据serviceId（servicename）获取ILoadBalancer
   return this.clientFactory.getLoadBalancer(serviceId);
}
```

```java
public ILoadBalancer getLoadBalancer(String name) {
   return getInstance(name, ILoadBalancer.class);
}
```

```java
public <C> C getInstance(String name, Class<C> type) {
   C instance = super.getInstance(name, type);
   if (instance != null) {
      return instance;
   }
   IClientConfig config = getInstance(name, IClientConfig.class);
   return instantiateWithConfig(getContext(name), type, config);
}
```

org.springframework.cloud.context.named.NamedContextFactory#getInstance(java.lang.String, java.lang.Class<T>)

```java
public <T> T getInstance(String name, Class<T> type) {
  	//根据服务名称获取AnnotationConfigApplicationContext
    AnnotationConfigApplicationContext context = this.getContext(name);
    return BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, type).length > 0 ? context.getBean(type) : null;
}
```



```java
protected AnnotationConfigApplicationContext getContext(String name) {
    if (!this.contexts.containsKey(name)) { //没有AnnotationConfigApplicationContext
        synchronized(this.contexts) {
            if (!this.contexts.containsKey(name)) {
              	//创建AnnotationConfigApplicationContext
                this.contexts.put(name, this.createContext(name));
            }
        }
    }
    return (AnnotationConfigApplicationContext)this.contexts.get(name);
}
```

```java
protected AnnotationConfigApplicationContext createContext(String name) {
  //传递过来的name其实就是服务名称
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    //默认情况下是没有的
   if (this.configurations.containsKey(name)) {
      for (Class<?> configuration : this.configurations.get(name)
            .getConfiguration()) {
         //有的话，注册到AnnotationConfigApplicationContext
         context.register(configuration);
      }
   }
   for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
      if (entry.getKey().startsWith("default.")) {
         for (Class<?> configuration : entry.getValue().getConfiguration()) {
            context.register(configuration);
         }
      }
   }
   context.register(PropertyPlaceholderAutoConfiguration.class,
         this.defaultConfigType);
   context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
         this.propertySourceName,
         Collections.<String, Object> singletonMap(this.propertyName, name)));
   if (this.parent != null) {
      // Uses Environment from parent as well as beans
      context.setParent(this.parent);
   }
   context.setDisplayName(generateDisplayName(name));
   context.refresh();
   return context;
}
```



# 如何注册自定义配置信息？

org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration

```java
//默认情况下是缺省的，所有的服务使用全局配置
//RibbonClientSpecification，每个服务实例对应一个
//属性name(服务名称)和configuration（配置类，比如每个服务实例对应的负载策略）
@Autowired(required = false)
private List<RibbonClientSpecification> configurations = new ArrayList<>();
//使用
//对user-service这个服务设置自定义的负载策略，其他的使用全局配置
@RibbonClient(name="user-service", configuration=BeanConfiguration.class)
public class RibbonClientConfig {
}
//当获取IRule时，先根据服务名称获取AnnotationConfigApplicationContext
//然后再从AnnotationConfigApplicationContext获取IRule类型的实例
public class BeanConfiguration {
	@Bean
	public IRule myRule() {
		return new MyRule();
	}
}
```

org.springframework.cloud.netflix.ribbon.RibbonClient

```java
@Configuration
@Import(RibbonClientConfigurationRegistrar.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RibbonClient {
   String value() default "";
   String name() default "";
   Class<?>[] configuration() default {};
}
```

org.springframework.cloud.netflix.ribbon.RibbonClientConfigurationRegistrar#registerBeanDefinitions

```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
   Map<String, Object> attrs = metadata.getAnnotationAttributes(
         RibbonClients.class.getName(), true);
   if (attrs != null && attrs.containsKey("value")) {
      AnnotationAttributes[] clients = (AnnotationAttributes[]) attrs.get("value");
      for (AnnotationAttributes client : clients) {
         registerClientConfiguration(registry, getClientName(client),
               client.get("configuration"));
      }
   }
   if (attrs != null && attrs.containsKey("defaultConfiguration")) {
      String name;
      if (metadata.hasEnclosingClass()) {
         name = "default." + metadata.getEnclosingClassName();
      } else {
         name = "default." + metadata.getClassName();
      }
      registerClientConfiguration(registry, name,
            attrs.get("defaultConfiguration"));
   }
   Map<String, Object> client = metadata.getAnnotationAttributes(
         RibbonClient.class.getName(), true);
   String name = getClientName(client);
   if (name != null) {
      registerClientConfiguration(registry, name, client.get("configuration"));
   }
}

private String getClientName(Map<String, Object> client) {
   if (client == null) {
      return null;
   }
   String value = (String) client.get("value");
   if (!StringUtils.hasText(value)) {
      value = (String) client.get("name");
   }
   if (StringUtils.hasText(value)) {
      return value;
   }
   throw new IllegalStateException(
         "Either 'name' or 'value' must be provided in @RibbonClient");
}

private void registerClientConfiguration(BeanDefinitionRegistry registry,
      Object name, Object configuration) {
  //注册RibbonClientSpecification
   BeanDefinitionBuilder builder = BeanDefinitionBuilder
         .genericBeanDefinition(RibbonClientSpecification.class);
   builder.addConstructorArgValue(name);
   builder.addConstructorArgValue(configuration);
   registry.registerBeanDefinition(name + ".RibbonClientSpecification",
         builder.getBeanDefinition());
}
```

# 如何提前初始化？

```yaml
#SpringClientFactory（feign对应FeignContext）
ribbon:
	eager-load:
    enabled: true
    clients:  #服务名称，初始化服务对应的AnnotationConfigApplicationContext
```

```java
@EnableConfigurationProperties({RibbonEagerLoadProperties.class, ServerIntrospectorProperties.class})
public class RibbonAutoConfiguration {
	@Autowired
	private RibbonEagerLoadProperties ribbonEagerLoadProperties;
  @Bean
  //默认false，不会提前初始化
	@ConditionalOnProperty(value = "ribbon.eager-load.enabled")
	public RibbonApplicationContextInitializer ribbonApplicationContextInitializer() {
		return new RibbonApplicationContextInitializer(springClientFactory(),
				ribbonEagerLoadProperties.getClients());
	}
}
```

```java
@ConfigurationProperties(prefix = "ribbon.eager-load")
public class RibbonEagerLoadProperties {
   private boolean enabled = false;
   private List<String> clients;

   public boolean isEnabled() {
      return enabled;
   }

   public void setEnabled(boolean enabled) {
      this.enabled = enabled;
   }

   public List<String> getClients() {
      return clients;
   }

   public void setClients(List<String> clients) {
      this.clients = clients;
   }
}
```

org.springframework.cloud.netflix.ribbon.RibbonApplicationContextInitializer#onApplicationEvent

```java
@Override
public void onApplicationEvent(ApplicationReadyEvent event) {
   initialize();
}
```

```java
protected void initialize() {
   if (clientNames != null) {
      for (String clientName : clientNames) {
         this.springClientFactory.getContext(clientName);
      }
   }
}
```

