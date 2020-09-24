MyBatis缓存

说到MyBatis缓存我们应该知道其分为一级缓存和二级缓存，一级缓存对应sqlsession二级缓存对应namespace，缓存在进行增删改时进行刷新。但是具体对应源代码你知道是如何进行刷新的吗？我们一起来看下源代码。

MyBatis的查询起始于DefaultSqlSession的select，这里我们能看到DefaultSqlSession将查询交给执行器进行查询，这里查询的方法有多个，但是大同小异最终都是交给执行器，这里以select方法为例进行查看。

```java
@Override
public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    executor.query(ms, wrapCollection(parameter), rowBounds, handler);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

这里我们看下query方法，这里我们能看到query方法的实现类只有BaseExecutor和CachingExecutor两个方法，所有的执行器都继承自BaseExecutor，这里只有CachingExecutor进行了重写，这里重写既是对二级缓存的使用。所以开启二级缓存的执行器会通过CachingExecutor进行处理。

![query方法实现](C:\Users\Administrator\Desktop\query方法实现.png)

这里我们看下CachingExecutor类的query方法

```java
//这是CachingExecutor的变量，它是二级缓存的临时存储位置，临时、临时、临时重要的事说三遍
private TransactionalCacheManager tcm = new TransactionalCacheManager();

@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  //这里进行缓存的key计算
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
  
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
  throws SQLException {
  //这里才是二级缓存的真正位置，MappedStatement对象中，这样保证了每次新建执行器，缓存不被删除
  Cache cache = ms.getCache();
  if (cache != null) {
    //设置清空缓存属性,清空缓存
    flushCacheIfRequired(ms);
    //设置useCache属性
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, parameterObject, boundSql);
      //这里将缓存加入到TransactionalCache对象，如果不存在加入，存在直接查找
      @SuppressWarnings("unchecked")
      List<E> list = (List<E>) tcm.getObject(cache, key);
      //这里如果二级缓存查询到数据，直接返回，查询不到会调用BaseExecutor类的查询方法，里面对应         //有一级缓存的调用,所以二级缓存的优先级大于一级缓存的优先级
      if (list == null) {
        list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        //这里如果二级缓存没数据，会将查询结果刷入临时缓存，最终将临时缓存刷入二级缓存
        tcm.putObject(cache, key, list); 
      }
      return list;
    }
  }
  return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

计算缓存的键值上面是CachingExecutor类的方法，下面是BaseExecutor类的方法。CachingExecutor方法初始化时会传入一个执行器，这里最终都会调用BaseExecutor的key生成方法。

```java
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql);
}
  
  
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  CacheKey cacheKey = new CacheKey();
  //MappedStatement的id，这里是namespace，会在其它文章中介绍
  cacheKey.update(ms.getId());
  //偏移量
  cacheKey.update(rowBounds.getOffset());
  //limit分页字段
  cacheKey.update(rowBounds.getLimit());
  //sql
  cacheKey.update(boundSql.getSql());
  //参数
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
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
    cacheKey.update(configuration.getEnvironment().getId());
  }
  return cacheKey;
}

//这里是一个更新方法，实质是一个hashCode计算,每个查询的hashCode都会有，根据查询的所有元素进行计算
public void update(Object object) {
  int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object); 

  count++;
  checksum += baseHashCode;
  baseHashCode *= count;

  hashcode = multiplier * hashcode + baseHashCode;
  //所有参与计算的变量都会存入集合
  updateList.add(object);
}
```

这里是查询临时缓存，如果没有，会将缓存加入进去。

```java
private TransactionalCache getTransactionalCache(Cache cache) {
  TransactionalCache txCache = transactionalCaches.get(cache);
  //加入缓存，这里保证了二级缓存能够对应namespace，没有缓存会将查到的缓存刷进去
  if (txCache == null) {
    txCache = new TransactionalCache(cache);
    transactionalCaches.put(cache, txCache);
  }
  return txCache;
}
```

TransactionalCache这里是获取缓存对象，根据key进行获取。这里既是二级缓存对应的缓存获取。

