# BeanNameUrlHandlerMapping

本篇介绍BeanNameUrlHandlerMapping初始化流程

BeanNameUrlHandlerMapping#setApplicationContext

```java
@Override
public final void setApplicationContext(@Nullable ApplicationContext context) throws BeansException {
   if (context == null && !isContextRequired()) {
      // Reset internal context state.
      this.applicationContext = null;
      this.messageSourceAccessor = null;
   }
   else if (this.applicationContext == null) {
      // Initialize with passed-in context.
      if (!requiredContextClass().isInstance(context)) {
         throw new ApplicationContextException(
               "Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
      }
      this.applicationContext = context;
      this.messageSourceAccessor = new MessageSourceAccessor(context);
      initApplicationContext(context);
   }
   else {
      // Ignore reinitialization if same context passed in.
      if (this.applicationContext != context) {
         throw new ApplicationContextException(
               "Cannot reinitialize with different application context: current one is [" +
               this.applicationContext + "], passed-in one is [" + context + "]");
      }
   }
}
```

> BeanNameUrlHandlerMapping前面提到过，会被注册为bean，因为继承了ApplicationContextAware，所以会自动执行setApplicationContext方法（回顾之前的ioc流程）

BeanNameUrlHandlerMapping#initApplicationContext

```java
@Override
public void initApplicationContext() throws ApplicationContextException {
   super.initApplicationContext();
   detectHandlers();
}
```

AbstractDetectingUrlHandlerMapping#detectHandlers

```java
protected void detectHandlers() throws BeansException {
   ApplicationContext applicationContext = obtainApplicationContext();
   // 获取所有的bean
   String[] beanNames = (this.detectHandlersInAncestorContexts ?
         BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
         applicationContext.getBeanNamesForType(Object.class));

   // Take any bean name that we can determine URLs for.
   // 遍历所有的bean
   for (String beanName : beanNames) {
      String[] urls = determineUrlsForHandler(beanName);
      if (!ObjectUtils.isEmpty(urls)) {
         // URL paths found: Let's consider it a handler.
         // key
         registerHandler(urls, beanName);
      }
   }

   if ((logger.isDebugEnabled() && !getHandlerMap().isEmpty()) || logger.isTraceEnabled()) {
      logger.debug("Detected " + getHandlerMap().size() + " mappings in " + formatMappingName());
   }
}
```

BeanNameUrlHandlerMapping#determineUrlsForHandler

```java
@Override
protected String[] determineUrlsForHandler(String beanName) {
   List<String> urls = new ArrayList<>();
   // bean名称以'/'开头
   if (beanName.startsWith("/")) {
      urls.add(beanName);
   }
   // 获取bean的相关别名，以'/'开头的添加到urls中并返回
   String[] aliases = obtainApplicationContext().getAliases(beanName);
   for (String alias : aliases) {
      if (alias.startsWith("/")) {
         urls.add(alias);
      }
   }
   return StringUtils.toStringArray(urls);
}
```

AbstractUrlHandlerMapping#registerHandler

```java
protected void registerHandler(String[] urlPaths, String beanName) throws BeansException, IllegalStateException {
   Assert.notNull(urlPaths, "URL path array must not be null");
   for (String urlPath : urlPaths) {
      registerHandler(urlPath, beanName);
   }
}
```

AbstractUrlHandlerMapping#registerHandler

```java
protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
   Assert.notNull(urlPath, "URL path must not be null");
   Assert.notNull(handler, "Handler object must not be null");
   Object resolvedHandler = handler;

   // Eagerly resolve handler if referencing singleton via name.
   // this.lazyInitHandlers默认值为false
   if (!this.lazyInitHandlers && handler instanceof String) {
      String handlerName = (String) handler;
      ApplicationContext applicationContext = obtainApplicationContext();
      // 判断是否为该bean是否为单例
      if (applicationContext.isSingleton(handlerName)) {
         // 获取bean
         resolvedHandler = applicationContext.getBean(handlerName);
      }
   }
	 // 获取mappedHandler
   Object mappedHandler = this.handlerMap.get(urlPath);
   if (mappedHandler != null) {
      if (mappedHandler != resolvedHandler) {
         // mappedHandler不为空 && mappedHandler！= resolvedHandler（说明存在2个不同的'/${beanName}'的bean）就会抛出异常
         throw new IllegalStateException(
               "Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
               "]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
      }
   }
   else {
      if (urlPath.equals("/")) {
         // beanName为'/'设置rootHandler
         if (logger.isTraceEnabled()) {
            logger.trace("Root mapping to " + getHandlerDescription(handler));
         }
         setRootHandler(resolvedHandler);
      }
      else if (urlPath.equals("/*")) {
         if (logger.isTraceEnabled()) {
            logger.trace("Default mapping to " + getHandlerDescription(handler));
         }
         // beanName为'*'设置rootHandler
         setDefaultHandler(resolvedHandler);
      }
      else {
         // 正常设置
         this.handlerMap.put(urlPath, resolvedHandler);
         if (logger.isTraceEnabled()) {
            logger.trace("Mapped [" + urlPath + "] onto " + getHandlerDescription(handler));
         }
      }
   }
}
```

> key为以'/'开头的beanName， 单例时value为bean对象，非单例时value还是为字符串