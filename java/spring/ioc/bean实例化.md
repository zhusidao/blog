# ioc源码分析第二篇

**前言**：[上一篇](https://github.com/zhusidao/zhusidao.github.io/wiki/bean%E6%B3%A8%E5%86%8C)介绍完了invokeBeanFactoryPostProcessors()方法，简单总结：该方法就是主要是介绍整个Bean的注册流程，接着refresh()方法接着往下分析，开始介绍bean的**实例化**流程，本篇主要是介绍构造方法进行实例化，本篇是从registerBeanPostProcessors()往下面开始分析：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 这里需要进行同步操作，防止多个线程同时刷新容器
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      // 容器刷新前的准备，设置上下文状态，获取属性，验证必要的属性等
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      // 这里是IOC容器初始化的核心方法，告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      // 对beanFactory进行属性设置
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         // 子类中允许对beanFactory做前置处理（重写该方法）
         // 默认实现没有方法体
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         // 注解模式是使用AnnotationConfigApplicationContext来初始化环境(该方法是解析注册bean的重要入口)
         invokeBeanFactoryPostProcessors(beanFactory);

         // *******下面代码从本篇开始介绍********
         // Register bean processors that intercept bean creation.
         // 注册一些BeanPostProcessor
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context
         // 为上下文初始化Message源,即不同的语言体,国际化处理.
         initMessageSource();

         // Initialize event multicaster for this context.
         // 初始化应用消息广播,并放入"appliactionEventMulitcaster"bean中
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         // 留给子类来初始化其他的Bean，默认是一个空的方法
         onRefresh();

         // Check for listener beans and register them.
         // 检查监听器并注册他们
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         // 实例化所有的还存在的单例bean（非加载）
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

AbstractApplicationContext#registerBeanPostProcessors

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

PostProcessorRegistrationDelegate#registerBeanPostProcessors

```java
/**
 * 在处理实现了BeanPostProcessor的bean之前添加一个BeanPostProcessorChecker
 * 先添加实现了PriorityOrdered的BeanPostProcessor
 * 然后添加ordered的BeanPostProcessor
 * 接着添加都没有实现的BeanPostProcessor
 * 最后添加实现了MergedBeanDefinitionPostProcessor的BeanPostProcessor
 * 处理完获取BeanPostProcessor的bean之后添加一个ApplicationListenerDetector
 */
public static void registerBeanPostProcessors(
      ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
	 // 获取所有的BeanPostProcessor
   String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

   // Register BeanPostProcessorChecker that logs an info message when
   // a bean is created during BeanPostProcessor instantiation, i.e. when
   // a bean is not eligible for getting processed by all BeanPostProcessors.
   int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
   // 添加第一个BeanPostProcessor
   beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

   // Separate between BeanPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  
   for (String ppName : postProcessorNames) {
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
         priorityOrderedPostProcessors.add(pp);
         if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
         }
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, register the BeanPostProcessors that implement PriorityOrdered.
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

   // Next, register the BeanPostProcessors that implement Ordered.
   List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
   for (String ppName : orderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      orderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, orderedPostProcessors);

   // Now, register all regular BeanPostProcessors.
   List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
   for (String ppName : nonOrderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      nonOrderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

   // Finally, re-register all internal BeanPostProcessors.
   sortPostProcessors(internalPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, internalPostProcessors);

   // Re-register post-processor for detecting inner beans as ApplicationListeners,
   // moving it to the end of the processor chain (for picking up proxies etc).
   // 最后添加一个ApplicationListenerDetector
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```



### 初始化国际化主键

AbstractApplicationContext#initMessageSource国际化主键

```java
/**
* Initialize the MessageSource.
* Use parent's if none defined in this context.
*/
protected void initMessageSource() {
   //获取Bean工厂，一般是DefaultListBeanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   //首先判断是否已有xml文件定义了id为messageSource的bean对象
   if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
      //如果有，则从Bean工厂得到这个bean对象
      this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
      // Make MessageSource aware of parent MessageSource.
      //当父类Bean工厂不为空，并且这个bean对象是HierarchicalMessageSource类型
      if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
         //为HierarchicalMessageSource的实现类
         HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
         //设置父类MessageSource，此处设置内部的parent messageSource
         if (hms.getParentMessageSource() == null) {
            // Only set parent context as parent MessageSource if no parent MessageSource
            // registered already.
            hms.setParentMessageSource(getInternalParentMessageSource());
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using MessageSource [" + this.messageSource + "]");
      }
   }
   else {
      // Use empty MessageSource to be able to accept getMessage calls.
      //如果没有xml文件定义信息源对象，新建DelegatingMessageSource类作为messageSource的Bean
      //因为DelegatingMessageSource类实现了HierarchicalMessageSource接口，而这个接口继承了MessageSource这个类
      //因此实现了这个接口的类，都是MessageSource的子类，因此DelegatingMessageSource也是一个MessageSource
      DelegatingMessageSource dms = new DelegatingMessageSource();
      //给这个DelegatingMessageSource添加父类消息源
      dms.setParentMessageSource(getInternalParentMessageSource());
      this.messageSource = dms;
      //将这个messageSource实例注册到Bean工厂中
      beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
      }
   }
}

```



### 执行事件监听

AbstractApplicationContext#initApplicationEventMulticaster

```java
/**
 * 处理ApplicationListener事件
 */
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   // 判断是否存在name为applicationEventMulticaster的bean
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      // 存在就执行初始化bean
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
      // 创建一个SimpleApplicationEventMulticaster并注册为单例bean
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
               "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
      }
   }
}
```

AbstractApplicationContext#registerListeners

```java
protected void registerListeners() {
   // Register statically specified listeners first.
   // 获取ApplicationListeners并添加到applicationEventMulticaster
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
   // 从容器中获取所有实现了ApplicationListener接口的beanName，放入applicationListenerBeans
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }
   // Publish early application events now that we finally have a multicaster...
   // 现在我们终于有了一个广播，可以发布早期的应用程序事件。
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         // 执行onApplicationEvent()
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

SimpleApplicationEventMulticaster#multicastEvent

```java
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
   Executor executor = getTaskExecutor();
   for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      if (executor != null) {
         executor.execute(() -> invokeListener(listener, event));
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```

SimpleApplicationEventMulticaster#invokeListener

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
   ErrorHandler errorHandler = getErrorHandler();
   // 存在errorHandler，如果发生异常，就用errorHandler去处理
   if (errorHandler != null) {
      try {
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         errorHandler.handleError(err);
      }
   }
   else {
      doInvokeListener(listener, event);
   }
}
```

SimpleApplicationEventMulticaster#doInvokeListener

```java
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
   try {
      // 执行onApplicationEvent方法
      listener.onApplicationEvent(event);
   }
   catch (ClassCastException ex) {
      String msg = ex.getMessage();
      if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
         // Possibly a lambda-defined listener which we could not resolve the generic event type for
         // -> let's suppress the exception and just log a debug message.
         Log logger = LogFactory.getLog(getClass());
         if (logger.isTraceEnabled()) {
            logger.trace("Non-matching event type for listener: " + listener, ex);
         }
      }
      else {
         throw ex;
      }
   
```



#### 初始化bean

AbstractApplicationContext#finishBeanFactoryInitialization

```java
/**
 * 完成此上下文的bean工厂的初始化，初始化所有剩余的单例（非懒加载）bean
 */
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
   // 实例化所有剩余的（非懒加载）单例
   beanFactory.preInstantiateSingletons();

```

DefaultListableBeanFactory#preInstantiateSingletons

```java
/**
 * 实例化所有剩余的（非懒加载）单例
 */
@Override
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }

   // Iterate over a copy to allow for init methods which in turn register new bean definitions.
   // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // Trigger initialization of all non-lazy singleton beans...
   for (String beanName : beanNames) {
      // 获取指定名称的Bean定义
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      // Bean不是抽象的，是单态模式的，且lazy-init属性配置为false
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         // FactoryBean是beanName的父类或者是超类
         if (isFactoryBean(beanName)) {
            // beanName + "&"去获取bean
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               // 是否直接进行初始化
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
               // 非懒加载
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
            // 非工厂类直接进行
            getBean(beanName);
         }
      }
   }

   // Trigger post-initialization callback for all applicable beans...
   // 执行所有SmartInitializingSingleton#afterSingletonsInstantiated
   // 这里会有一个EventListenerMethodProcessor去处理@EventListener注解
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         // 如果实现了SmartInitializingSingleton
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            // 执行afterSingletonsInstantiated()
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

AbstractBeanFactory#getMergedLocalBeanDefinition

```java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
   // Quick check on the concurrent map first, with minimal locking.
   RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
   if (mbd != null && !mbd.stale) {
      return mbd;
   }
   return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}
```

AbstractBeanFactory#getMergedLocalBeanDefinition(String, BeanDefinition)

```java
protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd)
      throws BeanDefinitionStoreException {

   return getMergedBeanDefinition(beanName, bd, null);
}
```

AbstractBeanFactory#getMergedLocalBeanDefinition(String, BeanDefinition, BeanDefinition)

```java
protected RootBeanDefinition getMergedBeanDefinition(
      String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
      throws BeanDefinitionStoreException {
	 // 进行同步操作
   synchronized (this.mergedBeanDefinitions) {
      RootBeanDefinition mbd = null;
      RootBeanDefinition previous = null;

      // Check with full lock now in order to enforce the same merged instance.
      if (containingBd == null) {
         mbd = this.mergedBeanDefinitions.get(beanName);
      }

      if (mbd == null || mbd.stale) {
         previous = mbd;
         // 父级bean节点为null
         if (bd.getParentName() == null) {
            // Use copy of given root bean definition.
            if (bd instanceof RootBeanDefinition) {
               mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
            }
            else {
               mbd = new RootBeanDefinition(bd);
            }
         }
         else {
            // Child bean definition: needs to be merged with parent.
            // 父类bean非空
            BeanDefinition pbd;
            try {
               // 去除name的&前缀，把
               String parentBeanName = transformedBeanName(bd.getParentName());
               // bean名称跟父类的bean名称不等
               if (!beanName.equals(parentBeanName)) {
                  // 
                  pbd = getMergedBeanDefinition(parentBeanName);
               }
               else {
                  BeanFactory parent = getParentBeanFactory();
                  if (parent instanceof ConfigurableBeanFactory) {
                     pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
                  }
                  else {
                     throw new NoSuchBeanDefinitionException(parentBeanName,
                           "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
                           "': cannot be resolved without a ConfigurableBeanFactory parent");
                  }
               }
            }
            catch (NoSuchBeanDefinitionException ex) {
               throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
                     "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
            }
            // Deep copy with overridden values.
            mbd = new RootBeanDefinition(pbd);
            mbd.overrideFrom(bd);
         }

         // Set default singleton scope, if not configured before.
         if (!StringUtils.hasLength(mbd.getScope())) {
            mbd.setScope(SCOPE_SINGLETON);
         }

         // A bean contained in a non-singleton bean cannot be a singleton itself.
         // Let's correct this on the fly here, since this might be the result of
         // parent-child merging for the outer bean, in which case the original inner bean
         // definition will not have inherited the merged outer bean's singleton status.
         if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
            mbd.setScope(containingBd.getScope());
         }

         // Cache the merged bean definition for the time being
         // (it might still get re-merged later on in order to pick up metadata changes)
         // 缓存bean定义
         if (containingBd == null && isCacheBeanMetadata()) {
            this.mergedBeanDefinitions.put(beanName, mbd);
         }
      }
      if (previous != null) {
         copyRelevantMergedBeanDefinitionCaches(previous, mbd);
      }
      return mbd;
   }
}
```

AbstractBeanFactory#transformedBeanName

```java
/**
 * 去除name的&前缀，如果是别名就转化为beanName
 */
