# RequestMappingHandlerMapping

本篇介绍RequestMappingHandlerMapping，该类主要功能是在springmvc启动的时候，扫描@RequestMapping（PutMapping、PostMapping、GetMapping、DeleteMapping、PatchMapping）相关的方法，做一些相关的路径解析、注册、缓存映射关系。



下面直接开始从@EnableWebMvc注解开始进行源码分析

```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {
}
```

> 没有配置上面这种配置的时候

 DispatcherServlet#initHandlerMappings(context);会去读取DispatcherServlet.properties中的配置好的类并进行初始化



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

WebMvcConfigurationSupport#requestMappingHandlerMapping

```java
@Bean
@SuppressWarnings("deprecation")
public RequestMappingHandlerMapping requestMappingHandlerMapping(
      @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
      @Qualifier("mvcConversionService") FormattingConversionService conversionService,
      @Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {
	 // new RequestMappingHandlerMapping()
   RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
   // 设置order值
   mapping.setOrder(0);
   // 设置拦截器
   mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
   mapping.setContentNegotiationManager(contentNegotiationManager);
   // 设置跨越配置
   mapping.setCorsConfigurations(getCorsConfigurations());
	 // 获取PathMatchConfigurer
   PathMatchConfigurer configurer = getPathMatchConfigurer();

   Boolean useSuffixPatternMatch = configurer.isUseSuffixPatternMatch();
   if (useSuffixPatternMatch != null) {
      mapping.setUseSuffixPatternMatch(useSuffixPatternMatch);
   }
   Boolean useRegisteredSuffixPatternMatch = configurer.isUseRegisteredSuffixPatternMatch();
   if (useRegisteredSuffixPatternMatch != null) {
      mapping.setUseRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch);
   }
   Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
   if (useTrailingSlashMatch != null) {
      mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
   }

   UrlPathHelper pathHelper = configurer.getUrlPathHelper();
   if (pathHelper != null) {
      mapping.setUrlPathHelper(pathHelper);
   }
   PathMatcher pathMatcher = configurer.getPathMatcher();
   if (pathMatcher != null) {
      mapping.setPathMatcher(pathMatcher);
   }
   Map<String, Predicate<Class<?>>> pathPrefixes = configurer.getPathPrefixes();
   if (pathPrefixes != null) {
      mapping.setPathPrefixes(pathPrefixes);
   }

   return mapping;
}
```



**主要配置**

拦截器获取getInterceptors#getInterceptors

```java
protected final Object[] getInterceptors(
      FormattingConversionService mvcConversionService,
      ResourceUrlProvider mvcResourceUrlProvider) {
   if (this.interceptors == null) {
      InterceptorRegistry registry = new InterceptorRegistry();
      // WebMvcConfigurer子类的bean中添加的拦截器
      addInterceptors(registry);
      // 添加ConversionServiceExposingInterceptor拦截器
      registry.addInterceptor(new ConversionServiceExposingInterceptor(mvcConversionService));
      // 添加ResourceUrlProviderExposingInterceptor拦截器
      registry.addInterceptor(new ResourceUrlProviderExposingInterceptor(mvcResourceUrlProvider));
      // 对拦截器根据InterceptorRegistry的order值进行排序并返回
      this.interceptors = registry.getInterceptors();
   }
   return this.interceptors.toArray();
}
```

InterceptorRegistry#addInterceptor

```java
public InterceptorRegistration addInterceptor(HandlerInterceptor interceptor) {
   // 拦截器被包装为InterceptorRegistration
   InterceptorRegistration registration = new InterceptorRegistration(interceptor);
   this.registrations.add(registration);
   return registration;
}
```

**跨域配置**

WebMvcConfigurationSupport#getCorsConfigurations

```java
protected final Map<String, CorsConfiguration> getCorsConfigurations() {
   if (this.corsConfigurations == null) {
      CorsRegistry registry = new CorsRegistry();
      // WebMvcConfigurer子类的bean中添加的跨域配置
      addCorsMappings(registry);
     
      this.corsConfigurations = registry.getCorsConfigurations();
   }
   return this.corsConfigurations;
}
```

