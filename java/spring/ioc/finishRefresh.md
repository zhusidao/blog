# finishRefresh

前面的篇幅已经结束了spring的 bean注册->bean实例化->bean属性赋值->bean初始化，本篇开始介绍**finishRefresh()**方法

AbstractApplicationContext#finishRefresh

```java
protected void finishRefresh() {
   // Clear context-level resource caches (such as ASM metadata from scanning).
   // 清除上下文级别的资源缓存（例如来自扫描的ASM元数据）。
   clearResourceCaches();

   // Initialize lifecycle processor for this context.
   // 为此上下文初始化生命周期处理器
   initLifecycleProcessor();

   // Propagate refresh to lifecycle processor first.
   // 首先将刷新完毕事件传播到生命周期处理器（触发isAutoStartup方法返回true的SmartLifecycle的start方法）
   getLifecycleProcessor().onRefresh();

   // Publish the final event.
   // 推送上下文刷新完毕事件到相应的监听器
   publishEvent(new ContextRefreshedEvent(this));

   // Participate in LiveBeansView MBean, if active.
   // 和MBeanServer和MBean有关的。相当于把当前容器上下文，注册到MBeanServer里面去。
	// 这样子，MBeanServer持久了容器的引用，就可以拿到容器的所有内容了，也就让Spring支持到了MBean的相关功能
   LiveBeansView.registerApplicationContext(this);
}
```



#### 初始化bean生命周期处理器

AbstractApplicationContext#initLifecycleProcessor

```java
protected void initLifecycleProcessor() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   // 判断BeanFactory是否已经存在生命周期处理器（固定使用beanName=lifecycleProcessor）
   if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
      // 如果已经存在，则将该bean赋值给lifecycleProcessor
      this.lifecycleProcessor =
            beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
      }
   }
   else {
      // 如果不存在，则使用DefaultLifecycleProcessor
      DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
      defaultProcessor.setBeanFactory(beanFactory);
      this.lifecycleProcessor = defaultProcessor;
      // 并将DefaultLifecycleProcessor作为默认的生命周期处理器，注册到BeanFactory中
      beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
               "[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
      }
   }
}
```



#### bean生命周期回调钩子

DefaultLifecycleProcessor#onRefresh

```java
@Override
public void onRefresh() {
   startBeans(true);
   this.running = true;
}
```

DefaultLifecycleProcessor#startBeans

```java
private void startBeans(boolean autoStartupOnly) {
   // 获取所有的Lifecycle bean
   Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
   // 将Lifecycle bean 按阶段分组，阶段通过实现Phased接口得到
   Map<Integer, LifecycleGroup> phases = new HashMap<>();
   // 遍历所有Lifecycle bean，按阶段值分组
   lifecycleBeans.forEach((beanName, bean) -> {
      if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
         // 获取bean的阶段值（如果没有实现Phased接口，则值为0）
         int phase = getPhase(bean);
         // 拿到存放该阶段值的LifecycleGroup
         LifecycleGroup group = phases.get(phase);
         if (group == null) {
            // 如果该阶段值的LifecycleGroup为null，则新建一个
            group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
            phases.put(phase, group);
         }
         // 将bean添加到该LifecycleGroup
         group.add(beanName, bean);
      }
   });
   // 如果phases不为空
   if (!phases.isEmpty()) {
      List<Integer> keys = new ArrayList<>(phases.keySet());
      // 按阶段值进行排序
      Collections.sort(keys);
      // 按阶段值顺序，调用LifecycleGroup中的所有Lifecycle的start方法
      for (Integer key : keys) {
         phases.get(key).start();
      }
   }
}
```

DefaultLifecycleProcessor#getLifecycleBeans

