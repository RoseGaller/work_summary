初始化父容器

ContextLoaderListener继承ContextLoader，实现了接口ServletContextListener

```java
public void contextInitialized(ServletContextEvent event) {
   initWebApplicationContext(event.getServletContext());
}
```

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
  //WebApplicationContext创建成功后，回放入到ServletContext中
   if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
      throw new IllegalStateException(
            "Cannot initialize context because there is already a root application context present - " +
            "check whether you have multiple ContextLoader* definitions in your web.xml!");
   }

   Log logger = LogFactory.getLog(ContextLoader.class);
   servletContext.log("Initializing Spring root WebApplicationContext");
   if (logger.isInfoEnabled()) {
      logger.info("Root WebApplicationContext: initialization started");
   }
   long startTime = System.currentTimeMillis();

   try {
      // Store context in local instance variable, to guarantee that
      // it is available on ServletContext shutdown.
      if (this.context == null) {
         this.context = createWebApplicationContext(servletContext);
      }
      if (this.context instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
         if (!cwac.isActive()) {
            // The context has not yet been refreshed -> provide services such as
            // setting the parent context, setting the application context id, etc
            if (cwac.getParent() == null) {
               // The context instance was injected without an explicit parent ->
               // determine parent for root web application context, if any.
               ApplicationContext parent = loadParentContext(servletContext);
               cwac.setParent(parent);
            }
            configureAndRefreshWebApplicationContext(cwac, servletContext);
         }
      } //将容器放入servletContext中
	      	servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

      ClassLoader ccl = Thread.currentThread().getContextClassLoader();
      if (ccl == ContextLoader.class.getClassLoader()) {
         currentContext = this.context;
      }
      else if (ccl != null) {
         currentContextPerThread.put(ccl, this.context);
      }

      if (logger.isDebugEnabled()) {
         logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
               WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
      }
      if (logger.isInfoEnabled()) {
         long elapsedTime = System.currentTimeMillis() - startTime;
         logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
      }

      return this.context;
   }
   catch (RuntimeException ex) {
      logger.error("Context initialization failed", ex);
      servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
      throw ex;
   }
   catch (Error err) {
      logger.error("Context initialization failed", err);
      servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
      throw err;
   }
}
```

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
   Class<?> contextClass = determineContextClass(sc);
   if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
      throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
            "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
   }
   return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

```java
//决定context的class
protected Class<?> determineContextClass(ServletContext servletContext) {
   String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
   if (contextClassName != null) {
      try {
         return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
      }
      catch (ClassNotFoundException ex) {
         throw new ApplicationContextException(
               "Failed to load custom context class [" + contextClassName + "]", ex);
      }
   }
   else {
     //ContextLoader.properties文件，默认配置XmlWebApplicationContext
      contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
      try {
         return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
      }
      catch (ClassNotFoundException ex) {
         throw new ApplicationContextException(
               "Failed to load default context class [" + contextClassName + "]", ex);
      }
   }
}
```

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) { //刷新容器
   if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
      String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
      if (idParam != null) {
         wac.setId(idParam);
      }
      else {
         wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
               ObjectUtils.getDisplayString(sc.getContextPath()));
      }
   }

   wac.setServletContext(sc);
  //web.xm中的contextConfigLocation
   String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
   if (configLocationParam != null) {
      wac.setConfigLocation(configLocationParam);
   }

   ConfigurableEnvironment env = wac.getEnvironment();
   if (env instanceof ConfigurableWebEnvironment) {
      ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
   }

   customizeContext(sc, wac);
   wac.refresh();
}
```

## 加载默认实现

org.springframework.web.servlet.DispatcherServlet

```java
static {
   // Load default strategy implementations from properties file.
   try {
      ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
     	//DispatcherServlet.properties
      defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
   }
   catch (IOException ex) {
      throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "': " + ex.getMessage());
   }
}
```

DispatcherServlet.properties

```properties
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
   org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
   org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
   org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
   org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
   org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

## 创建springmvc容器

org.springframework.web.servlet.FrameworkServlet#initServletBean

```java
protected final void initServletBean() throws ServletException {
   getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
   if (this.logger.isInfoEnabled()) {
      this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
   }
   long startTime = System.currentTimeMillis();

   try {

      this.webApplicationContext = initWebApplicationContext();
      initFrameworkServlet();
   }
   catch (ServletException | RuntimeException ex) {
      this.logger.error("Context initialization failed", ex);
      throw ex;
   }

   if (this.logger.isInfoEnabled()) {
      long elapsedTime = System.currentTimeMillis() - startTime;
      this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
            elapsedTime + " ms");
   }
}
```

org.springframework.web.servlet.FrameworkServlet#initWebApplicationContext

```java
protected WebApplicationContext initWebApplicationContext() {
  //从ServletContext获取WebApplicationContext
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   WebApplicationContext wac = null;

   if (this.webApplicationContext != null) { //在构造器中已经注入
      wac = this.webApplicationContext;
      if (wac instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
         if (!cwac.isActive()) { //尚未执行refreshed方法
            if (cwac.getParent() == null) {
               cwac.setParent(rootContext); //设置父容器
            }
            configureAndRefreshWebApplicationContext(cwac); //执行refreshed
         }
      }
   }
   if (wac == null) {
     //从ServletContext中获取springmvc容器
      wac = findWebApplicationContext();
   }
   if (wac == null) { //创建子容器，从web.xml文件读取contextConfigLocation参数，
      wac = createWebApplicationContext(rootContext);
   }

   if (!this.refreshEventReceived) {
      onRefresh(wac);
   }

   if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      String attrName = getServletContextAttributeName();
      getServletContext().setAttribute(attrName, wac);
      if (this.logger.isDebugEnabled()) {
         this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
               "' as ServletContext attribute with name [" + attrName + "]");
      }
   }

   return wac;
}
```

org.springframework.web.servlet.FrameworkServlet#createWebApplicationContext(org.springframework.web.context.WebApplicationContext)

```java
protected WebApplicationContext createWebApplicationContext(@Nullable WebApplicationContext parent) {
   return createWebApplicationContext((ApplicationContext) parent);
}
```

org.springframework.web.servlet.FrameworkServlet#createWebApplicationContext(org.springframework.context.ApplicationContext)

```java
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
   Class<?> contextClass = getContextClass();//默认XmlWebApplicationContext
   if (this.logger.isDebugEnabled()) {
      this.logger.debug("Servlet with name '" + getServletName() +
            "' will try to create custom WebApplicationContext context of class '" +
            contextClass.getName() + "'" + ", using parent context [" + parent + "]");
   }
   if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
      throw new ApplicationContextException(
            "Fatal initialization error in servlet with name '" + getServletName() +
            "': custom WebApplicationContext class [" + contextClass.getName() +
            "] is not of type ConfigurableWebApplicationContext");
   }
  //创建XmlWebApplicationContext
   ConfigurableWebApplicationContext wac =
         (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

   wac.setEnvironment(getEnvironment()); //StandardServletEnvironment
   wac.setParent(parent); //设置父容器
   String configLocation = getContextConfigLocation(); //从web.xml中读取
   if (configLocation != null) {
      wac.setConfigLocation(configLocation);
   }
   configureAndRefreshWebApplicationContext(wac); //初始化springmvc容器
   return wac;
}
```

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        if (this.contextId != null) {
            wac.setId(this.contextId);
        } else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX + ObjectUtils.getDisplayString(this.getServletContext().getContextPath()) + '/' + this.getServletName());
        }
    }

    wac.setServletContext(this.getServletContext());
    wac.setServletConfig(this.getServletConfig());
    wac.setNamespace(this.getNamespace());
    wac.addApplicationListener(new SourceFilteringListener(wac, new FrameworkServlet.ContextRefreshListener()));
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment)env).initPropertySources(this.getServletContext(), this.getServletConfig());
    }

    this.postProcessWebApplicationContext(wac);
    this.applyInitializers(wac); //执行ApplicationContextInitializer接口的initialize方法
    wac.refresh();
}
```

