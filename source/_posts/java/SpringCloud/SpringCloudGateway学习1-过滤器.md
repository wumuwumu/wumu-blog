---
title: SpringCloudGateway基本操作-过滤器
date: 2021-01-25 15:00:00
---

Spring Cloud Gateway 已经内置了很多实用的过滤器，但并不能完全满足我们的需求。本文我们就来实现自定义过滤器。虽然现在 Spring Cloud Gateway 的文档还不完善，但是我们依旧可以照猫画虎来定制自己的过滤器。

[![img](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tNc79ly1fr4wcu9jgkj30nk0bqabh.jpg)](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tNc79ly1fr4wcu9jgkj30nk0bqabh.jpg)



## Filter 的作用

其实前边在介绍 Zuul 的的时候已经介绍过 Zuul 的 Filter 的作用了，同作为网关服务，Spring Cloud Gateway 的 Filter 作用也类似。

这里就简单用两张图来解释一下吧。

[![img](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr43eek154j316c0g4wgz.jpg)](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr43eek154j316c0g4wgz.jpg)

当使用微服务构建整个 API 服务时，一般有许多不同的应用在运行，如上图所示的`mst-user-service`、`mst-good-service`和`mst-order-service`，这些服务都需要对客户端的请求的进行 Authentication。最简单粗暴的方法就是像上图一样，为每个微服务应用都实现一套用于校验的过滤器或拦截器。

对于这样的问题，更好的做法是通过前置的网关服务来完成这些非业务性质的校验，就像下图

[![img](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr43dop520j31j60ni413.jpg)](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr43dop520j31j60ni413.jpg)

## Filter 的生命周期

Spring Cloud Gateway 的 Filter 的生命周期不像 Zuul 的那么丰富，它只有两个：“pre”和“post”。

[![image-20180508184542206](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr48yqx3ouj31kw17pn81.jpg)](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr48yqx3ouj31kw17pn81.jpg)

[image-20180508184542206](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tKfTcly1fr48yqx3ouj31kw17pn81.jpg)



“pre”和“post”分别会在请求被执行前调用和被执行后调用，和 Zuul Filter 或 Spring Interceptor 中相关生命周期类似，但在形式上有些不一样。

Zuul 的 Filter 是通过`filterType()`方法来指定，一个 Filter 只能对应一种类型，要么是“pre”要么是“post”。Spring Interceptor 是通过重写`HandlerInterceptor`中的三个方法来实现的。而 Spring Cloud Gateway 基于 Project Reactor 和 WebFlux，采用响应式编程风格，打开它的 Filter 的接口`GatewayFilter`你会发现它只有一个方法`filter`。

仅通过这一个方法，怎么来区分是“pre”还是“post”呢？我们下边就通过自定义过滤器来看看。

## 自定义过滤器

现在假设我们要统计某个服务的响应时间，我们可以在代码中

```java
long beginTime = System.currentTimeMillis();
// do something...
long elapsed = System.currentTimeMillis() - beginTime;
log.info("elapsed: {}ms", elapsed);
```

每次都要这么写是不是很烦？Spring 告诉我们有个东西叫 AOP。但是我们是微服务啊，在每个服务里都写也很烦。这时候就该网关的过滤器登台表演了。

自定义过滤器需要实现`GatewayFilter`和`Ordered`。其中`GatewayFilter`中的这个方法就是用来实现你的自定义的逻辑的

```java
Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
Copy
```

而`Ordered`中的`int getOrder()`方法是来给过滤器设定优先级别的，值越大则优先级越低。

好了，让我们来撸代码吧

