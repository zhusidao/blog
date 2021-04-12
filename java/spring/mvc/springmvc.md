# spring启动流程

FrameworkServlet#initWebApplicationContext

```java
protected WebApplicationContext initWebApplicationContext() {
   // 以key为org.springframework.web.context.WebApplicationContext.ROOT去ServletContext中去获取属性
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   WebApplicationContext wac = null;
	 // webApplicationContext属性不为空
   if (this.webApplicationContext != null) {
      // A context instance was injected at construction time -> use it
      // 在构造器注入了一个上下文实例->使用它
      wac = this.webApplicationContext;
      // 上下文为ConfigurableWebApplicationContext类型
      if (wac instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
         // 确定此应用程序上下文是否处于活动状态，即，是否至少刷新一次并且尚未关闭
         if (!cwac.isActive()) {
            // The context has not yet been refreshed -> provide services such as
            // setting the parent context, setting the application context id, etc
            // 上下文尚未刷新->提供诸如设置父上下文，设置应用程序上下文ID等服务
            if (cwac.getParent() == null) {
               // The context instance was injected without an explicit parent -> set
               // the root application context (if any; may be null) as the parent
               // 上下文实例注入没有明确的父 - >设置的根应用上下文（如果有的话;可以为null）作为父
               cwac.setParent(rootContext);
            }
            // springioc整个流程在这里执行
            configureAndRefreshWebApplicationContext(cwac);
         }
      }
   }
   if (wac == null) {
      // No context instance was injected at construction time -> see if one
      // has been registered in the servlet context. If one exists, it is assumed
      // that the parent context (if any) has already been set and that the
      // user has performed any initialization such as setting the context id
      // 在构造时未注入任何上下文实例->查看是否已在servlet上下文中注册了一个实例。
      // 如果存在，则假定已经设置了父上下文（如果有），并且用户已经执行了任何初始化操作，例如设置了上下文ID。
      wac = findWebApplicationContext();
   }
   if (wac == null) {
      // No context instance is defined for this servlet -> create a local one
      // 没有为此Servlet定义上下文实例->创建本地实例
      wac = createWebApplicationContext(rootContext);
   }

   if (!this.refreshEventReceived) {
      // Either the context is not a ConfigurableApplicationContext with refresh
      // support or the context injected at construction time had already been
      // refreshed -> trigger initial onRefresh manually here.
      synchronized (this.onRefreshMonitor) {
         // 
         onRefresh(wac);
      }
   }

   if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      // 发布上下文作为servlet上下文的属性
      String attrName = getServletContextAttributeName();
      getServletContext().setAttribute(attrName, wac);
   }

   return wac;
}
```

WebApplicationContextUtils#getWebApplicationContext

```java
@Nullable
public static WebApplicationContext getWebApplicationContext(ServletContext sc) {
   return getWebApplicationContext(sc, WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
}
```

WebApplicationContextUtils#getWebApplicationContext

```java
@Nullable
public static WebApplicationContext getWebApplicationContext(ServletContext sc, String attrName) {
   Assert.notNull(sc, "ServletContext must not be null");
   // 以key为org.springframework.web.context.WebApplicationContext.ROOT去ServletContext中去获取属性
   Object attr = sc.getAttribute(attrName);
   if (attr == null) {
      return null;
   }
   if (attr instanceof RuntimeException) {
      throw (RuntimeException) attr;
   }
   if (attr instanceof Error) {
      throw (Error) attr;
   }
   if (attr instanceof Exception) {
      throw new IllegalStateException((Exception) attr);
   }
   // 不是WebApplicationContext就会抛出异常
   if (!(attr instanceof WebApplicationContext)) {
      throw new IllegalStateException("Context attribute is not of type WebApplicationContext: " + attr);
   }
   return (WebApplicationContext) attr;
}
```

