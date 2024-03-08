# 配置解析

org.springframework.boot.autoconfigure.SpringBootApplication

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration 
@EnableAutoConfiguration// 启动自动配置功能
@ComponentScan(excludeFilters = { //包扫描器，扫描启动类所在的包及其子包
      @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

   /**
    * Exclude specific auto-configuration classes such that they will never be applied.
    * @return the classes to exclude
    */
   @AliasFor(annotation = EnableAutoConfiguration.class)
   Class<?>[] exclude() default {};

   /**
    * Exclude specific auto-configuration class names such that they will never be
    * applied.
    * @return the class names to exclude
    * @since 1.3.0
    */
   @AliasFor(annotation = EnableAutoConfiguration.class)
   String[] excludeName() default {};

   /**
    * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
    * for a type-safe alternative to String-based package names.
    * @return base packages to scan
    * @since 1.3.0
    */
   @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
   String[] scanBasePackages() default {};

   /**
    * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
    * scan for annotated components. The package of each class specified will be scanned.
    * <p>
    * Consider creating a special no-op marker class or interface in each package that
    * serves no purpose other than being referenced by this attribute.
    * @return base packages to scan
    * @since 1.3.0
    */
   @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
   Class<?>[] scanBasePackageClasses() default {};

}
```

## EnableAutoConfiguration

```java
Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage //会拿到@SpringBootApplication注解标注的类的所在的包名，并对该包及子包扫描
@Import(AutoConfigurationImportSelector.class)//借助@import来手机所有符合自动配置条件的bean定义，并加载到ioc容器
public @interface EnableAutoConfiguration {

   String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

   /**
    * Exclude specific auto-configuration classes such that they will never be applied.
    * @return the classes to exclude
    */
   Class<?>[] exclude() default {};

   /**
    * Exclude specific auto-configuration class names such that they will never be
    * applied.
    * @return the class names to exclude
    * @since 1.3.0
    */
   String[] excludeName() default {};

}
```

### AutoConfigurationPackage

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
//@import的作用就是给容器中导入某个组件类
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

   @Override
   public void registerBeanDefinitions(AnnotationMetadata metadata,
         BeanDefinitionRegistry registry) {
      register(registry, new PackageImport(metadata).getPackageName());
   }

   @Override
   public Set<Object> determineImports(AnnotationMetadata metadata) {
      return Collections.singleton(new PackageImport(metadata));
   }
}
```

```java
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
   if (registry.containsBeanDefinition(BEAN)) {
      BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
      ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
      constructorArguments.addIndexedArgumentValue(0, addBasePackages(constructorArguments, packageNames));
   }
   else {
     //注册BasePackages的定义
      GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
      beanDefinition.setBeanClass(BasePackages.class);
      beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
      beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
      registry.registerBeanDefinition(BEAN, beanDefinition);
   }
}
```

### AutoConfigurationImportSelector

org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#selectImports

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) { //返回自动导入的类
  //判断EnableAutoConfiguration注解有没有开启，默认开启
   if (!isEnabled(annotationMetadata)) {
      return NO_IMPORTS;
   }
  //加载spring-autoconfigure-metadata.properties文件，从中获取支持自动配置的类的条件
  //自动配置的类名.条件=类全限定名
   AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
         .loadMetadata(this.beanClassLoader);
  //METAINF/spring.factories
  //org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration.ConditionalOnBean=org.springframework.data.redis.connection.RedisConnectionFactory
   AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(
         autoConfigurationMetadata, annotationMetadata);
   return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(
      AutoConfigurationMetadata autoConfigurationMetadata,
      AnnotationMetadata annotationMetadata) {
   if (!isEnabled(annotationMetadata)) {
      return EMPTY_ENTRY;
   }
   AnnotationAttributes attributes = getAttributes(annotationMetadata);
  //扫描META-INF/spring.factories
  //获取自动配置类名列表
   List<String> configurations = getCandidateConfigurations(annotationMetadata,
         attributes);
  //去除重复的配置类
   configurations = removeDuplicates(configurations);
  //去除不希望自动配置的配置类
   Set<String> exclusions = getExclusions(annotationMetadata, attributes);
   checkExcludedClasses(configurations, exclusions);
   configurations.removeAll(exclusions);
  //判断是否要加载某个配置类有两种方式
  //1、根据spring-autoconfigure-metadata.properties进行判断
  //2、根据@ConditionalOnClass（{}）,表示需要在类路径中存在
   configurations = filter(configurations, autoConfigurationMetadata);
   fireAutoConfigurationImportEvents(configurations, exclusions);
   return new AutoConfigurationEntry(configurations, exclusions);
}
```



# 初始化

org.springframework.boot.SpringApplication#run(java.lang.Class<?>[], java.lang.String[])

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return (new SpringApplication(primarySources)).run(args);
}
```

```java
public SpringApplication(Class<?>... primarySources) {
    this((ResourceLoader)null, primarySources);
}
```

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = new HashSet();
    this.isCustomEnvironment = false;
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
  //判断当前应用程序的类型
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
  //扫描jar包中的spring.factories文件，获取ApplicationContextInitializer
//初始化器的实例对象
 this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
  //获取监听器的实例对象  
  this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

```java
//META-INF/spring.factories"读取type
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return this.getSpringFactoriesInstances(type, new Class[0]);
}
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = this.getClassLoader();
    Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList(names.size());
    Iterator var7 = names.iterator();
    while(var7.hasNext()) {
        String name = (String)var7.next();
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        } catch (Throwable var12) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, var12);
        }
    }
    return instances;
}
```

```java
//判断当前应用程序的类型
static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", (ClassLoader)null) && !ClassUtils.isPresent("org.springframework.web.servlet.DispatcherServlet", (ClassLoader)null) && !ClassUtils.isPresent("org.glassfish.jersey.servlet.ServletContainer", (ClassLoader)null)) {
        return REACTIVE;
    } else {
        String[] var0 = SERVLET_INDICATOR_CLASSES;
        int var1 = var0.length;

        for(int var2 = 0; var2 < var1; ++var2) {
            String className = var0[var2];
            if (!ClassUtils.isPresent(className, (ClassLoader)null)) {
                return NONE;
            }
        }
        return SERVLET;
    }
}
```

# 启动

```java
public ConfigurableApplicationContext run(String... args) {
  	//记录启动时间
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
    this.configureHeadlessProperty();
   //1、获取并启动监听器
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    listeners.starting();

    Collection exceptionReporters;
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      //2、项目运行环境Environment的预配置
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
        this.configureIgnoreBeanInfo(environment);
        Banner printedBanner = this.printBanner(environment);
      //3、创建Spring容器
        context = this.createApplicationContext();
        exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
      //4、spring容器前置处理，将启动类注入容器，为后续开启自动化配置奠定基础
        this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
      //5、刷新应用上下文，创建Bean
        this.refreshContext(context);
      //6、容器的后置处理
        this.afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }
				//7、发布ApplicationStartedEvent
        listeners.started(context);
     		 //8、执行Runners
         //在项目启动后立即执行一些特定程序
      	 //Spring提供了ApplicationRunner和CommandLineRunner两种接口
        this.callRunners(context, applicationArguments);
    } catch (Throwable var10) {
        this.handleRunFailure(context, var10, exceptionReporters, listeners);
        throw new IllegalStateException(var10);
    }

    try {
       //9、发布ApplicationReadyEvent
        listeners.running(context);
        return context;
    } catch (Throwable var9) {
        this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
        throw new IllegalStateException(var9);
    }
}
```

## 1、获取并启动监听器

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class[]{SpringApplication.class, String[].class};
    return new SpringApplicationRunListeners(logger, this.getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = this.getClassLoader();
  //从spring.factories文件获取clas字符串
    Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
  //实例化
    List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
  //排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

```java
public void starting() {
    Iterator var1 = this.listeners.iterator();

    while(var1.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var1.next();
        listener.starting();
    }

}
```

```java
public void starting() {
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

```java
public void multicastEvent(ApplicationEvent event) {
   multicastEvent(event, resolveDefaultEventType(event));
}
```

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
  //根据事件类型获取对应的监听器，执行监听事件
  //应用程序中一般通过调用AbstractApplicationContext中的publishEvent发布事件
  for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      Executor executor = getTaskExecutor();
      if (executor != null) {
        //由线程池执行
         executor.execute(() -> invokeListener(listener, event));
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
   ErrorHandler errorHandler = getErrorHandler();
   if (errorHandler != null) {
      try {
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         errorHandler.handleError(err);
      }
   }
   else {
      doInvokeListener(listener, event);
   }
}
```

```java
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
   try {
      listener.onApplicationEvent(event);
   }
   catch (ClassCastException ex) {
      String msg = ex.getMessage();
      if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
         // Possibly a lambda-defined listener which we could not resolve the generic event type for
         // -> let's suppress the exception and just log a debug message.
         Log logger = LogFactory.getLog(getClass());
         if (logger.isDebugEnabled()) {
            logger.debug("Non-matching event type for listener: " + listener, ex);
         }
      }
      else {
         throw ex;
      }
   }
}
```

## 2、 准备Environment

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
    //创建ConfigurableEnvironment
    ConfigurableEnvironment environment = this.getOrCreateEnvironment();
    this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
    
//广播ApplicationEnvironmentPreparedEvent  
  listeners.environmentPrepared((ConfigurableEnvironment)environment);
    this.bindToSpringApplication((ConfigurableEnvironment)environment);
    if (!this.isCustomEnvironment) {
        environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
    }

    ConfigurationPropertySources.attach((Environment)environment);
    return (ConfigurableEnvironment)environment;
}
```

### getOrCreateEnvironment

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    } else {
        switch(this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment(); //创建web类型的Environment
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
        }
    }
}
```

```java
public AbstractEnvironment() {//父类的构造方法
    this.propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);
    this.customizePropertySources(this.propertySources);
}
```

StandardServletEnvironment

```java
//调用自己重写的方法
protected void customizePropertySources(MutablePropertySources propertySources) {
    propertySources.addLast(new StubPropertySource("servletConfigInitParams"));
    propertySources.addLast(new StubPropertySource("servletContextInitParams"));
    if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
        propertySources.addLast(new JndiPropertySource("jndiProperties"));
    }
		//调用父类StandardEnvironment的方法
    super.customizePropertySources(propertySources);
}
```

```java
protected void customizePropertySources(MutablePropertySources propertySources) {
  //System.getProperties()
    propertySources.addLast(new MapPropertySource("systemProperties", this.getSystemProperties()));
  //System.getenv() 
  propertySources.addLast(new SystemEnvironmentPropertySource("systemEnvironment", this.getSystemEnvironment()));
}
```

### configureEnvironment

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService)conversionService);
    }
		//加载defaultProperties、解析命令行参数
    this.configurePropertySources(environment, args);
 	 //获取spring.profiles.active
    this.configureProfiles(environment, args);
}
```

```java
public DefaultApplicationArguments(String[] args) { //解析命令行参数
    Assert.notNull(args, "Args must not be null");
    this.source = new DefaultApplicationArguments.Source(args);
    this.args = args;
}
```

```java
public SimpleCommandLinePropertySource(String... args) {
    super((new SimpleCommandLineArgsParser()).parse(args));
}
```

```java
public CommandLineArgs parse(String... args) {
    CommandLineArgs commandLineArgs = new CommandLineArgs();
    String[] var3 = args;
    int var4 = args.length;

    for(int var5 = 0; var5 < var4; ++var5) {
        String arg = var3[var5];
        if (arg.startsWith("--")) {
            String optionText = arg.substring(2, arg.length());
            String optionValue = null;
            String optionName;
            if (optionText.contains("=")) {
                optionName = optionText.substring(0, optionText.indexOf(61));
                optionValue = optionText.substring(optionText.indexOf(61) + 1, optionText.length());
            } else {
                optionName = optionText;
            }

            if (optionName.isEmpty() || optionValue != null && optionValue.isEmpty()) {
                throw new IllegalArgumentException("Invalid argument syntax: " + arg);
            }

            commandLineArgs.addOptionArg(optionName, optionValue);
        } else {
            commandLineArgs.addNonOptionArg(arg);
        }
    }

    return commandLineArgs;
}
```

### environmentPrepared

```java
public void environmentPrepared(ConfigurableEnvironment environment) {
    Iterator var2 = this.listeners.iterator();

    while(var2.hasNext()) {
        SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
        listener.environmentPrepared(environment);
    }

}
```

org.springframework.boot.context.config.ConfigFileApplicationListener#onApplicationEvent

```java
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationEnvironmentPreparedEvent) {
        this.onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent)event);
    }

    if (event instanceof ApplicationPreparedEvent) {
        this.onApplicationPreparedEvent(event);
    }

}
```

```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
  //还是从META-INF/spring.factories文件加载
    List<EnvironmentPostProcessor> postProcessors = this.loadPostProcessors();
    postProcessors.add(this);
    AnnotationAwareOrderComparator.sort(postProcessors);
    Iterator var3 = postProcessors.iterator();

    while(var3.hasNext()) {
        EnvironmentPostProcessor postProcessor = (EnvironmentPostProcessor)var3.next();
        postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    }

}
```

```java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    this.addPropertySources(environment, application.getResourceLoader());
}
```

```java
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    RandomValuePropertySource.addToEnvironment(environment);
    (new ConfigFileApplicationListener.Loader(environment, resourceLoader)).load();
}
```

```java
Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    this.logger = ConfigFileApplicationListener.this.logger;
    this.loadDocumentsCache = new HashMap();
    this.environment = environment;
  //占位符解析器
    this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
    this.resourceLoader = (ResourceLoader)(resourceLoader != null ? resourceLoader : new DefaultResourceLoader());
  //加载PropertySourceLoader，比如PropertiesPropertySourceLoader、YamlPropertySourceLoader
    this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class, this.getClass().getClassLoader());
}
```

### attach

```java
public static void attach(Environment environment) {
    Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
    MutablePropertySources sources = ((ConfigurableEnvironment)environment).getPropertySources();
    PropertySource<?> attached = sources.get("configurationProperties");
    if (attached != null && attached.getSource() != sources) {
        sources.remove("configurationProperties");
        attached = null;
    }

    if (attached == null) {
        sources.addFirst(new ConfigurationPropertySourcesPropertySource("configurationProperties", new SpringConfigurationPropertySources(sources)));
    }

}
```



## 3、创建Spring容器

```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch(this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName("org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext");
                break;
            case REACTIVE:
                contextClass = Class.forName("org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext");
                break;
            default:
                contextClass = Class.forName("org.springframework.context.annotation.AnnotationConfigApplicationContext");
            }
        } catch (ClassNotFoundException var3) {
            throw new IllegalStateException("Unable create a default ApplicationContext, please specify an ApplicationContextClass", var3);
        }
    }

    return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
}
```

```java
public AnnotationConfigServletWebServerApplicationContext() {
    this.annotatedClasses = new LinkedHashSet();
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

## 4、prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    this.postProcessApplicationContext(context);
    this.applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        this.logStartupInfo(context.getParent() == null);
        this.logStartupProfileInfo(context);
    }

    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }

    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory)beanFactory).setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }

    Set<Object> sources = this.getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    this.load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