```java
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.core.Ordered;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

public class ElapsedFilter implements GatewayFilter, Ordered {

    private static final Log log = LogFactory.getLog(GatewayFilter.class);
    private static final String ELAPSED_TIME_BEGIN = "elapsedTimeBegin";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        exchange.getAttributes().put(ELAPSED_TIME_BEGIN, System.currentTimeMillis());
        return chain.filter(exchange).then(
                Mono.fromRunnable(() -> {
                    Long startTime = exchange.getAttribute(ELAPSED_TIME_BEGIN);
                    if (startTime != null) {
                        log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - startTime) + "ms");
                    }
                })
        );
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

我们在请求刚刚到达时，往`ServerWebExchange`中放入了一个属性`elapsedTimeBegin`，属性值为当时的毫秒级时间戳。然后在请求执行结束后，又从中取出我们之前放进去的那个时间戳，与当前时间的差值即为该请求的耗时。因为这是与业务无关的日志所以将`Ordered`设为`Integer.MAX_VALUE`以降低优先级。

现在再来看我们之前的问题：怎么来区分是“pre”还是“post”呢？其实就是`chain.filter(exchange)`之前的就是“pre”部分，之后的也就是`then`里边的是“post”部分。

创建好 Filter 之后我们将它添加到我们的 Filter Chain 里边

```java
@Bean
public RouteLocator customerRouteLocator(RouteLocatorBuilder builder) {
    // @formatter:off
    return builder.routes()
            .route(r -> r.path("/fluent/customer/**")
                         .filters(f -> f.stripPrefix(2)
                                        .filter(new ElapsedFilter())
                                        .addResponseHeader("X-Response-Default-Foo", "Default-Bar"))
                         .uri("lb://CONSUMER")
                         .order(0)
                         .id("fluent_customer_service")
            )
            .build();
    // @formatter:on
}
```

现在再尝试访问 http://localhost:10000/customer/hello/yibo 即可在控制台里看到请求路径与对应的耗时

```java
2018-05-08 16:07:04.197  INFO 83726 --- [ctor-http-nio-4] o.s.cloud.gateway.filter.GatewayFilter   : /hello/yibo: 40ms
```

> 实际在使用 Spring Cloud 的过程中，我们会[使用 Sleuth+Zipkin 来进行耗时分析](https://www.haoyizebo.com/posts/6d06094e/)。

## 自定义全局过滤器

前边讲了自定义的过滤器，那个过滤器只是局部的，如果我们有多个路由就需要一个一个来配置，**并不能**通过像下面这样来实现全局有效（也未在 Fluent Java API 中找到能设置 defaultFilters 的方法）

```java
@Bean
public ElapsedFilter elapsedFilter(){
    return new ElapsedFilter();
}
```

这在我们要全局统一处理某些业务的时候就显得比较麻烦，比如像最开始我们说的要做身份校验，有没有简单的方法呢？这时候就该全局过滤器出场了。

有了前边的基础，我们创建全局过滤器就简单多了。只需要把实现的接口`GatewayFilter`换成`GlobalFilter`，就完事大吉了。比如下面的 Demo 就是从请求参数中获取`token`字段，如果能获取到就 pass，获取不到就直接返回`401`错误，虽然简单，但足以说明问题了。

```
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

public class TokenFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        if (token == null || token.isEmpty()) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

然后在 Spring Config 中配置这个 Bean

```java
@Bean
public TokenFilter tokenFilter(){
    return new TokenFilter();
}
```

重启应用就能看到效果了

```java
2018-05-08 20:41:06.528 DEBUG 87751 --- [ctor-http-nio-2] o.s.c.g.h.RoutePredicateHandlerMapping   : Mapping [Exchange: GET http://localhost:10000/customer/hello/yibo?token=1000] to Route{id='service_customer', uri=lb://CONSUMER, order=0, predicate=org.springframework.cloud.gateway.handler.predicate.PathRoutePredicateFactory$$Lambda$334/1871259950@2aa090be, gatewayFilters=[OrderedGatewayFilter{delegate=org.springframework.cloud.gateway.filter.factory.StripPrefixGatewayFilterFactory$$Lambda$337/577037372@22e84be7, order=1}, OrderedGatewayFilter{delegate=org.springframework.cloud.gateway.filter.factory.AddResponseHeaderGatewayFilterFactory$$Lambda$339/1061806694@1715f608, order=2}]}
2018-05-08 20:41:06.530 DEBUG 87751 --- [ctor-http-nio-2] o.s.c.g.handler.FilteringWebHandler      : Sorted gatewayFilterFactories: [OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=com.yibo.filter.TokenFilter@309028af}, order=-100}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.NettyWriteResponseFilter@70e889e9}, order=-1}, OrderedGatewayFilter{delegate=org.springframework.cloud.gateway.filter.factory.StripPrefixGatewayFilterFactory$$Lambda$337/577037372@22e84be7, order=1}, OrderedGatewayFilter{delegate=org.springframework.cloud.gateway.filter.factory.AddResponseHeaderGatewayFilterFactory$$Lambda$339/1061806694@1715f608, order=2}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter@51351f28}, order=10000}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.LoadBalancerClientFilter@724c5cbe}, order=10100}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.AdaptCachedBodyGlobalFilter@418c020b}, order=2147483637}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.WebsocketRoutingFilter@15f2eda3}, order=2147483646}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.NettyRoutingFilter@70101687}, order=2147483647}, OrderedGatewayFilter{delegate=GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.ForwardRoutingFilter@21618fa7}, order=2147483647}]
Copy
```