```java
public class CorsRegistry {

   private final List<CorsRegistration> registrations = new ArrayList<>();


   /**
    * Enable cross-origin request handling for the specified path pattern.
    *
    * <p>Exact path mapping URIs (such as {@code "/admin"}) are supported as
    * well as Ant-style path patterns (such as {@code "/admin/**"}).
    *
    * <p>By default, the {@code CorsConfiguration} for this mapping is
    * initialized with default values as described in
    * {@link CorsConfiguration#applyPermitDefaultValues()}.
    */
   public CorsRegistration addMapping(String pathPattern) {
      // 创建CorsRegistration
      CorsRegistration registration = new CorsRegistration(pathPattern);
      this.registrations.add(registration);
      return registration;
   }

   /**
    * Return the registered {@link CorsConfiguration} objects,
    * keyed by path pattern.
    */
   protected Map<String, CorsConfiguration> getCorsConfigurations() {
      Map<String, CorsConfiguration> configs = new LinkedHashMap<>(this.registrations.size());
      for (CorsRegistration registration : this.registrations) {
         // key为跨域路径，value为跨域配置信息
         configs.put(registration.getPathPattern(), registration.getCorsConfiguration());
      }
      return configs;
   }

}
```

CorsRegistration

```java
public CorsRegistration(String pathPattern) {
   this.pathPattern = pathPattern;
   // Same implicit default values as the @CrossOrigin annotation + allows simple methods
   // 创建跨域配置并设置默认值
   this.config = new CorsConfiguration().applyPermitDefaultValues();
}
```

CorsRegistration#applyPermitDefaultValues

```java
public CorsConfiguration applyPermitDefaultValues() {
   if (this.allowedOrigins == null) {
      // *
      this.allowedOrigins = DEFAULT_PERMIT_ALL;
   }
   if (this.allowedMethods == null) {
      // GET HEAD POST
      this.allowedMethods = DEFAULT_PERMIT_METHODS;
      // GET HEAD POST
      this.resolvedMethods = DEFAULT_PERMIT_METHODS
            .stream().map(HttpMethod::resolve).collect(Collectors.toList());
   }
   if (this.allowedHeaders == null) {
      // *
      this.allowedHeaders = DEFAULT_PERMIT_ALL;
   }
   if (this.maxAge == null) {
      // session时效性30s
      this.maxAge = 1800L;
   }
   return this;
}
```

AbstractHandlerMapping#setCorsConfigurations

```java
public void setCorsConfigurations(Map<String, CorsConfiguration> corsConfigurations) {
   Assert.notNull(corsConfigurations, "corsConfigurations must not be null");
   if (!corsConfigurations.isEmpty()) {
      UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
      source.setCorsConfigurations(corsConfigurations);
      // 设置pathMatcher，默认值为new AntPathMatcher()
      source.setPathMatcher(this.pathMatcher);
      // 设置urlPathHelper，默认值为new UrlPathHelper()
      source.setUrlPathHelper(this.urlPathHelper);
      source.setLookupPathAttributeName(LOOKUP_PATH);
      // 设置corsConfigurationSource
      this.corsConfigurationSource = source;
   }
   else {
      this.corsConfigurationSource = null;
   }
}
```



**这里介绍RequestMappingHandlerMapping的初始化流程**

RequestMappingHandlerMapping继承了InitializingBean，所以在初始化之前会调用afterPropertiesSet

RequestMappingHandlerMapping#afterPropertiesSet

```java
@Override
@SuppressWarnings("deprecation")
public void afterPropertiesSet() {
   // 设置一些config属性
   this.config = new RequestMappingInfo.BuilderConfiguration();
   this.config.setUrlPathHelper(getUrlPathHelper());
   this.config.setPathMatcher(getPathMatcher());
   this.config.setSuffixPatternMatch(useSuffixPatternMatch());
   this.config.setTrailingSlashMatch(useTrailingSlashMatch());
   this.config.setRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch());
   this.config.setContentNegotiationManager(getContentNegotiationManager());

   super.afterPropertiesSet();
}
```

AbstractHandlerMethodMapping#afterPropertiesSet

```java
@Override
public void afterPropertiesSet() {
   initHandlerMethods();
}
```

AbstractHandlerMethodMapping#initHandlerMethods

```java
protected void initHandlerMethods() {
   // 获取所有的bean
   for (String beanName : getCandidateBeanNames()) {
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
         // 非"scopedTarget."开头进行处理
         processCandidateBean(beanName);
      }
   }
   // 留给子类扩展
   handlerMethodsInitialized(getHandlerMethods());
}
```

AbstractHandlerMethodMapping#processCandidateBean

