---
title: SpringCloudGateway基本操作-断言
date: 2021-1-25 15:00:00
---

# 基本配置

## 配置文件

```yaml
spring:
  application:
    name: cloud-gateway
  profiles:
    active: dev
    ################################################################spring cloud gateway##############################################################
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false              #当访问http://网关地址/服务名称（大写）/**地址会自动转发到http://服务名称（大写）/**地址，如果为false就不会自动转发
          lowerCaseServiceId: false   #为true表示服务名称（小写）
      routes:
        - id: cloud-provider-payment  #路由id，需要全局统一，建议使用对应的spring.application.name
          uri: http://localhost:8001  #路由到对应服务的地址
          predicates:
            - Path=/service/**/*      #断言，匹配规则，ant匹配
        ############################################################
        - id: cloud-provider-payment2  #路由id，需要全局统一，建议使用对应的spring.application.name
          uri: http://localhost:8002  #路由到对应服务的地址
          predicates:
            - Path=/service/**/*      #断言，匹配规则，ant匹配
```

## JavaBean配置

```java
@SpringBootApplication
public class DemogatewayApplication {
	@Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
		return builder.routes()
			.route("path_route", r -> r.path("/get")
				.uri("http://httpbin.org"))
			.route("host_route", r -> r.host("*.myhost.org")
				.uri("http://httpbin.org"))
			.route("rewrite_route", r -> r.host("*.rewrite.org")
				.filters(f -> f.rewritePath("/foo/(?<segment>.*)", "/${segment}"))
				.uri("http://httpbin.org"))
			.route("hystrix_route", r -> r.host("*.hystrix.org")
				.filters(f -> f.hystrix(c -> c.setName("slowcmd")))
				.uri("http://httpbin.org"))
			.route("hystrix_fallback_route", r -> r.host("*.hystrixfallback.org")
				.filters(f -> f.hystrix(c -> c.setName("slowcmd").setFallbackUri("forward:/hystrixfallback")))
				.uri("http://httpbin.org"))
			.route("limit_route", r -> r
				.host("*.limited.org").and().path("/anything/**")
				.filters(f -> f.requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter())))
				.uri("http://httpbin.org"))
			.build();
	}
}
```



#### 自定义路由Predicate 断言

在`spring-cloud-gateway`的官方文档中没有给出自定义Predicate ,只留下一句`TODO: document writing Custom Route Predicate Factories`

##### 创建RoutePredicateFactory

```java
/**
 * @author WXY
 */
@Slf4j
public class TokenRoutePredicateFactory extends AbstractRoutePredicateFactory<TokenRoutePredicateFactory.Config> {


    private static final String DATETIME_KEY = "headerName";

    public TokenRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList(DATETIME_KEY);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        log.debug("TokenRoutePredicateFactory Start...");
        return exchange -> {
            //判断header里有放token
            HttpHeaders headers = exchange.getRequest().getHeaders();
            List<String> header = headers.get(config.getHeaderName());
            log.info("Token Predicate headers:{}", header);
            return header.size() > 0;
        };
    }

    public static class Config {
        /**
         * 传输token header key
         */
        private String headerName;

        public String getHeaderName() {
            return headerName;
        }

        public void setHeaderName(String headerName) {
            this.headerName = headerName;
        }
    }
}

```

继承`AbstractRoutePredicateFactory<C>`主要实现其中的两个方法

`shortcutFieldOrder()`-Config对应的字段

`Predicate<ServerWebExchange> apply(Config config)`-具体的逻辑

还有就是构造方法传入用来装配置的类程序会自动把配置的`value`传入`apply`中的入参

##### 初始化RoutePredicateFactory为bean

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author WXY
 *
 */
@Configuration
public class RoutesConfiguration {

    @Bean
    public TokenRoutePredicateFactory initTokenRoutePredicateFactory(){
        return new TokenRoutePredicateFactory();
    }

}
```

或者直接在`TokenRoutePredicateFactory`类上加`@Component`也行

#### 配置自定义的Predicate

##### 使用属性文件配置自定义Predicate

```properties
spring.cloud.gateway.routes[1].predicates[1]=Token=Authorization
```

其中`Toekn`为命名`RoutePredicateFactory`时的前面部分，所以在定义`RoutePredicateFactory`时类名必须后缀为`RoutePredicateFactory`,否则找不到自定义的`Predicate`

##### 使用代码配置

```java
/**
 * @author WXY
 *
 */
@Configuration
public class RoutesConfiguration {
    /**
     * 代码配置路由
     */
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes().route(predicateSpec ->
                predicateSpec.path("/order/**")
                     .and().asyncPredicate(initTokenRoutePredicateFactory().applyAsync(config -> config.setHeaderName("Authorization")))
                     .uri("lb://order-service").id("order-service")
        ).build();
    }

    @Bean
    public TokenRoutePredicateFactory initTokenRoutePredicateFactory(){
        return new TokenRoutePredicateFactory();
    }

}


```

使用代码配置自定义Predicate，主要使用`asyncPredicate`方法，把所需的自定义`RoutePredicateFactory`对象传进去配置`applyAsync`方法传入配置的属性。

# 参考

> https://my.oschina.net/zhousc1992/blog/3194740