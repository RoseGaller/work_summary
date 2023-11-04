# Jvm-Sandbox源码分析

利用了JVM的Attach机制实现

# 启动

com.alibaba.jvm.sandbox.core.CoreLauncher#main

```java
public static void main(String[] args) {
    try {
        // check args
        if (args.length != 3
                || StringUtils.isBlank(args[0]) //PID
                || StringUtils.isBlank(args[1]) // agent.jar's value
                || StringUtils.isBlank(args[2])) {//token
            throw new IllegalArgumentException("illegal args");
        }
        // call the core launcher
        new CoreLauncher(args[0], args[1], args[2]);
    } catch (Throwable t) {
        t.printStackTrace(System.err);
        System.err.println("sandbox load jvm failed : " + getCauseMessage(t));
        System.exit(-1);
    }
}
```

com.alibaba.jvm.sandbox.core.CoreLauncher#CoreLauncher

```java
public CoreLauncher(final String targetJvmPid,
                    final String agentJarPath,
                    final String token) throws Exception {

    // 加载agent
    attachAgent(targetJvmPid, agentJarPath, token);

}
```

com.alibaba.jvm.sandbox.core.CoreLauncher#attachAgent

```java
private void attachAgent(final String targetJvmPid,
                         final String agentJarPath,
                         final String cfg) throws Exception {

    final ClassLoader loader = Thread.currentThread().getContextClassLoader();
    final Class<?> vmdClass = loader.loadClass("com.sun.tools.attach.VirtualMachineDescriptor");
    final Class<?> vmClass = loader.loadClass("com.sun.tools.attach.VirtualMachine");

    Object attachVmdObj = null;
    for (Object obj : (List<?>) vmClass.getMethod("list", (Class<?>[]) null).invoke(null, (Object[]) null)) {
        if ((vmdClass.getMethod("id", (Class<?>[]) null).invoke(obj, (Object[]) null))
                .equals(targetJvmPid)) {
            attachVmdObj = obj;
        }
    }

    Object vmObj = null;
    try {
        // 使用 attach(String pid) 这种方式
        if (null == attachVmdObj) {
            vmObj = vmClass.getMethod("attach", String.class).invoke(null, targetJvmPid);
        } else {
            vmObj = vmClass.getMethod("attach", vmdClass).invoke(null, attachVmdObj);
        }
      	//加载agentJar,执行agentmain方法
        vmClass
                .getMethod("loadAgent", String.class, String.class)
                .invoke(vmObj, agentJarPath, cfg);
    } finally {
        if (null != vmObj) {
            vmClass.getMethod("detach", (Class<?>[]) null).invoke(vmObj, (Object[]) null);
        }
    }

}
```

com.alibaba.jvm.sandbox.agent.AgentLauncher#agentmain

```java
public static void agentmain(String cfg, Instrumentation inst) {


    final String[] cfgSegmentArray = cfg.split(";");

    // token
    final String token = cfgSegmentArray.length >= 1
            ? cfgSegmentArray[0]
            : "";

    // server ip
    final String ip = cfgSegmentArray.length >= 2
            ? cfgSegmentArray[1]
            : "127.0.0.1";

    // server port
    final String port = cfgSegmentArray.length >= 3
            ? cfgSegmentArray[2]
            : "0";

    if (token.matches("^\\s*$")) {
        throw new IllegalArgumentException("sandbox attach token was blank.");
    }

    final InetSocketAddress local = main(
            SANDBOX_CORE_JAR_PATH,
            String.format(";cfg=%s;system_module=%s;mode=%s;sandbox_home=%s;user_module=%s;provider=%s;server.ip=%s;server.port=%s;",
                    SANDBOX_CFG_PATH, SANDBOX_MODULE_PATH, LAUNCH_MODE_ATTACH, SANDBOX_HOME, SANDBOX_USER_MODULE_PATH, SANDBOX_PROVIDER_LIB_PATH, ip, port),
            SANDBOX_PROPERTIES_PATH,
            inst
    );
    writeTokenResult(token, local);
}
```

```java
private static synchronized InetSocketAddress main(final String coreJarPath,
                                                   final String coreFeatureString,
                                                   final String propertiesFilePath,
                                                   final Instrumentation inst) {

    try {

        // 将Spy注入到BootstrapClassLoader
        inst.appendToBootstrapClassLoaderSearch(new JarFile(new File(SANDBOX_SPY_JAR_PATH)));

        // 构造自定义的类加载器，尽量减少Sandbox对现有工程的侵蚀
        final ClassLoader agentLoader = loadOrDefineClassLoader(coreJarPath);

        // 加载CoreConfigure类
        final Class<?> classOfConfigure = agentLoader.loadClass(CLASS_OF_CORE_CONFIGURE);

        // 反序列化成CoreConfigure类实例
        final Object objectOfCoreConfigure = classOfConfigure.getMethod("toConfigure", String.class, String.class)
                .invoke(null, coreFeatureString, propertiesFilePath);

        //加载JettyCoreServer
        final Class<?> classOfJtServer = agentLoader.loadClass(CLASS_OF_JETTY_CORE_SERVER);

        // 获取JettyCoreServer单例
        final Object objectOfJtServer = classOfJtServer
                .getMethod("getInstance")
                .invoke(null);

        // 查看是否已经绑定
        final boolean isBind = (Boolean)classOfJtServer.getMethod("isBind").invoke(objectOfJtServer);


        // 如果未绑定,则需要绑定一个地址
        if (!isBind) {
            try {
                classOfJtServer
                        .getMethod("bind", classOfConfigure, Instrumentation.class)
                        .invoke(objectOfJtServer, objectOfCoreConfigure, inst);
            } catch (Throwable t) {
                classOfJtServer.getMethod("destroy").invoke(objectOfJtServer);
                throw t;
            }

        }

        // 返回服务器绑定的地址
        return (InetSocketAddress) classOfJtServer
                .getMethod("getLocal")
                .invoke(objectOfJtServer);


    } catch (Throwable cause) {
        throw new RuntimeException("sandbox attach failed.", cause);
    }

}
```

