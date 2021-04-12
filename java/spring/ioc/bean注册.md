

# ioc源码解第一篇

## Demo

话不多说，首先附上启动demo，这里是通过配置类的方式编写的demo，所以源码分析也是基于AnnotationConfigApplicationContext进行， 这里用的是spring5+的版本。

```java
public class SpringMain {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(MainConfig.class);
        MainConfig mainConfig = context.getBean(MainConfig.class);
        mainConfig.sayHello();
    }
}
```



## 源码解读

进入AnnotationConfigApplicationContext的构造方法

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   // 初始化上下文环境
   // 初始化注解bean定义解析器和类路径bean定义扫描器
   this();
   register(componentClasses);
   refresh();
}
```

进入this方法，会看到下面2个构造方法执行

```java
public AnnotationConfigApplicationContext() {
   // 创建工厂beanbeanFactory = new DefaultListableBeanFactory()
   // 创建一个资源解析器resourcePatternResolver = new PathMatchingResourcePatternResolver()
   // 创建类加载器classLoader = ClassUtils.getDefaultClassLoader()
   // 创建new StandardEnvironment();
   // 注册7个PostProcessor
   this.reader = new AnnotatedBeanDefinitionReader(this);
   // 初始化ClassPathBeanDefinitionScanner
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

AnnotationConfigApplicationContext类是有继承关系的，在执行AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner之前，会先隐式调用父类的构造方法：

```java
public GenericApplicationContext() {
   this.beanFactory = new DefaultListableBeanFactory();
}
```

> DefaultListableBeanFactory是相当重要的，从字面意思就可以看出它是一个Bean的工厂，用来生产和获得Bean的。


接着进入到AnnotatedBeanDefinitionReader(this)的构造方法中，顺着构造方法的调用链，进入到AnnotatedBeanDefinitionReader(BeanDefinitionRegistry, Environment)方法中

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   Assert.notNull(environment, "Environment must not be null");
   this.registry = registry;
   this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   /**
    * 设置beanFactory中的DependencyComparator和AutowireCandidateResolver属性
    * 注册一些beanFactory内部的注解配置处理器类
    */
   AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

AnnotationConfigUtils.registerAnnotationConfigProcessors的实现方式：

```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      BeanDefinitionRegistry registry, @Nullable Object source) {
   // 获取注册器中维护的DefaultListableBeanFactory工厂
   DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   if (beanFactory != null) {
      if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
         // AnnotationAwareOrderComparator主要能解析@Order注解和@Priority
         beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
      }
      if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
         // ContextAnnotationAutowireCandidateResolver提供处理延迟加载的功能
         beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
      }
   }

   Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
	 // spring注解化配置bean注册
   if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   }
	 // @Autowired/@Value自动注入
   if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   // @Resource自动注入
   // bean初始化阶段处理@PostConstruct和@PreDestroy修饰的方法，并执行@PostConstruct修饰的方法
   if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
   // 支持jpa
   if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition();
      try {
         def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
               AnnotationConfigUtils.class.getClassLoader()));
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
      }
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   }
	 // 处理@EventListener
   if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   }

   return beanDefs;
}
```

> 1. 对beanFactory的属性进行设置
> 2. 注册Spring内部的处理器类(实现BeanDefinitionRegistryPostProcessor接口和BeanFactoryPostProcessor接口), Spring在内部类维护这些类, 为BeanFactory提供特定的功能
> 以下6个类是最先注册到beanFactory中的BeanDefinitionMap集合中去
>        **(1). org.springframework.context.annotation.internalConfigurationAnnotationProcessor(重点)** 
>        (2). org.springframework.context.annotation.internalAutowiredAnnotationProcessor
>        (3). org.springframework.context.annotation.internalCommonAnnotationProcessor
>        (4). org.springframework.context.annotation.internalPersistenceAnnotationProcessor    
>        (5). org.springframework.context.event.internalEventListenerProcessor
>        (6). org.springframework.context.event.internalEventListenerFactory
> 3. 将注册Spring内部的类封装到Set集合中返回

AnnotationConfigUtils#registerPostProcessor

```java
private static BeanDefinitionHolder registerPostProcessor(
      BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

   definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(beanName, definition);
   return new BeanDefinitionHolder(definition, beanName);
}
```

GenericApplicationContext#registerBeanDefinition ==> DefaultListableBeanFactory#registerBeanDefinition实现

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
      throws BeanDefinitionStoreException {

   Assert.hasText(beanName, "Bean name must not be empty");
   Assert.notNull(beanDefinition, "BeanDefinition must not be null");

   if (beanDefinition instanceof AbstractBeanDefinition) {
      try {
         ((AbstractBeanDefinition) beanDefinition).validate();
      }
      catch (BeanDefinitionValidationException ex) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Validation of bean definition failed", ex);
      }
   }

   BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
   if (existingDefinition != null) {
      // bean已经存在
      if (!isAllowBeanDefinitionOverriding()) {
         // 如果对应的BeanName已经注册且在配置中配置了bean不允许被覆盖,则抛出异常
         throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
      }
      // ...此处省略日志代码
      // 注册beanDefinition(将beanDefinition放入map缓存中)
      this.beanDefinitionMap.put(beanName, beanDefinition);
   }
   else {
      if (hasBeanCreationStarted()) {
         // Cannot modify startup-time collection elements anymore (for stable iteration)
         synchronized (this.beanDefinitionMap) {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
            updatedDefinitions.addAll(this.beanDefinitionNames);
            updatedDefinitions.add(beanName);
            this.beanDefinitionNames = updatedDefinitions;
            removeManualSingletonName(beanName);
         }
      }
      else {
         // Still in startup registration phase
         this.beanDefinitionMap.put(beanName, beanDefinition);
         this.beanDefinitionNames.add(beanName);
         removeManualSingletonName(beanName);
      }
      this.frozenBeanDefinitionNames = null;
   }

   if (existingDefinition != null || containsSingleton(beanName)) {
      resetBeanDefinition(beanName);
   }
   else if (isConfigurationFrozen()) {
      clearByTypeCache();
   }
}
```

> 分析:
>
> 注册前的最后一次校验,这里的校验不同于之前的XML文件校验, 主要是对于AbstracBeanDefinition属性中的methodOverrides校验, 校验methodOverrides是否与工厂方法并存或者methodOverrides对应的方法不存在；
> 判断该beanName是否已经注册; 如果beanName已经注册过: 根据不同的设置进行处理(是否允许覆盖, 如果不允许,则抛出异常; 允许的话, 则进行注册)；
> 如果beanName没有被注册过; 则开始检查这个工厂的bean创建阶段是否已经开始，即是否有任何bean被标记为同时创建。如果bean工厂的bean创建已经开始,  那么就不能在beanDefinitionNames集合中添加beanName;  因为创建bean时,  会迭代beanDefinitionNames集合,  获取beanName， 获取BeanDefinition创建bean实例;  一旦开始迭代, 则不允许在向该list集合中插入元素,  但是beanDefinitionMap集合可以,  因为beanDefinitionMap集合并没有被迭代,  而是通过beanName获取BeanDefinition信息;
> 如果该bean定义已经注册, 并且为单例, 则进行重置

回到下面该方法，接下来看查看ClassPathBeanDefinitionScanner的构造方法

```java
public AnnotationConfigApplicationContext() {
   this.reader = new AnnotatedBeanDefinitionReader(this);
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

追踪ClassPathBeanDefinitionScanner构造方法的调用链

```java
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;
    // 注册默认过滤器
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
    // 设置环境
		setEnvironment(environment);
    // 设置类加载器
		setResourceLoader(resourceLoader);
	}
```

这里的做的初始化都是为ClassPathScanningCandidateComponentProvider中的变量进行初始化，会为后面的自动注入

ClassPathScanningCandidateComponentProvider#registerDefaultFilters，该方法将几个带有注解的过滤器Component、javax.annotation.ManagedBean、javax.inject.Named添加到includeFilters变量

```java
protected void registerDefaultFilters() {
   this.includeFilters.add(new AnnotationTypeFilter(Component.class));
   ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
   try {
      this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
      logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
   }
   catch (ClassNotFoundException ex) {
      // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
   }
   try {
      this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
      logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
   }
   catch (ClassNotFoundException ex) {
      // JSR-330 API not available - simply skip.
   }
}
```

回到AnnotationConfigApplicationContext构造方法，接下来看AnnotatedBeanDefinitionReader#register方法

```java
public void register(Class<?>... componentClasses) {
   for (Class<?> componentClass : componentClasses) {
      registerBean(componentClass);
   }
}
```

AnnotatedBeanDefinitionReader#registerBean

```java
public void registerBean(Class<?> beanClass) {
   doRegisterBean(beanClass, null, null, null, null);
}
```

AnnotatedBeanDefinitionReader#doRegisterBean

```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
      @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
      @Nullable BeanDefinitionCustomizer[] customizers) {

   AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
   if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
      return;
   }

   abd.setInstanceSupplier(supplier);
   // 获取bean的作用域
   ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
   abd.setScope(scopeMetadata.getScopeName());
   // 获取bean名称
   String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
   // 处理一些通用的注解属性Lazy、Primary、DependsOn、Role、Description
   AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
   if (qualifiers != null) {
      for (Class<? extends Annotation> qualifier : qualifiers) {
         if (Primary.class == qualifier) {
            abd.setPrimary(true);
         }
         else if (Lazy.class == qualifier) {
            abd.setLazyInit(true);
         }
         else {
            abd.addQualifier(new AutowireCandidateQualifier(qualifier));
         }
      }
   }
   if (customizers != null) {
      for (BeanDefinitionCustomizer customizer : customizers) {
         customizer.customize(abd);
      }
   }

   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
   // 代理模式
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   // 注册bean
   BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

> 定义bean的一些属性并对bean进行注册

接下来看如何获取bean的作用域AnnotationScopeMetadataResolver#resolveScopeMetadata

```java
@Override
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
   // new一个ScopeMetadata，默认是单例模式，默认的代理模式是ScopedProxyMode.NO
   ScopeMetadata metadata = new ScopeMetadata();
   if (definition instanceof AnnotatedBeanDefinition) {
      AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
      // 获取@Scope上面所有属性
      AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
            annDef.getMetadata(), this.scopeAnnotationType);
      if (attributes != null) {
         metadata.setScopeName(attributes.getString("value"));
         ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
         if (proxyMode == ScopedProxyMode.DEFAULT) {
            // 如果代理模式是ScopedProxyMode.DEFAULT就变为ScopedProxyMode.NO
            proxyMode = this.defaultProxyMode;
         }
         metadata.setScopedProxyMode(proxyMode);
      }
   }
   return metadata;
}
```

bean名称获取AnnotationBeanNameGenerator#generateBeanName

```java
public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
   if (definition instanceof AnnotatedBeanDefinition) {
      // 如果是注解的定义的方式
      String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);
      if (StringUtils.hasText(beanName)) {
         // Explicit bean name found.
         return beanName;
      }
   }
   // Fallback: generate a unique default bean name.
   return buildDefaultBeanName(definition, registry);
}
```

首先会通过AnnotationBeanNameGenerator#determineBeanNameFromAnnotation去获取bean名称

```java
/**
 * 该方法会获取@Conponent或@ManagedBean或@Named注解的value值返回
 * 不存在注解或不存在value值就会返回空
 */