protected String transformedBeanName(String name) {
   return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}
```

BeanFactoryUtils#transformedBeanName

```java
/**
 * 去除name的&前缀
 */
public static String transformedBeanName(String name) {
   Assert.notNull(name, "'name' must not be null");
   if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
      return name;
   }
   return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
      do {
         beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
      }
      while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
      return beanName;
   });
}
```

SimpleAliasRegistry#canonicalName

```java
/**
 * 别名转化为可解析的bean名称
 */
public String canonicalName(String name) {
   String canonicalName = name;
   // Handle aliasing...
   String resolvedName;
   do {
      resolvedName = this.aliasMap.get(canonicalName);
      if (resolvedName != null) {
         canonicalName = resolvedName;
      }
   }
   while (resolvedName != null);
   return canonicalName;
}
```

AbstractBeanFactory#getBean

```java
@Override
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}
```

AbstractBeanFactory#doGetBean

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	 // 根据指定的名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖,如果指定的是别名，将别名转换为规范的Bean名称
   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
   // 根据beanName尝试从singletonObjects获取Bean
   // 获取不到则再尝试从earlySingletonObjects,singletonFactories 从获取Bean
   // 这段代码和解决循环依赖有关
   // 第一次执行是无法获取的
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      // 判断是否循环依赖
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      // 获取父BeanFactory,一般情况下，父BeanFactory为null，如果存在父BeanFactory，就先去父级容器去查找
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         // 是否为AbstractBeanFactory的实例
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else if (requiredType != null) {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
         else {
            return (T) parentBeanFactory.getBean(nameToLookup);
         }
      }
 			// 创建的Bean是否需要进行类型验证,一般情况下都不需要
      if (!typeCheckOnly) {
         // 标记 bean 已经被创建
         markBeanAsCreated(beanName);
      }

      try {
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         // 获取依赖的bean名称
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            for (String dep : dependsOn) {
               /*
                * @DependsOn({"B"})
                * class A{
                * }
                *
                * @DependsOn({"A"})
                * class B{
                * }
                * 若先初始化A
                * 第一次执行的时候，isDependent会通过，
                * 会继续执行registerDependentBean，然后dependentBeanMap.put("B", Array(A))；
                * 执行到getBean的时候会递归调用去初始化B；
                * 第二次执行到这里的时候isDependent无法通过（dependentBeanMap.get("B")去判断）
                * 就会抛出循环依赖的异常
                */
               // 判断beanName是否依赖于dep
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               // 将依赖关系用map缓存-->用于isDependent做判断
               registerDependentBean(dep, beanName);
               try {
                  // 递归先获取依赖的bean
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // Create bean instance.
         // 创建单例bean
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }
				 // 原型
         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               // 每次都回去创建一个bean
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
               Object scopedInstance = scope.get(beanName, () -> {
                  beforePrototypeCreation(beanName);
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     afterPrototypeCreation(beanName);
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
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```

DefaultSingletonBeanRegistry#isDependent(String, String)

```java
protected boolean isDependent(String beanName, String dependentBeanName) {
   synchronized (this.dependentBeanMap) {
      return isDependent(beanName, dependentBeanName, null);
   }
}
```

DefaultSingletonBeanRegistry#isDependent(String, String, Set<String>)

```java
private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
   if (alreadySeen != null && alreadySeen.contains(beanName)) {
      return false;
   }
   // 把别名转化为bean名称
   String canonicalName = canonicalName(beanName);
   // 获取依赖关系
   Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
   if (dependentBeans == null) {
      return false;
   }
   // 判断依赖关系是否存在
   if (dependentBeans.contains(dependentBeanName)) {
      return true;
   }
   for (String transitiveDependency : dependentBeans) {
      if (alreadySeen == null) {
         alreadySeen = new HashSet<>();
      }
      alreadySeen.add(beanName);
      // 进行递归判断
      if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
         return true;
      }
   }
   return false;
}
```

DefaultSingletonBeanRegistry#getSingleton

```java
/**
 * 返回以给定名称注册的（原始）单例对象。
 * 检查已经实例化的单例，并且还允许早期引用当前创建的单例（解析循环引用）。
 */
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   // singletonObjects能获取到单例就直接返回
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         // 早期的单例对象
         singletonObject = this.earlySingletonObjects.get(beanName);
         // 允许提前曝光
         if (singletonObject == null && allowEarlyReference) {
            // 三级缓存中获取对象
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               // 从singletonFactory获取早期的单例对象
               singletonObject = singletonFactory.getObject();
               // 放入二级缓存
               this.earlySingletonObjects.put(beanName, singletonObject);
               // 移除三级缓存
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```



##### 获取单例bean

DefaultSingletonBeanRegistry#getSingleton

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(beanName, "Bean name must not be null");
   synchronized (this.singletonObjects) {
      // 单例map中获取
      Object singletonObject = this.singletonObjects.get(beanName);
      if (singletonObject == null) {
         // 如果正在被销毁
         if (this.singletonsCurrentlyInDestruction) {
            throw new BeanCreationNotAllowedException(beanName,
                  "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                  "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
         }
         // 创建之前的状态校验
         beforeSingletonCreation(beanName);
         boolean newSingleton = false;
         boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
         if (recordSuppressedExceptions) {
            this.suppressedExceptions = new LinkedHashSet<>();
         }
         try {
            // 获取单例对象
            singletonObject = singletonFactory.getObject();
            newSingleton = true;
         }
         catch (IllegalStateException ex) {
            // Has the singleton object implicitly appeared in the meantime ->
            // if yes, proceed with it since the exception indicates that state.
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
               throw ex;
            }
         }
         catch (BeanCreationException ex) {
            if (recordSuppressedExceptions) {
               for (Exception suppressedException : this.suppressedExceptions) {
                  ex.addRelatedCause(suppressedException);
               }
            }
            throw ex;
         }
         finally {
            if (recordSuppressedExceptions) {
               this.suppressedExceptions = null;
            }
            // 创建之前的状态校验
            afterSingletonCreation(beanName);
         }
         if (newSingleton) {
            // 添加到缓存 建立一级缓存
            addSingleton(beanName, singletonObject);
         }
      }
      return singletonObject;
   }
}
```

DefaultSingletonBeanRegistry#addSingleton

```java
protected void addSingleton(String beanName, Object singletonObject) {
   synchronized (this.singletonObjects) {
      // 添加单粒对象到缓存
      this.singletonObjects.put(beanName, singletonObject);
      // 移除singletonFactories和earlySingletonObjects缓存
      this.singletonFactories.remove(beanName);
      this.earlySingletonObjects.remove(beanName);
      // 添加注册单例对象
      this.registeredSingletons.add(beanName);
   }
}
```



#### 创建bean

AbstractAutowireCapableBeanFactory#createBean

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   // 如果resolvedClass存在，并且mdb的beanClass类型不是Class，并且mdb的beanClass不为空（则代表beanClass存的是Class的name）,
   // 则使用mdb深拷贝一个新的RootBeanDefinition副本，并且将解析的Class赋值给拷贝的RootBeanDefinition副本的beanClass属性，
   // 该拷贝副本取代mdb用于后续的操作
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   // 准备方法重写
   try {
      // 2.验证及准备覆盖的方法（对override属性进行标记及验证）
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance. 
      // 3.实例化前的处理，给InstantiationAwareBeanPostProcessor一个机会返回代理对象来替代真正的bean实例，达到“短路”效果
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      // 4.如果bean不为空，则会跳过Spring默认的实例化过程，直接使用返回的bean
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      // 开始创建bean
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

AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        // 1.mbd不是合成的，并且BeanFactory中存在InstantiationAwareBeanPostProcessor
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            // 2.解析beanName对应的Bean实例的类型
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 3.实例化前的后置处理器应用（处理InstantiationAwareBeanPostProcessor）
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 4.如果返回的bean不为空，会跳过Spring默认的实例化过程，
                    // 所以只能在这里调用BeanPostProcessor实现类的postProcessAfterInitialization方法
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        // 5.如果bean不为空，则将beforeInstantiationResolved赋值为true，代表在实例化之前已经解析
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```

AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation

```java
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    // 遍历当前BeanFactory中的BeanPostProcessor
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // 应用InstantiationAwareBeanPostProcessor后置处理器，允许postProcessBeforeInstantiation方法返回bean对象的代理
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 执行postProcessBeforeInstantiation方法，在Bean实例化前操作，
            // 该方法可以返回一个构造方法完成的Bean实例，从而不会继续执行创建Bean实例的“正规的流程”，达到“短路”的效果。
            // CommonAnnotationBeanPostProcessor不做处理返回为null
            // InstantiationAwareBeanPostProcessor不做处理返回为null
            // InstantiationAwareBeanPostProcessorAdapter不做处理返回为null
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                // 如果result不为空，也就是有后置处理器返回了bean实例对象，则会跳过Spring默认的实例化过程
                return result;
            }
        }
    }
    return null;

```