```java
private static synchronized void writeTokenResult(final String token, final InetSocketAddress local) {
    final File file = new File(RESULT_FILE_PATH);

    if (file.exists()
            && (!file.isFile()
            || !file.canWrite())) {
        throw new RuntimeException("write to result file : " + file + " failed.");
    } else {
        FileWriter fw = null;
        try {
            fw = new FileWriter(file, true);
            fw.append(String.format("%s;%s;%s;\n", token, local.getHostName(), local.getPort()));
            fw.flush();
        } catch (IOException e) {
            throw new RuntimeException(e);
        } finally {
            if (null != fw) {
                try {
                    fw.close();
                } catch (IOException e) {
                    // ignore
                }
            }
        }
    }
}
```

# 启动Jetty

com.alibaba.jvm.sandbox.core.server.jetty.JettyCoreServer#bind

```java
public synchronized void bind(final CoreConfigure cfg, final Instrumentation inst) throws IOException {
    try {
        initializer.initProcess(new Initializer.Processor() {
            @Override
            public void process() throws Throwable {

                // logger.info("prepare init sandbox start.");

                initLogback(cfg);
                logger.debug("init logback finished.");
                logger.info("cfg={}", cfg.toString());

                initManager(inst, cfg);
                logger.debug("init resource finished.");

                initHttpServer(cfg);
                logger.debug("init http-server finished.");

                initJettyContextHandler();
                logger.debug("init servlet finished.");

                httpServer.start();
                logger.debug("http-server started.");

                logger.info("sandbox start finished.");

            }
        });
    } catch (Throwable cause) {
        logger.warn("server bind failed. cfg={}", cfg, cause);
        throw new IOException(
                String.format("server bind to %s:%s failed.", cfg.getServerIp(), cfg.getServerPort()),
                cause
        );
    }

    logger.info("server bind to {} success. cfg={}", getLocal(), cfg);
}
```

com.alibaba.jvm.sandbox.core.server.jetty.JettyCoreServer#initManager

```java
private void initManager(final Instrumentation inst,
                         final CoreConfigure cfg) {

    logger.debug("{} was init", EventListenerHandlers.getSingleton());

    final ModuleLifeCycleEventBus moduleLifeCycleEventBus = new DefaultModuleLifeCycleEventBus();
    final LoadedClassDataSource classDataSource = new DefaultLoadedClassDataSource(inst);
    final ClassLoader sandboxClassLoader = getClass().getClassLoader();

    this.moduleResourceManager = new DefaultModuleResourceManager();
    moduleLifeCycleEventBus.append(this.moduleResourceManager);

    // 初始化服务管理器
    final ProviderManager providerManager = new DefaultProviderManager(cfg, sandboxClassLoader);

    // 初始化模块管理器
    this.coreModuleManager = new DefaultCoreModuleManager(
            inst, classDataSource, cfg, sandboxClassLoader, moduleLifeCycleEventBus, providerManager
    );
}
```

## DefaultProviderManager

com.alibaba.jvm.sandbox.core.manager.impl.DefaultProviderManager#DefaultProviderManager

```java
public DefaultProviderManager(final CoreConfigure cfg,
                              final ClassLoader sandboxClassLoader) {
   //默认服务提供管理器实现
    this.cfg = cfg;
    try {
        init(cfg, sandboxClassLoader);
    } catch (Throwable cause) {
        logger.warn("loading sandbox's provider-lib[{}] failed.", cfg.getProviderLibPath(), cause);
    }
}
```

```java
private void init(final CoreConfigure cfg,
                  final ClassLoader sandboxClassLoader) {
    final File providerLibDir = new File(cfg.getProviderLibPath());
    if (!providerLibDir.exists()
            || !providerLibDir.canRead()) {
        logger.warn("loading provider-lib[{}] was failed, doest existed or access denied.", providerLibDir);
        return;
    }

    for (final File providerJarFile : FileUtils.listFiles(providerLibDir, new String[]{"jar"}, false)) {

        try {
            final ProviderClassLoader providerClassLoader = new ProviderClassLoader(providerJarFile, sandboxClassLoader);

            // load ModuleJarLoadingChain
            inject(moduleJarLoadingChains, ModuleJarLoadingChain.class, providerClassLoader, providerJarFile);

            // load ModuleLoadingChain
            inject(moduleLoadingChains, ModuleLoadingChain.class, providerClassLoader, providerJarFile);

            logger.info("loading provider-jar[{}] was success.", providerJarFile);
        } catch (IllegalAccessException cause) {
            logger.warn("loading provider-jar[{}] occur error, inject provider resource failed.", providerJarFile, cause);
        } catch (IOException ioe) {
            logger.warn("loading provider-jar[{}] occur error, ignore load this provider.", providerJarFile, ioe);
        }
    }
}
```

## DefaultCoreModuleManager

