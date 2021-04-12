---
typora-root-url: ../../../images
---

# AOP源码分析

本篇只介绍基于@EnableAspectJAutoProxy注解的分析，

@EnableAspectJAutoProxy

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 这里引入了AspectJAutoProxyRegistrar
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

   /**
    * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
    * to standard Java interface-based proxies. The default is {@code false}.
    */
   boolean proxyTargetClass() default false;

   /**
    * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
    * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
    * Off by default, i.e. no guarantees that {@code AopContext} access will work.
    * @since 4.3.1
    */
   boolean exposeProxy() default false;

}
```

spring在bean注册过程中会调用registerBeanDefinitions()

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   /**
    * Register, escalate, and configure the AspectJ auto proxy creator based on the value
    * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
    * {@code @Configuration} class.
    */
   @Override
   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
			// 注册AnnotationAwareAspectJAutoProxyCreator
      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
		  // 获取EnableAspectJAutoProxy注解
      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            // proxyTargetClass=true，使用cglib作为动态代理
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            // 将代理暴露出来
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }

}
```

AnnotationAwareAspectJAutoProxyCreator的类结构图如下

![AnnotationAwareAspectJAutoProxyCreator](/AnnotationAwareAspectJAutoProxyCreator.png)

该类继承SmartInstantiationAwareBeanPostProcessor，重写了postProcessBeforeInstantiation和（实例化之前执行）postProcessAfterInitialization（初始化之后执行）方法。



#### postProcessBeforeInstantiation