@Nullable
protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {
   AnnotationMetadata amd = annotatedDef.getMetadata();
   // 获取所有注解的类型
   Set<String> types = amd.getAnnotationTypes();
   String beanName = null;
   for (String type : types) {
      // 获取注解上的所有属性
      AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type);
      if (attributes != null) {
         Set<String> metaTypes = this.metaAnnotationTypesCache.computeIfAbsent(type, key -> {
            // 获取该注解上的其它直接注解
            Set<String> result = amd.getMetaAnnotationTypes(key);
            return (result.isEmpty() ? Collections.emptySet() : result);
         });
         // 如果有@Conponent或@ManagedBean或@Named，且拥有value属性
         if (isStereotypeWithNameValue(type, metaTypes, attributes)) {
            Object value = attributes.get("value");
            if (value instanceof String) {
               String strVal = (String) value;
               if (StringUtils.hasLength(strVal)) {
                  if (beanName != null && !strVal.equals(beanName)) {
                       // 如果拥有多个@Conponent或@ManagedBean或@Named注解， 且value值不一样就会抛出异常
                     throw new IllegalStateException("Stereotype annotations suggest inconsistent " +
                           "component names: '" + beanName + "' versus '" + strVal + "'");
                  }
                  beanName = strVal;
               }
            }
         }
      }
   }
   return beanName;
}
```

若果没有获取到就会调用AnnotationBeanNameGenerator#buildDefaultBeanName去获取bean名称

```java
/**
 * 根绝bean的定义去获取一个默认的名称
 * mypackage.MyJdbcDao -> myJdbcDao
 */
