# 如何使用？

application.yml

```yaml
spring:
  cloud:
    zookeeper:
      connect-string: localhost:2130
  application:
    name: zk-test
server:
  port: 7771
```

pom.xml

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
```

# 如何整合？

## 建立连接

```java
@Configuration
@ConditionalOnZookeeperEnabled
@EnableConfigurationProperties
public class ZookeeperAutoConfiguration {
    private static final Log log = LogFactory.getLog(ZookeeperAutoConfiguration.class);
    @Autowired(
        required = false
    )
    private EnsembleProvider ensembleProvider;

    public ZookeeperAutoConfiguration() {
    }

  	//zk连接信息
    @Bean
    @ConditionalOnMissingBean
    public ZookeeperProperties zookeeperProperties() {
        return new ZookeeperProperties();
    }

   //创建CuratorFramework，建立连接
    @Bean(
        destroyMethod = "close"
    )
    @ConditionalOnMissingBean
    public CuratorFramework curatorFramework(RetryPolicy retryPolicy, ZookeeperProperties properties) throws Exception {
        Builder builder = CuratorFrameworkFactory.builder();
        if (this.ensembleProvider != null) {
            builder.ensembleProvider(this.ensembleProvider);
        } else {
            builder.connectString(properties.getConnectString());
        }

        CuratorFramework curator = builder.retryPolicy(retryPolicy).build();
        curator.start();
        log.trace("blocking until connected to zookeeper for " + properties.getBlockUntilConnectedWait() + properties.getBlockUntilConnectedUnit());
        curator.blockUntilConnected(properties.getBlockUntilConnectedWait(), properties.getBlockUntilConnectedUnit());
        log.trace("connected to zookeeper");
        return curator;
    }

  	//重试策略
    @Bean
    @ConditionalOnMissingBean
    public RetryPolicy exponentialBackoffRetry(ZookeeperProperties properties) {
        return new ExponentialBackoffRetry(properties.getBaseSleepTimeMs(), properties.getMaxRetries(), properties.getMaxSleepMs());
    }
}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
//默认启用
@ConditionalOnProperty(
    value = {"spring.cloud.zookeeper.enabled"},
    matchIfMissing = true
)
public @interface ConditionalOnZookeeperEnabled {
}
```

## 服务注册

服务注册时机：容器完全初始化成功之后注册

```
ZookeeperServiceRegistry实现了ServiceRegistry接口，实现注册中心切换到Zookeeper，注册对象为ZookeeperRegistration，同时也实现了SmartInitializingSingleton接口的afterSingletonsInstantiated方法	，在所有的Bean都创建之后执行
EurekaServiceRegistry也实现ServiceRegistry，注册对象为EurekaRegistration
```

```java
//注册服务
@Override
public void afterSingletonsInstantiated() {
   try {
      getServiceDiscovery().start();
   } catch (Exception e) {
      rethrowRuntimeException(e);
   }
}
```

CuratorServiceDiscoveryAutoConfiguration

```java
//创建ServiceDiscoveryImpl
@Configuration
@ConditionalOnZookeeperDiscoveryEnabled
@AutoConfigureBefore({ ZookeeperDiscoveryAutoConfiguration.class,
		ZookeeperServiceRegistryAutoConfiguration.class })
public class CuratorServiceDiscoveryAutoConfiguration {
@Bean
@ConditionalOnMissingBean(ServiceDiscoveryCustomizer.class)
public DefaultServiceDiscoveryCustomizer defaultServiceDiscoveryCustomizer(
      CuratorFramework curator, ZookeeperDiscoveryProperties properties,
      InstanceSerializer<ZookeeperInstance> serializer) {
   return new DefaultServiceDiscoveryCustomizer(curator, properties, serializer);
}

@Bean
@ConditionalOnMissingBean
public InstanceSerializer<ZookeeperInstance> deprecatedInstanceSerializer() {
   return new JsonInstanceSerializer<>(ZookeeperInstance.class);
}

@Bean
@ConditionalOnMissingBean
public ServiceDiscovery<ZookeeperInstance> curatorServiceDiscovery(
      ServiceDiscoveryCustomizer customizer) {
   return customizer.customize(ServiceDiscoveryBuilder.builder(ZookeeperInstance.class));
}
}
```

ZookeeperAutoServiceRegistrationAutoConfiguration

```java
@Configuration
@ConditionalOnMissingBean(type = "org.springframework.cloud.zookeeper.discovery.ZookeeperLifecycle")
@ConditionalOnZookeeperDiscoveryEnabled
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
//先配置ZookeeperServiceRegistryAutoConfiguration
@AutoConfigureAfter( { ZookeeperServiceRegistryAutoConfiguration.class} )
//后配置
@AutoConfigureBefore( {AutoServiceRegistrationAutoConfiguration.class,ZookeeperDiscoveryAutoConfiguration.class} )
public class ZookeeperAutoServiceRegistrationAutoConfiguration {

