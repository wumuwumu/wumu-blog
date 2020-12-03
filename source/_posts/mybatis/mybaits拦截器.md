---
title: mybatis拦截器
date: 2020-9-19 16:00:00
---

# 基本知识

## 拦截器注解的规则：

具体规则如下：

```css
@Intercepts({
    @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class}),
    @Signature(type = StatementHandler.class, method = "update", args = {Statement.class}),
    @Signature(type = StatementHandler.class, method = "batch", args = {Statement.class})
})
```

1. @Intercepts：标识该类是一个拦截器；
2. @Signature：指明自定义拦截器需要拦截哪一个类型，哪一个方法；
    2.1 type：对应四种类型中的一种；
    2.2 method：对应接口中的哪类方法（因为可能存在重载方法）；
    2.3 args：对应哪一个方法；

> **5. 拦截器可拦截的方法：**

| 拦截的类         | 拦截的方法                                                   |
| ---------------- | ------------------------------------------------------------ |
| Executor         | update, query, flushStatements, commit, rollback,getTransaction, close, isClosed |
| ParameterHandler | getParameterObject, setParameters                            |
| StatementHandler | prepare, parameterize, batch, update, query                  |
| ResultSetHandler | handleResultSets, handleOutputParameters                     |

```java
@Intercepts({@Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
public class TestInterceptor implements Interceptor {
   public Object intercept(Invocation invocation) throws Throwable {
     Object target = invocation.getTarget(); //被代理对象
     Method method = invocation.getMethod(); //代理方法
     Object[] args = invocation.getArgs(); //方法参数
     // do something ...... 方法拦截前执行代码块
     Object result = invocation.proceed();
     // do something .......方法拦截后执行代码块
     return result;
   }
   public Object plugin(Object target) {
     return Plugin.wrap(target, this);
   }
}
```

## setProperties方法