org.springframework.web.servlet.DispatcherServlet#onRefresh

```java
protected void onRefresh(ApplicationContext context) {
   initStrategies(context); //初始化策略
}
```

## 初始化策略

org.springframework.web.servlet.DispatcherServlet#initStrategies

```java
protected void initStrategies(ApplicationContext context) {
   initMultipartResolver(context); //初始化文件上传解析器，一般手动配置CommonsMultipartResolver
   initLocaleResolver(context); 
   initThemeResolver(context); 
   initHandlerMappings(context); //初始化HandlerMapping
   initHandlerAdapters(context); //初始化HandlerAdapter
   initHandlerExceptionResolvers(context); //初始化异常拦截器
   initRequestToViewNameTranslator(context); 
   initViewResolvers(context); 
   initFlashMapManager(context); 
}
```

### InitHandlerMapping

org.springframework.web.servlet.DispatcherServlet#initHandlerMappings

```java
private void initHandlerMappings(ApplicationContext context) {
   this.handlerMappings = null;

   if (this.detectAllHandlerMappings) {//默认true，从容器获取实现HandlerMapping接口的bean
      // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
      Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerMappings = new ArrayList<>(matchingBeans.values());
         // We keep HandlerMappings in sorted order.
         AnnotationAwareOrderComparator.sort(this.handlerMappings);
      }
   }
   else {
      try {
         HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
         this.handlerMappings = Collections.singletonList(hm);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerMapping later.
      }
   }

   if (this.handlerMappings == null) { //容器中未获取到，加载DispatcherServlet.properties配置文件中的HandlerMapping,BeanNameUrlHandlerMapping、DefaultAnnotationHandlerMapping
      this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
      if (logger.isDebugEnabled()) {
         logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
      }
   }
}
```

### GetDefaultStrategies

org.springframework.web.servlet.DispatcherServlet#getDefaultStrategies

```java
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) { //从配置文件获取默认实现
   String key = strategyInterface.getName();
   String value = defaultStrategies.getProperty(key);
   if (value != null) {
      String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
      List<T> strategies = new ArrayList<>(classNames.length);
      for (String className : classNames) {
         try {
            Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
            Object strategy = createDefaultStrategy(context, clazz);
            strategies.add((T) strategy);
         }
         catch (ClassNotFoundException ex) {
            throw new BeanInitializationException(
                  "Could not find DispatcherServlet's default strategy class [" + className +
                  "] for interface [" + key + "]", ex);
         }
         catch (LinkageError err) {
            throw new BeanInitializationException(
                  "Unresolvable class definition for DispatcherServlet's default strategy class [" +
                  className + "] for interface [" + key + "]", err);
         }
      }
      return strategies;
   }
   else {
      return new LinkedList<>();
   }
}
```

org.springframework.web.servlet.DispatcherServlet#createDefaultStrategy

```java
protected Object createDefaultStrategy(ApplicationContext context, Class<?> clazz) {
   return context.getAutowireCapableBeanFactory().createBean(clazz);//容器创建bean
}
```

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.Class<T>)

```java
public <T> T createBean(Class<T> beanClass) throws BeansException {
   // Use prototype bean definition, to avoid registering bean as dependent bean.
   RootBeanDefinition bd = new RootBeanDefinition(beanClass); //创建BeanDefinition
   bd.setScope(SCOPE_PROTOTYPE);
   bd.allowCaching = ClassUtils.isCacheSafe(beanClass, getBeanClassLoader());
   return (T) createBean(beanClass.getName(), bd, null); //创建bean
}
```

### InitHandlerMethods

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#afterPropertiesSet

```java
public void afterPropertiesSet() {
   this.config = new RequestMappingInfo.BuilderConfiguration();
   this.config.setUrlPathHelper(getUrlPathHelper());
   this.config.setPathMatcher(getPathMatcher());
   this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
   this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
   this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
   this.config.setContentNegotiationManager(getContentNegotiationManager());

   super.afterPropertiesSet();
}
```

org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#afterPropertiesSet

```java
public void afterPropertiesSet() {
   initHandlerMethods();
}
```

org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#initHandlerMethods

```java
protected void initHandlerMethods() {
   if (logger.isDebugEnabled()) {
      logger.debug("Looking for request mappings in application context: " + getApplicationContext());
   }
  //默认只在子容器中获取
   String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
         BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
         obtainApplicationContext().getBeanNamesForType(Object.class));

   for (String beanName : beanNames) {
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
         Class<?> beanType = null;
         try {
           //根据beannanme获取beantype
            beanType = obtainApplicationContext().getType(beanName);
         }
         catch (Throwable ex) {
            if (logger.isDebugEnabled()) {
               logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
            }
         }
         if (beanType != null && isHandler(beanType)) {
            detectHandlerMethods(beanName);
         }
      }
   }
   handlerMethodsInitialized(getHandlerMethods());
}
```

```java
protected boolean isHandler(Class<?> beanType) {
    return AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) || AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class);
}
```

RequestMappingHandlerAdapter