AbstractAutoProxyCreator#postProcessBeforeInstantiation

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
   Object cacheKey = getCacheKey(beanClass, beanName);
	 // beanName为空 || targetSourcedBeans不包含beanName
   if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
      if (this.advisedBeans.containsKey(cacheKey)) {
         return null;
      }
      // 实现Advice、Advisor、Pointcut、AopInfrastructureBean || 带有@Aspect注解
      // bean应该跳过，则标记为无需处理，并返回
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return null;
      }
   }

   // Create proxy here if we have a custom TargetSource.
   // Suppresses unnecessary default instantiation of the target bean:
   // The TargetSource will handle target instances in a custom fashion.
   // 如果我们有自定义的TargetSource，请在此处创建代理。 
   // 抑制目标bean的不必要的默认实例化：TargetSource将以自定义方式处理目标实例。
   // customTargetSourceCreators默认是空
   TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
   if (targetSource != null) {
      if (StringUtils.hasLength(beanName)) {
         this.targetSourcedBeans.add(beanName);
      }
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
      Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   return null;
}
```

AbstractAutoProxyCreator#shouldSkip

```java
@Override
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
   // TODO: Consider optimization by caching the list of the aspect names
   // 找到通知器的候选者
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   for (Advisor advisor : candidateAdvisors) {
      // 为AspectJPointcutAdvisor类型 && 切面名称.equals(beanName)
      if (advisor instanceof AspectJPointcutAdvisor &&
            ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
         return true;
      }
   }
   // beanName+".ORIGINAL.equals(beanClass.getName())
   return super.shouldSkip(beanClass, beanName);
}
```

获取所有的AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

```java
@Override
protected List<Advisor> findCandidateAdvisors() {
   // Add all the Spring advisors found according to superclass rules.
   // 添加所有的spring通知器（继承Advisor接口的bean）
   List<Advisor> advisors = super.findCandidateAdvisors();
   // Build Advisors for all AspectJ aspects in the bean factory.
   // 处理Bean工厂中所有AspectJ切面构建通知器(添加@Aspect注解的bean带有@Pointcut、@Around、@Before、@After、@AfterReturning、@AfterThrowing的方法)
   if (this.aspectJAdvisorsBuilder != null) {
      advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
   }
   return advisors;
}
```

AbstractAdvisorAutoProxyCreator#findCandidateAdvisors

```java
protected List<Advisor> findCandidateAdvisors() {
   Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
   return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans

```java
public List<Advisor> findAdvisorBeans() {
   // Determine list of advisor bean names, if not cached already.
   // 尝试去缓存中获取bean名称
   String[] advisorNames = this.cachedAdvisorBeanNames;
   if (advisorNames == null) {
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the auto-proxy creator apply to them!
      // 获取Advisor类型的bean名称
      advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
      // 设为缓存
      this.cachedAdvisorBeanNames = advisorNames;
   }
   if (advisorNames.length == 0) {
      return new ArrayList<>();
   }

   List<Advisor> advisors = new ArrayList<>();
   for (String name : advisorNames) {
      if (isEligibleBean(name)) {
         // 合格bean
         if (this.beanFactory.isCurrentlyInCreation(name)) {
            // 该bean当前正在创建就直接跳过
            if (logger.isTraceEnabled()) {
               logger.trace("Skipping currently created advisor '" + name + "'");
            }
         }
         else {
            // 非当前正在创建的bean添加到advisors
            try {
               advisors.add(this.beanFactory.getBean(name, Advisor.class));
            }
            catch (BeanCreationException ex) {
               Throwable rootCause = ex.getMostSpecificCause();
               if (rootCause instanceof BeanCurrentlyInCreationException) {
                  BeanCreationException bce = (BeanCreationException) rootCause;
                  String bceBeanName = bce.getBeanName();
                  if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                     if (logger.isTraceEnabled()) {
                        logger.trace("Skipping advisor '" + name +
                              "' with dependency on currently created bean: " + ex.getMessage());
                     }
                     // Ignore: indicates a reference back to the bean we're trying to advise.
                     // We want to find advisors other than the currently created bean itself.
                     continue;
                  }
               }
               throw ex;
            }
         }
      }
   }
   return advisors;
}
```

BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

```java
public List<Advisor> buildAspectJAdvisors() {
   List<String> aspectNames = this.aspectBeanNames;

   if (aspectNames == null) {
      synchronized (this) {
         aspectNames = this.aspectBeanNames;
         if (aspectNames == null) {
            // 同步的二次检查通过，double-check单例写法
            List<Advisor> advisors = new ArrayList<>();
            aspectNames = new ArrayList<>();
            // 获取工厂中所有的bean名称
            String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                  this.beanFactory, Object.class, true, false);
            for (String beanName : beanNames) {
               // 是否合格
               // AspectJAwareAdvisorAutoProxyCreator中实现是通过includePatterns进行判断
               if (!isEligibleBean(beanName)) {
                  // 不合格就跳过
                  continue;
               }
               // We must be careful not to instantiate beans eagerly as in this case they
               // would be cached by the Spring container but would not have been weaved.
               // 获取beanName的类型
               Class<?> beanType = this.beanFactory.getType(beanName);
               if (beanType == null) {
                  continue;
               }
               if (this.advisorFactory.isAspect(beanType)) {
                  // 存在@Aspect注解
                  aspectNames.add(beanName);
                  AspectMetadata amd = new AspectMetadata(beanType, beanName);
                  //
                  if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                     MetadataAwareAspectInstanceFactory factory =
                           new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                     // 获取Advisor
                     // 获取带有@Pointcut, @Around, @Before, @After, @AfterReturning, @AfterThrowing的方法
      							 // 获取带有@DeclareParents的属性
                     List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                     if (this.beanFactory.isSingleton(beanName)) {
                        // 为单例添加到advisorsCache中
                        this.advisorsCache.put(beanName, classAdvisors);
                     }
                     else {
                        this.aspectFactoryCache.put(beanName, factory);
                     }
                     // 添加到Advisor中
                     advisors.addAll(classAdvisors);
                  }
                  else {
                     // Per target or per this.
                     if (this.beanFactory.isSingleton(beanName)) {
                        // // 名称为beanName的Bean是单例，但切面实例化模型不是单例，则抛异常
                        throw new IllegalArgumentException("Bean with name '" + beanName +
                              "' is a singleton, but aspect instantiation model is not singleton");
                     }
                     MetadataAwareAspectInstanceFactory factory =
                           new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                     this.aspectFactoryCache.put(beanName, factory);
                     advisors.addAll(this.advisorFactory.getAdvisors(factory));
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
   }
   List<Advisor> advisors = new ArrayList<>();
   for (String aspectName : aspectNames) {
      List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
      if (cachedAdvisors != null) {
         // 缓存不为空
         advisors.addAll(cachedAdvisors);
      }
      else {
         // 为空就从aspectFactoryCache中获取
         MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
         advisors.addAll(this.advisorFactory.getAdvisors(factory));
      }
   }
   return advisors;
}
```

ReflectiveAspectJAdvisorFactory#getAdvisors(MetadataAwareAspectInstanceFactory)

```java
@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
   // 获取获取切面类
   Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   // 获取切面名称
   String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
   // 进行校验
   validate(aspectClass);

   // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
   // so that it will only instantiate once.
   // 使用装饰器包装MetadataAwareAspectInstanceFactory，以便它只实例化一次
   MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
         new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

   List<Advisor> advisors = new ArrayList<>();
   // 获取通知器上不带@Pointcut的方法，	并进行遍历
   for (Method method : getAdvisorMethods(aspectClass)) {
      // Prior to Spring Framework 5.2.7, advisors.size() was supplied as the declarationOrderInAspect
      // to getAdvisor(...) to represent the "current position" in the declared methods list.
      // However, since Java 7 the "current position" is not valid since the JDK no longer
      // returns declared methods in the order in which they are declared in the source code.
      // Thus, we now hard code the declarationOrderInAspect to 0 for all advice methods
      // discovered via reflection in order to support reliable advice ordering across JVM launches.
      // Specifically, a value of 0 aligns with the default value used in
      // AspectJPrecedenceComparator.getAspectDeclarationOrder(Advisor).
      // 获取带有@Pointcut, @Around, @Before, @After, @AfterReturning, @AfterThrowing的方法
      Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 0, aspectName);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   // If it's a per target aspect, emit the dummy instantiating aspect.
   if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
      // 如果寻找的advisors不为空而且又配置了增强延迟初始化，那么需要在首位加入同步实例化advisors
      //（用以保证advisors使用之前的实例化）
      Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
      advisors.add(0, instantiationAdvisor);
   }

   // Find introduction fields.
   for (Field field : aspectClass.getDeclaredFields()) {
      // 获取字段上的@DeclareParents
      Advisor advisor = getDeclareParentsAdvisor(field);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   return advisors;
}
```

ReflectiveAspectJAdvisorFactory#getAdvisors(Method, MetadataAwareAspectInstanceFactory)

```java
@Override
@Nullable
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
      int declarationOrderInAspect, String aspectName) {

   validate(aspectInstanceFactory.getAspectMetadata().getAspectClass()); 
   // 因为我们之前把@Pointcut注解的方法跳过了，所以这边必然不会获取到@Pointcut注解
	 // 获取带有@Pointcut, @Around, @Before, @After, @AfterReturning, @AfterThrowing的方法，
   // 然后封装为AspectJExpressionPointcut对象
   AspectJExpressionPointcut expressionPointcut = getPointcut(
         candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
   if (expressionPointcut == null) {
      // 为空就返回
      return null;
   }

   return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
         this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

ReflectiveAspectJAdvisorFactory#getPointcut

```java
@Nullable
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
   // 获取带有@Pointcut, @Around, @Before, @After, @AfterReturning, @AfterThrowing的方法，
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }
	 // 创建一个AspectJExpressionPointcut对象
   AspectJExpressionPointcut ajexp =
         new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
   // set表示式示例
   ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
   if (this.beanFactory != null) {
      ajexp.setBeanFactory(this.beanFactory);
   }
   return ajexp;
}
```

ReflectiveAspectJAdvisorFactory#findAspectJAnnotationOnMethod

```java
@Nullable
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
   // 遍历Pointcut.class, Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class注解
   for (Class<?> clazz : ASPECTJ_ANNOTATION_CLASSES) {
      AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) clazz);
      if (foundAnnotation != null) {
         return foundAnnotation;
      }
   }
   return null;
}
```



构造方法InstantiationModelAwarePointcutAdvisorImpl

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		this.declaredPointcut = declaredPointcut;
		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
		this.methodName = aspectJAdviceMethod.getName();
		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
		this.aspectInstanceFactory = aspectInstanceFactory;
		this.declarationOrder = declarationOrder;
		this.aspectName = aspectName;
		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			// Static part of the pointcut is a lazy type.
			Pointcut preInstantiationPointcut = Pointcuts.union(
					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
			// If it's not a dynamic pointcut, it may be optimized out
			// by the Spring AOP infrastructure after the first evaluation.
			this.pointcut = new PerTargetInstantiationModelPointcut(
					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
			this.lazy = true;
		}
		else {
			// A singleton aspect.
			this.pointcut = this.declaredPointcut;
			this.lazy = false;
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
}
```

