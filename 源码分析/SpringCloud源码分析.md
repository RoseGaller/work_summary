# SpringCloud源码分析

* [EurekaServer](#eurekaserver)
  * [注册服务](#注册服务)
  * [服务续约](#服务续约)
* [EurekaClient](#eurekaclient)
  * [实例化Marker](#实例化marker)
  * [实例化EurekaClientConfigBean](#实例化eurekaclientconfigbean)
  * [实例化ApplicationInfoManager](#实例化applicationinfomanager)
  * [实例化EurekaClient](#实例化eurekaclient)
    * [拉取服务](#拉取服务)
    * [注册实例](#注册实例)
    * [初始化调度任务](#初始化调度任务)
      * [续约](#续约)
  * [下线服务](#下线服务)
* [Ribbon](#ribbon)
  * [组件](#组件)
    * [LoadBalanced](#loadbalanced)
    * [RestTemplates](#resttemplates)
    * [SmartInitializingSingleton](#smartinitializingsingleton)
    * [LoadBalancerInterceptor](#loadbalancerinterceptor)
    * [RestTemplateCustomizer](#resttemplatecustomizer)
    * [LoadBalancerRequestFactory](#loadbalancerrequestfactory)
  * [执行流程](#执行流程)
* [Feign](#feign)
  * [组件](#组件-1)
    * [EnableFeignClients](#enablefeignclients)
    * [FeignClientsRegistrar](#feignclientsregistrar)
    * [FeignClientSpecification](#feignclientspecification)
    * [FeignClientFactoryBean](#feignclientfactorybean)
  * [获取代理对象](#获取代理对象)

# EurekaServer

1、打开开发工具，创建一个 Spring Cloud 的项目，然后在 pom 中增加 spring-cloud-starter-netflix-eureka-server 的依赖

2、创建一个 EurekaServerApplication 的启动类，启动类上使用@EnableEurekaServer 开启 EurekaServer 的自动装配功能

3、 需要配置 Eureka Server 需要的信息，端口配置成 8761，

添加一个 eureka.client.register-with-eureka=false 的配置，本身是 Eureka Server 节点，不需要将自己进行注册。

再添加一个 eureka.client.fetch-registry=false 的配置，这里也设置成 false，因为不需要消费其他服务信息，所以也不需要拉取注册表信息

4、启动项目，然后访问 8761 端口，可以看到 Eureka 的管理页面，表示 Eureka 启动成功了

springboot应用启动时会从spring.factories文件中加载EurekaServerAutoConfiguration自动配置类

需要有一个marker bean，才能装配Eureka Server，marker bean 其实是由@EnableEurekaServer注解决定的

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguratio
```

在启动上标注@EnableEurekaServer注解,

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```

```java
@Configuration
public class EurekaServerMarkerConfiguration {

  //注入Marker，表明EurekaServer
   @Bean
   public Marker eurekaServerMarkerBean() {
      return new Marker();
   }

   class Marker {
   }
}
```



org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration#eurekaController

```java
@Bean
@ConditionalOnProperty(
    prefix = "eureka.dashboard",
    name = {"enabled"},
    matchIfMissing = true
)
public EurekaController eurekaController() {
    return new EurekaController(this.applicationInfoManager);
}
```

## 注册服务

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#register

```java
public void register(InstanceInfo info, boolean isReplication) {
    int leaseDuration = 90; //服务时效间隔
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
		//注册实例
    super.register(info, leaseDuration, isReplication); 
  	//复制实例到其他节点
    this.replicateToPeers(PeerAwareInstanceRegistryImpl.Action.Register, info.getAppName(), info.getId(), info, (InstanceStatus)null, isReplication);
}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#register

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try {
        this.read.lock();
        Map<String, Lease<InstanceInfo>> gMap = (Map)this.registry.get(registrant.getAppName());
        EurekaMonitors.REGISTER.increment(isReplication);
        if (gMap == null) {
            ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap();
            gMap = (Map)this.registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }

        Lease<InstanceInfo> existingLease = (Lease)((Map)gMap).get(registrant.getId());
        if (existingLease != null && existingLease.getHolder() != null) {
            Long existingLastDirtyTimestamp = ((InstanceInfo)existingLease.getHolder()).getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = (InstanceInfo)existingLease.getHolder();
            }
        } else {
            synchronized(this.lock) {
                if (this.expectedNumberOfRenewsPerMin > 0) {
                    this.expectedNumberOfRenewsPerMin += 2;
                    this.numberOfRenewsPerMinThreshold = (int)((double)this.expectedNumberOfRenewsPerMin * this.serverConfig.getRenewalPercentThreshold());
                }
            }

            logger.debug("No previous lease information found; it is new registration");
        }

        Lease<InstanceInfo> lease = new Lease(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }

        ((Map)gMap).put(registrant.getId(), lease);
        synchronized(this.recentRegisteredQueue) { //存放最近注册的服务
            this.recentRegisteredQueue.add(new Pair(System.currentTimeMillis(), registrant.getAppName() + "(" + registrant.getId() + ")"));
        }

        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!this.overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                this.overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }

        InstanceStatus overriddenStatusFromMap = (InstanceStatus)this.overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);
        if (InstanceStatus.UP.equals(registrant.getStatus())) { //上线
            lease.serviceUp();
        }
        registrant.setActionType(ActionType.ADDED);
        this.recentlyChangedQueue.add(new AbstractInstanceRegistry.RecentlyChangedItem(lease));
        registrant.setLastUpdatedTimestamp();
        this.invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})", new Object[]{registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication});
    } finally {
        this.read.unlock();
    }

}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#replicateToPeers

```java
private void replicateToPeers(PeerAwareInstanceRegistryImpl.Action action, String appName, String id, InstanceInfo info, InstanceStatus newStatus, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            this.numberOfReplicationsLastMin.increment();
        }
        if (this.peerEurekaNodes != Collections.EMPTY_LIST && !isReplication) {
          //获取集群所有节点
            Iterator var8 = this.peerEurekaNodes.getPeerEurekaNodes().iterator();
            while(var8.hasNext()) {
                PeerEurekaNode node = (PeerEurekaNode)var8.next();
                if (!this.peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {//过滤本机
                    this.replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node); // 复制服务实例操作到node节点
                }
            }
            return;
        }
    } finally {
        tracer.stop();
    }
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#replicateInstanceActionsToPeers

```java
private void replicateInstanceActionsToPeers(PeerAwareInstanceRegistryImpl.Action action, String appName, String id, InstanceInfo info, InstanceStatus newStatus, PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        switch(action) {
        case Cancel: // 服务下线
            node.cancel(appName, id);
            break;
        case Heartbeat: //续约
            InstanceStatus overriddenStatus = (InstanceStatus)this.overriddenInstanceStatusMap.get(id);
            infoFromRegistry = this.getInstanceByAppAndId(appName, id, false);
            node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
            break;
        case Register: //服务上线
            node.register(info);
            break;
        case StatusUpdate:
            infoFromRegistry = this.getInstanceByAppAndId(appName, id, false);
            node.statusUpdate(appName, id, newStatus, infoFromRegistry);
            break;
        case DeleteStatusOverride:
            infoFromRegistry = this.getInstanceByAppAndId(appName, id, false);
            node.deleteStatusOverride(appName, id, infoFromRegistry);
        }
    } catch (Throwable var9) {
        logger.error("Cannot replicate information to {} for action {}", new Object[]{node.getServiceUrl(), action.name(), var9});
    }

}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#getApplication(java.lang.String)

```java
public Application getApplication(String appName) {
    boolean disableTransparentFallback = this.serverConfig.disableTransparentFallbackToOtherRegion();
    return this.getApplication(appName, !disableTransparentFallback);
}
```

```java
public Application getApplication(String appName, boolean includeRemoteRegion) {
    Application app = null;
    Map<String, Lease<InstanceInfo>> leaseMap = (Map)this.registry.get(appName);
    Iterator var5;
    Entry entry;
    if (leaseMap != null && leaseMap.size() > 0) {
        for(var5 = leaseMap.entrySet().iterator(); var5.hasNext(); app.addInstance(this.decorateInstanceInfo((Lease)entry.getValue()))) {
            entry = (Entry)var5.next();
            if (app == null) {
                app = new Application(appName);
            }
        }
    } else if (includeRemoteRegion) {
        var5 = this.regionNameVSRemoteRegistry.values().iterator();

        while(var5.hasNext()) {
            RemoteRegionRegistry remoteRegistry = (RemoteRegionRegistry)var5.next();
            Application application = remoteRegistry.getApplication(appName);
            if (application != null) {
                return application;
            }
        }
    }

    return app;
}
```

## 服务续约

com.netflix.eureka.resources.InstanceResource#renewLease

```java
@PUT
public Response renewLease(@HeaderParam("x-netflix-discovery-replication") String isReplication, @QueryParam("overriddenstatus") String overriddenStatus, @QueryParam("status") String status, @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    boolean isFromReplicaNode = "true".equals(isReplication);
		//续约
    boolean isSuccess = this.registry.renew(this.app.getName(), this.id, isFromReplicaNode);
    if (!isSuccess) {
        logger.warn("Not Found (Renew): {} - {}", this.app.getName(), this.id);
        return Response.status(Status.NOT_FOUND).build();
    } else {
        Response response = null;
        if (lastDirtyTimestamp != null && this.serverConfig.shouldSyncWhenTimestampDiffers()) {
            response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
            if (response.getStatus() == Status.NOT_FOUND.getStatusCode() && overriddenStatus != null && !InstanceStatus.UNKNOWN.name().equals(overriddenStatus) && isFromReplicaNode) {
                this.registry.storeOverriddenStatusIfRequired(this.app.getAppName(), this.id, InstanceStatus.valueOf(overriddenStatus));
            }
        } else {
            response = Response.ok().build();
        }

        logger.debug("Found (Renew): {} - {}; reply status={}" + this.app.getName(), this.id, response.getStatus());
        return response;
    }
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#renew

```java
public boolean renew(String appName, String id, boolean isReplication) {
  		 //本机续约
    if (super.renew(appName, id, isReplication)) {
       //集群其他节点续约
        this.replicateToPeers(PeerAwareInstanceRegistryImpl.Action.Heartbeat, appName, id, (InstanceInfo)null, (InstanceStatus)null, isReplication);
        return true;
    } else {
        return false;
    }
}
```

本机续约

com.netflix.eureka.registry.AbstractInstanceRegistry#renew

```java
public boolean renew(String appName, String id, boolean isReplication) {
    EurekaMonitors.RENEW.increment(isReplication);
    Map<String, Lease<InstanceInfo>> gMap = (Map)this.registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    if (gMap != null) {
        leaseToRenew = (Lease)gMap.get(id);
    }
		//该实例不存在
    if (leaseToRenew == null) {
        EurekaMonitors.RENEW_NOT_FOUND.increment(isReplication);
        logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
        return false;
    } else {//存在
        InstanceInfo instanceInfo = (InstanceInfo)leaseToRenew.getHolder();
        if (instanceInfo != null) {
            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(instanceInfo, leaseToRenew, isReplication);
            if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}; re-register required", instanceInfo.getId());
                EurekaMonitors.RENEW_NOT_FOUND.increment(isReplication);
                return false;
            }

            if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                logger.info("The instance status {} is different from overridden instance status {} for instance {}. Hence setting the status to overridden status", new Object[]{instanceInfo.getStatus().name(), instanceInfo.getOverriddenStatus().name(), instanceInfo.getId()});
                instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);
            }
        }

        this.renewsLastMin.increment();
        //更新最后的修改时间戳
        leaseToRenew.renew();
        return true;
    }
}
```



# EurekaClient

EurekaClientAutoConfiguration

## 实例化Marker

org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration

```java
@Bean
public EurekaDiscoveryClientConfiguration.Marker eurekaDiscoverClientMarker() {
    return new EurekaDiscoveryClientConfiguration.Marker();
}
```

## 实例化EurekaClientConfigBean

读取配置

```java
@Bean
@ConditionalOnMissingBean(
    value = {EurekaClientConfig.class},
    search = SearchStrategy.CURRENT
)
public EurekaClientConfigBean eurekaClientConfigBean(ConfigurableEnvironment env) {
    EurekaClientConfigBean client = new EurekaClientConfigBean();
    if ("bootstrap".equals(this.env.getProperty("spring.config.name"))) {
        client.setRegisterWithEureka(false);
    }

    return client;
}
```

```java
@Bean
@ConditionalOnMissingBean(
    value = {EurekaInstanceConfig.class},
    search = SearchStrategy.CURRENT
)
public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils, ManagementMetadataProvider managementMetadataProvider) {
    String hostname = this.getProperty("eureka.instance.hostname");
    boolean preferIpAddress = Boolean.parseBoolean(this.getProperty("eureka.instance.prefer-ip-address"));
    String ipAddress = this.getProperty("eureka.instance.ip-address");
    boolean isSecurePortEnabled = Boolean.parseBoolean(this.getProperty("eureka.instance.secure-port-enabled"));
    String serverContextPath = this.env.getProperty("server.context-path", "/");
    int serverPort = Integer.valueOf(this.env.getProperty("server.port", this.env.getProperty("port", "8080")));
    Integer managementPort = (Integer)this.env.getProperty("management.server.port", Integer.class);
    String managementContextPath = this.env.getProperty("management.server.servlet.context-path");
    Integer jmxPort = (Integer)this.env.getProperty("com.sun.management.jmxremote.port", Integer.class);
    EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);
    instance.setNonSecurePort(serverPort);
    instance.setInstanceId(IdUtils.getDefaultInstanceId(this.env));
    instance.setPreferIpAddress(preferIpAddress);
    instance.setSecurePortEnabled(isSecurePortEnabled);
    if (StringUtils.hasText(ipAddress)) {
        instance.setIpAddress(ipAddress);
    }

    if (isSecurePortEnabled) {
        instance.setSecurePort(serverPort);
    }

    if (StringUtils.hasText(hostname)) {
        instance.setHostname(hostname);
    }

    String statusPageUrlPath = this.getProperty("eureka.instance.status-page-url-path");
    String healthCheckUrlPath = this.getProperty("eureka.instance.health-check-url-path");
    if (StringUtils.hasText(statusPageUrlPath)) {
        instance.setStatusPageUrlPath(statusPageUrlPath);
    }

    if (StringUtils.hasText(healthCheckUrlPath)) {
        instance.setHealthCheckUrlPath(healthCheckUrlPath);
    }

    ManagementMetadata metadata = managementMetadataProvider.get(instance, serverPort, serverContextPath, managementContextPath, managementPort);
    if (metadata != null) {
        instance.setStatusPageUrl(metadata.getStatusPageUrl());
        instance.setHealthCheckUrl(metadata.getHealthCheckUrl());
        if (instance.isSecurePortEnabled()) {
            instance.setSecureHealthCheckUrl(metadata.getSecureHealthCheckUrl());
        }

        Map<String, String> metadataMap = instance.getMetadataMap();
        if (metadataMap.get("management.port") == null) {
            metadataMap.put("management.port", String.valueOf(metadata.getManagementPort()));
        }
    } else if (StringUtils.hasText(managementContextPath)) {
        instance.setHealthCheckUrlPath(managementContextPath + instance.getHealthCheckUrlPath());
        instance.setStatusPageUrlPath(managementContextPath + instance.getStatusPageUrlPath());
    }

    this.setupJmxPort(instance, jmxPort);
    return instance;
}
```

## 实例化ApplicationInfoManager

```java
@Bean
@ConditionalOnMissingBean(
    value = {ApplicationInfoManager.class},
    search = SearchStrategy.CURRENT
)
public ApplicationInfoManager eurekaApplicationInfoManager(EurekaInstanceConfig config) {
    //创建实例信息
    InstanceInfo instanceInfo = (new InstanceInfoFactory()).create(config);
    return new ApplicationInfoManager(config, instanceInfo);
}
```

## 实例化EurekaClient

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

   /**
    * If true, the ServiceRegistry will automatically register the local server.
    */
   boolean autoRegister() default true;
}
```

org.springframework.cloud.client.discovery.EnableDiscoveryClientImportSelector#selectImports

```java
public String[] selectImports(AnnotationMetadata metadata) {
   String[] imports = super.selectImports(metadata);

   AnnotationAttributes attributes = AnnotationAttributes.fromMap(
         metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));

   boolean autoRegister = attributes.getBoolean("autoRegister");

  //自动注册
   if (autoRegister) {
      List<String> importsList = new ArrayList<>(Arrays.asList(imports));
      importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
      imports = importsList.toArray(new String[0]);
   } else {
      Environment env = getEnvironment();
      if(ConfigurableEnvironment.class.isInstance(env)) {
         ConfigurableEnvironment configEnv = (ConfigurableEnvironment)env;
         LinkedHashMap<String, Object> map = new LinkedHashMap<>();
         map.put("spring.cloud.service-registry.auto-registration.enabled", false);
         MapPropertySource propertySource = new MapPropertySource(
               "springCloudDiscoveryClient", map);
         configEnv.getPropertySources().addLast(propertySource);
      }

   }

   return imports;
}
```

org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration

```java
@Configuration
@Import(AutoServiceRegistrationConfiguration.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public class AutoServiceRegistrationAutoConfiguration {

   @Autowired(required = false)
   private AutoServiceRegistration autoServiceRegistration;

   @Autowired
   private AutoServiceRegistrationProperties properties;

   @PostConstruct
   protected void init() {
     //自动服务注册不能为空
      if (autoServiceRegistration == null && this.properties.isFailFast()) {
         throw new IllegalStateException("Auto Service Registration has been requested, but there is no AutoServiceRegistration bean");
      }
   }
}
```

```java
@Bean(
    destroyMethod = "shutdown"
)
@ConditionalOnMissingBean(
    value = {EurekaClient.class},
    search = SearchStrategy.CURRENT
)
public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config) {
    return new CloudEurekaClient(manager, config, this.optionalArgs, this.context);
}
```

### 拉取服务

```java
if (this.clientConfig.shouldFetchRegistry() && !this.fetchRegistry(false)) {//从注册中心获取信息
    this.fetchRegistryFromBackup();
}
```

com.netflix.discovery.DiscoveryClient#fetchRegistry

```java
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = this.FETCH_REGISTRY_TIMER.start();

    label122: {
        boolean var4;
        try {
            Applications applications = this.getApplications();//本地缓存获取
            if (!this.clientConfig.shouldDisableDelta() && Strings.isNullOrEmpty(this.clientConfig.getRegistryRefreshSingleVipAddress()) && !forceFullRegistryFetch && applications != null && applications.getRegisteredApplications().size() != 0 && applications.getVersion() != -1L) {
                this.getAndUpdateDelta(applications); //增量获取
            } else {
                logger.info("Disable delta property : {}", this.clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", this.clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", applications == null);
                logger.info("Registered Applications size is zero : {}", applications.getRegisteredApplications().size() == 0);
                logger.info("Application version is -1: {}", applications.getVersion() == -1L);
                this.getAndStoreFullRegistry(); //全量获取
            }

            applications.setAppsHashCode(applications.getReconcileHashCode());
            this.logTotalInstances();
            break label122;
        } catch (Throwable var8) {
            logger.error("DiscoveryClient_{} - was unable to refresh its cache! status = {}", new Object[]{this.appPathIdentifier, var8.getMessage(), var8});
            var4 = false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }

        }

        return var4;
    }

    this.onCacheRefreshed();
    this.updateInstanceRemoteStatus();
    return true;
}
```

### 注册实例

```java
if (this.clientConfig.shouldRegisterWithEureka() && this.clientConfig.shouldEnforceRegistrationAtInit()) {
    try {
        if (!this.register()) {
            throw new IllegalStateException("Registration error at startup. Invalid server response.");
        }
    } catch (Throwable var8) {
        logger.error("Registration error at startup: {}", var8.getMessage());
        throw new IllegalStateException(var8);
    }
}
```

com.netflix.discovery.DiscoveryClient#register

```java
boolean register() throws Throwable {
    logger.info("DiscoveryClient_{}: registering service...", this.appPathIdentifier);

    EurekaHttpResponse httpResponse;
    try {
      //向EurekServer注册Instance
        httpResponse = this.eurekaTransport.registrationClient.register(this.instanceInfo);
    } catch (Exception var3) {
        logger.warn("DiscoveryClient_{} - registration failed {}", new Object[]{this.appPathIdentifier, var3.getMessage(), var3});
        throw var3;
    }

    if (logger.isInfoEnabled()) {
        logger.info("DiscoveryClient_{} - registration status: {}", this.appPathIdentifier, httpResponse.getStatusCode());
    }

    return httpResponse.getStatusCode() == 204;
}
```

### 初始化调度任务

com.netflix.discovery.DiscoveryClient#initScheduledTasks

```java
private void initScheduledTasks() {
    int renewalIntervalInSecs;
    int expBackOffBound;
  //定时刷新本地缓存的服务
    if (this.clientConfig.shouldFetchRegistry()) {
        renewalIntervalInSecs = this.clientConfig.getRegistryFetchIntervalSeconds();
        expBackOffBound = this.clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        this.scheduler.schedule(new TimedSupervisorTask("cacheRefresh", this.scheduler, this.cacheRefreshExecutor, renewalIntervalInSecs, TimeUnit.SECONDS, expBackOffBound, new DiscoveryClient.CacheRefreshThread()), (long)renewalIntervalInSecs, TimeUnit.SECONDS);
    }
	//定时续约
    if (this.clientConfig.shouldRegisterWithEureka()) {
        renewalIntervalInSecs = this.instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        expBackOffBound = this.clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: renew interval is: {}", renewalIntervalInSecs);
        this.scheduler.schedule(new TimedSupervisorTask("heartbeat", this.scheduler, this.heartbeatExecutor, renewalIntervalInSecs, TimeUnit.SECONDS, expBackOffBound, new DiscoveryClient.HeartbeatThread()), (long)renewalIntervalInSecs, TimeUnit.SECONDS);
        this.instanceInfoReplicator = new InstanceInfoReplicator(this, this.instanceInfo, this.clientConfig.getInstanceInfoReplicationIntervalSeconds(), 2);
        this.statusChangeListener = new StatusChangeListener() {
            public String getId() {
                return "statusChangeListener";
            }

            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN != statusChangeEvent.getStatus() && InstanceStatus.DOWN != statusChangeEvent.getPreviousStatus()) {
                    DiscoveryClient.logger.info("Saw local status change event {}", statusChangeEvent);
                } else {
                    DiscoveryClient.logger.warn("Saw local status change event {}", statusChangeEvent);
                }

                DiscoveryClient.this.instanceInfoReplicator.onDemandUpdate();
            }
        };
        if (this.clientConfig.shouldOnDemandUpdateStatusChange()) {
            this.applicationInfoManager.registerStatusChangeListener(this.statusChangeListener);
        }

        this.instanceInfoReplicator.start(this.clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }

}
```

com.netflix.discovery.DiscoveryClient.CacheRefreshThread

```java
public void run() {
    DiscoveryClient.this.refreshRegistry();
}
```

#### 续约

com.netflix.discovery.DiscoveryClient.HeartbeatThread

```java
public void run() {
    //续约
    if (DiscoveryClient.this.renew()) {
        //设置续约的时间
        DiscoveryClient.this.lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
    }
}
```

com.netflix.discovery.DiscoveryClient#renew

```java
boolean renew() {
    try {
        //发送续约请求
        EurekaHttpResponse<InstanceInfo> httpResponse = this.eurekaTransport.registrationClient.sendHeartBeat(this.instanceInfo.getAppName(), this.instanceInfo.getId(), this.instanceInfo, (InstanceStatus)null);
        logger.debug("DiscoveryClient_{} - Heartbeat status: {}", this.appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == 404) {//注册中心不存在此实例
            this.REREGISTER_COUNTER.increment();
            logger.info("DiscoveryClient_{} - Re-registering apps/{}", this.appPathIdentifier, this.instanceInfo.getAppName());
            long timestamp = this.instanceInfo.setIsDirtyWithTime();
            boolean success = this.register(); //注册
            if (success) {
                this.instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        } else {
            return httpResponse.getStatusCode() == 200;
        }
    } catch (Throwable var5) {
        logger.error("DiscoveryClient_{} - was unable to send heartbeat!", this.appPathIdentifier, var5);
        return false;
    }
}
```

## 下线服务

当容器关闭时，调用DiscoveryClient的shutdown方法

com.netflix.discovery.DiscoveryClient#unregister

```java
void unregister() {
    if (this.eurekaTransport != null && this.eurekaTransport.registrationClient != null) {
        try {
            logger.info("Unregistering ...");
          //向注册中心发送下线实例的请求
            EurekaHttpResponse<Void> httpResponse = this.eurekaTransport.registrationClient.cancel(this.instanceInfo.getAppName(), this.instanceInfo.getId());
            logger.info("DiscoveryClient_{} - deregister  status: {}", this.appPathIdentifier, httpResponse.getStatusCode());
        } catch (Exception var2) {
            logger.error("DiscoveryClient_{} - de-registration failed{}", new Object[]{this.appPathIdentifier, var2.getMessage(), var2});
        }
    }
}
```

org.springframework.boot.SpringApplication#run(java.lang.String...)

```java
public ConfigurableApplicationContext run(String... args) {
  //...
 SpringApplicationRunListeners listeners = this.getRunListeners(args);
  //...
  listeners.starting();
   try {
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
}
```

org.springframework.boot.SpringApplicationRunListeners#running

```java
public void running(ConfigurableApplicationContext context) {
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.running(context);
    }

}
```

```java
public void running(ConfigurableApplicationContext context) {
  //发布ApplicationReadyEvent
    context.publishEvent(new     public void running(ConfigurableApplicationContext context) {
        context.publishEvent(new ApplicationReadyEvent(this.application, this.args, context));
    }(this.application, this.args, context));
}
```

控制器

```java
@Bean
public ZuulController zuulControlleco) {
    return new ZuulController(); //控制器 
}

@Bean
public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes) { //handlermapping
    ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, this.zuulController());
    mapping.setErrorController(this.errorController);
    return mapping;
}
```

```java
public ZuulController() {
    this.setServletClass(ZuulServlet.class);
    this.setServletName("zuul");
    this.setSupportedMethods((String[])null);
}
```

org.springframework.web.servlet.mvc.ServletWrappingController#afterPropertiesSet

```java
public void afterPropertiesSet() throws Exception {
    if (this.servletClass == null) {
        throw new IllegalArgumentException("'servletClass' is required");
    } else {
        if (this.servletName == null) {
            this.servletName = this.beanName;
        }
				//实例化ZuulServlet
        this.servletInstance = (Servlet)this.servletClass.newInstance();
        this.servletInstance.init(new ServletWrappingController.DelegatingServletConfig());
    }
}
```

过滤器

```java
@Bean
public ServletDetectionFilter servletDetectionFilter() {
    return new ServletDetectionFilter();
}

@Bean
public FormBodyWrapperFilter formBodyWrapperFilter() {
    return new FormBodyWrapperFilter();
}

@Bean
public DebugFilter debugFilter() {
    return new DebugFilter();
}

@Bean
public Servlet30WrapperFilter servlet30WrapperFilter() {
    return new Servlet30WrapperFilter();
}

@Bean
public SendResponseFilter sendResponseFilter() {
    return new SendResponseFilter();
}

@Bean
public SendErrorFilter sendErrorFilter() {
    return new SendErrorFilter();
}

@Bean
public SendForwardFilter sendForwardFilter() {
    return new SendForwardFilter();
}
```

处理

com.netflix.zuul.http.ZuulServlet#service

```java
public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
    try {
        this.init((HttpServletRequest)servletRequest, (HttpServletResponse)servletResponse);
        RequestContext context = RequestContext.getCurrentContext();
        context.setZuulEngineRan();

        try { 
            this.preRoute(); //前置处理
        } catch (ZuulException var13) {
            this.error(var13);
            this.postRoute();
            return;
        }

        try {
            this.route(); //处理请求
        } catch (ZuulException var12) {
            this.error(var12);
            this.postRoute();
            return;
        }

        try {
            this.postRoute(); //后置处理
        } catch (ZuulException var11) {
            this.error(var11);
        }
    } catch (Throwable var14) {
        this.error(new ZuulException(var14, 500, "UNHANDLED_EXCEPTION_" + var14.getClass().getName())); //处理error
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}
```

org.springframework.cloud.netflix.zuul.filters.route.RibbonRoutingFilter#run

```java
public Object run() {
    RequestContext context = RequestContext.getCurrentContext();
    this.helper.addIgnoredHeaders(new String[0]);

    try {
        RibbonCommandContext commandContext = this.buildCommandContext(context);
        ClientHttpResponse response = this.forward(commandContext);
        this.setResponse(response);
        return response;
    } catch (ZuulException var4) {
        throw new ZuulRuntimeException(var4);
    } catch (Exception var5) {
        throw new ZuulRuntimeException(var5);
    }
}
```

https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-loadbalancer

官网文档总结

应用程序初始化时，就加载LoadBalancer contexts

Spring Cloud LoadBalancer为每个服务创建一个单独的Spring子上下文。默认情况下，只有每第一次对服务发起远程调用时，这些上下文才会初始化

```yaml
ribbon:
	eager-load:
    enabled: true #是否提前初始化
    clients:  #指定服务名称
```



# 自定义健康检测

eureka.client.healthcheck.enabled=true

org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration.EurekaHealthCheckHandlerConfiguration

```java
@Configuration
@ConditionalOnProperty(
    value = {"eureka.client.healthcheck.enabled"},
    matchIfMissing = false
)
protected static class EurekaHealthCheckHandlerConfiguration {
    @Autowired(
        required = false
    )
    private HealthAggregator healthAggregator = new OrderedHealthAggregator();

    protected EurekaHealthCheckHandlerConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean({HealthCheckHandler.class})
    public EurekaHealthCheckHandler eurekaHealthCheckHandler() {
        return new EurekaHealthCheckHandler(this.healthAggregator);
    }

```

会将com.netflix.appinfo.HealthCheckHandler注入到DiscoveryClient

也会将com.netflix.discovery.InstanceInfoReplicator注入到DiscoveryClient

com.netflix.discovery.InstanceInfoReplicator#run

```java
public void run() {
    try {
      //刷新服务提供者的信息
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

com.netflix.discovery.DiscoveryClient#refreshInstanceInfo

```java
void refreshInstanceInfo() {
    applicationInfoManager.refreshDataCenterInfoIfRequired();
    applicationInfoManager.refreshLeaseInfoIfRequired();

    InstanceStatus status;
    try {
      //调用自定义的状态检测
        status = getHealthCheckHandler().getStatus(instanceInfo.getStatus());
    } catch (Exception e) {
        logger.warn("Exception from healthcheckHandler.getStatus, setting status to DOWN", e);
        status = InstanceStatus.DOWN;
    }

  //发布事件：状态变更
    if (null != status) {
        applicationInfoManager.setInstanceStatus(status);
    }
}
```

```java
public synchronized void setInstanceStatus(InstanceStatus status) {
    InstanceStatus next = instanceStatusMapper.map(status);
    if (next == null) {
        return;
    }
		//修改instanceInfo的status属性，
  	//DiscoveryClient销毁的时候也会调用此方法
    InstanceStatus prev = instanceInfo.setStatus(next);
    if (prev != null) {
        for (StatusChangeListener listener : listeners.values()) {
            try {
                listener.notify(new StatusChangeEvent(prev, next));
            } catch (Exception e) {
                logger.warn("failed to notify listener: {}", listener.getId(), e);
            }
        }
    }
}
```

```java
//DiscoveryClient构造函数中创建
statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
    @Override
    public String getId() {
        return "statusChangeListener";
    }

    @Override
    public void notify(StatusChangeEvent statusChangeEvent) {
        if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
            logger.warn("Saw local status change event {}", statusChangeEvent);
        } else {
            logger.info("Saw local status change event {}", statusChangeEvent);
        }
        instanceInfoReplicator.onDemandUpdate();
    }
};
```

自定义HealthIndicator，实现doHealthCheck方法

org.springframework.cloud.netflix.eureka.EurekaHealthCheckHandler#afterPropertiesSet

```java
public void afterPropertiesSet() throws Exception {
  //获取所有实现接口HealthIndicator的bean
    Map<String, HealthIndicator> healthIndicators = this.applicationContext.getBeansOfType(HealthIndicator.class);
    Iterator var2 = healthIndicators.entrySet().iterator();

    while(true) {
        while(var2.hasNext()) {
            Entry<String, HealthIndicator> entry = (Entry)var2.next();
            
            if (entry.getValue() instanceof DiscoveryCompositeHealthIndicator) {
                DiscoveryCompositeHealthIndicator indicator = (DiscoveryCompositeHealthIndicator)entry.getValue();
                Iterator var5 = indicator.getHealthIndicators().iterator();

                while(var5.hasNext()) {
                    Holder holder = (Holder)var5.next();
                    if (!(holder.getDelegate() instanceof EurekaHealthIndicator)) {
                        this.healthIndicator.addHealthIndicator(holder.getDelegate().getName(), holder);
                    }
                }
            } else {
              	//healthIndicator为CompositeHealthIndicator
              //负责组合所有实现healthIndicator接口的bean
                this.healthIndicator.addHealthIndicator((String)entry.getKey(), (HealthIndicator)entry.getValue());
            }
        }

        return;
    }
}
```

org.springframework.cloud.netflix.eureka.EurekaHealthCheckHandler#getHealthStatus

```java
protected InstanceStatus getHealthStatus() {
    Status status = this.healthIndicator.health().getStatus();
    return this.mapToInstanceStatus(status);
}
```

```java
protected InstanceStatus mapToInstanceStatus(Status status) {
    return !STATUS_MAPPING.containsKey(status) ? InstanceStatus.UNKNOWN : (InstanceStatus)STATUS_MAPPING.get(status);
}
```

org.springframework.boot.actuate.health.CompositeHealthIndicator#health

```java
public Health health() {
    Map<String, Health> healths = new LinkedHashMap();
    Iterator var2 = this.indicators.entrySet().iterator();
		//逐个调用
    while(var2.hasNext()) {
        Entry<String, HealthIndicator> entry = (Entry)var2.next();
      	//调用自定义的health方法
        healths.put(entry.getKey(), ((HealthIndicator)entry.getValue()).health());
    }

    return this.healthAggregator.aggregate(healths);
}
```

org.springframework.boot.actuate.health.OrderedHealthAggregator#aggregateStatus

```java
public OrderedHealthAggregator() {
    this.setStatusOrder(Status.DOWN, Status.OUT_OF_SERVICE, Status.UP, Status.UNKNOWN); //四种状态码
}
```

```java
protected Status aggregateStatus(List<Status> candidates) {
    List<Status> filteredCandidates = new ArrayList();
    Iterator var3 = candidates.iterator();

    while(var3.hasNext()) {
        Status candidate = (Status)var3.next();
        if (this.statusOrder.contains(candidate.getCode())) {
            filteredCandidates.add(candidate);
        }
    }

    if (filteredCandidates.isEmpty()) {
        return Status.UNKNOWN;
    } else {
      	//排序，DOWN排在前面
        Collections.sort(filteredCandidates, new OrderedHealthAggregator.StatusComparator(this.statusOrder));
        return (Status)filteredCandidates.get(0);
    }
}
```