```java
public void afterPropertiesSet() {
  	//初始化标有ControllerAdvice注解的bean
    this.initControllerAdviceCache();
    List handlers;
  	//参数解析、返回值解析
    if (this.argumentResolvers == null) {
        handlers = this.getDefaultArgumentResolvers();
        this.argumentResolvers = (new HandlerMethodArgumentResolverComposite()).addResolvers(handlers);
    }

    if (this.initBinderArgumentResolvers == null) {
        handlers = this.getDefaultInitBinderArgumentResolvers();
        this.initBinderArgumentResolvers = (new HandlerMethodArgumentResolverComposite()).addResolvers(handlers);
    }

    if (this.returnValueHandlers == null) {
        handlers = this.getDefaultReturnValueHandlers();
        this.returnValueHandlers = (new HandlerMethodReturnValueHandlerComposite()).addHandlers(handlers);
    }

}
```

## 请求入口

org.springframework.web.servlet.DispatcherServlet#doDispatch

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HttpServletRequest processedRequest = request;
   HandlerExecutionChain mappedHandler = null;
   boolean multipartRequestParsed = false;

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
        //检测是否是文件上传的请求
         processedRequest = checkMultipart(request);
         multipartRequestParsed = (processedRequest != request);

         //取得处理当前请求的Controller
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null) {
           // 如果 handler 为空，则返回404
            noHandlerFound(processedRequest, response);
            return;
         }

         // 获取handler适配器,适配三种handler：注有Controller注解、实现HttpRequestHandler接口、实现Controller接口
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

         // Process last-modified header, if supported by the handler.
         String method = request.getMethod();
         boolean isGet = "GET".equals(method);
         if (isGet || "HEAD".equals(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (logger.isDebugEnabled()) {
               logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
            }
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
               return;
            }
         }

         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

         // 实际处理器处理请求，返回结果视图对象
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }
				// 结果视图对象的处理
         applyDefaultViewName(processedRequest, mv);
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         // As of 4.3, we're processing Errors thrown from handler methods as well,
         // making them available for @ExceptionHandler methods and other scenarios.
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
     // 跳转⻚面，渲染视图
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
     //最终会调用HandlerInterceptor的afterCompletion 方法
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
     //最终会调用HandlerInterceptor的afterCompletion 方法
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
      if (asyncManager.isConcurrentHandlingStarted()) {
         // Instead of postHandle and afterCompletion
         if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
         }
      }
      else {
         // Clean up any resources used by a multipart request.
         if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
         }
      }
   }
}
```

getHandler

org.springframework.web.servlet.DispatcherServlet#getHandler

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   if (this.handlerMappings != null) {
      for (HandlerMapping hm : this.handlerMappings) {
         if (logger.isTraceEnabled()) {
            logger.trace(
                  "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
         }
         HandlerExecutionChain handler = hm.getHandler(request);
         if (handler != null) {
            return handler;
         }
      }
   }
   return null;
}
```

org.springframework.web.servlet.DispatcherServlet#getHandlerAdapter

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   if (this.handlerAdapters != null) {
      for (HandlerAdapter ha : this.handlerAdapters) {
         if (logger.isTraceEnabled()) {
            logger.trace("Testing handler adapter [" + ha + "]");
         }
         if (ha.supports(handler)) {
            return ha;
         }
      }
   }
   throw new ServletException("No adapter for handler [" + handler +
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

### 前置拦截

org.springframework.web.servlet.HandlerExecutionChain#applyPreHandle

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
      for (int i = 0; i < interceptors.length; i++) {
         HandlerInterceptor interceptor = interceptors[i];
         if (!interceptor.preHandle(request, response, this.handler)) {
            triggerAfterCompletion(request, response, null);
            return false;
         }
         this.interceptorIndex = i;
      }
   }
   return true;
}
```

### 处理请求

```java
public interface HandlerAdapter { //适配器
    boolean supports(Object var1);
    @Nullable
    ModelAndView handle(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;
    long getLastModified(HttpServletRequest var1, Object var2);
}
```

SimpleControllerHandlerAdapter

```java
public boolean supports(Object handler) { //实现Controller的handler
    return handler instanceof Controller;
}
```

```java
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    return ((Controller)handler).handleRequest(request, response);
}
```

RequestMappingHandlerAdapter

org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter

```java
public final boolean supports(Object handler) { //使用了controller注解的handler
    return handler instanceof HandlerMethod && this.supportsInternal((HandlerMethod)handler);
}
```

```java
protected boolean supportsInternal(HandlerMethod handlerMethod) {
    return true;
}
```

org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter#handle

```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
      throws Exception {
   return handleInternal(request, response, (HandlerMethod) handler);
}
```

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#handleInternal

```java
protected ModelAndView handleInternal(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ModelAndView mav;
   checkRequest(request);

   // Execute invokeHandlerMethod in synchronized block if required.
   if (this.synchronizeOnSession) {
      HttpSession session = request.getSession(false);
      if (session != null) {
         Object mutex = WebUtils.getSessionMutex(session);
         synchronized (mutex) {
            mav = invokeHandlerMethod(request, response, handlerMethod);
         }
      }
      else {
         // No HttpSession available -> no mutex necessary
         mav = invokeHandlerMethod(request, response, handlerMethod);
      }
   }
   else {
      // No synchronization on session demanded at all...
      mav = invokeHandlerMethod(request, response, handlerMethod);
   }

   if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
      if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
         applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
      }
      else {
         prepareResponse(response);
      }
   }

   return mav;
}
```

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#invokeHandlerMethod

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   try {
      WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

      ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
      if (this.argumentResolvers != null) {
          //解析参数
         invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
      }
      if (this.returnValueHandlers != null) {
         //封装返回值
         invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
      }
      invocableMethod.setDataBinderFactory(binderFactory);
      //查找参数名称
      invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

      ModelAndViewContainer mavContainer = new ModelAndViewContainer();
      mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
      modelFactory.initModel(webRequest, mavContainer, invocableMethod);
      mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

      AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
      asyncWebRequest.setTimeout(this.asyncRequestTimeout);

      WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
      asyncManager.setTaskExecutor(this.taskExecutor);
      asyncManager.setAsyncWebRequest(asyncWebRequest);
      asyncManager.registerCallableInterceptors(this.callableInterceptors);
      asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

      if (asyncManager.hasConcurrentResult()) {
         Object result = asyncManager.getConcurrentResult();
         mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
         asyncManager.clearConcurrentResult();
         if (logger.isDebugEnabled()) {
            logger.debug("Found concurrent result value [" + result + "]");
         }
         invocableMethod = invocableMethod.wrapConcurrentResult(result);
      }

      invocableMethod.invokeAndHandle(webRequest, mavContainer);
      if (asyncManager.isConcurrentHandlingStarted()) {
         return null;
      }

      return getModelAndView(mavContainer, modelFactory, webRequest);
   }
   finally {
      webRequest.requestCompleted();
   }
}
```

