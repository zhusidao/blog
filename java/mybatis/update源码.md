# mybatis源码分析

本篇介绍原生mybatis源码insert源码分析，首先搭建mybatis环境

mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://10.211.55.10:3306/t_shop" />
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="BookMapper.xml"/>
    </mappers>
</configuration>
```

BookMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="Book">
    <!-- 目的：为dao接口方法提供sql语句配置 -->
    <insert id="insert" >
        insert into book (name,number) values (#{name},#{number})
    </insert>
</mapper>
```

Book

```java
package com.example.demo.entity;

import lombok.Data;

/**
 * 图书实体
 */
@Data
public class Book {

    private long id;// 图书ID

    private String name;// 图书名称

    private int number;// 馆藏数量

}
```

启动类

```java
public class Main {
    public static void main(String[] args) throws IOException {
        // 创建一个book对象
        Book book = new Book();
        book.setId(1006);
        book.setName("Easy Coding");
        book.setNumber(110);
        // 加载配置文件 并构建SqlSessionFactory对象
        String resource = "mybatis-config.xml";
        // 获取mybatis-config.xml配置文件转化为流
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
        // 从SqlSessionFactory对象中获取 SqlSession对象
        SqlSession sqlSession = factory.openSession();
        // 执行操作
        sqlSession.insert("insert", book);
        // 提交事务操作
        sqlSession.commit();
        // 关闭SqlSession
        sqlSession.close();
    }
}
```



#### 下面开始源码分析

**加载文件配置**

Resources#getResourceAsStream

```java
public static InputStream getResourceAsStream(String resource) throws IOException {
  return getResourceAsStream(null, resource);
}
```

Resources#getResourceAsStream

```java
public static InputStream getResourceAsStream(ClassLoader loader, String resource) throws IOException {
  InputStream in = classLoaderWrapper.getResourceAsStream(resource, loader);
  if (in == null) {
    throw new IOException("Could not find resource " + resource);
  }
  return in;
}
```

ClassLoaderWrapper#getResourceAsStream

```java
public InputStream getResourceAsStream(String resource, ClassLoader classLoader) {
  return getResourceAsStream(resource, getClassLoaders(classLoader));
}
```

ClassLoaderWrapper#getResourceAsStream

```java
InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
  for (ClassLoader cl : classLoader) {
    if (null != cl) {

      // try to find the resource as passed
      // 相对路径去获取资源
      InputStream returnValue = cl.getResourceAsStream(resource);

      // now, some class loaders want this leading "/", so we'll add it and try again if we didn't find the resource
      // 加上'/'以classPath为根目录的方式去获取资源
      if (null == returnValue) {
        returnValue = cl.getResourceAsStream("/" + resource);
      }

      if (null != returnValue) {
        return returnValue;
      }
    }
  }
  return null;
}
```

SqlSessionFactoryBuilder#build

```java
public SqlSessionFactory build(InputStream inputStream) {
  return build(inputStream, null, null);
}
```

