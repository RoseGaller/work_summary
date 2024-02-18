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

# 如何使用AOP?

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
   boolean proxyTargetClass() default false;
   boolean exposeProxy() default false;
}
```

```
AspectJAutoProxyRegistrar实现接口ImportBeanDefinitionRegistrar的registerBeanDefinitions方法
，此方法的调用入口在ConfigurationClassPostProcessor中，又实现了BeanFactoryPostProcessor
```

org.springframework.context.annotation.AspectJAutoProxyRegistrar#registerBeanDefinitions

```java
@Override
public void registerBeanDefinitions(
      AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	//向registry注册AnnotationAwareAspectJAutoProxyCreator
   AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

   AnnotationAttributes enableAspectJAutoProxy =
         AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
   if (enableAspectJAutoProxy != null) {
      if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
         AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
      }
      if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
         AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
      }
   }
}
```

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars

```java
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
   registrars.forEach((registrar, metadata) ->
         registrar.registerBeanDefinitions(metadata, this.registry)); //注册
}
```

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
          	//没有提前暴露过，如有必要进行包装
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
    } else if (!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) {//过滤掉不会代理的class
      	//获取所有的Advisor，过滤出适用此bean的Advisor（前置通知、后置通知等等）
        Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
        if (specificInterceptors != DO_NOT_PROXY) { 
          	//Advisor不为空才会被代理
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
          //创建代理对象
            Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        } else { 
          	//Advisor为空，不需要创建代理
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
  	//获取所有的Advisor
    List<Advisor> candidateAdvisors = this.findCandidateAdvisors(); 
  	//获取对beanClass适用的Advisor
    List<Advisor> eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    this.extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = this.sortAdvisors(eligibleAdvisors); //排序
    }

    return eligibleAdvisors;
}
```

org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

```java
protected List<Advisor> findCandidateAdvisors() {
  	//获取实现Advisor的Bean
    List<Advisor> advisors = super.findCandidateAdvisors(); 
    if (this.aspectJAdvisorsBuilder != null) {
       	//解析Aspect注解的类，获取Advisor
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }

    return advisors;
}
```

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findCandidateAdvisors

```java
protected List<Advisor> findCandidateAdvisors() {
  //获取实现Advisor接口的Bean
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
                       //标有@Aspect、字段名称没有以ajc$开头的Class
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

org.springframework.aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory#getAdvisors

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
  //标有@Aspect的类
    Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
  //切面名称
    String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
    this.validate(aspectClass);
    MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory = new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
    List<Advisor> advisors = new ArrayList();
  //获取标有@PointCut的方法
    Iterator var6 = this.getAdvisorMethods(aspectClass).iterator();

    while(var6.hasNext()) {
        Method method = (Method)var6.next();
        Advisor advisor = this.getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
        Advisor instantiationAdvisor = new ReflectiveAspectJAdvisorFactory.SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
        advisors.add(0, instantiationAdvisor);
    }

    Field[] var12 = aspectClass.getDeclaredFields();
    int var13 = var12.length;

    for(int var14 = 0; var14 < var13; ++var14) {
        Field field = var12[var14];
        Advisor advisor = this.getDeclareParentsAdvisor(field);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    return advisors;
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

# 如何使用事务？

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
                    //注册InfrastructureAdvisorAutoProxyCreator到BeanDefinitionRegistry
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

```java
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
        BeanDefinition apcDefinition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
          //比较优先级，在APC_PRIORITY_LIST中越靠后，优先级越高
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }

        return null;
    } else {
        RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
        beanDefinition.setSource(source);
        beanDefinition.getPropertyValues().add("order", -2147483648);
        beanDefinition.setRole(2);
        registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator", beanDefinition);
        return beanDefinition;
    }
}
```

```java
private static int findPriorityForClass(@Nullable String className) {
    for(int i = 0; i < APC_PRIORITY_LIST.size(); ++i) {
        Class<?> clazz = (Class)APC_PRIORITY_LIST.get(i);
        if (clazz.getName().equals(className)) {
            return i;
        }
    }

    throw new IllegalArgumentException("Class name [" + className + "] is not a known auto-proxy creator class");
}

static {
    APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
    APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
    APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
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
   //对拦截器、事务属性解析器的封装
    BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
   //事务属性解析
    advisor.setTransactionAttributeSource(this.transactionAttributeSource());
  	//拦截器，执行具体方法
    advisor.setAdvice(this.transactionInterceptor()); 
    if (this.enableTx != null) {
        advisor.setOrder((Integer)this.enableTx.getNumber("order"));
    }

    return advisor;
}
```

```java
@Bean
@Role(2)
public TransactionAttributeSource transactionAttributeSource() {//解析事务属性
    return new AnnotationTransactionAttributeSource();
}
```

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

org.springframework.transaction.annotation.AbstractTransactionManagementConfiguration#setConfigurers

```java
@Autowired(required = false)
void setConfigurers(Collection<TransactionManagementConfigurer> configurers) {
   if (CollectionUtils.isEmpty(configurers)) {
      return;
   }
   if (configurers.size() > 1) {
      throw new IllegalStateException("Only one TransactionManagementConfigurer may exist");
   }
   TransactionManagementConfigurer configurer = configurers.iterator().next();
   this.txManager = configurer.annotationDrivenTransactionManager();
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
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// 获取事务信息
		TransactionAttributeSource tas = getTransactionAttributeSource();
    //RuleBasedTransactionAttribute
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    //事务管理器
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
        //对异常的处理
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
      //提交事务
			commitTransactionAfterReturning(txInfo); 
			return retVal;
		}
  }