org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
		//执行请求
   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   setResponseStatus(webRequest);

   if (returnValue == null) {
      if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
         mavContainer.setRequestHandled(true);
         return;
      }
   }
   else if (StringUtils.hasText(getResponseStatusReason())) {
      mavContainer.setRequestHandled(true);
      return;
   }

   mavContainer.setRequestHandled(false);
   Assert.state(this.returnValueHandlers != null, "No return value handlers");
   try {
     	//封装响应信息
      this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   }
   catch (Exception ex) {
      if (logger.isTraceEnabled()) {
         logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
      }
      throw ex;
   }
}
```

执行请求

org.springframework.web.method.support.InvocableHandlerMethod#invokeForRequest

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
	//解析参数
   Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
   if (logger.isTraceEnabled()) { 
      logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
            "' with arguments " + Arrays.toString(args));
   }
  //执行具体的业务方法
   Object returnValue = doInvoke(args); 
   if (logger.isTraceEnabled()) {
      logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
            "] returned [" + returnValue + "]");
   }
   return returnValue;
}
```

#### 获取方法参数

org.springframework.web.method.support.InvocableHandlerMethod#getMethodArgumentValues

```java
private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {//获取参数值
   MethodParameter[] parameters = getMethodParameters();
   Object[] args = new Object[parameters.length];
   for (int i = 0; i < parameters.length; i++) {
      MethodParameter parameter = parameters[i];

      parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
      args[i] = resolveProvidedArgument(parameter, providedArgs);
      if (args[i] != null) {
         continue;
      }
      if (this.argumentResolvers.supportsParameter(parameter)) {//是否支持解析此参数
         try {
            args[i] = this.argumentResolvers.resolveArgument(
                  parameter, mavContainer, request, this.dataBinderFactory); //获取参数解析器，进行参数的解析
            continue;
         }
         catch (Exception ex) {
            if (logger.isDebugEnabled()) {
               logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
            }
            throw ex;
         }
      }
      if (args[i] == null) {
         throw new IllegalStateException("Could not resolve method parameter at index " +
               parameter.getParameterIndex() + " in " + parameter.getExecutable().toGenericString() +
               ": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
      }
   }
   return args;
}
```

#### 执行目标方法

org.springframework.web.method.support.InvocableHandlerMethod#doInvoke

```java
protected Object doInvoke(Object... args) throws Exception {
   ReflectionUtils.makeAccessible(getBridgedMethod());
   try {
      return getBridgedMethod().invoke(getBean(), args);
   }
   catch (IllegalArgumentException ex) {
      assertTargetBean(getBridgedMethod(), getBean(), args);
      String text = (ex.getMessage() != null ? ex.getMessage() : "Illegal argument");
      throw new IllegalStateException(getInvocationErrorMessage(text, args), ex);
   }
   catch (InvocationTargetException ex) {
      // Unwrap for HandlerExceptionResolvers ...
      Throwable targetException = ex.getTargetException();
      if (targetException instanceof RuntimeException) {
         throw (RuntimeException) targetException;
      }
      else if (targetException instanceof Error) {
         throw (Error) targetException;
      }
      else if (targetException instanceof Exception) {
         throw (Exception) targetException;
      }
      else {
         String text = getInvocationErrorMessage("Failed to invoke handler method", args);
         throw new IllegalStateException(text, targetException);
      }
   }
}
```

#### 处理返回结果

org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite#handleReturnValue

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
   //获取HandlerMethodReturnValueHandler
   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
    //处理返回结果
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite#selectHandler

```java
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
  //判断是否异步
   boolean isAsyncValue = isAsyncReturnValue(value, returnType);
   for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
       //如果是异步但是hanler不是AsyncHandlerMethodReturnValueHandler类型，过滤
      if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
         continue;
      }
      //查找支持此returnType的HandlerMethodReturnValueHandler
      if (handler.supportsReturnType(returnType)) { 
         return handler;
      }
   }
   return null;
}
```

### 后置拦截

org.springframework.web.servlet.HandlerExecutionChain#applyPostHandle

```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
      throws Exception {

   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
      for (int i = interceptors.length - 1; i >= 0; i--) {
         HandlerInterceptor interceptor = interceptors[i];
         interceptor.postHandle(request, response, this.handler, mv);
      }
   }
}
```

org.springframework.web.servlet.DispatcherServlet#processDispatchResult

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
      @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {

   boolean errorView = false;

   if (exception != null) {
      if (exception instanceof ModelAndViewDefiningException) {
         logger.debug("ModelAndViewDefiningException encountered", exception);
         mv = ((ModelAndViewDefiningException) exception).getModelAndView();
      }
      else {
         Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
         mv = processHandlerException(request, response, handler, exception);
         errorView = (mv != null);
      }
   }

   // Did the handler return a view to render?
   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }
   else {
      if (logger.isDebugEnabled()) {
         logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
               "': assuming HandlerAdapter completed request handling");
      }
   }

   if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
      // Concurrent handling started during a forward
      return;
   }

   if (mappedHandler != null) {
      mappedHandler.triggerAfterCompletion(request, response, null);
   }
}
```

org.springframework.web.servlet.DispatcherServlet#render

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
   // Determine locale for request and apply it to the response.
   Locale locale =
         (this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
   response.setLocale(locale);

   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      // We need to resolve the view name.
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
      if (view == null) {
         throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
               "' in servlet with name '" + getServletName() + "'");
      }
   }
   else {
      // No need to lookup: the ModelAndView object contains the actual View object.
      view = mv.getView();
      if (view == null) {
         throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
               "View object in servlet with name '" + getServletName() + "'");
      }
   }

   // Delegate to the View object for rendering.
   if (logger.isDebugEnabled()) {
      logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
   }
   try {
      if (mv.getStatus() != null) {
         response.setStatus(mv.getStatus().value());
      }
      view.render(mv.getModelInternal(), request, response);
   }
   catch (Exception ex) {
      if (logger.isDebugEnabled()) {
         logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
               getServletName() + "'", ex);
      }
      throw ex;
   }
}
```

## 其他

### 直接参数绑定

通过注解进行绑定,@RequestParam

使用注解进行绑定,我们只要在方法参数前面声明@RequestParam("a"),就可以将 Request 中参数 a 的值绑定到方法的该参数上

使用参数名称进行绑定的前提是必须要获取方法中参数的名称,Java反射只提供了获取方法的参数的类型,并没有提供获取参数名称的方法，springmvc借助ASM获取方法的参数名称

org.springframework.core.LocalVariableTableParameterNameDiscoverer#getParameterNames(java.lang.reflect.Method)