protected String buildDefaultBeanName(BeanDefinition definition) {
   String beanClassName = definition.getBeanClassName();
   Assert.state(beanClassName != null, "No bean class name set");
   // 获取不带包的类名称
   String shortClassName = ClassUtils.getShortName(beanClassName);
   // 首字母变为小写
   return Introspector.decapitalize(shortClassName);
}
```

> 这里就会默认创建bean的名称

接下来看bean的注册

```java
public static void registerBeanDefinition(
      BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
      throws BeanDefinitionStoreException {

   // Register bean definition under primary name.
   String beanName = definitionHolder.getBeanName();
   // 注册bean
   registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

   // Register aliases for bean name, if any.
   // 注册bean的相关别名
   String[] aliases = definitionHolder.getAliases();
   if (aliases != null) {
      for (String alias : aliases) {
         registry.registerAlias(beanName, alias);
      }
   }
}
```

> 进入到refresh方法前一共注册了6个后置处理器（如果包含jpa就是7个），加上一个启动类



下面看核心方法**refresh()**

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

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
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

首先看AbstractApplicationContext#prepareRefresh方法

```java
protected void prepareRefresh() {
   // Switch to active.
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   if (logger.isDebugEnabled()) {
      if (logger.isTraceEnabled()) {
         logger.trace("Refreshing " + this);
      }
      else {
         logger.debug("Refreshing " + getDisplayName());
      }
   }

   // Initialize any placeholder property sources in the context environment.
   // 初始化一些属性（如果有需要，重写该方法），默认是没有实现的
   initPropertySources();

   // Validate that all properties marked as required are resolvable:
   // see ConfigurablePropertyResolver#setRequiredProperties
   // 这里会environment = new StandardEnvironment()创建一个默认的环境
   // 验证一些必要属性？？？
   getEnvironment().validateRequiredProperties();

   // Store pre-refresh ApplicationListeners...
   // 保存refresh()方法执行前的ApplicationListeners
   if (this.earlyApplicationListeners == null) {
      // 这里可以定义一些监听器
      this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
   }
   else {
      // Reset local application listeners to pre-refresh state.
      // 重置本地应用监听到refresh()前的状态
      this.applicationListeners.clear();
      // 添加一些早期监听器
      this.applicationListeners.addAll(this.earlyApplicationListeners);
   }

   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

接下来看obtainFreshBeanFactory#obtainFreshBeanFactory

```java
/*
 * obtainFreshBeanFactory()方法中根据实现类的不同调用不同的refreshBeanFactory()方法
 * <-----  注解模式  ----->
 * 1. 如果是使用AnnotationConfigApplicationContext来初始化环境
 * 调用的是GenericApplicationContext#refreshBeanFactory()方法
 * {@link GenericApplicationContext#refreshBeanFactory()}
 * 在该方法中只是对beanFactory的一些变量进行设置
 * <----- XML配置模式  ----->
 * 2. 如果是使用ClassPathXmlApplicationContext来初始化环境
 * 调用的是{@link AbstractRefreshableApplicationContext#refreshBeanFactory()}
 * 由于还没有对beanFactory进行初始化,所以在该方法中,完成了对beanFactory的初始化操作
 * 并对设置的资源位置进行扫描, 解析
 * 注意:此方法是使用ClassPathXmlApplicationContext来初始化上下文是解析注册bean的重要入口
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   refreshBeanFactory();
   // 返回DefaultListableBeanFactory
   return getBeanFactory();
}
```

这里只对注解的形式进行分析调用的是GenericApplicationContext#refreshBeanFactory

```java
/**
 * 1.首先refreshed的状态变为true
 * 2.设置序列化id
 */
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
   if (!this.refreshed.compareAndSet(false, true)) {
      throw new IllegalStateException(
            "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
   }
   this.beanFactory.setSerializationId(getId());
}
```

DefaultListableBeanFactory#setSerializationId

```java
/**
 * 1.把当前的BeanFactory对象用弱引用以serializationId为key存起来
 * 2.存在旧的序列化id就移除旧的BeanFactory对象
 */
public void setSerializationId(@Nullable String serializationId) {
   if (serializationId != null) {
      serializableFactories.put(serializationId, new WeakReference<>(this));
   }
   else if (this.serializationId != null) {
      serializableFactories.remove(this.serializationId);
   }
   this.serializationId = serializationId;
}
```

> 官方的注释说明是： 
>
> Specify an id for serialization purposes, allowing this BeanFactory to be
> deserialized from this id back into the BeanFactory object, if needed.（指定一个序列化的id， 如果需要的话，允许将该BeanFactory从该ID反序列化回BeanFactory对象）

直接返回beanFactory

```java
@Override
public final ConfigurableListableBeanFactory getBeanFactory() {
   return this.beanFactory;
}
```

AbstractApplicationContext#prepareBeanFactory对beanFactory设置一直属性

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // Tell the internal bean factory to use the context's class loader etc.
   // bean加载器
   beanFactory.setBeanClassLoader(getClassLoader());
   // spring的Spel表达式解析器
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   // 为beanfactory增加了一个默认的属性编辑器,这个主要是对bean的属性等设置管理的一个工具 
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // Configure the bean factory with context callbacks.
   // 给beanFactory添加后置处理器
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   // 忽略的几个接口
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // Register early post-processor for detecting inner beans as ApplicationListeners.
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // Detect a LoadTimeWeaver and prepare for weaving, if found.
   // //增加对AspectJ的支持
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // Register default environment beans.
   // 注册默认环境bean
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   // 注册系统属性
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   // 注册系统环境
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   /*
    * getBeanFactoryPostProcessors()方法:
    * 获取通过{@link AbstractApplicationContext#addBeanFactoryPostProcessor(BeanFactoryPostProcessor)}
    * 方法添加的实现BeanDefinitiopostProcessBeanFactoryinvokeBeanFactoryPostProcessorsnRegistryPostProcessor接口和BeanFactoryPostProcessor接口的beanFactory处理器类
    *
    * 注意: 如果是自定义的添加@Component注解的,此处是获取不到的, 后面Spring会自动去扫描
    *       如果此处存在自定义的类,则会在ConfigurationClassPostProcessor#processConfigBeanDefinitions()方法之前执行
    */
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

   // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   // 检测LoadTimeWeaver并准备编织（如果在此期间发现）
   // (e.g. 通过ConfigurationClassPostProcessor注册的@Bean方法)
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors这个方法有点长，逻辑其实也不是很复杂，搞清楚这个方法就能明白BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor的用法了

```java

public static void invokeBeanFactoryPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

   // Invoke BeanDefinitionRegistryPostProcessors first, if any.
   // 定一个Set来标示已经被处理的bean的名称（这里的所有的bean都是实现了BeanDefinitionRegistryPostProcessor或
   // BeanFactoryPostProcessor接口），该参数将会用在：获取实现了BeanFactoryPostProcessord bean的时候会用该参数排除
   // BeanDefinitionRegistryPostProcessor类型的bean
   Set<String> processedBeans = new HashSet<>();
   // 判断是否实现了BeanDefinitionRegistry接口
   if (beanFactory instanceof BeanDefinitionRegistry) {
      BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
      /*
       * 首先会处理入参中的beanFactoryPostProcessors；
       * beanFactoryPostProcessors参数中可能有的实现了BeanFactoryPostProcessor或BeanDefinitionRegistryPostProcessor
       * 1.定义两个类型的集合变量
       * 2.循环逻辑就是为了把这种区分开来并放入不同的集合，如果是BeanDefinitionRegistryPostProcessor类型的参数，就先执行      				* postProcessBeanDefinitionRegistry()方法，并添加到registryProcessors中，后面统一进行执行
       */
      List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
      List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

      for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
         if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                  (BeanDefinitionRegistryPostProcessor) postProcessor;
            // 
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            registryProcessors.add(registryProcessor);
         }
         else {
            regularPostProcessors.add(postProcessor);
         }
      }

      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the bean factory post-processors apply to them!
      // Separate between BeanDefinitionRegistryPostProcessors that implement
      // PriorityOrdered, Ordered, and the rest.
      /*
       * 下面一段逻辑分了四个步骤
       * 1.优先执行实现了PriorityOrdered接口BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
       * 2.执行实现了Ordered接口BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
       * 3.执行上面两者（PriorityOrdered、Ordered）都没有实现的
       * BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
       * 4.添加到registryProcessors中，后面统一进行执行BeanFactoryPostProcessor#postProcessBeanFactory
       */
      List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

      // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
      // 首先执行实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessor
      // 这里会有一个org.springframework.context.annotation.internalConfigurationAnnotationProcessor（可以回顾最开始的时候添加的几个bean）---重点关注
      String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            // 这会拿到ConfigurationClassPostProcessor这个对象
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
      // 然后执行实现了Ordered接口的BeanDefinitionRegistryPostProcessor
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      // 最后执行其他接口（两者都没有实现）的BeanDefinitionRegistryPostProcessor
      boolean reiterate = true;
      while (reiterate) {
         reiterate = false;
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
         for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName)) {
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
               reiterate = true;
            }
         }
         sortPostProcessors(currentRegistryProcessors, beanFactory);
         registryProcessors.addAll(currentRegistryProcessors);
         invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
         currentRegistryProcessors.clear();
      }

      // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
      // 统一进行执行BeanFactoryPostProcessor#postProcessBeanFactory
      invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
      invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
   }

   else {
      // Invoke factory processors registered with the context instance.
      invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let the bean factory post-processors apply to them!
   String[] postProcessorNames =
         beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

   // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   /*
    * 将实现PriorityOrdered，Ordered的BeanFactoryPostProcessor与其他的分开，
    * 按照先后顺序执行（PriorityOrdered、Ordered、都没有实现的）BeanFactoryPostProcessor#postProcessBeanFactory
    */
   List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (processedBeans.contains(ppName)) {
         // skip - already processed in first phase above
      }
      else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
   // 先执行实现了PriorityOrdered类型的BeanFactoryPostProcessors
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
   // 执行实现了Ordered类型的BeanFactoryPostProcessors
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // Finally, invoke all other BeanFactoryPostProcessors.
   // 执行实现了都没有实现的BeanFactoryPostProcessors
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
   for (String postProcessorName : nonOrderedPostProcessorNames) {
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

   // Clear cached merged bean definitions since the post-processors might have
   // modified the original metadata, e.g. replacing placeholders in values...
   // 清除缓存的合并bean定义，因为后处理器可能有修改原始元数据，例如替换值中的占位符…
   beanFactory.clearMetadataCache();
}
```

> 总结一下postProcessBeanDefinitionRegistry和postProcessBeanFactory方法的执行优先级
>
> getBeanFactoryPostProcessors<BeanDefinitionRegistryPostProcessor>()#postProcessBeanDefinitionRegistry ->
>
> <T extends BeanDefinitionRegistryPostProcessor & PriorityOrdered>#postProcessBeanDefinitionRegistry ->
>
> <T extends BeanDefinitionRegistryPostProcessor & Ordered>#postProcessBeanDefinitionRegistry -> 
>
> <T extends BeanDefinitionRegistryPostProcessor>#postProcessBeanDefinitionRegistry -> 
>
> getBeanFactoryPostProcessors<BeanDefinitionRegistryPostProcessors>()#postProcessBeanFactory ->
>
> <T extends BeanDefinitionRegistryPostProcessor & PriorityOrdered>#postProcessBeanFactory ->
>
> <T extends BeanDefinitionRegistryPostProcessor & Ordered>#postProcessBeanFactory -> 
>
> <T extends BeanDefinitionRegistryPostProcessor>#postProcessBeanFactory -> 
>
> getBeanFactoryPostProcessors<BeanFactoryPostProcessor>()#postProcessBeanFactory ->
>
> <T extends BeanFactoryPostProcessor & PriorityOrdered>#postProcessBeanFactory ->
>
> <T extends BeanFactoryPostProcessor & Ordered>#postProcessBeanFactory -> 
>
> <T extends BeanFactoryPostProcessor>#postProcessBeanFactory -> 

PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors实现

```java
private static void invokeBeanDefinitionRegistryPostProcessors(Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

   for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
      /**
       * <-----  注解模式下   ----->
       * 当postProcessor为ConfigurationClassPostProcessor时, 执行
       * {@link org.springframework.context.annotation.ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry}
       * 完成了注解类中的扫描和注册
       */
      postProcessor.postProcessBeanDefinitionRegistry(registry);
   }
}
```

PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors

```java
private static void invokeBeanFactoryPostProcessors(
      Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
	 // 循环执行
   for (BeanFactoryPostProcessor postProcessor : postProcessors) {
      postProcessor.postProcessBeanFactory(beanFactory);
   }
}
```

> 这里先对BeanFactoryPostProcessor做一个简单的总结：
>
> #### BeanFactoryPostProcessor
>
>   BeanFactoryPostProcessor是Spring中一个重要的接口，依赖该接口可以实现对Spring中bean工厂中的beandefinition（未实例化）数据属性的修改。接口定义如下：
>
> ```java
> public interface BeanFactoryPostProcessor {
>     //在初始化之后修改应用程序上下文的内部bean工厂。所有bean定义都已加载，实例化bean之前，可以覆盖或添加属性
>     void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
> }
> ```
>
> 官方文档对接口的说明如下：
>
> - 允许自定义修改应用程序上下文（容器）的未实例化bean的定义，修改上下文的底层bean工厂的bean属性值。
> - 应用程序上下文可以在其bean定义中自动检测BeanFactoryPostProcessor bean，并在创建任何其他bean之前应用它们。
> - BeanFactoryPostProcessor可以与bean定义交互并修改bean定义，但从不与bean实例交互。这样做可能会导致过早的bean实例化，破坏容器并导致意外的副作用。如果需要bean实例交互，考虑实BeanPostProcessor。
>
> 也就是说这个接口的作用是在容器中的bean实例化之前，对加载进容器的bean进行一些属性的修改。
>
> #### BeanDefinitionRegistryPostProcessor
>
>   BeanFactoryPostProcessor接口有一个子接口BeanDefinitionRegistryPostProcessor，BeanDefinitionRegistryPostProcessor实现了BeanFactoryPostProcessor，同时扩展了这个接口，其定义如下：
>
> ```java
> public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor { 
> 	  //在应用程序上下文中的bean注册器实例化之后可以对其进行修改，这可以向bean注册器中添加更多的bean定义 
>     void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException; 
> }
> ```
>
>  官方文档对于这个接口的说明如下：
>
> - 扩展了标准BeanFactoryPostProcessor SPI，允许在常规BeanFactoryPostProcessor postProcessBeanFactory方法执行之前向容器中注册更多的bean定义。特别是，BeanDefinitionRegistryPostProcessor可以注册更多的bean定义，这些定义反过来又定义了BeanFactoryPostProcessor实例。
>
>  BeanDefinitionRegistryPostProcessor作为BeanFactoryPostProcessor的子接口，其postProcessBeanDefinitionRegistry方法会在所有BeanFactoryPostProcessor post-processing前执行。因此
>
>  BeanDefinitionRegistryPostProcessor这个接口用于向容器中注册bean定义。
>



#### 注解bean注册

ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry这里就正式开始介绍整个关于注解bean注册的顺序了

```java
/**
 * 从注册表中的配置类派生其他Bean定义
 */
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
   int registryId = System.identityHashCode(registry);
   if (this.registriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
   }
   if (this.factoriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
   }
   this.registriesPostProcessed.add(registryId);

   processConfigBeanDefinitions(registry);
}
```

**ConfigurationClassPostProcessor#processConfigBeanDefinitions重点方法**

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
   List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
   /**
	  * 获取到的bean
	  * 0 = "org.springframework.context.annotation.internalConfigurationAnnotationProcessor"
		* 1 = "org.springframework.context.annotation.internalAutowiredAnnotationProcessor"
		* 2 = "org.springframework.context.annotation.internalCommonAnnotationProcessor"
		* 3 = "org.springframework.context.event.internalEventListenerProcessor"
		* 4 = "org.springframework.context.event.internalEventListenerFactory"
		* 5 = "abc"
	  */
   String[] candidateNames = registry.getBeanDefinitionNames();

   for (String beanName : candidateNames) {
      BeanDefinition beanDef = registry.getBeanDefinition(beanName);
      // 如果BeanDefinition中的configurationClass属性为full或者lite,则意味着已经处理过了,直接跳过(Spring内部的BeanDefinition会直接跳过)
      if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
         if (logger.isDebugEnabled()) {
            logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
         }
      }
      // 只要是含有@Configuration @Component @Controller @Service @Repository @ComponentScan @Import @ImportResource或者方法中含有@Bean，就会被添加到configCandidates
      else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
         configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
      }
   }

   // Return immediately if no @Configuration classes were found
   // 启动类如果没有找到带有@Configuration @Component @Controller @Service @Repository @ComponentScan @Import @ImportResource中的任何一个就直接返回
   if (configCandidates.isEmpty()) {
      return;
   }

   // Sort by previously determined @Order value, if applicable
   // 如果有@Order注解就按给定的Order注解上的值升序排列，缺省的排在后面
   configCandidates.sort((bd1, bd2) -> {
      int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
      int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
      return Integer.compare(i1, i2);
   });

   // Detect any custom bean name generation strategy supplied through the enclosing application context
   //检测通过封装的应用程序上下文提供的任何自定义bean名称生成策略
   SingletonBeanRegistry sbr = null;
   // 如果BeanDefinitionRegistry是SingletonBeanRegistry子类的话,
   // 由于我们当前传入的是DefaultListableBeanFactory,是SingletonBeanRegistry的子类, 因此会将registry强转为SingletonBeanRegistry
   if (registry instanceof SingletonBeanRegistry) {
      sbr = (SingletonBeanRegistry) registry;
      if (!this.localBeanNameGeneratorSet) {
         //SingletonBeanRegistry中有id为 org.springframework.context.annotation.internalConfigurationBeanNameGenerator
         //如果有则利用他的，否则则是spring默认的
         BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
               AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
         if (generator != null) {
            this.componentScanBeanNameGenerator = generator;
            this.importBeanNameGenerator = generator;
         }
      }
   }

   if (this.environment == null) {
      this.environment = new StandardEnvironment();
   }

   // Parse each @Configuration class
   // 循环解析每个配置类
   // 初始化ConfigurationClassParser解析器, 解析每个注解类
   ConfigurationClassParser parser = new ConfigurationClassParser(
         this.metadataReaderFactory, this.problemReporter, this.environment,
         this.resourceLoader, this.componentScanBeanNameGenerator, registry);

   Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
   // alreadyParsed用于存储已处理过的
   Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
   do {
      // 到这里已经完成所有bean的解析，以及部分bean的注册工作
      parser.parse(candidates);
      // 进行校验
      parser.validate();

      Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
      configClasses.removeAll(alreadyParsed);

      // Read the model and create bean definitions based on its content
      if (this.reader == null) {
         this.reader = new ConfigurationClassBeanDefinitionReader(
               registry, this.sourceExtractor, this.resourceLoader, this.environment,
               this.importBeanNameGenerator, parser.getImportRegistry());
      }
      this.reader.loadBeanDefinitions(configClasses);
      alreadyParsed.addAll(configClasses);
      // 对candidates进行clear()
      candidates.clear();
      /**
       * candidateNames是我们最开始的几个bean，即一些系统最开始加载的几个bean，还包括一个配置主类internalCommonAnnotationProcessor、internalEventListenerProcessor、internalEventListenerFactory、internalConfigurationAnnotationProcessor、internalAutowiredAnnotationProcessor、helloConfiguration
       * 而registry.getBeanDefinitionNames()所得到的bean就是所有已经注册过了的
       */
      if (registry.getBeanDefinitionCount() > candidateNames.length) {
         String[] newCandidateNames = registry.getBeanDefinitionNames();
         Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
         Set<String> alreadyParsedClasses = new HashSet<>();
         for (ConfigurationClass configurationClass : alreadyParsed) {
            alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
         }
         for (String candidateName : newCandidateNames) {
            if (!oldCandidateNames.contains(candidateName)) {
               BeanDefinition bd = registry.getBeanDefinition(candidateName);
               if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                     !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                  candidates.add(new BeanDefinitionHolder(bd, candidateName));
               }
            }
         }
         candidateNames = newCandidateNames;
      }
   }
   while (!candidates.isEmpty());

   // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
   if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
      sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
   }

   if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
      // Clear cache in externally provided MetadataReaderFactory; this is a no-op
      // for a shared cache since it'll be cleared by the ApplicationContext.
      ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
   }
}
```

如何判断是否属于配置类呢？

ConfigurationClassUtils#checkConfigurationClassCandidate这个方法就是找配置类，如果找到类上面有@Configuration @Component（只要是含有@Component注解的--@Controller @Service @Repository @Configuration） @ComponentScan @Import @ImportResource其中的一个注解就返回true，这里会过滤掉前面已经被注册过的dorg.springframework.context.annotation.internalConfigurationAnnotationProcessor、org.springframework.context.annotation.internalAutowiredAnnotationProcessor、org.springframework.context.annotation.internalCommonAnnotationProcessor、org.springframework.context.event.internalEventListenerProcessor、org.springframework.context.event.internalEventListenerFactory

```java
public static boolean checkConfigurationClassCandidate(
      BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

   String className = beanDef.getBeanClassName();
   if (className == null || beanDef.getFactoryMethodName() != null) {
      return false;
   }

   AnnotationMetadata metadata;
   // 如果是我们的配置类
   if (beanDef instanceof AnnotatedBeanDefinition &&
         className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
      // Can reuse the pre-parsed metadata from the given BeanDefinition...
      metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
   }
   else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
      // Check already loaded Class if present...
      // since we possibly can't even load the class file for this Class.
      Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
      // 等于BeanFactoryPostProcessor、BeanPostProcessor、AopInfrastructureBean、EventListenerFactory，或者为beanClass的父类或者超类
      // 之前注册的PostProcessor会在这里被过滤
      if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
            BeanPostProcessor.class.isAssignableFrom(beanClass) ||
            AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
            EventListenerFactory.class.isAssignableFrom(beanClass)) {
         return false;
      }
      metadata = AnnotationMetadata.introspect(beanClass);
   }
   else {
      try {
         MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
         metadata = metadataReader.getAnnotationMetadata();
      }
      catch (IOException ex) {
         if (logger.isDebugEnabled()) {
            logger.debug("Could not find class file for introspecting configuration annotations: " +
                  className, ex);
         }
         return false;
      }
   }
   Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
   // 判断是否是有@Configuration注解
   if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
      beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
   }
   // 判断是否含下面这写注解@Component（@Configuration @Controller @Service @Repository）、@ComponentScan、@Import、@ImportResource 或者 含有带@Bean方法的类
   else if (config != null || isConfigurationCandidate(metadata)) {
      beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
   }
   else {
      return false;
   }

   // It's a full or lite configuration candidate... Let's determine the order value, if any.
   Integer order = getOrder(metadata);
   if (order != null) {
      beanDef.setAttribute(ORDER_ATTRIBUTE, order);
   }

   return true;
}
```

如果找到配置类，下面就开始执行ConfigurationClassParser#parse进行正式的处理

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
   for (BeanDefinitionHolder holder : configCandidates) {
      BeanDefinition bd = holder.getBeanDefinition();
      /**
       * 根据不同的类型去解析
       */
      try {
         if (bd instanceof AnnotatedBeanDefinition) {
            // 因为我们以注解的形式开启的，会执行到这里
            parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
         }
         else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
            parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
         }
         else {
            parse(bd.getBeanClassName(), holder.getBeanName());
         }
      }
      catch (BeanDefinitionStoreException ex) {
         throw ex;
      }
      catch (Throwable ex) {
         throw new BeanDefinitionStoreException(
               "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
      }
   }

   this.deferredImportSelectorHandler.process();
}
```

