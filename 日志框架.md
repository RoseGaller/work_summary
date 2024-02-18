# 扩展功能

1、重新加载配置文件

2、自动清理归档日志文件

3、自动压缩归档日志文件

4、支持配置文件的条件处理

5、根据任何给定的运行时属性分离(或筛选)日志记录

6、可以根据用户设置日志级别，不同用户之间不会干扰

# 初始化

org.slf4j.LoggerFactory#getILoggerFactory

```java
public static ILoggerFactory getILoggerFactory() {
 	 //初始为0
    if (INITIALIZATION_STATE == 0) { 
        INITIALIZATION_STATE = 1;
        performInitialization();
    }

    switch(INITIALIZATION_STATE) {
    case 1:
        return TEMP_FACTORY;
    case 2:
        throw new IllegalStateException("org.slf4j.LoggerFactory could not be successfully initialized. See also http://www.slf4j.org/codes.html#unsuccessfulInit");
    case 3:
        //返回LoggerContext
        return StaticLoggerBinder.getSingleton().getLoggerFactory();
    case 4:	
        return NOP_FALLBACK_FACTORY;
    default:
        throw new IllegalStateException("Unreachable code");
    }
}
```

```java
//执行初始化
private static final void performInitialization() { 
    bind();
    if (INITIALIZATION_STATE == 3) {
        versionSanityCheck();
    }

}
```

```java
private static final void bind() {
    String msg;
    try {
      	//加载StaticLoggerBinder.class
        Set staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
      //报告存在多个StaticLoggerBinder.class
        reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
       //获取单例StaticLoggerBinder，执行初始化操作
        StaticLoggerBinder.getSingleton();
      	//初始化完成
        INITIALIZATION_STATE = 3;
        reportActualBinding(staticLoggerBinderPathSet);
        emitSubstituteLoggerWarning();
    } catch (NoClassDefFoundError var2) {
        msg = var2.getMessage();
        if (!messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
            failedBinding(var2);
            throw var2;
        }

        INITIALIZATION_STATE = 4;
        Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
        Util.report("Defaulting to no-operation (NOP) logger implementation");
        Util.report("See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.");
    } catch (NoSuchMethodError var3) {
        msg = var3.getMessage();
        if (msg != null && msg.indexOf("org.slf4j.impl.StaticLoggerBinder.getSingleton()") != -1) {
            INITIALIZATION_STATE = 2;
            Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");
            Util.report("Your binding is version 1.5.5 or earlier.");
            Util.report("Upgrade your binding to version 1.6.x.");
        }

        throw var3;
    } catch (Exception var4) {
        failedBinding(var4);
        throw new IllegalStateException("Unexpected initialization failure", var4);
    }

}
```

```java
private static Set findPossibleStaticLoggerBinderPathSet() {
    LinkedHashSet staticLoggerBinderPathSet = new LinkedHashSet();

    try {
      	//寻找org/slf4j/impl/StaticLoggerBinder.class
        ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
        Enumeration paths;
        if (loggerFactoryClassLoader == null) {
            paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
        } else {
            paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);
        }

        while(paths.hasMoreElements()) {
            URL path = (URL)paths.nextElement();
            staticLoggerBinderPathSet.add(path);
        }
    } catch (IOException var4) {
        Util.report("Error getting resources from path", var4);
    }

    return staticLoggerBinderPathSet;
}
```

org.slf4j.impl

```java
static {
    SINGLETON.init();
}
```

```java
void init() {
    try {
        try {
          //ContextInitializer属于logback
            (new ContextInitializer(this.defaultLoggerContext)).autoConfig();
        } catch (JoranException var2) {
            Util.report("Failed to auto configure default logger context", var2);
        }

        if (!StatusUtil.contextHasStatusListener(this.defaultLoggerContext)) {
            StatusPrinter.printInCaseOfErrorsOrWarnings(this.defaultLoggerContext);
        }

        this.contextSelectorBinder.init(this.defaultLoggerContext, KEY);
        this.initialized = true;
    } catch (Throwable var3) {
        Util.report("Failed to instantiate [" + LoggerContext.class.getName() + "]", var3);
    }

}
```

# 加载配置文件

```
1、配置文件查找顺序
SystemProperties.getProperty("logback.configurationFile")
	logback-test.xml
	logback.groovy
	logback.xml
2、BasicConfigurator(查找不到，通过SPI，加载BasicConfigurator)
```