```java
public String[] getParameterNames(Method method) {
   Method originalMethod = BridgeMethodResolver.findBridgedMethod(method);
   Class<?> declaringClass = originalMethod.getDeclaringClass();
   Map<Member, String[]> map = this.parameterNamesCache.get(declaringClass);
   if (map == null) {
      map = inspectClass(declaringClass);
      this.parameterNamesCache.put(declaringClass, map);
   }
   if (map != NO_DEBUG_INFO_MAP) {
      return map.get(originalMethod);
   }
   return null;
}
```

org.springframework.core.LocalVariableTableParameterNameDiscoverer#inspectClass

```java
private Map<Member, String[]> inspectClass(Class<?> clazz) {
   InputStream is = clazz.getResourceAsStream(ClassUtils.getClassFileName(clazz));
   if (is == null) {
      // We couldn't load the class file, which is not fatal as it
      // simply means this method of discovering parameter names won't work.
      if (logger.isDebugEnabled()) {
         logger.debug("Cannot find '.class' file for class [" + clazz +
               "] - unable to determine constructor/method parameter names");
      }
      return NO_DEBUG_INFO_MAP;
   }
   try {
      ClassReader classReader = new ClassReader(is);
      Map<Member, String[]> map = new ConcurrentHashMap<Member, String[]>(32);
      classReader.accept(new ParameterNameDiscoveringVisitor(clazz, map), 0);
      return map;
   }
   catch (IOException ex) {
      if (logger.isDebugEnabled()) {
         logger.debug("Exception thrown while reading '.class' file for class [" + clazz +
               "] - unable to determine constructor/method parameter names", ex);
      }
   }
   catch (IllegalArgumentException ex) {
      if (logger.isDebugEnabled()) {
         logger.debug("ASM ClassReader failed to parse class file [" + clazz +
               "], probably due to a new Java class file version that isn't supported yet " +
               "- unable to determine constructor/method parameter names", ex);
      }
   }
   finally {
      try {
         is.close();
      }
      catch (IOException ex) {
         // ignore
      }
   }
   return NO_DEBUG_INFO_MAP;
}
```

LocalVariableTableParameterNameDiscoverer

org.springframework.core.LocalVariableTableParameterNameDiscoverer.ParameterNameDiscoveringVisitor#visitMethod

```java
public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
   // exclude synthetic + bridged && static class initialization
   if (!isSyntheticOrBridged(access) && !STATIC_CLASS_INIT.equals(name)) {
      return new LocalVariableTableVisitor(clazz, memberMap, name, desc, isStatic(access));
   }
   return null;
}
```

LocalVariableTableVisitor

org.springframework.core.LocalVariableTableParameterNameDiscoverer.LocalVariableTableVisitor#LocalVariableTableVisitor

```java
public LocalVariableTableVisitor(Class<?> clazz, Map<Member, String[]> map, String name, String desc, boolean isStatic) {
   super(SpringAsmInfo.ASM_VERSION);
   this.clazz = clazz;
   this.memberMap = map;
   this.name = name;
   this.args = Type.getArgumentTypes(desc);
   this.parameterNames = new String[this.args.length];
   this.isStatic = isStatic;
   this.lvtSlotIndex = computeLvtSlotIndices(isStatic, this.args);
}
```

org.springframework.core.LocalVariableTableParameterNameDiscoverer.LocalVariableTableVisitor#computeLvtSlotIndices

```java
private static int[] computeLvtSlotIndices(boolean isStatic, Type[] paramTypes) {
   int[] lvtIndex = new int[paramTypes.length];//存放参数在本地变量表中的slot
   int nextIndex = (isStatic ? 0 : 1); //如果是实例方法，本地变量表第一个slot存储this，之后的slot开始存方方法的参数；静态方法，从slot0开始存放参数
   for (int i = 0; i < paramTypes.length; i++) {
      lvtIndex[i] = nextIndex;
      if (isWideType(paramTypes[i])) { //long或double占用两个slot
         nextIndex += 2;
      }
      else {
         nextIndex++;
      }
   }
   return lvtIndex;
}
```

org.springframework.core.LocalVariableTableParameterNameDiscoverer.LocalVariableTableVisitor#visitLocalVariable

```java
public void visitLocalVariable(String name, String description, String signature, Label start, Label end, int index) {
   this.hasLvtInfo = true;
   for (int i = 0; i < this.lvtSlotIndex.length; i++) {
      if (this.lvtSlotIndex[i] == index) { //slot相同
         this.parameterNames[i] = name; //设置参数名称
      }
   }
}
```

org.springframework.core.LocalVariableTableParameterNameDiscoverer.LocalVariableTableVisitor#visitEnd

```java
public void visitEnd() {
   if (this.hasLvtInfo || (this.isStatic && this.parameterNames.length == 0)) {
      // visitLocalVariable will never be called for static no args methods
      // which doesn't use any local variables.
      // This means that hasLvtInfo could be false for that kind of methods
      // even if the class has local variable info.
      this.memberMap.put(resolveMember(), this.parameterNames); //method -> parameter[]
   }
}
```

org.springframework.core.LocalVariableTableParameterNameDiscoverer.LocalVariableTableVisitor#resolve

```java
private Member resolveMember() {
   ClassLoader loader = this.clazz.getClassLoader();
   Class<?>[] argTypes = new Class<?>[this.args.length];
   for (int i = 0; i < this.args.length; i++) {
      //根据参数的classname获取对应的Class
      argTypes[i] = ClassUtils.resolveClassName(this.args[i].getClassName(), loader);
   }
   try {
      if (CONSTRUCTOR.equals(this.name)) {
         return this.clazz.getDeclaredConstructor(argTypes);
      }
      return this.clazz.getDeclaredMethod(this.name, argTypes);//根据方法名称和参数类型获取方法
   }
   catch (NoSuchMethodException ex) {
      throw new IllegalStateException("Method [" + this.name +
            "] was discovered in the .class file but cannot be resolved in the class object", ex);
   }
}
```