InstantiationModelAwarePointcutAdvisorImpl#instantiateAdvice

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
   Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
         this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
   return (advice != null ? advice : EMPTY_ADVICE);
}
```

ReflectiveAspectJAdvisorFactory#getAdvice

```java
@Override
@Nullable
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
	 // 获取切面类
   Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   // 切面类校验
   validate(candidateAspectClass);
   // 获取AspectJ注解
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }

   // If we get here, we know we have an AspectJ method.
   // Check that it's an AspectJ-annotated class
   if (!isAspect(candidateAspectClass)) {
      // 非切面类抛出异常
      throw new AopConfigException("Advice must be declared inside an aspect type: " +
            "Offending method '" + candidateAdviceMethod + "' in class [" +
            candidateAspectClass.getName() + "]");
   }

   if (logger.isDebugEnabled()) {
      logger.debug("Found AspectJ method: " + candidateAdviceMethod);
   }

   AbstractAspectJAdvice springAdvice;
	 // 获取AspectJ注解类型，根据不同类型建立不同的Advice对象
   switch (aspectJAnnotation.getAnnotationType()) {
      case AtPointcut:
         if (logger.isDebugEnabled()) {
            logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
         }
         return null;
      case AtAround:
         springAdvice = new AspectJAroundAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtBefore:
         springAdvice = new AspectJMethodBeforeAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfter:
         springAdvice = new AspectJAfterAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfterReturning:
         springAdvice = new AspectJAfterReturningAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterReturningAnnotation.returning())) {
            springAdvice.setReturningName(afterReturningAnnotation.returning());
         }
         break;
      case AtAfterThrowing:
         springAdvice = new AspectJAfterThrowingAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
            springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
         }
         break;
      default:
         throw new UnsupportedOperationException(
               "Unsupported advice type on method: " + candidateAdviceMethod);
   }

   // Now to configure the advice...
   springAdvice.setAspectName(aspectName);
   springAdvice.setDeclarationOrder(declarationOrder);
   // 获取AspectJ注解上的argNames参数
   String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
   if (argNames != null) {
      springAdvice.setArgumentNamesFromStringArray(argNames);
   }
   springAdvice.calculateArgumentBindings();

   return springAdvice;
}
```

AbstractAspectJAdvisorFactory.AspectJAnnotationParameterNameDiscoverer#getParameterNames

```java
@Override
@Nullable
public String[] getParameterNames(Method method) {
   if (method.getParameterCount() == 0) {
      return new String[0];
   }
   // 获取方法上面的AspectJ注解
   AspectJAnnotation<?> annotation = findAspectJAnnotationOnMethod(method);
   if (annotation == null) {
      return null;
   }
   // 获取argNames参数并封装为StringTokenizer
   StringTokenizer nameTokens = new StringTokenizer(annotation.getArgumentNames(), ",");
   if (nameTokens.countTokens() > 0) {
      // 参数不为空
      String[] names = new String[nameTokens.countTokens()];
      for (int i = 0; i < names.length; i++) {
         names[i] = nameTokens.nextToken();
      }
      return names;
   }
   else {
      return null;
   }
}
```

AbstractAspectJAdvice#setArgumentNamesFromStringArray

```java
public void setArgumentNamesFromStringArray(String... args) {
   this.argumentNames = new String[args.length];
   for (int i = 0; i < args.length; i++) {
      // 去除首位空格
      this.argumentNames[i] = StringUtils.trimWhitespace(args[i]);
      // 校验参数是否为有效变量
      if (!isVariableName(this.argumentNames[i])) {
         throw new IllegalArgumentException(
               "'argumentNames' property of AbstractAspectJAdvice contains an argument name '" +
               this.argumentNames[i] + "' that is not a valid Java identifier");
      }
   }
   if (this.argumentNames != null) {
      if (this.aspectJAdviceMethod.getParameterCount() == this.argumentNames.length + 1) {
         // May need to add implicit join point arg name...
         // 增强方法参数个数 = argNames.length+1
         Class<?> firstArgType = this.aspectJAdviceMethod.getParameterTypes()[0];
         // 第一个参数类型为JoinPoint||ProceedingJoinPoint||JoinPoint.StaticPart
         if (firstArgType == JoinPoint.class ||
               firstArgType == ProceedingJoinPoint.class ||
               firstArgType == JoinPoint.StaticPart.class) {
            String[] oldNames = this.argumentNames;
            this.argumentNames = new String[oldNames.length + 1];
            this.argumentNames[0] = "THIS_JOIN_POINT";
            // 等效于直接oldNames.add(0, "THIS_JOIN_POINT")，这里是创建了一个新的数组进行插入
            System.arraycopy(oldNames, 0, this.argumentNames, 1, oldNames.length);
         }
      }
   }
}
```

AbstractAspectJAdvice#calculateArgumentBindings

```java
public final synchronized void calculateArgumentBindings() {
   // The simple case... nothing to bind.
   // 没有参数
   if (this.argumentsIntrospected || this.parameterTypes.length == 0) {
      return;
   }
	 // 参数个数
   int numUnboundArgs = this.parameterTypes.length;
   Class<?>[] parameterTypes = this.aspectJAdviceMethod.getParameterTypes();
   // 第一个参数类型为JoinPoint||ProceedingJoinPoint||JoinPoint.StaticPart
   if (maybeBindJoinPoint(parameterTypes[0]) || maybeBindProceedingJoinPoint(parameterTypes[0]) ||
         maybeBindJoinPointStaticPart(parameterTypes[0])) {
      numUnboundArgs--;
   }
	 // 还存在其它参数
   if (numUnboundArgs > 0) {
      // need to bind arguments by name as returned from the pointcut match
      // 需要通过切入点匹配返回的名称绑定参数
      bindArgumentsByName(numUnboundArgs);
   }

   this.argumentsIntrospected = true;
}
```

AbstractAspectJAdvice#bindArgumentsByName

```java
private void bindArgumentsByName(int numArgumentsExpectingToBind) {
   if (this.argumentNames == null) {
      this.argumentNames = createParameterNameDiscoverer().getParameterNames(this.aspectJAdviceMethod);
   }
   if (this.argumentNames != null) {
      // We have been able to determine the arg names.
      bindExplicitArguments(numArgumentsExpectingToBind);
   }
   else {
      throw new IllegalStateException("Advice method [" + this.aspectJAdviceMethod.getName() + "] " +
            "requires " + numArgumentsExpectingToBind + " arguments to be bound by name, but " +
            "the argument names were not specified and could not be discovered.");
   }
}
```

AbstractAspectJAdvice#bindExplicitArguments

```java
private void bindExplicitArguments(int numArgumentsLeftToBind) {
   Assert.state(this.argumentNames != null, "No argument names available");
   this.argumentBindings = new HashMap<>();
	 // 获取参数的数目
   int numExpectedArgumentNames = this.aspectJAdviceMethod.getParameterCount();
   if (this.argumentNames.length != numExpectedArgumentNames) {
      throw new IllegalStateException("Expecting to find " + numExpectedArgumentNames +
            " arguments to bind by name in advice, but actually found " +
            this.argumentNames.length + " arguments.");
   }

   // So we match in number...
   // 计算偏移量
   int argumentIndexOffset = this.parameterTypes.length - numArgumentsLeftToBind;
   for (int i = argumentIndexOffset; i < this.argumentNames.length; i++) {
      this.argumentBindings.put(this.argumentNames[i], i);
   }

   // Check that returning and throwing were in the argument names list if
   // specified, and find the discovered argument types.
   if (this.returningName != null) {
      if (!this.argumentBindings.containsKey(this.returningName)) {
         throw new IllegalStateException("Returning argument name '" + this.returningName +
               "' was not bound in advice arguments");
      }
      else {
         Integer index = this.argumentBindings.get(this.returningName);
         this.discoveredReturningType = this.aspectJAdviceMethod.getParameterTypes()[index];
         this.discoveredReturningGenericType = this.aspectJAdviceMethod.getGenericParameterTypes()[index];
      }
   }
   if (this.throwingName != null) {
      if (!this.argumentBindings.containsKey(this.throwingName)) {
         throw new IllegalStateException("Throwing argument name '" + this.throwingName +
               "' was not bound in advice arguments");
      }
      else {
         Integer index = this.argumentBindings.get(this.throwingName);
         this.discoveredThrowingType = this.aspectJAdviceMethod.getParameterTypes()[index];
      }
   }

   // configure the pointcut expression accordingly.
   configurePointcutParameters(this.argumentNames, argumentIndexOffset);
}
```



#### postProcessAfterInitialization

接下来，重点关注初始化之后的后置处理器AbstractAutoProxyCreator#postProcessAfterInitialization

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      // 缓存中存在就直接返回
      if (this.earlyProxyReferences.remove(cacheKey) != bean) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

AbstractAutoProxyCreator#wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   // 判断当前bean是否在targetSourcedBeans缓存中存在（已经处理过），如果存在，则直接返回当前bean
   if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   // 在advisedBeans缓存中存在，并且value为false，则代表无需处理
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   // bean的类是aop基础设施类 || bean应该跳过，则标记为无需处理，并返回
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // Create proxy if we have advice.
   // 
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      // 创建代理
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean

```java
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
      Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

   List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
   if (advisors.isEmpty()) {
      return DO_NOT_PROXY;
   }
   return advisors.toArray();
}
```

AbstractAdvisorAutoProxyCreator#findEligibleAdvisors

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
   // 获取所有的Advisor
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   // 搜索给定的候选Advisor，以查找可以应用于指定bean的所有Advisor。
   List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
   // 扩展方法，留个子类实现
   extendAdvisors(eligibleAdvisors);
   if (!eligibleAdvisors.isEmpty()) {
      // 对符合条件的Advisor进行排序
      eligibleAdvisors = sortAdvisors(eligibleAdvisors);
   }
   return eligibleAdvisors;
}
```

