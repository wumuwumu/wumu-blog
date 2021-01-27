---
title: SpringCloudGateway基础操作-熔断
date: 2021-1-26 15:00:00
---

微服务系统中熔断限流环节，对保护系统的稳定性起到了很大的作用，作为网关，Spring Cloud Gateway也提供了很好的支持。先来理解下熔断限流概念：

> - `熔断降级`：在分布式系统中，网关作为流量的入口，大量请求进入网关，向后端远程系统或服务发起调用，后端服务不可避免的会产生调用失败（超时或者异常），失败时不能让请求堆积在网关上，需要快速失败并返回回去，这就需要在网关上做熔断、降级操作。
> - `限流`：网关上有大量请求，对指定服务进行限流，可以很大程度上提高服务的可用性与稳定性，限流的目的是通过对并发访问/请求进行限速，或对一个时间窗口内的请求进行限速来保护系统。一旦达到限制速率则可以拒绝服务、排队或等待、降级。

下文就网关如何进行超时熔断、异常熔断和访问限流进行示例说明。示例包含两个模块项目，一个为网关项目`gateway`，一个为下游业务项目`downstream`。

![img](https:////upload-images.jianshu.io/upload_images/5056014-42c7c5a4b2b0a8b9.png?imageMogr2/auto-orient/strip|imageView2/2/w/467/format/webp)

## 超时异常熔断

### 构建网关目：

pom.xml

```xml
  <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring.cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>io.spring.platform</groupId>
                <artifactId>platform-bom</artifactId>
                <version>${spring.platform.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

 <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
    </dependencies>
```

application.yml

```yml
server:
  port: 8089

spring:
  application:
    name: spring-cloud-gateway
  cloud:
    gateway:
      routes:
        - id: service_customer
          #下游服务地址
          uri: http://127.0.0.1:8083/
          order: 0
          #网关断言匹配
          predicates:
            - Path=/gateway/**
          filters:
            #熔断过滤器
            - name: Hystrix
              args:
                name: fallbackcmd
                fallbackUri: forward:/defaultfallback
            - StripPrefix=1

#熔断器配置
hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: SEMAPHORE
          thread:
            timeoutInMilliseconds: 3000
  shareSecurityContext: true

#网关日志输出
logging:
  level:
    org.springframework.cloud.gateway: TRACE
    org.springframework.http.server.reactive: DEBUG
    org.springframework.web.reactive: DEBUG
    reactor.ipc.netty: DEBUG
```

以上配置的意思是：

- 网关服务以端口8089暴露
- 访问`http://127.0.0.1:8089/gateway/`开头的请求，将都被路由到下游`http://127.0.0.1:8083/`下，且`gateway`部分将被移除（`StripPrefix=1`）。比如[http://127.0.0.1:8089/gateway/test](https://links.jianshu.com/go?to=http%3A%2F%2F127.0.0.1%3A8089%2Fgateway%2Ftest) ----> [http://127.0.0.1:8083/test](https://links.jianshu.com/go?to=http%3A%2F%2F127.0.0.1%3A8083%2Ftest)
- 超时异常熔断采用hystrix的SEMAPHORE策略，超时时间为3秒，如果下游服务不可达（异常），将由fallbackcmd处理，路由到本地[http://127.0.0.1:8089/defaultfallback](https://links.jianshu.com/go?to=http%3A%2F%2F127.0.0.1%3A8089%2Fdefaultfallback) 处理。

### 构建defaultfallback处理器

```java
@RestController
public class SelfHystrixController {

    @RequestMapping("/defaultfallback")
    public Map<String,String> defaultfallback(){
        System.out.println("请求被熔断.");
        Map<String,String> map = new HashMap<>();
        map.put("Code","fail");
        map.put("Message","服务异常");
        map.put("result","");
        return map;
    }

}
```

先不构建下游服务，直接运行网关，访问地址`http://127.0.0.1:8089/gateway/test`，出现如下情况：

![img](https:////upload-images.jianshu.io/upload_images/5056014-f2a77eedb84ae8bf.png?imageMogr2/auto-orient/strip|imageView2/2/w/589/format/webp)

构建下游服务项目，该项目为简单的spring boot web项目，具体配置不详述，添加服务类：

```java
@RestController
public class TestController {
    @RequestMapping("/timeout")
    public String timeout(String name) {
        try {
            Thread.sleep(5000);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "timeout params:" + name;
    }
}
```

# 参考

> https://www.jianshu.com/p/b58c13b227bf