ch.qos.logback.classic.util.ContextInitializer#autoConfig

```java
public void autoConfig() throws JoranException {
    StatusListenerConfigHelper.installIfAsked(this.loggerContext);
  	//寻找默认的配置文件
    URL url = this.findURLOfDefaultConfigurationFile(true);
  	//解析配置文件
    if (url != null) {
        this.configureByResource(url);
    } else {
        Configurator c = (Configurator)EnvUtil.loadFromServiceLoader(Configurator.class);
        if (c != null) {
            try {
                c.setContext(this.loggerContext);
                c.configure(this.loggerContext);
            } catch (Exception var4) {
                throw new LogbackException(String.format("Failed to initialize Configurator: %s using ServiceLoader", c != null ? c.getClass().getCanonicalName() : "null"), var4);
            }
        } else {
            BasicConfigurator.configure(this.loggerContext);
        }
    }
```

```java
public URL findURLOfDefaultConfigurationFile(boolean updateStatus) {
    ClassLoader myClassLoader = Loader.getClassLoaderOfObject(this);
  //先从system.properties获取logback.configurationFile
    URL url = this.findConfigFileURLFromSystemProperties(myClassLoader, updateStatus);
    if (url != null) {
        return url;
    } else {
        url = this.getResource("logback.groovy", myClassLoader, updateStatus);
        if (url != null) {
            return url;
        } else {
            url = this.getResource("logback-test.xml", myClassLoader, updateStatus);
            return url != null ? url : this.getResource("logback.xml", myClassLoader, updateStatus);
        }
    }
}
```

```java
public void configureByResource(URL url) throws JoranException {
    if (url == null) {
        throw new IllegalArgumentException("URL argument cannot be null");
    } else {
        String urlString = url.toString();
        if (urlString.endsWith("groovy")) {
            if (EnvUtil.isGroovyAvailable()) {
                GafferUtil.runGafferConfiguratorOn(this.loggerContext, this, url);
            } else {
                StatusManager sm = this.loggerContext.getStatusManager();
                sm.add(new ErrorStatus("Groovy classes are not available on the class path. ABORTING INITIALIZATION.", this.loggerContext));
            }
        } else {
            if (!urlString.endsWith("xml")) {
                throw new LogbackException("Unexpected filename extension of file [" + url.toString() + "]. Should be either .groovy or .xml");
            }

            JoranConfigurator configurator = new JoranConfigurator();
            configurator.setContext(this.loggerContext);
            configurator.doConfigure(url);
        }

    }
}
```

```java
public void addInstanceRules(RuleStore rs) {
    super.addInstanceRules(rs);
    rs.addRule(new ElementSelector("configuration"), new ConfigurationAction());
    rs.addRule(new ElementSelector("configuration/contextName"), new ContextNameAction());
    rs.addRule(new ElementSelector("configuration/contextListener"), new LoggerContextListenerAction());
    rs.addRule(new ElementSelector("configuration/insertFromJNDI"), new InsertFromJNDIAction());
    rs.addRule(new ElementSelector("configuration/evaluator"), new EvaluatorAction());
    rs.addRule(new ElementSelector("configuration/appender/sift"), new SiftAction());
    rs.addRule(new ElementSelector("configuration/appender/sift/*"), new NOPAction());
    rs.addRule(new ElementSelector("configuration/logger"), new LoggerAction());
    rs.addRule(new ElementSelector("configuration/logger/level"), new LevelAction());
    rs.addRule(new ElementSelector("configuration/root"), new RootLoggerAction());
    rs.addRule(new ElementSelector("configuration/root/level"), new LevelAction());
    rs.addRule(new ElementSelector("configuration/logger/appender-ref"), new AppenderRefAction());
    rs.addRule(new ElementSelector("configuration/root/appender-ref"), new AppenderRefAction());
    rs.addRule(new ElementSelector("*/if"), new IfAction());
    rs.addRule(new ElementSelector("*/if/then"), new ThenAction());
    rs.addRule(new ElementSelector("*/if/then/*"), new NOPAction());
    rs.addRule(new ElementSelector("*/if/else"), new ElseAction());
    rs.addRule(new ElementSelector("*/if/else/*"), new NOPAction());
    if (PlatformInfo.hasJMXObjectName()) {
        rs.addRule(new ElementSelector("configuration/jmxConfigurator"), new JMXConfiguratorAction());
    }

    rs.addRule(new ElementSelector("configuration/include"), new IncludeAction());
    rs.addRule(new ElementSelector("configuration/consolePlugin"), new ConsolePluginAction());
    rs.addRule(new ElementSelector("configuration/receiver"), new ReceiverAction());
}
```