```

```java
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }

  //创建BeanDefinitionLoader
    BeanDefinitionLoader loader = this.createBeanDefinitionLoader(this.getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }

    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }

    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
		//加载BeanDefinition
    loader.load();
}
```

```java
public int load() {
    int count = 0;
    Object[] var2 = this.sources;
    int var3 = var2.length;

    for(int var4 = 0; var4 < var3; ++var4) {
        Object source = var2[var4];
        count += this.load(source);
    }

    return count;
}
```

```java
private int load(Object source) {
    Assert.notNull(source, "Source must not be null");
    if (source instanceof Class) { //当前启动类的class
        return this.load((Class)source);
    } else if (source instanceof Resource) {
        return this.load((Resource)source);
    } else if (source instanceof Package) {
        return this.load((Package)source);
    } else if (source instanceof CharSequence) {
        return this.load((CharSequence)source);
    } else {
        throw new IllegalArgumentException("Invalid source type " + source.getClass());
    }
}
```

```java
private int load(Class<?> source) {
    if (this.isGroovyPresent() && BeanDefinitionLoader.GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        BeanDefinitionLoader.GroovyBeanDefinitionSource loader = (BeanDefinitionLoader.GroovyBeanDefinitionSource)BeanUtils.instantiateClass(source, BeanDefinitionLoader.GroovyBeanDefinitionSource.class);
        this.load(loader);
    }

    if (this.isComponent(source)) {
        this.annotatedReader.register(new Class[]{source});
        return 1;
    } else {
        return 0;
    }
}
```



```java
private boolean isComponent(Class<?> type) {
    if (AnnotationUtils.findAnnotation(type, Component.class) != null) {
        return true;
    } else {
        return !type.getName().matches(".*\\$_.*closure.*") && !type.isAnonymousClass() && type.getConstructors() != null && type.getConstructors().length != 0;
    }
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

   /**
    * Explicitly specify the name of the Spring bean definition associated with the
    * {@code @Configuration} class. If left unspecified (the common case), a bean
    * name will be automatically generated.
    * <p>The custom name applies only if the {@code @Configuration} class is picked
    * up via component scanning or supplied directly to an
    * {@link AnnotationConfigApplicationContext}. If the {@code @Configuration} class
    * is registered as a traditional XML bean definition, the name/id of the bean
    * element will take precedence.
    * @return the explicit component name, if any (or empty String otherwise)
    * @see org.springframework.beans.factory.support.DefaultBeanNameGenerator
    */
   @AliasFor(annotation = Component.class)
   String value() default "";

}
```

## tomcat导入

```java
@Configuration //配置类
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET) //web运行环境
@EnableConfigurationProperties(ServerProperties.class) //以server开头的配置信息
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,//注入自定义的RootBeanDefinition
      ServletWebServerFactoryConfiguration.EmbeddedTomcat.class, //web容器
      ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
      ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {

   @Bean
   public ServletWebServerFactoryCustomizer servletWebServerFactoryCustomizer(
         ServerProperties serverProperties) {
      return new ServletWebServerFactoryCustomizer(serverProperties);
   }

   @Bean
   @ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
   public TomcatServletWebServerFactoryCustomizer tomcatServletWebServerFactoryCustomizer(
         ServerProperties serverProperties) {
      return new TomcatServletWebServerFactoryCustomizer(serverProperties);
   }

   /**
    * Registers a {@link WebServerFactoryCustomizerBeanPostProcessor}. Registered via
    * {@link ImportBeanDefinitionRegistrar} for early registration.
    */
   public static class BeanPostProcessorsRegistrar
         implements ImportBeanDefinitionRegistrar, BeanFactoryAware {

      private ConfigurableListableBeanFactory beanFactory;

      @Override
      public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
         if (beanFactory instanceof ConfigurableListableBeanFactory) {
            this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
         }
      }

      @Override
      public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
         if (this.beanFactory == null) {
            return;
         }
         registerSyntheticBeanIfMissing(registry,
               "webServerFactoryCustomizerBeanPostProcessor",
               WebServerFactoryCustomizerBeanPostProcessor.class);
         registerSyntheticBeanIfMissing(registry,
               "errorPageRegistrarBeanPostProcessor",
               ErrorPageRegistrarBeanPostProcessor.class);
      }

      private void registerSyntheticBeanIfMissing(BeanDefinitionRegistry registry,
            String name, Class<?> beanClass) {
         if (ObjectUtils.isEmpty(
               this.beanFactory.getBeanNamesForType(beanClass, true, false))) {
            RootBeanDefinition beanDefinition = new RootBeanDefinition(beanClass);
            beanDefinition.setSynthetic(true);
            registry.registerBeanDefinition(name, beanDefinition);
         }
      }

   }

}
```

org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryConfiguration.EmbeddedTomcat

```java
@Configuration
//类路径可以找到对应的class
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
//不存在不对应的bean对象
@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
public static class EmbeddedTomcat {

