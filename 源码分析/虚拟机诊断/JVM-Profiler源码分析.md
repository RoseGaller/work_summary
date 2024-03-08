# JVM Profiler源码分析

* [概述](#概述)
* [启动例子](#启动例子)
* [组件](#组件)
  * [Profiler](#profiler)
  * [Reporter](#reporter)
  * [Transformer](#transformer)
* [JavaAgent](#javaagent)
* [创建Profiler](#创建profiler)
* [启动Profiler](#启动profiler)
* [Transformer](#transformer-1)
  * [修改字节码](#修改字节码)
  * [增强方法](#增强方法)


# 概述

Uber JVM Profiler提供了一个Java Agent来收集Hadoop/Spark JVM进程的各种指标（CPU/内存/IO指标）和堆栈跟踪

Uber JVM Profiler还提供了高级的分析功能，无需修改用户代码便可以跟踪任意的Java方法和参数。此特性可用于为每个Spark应用程序跟踪HDFS集群中namenode的调用延迟并识别namenode的瓶颈。它还可以跟踪每个Spark应用程序读写的HDFS文件路径并识别热文件，以便进一步优化。

# 启动例子

```
java -javaagent:target/jvm-profiler-1.0.0.jar=
reporter=com.uber.profiling.reporters.ConsoleOutputReporter,
tag=mytag,metricInterval=5000,
durationProfiling=com.uber.profiling.examples.HelloWorldApplication.publicSleepMethod,
argumentProfiling=com.uber.profiling.examples.HelloWorldApplication.publicSleepMethod.1,
sampleInterval=100 
-cp target/jvm-profiler-1.0.0.jar com.uber.profiling.examples.HelloWorldApplication

```

# 组件

## Profiler

负责收集监测信息

com.uber.profiling.profilers.IOProfiler

com.uber.profiling.profilers.MethodDurationProfiler

com.uber.profiling.profilers.ProcessInfoProfiler

com.uber.profiling.profilers.StacktraceReporterProfiler

com.uber.profiling.profilers.CpuAndMemoryProfiler

## Reporter

负责上报检测信息

com.uber.profiling.reporters.ConsoleOutputReporter

com.uber.profiling.reporters.FileOutputReporter

com.uber.profiling.reporters.GraphiteOutputReporter

com.uber.profiling.reporters.KafkaOutputReporter

## Transformer

修改字节码文件，记录方法参数、执行时间

# JavaAgent

com.uber.profiling.Agent#premain

```java
public static void premain(final String args, final Instrumentation instrumentation) {//在main执行之前调用此方法
    System.out.println("Java Agent " + AgentImpl.VERSION + " premain args: " + args);
    //解析参数
    Arguments arguments = Arguments.parseArgs(args);
    arguments.runConfigProvider();
    //创建启动Profiler，添加自定义的Transformer到instrumentation
    agentImpl.run(arguments, instrumentation, null);
}
```

com.uber.profiling.AgentImpl#run

```java
public void run(Arguments arguments, Instrumentation instrumentation, Collection<AutoCloseable> objectsToCloseOnShutdown) {
    if (arguments.isNoop()) {
        logger.info("Agent noop is true, do not run anything");
        return;
    }
    //负责上报监测信息
    Reporter reporter = arguments.getReporter();
    String processUuid = UUID.randomUUID().toString();
    String appId = null;
    String appIdVariable = arguments.getAppIdVariable();
    if (appIdVariable != null && !appIdVariable.isEmpty()) {
        appId = System.getenv(appIdVariable);
    }
    if (appId == null || appId.isEmpty()) {
        appId = SparkUtils.probeAppId(arguments.getAppIdRegex());
    }
    //添加自定义的Transformer，对方法增强
    if (!arguments.getDurationProfiling().isEmpty()
            || !arguments.getArgumentProfiling().isEmpty()) {
        //e.g. com.uber.profiling.examples.HelloWorldApplication.publicSleepMethod.1
        //e.g. com.uber.profiling.examples.HelloWorldApplication.publicSleepMethod,e.g. com.uber.profiling.examples.HelloWorldApplication.*
        instrumentation.addTransformer(new JavaAgentFileTransformer(arguments.getDurationProfiling(), arguments.getArgumentProfiling()));
    }
    //创建Profiler
    List<Profiler> profilers = createProfilers(reporter, arguments, processUuid, appId);
    //启动Profiler
    ProfilerGroup profilerGroup = startProfilers(profilers);
    //注册关闭钩子
    Thread shutdownHook = new Thread(new ShutdownHookRunner(profilerGroup.getPeriodicProfilers(), Arrays.asList(reporter), objectsToCloseOnShutdown));
    Runtime.getRuntime().addShutdownHook(shutdownHook);
}
```

# 创建Profiler

```java
private List<Profiler> createProfilers(Reporter reporter, Arguments arguments, String processUuid, String appId) {
    String tag = arguments.getTag();
    //集群名称
    String cluster = arguments.getCluster();
    //上报的间隔
    long metricInterval = arguments.getMetricInterval();

    List<Profiler> profilers = new ArrayList<>();
    //监测cpu、memory
    CpuAndMemoryProfiler cpuAndMemoryProfiler = new CpuAndMemoryProfiler(reporter);
    cpuAndMemoryProfiler.setTag(tag);
    cpuAndMemoryProfiler.setCluster(cluster);
    cpuAndMemoryProfiler.setIntervalMillis(metricInterval);
    cpuAndMemoryProfiler.setProcessUuid(processUuid);
    cpuAndMemoryProfiler.setAppId(appId);

    profilers.add(cpuAndMemoryProfiler);
    //监测进程
    ProcessInfoProfiler processInfoProfiler = new ProcessInfoProfiler(reporter);
    processInfoProfiler.setTag(tag);
    processInfoProfiler.setCluster(cluster);
    processInfoProfiler.setProcessUuid(processUuid);
    processInfoProfiler.setAppId(appId);

    profilers.add(processInfoProfiler);

    //监测方法耗时
    if (!arguments.getDurationProfiling().isEmpty()) {
        //暂存检测信息，对信息进行本地合并
        ClassAndMethodLongMetricBuffer classAndMethodMetricBuffer = new ClassAndMethodLongMetricBuffer();
        //周期性从classAndMethodMetricBuffer获取信息并上报，将之前的暂存信息清空
        MethodDurationProfiler methodDurationProfiler = new MethodDurationProfiler(classAndMethodMetricBuffer, reporter);
        methodDurationProfiler.setTag(tag);
        methodDurationProfiler.setCluster(cluster);
        methodDurationProfiler.setIntervalMillis(metricInterval);
        methodDurationProfiler.setProcessUuid(processUuid);
        methodDurationProfiler.setAppId(appId);

        MethodDurationCollector methodDurationCollector = new MethodDurationCollector(classAndMethodMetricBuffer);
        //代理收集检测信息
        MethodProfilerStaticProxy.setCollector(methodDurationCollector);

        profilers.add(methodDurationProfiler);
    }
    //监测方法参数
    if (!arguments.getArgumentProfiling().isEmpty()) {
        ClassMethodArgumentMetricBuffer classAndMethodArgumentBuffer = new ClassMethodArgumentMetricBuffer();

        MethodArgumentProfiler methodArgumentProfiler = new MethodArgumentProfiler(classAndMethodArgumentBuffer, reporter);
        methodArgumentProfiler.setTag(tag);
        methodArgumentProfiler.setCluster(cluster);
        methodArgumentProfiler.setIntervalMillis(metricInterval);
        methodArgumentProfiler.setProcessUuid(processUuid);
        methodArgumentProfiler.setAppId(appId);

        MethodArgumentCollector methodArgumentCollector = new MethodArgumentCollector(classAndMethodArgumentBuffer);
        //代理收集检测信息
        MethodProfilerStaticProxy.setArgumentCollector(methodArgumentCollector);

        profilers.add(methodArgumentProfiler);
    }
    //监测堆栈信息
    if (arguments.getSampleInterval() > 0) {
        //暂存堆栈信息
        StacktraceMetricBuffer stacktraceMetricBuffer = new StacktraceMetricBuffer();
        //定时收集堆栈信息，存放到stacktraceMetricBuffer
        StacktraceCollectorProfiler stacktraceCollectorProfiler = new StacktraceCollectorProfiler(stacktraceMetricBuffer, AgentThreadFactory.NAME_PREFIX);
        stacktraceCollectorProfiler.setIntervalMillis(arguments.getSampleInterval());
        //定时从stacktraceMetricBuffer获取堆栈信息并上报
        StacktraceReporterProfiler stacktraceReporterProfiler = new StacktraceReporterProfiler(stacktraceMetricBuffer, reporter);
        stacktraceReporterProfiler.setTag(tag);
        stacktraceReporterProfiler.setCluster(cluster);
        stacktraceReporterProfiler.setIntervalMillis(metricInterval);
        stacktraceReporterProfiler.setProcessUuid(processUuid);
        stacktraceReporterProfiler.setAppId(appId);

        profilers.add(stacktraceCollectorProfiler);
        profilers.add(stacktraceReporterProfiler);
    }
    //检测IO信息
    if (arguments.isIoProfiling()) {
        IOProfiler ioProfiler = new IOProfiler(reporter);
        ioProfiler.setTag(tag);
        ioProfiler.setCluster(cluster);
        ioProfiler.setIntervalMillis(metricInterval);
        ioProfiler.setProcessUuid(processUuid);
        ioProfiler.setAppId(appId);

        profilers.add(ioProfiler);
    }
    
    return profilers;
}
```

# 启动Profiler

com.uber.profiling.AgentImpl#startProfilers

```java
public ProfilerGroup startProfilers(Collection<Profiler> profilers) {
    if (started) {
        logger.warn("Profilers already started, do not start it again");
        return new ProfilerGroup(new ArrayList<>(), new ArrayList<>());
    }
    //只上报一次
    List<Profiler> oneTimeProfilers = new ArrayList<>();
    //周期性上报
    List<Profiler> periodicProfilers = new ArrayList<>();

    for (Profiler profiler : profilers) {
        if (profiler.getIntervalMillis() == 0) {
            oneTimeProfilers.add(profiler);
        } else if (profiler.getIntervalMillis() > 0) {
            periodicProfilers.add(profiler);
        } else {
            logger.log(String.format("Ignored profiler %s due to its invalid interval %s", profiler, profiler.getIntervalMillis()));
        }
    }
    //启动只上报一次的Profiler
    for (Profiler profiler : oneTimeProfilers) {
        try {
            profiler.profile();
            logger.info("Finished one time profiler: " + profiler);
        } catch (Throwable ex) {
            logger.warn("Failed to run one time profiler: " + profiler, ex);
        }
    }
    //启动周期性上报的Profiler
    for (Profiler profiler : periodicProfilers) {
        try {
            profiler.profile();
            logger.info("Ran periodic profiler (first run): " + profiler);
        } catch (Throwable ex) {
            logger.warn("Failed to run periodic profiler (first run): " + profiler, ex);
        }
    }
    //创建定时器，调度周期性上报的Profiler
    scheduleProfilers(periodicProfilers);
    started = true;

    return new ProfilerGroup(oneTimeProfilers, periodicProfilers);
}
```

# Transformer

com.uber.profiling.transformers.JavaAgentFileTransformer#transform

```java
public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
    try {
        if (className == null || className.isEmpty()) {
            logger.debug("Hit null or empty class name");
            return null;
        }
        //对方法增强
        return transformImpl(loader, className, classfileBuffer);
    } catch (Throwable ex) {
        logger.warn("Failed to transform class " + className, ex);
        return classfileBuffer;
    }
}
```

## 修改字节码

```java
private byte[] transformImpl(ClassLoader loader, String className, byte[] classfileBuffer) {
    //监测方法执行时间、方法的参数
    if (durationProfilingFilter.isEmpty()
            && argumentFilterProfilingFilter.isEmpty()) {
        return null; //不需要增强此类
    }
    //获取类的全限定名
    String normalizedClassName = className.replaceAll("/", ".");
    logger.debug("Checking class for transform: " + normalizedClassName);

    if (!durationProfilingFilter.matchClass(normalizedClassName)
            && !argumentFilterProfilingFilter.matchClass(normalizedClassName)) {
        return null;//不需要增强此类
    }

    byte[] byteCode;  //存放修改后的字节码

    logger.info("Transforming class: " + normalizedClassName);

    try {
        ClassPool classPool = new ClassPool();
        classPool.appendClassPath(new LoaderClassPath(loader));
        final CtClass ctClass;
        try (ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(classfileBuffer)) {
            //基于原字节码文件创建ctClass
            ctClass = classPool.makeClass(byteArrayInputStream);
        }

        CtMethod[] ctMethods = ctClass.getDeclaredMethods(); //获取类的方法
        for (CtMethod ctMethod : ctMethods) {
            //是否监测方法的执行时间
            boolean enableDurationProfiling = durationProfilingFilter.matchMethod(ctClass.getName(), ctMethod.getName());
            //是否获取拦截此方法的参数
            List<Integer> enableArgumentProfiler = argumentFilterProfilingFilter.matchMethod(ctClass.getName(), ctMethod.getName());
            //方法增强
            transformMethod(normalizedClassName, ctMethod, enableDurationProfiling, enableArgumentProfiler);
        }
        //生成字节码
        byteCode = ctClass.toBytecode();
        ctClass.detach(); //删除创建的ctClass，释放内存

    } catch (Throwable ex) {
        ex.printStackTrace();
        logger.warn("Failed to transform class: " + normalizedClassName, ex);
        byteCode = null;
    }
    return byteCode;
}
```

## 增强方法

在原方法的前面追加记录方法的开始执行时间、方法的耗时的变量，追加记录方法开始执行时间的代码，追加收集方法参数的代码

在原方法的后面追加收集方法执行耗时的代码

com.uber.profiling.transformers.JavaAgentFileTransformer#transformMethod

```java
private void transformMethod(String normalizedClassName, CtMethod method, boolean enableDurationProfiling, List<Integer> argumentsForProfile) {
    if (method.isEmpty()) {
        logger.info("Ignored empty class method: " + method.getLongName());
        return;
    }
    if (!enableDurationProfiling && argumentsForProfile.isEmpty()) {
        return;
    }
    try {
        //方法执行时间，增加方法的本地变量
        if (enableDurationProfiling) {
            //开始时间
            method.addLocalVariable("startMillis_java_agent_instrument", CtClass.longType);
            //方法耗时
            method.addLocalVariable("durationMillis_java_agent_instrument", CtClass.longType);
        }

        StringBuilder sb = new StringBuilder();
        sb.append("{");

        if (enableDurationProfiling) {
            sb.append("startMillis_java_agent_instrument = System.currentTimeMillis();"); //记录开始时间
        }
        //追加收集方法的参数的代码
        for (Integer argument : argumentsForProfile) {
            if (argument >= 1) {
                sb.append(String.format("try{com.uber.profiling.transformers.MethodProfilerStaticProxy.collectMethodArgument(\"%s\", \"%s\", %s, String.valueOf($%s));}catch(Throwable ex){ex.printStackTrace();}",
                        normalizedClassName,
                        method.getName(),
                        argument,
                        argument));
            } else {
                sb.append(String.format("try{com.uber.profiling.transformers.MethodProfilerStaticProxy.collectMethodArgument(\"%s\", \"%s\", %s, \"\");}catch(Throwable ex){ex.printStackTrace();}",
                        normalizedClassName,
                        method.getName(),
                        argument,
                        argument));
            }
        }

        sb.append("}");
        //将方法执行的开始时间、参数拦截的方法插入原方法的前面
        method.insertBefore(sb.toString());

        //在原方法后面插入
        if (enableDurationProfiling) {
            //收集方法执行的时间
            method.insertAfter("{" +
                    "durationMillis_java_agent_instrument = System.currentTimeMillis() - startMillis_java_agent_instrument;" +
                    String.format("try{com.uber.profiling.transformers.MethodProfilerStaticProxy.collectMethodDuration(\"%s\", \"%s\", durationMillis_java_agent_instrument);}catch(Throwable ex){ex.printStackTrace();}", normalizedClassName, method.getName()) +
                    // "System.out.println(\"Method Executed in ms: \" + durationMillis);" +
                    "}");
        }

        logger.info("Transformed class method: " + method.getLongName() + ", durationProfiling: " + enableDurationProfiling + ", argumentProfiling: " + argumentsForProfile);
    } catch (Throwable ex) {
        ex.printStackTrace();
        logger.warn("Failed to transform class method: " + method.getLongName(), ex);
    }
}
```