ConfigurationClassParser#parse(AnnotationMetadata, String)

```java
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
   processConfigurationClass(new ConfigurationClass(metadata, beanName), DEFAULT_EXCLUSION_FILTER);
}
```

ConfigurationClassParser#processConfigurationClass

```java
protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
   // 是否跳过不进行处理
   if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
      return;
   }

   ConfigurationClass existingClass = this.configurationClasses.get(configClass);
   if (existingClass != null) {
      if (configClass.isImported()) {
         if (existingClass.isImported()) {
            existingClass.mergeImportedBy(configClass);
         }
         // 注册过了就跳过
         // Otherwise ignore new imported config class; existing non-imported class overrides it.
         return;
      }
      else {
         // Explicit bean definition found, probably replacing an import.
         // Let's remove the old one and go with the new one.
         // 重复的bean找到
         // 替换新的
         this.configurationClasses.remove(configClass);
         this.knownSuperclasses.values().removeIf(configClass::equals);
      }
   }

   // Recursively process the configuration class and its superclass hierarchy.
   // 递归处理配置类及其超类层次结构
   SourceClass sourceClass = asSourceClass(configClass, filter);
   // 先处理当前类，然后循环对父类进行处理
   do {
      // 处理配置类，返回值为父类
      sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
   }
   while (sourceClass != null);

   this.configurationClasses.put(configClass, configClass);
}
```