```java
/**
 * 搜索给定的候选Advisor，以查找可以应用于指定bean的所有Advisor。
 */
protected List<Advisor> findAdvisorsThatCanApply(
      List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
	 // 名称添加到线程上下文
   ProxyCreationContext.setCurrentProxiedBeanName(beanName);
   try {
      // 
      return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
   }
   finally {
      // 移除当前线程的bean名称
      ProxyCreationContext.setCurrentProxiedBeanName(null);
   }
}
```

AbstractAutoProxyCreator#findAdvisorsThatCanApply

```java
/**
 * 确定适用于给定类的advisoryAdvisors列表的子列表。
 */
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
   if (candidateAdvisors.isEmpty()) {
      return candidateAdvisors;
   }
   List<Advisor> eligibleAdvisors = new ArrayList<>();
   // 首先处理引介增强（@DeclareParents）用的比较少可以忽略
   for (Advisor candidate : candidateAdvisors) {
      if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
         eligibleAdvisors.add(candidate);
      }
   }
   boolean hasIntroductions = !eligibleAdvisors.isEmpty();
   // 遍历所有的candidateAdvisors
   for (Advisor candidate : candidateAdvisors) {
      // @DeclareParents已经处理，直接跳过
      if (candidate instanceof IntroductionAdvisor) {
         // already processed
         continue;
      }
      // 正常增强处理，判断当前bean是否可以应用于当前遍历的Advisor（bean是否包含在Advisor的execution指定的表达式中）
      // 这边表达式判断的逻辑比较复杂，可以简单的理解为：判断 bean 是否包含在增强器的execution指定的表达式中
      if (canApply(candidate, clazz, hasIntroductions)) {
         eligibleAdvisors.add(candidate);
      }
   }
   return eligibleAdvisors;
}
```

