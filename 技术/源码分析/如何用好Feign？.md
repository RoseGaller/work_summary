# 如何启用？

在启动类添加@EnableFeignClients

在服务接口上添加@FeignClient，属性name表明服务名称，属性configuration指定服务对应的配置类

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({FeignClientsRegistrar.class})
public @interface EnableFeignClients {
    String[] value() default {};
		//指定扫描的包，创建代理
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
  	//默认配置
    Class<?>[] defaultConfiguration() default {};
    Class<?>[] clients() default {};
}
```

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FeignClient {
    @AliasFor("name")
    String value() default "";
    @Deprecated
    String serviceId() default "";
    String contextId() default "";
    @AliasFor("value")
    String name() default "";
    String qualifier() default "";
    String url() default "";
    boolean decode404() default false;
    Class<?>[] configuration() default {};
    Class<?> fallback() default void.class;
    Class<?> fallbackFactory() default void.class;
    String path() default "";
    boolean primary() default true;
}
```

# 注册信息

向容器中注册Feign相关的配置类、接口类

org.springframework.cloud.openfeign.FeignClientsRegistrar#registerBeanDefinitions

```java
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    this.registerDefaultConfiguration(metadata, registry);
    this.registerFeignClients(metadata, registry);
}
```

注册配置信息

org.springframework.cloud.openfeign.FeignClientsRegistrar#registerDefaultConfiguration

```java
//注册FeignClient的全局配置到容器中
private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    Map<String, Object> defaultAttrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
    if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
        String name;
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        } else {
            name = "default." + metadata.getClassName();
        }
        this.registerClientConfiguration(registry, name, defaultAttrs.get("defaultConfiguration"));
    }

}
```

```java
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name, Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(name + "." + FeignClientSpecification.class.getSimpleName(), builder.getBeanDefinition());
}
```

注册Bean信息

```java
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {//为标有@FeignClient的接口生成代理对象
  	//扫描标有FeignClient注解的接口，注册到容器
   ClassPathScanningCandidateComponentProvider scanner = getScanner();
   scanner.setResourceLoader(this.resourceLoader);

   Set<String> basePackages;

   Map<String, Object> attrs = metadata
         .getAnnotationAttributes(EnableFeignClients.class.getName());
   AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
         FeignClient.class);
  //标有FeignClient注解的接口
   final Class<?>[] clients = attrs == null ? null
         : (Class<?>[]) attrs.get("clients");
   if (clients == null || clients.length == 0) {
      scanner.addIncludeFilter(annotationTypeFilter);
      basePackages = getBasePackages(metadata);
   }
   else {//获取包名
      final Set<String> clientClasses = new HashSet<>();
      basePackages = new HashSet<>();
      for (Class<?> clazz : clients) {
         basePackages.add(ClassUtils.getPackageName(clazz));
         clientClasses.add(clazz.getCanonicalName());
      }
      AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
         @Override
         protected boolean match(ClassMetadata metadata) {
            String cleaned = metadata.getClassName().replaceAll("\\$", ".");
            return clientClasses.contains(cleaned);
         }
      };
      scanner.addIncludeFilter(
            new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
   }
	 //扫描每个包下标有FeignClient注解的接口
   for (String basePackage : basePackages) {
      Set<BeanDefinition> candidateComponents = scanner
            .findCandidateComponents(basePackage);
      for (BeanDefinition candidateComponent : candidateComponents) {
         if (candidateComponent instanceof AnnotatedBeanDefinition) {
           
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            // FeignClient只能用在接口上
            Assert.isTrue(annotationMetadata.isInterface(),
                  "@FeignClient can only be specified on an interface");
						//获取注解的属性值
            Map<String, Object> attributes = annotationMetadata
                  .getAnnotationAttributes(
                        FeignClient.class.getCanonicalName());
						//服务名称
            String name = getClientName(attributes);
           //注册独享的FeignClient配置信息
            registerClientConfiguration(registry, name,
                  attributes.get("configuration"));
					 //注册FeignClientFactoryBean
            registerFeignClient(registry, annotationMetadata, attributes);
         }
      }
   }
}
```



