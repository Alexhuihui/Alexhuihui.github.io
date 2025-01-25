---
title: 【Mybatis】Mybatis运行流程
date: 2023-06-07 19:54:06
categories: Mybatis
tags: Mybatis
---

今天让我们来探寻一下`Mybatis`的运行流程，我们将它的运行流程分为2个阶段。

- 第一阶段：`MyBatis`初始化阶段。该阶段用来完成 `MyBatis`运行环境的准备工作，只在 `MyBatis`启动时运行一次。
- 第二阶段：数据读写阶段。该阶段由数据读写操作触发，将根据要求完成具体的增、删、改、查等数据库操作。

<!-- more --> 

### 初始化阶段

- 根据配置文件的位置，获取它的输入流 `InputStream`

```java
InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
    for (ClassLoader cl : classLoader) {
      if (null != cl) {

        // try to find the resource as passed
        InputStream returnValue = cl.getResourceAsStream(resource);

        // now, some class loaders want this leading "/", so we'll add it and try again if we didn't find the resource
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

- 从配置文件的根节点开始，逐层解析配置文件，也包括相关的映射文件。解析过程中不断将解析结果放入 `Configuration`对象。
- 以配置好的 `Configuration`对象为参数，获取一个` SqlSessionFactory`对象。

```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        if (reader != null) {
          reader.close();
        }
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
```

 重要代码注释:

 > 1. 生成了一个 `XMLConfigBuilder `对象，并调用了其 `parse `方法，得到一个`Configuration`对象（因为 parse方法的输出结果为` Configuration`对象）。

 > 2. 调用了 `SqlSessionFactoryBuilder` 自身的 `build`方法， 传入参数为上一步得到的`Configuration`对象。

### 数据读写阶段追踪

进行一次数据库的读或写操作时，`MyBatis`内部都要经过哪些步骤

#### 获得`SqlSession`

通过初始化阶段获取的`SqlSessionFactory`生成数据库操作中所需要的`SqlSession`

```java
SqlSession sqlSession = sqlSessionFactory.openSession();
```

#### 映射接口文件与映射文件的绑定

映射接口文件是指存有Java接口的文件，而映射文件是指存有`sql`操作的`xml`文件

在进行数据查询之前，先通过`SqlSession`的`getMapper()`获取映射接口的实现, 该操作通过 Configuration类的 `getMapper`方法转接，最终进入` MapperRegistry`类中的`getMapper`方法。`MapperRegistry`类中的 `getMapper()`

```java
// DefaultSqlSession.class
@Override
public <T> T getMapper(Class<T> type) {
   return configuration.getMapper(type, this);
}

// Configuration.class
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}

// MapperRegistry.class
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

#### 映射接口的代理

`session.getMapper()` 得到的是一个`mapperProxyFactory.newInstance(sqlSession)`返回的对象，这个对象是一个基于反射的动态代理对象

```java
@SuppressWarnings("unchecked")
protected T newInstance(MapperProxy<T> mapperProxy) {
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

最终数据查询会进入`MapperProxy`的`invoke()`，这是因为被代理对象的方法会被代理对象的`invoke()`拦截

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    }
    return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```

然后会触发`MapperMethod`对象的`execute()`

```java
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
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() && (result == null || !method.getReturnType().equals(result.getClass()))) {
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
          + "' attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

`MyBatis`根据不同数据库操作类型调用了不同的处理方法

`executeForMany` 方法中，`MyBatis `开始通过 `SqlSession` 对象的`selectList`方法开展后续的查询工作。 追踪到这里，`MyBatis `已经完成了为映射接口注入实现的过程。 于是，对映射接口中抽象方法的调用转变为了数据查询操作。

#### `SQL`语句的查找

每个` MappedStatement` 对象对应了我们设置的一个数据库操作节 点，它主要定义了数据库操作语句、输入/输出参数等信息。configuration.getMappedStatement（statement）`语句将要执行的`MappedStatement`对象从 `Configuration`对象存储的映射文件信息中找了出来。

```java
private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      dirty |= ms.isDirtySelect();
      return executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

#### 查询结果缓存

query方法是一个 Executor接口中的抽象方法，实际执行的是` CachingExecutor`类中的方法

```java
// CachingExecutor.class
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler)
      throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

`BoundSql`是经过层层转化后去除掉 if、where等标签的 `SQL`语 句，而` CacheKey`是为该次查询操作计算出来的缓存键

#### 数据库查询

```java
// CachingExecutor.class
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler,
      CacheKey key, BoundSql boundSql) throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
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

`MyBatis`查看当前的查询操作是否命中缓存。如果 是，则从缓存中获取数据结果；否则，便通过 delegate调用 query方 法。

delegate调用的 query方法实际上是`BaseExecutor`类中的 query方法

```java
// BaseExecutor.class
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler,
      CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
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
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```

`MyBatis`开始调用数据库展开查询操 作

```java
// BaseExecutor.class
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds,
      ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

`MyBatis`先在缓存中放置一个占位符，然 后调用 `doQuery`方法实际执行查询操作。最后，又把缓存中的占位符 替换成真正的查询结果

```java
// SimpleExecutor.class
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler,
      BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler,
          boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

上述方法生成了Statement对象stmt。Statement类并不是MyBatis 中的类，而是java.sql包中的类。Statement类能够执行静态 SQL语句 并返回结果。

程序还通过 Configuration的 newStatementHandler方法获得了 一个 StatementHandler对象 handler，然后将查询操作交给 StatementHandler对象进行。StatementHandler是一个语句处理器类，其中封 装了很多语句操作方法

```java
// PreparedStatementHandler.class
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.handleResultSets(ps);
  }
```

这里 ps.execute（）真正执行了 SQL 语句，然后把执行结果交 给 ResultHandler 对象处理。而PreparedStatement类并不是MyBatis 中的类，因而ps.execute（）的执行不再由MyBatis负责，而是由 com.mysql.cj.jdbc包中的类负责，这里不再继续追踪。

总结：

1. 在进行数据库查询前，先查询缓存；如果确实需要查询数据 库，则数据库查询之后的结果也放入缓存中。 ·
2. SQL 语句的执行经过了层层转化，依次经过了 MappedStatement 对象、Statement对象和 PreparedStatement对象，最后才得以执 行。 ·
3. 最终数据库查询得到的结果交给 ResultHandler对象处理。

#### 处理结果集

```java
// DefaultResultSetHandler.class
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
```

