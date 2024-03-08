# Chaosblade-exec-Jvm

https://chaosblade-io.gitbook.io/chaosblade-help-zh-cn/

混沌工程，借助Jvm-Sandbox实现注入故障

# SandboxModule

启动之初，加载所有的模块时调用onLoad方法，激活模块时调用onActive方法

## onLoad

com.alibaba.chaosblade.exec.bootstrap.jvmsandbox.SandboxModule#onLoad

```java
public void onLoad() throws Throwable {
    LOGGER.info("load chaosblade module");
    ManagerFactory.getListenerManager().setPluginLifecycleListener(this);
  	//注册处理用户请求的处理器
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

## onActive

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

# create

注入延迟故障案例

请求延迟 3 秒，影响 2 条请求

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
  	//启动时，会注册CreateHandler、DestroyHandler、StatusHandler
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
   
    // 例如对Servlet、Jedis、Mysql、Dubbo、Http等注入故障
    ModelSpec modelSpec = this.modelSpecManager.getModelSpec(target);
    if (modelSpec == null) {
        return Response.ofFailure(Response.Code.ILLEGAL_PARAMETER, "the target not supported");
    }
    
    //故障类型，例如DelayActionSpec延迟故障
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

# References

https://chaosblade-io.gitbook.io/chaosblade-help-zh-cn/