FrameworkServlet#configureAndRefreshWebApplicationContext

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
   if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
      // The application context id is still set to its original default value
      // -> assign a more useful id based on available information
      // 上下文id存在就直接进行set
      if (this.contextId != null) {
         wac.setId(this.contextId);
      }
      else {
         // Generate default id...
         // 产生一个默认id
         wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
               ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
      }
   }
	 // 设置servlet上下文
   wac.setServletContext(getServletContext());
   // 设置servlet配置
   wac.setServletConfig(getServletConfig());
   // 设置命名空间
   wac.setNamespace(getNamespace());
   wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

   // The wac environment's #initPropertySources will be called in any case when the context
   // is refreshed; do it eagerly here to ensure servlet property sources are in place for
   // use in any post-processing or initialization that occurs below prior to #refresh
   // 在刷新上下文的任何情况下，都会调用wac环境的#initPropertySources。 
   // 请务必在此处急于执行此操作，以确保servlet属性源已准备就绪，可用于以下#refresh之前发生的任何后置处理或初始化
   ConfigurableEnvironment env = wac.getEnvironment();
   if (env instanceof ConfigurableWebEnvironment) {
      ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
   }
	 // 留给子类去实现
   postProcessWebApplicationContext(wac);
   /
   applyInitializers(wac);
   // 很熟悉，有没有? spring整个核心启动流程
   wac.refresh();
}
```

AbstractApplicationContext#getEnvironment

```java
@Override
public ConfigurableEnvironment getEnvironment() {
   if (this.environment == null) {
      // 为空就new StandardEnvironment()
      this.environment = createEnvironment();
   }
   return this.environment;
}
```

WebApplicationContextUtils#initServletPropertySources

```java
public static void initServletPropertySources(MutablePropertySources sources,
      @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
	 // 替换掉servletContextInitParams属性的值
   Assert.notNull(sources, "'propertySources' must not be null");
   String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME;
   if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
      sources.replace(name, new ServletContextPropertySource(name, servletContext));
   }
   // 替换掉servletConfigInitParams属性的值
   name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME;
   if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
      sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
   }
}
```

FrameworkServlet#applyInitializers

```java
protected void applyInitializers(ConfigurableApplicationContext wac) {
   String globalClassNames = getServletContext().getInitParameter(ContextLoader.GLOBAL_INITIALIZER_CLASSES_PARAM);
   if (globalClassNames != null) {
      for (String className : StringUtils.tokenizeToStringArray(globalClassNames, INIT_PARAM_DELIMITERS)) {
         this.contextInitializers.add(loadInitializer(className, wac));
      }
   }

   if (this.contextInitializerClasses != null) {
      for (String className : StringUtils.tokenizeToStringArray(this.contextInitializerClasses, INIT_PARAM_DELIMITERS)) {
         this.contextInitializers.add(loadInitializer(className, wac));
      }
   }

   AnnotationAwareOrderComparator.sort(this.contextInitializers);
   for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
      initializer.initialize(wac);
   }
}
```

FrameworkServlet#findWebApplicationContext

```java
@Nullable
protected WebApplicationContext findWebApplicationContext() {
   // 获取上下文属性
   String attrName = getContextAttribute();
   if (attrName == null) {
      return null;
   }
   // 通过上下文属性（如果不为空）去获取WebApplicationContext实例
   WebApplicationContext wac =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName);
   if (wac == null) {
      throw new IllegalStateException("No WebApplicationContext found: initializer not registered?");
   }
   return wac;
}
```

FrameworkServlet#createWebApplicationContext

```java
protected WebApplicationContext createWebApplicationContext(@Nullable WebApplicationContext parent) {
   return createWebApplicationContext((ApplicationContext) parent);
}
```

```java
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
   // 获取上下文类，默认为XmlWebApplicationContext
   Class<?> contextClass = getContextClass();
   if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
      throw new ApplicationContextException(
            "Fatal initialization error in servlet with name '" + getServletName() +
            "': custom WebApplicationContext class [" + contextClass.getName() +
            "] is not of type ConfigurableWebApplicationContext");
   }
   // 通过反射创建一个XmlWebApplicationContext
   ConfigurableWebApplicationContext wac =
         (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	 // 设置环境
   wac.setEnvironment(getEnvironment());
   // 设置父节点（可能为空）
   wac.setParent(parent);
   // 获取上下文配置的位置
   String configLocation = getContextConfigLocation();
   if (configLocation != null) {
      wac.setConfigLocation(configLocation);
   }
   configureAndRefreshWebApplicationContext(wac);

   return wac;
}
```

DispatcherServlet#onRefresh

```java
@Override
protected void onRefresh(ApplicationContext context) {
   // 执行各种初始化
   initStrategies(context);
}
```

DispatcherServlet#initStrategies

```java
protected void initStrategies(ApplicationContext context) {
   initMultipartResolver(context);
   initLocaleResolver(context);
   initThemeResolver(context);
   initHandlerMappings(context);
   initHandlerAdapters(context);
   initHandlerExceptionResolvers(context);
   initRequestToViewNameTranslator(context);
   initViewResolvers(context);
   initFlashMapManager(context);
}
```



**MultipartResolver**

解析器处理文件上传

DispatcherServlet#initMultipartResolver

```java
private void initMultipartResolver(ApplicationContext context) {
   try {
      // multipart解析器处理文件上传
      this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Detected " + this.multipartResolver);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Detected " + this.multipartResolver.getClass().getSimpleName());
      }
   }
   catch (NoSuchBeanDefinitionException ex) {
      // Default is no multipart resolver.
      this.multipartResolver = null;
      if (logger.isTraceEnabled()) {
         logger.trace("No MultipartResolver '" + MULTIPART_RESOLVER_BEAN_NAME + "' declared");
      }
   }
}
```



**LocaleResolver**

DispatcherServlet#initLocaleResolver

```java
private void initLocaleResolver(ApplicationContext context) {
   try {
      // 国际化LocaleResolver
      this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Detected " + this.localeResolver);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Detected " + this.localeResolver.getClass().getSimpleName());
      }
   }
   catch (NoSuchBeanDefinitionException ex) {
      // We need to use the default.
      // 使用默认值
      this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No LocaleResolver '" + LOCALE_RESOLVER_BEAN_NAME +
               "': using default [" + this.localeResolver.getClass().getSimpleName() + "]");
      }
   }
}
```

DispatcherServlet#getDefaultStrategy

```java
protected <T> T getDefaultStrategy(ApplicationContext context, Class<T> strategyInterface) {
   List<T> strategies = getDefaultStrategies(context, strategyInterface);
   // 不存在或存在多个就会抛出异常
   if (strategies.size() != 1) {
      throw new BeanInitializationException(
            "DispatcherServlet needs exactly 1 strategy for interface [" + strategyInterface.getName() + "]");
   }
   return strategies.get(0);
}
```

在分析getDefaultStrategies方法前，我们先来看下DispatcherServlet的静态代码块

```java
static {
   // Load default strategy implementations from properties file.
   // This is currently strictly internal and not meant to be customized
   // by application developers.
   try {
      // 获取DispatcherServlet相同目录下的DispatcherServlet.properties文件
      ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
      // 读入Properties对象
      defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
   }
   catch (IOException ex) {
      throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "': " + ex.getMessage());
   }
}
```

DispatcherServlet#getDefaultStrategies

```java
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
   String key = strategyInterface.getName();
   // 获取key为org.springframework.web.servlet.LocaleResolver的值
   String value = defaultStrategies.getProperty(key);
   if (value != null) {
      // 以','分割并转化为字符串数组
      String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
      List<T> strategies = new ArrayList<>(classNames.length);
      for (String className : classNames) {
         try {
            // 获取Class对象
            Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
            // 创建为bean对象
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

DispatcherServlet#createDefaultStrategy

```java
protected Object createDefaultStrategy(ApplicationContext context, Class<?> clazz) {
   // 开始创建bean（ioc的整个bean创建流程）
   return context.getAutowireCapableBeanFactory().createBean(clazz);
}
```

DispatcherServlet.properties的默认的LocaleResolver

```java
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
```



**ThemeResolver**

Spring MVC的主题就是一些静态资源的集合，即包括样式及图片，用来控制应用的视觉风格。

DispatcherServlet#initThemeResolver

```java
private void initThemeResolver(ApplicationContext context) {
   try {
      // 尝试去获取
      this.themeResolver = context.getBean(THEME_RESOLVER_BEAN_NAME, ThemeResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Detected " + this.themeResolver);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Detected " + this.themeResolver.getClass().getSimpleName());
      }
   }
   catch (NoSuchBeanDefinitionException ex) {
      // We need to use the default.
      // 创建一个默认的
      this.themeResolver = getDefaultStrategy(context, ThemeResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No ThemeResolver '" + THEME_RESOLVER_BEAN_NAME +
               "': using default [" + this.themeResolver.getClass().getSimpleName() + "]");
      }
   }
}
```

DispatcherServlet.properties的默认的ThemeResolver

```java
org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver
```



**HandlerMapping**

DispatcherServlet#initHandlerMappings

```java
private void initHandlerMappings(ApplicationContext context) {
   this.handlerMappings = null;
	 // 
   if (this.detectAllHandlerMappings) {
      // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
      // 在ApplicationContext获取所有的HandlerMappings，包括在祖先的contexts
      Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
      if (!matchingBeans.isEmpty()) {
         // 获取所有HandlerMapping类型的bean
         this.handlerMappings = new ArrayList<>(matchingBeans.values());
         // We keep HandlerMappings in sorted order.
         // 根据@order注解排序
         AnnotationAwareOrderComparator.sort(this.handlerMappings);
      }
   }
   else {
      try {
         // 根据beanName和HandlerMapping.class获取bean
         HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
         // 封装为List
         this.handlerMappings = Collections.singletonList(hm);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerMapping later.
      }
   }

   // Ensure we have at least one HandlerMapping, by registering
   // a default HandlerMapping if no other mappings are found.
	 // 通过注册确保至少有一个HandlerMapping
   // 如果未找到其他映射，则为默认的HandlerMapping
   if (this.handlerMappings == null) {
      this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

DispatcherServlet.properties的默认的HandlerMapping

```java
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
   org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
   org.springframework.web.servlet.function.support.RouterFunctionMapping
```



**HandlerAdapter**

DispatcherServlet#initHandlerAdapters

```java
private void initHandlerAdapters(ApplicationContext context) {
   this.handlerAdapters = null;

   if (this.detectAllHandlerAdapters) {
      // Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
      // 在ApplicationContext获取所有的HandlerAdapter，包括在祖先的contexts
      Map<String, HandlerAdapter> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerAdapters = new ArrayList<>(matchingBeans.values());
         // We keep HandlerAdapters in sorted order.
 				 // 根据@order注解排序
         AnnotationAwareOrderComparator.sort(this.handlerAdapters);
      }
   }
   else {
      try {
         // 根据beanName和HandlerAdapter.class获取bean
         HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
         // 封装为List
         this.handlerAdapters = Collections.singletonList(ha);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerAdapter later.
      }
   }

   // Ensure we have at least some HandlerAdapters, by registering
   // default HandlerAdapters if no other adapters are found.
   // 通过注册确保至少有一个HandlerAdapter
   // 如果未找到其他映射，则为默认的HandlerAdapter
   if (this.handlerAdapters == null) {
      this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

DispatcherServlet.properties的默认的HandlerAdapter

```java
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter
```



**HandlerExceptionResolver**

DispatcherServlet#initHandlerExceptionResolvers

```java
private void initHandlerExceptionResolvers(ApplicationContext context) {
   this.handlerExceptionResolvers = null;

   if (this.detectAllHandlerExceptionResolvers) {
      // Find all HandlerExceptionResolvers in the ApplicationContext, including ancestor contexts.
      // 在ApplicationContext获取所有的HandlerExceptionResolver，包括在祖先的contexts
      Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
            .beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerExceptionResolvers = new ArrayList<>(matchingBeans.values());
         // We keep HandlerExceptionResolvers in sorted order.
         // 根据@order注解排序
         AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
      }
   }
   else {
      try {
         // 根据beanName和HandlerExceptionResolver.class获取bean
         HandlerExceptionResolver her =
               context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
         // 封装为List
         this.handlerExceptionResolvers = Collections.singletonList(her);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, no HandlerExceptionResolver is fine too.
      }
   }

   // Ensure we have at least some HandlerExceptionResolvers, by registering
   // default HandlerExceptionResolvers if no other resolvers are found.
   // 通过注册确保至少有一个HandlerExceptionResolver
   // 如果未找到其他映射，则为默认的HandlerExceptionResolver
   if (this.handlerExceptionResolvers == null) {
      this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerExceptionResolvers declared in servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

DispatcherServlet.properties的默认的HandlerExceptionResolver

```java
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```



**RequestToViewNameTranslator**

DispatcherServlet#initRequestToViewNameTranslator

```java
private void initRequestToViewNameTranslator(ApplicationContext context) {
   try {
      // 尝试去获取
      this.viewNameTranslator =
            context.getBean(REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME, RequestToViewNameTranslator.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Detected " + this.viewNameTranslator.getClass().getSimpleName());
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Detected " + this.viewNameTranslator);
      }
   }
   catch (NoSuchBeanDefinitionException ex) {
      // We need to use the default.
      // 创建一个默认的
      this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No RequestToViewNameTranslator '" + REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME +
               "': using default [" + this.viewNameTranslator.getClass().getSimpleName() + "]");
      }
   }
}
```

DispatcherServlet.properties的默认的RequestToViewNameTranslator

```java
org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
```



**ViewResolver**

DispatcherServlet#initViewResolvers

```java
private void initViewResolvers(ApplicationContext context) {
   this.viewResolvers = null;

   if (this.detectAllViewResolvers) {
      // Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
      // 在ApplicationContext获取所有的ViewResolver，包括在祖先的contexts
      Map<String, ViewResolver> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.viewResolvers = new ArrayList<>(matchingBeans.values());
         // We keep ViewResolvers in sorted order.
         // 根据@order注解排序
         AnnotationAwareOrderComparator.sort(this.viewResolvers);
      }
   }
   else {
      try {
         // 根据beanName和ViewResolver.class获取bean
         ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
         this.viewResolvers = Collections.singletonList(vr);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default ViewResolver later.
      }
   }

   // Ensure we have at least one ViewResolver, by registering
   // a default ViewResolver if no other resolvers are found.
   // 通过注册确保至少有一个ViewResolver
   // 如果未找到其他映射，则为默认的ViewResolver
   if (this.viewResolvers == null) {
      this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No ViewResolvers declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

DispatcherServlet.properties的默认的InternalResourceViewResolver

```java
org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver
```



**SessionFlashMapManager**

DispatcherServlet#initFlashMapManager

```java
private void initFlashMapManager(ApplicationContext context) {
   try {
      // 尝试去获取
      this.flashMapManager = context.getBean(FLASH_MAP_MANAGER_BEAN_NAME, FlashMapManager.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Detected " + this.flashMapManager.getClass().getSimpleName());
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Detected " + this.flashMapManager);
      }
   }
   catch (NoSuchBeanDefinitionException ex) {
      // We need to use the default.
      // 创建一个默认的
      this.flashMapManager = getDefaultStrategy(context, FlashMapManager.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No FlashMapManager '" + FLASH_MAP_MANAGER_BEAN_NAME +
               "': using default [" + this.flashMapManager.getClass().getSimpleName() + "]");
      }
   }
}
```

DispatcherServlet.properties的默认的SessionFlashMapManager

```java
org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```



