---
title: SpringCloudGateway基本操作-动态路由
date: 2021-1-25 16:00:00
---

gateway配置路由主要有两种方式，一种是用yml配置文件，一种是写代码里，这两种方式都是不支持动态配置的。如：

![img](https://img-blog.csdnimg.cn/20181026112026523.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)![img](https://img-blog.csdnimg.cn/20181026112045783.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

下面就来看看gateway是如何加载这些配置信息的。

### 1 路由初始化

无论是yml还是代码，这些配置最终都是被封装到RouteDefinition对象中。

![img](https://img-blog.csdnimg.cn/20181026112443317.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

一个RouteDefinition有个唯一的ID，如果不指定，就默认是UUID，多个RouteDefinition组成了gateway的路由系统。

所有路由信息在系统启动时就被加载装配好了，并存到了内存里。我们从源码来看看。![img](https://img-blog.csdnimg.cn/20181026113033137.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

圆圈里就是装配yml文件的，它返回的是PropertiesRouteDefinitionLocator，该类继承了RouteDefinitionLocator，RouteDefinitionLocator就是路由的装载器，里面只有一个方法，就是获取路由信息的。该接口有多个实现类，分别对应不同方式配置的路由方式。

![img](https://img-blog.csdnimg.cn/20181026113950773.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20181026113753850.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/2018102612081129.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

通过这几个实现类，再结合上面的AutoConfiguration里面的Primary信息，就知道加载配置信息的顺序。

PropertiesRouteDefinitionLocator-->|配置文件加载初始化| CompositeRouteDefinitionLocator
RouteDefinitionRepository-->|存储器中加载初始化| CompositeRouteDefinitionLocator
DiscoveryClientRouteDefinitionLocator-->|注册中心加载初始化| CompositeRouteDefinitionLocator

参考：https://www.jianshu.com/p/b02c7495eb5e

https://blog.csdn.net/X5fnncxzq4/article/details/80221488

![img](https://img-blog.csdnimg.cn/20181026114644355.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

这是第一顺序，就是从CachingRouteLocator中获取路由信息，我们可以打开该类进行验证。![img](https://img-blog.csdnimg.cn/20181026114836900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

不管发起什么请求，必然会走上面的断点处。请求一次，走一次。这是将路由信息缓存到了Map中。配置信息一旦请求过一次，就会被缓存到上图的CachingRouteLocator类中，再次发起请求后，会直接从map中读取。

如果想动态刷新配置信息，就需要发起一个RefreshRoutesEvent的事件，上图的cache会监听该事件，并重新拉取路由配置信息。

通过下图，可以看到如果没有RouteDefinitionRepository的实例，则默认用InMemoryRouteDefinitionRepository。而做动态路由的关键就在这里。即通过自定义的RouteDefinitionRepository类，来提供路由配置信息。

![img](https://img-blog.csdnimg.cn/20181026120724858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

例如：

![img](https://img-blog.csdnimg.cn/2018102612162119.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

在getRouteDefinitions方法返回你自定义的路由配置信息即可。这里可以用数据库、nosql等等任意你喜欢的方式来提供。而且配置信息修改后，发起一次RefreshRoutesEvent事件即可让配置生效。这就是动态配置路由的核心所在，下面来看具体代码实现。

### 2 基于数据库、缓存的动态路由

pom.xml如下

```xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.maimeng</groupId>
    <artifactId>apigateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
 
    <name>apigateway</name>
    <description>Demo project for Spring Boot</description>
 
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
 
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.SR1</spring-cloud.version>
    </properties>
 
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.51</version>
        </dependency>
        <!--<dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
 
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
 
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
 
 
</project>
```

![img](https://img-blog.csdnimg.cn/20181026160616350.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

注意这里是SR1，经测试SR2有bug，会出问题。

```java

@Configuration
public class RedisConfig {
 
    @Bean(name = {"redisTemplate", "stringRedisTemplate"})
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory factory) {
        StringRedisTemplate redisTemplate = new StringRedisTemplate();
        redisTemplate.setConnectionFactory(factory);
        return redisTemplate;
    }
 
}
```

核心类：

```java

@Component
public class RedisRouteDefinitionRepository implements RouteDefinitionRepository {
 
    public static final String GATEWAY_ROUTES = "geteway_routes";
 
    @Resource
    private StringRedisTemplate redisTemplate;
 
    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        List<RouteDefinition> routeDefinitions = new ArrayList<>();
        redisTemplate.opsForHash().values(GATEWAY_ROUTES).stream()
                .forEach(routeDefinition -> routeDefinitions.add(JSON.parseObject(routeDefinition.toString(), RouteDefinition.class)));
        return Flux.fromIterable(routeDefinitions);
    }
 
    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return null;
    }
 
    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return null;
    }
 
}
```

主要是在get方法里，此处从redis里获取配置好的Definition。

然后我们的工作就是将配置信息，放到redis里即可。

下面就是我模拟的一个配置，等同于在yml里

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header
        uri: http://localhost:8888/header
        filters:
        - AddRequestHeader=header, addHeader
        - AddRequestParameter=param, addParam
        predicates:
        - Path=/jd
```

定义好后，将其放到redis里，之后启动项目访问/jd，再启动后台的localhost:8888项目。即可进行验证。

之后如果要动态修改配置，就可以通过类似于上面的方式，来获取json字符串，然后将字符串放到redis里进行替换。替换后，需要通知gateway主动刷新一下。

![img](https://img-blog.csdnimg.cn/20181026161549838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20181026161611891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RpYW55YWxlaXhpYW93dQ==,size_27,color_FFFFFF,t_70)

刷新时，可以定义一个controller，然后调用一下notifyChanged()方法，就能完成新配置的替换了。

# 参考

> https://blog.csdn.net/tianyaleixiaowu/article/details/83412301
>
> https://www.haoyizebo.com/posts/1962f450/