> 实例化前的处理，给 InstantiationAwareBeanPostProcessor 一个机会返回代理对象来替代真正的 bean 实例，从而跳过 Spring 默认的实例化过程，达到“短路”效果
>
> 如果我们想自己去进行实例化bean，可以进行实现InstantiationAwareBeanPostProcessor去重写postProcessBeforeInstantiation方法。

AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization

```java
	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;	
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

AbstractAutowireCapableBeanFactory#doCreateBean

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {
   // Instantiate the bean.
   // 创建一个bean的包装对象
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      // 从factoryBeanInstanceCache获取instanceWrapper并进行移除
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      // 创建bean实例
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   // 获取创建好的Bean实例
   final Object bean = instanceWrapper.getWrappedInstance();
   // 获取bean类型
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            // 应用后置处理器MergedBeanDefinitionPostProcessor，允许修改MergedBeanDefinition，
            // @Autowired、@Value、@Resource注解、@PreDestroy、@PostConstruct都是在这里预解析，建立缓存
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   // 判断是否需要提早曝光实例：单例 && 允许循环依赖 && 当前bean正在创建中
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      // 提前曝光beanName的ObjectFactory，用于解决循环引用，建立三级缓存，移除二级缓存
      // 应用后置处理器SmartInstantiationAwareBeanPostProcessor，允许返回指定bean的早期引用，若没有则直接返回bean
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      // xml属性赋值，依赖注入
      populateBean(beanName, mbd, instanceWrapper);
      // bean初始化
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
	 // 允许提早曝光
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
                     "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable.
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



#### **bean实例化**

AbstractAutowireCapableBeanFactory#createBeanInstance

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // Make sure bean class is actually resolved at this point.
   Class<?> beanClass = resolveBeanClass(mbd, beanName);

   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }

   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }
	 // 通过工厂方法创建
   // @Bean方式定义的bean算工厂方法创建
   if (mbd.getFactoryMethodName() != null) {
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // Shortcut when re-creating the same bean...
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) 
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            // 已经解析过了构造方法或工厂方法
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   if (resolved) {
      // 判断构造方法参数是否已经解析
      if (autowireNecessary) {
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         return instantiateBean(beanName, mbd);
      }
   }

   // Candidate constructors for autowiring?
   // 获取构造方法
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // Preferred constructors for default construction?
   // Kotlin?
   ctors = mbd.getPreferredConstructors();
   if (ctors != null) {
      return autowireConstructor(beanName, mbd, ctors, null);
   }

   // No special handling: simply use no-arg constructor.
   // 没有特殊处理，直接使用无参构造方法
   return instantiateBean(beanName, mbd);
}
```

> 这里只做构造方法初始化进行分析
>

#### 获取构造方法

AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors

```java
@Override
@Nullable
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
      throws BeanCreationException {

   // Let's check for lookup methods here...
   /*
    * 遍历beanClass的所有方法，若方法上有Lookup，则创建一个LookupOverride去记录方法名称、Lookup的value属性，
    * 然后记录到RootBeanDefinition的methodOverrides属性中
    */
   if (!this.lookupMethodsChecked.contains(beanName)) {
      if (AnnotationUtils.isCandidateClass(beanClass, Lookup.class)) {
         try {
            Class<?> targetClass = beanClass;
            do {
               ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                  Lookup lookup = method.getAnnotation(Lookup.class);
                  if (lookup != null) {
                     Assert.state(this.beanFactory != null, "No BeanFactory available");
                     LookupOverride override = new LookupOverride(method, lookup.value());
                     try {
                        RootBeanDefinition mbd = (RootBeanDefinition)
                              this.beanFactory.getMergedBeanDefinition(beanName);
                        mbd.getMethodOverrides().addOverride(override);
                     }
                     catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(beanName,
                              "Cannot apply @Lookup to beans without corresponding bean definition");
                     }
                  }
               });
               targetClass = targetClass.getSuperclass();
            }
            while (targetClass != null && targetClass != Object.class);

         }
         catch (IllegalStateException ex) {
            throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
         }
      }
      this.lookupMethodsChecked.add(beanName);
   }

   // Quick check on the concurrent map first, with minimal locking.
   // 后选择缓存
   Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
   // 构造方法为空
   if (candidateConstructors == null) {
      // Fully synchronized resolution now...
      // 加锁同步
      synchronized (this.candidateConstructorsCache) {
         candidateConstructors = this.candidateConstructorsCache.get(beanClass);
         // 加锁后的二次检查是否为空
         if (candidateConstructors == null) {
            // 
            Constructor<?>[] rawCandidates;
            try {
               // 获取该类的构造方法
               rawCandidates = beanClass.getDeclaredConstructors();
            }
            catch (Throwable ex) {
               throw new BeanCreationException(beanName,
                     "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                     "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
            }
            List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
            Constructor<?> requiredConstructor = null;
            Constructor<?> defaultConstructor = null;
            // 这个貌似是 Kotlin 上用的, 不用管它
            Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
            int nonSyntheticConstructors = 0;
            // 构造方法遍历
            for (Constructor<?> candidate : rawCandidates) {
               if (!candidate.isSynthetic()) {
                  nonSyntheticConstructors++;
               }
               else if (primaryConstructor != null) {
                  continue;
               }
               //  找到Autowired的注解
               MergedAnnotation<?> ann = findAutowiredAnnotation(candidate);
               if (ann == null) {
                  Class<?> userClass = ClassUtils.getUserClass(beanClass);
                  if (userClass != beanClass) {
                     try {
                        Constructor<?> superCtor =
                              userClass.getDeclaredConstructor(candidate.getParameterTypes());
                        ann = findAutowiredAnnotation(superCtor);
                     }
                     catch (NoSuchMethodException ex) {
                        // Simply proceed, no equivalent superclass constructor found...
                     }
                  }
               }
               // 注解不为空
               if (ann != null) {
                  if (requiredConstructor != null) {
                     throw new BeanCreationException(beanName,
                           "Invalid autowire-marked constructor: " + candidate +
                           ". Found constructor with 'required' Autowired annotation already: " +
                           requiredConstructor);
                  }
                  // 获取Autowired注解上的required属性值
                  boolean required = determineRequiredStatus(ann);
                  if (required) {
                     if (!candidates.isEmpty()) {
                        // 存在多个被Autowired修饰的构造方法，且其中一个的required=true
                        throw new BeanCreationException(beanName,
                              "Invalid autowire-marked constructors: " + candidates +
                              ". Found constructor with 'required' Autowired annotation: " +
                              candidate);
                     }
                     requiredConstructor = candidate;
                  }
                  candidates.add(candidate);
               }
               // 方法参数为空，标示默认构造方法
               else if (candidate.getParameterCount() == 0) {
                  defaultConstructor = candidate;
               }
            }
             }
       			//到这里, 已经循环完了所有的构造方法
        		//候选者不为空时
            if (!candidates.isEmpty()) {
               // Add default constructor to list of optional constructors, as fallback.
               if (requiredConstructor == null) {
                  if (defaultConstructor != null) {
                     candidates.add(defaultConstructor);
                  }
                  else if (candidates.size() == 1 && logger.isInfoEnabled()) {
                     logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
                           "': single autowire-marked constructor flagged as optional - " +
                           "this constructor is effectively required since there is no " +
                           "default constructor to fall back to: " + candidates.get(0));
                  }
               }
               candidateConstructors = candidates.toArray(new Constructor<?>[0]);
            }
            // 类的构造方法只有1个, 且该构造方法有多个参数
            else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
               candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
            }
            // 这里不会进, 因为 primaryConstructor = null
            else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
                  defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
               candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
            }
            // 这里也不会进, 因为 primaryConstructor = null
            else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
               candidateConstructors = new Constructor<?>[] {primaryConstructor};
            }
            else {
               // 如果方法进了这里, 就是没找到合适的构造方法
               // 1. 类定义了多个构造方法, 且没有 @Autowired , 则有可能会进这里
               candidateConstructors = new Constructor<?>[0];
            }
            this.candidateConstructorsCache.put(beanClass, candidateConstructors);
         }
      }
   }
   return (candidateConstructors.length > 0 ? candidateConstructors : null);
}
```

> 总结下构造方法返回
>
> 只有一个无参构造方法, 则返回null
>
> 只有一个有参构造方法, 则返回这个构造方法
>
> 有多个构造方法（不含无参构造方法），且在其中一个方法上标注了 @Autowired , 则会返回这个标注的构造方法
>
> 有多个构造方法，且在多个方法上全部标注了@Autowired（required=false），就会返回这些构造方法
>
> 有多个构造方法，且在多个方法上全部标注了@Autowired（required=false），且无参构造方法没有标注 @Autowired，也会被添加到集合并返回，并都进行返回
>
> 有多个构造方法，且在多个方法上标注了@Autowired（其中一个required=true），就会抛出异常
>
> 有多个构造方法，都没有@Autowired，就会返回null



#### 构造方法进行初始化

带有参数的构造方法进行初始化ConstructorResolver#autowireConstructor