ch.qos.logback.classic.joran.action.RootLoggerAction#begin

```java
public void begin(InterpretationContext ec, String name, Attributes attributes) {
    inError = false;

    LoggerContext loggerContext = (LoggerContext) this.context;
    root = loggerContext.getLogger(Logger.ROOT_LOGGER_NAME);

    String levelStr = ec.subst(attributes.getValue(ActionConst.LEVEL_ATTRIBUTE));
    if (!OptionHelper.isEmpty(levelStr)) {
        Level level = Level.toLevel(levelStr);
        addInfo("Setting level of ROOT logger to " + level);
        root.setLevel(level);
    }
    ec.pushObject(root);
}

```

```java
private static void reportMultipleBindingAmbiguity(Set staticLoggerBinderPathSet) {
    if (isAmbiguousStaticLoggerBinderPathSet(staticLoggerBinderPathSet)) {
        Util.report("Class path contains multiple SLF4J bindings.");
        Iterator iterator = staticLoggerBinderPathSet.iterator();

        while(iterator.hasNext()) {
            URL path = (URL)iterator.next();
            Util.report("Found binding in [" + path + "]");
        }

        Util.report("See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.");
    }

}
```

# 获取Logger

org.slf4j.LoggerFactory#getLogger(java.lang.Class)

```java
//根据class查找Logger
public static Logger getLogger(Class clazz) {
    return getLogger(clazz.getName());
}
```

ch.qos.logback.classic.LoggerContext#getLogger(java.lang.String)

```java
public final Logger getLogger(String name) {
    if (name == null) {
        throw new IllegalArgumentException("name argument cannot be null");
    } else if ("ROOT".equalsIgnoreCase(name)) { //name为ROOT
        return this.root;
    } else {
        int i = 0;
        Logger logger = this.root;
     	 //先从map中查找是否已经存在
        Logger childLogger = (Logger)this.loggerCache.get(name);
        if (childLogger != null) {
            return childLogger;
        } else {
            int h;
            do {
                h = LoggerNameUtil.getSeparatorIndexOf(name, i);
                String childName;
                if (h == -1) {
                    childName = name;
                } else {
                    childName = name.substring(0, h);
                }

                i = h + 1;
              	//加锁，保证并发安全
                synchronized(logger) { 
                  	//从root中获取
                    childLogger = logger.getChildByName(childName);
                    if (childLogger == null) {//创建Logger
                        childLogger = logger.createChildByName(childName);
                      	//放到map
                        this.loggerCache.put(childName, childLogger);
                        this.incSize();
                    }
                }

                logger = childLogger;
            } while(h != -1);

            return childLogger;
        }
    }
}
```

```java
Logger createChildByName(String childName) {
    int i_index = LoggerNameUtil.getSeparatorIndexOf(childName, this.name.length() + 1);
    if (i_index != -1) {
        throw new IllegalArgumentException("For logger [" + this.name + "] child name [" + childName + " passed as parameter, may not include '.' after index" + (this.name.length() + 1));
    } else {
        if (this.childrenList == null) {
            this.childrenList = new ArrayList(5);
        }
				//创建Logger，构造中传入parent logger
        Logger childLogger = new Logger(childName, this, this.loggerContext);
        this.childrenList.add(childLogger);
        childLogger.effectiveLevelInt = this.effectiveLevelInt;
        return childLogger;
    }
}
```

# 输出过程

```java
public void error(String msg) {
    this.filterAndLog_0_Or3Plus(FQCN, (Marker)null, Level.ERROR, msg, (Object[])null, (Throwable)null);
}
```