```java
@Override
public Object getObject(Object key) {
  Object object = delegate.getObject(key);
  if (object == null) {
    entriesMissedInCache.add(key);
  }
  if (clearOnCommit) {
    return null;
  } else {
    return object;
  }
}
```

BaseExecutor这里是一级缓存对应如下。

```java
//一级缓存对应位置
protected PerpetualCache localCache;
//存入查询条件
protected PerpetualCache localOutputParameterCache;
//初始化
protected BaseExecutor(Configuration configuration, Transaction transaction) {
  this.transaction = transaction;
  this.deferredLoads = new ConcurrentLinkedQueue<DeferredLoad>();
  this.localCache = new PerpetualCache("LocalCache");
  this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
  this.closed = false;
  this.configuration = configuration;
  this.wrapper = this;
}

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
    queryStack++;
    //一级缓存获取
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      //这里一级缓存查不到才会查数据库，这里暂不进行介绍
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      clearLocalCache();
    }
  }
  return list;
}
```

这里对应为什么一级缓存insert、update或者不同事务会存在清空。这里能够看到事务的提交和回滚都会进行缓存的清空，update之前也会进行清空。

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  //清空缓存
  clearLocalCache();
  return doUpdate(ms, parameter);
}

@Override
public void commit(boolean required) throws SQLException {
  if (closed) {
    throw new ExecutorException("Cannot commit, transaction is already closed");
  }
  //清空缓存
  clearLocalCache();
  flushStatements();
  if (required) {
    transaction.commit();
  }
}


@Override
public void rollback(boolean required) throws SQLException {
  if (!closed) {
    try {
      //清空缓存
      clearLocalCache();
      flushStatements(true);
    } finally {
      if (required) {
        transaction.rollback();
      }
    }
  }
}
```

这里是二级缓存的缓存处理，会先清空一级缓存，在处理二级缓存

```java
//CachingExecutor事务提交
@Override
public void commit(boolean required) throws SQLException {
  delegate.commit(required);
  //二级缓存的处理
  tcm.commit();
}

//TransactionalCacheManager事务提交
public void commit() {
  for (TransactionalCache txCache : transactionalCaches.values()) {
    txCache.commit();
  }
}
//TransactionalCache事务提交
public void commit() {
  if (clearOnCommit) {
    delegate.clear();
  }
  flushPendingEntries();
  reset();
}

//TransactionalCache缓存处理，这里注意下delegate是Cache缓存不是Executor执行器
private void flushPendingEntries() {
  for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
    //这里的delegate既是我们通过MappedStatement.getCache获取到的缓存，将临时缓存刷入二级缓存
    delegate.putObject(entry.getKey(), entry.getValue());
  }
  for (Object entry : entriesMissedInCache) {
    if (!entriesToAddOnCommit.containsKey(entry)) {
      delegate.putObject(entry, null);
    }
  }
}
```

这里是二级缓存修改时进行进行清空缓存。

```java
@Override
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
  //清空缓存
  flushCacheIfRequired(ms);
  return delegate.update(ms, parameterObject);
}

private void flushCacheIfRequired(MappedStatement ms) {
  //获取缓存
  Cache cache = ms.getCache();
  //缓存存在，且FlushCache设置为是清空缓存
  if (cache != null && ms.isFlushCacheRequired()) {      
    tcm.clear(cache);
  }
}
```
CacheKey重写了equals方法，他的hashcode、updateList及其它属性相等才能相等。

```java
@Override
public boolean equals(Object object) {
  if (this == object) {
    return true;
  }
  if (!(object instanceof CacheKey)) {
    return false;
  }

  final CacheKey cacheKey = (CacheKey) object;

  if (hashcode != cacheKey.hashcode) {
    return false;
  }
  if (checksum != cacheKey.checksum) {
    return false;
  }
  if (count != cacheKey.count) {
    return false;
  }

  for (int i = 0; i < updateList.size(); i++) {
    Object thisObject = updateList.get(i);
    Object thatObject = cacheKey.updateList.get(i);
    if (!ArrayUtil.equals(thisObject, thatObject)) {
      return false;
    }
  }
  return true;
}
```

这里cache有多种，我们可以根据需要进行设置。

 ![cache种类](cache种类.png)