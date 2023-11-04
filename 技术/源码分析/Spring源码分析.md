# 循环依赖 

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry

```java
//一级缓存，存放实例化并且属性设置完成的对象
private final Map<String, Object> singletonObjects = new ConcurrentHashMap(256);
```

```java
//二级缓存，存放实例化但是属性尚未设置的对象
private final Map<String, Object> earlySingletonObjects = new HashMap(16);
```

```java
//三级缓存，存放创建对象的ObjectFactory
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap(16);
```

例如：对象A依赖对象B，对象B又依赖对象A（set方法注入）

当A实例化成功后，会执行以下代码

```java
//判断是否提前暴露
boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
if (earlySingletonExposure) {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
    }
		//加入三级缓存
    this.addSingletonFactory(beanName, new ObjectFactory<Object>() {
        public Object getObject() throws BeansException {
            return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
        }
    });
}
```

当B对象进行属性注入时，发现A对象在三级缓存中存在，会执行以下代码

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName); //先从一级缓存获取
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
        //二级缓存
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
           //三级缓存
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
              //创建对象
               singletonObject = singletonFactory.getObject();
              //放入二级缓存
               this.earlySingletonObjects.put(beanName, singletonObject);
              //从三级缓存移除
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
  //如果设置了代理，才会走以下流程
    if (bean != null && !mbd.isSynthetic() &&this.hasInstantiationAwareBeanPostProcessors()) {
        Iterator var5 = this.getBeanPostProcessors().iterator();

        while(var5.hasNext()) {
            BeanPostProcessor bp = (BeanPostProcessor)var5.next();
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor)bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                if (exposedObject == null) {
                    return null;
                }
            }
        }
    }
  //如果设置代理，返回实例化之后的对象，否则返回代理对象
    return exposedObject;
}
```

对象B属性注入完成后，执行以下方法

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton

```java
protected void addSingleton(String beanName, Object singletonObject) {
   synchronized (this.singletonObjects) {
     //放入一级缓存
      this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
      this.singletonFactories.remove(beanName);
      this.earlySingletonObjects.remove(beanName);
      this.registeredSingletons.add(beanName);
   }
}
```

对象A的属性注入完成，执行以下代码

```java
if (earlySingletonExposure) {
   Object earlySingletonReference = getSingleton(beanName, false);
   if (earlySingletonReference != null) {
      //对象A有提前暴露
      if (exposedObject == bean) { 
        	//在暴露之后，populateBean和initializeBean方法都没有对实例化的A进行引用的修改
          //直接返回提前暴露的对象A
          //否则返initializeBean方法返回的对象
         exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
         String[] dependentBeans = getDependentBeans(beanName);
         Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
         for (String dependentBean : dependentBeans) {
            if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
               actualDependentBeans.add(dependentBean);
            }
         }
         if (!actualDependentBeans.isEmpty()) {
            throw new BeanCurrentlyInCreationException(beanName,
                  "Bean with name '" + beanName + "' has been injected into other beans [" +
                  StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                  "] in its raw version as part of a circular reference, but has eventually been " +
                  "wrapped. This means that said other beans do not use the final version of the " +
                  "bean. This is often the result of over-eager type matching - consider using " +
                  "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
         }
      }
   }
```

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
        //返回半成品对象A
         singletonObject = this.earlySingletonObjects.get(beanName);
        //此时allowEarlyReference为false，if中的不会执行
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

如果在没有代理的场景下，只需要两级缓存即可实现循环依赖。

在有代理的场景下，如果还使用二级缓存，在现有流程下是行不通的，二级缓存还是只能存放实例化后的对象，

B对象获取到只是实例化后的对象，不是代理对象，A对象的代理是在属性注入成功之后，执行initializeBean中的

以下代码块获取的，此时B对象中注入的对象A和spring容器中的对象A并不是同一个

三级缓存只是在把对象放入二级缓存之前做些扩展操作，比如设置代理

```java
if (mbd == null || !mbd.isSynthetic()) {
   wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
}
```

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
      result = beanProcessor.postProcessBeforeInitialization(result, beanName);
      if (result == null) {
         return result;
      }
   }
   return result;
}
```

# IOC

org.springframework.context.support.AbstractApplicationContext#refresh

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // 获取BeanFactory;默认实现是DefaultListableBeanFactory
      //加载BeanDefition
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // BeanFactory的预准备工作, context's ClassLoader/ post-processors
      prepareBeanFactory(beanFactory);

      try {
         // BeanFactory准备工作完成后进行的后置处理
         postProcessBeanFactory(beanFactory);

         // 实例化并调用实现了BeanFactoryPostProcessor接口的Bean
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册BeanPostProcessor，在创建bean的前后执行
         registerBeanPostProcessors(beanFactory);

         // 初始化MessageSource组件
         initMessageSource();

         // 初始化事件派发器
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // 注册监听器
         registerListeners();

         // 初始化非懒加载的单例Bean
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

## obtainFreshBeanFactory

org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   refreshBeanFactory();
   return getBeanFactory();
}
```

org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory

```java
protected final void refreshBeanFactory() throws BeansException {
    if (this.hasBeanFactory()) {
        this.destroyBeans();
        this.closeBeanFactory();
    }

    try {
        //创建DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = this.createBeanFactory();
        beanFactory.setSerializationId(this.getId());
        this.customizeBeanFactory(beanFactory);
        //加载BeanDefinition
        this.loadBeanDefinitions(beanFactory);
        synchronized(this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    } catch (IOException var5) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
    }
}
```

### createBeanFactory

org.springframework.context.support.AbstractRefreshableApplicationContext#createBeanFactory

```java
protected DefaultListableBeanFactory createBeanFactory() {
    return new DefaultListableBeanFactory(this.getInternalParentBeanFactory());
}
```

### loadBeanDefinitions

org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    this.initBeanDefinitionReader(beanDefinitionReader);
    this.loadBeanDefinitions(beanDefinitionReader);
}
```

org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.xml.XmlBeanDefinitionReader)

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = this.getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources); //根据Resource
    }

    String[] configLocations = this.getConfigLocations(); //根据目录
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }

}
```

org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String...)

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
   Assert.notNull(locations, "Location array must not be null");
   int count = 0;
   for (String location : locations) {
      count += loadBeanDefinitions(location);
   }
   return count;
}
```