AbstractAutoProxyCreator#createProxy

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
      @Nullable Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

   ProxyFactory proxyFactory = new ProxyFactory();
   // 复制属性
   proxyFactory.copyFrom(this);
	 // 检查proxyTargetClass属性，判断对于给定的bean使用类代理还是接口代理，
   // proxyTargetClass值默认为false，可以通过@EnableAspectJAutoProxy的proxyTargetClass属性设置为true
   if (!proxyFactory.isProxyTargetClass()) {
      // 检查preserveTargetClass属性，判断beanClass是应该基于类代理还是基于接口代理
      if (shouldProxyTargetClass(beanClass, beanName)) {
         // 如果是基于类代理，则将proxyTargetClass赋值为true
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 评估bean的代理接口
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }
	 // 将拦截器封装为Advisor（advice持有者）
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   // 将advisors添加到proxyFactory
   proxyFactory.addAdvisors(advisors);
   // 设置要代理的类，将targetSource赋值给proxyFactory的targetSource属性，之后可以通过该属性拿到被代理的bean的实例
   proxyFactory.setTargetSource(targetSource);
   // 自定义ProxyFactory，空方法，留给子类实现
   customizeProxyFactory(proxyFactory);
	 // 用来控制proxyFactory被配置之后，是否还允许修改通知。默认值为false（即在代理被配置之后，不允许修改代理类的配置）
   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }
	 // 使用proxyFactory获取代理
   return proxyFactory.getProxy(getProxyClassLoader());
}
```

ProxyFactory#getProxy

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```