```java
protected void processCandidateBean(String beanName) {
   Class<?> beanType = null;
   try {
      // 获取该类型
      beanType = obtainApplicationContext().getType(beanName);
   }
   catch (Throwable ex) {
      // An unresolvable bean type, probably from a lazy bean - let's ignore it.
      if (logger.isTraceEnabled()) {
         logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
      }
   }
   // 该类上拥有@Controller或@RequestMapping注解
   if (beanType != null && isHandler(beanType)) {
      detectHandlerMethods(beanName);
   }
}
```

AbstractHandlerMethodMapping#detectHandlerMethods

```java
protected void detectHandlerMethods(Object handler) {
   // handler为String类型就去applicationContext中通过名称去取handlerType
   Class<?> handlerType = (handler instanceof String ?
         obtainApplicationContext().getType((String) handler) : handler.getClass());
	
   if (handlerType != null) {
      // 返回给定类的用户定义类：通常只是给定类，但对于CGLIB生成的子类，则返回原始类
      Class<?> userType = ClassUtils.getUserClass(handlerType);
      // 遍历所有带有@RequestMapping注解的方法（包括父类），key为方法，value为RequestMappingInfo对象
      Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
               try {
                  return getMappingForMethod(method, userType);
               }
               catch (Throwable ex) {
                  throw new IllegalStateException("Invalid mapping on handler class [" +
                        userType.getName() + "]: " + method, ex);
               }
            });
      if (logger.isTraceEnabled()) {
         logger.trace(formatMappings(userType, methods));
      }
      // key为Method， value为RequestMapping中解析出来并封装为RequestMappingInfo对象
      methods.forEach((method, mapping) -> {
         // 包装为一个可执行的方法
         Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
         // 进行注册
         registerHandlerMethod(handler, invocableMethod, mapping);
      });
   }
}
```

MethodIntrospector#selectMethods

```java
public static <T> Map<Method, T> selectMethods(Class<?> targetType, final MetadataLookup<T> metadataLookup) {
   final Map<Method, T> methodMap = new LinkedHashMap<>();
   Set<Class<?>> handlerTypes = new LinkedHashSet<>();
   Class<?> specificHandlerType = null;
	
   if (!Proxy.isProxyClass(targetType)) {
      // 非代理类
      specificHandlerType = ClassUtils.getUserClass(targetType);
      handlerTypes.add(specificHandlerType);
   }
   // 添加该类上面所有的接口
   handlerTypes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetType));

   for (Class<?> currentHandlerType : handlerTypes) {
      final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);

      ReflectionUtils.doWithMethods(currentHandlerType, method -> {
         Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
         T result = metadataLookup.inspect(specificMethod);
         if (result != null) {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
            if (bridgedMethod == specificMethod || metadataLookup.inspect(bridgedMethod) == null) {
               methodMap.put(specificMethod, result);
            }
         }
      }, ReflectionUtils.USER_DECLARED_METHODS);
   }

   return methodMap;
}
```

RequestMappingHandlerMapping#getMappingForMethod

```java
@Override
@Nullable
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
   // 先用method构建一个RequestMappingInfo对象
   RequestMappingInfo info = createRequestMappingInfo(method);
   if (info != null) {
      // 用handlerType类构建一个RequestMappingInfo对象
      RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
      if (typeInfo != null) {
         // 这里就是合并类上定义RequestMappingInfo和方法定义的RequestMappingInfo
         info = typeInfo.combine(info);
      }
      String prefix = getPathPrefix(handlerType);
      if (prefix != null) {
         // 如果存在PathPrefix，使用PathPrefix构建一个RequestMappingInfo对象，并和info合并
         info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
      }
   }
   return info;
}
```

RequestMappingHandlerMapping#createRequestMappingInfo

```java
@Nullable
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
   // 获取方法上的RequestMapping注解（PutMapping、PostMapping、GetMapping、DeleteMapping、PatchMapping也能获取到）
   RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
   // element此时为Method，所以此时condition=null
   RequestCondition<?> condition = (element instanceof Class ?
         getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
   //
   return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}
```

RequestMappingHandlerMapping#createRequestMappingInfo

```java
protected RequestMappingInfo createRequestMappingInfo(
      RequestMapping requestMapping, @Nullable RequestCondition<?> customCondition) {
	 // 获取requestMapping属性到RequestMappingInfo.Builder对象中
   RequestMappingInfo.Builder builder = RequestMappingInfo
         .paths(resolveEmbeddedValuesInPatterns(requestMapping.path()))
         .methods(requestMapping.method())
         .params(requestMapping.params())
         .headers(requestMapping.headers())
         .consumes(requestMapping.consumes())
         .produces(requestMapping.produces())
         .mappingName(requestMapping.name());
   if (customCondition != null) {
      builder.customCondition(customCondition);
   }
   // 并创建一个RequestMappingInfo对象
   return builder.options(this.config).build();
}
```