org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String)

```java
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
   return loadBeanDefinitions(location, null);
}
```

org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String, java.util.Set<org.springframework.core.io.Resource>)

```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
   ResourceLoader resourceLoader = getResourceLoader();
   if (resourceLoader == null) {
      throw new BeanDefinitionStoreException(
            "Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
   }

   if (resourceLoader instanceof ResourcePatternResolver) {
      // Resource pattern matching available.
      try {
         Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
         int count = loadBeanDefinitions(resources);
         if (actualResources != null) {
            Collections.addAll(actualResources, resources);
         }
         if (logger.isTraceEnabled()) {
            logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
         }
         return count;
      }
      catch (IOException ex) {
         throw new BeanDefinitionStoreException(
               "Could not resolve bean definition resource pattern [" + location + "]", ex);
      }
   }
   else {
      // Can only load single resources by absolute URL.
      Resource resource = resourceLoader.getResource(location);
      int count = loadBeanDefinitions(resource);
      if (actualResources != null) {
         actualResources.add(resource);
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
      }
      return count;
   }
}
```

org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource)

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
   return loadBeanDefinitions(new EncodedResource(resource));
}
```

org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.support.EncodedResource)

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Loading XML bean definitions from " + encodedResource);
   }

   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
   if (currentResources == null) {
      currentResources = new HashSet<>(4);
      this.resourcesCurrentlyBeingLoaded.set(currentResources);
   }
   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }
   try {
      InputStream inputStream = encodedResource.getResource().getInputStream();
      try {
         InputSource inputSource = new InputSource(inputStream);
         if (encodedResource.getEncoding() != null) {
            inputSource.setEncoding(encodedResource.getEncoding());
         }
         return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
      }
      finally {
         inputStream.close();
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
   }
   finally {
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
         this.resourcesCurrentlyBeingLoaded.remove();
      }
   }
}
```

org.springframework.beans.factory.xml.XmlBeanDefinitionReader#doLoadBeanDefinitions

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
      throws BeanDefinitionStoreException {

   try {
      //读取xml文件，封装成Document
      Document doc = doLoadDocument(inputSource, resource);
     //解析Document，封装成BeanDefinition并注册
      int count = registerBeanDefinitions(doc, resource);
      if (logger.isDebugEnabled()) {
         logger.debug("Loaded " + count + " bean definitions from " + resource);
      }
      return count;
   }
   catch (BeanDefinitionStoreException ex) {
      throw ex;
   }
   catch (SAXParseException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
   }
   catch (SAXException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "XML document from " + resource + " is invalid", ex);
   }
   catch (ParserConfigurationException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Parser configuration exception parsing XML from " + resource, ex);
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "IOException parsing XML document from " + resource, ex);
   }
   catch (Throwable ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Unexpected exception parsing XML document from " + resource, ex);
   }
}
```

org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
   BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
   int countBefore = getRegistry().getBeanDefinitionCount();
   documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#registerBeanDefinitions

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
   this.readerContext = readerContext;
   doRegisterBeanDefinitions(doc.getDocumentElement());
}
```

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions

```java
protected void doRegisterBeanDefinitions(Element root) {
   // Any nested <beans> elements will cause recursion in this method. In
   // order to propagate and preserve <beans> default-* attributes correctly,
   // keep track of the current (parent) delegate, which may be null. Create
   // the new (child) delegate with a reference to the parent for fallback purposes,
   // then ultimately reset this.delegate back to its original (parent) reference.
   // this behavior emulates a stack of delegates without actually necessitating one.
   BeanDefinitionParserDelegate parent = this.delegate;
   this.delegate = createDelegate(getReaderContext(), root, parent);

   if (this.delegate.isDefaultNamespace(root)) {
      String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
      if (StringUtils.hasText(profileSpec)) {
         String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
               profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
         // We cannot use Profiles.of(...) since profile expressions are not supported
         // in XML config. See SPR-12458 for details.
         if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
            if (logger.isDebugEnabled()) {
               logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                     "] not matching: " + getReaderContext().getResource());
            }
            return;
         }
      }
   }

   preProcessXml(root);
   parseBeanDefinitions(root, this.delegate);
   postProcessXml(root);

   this.delegate = parent;
}
```

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
   if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
         Node node = nl.item(i);
         if (node instanceof Element) {
            Element ele = (Element) node;
            if (delegate.isDefaultNamespace(ele)) {
               parseDefaultElement(ele, delegate);
            }
            else {
               delegate.parseCustomElement(ele);
            }
         }
      }
   }
   else {
      delegate.parseCustomElement(root);
   }
}
```

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) { //解析Import
      importBeanDefinitionResource(ele);
   }
   else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) { //解析alias
      processAliasRegistration(ele);
   }
   else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) { //解析bean
      processBeanDefinition(ele, delegate);
   }
   else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) { //解析嵌套bean
      // recurse
      doRegisterBeanDefinitions(ele);
   }
}
```

