# 启动流程

com.a.eye.skywalking.agent.SkyWalkingAgent#premain

```java
public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
        try {
          	//初始化配置信息
            SnifferConfigInitializer.initialize(agentArgs);
						//查找并解析skywalking-plugin.def文件
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
        } catch (ConfigNotFoundException ce) {
            logger.error(ce, "Skywalking agent could not find config. Shutting down.");
            return;
        } catch (AgentPackageNotFoundException ape) {
            logger.error(ape, "Locate agent.jar failure. Shutting down.");
            return;
        } catch (Exception e) {
            logger.error(e, "Skywalking agent initialized failure. Shutting down.");
            return;
        }
				//使用ByteBuddy创建AgentBuilder
        final ByteBuddy byteBuddy = new ByteBuddy()
            .with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
        new AgentBuilder.Default(byteBuddy)
            .ignore( //忽略指定包中的类
                nameStartsWith("net.bytebuddy.")
                    .or(nameStartsWith("org.slf4j."))
                    .or(nameStartsWith("org.apache.logging."))
                    .or(nameStartsWith("org.groovy."))
                    .or(nameContains("javassist"))
                    .or(nameContains(".asm."))
                    .or(nameStartsWith("sun.reflect"))
                    .or(allSkyWalkingAgentExcludeToolkit())
                    .or(ElementMatchers.<TypeDescription>isSynthetic()))
            .type(pluginFinder.buildMatch())//拦截,根据className拦截、其他的注解拦截等等
            .transform(new Transformer(pluginFinder)) //设置Transform，对匹配类增强
            .with(new Listener())//设置监听器
            .installOn(instrumentation);
        try {
          //通过SPI加载BootService并启动
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            logger.error(e, "Skywalking agent boot failure.");
        }
				//添加JVM钩子
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override public void run() {
                ServiceManager.INSTANCE.shutdown(); //关闭BootService
            }
        }, "skywalking service shutdown thread"));
    }
```

## PluginBootstrap

查找并解析skywalking-plugin.def文件

```properties
jedis-2.x=org.apache.skywalking.apm.plugin.jedis.v2.define.JedisClusterInstrumentation
jedis-2.x=org.apache.skywalking.apm.plugin.jedis.v2.define.JedisInstrumentation
```

org.apache.skywalking.apm.agent.core.plugin.PluginBootstrap#loadPlugins

```java
public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
    AgentClassLoader.initDefaultLoader();
    PluginResourcesResolver resolver = new PluginResourcesResolver();
    //查找所有的skywalking-plugin.def
    List<URL> resources = resolver.getResources();
    if (resources == null || resources.size() == 0) {
        logger.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
        return new ArrayList<AbstractClassEnhancePluginDefine>();
    }
    //解析skywalking-plugin.def，封装成PluginDefine
    for (URL pluginUrl : resources) {
        try {
            PluginCfg.INSTANCE.load(pluginUrl.openStream());
        } catch (Throwable t) {
            logger.error(t, "plugin file [{}] init failure.", pluginUrl);
        }
    }
    List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
    List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
    for (PluginDefine pluginDefine : pluginClassList) {
        try {
            logger.debug("loading plugin class {}.", pluginDefine.getDefineClass());
            //AgentClassLoader加载PluginDefine
            AbstractClassEnhancePluginDefine plugin =
                (AbstractClassEnhancePluginDefine)Class.forName(pluginDefine.getDefineClass(),
                    true,
                    AgentClassLoader.getDefault())
                    .newInstance();
            plugins.add(plugin);
        } catch (Throwable t) {
            logger.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
        }
    }
    plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));
    return plugins;
}
```

## PluginFinder

维护AbstractClassEnhancePluginDefine

```java
private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new LinkedList<AbstractClassEnhancePluginDefine>();
```

```java
public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
    for (AbstractClassEnhancePluginDefine plugin : plugins) {
        ClassMatch match = plugin.enhanceClass();
        if (match == null) {
            continue;
        }
        if (match instanceof NameMatch) { //classname匹配
            NameMatch nameMatch = (NameMatch)match;
            LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
            if (pluginDefines == null) {
                pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
            }
            pluginDefines.add(plugin);
        } else { //其他匹配，比如注解
            signatureMatchDefine.add(plugin);
        }
    }
}
```

## Transformer

负责对类进行增强

org.apache.skywalking.apm.agent.SkyWalkingAgent.Transformer#transform