```java
protected Map<String, Lifecycle> getLifecycleBeans() {
   // 获取beanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   Map<String, Lifecycle> beans = new LinkedHashMap<>();
   // 获取所有Lifecycle类型的bean
   String[] beanNames = beanFactory.getBeanNamesForType(Lifecycle.class, false, false);
   for (String beanName : beanNames) {
      // 去除beanName前面的'&'
      String beanNameToRegister = BeanFactoryUtils.transformedBeanName(beanName);
      // 判断是否是factoryBean
      boolean isFactoryBean = beanFactory.isFactoryBean(beanNameToRegister);
      // 如果是factoryBean就在前面加上'&'
      String beanNameToCheck = (isFactoryBean ? BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
      // 为单例 || 为SmartLifecycle类型的bean
      if ((beanFactory.containsSingleton(beanNameToRegister) &&
            (!isFactoryBean || matchesBeanType(Lifecycle.class, beanNameToCheck, beanFactory))) ||
            matchesBeanType(SmartLifecycle.class, beanNameToCheck, beanFactory)) {
         Object bean = beanFactory.getBean(beanNameToCheck);
         // 不是当前类 && 是Lifecycle类型的bean
         if (bean != this && bean instanceof Lifecycle) {
            beans.put(beanNameToRegister, (Lifecycle) bean);
         }
      }
   }
   return beans;
}
```

DefaultLifecycleProcessor#start

```java
public void start() {
   if (this.members.isEmpty()) {
      return;
   }
   if (logger.isDebugEnabled()) {
      logger.debug("Starting beans in phase " + this.phase);
   }
   // 排序
   Collections.sort(this.members);
   // 遍历LifecycleGroup的members值
   for (LifecycleGroupMember member : this.members) {
      doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
   }
}
```

DefaultLifecycleProcessor#doStart

```java
private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
   // 移除beanName
   Lifecycle bean = lifecycleBeans.remove(beanName);
   if (bean != null && bean != this) {
      // 获取依赖bean
      String[] dependenciesForBean = getBeanFactory().getDependenciesForBean(beanName);
      for (String dependency : dependenciesForBean) {
         // 先执行依赖bean
         doStart(lifecycleBeans, dependency, autoStartupOnly);
      }
      if (!bean.isRunning() &&
            (!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
         if (logger.isTraceEnabled()) {
            logger.trace("Starting bean '" + beanName + "' of type [" + bean.getClass().getName() + "]");
         }
         try {
            // 调用bean.start()
            bean.start();
         }
         catch (Throwable ex) {
            throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Successfully started bean '" + beanName + "'");
         }
      }
   }
}
```

> Lifecycle为bean生命周期回调钩子，但是如果直接去实现，并不会直接进行执行start和stop方法；SmartLifecycle继承了Lifecycle，生命周期的回调的顺序需要根据Phased#getPhase的返回值来判断（升序），默认值为0。

下面我们看下stop()方法执行

```java
private void stopBeans() {
   // 获取lifecycleBeans
   Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
   Map<Integer, LifecycleGroup> phases = new HashMap<>();
   lifecycleBeans.forEach((beanName, bean) -> {
      // 获取阶段值
      int shutdownPhase = getPhase(bean);
      // 根绝阶段值进行分组
      LifecycleGroup group = phases.get(shutdownPhase);
      if (group == null) {
         group = new LifecycleGroup(shutdownPhase, this.timeoutPerShutdownPhase, lifecycleBeans, false);
         phases.put(shutdownPhase, group);
      }
      group.add(beanName, bean);
   });
   if (!phases.isEmpty()) {
      List<Integer> keys = new ArrayList<>(phases.keySet());
      // 降序排列
      keys.sort(Collections.reverseOrder());
      for (Integer key : keys) {
         // 执行stop
         phases.get(key).stop();
      }
   }
}
```

DefaultLifecycleProcessor#stop