   //负责服务信息的注册
   @Bean
   public ZookeeperAutoServiceRegistration zookeeperAutoServiceRegistration(
         ZookeeperServiceRegistry registry, ZookeeperRegistration registration,
         ZookeeperDiscoveryProperties properties) {
      return new ZookeeperAutoServiceRegistration(registry, registration, properties);
   }

  //负责服务信息的创建
   @Bean
   @ConditionalOnMissingBean(ZookeeperRegistration.class)
   public ServiceInstanceRegistration serviceInstanceRegistration(
         ApplicationContext context, ZookeeperDiscoveryProperties properties) {
      String appName = context.getEnvironment().getProperty("spring.application.name",
            "application");
      String host = properties.getInstanceHost();
      if (!StringUtils.hasText(host)) {
         throw new IllegalStateException("instanceHost must not be empty");
      }

      ZookeeperInstance zookeeperInstance = new ZookeeperInstance(context.getId(),
            appName, properties.getMetadata());
      RegistrationBuilder builder = ServiceInstanceRegistration.builder().address(host)
            .name(appName).payload(zookeeperInstance)
            .uriSpec(properties.getUriSpec());

      if (properties.getInstanceSslPort() != null) {
         builder.sslPort(properties.getInstanceSslPort());
      }
      if (properties.getInstanceId() != null) {
         builder.id(properties.getInstanceId());
      }
      return builder.build();
   }

}
```

```java
public class ZookeeperServiceRegistry implements ServiceRegistry<ZookeeperRegistration>, SmartInitializingSingleton,
      Closeable {
   //主要方法
   @Override
	public void register(ZookeeperRegistration registration) {//注册
		try {
			getServiceDiscovery().registerService(registration.getServiceInstance());
		} catch (Exception e) {
			rethrowRuntimeException(e);
		}
	}
	@Override
	public void deregister(ZookeeperRegistration registration) {//下线
		try {
			getServiceDiscovery().unregisterService(registration.getServiceInstance());
		} catch (Exception e) {
			rethrowRuntimeException(e);
		}
	}
        
	@Override
	public void afterSingletonsInstantiated() {//所有Bean创建之后执行此方法
		try {
			getServiceDiscovery().start();  //注册服务
		} catch (Exception e) {
			rethrowRuntimeException(e);
		}
	}
}
```

org.springframework.cloud.zookeeper.serviceregistry.ZookeeperAutoServiceRegistration

```java
public abstract class AbstractAutoServiceRegistration<R extends Registration>
		implements AutoServiceRegistration, ApplicationContextAware, ApplicationListener<WebServerInitializedEvent> {
  @Override
	@SuppressWarnings("deprecation")
	public void onApplicationEvent(WebServerInitializedEvent event) {//监听事件
		bind(event); //WebServerInitializedEvent
	}
  	@Deprecated
	public void bind(WebServerInitializedEvent event) {
		ApplicationContext context = event.getApplicationContext();
		if (context instanceof ConfigurableWebServerApplicationContext) {
			if ("management".equals(
					((ConfigurableWebServerApplicationContext) context).getServerNamespace())) {
				return;
			}
		}
		this.port.compareAndSet(0, event.getWebServer().getPort());
		this.start();
	}
  public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get()) {
			this.context.publishEvent(new InstancePreRegisteredEvent(this, getRegistration()));
			register(); //注册服务
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, getConfiguration()));
			this.running.compareAndSet(false, true);
		}
	}
  protected void register() {
		this.serviceRegistry.register(getRegistration());
	}
}
//如果切换注册注册中心，需继承AbstractAutoServiceRegistration，
//还要创建实现接口ServiceRegistry
//ZookeeperServiceRegistry
public class ZookeeperAutoServiceRegistration extends AbstractAutoServiceRegistration<ZookeeperRegistration> {
  	@Override
	protected ZookeeperRegistration getRegistration() {
		return this.registration;
	}
}
```

## 服务发现

CuratorServiceDiscoveryAutoConfiguration

```java
@Configuration
@ConditionalOnZookeeperDiscoveryEnabled
//后配置ZookeeperDiscoveryAutoConfiguration、ZookeeperServiceRegistryAutoConfiguration
@AutoConfigureBefore({ ZookeeperDiscoveryAutoConfiguration.class,
      ZookeeperServiceRegistryAutoConfiguration.class })
public class CuratorServiceDiscoveryAutoConfiguration {

   //定制ServiceDiscovery
   @Bean
   @ConditionalOnMissingBean(ServiceDiscoveryCustomizer.class)
   public DefaultServiceDiscoveryCustomizer defaultServiceDiscoveryCustomizer(
         CuratorFramework curator, ZookeeperDiscoveryProperties properties,
         InstanceSerializer<ZookeeperInstance> serializer) {
      return new DefaultServiceDiscoveryCustomizer(curator, properties, serializer);
   }