RequestMappingHandlerMapping#resolveEmbeddedValuesInPatterns

```java
protected String[] resolveEmbeddedValuesInPatterns(String[] patterns) {
   if (this.embeddedValueResolver == null) {
      return patterns;
   }
   else {
      String[] resolvedPatterns = new String[patterns.length];
      for (int i = 0; i < patterns.length; i++) {
         // 调用EmbeddedValueResolver进行解析字符串
         resolvedPatterns[i] = this.embeddedValueResolver.resolveStringValue(patterns[i]);
      }
      return resolvedPatterns;
   }
}
```

RequestMappingInfo.DefaultBuilder#build

```java
@Override
@SuppressWarnings("deprecation")
public RequestMappingInfo build() {
	 // 构造一个PatternsRequestCondition对象
   PatternsRequestCondition patternsCondition = ObjectUtils.isEmpty(this.paths) ? null :
         new PatternsRequestCondition(
               this.paths, this.options.getUrlPathHelper(), this.options.getPathMatcher(),
               this.options.useSuffixPatternMatch(), this.options.useTrailingSlashMatch(),
               this.options.getFileExtensions());

   ContentNegotiationManager manager = this.options.getContentNegotiationManager();
	 // 构造RequestMappingInfo对象
   return new RequestMappingInfo(this.mappingName, patternsCondition,
         ObjectUtils.isEmpty(this.methods) ?
               null : new RequestMethodsRequestCondition(this.methods),
         ObjectUtils.isEmpty(this.params) ?
               null : new ParamsRequestCondition(this.params),
         ObjectUtils.isEmpty(this.headers) ?
               null : new HeadersRequestCondition(this.headers),
         ObjectUtils.isEmpty(this.consumes) && !this.hasContentType ?
               null : new ConsumesRequestCondition(this.consumes, this.headers),
         ObjectUtils.isEmpty(this.produces) && !this.hasAccept ?
               null : new ProducesRequestCondition(this.produces, this.headers, manager),
         this.customCondition);
}
```

RequestMappingInfo.DefaultBuilder#combine

```java
@Override
public RequestMappingInfo combine(RequestMappingInfo other) {
   // 合并名称this.name + # + other.name
   String name = combineNames(other);
   // 合并路径
   PatternsRequestCondition patterns = this.patternsCondition.combine(other.patternsCondition);
   // set.addAll(other.methodsCondition.methods);
   RequestMethodsRequestCondition methods = this.methodsCondition.combine(other.methodsCondition);
   // set.addAll(other.paramsCondition.expressions);
   ParamsRequestCondition params = this.paramsCondition.combine(other.paramsCondition);
   // set.addAll(other.headersCondition.expressions);
   HeadersRequestCondition headers = this.headersCondition.combine(other.headersCondition);
   // (!other.expressions.isEmpty() ? other : this);
   ConsumesRequestCondition consumes = this.consumesCondition.combine(other.consumesCondition);
   // (!other.expressions.isEmpty() ? other : this);
   ProducesRequestCondition produces = this.producesCondition.combine(other.producesCondition);
   // 合并RequestCondition
   RequestConditionHolder custom = this.customConditionHolder.combine(other.customConditionHolder);
	 // 合并之后创建一个新的对象
   return new RequestMappingInfo(name, patterns,
         methods, params, headers, consumes, produces, custom.getCondition());
}
```

RequestMappingHandlerMapping#registerHandlerMethod

```java
@Override
protected void registerHandlerMethod(Object handler, Method method, RequestMappingInfo mapping) {
   super.registerHandlerMethod(handler, method, mapping);
   updateConsumesCondition(mapping, method);
}
```

AbstractHandlerMethodMapping#register