```java
private void filterAndLog_0_Or3Plus(String localFQCN, Marker marker, Level level, String msg, Object[] params, Throwable t) {
  //TurboFilter，判断是否过滤
  //默认没有配置，返回NEUTRAL
    FilterReply decision = this.loggerContext.getTurboFilterChainDecision_0_3OrMore(marker, this, level, msg, params, t);
  	//判断日志级别，比effectiveLevelInt小的不会打印
    if (decision == FilterReply.NEUTRAL) {
        if (this.effectiveLevelInt > level.levelInt) {
            return;
        }
    } else if (decision == FilterReply.DENY) {
        return;
    }

    this.buildLoggingEventAndAppend(localFQCN, marker, level, msg, params, t);
}
```

ch.qos.logback.classic.LoggerContext#getTurboFilterChainDecision_0_3OrMore

```java
final FilterReply getTurboFilterChainDecision_0_3OrMore(final Marker marker, final Logger logger, final Level level, final String format,
                final Object[] params, final Throwable t) {
    if (turboFilterList.size() == 0) {
        return FilterReply.NEUTRAL;
    }
    return turboFilterList.getTurboFilterChainDecision(marker, logger, level, format, params, t);
}
```

```java
private void buildLoggingEventAndAppend(String localFQCN, Marker marker, Level level, String msg, Object[] params, Throwable t) {
  	//创建日志事件
    LoggingEvent le = new LoggingEvent(localFQCN, this, level, msg, t, params);
    le.setMarker(marker);
    this.callAppenders(le);
}
```

```java
public void callAppenders(ILoggingEvent event) {
    int writes = 0;
  	//logger里有appender，logger还有父logger
    for(Logger l = this; l != null; l = l.parent) {
        writes += l.appendLoopOnAppenders(event);
      	//additvity="false" 不继承父元素,否则出现重复输出
        if (!l.additive) {
            break;
        }
    }
 	 	//没有appender
    if (writes == 0) {
        this.loggerContext.noAppenderDefinedWarning(this);
    }
}
```

ch.qos.logback.classic.Logger#appendLoopOnAppenders

```java
private int appendLoopOnAppenders(ILoggingEvent event) {
    return this.aai != null ? this.aai.appendLoopOnAppenders(event) : 0;
}
```

```java
public int appendLoopOnAppenders(E e) {
    int size = 0;

    for(Iterator i$ = this.appenderList.iterator(); i$.hasNext(); ++size) {
        Appender<E> appender = (Appender)i$.next();
        appender.doAppend(e);
    }

    return size;
}
```

ch.qos.logback.core.UnsynchronizedAppenderBase#doAppend

```java
public void doAppend(E eventObject) {
    // WARNING: The guard check MUST be the first statement in the
    // doAppend() method.

    // prevent re-entry.
    if (Boolean.TRUE.equals(guard.get())) {
        return;
    }

    try {
        guard.set(Boolean.TRUE);

        if (!this.started) {
            if (statusRepeatCount++ < ALLOWED_REPEATS) {
                addStatus(new WarnStatus("Attempted to append to non started appender [" + name + "].", this));
            }
            return;
        }

      	//执行过滤器
        if (getFilterChainDecision(eventObject) == FilterReply.DENY) {
            return;
        }

        //执行追加操作
        this.append(eventObject);

    } catch (Exception e) {
        if (exceptionCount++ < ALLOWED_REPEATS) {
            addError("Appender [" + name + "] failed to append.", e);
        }
    } finally {
        guard.set(Boolean.FALSE);
    }
}
//子类实现append方法
abstract protected void append(E eventObject);
```

执行过滤器

```java
public FilterReply getFilterChainDecision(E event) {
    return fai.getFilterChainDecision(event);
}
```

```java
public FilterReply getFilterChainDecision(E event) {

    final Filter<E>[] filterArrray = filterList.asTypedArray();
    final int len = filterArrray.length;

    for (int i = 0; i < len; i++) {
        final FilterReply r = filterArrray[i].decide(event);
      	//只要返回DENY或者ACCEPT，后面的filter不再执行
        if (r == FilterReply.DENY || r == FilterReply.ACCEPT) {
            return r;
        }
    }
    return FilterReply.NEUTRAL;
}
```

控制台打印

ch.qos.logback.core.OutputStreamAppender#append

```java
@Override
protected void append(E eventObject) {
    if (!isStarted()) {
        return;
    }

    subAppend(eventObject);
}
```