SqlSessionFactoryBuilder#build

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // xml解析这里就不展示分析了
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    // 构建DefaultSqlSessionFactory对象
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      inputStream.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
```

> 进行xml解析，并转化为SqlSessionFactory对象

DefaultSqlSessionFactory#openSession

```java
@Override
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}
```

DefaultSqlSessionFactory#openSessionFromDataSource

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    // 解析出来的environment配置
    final Environment environment = configuration.getEnvironment();
    // 获取事务管理工厂
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    // 创建事务
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    // 执行器
    final Executor executor = configuration.newExecutor(tx, execType);
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

DefaultSqlSessionFactory#getTransactionFactoryFromEnvironment

```java
private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
  if (environment == null || environment.getTransactionFactory() == null) {
    return new ManagedTransactionFactory();
  }
  // 获取事务管理工厂
  return environment.getTransactionFactory();
}
```

JdbcTransactionFactory#newTransaction

```java
@Override
public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
  return new JdbcTransaction(ds, level, autoCommit);
}
```

Configuration#newExecutor

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    
    executor = new ReuseExecutor(this, transaction);
  } else {
    // 
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    // 缓存执行器
    executor = new CachingExecutor(executor);
  }
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

DefaultSqlSession#insert

```java
@Override
public int insert(String statement, Object parameter) {
  return update(statement, parameter);
}
```

DefaultSqlSession#update

```java
@Override
public int update(String statement, Object parameter) {
  try {
    dirty = true;
    // 获取xml定义的Statement
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.update(ms, wrapCollection(parameter));
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

> `protected final Map<String, MappedStatement> mappedStatements;`
>
> 可见它是一个Map集合，在我们加载xml配置的时候，`mapping.xml`的`namespace`和`id`信息就会存放为`mappedStatements`的`key`，对应的，sql语句就是对应的`value`.

DefaultSqlSession#wrapCollection

```java
private Object wrapCollection(final Object object) {
  return ParamNameResolver.wrapToMapIfCollection(object, null);
}
```

ParamNameResolver#wrapToMapIfCollection

```java
/**
 * 若果参数类型为集合，就用map进行包装
 */
public static Object wrapToMapIfCollection(Object object, String actualParamName) {
  if (object instanceof Collection) {
    ParamMap<Object> map = new ParamMap<>();
    // collection类型
    map.put("collection", object);
    if (object instanceof List) {
      // list类型
      map.put("list", object);
    }
    Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
    return map;
  } else if (object != null && object.getClass().isArray()) {
    ParamMap<Object> map = new ParamMap<>();
    // 数组
    map.put("array", object);
    Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
    return map;
  }
  return object;
}
```

CachingExecutor#update

```java
@Override
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
  flushCacheIfRequired(ms);
  return delegate.update(ms, parameterObject);
}
```

CachingExecutor#flushCacheIfRequired

```java
private void flushCacheIfRequired(MappedStatement ms) {
  // 获取二级缓存
  Cache cache = ms.getCache();
  if (cache != null && ms.isFlushCacheRequired()) {
    tcm.clear(cache);
  }
}
```

BaseExecutor#update

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  // 清除一级缓存
  clearLocalCache();
  return doUpdate(ms, parameter);
}
```

BaseExecutor#clearLocalCache

```java
@Override
public void clearLocalCache() {
  if (!closed) {
    localCache.clear();
    localOutputParameterCache.clear();
  }
}
```

> localCache为一级缓存，这里会被清除
>
> 回顾一下一级缓存失效的场景：
>
> 1. 必须是相同的会话SqlSession
> 2. 必须是同一个mapper 接口中的同一个方法
> 3. 中间没有执行session.clearCache() 方法
> 4. 查询语句中间没有执行insert、update、delete方法



SimpleExecutor#doUpdate真正执行操作的方法

```java
@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
  Statement stmt = null;
  try {
    // 获取配置
    Configuration configuration = ms.getConfiguration();
    // 创建StatementHandler对象，从而创建Statement对象
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    // 预备Statement
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.update(stmt);
  } finally {
    closeStatement(stmt);
  }
}
```

SimpleExecutor#prepareStatement

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  // 获取数据库连接
  Connection connection = getConnection(statementLog);
  // sql预处理
  stmt = handler.prepare(connection, transaction.getTimeout());
  // 设置参数
  handler.parameterize(stmt);
  return stmt;
}
```

> prepareStatement该方法进行了3个操作
>
> 1.获取数据库连接
>
> 2.参数预处理
>
> 3.设置参数

**1.获取数据库连接**

BaseExecutor#getConnection

```java
protected Connection getConnection(Log statementLog) throws SQLException {
  // 打开数据库连接
  Connection connection = transaction.getConnection();
  if (statementLog.isDebugEnabled()) {
    return ConnectionLogger.newInstance(connection, statementLog, queryStack);
  } else {
    return connection;
  }
}
```

JdbcTransaction#getConnection

```java
@Override
public Connection getConnection() throws SQLException {
  if (connection == null) {
    // 打开数据库连接
    openConnection();
  }
  return connection;
}
```

JdbcTransaction#openConnection