```java
public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
      @Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

   // 定义bean包装类
   BeanWrapperImpl bw = new BeanWrapperImpl();
   this.beanFactory.initBeanWrapper(bw);

   // 最终用于实例化的构造函数
   Constructor<?> constructorToUse = null;
   // 最终用于实例化的构造函数
   ArgumentsHolder argsHolderToUse = null;
   // 最终用于实例化的构造函数参数
   Object[] argsToUse = null;
	 // 1.解析出要用于实例化的构造函数参数
   if (explicitArgs != null) {
      // 1.1 如果explicitArgs不为空，则构造函数的参数直接使用explicitArgs
      // 通过getBean方法调用时，显示指定了参数，则explicitArgs就不为null
      argsToUse = explicitArgs;
   }
   else {
      // 1.2 尝试从缓存中获取已经解析过的构造方法参数
      Object[] argsToResolve = null;
      synchronized (mbd.constructorArgumentLock) {
         // 1.2.1 拿到缓存中已解析的构造方法或工厂方法
         constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
         // 1.2.2 如果constructorToUse不为空 && mbd标记了构造函数参数已解析
         if (constructorToUse != null && mbd.constructorArgumentsResolved) {
            // Found a cached constructor...
            // 1.2.3 从缓存中获取已解析的构造方法参数
            argsToUse = mbd.resolvedConstructorArguments;
            if (argsToUse == null) {
               // 1.2.4 如果resolvedConstructorArguments为空，则从缓存中获取准备用于解析的构造方法参数，
               // constructorArgumentsResolved为true时，resolvedConstructorArguments和
               // preparedConstructorArguments必然有一个缓存了构造方法的参数
               argsToResolve = mbd.preparedConstructorArguments;
            }
         }
      }
      // 1.2.5 如果argsToResolve不为空，则对构造方法参数进行解析
      if (argsToResolve != null) {
         argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
      }
   }
	 // 2.如果构造方法没有被缓存，则通过配置文件获取
   if (constructorToUse == null || argsToUse == null) {
      // Take specified constructors, if any.
      Constructor<?>[] candidates = chosenCtors;
      // 构造方法后选择为null
      if (candidates == null) {
         Class<?> beanClass = mbd.getBeanClass();
         try {
            // 3.2 如果入参chosenCtors为空，则获取beanClass的构造函数
            // （mbd是否允许访问非公共构造函数和方法 ? 所有声明的构造函数：公共构造函数）
            candidates = (mbd.isNonPublicAccessAllowed() ?
                  beanClass.getDeclaredConstructors() : beanClass.getConstructors());
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                  "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
         }
      }
			// 采用指定的构造方法（如果只有一个）且为无参的构造方法
      if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
         Constructor<?> uniqueCandidate = candidates[0];
         if (uniqueCandidate.getParameterCount() == 0) {
            synchronized (mbd.constructorArgumentLock) {
               // 建立缓存
               mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
               mbd.constructorArgumentsResolved = true;
               mbd.resolvedConstructorArguments = EMPTY_ARGS;
            }
            // 进行初始化
            bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
            return bw;
         }
      }

      // Need to resolve the constructor.
      // 2.1 检查是否需要自动装配：chosenCtors不为空 || autowireMode为AUTOWIRE_CONSTRUCTOR
      // 例子：当chosenCtors不为空时，代表有构造函数通过@Autowire修饰，因此需要自动装配
      boolean autowiring = (chosenCtors != null ||
            mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
      ConstructorArgumentValues resolvedValues = null;

      // 构造函数参数个数	
      int minNrOfArgs;
      if (explicitArgs != null) {
         // 2.2 explicitArgs不为空，则使用explicitArgs的length作为minNrOfArgs的值
         minNrOfArgs = explicitArgs.length;
      }
      else {
         // 2.3 获得mbd的构造方法的参数值（indexedArgumentValues：带index的参数值；genericArgumentValues：通用的参数值）
         ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
         // 2.4 创建ConstructorArgumentValues对象resolvedValues，用于承载解析后的构造方法参数的值
         resolvedValues = new ConstructorArgumentValues();
         // 2.5 解析mbd的构造函数的参数，并返回参数个数
         // 注：这边解析mbd中的构造函数参数值，主要是处理我们通过xml方式定义的构造函数注入的参数，
         // 但是如果我们是通过@Autowire注解直接修饰构造函数，则mbd是没有这些参数值的
         minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
      }
			// 3.3 对给定的构造函数排序：先按方法修饰符排序：public排非public前面，再按构造函数参数个数排序：参数多的排前面
      AutowireUtils.sortConstructors(candidates);
      int minTypeDiffWeight = Integer.MAX_VALUE;
      Set<Constructor<?>> ambiguousConstructors = null;
      LinkedList<UnsatisfiedDependencyException> causes = null;

      // 4.遍历所有构造方法候选者，找出符合条件的构造方法
      for (Constructor<?> candidate : candidates) {
         int parameterCount = candidate.getParameterCount();

         if (constructorToUse != null && argsToUse != null && argsToUse.length > parameterCount) {
            // Already found greedy constructor that can be satisfied ->
            // do not look any further, there are only less greedy constructors left.
            // 4.2 如果已经找到满足的构造方法 && 构造参数存在 && 目标构造函数需要的参数个数大于当前遍历的构造函数的参数个数则终止，
            // 因为遍历的构造函数已经排过序，后面不会有更合适的候选者了（参数数目多的先遍历）
            break;
         }
         // 4.3 如果当前遍历到的构造函数的参数个数小于我们所需的参数个数，则直接跳过该构造方法
         if (parameterCount < minNrOfArgs) {
            continue;
         }

         ArgumentsHolder argsHolder;
         // 4.1 拿到当前遍历的构造方法的参数类型数组
         Class<?>[] paramTypes = candidate.getParameterTypes();
         if (resolvedValues != null) {
            try {
               // 4.4 resolvedValues不为空，
               // 4.4.1 获取当前遍历的构造方法的参数名称
               // 4.4.1.1 解析使用ConstructorProperties注解的构造方法参数名称
               String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, parameterCount);
               if (paramNames == null) {
                  // 4.4.1.2 获取参数名称解析器
                  ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                  if (pnd != null) {
                     // 4.4.1.3 使用参数名称解析器获取当前遍历的构造方法的参数名称
                     paramNames = pnd.getParameterNames(candidate);
                  }
               }
               // 4.4.2 创建一个参数数组以调用构造方法或工厂方法，
               // 主要是通过参数类型和参数名解析构造方法或工厂方法所需的参数（如果参数是其他bean，则会解析依赖的bean）
               argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
                     getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
            }
            catch (UnsatisfiedDependencyException ex) {
               // 4.4.3 参数匹配失败，打印异常
               if (logger.isTraceEnabled()) {
                  logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
               }
               // Swallow and try next constructor.
               if (causes == null) {
                  causes = new LinkedList<>();
               }
               causes.add(ex);
               continue;
            }
         }
         else {
            // Explicit arguments given -> arguments length must match exactly.
            // 4.5 resolvedValues为空，则explicitArgs不为空，即给出了显式参数
            // 4.5.1 如果当前遍历的构造函数参数个数与explicitArgs长度不相同，则跳过该构造函
            if (parameterCount != explicitArgs.length) {
               continue;
            }
            // 4.5.2 使用显式给出的参数构造ArgumentsHolder
            argsHolder = new ArgumentsHolder(explicitArgs);
         }

         // 4.6 根据mbd的解析构造函数模式（true: 宽松模式(默认)，false：严格模式），
         // 将argsHolder的参数和paramTypes进行比较，计算paramTypes的类型差异权重值
         int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
               argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
         // Choose this constructor if it represents the closest match.
         // 4.7 类型差异权重值越小,则说明构造函数越匹配，则选择此构造函数
         if (typeDiffWeight < minTypeDiffWeight) {
            // 将要使用的参数都替换成差异权重值更小的
            constructorToUse = candidate;
            argsHolderToUse = argsHolder;
            argsToUse = argsHolder.arguments;
            minTypeDiffWeight = typeDiffWeight;
            // 如果出现权重值更小的候选者，则将ambiguousConstructors清空，允许之前存在权重值相同的候选者
            ambiguousConstructors = null;
         }
         // 4.8 如果存在两个候选者的权重值相同，并且是当前遍历过权重值最小的
         else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
            // 将这两个候选者都添加到ambiguousConstructors
            if (ambiguousConstructors == null) {
               ambiguousConstructors = new LinkedHashSet<>();
               ambiguousConstructors.add(constructorToUse);
            }
            ambiguousConstructors.add(candidate);
         }
      }

      if (constructorToUse == null) {
         // 5.如果最终没有找到匹配的构造函数，则进行异常处理
         if (causes != null) {
            UnsatisfiedDependencyException ex = causes.removeLast();
            for (Exception cause : causes) {
               this.beanFactory.onSuppressedException(cause);
            }
            throw ex;
         }
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Could not resolve matching constructor " +
               "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
      }
      else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
         // 6.如果找到了匹配的构造函数，但是存在多个（ambiguousConstructors不为空） && 解析构造函数的模式为严格模式，则抛出异常
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Ambiguous constructor matches found in bean '" + beanName + "' " +
               "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
               ambiguousConstructors);
      }

      if (explicitArgs == null && argsHolderToUse != null) {
         // 7.将解析的构造函数和参数放到缓存
         argsHolderToUse.storeCache(mbd, constructorToUse);
      }
   }

   Assert.state(argsToUse != null, "Unresolved constructor arguments");
   // 进行bean的实例化
   bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
   return bw;
}
```

ConstructorResolver#resolveConstructorArguments