> 官方说，未来的版本将对这个接口作出一些调整：
> This interface and usage are subject to change in future milestones.
> from [Spring Cloud Gateway - Global Filters](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.0.RC1/single/spring-cloud-gateway.html#_global_filters)

## 自定义过滤器工厂

如果你还对上一篇关于路由的文章有印象，你应该还得我们在配置中有这么一段

```
filters:
  - StripPrefix=1
  - AddResponseHeader=X-Response-Default-Foo, Default-Bar
Copy
```

`StripPrefix`、`AddResponseHeader`这两个实际上是两个过滤器工厂（GatewayFilterFactory），用这种配置的方式更灵活方便。

我们就将之前的那个`ElapsedFilter`改造一下，让它能接收一个`boolean`类型的参数，来决定是否将请求参数也打印出来。

```java
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import reactor.core.publisher.Mono;
import java.util.Arrays;
import java.util.List;

public class ElapsedGatewayFilterFactory extends AbstractGatewayFilterFactory<ElapsedGatewayFilterFactory.Config> {

    private static final Log log = LogFactory.getLog(GatewayFilter.class);
    private static final String ELAPSED_TIME_BEGIN = "elapsedTimeBegin";
    private static final String KEY = "withParams";

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList(KEY);
    }

    public ElapsedGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            exchange.getAttributes().put(ELAPSED_TIME_BEGIN, System.currentTimeMillis());
            return chain.filter(exchange).then(
                    Mono.fromRunnable(() -> {
                        Long startTime = exchange.getAttribute(ELAPSED_TIME_BEGIN);
                        if (startTime != null) {
                            StringBuilder sb = new StringBuilder(exchange.getRequest().getURI().getRawPath())
                                    .append(": ")
                                    .append(System.currentTimeMillis() - startTime)
                                    .append("ms");
                            if (config.isWithParams()) {
                                sb.append(" params:").append(exchange.getRequest().getQueryParams());
                            }
                            log.info(sb.toString());
                        }
                    })
            );
        };
    }


    public static class Config {

        private boolean withParams;

        public boolean isWithParams() {
            return withParams;
        }

        public void setWithParams(boolean withParams) {
            this.withParams = withParams;
        }

    }
}
```

过滤器工厂的顶级接口是`GatewayFilterFactory`，我们可以直接继承它的两个抽象类来简化开发`AbstractGatewayFilterFactory`和`AbstractNameValueGatewayFilterFactory`，这两个抽象类的区别就是前者接收一个参数（像`StripPrefix`和我们创建的这种），后者接收两个参数（像`AddResponseHeader`）。

[![img](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tNc79ly1fr4w5hwis7j30kx09v3zj.jpg)](https://cdn.jsdelivr.net/gh/zhaoyibo/resource@gh-pages/img/006tNc79ly1fr4w5hwis7j30kx09v3zj.jpg)

`GatewayFilter apply(Config config)`方法内部实际上是创建了一个`GatewayFilter`的匿名类，具体实现和之前的几乎一样，就不解释了。

静态内部类`Config`就是为了接收那个`boolean`类型的参数服务的，里边的变量名可以随意写，但是要重写`List<String> shortcutFieldOrder()`这个方法。

这里注意一下，一定要调用一下父类的构造器把`Config`类型传过去，否则会报`ClassCastException`

```java
public ElapsedGatewayFilterFactory() {
    super(Config.class);
}
```

工厂类我们有了，再把它注册到 Spring 当中

```java
@Bean
public ElapsedGatewayFilterFactory elapsedGatewayFilterFactory() {
    return new ElapsedGatewayFilterFactory();
}
```

然后添加配置（主要改动在第 8 行）

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      default-filters:
        - Elapsed=true
      routes:
        - id: service_customer
          uri: lb://CONSUMER
          order: 0
          predicates:
            - Path=/customer/**
          filters:
            - StripPrefix=1
            - AddResponseHeader=X-Response-Default-Foo, Default-Bar
```

然后我们再次访问 http://localhost:10000/customer/hello/yibo?token=1000 即可在控制台看到以下内容

```
2018-05-08 16:53:02.030  INFO 84423 --- [ctor-http-nio-1] o.s.cloud.gateway.filter.GatewayFilter   : /hello/yibo: 656ms params:{token=[1000]}
```

## 总结

本文主要介绍了 Spring Cloud Gateway 的过滤器，我们实现了自定义局部过滤器、自定义全局过滤器和自定义过滤器工厂，相信大家对 Spring Cloud Gateway 的过滤器有了一定的了解。之后我们将继续在过滤器的基础上研究 如何使用 Spring Cloud Gateway 实现限流和 fallback。

# 参考

> https://www.haoyizebo.com/posts/1e919f7d/
>
> https://blog.csdn.net/forezp/article/details/85057268