org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processBeanDefinition

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  //解析bean标签
   BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
   if (bdHolder != null) {
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
         // 注册BeanDefinition
         BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error("Failed to register bean definition with name '" +
               bdHolder.getBeanName() + "'", ele, ex);
      }
      // Send registration event.
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
   }
}
```

## invokeBeanFactoryPostProcessors

### PropertySourcesPlaceholderConfigurer

加载配置文件,实现了EnvironmentAware接口，注入Environment

org.springframework.context.support.PropertySourcesPlaceholderConfigurer#postProcessBeanFactory

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
   if (this.propertySources == null) {
      this.propertySources = new MutablePropertySources();
      if (this.environment != null) {
         this.propertySources.addLast(
            new PropertySource<Environment>(ENVIRONMENT_PROPERTIES_PROPERTY_SOURCE_NAME, this.environment) {
               @Override
               @Nullable
               public String getProperty(String key) {
                  return this.source.getProperty(key);
               }
            }
         );
      }
      try {
         PropertySource<?> localPropertySource =
               new PropertiesPropertySource(LOCAL_PROPERTIES_PROPERTY_SOURCE_NAME, mergeProperties()); //读取配置文件
         if (this.localOverride) {
            this.propertySources.addFirst(localPropertySource);
         }
         else {
            this.propertySources.addLast(localPropertySource);
         }
      }
      catch (IOException ex) {
         throw new BeanInitializationException("Could not load properties", ex);
      }
   }
	//解析配置属性占位符
   processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
   this.appliedPropertySources = this.propertySources;
}
```

#### 读取配置文件

org.springframework.core.io.support.PropertiesLoaderSupport#mergeProperties

```java
protected Properties mergeProperties() throws IOException {
   Properties result = new Properties();

   if (this.localOverride) {
      // Load properties from file upfront, to let local properties override.
      loadProperties(result);
   }

   if (this.localProperties != null) {
      for (Properties localProp : this.localProperties) {
         CollectionUtils.mergePropertiesIntoMap(localProp, result);
      }
   }

   if (!this.localOverride) {
      // Load properties from file afterwards, to let those properties override.
      loadProperties(result);
   }

   return result;
}
```

org.springframework.core.io.support.PropertiesLoaderSupport#loadProperties

```java
protected void loadProperties(Properties props) throws IOException {
   if (this.locations != null) {
      for (Resource location : this.locations) {
         if (logger.isDebugEnabled()) {
            logger.debug("Loading properties file from " + location);
         }
         try {
            PropertiesLoaderUtils.fillProperties(
                  props, new EncodedResource(location, this.fileEncoding), this.propertiesPersister);
         }
         catch (FileNotFoundException | UnknownHostException ex) {
            if (this.ignoreResourceNotFound) {
               if (logger.isInfoEnabled()) {
                  logger.info("Properties resource not found: " + ex.getMessage());
               }
            }
            else {
               throw ex;
            }
         }
      }
   }
}
```

#### 解析属性

遍历BeanDefinition，解析占位符，替换为真正的值

org.springframework.context.support.PropertySourcesPlaceholderConfigurer#processProperties

```java
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
      final ConfigurablePropertyResolver propertyResolver) throws BeansException {
   propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);//${
   propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);//}
   propertyResolver.setValueSeparator(this.valueSeparator);//:
	 //解析器
   StringValueResolver valueResolver = strVal -> {
      String resolved = (this.ignoreUnresolvablePlaceholders ?
            propertyResolver.resolvePlaceholders(strVal) :
            propertyResolver.resolveRequiredPlaceholders(strVal));
      if (this.trimValues) {
         resolved = resolved.trim();
      }
      return (resolved.equals(this.nullValue) ? null : resolved);
   };
	//解析
   doProcessProperties(beanFactoryToProcess, valueResolver);
}
```

org.springframework.beans.factory.config.PlaceholderConfigurerSupport#doProcessProperties

```java
protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
      StringValueResolver valueResolver) {

   BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

   String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
   for (String curName : beanNames) {
      // Check that we're not parsing our own bean definition,
      // to avoid failing on unresolvable placeholders in properties file locations.
      if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
         BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
         try {
            visitor.visitBeanDefinition(bd); //解析BeanDefinition
         }
         catch (Exception ex) {
            throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
         }
      }
   }

   // New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
   beanFactoryToProcess.resolveAliases(valueResolver);

   // New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
   beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#visitBeanDefinition

```java
public void visitBeanDefinition(BeanDefinition beanDefinition) {
   visitParentName(beanDefinition);
   visitBeanClassName(beanDefinition);
   visitFactoryBeanName(beanDefinition);
   visitFactoryMethodName(beanDefinition);
   visitScope(beanDefinition);
   if (beanDefinition.hasPropertyValues()) {
      visitPropertyValues(beanDefinition.getPropertyValues()); //解析属性
   }
   if (beanDefinition.hasConstructorArgumentValues()) {
      ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
      visitIndexedArgumentValues(cas.getIndexedArgumentValues());
      visitGenericArgumentValues(cas.getGenericArgumentValues());
   }
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#visitPropertyValues

```java
protected void visitPropertyValues(MutablePropertyValues pvs) {
   PropertyValue[] pvArray = pvs.getPropertyValues();
   for (PropertyValue pv : pvArray) {
      Object newVal = resolveValue(pv.getValue());
      if (!ObjectUtils.nullSafeEquals(newVal, pv.getValue())) {
         pvs.add(pv.getName(), newVal);
      }
   }
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#resolveValue

```java
protected Object resolveValue(@Nullable Object value) {
   if (value instanceof BeanDefinition) {
      visitBeanDefinition((BeanDefinition) value);
   }
   else if (value instanceof BeanDefinitionHolder) {
      visitBeanDefinition(((BeanDefinitionHolder) value).getBeanDefinition());
   }
   else if (value instanceof RuntimeBeanReference) {
      RuntimeBeanReference ref = (RuntimeBeanReference) value;
      String newBeanName = resolveStringValue(ref.getBeanName());
      if (newBeanName == null) {
         return null;
      }
      if (!newBeanName.equals(ref.getBeanName())) {
         return new RuntimeBeanReference(newBeanName);
      }
   }
   else if (value instanceof RuntimeBeanNameReference) {
      RuntimeBeanNameReference ref = (RuntimeBeanNameReference) value;
      String newBeanName = resolveStringValue(ref.getBeanName());
      if (newBeanName == null) {
         return null;
      }
      if (!newBeanName.equals(ref.getBeanName())) {
         return new RuntimeBeanNameReference(newBeanName);
      }
   }
   else if (value instanceof Object[]) {
      visitArray((Object[]) value);
   }
   else if (value instanceof List) {
      visitList((List) value);
   }
   else if (value instanceof Set) {
      visitSet((Set) value);
   }
   else if (value instanceof Map) {
      visitMap((Map) value);
   }
   else if (value instanceof TypedStringValue) {
      TypedStringValue typedStringValue = (TypedStringValue) value;
      String stringValue = typedStringValue.getValue();
      if (stringValue != null) {
         String visitedString = resolveStringValue(stringValue);
         typedStringValue.setValue(visitedString);
      }
   }
   else if (value instanceof String) {
      return resolveStringValue((String) value);
   }
   return value;
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#resolveStringValue

```java
protected String resolveStringValue(String strVal) {
   if (this.valueResolver == null) {
      throw new IllegalStateException("No StringValueResolver specified - pass a resolver " +
            "object into the constructor or override the 'resolveStringValue' method");
   }
   String resolvedValue = this.valueResolver.resolveStringValue(strVal); //解析获取真正的value
   // Return original String if not modified.
   return (strVal.equals(resolvedValue) ? strVal : resolvedValue);
}
```

## finishBeanFactoryInitialization

org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // Initialize conversion service for this context.
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }

   // Register a default embedded value resolver if no bean post-processor
   // (such as a PropertyPlaceholderConfigurer bean) registered any before:
   // at this point, primarily for resolution in annotation attribute values.
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
   }

   // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // Stop using the temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(null);

   // Allow for caching all bean definition metadata, not expecting further changes.
   beanFactory.freezeConfiguration();

   // Instantiate all remaining (non-lazy-init) singletons.
   beanFactory.preInstantiateSingletons();
}
```

### preInstantiateSingleton

org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons

```java
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }
		//获取注册BeanDefinition的名称
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // 初始化非懒加载的单例Bean
   for (String beanName : beanNames) {
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         if (isFactoryBean(beanName)) { //实现FactoryBean
            //根据&+beanname获取FactoryBean
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName); 
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                              ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
           //实例化当前bean
            getBean(beanName);
         }
      }
   }

   // Trigger post-initialization callback for all applicable beans...
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