```java
private int resolveConstructorArguments(String beanName, RootBeanDefinition mbd, BeanWrapper bw,
      ConstructorArgumentValues cargs, ConstructorArgumentValues resolvedValues) {
	 // 构建bean定义值解析器
   TypeConverter customConverter = this.beanFactory.getCustomTypeConverter();
   TypeConverter converter = (customConverter != null ? customConverter : bw);
   BeanDefinitionValueResolver valueResolver =
         new BeanDefinitionValueResolver(this.beanFactory, beanName, mbd, converter);
	 // minNrOfArgs初始化为indexedArgumentValues和genericArgumentValues的的参数个数总和
   int minNrOfArgs = cargs.getArgumentCount();
	 // 遍历解析带index的参数值
   for (Map.Entry<Integer, ConstructorArgumentValues.ValueHolder> entry : cargs.getIndexedArgumentValues().entrySet()) {
      int index = entry.getKey();
      if (index < 0) {
         // index从0开始，不允许小于0
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Invalid constructor argument index: " + index);
      }
      // 如果index大于minNrOfArgs，则修改minNrOfArgs
      if (index + 1 > minNrOfArgs) {
         minNrOfArgs = index + 1;
      }
      ConstructorArgumentValues.ValueHolder valueHolder = entry.getValue();
      // 解析参数值
      if (valueHolder.isConverted()) {
         // 如果参数值已经转换过，则直接将index和valueHolder添加到resolvedValues的indexedArgumentValues属性
         resolvedValues.addIndexedArgumentValue(index, valueHolder);
      }
      else {
         // 如果值还未转换过，则先进行转换
         Object resolvedValue =
               valueResolver.resolveValueIfNecessary("constructor argument", valueHolder.getValue());
         // 使用转换后的resolvedValue构建新的ValueHolder
         ConstructorArgumentValues.ValueHolder resolvedValueHolder =
               new ConstructorArgumentValues.ValueHolder(resolvedValue, valueHolder.getType(), valueHolder.getName());
         // 将转换前的valueHolder保存到新的ValueHolder的source属性
         resolvedValueHolder.setSource(valueHolder);
         // 将index和新的ValueHolder添加到resolvedValues的indexedArgumentValues属性
         resolvedValues.addIndexedArgumentValue(index, resolvedValueHolder);
      }
   }
	 // 遍历解析通用参数值（不带index）
   for (ConstructorArgumentValues.ValueHolder valueHolder : cargs.getGenericArgumentValues()) {
      if (valueHolder.isConverted()) {
         // 如果参数值已经转换过，则直接将valueHolder添加到resolvedValues的genericArgumentValues属性
         resolvedValues.addGenericArgumentValue(valueHolder);
      }
      else { 
         // 如果值还未转换过，则先进行转换
         Object resolvedValue =
               valueResolver.resolveValueIfNecessary("constructor argument", valueHolder.getValue());
         // 使用转换后的resolvedValue构建新的ValueHolder
         ConstructorArgumentValues.ValueHolder resolvedValueHolder = new ConstructorArgumentValues.ValueHolder(
               resolvedValue, valueHolder.getType(), valueHolder.getName());
         // 将转换前的valueHolder保存到新的ValueHolder的source属性
         resolvedValueHolder.setSource(valueHolder);
         // 将新的ValueHolder添加到resolvedValues的genericArgumentValues属性
         resolvedValues.addGenericArgumentValue(resolvedValueHolder);
      }
   }
	 // 返回统计数
   return minNrOfArgs;
}
```

ConstructorResolver#createArgumentArray

```java
private ArgumentsHolder createArgumentArray(
      String beanName, RootBeanDefinition mbd, @Nullable ConstructorArgumentValues resolvedValues,
      BeanWrapper bw, Class<?>[] paramTypes, @Nullable String[] paramNames, Executable executable,
      boolean autowiring, boolean fallback) throws UnsatisfiedDependencyException {

   TypeConverter customConverter = this.beanFactory.getCustomTypeConverter();
   // 获取类型转换器
   TypeConverter converter = (customConverter != null ? customConverter : bw);

   // 新建一个ArgumentsHolder来存放匹配到的参数
   ArgumentsHolder args = new ArgumentsHolder(paramTypes.length);
   Set<ConstructorArgumentValues.ValueHolder> usedValueHolders = new HashSet<>(paramTypes.length);
   Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
   // 遍历参数类型数组
   for (int paramIndex = 0; paramIndex < paramTypes.length; paramIndex++) {
      // 拿到当前遍历的参数类型 
      Class<?> paramType = paramTypes[paramIndex];
      // 拿到当前遍历的参数名
      String paramName = (paramNames != null ? paramNames[paramIndex] : "");
      // Try to find matching constructor argument value, either indexed or generic.
      // 查找当前遍历的参数，是否在mdb对应的bean的构造方法参数中存在index、类型和名称匹配的
      ConstructorArgumentValues.ValueHolder valueHolder = null;
      if (resolvedValues != null) {
         // xml如果已经解析到值，这里是可以直接拿到的
         valueHolder = resolvedValues.getArgumentValue(paramIndex, paramType, paramName, usedValueHolders);
         // If we couldn't find a direct match and are not supposed to autowire,
         // let's try the next generic, untyped argument value as fallback:
         // it could match after type conversion (for example, String -> int). 
         // 3.如果我们找不到直接匹配并且不应该自动装配，那么让我们尝试下一个通用的无类型参数值作为降级方法：它可以在类型转换后匹配（例如，String - > int）。
         if (valueHolder == null && (!autowiring || paramTypes.length == resolvedValues.getArgumentCount())) {
            valueHolder = resolvedValues.getGenericArgumentValue(null, null, usedValueHolders);
         }
      }
      if (valueHolder != null) {
         // We found a potential match - let's give it a try.
         // Do not consider the same value definition multiple times!
         // valueHolder不为空，存在匹配的参数
         // 将valueHolder添加到usedValueHolders
         usedValueHolders.add(valueHolder);
         // 原始属性值
         Object originalValue = valueHolder.getValue();
         // 转换后的属性值
         Object convertedValue;
         if (valueHolder.isConverted()) {
            // 如果valueHolder已经转换过
            // 则直接获取转换后的值
            convertedValue = valueHolder.getConvertedValue();
            // 将convertedValue作为args在paramIndex位置的预备参数
            args.preparedArguments[paramIndex] = convertedValue;
         }
         else {
     				// 将方法（此处为构造方法）和参数索引封装成MethodParameter(MethodParameter是封装方法和参数索引的工具类)
            MethodParameter methodParam = MethodParameter.forExecutable(executable, paramIndex);
            try {
               // 将原始值转换为paramType类型的值（如果类型无法转，抛出TypeMismatchException）
               convertedValue = converter.convertIfNecessary(originalValue, paramType, methodParam);
            }
            catch (TypeMismatchException ex) {
               // 如果类型转换失败，则抛出异常
               throw new UnsatisfiedDependencyException(
                     mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam),
                     "Could not convert argument value of type [" +
                           ObjectUtils.nullSafeClassName(valueHolder.getValue()) +
                           "] to required type [" + paramType.getName() + "]: " + ex.getMessage());
            }
            // 拿到原始的ValueHolder
            Object sourceHolder = valueHolder.getSource();
            if (sourceHolder instanceof ConstructorArgumentValues.ValueHolder) {
               // 拿到原始参数值
               Object sourceValue = ((ConstructorArgumentValues.ValueHolder) sourceHolder).getValue();
               // 标记为已经解析
               args.resolveNecessary = true;
               // 将convertedValue作为args在paramIndex位置的预备参数
               args.preparedArguments[paramIndex] = sourceValue;
            }
         }
         // 将convertedValue作为args在paramIndex位置的参数
         args.arguments[paramIndex] = convertedValue;
         // 将originalValue作为args在paramIndex位置的原始参数
         args.rawArguments[paramIndex] = originalValue;
      }
      else {
         // valueHolder为空，不存在匹配的参数，尝试用自动注入的方法去赋值
         // 将方法（此处为构造函数）和参数索引封装成MethodParameter
         MethodParameter methodParam = MethodParameter.forExecutable(executable, paramIndex);
         // No explicit match found: we're either supposed to autowire or
         // have to fail creating an argument array for the given constructor.
         // 找不到明确的匹配，并且不是自动装配，则抛出异常
         if (!autowiring) {
            throw new UnsatisfiedDependencyException(
                  mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam),
                  "Ambiguous argument values for parameter of type [" + paramType.getName() +
                  "] - did you specify the correct bean references as arguments?");
         }
         try {
            // 如果是自动装配，则调用用于解析自动装配参数的方法，返回的结果为依赖的bean实例对象
            // @value、@Autowire修饰构造方法，自动注入构造方法中的参数bean就是在这边处理
            Object autowiredArgument = resolveAutowiredArgument(
                  methodParam, beanName, autowiredBeanNames, converter, fallback);
            // 将通过自动装配解析出来的参数赋值给args
            args.rawArguments[paramIndex] = autowiredArgument;
            args.arguments[paramIndex] = autowiredArgument;
            args.preparedArguments[paramIndex] = autowiredArgumentMarker;
            args.resolveNecessary = true;
         }
         catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(
                  mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam), ex);
         }
      }
   }
	 // 如果依赖了其他的bean，则注册依赖关系
   for (String autowiredBeanName : autowiredBeanNames) {
      this.beanFactory.registerDependentBean(autowiredBeanName, beanName);
      if (logger.isDebugEnabled()) {
         logger.debug("Autowiring by type from bean name '" + beanName +
               "' via " + (executable instanceof Constructor ? "constructor" : "factory method") +
               " to bean named '" + autowiredBeanName + "'");
      }
   }

   return args;
}
```

> 先看xml对应的属性是否存在解析的值，不存在就去用自动注入的方式去赋值

ConstructorResolver#resolveAutowiredArgument

```java
@Nullable
protected Object resolveAutowiredArgument(MethodParameter param, String beanName,
      @Nullable Set<String> autowiredBeanNames, TypeConverter typeConverter, boolean fallback) {

   Class<?> paramType = param.getParameterType();
   // 如果参数类型为InjectionPoint
   if (InjectionPoint.class.isAssignableFrom(paramType)) {
      // 拿到当前的InjectionPoint（存储了当前正在解析依赖的方法参数信息，DependencyDescriptor）
      InjectionPoint injectionPoint = currentInjectionPoint.get();
      if (injectionPoint == null) {
         throw new IllegalStateException("No current InjectionPoint available for " + param);
      }
      return injectionPoint;
   }
   try {
      // 解析指定依赖，DependencyDescriptor：将MethodParameter的方法参数索引信息封装成DependencyDescriptor
      return this.beanFactory.resolveDependency(
            new DependencyDescriptor(param, true), beanName, autowiredBeanNames, typeConverter);
   }
   catch (NoUniqueBeanDefinitionException ex) {
      throw ex;
   }
   catch (NoSuchBeanDefinitionException ex) {
      if (fallback) {
         // Single constructor or factory method -> let's return an empty array/collection
         // for e.g. a vararg or a non-null List/Set/Map parameter.
         if (paramType.isArray()) {
            return Array.newInstance(paramType.getComponentType(), 0);
         }
         else if (CollectionFactory.isApproximableCollectionType(paramType)) {
            return CollectionFactory.createCollection(paramType, 0);
         }
         else if (CollectionFactory.isApproximableMapType(paramType)) {
            return CollectionFactory.createMap(paramType, 0);
         }
      }
      throw ex;
   }
}
```