  //序列化
   @Bean
   @ConditionalOnMissingBean
   public InstanceSerializer<ZookeeperInstance> deprecatedInstanceSerializer() {
      return new JsonInstanceSerializer<>(ZookeeperInstance.class);
   }

  //创建ServiceDiscovery
   @Bean
   @ConditionalOnMissingBean
   public ServiceDiscovery<ZookeeperInstance> curatorServiceDiscovery(
         ServiceDiscoveryCustomizer customizer) {
      return customizer.customize(ServiceDiscoveryBuilder.builder(ZookeeperInstance.class));
   }
}
```

ZookeeperDiscoveryAutoConfiguration

```java
@Configuration
@ConditionalOnBean(ZookeeperDiscoveryClientConfiguration.Marker.class)
@ConditionalOnZookeeperDiscoveryEnabled
@AutoConfigureBefore({CommonsClientAutoConfiguration.class, NoopDiscoveryClientAutoConfiguration.class})
@AutoConfigureAfter({ZookeeperDiscoveryClientConfiguration.class})
public class ZookeeperDiscoveryAutoConfiguration {

   @Autowired(required = false)
   private ZookeeperDependencies zookeeperDependencies;

   @Autowired
   private CuratorFramework curator;

  //roperties related to Zookeeper's Service Discovery
   @Bean
   @ConditionalOnMissingBean
   public ZookeeperDiscoveryProperties zookeeperDiscoveryProperties(InetUtils inetUtils) {
      return new ZookeeperDiscoveryProperties(inetUtils);
   }

  //Zookeeper version of DiscoveryClient
   @Bean
   @ConditionalOnMissingBean
   public ZookeeperDiscoveryClient zookeeperDiscoveryClient(
         ServiceDiscovery<ZookeeperInstance> serviceDiscovery,
         ZookeeperDiscoveryProperties zookeeperDiscoveryProperties) {
      return new ZookeeperDiscoveryClient(serviceDiscovery, this.zookeeperDependencies,
            zookeeperDiscoveryProperties);
   }

   @Configuration
   @ConditionalOnEnabledHealthIndicator("zookeeper")
   @ConditionalOnClass(Endpoint.class)
   protected static class ZookeeperDiscoveryHealthConfig {
      @Autowired(required = false)
      private ZookeeperDependencies zookeeperDependencies;

      @Bean
      @ConditionalOnMissingBean
      public ZookeeperDiscoveryHealthIndicator zookeeperDiscoveryHealthIndicator(
            CuratorFramework curatorFramework,
            ServiceDiscovery<ZookeeperInstance> serviceDiscovery,
            ZookeeperDiscoveryProperties properties) {
         return new ZookeeperDiscoveryHealthIndicator(curatorFramework,
               serviceDiscovery, this.zookeeperDependencies, properties);
      }
   }

  //事件监听
   @Bean
   public ZookeeperServiceWatch zookeeperServiceWatch(ZookeeperDiscoveryProperties zookeeperDiscoveryProperties) {
      return new ZookeeperServiceWatch(this.curator, zookeeperDiscoveryProperties);
   }

}
```

org.springframework.cloud.zookeeper.discovery.ZookeeperDiscoveryClient#getInstances

```java
@Override
public List<org.springframework.cloud.client.ServiceInstance> getInstances(
      final String serviceId) { //发现服务实例
   try {
      if (getServiceDiscovery() == null) {
         return Collections.EMPTY_LIST;
      }
      String serviceIdToQuery = getServiceIdToQuery(serviceId);
      Collection<ServiceInstance<ZookeeperInstance>> zkInstances = getServiceDiscovery().queryForInstances(serviceIdToQuery);
      List<org.springframework.cloud.client.ServiceInstance> instances = new ArrayList<>();
      for (ServiceInstance<ZookeeperInstance> instance : zkInstances) {
         instances.add(createServiceInstance(serviceIdToQuery, instance));
      }
      return instances;
   } catch (KeeperException.NoNodeException e) {
      if (log.isDebugEnabled()) {
         log.debug("Error getting instances from zookeeper. Possibly, no service has registered.", e);
      }
      // this means that nothing has registered as a service yes
      return Collections.emptyList();
   } catch (Exception exception) {
      rethrowRuntimeException(exception);
   }
   return new ArrayList<>();
}
```

```java
//zk健康检测
@Configuration
@ConditionalOnZookeeperEnabled
@ConditionalOnClass({Endpoint.class})
//在ZookeeperAutoConfiguration配置之后执行
@AutoConfigureAfter({ZookeeperAutoConfiguration.class})
public class ZookeeperHealthAutoConfiguration {
    public ZookeeperHealthAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean({ZookeeperHealthIndicator.class})
    @ConditionalOnBean({CuratorFramework.class})
    @ConditionalOnEnabledHealthIndicator("zookeeper")
    public ZookeeperHealthIndicator zookeeperHealthIndicator(CuratorFramework curator) {
        return new ZookeeperHealthIndicator(curator);
    }
}
```