```java
public void stop() {
   if (this.members.isEmpty()) {
      return;
   }
   if (logger.isDebugEnabled()) {
      logger.debug("Stopping beans in phase " + this.phase);
   }
   this.members.sort(Collections.reverseOrder());
   // 创建一个CountDownLatch
   CountDownLatch latch = new CountDownLatch(this.smartMemberCount);
   Set<String> countDownBeanNames = Collections.synchronizedSet(new LinkedHashSet<>());
   Set<String> lifecycleBeanNames = new HashSet<>(this.lifecycleBeans.keySet());
   for (LifecycleGroupMember member : this.members) {
      if (lifecycleBeanNames.contains(member.name)) {
         // 调用stop
         doStop(this.lifecycleBeans, member.name, latch, countDownBeanNames);
      }
      else if (member.bean instanceof SmartLifecycle) {
         // Already removed: must have been a dependent bean from another phase
         latch.countDown();
      }
   }
   try {
      // 超时阻塞
      latch.await(this.timeout, TimeUnit.MILLISECONDS);
      if (latch.getCount() > 0 && !countDownBeanNames.isEmpty() && logger.isInfoEnabled()) {
         logger.info("Failed to shut down " + countDownBeanNames.size() + " bean" +
               (countDownBeanNames.size() > 1 ? "s" : "") + " with phase value " +
               this.phase + " within timeout of " + this.timeout + "ms: " + countDownBeanNames);
      }
   }
   catch (InterruptedException ex) {
      Thread.currentThread().interrupt();
   }
}
```

> 每个相同phased(阶段)值的stop方法调用超时时间为30秒，每个阶段的超时值用CountDownLatch进行实现

DefaultLifecycleProcessor#stop

```java
private void doStop(Map<String, ? extends Lifecycle> lifecycleBeans, final String beanName,
      final CountDownLatch latch, final Set<String> countDownBeanNames) {
	 // 调用过的进行移除
   Lifecycle bean = lifecycleBeans.remove(beanName);
   if (bean != null) {
      // 获取依赖
      String[] dependentBeans = getBeanFactory().getDependentBeans(beanName);
      for (String dependentBean : dependentBeans) {
         // 递归调用
         doStop(lifecycleBeans, dependentBean, latch, countDownBeanNames);
      }
      try {
         if (bean.isRunning()) {
            // bean.isRunning()==true
            if (bean instanceof SmartLifecycle) {
               if (logger.isTraceEnabled()) {
                  logger.trace("Asking bean '" + beanName + "' of type [" +
                        bean.getClass().getName() + "] to stop");
               }
               countDownBeanNames.add(beanName);
               // bean.stop()
               ((SmartLifecycle) bean).stop(() -> {
                  latch.countDown();
                  countDownBeanNames.remove(beanName);
                  if (logger.isDebugEnabled()) {
                     logger.debug("Bean '" + beanName + "' completed its stop procedure");
                  }
               });
            }
            else {
               if (logger.isTraceEnabled()) {
                  logger.trace("Stopping bean '" + beanName + "' of type [" +
                        bean.getClass().getName() + "]");
               }
               bean.stop();
               if (logger.isDebugEnabled()) {
                  logger.debug("Successfully stopped bean '" + beanName + "'");
               }
            }
         }
         else if (bean instanceof SmartLifecycle) {
            // Don't wait for beans that aren't running...
            latch.countDown();
         }
      }
      catch (Throwable ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Failed to stop bean '" + beanName + "'", ex);
         }
      }
   }
}
```

#### 推送上下文刷新完毕事件

AbstractApplicationContext#publishEvent

```java
@Override
public void publishEvent(ApplicationEvent event) {
   publishEvent(event, null);
}
```

