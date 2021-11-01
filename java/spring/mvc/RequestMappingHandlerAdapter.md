# RequestMappingHandlerAdapter

RequestMappingHandlerAdapter#afterPropertiesSet1

```java
@Override
public void afterPropertiesSet() {
   // Do this first, it may add ResponseBody advice beans
   // 首先解析@ControllerAdvice的bean
   initControllerAdviceCache();
	 // 参数解析器
   if (this.argumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
      this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   // 参数绑定解析起
   if (this.initBinderArgumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
      this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   // 返回值处理器
   if (this.returnValueHandlers == null) {
      List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
      this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   }
}
```

RequestMappingHandlerAdapter#initControllerAdviceCache

```java
private void initControllerAdviceCache() {
   if (getApplicationContext() == null) {
      return;
   }
   // 遍历工程中的bean, 获取bean上面的ControllerAdvice注解（包括包含@ControllerAdvice的注解）
   List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());

   List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

   for (ControllerAdviceBean adviceBean : adviceBeans) {
      Class<?> beanType = adviceBean.getBeanType();
      if (beanType == null) {
         throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
      }
      // 方法上拥有@ModelAttribute但不含@RequestMapping注解的方法
      Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
      if (!attrMethods.isEmpty()) {
         this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
      }
      // 获取bean上（包括其实现的接口）的@InitBinder接口
      Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
      if (!binderMethods.isEmpty()) {
         // 放入缓存
         this.initBinderAdviceCache.put(adviceBean, binderMethods);
      }
      // beanType继承了RequestBodyAdvice || beanType继承了ResponseBodyAdvice
      if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
         // 添加到requestResponseBodyAdviceBeans
         requestResponseBodyAdviceBeans.add(adviceBean);
      }
   }
	
   if (!requestResponseBodyAdviceBeans.isEmpty()) {
      // 添加到requestResponseBodyAdvice
      this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
   }

   if (logger.isDebugEnabled()) {
      int modelSize = this.modelAttributeAdviceCache.size();
      int binderSize = this.initBinderAdviceCache.size();
      int reqCount = getBodyAdviceCount(RequestBodyAdvice.class);
      int resCount = getBodyAdviceCount(ResponseBodyAdvice.class);
      if (modelSize == 0 && binderSize == 0 && reqCount == 0 && resCount == 0) {
         logger.debug("ControllerAdvice beans: none");
      }
      else {
         logger.debug("ControllerAdvice beans: " + modelSize + " @ModelAttribute, " + binderSize +
               " @InitBinder, " + reqCount + " RequestBodyAdvice, " + resCount + " ResponseBodyAdvice");
      }
   }
}
```

ControllerAdviceBean#findAnnotatedBeans

```java
public static List<ControllerAdviceBean> findAnnotatedBeans(ApplicationContext context) {
   List<ControllerAdviceBean> adviceBeans = new ArrayList<>();
   // 从bean工厂中获取所有的bean
   for (String name : BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, Object.class)) {
      // 非代理目标
      if (!ScopedProxyUtils.isScopedTarget(name)) {
         // 获取bean上面的ControllerAdvice注解（包括包含ControllerAdvice的注解）
         ControllerAdvice controllerAdvice = context.findAnnotationOnBean(name, ControllerAdvice.class);
         if (controllerAdvice != null) {
            // Use the @ControllerAdvice annotation found by findAnnotationOnBean()
            // in order to avoid a subsequent lookup of the same annotation.
            adviceBeans.add(new ControllerAdviceBean(name, context, controllerAdvice));
         }
      }
   }
   // 根绝Order排序
   OrderComparator.sort(adviceBeans);
   return adviceBeans;
}
```

构造方法ControllerAdviceBean

```java
public ControllerAdviceBean(String beanName, BeanFactory beanFactory, @Nullable ControllerAdvice controllerAdvice) {
   Assert.hasText(beanName, "Bean name must contain text");
   Assert.notNull(beanFactory, "BeanFactory must not be null");
   Assert.isTrue(beanFactory.containsBean(beanName), () -> "BeanFactory [" + beanFactory +
         "] does not contain specified controller advice bean '" + beanName + "'");
	 // beanName
   this.beanOrName = beanName;
   // 是否为单例
   this.isSingleton = beanFactory.isSingleton(beanName);
   // beanType
   this.beanType = getBeanType(beanName, beanFactory);
   // 创建一个HandlerTypePredicate对象
   this.beanTypePredicate = (controllerAdvice != null ? createBeanTypePredicate(controllerAdvice) :
         createBeanTypePredicate(this.beanType));
   // beanFactory
   this.beanFactory = beanFactory;
}
```

ControllerAdviceBean#createBeanTypePredicate

```java
private static HandlerTypePredicate createBeanTypePredicate(@Nullable ControllerAdvice controllerAdvice) {
   if (controllerAdvice != null) {
      return HandlerTypePredicate.builder()
            .basePackage(controllerAdvice.basePackages())
            .basePackageClass(controllerAdvice.basePackageClasses())
            .assignableType(controllerAdvice.assignableTypes())
            .annotation(controllerAdvice.annotations())
            .build();
   }
   return HandlerTypePredicate.forAnyHandlerType();
}
```



**介绍完初始化流程，下面我们介绍何时该bean进行初始化呢？**

1.当没有启用@EnableWebMvc的时候，入下面这段代码

```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {
}
```

> 没有配置上面这种配置的时候

 DispatcherServlet#initHandlerAdapters(context);会去读取DispatcherServlet.properties中的配置好的类并进行初始化



