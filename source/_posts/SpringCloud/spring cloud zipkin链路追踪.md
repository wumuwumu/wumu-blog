---
title: spring cloud zipkin链路追踪
date: 2020-10-17 18:00:00
---

# 下载zipkin

```bash
docker run -d -p 9411:9411 openzipkin/zipkin


curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

# 依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

# 配置

- spring.zipkin.base-url指定了Zipkin服务器的地址
- spring.sleuth.sampler.percentage将采样比例设置为1.0，说明全部都需要。

```
spring:
  zipkin:
    base-url: http://localhost:9000
  sleuth:
    sampler:
      percentage: 1.0
```