```java
protected void openConnection() throws SQLException {
  if (log.isDebugEnabled()) {
    log.debug("Opening JDBC Connection");
  }
  // 获取数据库连接
  // 这里会通过连接池获取数据库连接
  connection = dataSource.getConnection();
  if (level != null) {
    // 隔离级别
    connection.setTransactionIsolation(level.getLevel());
  }
  // 自动提交
  setDesiredAutoCommit(autoCommit);
}
```

**2.参数预处理**

RoutingStatementHandler#prepare

```java
@Override
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
  // deledate为PreparedStatementHandler
  return delegate.prepare(connection, transactionTimeout);
}
```

BaseStatementHandler#prepare

```java
@Override
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
  ErrorContext.instance().sql(boundSql.getSql());
  Statement statement = null;
  try {
    // 打印日志（代理去实现），con.prepareStatement预编译的SQL语句
    statement = instantiateStatement(connection);
    // 设置超时间
    setStatementTimeout(statement, transactionTimeout);
    // fetchSize
    setFetchSize(statement);
    return statement;
  } catch (SQLException e) {
    closeStatement(statement);
    throw e;
  } catch (Exception e) {
    closeStatement(statement);
    throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
  }
}
```

BaseStatementHandler#setStatementTimeout

```java
protected void setStatementTimeout(Statement stmt, Integer transactionTimeout) throws SQLException {
  Integer queryTimeout = null;
  if (mappedStatement.getTimeout() != null) {
    // mapper.xml中取出超时时间
    queryTimeout = mappedStatement.getTimeout();
  } else if (configuration.getDefaultStatementTimeout() != null) {
    // 全局配置取出超时时间
    queryTimeout = configuration.getDefaultStatementTimeout();
  }
  if (queryTimeout != null) {
    // 设置超时时间
    stmt.setQueryTimeout(queryTimeout);
  }
  // 应用事务的超时时间
  StatementUtil.applyTransactionTimeout(stmt, queryTimeout, transactionTimeout);
}
```

StatementUtil#applyTransactionTimeout

```java
/**
 * 设置的超时间如果大于事务的超时时间，就把超时时间设置为事务的超时时间
 */
public static void applyTransactionTimeout(Statement statement, Integer queryTimeout, Integer transactionTimeout) throws SQLException {
  if (transactionTimeout == null) {
    return;
  }
  if (queryTimeout == null || queryTimeout == 0 || transactionTimeout < queryTimeout) {
    statement.setQueryTimeout(transactionTimeout);
  }
}
```

BaseStatementHandler#setFetchSize

```java
protected void setFetchSize(Statement stmt) throws SQLException {
  Integer fetchSize = mappedStatement.getFetchSize();
  if (fetchSize != null) {
    stmt.setFetchSize(fetchSize);
    return;
  }
  Integer defaultFetchSize = configuration.getDefaultFetchSize();
  if (defaultFetchSize != null) {
    stmt.setFetchSize(defaultFetchSize);
  }
}
```

> 网上查了一下资料， 解决大量数据情况下的orm内存溢出，感觉还不如用分页查询来的舒适

**3.设置参数**

RoutingStatementHandler#parameterize

```java
@Override
public void parameterize(Statement statement) throws SQLException {
  delegate.parameterize(statement);
}
```

PreparedStatementHandler#parameterize

```java
@Override
public void parameterize(Statement statement) throws SQLException {
  parameterHandler.setParameters((PreparedStatement) statement);
}
```



**执行sql**

RoutingStatementHandler#update

```java
@Override
public int update(Statement statement) throws SQLException {
  return delegate.update(statement);
}
```

PreparedStatementHandler#update

```java
@Override
public int update(Statement statement) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  // 执行sql
  ps.execute();
  // 获取更新的数
  int rows = ps.getUpdateCount();
  // 获取参数
  Object parameterObject = boundSql.getParameterObject();
  // 获取主键生成器
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  // 执行后置处理方法
  keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
  return rows;
}
```



**关闭Statement**

```java
protected void closeStatement(Statement statement) {
  if (statement != null) {
    try {
      statement.close();
    } catch (SQLException e) {
      // ignore
    }
  }
}
```