```java
public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder, TypeDescription typeDescription,
    ClassLoader classLoader, JavaModule module) { //增强
    //查找匹配该类的插件
    List<AbstractClassEnhancePluginDefine> pluginDefines = pluginFinder.find(typeDescription);
    if (pluginDefines.size() > 0) { //需要增强
        DynamicType.Builder<?> newBuilder = builder;
        EnhanceContext context = new EnhanceContext();
        for (AbstractClassEnhancePluginDefine define : pluginDefines) {
            DynamicType.Builder<?> possibleNewBuilder = define.define(typeDescription, newBuilder, classLoader, context);
            if (possibleNewBuilder != null) {
                newBuilder = possibleNewBuilder;
            }
        }
        if (context.isEnhanced()) {
            logger.debug("Finish the prepare stage for {}.", typeDescription.getName());
        }
        return newBuilder;
    }
    return builder;
}
```

### 查找匹配该类的插件

org.apache.skywalking.apm.agent.core.plugin.PluginFinder#find

```java
public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
    List<AbstractClassEnhancePluginDefine> matchedPlugins = new LinkedList<AbstractClassEnhancePluginDefine>();
  	//首先classname匹配
    String typeName = typeDescription.getTypeName();
    if (nameMatchDefine.containsKey(typeName)) {
        matchedPlugins.addAll(nameMatchDefine.get(typeName));
    }
    //然后其他匹配
    for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
        IndirectMatch match = (IndirectMatch)pluginDefine.enhanceClass();
        if (match.isMatch(typeDescription)) {
            matchedPlugins.add(pluginDefine);
        }
    }
    return matchedPlugins;
}
```

增强

org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#define

```java
public DynamicType.Builder<?> define(TypeDescription typeDescription,
                                     DynamicType.Builder<?> builder, ClassLoader classLoader, EnhanceContext context) throws PluginException {
    String interceptorDefineClassName = this.getClass().getName();
    String transformClassName = typeDescription.getTypeName();
    if (StringUtil.isEmpty(transformClassName)) {
        logger.warn("classname of being intercepted is not defined by {}.", interceptorDefineClassName);
        return null;
    }

    logger.debug("prepare to enhance class {} by {}.", transformClassName, interceptorDefineClassName);

    /**
     * find witness classes for enhance class
     */
    String[] witnessClasses = witnessClasses();
    if (witnessClasses != null) {
        for (String witnessClass : witnessClasses) {
            if (!WitnessClassFinder.INSTANCE.exist(witnessClass, classLoader)) {
                logger.warn("enhance class {} by plugin {} is not working. Because witness class {} is not existed.", transformClassName, interceptorDefineClassName,
                    witnessClass);
                return null;
            }
        }
    }

    /**
     * find origin class source code for interceptor
     */
    DynamicType.Builder<?> newClassBuilder = this.enhance(typeDescription, builder, classLoader, context);

    context.initializationStageCompleted();
    logger.debug("enhance class {} by {} completely.", transformClassName, interceptorDefineClassName);

    return newClassBuilder;
}
```

org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhance

```java
protected DynamicType.Builder<?> enhance(TypeDescription typeDescription,
                                         DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
                                         EnhanceContext context) throws PluginException {
  	//对静态方法增强
    newClassBuilder = this.enhanceClass(typeDescription, newClassBuilder, classLoader);
	  //对实例方法增强
    newClassBuilder = this.enhanceInstance(typeDescription, newClassBuilder, classLoader, context);

    return newClassBuilder;
}
```

org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhanceClass

```java
private DynamicType.Builder<?> enhanceClass(TypeDescription typeDescription,
    DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader) throws PluginException {
  	//获取静态方法拦截点(拦截的方法、拦截器)
    StaticMethodsInterceptPoint[] staticMethodsInterceptPoints = getStaticMethodsInterceptPoints();
    String enhanceOriginClassName = typeDescription.getTypeName();
  	//无需拦截
    if (staticMethodsInterceptPoints == null || staticMethodsInterceptPoints.length == 0) {
        return newClassBuilder;
    }

  	//增强
    for (StaticMethodsInterceptPoint staticMethodsInterceptPoint : staticMethodsInterceptPoints) {
        String interceptor = staticMethodsInterceptPoint.getMethodsInterceptor();
        if (StringUtil.isEmpty(interceptor)) {
            throw new EnhanceException("no StaticMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
        }

        if (staticMethodsInterceptPoint.isOverrideArgs()) {
            newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                .intercept(
                    MethodDelegation.withDefaultConfiguration()
                        .withBinders(
                            Morph.Binder.install(OverrideCallable.class)
                        )
                        .to(new StaticMethodsInterWithOverrideArgs(interceptor))
                );
        } else {
            newClassBuilder = newClassBuilder.method(isStatic().and(staticMethodsInterceptPoint.getMethodsMatcher()))
                .intercept(
                    MethodDelegation.withDefaultConfiguration()
                        .to(new StaticMethodsInter(interceptor))
                );
        }

    }

    return newClassBuilder;
}
```