#### doGetBean

org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

```java
protected <T> T doGetBean(
      final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
      throws BeansException {

   final String beanName = transformedBeanName(name);
   Object bean;

   // 获取单例bean，有可能不完整（属性尚未设置）
   Object sharedInstance = getSingleton(beanName);
  
   if (sharedInstance != null && args == null) {
      if (logger.isDebugEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) { //正在创建中，获取的是提前暴露的bean
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
      }

      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      try {
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            for (String dep : dependsOn) { //当前bean的实例化依赖其他bean
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               registerDependentBean(dep, beanName);
               try {
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         if (mbd.isSingleton()) { //单例bean
            sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
               @Override
               public Object getObject() throws BeansException {
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  catch (BeansException ex) {
                     destroySingleton(beanName);
                     throw ex;
                  }
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         else {
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                  @Override
                  public Object getObject() throws BeansException {
                     beforePrototypeCreation(beanName);
                     try {
                        return createBean(beanName, mbd, args);
                     }
                     finally {
                        afterPrototypeCreation(beanName);
                     }
                  }
               });
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
      try {
         return getTypeConverter().convertIfNecessary(bean, requiredType);
      }
      catch (TypeMismatchException ex) {
         if (logger.isDebugEnabled()) {
            logger.debug("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```

getSingleton

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String)

```java
public Object getSingleton(String beanName) {
   return getSingleton(beanName, true);
}
```

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName); //获取完整的对象
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) { //正在创建中
      synchronized (this.singletonObjects) {
         singletonObject = this.earlySingletonObjects.get(beanName); //提前暴露的对象
         if (singletonObject == null && allowEarlyReference) { //允许提前暴露
           //在实例化之后尚未填充属性之前，会提前暴露ObjectFactory对象，放入singletonFactories集合
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               //获取对象
               singletonObject = singletonFactory.getObject(); 
               //放入earlySingletonObjects
               this.earlySingletonObjects.put(beanName, singletonObject);
               //从singletonFactories移除
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance

```java
protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

   // beanName以&开头，说明获取FactoryBean
   if (BeanFactoryUtils.isFactoryDereference(name)) {
      if (beanInstance instanceof NullBean) {
         return beanInstance;
      }
      if (!(beanInstance instanceof FactoryBean)) {
         throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
      }
   }

   if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) { //获取正常的bean或者获取FactoryBean，直接返回beanInstance
      return beanInstance;
   }

   Object object = null;
   if (mbd == null) {
      object = getCachedObjectForFactoryBean(beanName);
   }
   if (object == null) {
      // Return bean instance from factory.
      FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
      // Caches object obtained from FactoryBean if it is a singleton.
      if (mbd == null && containsBeanDefinition(beanName)) {
         mbd = getMergedLocalBeanDefinition(beanName);
      }
      boolean synthetic = (mbd != null && mbd.isSynthetic());
      //调用FactoryBean的getObject方法获取bean
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
   }
   return object;
}
```

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
     //创建bean
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // 创建BeanWrapper
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // 是否提前暴露
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      } 
     //提前暴露ObjectFactory
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      //填充属性
      populateBean(beanName, mbd, instanceWrapper);
      //Aware -> postProcessBeforeInitialization ->Init(afterPropertiesSet、自定义init)->postProcessAfterInitialization
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }

   if (earlySingletonExposure) {
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // 实现DisposableBean、或者自定义destroyMethod
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```

# AOP

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      result = processor.postProcessAfterInitialization(result, beanName);
      if (result == null) {
         return result;
      }
   }
   return result;
}
```

org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
    if (bean != null) {
        //检测是否已经创建过代理对象，避免重复创建
        Object cacheKey = this.getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return this.wrapIfNecessary(bean, beanName, cacheKey);
        }
    }

    return bean;
}
```

org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    } else if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    } else if (!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) {
      //获取所有的Advisor，过滤出适用此bean的Advisor（前置通知、后置通知等等）
        Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
        if (specificInterceptors != DO_NOT_PROXY) { //Advisor不为空
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
          //创建代理对象
            Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        } else { //Advisor为空，不需要创建代理
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
    } else {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
}
```

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean

```java
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    List<Advisor> advisors = this.findEligibleAdvisors(beanClass, beanName);
    return advisors.isEmpty() ? DO_NOT_PROXY : advisors.toArray();
}
```

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findEligibleAdvisors

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    List<Advisor> candidateAdvisors = this.findCandidateAdvisors(); //获取所有的Advisor
    List<Advisor> eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);//获取对beanClass适用的Advisor
    this.extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = this.sortAdvisors(eligibleAdvisors); //排序
    }

    return eligibleAdvisors;
}
```

获取Advisor

org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

```java
protected List<Advisor> findCandidateAdvisors() {
    List<Advisor> advisors = super.findCandidateAdvisors(); //获取实现Advisor的Bean
    if (this.aspectJAdvisorsBuilder != null) {
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors()); //解析Aspect注解的类，获取Advisor
    }

    return advisors;
}
```

获取实现Advisor的Bean

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findCandidateAdvisors

```java
protected List<Advisor> findCandidateAdvisors() {
    Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
    return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

org.springframework.aop.framework.autoproxy.BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans

```java
public List<Advisor> findAdvisorBeans() {
    String[] advisorNames = this.cachedAdvisorBeanNames;
    if (advisorNames == null) {
        advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Advisor.class, true, false);
        this.cachedAdvisorBeanNames = advisorNames;
    }

    if (advisorNames.length == 0) {
        return new ArrayList();
    } else {
        List<Advisor> advisors = new ArrayList();
        String[] var3 = advisorNames;
        int var4 = advisorNames.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String name = var3[var5];
            if (this.isEligibleBean(name)) {
                if (this.beanFactory.isCurrentlyInCreation(name)) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Skipping currently created advisor '" + name + "'");
                    }
                } else {
                    try {
                        advisors.add(this.beanFactory.getBean(name, Advisor.class));
                    } catch (BeanCreationException var11) {
                        Throwable rootCause = var11.getMostSpecificCause();
                        if (rootCause instanceof BeanCurrentlyInCreationException) {
                            BeanCreationException bce = (BeanCreationException)rootCause;
                            String bceBeanName = bce.getBeanName();
                            if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                                if (logger.isDebugEnabled()) {
                                    logger.debug("Skipping advisor '" + name + "' with dependency on currently created bean: " + var11.getMessage());
                                }
                                continue;
                            }
                        }

                        throw var11;
                    }
                }
            }
        }

        return advisors;
    }
}
```

解析Aspect注解获取Advisor

org.springframework.aop.aspectj.annotation.BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

```java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;
    if (aspectNames == null) {
        synchronized(this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new ArrayList();
                List<String> aspectNames = new ArrayList();
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false);
                String[] var18 = beanNames;
                int var19 = beanNames.length;

                for(int var7 = 0; var7 < var19; ++var7) {
                    String beanName = var18[var7];
                    if (this.isEligibleBean(beanName)) {
                       //被注有Aspect、字段名称没有以ajc$开头
                        Class<?> beanType = this.beanFactory.getType(beanName);
                        if (beanType != null && this.advisorFactory.isAspect(beanType)) {
                            aspectNames.add(beanName);
                            AspectMetadata amd = new AspectMetadata(beanType, beanName);
                            if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                                MetadataAwareAspectInstanceFactory factory = new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                                List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory); //获取Advisor
                                if (this.beanFactory.isSingleton(beanName)) {
                                    this.advisorsCache.put(beanName, classAdvisors);
                                } else {
                                    this.aspectFactoryCache.put(beanName, factory);
                                }

                                advisors.addAll(classAdvisors);
                            } else {
                                if (this.beanFactory.isSingleton(beanName)) {
                                    throw new IllegalArgumentException("Bean with name '" + beanName + "' is a singleton, but aspect instantiation model is not singleton");
                                }

                                MetadataAwareAspectInstanceFactory factory = new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                                this.aspectFactoryCache.put(beanName, factory);
                                advisors.addAll(this.advisorFactory.getAdvisors(factory));
                            }
                        }
                    }
                }

                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    } else {
        List<Advisor> advisors = new ArrayList();
        Iterator var3 = aspectNames.iterator();

        while(var3.hasNext()) {
            String aspectName = (String)var3.next();
            List<Advisor> cachedAdvisors = (List)this.advisorsCache.get(aspectName);
            if (cachedAdvisors != null) {
                advisors.addAll(cachedAdvisors);
            } else {
                MetadataAwareAspectInstanceFactory factory = (MetadataAwareAspectInstanceFactory)this.aspectFactoryCache.get(aspectName);
                advisors.addAll(this.advisorFactory.getAdvisors(factory));
            }
        }

        return advisors;
    }
}
```

创建Proxy

org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#createProxy

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName, @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory)this.beanFactory, beanName, beanClass);
    }
		//生成代理对象的工厂类
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    if (!proxyFactory.isProxyTargetClass()) {
        if (this.shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        } else {
            this.evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }
		//创建Advisor
    Advisor[] advisors = this.buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    this.customizeProxyFactory(proxyFactory);
    proxyFactory.setFrozen(this.freezeProxy);
    if (this.advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
		//获取代理对象
    return proxyFactory.getProxy(this.getProxyClassLoader());
}
```

org.springframework.aop.framework.ProxyFactory#getProxy(java.lang.ClassLoader)

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return this.createAopProxy().getProxy(classLoader);
}
```

org.springframework.aop.framework.ProxyCreatorSupport#createAopProxy

```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        this.activate();
    }

    return this.getAopProxyFactory().createAopProxy(this);
}
```

DefaultAopProxyFactory

创建代理对象

1、设置 proxyTargetClass=true强制使用Cglib 代理，没有实现接口则也必须用Cglib

2、什么参数都不设置并且对象类实现了接口则默认用JDK代理

org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
        return new JdkDynamicAopProxy(config);
    } else {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
        } else {
            return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
        }
    }
}
```

org.springframework.aop.framework.JdkDynamicAopProxy#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    Class var9;
    try {
        //eqauls()方法，具目标对象未实现此方法
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            Boolean var19 = this.equals(args[0]);
            return var19;
        }
				//hashCode()方法，具目标对象未实现此方法
        if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            Integer var18 = this.hashCode();
            return var18;
        }

        if (method.getDeclaringClass() != DecoratingProxy.class) {
            Object retVal;
            if (!this.advised.opaque && method.getDeclaringClass().isInterface() && method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                retVal = AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
                return retVal;
            }

            if (this.advised.exposeProxy) {
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }
						//获得目标对象的类
            target = targetSource.getTarget();
            Class<?> targetClass = target != null ? target.getClass() : null;
            //获取可以应用到此方法上的Advice列表
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
          //如果没有可以应用到此方法的通知，直接反射调用 Method.invoke(target, args) 
            if (chain.isEmpty()) {
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
            } else {
              //创建 MethodInvocation
                MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();
            }
						//方法返回值类型
            Class<?> returnType = method.getReturnType();
            if (retVal != null && retVal == target && returnType != Object.class && returnType.isInstance(proxy) && !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                retVal = proxy;
            } else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
            }

            Object var13 = retVal;
            return var13;
        }

        var9 = AopProxyUtils.ultimateTargetClass(this.advised);
    } finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }

        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }

    }

    return var9;
}
```

