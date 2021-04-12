# spring session源码分析

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
// spring session核心配置类
@Import(RedisHttpSessionConfiguration.class)
@Configuration(proxyBeanMethods = false)
public @interface EnableRedisHttpSession {
  //...
}
```

RedisHttpSessionConfiguration spring session启动类

```java
@Configuration(proxyBeanMethods = false)
public class RedisHttpSessionConfiguration extends SpringHttpSessionConfiguration
      implements BeanClassLoaderAware, EmbeddedValueResolverAware, ImportAware {
   /**
    * 配置RedisIndexedSessionRepository
    */
   @Bean
   public RedisIndexedSessionRepository sessionRepository() {
      RedisTemplate<Object, Object> redisTemplate = createRedisTemplate();
      RedisIndexedSessionRepository sessionRepository = new RedisIndexedSessionRepository(redisTemplate);
      sessionRepository.setApplicationEventPublisher(this.applicationEventPublisher);
      if (this.indexResolver != null) {
         sessionRepository.setIndexResolver(this.indexResolver);
      }
      if (this.defaultRedisSerializer != null) {
         sessionRepository.setDefaultSerializer(this.defaultRedisSerializer);
      }
      sessionRepository.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
      if (StringUtils.hasText(this.redisNamespace)) {
         sessionRepository.setRedisKeyNamespace(this.redisNamespace);
      }
      sessionRepository.setFlushMode(this.flushMode);
      sessionRepository.setSaveMode(this.saveMode);
      int database = resolveDatabase();
      sessionRepository.setDatabase(database);
      this.sessionRepositoryCustomizers
            .forEach((sessionRepositoryCustomizer) -> sessionRepositoryCustomizer.customize(sessionRepository));
      return sessionRepository;
   }
   // 省略其他配置和属性
}
```

SpringHttpSessionConfiguration为RedisHttpSessionConfiguration的父类，也会进行加载

```java
@Configuration(proxyBeanMethods = false)
public class SpringHttpSessionConfiguration implements ApplicationContextAware {
    @Bean
    public <S extends Session> SessionRepositoryFilter<? extends Session> springSessionRepositoryFilter(
          SessionRepository<S> sessionRepository) {
       SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter<>(sessionRepository);
       sessionRepositoryFilter.setHttpSessionIdResolver(this.httpSessionIdResolver);
       return sessionRepositoryFilter;
    }
      // 省略其他配置和属性
}
```

SessionRepositoryFilter过滤器就是核心的处理类

```java
@Order(SessionRepositoryFilter.DEFAULT_ORDER)
public class SessionRepositoryFilter<S extends Session> extends OncePerRequestFilter {
   /**
    * 这里就很明显啦， 该过滤器对request、repsonse对象进行了包装，后面的基于request、response操作的逻辑操作都是  
    * 对SessionRepositoryRequestWrappe和SessionRepositoryResponseWrapper对象进行操作
    */
   @Override
   protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
         throws ServletException, IOException {
      request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

      SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(request, response);
      SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(wrappedRequest,
            response);

      try {
         filterChain.doFilter(wrappedRequest, wrappedResponse);
      }
      finally {
         wrappedRequest.commitSession();
      }
   }
   // 省略其他代码和属性
}
```





SessionRepositoryFilter.SessionRepositoryRequestWrapper#getSession

```java
@Override
public HttpSessionWrapper getSession(boolean create) {
   // request中获取session
   HttpSessionWrapper currentSession = getCurrentSession();
   if (currentSession != null) {
      return currentSession;
   }
   // 获取session
   // 如果取缓存中的值，就返回SessionRepositoryRequestWrapper.requestedSession
   // 否则去redis中获取值，并将值缓存
   S requestedSession = getRequestedSession();
   if (requestedSession != null) {
      if (getAttribute(INVALID_SESSION_ID_ATTR) == null) {
         // 不存在无效sessionId的标记
         requestedSession.setLastAccessedTime(Instant.now());
         this.requestedSessionIdValid = true;
         currentSession = new HttpSessionWrapper(requestedSession, getServletContext());
         currentSession.markNotNew();
         // request进行缓存
         setCurrentSession(currentSession);
         return currentSession;
      }
   }
   else {
      // This is an invalid session id. No need to ask again if
      // request.getSession is invoked for the duration of this request
      if (SESSION_LOGGER.isDebugEnabled()) {
         SESSION_LOGGER.debug(
               "No session found by id: Caching result for getSession(false) for this HttpServletRequest.");
      }
      // 进行无效标记
      setAttribute(INVALID_SESSION_ID_ATTR, "true");
   }
   // 是否进行创建
   if (!create) {
      return null;
   }
   if (SESSION_LOGGER.isDebugEnabled()) {
      SESSION_LOGGER.debug(
            "A new session was created. To help you troubleshoot where the session was created we provided a StackTrace (this is not an error). You can prevent this from appearing by disabling DEBUG logging for "
                  + SESSION_LOGGER_NAME,
            new RuntimeException("For debugging purposes only (not an error)"));
   }
   // 创建session
   S session = SessionRepositoryFilter.this.sessionRepository.createSession();
   session.setLastAccessedTime(Instant.now());
   // 封装为HttpSessionWrapper对象
   currentSession = new HttpSessionWrapper(session, getServletContext());
   // request进行缓存
   setCurrentSession(currentSession);
   return currentSession;
}
```

SessionRepositoryFilter.SessionRepositoryRequestWrapper#getRequestedSession

```java
private S getRequestedSession() {
   if (!this.requestedSessionCached) {
      // 获取cookie中key为SESSION的值
      List<String> sessionIds = SessionRepositoryFilter.this.httpSessionIdResolver.resolveSessionIds(this);
      for (String sessionId : sessionIds) {
         if (this.requestedSessionId == null) {
            this.requestedSessionId = sessionId;
         }
         // 根据sessionId去redis中获取HashMap
         S session = SessionRepositoryFilter.this.sessionRepository.findById(sessionId);
         if (session != null) {
            // 进行缓存
            this.requestedSession = session;
            this.requestedSessionId = sessionId;
            break;
         }
      }
      // 将Session的值缓存起来
      this.requestedSessionCached = true;
   }
   return this.requestedSession;
}
```

> Redis中获取的HashMap到的值
>
> "maxInactiveInterval" -> {Integer@12352} 1800
> "sessionAttr:ceshi" -> "svssss"  该列值就是session.setAttribute("ceshi","svssss")
> "creationTime" -> {Long@12356} 1602665444973
> "lastAccessedTime" -> {Long@12358} 1602666398051

CookieHttpSessionIdResolver#resolveSessionIds

```java
@Override
public List<String> resolveSessionIds(HttpServletRequest request) {
   return this.cookieSerializer.readCookieValues(request);
}
```

CookieHttpSessionIdResolver#readCookieValues

```java
@Override
public List<String> readCookieValues(HttpServletRequest request) {
   Cookie[] cookies = request.getCookies();
   List<String> matchingCookieValues = new ArrayList<>();
   if (cookies != null) {
      // 遍历cookie
      for (Cookie cookie : cookies) {
         if (this.cookieName.equals(cookie.getName())) {
            // 获取cookie中key为SESSION的值
            String sessionId = (this.useBase64Encoding ? base64Decode(cookie.getValue()) : cookie.getValue());
            if (sessionId == null) {
               continue;
            }
            if (this.jvmRoute != null && sessionId.endsWith(this.jvmRoute)) {
               sessionId = sessionId.substring(0, sessionId.length() - this.jvmRoute.length());
            }
            matchingCookieValues.add(sessionId);
         }
      }
   }
   return matchingCookieValues;
}
```

RedisIndexedSessionRepository#findById

```java
@Override
public RedisSession findById(String id) {
   return getSession(id, false);
}
```

RedisIndexedSessionRepository#getSession

```java
private RedisSession getSession(String id, boolean allowExpired) {
   Map<Object, Object> entries = getSessionBoundHashOperations(id).entries();
   if (entries.isEmpty()) {
      return null;
   }
   MapSession loaded = loadSession(id, entries);
   if (!allowExpired && loaded.isExpired()) {
      return null;
   }
   // 创建一个RedisSession对象
   RedisSession result = new RedisSession(loaded, false);
   // 记录最后一次访问的时间
   result.originalLastAccessTime = loaded.getLastAccessedTime();
   return result;
}
```

RedisIndexedSessionRepository#loadSession

```java
private MapSession loadSession(String id, Map<Object, Object> entries) {
   MapSession loaded = new MapSession(id);
   for (Map.Entry<Object, Object> entry : entries.entrySet()) {
      String key = (String) entry.getKey();
      if (RedisSessionMapper.CREATION_TIME_KEY.equals(key)) {
         loaded.setCreationTime(Instant.ofEpochMilli((long) entry.getValue()));
      }
      else if (RedisSessionMapper.MAX_INACTIVE_INTERVAL_KEY.equals(key)) {
         loaded.setMaxInactiveInterval(Duration.ofSeconds((int) entry.getValue()));
      }
      else if (RedisSessionMapper.LAST_ACCESSED_TIME_KEY.equals(key)) {
         loaded.setLastAccessedTime(Instant.ofEpochMilli((long) entry.getValue()));
      }
      else if (key.startsWith(RedisSessionMapper.ATTRIBUTE_PREFIX)) {
         loaded.setAttribute(key.substring(RedisSessionMapper.ATTRIBUTE_PREFIX.length()), entry.getValue());
      }
   }
   return loaded;
}
```



在Response进行write之前，会执行下面这个方式

SessionRepositoryRequestWrapper#commitSession

```java
private void commitSession() {
   // 获取session， 此时应该是request中获取的
   HttpSessionWrapper wrappedSession = getCurrentSession();
   if (wrappedSession == null) {
      if (isInvalidateClientSession()) {
         // 无效session就进行cookie清除
         // 设置cookie的Max-Age=0
         SessionRepositoryFilter.this.httpSessionIdResolver.expireSession(this, this.response);
      }
   }
   else {
      // 获取RedisIndexedSessionRepository#RedisSession对象
      S session = wrappedSession.getSession();
      // 清除缓存
      // this.requestedSessionCached = false;
			// this.requestedSession = null;
			// this.requestedSessionId = null;
      clearRequestedSessionCache();
      // 
      SessionRepositoryFilter.this.sessionRepository.save(session);
      String sessionId = session.getId();
      if (!isRequestedSessionIdValid() || !sessionId.equals(getRequestedSessionId())) {
         SessionRepositoryFilter.this.httpSessionIdResolver.setSessionId(this, this.response, sessionId);
      }
   }
}
```

RedisIndexedSessionRepository.RedisSession.#save

```java
@Override
public void save(RedisSession session) {
   session.save();
   if (session.isNew) {
      String sessionCreatedKey = getSessionCreatedChannel(session.getId());
      this.sessionRedisOperations.convertAndSend(sessionCreatedKey, session.delta);
      session.isNew = false;
   }
}
```

RedisIndexedSessionRepository.RedisSession.save

```java
private void save() {
   saveChangeSessionId();
   //
   saveDelta();
}
```

RedisIndexedSessionRepository.RedisSession#saveChangeSessionId

```java
private void saveChangeSessionId() {
   // 获取sessionId
   String sessionId = getId();
   // 和之前的sessionId比较
   if (sessionId.equals(this.originalSessionId)) {
      return;
   }
   // 不是新的Session
   if (!this.isNew) {
      // 获取originalSessionIdKey
      String originalSessionIdKey = getSessionKey(this.originalSessionId);
      // 获取sessionIdKey
      String sessionIdKey = getSessionKey(sessionId);
      try {
         // 将originalSessionIdKey更新为sessionIdKey
         RedisIndexedSessionRepository.this.sessionRedisOperations.rename(originalSessionIdKey,
               sessionIdKey);
      }
      catch (NonTransientDataAccessException ex) {
         handleErrNoSuchKeyError(ex);
      }
      String originalExpiredKey = getExpiredKey(this.originalSessionId);
      String expiredKey = getExpiredKey(sessionId);
      try {
         RedisIndexedSessionRepository.this.sessionRedisOperations.rename(originalExpiredKey, expiredKey);
      }
      catch (NonTransientDataAccessException ex) {
         handleErrNoSuchKeyError(ex);
      }
   }
   this.originalSessionId = sessionId;
}
```

RedisIndexedSessionRepository.RedisSession#saveDelta

```java
private void saveDelta() {
   if (this.delta.isEmpty()) {
      return;
   }
   String sessionId = getId();
   // 保存delta
   getSessionBoundHashOperations(sessionId).putAll(this.delta);
   String principalSessionKey = getSessionAttrNameKey(
         FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME);
   String securityPrincipalSessionKey = getSessionAttrNameKey(SPRING_SECURITY_CONTEXT);
   // SPRING_SECURITY_CONTEXT部分逻辑
   if (this.delta.containsKey(principalSessionKey) || this.delta.containsKey(securityPrincipalSessionKey)) {
      if (this.originalPrincipalName != null) {
         String originalPrincipalRedisKey = getPrincipalKey(this.originalPrincipalName);
         RedisIndexedSessionRepository.this.sessionRedisOperations.boundSetOps(originalPrincipalRedisKey)
               .remove(sessionId);
      }
      Map<String, String> indexes = RedisIndexedSessionRepository.this.indexResolver.resolveIndexesFor(this);
      String principal = indexes.get(PRINCIPAL_NAME_INDEX_NAME);
      this.originalPrincipalName = principal;
      if (principal != null) {
         String principalRedisKey = getPrincipalKey(principal);
         RedisIndexedSessionRepository.this.sessionRedisOperations.boundSetOps(principalRedisKey)
               .add(sessionId);
      }
   }

   this.delta = new HashMap<>(this.delta.size());

   Long originalExpiration = (this.originalLastAccessTime != null)
         ? this.originalLastAccessTime.plus(getMaxInactiveInterval()).toEpochMilli() : null;
   // 
   RedisIndexedSessionRepository.this.expirationPolicy.onExpirationUpdated(originalExpiration, this);
}
```

下面是spring session存储的数据结构如下

```
127.0.0.1:6379> keys *
1) "spring:session:sessions:expires:170ff608-74f6-41bc-887d-d0b6b0fed680"
2) "spring:session:expirations:1603443360000"
3) "spring:session:sessions:170ff608-74f6-41bc-887d-d0b6b0fed680"
   // 保存的相关value值
   "sessionAttr:ceshi"
   "creationTime"
   "maxInactiveInterval"
   "lastAccessedTime"
```