com.alibaba.jvm.sandbox.core.manager.impl.DefaultCoreModuleManager#DefaultCoreModuleManager

```java
public DefaultCoreModuleManager(final Instrumentation inst,
                                final LoadedClassDataSource classDataSource,
                                final CoreConfigure cfg,
                                final ClassLoader sandboxClassLoader,
                                final ModuleLifeCycleEventBus moduleLifeCycleEventBus,
                                final ProviderManager providerManager) {
   //模块管理器
    this.inst = inst;
    this.classDataSource = classDataSource;
    this.cfg = cfg;
    this.sandboxClassLoader = sandboxClassLoader;
    this.moduleLifeCycleEventBus = moduleLifeCycleEventBus;
    this.providerManager = providerManager;

    // 初始化模块目录
    this.moduleLibDirArray = mergeFileArray(
            new File[]{new File(cfg.getSystemModuleLibPath())},
            cfg.getUserModuleLibFilesWithCache()
    );

    // 初始化加载所有的模块
    try {
        reset();
    } catch (ModuleException e) {
        logger.warn("init module[id={};] occur error={}.", e.getUniqueId(), e.getErrorCode(), e);
    }
}
```

com.alibaba.jvm.sandbox.core.manager.impl.DefaultCoreModuleManager#reset

```java
public synchronized void reset() throws ModuleException {

    // 1. 强制卸载所有模块
    for (final CoreModule coreModule : new ArrayList<CoreModule>(loadedModuleBOMap.values())) {
        unload(coreModule, true);
    }

    // 2. 加载所有模块
    for (final File moduleLibDir : moduleLibDirArray) {
        // 用户模块加载目录，加载用户模块目录下的所有模块
        // 对模块访问权限进行校验
        if (moduleLibDir.exists()
                && moduleLibDir.canRead()) {
            new ModuleJarLoader(moduleLibDir, cfg.getLaunchMode(), sandboxClassLoader)
                    .load(new InnerModuleJarLoadCallback(), new InnerModuleLoadCallback());
        } else {
            logger.warn("MODULE-LIB[{}] can not access, ignore flush load this lib.", moduleLibDir);
        }
    }

}
```

com.alibaba.jvm.sandbox.core.manager.impl.ModuleJarLoader#load

```java
public void load(final ModuleJarLoadCallback mjCb,
                 final ModuleLoadCallback mCb) {

    // 查找所有可加载的Jar文件
    final File[] moduleJarFileArray = toModuleJarFileArray();
    Arrays.sort(moduleJarFileArray);
    logger.info("module loaded found {} jar in {}", moduleJarFileArray.length, moduleLibDir);

    // 开始逐条加载
    for (final File moduleJarFile : moduleJarFileArray) {

        ModuleClassLoader moduleClassLoader = null;

        try {

            // 是否有模块加载成功
            boolean hasModuleLoadedSuccessFlag = false;

            if (null != mjCb) {
                mjCb.onLoad(moduleJarFile);
            }

            // 模块ClassLoader
            moduleClassLoader = new ModuleClassLoader(moduleJarFile, sandboxClassLoader);
						//加载所有实现Module接口的类
            final ServiceLoader<Module> moduleServiceLoader = ServiceLoader.load(Module.class, moduleClassLoader);
            final Iterator<Module> moduleIt = moduleServiceLoader.iterator();
            while (moduleIt.hasNext()) {

                final Module module;
                try {
                    module = moduleIt.next();
                } catch (Throwable cause) {
                    logger.warn("module SPI new instance failed, ignore this SPI.", cause);
                    continue;
                }

                final Class<?> classOfModule = module.getClass();
								//类上必须标有Information注解
                if (!classOfModule.isAnnotationPresent(Information.class)) {
                    logger.info("{} was not implements @Information, ignore this SPI.", classOfModule);
                    continue;
                }
								//Information有唯一标志
                final Information info = classOfModule.getAnnotation(Information.class);
                if (StringUtils.isBlank(info.id())) {
                    logger.info("{} was not implements @Information[id], ignore this SPI.", classOfModule);
                    continue;
                }

                final String uniqueId = info.id();
                if (!ArrayUtils.contains(info.mode(), mode)) {
                    logger.info("module[id={};class={};mode={};] was not matching sandbox launch mode : {}, ignore this SPI.",
                            uniqueId, classOfModule, Arrays.asList(info.mode()), mode);
                    continue;
                }

                try {
                    if (null != mCb) {
                        mCb.onLoad(
                                uniqueId, classOfModule, module, moduleJarFile,
                                moduleClassLoader
                        );
                    }
                    hasModuleLoadedSuccessFlag = true;
                } catch (Throwable cause) {
                    logger.warn("load module[id={};class={};] from JAR[file={};] failed, ignore this module.",
                            uniqueId, classOfModule, moduleJarFile, cause);
                }

            }//while

            if (!hasModuleLoadedSuccessFlag) {
                logger.warn("load sandbox module JAR[file={}], but none module loaded, close this ModuleClassLoader.", moduleJarFile);
                moduleClassLoader.closeIfPossible();
            }

        } catch (Throwable cause) {
            logger.warn("load sandbox module JAR[file={}] failed.", moduleJarFile, cause);
            if (null != moduleClassLoader) {
                moduleClassLoader.closeIfPossible();
            }
        }

    }

}
```

com.alibaba.jvm.sandbox.core.manager.impl.DefaultCoreModuleManager.InnerModuleLoadCallback#onLoad