```java
public void register(T mapping, Object handler, Method method) {
   // Assert that the handler method is not a suspending one.
   if (KotlinDetector.isKotlinType(method.getDeclaringClass())) {
      Class<?>[] parameterTypes = method.getParameterTypes();
      if ((parameterTypes.length > 0) && "kotlin.coroutines.Continuation".equals(parameterTypes[parameterTypes.length - 1].getName())) {
         throw new IllegalStateException("Unsupported suspending handler method detected: " + method);
      }
   }
   this.readWriteLock.writeLock().lock();
   try {
      // 创建HandlerMethod
      HandlerMethod handlerMethod = createHandlerMethod(handler, method);
      // 校验是否有相同的路由地址
      validateMethodMapping(handlerMethod, mapping);
      // mappingLookup存放所有的路由地址, key为RequestMappingInfo对象, value为HandlerMethod对象
      this.mappingLookup.put(mapping, handlerMethod);
			// 获取直接链接
      List<String> directUrls = getDirectUrls(mapping);
      for (String url : directUrls) {
         this.urlLookup.add(url, mapping);
      }

      String name = null;
      if (getNamingStrategy() != null) {
         // 类大写字母 + '#' + 方法名
         name = getNamingStrategy().getName(handlerMethod, mapping);
         addMappingName(name, handlerMethod);
      }
			// 跨域的处理的解析
      CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
      if (corsConfig != null) {
         // HandlerMethod对象为key，value为@CrossOrigin配置的跨域参数
         this.corsLookup.put(handlerMethod, corsConfig);
      }
			// RequestMappingInfo对象为key
      this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
   }
   finally {
      this.readWriteLock.writeLock().unlock();
   }
}
```

> 这里就是最核心注册逻辑，解析@RequestMapping,mappingLookup存放所有的路有地址;
>
> urlLookup存放直接路由地址(带有/{}路径是非执行路由地址)
>
> corsLookup存放的是

RequestMappingInfoHandlerMethodMappingNamingStrategy#getName

```java
// handlerMethod对应的方法TestController#hello，返回结果为TC#hello
@Override
public String getName(HandlerMethod handlerMethod, RequestMappingInfo mapping) {
   if (mapping.getName() != null) {
      // mapping如果有定义name就直接返回
      return mapping.getName();
   }
   StringBuilder sb = new StringBuilder();
   String simpleTypeName = handlerMethod.getBeanType().getSimpleName();
   for (int i = 0; i < simpleTypeName.length(); i++) {
      if (Character.isUpperCase(simpleTypeName.charAt(i))) {
         // 大写字母就添加到sb
         sb.append(simpleTypeName.charAt(i));
      }
   }
   // 类大写字母 + '#' + 方法名
   sb.append(SEPARATOR).append(handlerMethod.getMethod().getName());
   return sb.toString();
}
```

AbstractHandlerMethodMapping#addMappingName

```java
private void addMappingName(String name, HandlerMethod handlerMethod) {
   // 获取旧的
   List<HandlerMethod> oldList = this.nameLookup.get(name);
   if (oldList == null) {
      oldList = Collections.emptyList();
   }

   for (HandlerMethod current : oldList) {
      if (handlerMethod.equals(current)) {
         // HandlerMethod存在就跳过
         return;
      }
   }
	 // 创建一个新的并添加handlerMethod
   List<HandlerMethod> newList = new ArrayList<>(oldList.size() + 1);
   newList.addAll(oldList);
   newList.add(handlerMethod);
   this.nameLookup.put(name, newList);
}
```

RequestMappingHandlerMapping#initCorsConfiguration

```java
@Override
protected CorsConfiguration initCorsConfiguration(Object handler, Method method, RequestMappingInfo mappingInfo) {
   HandlerMethod handlerMethod = createHandlerMethod(handler, method);
   Class<?> beanType = handlerMethod.getBeanType();
   // 找到类上面的CrossOrigin注解
   CrossOrigin typeAnnotation = AnnotatedElementUtils.findMergedAnnotation(beanType, CrossOrigin.class);
   // 找到method上面的CrossOrigin注解
   CrossOrigin methodAnnotation = AnnotatedElementUtils.findMergedAnnotation(method, CrossOrigin.class);

   if (typeAnnotation == null && methodAnnotation == null) {
      return null;
   }

   CorsConfiguration config = new CorsConfiguration();
  	// 先解析类上面的
   updateCorsConfig(config, typeAnnotation);
   // 再解析方法上的进行覆盖
   updateCorsConfig(config, methodAnnotation);
   if (CollectionUtils.isEmpty(config.getAllowedMethods())) {
      // @CrossOrigin.methods()为空
      // 将允许跨越的方法默认为@RequestMapping允许的方法
      for (RequestMethod allowedMethod : mappingInfo.getMethodsCondition().getMethods()) {
         config.addAllowedMethod(allowedMethod.name());
      }
   }
   // 配置默认参数
   return config.applyPermitDefaultValues();
}
```