org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhanceInstance

```java
private DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
    DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
    EnhanceContext context) throws PluginException {
  //构造函数拦截
    ConstructorInterceptPoint[] constructorInterceptPoints = getConstructorsInterceptPoints();
  //实例方法拦截
    InstanceMethodsInterceptPoint[] instanceMethodsInterceptPoints = getInstanceMethodsInterceptPoints();
    String enhanceOriginClassName = typeDescription.getTypeName();
    boolean existedConstructorInterceptPoint = false;
    if (constructorInterceptPoints != null && constructorInterceptPoints.length > 0) {
        existedConstructorInterceptPoint = true;
    }
    boolean existedMethodsInterceptPoints = false;
    if (instanceMethodsInterceptPoints != null && instanceMethodsInterceptPoints.length > 0) 		{
        existedMethodsInterceptPoints = true;
    }
		//无需拦截
    if (!existedConstructorInterceptPoint && !existedMethodsInterceptPoints) {
        return newClassBuilder;
    }

    /**
     * Manipulate class source code.<br/>
     *
     * new class need:<br/>
     * 1.Add field, name {@link #CONTEXT_ATTR_NAME}.
     * 2.Add a field accessor for this field.
     *
     * And make sure the source codes manipulation only occurs once.
     *
     */
  	
    if (!context.isObjectExtended()) { //尚未扩展
        newClassBuilder = newClassBuilder.defineField(CONTEXT_ATTR_NAME, Object.class, ACC_PRIVATE | ACC_VOLATILE)
            .implement(EnhancedInstance.class)
            .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME)); //添加字段
        context.extendObjectCompleted(); //设置已经扩展的标志
    }

    if (existedConstructorInterceptPoint) { //增强构造器
        for (ConstructorInterceptPoint constructorInterceptPoint : constructorInterceptPoints) {
            newClassBuilder = newClassBuilder.constructor(constructorInterceptPoint.getConstructorMatcher()).intercept(SuperMethodCall.INSTANCE
                .andThen(MethodDelegation.withDefaultConfiguration()
                    .to(new ConstructorInter(constructorInterceptPoint.getConstructorInterceptor(), classLoader))
                )
            );
        }
    }

    if (existedMethodsInterceptPoints) { //增强实例方法
        for (InstanceMethodsInterceptPoint instanceMethodsInterceptPoint : instanceMethodsInterceptPoints) {
            String interceptor = instanceMethodsInterceptPoint.getMethodsInterceptor();
            if (StringUtil.isEmpty(interceptor)) {
                throw new EnhanceException("no InstanceMethodsAroundInterceptor define to enhance class " + enhanceOriginClassName);
            }
            ElementMatcher.Junction<MethodDescription> junction = not(isStatic()).and(instanceMethodsInterceptPoint.getMethodsMatcher());
            if (instanceMethodsInterceptPoint instanceof DeclaredInstanceMethodsInterceptPoint) {
                junction = junction.and(ElementMatchers.<MethodDescription>isDeclaredBy(typeDescription));
            }
            if (instanceMethodsInterceptPoint.isOverrideArgs()) {
                newClassBuilder =
                    newClassBuilder.method(junction)
                        .intercept(
                            MethodDelegation.withDefaultConfiguration()
                                .withBinders(
                                    Morph.Binder.install(OverrideCallable.class)
                                )
                                .to(new InstMethodsInterWithOverrideArgs(interceptor, classLoader))
                        );
            } else {
                newClassBuilder =
                    newClassBuilder.method(junction)
                        .intercept(
                            MethodDelegation.withDefaultConfiguration()
                                .to(new InstMethodsInter(interceptor, classLoader))
                        );
            }
        }
    }

    return newClassBuilder;
}
```

##  BootService

org.apache.skywalking.apm.agent.core.boot.ServiceManager#boot

```java
public void boot() {
    bootedServices = loadAllServices();//加载BootService实现
    prepare(); // 调用全部BootService对象的prepare()方法
    startup(); // 调用全部BootService对象的boot()方法
    onComplete(); //调用全部BootService对象的onComplete()方法
}
```

org.apache.skywalking.apm.agent.core.boot.ServiceManager#load

```java
void load(List<BootService> allServices) {
  //使用JDKSPI技术加载并实例化META-INF/services下的全部BootService接口实现
    Iterator<BootService> iterator = ServiceLoader.load(BootService.class, AgentClassLoader.getDefault()).iterator();
    while (iterator.hasNext()) {
        allServices.add(iterator.next());
    }
}
```