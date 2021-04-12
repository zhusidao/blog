# 基于接口查询的mybatis源码分析

前言：本篇会基于mybatis查询的源码进行分析，重点会讲解mybatis的一级缓存代码实现，以及会总结一级缓存所带来的问题。

启动类

```java
public class MainQuery {

    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
        // 从SqlSessionFactory对象中获取 SqlSession对象
        SqlSession sqlSession = factory.openSession();
        BookDao bookDao = sqlSession.getMapper(BookDao.class);
        // 执行操作
        System.out.println(bookDao.findById(1007));
        System.out.println(bookDao.findById(1007));
        // 提交操作
        sqlSession.commit();
        // 关闭SqlSession
        sqlSession.close();
    }
}
```

> SqlSession sqlSession = factory.openSession();上一篇一级分析过，这里不展开分析

DefaultSqlSession#getMapper进行接口实现的获取

```java
@Override
public <T> T getMapper(Class<T> type) {
  return configuration.getMapper(type, this);
}
```

Configuration#getMapper

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

MapperRegistry#getMapper

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

> 这里就可以看出最终通过mapperProxyFactory.newInstance(sqlSession)创建的一个代理对象

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
  // 使用jdk代理创建一个代理实例
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
  // 创建一个MapperProxy代理对象
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

> 也就是说，我们获取接口实现的时候，实际上获取的是一个MapperProxy代理对象
>

MapperProxy#invoke

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else {
      return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```

MapperProxy.PlainMethodInvoker#invoke接口的执行逻辑

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
  return mapperMethod.execute(sqlSession, args);
}
```

MapperMethod#execute这里就是整个基于接口查询的核心方法

```java
/**
 * 这个方法是对SqlSession的包装，对应insert、delete、update、select四种操作
 */
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  switch (command.getType()) {
    case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        // 如果返回void 并且参数有resultHandler  ,则调用 void select(String statement, Object parameter, ResultHandler handler);方法  
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        // 如果返回多行结果,executeForMany这个方法调用 <E> List<E> selectList(String statement, Object parameter);   
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        // 如果返回类型是MAP 则调用executeForMap方法 
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        // 只有一个参数
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
        if (method.returnsOptional()
            && (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName()
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```

DefaultSqlSession#selectOne

```java
@Override
public <T> T selectOne(String statement, Object parameter) {
  // Popular vote was to return null on 0 results and throw exception on too many.
  List<T> list = this.selectList(statement, parameter);
  if (list.size() == 1) {
    return list.get(0);
  } else if (list.size() > 1) {
    throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
  } else {
    return null;
  }
}
```

DefaultSqlSession#selectList

```java
@Override
public <E> List<E> selectList(String statement, Object parameter) {
  // RowBounds.DEFAULT为0~2147483647
  return this.selectList(statement, parameter, RowBounds.DEFAULT);
}
```

DefaultSqlSession#selectList

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    // 获取xml定义的Statement
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

CachingExecutor#query

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  // 创建缓存
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
  // 执行查询
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

CachingExecutor#createCacheKey

```java
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql);
}
```

BaseExecutor#createCacheKey

```java
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  // 创建一个缓存key
  CacheKey cacheKey = new CacheKey();
  // id、offset、limit、sql、参数、类型处理器
  cacheKey.update(ms.getId());
  cacheKey.update(rowBounds.getOffset());
  cacheKey.update(rowBounds.getLimit());
  cacheKey.update(boundSql.getSql());
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
  // mimic DefaultParameterHandler logic
  // 
  for (ParameterMapping parameterMapping : parameterMappings) {
    if (parameterMapping.getMode() != ParameterMode.OUT) {
      Object value;
      String propertyName = parameterMapping.getProperty();
      if (boundSql.hasAdditionalParameter(propertyName)) {
        value = boundSql.getAdditionalParameter(propertyName);
      } else if (parameterObject == null) {
        value = null;
      } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
        value = parameterObject;
      } else {
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        value = metaObject.getValue(propertyName);
      }
      cacheKey.update(value);
    }
  }
  if (configuration.getEnvironment() != null) {
    // issue #176
    // 环境
    cacheKey.update(configuration.getEnvironment().getId());
  }
  return cacheKey;
}
```

> 将`MappedStatement`的Id、SQL的offset、SQL的limit、SQL本身以及SQL中的参数传入了CacheKey这个类，最终构成CacheKey给一级缓存使用

CachingExecutor#query

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
  Cache cache = ms.getCache();
  if (cache != null) {
    // 缓存不为空
    // 刷新缓存，如果xml定义了flushCache=true
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      // 使用二级缓存
      // 直接从二级缓存中取，取不到就数据库查询
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

> 通过CachingExecutor装饰BaseExecutor从而实现二级缓存，二级缓存讲会放到后面进行分析



### 一级缓存

#### 如何启用

默认就是启用的

#### 特点

1. 作用范围：同一个sqlSession类生效，且如果发生update、insert、delete一级缓存就会失效
2. 基于本地内存缓存

接着上面的逻辑分析BaseExecutor#query

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    // 查询执行栈
    queryStack++;
    // 没有resultHandler则取缓存中的值
    // localCache为一级缓存
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      // 执行查询
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // issue #601
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // localCacheScope值为STATEMENT就直接清除一级缓存
      // issue #482
      clearLocalCache();
    }
  }
  return list;
}
```

> 看到这里其实有疑问， 为什么查询栈不用volatile去修饰呢？
>
> 一级缓存用localCache变量存储，数据结构就是一个HashMap

BaseExecutor#queryFromDatabase

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  // 标记为正在执行
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    // 执行查询
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    // 移除正在执行的标记
    localCache.removeObject(key);
  }
  // 缓存执行的结果
  localCache.putObject(key, list);
  // 默认类型为PREPARED
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

> 在mapper文件中可以使用statementType标记使用什么对象操作SQL语句。
>
> statementType：标记操作SQL的对象
>
> 取值说明：
>
> 1、STATEMENT：直接操作sql，不进行预编译，获取数据：$--Statement
>
> 2、PREPARED：预处理，参数，进行预编译，获取数据：#--PreparedStatement：默认
>
> 3、CALLABLE：执行存储过程—CallableStatement

SimpleExecutor#doQuery

```java
@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    // 获取配置
    Configuration configuration = ms.getConfiguration();
    // Statement处理器
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    // 预处理（获取数据库连接、sql预处理、设置参数），上篇已经介绍过，这里不展示解释
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行查询并返回结果集
    return handler.query(stmt, resultHandler);
  } finally {
    // statement.close();
    closeStatement(stmt);
  }
}
```

RoutingStatementHandler#query

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  return delegate.query(statement, resultHandler);
}
```