```java
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {//注册FeignClientFactoryBean
   String className = annotationMetadata.getClassName();
   BeanDefinitionBuilder definition = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientFactoryBean.class);
   validate(attributes);
    //填充属性信息
  	//指定固定服务实例的地址，用于测试环境中，不需要负载均衡功能
   definition.addPropertyValue("url", getUrl(attributes)); 
   definition.addPropertyValue("path", getPath(attributes));
   String name = getName(attributes);
   definition.addPropertyValue("name", name);
   definition.addPropertyValue("type", className);
   definition.addPropertyValue("decode404", attributes.get("decode404"));
   definition.addPropertyValue("fallback", attributes.get("fallback"));
   definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
   definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

   String alias = name + "FeignClient";
   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

   boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

   beanDefinition.setPrimary(primary);

   String qualifier = getQualifier(attributes);
   if (StringUtils.hasText(qualifier)) {
      alias = qualifier;
   }

   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
         new String[] { alias });
   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

# 获取代理对象

org.springframework.cloud.openfeign.FeignClientFactoryBean#getObject

```java
public Object getObject() throws Exception {
  //每个服务对应一个FeignContext
   FeignContext context = applicationContext.getBean(FeignContext.class);
  //封装客户端信息
   Feign.Builder builder = feign(context);
   //url为空，具有负载均衡功能
   if (!StringUtils.hasText(this.url)) {
      String url;
      if (!this.name.startsWith("http")) {
         url = "http://" + this.name;
      }
      else {
         url = this.name;
      }
      url += cleanPath();
      
      return loadBalance(builder, context, new HardCodedTarget<>(this.type,
            this.name, url));
   }
  //url不为空，表明固定访问某一服务实例
   if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
      this.url = "http://" + this.url;
   }
   String url = this.url + cleanPath();
   //LoadBalancerFeignClient(DefaultFeignLoadBalancedConfiguration中注入)
   Client client = getOptional(context, Client.class);
   if (client != null) {
      if (client instanceof LoadBalancerFeignClient) {
        //由于是url不需要负载均衡，由于ribbon的存在，交给内部的委托类（feign.Client.Default）实现连接
         client = ((LoadBalancerFeignClient)client).getDelegate();
      }
      builder.client(client);
   }
   //HystrixTargeter or DefaultTargeter
   Targeter targeter = get(context, Targeter.class);
   return targeter.target(this, builder, context, new HardCodedTarget<>(
         this.type, this.name, url));
}
```

```java
//获取Builder，此对象封装了发送请求所需要的一些组件
protected Builder feign(FeignContext context) {
  	//日志工程
    FeignLoggerFactory loggerFactory = (FeignLoggerFactory)this.get(context, FeignLoggerFactory.class);
   //创建日志
    Logger logger = loggerFactory.create(this.type);
   //创建Builder
    Builder builder = ((Builder)this.get(context, Builder.class)).logger(logger).encoder((Encoder)this.get(context, Encoder.class)).decoder((Decoder)this.get(context, Decoder.class)).contract((Contract)this.get(context, Contract.class));
    //配置 Builder
    this.configureFeign(context, builder);
    return builder;
}
```

```java
protected void configureFeign(FeignContext context, Builder builder) {
    FeignClientProperties properties = (FeignClientProperties)this.applicationContext.getBean(FeignClientProperties.class);
    if (properties != null) {
        if (properties.isDefaultToProperties()) { //默认为true
          	//文件信息会覆盖配置类中的信息
            //优先使用配置类中信息
            this.configureUsingConfiguration(context, builder);  	
						//再用配置文件中信息
          //优先使用配置文件中default的信息
          this.configureUsingProperties((FeignClientConfiguration)properties.getConfig().get(properties.getDefaultConfig()), builder);
    //根据服务名称获取配置文件中的信息
          this.configureUsingProperties((FeignClientConfiguration)properties.getConfig().get(this.contextId), builder);
        } else {
          //优先使用配置文件中的信息
            this.configureUsingProperties((FeignClientConfiguration)properties.getConfig().get(properties.getDefaultConfig()), builder);
            this.configureUsingProperties((FeignClientConfiguration)properties.getConfig().get(this.contextId), builder);
            //后使用配置类中的信息，配置类会覆盖配置文件中的信息
            this.configureUsingConfiguration(context, builder);
        }
    } else {
        this.configureUsingConfiguration(context, builder);
    }

}
```

```java
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })
@Configuration
@AutoConfigureBefore(FeignAutoConfiguration.class)
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
//Order is important here, last should be the default, first should be optional
// see https://github.com/spring-cloud/spring-cloud-netflix/issues/2086#issuecomment-316281653
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
		OkHttpFeignLoadBalancedConfiguration.class,
		DefaultFeignLoadBalancedConfiguration.class }) //此处引入LoadBalancerFeignClient
public class FeignRibbonClientAutoConfiguration {
	@Bean
	@Primary
	@ConditionalOnMissingBean
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	public CachingSpringLoadBalancerFactory cachingLBClientFactory(
			SpringClientFactory factory) {
		return new CachingSpringLoadBalancerFactory(factory);
	}

	@Bean
	@Primary
	@ConditionalOnMissingBean
	@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
	public CachingSpringLoadBalancerFactory retryabeCachingLBClientFactory(
		SpringClientFactory factory,
		LoadBalancedRetryFactory retryFactory) {
		return new CachingSpringLoadBalancerFactory(factory, retryFactory);
	}

  //封装了连接、读取超时时间
	@Bean
	@ConditionalOnMissingBean
	public Request.Options feignRequestOptions() {
		return LoadBalancerFeignClient.DEFAULT_OPTIONS;
	}
}
@Configuration
class DefaultFeignLoadBalancedConfiguration {
   //注入LoadBalancerFeignClient
   @Bean
   @ConditionalOnMissingBean
   public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
                       SpringClientFactory clientFactory) {
      return new LoadBalancerFeignClient(new Client.Default(null, null),
            cachingFactory, clientFactory);
   }
}
```

org.springframework.cloud.openfeign.FeignAutoConfiguration

```java
@Configuration
//classpath中存在Feign
@ConditionalOnClass(Feign.class)
//FeignClientProperties:服务对应的配置信息
//FeignHttpClientProperties：连接信息
@EnableConfigurationProperties({FeignClientProperties.class, FeignHttpClientProperties.class})
public class FeignAutoConfiguration {
  
  //每个服务对应的配置信息（@FeignClient中的name和configuration）
  @Autowired(required = false)
  private List<FeignClientSpecification> configurations = new ArrayList<>();
  