ProxyCreatorSupport#createAopProxy

```java
protected final synchronized AopProxy createAopProxy() {
   if (!this.active) {
      activate();
   }
   return getAopProxyFactory().createAopProxy(this);
}
```

AbstractAutoProxyCreator#createProxy

```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	  // 判断使用JDK动态代理还是Cglib代理
    // optimize：用于控制通过cglib创建的代理是否使用激进的优化策略。除非完全了解AOP如何处理代理优化，
    // 否则不推荐使用这个配置，目前这个属性仅用于cglib代理，对jdk动态代理无效
    // proxyTargetClass：默认为false，设置为true时，强制使用cglib代理，设置方式：@EnableAspectJAutoProxy(proxyTargetClass = true)
    // hasNoUserSuppliedProxyInterfaces：是否不存在代理接口
   if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
      // 拿到要被代理的对象的类型
      Class<?> targetClass = config.getTargetClass();
      if (targetClass == null) {
         throw new AopConfigException("TargetSource cannot determine target class: " +
               "Either an interface or a target is required for proxy creation.");
      }
      // 要被代理的对象是接口 || targetClass是Proxy class
      // 当且仅当使用getProxyClass方法或newProxyInstance方法动态生成指定的类作为代理类时，才返回true。
      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
         // JDK动态代理，这边的入参config(AdvisedSupport)实际上是ProxyFactory对象
         // 具体为：AbstractAutoProxyCreator中的proxyFactory.getProxy发起的调用，在ProxyCreatorSupport使用了this作为参数，
         // 调用了的本方法，这边的this就是发起调用的proxyFactory对象，而proxyFactory对象中包含了要执行的的拦截器
         return new JdkDynamicAopProxy(config);
      }
      // cglib代理
      return new ObjenesisCglibAopProxy(config);
   }
   else {
      // jdk代理
      return new JdkDynamicAopProxy(config);
   }
}
```

