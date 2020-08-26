# mybatis 的缓存机制
mybatis 提供一级缓存和二级缓存，两种缓存的作用机制不一样。

## 一级缓存
mybatis 的一级缓存是默认打开的，查询条件相同的 sql，mybatis 会将查询结果缓存起来，在同一个 SqlSeesion 中，相同的查询会优先命中一级缓存，直接从缓存读取返回，从而提高了性能。

看 BaseExecutor 的 query 方法中的一段代码
``` java
	list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
```

这段代码优先从 localCache 获取，如果缓存已经存在，则直接从缓存中返回，否则查询数据库，然后将查询回来的数据放在 localCache 中。

**缓存的有效期**
BaseExecutor 提供了 clearCache 方法进行清空缓存，该方法会将缓存中的所有内容全部清空，缓存被清空的时机：
- 执行了 update、insert、delete 语句
- statement 配置了flushCache 为 true，每次执行该 statement 都会清空
- mybaitis 全局配置了 LocalCacheScope 为 statement，执行任何 statement 都会清空
- 数据库事务提交
- 数据库事务回滚

**禁用一级缓存**
1. 在指定 statement 上配置 flushCache 为 true
2. 全局配置配置 LocalCacheScope 为 statement

## 二级缓存
上面提到的一级缓存，它的作用域只作用在同一个 sqlSession。如果多个 sqlSession 之间需要共享内存，则需要使用二级缓存，当开启了二级缓存，会使用 CachingExecutor
![Alt text](./1565697138242.png)
![Alt text](./1592481778950.png)

**使用二级缓存**
1. 全局配置配置 cacheEnabled 为 true
2. 在指定的xml 文件中配置 <cache/>


## 一次分表失败的问题排查
## 问题背景
	应用中使用在 repository 层配置了分表拦截器，在service 中希望对所有分表查询同一条sql，但是只有第一次查询正确了应用了分表策略，其他查询并没有进入分表策略，返回的仍然是第一次查询的数据

## 问题分析
	业务反馈没有正确地使用分表策略，业务场景是在一次事务中同时访问多个分表，首要怀疑方向为在一次事务中，分表策略只能被计算一次，导致不能正确分表，因此查看分表拦截器代码

### 分表实现机制
从代码中分析，分表拦截器是通过自定义 mybatis 插件实现的，在 mybatis 插件中，调用用户自定义的分表策略存放在一个 ThreadLocal 中，该 ThreadLocal 存放了一个 HashMap，Map 存放了使用的策略类，策略类内部再封装了一个ThreadLocal 变量来存放分表结果。
关于 ThreadLocal 的原理具体可查看 ThreadLocal 的实现原理。

从分表实现机制来看，不存在分表策略被缓存，导致不能正确分表的情况。

### mybati 缓存
	通过 debug 程序调用，发现分表策略只进入了一次，后从 mybatis 的执行器内部发现命中了 mybatis 的缓存，因此可以确定分表策略不生效是因为开启了 mybatis 缓存。
``` java
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