#### 自动注入依赖解析

DefaultListableBeanFactory#resolveDependency

```java
@Override
@Nullable
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

   descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
   // 类型为Optional
   if (Optional.class == descriptor.getDependencyType()) {
      // javaUtilOptionalClass类注入的特殊处理
      return createOptionalDependency(descriptor, requestingBeanName);
   }
   else if (ObjectFactory.class == descriptor.getDependencyType() ||
         ObjectProvider.class == descriptor.getDependencyType()) {
      return new DependencyObjectProvider(descriptor, requestingBeanName);
   }
   else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
      // javaxInjectProviderClass类注入的特殊处理
      return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
   }
   else {
      // 通用类注入的处理
      // 如有必要，请获取延迟解析代理
      Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
            descriptor, requestingBeanName);
      if (result == null) {
         // 解析依赖关系，返回的result为创建好的依赖对象的bean实例
         result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
      }
      return result;
   }
}
```

DefaultListableBeanFactory#doResolveDependency

```java
@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
   // 设置当前的descriptor(存储了方法参数等信息)为当前注入点
   InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
   try {
      // 如果是ShortcutDependencyDescriptor，则直接通过getBean方法获取Bean实例，并返回；否则返回null
      Object shortcut = descriptor.resolveShortcut(this);
      if (shortcut != null) {
         return shortcut;
      }

      // 拿到descriptor包装的方法的参数类型（通过参数索引定位到具体的参数）
      Class<?> type = descriptor.getDependencyType();
      // 用于支持@Value（确定给定的依赖项是否声明Value注解，如果有则拿到值）
      Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
      if (value != null) {
         if (value instanceof String) {
            // 解析el表达式
            String strVal = resolveEmbeddedValue((String) value);
            BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                  getMergedBeanDefinition(beanName) : null);
            // 计算el表达式中的值
            value = evaluateBeanDefinitionString(strVal, bd);
         }
         TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
         try {
            // @Value注解上的value值进行转化并返回
            return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
         }
         catch (UnsupportedOperationException ex) {
            // A custom TypeConverter which does not support TypeDescriptor resolution...
            return (descriptor.getField() != null ?
                  converter.convertIfNecessary(value, type, descriptor.getField()) :
                  converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
         }
      }
      // 需要注入的接口有多个实现类
			// 解析MultipleBean（下文的MultipleBean都是指类型为：Array、Collection、Map）
      Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
      if (multipleBeans != null) {
         return multipleBeans;
      }
			// 根据类型找到候选者
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
      if (matchingBeans.isEmpty()) {
         if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
         }
         return null;
      }

      String autowiredBeanName;
      Object instanceCandidate;
			// 匹配到的自动注入的bean有多个
      if (matchingBeans.size() > 1) {
         autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
         if (autowiredBeanName == null) {
            // required=true && 存在多个候选者	
            if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
               return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
            }
            else {
               // In case of an optional Collection/Map, silently ignore a non-unique case:
               // possibly it was meant to be an empty collection of multiple regular beans
               // (before 4.3 in particular when we didn't even look for collection beans).
               return null;
            }
         }
         instanceCandidate = matchingBeans.get(autowiredBeanName);
      }
      else {
         // We have exactly one match.
         Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
         autowiredBeanName = entry.getKey();
         instanceCandidate = entry.getValue();
      }

      if (autowiredBeanNames != null) {
         autowiredBeanNames.add(autowiredBeanName);
      }
      if (instanceCandidate instanceof Class) {
         instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
      }
      Object result = instanceCandidate;
      if (result instanceof NullBean) {
         if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
         }
         result = null;
      }
      if (!ClassUtils.isAssignableValue(type, result)) {
         throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
      }
      return result;
   }
   finally {
      ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
   }
}
```

DefaultListableBeanFactory#resolveMultipleBeans

```java
@Nullable
private Object resolveMultipleBeans(DependencyDescriptor descriptor, @Nullable String beanName,
      @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) {

   final Class<?> type = descriptor.getDependencyType();

   if (descriptor instanceof StreamDependencyDescriptor) {
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
      if (autowiredBeanNames != null) {
         autowiredBeanNames.addAll(matchingBeans.keySet());
      }
      Stream<Object> stream = matchingBeans.keySet().stream()
            .map(name -> descriptor.resolveCandidate(name, type, this))
            .filter(bean -> !(bean instanceof NullBean));
      if (((StreamDependencyDescriptor) descriptor).isOrdered()) {
         stream = stream.sorted(adaptOrderComparator(matchingBeans));
      }
      return stream;
   }
   else if (type.isArray()) {
      Class<?> componentType = type.getComponentType();
      ResolvableType resolvableType = descriptor.getResolvableType();
      // 解析数组类型
      Class<?> resolvedArrayType = resolvableType.resolve(type);
      if (resolvedArrayType != type) {
         componentType = resolvableType.getComponentType().resolve();
      }
      if (componentType == null) {
         return null;
      }
      // 根据类型找到后选择
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, componentType,
            new MultiElementDescriptor(descriptor));
      if (matchingBeans.isEmpty()) {
         return null;
      }
      if (autowiredBeanNames != null) {
         autowiredBeanNames.addAll(matchingBeans.keySet());
      }
      TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
      // 转化values值
      Object result = converter.convertIfNecessary(matchingBeans.values(), resolvedArrayType);
      if (result instanceof Object[]) {
         Comparator<Object> comparator = adaptDependencyComparator(matchingBeans);
         if (comparator != null) {
            Arrays.sort((Object[]) result, comparator);
         }
      }
      return result;
   }
   else if (Collection.class.isAssignableFrom(type) && type.isInterface()) {
      // 解析Collection类型
      Class<?> elementType = descriptor.getResolvableType().asCollection().resolveGeneric();
      if (elementType == null) {
         return null;
      }
     // 根据类型找到候选者
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, elementType,
            new MultiElementDescriptor(descriptor));
      if (matchingBeans.isEmpty()) {
         return null;
      }
      if (autowiredBeanNames != null) {
         autowiredBeanNames.addAll(matchingBeans.keySet());
      }
      TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
      // 转化values值
      Object result = converter.convertIfNecessary(matchingBeans.values(), type);
      if (result instanceof List) {
         if (((List<?>) result).size() > 1) {
            Comparator<Object> comparator = adaptDependencyComparator(matchingBeans);
            if (comparator != null) {
               ((List<?>) result).sort(comparator);
            }
         }
      }
      return result;
   }
   else if (Map.class == type) {
      // 根据类型找到候选者
      ResolvableType mapType = descriptor.getResolvableType().asMap();
      Class<?> keyType = mapType.resolveGeneric(0);
      if (String.class != keyType) {
         return null;
      }
      Class<?> valueType = mapType.resolveGeneric(1);
      if (valueType == null) {
         return null;
      }
      // 根据类型找到候选者
      Map<String, Object> matchingBeans = findAutowireCandidates(beanName, valueType,
            new MultiElementDescriptor(descriptor));
      if (matchingBeans.isEmpty()) {
         return null;
      }
      if (autowiredBeanNames != null) {
         autowiredBeanNames.addAll(matchingBeans.keySet());
      }
      return matchingBeans;
   }
   else {
      return null;
   }
}
```

> 先取字段@Value的值，如果成功获取，就直接返回解析出来的值
>
> 否则，进行@Autowired这一套逻辑进行解析，并找出最优的依赖进行返回

DefaultListableBeanFactory#findAutowireCandidates

```java
protected Map<String, Object> findAutowireCandidates(
      @Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {
	 // 获取给定类型的所有beanName，包括在祖先工厂中定义的beanName
   String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
         this, requiredType, true, descriptor.isEager());
   Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
   // 首先从已经解析的依赖关系缓存中寻找是否存在我们想要的类型，有就只用利用
   for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
      Class<?> autowiringType = classObjectEntry.getKey();
      // autowiringType是否与requiredType相同，或者是requiredType的超类、超接口
      if (autowiringType.isAssignableFrom(requiredType)) {
         // 如果requiredType匹配，则从缓存中拿到相应的自动装配值（bean实例）
         Object autowiringValue = classObjectEntry.getValue();
         // 根据给定的所需类型解析给定的自动装配值
         autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
         if (requiredType.isInstance(autowiringValue)) {
            // 将autowiringValue放到结果集中，此时的value为bean实例
            result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
            break;
         }
      }
   }
   // 遍历从容器中获取到的类型符合的beanName
   for (String candidate : candidateNames) {
      // isAutowireCandidate：判断是否有资格作为依赖注入的候选者
      // 如果不是自引用 && candidate有资格作为依赖注入的候选者
      if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
         // 将候选者添加到result中
         addCandidateEntry(result, candidate, descriptor, requiredType);
      }
   }
   // 如果结果为空 && type不是MultipleBean（Array、Collection、Map），则使用降级匹配
   if (result.isEmpty()) {
      boolean multiple = indicatesMultipleBeans(requiredType);
      // Consider fallback matches if the first pass failed to find anything...
      // 使用降级匹配（跟正常匹配类似）
      DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
      for (String candidate : candidateNames) {
         if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor) &&
               (!multiple || getAutowireCandidateResolver().hasQualifier(descriptor))) {
            addCandidateEntry(result, candidate, descriptor, requiredType);
         }
      }
      if (result.isEmpty() && !multiple) {
         // Consider self references as a final pass...
         // but in the case of a dependency collection, not the very same bean itself.
         // 如果使用降级匹配结果还是空，则考虑自引
         for (String candidate : candidateNames) {
            if (isSelfReference(beanName, candidate) &&
                  (!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
                  isAutowireCandidate(candidate, fallbackDescriptor)) {
               // 5.1 如果是自引用 && (descriptor不是MultiElementDescriptor || beanName不等于候选者)
               // && candidate允许依赖注入，则将候选者添加到result中
               addCandidateEntry(result, candidate, descriptor, requiredType);
            }
         }
      }
   }
   return result;
}
```

DefaultListableBeanFactory#isAutowireCandidate(String, DependencyDescriptor)