org.springframework.web.servlet.DispatcherServlet#processDispatchResult

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
    boolean errorView = false;
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            this.logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException)exception).getModelAndView();
        } else {
           //处理异常
            Object handler = mappedHandler != null ? mappedHandler.getHandler() : null;
            mv = this.processHandlerException(request, response, handler, exception);
            errorView = mv != null;
        }
    }

    if (mv != null && !mv.wasCleared()) {
        this.render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    } else if (this.logger.isDebugEnabled()) {
        this.logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + this.getServletName() + "': assuming HandlerAdapter completed request handling");
    }

    if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, (Exception)null);
        }

    }
}
```

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
    ModelAndView exMv = null;
    Iterator var6 = this.handlerExceptionResolvers.iterator();
//可实现HandlerExceptionResolver接口，实现resolveException返回ModelAndView
    while(var6.hasNext()) {
        HandlerExceptionResolver handlerExceptionResolver = (HandlerExceptionResolver)var6.next();
        exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
        if (exMv != null) {
            break;
        }
    }

    if (exMv != null) {
        if (exMv.isEmpty()) {
            request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
            return null;
        } else {
            if (!exMv.hasView()) {
                exMv.setViewName(this.getDefaultViewName(request));
            }

            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
            }

            WebUtils.exposeErrorRequestAttributes(request, ex, this.getServletName());
            return exMv;
        }
    } else {
        throw ex;
    }
}
```



MultipartResolver

org.springframework.web.servlet.DispatcherServlet#checkMultipart

```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
   if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
      if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
         logger.debug("Request is already a MultipartHttpServletRequest - if not in a forward, " +
               "this typically results from an additional MultipartFilter in web.xml");
      }
      else if (hasMultipartException(request) ) {
         logger.debug("Multipart resolution failed for current request before - " +
               "skipping re-resolution for undisturbed error rendering");
      }
      else {
         try {
           	//转换成MultipartHttpServletRequest
            return this.multipartResolver.resolveMultipart(request);
         }
         catch (MultipartException ex) {
            if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
               logger.debug("Multipart resolution failed for error dispatch", ex);
            }
            else {
               throw ex;
            }
         }
      }
   }
   return request;
}
```

org.springframework.web.multipart.support.StandardServletMultipartResolver#resolveMultipart

```java
public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
   return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
}
```

```java
public StandardMultipartHttpServletRequest(HttpServletRequest request, boolean lazyParsing)
      throws MultipartException {

   super(request);
   if (!lazyParsing) { //默认false
      parseRequest(request);
   }
}
```

```java
private void parseRequest(HttpServletRequest request) {
   try {
      Collection<Part> parts = request.getParts();
      this.multipartParameterNames = new LinkedHashSet<String>(parts.size());
      MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<String, MultipartFile>(parts.size());
      for (Part part : parts) {
         String disposition = part.getHeader(CONTENT_DISPOSITION);
         String filename = extractFilename(disposition);
        	//文件名、StandardMultipartFile
         if (filename == null) {
            filename = extractFilenameWithCharset(disposition);
         }
         if (filename != null) {
            files.add(part.getName(), new StandardMultipartFile(part, filename));
         }
         else {
            this.multipartParameterNames.add(part.getName());
         }
      }
     //map存放：文件名->StandardMultipartFile
      setMultipartFiles(files);
   }
   catch (Throwable ex) {
      throw new MultipartException("Could not parse multipart servlet request", ex);
   }
}
```

在上传文件的方法中，如果参数为MultipartHttpServletRequest

org.springframework.web.multipart.support.AbstractMultipartHttpServletRequest#getFile

```java
 public MultipartFile getFile(String name) {
   return getMultipartFiles().getFirst(name);//根据文件名获取MultipartFile
}
```

```java
@Override
public byte[] getBytes() throws IOException {
   return FileCopyUtils.copyToByteArray(this.part.getInputStream());
}

@Override
public InputStream getInputStream() throws IOException {
   return this.part.getInputStream();
}

@Override
public void transferTo(File dest) throws IOException, IllegalStateException {
   this.part.write(dest.getPath());
   if (dest.isAbsolute() && !dest.exists()) {
      FileCopyUtils.copy(this.part.getInputStream(), new FileOutputStream(dest));
   }
}
```

# 异常组件

org.springframework.web.servlet.DispatcherServlet#initHandlerExceptionResolvers

```java
private void initHandlerExceptionResolvers(ApplicationContext context) {
    this.handlerExceptionResolvers = null;
    if (this.detectAllHandlerExceptionResolvers) { //默认为true
      	//先从子容器查找、再从父容器查找
        Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerExceptionResolvers = new ArrayList(matchingBeans.values());
            AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
        }
    } else {
        try {
          	//只从子容器查找
            HandlerExceptionResolver her = (HandlerExceptionResolver)context.getBean("handlerExceptionResolver", HandlerExceptionResolver.class);
            this.handlerExceptionResolvers = Collections.singletonList(her);
        } catch (NoSuchBeanDefinitionException var3) {
        }
    }
  	//从disapatcher.properties中查找
		//AnnotationMethodHandlerExceptionResolver
  	//ResponseStatusExceptionResolver
  	//DefaultHandlerExceptionResolver
    if (this.handlerExceptionResolvers == null) {
        this.handlerExceptionResolvers = this.getDefaultStrategies(context, HandlerExceptionResolver.class);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("No HandlerExceptionResolvers found in servlet '" + this.getServletName() + "': using default");
        }
    }

}
```

org.springframework.web.servlet.DispatcherServlet#processDispatchResult

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
    boolean errorView = false;
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {//ModelAndView is null
            this.logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException)exception).getModelAndView();
        } else { 
            Object handler = mappedHandler != null ? mappedHandler.getHandler() : null;
          	//处理异常获取modelView
            mv = this.processHandlerException(request, response, handler, exception);
            errorView = mv != null;
        }
    }

    if (mv != null && !mv.wasCleared()) { //modelview不为空，且有值
        this.render(mv, request, response);//渲染
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    } else if (this.logger.isDebugEnabled()) { //空的modelview，值为空
        this.logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + this.getServletName() + "': assuming HandlerAdapter completed request handling");
    }

    if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        if (mappedHandler != null) { 
          	//执行拦截器的afterCompletion方法
            mappedHandler.triggerAfterCompletion(request, response, (Exception)null);
        }

    }
}
```

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
    ModelAndView exMv = null;
    Iterator var6 = this.handlerExceptionResolvers.iterator();
		//迭代异常解析器
    while(var6.hasNext()) {
        HandlerExceptionResolver handlerExceptionResolver = (HandlerExceptionResolver)var6.next();
        exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
        if (exMv != null) {
            break;
        }
    }

    if (exMv != null) {
        if (exMv.isEmpty()) { //返回空的modelview
            request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
            return null;
        } else {
            if (!exMv.hasView()) {
                exMv.setViewName(this.getDefaultViewName(request));
            }

            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
            }

            WebUtils.exposeErrorRequestAttributes(request, ex, this.getServletName());
            return exMv;
        }
    } else {
        throw ex;
    }
}
```

org.springframework.web.servlet.handler.AbstractHandlerExceptionResolver#resolveException

