# JavaAgent源码分析

* [Arthas](#arthas)
  * [入口方法](#入口方法)
  * [绑定端口，接收客户端连接](#绑定端口接收客户端连接)
* [Jvm-Sandbox](#jvm-sandbox)
  * [启动](#启动)
  * [启动HTTP服务器](#启动http服务器)
    * [Initmanager](#initmanager)
      * [加载模块](#加载模块)
    * [InitJettyContextHandler](#initjettycontexthandler)
      * [接收请求进行增强](#接收请求进行增强)
        * [Transformer](#transformer)
        * [EventEnhancer](#eventenhancer)
        * [EventWeaver](#eventweaver)
        * [ReWriteMethod](#rewritemethod)
* [Chaosblade-exec-Jvm](#chaosblade-exec-jvm)
  * [SandboxModule](#sandboxmodule)
    * [onLoad](#onload)
    * [onActive](#onactive)
    * [create](#create)


# Arthas

在线诊断利器

## 入口方法

com.taobao.arthas.agent.AgentBootstrap#premain

```java
public static void premain(String args, Instrumentation inst) {//在main方法运行前执行此方法
    main(args, inst); 
}
```

```java
public static void agentmain(String args, Instrumentation inst) {//控制类运行时的行为，利用了JVM的Attach机制实现
    main(args, inst);
}
```

com.taobao.arthas.agent.AgentBootstrap#main

```java
private static synchronized void main(final String args, final Instrumentation inst) {
    try {
        ps.println("Arthas server agent start...");
        // 传递的args参数分两个部分:agentJar路径和agentArgs, 分别是Agent的JAR包路径和期望传递到服务端的参数
        int index = args.indexOf(';');
        String agentJar = args.substring(0, index);
        final String agentArgs = args.substring(index, args.length());

        File agentJarFile = new File(agentJar);
        if (!agentJarFile.exists()) {
            ps.println("Agent jar file does not exist: " + agentJarFile);
            return;
        }

        File spyJarFile = new File(agentJarFile.getParentFile(), ARTHAS_SPY_JAR);
        if (!spyJarFile.exists()) {
            ps.println("Spy jar file does not exist: " + spyJarFile);
            return;
        }

			 //创建自定义的类加载器
        final ClassLoader agentLoader = getClassLoader(inst, spyJarFile, agentJarFile);
        initSpy(agentLoader);

        Thread bindingThread = new Thread() {
            @Override
            public void run() {
                try {
                    bind(inst, agentLoader, agentArgs);//绑定端口号，监听客户端请求
                } catch (Throwable throwable) {
                    throwable.printStackTrace(ps);
                }
            }
        };

        bindingThread.setName("arthas-binding-thread");
        bindingThread.start(); //启动线程
        bindingThread.join(); //阻塞，直到bindingThread线程结束
    } catch (Throwable t) {
        t.printStackTrace(ps);
        try {
            if (ps != System.err) {
                ps.close();
            }
        } catch (Throwable tt) {
            // ignore
        }
        throw new RuntimeException(t);
    }
}
```

## 绑定端口，接收客户端连接

com.taobao.arthas.agent.AgentBootstrap#bind

```java
private static void bind(Instrumentation inst, ClassLoader agentLoader, String args) throws Throwable {
  //自定义类加载器加载com.taobao.arthas.core.config.Configure
    Class<?> classOfConfigure = agentLoader.loadClass(ARTHAS_CONFIGURE);
  //根据参数通过反射创建Configure
    Object configure = classOfConfigure.getMethod(TO_CONFIGURE, String.class).invoke(null, args);
    int javaPid = (Integer) classOfConfigure.getMethod(GET_JAVA_PID).invoke(configure);
  //加载com.taobao.arthas.core.server.ArthasBootstrap
    Class<?> bootstrapClass = agentLoader.loadClass(ARTHAS_BOOTSTRAP);
  //创建ArthasBootstrap
    Object bootstrap = bootstrapClass.getMethod(GET_INSTANCE, int.class, Instrumentation.class).invoke(null, javaPid, inst);
    boolean isBind = (Boolean) bootstrapClass.getMethod(IS_BIND).invoke(bootstrap);
    if (!isBind) { //尚未绑定
        try {
            ps.println("Arthas start to bind...");
          //绑定端口
            bootstrapClass.getMethod(BIND, classOfConfigure).invoke(bootstrap, configure);
            ps.println("Arthas server bind success.");
            return;
        } catch (Exception e) {
            ps.println("Arthas server port binding failed! Please check $HOME/logs/arthas/arthas.log for more details.");
            throw e;
        }
    }
    ps.println("Arthas server already bind.");
}
```

com.taobao.arthas.core.server.ArthasBootstrap#bind

```java
public void bind(Configure configure) throws Throwable {

    long start = System.currentTimeMillis();

    if (!isBindRef.compareAndSet(false, true)) {
        throw new IllegalStateException("already bind");
    }

    try {
        ShellServerOptions options = new ShellServerOptions().setInstrumentation(instrumentation).setPid(pid);
        shellServer = new ShellServerImpl(options, this);
        //维护支持的所有命令
        BuiltinCommandPack builtinCommands = new BuiltinCommandPack();
        List<CommandResolver> resolvers = new ArrayList<CommandResolver>();
        resolvers.add(builtinCommands);
        // TODO: discover user provided command resolver
        shellServer.registerTermServer(new TelnetTermServer(
                configure.getIp(), configure.getTelnetPort(), options.getConnectionTimeout()));
        shellServer.registerTermServer(new HttpTermServer(
                configure.getIp(), configure.getHttpPort(), options.getConnectionTimeout()));

        for (CommandResolver resolver : resolvers) {
            shellServer.registerCommandResolver(resolver);
        }
        //启动HttpTermServer、TelnetTermServer
        shellServer.listen(new BindHandler(isBindRef));

        logger.info("as-server listening on network={};telnet={};http={};timeout={};", configure.getIp(),
                configure.getTelnetPort(), configure.getHttpPort(), options.getConnectionTimeout());
        // 异步回报启动次数
        UserStatUtil.arthasStart();

        logger.info("as-server started in {} ms", System.currentTimeMillis() - start );
    } catch (Throwable e) {
        logger.error(null, "Error during bind to port " + configure.getTelnetPort(), e);
        if (shellServer != null) {
            shellServer.close();
        }
        throw e;
    }
}
```

com.taobao.arthas.core.shell.impl.ShellServerImpl#listen

```java
public ShellServer listen(final Handler<Future<Void>> listenHandler) {
    final List<TermServer> toStart;
    synchronized (this) {
        if (!closed) {
            throw new IllegalStateException("Server listening");
        }
        toStart = termServers;
    }
    final AtomicInteger count = new AtomicInteger(toStart.size());
    if (count.get() == 0) {
        setClosed(false);
        listenHandler.handle(Future.<Void>succeededFuture());
        return this;
    }
    Handler<Future<TermServer>> handler = new TermServerListenHandler(this, listenHandler, toStart);
    for (TermServer termServer : toStart) {
        termServer.termHandler(new TermServerTermHandler(this));//处理连接事件
        termServer.listen(handler);//绑定端口
    }
    return this;
}
```

com.taobao.arthas.core.shell.term.impl.HttpTermServer#listen

```java
public TermServer listen(Handler<Future<TermServer>> listenHandler) {
    // TODO: charset and inputrc from options
    bootstrap = new NettyWebsocketTtyBootstrap().setHost(hostIp).setPort(port);
    try {
        bootstrap.start(new Consumer<TtyConnection>() {
            @Override
            public void accept(final TtyConnection conn) {
                termHandler.handle(new TermImpl(Helper.loadKeymap(), conn)); //处理连接
            }
        }).get(connectionTimeout, TimeUnit.MILLISECONDS);
        listenHandler.handle(Future.<TermServer>succeededFuture());
    } catch (Throwable t) {
        logger.error(null, "Error listening to port " + port, t);
        listenHandler.handle(Future.<TermServer>failedFuture(t));
    }
    return this;
}
```

com.taobao.arthas.core.shell.impl.ShellServerImpl#handleTerm

```java
public void handleTerm(Term term) {
    synchronized (this) {
        // That might happen with multiple ser
        if (closed) {
            term.close();
            return;
        }
    }

    ShellImpl session = createShell(term);
    session.setWelcome(welcomeMessage);
    session.closedFuture.setHandler(new SessionClosedHandler(this, session));
    session.init();
    sessions.put(session.id, session); // Put after init so the close handler on the connection is set
    session.readline(); // Now readline
}
```

com.taobao.arthas.core.shell.command.impl.AnnotatedCommandImpl#process

```java
private void process(CommandProcess process) {//命令都封装从AnnotatedCommandImpl
    AnnotatedCommand instance;
    try {
        instance = clazz.newInstance();
    } catch (Exception e) {
        process.end();
        return;
    }
    CLIConfigurator.inject(process.commandLine(), instance);
    instance.process(process); //执行
    UserStatUtil.arthasUsageSuccess(name(), process.args());
}
```

com.taobao.arthas.core.command.monitor200.EnhancerCommand#process

```java
public void process(final CommandProcess process) {
    process.interruptHandler(new CommandInterruptHandler(process));

    enhance(process); //开始增强
}
```

com.taobao.arthas.core.command.monitor200.EnhancerCommand#enhance

```java
protected void enhance(CommandProcess process) {//开始增强
    Session session = process.session();
    if (!session.tryLock()) {
        process.write("someone else is enhancing classes, pls. wait.\n");
        process.end();
        return;
    }
    int lock = session.getLock();
    try {
        Instrumentation inst = session.getInstrumentation();
      //获取AdviceListener，例如WatchAdviceListener
        AdviceListener listener = getAdviceListener(process);
        if (listener == null) {
            warn(process, "advice listener is null");
            return;
        }
        boolean skipJDKTrace = false;
        if(listener instanceof AbstractTraceAdviceListener) {
            skipJDKTrace = ((AbstractTraceAdviceListener) listener).getCommand().isSkipJDKTrace();
        }
				//根据classMatcher、methodMatcher过滤出需要增强的类，然后进行增强
        EnhancerAffect effect = Enhancer.enhance(inst, lock, listener instanceof InvokeTraceable,
                skipJDKTrace, getClassNameMatcher(), getMethodNameMatcher());

        if (effect.cCnt() == 0 || effect.mCnt() == 0) {
            // no class effected
            // might be method code too large
            process.write("No class or method is affected, try:\n"
                          + "1. sm CLASS_NAME METHOD_NAME to make sure the method you are tracing actually exists (it might be in your parent class).\n"
                          + "2. reset CLASS_NAME and try again, your method body might be too large.\n"
                          + "3. visit middleware-container/arthas/issues/278 for more detail\n");
            process.end();
            return;
        }

        // 这里做个补偿,如果在enhance期间,unLock被调用了,则补偿性放弃
        if (session.getLock() == lock) {
            // 注册通知监听器
            process.register(lock, listener);
            if (process.isForeground()) {
                process.echoTips(Constants.ABORT_MSG + "\n");
            }
        }

        process.write(effect + "\n");
    } catch (UnmodifiableClassException e) {
        logger.error(null, "error happens when enhancing class", e);
    } finally {
        if (session.getLock() == lock) {
            // enhance结束后解锁
            process.session().unLock();
        }
    }
}
```

com.taobao.arthas.core.advisor.Enhancer#enhance(java.lang.instrument.Instrumentation, int, boolean, boolean, com.taobao.arthas.core.util.matcher.Matcher, com.taobao.arthas.core.util.matcher.Matcher)

```java
public static synchronized EnhancerAffect enhance(
        final Instrumentation inst,
        final int adviceId,
        final boolean isTracing,
        final boolean skipJDKTrace,
        final Matcher classNameMatcher,
        final Matcher methodNameMatcher) throws UnmodifiableClassException {

    final EnhancerAffect affect = new EnhancerAffect();
		//从Instrumentation获取所有已经加载的类
    // 获取需要增强的类集合
    final Set<Class<?>> enhanceClassSet = GlobalOptions.isDisableSubClass//是否禁用子类
            ? SearchUtils.searchClass(inst, classNameMatcher)
            : SearchUtils.searchSubClass(inst, SearchUtils.searchClass(inst, classNameMatcher));

    // 过滤掉无法被增强的类
    filter(enhanceClassSet);

    // 构建增强器
    final Enhancer enhancer = new Enhancer(adviceId, isTracing, skipJDKTrace, enhanceClassSet, methodNameMatcher, affect);
    try {
        inst.addTransformer(enhancer, true); //此Enhancer用于重新增强

        // 批量增强
        if (GlobalOptions.isBatchReTransform) {
            final int size = enhanceClassSet.size();
            final Class<?>[] classArray = new Class<?>[size];
            arraycopy(enhanceClassSet.toArray(), 0, classArray, 0, size);
            if (classArray.length > 0) {
                inst.retransformClasses(classArray); //批量增强
                logger.info("Success to batch transform classes: " + Arrays.toString(classArray));
            }
        } else {
            // for each 增强
            for (Class<?> clazz : enhanceClassSet) {
                try {
                    inst.retransformClasses(clazz); //单个class增强
                    logger.info("Success to transform class: " + clazz);
                } catch (Throwable t) {
                    logger.warn("retransform {} failed.", clazz, t);
                    if (t instanceof UnmodifiableClassException) {
                        throw (UnmodifiableClassException) t;
                    } else if (t instanceof RuntimeException) {
                        throw (RuntimeException) t;
                    } else {
                        throw new RuntimeException(t);
                    }
                }
            }
        }
    } finally {
        inst.removeTransformer(enhancer); //移除enhancer
    }

    return affect;
}
```

重置已增强的类

com.taobao.arthas.core.advisor.Enhancer#reset

```java
public static synchronized EnhancerAffect reset(
        final Instrumentation inst,
        final Matcher classNameMatcher) throws UnmodifiableClassException {

    final EnhancerAffect affect = new EnhancerAffect();
    final Set<Class<?>> enhanceClassSet = new HashSet<Class<?>>();
		//过滤出已增强的类
    for (Class<?> classInCache : classBytesCache.keySet()) {
        if (classNameMatcher.matching(classInCache.getName())) {
            enhanceClassSet.add(classInCache);
        }
    }

    final ClassFileTransformer resetClassFileTransformer = new ClassFileTransformer() {
        @Override
        public byte[] transform(
                ClassLoader loader,
                String className,
                Class<?> classBeingRedefined,
                ProtectionDomain protectionDomain,
                byte[] classfileBuffer) throws IllegalClassFormatException {
            return null;//返回空
        }
    };

    try {
        enhance(inst, resetClassFileTransformer, enhanceClassSet);
        logger.info("Success to reset classes: " + enhanceClassSet);
    } finally {
        for (Class<?> resetClass : enhanceClassSet) {
            classBytesCache.remove(resetClass);
            affect.cCnt(1);
        }
    }

    return affect;
}
```

com.taobao.arthas.core.advisor.Enhancer#enhance(java.lang.instrument.Instrumentation, java.lang.instrument.ClassFileTransformer, java.util.Set<java.lang.Class<?>>)

```java
public static void enhance(Instrumentation inst, ClassFileTransformer transformer, Set<Class<?>> classes)
        throws UnmodifiableClassException {
    try {
        inst.addTransformer(transformer, true);//添加可以重新增强的Instrumentation
        int size = classes.size();
        Class<?>[] classArray = new Class<?>[size];
        arraycopy(classes.toArray(), 0, classArray, 0, size);
        if (classArray.length > 0) {
            inst.retransformClasses(classArray); //增强
        }
    } finally {
        inst.removeTransformer(transformer);//移除Instrumentation
    }
}
```

Enhancer

com.taobao.arthas.core.advisor.Enhancer#transform

```java
public byte[] transform(
        final ClassLoader inClassLoader,
        String className,
        Class<?> classBeingRedefined,
        ProtectionDomain protectionDomain,
        byte[] classfileBuffer) throws IllegalClassFormatException {


    // 这里要再次过滤一次，为啥？因为在transform的过程中，有可能还会再诞生新的类
    // 所以需要将之前需要转换的类集合传递下来，再次进行判断
    if (!matchingClasses.contains(classBeingRedefined)) {
        return null;
    }

    final ClassReader cr;

    // 首先先检查是否在缓存中存在Class字节码
    // 因为要支持多人协作,存在多人同时增强的情况
    final byte[] byteOfClassInCache = classBytesCache.get(classBeingRedefined);
    if (null != byteOfClassInCache) {
        cr = new ClassReader(byteOfClassInCache); //已经缓存的增强的字节码
    }

    // 如果没有命中缓存,则从原始字节码开始增强
    else {
        cr = new ClassReader(classfileBuffer);//原始字节码
    }

    // 字节码增强
    final ClassWriter cw = new ClassWriter(cr, COMPUTE_FRAMES | COMPUTE_MAXS) {


        /*
         * 注意，为了自动计算帧的大小，有时必须计算两个类共同的父类。
         * 缺省情况下，ClassWriter将会在getCommonSuperClass方法中计算这些，通过在加载这两个类进入虚拟机时，使用反射API来计算。
         * 但是，如果你将要生成的几个类相互之间引用，这将会带来问题，因为引用的类可能还不存在。
         * 在这种情况下，你可以重写getCommonSuperClass方法来解决这个问题。
         *
         * 通过重写 getCommonSuperClass() 方法，更正获取ClassLoader的方式，改成使用指定ClassLoader的方式进行。
         * 规避了原有代码采用Object.class.getClassLoader()的方式
         */
        @Override
        protected String getCommonSuperClass(String type1, String type2) {
            Class<?> c, d;
            final ClassLoader classLoader = inClassLoader;
            try {
                c = Class.forName(type1.replace('/', '.'), false, classLoader);
                d = Class.forName(type2.replace('/', '.'), false, classLoader);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
            if (c.isAssignableFrom(d)) {
                return type1;
            }
            if (d.isAssignableFrom(c)) {
                return type2;
            }
            if (c.isInterface() || d.isInterface()) {
                return "java/lang/Object";
            } else {
                do {
                    c = c.getSuperclass();
                } while (!c.isAssignableFrom(d));
                return c.getName().replace('.', '/');
            }
        }

    };

    try {

        // 生成增强字节码
        cr.accept(new AdviceWeaver(adviceId, isTracing, skipJDKTrace, cr.getClassName(), methodNameMatcher, affect, cw), EXPAND_FRAMES);
        final byte[] enhanceClassByteArray = cw.toByteArray(); //增强后的字节码

        // 生成成功,推入缓存
        classBytesCache.put(classBeingRedefined, enhanceClassByteArray);

        // dump the class
        dumpClassIfNecessary(className, enhanceClassByteArray, affect);

        // 成功计数
        affect.cCnt(1);

        // 排遣间谍
        try {
            spy(inClassLoader);
        } catch (Throwable t) {
            logger.warn("print spy failed. classname={};loader={};", className, inClassLoader, t);
            throw t;
        }

        return enhanceClassByteArray;
    } catch (Throwable t) {
        logger.warn("transform loader[{}]:class[{}] failed.", inClassLoader, className, t);
    }

    return null;
}
```



# Jvm-Sandbox

利用了JVM的Attach机制实现

## 启动

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

## 启动HTTP服务器

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

### Initmanager

com.alibaba.jvm.sandbox.core.server.jetty.JettyCoreServer#initManager

```java
private void initManager(final Instrumentation inst,
                         final CoreConfigure cfg) {

    logger.debug("{} was init", EventListenerHandlers.getSingleton());

    final ModuleLifeCycleEventBus moduleLifeCycleEventBus = new DefaultModuleLifeCycleEventBus();
    final LoadedClassDataSource classDataSource = new DefaultLoadedClassDataSource(inst);
    final ClassLoader sandboxClassLoader = getClass().getClassLoader();

    // 初始化模块资源管理器
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

#### 加载模块

com.alibaba.jvm.sandbox.core.manager.impl.DefaultCoreModuleManager#DefaultCoreModuleManager

```java
public DefaultCoreModuleManager(final Instrumentation inst,
                                final LoadedClassDataSource classDataSource,
                                final CoreConfigure cfg,
                                final ClassLoader sandboxClassLoader,
                                final ModuleLifeCycleEventBus moduleLifeCycleEventBus,
                                final ProviderManager providerManager) {
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



### InitJettyContextHandler

com.alibaba.jvm.sandbox.core.server.jetty.JettyCoreServer#initJettyContextHandler

```java
private void initJettyContextHandler() {
    final ServletContextHandler context = new ServletContextHandler(SESSIONS);

    // websocket-servlet
    context.addServlet(new ServletHolder(new WebSocketAcceptorServlet(coreModuleManager, moduleResourceManager)), "/module/websocket/*");

    // module-http-servlet
    context.addServlet(new ServletHolder(new ModuleHttpServlet(coreModuleManager, moduleResourceManager)), "/module/http/*"); //处理客户端请求

    context.setContextPath("/sandbox");
    context.setClassLoader(getClass().getClassLoader());
    httpServer.setHandler(context);
}
```

#### 接收请求进行增强

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

JVM-SANDBOX自带的用例

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

对方法增强的入口方法

public int watch(final Filter filter,  final EventListener listener,  final Progress progress, final Event.Type... eventType) 

```java
public int watch(final Filter filter, //帅选出需要增强的方法
                 final EventListener listener, //如何增强
                 final Progress progress, //增强的进度
                 final Event.Type... eventType) { //类型（ before、after、throws）
    final int watchId = watchIdSequencer.next();

    // 给对应的模块追加ClassFileTransformer
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

    // 查找需要渲染的类集合
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

增强

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

##### Transformer

com.alibaba.jvm.sandbox.core.manager.impl.SandboxClassFileTransformer#transform

```java
public byte[] transform(final ClassLoader loader,
                        final String javaClassName,
                        final Class<?> classBeingRedefined,
                        final ProtectionDomain protectionDomain,
                        final byte[] srcByteCodeArray) throws IllegalClassFormatException {

    final String uniqueCodePrefix = computeUniqueCodePrefix(loader, javaClassName);

    final Enhancer enhancer = new EventEnhancer(
            listenerId,
            filter,
            uniqueCodePrefix,
            affectMethodUniqueSet,
            isEnableUnsafe,
            eventTypeArray
    );
    try {
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

##### EventEnhancer

获取增强后的字节码

com.alibaba.jvm.sandbox.core.enhance.EventEnhancer#toByteCodeArray

```java
public byte[] toByteCodeArray(final ClassLoader targetClassLoader,
                              final byte[] byteCodeArray) {

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
    if (reWriteMark.get()) {
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

##### EventWeaver

方法事件编织者

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

                // 这里修改为
                push(targetClassLoaderObjectID);

//                if (this.isStaticMethod()) {
//
////                    // fast enhance
////                    if (GlobalOptions.isEnableFastEnhance) {
////                        visitLdcInsn(Type.getType(String.format("L%s;", internalClassName)));
////                        visitMethodInsn(INVOKEVIRTUAL, "java/lang/Class", "getClassLoader", "()Ljava/lang/ClassLoader;", false);
////                    }
//
//                    // normal enhance
////                    else {
//
//                    // 这里不得不用性能极差的Class.forName()来完成类的获取,因为有可能当前这个静态方法在执行的时候
//                    // 当前类并没有完成实例化,会引起JVM对class文件的合法性校验失败
//                    // 未来我可能会在这一块考虑性能优化,但对于当前而言,功能远远重要于性能,也就不打算折腾这么复杂了
//                    visitLdcInsn(targetJavaClassName);
//                    invokeStatic(ASM_TYPE_CLASS, ASM_METHOD_Class$forName);
//                    invokeVirtual(ASM_TYPE_CLASS, ASM_METHOD_Class$getClassLoader);
////                    }
//
//                } else {
//                    loadThis();
//                    invokeVirtual(ASM_TYPE_OBJECT, ASM_METHOD_Object$getClass);
//                    invokeVirtual(ASM_TYPE_CLASS, ASM_METHOD_Class$getClassLoader);
//                }

            }

            @Override
            protected void onMethodEnter() {

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
            protected void onMethodExit(final int opcode) {
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
                codeLockForTracing.lock(new CodeLock.Block() {
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

##### ReWriteMethod

方法重写

# Chaosblade-exec-Jvm

https://chaosblade-io.gitbook.io/chaosblade-help-zh-cn/

混沌工程，借助Jvm-Sandbox实现注入故障

## SandboxModule

启动之初，加载所有的模块时调用onLoad方法，激活模块时调用onActive方法

### onLoad

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#onLoad

```java
public void onLoad() throws Throwable {
    LOGGER.info("load chaosblade module");
    ManagerFactory.getListenerManager().setPluginLifecycleListener(this);
    dispatchService.load();
    ManagerFactory.load();
}
```

com.alibaba.chaosblade.exec.service.handler.DefaultDispatchService#load

```java
public void load() {
    registerHandler(new CreateHandler());
    registerHandler(new DestroyHandler());
    registerHandler(new StatusHandler());
}
```

### onActive

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#onActive

```java
public void onActive() throws Throwable {
    LOGGER.info("active chaosblade module");
    loadPlugins();  //加载插件
}
```

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#loadPlugins

```java
private void loadPlugins() throws Exception {
    List<Plugin> plugins = PluginLoader.load(Plugin.class, PluginJarUtil.getPluginFiles(getClass()));
    for (Plugin plugin : plugins) {
        try {
            PluginBean pluginBean = new PluginBean(plugin);
            final ModelSpec modelSpec = pluginBean.getModelSpec();
            // register model
            ManagerFactory.getModelSpecManager().registerModelSpec(modelSpec);
            add(pluginBean);
        } catch (Throwable e) {
            LOGGER.warn("Load " + plugin.getClass().getName() + " occurs exception", e);
        }
    }
}
```

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#add

```java
public void add(PluginBean plugin) {
    PointCut pointCut = plugin.getPointCut();
    if (pointCut == null) {
        return;
    }
    //增强类的名称，例如ServletEnhancer
    String enhancerName = plugin.getEnhancer().getClass().getSimpleName();
    //筛选出增强的方法
    Filter filter = SandboxEnhancerFactory.createFilter(enhancerName, pointCut);
    //增强
    if (plugin.isAfterEvent()) { //后置拦截
        int watcherId = moduleEventWatcher.watch(filter, SandboxEnhancerFactory.createAfterEventListener(plugin),
            Type.BEFORE, Type.RETURN);
        watchIds.put(PluginUtil.getIdentifierForAfterEvent(plugin), watcherId);
    } else { //前置拦截
        int watcherId = moduleEventWatcher.watch(
            filter, SandboxEnhancerFactory.createBeforeEventListener(plugin), Event.Type.BEFORE);
        watchIds.put(PluginUtil.getIdentifier(plugin), watcherId);
    }
}
```

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.BeforeEventListener#onEvent

```java
public void onEvent(Event event) throws Throwable {
    if (event instanceof BeforeEvent) {
        handleEvent((BeforeEvent)event);
    }
}
```

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.BeforeEventListener#handleEvent

```java
private void handleEvent(BeforeEvent event) throws Throwable {
    Class<?> clazz;
    // get the method class
    if (event.target == null) {
        clazz = event.javaClassLoader.loadClass(event.javaClassName);
    } else {
        clazz = event.target.getClass();
    }
    Method method;
    try {
        method = ReflectUtil.getMethod(clazz, event.javaMethodDesc, event.javaMethodName);
    } catch (NoSuchMethodException e) {
        LOGGER.warn("get method by reflection exception. class: {}, method: {}, arguments: {}, desc: {}", event
            .javaClassName, event.javaMethodName, Arrays.toString(event.argumentArray), event.javaMethodDesc, e);
        return;
    }
    try {
        // do enhancer
        plugin.getEnhancer().beforeAdvice(plugin.getModelSpec().getTarget(), event.javaClassLoader,
            event.javaClassName, event.target, method, event.argumentArray);
    } catch (Exception e) {
        // handle return or throw exception
        if (e instanceof InterruptProcessException) {
            InterruptProcessException exception = (InterruptProcessException)e;
            if (exception.getState() == State.RETURN_IMMEDIATELY) {
                ProcessControlException.throwReturnImmediately(exception.getResponse());
            } else if (exception.getState() == State.THROWS_IMMEDIATELY) {
                ProcessControlException.throwThrowsImmediately((Throwable)exception.getResponse());
            }
        } else {
            throw e;
        }
    }
}
```

com.alibaba.chaosblade.exec.common.aop.BeforeEnhancer#beforeAdvice

```java
public void beforeAdvice(String targetName, ClassLoader classLoader, String className, Object object,
                         Method method, Object[] methodArguments) throws Exception {
    if (!ManagerFactory.getStatusManager().expExists(targetName)) { //判断客户端是否已经激活，否则直接返回
        return;
    }
  //获取xxxEnhancer,例如ServletEnhancer
    EnhancerModel model = doBeforeAdvice(classLoader, className, object, method, methodArguments);
    if (model == null) {
        return;
    }   	  model.setTarget(targetName).setMethod(method).setObject(object).setMethodArguments(methodArguments);
    Injector.inject(model);
}
```

com.alibaba.chaosblade.exec.common.injection.Injector#inject

```java
public static void inject(EnhancerModel enhancerModel) throws InterruptProcessException {
    String target = enhancerModel.getTarget();
    List<StatusMetric> statusMetrics = ManagerFactory.getStatusManager().getExpByTarget(
        target);
    for (StatusMetric statusMetric : statusMetrics) {
        Model model = statusMetric.getModel();
        if (!compare(model, enhancerModel)) {
            continue;
        }
        try {
            boolean pass = limitAndIncrease(statusMetric);
            if (!pass) {
                LOGGER.info("Limited by: {}", JSON.toJSONString(model));
                break;
            }
            LOGGER.info("Match rule: {}", JSON.toJSONString(model));
            //开始执行增强方法，比如注入延迟故障，DelayActionSpec
            enhancerModel.merge(model);
            ModelSpec modelSpec = ManagerFactory.getModelSpecManager().getModelSpec(target);
            ActionSpec actionSpec = modelSpec.getActionSpec(model.getActionName());
            actionSpec.getActionExecutor().run(enhancerModel);
        } catch (InterruptProcessException e) {
            throw e;
        } catch (UnsupportedReturnTypeException e) {
            LOGGER.warn("unsupported return type for return experiment", e);
            // decrease the count if throw unexpected exception
            statusMetric.decrease();
        } catch (Throwable e) {
            LOGGER.warn("inject exception", e);
            // decrease the count if throw unexpected exception
            statusMetric.decrease();
        }
        // break it if compared success
        break;
    }
}
```

### create

注入延迟故障案例

访问 http://localhost:8080/dubbodemo/servlet/path?name=bob 请求延迟 3 秒，影响 2 条请求

blade c servlet delay --time 3000 --requestpath /servlet/path --effect-count 2

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#create

```java
@Http("/create")
public void create(HttpServletRequest request, HttpServletResponse response) {
    service("create", request, response);
}
```

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#service

```java
private void service(String command, HttpServletRequest httpServletRequest,
                     HttpServletResponse httpServletResponse) {
    Request request;
    if ("POST".equalsIgnoreCase(httpServletRequest.getMethod())) {
        try {
            request = getRequestFromBody(httpServletRequest); //获取参数
        } catch (IOException e) {
            Response response = Response.ofFailure(Code.ILLEGAL_PARAMETER, e.getMessage());
            output(httpServletResponse, response);
            return;
        }
    } else {
        request = getRequestFromParams(httpServletRequest);//获取参数
    }
    Response response = dispatchService.dispatch(command, request);
    output(httpServletResponse, response);
}
```

com.alibaba.chaosblade.exec.service.handler.DefaultDispatchService#dispatch

```java
public Response dispatch(String command, Request request) {
    LOGGER.info("command: {}, request: {}", command, JSON.toJSONString(request));
    if (StringUtil.isBlank(command)) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, "less request command");
    }
    RequestHandler handler = this.handles.get(command);
    if (handler == null) {
        return Response.ofFailure(Response.Code.NOT_FOUND, command + " command not found");
    }
    try {
        return handler.handle(request);
    } catch (Throwable e) {
        LOGGER.warn("handle {} request exception, params: {}",
            new Object[] {command, JSON.toJSONString(request.getParams()), e});
        return Response.ofFailure(Response.Code.SERVER_ERROR, e.getMessage());
    }
}
```

com.alibaba.chaosblade.exec.service.handler.CreateHandler#handle

```java
public Response handle(Request request) {
    if (unloaded) {
        return Response.ofFailure(Code.ILLEGAL_STATE, "the agent is uninstalling");
    }
    // check necessary arguments
    String suid = request.getParam("suid");
    if (StringUtil.isBlank(suid)) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, "less experiment argument");
    }
    String target = request.getParam("target");
    if (StringUtil.isBlank(target)) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, "less target argument");
    }
    String actionArg = request.getParam("action");
    if (StringUtil.isBlank(actionArg)) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, "less action argument");
    }
    // change level from info to debug if open debug mode
    checkAndSetLogLevel(request);
    // ServletModelSpec、JedisModelSpec
    ModelSpec modelSpec = this.modelSpecManager.getModelSpec(target);
    if (modelSpec == null) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, "the target not supported");
    }
    //DelayActionSpec，注入延迟故障
    ActionSpec actionSpec = modelSpec.getActionSpec(actionArg);
    if (actionSpec == null) {
        return Response.ofFailure(Code.NOT_FOUND, "the action not supported");
    }
    // parse request to model
    Model model = ModelParser.parseRequest(target, request, actionSpec);
    // check command arguments
    PredicateResult predicate = modelSpec.predicate(model);
    if (!predicate.isSuccess()) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, predicate.getErr());
    }

    return handleInjection(suid, model, modelSpec);
}
```

com.alibaba.chaosblade.exec.service.handler.CreateHandler#handleInjection

```java
private Response handleInjection(String suid, Model model, ModelSpec modelSpec) {
   //statusManager维护注册的model
    RegisterResult result = this.statusManager.registerExp(suid, model);
    if (result.isSuccess()) {
        // handle injection
        try {
            applyPreInjectionModelHandler(suid, modelSpec, model);
        } catch (ExperimentException ex) {
            this.statusManager.removeExp(suid);
            return Response.ofFailure(Response.Code.SERVER_ERROR, ex.getMessage());
        }

        return Response.ofSuccess(model.toString());
    }
    return Response.ofFailure(Response.Code.DUPLICATE_INJECTION, "the experiment exists");
}
```