   @Bean
   public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
      return new TomcatServletWebServerFactory();
   }

}
```

## tomcat创建

Server ->Service ->Connector-> Engine->Host->Contexts

org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefreshorg.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh

```java
protected void onRefresh() {
    super.onRefresh();

    try {
        this.createWebServer();
    } catch (Throwable var2) {
        throw new ApplicationContextException("Unable to start web server", var2);
    }
}
```

```java
private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = this.getServletContext();
    if (webServer == null && servletContext == null) {
      //从容器获取对象
        ServletWebServerFactory factory = this.getWebServerFactory();
      //创建Tomcat
        this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
    } else if (servletContext != null) {
        try {
            this.getSelfInitializer().onStartup(servletContext);
        } catch (ServletException var4) {
            throw new ApplicationContextException("Cannot initialize servlet context", var4);
        }
    }

    this.initPropertySources();
}
```

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
    Tomcat tomcat = new Tomcat();
    File baseDir = this.baseDirectory != null ? this.baseDirectory : this.createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
 		
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    this.customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    this.configureEngine(tomcat.getEngine());
    Iterator var5 = this.additionalTomcatConnectors.iterator();

    while(var5.hasNext()) {
        Connector additionalConnector = (Connector)var5.next();
        tomcat.getService().addConnector(additionalConnector);
    }

    this.prepareContext(tomcat.getHost(), initializers);
    return this.getTomcatWebServer(tomcat);
}
```