2.当启动@EnableWebMvc，RequestMappingHandlerMapping配装流程如下（还将介绍主要配置）

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
// 导出DelegatingWebMvcConfiguration该类
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

DelegatingWebMvcConfiguration

```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
   // 方法省略...
}
```

WebMvcConfigurationSupport#requestMappingHandlerAdapter

```java
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
      @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
      @Qualifier("mvcConversionService") FormattingConversionService conversionService,
      @Qualifier("mvcValidator") Validator validator) {

   RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
   adapter.setContentNegotiationManager(contentNegotiationManager);
   // 添加继承于WebMvcConfigurer的添加的消息转化器
   adapter.setMessageConverters(getMessageConverters());
   adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer(conversionService, validator));
   adapter.setCustomArgumentResolvers(getArgumentResolvers());
   adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

   if (jackson2Present) {
      adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
      adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
   }

   AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
   configureAsyncSupport(configurer);
   if (configurer.getTaskExecutor() != null) {
      adapter.setTaskExecutor(configurer.getTaskExecutor());
   }
   if (configurer.getTimeout() != null) {
      adapter.setAsyncRequestTimeout(configurer.getTimeout());
   }
   adapter.setCallableInterceptors(configurer.getCallableInterceptors());
   adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

   return adapter;
}
```

> 因为先执行的ioc的整个流程，所以会先用这个方法进行实例化

```java
protected final List<HttpMessageConverter<?>> getMessageConverters() {
   if (this.messageConverters == null) {
      this.messageConverters = new ArrayList<>();
      configureMessageConverters(this.messageConverters);
      if (this.messageConverters.isEmpty()) {
         addDefaultHttpMessageConverters(this.messageConverters);
      }
      extendMessageConverters(this.messageConverters);
   }
   return this.messageConverters;
}
```

WebMvcConfigurationSupport#addDefaultHttpMessageConverters

```java
protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
   // 字节数组消息转化器
   messageConverters.add(new ByteArrayHttpMessageConverter());
   // 字符串消息转化器
   messageConverters.add(new StringHttpMessageConverter());
   messageConverters.add(new ResourceHttpMessageConverter());
   messageConverters.add(new ResourceRegionHttpMessageConverter());
   try {
      messageConverters.add(new SourceHttpMessageConverter<>());
   }
   catch (Throwable ex) {
      // Ignore when no TransformerFactory implementation is available...
   }
   messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	 // 存在"com.rometools.rome.feed.WireFeed"
   if (romePresent) {
      messageConverters.add(new AtomFeedHttpMessageConverter());
      messageConverters.add(new RssChannelHttpMessageConverter());
   }
	 // 存在"com.fasterxml.jackson.dataformat.xml.XmlMapper"
   if (jackson2XmlPresent) {
      Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
      if (this.applicationContext != null) {
         builder.applicationContext(this.applicationContext);
      }
      messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
   }
   // 存在"javax.xml.bind.Binder"
   else if (jaxb2Present) {
      messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
   }
	 // 存在"com.fasterxml.jackson.databind.ObjectMapper" || "com.fasterxml.jackson.core.JsonGenerator"
   if (jackson2Present) {
      Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();
      if (this.applicationContext != null) {
         builder.applicationContext(this.applicationContext);
      }
      messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
   }
   // 存在"com.google.gson.Gson"
   else if (gsonPresent) {
      messageConverters.add(new GsonHttpMessageConverter());
   }
   // 存在"javax.json.bind.Json"
   else if (jsonbPresent) {
      messageConverters.add(new JsonbHttpMessageConverter());
   }
   // 存在"com.fasterxml.jackson.dataformat.smile.SmileFactory"
   if (jackson2SmilePresent) {
      Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.smile();
      if (this.applicationContext != null) {
         builder.applicationContext(this.applicationContext);
      }
      messageConverters.add(new MappingJackson2SmileHttpMessageConverter(builder.build()));
   }
   // 存在"com.fasterxml.jackson.dataformat.cbor.CBORFactory"
   if (jackson2CborPresent) {
      Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.cbor();
      if (this.applicationContext != null) {
         builder.applicationContext(this.applicationContext);
      }
      messageConverters.add(new MappingJackson2CborHttpMessageConverter(builder.build()));
   }
}
```

所以这里会默认为我们添加几个消息转化器

> ByteArrayHttpMessageConverter
> StringHttpMessageConverter -- 默认编码ISO-8859-1
> ResourceHttpMessageConverter
> ResourceRegionHttpMessageConverte
> SourceHttpMessageConverter
> AllEncompassingFormHttpMessageConverter
> Jaxb2RootElementHttpMessageConverter 
>
> Json转化器需要我们引入相应的依赖，MappingJackson2HttpMessageConverter才会被自动装配进去，默认编码
>
> ```java
> <!-- 引入下面依赖，jackson会被自动装配进去 -->
>   <dependency>
>     <groupId>com.fasterxml.jackson.core</groupId>
>     <artifactId>jackson-core</artifactId>
>     <version>${jackjson.version}</version>
>   </dependency>
>   <dependency>
>     <groupId>com.fasterxml.jackson.core</groupId>
>     <artifactId>jackson-annotations</artifactId>
>     <version>${jackjson.version}</version>
>   </dependency>
>   <dependency>
>     <groupId>com.fasterxml.jackson.core</groupId>
>     <artifactId>jackson-databind</artifactId>
>     <version>${jackjson.version}</version>
>   </dependency>
> ```
>
> 默认转化编码为utf-8，获取编码的逻辑在AbstractJackson2HttpMessageConverter#getJsonEncodingl，会去contentType中获取返回类型，如果为空，就默认采用UTF-8。