CglibAopProxy#getProxy

```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
   }

   try {
      // 拿到要代理目标类
      Class<?> rootClass = this.advised.getTargetClass();
      Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");
		  // proxySuperClass默认为rootClass
      Class<?> proxySuperClass = rootClass;
      // 是否名称以$$开头
      if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
         proxySuperClass = rootClass.getSuperclass();
         Class<?>[] additionalInterfaces = rootClass.getInterfaces();
         for (Class<?> additionalInterface : additionalInterfaces) {
            this.advised.addInterface(additionalInterface);
         }
      }

      // Validate the class, writing log messages as necessary.
      validateClassIfNecessary(proxySuperClass, classLoader);

      // Configure CGLIB Enhancer...
      // 配置CGLIB的Enhancer
      Enhancer enhancer = createEnhancer();
      if (classLoader != null) {
         enhancer.setClassLoader(classLoader);
         if (classLoader instanceof SmartClassLoader &&
               ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
            enhancer.setUseCache(false);
         }
      }
      enhancer.setSuperclass(proxySuperClass);
      enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
      enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
      enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));
			// 获取所有要回调的拦截器
      Callback[] callbacks = getCallbacks(rootClass);
      Class<?>[] types = new Class<?>[callbacks.length];
      for (int x = 0; x < types.length; x++) {
         types[x] = callbacks[x].getClass();
      }
      // fixedInterceptorMap only populated at this point, after getCallbacks call above
      enhancer.setCallbackFilter(new ProxyCallbackFilter(
            this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
      enhancer.setCallbackTypes(types);

      // Generate the proxy class and create a proxy instance.
      // 生产代理类和创建代理实例
      return createProxyClassAndInstance(enhancer, callbacks);
   }
   catch (CodeGenerationException | IllegalArgumentException ex) {
      throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
            ": Common causes of this problem include using a final class or a non-visible class",
            ex);
   }
   catch (Throwable ex) {
      // TargetSource.getTarget() failed
      throw new AopConfigException("Unexpected AOP exception", ex);
   }
}
```