```java
@Override
public boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor)
      throws NoSuchBeanDefinitionException {
	 // getAutowireCandidateResolver: 返回BeanFactory的@Autowire解析器，开启注解后的解析器为：ContextAnnotationAutowireCandidateResolver
   // 解析beanName对应的bean是否有资格作为候选者
   return isAutowireCandidate(beanName, descriptor, getAutowireCandidateResolver());
}
```

DefaultListableBeanFactory#isAutowireCandidate(String, DependencyDescriptor, AutowireCandidateResolver)

```java
protected boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor, AutowireCandidateResolver resolver)
      throws NoSuchBeanDefinitionException {
	 // 解析beanName，去掉FactoryBean的修饰符“&”
   String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
   if (containsBeanDefinition(beanDefinitionName)) {
      // beanDefinitionMap缓存中存在beanDefinitionName：通过beanDefinitionName缓存拿到MergedBeanDefinition，
      // 将MergedBeanDefinition作为参数，解析beanName是否有资格作为候选者
      return isAutowireCandidate(beanName, getMergedLocalBeanDefinition(beanDefinitionName), descriptor, resolver);
   }
   else if (containsSingleton(beanName)) {
      // singletonObjects缓存中存在beanName：使用beanName构建RootBeanDefinition作为参数，解析beanName是否有资格作为候选者
      return isAutowireCandidate(beanName, new RootBeanDefinition(getType(beanName)), descriptor, resolver);
   }
	 // 在beanDefinitionMap缓存和singletonObjects缓存中都不存在，则在parentBeanFactory中递归解析beanName是否有资格作为候选者

   BeanFactory parent = getParentBeanFactory();
   if (parent instanceof DefaultListableBeanFactory) {
      // No bean definition found in this factory -> delegate to parent.
      return ((DefaultListableBeanFactory) parent).isAutowireCandidate(beanName, descriptor, resolver);
   }
   else if (parent instanceof ConfigurableListableBeanFactory) {
      // If no DefaultListableBeanFactory, can't pass the resolver along.
      return ((ConfigurableListableBeanFactory) parent).isAutowireCandidate(beanName, descriptor);
   }
   else {
      return true;
   }
}
```

DefaultListableBeanFactory#isAutowireCandidate(String, RootBeanDefinition, DependencyDescriptor, AutowireCandidateResolver)

```java
protected boolean isAutowireCandidate(String beanName, RootBeanDefinition mbd,
      DependencyDescriptor descriptor, AutowireCandidateResolver resolver) {
	 // 解析beanName，去掉FactoryBean的修饰符“&”
   String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
   // 解析mbd的beanClass
   resolveBeanClass(mbd, beanDefinitionName);
   if (mbd.isFactoryMethodUnique && mbd.factoryMethodToIntrospect == null) {
      new ConstructorResolver(this).resolveFactoryMethodIfPossible(mbd);
   }
   BeanDefinitionHolder holder = (beanName.equals(beanDefinitionName) ?
         this.mergedBeanDefinitionHolders.computeIfAbsent(beanName,
               key -> new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName))) :
         new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName)));
   return resolver.isAutowireCandidate(holder, descriptor);
}
```

QualifierAnnotationAutowireCandidateResolver#isAutowireCandidate

```java
@Override
public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
   boolean match = super.isAutowireCandidate(bdHolder, descriptor);
   if (match) {
      match = checkQualifiers(bdHolder, descriptor.getAnnotations());
      if (match) {
         MethodParameter methodParam = descriptor.getMethodParameter();
         if (methodParam != null) {
            Method method = methodParam.getMethod();
            if (method == null || void.class == method.getReturnType()) {
               match = checkQualifiers(bdHolder, methodParam.getMethodAnnotations());
            }
         }
      }
   }
   return match;
}

// GenericTypeAwareAutowireCandidateResolver.java
@Override
public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
    if (!super.isAutowireCandidate(bdHolder, descriptor)) {
        // If explicitly false, do not proceed with any other checks...
        // 如果父类返回false，则直接返回false
        return false;
    }
    // descriptor为空 || bdHolder的类型与descriptor的类型匹配，则返回true
    return (descriptor == null || checkGenericTypeMatch(bdHolder, descriptor));
}
 
// SimpleAutowireCandidateResolver.java
@Override
public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
    // 获取此bean是否可以自动注入到其他bean（autowireCandidate属性），默认为true，一般不修改，因此这边返回true
    return bdHolder.getBeanDefinition().isAutowireCandidate();
}
```

DefaultListableBeanFactory#addCandidateEntry

```java
  private void addCandidateEntry(Map<String, Object> candidates, String candidateName,
      DependencyDescriptor descriptor, Class<?> requiredType) {

   if (descriptor instanceof MultiElementDescriptor) {
      Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
      if (!(beanInstance instanceof NullBean)) {
         candidates.put(candidateName, beanInstance);
      }
   }
   else if (containsSingleton(candidateName) || (descriptor instanceof StreamDependencyDescriptor &&
         ((StreamDependencyDescriptor) descriptor).isOrdered())) {
      // 获取bean，通过调用beanFactory.getBean(beanName)递归解析bean
      Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
      // put到candidates中
      candidates.put(candidateName, (beanInstance instanceof NullBean ? null : beanInstance));
   }
   else {
      candidates.put(candidateName, getType(candidateName));
   }
}
```



#### @Qualifier注解

QualifierAnnotationAutowireCandidateResolver#checkQualifiers

```java
protected boolean checkQualifiers(BeanDefinitionHolder bdHolder, Annotation[] annotationsToSearch) {
   if (ObjectUtils.isEmpty(annotationsToSearch)) {
      return true;
   }
   SimpleTypeConverter typeConverter = new SimpleTypeConverter();
   for (Annotation annotation : annotationsToSearch) {
      Class<? extends Annotation> type = annotation.annotationType();
      boolean checkMeta = true;
      boolean fallbackToMeta = false;
      // 是否含有@Qualifier注解
      if (isQualifier(type)) {
         if (!checkQualifier(bdHolder, annotation, typeConverter)) {
            fallbackToMeta = true;
         }
         else {
            checkMeta = false;
         }
      }
      if (checkMeta) {
         boolean foundMeta = false;
         for (Annotation metaAnn : type.getAnnotations()) {
            Class<? extends Annotation> metaType = metaAnn.annotationType();
            if (isQualifier(metaType)) {
               foundMeta = true;
               // Only accept fallback match if @Qualifier annotation has a value...
               // Otherwise it is just a marker for a custom qualifier annotation.
               if ((fallbackToMeta && StringUtils.isEmpty(AnnotationUtils.getValue(metaAnn))) ||
                     !checkQualifier(bdHolder, metaAnn, typeConverter)) {
                  return false;
               }
            }
         }
         if (fallbackToMeta && !foundMeta) {
            return false;
         }
      }
   }
   return true;
}
```

QualifierAnnotationAutowireCandidateResolver#checkQualifier

```java
protected boolean checkQualifier(
      BeanDefinitionHolder bdHolder, Annotation annotation, TypeConverter typeConverter) {

   Class<? extends Annotation> type = annotation.annotationType();
   RootBeanDefinition bd = (RootBeanDefinition) bdHolder.getBeanDefinition();
	 // 检查bean定义有没有这个type全限定类名的AutowireCandidateQualifier对象
   AutowireCandidateQualifier qualifier = bd.getQualifier(type.getName());
   if (qualifier == null) {
      // 没有的话看下有没短的类名的
      qualifier = bd.getQualifier(ClassUtils.getShortName(type));
   }
   if (qualifier == null) {
      // First, check annotation on qualified element, if any
      // 检查bean定义里是否设置了type类型的限定类
      Annotation targetAnnotation = getQualifiedElementAnnotation(bd, type);
      // Then, check annotation on factory method, if applicable
      if (targetAnnotation == null) {
         // 从工厂方法找是否有限定注解
         targetAnnotation = getFactoryMethodAnnotation(bd, type);
      }
      if (targetAnnotation == null) {
         RootBeanDefinition dbd = getResolvedDecoratedDefinition(bd);
         if (dbd != null) {
            // 从bean装饰的定义中找注解
            targetAnnotation = getFactoryMethodAnnotation(dbd, type);
         }
      }
      if (targetAnnotation == null) {
         // 为空的话就从bean定义的目标类上去找
         // Look for matching annotation on the target class
         if (getBeanFactory() != null) {
            try {
               // beanFactory去获取
               Class<?> beanType = getBeanFactory().getType(bdHolder.getBeanName());
               if (beanType != null) {
                  // 去bean定义本身上去获取@Qualifier注解
                  targetAnnotation = AnnotationUtils.getAnnotation(ClassUtils.getUserClass(beanType), type);
               }
            }
            catch (NoSuchBeanDefinitionException ex) {
               // Not the usual case - simply forget about the type check...
            }
         }
         if (targetAnnotation == null && bd.hasBeanClass()) {
            targetAnnotation = AnnotationUtils.getAnnotation(ClassUtils.getUserClass(bd.getBeanClass()), type);
         }
      }
      // 如果bean定义本身上的@Qualifier和需要自动注入处的@Qualifier中的值相等
      if (targetAnnotation != null && targetAnnotation.equals(annotation)) {
         return true;
      }
   }

   Map<String, Object> attributes = AnnotationUtils.getAnnotationAttributes(annotation);
   if (attributes.isEmpty() && qualifier == null) {
      // If no attributes, the qualifier must be present
      return false;
   }
   // 匹配的限定注解的属性
   for (Map.Entry<String, Object> entry : attributes.entrySet()) {
      String attributeName = entry.getKey();
      Object expectedValue = entry.getValue();
      Object actualValue = null;
      // Check qualifier first
      if (qualifier != null) {
         actualValue = qualifier.getAttribute(attributeName);
      }
      if (actualValue == null) {
         // Fall back on bean definition attribute
         actualValue = bd.getAttribute(attributeName);
      }
      if (actualValue == null && attributeName.equals(AutowireCandidateQualifier.VALUE_KEY) &&
            expectedValue instanceof String && bdHolder.matchesName((String) expectedValue)) {
         // Fall back on bean name (or alias) match
         continue;
      }
      if (actualValue == null && qualifier != null) {
         // Fall back on default, but only if the qualifier is present
         actualValue = AnnotationUtils.getDefaultValue(annotation, attributeName);
      }
      if (actualValue != null) {
         actualValue = typeConverter.convertIfNecessary(actualValue, expectedValue.getClass());
      }
      if (!expectedValue.equals(actualValue)) {
         return false;
      }
   }
   return true;
}
```

