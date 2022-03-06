---
title: Mybatis源码解析-BoundSql
abbrlink: 4620db30
date: 2020-11-15 18:00:00
---

### 前提

> 1. 针对mybatis的配置文件的节点解析，比如`where`/`if`/`trim`的节点解析可见文章[Spring mybatis源码篇章-NodeHandler实现类具体解析保存Dynamic sql节点信息](http://www.cnblogs.com/question-sky/p/6642263.html)
> 2. 针对mybatis配置文件的解析帮助类SqlSource[一般为DynamicSqlSource]的使用可见文章[Spring mybatis源码篇章-XMLLanguageDriver解析sql包装为SqlSource](http://www.cnblogs.com/question-sky/p/6629177.html)
> 3. 对BoundSql对象的调用获取可见文章[Mybatis源码分析-BaseExecutor](http://www.cnblogs.com/question-sky/p/7353418.html)

本文将在上述的知识前提下展开对Sql语句的解析

### BoundSql的引用

主要是通过`MappedStatement#getBoundSql()`方法调用获取的。我们可以简单跟踪下其中的源码，如下

```java
  public BoundSql getBoundSql(Object parameterObject) {
    // 通过SqlSource获取BoundSql对象
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 校验当前的sql语句有无绑定parameterMapping属性
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings == null || parameterMappings.isEmpty()) {
      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
      String rmId = pm.getResultMapId();
      if (rmId != null) {
        ResultMap rm = configuration.getResultMap(rmId);
        if (rm != null) {
          hasNestedResultMaps |= rm.hasNestedResultMaps();
        }
      }
    }

    return boundSql;
  }
```

#### RawSqlSource-常用的mybatis解析sql帮助类

我们观察下其getBoundSql()方法，源码如下

```java
  public BoundSql getBoundSql(Object parameterObject) {
    //此处的sqlSource为RawSqlSource的内部属性
    return sqlSource.getBoundSql(parameterObject);
  }
```

我们看下sqlSource是如何生成的，由此观察其构造函数

```java
public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
   // 通过SqlSourceBuilder来创建sqlSource
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<String, Object>());
  }
```

> `#{}`的使用这里稍微提下，一般的写法都为`{name,jdbcType=String，mode=out,javaType=java.lang.String...}`，其中jdbcType也可以不指定，系统会自动识别。上述的代码其实主要就是针对`#{}`字符内容的处理
>
> **注意：${}这样的字符是通过DynamicSqlSource来完成解析的，具体的解析读者可自行分析**

我们可以继续看下SqlSourceBuilder类是如何解析获取sql语句的

#### SqlSourceBuilder#parse()

直接上源码

```java
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    // 对#{}这样的字符串内容的解析处理类
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    // 获取真实的可执行性的sql语句
    String sql = parser.parse(originalSql);
    // 包装成StaticSqlSource返回
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
```

简单的看下ParameterMappingTokenHandler是如何解析的，其是`TokenHandler`接口的实现类，我们就关注实现方法handleToken

```java
@Override
    public String handleToken(String content) {
      // 此处的作用就是对`#{}`节点中的key值保存映射，比如javaType/jdbcType/mode等信息，限于篇幅过长，读者可自行分析          
      parameterMappings.add(buildParameterMapping(content));
      // 将`#{}`替换为?，即一般包装成`select * form test where name=? and age=?`预表达式语句
      return "?";
    }
```

> 上述主要通过`ParameterMappingTokenHandler`类来完成对`#{}`字符串的解析，其中的映射信息则保存至BoundSql的parameterMappings属性中

### 总结

> 1. **BoundSql语句的解析主要是通过对#{}字符的解析，将其替换成?。最后均包装成预表达式供PrepareStatement调用执行**
> 2. **#{}中的key属性以及相应的参数映射，比如javaType、jdbcType等信息均保存至BoundSql的parameterMappings属性中供最后的预表达式对象PrepareStatement赋值使用**