```java
public void onLoad(final String uniqueId,
                   final Class moduleClass,
                   final Module module,
                   final File moduleJarFile,
                   final ModuleClassLoader moduleClassLoader) throws Throwable {

    // 如果之前已经加载过了相同ID的模块，则放弃当前模块的加载
    if (loadedModuleBOMap.containsKey(uniqueId)) {
        final CoreModule existedCoreModule = get(uniqueId);
        logger.info("module[id={};class={};loader={};] already loaded, ignore load this module. existed-module[id={};class={};loader={};]",
                uniqueId, moduleClass, moduleClassLoader,
                existedCoreModule.getUniqueId(), existedCoreModule.getModule().getClass(), existedCoreModule.getLoader());
        return;
    }

    // 需要经过ModuleLoadingChain的过滤
    providerManager.loading(
            uniqueId,
            moduleClass,
            module,
            moduleJarFile,
            moduleClassLoader
    );

    // 之前没有加载过，这里进行加载
    logger.debug("found new module[id={};class={};loader={};], prepare to load.",
            uniqueId, moduleClass, moduleClassLoader);
    load(uniqueId, module, moduleJarFile, moduleClassLoader);
}
```

com.alibaba.jvm.sandbox.core.manager.impl.DefaultCoreModuleManager#load

```java
private synchronized void load(final String uniqueId,
                               final Module module,
                               final File moduleJarFile,
                               final ModuleClassLoader moduleClassLoader) throws ModuleException {

    if (loadedModuleBOMap.containsKey(uniqueId)) {
        logger.info("module[id={};] already loaded, ignore this load.", uniqueId);
        return;
    }

    // 初始化模块信息
    final CoreModule coreModule = new CoreModule(uniqueId, moduleJarFile, moduleClassLoader, module);

    // 注入@Resource资源
    try {
        injectRequiredResource(coreModule);
    } catch (IllegalAccessException iae) {
        throw new ModuleException(uniqueId, MODULE_LOAD_ERROR, iae);
    }

    // 通知生命周期:模块加载开始
    fireModuleLifecycle(coreModule, ModuleLifeCycleEventBus.Event.LOAD);

    // 设置为已经加载
    coreModule.setLoaded(true);

    // 如果模块被标记为启动时激活，这里需要主动对模块进行一次激活操作
    final Information info = module.getClass().getAnnotation(Information.class);
    if (info.isActiveOnLoad()) {
        active(coreModule);
    }

    // 注册到模块列表中
    loadedModuleBOMap.put(uniqueId, coreModule);

    // 通知声明周期，模块加载完成
    fireModuleLifecycle(coreModule, ModuleLifeCycleEventBus.Event.LOAD_COMPLETED);

    logger.info("loaded module[id={};class={};] success, loader={}", uniqueId, module.getClass(), moduleClassLoader);

}
```

## InitJettyContextHandler

com.alibaba.jvm.sandbox.core.server.jetty.JettyCoreServer#initJettyContextHandler

```java
private void initJettyContextHandler() { //处理客户端请求
    final ServletContextHandler context = new ServletContextHandler(SESSIONS);

    // websocket-servlet
    context.addServlet(new ServletHolder(new WebSocketAcceptorServlet(coreModuleManager, moduleResourceManager)), "/module/websocket/*");

    // module-http-servlet
    context.addServlet(new ServletHolder(new ModuleHttpServlet(coreModuleManager, moduleResourceManager)), "/module/http/*"); 

    context.setContextPath("/sandbox");
    context.setClassLoader(getClass().getClassLoader());
    httpServer.setHandler(context);
}
```

# 测试用例

```java
@Information(id = "broken-clock-tinker") //模块名称
public class BrokenClockTinkerModule implements Module {
  @Resource
  private ModuleEventWatcher moduleEventWatcher; //启动时，自动加载所有模块，并自动注入

  @Http("/repairCheckState") //请求方法
  public void repairCheckState() {

      moduleEventWatcher.watch(

              // 匹配到Clock$BrokenClock#checkState()
              new NameRegexFilter("Clock\\$BrokenClock", "checkState"), //类、方法的拦截

              // 监听THROWS事件并且改变原有方法抛出异常为正常返回
              new EventListener() {
                  @Override
                  public void onEvent(Event event) throws Throwable {
                      // 立即返回
                      ProcessControlException.throwReturnImmediately(null);
                  }
              }, //增强

              // 指定监听的事件为抛出异常
              Event.Type.THROWS //事件类型 before、after、throws
      );
  }
}
```

# 接收请求进行增强

com.alibaba.jvm.sandbox.core.server.jetty.servlet.ModuleHttpServlet#doMethod