  //feign容器，内部维护了每个服务对应的Context
	@Bean
	public FeignContext feignContext() {
		FeignContext context = new FeignContext();
		context.setConfigurations(this.configurations);
		return context;
	}
  
  //classpath中存在HystrixFeign
  //如果想禁用hystrix,则排除io.github.openfeign:feign-hystrix jar
  @Configuration
	@ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
	protected static class HystrixFeignTargeterConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public Targeter feignTargeter() {
			return new HystrixTargeter();
		}
	}
   //classpath中不存在HystrixFeign
  @Configuration
	@ConditionalOnMissingClass("feign.hystrix.HystrixFeign")
	protected static class DefaultFeignTargeterConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public Targeter feignTargeter() {
			return new DefaultTargeter();
		}
	}
}
```

org.springframework.cloud.openfeign.FeignClientFactoryBean#loadBalance

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
      HardCodedTarget<T> target) {
 	// LoadBalancerFeignClient
   Client client = getOptional(context, Client.class);
   if (client != null) {
      builder.client(client);
     //DefaultTargeter or HystrixTargeter
      Targeter targeter = get(context, Targeter.class);
     //生成代理对象
      return targeter.target(this, builder, context, target);
   }

   throw new IllegalStateException(
         "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```

生成代理对象

org.springframework.cloud.openfeign.DefaultTargeter#target

```java
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
               Target.HardCodedTarget<T> target) {
   return feign.target(target);
}
```

feign.Feign.Builder#target(feign.Target<T>)

```java
public <T> T target(Target<T> target) {
    return this.build().newInstance(target);
}
```

```java
public Feign build() {

  SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
      new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
                                           logLevel, decode404);
  ParseHandlersByName handlersByName =
      new ParseHandlersByName(contract, options, encoder, decoder,
                              errorDecoder, synchronousMethodHandlerFactory);
  return new ReflectiveFeign(handlersByName, invocationHandlerFactory);
}
```

feign.ReflectiveFeign#newInstance

```java
public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = this.targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList();
    Method[] var5 = target.type().getMethods();
    int var6 = var5.length;

    for(int var7 = 0; var7 < var6; ++var7) {
        Method method = var5[var7];
        if (method.getDeclaringClass() != Object.class) {
            if (Util.isDefault(method)) {
                DefaultMethodHandler handler = new DefaultMethodHandler(method);
                defaultMethodHandlers.add(handler);
                methodToHandler.put(method, handler);
            } else {
                methodToHandler.put(method, (MethodHandler)nameToHandler.get(Feign.configKey(target.type(), method)));
            }
        }
    }

    InvocationHandler handler = this.factory.create(target, methodToHandler);
  //创建代理对象
    T proxy = Proxy.newProxyInstance(target.type().getClassLoader(), new Class[]{target.type()}, handler);
    Iterator var12 = defaultMethodHandlers.iterator();

    while(var12.hasNext()) {
        DefaultMethodHandler defaultMethodHandler = (DefaultMethodHandler)var12.next();
        defaultMethodHandler.bindTo(proxy);
    }

    return proxy;
}
```

# 执行

feign.SynchronousMethodHandler#invoke