获取应用到此方法的通知

org.springframework.aop.framework.AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    AdvisedSupport.MethodCacheKey cacheKey = new AdvisedSupport.MethodCacheKey(method);
    List<Object> cached = (List)this.methodCache.get(cacheKey);
    if (cached == null) {
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }

    return cached;
}
```

org.springframework.aop.framework.DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class<?> targetClass) {
    List<Object> interceptorList = new ArrayList(config.getAdvisors().length);
    Class<?> actualClass = targetClass != null ? targetClass : method.getDeclaringClass();
   //查看是否包含 IntroductionAdvisor
    boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
    //初始化时注册一系列 AdvisorAdapter,用于将 Advisor 转化成 MethodInterceptor
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
    Advisor[] var8 = config.getAdvisors();
    int var9 = var8.length;

    for(int var10 = 0; var10 < var9; ++var10) {
        Advisor advisor = var8[var10];
        if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor)advisor;
             //匹配类
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) { 
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
               	//匹配方法
                if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
                   //将 Advisor 转化成 Interceptor
                    MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                    if (mm.isRuntime()) {
                        MethodInterceptor[] var15 = interceptors;
                        int var16 = interceptors.length;

                        for(int var17 = 0; var17 < var16; ++var17) {
                            MethodInterceptor interceptor = var15[var17];
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    } else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        } else if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor)advisor;
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        } else {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }

    return interceptorList;
}
```

org.springframework.aop.framework.ReflectiveMethodInvocation#proceed

```java
public Object proceed() throws Throwable {
  	//如果 Interceptor 执行完了，则通过反射执行目标方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return this.invokeJoinpoint();
    } else {
        Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
            return dm.methodMatcher.matches(this.method, this.targetClass, this.arguments) ? dm.interceptor.invoke(this) //动态匹配:运行时参数是否满足匹配条件
              : this.proceed(); //动态匹配失败时,略过当前 Intercetpor,调用下一个 Interceptor
        } else {
            //执行当前 Intercetpor
            return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
        }
    }
}
```

# 声明式事务

## @EnableTransactionManagement

在 Spring 的配置类上添加 @EnableTransactionManagement 注解即可开启事务

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({TransactionManagementConfigurationSelector.class})
public @interface EnableTransactionManagement {
    boolean proxyTargetClass() default false;

    AdviceMode mode() default AdviceMode.PROXY;