PreparedStatementHandler#query

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  // 执行查询
  ps.execute();
  // 封装结果集，不做介绍
  return resultSetHandler.handleResultSets(ps);
}
```

总结，

1. 注意在与spring整合的时候，需要加上@Transactional才会生效
2. mybatis如果有使用二级缓存会先使用二级缓存，后才使用一级缓存
3. MyBatis一级缓存的生命周期和SqlSession一致。
4. MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。
5. MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为Statement（默认为SESSION）。

这里我们给出mybatis缓存时序图

![mybatis缓存](/Users/zhusidao/Documents/wiki/zhusidao.github.io.wiki/images/mybatis缓存.jpg)



### 二级缓存

#### 如何开启二级缓存

一是在配置文件中开启，这是开启二级缓存的总开关，默认是开启状态的：

```xml
<setting name="cacheEnabled" value="true"/>
```

二是在Mapper文件中开启缓存，默认是不开启的，需要手动开启：

```xml
<!-- 每个Mapper.xml文件使用一个缓存对象 -->
<cache/>
<!-- 接口上也能开启，使用@CacheNamespace注解和<cache/>作用一样 -->
@CacheNamespace

<!-- 如果是多个Mapper文件共用一个缓存对象 -->
<cache-ref />
<!-- 接口上也能声明，使用@CacheNamespaceRef注解和<cache-ref />作用一样 -->
@CacheNamespaceRef
```

三是针对要查询的statement使用缓存，即在<select>节点中配置如下属性（默认就为true）：

```xml
useCache="true"
```



#### 二级缓存特点

1.当`sqlsession`没有调用`commit()`方法时，二级缓存并没有起到作用

2.当发生更新操作后，二级缓存会清除（insert、update、delete语句时，flushCache=true）

3.不同的`namespace`查出的数据都会将数据存在自己对应的缓存中，也就说会产生脏读的问题

4.使用二级缓存的pojo需要实现序列化

ANamespace读出data1，如果BNamespace对data1做出修改，A再一次去读data1就就发生脏读



我们接着上面的方法进行分析，下面对二级缓存做相关分析

CachingExecutor#query

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
  Cache cache = ms.getCache();
  if (cache != null) {
    // 如有有需要就清楚二级缓存
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      // 使用了二级缓存 && 不存在结果处理器
      // 这里是存储过程相关，这里忽略
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      // 获取二级缓存的值
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

> cache打断点后的结构图
>
> ![二级缓存结构图](/Users/zhusidao/Documents/wiki/zhusidao.github.io.wiki/images/二级缓存结构图.jpg)
>
> 本质上是装饰器模式的使用，具体的装饰链是：
>
> 以下是具体这些Cache实现类的介绍，他们的组合为Cache赋予了不同的能力。
>
> - `SynchronizedCache`：同步Cache，实现比较简单，直接使用synchronized修饰方法。
> - `LoggingCache`：日志功能，装饰类，用于记录缓存的命中率，如果开启了DEBUG模式，则会输出命中率日志。
> - `SerializedCache`：序列化功能，将值序列化后存到缓存中。该功能用于缓存返回一份实例的Copy，用于保存线程安全。
> - `LruCache`：采用了Lru算法的Cache实现，移除最近最少使用的Key/Value。
> - `PerpetualCache`： 作为为最基础的缓存类，底层实现比较简单，直接使用了HashMap。

CachingExecutor#flushCacheIfRequired

```java
private void flushCacheIfRequired(MappedStatement ms) {
  Cache cache = ms.getCache();
  if (cache != null && ms.isFlushCacheRequired()) {
    // 要求清除二级缓存，insert/update/delte会刷新缓存
    tcm.clear(cache);
  }
}
```

> ```java
> private final TransactionalCacheManager tcm = new TransactionalCacheManager();
> ```

TransactionalCacheManager#clear

```java
public void clear(Cache cache) {
  getTransactionalCache(cache).clear();
}
```

TransactionalCacheManager#getTransactionalCache

```java
private TransactionalCache getTransactionalCache(Cache cache) {
  return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
}
```

> 获取的是一个TransactionalCache对象

TransactionalCache#clear

```java
@Override
public void clear() {
  clearOnCommit = true;
  entriesToAddOnCommit.clear();
}
```



对于二级缓存特点一，下面我们开始源码分析；

CachingExecutor#commit

```java
@Override
public void commit(boolean required) throws SQLException {
  //
  delegate.commit(required);
  tcm.commit();
}
```

BaseExecutor#commit

```java
@Override
public void commit(boolean required) throws SQLException {
  if (closed) {
    throw new ExecutorException("Cannot commit, transaction is already closed");
  }
  // 清除一级缓存
  clearLocalCache();
  flushStatements();
  if (required) {
    transaction.commit();
  }
}
```

CachingExecutor#commit

```java
@Override
public void commit(boolean required) throws SQLException {
  // 清理一级缓存、提交事务
  delegate.commit(required);
  // 提交二级缓存
  tcm.commit();
}
```

TransactionalCacheManager#commit

```java
public void commit() {
  for (TransactionalCache txCache : transactionalCaches.values()) {
    txCache.commit();
  }
}
```

TransactionalCache#commit

```java
public void commit() {
  if (clearOnCommit) {
    delegate.clear();
  }
  flushPendingEntries();
  reset();
}
```

TransactionalCache#flushPendingEntries

```java
private void flushPendingEntries() {
  for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
    // 调用被装饰者的putObject
    delegate.putObject(entry.getKey(), entry.getValue());
  }
  for (Object entry : entriesMissedInCache) {
    if (!entriesToAddOnCommit.containsKey(entry)) {
      delegate.putObject(entry, null);
    }
  }
}
```

TransactionalCache#getObject

```java
@Override
public Object getObject(Object key) {
  // issue #116
  Object object = delegate.getObject(key);
  if (object == null) {
    entriesMissedInCache.add(key);
  }
  // issue #146
  if (clearOnCommit) {
    return null;
  } else {
    return object;
  }
}
```

> 直接调用被装饰者的getObject

TransactionalCache#putObject

```java
@Override
public void putObject(Object key, Object object) {
  entriesToAddOnCommit.put(key, object);
}
```

> putObject只是把值用entriesToAddOnCommit（HashMap结构）存起来，而二级缓存取值却是通过调用被装饰者获取的，二者就是通过sqlSession.commit()进行串联的

**总结**

1. MyBatis的二级缓存相对于一级缓存来说，实现了`SqlSession`之间缓存数据的共享，同时粒度更加的细，能够到`namespace`级别，通过Cache接口实现类不同的组合，对Cache的可控性也更强。
2. MyBatis在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。
3. 在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本，直接使用Redis、Memcached等分布式缓存可能成本更低，安全性也更高。