```java
public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = this.buildTemplateFromArgs.create(argv);
 	 //重试
    Retryer retryer = this.retryer.clone();
    while(true) {
        try {
          //执行、解码
            return this.executeAndDecode(template);
        } catch (RetryableException var5) {
          	//如果超过重试次数，抛出异常
            retryer.continueOrPropagate(var5);
            if (this.logLevel != Level.NONE) {
                this.logger.logRetry(this.metadata.configKey(), this.logLevel);
            }
        }
    }
}
```

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
  	//创建请求
    Request request = this.targetRequest(template);
    if (this.logLevel != Level.NONE) {
        this.logger.logRequest(this.metadata.configKey(), this.logLevel, request);
    }
    long start = System.nanoTime();
    Response response;
 	 //发送请求
    try {
        response = this.client.execute(request, this.options);
        response.toBuilder().request(request).build();
    } catch (IOException var15) {
        if (this.logLevel != Level.NONE) {
            this.logger.logIOException(this.metadata.configKey(), this.logLevel, var15, this.elapsedTime(start));
        }
        throw FeignException.errorExecuting(request, var15);
    }
  
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    boolean shouldClose = true;
    Response var9;
    try {
        if (this.logLevel != Level.NONE) {
            response = this.logger.logAndRebufferResponse(this.metadata.configKey(), this.logLevel, response, elapsedTime);
            response.toBuilder().request(request).build();
        }
        if (Response.class != this.metadata.returnType()) {
            Object var19;
            if (response.status() >= 200 && response.status() < 300) {
                if (Void.TYPE == this.metadata.returnType()) {
                    var9 = null;
                    return var9;
                }

                var19 = this.decode(response); //解码响应信息
                return var19;
            }

            if (this.decode404 && response.status() == 404 && Void.TYPE != this.metadata.returnType()) {
                var19 = this.decode(response);
                return var19;
            }
						//响应异常
            throw this.errorDecoder.decode(this.metadata.configKey(), response);
        }

        if (response.body() != null) {
            if (response.body().length() != null && (long)response.body().length() <= 8192L) {
                byte[] bodyData = Util.toByteArray(response.body().asInputStream());
                Response var10 = response.toBuilder().body(bodyData).build();
                return var10;
            }

            shouldClose = false;
            var9 = response;
            return var9;
        }

        var9 = response;
    } catch (IOException var16) {
        if (this.logLevel != Level.NONE) {
            this.logger.logIOException(this.metadata.configKey(), this.logLevel, var16, elapsedTime);
        }

        throw FeignException.errorReading(request, response, var16);
    } finally {
        if (shouldClose) {
            Util.ensureClosed(response.body());
        }

    }
    return var9;
}
```

org.springframework.cloud.netflix.feign.ribbon.LoadBalancerFeignClient#execute

```java
public Response execute(Request request, Options options) throws IOException {
    try {
        URI asUri = URI.create(request.url());
     	 //服务名称
        String clientName = asUri.getHost(); 
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        RibbonRequest ribbonRequest = new RibbonRequest(this.delegate, request, uriWithoutHost);
        IClientConfig requestConfig = this.getClientConfig(options, clientName);
        return ((RibbonResponse)this.lbClient(clientName)//FeignLoadBalancer
                .executeWithLoadBalancer(ribbonRequest, requestConfig)).toResponse();
    } catch (ClientException var8) {
        IOException io = this.findIOException(var8);
        if (io != null) {
            throw io;
        } else {
            throw new RuntimeException(var8);
        }
    }
}
```

org.springframework.cloud.netflix.feign.ribbon.CachingSpringLoadBalancerFactory#create

```java
public FeignLoadBalancer create(String clientName) {
    if (this.cache.containsKey(clientName)) {
        return (FeignLoadBalancer)this.cache.get(clientName);
    } else {
        IClientConfig config = this.factory.getClientConfig(clientName);
        ILoadBalancer lb = this.factory.getLoadBalancer(clientName);
        ServerIntrospector serverIntrospector = (ServerIntrospector)this.factory.getInstance(clientName, ServerIntrospector.class);
        FeignLoadBalancer client = this.enableRetry ? new RetryableFeignLoadBalancer(lb, config, serverIntrospector, this.loadBalancedRetryPolicyFactory, this.loadBalancedBackOffPolicyFactory, this.loadBalancedRetryListenerFactory) : new FeignLoadBalancer(lb, config, serverIntrospector);
        this.cache.put(clientName, client);
        return (FeignLoadBalancer)client;
    }
}
```

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    LoadBalancerCommand command = this.buildLoadBalancerCommand(request, requestConfig);

    try {
      //submit返回服务实例
        return (IResponse)command.submit(new ServerOperation<T>() {
            public Observable<T> call(Server server) {
                URI finalUri = AbstractLoadBalancerAwareClient.this.reconstructURIWithServer(server, request.getUri());
                ClientRequest requestForServer = request.replaceUri(finalUri);
                try {
                  //org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer
                    return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                } catch (Exception var5) {
                    return Observable.error(var5);
                }
            }
        }).toBlocking().single();
    } catch (Exception var6) {
        Throwable t = var6.getCause();
        if (t instanceof ClientException) {
            throw (ClientException)t;
        } else {
            throw new ClientException(var6);
        }
    }
}
```

```java
public Observable<T> submit(final ServerOperation<T> operation) {
    final LoadBalancerCommand<T>.ExecutionInfoContext context = new LoadBalancerCommand.ExecutionInfoContext();
    if (this.listenerInvoker != null) {
        try {
            this.listenerInvoker.onExecutionStart();
        } catch (AbortExecutionException var6) {
            return Observable.error(var6);
        }
    }

    final int maxRetrysSame = this.retryHandler.getMaxRetriesOnSameServer();
    final int maxRetrysNext = this.retryHandler.getMaxRetriesOnNextServer();
    //挑选服务实例
    Observable<T> o = (this.server == null ? this.selectServer() :  Observable.just(this.server)).concatMap(new Func1<Server, Observable<T>>() {
        public Observable<T> call(Server server) {
            context.setServer(server);
          //服务server的状态信息
            final ServerStats stats = LoadBalancerCommand.this.loadBalancerContext.getServerStats(server);
            Observable<T> o = Observable.just(server).concatMap(new Func1<Server, Observable<T>>() {
                public Observable<T> call(final Server server) {
                    context.incAttemptCount();
                    LoadBalancerCommand.this.loadBalancerContext.noteOpenConnection(stats);
                    if (LoadBalancerCommand.this.listenerInvoker != null) {
                        try {
                            LoadBalancerCommand.this.listenerInvoker.onStartWithServer(context.toExecutionInfo());
                        } catch (AbortExecutionException var3) {
                            return Observable.error(var3);
                        }
                    }

                    final Stopwatch tracer = LoadBalancerCommand.this.loadBalancerContext.getExecuteTracer().start();
                    return operation.call(server)//发送请求
                      .doOnEach(new Observer<T>() {
                        private T entity;

                        public void onCompleted() { //执行结束
                            this.recordStats(tracer, stats, this.entity, (Throwable)null);
                        }

                        public void onError(Throwable e) { //执行异常
                            this.recordStats(tracer, stats, (Object)null, e);
                            LoadBalancerCommand.logger.debug("Got error {} when executed on server {}", e, server);
                            if (LoadBalancerCommand.this.listenerInvoker != null) {
                                LoadBalancerCommand.this.listenerInvoker.onExceptionWithServer(e, context.toExecutionInfo());
                            }

                        }

                        public void onNext(T entity) {
                            this.entity = entity;
                            if (LoadBalancerCommand.this.listenerInvoker != null) {
                                LoadBalancerCommand.this.listenerInvoker.onExecutionSuccess(entity, context.toExecutionInfo());
                            }

                        }

                        private void recordStats(Stopwatch tracerx, ServerStats statsx, Object entity, Throwable exception) {
                            tracerx.stop();
                            LoadBalancerCommand.this.loadBalancerContext.noteRequestCompletion(statsx, entity, exception, tracerx.getDuration(TimeUnit.MILLISECONDS), LoadBalancerCommand.this.retryHandler);
                        }
                    });
                }
            });
            if (maxRetrysSame > 0) {
                o = o.retry(LoadBalancerCommand.this.retryPolicy(maxRetrysSame, true));
            }

            return o;
        }
    });
    if (maxRetrysNext > 0 && this.server == null) {
        o = o.retry(this.retryPolicy(maxRetrysNext, false));
    }

  //处理重试异常
    return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {
        public Observable<T> call(Throwable e) {
            if (context.getAttemptCount() > 0) {
                if (maxRetrysNext > 0 && context.getServerAttemptCount() == maxRetrysNext + 1) {
                    e = new ClientException(ErrorType.NUMBEROF_RETRIES_NEXTSERVER_EXCEEDED, "Number of retries on next server exceeded max " + maxRetrysNext + " retries, while making a call for: " + context.getServer(), (Throwable)e);
                } else if (maxRetrysSame > 0 && context.getAttemptCount() == maxRetrysSame + 1) {
                    e = new ClientException(ErrorType.NUMBEROF_RETRIES_EXEEDED, "Number of retries exceeded max " + maxRetrysSame + " retries, while making a call for: " + context.getServer(), (Throwable)e);
                }
            }

            if (LoadBalancerCommand.this.listenerInvoker != null) {
                LoadBalancerCommand.this.listenerInvoker.onExecutionFailed((Throwable)e, context.toFinalExecutionInfo());
            }

            return Observable.error((Throwable)e);
        }
    });
}
```