RequestMappingHandlerMapping#updateCorsConfig

```java
private void updateCorsConfig(CorsConfiguration config, @Nullable CrossOrigin annotation) {
   if (annotation == null) {
      return;
   }
   // 解析origins值并添加到config
   for (String origin : annotation.origins()) {
      config.addAllowedOrigin(resolveCorsAnnotationValue(origin));
   }
   // 解析methods值并添加到config
   for (RequestMethod method : annotation.methods()) {
      config.addAllowedMethod(method.name());
   }
   // 解析allowedHeaders
   for (String header : annotation.allowedHeaders()) {
      config.addAllowedHeader(resolveCorsAnnotationValue(header));
   }
   // 解析exposedHeaders
   for (String header : annotation.exposedHeaders()) {
      config.addExposedHeader(resolveCorsAnnotationValue(header));
   }
	 // 浏览器是否将本域名下的cookie信息携带至跨域服务器中
   String allowCredentials = resolveCorsAnnotationValue(annotation.allowCredentials());
   if ("true".equalsIgnoreCase(allowCredentials)) {
      config.setAllowCredentials(true);
   }
   else if ("false".equalsIgnoreCase(allowCredentials)) {
      config.setAllowCredentials(false);
   }
   // 值只能true或者false
   else if (!allowCredentials.isEmpty()) {
      throw new IllegalStateException("@CrossOrigin's allowCredentials value must be \"true\", \"false\", " +
            "or an empty string (\"\"): current value is [" + allowCredentials + "]");
   }
	 // Cookie的有效期单位为秒
   if (annotation.maxAge() >= 0 && config.getMaxAge() == null) {
      config.setMaxAge(annotation.maxAge());
   }
}
```

RequestMappingHandlerMapping#updateConsumesCondition

```java
private void updateConsumesCondition(RequestMappingInfo info, Method method) {
   ConsumesRequestCondition condition = info.getConsumesCondition();
   if (!condition.isEmpty()) {
      for (Parameter parameter : method.getParameters()) {
         // 获取参数上的RequestBody注解
         MergedAnnotation<RequestBody> annot = MergedAnnotations.from(parameter).get(RequestBody.class);
         if (annot.isPresent()) {
            // 存在就设置bodyRequired为RequestBody上的value值
            condition.setBodyRequired(annot.getBoolean("required"));
            break;
         }
      }
   }
}
```

AbstractHandlerMethodMapping#handlerMethodsInitialized

```java
protected void handlerMethodsInitialized(Map<T, HandlerMethod> handlerMethods) {
   // Total includes detected mappings + explicit registrations via registerMapping
   int total = handlerMethods.size();
   if ((logger.isTraceEnabled() && total == 0) || (logger.isDebugEnabled() && total > 0) ) {
      // trace、debug级别进行日志打印
      logger.debug(total + " mappings in " + formatMappingName());
   }
}
```

以上就是所有扫描

RequestMappingInfoHandlerMapping#handleMatch

```java
@Override
protected void handleMatch(RequestMappingInfo info, String lookupPath, HttpServletRequest request) {
   // org.springframework.web.servlet.HandlerMapping+".pathWithinHandlerMapping"
   super.handleMatch(info, lookupPath, request);

   String bestPattern;
   Map<String, String> uriVariables;
	 // 获取映射路径
   Set<String> patterns = info.getPatternsCondition().getPatterns();
   if (patterns.isEmpty()) {
        bestPattern = lookupPath;
      uriVariables = Collections.emptyMap();
   }
   else {
      // 获取首个映射路径
      bestPattern = patterns.iterator().next();
      // 可变url转为
      uriVariables = getPathMatcher().extractUriTemplateVariables(bestPattern, lookupPath);
   }

   request.setAttribute(BEST_MATCHING_PATTERN_ATTRIBUTE, bestPattern);

   if (isMatrixVariableContentAvailable()) {
      Map<String, MultiValueMap<String, String>> matrixVars = extractMatrixVariables(request, uriVariables);
      request.setAttribute(HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE, matrixVars);
   }

   Map<String, String> decodedUriVariables = getUrlPathHelper().decodePathVariables(request, uriVariables);
   request.setAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, decodedUriVariables);
	 // produces上面的MediaType
   if (!info.getProducesCondition().getProducibleMediaTypes().isEmpty()) {
      Set<MediaType> mediaTypes = info.getProducesCondition().getProducibleMediaTypes();
      // 放入request
      request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE, mediaTypes);
   }
}
```