```java
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) { //模板方法
    if (this.shouldApplyTo(request, handler)) {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Resolving exception from handler [" + handler + "]: " + ex);
        }

        this.prepareResponse(ex, response);
        ModelAndView result = this.doResolveException(request, response, handler, ex);
        if (result != null) {
            this.logException(ex, request);
        }

        return result;
    } else {
        return null;
    }
}
```

org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver#doResolveException

```java
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
    try {
        if (ex instanceof NoSuchRequestHandlingMethodException) {
            return this.handleNoSuchRequestHandlingMethod((NoSuchRequestHandlingMethodException)ex, request, response, handler);
        }

        if (ex instanceof HttpRequestMethodNotSupportedException) {
            return this.handleHttpRequestMethodNotSupported((HttpRequestMethodNotSupportedException)ex, request, response, handler);
        }

        if (ex instanceof HttpMediaTypeNotSupportedException) {
            return this.handleHttpMediaTypeNotSupported((HttpMediaTypeNotSupportedException)ex, request, response, handler);
        }

        if (ex instanceof HttpMediaTypeNotAcceptableException) {
            return this.handleHttpMediaTypeNotAcceptable((HttpMediaTypeNotAcceptableException)ex, request, response, handler);
        }

        if (ex instanceof MissingPathVariableException) {
            return this.handleMissingPathVariable((MissingPathVariableException)ex, request, response, handler);
        }

        if (ex instanceof MissingServletRequestParameterException) {
            return this.handleMissingServletRequestParameter((MissingServletRequestParameterException)ex, request, response, handler);
        }

        if (ex instanceof ServletRequestBindingException) {
            return this.handleServletRequestBindingException((ServletRequestBindingException)ex, request, response, handler);
        }

        if (ex instanceof ConversionNotSupportedException) {
            return this.handleConversionNotSupported((ConversionNotSupportedException)ex, request, response, handler);
        }

        if (ex instanceof TypeMismatchException) {
            return this.handleTypeMismatch((TypeMismatchException)ex, request, response, handler);
        }

        if (ex instanceof HttpMessageNotReadableException) {
            return this.handleHttpMessageNotReadable((HttpMessageNotReadableException)ex, request, response, handler);
        }

        if (ex instanceof HttpMessageNotWritableException) {
            return this.handleHttpMessageNotWritable((HttpMessageNotWritableException)ex, request, response, handler);
        }

        if (ex instanceof MethodArgumentNotValidException) {
            return this.handleMethodArgumentNotValidException((MethodArgumentNotValidException)ex, request, response, handler);
        }

        if (ex instanceof MissingServletRequestPartException) {
            return this.handleMissingServletRequestPartException((MissingServletRequestPartException)ex, request, response, handler);
        }

        if (ex instanceof BindException) {
            return this.handleBindException((BindException)ex, request, response, handler);
        }

        if (ex instanceof NoHandlerFoundException) {
            return this.handleNoHandlerFoundException((NoHandlerFoundException)ex, request, response, handler);
        }

        if (ex instanceof AsyncRequestTimeoutException) {
            return this.handleAsyncRequestTimeoutException((AsyncRequestTimeoutException)ex, request, response, handler);
        }
    } catch (Exception var6) {
        if (this.logger.isWarnEnabled()) {
            this.logger.warn("Handling of [" + ex.getClass().getName() + "] resulted in exception", var6);
        }
    }

    return null;
}
```

org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver#doResolveException

```java
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
    ResponseStatus status = (ResponseStatus)AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
    if (status != null) {
        try {
            return this.resolveResponseStatus(status, request, response, handler, ex);
        } catch (Exception var7) {
            this.logger.warn("ResponseStatus handling resulted in exception", var7);
        }
    } else if (ex.getCause() instanceof Exception) {
        ex = (Exception)ex.getCause();
        return this.doResolveException(request, response, handler, ex);
    }

    return null;
}
```

org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver#doResolveException

```java
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
    if (handler != null) {
        Method handlerMethod = this.findBestExceptionHandlerMethod(handler, ex);
        if (handlerMethod != null) {
            ServletWebRequest webRequest = new ServletWebRequest(request, response);

            try {
                Object[] args = this.resolveHandlerArguments(handlerMethod, handler, webRequest, ex);
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Invoking request handler method: " + handlerMethod);
                }

                Object retVal = this.doInvokeMethod(handlerMethod, handler, args);
                return this.getModelAndView(handlerMethod, retVal, webRequest);
            } catch (Exception var9) {
                this.logger.error("Invoking request method resulted in exception : " + handlerMethod, var9);
            }
        }
    }

    return null;
}
```



ExceptionHandlerExceptionResolver

实现了接口InitializingBean

```java
public void afterPropertiesSet() {
  //扫描注有ControllerAdvice注解的类
    this.initExceptionHandlerAdviceCache();
    List handlers;
  //参数解析器
    if (this.argumentResolvers == null) {
        handlers = this.getDefaultArgumentResolvers();
        this.argumentResolvers = (new HandlerMethodArgumentResolverComposite()).addResolvers(handlers);
    }
		//返回值解析器
    if (this.returnValueHandlers == null) {
        handlers = this.getDefaultReturnValueHandlers();
        this.returnValueHandlers = (new HandlerMethodReturnValueHandlerComposite()).addHandlers(handlers);
    }

}
```

```java
private void initExceptionHandlerAdviceCache() {
    if (this.getApplicationContext() != null) {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Looking for exception mappings: " + this.getApplicationContext());
        }

        List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(this.getApplicationContext());
        AnnotationAwareOrderComparator.sort(adviceBeans);
        Iterator var2 = adviceBeans.iterator();

        while(var2.hasNext()) {
            ControllerAdviceBean adviceBean = (ControllerAdviceBean)var2.next();
            ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(adviceBean.getBeanType());
            if (resolver.hasExceptionMappings()) {
                this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
                if (this.logger.isInfoEnabled()) {
                    this.logger.info("Detected @ExceptionHandler methods in " + adviceBean);
                }
            }

            if (ResponseBodyAdvice.class.isAssignableFrom(adviceBean.getBeanType())) {
                this.responseBodyAdvice.add(adviceBean);
                if (this.logger.isInfoEnabled()) {
                    this.logger.info("Detected ResponseBodyAdvice implementation in " + adviceBean);
                }
            }
        }

    }
}
```