```java
private Observable<Server> selectServer() {
    return Observable.create(new OnSubscribe<Server>() {
        public void call(Subscriber<? super Server> next) {
            try {
                Server server = LoadBalancerCommand.this.loadBalancerContext.getServerFromLoadBalancer(LoadBalancerCommand.this.loadBalancerURI, LoadBalancerCommand.this.loadBalancerKey);
                next.onNext(server);
                next.onCompleted();
            } catch (Exception var3) {
                next.onError(var3);
            }

        }
    });
}
```

```java
public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
    String host = null;
    int port = -1;
    if (original != null) {
        host = original.getHost();
    }

    if (original != null) {
        Pair<String, Integer> schemeAndPort = this.deriveSchemeAndPortFromPartialUri(original);
        port = (Integer)schemeAndPort.second();
    }

    ILoadBalancer lb = this.getLoadBalancer();
    if (host == null) {
        if (lb != null) {
            Server svc = lb.chooseServer(loadBalancerKey);
            if (svc == null) {
                throw new ClientException(ErrorType.GENERAL, "Load balancer does not have available server for client: " + this.clientName);
            }

            host = svc.getHost();
            if (host == null) {
                throw new ClientException(ErrorType.GENERAL, "Invalid Server for :" + svc);
            }

            logger.debug("{} using LB returned Server: {} for request {}", new Object[]{this.clientName, svc, original});
            return svc;
        }

        if (this.vipAddresses != null && this.vipAddresses.contains(",")) {
            throw new ClientException(ErrorType.GENERAL, "Method is invoked for client " + this.clientName + " with partial URI of (" + original + ") with no load balancer configured. Also, there are multiple vipAddresses and hence no vip address can be chosen to complete this partial uri");
        }

        if (this.vipAddresses == null) {
            throw new ClientException(ErrorType.GENERAL, this.clientName + " has no LoadBalancer registered and passed in a partial URL request (with no host:port). Also has no vipAddress registered");
        }

        try {
            Pair<String, Integer> hostAndPort = this.deriveHostAndPortFromVipAddress(this.vipAddresses);
            host = (String)hostAndPort.first();
            port = (Integer)hostAndPort.second();
        } catch (URISyntaxException var8) {
            throw new ClientException(ErrorType.GENERAL, "Method is invoked for client " + this.clientName + " with partial URI of (" + original + ") with no load balancer configured.  Also, the configured/registered vipAddress is unparseable (to determine host and port)");
        }
    } else {
        boolean shouldInterpretAsVip = false;
        if (lb != null) {
            shouldInterpretAsVip = this.isVipRecognized(original.getAuthority());
        }

        if (shouldInterpretAsVip) {
            Server svc = lb.chooseServer(loadBalancerKey);
            if (svc != null) {
                host = svc.getHost();
                if (host == null) {
                    throw new ClientException(ErrorType.GENERAL, "Invalid Server for :" + svc);
                }

                logger.debug("using LB returned Server: {} for request: {}", svc, original);
                return svc;
            }

            logger.debug("{}:{} assumed to be a valid VIP address or exists in the DNS", host, port);
        } else {
            logger.debug("Using full URL passed in by caller (not using load balancer): {}", original);
        }
    }

    if (host == null) {
        throw new ClientException(ErrorType.GENERAL, "Request contains no HOST to talk to");
    } else {
        return new Server(host, port);
    }
}
```