```java
protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
    return new TomcatWebServer(tomcat, this.getPort() >= 0);
}
```

```java
public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
    this.monitor = new Object();
    this.serviceConnectors = new HashMap();
    Assert.notNull(tomcat, "Tomcat Server must not be null");
    this.tomcat = tomcat;
    this.autoStart = autoStart;
    this.initialize();
}
```

```java
private void initialize() throws WebServerException {
    logger.info("Tomcat initialized with port(s): " + this.getPortsDescription(false));
    synchronized(this.monitor) {
        try {
            this.addInstanceIdToEngineName();
            Context context = this.findContext();
            context.addLifecycleListener((event) -> {
                if (context.equals(event.getSource()) && "start".equals(event.getType())) {
                    this.removeServiceConnectors();
                }

            });
            this.tomcat.start();
            this.rethrowDeferredStartupExceptions();

            try {
                ContextBindings.bindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
            } catch (NamingException var5) {
            }

            this.startDaemonAwaitThread();
        } catch (Exception var6) {
            this.stopSilently();
            throw new WebServerException("Unable to start embedded Tomcat", var6);
        }

    }
}
```

tomcat启动

org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#finishRefresh

```java
protected void finishRefresh() {
    super.finishRefresh();
    WebServer webServer = this.startWebServer();
    if (webServer != null) {
        this.publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }

}
```

