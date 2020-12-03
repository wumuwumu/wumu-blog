---
title: Mybatis-StatementHandler解析
date: 2020-11-15 18:00:00
---

#### 概述

  在上篇文章中，我们学习了Executor执行器相关的操作，而接下来，我们接着来看Executor的下一步进行操作的对象：StatementHandler。

StatementHandler负责处理Mybatis与JDBC之间Statement的交互，而JDBC中的Statement，我们在学习JDBC的时候就了解过，就是负责与数据库进行交互的对象。这其中会涉及到一些对象，我们用到的时候再学习。首先，我们来看下StatementHandler的体系结构。

##### StatementHandler

StatementHandler接口的实现大致有四个，其中三个实现类都是和JDBC中的Statement响对应的：

> 1. SimpleStatementHandler，这个很简单了，就是对应我们JDBC中常用的Statement接口，用于简单SQL的处理；
> 2. PreparedStatementHandler，这个对应JDBC中的PreparedStatement，预编译SQL的接口；
> 3. CallableStatementHandler，这个对应JDBC中CallableStatement，用于执行存储过程相关的接口；
> 4. RoutingStatementHandler，这个接口是以上三个接口的路由，没有实际操作，只是负责上面三个StatementHandler的创建及调用。

#### 实现

接下来，我们来看下对应的源码实现，我们还是拿查询方法query来学习。

1. 首先，我们从DefaultSqlSession中调用Executor的query方法：

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      // 注意下这里的方法
      MappedStatement ms = configuration.getMappedStatement(statement);
      // 调用Executor的query方法
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
}
```

然后，我们进入BaseExecutor的query方法：

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

这里涉及到了一个BoundSql 对象。BoundSql对象是用于存储sql语句及对应的参数相关的对象。
 然后我们接着看下一步：

```php
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ...
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 从数据库里查询数据
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    ...
}
```

这里涉及到了对缓存的处理，等到学习Mybatis缓存的时候再一并解释，然后接着看queryFromDatabase方法，这里面调用了doQuery方法，我们跳转到SimpleExecutor执行器进行查看对应的doQuery方法：

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      // 这里，从Configuration中获取StatementHandler
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
}
```

从这里我们可以终于看到了StatementHandler的来源了，来自于Configuration对象的方法newStatementHandler，我们查看下该方法：

```cpp
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // RoutingStatementHandler对象出来了
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

这里，我们终于看到了是如何获取StatementHandler的了，通过RoutingStatementHandler的构造方法来进行获取，我们再来看下RoutingStatementHandler的构造方法：

```csharp
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
}
```

到了这一步就很明显了，根据statementType的类型来判断是哪一种StatementHandler的实现，并且RoutingStatementHandler维护了一个delegate对象，通过delegate对象来实现对实际Handler对象的调用。这里涉及到了一个对象MappedStatement。

###### MappedStatement

而所谓的MappedStatement对象就是对mapper.xml中的某个方法`select|update|delete|insert`的封装，如对于下面的`getAll`方法，就对应一个MappedStatement：

```csharp
<select id="getAll" resultType="Student2">
        SELECT * FROM Student
    </select>
```

MappedStatement对象的默认的statementType是PREPARED，所以默认情况下我们所生成的StatementHandler就是PreparedStatementHandler对象。那为什么默认是PREPARED呢，当然，我们也可以从代码中找到原因。
 还记得我们最开始的DefaultSqlSession中的selectList方法中的：

```undefined
MappedStatement ms = configuration.getMappedStatement(statement);
```

这里，我们获取到了MappedStatement，我们来简单看下获取的过程：

```kotlin
public MappedStatement getMappedStatement(String id) {
    return this.getMappedStatement(id, true);
}

public MappedStatement getMappedStatement(String id, boolean validateIncompleteStatements) {
    if (validateIncompleteStatements) {
      buildAllStatements();
    }
    return mappedStatements.get(id);
}
```

这里，方法进入了buildAllStatements方法，我们看到buildAllStatements方法的如下代码：

```css
incompleteStatements.iterator().next().parseStatementNode();
```

同样，我们进入parseStatementNode方法，然后进入：

```css
builderAssistant.addMappedStatement
```

然后进入：

```cpp
MappedStatement.Builder statementBuilder = new MappedStatement.Builder......
```

最终，兜兜转转进入MappedStatement的内部类Builder的构造方法：

```undefined
mappedStatement.statementType = StatementType.PREPARED;
```

最终，我们看到Builder构造方法中设置了默认的statementType类型是PREPARED。当然，如果我们不想使用PREPARED，也可以自己配置，当然配置的方式就是在mapper.xml中对应的某个方法上添加对应属性即可：

```csharp
<select id="getAll" resultType="Student2" statementType="CALLABLE">
    SELECT * FROM Student
</select>
```

------

大致了解了MappedStatement，我们接着上面的SimpleExecutor中的doQuery方法来学习。

1. 获取到StatementHandler之后，首先进入prepareStatement方法，该方法就是为了获取Statement对象，并设置Statement对象中的参数：

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}
```

1. 我们来看下StatementHandler的prepare和parameterize方法，prepare方法负责生成Statement实例对象，而parameterize方法用于处理Statement实例多对应的参数。
    我们大致看下prepare方法，首先进入BaseStatementHandler，查看prepare方法：

```java
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
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

获取实例的方法instantiateStatement，我们可以看下它在PreparedStatementHandler的实现：

```kotlin
protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.prepareStatement(sql);
    }
}
```

这里就不多说了，就是通过Connection的方法来获取Statement实例对象而已。
 而对于parameterize而言，设置参数也很简单的：

```java
public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
}
```

当然，感兴趣的童鞋也可以去看下ParameterHandler的实现DefaultParameterHandler中的实现：setParameters方法。

------

然后，我们接着doQuery方法来看，SimpleExector的doQuery方法会调用StatementHandler的query方法，然后我们来看下PreparedStatementHandler的query实现：

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleResultSets(ps);
}
```

这里又涉及到了另一个对象：ResultSetHandler。这个对象就比较简单了，就是将Statement实例执行之后返回的ResultSet结果集转换成我们需要的List结果集。而这里的PreparedStatement接口的实现则对应于JDBC中PreparedStatement类，这样，最终的调用就到了JDBC这边。





链接：https://www.jianshu.com/p/19686af69b0d