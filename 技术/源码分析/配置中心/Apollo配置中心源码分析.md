# Apollo源码分析

* [客户端](#客户端)
  * [@EnableApolloConfig](#enableapolloconfig)
    * [案例](#案例)
    * [剖析](#剖析)
      * [ApolloConfigRegistrar](#apolloconfigregistrar)
      * [PropertySourcesProcessor](#propertysourcesprocessor)
  * [apollo.bootstrap.enabled=true](#apollobootstrapenabledtrue)
      * [DefaultConfigPropertySourcesProcessorHelper](#defaultconfigpropertysourcesprocessorhelper)
      * [ApolloApplicationContextInitializer](#apolloapplicationcontextinitializer)
  * [拉取配置](#拉取配置)


# 客户端

## @EnableApolloConfig

在加载了BeanDefinition之后、Bean实例化之前加载配置文件

### 案例

```java
@Configuration
@EnableApolloConfig({"someNamespace","anotherNamespace"})//使用注解，根据namespace加载配置文件
public class AppConfig {
}
```

### 剖析

com.ctrip.framework.apollo.spring.annotation.EnableApolloConfig

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(ApolloConfigRegistrar.class)
public @interface EnableApolloConfig {
  String[] value() default {ConfigConsts.NAMESPACE_APPLICATION};
  int order() default Ordered.LOWEST_PRECEDENCE;
}
```

#### ApolloConfigRegistrar

**1、解析注解ApolloConfigRegistrar，获取设置的namespace**

**2、将解析的namespace添加到PropertySourcesProcessor的NAMESPACE_NAMES中**

**3、向BeanDefinitionRegistry中注册PropertySourcesProcessor、ApolloAnnotationProcessor、SpringValueProcessor、SpringValueDefinitionProcessor、ApolloJsonValueProcessor。**

**4、PropertySourcesProcessor负责根据namespace构建Config，封装成PropertySource，填充到ConfigurableEnvironment˙中**

**5、给Config添加AutoUpdateConfigChangeListener监听器，自动更新属性值**

```java
private ApolloConfigRegistrarHelper helper = ServiceBootstrap.loadPrimary(ApolloConfigRegistrarHelper.class);

@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  helper.registerBeanDefinitions(importingClassMetadata, registry);
}
```

com.ctrip.framework.apollo.spring.spi.DefaultApolloConfigRegistrarHelper#registerBeanDefinitions

```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  //获取EnableApolloConfig注解中的value，即用户配置的namespace
  AnnotationAttributes attributes = AnnotationAttributes  .fromMap(importingClassMetadata.getAnnotationAttributes(EnableApolloConfig.class.getName()));
  String[] namespaces = attributes.getStringArray("value");
  
  //将用户配置的NameSpace注入到PropertySourcesProcessor
  int order = attributes.getNumber("order");
  PropertySourcesProcessor.addNamespaces(Lists.newArrayList(namespaces), order);
  
  Map<String, Object> propertySourcesPlaceholderPropertyValues = new HashMap<>();
  propertySourcesPlaceholderPropertyValues.put("order", 0);
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(),
      PropertySourcesPlaceholderConfigurer.class, propertySourcesPlaceholderPropertyValues);
  //注册PropertySourcesProcessor
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, c.class.getName(),
      PropertySourcesProcessor.class);
  //拦截注解@ApolloConfig、@ApolloConfigChangeListener
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(),
      ApolloAnnotationProcessor.class);
  //拦截@Value注解的字段、方法，对其维护保存至SpringValueRegistry
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(),
      SpringValueProcessor.class);
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueDefinitionProcessor.class.getName(),
      SpringValueDefinitionProcessor.class);
  //拦截@ApolloJsonValue注解的字段、方法，封装成SpringValue保存至SpringValueRegistry
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloJsonValueProcessor.class.getName(),
      ApolloJsonValueProcessor.class);
}
```

#### PropertySourcesProcessor

**实现了PriorityOrdered接口，具有最高的优先级，相比其他的BeanFactoryPostProcessor，保证优先执行**

**实现了BeanFactoryPostProcessor接口，在Bean实例化之前根据Namespace加载配置文件**

**实现了EnvironmentAware接口，注入ConfigurableEnvironment**

**将加载的配置存放到ConfigurableEnvironment，在查找属性时优先从远程加载的配置中查找**

com.ctrip.framework.apollo.spring.config.PropertySourcesProcessor#addNamespaces

```java
//xml文件中配置的namespace或者EnableApolloConfig注解上标注的namespace
public static boolean addNamespaces(Collection<String> namespaces, int order) {
  return NAMESPACE_NAMES.putAll(order, namespaces);
}
```

```java
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {//在Bean的实例化之前执行
  //将配置信息封装成ConfigPropertySource，存放到Env
  initializePropertySources();
  //初始化自动更新属性的特性
  initializeAutoUpdatePropertiesFeature(beanFactory);
}
```

```java
private void initializePropertySources() {
  //已经初始化
  if (environment.getPropertySources().contains(PropertySourcesConstants.APOLLO_PROPERTY_SOURCE_NAME)) {
    //already initialized
    return;
  }
  CompositePropertySource composite = new CompositePropertySource(PropertySourcesConstants.APOLLO_PROPERTY_SOURCE_NAME);

  //sort by order asc 对namespace进行排序
  ImmutableSortedSet<Integer> orders = ImmutableSortedSet.copyOf(NAMESPACE_NAMES.keySet());
  Iterator<Integer> iterator = orders.iterator();
  while (iterator.hasNext()) {
    int order = iterator.next();
    for (String namespace : NAMESPACE_NAMES.get(order)) {
      //本地或者远程加载配置
      Config config = ConfigService.getConfig(namespace);
  composite.addPropertySource(configPropertySourceFactory.getConfigPropertySource(namespace, config));
    }
  }
  // clean up
  NAMESPACE_NAMES.clear();
  // ensure ApolloBootstrapPropertySources is still the first
  ensureBootstrapPropertyPrecedence(environment);
  if (CollectionUtils.isEmpty(composite.getPropertySources())) {
    return;
  }
  // add after the bootstrap property source or to the first
  if (environment.getPropertySources()
      .contains(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
    //将其加到APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME之后
    environment.getPropertySources()
        .addAfter(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME, composite);
  } else {
    //将其加到第一个位置
    environment.getPropertySources().addFirst(composite);
  }
}
```

```java
private void initializeAutoUpdatePropertiesFeature(ConfigurableListableBeanFactory beanFactory) { //自动更新bean的属性
  if (!configUtil.isAutoUpdateInjectedSpringPropertiesEnabled() ||
      !AUTO_UPDATE_INITIALIZED_BEAN_FACTORIES.add(beanFactory)) {
    return;
  }
  //监听配置的变更，自动更新Bean的属性值
  AutoUpdateConfigChangeListener autoUpdateConfigChangeListener = new AutoUpdateConfigChangeListener(
      environment, beanFactory);
  //configPropertySourceFactory维护所有namespace对应的Config
  //所有的Config注册AutoUpdateConfigChangeListener
  List<ConfigPropertySource> configPropertySources = configPropertySourceFactory.getAllConfigPropertySources();
  for (ConfigPropertySource configPropertySource : configPropertySources) {
    //所有的Config添加AutoUpdateConfigChangeListener监听器，自动更新Bean的属性值
    configPropertySource.addChangeListener(autoUpdateConfigChangeListener);
  }
}
```

## apollo.bootstrap.enabled=true

在springboot初始化时，加载配置文件

```java
@Configuration
@ConditionalOnProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED)//配置文件中此属性不为空
@ConditionalOnMissingBean(PropertySourcesProcessor.class) //尚未实例化
public class ApolloAutoConfiguration {
  @Bean
  public ConfigPropertySourcesProcessor configPropertySourcesProcessor() {
    return new ConfigPropertySourcesProcessor();
  }
}
```

#### DefaultConfigPropertySourcesProcessorHelper

com.ctrip.framework.apollo.spring.spi.DefaultConfigPropertySourcesProcessorHelper#postProcessBeanDefinitionRegistry

```java
//向BeanDefinitionRegistry注册PropertySourcesPlaceholderConfigurer、ApolloAnnotationProcessor、SpringValueProcessor、ApolloJsonValueProcessor
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
  Map<String, Object> propertySourcesPlaceholderPropertyValues = new HashMap<>();
  // to make sure the default PropertySourcesPlaceholderConfigurer's priority is higher than PropertyPlaceholderConfigurer
  propertySourcesPlaceholderPropertyValues.put("order", 0);

  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(),
      PropertySourcesPlaceholderConfigurer.class, propertySourcesPlaceholderPropertyValues);
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(),
      ApolloAnnotationProcessor.class);
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(),
      SpringValueProcessor.class);
  BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloJsonValueProcessor.class.getName(),
      ApolloJsonValueProcessor.class);

  processSpringValueDefinition(registry);
}
```

#### ApolloApplicationContextInitializer

在springboot初始化阶段就获取apollo配置，注入Config

com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer#initialize(org.springframework.context.ConfigurableApplicationContext)

```java
public void initialize(ConfigurableApplicationContext context) {
  ConfigurableEnvironment environment = context.getEnvironment();
  //判断是否启动时就初始化apollo配置
  if (!environment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED, Boolean.class, false)) {
    logger.debug("Apollo bootstrap config is not enabled for context {}, see property: ${{}}", context, PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED);
    return;
  }
  logger.debug("Apollo bootstrap config is enabled for context {}", context);
  initialize(environment);
}
```



```java
protected void initialize(ConfigurableEnvironment environment) {
  //判断是否已经初始化
  if (environment.getPropertySources().contains(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
    //already initialized
    return;
  }
  //获取需要拉取的namespaces
  String namespaces = environment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_NAMESPACES, ConfigConsts.NAMESPACE_APPLICATION);
  logger.debug("Apollo bootstrap namespaces: {}", namespaces);
  List<String> namespaceList = NAMESPACE_SPLITTER.splitToList(namespaces);

  //将拉取的apollo配置存储到env
  CompositePropertySource composite = new CompositePropertySource(PropertySourcesConstants.APOLLO_BOOTSTRAP_PROPERTY_SOURCE_NAME);
  for (String namespace : namespaceList) {
    //Config客户端拉取配置信息入口
    Config config = ConfigService.getConfig(namespace);
composite.addPropertySource(configPropertySourceFactory.getConfigPropertySource(namespace, config));
  }
  //远程拉取的配置具有最高的优先级
  //Add the given property source object with highest precedence.
  environment.getPropertySources().addFirst(composite);
}
```

```java
void initializeSystemProperty(ConfigurableEnvironment environment) {//从env中获取属性值填充到系统属性中
  for (String propertyName : APOLLO_SYSTEM_PROPERTIES) {
    fillSystemPropertyFromEnvironment(environment, propertyName);
  }
}
```

com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer#postProcessEnvironment

```java
public void postProcessEnvironment(ConfigurableEnvironment configurableEnvironment, SpringApplication springApplication) {
  // should always initialize system properties like app.id in the first place
  initializeSystemProperty(configurableEnvironment);
  Boolean eagerLoadEnabled = configurableEnvironment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_EAGER_LOAD_ENABLED, Boolean.class, false);
  //EnvironmentPostProcessor should not be triggered if you don't want Apollo Loading before Logging System Initialization
  if (!eagerLoadEnabled) { //是否需要在日志系统初始化之前就加载apollo配置
    return;
  }
  //初始化Apollo配置
  Boolean bootstrapEnabled = configurableEnvironment.getProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED, Boolean.class, false);
  if (bootstrapEnabled) {
    initialize(configurableEnvironment);
  }
}
```

## 拉取配置

1、 定时从ConfigService拉取配置

2 、通过定时长轮询，ConfigService将挂起请求，当有配置发生变更时，唤醒请求，将变更的配置项推送给客户端。如果没有配置发生变更，返回304

3、执行ConfigChangeListener

com.ctrip.framework.apollo.ConfigService#getConfig

```java
public static Config getConfig(String namespace) {
  return s_instance.getManager().getConfig(namespace);
}
```

com.ctrip.framework.apollo.internals.DefaultConfigManager#getConfig

```java
public Config getConfig(String namespace) {
  Config config = m_configs.get(namespace);
  if (config == null) {
    synchronized (this) {
      config = m_configs.get(namespace);
      if (config == null) {
        ConfigFactory factory = m_factoryManager.getFactory(namespace);
        config = factory.create(namespace);
        m_configs.put(namespace, config);
      }
    }
  }
  return config;
}
```

com.ctrip.framework.apollo.spi.DefaultConfigFactory#create

```java
public Config create(String namespace) {
  ConfigFileFormat format = determineFileFormat(namespace);
  if (ConfigFileFormat.isPropertiesCompatible(format)) {
    return new DefaultConfig(namespace, createPropertiesCompatibleFileConfigRepository(namespace, format));
  }
  return new DefaultConfig(namespace, createLocalConfigRepository(namespace));
}
```

```java
LocalFileConfigRepository createLocalConfigRepository(String namespace) {
  if (m_configUtil.isInLocalMode()) { //判断是否是本地模式
    logger.warn(
        "==== Apollo is in local mode! Won't pull configs from remote server for namespace {} ! ====",
        namespace);
    return new LocalFileConfigRepository(namespace);
  }
  //远程模式。需要从远端拉取配置
  return new LocalFileConfigRepository(namespace, createRemoteConfigRepository(namespace));
}
```

com.ctrip.framework.apollo.spi.DefaultConfigFactory#createRemoteConfigRepository

```java
public RemoteConfigRepository(String namespace) {
  m_namespace = namespace;
  m_configCache = new AtomicReference<>();
  m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
  m_httpUtil = ApolloInjector.getInstance(HttpUtil.class);
  //获取ConfigService
  m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
  //Http long polling service
  remoteConfigLongPollService = ApolloInjector.getInstance(RemoteConfigLongPollService.class);
  m_longPollServiceDto = new AtomicReference<>();
  m_remoteMessages = new AtomicReference<>();
  //限制配置信息的远程拉取
  m_loadConfigRateLimiter = RateLimiter.create(m_configUtil.getLoadConfigQPS());
  m_configNeedForceRefresh = new AtomicBoolean(true);
  //拉取失败的调度策略
  m_loadConfigFailSchedulePolicy = new ExponentialSchedulePolicy(m_configUtil.getOnErrorRetryInterval(),
      m_configUtil.getOnErrorRetryInterval() * 8);
  gson = new Gson();
  //从远程同步，从config service拉取需要的配信息
  this.trySync();
  //周期调度执行从config service拉取需要的配信息
  this.schedulePeriodicRefresh();
  //定时调度http long polling
  this.scheduleLongPollingRefresh();
}
```

com.ctrip.framework.apollo.internals.RemoteConfigRepository#sync

```java
protected synchronized void sync() {
  Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "syncRemoteConfig");
  try {
    //本地缓存获取配置
    ApolloConfig previous = m_configCache.get();
     //创建Http请求，从ConfigService拉取配置
    ApolloConfig current = loadApolloConfig();
    // 配置发生变化，更新本地缓存
    if (previous != current) { 
      logger.debug("Remote Config refreshed!");
      m_configCache.set(current);
      // 执行RepositoryChangeListener
      this.fireRepositoryChange(m_namespace, this.getConfig());
    }
    if (current != null) {
      Tracer.logEvent(String.format("Apollo.Client.Configs.%s", current.getNamespaceName()),
          current.getReleaseKey());
    }
    transaction.setStatus(Transaction.SUCCESS);
  } catch (Throwable ex) {
    transaction.setStatus(ex);
    throw ex;
  } finally {
    transaction.complete();
  }
}
```

```java
protected void fireRepositoryChange(String namespace, Properties newProperties) {
  for (RepositoryChangeListener listener : m_listeners) {
    try {
      listener.onRepositoryChange(namespace, newProperties);
    } catch (Throwable ex) {
      Tracer.logError(ex);
      logger.error("Failed to invoke repository change listener {}", listener.getClass(), ex);
    }
  }
}
```

```java
public synchronized void onRepositoryChange(String namespace, Properties newProperties) {
  if (newProperties.equals(m_configProperties.get())) {
    return;
  }
  ConfigSourceType sourceType = m_configRepository.getSourceType();
  Properties newConfigProperties = new Properties();
  newConfigProperties.putAll(newProperties);
  //计算配置属性的变更类型（新增、修改、删除）
  Map<String, ConfigChange> actualChanges = updateAndCalcConfigChanges(newConfigProperties, sourceType);
  //check double checked result
  if (actualChanges.isEmpty()) {
    return;
  }
  //如果ConfigChangeListener对ConfigChange感兴趣，就会触发用户自定义的ConfigChangeListener的onChange方法
  this.fireConfigChange(new ConfigChangeEvent(m_namespace, actualChanges));
  Tracer.logEvent("Apollo.Client.ConfigChanges", m_namespace);
}
```

执行用户自定义的ConfigChangeListener,还有自带的AutoUpdateConfigChangeListener

```java
protected void fireConfigChange(final ConfigChangeEvent changeEvent) {
  for (final ConfigChangeListener listener : m_listeners) {
    if (!isConfigChangeListenerInterested(listener, changeEvent)) { //listener是否对change event感兴趣
      continue;
    }
    m_executorService.submit(new Runnable() {
      @Override
      public void run() {
        String listenerName = listener.getClass().getName();
        Transaction transaction = Tracer.newTransaction("Apollo.ConfigChangeListener", listenerName);
        try { //AutoUpdateConfigChangeListener
          listener.onChange(changeEvent);
          transaction.setStatus(Transaction.SUCCESS);
        } catch (Throwable ex) {
          transaction.setStatus(ex);
          Tracer.logError(ex);
          logger.error("Failed to invoke config change listener {}", listenerName, ex);
        } finally {
          transaction.complete();
        }
      }
    });
  }
}
```

周期性同步配置

com.ctrip.framework.apollo.internals.RemoteConfigRepository#schedulePeriodicRefresh

```java
private void schedulePeriodicRefresh() {
  logger.debug("Schedule periodic refresh with interval: {} {}",
      m_configUtil.getRefreshInterval(), m_configUtil.getRefreshIntervalTimeUnit());
  m_executorService.scheduleAtFixedRate(
      new Runnable() {
        @Override
        public void run() {
          Tracer.logEvent("Apollo.ConfigService", String.format("periodicRefresh: %s", m_namespace));
          logger.debug("refresh config for namespace: {}", m_namespace);
          trySync();
          Tracer.logEvent("Apollo.Client.Version", Apollo.VERSION);
        }
      }, m_configUtil.getRefreshInterval(), m_configUtil.getRefreshInterval(),
      m_configUtil.getRefreshIntervalTimeUnit());
}
```

定时调度http long polling

```java
public boolean submit(String namespace, RemoteConfigRepository remoteConfigRepository) {
  boolean added = m_longPollNamespaces.put(namespace, remoteConfigRepository);
  m_notifications.putIfAbsent(namespace, INIT_NOTIFICATION_ID);
  if (!m_longPollStarted.get()) {
    startLongPolling();
  }
  return added;
}
```

com.ctrip.framework.apollo.internals.RemoteConfigLongPollService#startLongPolling

```java
private void startLongPolling() {
  if (!m_longPollStarted.compareAndSet(false, true)) {
    //already started
    return;
  }
  try {
    final String appId = m_configUtil.getAppId();
    final String cluster = m_configUtil.getCluster();
    final String dataCenter = m_configUtil.getDataCenter();
    final String secret = m_configUtil.getAccessKeySecret();
    final long longPollingInitialDelayInMills = m_configUtil.getLongPollingInitialDelayInMills();
    m_longPollingService.submit(new Runnable() {
      @Override
      public void run() {
        if (longPollingInitialDelayInMills > 0) {
          try {
            logger.debug("Long polling will start in {} ms.", longPollingInitialDelayInMills);
            TimeUnit.MILLISECONDS.sleep(longPollingInitialDelayInMills);
          } catch (InterruptedException e) {
            //ignore
          }
        }
        doLongPollingRefresh(appId, cluster, dataCenter, secret); //调度长轮询
      }
    });
  } catch (Throwable ex) {
    m_longPollStarted.set(false);
    ApolloConfigException exception =
        new ApolloConfigException("Schedule long polling refresh failed", ex);
    Tracer.logError(exception);
    logger.warn(ExceptionUtil.getDetailMessage(exception));
  }
}
```

```java
private void doLongPollingRefresh(String appId, String cluster, String dataCenter, String secret) {
  final Random random = new Random();
  ServiceDTO lastServiceDto = null;
  while (!m_longPollingStopped.get() && !Thread.currentThread().isInterrupted()) {
    if (!m_longPollRateLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
      //wait at most 5 seconds
      try {
        TimeUnit.SECONDS.sleep(5);
      } catch (InterruptedException e) {
      }
    }
    Transaction transaction = Tracer.newTransaction("Apollo.ConfigService", "pollNotification");
    String url = null;
    try {
      if (lastServiceDto == null) {
        List<ServiceDTO> configServices = getConfigServices();
        lastServiceDto = configServices.get(random.nextInt(configServices.size()));
      }
			//构建url
      url =
          assembleLongPollRefreshUrl(lastServiceDto.getHomepageUrl(), appId, cluster, dataCenter,
              m_notifications);

      logger.debug("Long polling from {}", url);

      HttpRequest request = new HttpRequest(url);
      request.setReadTimeout(LONG_POLLING_READ_TIMEOUT); //默认90秒
      if (!StringUtils.isBlank(secret)) {
        Map<String, String> headers = Signature.buildHttpHeaders(url, appId, secret);
        request.setHeaders(headers);
      }

      transaction.addData("Url", url);

      final HttpResponse<List<ApolloConfigNotification>> response =
          m_httpUtil.doGet(request, m_responseType);

      logger.debug("Long polling response: {}, url: {}", response.getStatusCode(), url);
      if (response.getStatusCode() == 200 && response.getBody() != null) {
        updateNotifications(response.getBody());
        updateRemoteNotifications(response.getBody());
        transaction.addData("Result", response.getBody().toString());
        notify(lastServiceDto, response.getBody());
      }

      //try to load balance
      if (response.getStatusCode() == 304 && random.nextBoolean()) {
        lastServiceDto = null;
      }

      m_longPollFailSchedulePolicyInSecond.success();
      transaction.addData("StatusCode", response.getStatusCode());
      transaction.setStatus(Transaction.SUCCESS);
    } catch (Throwable ex) {
      lastServiceDto = null;
      Tracer.logEvent("ApolloConfigException", ExceptionUtil.getDetailMessage(ex));
      transaction.setStatus(ex);
      long sleepTimeInSecond = m_longPollFailSchedulePolicyInSecond.fail();
      logger.warn(
          "Long polling failed, will retry in {} seconds. appId: {}, cluster: {}, namespaces: {}, long polling url: {}, reason: {}",
          sleepTimeInSecond, appId, cluster, assembleNamespaces(), url, ExceptionUtil.getDetailMessage(ex));
      try {
        TimeUnit.SECONDS.sleep(sleepTimeInSecond);
      } catch (InterruptedException ie) {
        //ignore
      }
    } finally {
      transaction.complete();
    }
  }
}
```

com.ctrip.framework.apollo.internals.RemoteConfigLongPollService#updateNotifications

```java
private void updateNotifications(List<ApolloConfigNotification> deltaNotifications) {
  for (ApolloConfigNotification notification : deltaNotifications) {
    if (Strings.isNullOrEmpty(notification.getNamespaceName())) {
      continue;
    }
    String namespaceName = notification.getNamespaceName();
    if (m_notifications.containsKey(namespaceName)) {
      m_notifications.put(namespaceName, notification.getNotificationId());
    }
    //since .properties are filtered out by default, so we need to check if there is notification with .properties suffix
    String namespaceNameWithPropertiesSuffix =
        String.format("%s.%s", namespaceName, ConfigFileFormat.Properties.getValue());
    if (m_notifications.containsKey(namespaceNameWithPropertiesSuffix)) {
      m_notifications.put(namespaceNameWithPropertiesSuffix, notification.getNotificationId());
    }
  }
}
```

com.ctrip.framework.apollo.internals.RemoteConfigLongPollService#updateRemoteNotifications

```java
private void updateRemoteNotifications(List<ApolloConfigNotification> deltaNotifications) {
  for (ApolloConfigNotification notification : deltaNotifications) {
    if (Strings.isNullOrEmpty(notification.getNamespaceName())) {
      continue;
    }

    if (notification.getMessages() == null || notification.getMessages().isEmpty()) {
      continue;
    }

    ApolloNotificationMessages localRemoteMessages =
        m_remoteNotificationMessages.get(notification.getNamespaceName());
    if (localRemoteMessages == null) {
      localRemoteMessages = new ApolloNotificationMessages();
      m_remoteNotificationMessages.put(notification.getNamespaceName(), localRemoteMessages);
    }

    localRemoteMessages.mergeFrom(notification.getMessages());
  }
}
```

com.ctrip.framework.apollo.internals.RemoteConfigLongPollService#notify

```java
private void notify(ServiceDTO lastServiceDto, List<ApolloConfigNotification> notifications) {
  if (notifications == null || notifications.isEmpty()) {
    return;
  }
  for (ApolloConfigNotification notification : notifications) {
    String namespaceName = notification.getNamespaceName();
    //create a new list to avoid ConcurrentModificationException
    List<RemoteConfigRepository> toBeNotified =
        Lists.newArrayList(m_longPollNamespaces.get(namespaceName));
    ApolloNotificationMessages originalMessages = m_remoteNotificationMessages.get(namespaceName);
    ApolloNotificationMessages remoteMessages = originalMessages == null ? null : originalMessages.clone();
    //since .properties are filtered out by default, so we need to check if there is any listener for it
    toBeNotified.addAll(m_longPollNamespaces
        .get(String.format("%s.%s", namespaceName, ConfigFileFormat.Properties.getValue())));
    for (RemoteConfigRepository remoteConfigRepository : toBeNotified) {
      try {
        //拉取发生变更的配置
        remoteConfigRepository.onLongPollNotified(lastServiceDto, remoteMessages);
      } catch (Throwable ex) {
        Tracer.logError(ex);
      }
    }
  }
}
```