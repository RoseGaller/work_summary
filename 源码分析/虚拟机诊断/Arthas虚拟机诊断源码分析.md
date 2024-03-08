# Arthas

在线诊断利器

# 入口方法

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

# 启动服务

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

# 监听端口

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

# 处理连接

com.taobao.arthas.core.shell.handlers.server.TermServerTermHandler#handle

```java
public void handle(Term term) {
    shellServer.handleTerm(term);
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

# 设置读处理器

com.taobao.arthas.core.shell.impl.ShellImpl#readline

```java
public void readline() {
    term.readline(Constants.DEFAULT_PROMPT, new ShellLineHandler(this),
            new CommandManagerCompletionHandler(commandManager));
}
```

```java
public void readline(String prompt, Handler<String> lineHandler, Handler<Completion> completionHandler) {
    if (conn.getStdinHandler() != echoHandler) {
        throw new IllegalStateException();
    }
    if (inReadline) {
        throw new IllegalStateException();
    }
    inReadline = true;
    readline.readline(conn, prompt, new RequestHandler(this, lineHandler), new CompletionHandler(completionHandler, session));
}
```

# 处理请求

com.taobao.arthas.core.shell.handlers.shell.ShellLineHandler#handle

```java
public void handle(String line) {
    if (line == null) {
        // EOF
        handleExit();
        return;
    }

    List<CliToken> tokens = CliTokens.tokenize(line);
    CliToken first = TokenUtils.findFirstTextToken(tokens);
    if (first == null) {
        // For now do like this
        shell.readline();
        return;
    }

    String name = first.value();
    if (name.equals("exit") || name.equals("logout") || name.equals("quit")) {
        handleExit();
        return;
    } else if (name.equals("jobs")) {
        handleJobs();
        return;
    } else if (name.equals("fg")) {
        handleForeground(tokens);
        return;
    } else if (name.equals("bg")) {
        handleBackground(tokens);
        return;
    } else if (name.equals("kill")) {
        handleKill(tokens);
        return;
    }
		//创建job
    Job job = createJob(tokens);
    if (job != null) {
      	//执行job
        job.run();
    }
}
```

```java
private Job createJob(List<CliToken> tokens) {
    Job job;
    try {
        job = shell.createJob(tokens);
    } catch (Exception e) {
        term.echo(e.getMessage() + "\n");
        shell.readline();
        return null;
    }
    return job;
}
```

com.taobao.arthas.core.shell.impl.ShellImpl#createJob(java.util.List<com.taobao.arthas.core.shell.cli.CliToken>)

```java
public synchronized Job createJob(List<CliToken> args) {
    Job job = jobController.createJob(commandManager, args, this);
    return job;
}
```

com.taobao.arthas.core.shell.system.impl.GlobalJobControllerImpl#createJob

```java
public Job createJob(InternalCommandManager commandManager, List<CliToken> tokens, ShellImpl shell) {
  	//创建job
    final Job job = super.createJob(commandManager, tokens, shell);

    /*
     * 达到超时时间将会停止job
     */
    TimerTask jobTimeoutTask = new TimerTask() {
        @Override
        public void run() {
            job.terminate();
        }
    };
    Date timeoutDate = new Date(System.currentTimeMillis() + (getJobTimeoutInSecond() * 1000));
  	//调度超时任务
    timer.schedule(jobTimeoutTask, timeoutDate);
    jobTimeoutTaskMap.put(job.id(), jobTimeoutTask);
    job.setTimeoutDate(timeoutDate);

    return job;
}
```

com.taobao.arthas.core.shell.system.impl.JobControllerImpl#createJob

```java
public Job createJob(InternalCommandManager commandManager, List<CliToken> tokens, ShellImpl shell) {
    int jobId = idGenerator.incrementAndGet();
    StringBuilder line = new StringBuilder();
    for (CliToken arg : tokens) {
        line.append(arg.raw());
    }
    boolean runInBackground = runInBackground(tokens);
    Process process = createProcess(tokens, commandManager, jobId, shell.term());
    process.setJobId(jobId);
    JobImpl job = new JobImpl(jobId, this, process, line.toString(), runInBackground, shell);
    jobs.put(jobId, job);
    return job;
}
```

com.taobao.arthas.core.shell.system.impl.JobImpl#run()

```java
public Job run() {
    return run(!runInBackground.get());
}
```

```java
public Job run(boolean foreground) {
    if (foreground && foregroundUpdatedHandler != null) {
        foregroundUpdatedHandler.handle(this);
    }

    actualStatus = ExecStatus.RUNNING;
    if (statusUpdateHandler != null) {
        statusUpdateHandler.handle(ExecStatus.RUNNING);
    }
    process.setTty(shell.term());
    process.setSession(shell.session());
    process.run(foreground);//

    if (!foreground && foregroundUpdatedHandler != null) {
        foregroundUpdatedHandler.handle(null);
    }

    if (foreground) {
        shell.setForegroundJob(this);
    } else {
        shell.setForegroundJob(null);
    }
    return this;
}
```

com.taobao.arthas.core.shell.system.impl.ProcessImpl.CommandProcessTask#run

```java
public void run() {
    try {
        handler.handle(process);
    } catch (Throwable t) {
        logger.error(null, "Error during processing the command:", t);
        process.write("Error during processing the command: " + t.getMessage() + "\n");
        terminate(1, null);
    }
}
```

com.taobao.arthas.core.shell.system.impl.ProcessImpl#run(boolean)

```java
public synchronized void run(boolean fg) {
    if (processStatus != ExecStatus.READY) {
        throw new IllegalStateException("Cannot run proces in " + processStatus + " state");
    }

    processStatus = ExecStatus.RUNNING;
    processForeground = fg;
    foreground = fg;
    startTime = new Date();

    // Make a local copy
    final Tty tty = this.tty;
    if (tty == null) {
        throw new IllegalStateException("Cannot execute process without a TTY set");
    }

    final List<String> args2 = new LinkedList<String>();
    for (CliToken arg : args) {
        if (arg.isText()) {
            args2.add(arg.value());
        }
    }

    CommandLine cl = null;
    try {
        if (commandContext.cli() != null) {
            if (commandContext.cli().parse(args2, false).isAskingForHelp()) {
                UsageMessageFormatter formatter = new StyledUsageFormatter(Color.green);
                formatter.setWidth(tty.width());
                StringBuilder usage = new StringBuilder();
                commandContext.cli().usage(usage, formatter);
                usage.append('\n');
                tty.write(usage.toString());
                terminate();
                return;
            }

            cl = commandContext.cli().parse(args2);
        }
    } catch (CLIException e) {
        tty.write(e.getMessage() + "\n");
        terminate();
        return;
    }

    process = new CommandProcessImpl(args2, tty, cl);
    if (cacheLocation() != null) {
        process.echoTips("job id  : " + this.jobId + "\n");
        process.echoTips("cache location  : " + cacheLocation() + "\n");
    }
    Runnable task = new CommandProcessTask(process);
    ArthasBootstrap.getInstance().execute(task);
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

# 增强

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

com.taobao.arthas.core.advisor.Enhancer#enhance

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

com.taobao.arthas.core.advisor.Enhancer#reset

```java
public static synchronized EnhancerAffect reset(
        final Instrumentation inst,
        final Matcher classNameMatcher) throws UnmodifiableClassException {
    //重置已增强的类
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
        inst.removeTransformer(transformer);//移除transformer
    }
}
```

# 增强器

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

# AdviceWeaver

通过ASM对方法进行增强

com.taobao.arthas.core.advisor.AdviceWeaver#visitMethod

```java
public MethodVisitor visitMethod(
            final int access,
            final String name,
            final String desc,
            final String signature,
            final String[] exceptions) {

        final MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);

        if (isIgnore(mv, access, name)) {
            return mv;
        }

        // 编织方法计数
        affect.mCnt(1);

        return new AdviceAdapter(ASM5, new JSRInlinerAdapter(mv, access, name, desc, signature, exceptions), access, name, desc) {

            // -- Label for try...catch block
            private final Label beginLabel = new Label();
            private final Label endLabel = new Label();

            // -- KEY of advice --
            private final int KEY_ARTHAS_ADVICE_BEFORE_METHOD = 0;
            private final int KEY_ARTHAS_ADVICE_RETURN_METHOD = 1;
            private final int KEY_ARTHAS_ADVICE_THROWS_METHOD = 2;
            private final int KEY_ARTHAS_ADVICE_BEFORE_INVOKING_METHOD = 3;
            private final int KEY_ARTHAS_ADVICE_AFTER_INVOKING_METHOD = 4;
            private final int KEY_ARTHAS_ADVICE_THROW_INVOKING_METHOD = 5;


            // -- KEY of ASM_TYPE or ASM_METHOD --
            private final Type ASM_TYPE_SPY = Type.getType("Ljava/arthas/Spy;");
            private final Type ASM_TYPE_OBJECT = Type.getType(Object.class);
            private final Type ASM_TYPE_OBJECT_ARRAY = Type.getType(Object[].class);
            private final Type ASM_TYPE_CLASS = Type.getType(Class.class);
            private final Type ASM_TYPE_INTEGER = Type.getType(Integer.class);
            private final Type ASM_TYPE_CLASS_LOADER = Type.getType(ClassLoader.class);
            private final Type ASM_TYPE_STRING = Type.getType(String.class);
            private final Type ASM_TYPE_THROWABLE = Type.getType(Throwable.class);
            private final Type ASM_TYPE_INT = Type.getType(int.class);
            private final Type ASM_TYPE_METHOD = Type.getType(java.lang.reflect.Method.class);
            private final Method ASM_METHOD_METHOD_INVOKE = Method.getMethod("Object invoke(Object,Object[])");

            // 代码锁
            private final CodeLock codeLockForTracing = new TracingAsmCodeLock(this);


            private void _debug(final StringBuilder append, final String msg) {

                if (!GlobalOptions.isDebugForAsm) {
                    return;
                }

                // println msg
                visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                if (StringUtils.isBlank(append.toString())) {
                    visitLdcInsn(append.append(msg).toString());
                } else {
                    visitLdcInsn(append.append(" >> ").append(msg).toString());
                }

                visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            }

            /**
             * 加载通知方法
             * @param keyOfMethod 通知方法KEY
             */
            private void loadAdviceMethod(int keyOfMethod) {

                switch (keyOfMethod) {

                    case KEY_ARTHAS_ADVICE_BEFORE_METHOD: {
                        getStatic(ASM_TYPE_SPY, "ON_BEFORE_METHOD", ASM_TYPE_METHOD);
                        break;
                    }

                    case KEY_ARTHAS_ADVICE_RETURN_METHOD: {
                        getStatic(ASM_TYPE_SPY, "ON_RETURN_METHOD", ASM_TYPE_METHOD);
                        break;
                    }

                    case KEY_ARTHAS_ADVICE_THROWS_METHOD: {
                        getStatic(ASM_TYPE_SPY, "ON_THROWS_METHOD", ASM_TYPE_METHOD);
                        break;
                    }

                    case KEY_ARTHAS_ADVICE_BEFORE_INVOKING_METHOD: {
                        getStatic(ASM_TYPE_SPY, "BEFORE_INVOKING_METHOD", ASM_TYPE_METHOD);
                        break;
                    }

                    case KEY_ARTHAS_ADVICE_AFTER_INVOKING_METHOD: {
                        getStatic(ASM_TYPE_SPY, "AFTER_INVOKING_METHOD", ASM_TYPE_METHOD);
                        break;
                    }

                    case KEY_ARTHAS_ADVICE_THROW_INVOKING_METHOD: {
                        getStatic(ASM_TYPE_SPY, "THROW_INVOKING_METHOD", ASM_TYPE_METHOD);
                        break;
                    }

                    default: {
                        throw new IllegalArgumentException("illegal keyOfMethod=" + keyOfMethod);
                    }

                }

            }

            /**
             * 加载ClassLoader<br/>
             * 这里分开静态方法中ClassLoader的获取以及普通方法中ClassLoader的获取
             * 主要是性能上的考虑
             */
            private void loadClassLoader() {

                if (this.isStaticMethod()) {
                    visitLdcInsn(StringUtils.normalizeClassName(className));
                    invokeStatic(ASM_TYPE_CLASS, Method.getMethod("Class forName(String)"));
                    invokeVirtual(ASM_TYPE_CLASS, Method.getMethod("ClassLoader getClassLoader()"));

                } else {
                    loadThis();
                    invokeVirtual(ASM_TYPE_OBJECT, Method.getMethod("Class getClass()"));
                    invokeVirtual(ASM_TYPE_CLASS, Method.getMethod("ClassLoader getClassLoader()"));
                }

            }

            /**
             * 加载before通知参数数组
             */
            private void loadArrayForBefore() {
                push(7);
                newArray(ASM_TYPE_OBJECT);

                dup();
                push(0);
                push(adviceId);
                box(ASM_TYPE_INT);
                arrayStore(ASM_TYPE_INTEGER);

                dup();
                push(1);
                loadClassLoader();
                arrayStore(ASM_TYPE_CLASS_LOADER);

                dup();
                push(2);
                push(className);
                arrayStore(ASM_TYPE_STRING);

                dup();
                push(3);
                push(name);
                arrayStore(ASM_TYPE_STRING);

                dup();
                push(4);
                push(desc);
                arrayStore(ASM_TYPE_STRING);

                dup();
                push(5);
                loadThisOrPushNullIfIsStatic();
                arrayStore(ASM_TYPE_OBJECT);

                dup();
                push(6);
                loadArgArray();
                arrayStore(ASM_TYPE_OBJECT_ARRAY);
            }


            @Override
            protected void onMethodEnter() {

                codeLockForTracing.lock(new CodeLock.Block() {
                    @Override
                    public void code() {

                        final StringBuilder append = new StringBuilder();
                        _debug(append, "debug:onMethodEnter()");

                        // 加载before方法
                        loadAdviceMethod(KEY_ARTHAS_ADVICE_BEFORE_METHOD);

                        _debug(append, "debug:onMethodEnter() > loadAdviceMethod()");

                        // 推入Method.invoke()的第一个参数
                        pushNull();

                        // 方法参数
                        loadArrayForBefore();

                        _debug(append, "debug:onMethodEnter() > loadAdviceMethod() > loadArrayForBefore()");

                        // 调用方法
                        invokeVirtual(ASM_TYPE_METHOD, ASM_METHOD_METHOD_INVOKE);
                        pop();

                        _debug(append, "debug:onMethodEnter() > loadAdviceMethod() > loadArrayForBefore() > invokeVirtual()");
                    }
                });

                mark(beginLabel);

            }


            /*
             * 加载return通知参数数组
             */
            private void loadReturnArgs() {
                dup2X1();
                pop2();
                push(1);
                newArray(ASM_TYPE_OBJECT);
                dup();
                dup2X1();
                pop2();
                push(0);
                swap();
                arrayStore(ASM_TYPE_OBJECT);
            }

            @Override
            protected void onMethodExit(final int opcode) {

                if (!isThrow(opcode)) {
                    codeLockForTracing.lock(new CodeLock.Block() {
                        @Override
                        public void code() {

                            final StringBuilder append = new StringBuilder();
                            _debug(append, "debug:onMethodExit()");

                            // 加载返回对象
                            loadReturn(opcode);
                            _debug(append, "debug:onMethodExit() > loadReturn()");


                            // 加载returning方法
                            loadAdviceMethod(KEY_ARTHAS_ADVICE_RETURN_METHOD);
                            _debug(append, "debug:onMethodExit() > loadReturn() > loadAdviceMethod()");

                            // 推入Method.invoke()的第一个参数
                            pushNull();

                            // 加载return通知参数数组
                            loadReturnArgs();
                            _debug(append, "debug:onMethodExit() > loadReturn() > loadAdviceMethod() > loadReturnArgs()");

                            invokeVirtual(ASM_TYPE_METHOD, ASM_METHOD_METHOD_INVOKE);
                            pop();

                            _debug(append, "debug:onMethodExit() > loadReturn() > loadAdviceMethod() > loadReturnArgs() > invokeVirtual()");
                        }
                    });
                }

            }


            /*
             * 创建throwing通知参数本地变量
             */
            private void loadThrowArgs() {
                dup2X1();
                pop2();
                push(1);
                newArray(ASM_TYPE_OBJECT);
                dup();
                dup2X1();
                pop2();
                push(0);
                swap();
                arrayStore(ASM_TYPE_THROWABLE);
            }

            @Override
            public void visitMaxs(int maxStack, int maxLocals) {

                mark(endLabel);
//                catchException(beginLabel, endLabel, ASM_TYPE_THROWABLE);
                visitTryCatchBlock(beginLabel, endLabel, mark(),
                        ASM_TYPE_THROWABLE.getInternalName());

                codeLockForTracing.lock(new CodeLock.Block() {
                    @Override
                    public void code() {

                        final StringBuilder append = new StringBuilder();
                        _debug(append, "debug:catchException()");

                        // 加载异常
                        loadThrow();
                        _debug(append, "debug:catchException() > loadThrow() > loadAdviceMethod()");

                        // 加载throwing方法
                        loadAdviceMethod(KEY_ARTHAS_ADVICE_THROWS_METHOD);
                        _debug(append, "debug:catchException() > loadThrow() > loadAdviceMethod()");


                        // 推入Method.invoke()的第一个参数
                        pushNull();

                        // 加载throw通知参数数组
                        loadThrowArgs();
                        _debug(append, "debug:catchException() > loadThrow() > loadAdviceMethod() > loadThrowArgs()");

                        // 调用方法
                        invokeVirtual(ASM_TYPE_METHOD, ASM_METHOD_METHOD_INVOKE);
                        pop();
                        _debug(append, "debug:catchException() > loadThrow() > loadAdviceMethod() > loadThrowArgs() > invokeVirtual()");

                    }
                });

                throwException();

                super.visitMaxs(maxStack, maxLocals);
            }

            /**
             * 是否静态方法
             * @return true:静态方法 / false:非静态方法
             */
            private boolean isStaticMethod() {
                return (methodAccess & ACC_STATIC) != 0;
            }

            /**
             * 是否抛出异常返回(通过字节码判断)
             * @param opcode 操作码
             * @return true:以抛异常形式返回 / false:非抛异常形式返回(return)
             */
            private boolean isThrow(int opcode) {
                return opcode == ATHROW;
            }

            /**
             * 将NULL推入堆栈
             */
            private void pushNull() {
                push((Type) null);
            }

            /**
             * 加载this/null
             */
            private void loadThisOrPushNullIfIsStatic() {
                if (isStaticMethod()) {
                    pushNull();
                } else {
                    loadThis();
                }
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

            /**
             * 加载异常
             */
            private void loadThrow() {
                dup();
            }


            /**
             * 加载方法调用跟踪通知所需参数数组
             */
            private void loadArrayForInvokeTracing(String owner, String name, String desc) {
                push(4);
                newArray(ASM_TYPE_OBJECT);

                dup();
                push(0);
                push(adviceId);
                box(ASM_TYPE_INT);
                arrayStore(ASM_TYPE_INTEGER);

                dup();
                push(1);
                push(owner);
                arrayStore(ASM_TYPE_STRING);

                dup();
                push(2);
                push(name);
                arrayStore(ASM_TYPE_STRING);

                dup();
                push(3);
                push(desc);
                arrayStore(ASM_TYPE_STRING);
            }


            @Override
            public void visitInsn(int opcode) {
                super.visitInsn(opcode);
                codeLockForTracing.code(opcode);
            }

            @Override
            public void visitTryCatchBlock(Label start, Label end, Label handler, String type) {
                tcbs.add(new AsmTryCatchBlock(start, end, handler, type));
            }

            List<AsmTryCatchBlock> tcbs = new ArrayList<AsmTryCatchBlock>();

            @Override
            public void visitEnd() {
                for (AsmTryCatchBlock tcb : tcbs) {
                    super.visitTryCatchBlock(tcb.start, tcb.end, tcb.handler, tcb.type);
                }

                super.visitEnd();
            }

            /*
             * 跟踪代码
             */
            private void tracing(final int tracingType, final String owner, final String name, final String desc) {

                final String label;
                switch (tracingType) {
                    case KEY_ARTHAS_ADVICE_BEFORE_INVOKING_METHOD: {
                        label = "beforeInvoking";
                        break;
                    }
                    case KEY_ARTHAS_ADVICE_AFTER_INVOKING_METHOD: {
                        label = "afterInvoking";
                        break;
                    }
                    case KEY_ARTHAS_ADVICE_THROW_INVOKING_METHOD: {
                        label = "throwInvoking";
                        break;
                    }
                    default: {
                        throw new IllegalStateException("illegal tracing type: " + tracingType);
                    }
                }

                codeLockForTracing.lock(new CodeLock.Block() {
                    @Override
                    public void code() {

                        final StringBuilder append = new StringBuilder();
                        _debug(append, "debug:" + label + "()");
                        //获取通知方法
                        loadAdviceMethod(tracingType);
                        _debug(append, "loadAdviceMethod()");
                        //将NULL推入堆栈（执行static方法）
                        pushNull();
                        //获取执行通知的参数
                        loadArrayForInvokeTracing(owner, name, desc);
                        _debug(append, "loadArrayForInvokeTracing()");
                        //执行通知方法
                        invokeVirtual(ASM_TYPE_METHOD, ASM_METHOD_METHOD_INVOKE);
                        //常用于舍弃调用指令的返回结果
                        //例如调用了方法但是却不用其返回值
                        pop();
                        _debug(append, "invokeVirtual()");

                    }
                });

            }

            @Override
            public void visitMethodInsn(int opcode, final String owner, final String name, final String desc, boolean itf) {
                if (isSuperOrSiblingConstructorCall(opcode, owner, name)) {
                    super.visitMethodInsn(opcode, owner, name, desc, itf);
                    return;
                }

                if (!isTracing || codeLockForTracing.isLock()) {
                    super.visitMethodInsn(opcode, owner, name, desc, itf);
                    return;
                }

                //是否要对JDK内部的方法调用进行trace
                if (skipJDKTrace && owner.startsWith("java/")) {
                    super.visitMethodInsn(opcode, owner, name, desc, itf);
                    return;
                }

                // 方法调用前通知
                tracing(KEY_ARTHAS_ADVICE_BEFORE_INVOKING_METHOD, owner, name, desc);

                final Label beginLabel = new Label();
                final Label endLabel = new Label();
                final Label finallyLabel = new Label();

                // try
                // {

                mark(beginLabel);
                super.visitMethodInsn(opcode, owner, name, desc, itf);
                mark(endLabel);

                // 方法调用后通知
                tracing(KEY_ARTHAS_ADVICE_AFTER_INVOKING_METHOD, owner, name, desc);
                goTo(finallyLabel);

                // }
                // catch
                // {

                catchException(beginLabel, endLabel, ASM_TYPE_THROWABLE);
                tracing(KEY_ARTHAS_ADVICE_THROW_INVOKING_METHOD, owner, name, desc);

                throwException();

                // }
                // finally
                // {
                mark(finallyLabel);
                // }
            }
        };
    }
```