    int order() default 2147483647;
}
```

## TransactionManagementConfigurationSelector

org.springframework.transaction.annotation.TransactionManagementConfigurationSelector#selectImports

```java
protected String[] selectImports(AdviceMode adviceMode) {
    switch(adviceMode) {
    case PROXY: //默认
        return new String[]{AutoProxyRegistrar.class.getName(), ProxyTransactionManagementConfiguration.class.getName()};
    case ASPECTJ:
        return new String[]{"org.springframework.transaction.aspectj.AspectJTransactionManagementConfiguration"};
    default:
        return null;
    }
}
```

AutoProxyRegistrar

org.springframework.context.annotation.AutoProxyRegistrar#registerBeanDefinitions

```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    boolean candidateFound = false;
    Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
    Iterator var5 = annoTypes.iterator();

    while(var5.hasNext()) {
        String annoType = (String)var5.next();
        AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
        if (candidate != null) {
            Object mode = candidate.get("mode");
            Object proxyTargetClass = candidate.get("proxyTargetClass");
            if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() && Boolean.class == proxyTargetClass.getClass()) {
                candidateFound = true;
                if (mode == AdviceMode.PROXY) {
                    //注册AutoProxyCreator
                    AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
                    if ((Boolean)proxyTargetClass) {
                        AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                        return;
                    }
                }
            }
        }
    }

    if (!candidateFound && this.logger.isWarnEnabled()) {
        String name = this.getClass().getSimpleName();
        this.logger.warn(String.format("%s was imported but no annotations were found having both 'mode' and 'proxyTargetClass' attributes of type AdviceMode and boolean respectively. This means that auto proxy creator registration and configuration may not have occurred as intended, and components may not be proxied as expected. Check to ensure that %s has been @Import'ed on the same class where these annotations are declared; otherwise remove the import of %s altogether.", name, name, name));
    }

}
```

org.springframework.aop.config.AopConfigUtils#registerAutoProxyCreatorIfNecessary(org.springframework.beans.factory.support.BeanDefinitionRegistry, java.lang.Object)

```java
public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, @Nullable Object source) { //注册BeanDefinition
    return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
}
```

## ProxyTransactionManagementConfiguration

BeanFactoryTransactionAttributeSourceAdvisor

```java
@Bean(
    name = {"org.springframework.transaction.config.internalTransactionAdvisor"}
)
@Role(2)
public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
    BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
    advisor.setTransactionAttributeSource(this.transactionAttributeSource());
    advisor.setAdvice(this.transactionInterceptor()); //拦截器
    if (this.enableTx != null) {
        advisor.setOrder((Integer)this.enableTx.getNumber("order"));
    }

    return advisor;
}
```

TransactionAttributeSource

```java
@Bean
@Role(2)
public TransactionAttributeSource transactionAttributeSource() {//解析事务属性
    return new AnnotationTransactionAttributeSource();
}
```

TransactionInterceptor

```java
@Bean
@Role(2)
public TransactionInterceptor transactionInterceptor() {//对标有Transactional注解的方法进行拦截
    TransactionInterceptor interceptor = new TransactionInterceptor();
    interceptor.setTransactionAttributeSource(this.transactionAttributeSource());
    if (this.txManager != null) {
        interceptor.setTransactionManager(this.txManager);
    }

    return interceptor;
}
```



事务执行

org.springframework.transaction.interceptor.TransactionInterceptor#invoke

```java
public Object invoke(MethodInvocation invocation) throws Throwable {
    Class<?> targetClass = invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null;
    Method var10001 = invocation.getMethod();
    invocation.getClass();
    return this.invokeWithinTransaction(var10001, targetClass, invocation::proceed);
}
```

org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass, TransactionAspectSupport.InvocationCallback invocation) throws Throwable {
   //获取事务属性解析器
    TransactionAttributeSource tas = this.getTransactionAttributeSource();
   //获取事务属性
    TransactionAttribute txAttr = tas != null ? tas.getTransactionAttribute(method, targetClass) : null;
  //获取事务管理器
    PlatformTransactionManager tm = this.determineTransactionManager(txAttr);
    String joinpointIdentification = this.methodIdentification(method, targetClass, txAttr);
    Object result;
    if (txAttr != null && tm instanceof CallbackPreferringPlatformTransactionManager) {
        TransactionAspectSupport.ThrowableHolder throwableHolder = new TransactionAspectSupport.ThrowableHolder();

        try {
            result = ((CallbackPreferringPlatformTransactionManager)tm).execute(txAttr, (status) -> {
                TransactionAspectSupport.TransactionInfo txInfo = this.prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);

                Object var9;
                try {
                    Object var8 = invocation.proceedWithInvocation();
                    return var8;
                } catch (Throwable var13) {
                    if (txAttr.rollbackOn(var13)) {
                        if (var13 instanceof RuntimeException) {
                            throw (RuntimeException)var13;
                        }

                        throw new TransactionAspectSupport.ThrowableHolderException(var13);
                    }

                    throwableHolder.throwable = var13;
                    var9 = null;
                } finally {
                    this.cleanupTransactionInfo(txInfo);
                }

                return var9;
            });
            if (throwableHolder.throwable != null) {
                throw throwableHolder.throwable;
            } else {
                return result;
            }
        } catch (TransactionAspectSupport.ThrowableHolderException var19) {
            throw var19.getCause();
        } catch (TransactionSystemException var20) {
            if (throwableHolder.throwable != null) {
                this.logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
                var20.initApplicationException(throwableHolder.throwable);
            }

            throw var20;
        } catch (Throwable var21) {
            if (throwableHolder.throwable != null) {
                this.logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
            }

            throw var21;
        }
    } else {
        TransactionAspectSupport.TransactionInfo txInfo = this.createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
        result = null;

        try {
            result = invocation.proceedWithInvocation();
        } catch (Throwable var17) {
           //事务发生异常，回滚
            this.completeTransactionAfterThrowing(txInfo, var17);
            throw var17;
        } finally {
            this.cleanupTransactionInfo(txInfo);
        }
				//事务执行正常，提交
        this.commitTransactionAfterReturning(txInfo);
        return result;
    }
}
```

创建TransactionInfo

org.springframework.transaction.interceptor.TransactionAspectSupport#createTransactionIfNecessary

```java
protected TransactionAspectSupport.TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm, @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {
    if (txAttr != null && ((TransactionAttribute)txAttr).getName() == null) {
        txAttr = new DelegatingTransactionAttribute((TransactionAttribute)txAttr) {
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            status = tm.getTransaction((TransactionDefinition)txAttr);
        } else if (this.logger.isDebugEnabled()) {
            this.logger.debug("Skipping transactional joinpoint [" + joinpointIdentification + "] because no transaction manager has been configured");
        }
    }

    return this.prepareTransactionInfo(tm, (TransactionAttribute)txAttr, joinpointIdentification, status);
}
```

org.springframework.transaction.support.AbstractPlatformTransactionManager#getTransaction

