---
title: springcloud-eureka
tags:
  - spring-cloud
abbrlink: 47816c18
date: 2019-01-06 18:27:25
---

# 建立工程

1. 添加依赖

   ```xml
   <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
           <version>${spring-cloud.version}</version>
   </dependency>
   ```

2. 添加`Application`

   ```java
   @SpringBootApplication
   @EnableEurekaServer
   public class EurekaApplication {
       public static void main(String[] arg){
           SpringApplication.run(EurekaApplication.class,arg);
       }
   }
   
   ```

3. 添加配置文件

   ```yaml
   server:
     port: 8761
   
   eureka:
     instance:
       hostname: localhost
     client:
       registerWithEureka: false ## 是否注册到eureka server
       fetchRegistry: false  ## 是否获取Eureka server 注册信息，单机可以设置为false
       serviceUrl:
         defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
   		## 默认http://localhost:8761/eureka
   spring:
     application:
       name: eurka-server
   ```

4. 运行工程，访问`127.0.0.1:9761`可以看到web界面。

# 安全

1. 添加依赖

   ```
    <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-security</artifactId>
           </dependency>
   ```

2. 添加配置

   - 老版本

   ```yaml
   security:
   	basic:
   		true
       user:
         name: wumu
         password: wumu 
   ```

   - 新版本

   ```yaml
   security:
       user:
         name: wumu
         password: wumu
   ```


# 问题

1. 在依赖包中同时添加的`spring-cloud-starter-netflix-eureka-server`与`springb-boot-starter-web`两个依赖会导致tomcat的依赖问题，应用不能启动。

