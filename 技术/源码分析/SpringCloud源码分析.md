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

springboot应用启动时会从spring.factories文件中加载EurekaServerAutoConfiguration自动配置类

```java
@Configuration
@Import({EurekaServerInitializerConfiguration.class})
@ConditionalOnBean({Marker.class})
@EnableConfigurationProperties({EurekaDashboardProperties.class, InstanceRegistryProperties.class})
@PropertySource({"classpath:/eureka/server.properties"})
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
```

需要有一个marker bean，才能装配Eureka Server，marker bean 其实是由@EnableEurekaServer注解决定的

## EnableEurekaServer注解

org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({EurekaServerMarkerConfiguration.class})
public @interface EnableEurekaServer {
}
```

org.springframework.cloud.netflix.eureka.server.EurekaServerMarkerConfiguration

```java
@Bean
public EurekaServerMarkerConfiguration.Marker eurekaServerMarkerBean() {
    return new EurekaServerMarkerConfiguration.Marker();
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
	
    super.register(info, leaseDuration, isReplication); //注册实例
    this.replicateToPeers(PeerAwareInstanceRegistryImpl.Action.Register, info.getAppName(), info.getId(), info, (InstanceStatus)null, isReplication);//复制实例到其他节点
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

获取服务

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

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#renew

```java
public boolean renew(String appName, String id, boolean isReplication) {
    if (super.renew(appName, id, isReplication)) { //本机续约
        this.replicateToPeers(PeerAwareInstanceRegistryImpl.Action.Heartbeat, appName, id, (InstanceInfo)null, (InstanceStatus)null, isReplication); //集群其他节点续约
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

```
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

# Ribbon

## 组件

### LoadBalanced

表明启用负载均衡功能

```java
/**
 * Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient
 * @author Spencer Gibb
 */
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {

   <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

   <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

   URI reconstructURI(ServiceInstance instance, URI original);
}
```

### LoadBalancerAutoConfiguration

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

   @Autowired(required = false)
   private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

  
   //注入LoadBalancerClient（在RibbonAutoConfiguration中生成）创建LoadBalancerRequestFactory，用于创建LoadBalancerReques
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
     
     //创建拦截器LoadBalancerInterceptor
      @Bean
      public LoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRequestFactory requestFactory) {
         return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
      }

     //注入拦截器，创建RestTemplateCustomizer，负责将拦截器注入到RestTemplate
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

给集合中的每一个Resttemplate对象添加一个拦截器

```java
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
```

### LoadBalancerInterceptor

