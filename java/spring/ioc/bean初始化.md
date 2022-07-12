# IOC源码第三篇-bean初始化

上篇[populateBean()](https://github.com/zhusidao/zhusidao.github.io/wiki/bean%E5%B1%9E%E6%80%A7%E8%B5%8B%E5%80%BC)主要是xml配置的属性值赋予、扩属性值注入，本篇接着上篇的属性实例化之后继续分析，本篇是介绍bean的初始化

AbstractAutowireCapableBeanFactory#initializeBean

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      // 执行Aware方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      // 初始化之前的前置处理
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 执行初始化方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
      // 初始化之后的前置处理
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

AbstractAutowireCapableBeanFactory#invokeAwareMethods

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
   if (bean instanceof Aware) {
      // bean如果实现了Aware

      if (bean instanceof BeanNameAware) {
         // bean如果实现了BeanNameAware
         // 以入参beanName执行setBeanName方法
         ((BeanNameAware) bean).setBeanName(beanName);
      }
      if (bean instanceof BeanClassLoaderAware) {
         // 如果实现了BeanClassLoaderAware
         // 获取类加载器
         ClassLoader bcl = getBeanClassLoader();
         if (bcl != null) {
            // 执行setBeanClassLoader
            ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
         }
      }
      if (bean instanceof BeanFactoryAware) {
         // 如果实现了BeanFactoryAware
         ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
      }
   }
}
```

AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization

```java
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      // ApplicationContextAwareProcessor处理AwareInterfaces下面会讲解
      // ApplicationListenerDetector不做处理
      // ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor
      // PostProcessorRegistrationDelegate.BeanPostProcessorChecker不做处理
      // AutowiredAnnotationBeanPostProcessor不做处理--没有重写
      // CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor执行带有@PostConstruct和@PreDestroy的方法
      // 执行实例化之前的后置处理
      Object current = processor.postProcessBeforeInitialization(result, beanName);
      if (current == null) {
         // 返回existingBean
         return result;
      }
      result = current;
   }
   return result;
}
```



#### 执行EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware相关接口的方法

ApplicationContextAwareProcessor#postProcessBeforeInitialization

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
         bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
         bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
      return bean;
   }

   AccessControlContext acc = null;

   if (System.getSecurityManager() != null) {
      acc = this.applicationContext.getBeanFactory().getAccessControlContext();
   }

   if (acc != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareInterfaces(bean);
         return null;
      }, acc);
   }
   else {
      // 执行AwareInterfaces中的相关
      invokeAwareInterfaces(bean);
   }

   return bean;
}
```

ApplicationContextAwareProcessor#invokeAwareInterfaces

```java
private void invokeAwareInterfaces(Object bean) {
   if (bean instanceof EnvironmentAware) {
      ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
   }
   if (bean instanceof EmbeddedValueResolverAware) {
      ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
   }
   if (bean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
   }
   if (bean instanceof ApplicationEventPublisherAware) {
      ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
   }
   if (bean instanceof MessageSourceAware) {
      ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
   }
   if (bean instanceof ApplicationContextAware) {
      ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
   }
}
```



#### 处理@PostConstruct和@PreDestroy

InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
   try {
      // 执行初始化方法
      metadata.invokeInitMethods(bean, beanName);
   }
   catch (InvocationTargetException ex) {
      throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
   }
   catch (Throwable ex) {
      throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
   }
   return bean;
}
```

InitDestroyAnnotationBeanPostProcessor#findLifecycleMetadata

```java
private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
   if (this.lifecycleMetadataCache == null) {
      // Happens after deserialization, during destruction...
      // 缓存为null直接执行buildLifecycleMetadata
      return buildLifecycleMetadata(clazz);
   }
   // Quick check on the concurrent map first, with minimal locking.
   // 缓存拿不到就buildLifecycleMetadata
   LifecycleMetadata metadata = this.lifecycleMetadataCache.get(clazz);
   if (metadata == null) {
      synchronized (this.lifecycleMetadataCache) {
         metadata = this.lifecycleMetadataCache.get(clazz);
         if (metadata == null) {
            metadata = buildLifecycleMetadata(clazz);
            this.lifecycleMetadataCache.put(clazz, metadata);
         }
         return metadata;
      }
   }
   return metadata;
}
```

InitDestroyAnnotationBeanPostProcessor#buildLifecycleMetadata

```java
private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
   if (!AnnotationUtils.isCandidateClass(clazz, Arrays.asList(this.initAnnotationType, this.destroyAnnotationType))) {
      return this.emptyLifecycleMetadata;
   }

   List<LifecycleElement> initMethods = new ArrayList<>();
   List<LifecycleElement> destroyMethods = new ArrayList<>();
   Class<?> targetClass = clazz;
   // 循环查询targetClass（及其父类）方法上面带有@PostConstruct、@PreDestroy注解的方法，并添加到各自的List
   do {
      final List<LifecycleElement> currInitMethods = new ArrayList<>();
      final List<LifecycleElement> currDestroyMethods = new ArrayList<>();

      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
            // 添加带有@PostConstruct的方法
            LifecycleElement element = new LifecycleElement(method);
            currInitMethods.add(element);
            if (logger.isTraceEnabled()) {
               logger.trace("Found init method on class [" + clazz.getName() + "]: " + method);
            }
         }
         if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
            // 添加带有@PreDestroy的方法
            currDestroyMethods.add(new LifecycleElement(method));
            if (logger.isTraceEnabled()) {
               logger.trace("Found destroy method on class [" + clazz.getName() + "]: " + method);
            }
         }
      });
      // 添加到前面
      initMethods.addAll(0, currInitMethods);
      destroyMethods.addAll(currDestroyMethods);
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);
	 // 带有@PostConstruct和@PreDestroy的List都为空，就会返回一个emptyLifecycleMetadata
   return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
         new LifecycleMetadata(clazz, initMethods, destroyMethods));
}
```

InitDestroyAnnotationBeanPostProcessor#invokeInitMethods

```java
public void invokeInitMethods(Object target, String beanName) throws Throwable {
   Collection<LifecycleElement> checkedInitMethods = this.checkedInitMethods;
   Collection<LifecycleElement> initMethodsToIterate =
         (checkedInitMethods != null ? checkedInitMethods : this.initMethods);
   if (!initMethodsToIterate.isEmpty()) {
      // 遍历带有@PostConstruct的方法
      for (LifecycleElement element : initMethodsToIterate) {
         if (logger.isTraceEnabled()) {
            logger.trace("Invoking init method on bean '" + beanName + "': " + element.getMethod());
         }
         // 反射调用
         element.invoke(target);
      }
   }
}
```

> EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware继承下列接口，就会自动执行相关方法，spring去继承这些方法的用法很常见



#### 执行InitMethod方法

AbstractAutowireCapableBeanFactory#invokeInitMethods

```java
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
      throws Throwable {

   boolean isInitializingBean = (bean instanceof InitializingBean);
   if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
      if (logger.isTraceEnabled()) {
         logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
      }
      if (System.getSecurityManager() != null) {
         try {
            AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
               ((InitializingBean) bean).afterPropertiesSet();
               return null;
            }, getAccessControlContext());
         }
         catch (PrivilegedActionException pae) {
            throw pae.getException();
         }
      }
      else {
         // 继承了InitializingBean，且security为null，执行afterPropertiesSet
         ((InitializingBean) bean).afterPropertiesSet();
      }
   }

   if (mbd != null && bean.getClass() != NullBean.class) {
      // 判断是否指定了 init-method()，
      // 如果指定了 init-method()，则再调用制定的init-method
      // @Bean注解上的initMethod、initMethod在注册阶段就已经被解析
      String initMethodName = mbd.getInitMethodName();
      if (StringUtils.hasLength(initMethodName) &&
            !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
            !mbd.isExternallyManagedInitMethod(initMethodName)) {
         // 利用反射机制执行
         invokeCustomInitMethod(beanName, bean, mbd);
      }
   }
}
```

AbstractAutowireCapableBeanFactory#invokeCustomInitMethod

```java
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd)
      throws Throwable {
	 // 初始化方法名称
   String initMethodName = mbd.getInitMethodName();
   Assert.state(initMethodName != null, "No init method set");
   // 反射获取初始化方法
   Method initMethod = (mbd.isNonPublicAccessAllowed() ?
         BeanUtils.findMethod(bean.getClass(), initMethodName) :
         ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

   if (initMethod == null) {
      if (mbd.isEnforceInitMethod()) {
         throw new BeanDefinitionValidationException("Could not find an init method named '" +
               initMethodName + "' on bean with name '" + beanName + "'");
      }
      else {
         if (logger.isTraceEnabled()) {
            logger.trace("No default init method named '" + initMethodName +
                  "' found on bean with name '" + beanName + "'");
         }
         // Ignore non-existent default lifecycle methods.
         return;
      }
   }

   if (logger.isTraceEnabled()) {
      logger.trace("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
   }
   // 获取接口上面的方法，没有接口则直接返回当前方法
   Method methodToInvoke = ClassUtils.getInterfaceMethodIfPossible(initMethod);

   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         ReflectionUtils.makeAccessible(methodToInvoke);
         return null;
      });
      try {
         AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
               methodToInvoke.invoke(bean), getAccessControlContext());
      }
      catch (PrivilegedActionException pae) {
         InvocationTargetException ex = (InvocationTargetException) pae.getException();
         throw ex.getTargetException();
      }
   }
   else {
      try {
         // 打破方法封装性
         ReflectionUtils.makeAccessible(methodToInvoke);
         // 执行方法
         methodToInvoke.invoke(bean);
      }
      catch (InvocationTargetException ex) {
         throw ex.getTargetException();
      }
   }
}
```

AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization

```java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   // ApplicationContextAwareProcessor不做处理
   // ApplicationListenerDetector不做处理
   // ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor不做处理
   // PostProcessorRegistrationDelegate.BeanPostProcessorChecker不做处理
   // AutowiredAnnotationBeanPostProcessor不做处理--没有重写
   // CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor不做处理
   // 执行实例化之后的后置处理
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessAfterInitialization(result, beanName);
      // current不为空就作为实例化之后的对象返回
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

到这里已接介绍完了bean的初始化，总结一下初始化做了哪些工作：
1.执行各种awareMethod
2.处理@PostConstruct和@PreDestroy（通过InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization）
3.执行初始化方法--bean在注册阶段准备好的方法