```java
protected void subAppend(E event) {
    if (!isStarted()) {
        return;
    }
    try {
        if (event instanceof DeferredProcessingAware) {
            ((DeferredProcessingAware) event).prepareForDeferredProcessing();
        }
				//编码
        byte[] byteArray = this.encoder.encode(event);
        writeBytes(byteArray);

    } catch (IOException ioe) {
        this.started = false;
        addStatus(new ErrorStatus("IO failure in appender", this, ioe));
    }
}
```

```java
private void writeBytes(byte[] byteArray) throws IOException {
    if(byteArray == null || byteArray.length == 0)
        return;
    
    lock.lock();
    try {
        this.outputStream.write(byteArray);
        if (immediateFlush) {
            this.outputStream.flush();
        }
    } finally {
        lock.unlock();
    }
}
```

异步打印

ch.qos.logback.core.AsyncAppenderBase#append

```java
protected void append(E eventObject) {
  //如果剩余容量少于20%，TRACE, DEBUG and INFO 就会丢弃
    if (isQueueBelowDiscardingThreshold() && isDiscardable(eventObject)) {
      return;
    }
  preprocess(eventObject);
  put(eventObject);
}
```

```java
private boolean isQueueBelowDiscardingThreshold() {
    return (blockingQueue.remainingCapacity() < discardingThreshold);
}
```

```java
protected boolean isDiscardable(ILoggingEvent event) {
    Level level = event.getLevel();
    return level.toInt() <= Level.INFO_INT;
}
```

ch.qos.logback.classic.AsyncAppender#preprocess

```java
protected void preprocess(ILoggingEvent eventObject) {
    eventObject.prepareForDeferredProcessing();
    if (includeCallerData)
        eventObject.getCallerData();
}
```

```java
private void put(E eventObject) {
    try {
     	 //放入队列
        this.blockingQueue.put(eventObject);
    } catch (InterruptedException var3) {
    }

}
```

ch.qos.logback.core.AsyncAppenderBase#start

```java
public void start() {
    if (this.appenderCount == 0) {
        this.addError("No attached appenders found.");
    } else if (this.queueSize < 1) {
        this.addError("Invalid queue size [" + this.queueSize + "]");
    } else {
        this.blockingQueue = new ArrayBlockingQueue(this.queueSize);
        if (this.discardingThreshold == -1) {
            this.discardingThreshold = this.queueSize / 5;
        }

        this.addInfo("Setting discardingThreshold to " + this.discardingThreshold);
        this.worker.setDaemon(true);
        this.worker.setName("AsyncAppender-Worker-" + this.getName());
        super.start();
        this.worker.start();
    }
}
```

ch.qos.logback.core.AsyncAppenderBase.Worker#run

```java
public void run() {
    AsyncAppenderBase<E> parent = AsyncAppenderBase.this;
    AppenderAttachableImpl aai = parent.aai;

    while(parent.isStarted()) {
        try {
          	//从队列获取数据
            E e = parent.blockingQueue.take();
          	//appender
            aai.appendLoopOnAppenders(e);
        } catch (InterruptedException var5) {
            break;
        }
    }

    AsyncAppenderBase.this.addInfo("Worker thread will flush remaining events before exiting. ");
    Iterator i$ = parent.blockingQueue.iterator();

    while(i$.hasNext()) {
        E ex = i$.next();
        aai.appendLoopOnAppenders(ex);
        parent.blockingQueue.remove(ex);
    }

    aai.detachAndStopAllAppenders();
}
```

# TurboFilter

TurboFilter对象绑定到日志上下文

因此，它们不仅在使用给定的appender时被调用，而且在每次发出日志记录请求时都被调用。它们的作用范围比附加的过滤器更广

更重要的是，它们在LoggingEvent对象创建之前被调用

TurboFilter对象不需要实例化日志事件来过滤日志请求

因此，TurboFilter用于在创建事件之前对日志事件进行高性能过滤

# Appender

```xml
<property name="LOG_FILE" value="LogFile" />
<property name="LOG_DIR" value="/var/logs/application" />
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_DIR}/${LOG_FILE}.log</file> 
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>${LOG_DIR}/%d{yyyy/MM}/${LOG_FILE}.gz</fileNamePattern>
        <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
</appender>
```



# Layouts

```xml
<layout class="ch.qos.logback.classic.PatternLayout">
    <pattern>%d [%thread] %level %mdc %logger{35} -%msg%n</pattern>
</layout>
```

logback.xml

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```



# 资料

[官方文档](https://logback.qos.ch/index.html)