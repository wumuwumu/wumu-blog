---
title: mybatis-sessions
date: 2019-04-10 10:25:51
tags:
- mybatis
- java
---



# SqlSessionFactory

`sqlSessionFactory`是工厂类的接口，默认实现是`DefaultSqlSessionFactory`，通过`sqlSessionFactoryBuilder`创建，我们不具体讨论配置文件的具体解析，主要分析mybatis的运行流程。

`SqlSessionFactory`主要是用来创建`SqlSession`，`SqlSession`是线程不安全的，因此每次操作都要重新创建。

```
// 通过数据源创建SqlSession，是我们比较常用的一种方式
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //通过事务工厂来产生一个事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //生成一个执行器(事务包含在执行器里)
      final Executor executor = configuration.newExecutor(tx, execType);
      //然后产生一个DefaultSqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      //如果打开事务出错，则关闭它
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      //最后清空错误上下文
      ErrorContext.instance().reset();
    }
  }
SqlSession
```

`SqlSession`有两方式调用方法，第一种方式是通过命名空间调用，第二种方式是`JavaBean`调用，也就是通过我们常用的Mapper接口进行调用。现在`Myabtis3`我们基本使用第二种方式。

通过Mapper接口进行调用，核心是 获取Mapper接口，并通过动态代理，进行方法拦截。

`SqlSession`通过`getMapper`获取相应的Mapper接口。`SqlSession`的的数据库操作是调用Executor的相关方法。

在`getMapper`调用的时候，有几个核心的类

1. `MapperProxyFactory`:用于创建`MapperProxyd`的工厂方法
2. `MapperProxy`:动态代理的`InvocationHandler`的实现，实际中就是执行sql语句
3. `MapperRegistry`
4. `MapperMethood`:调用`SqlSession`的方法