这里就是处理配置类的核心逻辑**ConfigurationClassParser#doProcessConfigurationClass重点**

```java
@Nullable
protected final SourceClass doProcessConfigurationClass(
      ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
      throws IOException {
   // 必须是@Controller @Service @Repository @Configuration @Component类型才对内部类进行解析
   // (@Controller @Service @Repository @Configuration上含有@Component注解)
   if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
      // Recursively process any member (nested) classes first
      // 递归处理内部类
      // 即内部类上面有
      processMemberClasses(configClass, sourceClass, filter);
   }

   // Process any @PropertySource annotations
   // 处理所有带有@PropertySource注解
   for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), PropertySources.class,
         org.springframework.context.annotation.PropertySource.class)) {
      if (this.environment instanceof ConfigurableEnvironment) {
         processPropertySource(propertySource);
      }
      else {
         logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
               "]. Reason: Environment must implement ConfigurableEnvironment");
      }
   }

   // Process any @ComponentScan annotations
   // 处理所有@ComponentScan、@ComponentScans注解
   Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
   if (!componentScans.isEmpty() &&
         !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
      for (AnnotationAttributes componentScan : componentScans) {
         // The config class is annotated with @ComponentScan -> perform the scan immediately
         // 如果这个配置类是加了@ComponentScan注解的, 则立刻执行扫描工作
         /*
          *  对@ComponentScan中配置的包路径信息进行扫描, 并注册扫描出来的BeanDefinition, 最后将所有扫描出来的BeanDefinition信息返回
          */
         Set<BeanDefinitionHolder> scannedBeanDefinitions =
               this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
         // Check the set of scanned definitions for any further config classes and parse recursively if needed
         for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
            BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
            if (bdCand == null) {
               bdCand = holder.getBeanDefinition();
            }
            // 判断是否是配置类
            // 这里的判断和ClassPathScanningCandidateComponentProvider.findCandidateComponents(basePackage)中的判断有点不太一样
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
               // 进行配置类的
               parse(bdCand.getBeanClassName(), holder.getBeanName());
            }
         }
      }
   }

   // Process any @Import annotations
   // 处理类上面的所有@Import注解(包括如果注解中含有@Import注解，这里也会被getImports()解析进去)
   processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

   // Process any @ImportResource annotations
   // 处理所有@ImportResource注解
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
   // 解析接口上带有注解Bean的方法，这里获取当前类上面的@Bean方法以及实现接口上面的@Bean方法
   // 注意: 所继承的类的@Bean方法这里是无法获取的
   // Process individual @Bean methods
   Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
   for (MethodMetadata methodMetadata : beanMethods) {
      configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
   }

   // Process default methods on interfaces
   // 处理接口上面default且带有@Bean的方法（addBeanMethod）
   // 这里只处理该类实现的接口！
   processInterfaces(configClass, sourceClass);

   // Process superclass, if any
   // 如果有父类就返回
   if (sourceClass.getMetadata().hasSuperClass()) {
      String superclass = sourceClass.getMetadata().getSuperClassName();
      if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
         this.knownSuperclasses.put(superclass, configClass);
         // Superclass found, return its annotation metadata and recurse
         return sourceClass.getSuperClass();
      }
   }

   // No superclass -> processing is complete
   // 没有父类就返回为空
   return null;
}
```