```java
private void doMethod(final HttpServletRequest req,
                      final HttpServletResponse resp,
                      final Http.Method httpMethod) throws ServletException, IOException {
    //./sandbox.sh -p 64229 -d 'broken-clock-tinker/repairCheckState'
    // 获取请求路径
    final String path = req.getPathInfo();

    // 获取模块ID
    final String uniqueId = parseUniqueId(path);
    if (StringUtils.isBlank(uniqueId)) {
        logger.warn("http request value={} was not found.", path);
        resp.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // 获取模块
    final CoreModule coreModule = coreModuleManager.get(uniqueId);
    if (null == coreModule) {
        logger.warn("module[id={}] was not existed, value={};", uniqueId, path);
        resp.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // 匹配对应的方法
    final Method method = matchingModuleMethod(
            path,
            httpMethod,
            uniqueId,
            coreModule.getModule().getClass()
    );
    if (null == method) {
        logger.warn("module[id={};class={};] request method not found, value={};",
                uniqueId,
                coreModule.getModule().getClass(),
                path
        );
        resp.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    } else {
        logger.debug("found method[class={};method={};] in module[id={};class={};]",
                method.getDeclaringClass().getName(),
                method.getName(),
                uniqueId,
                coreModule.getModule().getClass()
        );
    }

    // 生成方法调用参数
    final Object[] parameterObjectArray = generateParameterObjectArray(method, req, resp);

    final PrintWriter writer = resp.getWriter();

    
    moduleResourceManager.append(uniqueId,
            new ModuleResourceManager.WeakResource<PrintWriter>(writer) {

                @Override
                public void release() {
                    IOUtils.closeQuietly(get());
                }

            });
    final boolean isAccessible = method.isAccessible();
    final ClassLoader oriThreadContextClassLoader = Thread.currentThread().getContextClassLoader();
    try {
        // 调用方法
        method.setAccessible(true);
        Thread.currentThread().setContextClassLoader(coreModule.getLoader());
        method.invoke(coreModule.getModule(), parameterObjectArray);
        logger.debug("http request value={} invoke module[id={};] {}#{} success.",
                path, uniqueId, coreModule.getModule().getClass(), method.getName());
    } catch (IllegalAccessException iae) {
        logger.warn("impossible, http request value={} invoke module[id={};] {}#{} occur access denied.",
                path, uniqueId, coreModule.getModule().getClass(), method.getName(), iae);
        throw new ServletException(iae);
    } catch (InvocationTargetException ite) {
        logger.warn("http request value={} invoke module[id={};] {}#{} failed.",
                path, uniqueId, coreModule.getModule().getClass(), method.getName(), ite.getTargetException());
        final Throwable targetCause = ite.getTargetException();
        if (targetCause instanceof ServletException) {
            throw (ServletException) targetCause;
        }
        if (targetCause instanceof IOException) {
            throw (IOException) targetCause;
        }
        throw new ServletException(targetCause);
    } finally {
        Thread.currentThread().setContextClassLoader(oriThreadContextClassLoader);
        method.setAccessible(isAccessible);
        moduleResourceManager.remove(uniqueId, writer);
    }

}
```

## DefaultModuleEventWatcher

com.alibaba.jvm.sandbox.core.manager.impl.DefaultModuleEventWatcher#watch(com.alibaba.jvm.sandbox.api.filter.Filter, com.alibaba.jvm.sandbox.api.listener.EventListener, com.alibaba.jvm.sandbox.api.resource.ModuleEventWatcher.Progress, com.alibaba.jvm.sandbox.api.event.Event.Type...)

```java
public int watch(final Filter filter, //帅选出需要增强的方法
                 final EventListener listener, //如何增强
                 final Progress progress, //增强的进度
                 final Event.Type... eventType) { //类型（ before、after、throws、CALL_BEFORE）
  	//生成唯一标志
    final int watchId = watchIdSequencer.next();

    // 创建对应的ClassFileTransformer
    final SandboxClassFileTransformer sandClassFileTransformer = new SandboxClassFileTransformer(
            watchId,
            coreModule.getUniqueId(),
            filter,
            listener,
            isEnableUnsafe,
            eventType
    );

    // 注册到CoreModule中
    coreModule.getSandboxClassFileTransformers().add(sandClassFileTransformer);

    // 注册到JVM加载上ClassFileTransformer处理新增的类
    inst.addTransformer(sandClassFileTransformer, true);

    // 查找需要增强的类集合
    final Set<Class<?>> waitingReTransformClassSet = remove$$Lambda$(classDataSource.find(filter));

    int cCnt = 0, mCnt = 0;

    // 进度通知启动
    beginProgress(progress, waitingReTransformClassSet.size());
    try {

        // 应用JVM
        reTransformClasses(watchId, waitingReTransformClassSet, progress);

        // 计数
        cCnt += sandClassFileTransformer.cCnt();
        mCnt += sandClassFileTransformer.mCnt();


        // 激活增强类
        if (coreModule.isActivated()) {
            final int listenerId = sandClassFileTransformer.getListenerId();
            EventListenerHandlers.getSingleton()
                    .active(listenerId, listener, eventType);
        }

    } finally {
        finishProgress(progress, cCnt, mCnt);
    }

    return watchId;
}
```

### 增强

com.alibaba.jvm.sandbox.core.manager.impl.DefaultModuleEventWatcher#reTransformClasses