> 因为先处理的@Qualifier注解，所以@Qualifier->@Primary->@Priority三个注解的降序排列。
>
> 当@Qualifier出现在bean定义的时候，在自动注入的时候需要配合@Qualifie使用，否则@Qualifier在定义bean的时候没有作用
>
> @Qualifier必须和@Autowired配合使用



#### 获取最优的候选者

DefaultListableBeanFactory#determineAutowireCandidate

```java
@Nullable
protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
   Class<?> requiredType = descriptor.getDependencyType();
   // 根据@Primary注解来选择最优解
   String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
   if (primaryCandidate != null) {
      return primaryCandidate;
   }
   // 根据@Priority注解来选择最优解
   String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
   if (priorityCandidate != null) {
      return priorityCandidate;
   }
   // Fallback
   // 如果通过以上两步都不能选择出最优解，则使用最基本的策略（先类型再名称的原则）
   for (Map.Entry<String, Object> entry : candidates.entrySet()) {
      String candidateName = entry.getKey();
      Object beanInstance = entry.getValue();
      // 名称匹配就是根绝@Autowired修饰的变量命令来匹配
      if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
            matchesBeanName(candidateName, descriptor.getDependencyName())) {
         return candidateName;
      }
   }
   return null;
}
```

DefaultListableBeanFactory#determinePrimaryCandidate

```java
@Nullable
protected String determinePrimaryCandidate(Map<String, Object> candidates, Class<?> requiredType) {
   String primaryBeanName = null;
   for (Map.Entry<String, Object> entry : candidates.entrySet()) {
      String candidateBeanName = entry.getKey();
      Object beanInstance = entry.getValue();
      // 判断是否存在@Primary
      if (isPrimary(candidateBeanName, beanInstance)) {
         // 标识了@Primary的候选者不止一
         if (primaryBeanName != null) {
            boolean candidateLocal = containsBeanDefinition(candidateBeanName);
            boolean primaryLocal = containsBeanDefinition(primaryBeanName);
            // candidateBeanName和primaryBeanName都存在于beanDefinitionMap缓存中
            if (candidateLocal && primaryLocal) {
               throw new NoUniqueBeanDefinitionException(requiredType, candidates.size(),
                     "more than one 'primary' bean found among candidates: " + candidates.keySet());
            }
            // 当candidateLocalName存在于beanDefinitionMap，而primaryBeanName不存在于beanDefinitionMap，
            // 使用优先使用candidateBeanName
            else if (candidateLocal) {
               primaryBeanName = candidateBeanName;
            }
         }
         else {
            // primaryBeanName还为空，代表是第一个符合的候选者，直接将primaryBeanName赋值为candidateBeanName
            primaryBeanName = candidateBeanName;
         }
      }
   }
   return primaryBeanName;
}
```

DefaultListableBeanFactory#determineHighestPriorityCandidate

```java
@Nullable
protected String determineHighestPriorityCandidate(Map<String, Object> candidates, Class<?> requiredType) {
   // 用来保存最高优先级的beanName 
   String highestPriorityBeanName = null;
   // 用来保存最高优先级的优先级值
   Integer highestPriority = null;
   // 遍历所有候选者
   for (Map.Entry<String, Object> entry : candidates.entrySet()) {
      String candidateBeanName = entry.getKey();
      Object beanInstance = entry.getValue();
      if (beanInstance != null) {
         // 去@Priority拿到beanInstance的优先级（javax.annotation.Priority）
         Integer candidatePriority = getPriority(beanInstance);
         if (candidatePriority != null) {
            if (highestPriorityBeanName != null) {
               // 如果之前已经有候选者有优先级，则进行选择
               if (candidatePriority.equals(highestPriority)) {
                  // 如果存在两个优先级相同的Bean，则抛出异常
                  throw new NoUniqueBeanDefinitionException(requiredType, candidates.size(),
                        "Multiple beans found with the same priority ('" + highestPriority +
                        "') among candidates: " + candidates.keySet());
               }
               else if (candidatePriority < highestPriority) {
                  // 使用优先级值较小的Bean作为最优解（值越低，优先级越高）
                  highestPriorityBeanName = candidateBeanName;
                  highestPriority = candidatePriority;
               }
            }
            else {
               // 第一个候选者
               highestPriorityBeanName = candidateBeanName;
               highestPriority = candidatePriority;
            }
         }
      }
   }
   return highestPriorityBeanName;
}
```

> 如果需要注入的bean同时有@Primary和@Priority注解：
>
> 优先找@Primary注解做为依赖注入，如果含有多个@Primary修饰的候选者就会抛出异常。
>
> 然后会找@Priority的上面的value值，value值越小，优先级越高，如果遍历的过程中发现2个优先级的值一样的，就会抛出异常（感觉这里的判断写的有点怪异 --> 同一组值可能顺序不一样，而导致结果不一样）



 本篇到这里其实已经介绍完整个bean用构造方法（构造器选择、构造方法参数的自动注入）进行实例化的全部过程（基于annonation）。

Spring中有三种实例化Bean的方式，类构造器实例化、静态工厂方法实例化及实例工厂方法实例化。两种Bean类型，一种是普通Bean，另一种是工厂Bean，即FactoryBean，这两种Bean都被容器管理，但工厂Bean跟普通Bean不同，其返回的对象不是指定类的一个实例，其返回的是该FactoryBean的getObject方法所返回的对象。在Spring框架内部，有很多地方有FactoryBean的实现类，它们在很多应用如(Spring的AOP、ORM、事务管理)及与其它第三框架(ehCache)集成时都有体现。



#### @EventListener

EventListenerMethodProcessor#afterSingletonsInstantiated

```java
@Override
public void afterSingletonsInstantiated() {
   ConfigurableListableBeanFactory beanFactory = this.beanFactory;
   Assert.state(this.beanFactory != null, "No ConfigurableListableBeanFactory set");
   // 获取所有的beanName
   String[] beanNames = beanFactory.getBeanNamesForType(Object.class);
   for (String beanName : beanNames) {
      if (!ScopedProxyUtils.isScopedTarget(beanName)) {
         // !beanName.startsWith("scopedTarget.")
        
         Class<?> type = null;
         try {
            type = AutoProxyUtils.determineTargetClass(beanFactory, beanName);
         }
         catch (Throwable ex) {
            // An unresolvable bean type, probably from a lazy bean - let's ignore it.
            if (logger.isDebugEnabled()) {
               logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
            }
         }
         if (type != null) {
            // type是ScopedObject类型
            if (ScopedObject.class.isAssignableFrom(type)) {
               try {
                  Class<?> targetClass = AutoProxyUtils.determineTargetClass(
                        beanFactory, ScopedProxyUtils.getTargetBeanName(beanName));
                  if (targetClass != null) {
                     type = targetClass;
                  }
               }
               catch (Throwable ex) {
                  // An invalid scoped proxy arrangement - let's ignore it.
                  if (logger.isDebugEnabled()) {
                     logger.debug("Could not resolve target bean for scoped proxy '" + beanName + "'", ex);
                  }
               }
            }
            try {
               processBean(beanName, type);
            }
            catch (Throwable ex) {
               throw new BeanInitializationException("Failed to process @EventListener " +
                     "annotation on bean with name '" + beanName + "'", ex);
            }
         }
      }
   }
}
```

EventListenerMethodProcessor#processBean

```java
private void processBean(final String beanName, final Class<?> targetType) {
   // nonAnnotatedClasses不包含 && !type.getName().startsWith("java.") && 非spring容器类
   if (!this.nonAnnotatedClasses.contains(targetType) &&
         AnnotationUtils.isCandidateClass(targetType, EventListener.class) &&
         !isSpringContainerClass(targetType)) {

      Map<Method, EventListener> annotatedMethods = null;
      try {
         // 找到方法上带有@EventListener的方法
         annotatedMethods = MethodIntrospector.selectMethods(targetType,
               (MethodIntrospector.MetadataLookup<EventListener>) method ->
                     AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
      }
      catch (Throwable ex) {
         // An unresolvable type in a method signature, probably from a lazy bean - let's ignore it.
         if (logger.isDebugEnabled()) {
            logger.debug("Could not resolve methods for bean with name '" + beanName + "'", ex);
         }
      }

      if (CollectionUtils.isEmpty(annotatedMethods)) {
         // 方法上不包含EventListener的注解的bean会添加到nonAnnotatedClasses
         this.nonAnnotatedClasses.add(targetType);
         if (logger.isTraceEnabled()) {
            logger.trace("No @EventListener annotations found on bean class: " + targetType.getName());
         }
      }
      else {
         // Non-empty set of methods
         ConfigurableApplicationContext context = this.applicationContext;
         Assert.state(context != null, "No ApplicationContext set");
         // 事件监听工厂，默认只有一个DefaultEventListenerFactory
         List<EventListenerFactory> factories = this.eventListenerFactories;
         Assert.state(factories != null, "EventListenerFactory List not initialized");
         for (Method method : annotatedMethods.keySet()) {
            for (EventListenerFactory factory : factories) {
               if (factory.supportsMethod(method)) {
                  Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
                  // 创建一个ApplicationListenerMethodAdapter
                  ApplicationListener<?> applicationListener =
                        factory.createApplicationListener(beanName, targetType, methodToUse);
                  if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                     // 初始化
                     ((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
                  }
                  context.addApplicationListener(applicationListener);
                  break;
               }
            }
         }
         if (logger.isDebugEnabled()) {
            logger.debug(annotatedMethods.size() + " @EventListener methods processed on bean '" +
                  beanName + "': " + annotatedMethods);
         }
      }
   }
}
```



[下一篇](https://github.com/zhusidao/zhusidao.github.io/wiki/bean%E5%B1%9E%E6%80%A7%E8%B5%8B%E5%80%BC)将开启bean的下一个阶段->bean的属性赋值