```java
private WebServer startWebServer() {
    WebServer webServer = this.webServer;
    if (webServer != null) {
        webServer.start();
    }

    return webServer;
}
```

```java
public void start() throws WebServerException {
    synchronized(this.monitor) {
        if (!this.started) {
            boolean var10 = false;

            try {
                var10 = true;
                this.addPreviouslyRemovedConnectors();
                Connector var2 = this.tomcat.getConnector();
                if (var2 != null && this.autoStart) {
                    this.performDeferredLoadOnStartup();
                }

                this.checkThatConnectorsHaveStarted();
                this.started = true;
                logger.info("Tomcat started on port(s): " + this.getPortsDescription(true) + " with context path '" + this.getContextPath() + "'");
                var10 = false;
            } catch (ConnectorStartFailedException var11) {
                this.stopSilently();
                throw var11;
            } catch (Exception var12) {
                throw new WebServerException("Unable to start embedded Tomcat server", var12);
            } finally {
                if (var10) {
                    Context context = this.findContext();
                    ContextBindings.unbindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
                }
            }

            Context context = this.findContext();
            ContextBindings.unbindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
        }
    }
}
```

1、springboot默认扫描与启动类所在的包下的所有类，如果要扫描其他包下的类需要显式配置

@ComponentScans(value={“componentScan(value="包名")”}）

org.springframework.context.annotation.ComponentScanAnnotationParser#parse

```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
   Assert.state(this.environment != null, "Environment must not be null");
   Assert.state(this.resourceLoader != null, "ResourceLoader must not be null");

   ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
         componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

   Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
   boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
   scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
         BeanUtils.instantiateClass(generatorClass));

   ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
   if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
      scanner.setScopedProxyMode(scopedProxyMode);
   }
   else {
      Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
      scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
   }

   scanner.setResourcePattern(componentScan.getString("resourcePattern"));

   for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
      for (TypeFilter typeFilter : typeFiltersFor(filter)) {
         scanner.addIncludeFilter(typeFilter);
      }
   }
   for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
      for (TypeFilter typeFilter : typeFiltersFor(filter)) {
         scanner.addExcludeFilter(typeFilter);
      }
   }

   boolean lazyInit = componentScan.getBoolean("lazyInit");
   if (lazyInit) {
      scanner.getBeanDefinitionDefaults().setLazyInit(true);
   }

   Set<String> basePackages = new LinkedHashSet<String>();
  //注解ComponentScans中的value是一个ComponentScan数组
   String[] basePackagesArray = componentScan.getStringArray("basePackages");
   for (String pkg : basePackagesArray) {
      String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
            ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      basePackages.addAll(Arrays.asList(tokenized));
   }
   for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
      basePackages.add(ClassUtils.getPackageName(clazz));
   }
  //当basePackages为空时，扫描的包会是declaringClass所在的包，即启动类所在的包
   if (basePackages.isEmpty()) {
      basePackages.add(ClassUtils.getPackageName(declaringClass));
   }

   scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
      @Override
      protected boolean matchClassName(String className) {
         return declaringClass.equals(className);
      }
   });
   return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

当使用RestTemplate组装表单数据时，我们应该注意要使用MultiValueMap而非普通的HashMap。否则会以JSON请求体的形式发送请求而非表单

Spring默认配置不合理

要完整接收所有的Header,不能直接使用Map而是使用MultiValueMap

注意依赖的变化，可能当你的代码不变时，依赖变了，代码的执行可能会出现异常



Autowired标记的原型Bean被固定了，没有生效

org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject

```java
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
   Field field = (Field) this.member;
   Object value;
   if (this.cached) {
      value = resolvedCachedArgument(beanName, this.cachedFieldValue);
   }
   else {
      DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
      desc.setContainingClass(bean.getClass());
      Set<String> autowiredBeanNames = new LinkedHashSet<String>(1);
      TypeConverter typeConverter = beanFactory.getTypeConverter();
      try {
         value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
      }
      catch (BeansException ex) {
         throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
      }
      synchronized (this) {
         if (!this.cached) {
            if (value != null || this.required) {
               this.cachedFieldValue = desc;
               registerDependentBeans(beanName, autowiredBeanNames);
               if (autowiredBeanNames.size() == 1) {
                  String autowiredBeanName = autowiredBeanNames.iterator().next();
                  if (beanFactory.containsBean(autowiredBeanName) &&
                        beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                     this.cachedFieldValue = new ShortcutDependencyDescriptor(
                           desc, autowiredBeanName, field.getType());
                  }
               }
            }
            else {
               this.cachedFieldValue = null;
            }
            this.cached = true;
         }
      }
   }
  //待我们寻找到要自动注入的Bean后，即可通过反射设置给对应的field。这个field的执行只发生了一次，所
	//以后续就固定起来了
  //可以通过注入ApplicationContext，每次从ApplicationContext获取Bean
   if (value != null) {
      ReflectionUtils.makeAccessible(field);
      field.set(bean, value);
   }
}
```

当你去书写代码时，多问自己几句，我
使用的代码书写方式是正确的么？我的解决方案是正规的套路么？我解决问题的方式会不会有其他的负面影
响？等等

# 启动监听

在程序启动后，回调一些方法来处理某些事情，比如初始化本地缓存。这种场景我们可以使用 CommandLineRunner 和 ApplicationRunner 两个监听接口来实现。不同的就是 ApplicationRunner 对方法的参数进行了封装

```java
public interface ApplicationRunner {
    void run(ApplicationArguments args) throws Exception;
}
```

```java
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

org.springframework.boot.SpringApplication#callRunners

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    Iterator var4 = (new LinkedHashSet(runners)).iterator();

    while(var4.hasNext()) {
        Object runner = var4.next();
        if (runner instanceof ApplicationRunner) {
            this.callRunner((ApplicationRunner)runner, args);
        }

        if (runner instanceof CommandLineRunner) {
            this.callRunner((CommandLineRunner)runner, args);
        }
    }

}
```