```java
private void reTransformClasses(final int watchId,
                                final Set<Class<?>> waitingReTransformClassSet,
                                final Progress progress) {

    // 找出当前已经被加载的所有类
    // final Set<Class<?>> waitingReTransformClassSet = classDataSource.find(filter);

    // 需要形变总数
    final int total = waitingReTransformClassSet.size();

    // 如果找不到需要被重新增强的类则直接返回
    if (CollectionUtils.isEmpty(waitingReTransformClassSet)) {
        logger.info("waitingReTransformClassSet is empty, ignore reTransformClasses. module[id={};];watchId={};",
                coreModule.getUniqueId(), watchId);
        return;
    }

    logger.debug("reTransformClasses={};module[id={};];watchId={};",
            waitingReTransformClassSet, coreModule.getUniqueId(), watchId);

    // 如果不需要进行进度汇报,则可以进行批量形变
    boolean batchReTransformSuccess = true;
    if (null == progress) {
        try {
            inst.retransformClasses(waitingReTransformClassSet.toArray(new Class[waitingReTransformClassSet.size()]));
            logger.info("batch reTransform classes, watchId={};total={};", watchId, waitingReTransformClassSet.size());
        } catch (Throwable e) {
            logger.warn("batch reTransform class failed, watchId={};", watchId, e);
            batchReTransformSuccess = false;
        }
    }

    // 只有两种情况需要进行逐个形变
    // 1. 需要进行形变进度报告,则只能一个个进行形变
    // 2. 批量形变失败,需要转换为单个形变,以观察具体是哪个形变失败
    if (!batchReTransformSuccess
            || null != progress) {
        int failTotal = 0;
        int index = 0;
        for (final Class<?> waitingReTransformClass : waitingReTransformClassSet) {
            index++;
            try {
                if (null != progress) {
                    try {
                        progress.progressOnSuccess(waitingReTransformClass, index);
                    } catch (Throwable cause) {
                        // 在进行进度汇报的过程中抛出异常,直接进行忽略,因为不影响形变的主体流程
                        // 仅仅只是一个汇报作用而已
                        logger.warn("report progressOnSuccess occur exception, watchId={};waitingReTransformClass={};index={};total={};",
                                watchId, waitingReTransformClass, index - 1, total, cause);
                    }
                }
                inst.retransformClasses(waitingReTransformClass);
                logger.debug("single reTransform class={};index={};total={};", waitingReTransformClass, index - 1, total);
            } catch (Throwable causeOfReTransform) {
                logger.warn("reTransformClass={}; failed, ignore this class.", waitingReTransformClass, causeOfReTransform);
                if (null != progress) {
                    try {
                        progress.progressOnFailed(waitingReTransformClass, index, causeOfReTransform);
                    } catch (Throwable cause) {
                        logger.warn("report progressOnFailed occur exception, watchId={};waitingReTransformClass={};index={};total={};cause={};",
                                watchId, waitingReTransformClass, index - 1, total, getCauseMessage(causeOfReTransform), cause);
                    }
                }
                failTotal++;
            }
        }//for
        logger.info("single reTransform classes, total={};failed={};", waitingReTransformClassSet.size(), failTotal);
    }

}
```

### 激活

com.alibaba.jvm.sandbox.core.enhance.weaver.EventListenerHandlers#active

```java
public void active(final int listenerId,
                   final EventListener listener,
                   final Event.Type[] eventTypeArray) {
  	//listenerId = ObjectIDs.instance.identity(eventListener);
  	//listenerId->EventListenerWrap
    final EventListenerWrap wrap = new EventListenerWrap(listener, eventTypeArray);
    globalEventListenerMap.put(listenerId, wrap);
    logger.info("active listener success. listener-id={};listener={};", listenerId, listener);
}
```

## Transformer

com.alibaba.jvm.sandbox.core.manager.impl.SandboxClassFileTransformer#transform

```java
public byte[] transform(final ClassLoader loader,
                        final String javaClassName,
                        final Class<?> classBeingRedefined,
                        final ProtectionDomain protectionDomain,
                        final byte[] srcByteCodeArray) throws IllegalClassFormatException {
		//计算唯一编码前缀s
    final String uniqueCodePrefix = computeUniqueCodePrefix(loader, javaClassName);
		//创建事件代码增强器
    final Enhancer enhancer = new EventEnhancer(
            listenerId,
            filter,
            uniqueCodePrefix,
            affectMethodUniqueSet,
            isEnableUnsafe,
            eventTypeArray
    );
    try {
      	//生成增强之后的代码
        final byte[] toByteCodeArray = enhancer.toByteCodeArray(loader, srcByteCodeArray);
        if (srcByteCodeArray == toByteCodeArray) {
            logger.debug("enhancer ignore this class={}", javaClassName);
            return null;
        }

        // affect count
        affectClassUniqueSet.add(uniqueCodePrefix);

        logger.info("enhancer toByteCode success, module[id={}];class={};loader={};", uniqueId, javaClassName, loader);
        return toByteCodeArray;
    } catch (Throwable cause) {
        logger.warn("enhancer toByteCode failed, module[id={}];class={};loader={};", uniqueId, javaClassName, loader, cause);
        // throw new IllegalClassFormatException(cause.getMessage());
        return null;
    }
}
```

## EventEnhancer

com.alibaba.jvm.sandbox.core.enhance.EventEnhancer#toByteCodeArray

```java
public byte[] toByteCodeArray(final ClassLoader targetClassLoader,
                              final byte[] byteCodeArray) {//获取增强后的字节码

    final AtomicBoolean reWriteMark = new AtomicBoolean(false);
    final ClassReader cr = new ClassReader(byteCodeArray);

    // 如果目标对象不在类匹配范围,则主动忽略增强
    if (isIgnoreClass(cr, targetClassLoader)) {
        logger.debug("class={} is ignore by filter, return origin bytecode", cr.getClassName());
        return byteCodeArray;
    }
    logger.debug("class={} is matched filter, prepare to enhance.", cr.getClassName());

    // 如果定义间谍类失败了,则后续不需要增强
    try {
        SpyUtils.init();
        // defineSpyIfNecessary(targetClassLoader);
    } catch (Throwable cause) {
        logger.warn("define Spy to target ClassLoader={} failed. class={}", targetClassLoader, cr.getClassName(), cause);
        return byteCodeArray;
    }

    // 返回增强后字节码
    final byte[] returnByteCodeArray = weavingEvent(targetClassLoader, byteCodeArray, reWriteMark);
    logger.debug("enhance class={} success, before bytecode.size={}; after bytecode.size={}; reWriteMark={}",
            cr.getClassName(),
            ArrayUtils.getLength(byteCodeArray),
            ArrayUtils.getLength(returnByteCodeArray),
            reWriteMark.get()
    );
    if (reWriteMark.get()) { //重写成功
        return returnByteCodeArray;
    } else {
        return byteCodeArray;
    }
}
```

