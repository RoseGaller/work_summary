# 如何启用？

启动类添加@EnableZuulProxy ，创建Marker，表明自动装配时，装配zuul

在配置文件中，配置注册中心地址、路由信息

# 自动装配

## AutoConfiguration

```java
@EnableCircuitBreaker// Hystrix
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({ZuulProxyMarkerConfiguration.class})//注册Marker实例
public @interface EnableZuulProxy {
}
```

```java
@Bean
public ZuulProxyMarkerConfiguration.Marker zuulProxyMarkerBean() {
    return new ZuulProxyMarkerConfiguration.Marker();
}
```

spring.factories

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.springframework.cloud.netflix.zuul.ZuulServerAutoConfiguration,org.springframework.cloud.netflix.zuul.ZuulProxyAutoConfiguration
```

```java
@Configuration
@Import({ RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration.class,   RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration.class,     RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration.class, HttpClientConfiguration.class })
@ConditionalOnBean(ZuulProxyMarkerConfiguration.Marker.class)
public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguration {
```

```java
@Configuration
//zuul配置文件
@EnableConfigurationProperties({ZuulProperties.class}) 
//可以加载到此class
@ConditionalOnClass({ZuulServlet.class}) 
//Marker已经实例化
@ConditionalOnBean({Marker.class})
//服务端配置
@Import({ServerPropertiesAutoConfiguration.class})
public class ZuulServerAutoConfiguration {}
```

## RouteLocator 

```java
@Bean
@Primary //主（当同一个类型有多个实例时，选择Primary）
public CompositeRouteLocator primaryRouteLocator(Collection<RouteLocator> routeLocators) {
  	//首先注入所有的路由
    return new CompositeRouteLocator(routeLocators);
}
 //简单路由
@Bean
@ConditionalOnMissingBean({SimpleRouteLocator.class})
public SimpleRouteLocator simpleRouteLocator() {
    return new SimpleRouteLocator(this.server.getServletPrefix(), this.zuulProperties);
}


@Bean
@ConditionalOnMissingBean(DiscoveryClientRouteLocator.class)
public DiscoveryClientRouteLocator discoveryRouteLocator() {
  return new DiscoveryClientRouteLocator(this.server.getServlet().getContextPath(), this.discovery, this.zuulProperties,
                                         this.serviceRouteMapper, this.registration);
}

```

## Controller

```java
//controller
@Bean
public ZuulController zuulControlleco) {
  	//内部实例化ZuulServlet
    return new ZuulController(); 
}
//HandlerMapping
@Bean
public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes) {
    ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, this.zuulController());
    mapping.setErrorController(this.errorController);
    return mapping;
}
```

```java
public ZuulController() {
    this.setServletClass(ZuulServlet.class);
    this.setServletName("zuul");
    this.setSupportedMethods((String[])null);
}
```

org.springframework.web.servlet.mvc.ServletWrappingController#afterPropertiesSet

```java
public void afterPropertiesSet() throws Exception {
    if (this.servletClass == null) {
        throw new IllegalArgumentException("'servletClass' is required");
    } else {
        if (this.servletName == null) {
            this.servletName = this.beanName;
        }
				//实例化ZuulServlet
        this.servletInstance = (Servlet)this.servletClass.newInstance();
        this.servletInstance.init(new ServletWrappingController.DelegatingServletConfig());
    }
}
```

org.springframework.cloud.netflix.zuul.web.ZuulController#handleRequest

```java
public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ModelAndView var3;
    try {
        var3 = super.handleRequestInternal(request, response);
    } finally {
        RequestContext.getCurrentContext().unset();
    }

    return var3;
}
```

org.springframework.web.servlet.mvc.ServletWrappingController#handleRequestInternal

```java
protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response) throws Exception {
    Assert.state(this.servletInstance != null, "No Servlet instance");
 		 //ZuulServlet
    this.servletInstance.service(request, response);
    return null;
}
```

org.springframework.cloud.netflix.zuul.web.ZuulHandlerMapping#registerHandlers

```java
private void registerHandlers() {
   //获取路由信息，执行SimpleRouteLocator、DiscoveryClientRouteLocator
    Collection<Route> routes = this.routeLocator.getRoutes();
    if (routes.isEmpty()) {
        this.logger.warn("No routes found from RouteLocator");
    } else {
        Iterator var2 = routes.iterator();
     	 //根据路由注册handler，即ZuulController
        while(var2.hasNext()) {
            Route route = (Route)var2.next();
            this.registerHandler(route.getFullPath(), this.zuul);
        }
    }
}
```

## ZuulFilter

### Pre类型过滤器

```java
@Bean
public ServletDetectionFilter servletDetectionFilter() {
    return new ServletDetectionFilter();
}