```java
public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
      throws IOException {
   Request.Options options;
  //覆盖超时时间
   if (configOverride != null) {
      RibbonProperties override = RibbonProperties.from(configOverride);
      options = new Request.Options(
            override.connectTimeout(this.connectTimeout),
            override.readTimeout(this.readTimeout));
   }
   else {
      options = new Request.Options(this.connectTimeout, this.readTimeout);
   }
  //feign.Client.Default
   Response response = request.client().execute(request.toRequest(), options);
   return new RibbonResponse(request.getUri(), response);
}
```

feign.Client.Default#execute

```java
public Response execute(Request request, Options options) throws IOException {
  HttpURLConnection connection = convertAndSend(request, options);
  return convertResponse(connection).toBuilder().request(request).build();
}
```

```java
HttpURLConnection convertAndSend(Request request, Options options) throws IOException {
  final HttpURLConnection
      connection =
      (HttpURLConnection) new URL(request.url()).openConnection();
  if (connection instanceof HttpsURLConnection) {
    HttpsURLConnection sslCon = (HttpsURLConnection) connection;
    if (sslContextFactory != null) {
      sslCon.setSSLSocketFactory(sslContextFactory);
    }
    if (hostnameVerifier != null) {
      sslCon.setHostnameVerifier(hostnameVerifier);
    }
  }
  connection.setConnectTimeout(options.connectTimeoutMillis());
  connection.setReadTimeout(options.readTimeoutMillis());
  connection.setAllowUserInteraction(false);
  connection.setInstanceFollowRedirects(true);
  connection.setRequestMethod(request.method());

  Collection<String> contentEncodingValues = request.headers().get(CONTENT_ENCODING);
  boolean
      gzipEncodedRequest =
      contentEncodingValues != null && contentEncodingValues.contains(ENCODING_GZIP);
  boolean
      deflateEncodedRequest =
      contentEncodingValues != null && contentEncodingValues.contains(ENCODING_DEFLATE);

  boolean hasAcceptHeader = false;
  Integer contentLength = null;
  for (String field : request.headers().keySet()) {
    if (field.equalsIgnoreCase("Accept")) {
      hasAcceptHeader = true;
    }
    for (String value : request.headers().get(field)) {
      if (field.equals(CONTENT_LENGTH)) {
        if (!gzipEncodedRequest && !deflateEncodedRequest) {
          contentLength = Integer.valueOf(value);
          connection.addRequestProperty(field, value);
        }
      } else {
        connection.addRequestProperty(field, value);
      }
    }
  }
  // Some servers choke on the default accept string.
  if (!hasAcceptHeader) {
    connection.addRequestProperty("Accept", "*/*");
  }

  if (request.body() != null) {
    if (contentLength != null) {
      connection.setFixedLengthStreamingMode(contentLength);
    } else {
      connection.setChunkedStreamingMode(8196);
    }
    connection.setDoOutput(true);
    OutputStream out = connection.getOutputStream();
    if (gzipEncodedRequest) {
      out = new GZIPOutputStream(out);
    } else if (deflateEncodedRequest) {
      out = new DeflaterOutputStream(out);
    }
    try {
      out.write(request.body());
    } finally {
      try {
        out.close();
      } catch (IOException suppressed) { // NOPMD
      }
    }
  }
  return connection;
}
```

```java
Response convertResponse(HttpURLConnection connection) throws IOException {
  int status = connection.getResponseCode();
  String reason = connection.getResponseMessage();

  if (status < 0) {
    throw new IOException(format("Invalid status(%s) executing %s %s", status,
        connection.getRequestMethod(), connection.getURL()));
  }

  Map<String, Collection<String>> headers = new LinkedHashMap<String, Collection<String>>();
  for (Map.Entry<String, List<String>> field : connection.getHeaderFields().entrySet()) {
    // response message
    if (field.getKey() != null) {
      headers.put(field.getKey(), field.getValue());
    }
  }

  Integer length = connection.getContentLength();
  if (length == -1) {
    length = null;
  }
  InputStream stream;
  if (status >= 400) {
    stream = connection.getErrorStream();
  } else {
    stream = connection.getInputStream();
  }
  return Response.builder()
          .status(status)
          .reason(reason)
          .headers(headers)
          .body(stream, length)
          .build();
}
```