```java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
    Object transaction = this.doGetTransaction();
    boolean debugEnabled = this.logger.isDebugEnabled();
    if (definition == null) {
        definition = new DefaultTransactionDefinition();
    }

    if (this.isExistingTransaction(transaction)) {//已经存在事务
        return this.handleExistingTransaction((TransactionDefinition)definition, transaction, debugEnabled);
    } else if (((TransactionDefinition)definition).getTimeout() < -1) { //超时时间不合法
        throw new InvalidTimeoutException("Invalid transaction timeout", ((TransactionDefinition)definition).getTimeout());
    } else if (((TransactionDefinition)definition).getPropagationBehavior() == 2) { //传播属性MANDATORY，使用当前的事务，如果当前没有事务，就抛出异常
        throw new IllegalTransactionStateException("No existing transaction found for transaction marked with propagation 'mandatory'");
    } else if (((TransactionDefinition)definition).getPropagationBehavior() != 0 && ((TransactionDefinition)definition).getPropagationBehavior() != 3 && ((TransactionDefinition)definition).getPropagationBehavior() != 6) {
        if (((TransactionDefinition)definition).getIsolationLevel() != -1 && this.logger.isWarnEnabled()) {
            this.logger.warn("Custom isolation level specified but no actual transaction initiated; isolation level will effectively be ignored: " + definition);
        }
			//NEVER:以非事务方式执行，如果当前存在事务，则抛出异常;NOT_SUPPORTED:以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
        boolean newSynchronization = this.getTransactionSynchronization() == 0;
        return this.prepareTransactionStatus((TransactionDefinition)definition, (Object)null, true, newSynchronization, debugEnabled, (Object)null);
    } else {
       //当前事务挂起
        AbstractPlatformTransactionManager.SuspendedResourcesHolder suspendedResources = this.suspend((Object)null);
        if (debugEnabled) {
            this.logger.debug("Creating new transaction with name [" + ((TransactionDefinition)definition).getName() + "]: " + definition);
        }

        try {
            boolean newSynchronization = this.getTransactionSynchronization() != 2;
            DefaultTransactionStatus status = this.newTransactionStatus((TransactionDefinition)definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
          //开始事务，设置事务属性
            this.doBegin(transaction, (TransactionDefinition)definition);
            this.prepareSynchronization(status, (TransactionDefinition)definition);
            return status;
        } catch (Error | RuntimeException var7) {
            this.resume((Object)null, suspendedResources);
            throw var7;
        }
    }
}
```

org.springframework.jdbc.datasource.DataSourceTransactionManager#doGetTransaction

```java
protected Object doGetTransaction() {
    DataSourceTransactionManager.DataSourceTransactionObject txObject = new DataSourceTransactionManager.DataSourceTransactionObject();
    txObject.setSavepointAllowed(this.isNestedTransactionAllowed());//是否允许嵌套事务
    ConnectionHolder conHolder = (ConnectionHolder)TransactionSynchronizationManager.getResource(this.obtainDataSource());
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}
```

```
int PROPAGATION_REQUIRED = 0;
int PROPAGATION_SUPPORTS = 1;
int PROPAGATION_MANDATORY = 2;
int PROPAGATION_REQUIRES_NEW = 3;
int PROPAGATION_NOT_SUPPORTED = 4;
int PROPAGATION_NEVER = 5;
int PROPAGATION_NESTED = 6;
int ISOLATION_DEFAULT = -1;
int ISOLATION_READ_UNCOMMITTED = 1;
int ISOLATION_READ_COMMITTED = 2;
int ISOLATION_REPEATABLE_READ = 4;
int ISOLATION_SERIALIZABLE = 8;
int TIMEOUT_DEFAULT = -1;
```

# SpringMVC

## 加载默认实现

org.springframework.web.servlet.DispatcherServlet

```java
static {
   // Load default strategy implementations from properties file.
   // This is currently strictly internal and not meant to be customized
   // by application developers.
   try {
      ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);//DispatcherServlet.properties
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
     //创建springmvc容器，
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
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());//父容器
   WebApplicationContext wac = null;

   if (this.webApplicationContext != null) { //在构造器中已经注入
      // A context instance was injected at construction time -> use it
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
   if (wac == null) { //创建springmvc容器
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
   String configLocation = getContextConfigLocation(); //springmvc.xml
   if (configLocation != null) {
      wac.setConfigLocation(configLocation);
   }
   configureAndRefreshWebApplicationContext(wac); //初始化springmvc容器
   return wac;
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
   initMultipartResolver(context); //初始化文件上传解析器
   initLocaleResolver(context); //初始化本地语言环境
   initThemeResolver(context); //初始化模板解析器
   initHandlerMappings(context); //初始化HandlerMapping
   initHandlerAdapters(context); //初始化HandlerAdapter
   initHandlerExceptionResolvers(context); //初始化异常拦截器
   initRequestToViewNameTranslator(context); //初始化视图预解析器
   initViewResolvers(context); //初始化视图解析器
   initFlashMapManager(context); //初始化FlagshMap管理器
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

   if (this.handlerMappings == null) { //容器中未获取到，加载DispatcherServlet.properties配置文件中的HandlerMapping,BeanNameUrlHandlerMapping、RequestMappingHandlerMapping
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
  //默认只在子容器（springmvc容器）中获取
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
            // An unresolvable bean type, probably from a lazy bean - let's ignore it.
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

   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);//执行请求
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
      this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);//封装响应信息
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

   Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);//获取参数
   if (logger.isTraceEnabled()) { 
      logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
            "' with arguments " + Arrays.toString(args));
   }
   Object returnValue = doInvoke(args); //执行业务方法
   if (logger.isTraceEnabled()) {
      logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
            "] returned [" + returnValue + "]");
   }
   return returnValue;
}
```

#### 获取方法参数

org.springframework.core.DefaultParameterNameDiscoverer#DefaultParameterNameDiscoverer

```java
public DefaultParameterNameDiscoverer() { //用来获取方法参数的名称
   if (kotlinPresent) {
      addDiscoverer(new KotlinReflectionParameterNameDiscoverer());
   }
   addDiscoverer(new StandardReflectionParameterNameDiscoverer());
   addDiscoverer(new LocalVariableTableParameterNameDiscoverer());
}
```

org.springframework.web.method.support.InvocableHandlerMethod#getMethodArgumentValues

```java
private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {//获取参数值
   MethodParameter[] parameters = getMethodParameters();
   Object[] args = new Object[parameters.length];
   for (int i = 0; i < parameters.length; i++) {
      MethodParameter parameter = parameters[i];
      //获取参数名称
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

借助ASM获取方法的参数名称

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