如果我们的拦截器需要一些变量对象，而且这个对象是支持可配置的。
 类似于Spring中的@Value("${}")从[application.properties](https://links.jianshu.com/go?to=http%3A%2F%2Fapplication.properties)文件中获取。
 使用方法：

```xml
<plugin interceptor="com.plugin.mybatis.MyInterceptor">
     <property name="username" value="xxx"/>
     <property name="password" value="xxx"/>
</plugin>
```

## plugin方法

这个方法的作用是就是让mybatis判断，是否要进行拦截，然后做出决定是否生成一个代理。

```kotlin
    @Override
    public Object plugin(Object target) {
        if (target instanceof StatementHandler) {
            return Plugin.wrap(target, this);
        }
        return target;
    }
```

**需要注意的是：每经过一个拦截器对象都会调用插件的plugin方法，也就是说，该方法会调用4次。根据@Intercepts注解来决定是否进行拦截处理。**

> 问题1：**Plugin.wrap(target, this)**方法的作用？

解答：判断是否拦截这个类型对象（根据@Intercepts注解决定），然后决定是返回一个代理对象还是返回原对象。

故我们在实现plugin方法时，要判断一下目标类型，是本插件要拦截的对象时才执行Plugin.wrap方法，否则的话，直接返回目标本身。

> 问题2：拦截器代理对象可能经过多层代理，如何获取到真实的拦截器对象？

```dart
    /**
     * <p>
     * 获得真正的处理对象,可能多层代理.
     * </p>
     */
    @SuppressWarnings("unchecked")
    public static <T> T realTarget(Object target) {
        if (Proxy.isProxyClass(target.getClass())) {
            MetaObject metaObject = SystemMetaObject.forObject(target);
            return realTarget(metaObject.getValue("h.target"));
        }
        return (T) target;
```

# 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <properties resource="top/sciento/wumu/jdbc/mybatis/db.properties"></properties>
    <plugins>
        <plugin interceptor="top.sciento.wumu.jdbc.mybatis.plugin.ExamplePlugin">

        </plugin>
        <plugin interceptor="top.sciento.wumu.jdbc.mybatis.plugin.PagePlugin"/>
    </plugins>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${driver}"/>
                <property name="url" value="${url}"/>
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <package name="top.sciento.wumu.jdbc.mybatis.mapper"/>
    </mappers>
</configuration>
```

# 实战

```java

@Intercepts({@Signature(
        type= Executor.class,
        method = "update",
        args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("被拦截方法执行之前，做的辅助服务······");
        Object[] args = invocation.getArgs();
        Method method = invocation.getMethod();
        Object target  = invocation.getTarget();
        MappedStatement mappedStatement = (MappedStatement) args[0];

        Object proceed = invocation.proceed();
        System.out.println("被拦截方法执行之后，做的辅助服务······");
        return proceed;
    }
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
    public void setProperties(Properties properties) {
    }
}

```

```java
@Slf4j
@Intercepts(
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
)
public class PagePlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement mappedStatement = (MappedStatement) args[0];

        //获取参数
        Object param = invocation.getArgs()[1];
        BoundSql boundSql = mappedStatement.getBoundSql(param);
        Object parameterObject = boundSql.getParameterObject();

        /**
         * 判断是否是继承PageVo来判断是否需要进行分页
         */
        if (parameterObject instanceof Page) {
            //强转 为了拿到分页数据
            Page pagevo = (Page) param;
            String sql = boundSql.getSql();


            //获取相关配置
            Configuration config = mappedStatement.getConfiguration();
            Connection connection = config.getEnvironment().getDataSource().getConnection();

            //拼接查询当前条件的sql的总条数
            String countSql = "select count(*) from (" + sql + ") a";
            PreparedStatement preparedStatement = connection.prepareStatement(countSql);
            BoundSql countBoundSql = new BoundSql(config, countSql, boundSql.getParameterMappings(), boundSql.getParameterObject());
            ParameterHandler parameterHandler = new DefaultParameterHandler(mappedStatement, parameterObject, countBoundSql);
            parameterHandler.setParameters(preparedStatement);
            //执行获得总条数
            ResultSet rs = preparedStatement.executeQuery();
            int count = 0;
            if (rs.next()) {
                count = rs.getInt(1);
            }


            //拼接分页sql
            String pageSql = sql + " limit " + pagevo.getOffset() + " , " + pagevo.getSize();
            //重新执行新的sql
            doNewSql(invocation, pageSql);

            Object result = invocation.proceed();
            connection.close();
            // 这是使用了两种不同的方式返回最终的结果
            pagevo.setList((List)result);
            pagevo.setTotal(count);
            //处理新的结构
            PageResult<?> pageResult = new PageResult<List>((List) result,pagevo.getPage(), pagevo.getSize(), count );
            return new ArrayList<PageResult>(){{add(pageResult);}} ;
        }
        return invocation.proceed();
    }

    private void doNewSql(Invocation invocation, String sql){
        final Object[] args = invocation.getArgs();
        MappedStatement statement = (MappedStatement) args[0];
        Object parameterObject = args[1];
        BoundSql boundSql = statement.getBoundSql(parameterObject);
        MappedStatement newStatement = newMappedStatement(statement, new BoundSqlSqlSource(boundSql));
        MetaObject msObject = MetaObject.forObject(newStatement, new DefaultObjectFactory(), new DefaultObjectWrapperFactory(), new DefaultReflectorFactory());
        msObject.setValue("sqlSource.boundSql.sql", sql);
        args[0] = newStatement;
    }

    private MappedStatement newMappedStatement(MappedStatement ms, SqlSource newSqlSource) {
        MappedStatement.Builder builder =
                new MappedStatement.Builder(ms.getConfiguration(), ms.getId(), newSqlSource, ms.getSqlCommandType());
        builder.resource(ms.getResource());
        builder.fetchSize(ms.getFetchSize());
        builder.statementType(ms.getStatementType());
        builder.keyGenerator(ms.getKeyGenerator());
        if (ms.getKeyProperties() != null && ms.getKeyProperties().length != 0) {
            StringBuilder keyProperties = new StringBuilder();
            for (String keyProperty : ms.getKeyProperties()) {
                keyProperties.append(keyProperty).append(",");
            }
            keyProperties.delete(keyProperties.length() - 1, keyProperties.length());
            builder.keyProperty(keyProperties.toString());
        }
        builder.timeout(ms.getTimeout());
        builder.parameterMap(ms.getParameterMap());
        builder.resultMaps(ms.getResultMaps());
        builder.resultSetType(ms.getResultSetType());
        builder.cache(ms.getCache());
        builder.flushCacheRequired(ms.isFlushCacheRequired());
        builder.useCache(ms.isUseCache());

        return builder.build();
    }
    /**
     * 新的SqlSource需要实现
     */
    class BoundSqlSqlSource implements SqlSource {
        private BoundSql boundSql;

        public BoundSqlSqlSource(BoundSql boundSql) {
            this.boundSql = boundSql;
        }

        @Override
        public BoundSql getBoundSql(Object parameterObject) {
            return boundSql;
        }
    }
}
```

# 参考

https://www.jianshu.com/p/0a72bb1f6a21