https://docs.spring.io/spring-cloud-openfeign/docs/3.1.8/reference/html/

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
    Request request = this.targetRequest(template);
    if (this.logLevel != Level.NONE) {
        this.logger.logRequest(this.metadata.configKey(), this.logLevel, request);
    }

    long start = System.nanoTime();

    Response response;
    try {
        response = this.client.execute(request, this.options);
    } catch (IOException var15) {
        if (this.logLevel != Level.NONE) {
            this.logger.logIOException(this.metadata.configKey(), this.logLevel, var15, this.elapsedTime(start));
        }

        throw FeignException.errorExecuting(request, var15);
    }

    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    boolean shouldClose = true;

    Response var9;
    try {
        if (this.logLevel != Level.NONE) {
            response = this.logger.logAndRebufferResponse(this.metadata.configKey(), this.logLevel, response, elapsedTime);
        }

        if (Response.class != this.metadata.returnType()) {
            Object result;
            Object var20;
            if (response.status() >= 200 && response.status() < 300) {
                if (Void.TYPE == this.metadata.returnType()) {
                    var9 = null;
                    return var9;
                }

                result = this.decode(response);
                shouldClose = this.closeAfterDecode;
                var20 = result;
                return var20;
            }

            if (this.decode404 && response.status() == 404 && Void.TYPE != this.metadata.returnType()) {
                result = this.decode(response);
                shouldClose = this.closeAfterDecode;
                var20 = result;
                return var20;
            }

            throw this.errorDecoder.decode(this.metadata.configKey(), response);
        }

        if (response.body() == null) {
            var9 = response;
            return var9;
        }

        if (response.body().length() != null && (long)response.body().length() <= 8192L) {
            byte[] bodyData = Util.toByteArray(response.body().asInputStream());
            Response var10 = response.toBuilder().body(bodyData).build();
            return var10;
        }

        shouldClose = false;
        var9 = response;
    } catch (IOException var16) {
        if (this.logLevel != Level.NONE) {
            this.logger.logIOException(this.metadata.configKey(), this.logLevel, var16, elapsedTime);
        }

        throw FeignException.errorReading(request, response, var16);
    } finally {
        if (shouldClose) {
            Util.ensureClosed(response.body());
        }

    }

    return var9;
}
```

feign.Client.Default#execute

```java
public Response execute(Request request, Options options) throws IOException {
    HttpURLConnection connection = this.convertAndSend(request, options);
    return this.convertResponse(connection, request);
}
```

```java
HttpURLConnection convertAndSend(Request request, Options options) throws IOException {
    HttpURLConnection connection = (HttpURLConnection)(new URL(request.url())).openConnection();
    if (connection instanceof HttpsURLConnection) {
        HttpsURLConnection sslCon = (HttpsURLConnection)connection;
        if (this.sslContextFactory != null) {
            sslCon.setSSLSocketFactory(this.sslContextFactory);
        }

        if (this.hostnameVerifier != null) {
            sslCon.setHostnameVerifier(this.hostnameVerifier);
        }
    }

    connection.setConnectTimeout(options.connectTimeoutMillis());
    connection.setReadTimeout(options.readTimeoutMillis());
    connection.setAllowUserInteraction(false);
    connection.setInstanceFollowRedirects(options.isFollowRedirects());
    connection.setRequestMethod(request.httpMethod().name());
    Collection<String> contentEncodingValues = (Collection)request.headers().get("Content-Encoding");
    boolean gzipEncodedRequest = contentEncodingValues != null && contentEncodingValues.contains("gzip");
    boolean deflateEncodedRequest = contentEncodingValues != null && contentEncodingValues.contains("deflate");
    boolean hasAcceptHeader = false;
    Integer contentLength = null;
    Iterator var9 = request.headers().keySet().iterator();

    while(var9.hasNext()) {
        String field = (String)var9.next();
        if (field.equalsIgnoreCase("Accept")) {
            hasAcceptHeader = true;
        }

        Iterator var11 = ((Collection)request.headers().get(field)).iterator();

        while(var11.hasNext()) {
            String value = (String)var11.next();
            if (field.equals("Content-Length")) {
                if (!gzipEncodedRequest && !deflateEncodedRequest) {
                    contentLength = Integer.valueOf(value);
                    connection.addRequestProperty(field, value);
                }
            } else {
                connection.addRequestProperty(field, value);
            }
        }
    }

    if (!hasAcceptHeader) {
        connection.addRequestProperty("Accept", "*/*");
    }

    if (request.body() != null) {
        if (contentLength != null) {
            connection.setFixedLengthStreamingMode(contentLength);
        } else {
            connection.setChunkedStreamingMode(8196);
        }

        connection.setDoOutput(true);
        OutputStream out = connection.getOutputStream();
        if (gzipEncodedRequest) {
            out = new GZIPOutputStream((OutputStream)out);
        } else if (deflateEncodedRequest) {
            out = new DeflaterOutputStream((OutputStream)out);
        }

        try {
            ((OutputStream)out).write(request.body());
        } finally {
            try {
                ((OutputStream)out).close();
            } catch (IOException var18) {
            }

        }
    }

    return connection;
}
```

```java
    Response convertResponse(HttpURLConnection connection, Request request) throws IOException {
        int status = connection.getResponseCode();
        String reason = connection.getResponseMessage();
        if (status < 0) {
            throw new IOException(String.format("Invalid status(%s) executing %s %s", status, connection.getRequestMethod(), connection.getURL()));
        } else {
            Map<String, Collection<String>> headers = new LinkedHashMap();
            Iterator var6 = connection.getHeaderFields().entrySet().iterator();

            while(var6.hasNext()) {
                Entry<String, List<String>> field = (Entry)var6.next();
                if (field.getKey() != null) {
                    headers.put((String)field.getKey(), (Collection)field.getValue());
                }
            }

            Integer length = connection.getContentLength();
            if (length == -1) {
                length = null;
            }

            InputStream stream;
            if (status >= 400) {
                stream = connection.getErrorStream();
            } else {
                stream = connection.getInputStream();
            }

            return Response.builder().status(status).reason(reason).headers(headers).request(request).body(stream, length).build();
        }
    }
}
```

# 官方文档总结

## 如何引入feign

引入org.springframework.cloud、spring-cloud-starter-openfeign

```java
@SpringBootApplication
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
@FeignClient("stores")
public interface StoreClient {
  @RequestMapping(method = RequestMethod.GET, value = "/stores")
  List<Store> getStores();