```java
@Bean
public LoadBalancerInterceptor ribbonInterceptor(
      LoadBalancerClient loadBalancerClient,
      LoadBalancerRequestFactory requestFactory) {
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

创建LoadBalancerRequest

```java
@Bean
@ConditionalOnMissingBean
public LoadBalancerRequestFactory loadBalancerRequestFactory(
      LoadBalancerClient loadBalancerClient) {
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

## 执行流程

### 预先注入拦截器

```java
public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
   // Take getInterceptors() List as-is when passed in here
   if (this.interceptors != interceptors) {
      this.interceptors.clear();
      this.interceptors.addAll(interceptors); //注入拦截器LoadBalancerInterceptor
      AnnotationAwareOrderComparator.sort(this.interceptors);
   }
}
```

### 执行

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

#### 创建请求

```java
protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
   ClientHttpRequest request = getRequestFactory().createRequest(url, method);
   if (logger.isDebugEnabled()) {
      logger.debug("Created " + method.name() + " request for \"" + url + "\"");
   }
   return request;
}
```

##### 获取请求工厂

```java
public ClientHttpRequestFactory getRequestFactory() { //获取
   List<ClientHttpRequestInterceptor> interceptors = getInterceptors(); //获取注入的拦截器
   if (!CollectionUtils.isEmpty(interceptors)) {
      ClientHttpRequestFactory factory = this.interceptingRequestFactory;
      if (factory == null) { //创建InterceptingClientHttpRequestFactory
         factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
         this.interceptingRequestFactory = factory;
      }
      return factory;
   }
   else {
      return super.getRequestFactory();
   }
}
```

##### 创建请求

org.springframework.http.client.InterceptingClientHttpRequestFactory#createRequest

```java
protected ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) {
   return new InterceptingClientHttpRequest(requestFactory, this.interceptors, uri, httpMethod);
}
```

#### 真正执行

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
   if (this.iterator.hasNext()) { //执行拦截器
      ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
      return nextInterceptor.intercept(request, body, this);
   }
   else {
      HttpMethod method = request.getMethod();
      Assert.state(method != null, "No standard HTTP method");
      ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
      request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
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

org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor#intercept

```java
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
      final ClientHttpRequestExecution execution) throws IOException {
   final URI originalUri = request.getURI();
   String serviceName = originalUri.getHost();
   Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
   return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
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
        //发送请求
        RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
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

org.springframework.http.client.InterceptingClientHttpRequest.InterceptingRequestExecution#execute

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
   if (this.iterator.hasNext()) { //ClientHttpRequestInterceptor的迭代器，是否还有未迭代的
      ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
      return nextInterceptor.intercept(request, body, this);
   }
   else {//拦截器迭代结束
      HttpMethod method = request.getMethod();
      Assert.state(method != null, "No standard HTTP method");
     //创建ClientHttpRequest
      ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
      request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
      if (body.length > 0) {
         if (delegate instanceof StreamingHttpOutputMessage) {
            StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
            streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
         }
         else {
            StreamUtils.copy(body, delegate.getBody());
         }
      }
      return delegate.execute(); //发送请求
   }
}
```

org.springframework.cloud.netflix.ribbon.RibbonHttpRequest#executeInternal

```java
protected ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException {
    try {
        this.addHeaders(headers);
        if (this.outputStream != null) {
            this.outputStream.close();
            this.builder.entity(this.outputStream.toByteArray());
        }

        HttpRequest request = this.builder.build();
        HttpResponse response = (HttpResponse)this.client.executeWithLoadBalancer(request, this.config);
        return new RibbonHttpResponse(response);
    } catch (Exception var4) {
        throw new IOException(var4);
    }
}
```

# Feign

## 组件

### EnableFeignClients

启动Feign

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({FeignClientsRegistrar.class})
public @interface EnableFeignClients {
    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<?>[] defaultConfiguration() default {};

    Class<?>[] clients() default {};
}
```

### FeignClientsRegistrar

向容器中注册Feign相关的配置类、接口类

org.springframework.cloud.openfeign.FeignClientsRegistrar#registerBeanDefinitions

```java
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    this.registerDefaultConfiguration(metadata, registry);
    this.registerFeignClients(metadata, registry);
}
```

### FeignClientSpecification

注册FeignClient的全局配置到容器中

org.springframework.cloud.openfeign.FeignClientsRegistrar#registerDefaultConfiguration

```java
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

### FeignClientFactoryBean

为标有FeignClient注解的接口生成代理对象

```java
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {//扫描标有FeignClient注解的接口，注册到容器
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
            // FeignClient只能用在接口上
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
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
   definition.addPropertyValue("url", getUrl(attributes)); //指定固定服务实例的地址，用于测试环境中，不需要负载均衡功能
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

## 获取代理对象

org.springframework.cloud.openfeign.FeignClientFactoryBean#getObject

```java
public Object getObject() throws Exception {
   FeignContext context = applicationContext.getBean(FeignContext.class);
   Feign.Builder builder = feign(context);

   if (!StringUtils.hasText(this.url)) { //url为空
      String url;
      if (!this.name.startsWith("http")) {
         url = "http://" + this.name;
      }
      else {
         url = this.name;
      }
      url += cleanPath();
      //具有负载均衡功能
      return loadBalance(builder, context, new HardCodedTarget<>(this.type,
            this.name, url));
   }
  //url不为空，表明固定访问某一服务实例
   if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
      this.url = "http://" + this.url;
   }
   String url = this.url + cleanPath();
   Client client = getOptional(context, Client.class);
   if (client != null) {
      if (client instanceof LoadBalancerFeignClient) {
         // not lod balancing because we have a url,
         // but ribbon is on the classpath, so unwrap
         client = ((LoadBalancerFeignClient)client).getDelegate();
      }
      builder.client(client);
   }
   Targeter targeter = get(context, Targeter.class);
   return targeter.target(this, builder, context, new HardCodedTarget<>(
         this.type, this.name, url));
}
```

org.springframework.cloud.openfeign.FeignClientFactoryBean#loadBalance

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
      HardCodedTarget<T> target) {
   Client client = getOptional(context, Client.class);
   if (client != null) {
      builder.client(client);
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