com.alibaba.jvm.sandbox.core.enhance.EventEnhancer#weavingEvent

```java
private byte[] weavingEvent(final ClassLoader targetClassLoader,
                            final byte[] sourceByteCodeArray,
                            final AtomicBoolean reWriteMark) {
    final ClassReader cr = new ClassReader(sourceByteCodeArray);
    final ClassWriter cw = createClassWriter(targetClassLoader, cr);
    final int targetClassLoaderObjectID = ObjectIDs.instance.identity(targetClassLoader);
    cr.accept(
            new EventWeaver(
                    Opcodes.ASM5, cw, listenerId,
                    targetClassLoaderObjectID,
                    cr.getClassName(),
                    filter,
                    reWriteMark,
                    uniqueCodePrefix,
                    affectMethodUniqueSet,
                    eventTypeArray
            ),
            EXPAND_FRAMES
    );
    return cw.toByteArray();
    // return dumpClassIfNecessary(SandboxStringUtils.toJavaClassName(cr.getClassName()), cw.toByteArray());
}
```

## EventWeaver

方法事件编织者

com.alibaba.jvm.sandbox.core.enhance.weaver.asm.EventWeaver#visitMethod

```java
    public MethodVisitor visitMethod(final int access, final String name, final String desc, final String signature, final String[] exceptions) {

        final MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
        if (isIgnore(mv, access, name, desc, exceptions)) {
            logger.debug("{}.{} was ignored, listener-id={}", targetClassInternalName, name, listenerId);
            return mv;
        }

        logger.debug("{}.{} was matched, prepare to rewrite, listener-id={}", targetClassAsmType, name, listenerId);

        // mark reWrite
        reWriteMark.set(true);

        // 进入方法去重
        affectMethodUniqueSet.add(uniqueCodePrefix + name + desc + signature);

        return new ReWriteMethod(api, new JSRInlinerAdapter(mv, access, name, desc, signature, exceptions), access, name, desc) {

            private final Label beginLabel = new Label();
            private final Label endLabel = new Label();

            // 用来标记一个方法是否已经进入
            // JVM中的构造函数非常特殊，super();this();是在构造函数方法体执行之外进行，如果在这个之前进行了任何的流程改变操作
            // 将会被JVM加载类的时候判定校验失败，导致类加载出错
            // 所以这里需要用一个标记为告知后续的代码编织，绕开super()和this()
            private boolean isMethodEnter = false;

            // 代码锁
            private final CodeLock codeLockForTracing = new CallAsmCodeLock(this);

            /**
             * 流程控制
             */
            private void processControl() {
                final Label finishLabel = new Label();
                final Label returnLabel = new Label();
                final Label throwsLabel = new Label();
                dup();
                visitFieldInsn(GETFIELD, ASM_TYPE_SPY_RET, "state", ASM_TYPE_INT);
                dup();
                push(Spy.Ret.RET_STATE_RETURN);
                ifICmp(EQ, returnLabel);
                push(Spy.Ret.RET_STATE_THROWS);
                ifICmp(EQ, throwsLabel);
                goTo(finishLabel);
                mark(returnLabel);
                pop();
                visitFieldInsn(GETFIELD, ASM_TYPE_SPY_RET, "respond", ASM_TYPE_OBJECT);
                checkCastReturn(Type.getReturnType(desc));
                goTo(finishLabel);
                mark(throwsLabel);
                visitFieldInsn(GETFIELD, ASM_TYPE_SPY_RET, "respond", ASM_TYPE_OBJECT);
                checkCast(ASM_TYPE_THROWABLE);
                throwException();
                mark(finishLabel);
                pop();
            }

            /**
             * 加载ClassLoader
             * 这里分开静态方法中ClassLoader的获取以及普通方法中ClassLoader的获取
             * 主要是性能上的考虑
             */
            private void loadClassLoader() {
                push(targetClassLoaderObjectID);
            }

            @Override
            protected void onMethodEnter() {//进入方法前执行

                isMethodEnter = true;
                mark(beginLabel);

                codeLockForTracing.lock(new CodeLock.Block() {
                    @Override
                    public void code() {
                        loadArgArray();
                        dup();
                        push(listenerId);
                        loadClassLoader();
                        push(targetJavaClassName);
                        push(name);
                        push(desc);
                        loadThisOrPushNullIfIsStatic();
                        invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnBefore);
                        swap();
                        storeArgArray();
                        pop();
                        processControl();
                    }
                });
            }

            /**
             * 是否抛出异常返回(通过字节码判断)
             *
             * @param opcode 操作码
             * @return true:以抛异常形式返回 / false:非抛异常形式返回(return)
             */
            private boolean isThrow(int opcode) {
                return opcode == ATHROW;
            }

            /**
             * 加载返回值
             * @param opcode 操作吗
             */
            private void loadReturn(int opcode) {
                switch (opcode) {

                    case RETURN: {
                        pushNull();
                        break;
                    }

                    case ARETURN: {
                        dup();
                        break;
                    }

                    case LRETURN:
                    case DRETURN: {
                        dup2();
                        box(Type.getReturnType(methodDesc));
                        break;
                    }

                    default: {
                        dup();
                        box(Type.getReturnType(methodDesc));
                        break;
                    }

                }
            }

            @Override
            protected void onMethodExit(final int opcode) { //方法退出时执行
                if (!isThrow(opcode)) {
                    codeLockForTracing.lock(new CodeLock.Block() {
                        @Override
                        public void code() {
                            loadReturn(opcode);
                            push(listenerId);
                            invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnReturn);
                            processControl();
                        }
                    });
                }
            }

            /**
             * 加载异常
             */
            private void loadThrow() {
                dup();
            }

            @Override
            public void visitMaxs(int maxStack, int maxLocals) {
                mark(endLabel);
                visitTryCatchBlock(beginLabel, endLabel, mark(), ASM_TYPE_THROWABLE.getInternalName());

                codeLockForTracing.lock(new CodeLock.Block() {
                    @Override
                    public void code() {
                        loadThrow();
                        push(listenerId);
                        invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnThrows);
                        processControl();
                    }
                });

                throwException();
                super.visitMaxs(maxStack, maxLocals);
            }

            // 用于tracing的当前行号
            private int tracingCurrentLineNumber = -1;

            @Override
            public void visitLineNumber(final int lineNumber, Label label) {
                if (isMethodEnter && isLineEnable) {
                    codeLockForTracing.lock(new CodeLock.Block() {
                        @Override
                        public void code() {
                            push(lineNumber);
                            push(listenerId);
                            invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnLine);
                        }
                    });
                }
                super.visitLineNumber(lineNumber, label);
                this.tracingCurrentLineNumber = lineNumber;
            }

            @Override
            public void visitInsn(int opcode) {
                super.visitInsn(opcode);
                codeLockForTracing.code(opcode);
            }

            @Override
            public void visitMethodInsn(final int opcode,
                                        final String owner,
                                        final String name,
                                        final String desc,
                                        final boolean itf) {

                // 如果CALL事件没有启用，则不需要对CALL进行增强
                // 如果正在CALL的方法来自于SANDBOX本身，则不需要进行追踪
                if (!isMethodEnter || !isCallEnable || codeLockForTracing.isLock()) {
                    super.visitMethodInsn(opcode, owner, name, desc, itf);
                    return;
                }

                if (hasCallBefore) {
                    // 方法调用前通知
                    codeLockForTracing.lock(new CodeLock.Block() {
                        @Override
                        public void code() {
                            push(tracingCurrentLineNumber);
                            push(toJavaClassName(owner));
                            push(name);
                            push(desc);
                            push(listenerId);
                            invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnCallBefore);
                        }
                    });
                }

                // 如果没有CALL_THROWS事件,其实是可以不用对方法调用进行try...catch
                // 这样可以节省大量的字节码
                if (!hasCallThrows) {
                    super.visitMethodInsn(opcode, owner, name, desc, itf);
                    codeLockForTracing.lock(new CodeLock.Block() {
                        @Override
                        public void code() {
                            push(listenerId);
                            invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnCallReturn);
                        }
                    });
                    return;
                }


                // 这里是需要处理拥有CALL_THROWS事件的场景
                final Label tracingBeginLabel = new Label();
                final Label tracingEndLabel = new Label();
                final Label tracingFinallyLabel = new Label();

                // try
                // {

                mark(tracingBeginLabel);
                super.visitMethodInsn(opcode, owner, name, desc, itf);
                mark(tracingEndLabel);

                if (hasCallReturn) {
                    // 方法调用后通知
                    codeLockForTracing.lock(new CodeLock.Block() {
                        @Override
                        public void code() {
                            push(listenerId);
                            invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnCallReturn);
                        }
                    });
                }
                goTo(tracingFinallyLabel);

                // }
                // catch
                // {

                catchException(tracingBeginLabel, tracingEndLabel, ASM_TYPE_THROWABLE);
                codeLockForTracing.lock(new CodeLock.Block() {  //方法异常通知
                    @Override
                    public void code() {
                        dup();
                        invokeVirtual(ASM_TYPE_OBJECT, ASM_METHOD_Object$getClass);
                        invokeVirtual(ASM_TYPE_CLASS, ASM_METHOD_Class$getName);
                        push(listenerId);
                        invokeStatic(ASM_TYPE_SPY, ASM_METHOD_Spy$spyMethodOnCallThrows);
                    }
                });

                throwException();

                // }
                // finally
                // {
                mark(tracingFinallyLabel);
                // }

            }

            // 用于try-catch的重排序
            // 目的是让call的try...catch能在exceptions tables排在前边
            private final ArrayList<AsmTryCatchBlock> asmTryCatchBlocks = new ArrayList<AsmTryCatchBlock>();

            @Override
            public void visitTryCatchBlock(Label start, Label end, Label handler, String type) {
                asmTryCatchBlocks.add(new AsmTryCatchBlock(start, end, handler, type));
            }

            @Override
            public void visitEnd() {
                for (AsmTryCatchBlock tcb : asmTryCatchBlocks) {
                    super.visitTryCatchBlock(tcb.start, tcb.end, tcb.handler, tcb.type);
                }
                super.visitEnd();
            }

        };
    }
```