CglibAopProxy#getCallbacks

```java
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
   // Parameters used for optimization choices...
   boolean exposeProxy = this.advised.isExposeProxy();
   boolean isFrozen = this.advised.isFrozen();
   boolean isStatic = this.advised.getTargetSource().isStatic();

   // Choose an "aop" interceptor (used for AOP calls).
   Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

   // Choose a "straight to target" interceptor. (used for calls that are
   // unadvised but can return this). May be required to expose the proxy.
   Callback targetInterceptor;
   if (exposeProxy) {
      targetInterceptor = (isStatic ?
            new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
            new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
   }
   else {
      targetInterceptor = (isStatic ?
            new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
            new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
   }

   // Choose a "direct to target" dispatcher (used for
   // unadvised calls to static targets that cannot return this).
   Callback targetDispatcher = (isStatic ?
         new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());
	 // 所有的回调
   Callback[] mainCallbacks = new Callback[] {
         aopInterceptor,  // for normal advice
         targetInterceptor,  // invoke target without considering advice, if optimized
         new SerializableNoOp(),  // no override for methods mapped to this
         targetDispatcher, this.advisedDispatcher,
         new EqualsInterceptor(this.advised),
         new HashCodeInterceptor(this.advised)
   };

   Callback[] callbacks;

   // If the target is a static one and the advice chain is frozen,
   // then we can make some optimizations by sending the AOP calls
   // direct to the target using the fixed chain for that method.
   if (isStatic && isFrozen) {
      Method[] methods = rootClass.getMethods();
      Callback[] fixedCallbacks = new Callback[methods.length];
      this.fixedInterceptorMap = new HashMap<>(methods.length);

      // TODO: small memory optimization here (can skip creation for methods with no advice)
      for (int x = 0; x < methods.length; x++) {
         Method method = methods[x];
         List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
         fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
               chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
         this.fixedInterceptorMap.put(method, x);
      }

      // Now copy both the callbacks from mainCallbacks
      // and fixedCallbacks into the callbacks array.
      callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
      System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
      System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
      this.fixedInterceptorOffset = mainCallbacks.length;
   }
   else {
      callbacks = mainCallbacks;
   }
   return callbacks;
}
```

JdkDynamicAopProxy#getProxy

```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
   }
   // 拿到要被代理对象的所有接口
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```