AbstractApplicationContext#publishEvent

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
   Assert.notNull(event, "Event must not be null");

   // Decorate event as an ApplicationEvent if necessary
   // 如有必要，将事件装饰为ApplicationEvent
   ApplicationEvent applicationEvent;
   if (event instanceof ApplicationEvent) {
      applicationEvent = (ApplicationEvent) event;
   }
   else {
      applicationEvent = new PayloadApplicationEvent<>(this, event);
      if (eventType == null) {
         eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
      }
   }

   // Multicast right now if possible - or lazily once the multicaster is initialized
   if (this.earlyApplicationEvents != null) {
      this.earlyApplicationEvents.add(applicationEvent);
   }
   else {
      // 使用事件广播器广播事件到相应的监听器
      getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
   }

   // Publish event via parent context as well...
   if (this.parent != null) {
      if (this.parent instanceof AbstractApplicationContext) {
         ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
      }
      else {
         this.parent.publishEvent(event);
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
   // 
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

AbstractApplicationEventMulticaster#getApplicationListeners

```java
protected Collection<ApplicationListener<?>> getApplicationListeners(
      ApplicationEvent event, ResolvableType eventType) {

   Object source = event.getSource();
   Class<?> sourceType = (source != null ? source.getClass() : null);
   ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

   // Quick check for existing entry on ConcurrentHashMap...
   // 快速的从缓存中获取
   ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
   if (retriever != null) {
      return retriever.getApplicationListeners();
   }

   if (this.beanClassLoader == null ||
         (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
               (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
      // Fully synchronized building and caching of a ListenerRetriever
      synchronized (this.retrievalMutex) {
         retriever = this.retrieverCache.get(cacheKey);
         if (retriever != null) {
            return retriever.getApplicationListeners();
         }
         // 缓存中为空
         retriever = new ListenerRetriever(true);
         // 获取监听
         Collection<ApplicationListener<?>> listeners =
               retrieveApplicationListeners(eventType, sourceType, retriever);
         // 存入缓存
         this.retrieverCache.put(cacheKey, retriever);
         return listeners;
      }
   }
   else {
      // No ListenerRetriever caching -> no synchronization necessary
      // 缓存中没有，没有同步的必要
      return retrieveApplicationListeners(eventType, sourceType, null);
   }
}
```

AbstractApplicationEventMulticaster#retrieveApplicationListeners

```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
      ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {

   List<ApplicationListener<?>> allListeners = new ArrayList<>();
   Set<ApplicationListener<?>> listeners;
   Set<String> listenerBeans;
   synchronized (this.retrievalMutex) {
      listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
      listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
   }

   // Add programmatically registered listeners, including ones coming
   // from ApplicationListenerDetector (singleton beans and inner beans).
   // 先将defaultRetriever中的applicationListeners添加到retriever和allListeners中
   for (ApplicationListener<?> listener : listeners) {
      if (supportsEvent(listener, eventType, sourceType)) {
         if (retriever != null) {
            retriever.applicationListeners.add(listener);
         }
         allListeners.add(listener);
      }
   }

   // Add listeners by bean name, potentially overlapping with programmatically
   // registered listeners above - but here potentially with additional metadata.
   if (!listenerBeans.isEmpty()) {
      ConfigurableBeanFactory beanFactory = getBeanFactory();
      for (String listenerBeanName : listenerBeans) {
         try {
            if (supportsEvent(beanFactory, listenerBeanName, eventType)) {
               // 获取bean
               ApplicationListener<?> listener =
                     beanFactory.getBean(listenerBeanName, ApplicationListener.class);
               if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
                  if (retriever != null) {
                     if (beanFactory.isSingleton(listenerBeanName)) {
                        // 单例bean添加到applicationListeners
                        retriever.applicationListeners.add(listener);
                     }
                     else {
                        // 非单例bean添加到applicationListenerBeans
                        retriever.applicationListenerBeans.add(listenerBeanName);
                     }
                  }
                  allListeners.add(listener);
               }
            }
            else {
               // Remove non-matching listeners that originally came from
               // ApplicationListenerDetector, possibly ruled out by additional
               // BeanDefinition metadata (e.g. factory method generics) above.
               // 移除类型不匹配的监听器
               Object listener = beanFactory.getSingleton(listenerBeanName);
               if (retriever != null) {
                  retriever.applicationListeners.remove(listener);
               }
               allListeners.remove(listener);
            }
         }
         catch (NoSuchBeanDefinitionException ex) {
            // Singleton listener instance (without backing bean definition) disappeared -
            // probably in the middle of the destruction phase
         }
      }
   }
	 // 监听器根据order排序
   AnnotationAwareOrderComparator.sort(allListeners);
   if (retriever != null && retriever.applicationListenerBeans.isEmpty()) {
      retriever.applicationListeners.clear();
      retriever.applicationListeners.addAll(allListeners);
   }
   return allListeners;
}
```

SimpleApplicationEventMulticaster#invokeListener

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
   ErrorHandler errorHandler = getErrorHandler();
   if (errorHandler != null) {
      try {
         // 执行监听
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         // 用handleError处理异常
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
   }
}
```

到这里finishRefresh方法已经介绍完毕了，看完ioc实属不易，需要很耐心。

基本都这里，整个ioc核心代码基本都已经分析完毕，总体流程就是：bean注册->bean实例化->bean属性赋值->bean初始化