```java
  //处理异常返回modelview
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {
	//获取ServletInvocableHandlerMethod，用于执行异常的方法
    ServletInvocableHandlerMethod exceptionHandlerMethod = this.getExceptionHandlerMethod(handlerMethod, exception);
    if (exceptionHandlerMethod == null) {
        return null;
    } else {
        exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        	exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();

        try {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Invoking @ExceptionHandler method: " + exceptionHandlerMethod);
            }

         	 //执行方法
            Throwable cause = exception.getCause();
            if (cause != null) {
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, new Object[]{exception, cause, handlerMethod});
            } else {
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, new Object[]{exception, handlerMethod});
            }
        } catch (Throwable var12) {
            if (var12 != exception && this.logger.isWarnEnabled()) {
                this.logger.warn("Failed to invoke @ExceptionHandler method: " + exceptionHandlerMethod, var12);
            }

            return null;
        }

        if (mavContainer.isRequestHandled()) {
            return new ModelAndView();
        } else {
            ModelMap model = mavContainer.getModel();
            HttpStatus status = mavContainer.getStatus();
            ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
            mav.setViewName(mavContainer.getViewName());
            if (!mavContainer.isViewReference()) {
                mav.setView((View)mavContainer.getView());
            }

            if (model instanceof RedirectAttributes) {
                Map<String, ?> flashAttributes = ((RedirectAttributes)model).getFlashAttributes();
                RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
            }

            return mav;
        }
    }
}
```

```java
protected ServletInvocableHandlerMethod getExceptionHandlerMethod(HandlerMethod handlerMethod, Exception exception) {
    Class<?> handlerType = null;
    if (handlerMethod != null) {
        handlerType = handlerMethod.getBeanType();
      	//首先，从exceptionHandlerCache中获取，即从HandlerMethod所属的Bean（controller）中查找ExceptionHandler注解的方法
        ExceptionHandlerMethodResolver resolver = (ExceptionHandlerMethodResolver)this.exceptionHandlerCache.get(handlerType);
        if (resolver == null) {
            resolver = new ExceptionHandlerMethodResolver(handlerType);
            this.exceptionHandlerCache.put(handlerType, resolver);
        }

        Method method = resolver.resolveMethod(exception);
        if (method != null) {
            return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method);
        }

        if (Proxy.isProxyClass(handlerType)) {
            handlerType = AopUtils.getTargetClass(handlerMethod.getBean());
        }
    }
  //再从注有ControllerAdvice注解的bean中查找
    Iterator var9 = this.exceptionHandlerAdviceCache.entrySet().iterator();
    while(var9.hasNext()) {
        Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry = (Entry)var9.next();
        ControllerAdviceBean advice = (ControllerAdviceBean)entry.getKey();
        if (advice.isApplicableToBeanType(handlerType)) {
            ExceptionHandlerMethodResolver resolver = (ExceptionHandlerMethodResolver)entry.getValue();
            Method method = resolver.resolveMethod(exception);
            if (method != null) {
                return new ServletInvocableHandlerMethod(advice.resolveBean(), method);
            }
        }
    }

    return null;
}
```

# 视图解析器

```java
private void initViewResolvers(ApplicationContext context) {
 		 //初始化流程都类似
    this.viewResolvers = null;
    if (this.detectAllViewResolvers) {
        Map<String, ViewResolver> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.viewResolvers = new ArrayList(matchingBeans.values());
            AnnotationAwareOrderComparator.sort(this.viewResolvers);
        }
    } else {
        try {
            ViewResolver vr = (ViewResolver)context.getBean("viewResolver", ViewResolver.class);
            this.viewResolvers = Collections.singletonList(vr);
        } catch (NoSuchBeanDefinitionException var3) {
        }
    }

  	//根据dispatch.properties,从spring容器获取bean
    if (this.viewResolvers == null) {
        this.viewResolvers = this.getDefaultStrategies(context, ViewResolver.class);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("No ViewResolvers found in servlet '" + this.getServletName() + "': using default");
        }
    }

}
```

org.springframework.web.servlet.DispatcherServlet#processDispatchResult

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
    boolean errorView = false;
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            this.logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException)exception).getModelAndView();
        } else {
            Object handler = mappedHandler != null ? mappedHandler.getHandler() : null;
            mv = this.processHandlerException(request, response, handler, exception);
            errorView = mv != null;
        }
    }

  		//视图渲染
    if (mv != null && !mv.wasCleared()) {
        this.render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    } else if (this.logger.isDebugEnabled()) {
        this.logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + this.getServletName() + "': assuming HandlerAdapter completed request handling");
    }

    if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, (Exception)null);
        }

    }
}
```

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    Locale locale = this.localeResolver.resolveLocale(request);
    response.setLocale(locale);
    View view;
    if (mv.isReference()) { //字符串，创建view
        view = this.resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException("Could not resolve view with name '" + mv.getViewName() + "' in servlet with name '" + this.getServletName() + "'");
        }
    } else { //View对象
        view = mv.getView();
        if (view == null) {
            throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a View object in servlet with name '" + this.getServletName() + "'");
        }
    }

    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + this.getServletName() + "'");
    }

    try {
        if (mv.getStatus() != null) {
            response.setStatus(mv.getStatus().value());
        }
				//渲染
        view.render(mv.getModelInternal(), request, response);
    } catch (Exception var7) {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" + this.getServletName() + "'", var7);
        }

        throw var7;
    }
}
```

```java
public View resolveViewName(String viewName, Locale locale) throws Exception {
    if (!this.isCache()) {//禁用缓存
        return this.createView(viewName, locale);
    } else { //启用缓存
        Object cacheKey = this.getCacheKey(viewName, locale);
        View view = (View)this.viewAccessCache.get(cacheKey);
        if (view == null) {
            synchronized(this.viewCreationCache) {
                view = (View)this.viewCreationCache.get(cacheKey);
                if (view == null) {
                    view = this.createView(viewName, locale);
                    if (view == null && this.cacheUnresolved) {
                        view = UNRESOLVED_VIEW;
                    }

                    if (view != null) {
                        this.viewAccessCache.put(cacheKey, view);
                        this.viewCreationCache.put(cacheKey, view);
                        if (this.logger.isTraceEnabled()) {
                            this.logger.trace("Cached view [" + cacheKey + "]");
                        }
                    }
                }
            }
        }

        return view != UNRESOLVED_VIEW ? view : null;
    }
}
```



RequestToViewNameTranslator

DefaultRequestToViewNameTranslator，负责把请求转换成视图名

org.springframework.web.servlet.DispatcherServlet#getDefaultViewName

```java
protected String getDefaultViewName(HttpServletRequest request) throws Exception {
   return this.viewNameTranslator.getViewName(request);
}
```

org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator#getViewName

```java
public String getViewName(HttpServletRequest request) {
   String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
   return (this.prefix + transformPath(lookupPath) + this.suffix);
}
```