  @RequestMapping(method = RequestMethod.GET, value = "/stores")
  Page<Store> getStores(Pageable pageable);

  @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
  Store update(@PathVariable("storeId") Long storeId, Store store);

  @RequestMapping(method = RequestMethod.DELETE, value = "/stores/{storeId:\\d+}")
  void delete(@PathVariable Long storeId);
}
```

@FeignClient中的值会用来创建Spring Cloud LoadBalancer client，也可以指定url的值

Spring Cloud LoadBalancer client用来发现stores service的物理地址；如果没有注册中心，可以在配置文件中指定物理地址

比如

```properties
#service1 为服务名称
spring.cloud.discovery.client.simple.instances.service1[0].uri=http://s11:8080
```

@EnableFeignClients指定basePackages或者clients的值

## 覆盖默认配置

```java
//在未配置configuration时，所有的feignClient都使用默认的配置信息
//通过configuration可以设置自己的配置信息
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

```yaml
#@EnableFeignClients中的属性defaultConfiguration可以指定默认的全局配置信息
#获取配置信息时，如果自己没有，使用全局的
feign:
    client:
        config:
            feignName: #@FeignClient中的value的值
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: full
                errorDecoder: com.example.SimpleErrorDecoder
                retryer: com.example.SimpleRetryer
                defaultQueryParameters:
                    query: queryValue
                defaultRequestHeaders:
                    header: headerValue
                requestInterceptors:
                    - com.example.FooRequestInterceptor
                    - com.example.BarRequestInterceptor
                decode404: false
                encoder: com.example.SimpleEncoder
                decoder: com.example.SimpleDecoder
                contract: com.example.SimpleContract
                capabilities:
                    - com.example.FooCapability
                    - com.example.BarCapability
                queryMapEncoder: com.example.SimpleQueryMapEncoder
                metrics.enabled: false
```

```yaml
#全局默认配置信息
feign:
    client:
        config:
            default:
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: basic
```

```properties
feign.client.config.feignName.defaultQueryParameters=
feign.client.config.feignName.defaultRequestHeaders=
```

注意：如果既通过@Configuration也通过文件配置了属性了信息，优先使用文件中的信息。如果想改变优先级，可以设置`feign.client.default-to-properties` 为false

## 配置断路器

```yaml
feign:
  circuitbreaker:
    enabled: true
    alphanumeric-ids:
      enabled: true
resilience4j:
  circuitbreaker:
    instances:
      DemoClientgetDemo:
        minimumNumberOfCalls: 69
  timelimiter:
    instances:
      DemoClientgetDemo:
        timeoutDuration: 10s
```



## 降级方法

```java
@FeignClient(name = "test", url = "http://localhost:${server.port}/", fallback = Fallback.class)
protected interface TestClient {

  @RequestMapping(method = RequestMethod.GET, value = "/hello")
  Hello getHello();

  @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
  String getException();

}

@Component
static class Fallback implements TestClient { //降级类

  @Override
  public Hello getHello() {
    throw new NoFallbackAvailableException("Boom!", new RuntimeException());
  }

  @Override
  public String getException() {
    return "Fixed response";
  }
}
```

```java
@FeignClient(name = "testClientWithFactory", url = "http://localhost:${server.port}/",
            fallbackFactory = TestFallbackFactory.class) //降级工厂
protected interface TestClientWithFactory {

  @RequestMapping(method = RequestMethod.GET, value = "/hello")
  Hello getHello();

  @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
  String getException();

}

@Component
static class TestFallbackFactory implements FallbackFactory<FallbackWithFactory> {

  @Override
  public FallbackWithFactory create(Throwable cause) {
    return new FallbackWithFactory();
  }

}

static class FallbackWithFactory implements TestClientWithFactory {

  @Override
  public Hello getHello() {
    throw new NoFallbackAvailableException("Boom!", new RuntimeException());
  }

  @Override
  public String getException() {
    return "Fixed response";
  }

}
```



## feign and @Primary

当使用Feign且启用CircuitBreaker时，ApplicationContext中有多个相同类型的bean。这将导致@Autowired无法工作，因为没有唯一一个bean或者也没有一个标记为primary的bean。为了解决这个问题，Spring Cloud OpenFeign将所有Feign实例标记为@Primary，这样Spring框架就知道要注入哪个bean。在某些情况下，这可能是不可取的。要关闭此行为，请将@FeignClient的主要属性设置为false

```java
@FeignClient(name = "hello", primary = false)
public interface HelloClient {
    // methods here
}
```



## 单继承支持

```java
public interface UserService {
    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```

```java
@RestController
public class UserResource implements UserService {
}
```

```java
@FeignClient("users")
public interface UserClient extends UserService {
}
```

@FeignClient标注的接口不应该在服务器和客户端之间共享，并且不再支持在类级别上用@RequestMapping注释

## 请求响应压缩

```properties
feign.compression.request.enabled=true
feign.compression.response.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

## 日志记录

```yml
logging.level.project.user.UserClient: DEBUG #每个客户端配置不同的级别
```

```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