#### 内部类处理

先来看处理内部类的这段逻辑

```java
/**
 * @Configuration @Component（只要是含有@Component注解的--@Controller @Service @Repository @Configuration） @ComponentScan @Import @ImportResource，也就是说如果配置类上含有上面的其中的一个注解就会递归就行处理
 * 这里其实没有内部类进行注册
 */
private void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass,
      Predicate<String> filter) throws IOException {
   // 获取内部类
   Collection<SourceClass> memberClasses = sourceClass.getMemberClasses();
   if (!memberClasses.isEmpty()) {
      List<SourceClass> candidates = new ArrayList<>(memberClasses.size());
      for (SourceClass memberClass : memberClasses) {
         // @Configuration @Component（只要是含有@Component注解的--@Controller @Service @Repository @Configuration） @ComponentScan @Import @ImportResource 或者 含有带@Bean方法类(调用出不含这个条件)
         if (ConfigurationClassUtils.isConfigurationCandidate(memberClass.getMetadata()) &&
               !memberClass.getMetadata().getClassName().equals(configClass.getMetadata().getClassName())) {
            candidates.add(memberClass);
         }
      }
      // 排序
      OrderComparator.sort(candidates);
      for (SourceClass candidate : candidates) {
         // importStack不包含
         if (this.importStack.contains(configClass)) {
            this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
         }
         else {
            this.importStack.push(configClass);
            try {
               // 递归调用处理内部类
               processConfigurationClass(candidate.asConfigClass(configClass), filter);
            }
            finally {
               this.importStack.pop();
            }
         }
      }
   }
}
```



#### @ComponentScan注解处理

如果含有@ComponentScan、@ComponentScans就会执行AnnotationParser#parse

```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
   ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
         componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
   // 初始化bean名称生成器
   Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
   boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
   scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
         BeanUtils.instantiateClass(generatorClass));
   // scopedProxy
   ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
   if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
      scanner.setScopedProxyMode(scopedProxyMode);
   }
   else {
      Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
      scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
   }
	 // resourcePattern
   scanner.setResourcePattern(componentScan.getString("resourcePattern"));
   for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
      for (TypeFilter typeFilter : typeFiltersFor(filter)) {
         scanner.addIncludeFilter(typeFilter);
      }
   }
   for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
      for (TypeFilter typeFilter : typeFiltersFor(filter)) {
         scanner.addExcludeFilter(typeFilter);
      }
   }
	 // lazyInit
   boolean lazyInit = componentScan.getBoolean("lazyInit");
   if (lazyInit) {
      scanner.getBeanDefinitionDefaults().setLazyInit(true);
   }
	 // basePackages的包、basePackageClasses类的的包、当前类的包添加到basePackages中
   Set<String> basePackages = new LinkedHashSet<>();
   String[] basePackagesArray = componentScan.getStringArray("basePackages");
   for (String pkg : basePackagesArray) {
      String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
            ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      Collections.addAll(basePackages, tokenized);
   }
   for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
      basePackages.add(ClassUtils.getPackageName(clazz));
   }
   if (basePackages.isEmpty()) {
      basePackages.add(ClassUtils.getPackageName(declaringClass));
   }

   scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
      @Override
      protected boolean matchClassName(String className) {
         return declaringClass.equals(className);
      }
   });
   // 
   return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

ClassPathBeanDefinitionScanner#doScan

```java
/**
 * 在指定的基本程序包中执行扫描，返回已注册的bean定义
 */
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      // 获取所有的扫描出来的组件
      // 带有@Controller @Service @Configuration @Repository @Component的注解的类才会被扫描进去
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         // 获取Scope属性
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         // 根据Bean信息生成BeanName
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            // 处理@Lazy @Primary @DependsOn @Role @Description
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         // 判断注册器中是否已经存在该BeanName
         /**
          * 下面三种情况不会重复进注册
          * 1.显示覆盖（非自动扫描注册） 2.扫描了相同的文件2次 3.扫描等效类两次
          * 
          * 扫描出来的类所产生的bean名称已经被注册成bean，且不在上面三种情况中就会抛出异常
          */
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            // 代理模式默认为NO
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            // 注册bean
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

ClassPathScanningCandidateComponentProvider#findCandidateComponents

```java
/**
 * 扫描@Controller @Service @Configuration @Repository @Component注解的配置类
 */
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
   if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
      return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
   }
   else {
      return scanCandidateComponents(basePackage);
   }
}
```

ClassPathScanningCandidateComponentProvider#scanCandidateComponents

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
   Set<BeanDefinition> candidates = new LinkedHashSet<>();
   try {
      // 包路径进行处理
      String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
      // 读取packageSearchPath包下的所有.class类
      Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
      boolean traceEnabled = logger.isTraceEnabled();
      boolean debugEnabled = logger.isDebugEnabled();
      for (Resource resource : resources) {
         if (traceEnabled) {
            logger.trace("Scanning " + resource);
         }
         if (resource.isReadable()) {
            try {
               MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
               // 判断是否为组件候选者
               if (isCandidateComponent(metadataReader)) {
                  ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                  sbd.setSource(resource);
                  if (isCandidateComponent(sbd)) {
                     if (debugEnabled) {
                        logger.debug("Identified candidate component class: " + resource);
                     }
                     candidates.add(sbd);
                  }
                  else {
                     if (debugEnabled) {
                        logger.debug("Ignored because not a concrete top-level class: " + resource);
                     }
                  }
               }
               else {
                  if (traceEnabled) {
                     logger.trace("Ignored because not matching any filter: " + resource);
                  }
               }
            }
            catch (Throwable ex) {
               throw new BeanDefinitionStoreException(
                     "Failed to read candidate component class: " + resource, ex);
            }
         }
         else {
            if (traceEnabled) {
               logger.trace("Ignored because not readable: " + resource);
            }
         }
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
   }
   return candidates;
}
```

> 被扫描出来的内部类不会进行处理

ClassPathScanningCandidateComponentProvider#isCandidateComponent

```java
/**
 * 默认判断是否是@Component类型的组件（@Controller @Service @Configuration @Repository @Component）
 * 如果有指定includeFilters这里也会被当为组件返回true
 */
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
   // 遍历当中的过滤条件includeFilters和excludeFilters可以在@ComponentScan中添加
   for (TypeFilter tf : this.excludeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return false;
      }
   }
   /*
    * 在前面初始化过程就其实就已经把@Component注解给add进去了，当然也可以自己在添加TypeFilter
    * 指定特定类型new AssignableTypeFilter(HelloService.class);
    * 指定特定注解new AnnotationTypeFilter(Mapper.class)
    * <code>
    	protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
    * </code>
    */
   for (TypeFilter tf : this.includeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return isConditionMatch(metadataReader);
      }
   }
   return false;
}
```



#### 处理@import注解

前面说完了@ComponentScan注解，解析来逻辑就是处理@import了，也将学习到ImportSelector、ImportBeanDefinitionRegistrar将会怎么使用

ConfigurationClassParser#processImports

```java
/**
 * 如果是导入的类实现了ImportSelector、ImportBeanDefinitionRegistrar并不会被注册为bean
 * 1.ImportSelector所返回的String[]将会进行Bean的注册流程
 */
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
      Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
      boolean checkForCircularImports) {

   if (importCandidates.isEmpty()) {
      return;
   }

   if (checkForCircularImports && isChainedImportOnStack(configClass)) {
      this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
   }
   else {
      this.importStack.push(configClass);
      try {
         for (SourceClass candidate : importCandidates) {
            // 是否有实现ImportSelector
            if (candidate.isAssignable(ImportSelector.class)) {
               // Candidate class is an ImportSelector -> delegate to it to determine imports
               Class<?> candidateClass = candidate.loadClass();
               ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                     this.environment, this.resourceLoader, this.registry);
               Predicate<String> selectorFilter = selector.getExclusionFilter();
               if (selectorFilter != null) {
                  exclusionFilter = exclusionFilter.or(selectorFilter);
               }
               //如果为延迟导入处理则加入集合当中
               if (selector instanceof DeferredImportSelector) {
                  this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
               }
               else {
                  // 执行selectImports方法，得到需要导入的类
                  String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                  Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
                  // 递归进行导入类的注册
                  processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
               }
            }
            // 是否有实现ImportBeanDefinitionRegistrar
            else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
               // Candidate class is an ImportBeanDefinitionRegistrar ->
               // delegate to it to register additional bean definitions
               Class<?> candidateClass = candidate.loadClass();
               // 直接进行实例化
               ImportBeanDefinitionRegistrar registrar =
                     ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                           this.environment, this.resourceLoader, this.registry);
               // 给该类添加importBeanDefinitionRegistrars
               configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
            }
            else {
               // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
               // process it as an @Configuration class
               // 候选者类没有继承ImportSelector或ImportBeanDefinitionRegistrar
               // 用key-value的形式记录@Import的数据
               this.importStack.registerImport(
                     currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
               // 调用processConfigurationClass进行配置类的处理操作
               processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
            }
         }
      }
      catch (BeanDefinitionStoreException ex) {
         throw ex;
      }
      catch (Throwable ex) {
         throw new BeanDefinitionStoreException(
               "Failed to process import candidates for configuration class [" +
               configClass.getMetadata().getClassName() + "]", ex);
      }
      finally {
         this.importStack.pop();
      }
   }
}
```

ParserStrategyUtils#instantiateClass

```java
/**
 * 类实例化
 */