@Bean
public FormBodyWrapperFilter formBodyWrapperFilter() {
    return new FormBodyWrapperFilter();
}

@Bean
public DebugFilter debugFilter() {
    return new DebugFilter();
}

@Bean
public Servlet30WrapperFilter servlet30WrapperFilter() {
    return new Servlet30WrapperFilter();
}
@Bean
@ConditionalOnMissingBean(PreDecorationFilter.class)
public PreDecorationFilter preDecorationFilter(RouteLocator routeLocator, ProxyRequestHelper proxyRequestHelper) {
  	return new PreDecorationFilter(routeLocator, this.server.getServlet().getContextPath(), this.zuulProperties,
                                 proxyRequestHelper);
}
```

### Route类型

```java
//带有负载均衡的过滤器
//serviceId are found in RequestContext
@Bean
@ConditionalOnMissingBean(RibbonRoutingFilter.class)
public RibbonRoutingFilter ribbonRoutingFilter(ProxyRequestHelper helper,
      RibbonCommandFactory<?> ribbonCommandFactory) {
   RibbonRoutingFilter filter = new RibbonRoutingFilter(helper, ribbonCommandFactory,
         this.requestCustomizers);
   return filter;
}
@Override
public boolean shouldFilter() {
  RequestContext ctx = RequestContext.getCurrentContext();
  return (ctx.getRouteHost() == null && ctx.get(SERVICE_ID_KEY) != null&& ctx.sendZuulResponse());
}
```

```java
//URL are found in RequestContext
@Bean
@ConditionalOnMissingBean({SimpleHostRoutingFilter.class, CloseableHttpClient.class})
public SimpleHostRoutingFilter simpleHostRoutingFilter(ProxyRequestHelper helper,
      ZuulProperties zuulProperties,
      ApacheHttpClientConnectionManagerFactory connectionManagerFactory,
      ApacheHttpClientFactory httpClientFactory) {
   return new SimpleHostRoutingFilter(helper, zuulProperties,
         connectionManagerFactory, httpClientFactory);
}
@Override
public boolean shouldFilter() {
  return RequestContext.getCurrentContext().getRouteHost() != null
    && RequestContext.getCurrentContext().sendZuulResponse();
}
```

### Post类型过滤器

```java
@Bean
public SendResponseFilter sendResponseFilter(ZuulProperties properties) {
    return new SendResponseFilter(this.zuulProperties);
}
```

### Error类型

```java
@Bean
public SendErrorFilter sendErrorFilter() {
    return new SendErrorFilter();
}
```

## 自定义过滤器

```
1、继承ZuulFilter，重写filterType、filterOrder、run方法
2、如果不再执行后续的过滤器，直接返回，可设置RequestContext.getCurrentContext().setSendZuulResponse(false);
还可以直接抛出ZuulException类型的异常
```

# 处理流程

com.netflix.zuul.http.ZuulServlet#service

```java
public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
    try {
        this.init((HttpServletRequest)servletRequest, (HttpServletResponse)servletResponse);
        RequestContext context = RequestContext.getCurrentContext();
        context.setZuulEngineRan();

        try { 
            this.preRoute(); //前置处理
        } catch (ZuulException var13) { //只捕获ZuulException
            this.error(var13); //异常处理
            this.postRoute(); //后置处理
            return;
        }

        try {
            this.route(); //处理请求
        } catch (ZuulException var12) {
            this.error(var12);
            this.postRoute();
            return;
        }

        try {
            this.postRoute(); //后置处理
        } catch (ZuulException var11) {
            this.error(var11);
        }
    } catch (Throwable var14) { //其他类型异常
        this.error(new ZuulException(var14, 500, "UNHANDLED_EXCEPTION_" + var14.getClass().getName())); //处理error
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}
```

com.netflix.zuul.FilterProcessor#runFilters

```java
public Object runFilters(String sType) throws Throwable {
    if (RequestContext.getCurrentContext().debugRouting()) {
        Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
    }
    boolean bResult = false;
  //根据类型获取排好序的ZuulFilter
    List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
    if (list != null) {
        for (int i = 0; i < list.size(); i++) {
            ZuulFilter zuulFilter = list.get(i);
            Object result = processZuulFilter(zuulFilter);
            if (result != null && result instanceof Boolean) {
                bResult |= ((Boolean) result);
            }
        }
    }
    return bResult;
}
```

```java
public Object processZuulFilter(ZuulFilter filter) throws ZuulException {

    RequestContext ctx = RequestContext.getCurrentContext();
    boolean bDebug = ctx.debugRouting();
    final String metricPrefix = "zuul.filter-";
    long execTime = 0;
    String filterName = "";
    try {
        long ltime = System.currentTimeMillis();
        filterName = filter.getClass().getSimpleName();
        
        RequestContext copy = null;
        Object o = null;
        Throwable t = null;

        if (bDebug) {
            Debug.addRoutingDebug("Filter " + filter.filterType() + " " + filter.filterOrder() + " " + filterName);
            copy = ctx.copy();
        }
        
        ZuulFilterResult result = filter.runFilter();
        ExecutionStatus s = result.getStatus();
        execTime = System.currentTimeMillis() - ltime;

        switch (s) {
            case FAILED:
                t = result.getException();
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                break;
            case SUCCESS:
                o = result.getResult();
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
                if (bDebug) {
                    Debug.addRoutingDebug("Filter {" + filterName + " TYPE:" + filter.filterType() + " ORDER:" + filter.filterOrder() + "} Execution time = " + execTime + "ms");
                    Debug.compareContextState(filterName, copy);
                }
                break;
            default:
                break;
        }
        
        if (t != null) throw t;

        usageNotifier.notify(filter, s);
        return o;

    } catch (Throwable e) {
        if (bDebug) {
            Debug.addRoutingDebug("Running Filter failed " + filterName + " type:" + filter.filterType() + " order:" + filter.filterOrder() + " " + e.getMessage());
        }
        usageNotifier.notify(filter, ExecutionStatus.FAILED);
        if (e instanceof ZuulException) {
            throw (ZuulException) e;
        } else {
            ZuulException ex = new ZuulException(e, "Filter threw Exception", 500, filter.filterType() + ":" + filterName);
            ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
            throw ex;
        }
    }
}
```

com.netflix.zuul.ZuulFilter#runFilter

```java
public ZuulFilterResult runFilter() {
    ZuulFilterResult zr = new ZuulFilterResult();
    if (!isFilterDisabled()) {
        if (shouldFilter()) { //先判断是否执行
            Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
            try {
                Object res = run(); //执行run方法
                zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
            } catch (Throwable e) {
                t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                zr.setException(e);
            } finally {
                t.stopAndLog();
            }
        } else {
            zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
        }
    }
    return zr;
}
```

# 如何刷新路由信息？

自定义路由器实现接口RefreshableRouteLocator，继承SimpleRouteLocator

定时通过ApplicationEventPublisher发布RoutesRefreshedEvent

org.springframework.cloud.netflix.zuul.ZuulServerAutoConfiguration

```java
@Bean
public ApplicationListener<ApplicationEvent> zuulRefreshRoutesListener() {
   return new ZuulRefreshListener();
}
```

org.springframework.cloud.netflix.zuul.ZuulServerAutoConfiguration.ZuulRefreshListener#onApplicationEvent

```java
public void onApplicationEvent(ApplicationEvent event) {
   if (event instanceof ContextRefreshedEvent
         || event instanceof RefreshScopeRefreshedEvent
         //路由刷新事件
         || event instanceof RoutesRefreshedEvent
         || event instanceof InstanceRegisteredEvent) {
      reset();
   }
   else if (event instanceof ParentHeartbeatEvent) {
      ParentHeartbeatEvent e = (ParentHeartbeatEvent) event;
      resetIfNeeded(e.getValue());
   }
   else if (event instanceof HeartbeatEvent) {
      HeartbeatEvent e = (HeartbeatEvent) event;
      resetIfNeeded(e.getValue());
   }
}
```

```java
private void reset() {
  //路由发生了变更
   this.zuulHandlerMapping.setDirty(true);
}
```

```java
@Bean
@Primary
public CompositeRouteLocator primaryRouteLocator(
  Collection<RouteLocator> routeLocators) {
  //类似一个组合bean
  return new CompositeRouteLocator(routeLocators);
}
@Bean
public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes) {
   ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, zuulController());
   mapping.setErrorController(this.errorController);
   mapping.setCorsConfigurations(getCorsConfigurations());
   return mapping;
}
```

org.springframework.cloud.netflix.zuul.web.ZuulHandlerMapping#setDirty

```java
public void setDirty(boolean dirty) {
   this.dirty = dirty;
   if (this.routeLocator instanceof RefreshableRouteLocator) {
      ((RefreshableRouteLocator) this.routeLocator).refresh();
   }
}
```

org.springframework.cloud.netflix.zuul.filters.CompositeRouteLocator#refresh

```java
public void refresh() {
   for (RouteLocator locator : routeLocators) {
      if (locator instanceof RefreshableRouteLocator) {
        //刷新路由信息
         ((RefreshableRouteLocator) locator).refresh();
      }
   }
}
```