```

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
               "] after exception: " + ex);
      }
     //事务属性不为空，并且设置了回滚的异常跟实际发生的异常相匹配
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
         try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
         }
      }
      else {
         // We don't roll back on this exception.
         // Will still roll back if TransactionStatus.isRollbackOnly() is true.
         try {
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
         }
      }
   }
}
```

```java
public boolean rollbackOn(Throwable ex) {
   if (logger.isTraceEnabled()) {
      logger.trace("Applying rules to determine whether transaction should rollback on " + ex);
   }

   RollbackRuleAttribute winner = null;
   int deepest = Integer.MAX_VALUE;

   if (this.rollbackRules != null) {
      for (RollbackRuleAttribute rule : this.rollbackRules) {
         int depth = rule.getDepth(ex);
         if (depth >= 0 && depth < deepest) {
            deepest = depth;
            winner = rule;
         }
      }
   }

   if (logger.isTraceEnabled()) {
      logger.trace("Winning rollback rule is: " + winner);
   }

   // User superclass behavior (rollback on unchecked) if no rule matches.
   if (winner == null) {
      logger.trace("No relevant rollback rule found: applying default rules");
      return super.rollbackOn(ex);
   }

   return !(winner instanceof NoRollbackRuleAttribute);
}
```

```java
public boolean rollbackOn(Throwable ex) {//默认只有出现了RuntimeException、Error才回滚
   return (ex instanceof RuntimeException || ex instanceof Error);
}
```

```java
public final void rollback(TransactionStatus status) throws TransactionException {
   if (status.isCompleted()) {
      throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
   }

   DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
   processRollback(defStatus, false);
}
```

```java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
   try {
      boolean unexpectedRollback = unexpected;

      try {
         triggerBeforeCompletion(status);

        
         if (status.hasSavepoint()) { //回滚保存点
            if (status.isDebug()) {
               logger.debug("Rolling back transaction to savepoint");
            }
            status.rollbackToHeldSavepoint();
         }
         else if (status.isNewTransaction()) { //回滚事务
            if (status.isDebug()) {
               logger.debug("Initiating transaction rollback");
            }
            doRollback(status);
         }
         else {
            // Participating in larger transaction
            if (status.hasTransaction()) {
               if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                  if (status.isDebug()) {
                     logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                  }
                  doSetRollbackOnly(status);
               }
               else {
                  if (status.isDebug()) {
                     logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                  }
               }
            }
            else {
               logger.debug("Should roll back transaction but cannot - no transaction available");
            }
            // Unexpected rollback only matters here if we're asked to fail early
            if (!isFailEarlyOnGlobalRollbackOnly()) {
               unexpectedRollback = false;
            }
         }
      }
      catch (RuntimeException | Error ex) {
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         throw ex;
      }

      triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

      // Raise UnexpectedRollbackException if we had a global rollback-only marker
      if (unexpectedRollback) {
         throw new UnexpectedRollbackException(
               "Transaction rolled back because it has been marked as rollback-only");
      }
   }
   finally {
      cleanupAfterCompletion(status);
   }
}
```

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



org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass

```java
// Process any @ImportResource annotations
AnnotationAttributes importResource =
      AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
if (importResource != null) {
   String[] resources = importResource.getStringArray("locations");
   Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
   for (String resource : resources) {
      String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
      configClass.addImportedResource(resolvedResource, readerClass);
   }
}
```

```java
loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
```

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromImportedResources

```java
private void loadBeanDefinitionsFromImportedResources(
      Map<String, Class<? extends BeanDefinitionReader>> importedResources) {

   Map<Class<?>, BeanDefinitionReader> readerInstanceCache = new HashMap<>();

   importedResources.forEach((resource, readerClass) -> {
      // Default reader selection necessary?
      if (BeanDefinitionReader.class == readerClass) {
         if (StringUtils.endsWithIgnoreCase(resource, ".groovy")) {
            // When clearly asking for Groovy, that's what they'll get...
            readerClass = GroovyBeanDefinitionReader.class;
         }
         else {
            // Primarily ".xml" files but for any other extension as well
            readerClass = XmlBeanDefinitionReader.class;
         }
      }

      BeanDefinitionReader reader = readerInstanceCache.get(readerClass);
      if (reader == null) {
         try {
            // Instantiate the specified BeanDefinitionReader
            reader = readerClass.getConstructor(BeanDefinitionRegistry.class).newInstance(this.registry);
            // Delegate the current ResourceLoader to it if possible
            if (reader instanceof AbstractBeanDefinitionReader) {
               AbstractBeanDefinitionReader abdr = ((AbstractBeanDefinitionReader) reader);
               abdr.setResourceLoader(this.resourceLoader);
               abdr.setEnvironment(this.environment);
            }
            readerInstanceCache.put(readerClass, reader);
         }
         catch (Throwable ex) {
            throw new IllegalStateException(
                  "Could not instantiate BeanDefinitionReader class [" + readerClass.getName() + "]");
         }
      }

      // TODO SPR-6310: qualify relative path locations as done in AbstractContextLoader.modifyLocations
      reader.loadBeanDefinitions(resource);
   });
}
```

```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    ResourceLoader resourceLoader = this.getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException("Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
    } else {
        int count;
        if (resourceLoader instanceof ResourcePatternResolver) {
            try {
                Resource[] resources = ((ResourcePatternResolver)resourceLoader).getResources(location);
                count = this.loadBeanDefinitions(resources);
                if (actualResources != null) {
                    Collections.addAll(actualResources, resources);
                }

                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
                }

                return count;
            } catch (IOException var6) {
                throw new BeanDefinitionStoreException("Could not resolve bean definition resource pattern [" + location + "]", var6);
            }
        } else {
            Resource resource = resourceLoader.getResource(location);
            count = this.loadBeanDefinitions((Resource)resource);
            if (actualResources != null) {
                actualResources.add(resource);
            }

            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
            }

            return count;
        }
    }
}
```

```java
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
    Iterator var2 = this.protocolResolvers.iterator();

    Resource resource;
    do {
        if (!var2.hasNext()) {
            if (location.startsWith("/")) {//以/开头
                return this.getResourceByPath(location);
            }

            if (location.startsWith("classpath:")) {//classpath开头
                return new ClassPathResource(location.substring("classpath:".length()), this.getClassLoader());
            }

            try {
              //按url加载
                URL url = new URL(location);
                return (Resource)(ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            } catch (MalformedURLException var5) {
              //按路径加载
                return this.getResourceByPath(location);
            }
        }

        ProtocolResolver protocolResolver = (ProtocolResolver)var2.next();
        resource = protocolResolver.resolve(location, this);
    } while(resource == null);

    return resource;
}
```

spring启动时。ConfigurationClassPostProcessor会处理标记了@Configuration的类，具
体到每个配置类的处理是通过ConfigurationClassParser来完成的

# redis如何自动注入？

```java
@Configuration
//可以扫描到RedisOperations,引入spring-boot-starter-data-redis
@ConditionalOnClass({RedisOperations.class})
//注入RedisProperties（EnableConfigurationPropertie搭配ConfigurationProperties）
@EnableConfigurationProperties({RedisProperties.class})
//Jedis or Lettuce
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
    public RedisAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean(
        name = {"redisTemplate"}
    )
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

如何指定序列化方式

org.springframework.data.redis.core.StringRedisTemplate#StringRedisTemplate()

```java
public StringRedisTemplate() {
    this.setKeySerializer(RedisSerializer.string());
    this.setValueSerializer(RedisSerializer.string());
    this.setHashKeySerializer(RedisSerializer.string());
    this.setHashValueSerializer(RedisSerializer.string());
}
```

org.springframework.data.redis.core.RedisTemplate#afterPropertiesSet

```java
public void afterPropertiesSet() {
    super.afterPropertiesSet();
    boolean defaultUsed = false;
    if (this.defaultSerializer == null) {
      //默认JdkSerializationRedisSerializer
        this.defaultSerializer = new JdkSerializationRedisSerializer(this.classLoader != null ? this.classLoader : this.getClass().getClassLoader());
    }

    if (this.enableDefaultSerializer) {
        if (this.keySerializer == null) {
            this.keySerializer = this.defaultSerializer;
            defaultUsed = true;
        }

        if (this.valueSerializer == null) {
            this.valueSerializer = this.defaultSerializer;
            defaultUsed = true;
        }

        if (this.hashKeySerializer == null) {
            this.hashKeySerializer = this.defaultSerializer;
            defaultUsed = true;
        }

        if (this.hashValueSerializer == null) {
            this.hashValueSerializer = this.defaultSerializer;
            defaultUsed = true;
        }
    }

    if (this.enableDefaultSerializer && defaultUsed) {
        Assert.notNull(this.defaultSerializer, "default serializer null and not all serializers initialized");
    }

    if (this.scriptExecutor == null) {
        this.scriptExecutor = new DefaultScriptExecutor(this);
    }

    this.initialized = true;
}
```