static <T> T instantiateClass(Class<?> clazz, Class<T> assignableTo, Environment environment,
      ResourceLoader resourceLoader, BeanDefinitionRegistry registry) {

   Assert.notNull(clazz, "Class must not be null");
   Assert.isAssignable(assignableTo, clazz);
   if (clazz.isInterface()) {
      throw new BeanInstantiationException(clazz, "Specified class is an interface");
   }
   ClassLoader classLoader = (registry instanceof ConfigurableBeanFactory ?
         ((ConfigurableBeanFactory) registry).getBeanClassLoader() : resourceLoader.getClassLoader());
   T instance = (T) createInstance(clazz, environment, resourceLoader, registry, classLoader);
   ParserStrategyUtils.invokeAwareMethods(instance, environment, resourceLoader, registry, classLoader);
   return instance;
}
```

ParserStrategyUtils#invokeAwareMethods

```java
/**
 * 如果该类也实现了BeanClassLoaderAware、BeanFactoryAware、EnvironmentAware、ResourceLoaderAware
 * 就会分别执行上面对应的setBeanClassLoader、setBeanFactory、setEnvironment、setResourceLoader就能获取到它们相关的属性
 */
private static void invokeAwareMethods(Object parserStrategyBean, Environment environment,
      ResourceLoader resourceLoader, BeanDefinitionRegistry registry, @Nullable ClassLoader classLoader) {

   if (parserStrategyBean instanceof Aware) {
      if (parserStrategyBean instanceof BeanClassLoaderAware && classLoader != null) {
         ((BeanClassLoaderAware) parserStrategyBean).setBeanClassLoader(classLoader);
      }
      if (parserStrategyBean instanceof BeanFactoryAware && registry instanceof BeanFactory) {
         ((BeanFactoryAware) parserStrategyBean).setBeanFactory((BeanFactory) registry);
      }
      if (parserStrategyBean instanceof EnvironmentAware) {
         ((EnvironmentAware) parserStrategyBean).setEnvironment(environment);
      }
      if (parserStrategyBean instanceof ResourceLoaderAware) {
         ((ResourceLoaderAware) parserStrategyBean).setResourceLoader(resourceLoader);
      }
   }
}
```

ConfigurationClassParser#retrieveBeanMethodMetadata

```java
private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
   AnnotationMetadata original = sourceClass.getMetadata();
   // 获取带有@Bean注解的方法
   Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
   if (beanMethods.size() > 1 && original instanceof StandardAnnotationMetadata) {
      // Try reading the class file via ASM for deterministic declaration order...
      // Unfortunately, the JVM's standard reflection returns methods in arbitrary
      // order, even between different runs of the same application on the same JVM.
      //尝试通过ASM读取类文件以获得确定性声明顺序...
      //不幸的是，JVM的标准反射以任意方式返回方法
      //顺序，即使在同一JVM上同一应用程序的不同运行之间也是如此。
      try {
         AnnotationMetadata asm =
               this.metadataReaderFactory.getMetadataReader(original.getClassName()).getAnnotationMetadata();
         Set<MethodMetadata> asmMethods = asm.getAnnotatedMethods(Bean.class.getName());
         if (asmMethods.size() >= beanMethods.size()) {
            Set<MethodMetadata> selectedMethods = new LinkedHashSet<>(asmMethods.size());
            for (MethodMetadata asmMethod : asmMethods) {
               for (MethodMetadata beanMethod : beanMethods) {
                  if (beanMethod.getMethodName().equals(asmMethod.getMethodName())) {
                     selectedMethods.add(beanMethod);
                     break;
                  }
               }
            }
            if (selectedMethods.size() == beanMethods.size()) {
               // All reflection-detected methods found in ASM method set -> proceed
               beanMethods = selectedMethods;
            }
         }
      }
      catch (IOException ex) {
         logger.debug("Failed to read class file via ASM for determining @Bean method order", ex);
         // No worries, let's continue with the reflection metadata we started with...
      }
   }
   return beanMethods;
}
```

> 总结：整个processConfigurationClass方法
>
> a）该方法首先判断是否有Component注解。如果有，则调用processMemberClasses处理内部类。
>
> b）接着处理PropertySource 注解配置
>
> Process any `@PropertySource` annotations
>
> c）接着处理ComponentScan注解
>
> Process any `@ComponentScan` annotations
>
> d）、接着处理Import注解。可以通过Import导入相应的类
> Process any `@Import` annotations
>
> e）、接着处理ImportResource注解。
> Process any `@ImportResource` annotations
>
> f）、接着处理Bean方法。
> Process individual `@Bean` methods
>
> g）、接着处理默认方法
> Process `default` methods on interfaces
>
> h）、接着处理父类。
> Process superclass, `if` any



再回到processConfigBeanDefinitions()方法，接下来就是ConfigurationClassParser#validate方法了

```java
public void validate(ProblemReporter problemReporter) {
   // A configuration class may not be final (CGLIB limitation) unless it declares proxyBeanMethods=false
   // 一个带有@configuration配置类不应该被final修饰（CGLIB的限制），除非它被声明为proxyBeanMethods=false
   Map<String, Object> attributes = this.metadata.getAnnotationAttributes(Configuration.class.getName());
   if (attributes != null && (Boolean) attributes.get("proxyBeanMethods")) {
      // 判断是否为final类
      if (this.metadata.isFinal()) {
         problemReporter.error(new FinalConfigurationProblem());
      }
      for (BeanMethod beanMethod : this.beanMethods) {
         beanMethod.validate(problemReporter);
      }
   }
}
```

> 该方法校验一个带有@configuration配置类不应该被final修饰（CGLIB的限制），除非它被声明为proxyBeanMethods=false。如果该配置类中的方法没有被static修饰，那就不能被final or private修饰，否则加入到problemReporter

BeanMethod#validate

```java
public void validate(ProblemReporter problemReporter) {
   // 带有@Bean静态方法没有限制->直接返回
   if (getMetadata().isStatic()) {
      // static @Bean methods have no constraints to validate -> return immediately
      return;
   }
   // 判断该类是包含Configuration.class注解
   if (this.configurationClass.getMetadata().isAnnotated(Configuration.class.getName())) {
      // 如果该方法不能被覆盖（没有标记为static，被final or private修饰）
      if (!getMetadata().isOverridable()) {
         // instance @Bean methods within @Configuration classes must be overridable to accommodate CGLIB
         problemReporter.error(new NonOverridableMethodError());
      }
   }
}
```

接着看加载Bean定义ConfigurationClassBeanDefinitionReader#loadBeanDefinitions

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
   TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
   for (ConfigurationClass configClass : configurationModel) {
      loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
   }
}
```

ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass

```java
/**
 * 这里还是对配置类进行注册，分为下面几种:
 * 1.@Import或者内部配置类
 * 2.@Bean注解上面的方法进行注册
 * 3.@ImportResource注解导入的xml配置bean
 * 
 * loadBeanDefinitionsFromRegistrars()是对实现了BeanDefinitionsFromRegistrars的registerBeanDefinitions方法进行执行
 */
private void loadBeanDefinitionsForConfigurationClass(
      ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

   if (trackedConditionEvaluator.shouldSkip(configClass)) {
      String beanName = configClass.getBeanName();
      if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
         this.registry.removeBeanDefinition(beanName);
      }
      this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
      return;
   }
	 /*
	  * 该配置类是否是经由@Import引入或注册由于被嵌套（该内部类也要求是配置类）在另一个配置类内
	  */
   if (configClass.isImported()) {
      registerBeanDefinitionForImportedConfigurationClass(configClass);
   }
   // 处理该类上面的@Bean方法
   for (BeanMethod beanMethod : configClass.getBeanMethods()) {
      loadBeanDefinitionsForBeanMethod(beanMethod);
   }
	 // 加载当前ConfigurationClass中@ImportResource引入的xml文件中的Bean定义
   loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
   // 取出实现了ImportBeanDefinitionRegistrar接口的类，执行其registerBeanDefinitions方法。
   // 注册相应的BeanDefinition到仓库中
   loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

ConfigurationClassBeanDefinitionReader#registerBeanDefinitionForImportedConfigurationClass进行import注解上面的类、内部类上的bean注册

```java
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
   AnnotationMetadata metadata = configClass.getMetadata();
   AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);
	 // 处理@Scope
   ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
   configBeanDef.setScope(scopeMetadata.getScopeName());
   // 产生Bean名称
   String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
   // 处理一些通用的注解属性Lazy、Primary、DependsOn、Role、Description
   AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);
   
   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
   // 应用@Scope代理模式
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   // Bean注册--这里其实就是调用之前的注册bean方法
   this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
   configClass.setBeanName(configBeanName);

   if (logger.isTraceEnabled()) {
      logger.trace("Registered bean definition for imported class '" + configBeanName + "'");
   }
}
```

ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod进行带有@Bean注解的方法的bean进行注册

```java
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
   ConfigurationClass configClass = beanMethod.getConfigurationClass();
   MethodMetadata metadata = beanMethod.getMetadata();
   String methodName = metadata.getMethodName();

   // Do we need to mark the bean as skipped by its condition?
   if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
      configClass.skippedBeanMethods.add(methodName);
      return;
   }
   if (configClass.skippedBeanMethods.contains(methodName)) {
      return;
   }
   // 获取元数据上面的@Bean注解
   AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
   Assert.state(bean != null, "No @Bean annotation attributes");

   // Consider name and any aliases
   List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
   String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

   // Register aliases even when overridden
   // 注册别名
   for (String alias : names) {
      this.registry.registerAlias(beanName, alias);
   }

   // Has this effectively been overridden before (e.g. via XML)?
   if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
      if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
         throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
               beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
               "' clashes with bean name for containing configuration class; please make those names unique!");
      }
      return;
   }

   ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
   beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));
	 // 如果是静态
   if (metadata.isStatic()) {
      // static @Bean method
      if (configClass.getMetadata() instanceof StandardAnnotationMetadata) {
         beanDef.setBeanClass(((StandardAnnotationMetadata) configClass.getMetadata()).getIntrospectedClass());
      }
      else {
         beanDef.setBeanClassName(configClass.getMetadata().getClassName());
      }
      beanDef.setUniqueFactoryMethodName(methodName);
   }
   else {
      // instance @Bean method
      beanDef.setFactoryBeanName(configClass.getBeanName());
      beanDef.setUniqueFactoryMethodName(methodName);
   }

   if (metadata instanceof StandardMethodMetadata) {
      beanDef.setResolvedFactoryMethod(((StandardMethodMetadata) metadata).getIntrospectedMethod());
   }

   beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
   beanDef.setAttribute(org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
         SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

   AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);
	 // 获取Bean上面的autowire
   Autowire autowire = bean.getEnum("autowire");
   if (autowire.isAutowire()) {
      beanDef.setAutowireMode(autowire.value());
   }
   // 获取Bean上面的autowireCandidate
   boolean autowireCandidate = bean.getBoolean("autowireCandidate");
   if (!autowireCandidate) {
      beanDef.setAutowireCandidate(false);
   }
   // 获取Bean上面的initMethod
   String initMethodName = bean.getString("initMethod");
   if (StringUtils.hasText(initMethodName)) {
      beanDef.setInitMethodName(initMethodName);
   }
	 // 获取Bean上面的destroyMethod
   String destroyMethodName = bean.getString("destroyMethod");
   beanDef.setDestroyMethodName(destroyMethodName);

   // Consider scoping
   // 处理@scope
   ScopedProxyMode proxyMode = ScopedProxyMode.NO;
   AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
   if (attributes != null) {
      beanDef.setScope(attributes.getString("value"));
      proxyMode = attributes.getEnum("proxyMode");
      // 处理proxyMode
      if (proxyMode == ScopedProxyMode.DEFAULT) {
         proxyMode = ScopedProxyMode.NO;
      }
   }

   // Replace the original bean definition with the target one, if necessary
   BeanDefinition beanDefToRegister = beanDef;
   if (proxyMode != ScopedProxyMode.NO) {
      BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
            new BeanDefinitionHolder(beanDef, beanName), this.registry,
            proxyMode == ScopedProxyMode.TARGET_CLASS);
      beanDefToRegister = new ConfigurationClassBeanDefinition(
            (RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
   }

   if (logger.isTraceEnabled()) {
      logger.trace(String.format("Registering bean definition for @Bean method %s.%s()",
            configClass.getMetadata().getClassName(), beanName));
   }
   // 进行@Bean注册
   this.registry.registerBeanDefinition(beanName, beanDefToRegister);
}
```

ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars

```java
/**
 * 对registerBeanDefinitions()进行执行
 */
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
   registrars.forEach((registrar, metadata) ->
         registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
}
```

> 这里我们可以总结下ImportBeanDefinitionRegistrar的用法，通过实现ImportBeanDefinitionRegistrar去自定义bean注册，
>
> 先来看下接口
>
> ```java
> public interface ImportBeanDefinitionRegistrar {
> 
>   // 如果@Import上面的类为ImportBeanDefinitionRegistrar类型，那么bean在注册的过程中会执行该方法
> 	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
> 			BeanNameGenerator importBeanNameGenerator) {
> 
> 		registerBeanDefinitions(importingClassMetadata, registry);
> 	}
> 
> 	default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
> 	}
> 
> }
> ```
>
> 使用实例如下
>
> ```java
>   @Override
>   public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
>     AnnotationAttributes mapperScanAttrs = AnnotationAttributes
>         .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
>     BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
>     builder.addPropertyValue("processPropertyPlaceHolders", true);
> 
>     Class<? extends Annotation> annotationClass = mapperScanAttrs.getClass("annotationClass");
>     if (!Annotation.class.equals(annotationClass)) {
>       builder.addPropertyValue("annotationClass", annotationClass);
>     }
>     registry.registerBeanDefinition(beanName, builder.getBeanDefinition());
>   }
> ```

到这里一件介绍bean的注册流程，接下来的内容会在下一篇就继续讲到

