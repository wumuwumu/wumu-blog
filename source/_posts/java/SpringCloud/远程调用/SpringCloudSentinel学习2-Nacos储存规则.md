---
title: SpringCloudSentinel学习2-Nacos储存规则
tags:
  - SpringCloud
abbrlink: 88fae02d
date: 2021-02-19 14:00:00
---

要通过 Sentinel 控制台配置集群流控规则，需要对控制台进行改造。主要改造规则可以参考：

```javascript
https://github.com/alibaba/Sentinel/wiki/Sentinel-控制台（集群流控管理）#规则配置
```

其控制台推送规则：

- 将规则推送到Nacos或其他远程配置中心
- Sentinel客户端链接Nacos，获取规则配置；并监听Nacos配置变化，如发生变化，就更新本地缓存。

控制台监听Nacos配置变化，如发生变化就更新本地缓存。从而让控制台本地缓存总是和Nacos一致。

# 改造Sentinel

下载Sentinel 源代码，然后对sentinel-dashboard模块进行改造

```javascript
https://github.com/alibaba/Sentinel/archive/1.7.2.zip
```

- 对pom.xml进行修改

```javascript
  <dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <scope>test</scope>
  </dependency>
```

将**<scope>test</scope>**注释掉

```javascript
  <dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
  </dependency>
```

- 修改java代码

找到如下目录（位于test目录）

```javascript
sentinel-dashboard/src/test/java/com/alibaba/csp/sentinel/dashboard/rule/nacos
```

将整个目录拷贝到

```javascript
sentinel-dashboard/src/main/java/com/alibaba/csp/sentinel/dashboard/rule/nacos
```

- 修改com.alibaba.csp.sentinel.dashboard.controller.v2.FlowControllerV2.java

![](http://wumu.rescreate.cn/image20210219164606.png)修改成

- ![](http://wumu.rescreate.cn/image20210219164634.png)修改HTML页面

- 修改配置文件

  - NacosConfig.java

    ```java
    @Bean
    public ConfigService nacosConfigService() throws Exception {
        Properties properties= new Properties();
        if(DashboardConfig.getConfigNacosServerUrl() != null){
            properties.put(PropertyKeyConst.SERVER_ADDR, DashboardConfig.getConfigNacosServerUrl());
        }else {
            properties.put(PropertyKeyConst.SERVER_ADDR,"localhost:8848");
        }
        if(DashboardConfig.getConfigNacosServerNamespace() != null){
            properties.put(PropertyKeyConst.NAMESPACE,DashboardConfig.getConfigNacosServerNamespace());
        }
        if(DashboardConfig.getConfigNacosUsername() != null){
            properties.put(PropertyKeyConst.USERNAME,DashboardConfig.getConfigNacosUsername());
        }
        if(DashboardConfig.getConfigNacosPassword() != null){
            properties.put(PropertyKeyConst.PASSWORD,DashboardConfig.getConfigNacosPassword());
        }
        return ConfigFactory.createConfigService(properties);
    }
    ```

  - DashBoardConfig.java

    ```java
    public class DashboardConfig {
    
        public static final int DEFAULT_MACHINE_HEALTHY_TIMEOUT_MS = 60_000;
    
        /**
         * Login username
         */
        public static final String CONFIG_AUTH_USERNAME = "sentinel.dashboard.auth.username";
    
        /**
         * Login password
         */
        public static final String CONFIG_AUTH_PASSWORD = "sentinel.dashboard.auth.password";
    
        /**
         * Hide application name in sidebar when it has no healthy machines after specific period in millisecond.
         */
        public static final String CONFIG_HIDE_APP_NO_MACHINE_MILLIS = "sentinel.dashboard.app.hideAppNoMachineMillis";
        /**
         * Remove application when it has no healthy machines after specific period in millisecond.
         */
        public static final String CONFIG_REMOVE_APP_NO_MACHINE_MILLIS = "sentinel.dashboard.removeAppNoMachineMillis";
        /**
         * Timeout
         */
        public static final String CONFIG_UNHEALTHY_MACHINE_MILLIS = "sentinel.dashboard.unhealthyMachineMillis";
        /**
         * Auto remove unhealthy machine after specific period in millisecond.
         */
        public static final String CONFIG_AUTO_REMOVE_MACHINE_MILLIS = "sentinel.dashboard.autoRemoveMachineMillis";
    
        public static final String CONFIG_NACOS_SERVER_URL  = "sentinel.dashboard.nacos.server";
    
        public static final String CONFIG_NACOS_SERVER_NAMESPACE = "sentinel.dashboard.nacos.namespace";
    
        public static final String CONFIG_NACOS_USERNAME = "sentinel.dashboard.nacos.username";
    
        public static final String CONFIG_NACOS_PASSWORD = "sentinel.dashboard.nacos.password";
    
        private static final ConcurrentMap<String, Object> cacheMap = new ConcurrentHashMap<>();
        
        @NonNull
        private static String getConfig(String name) {
            // env
            String val = System.getenv(name);
            if (StringUtils.isNotEmpty(val)) {
                return val;
            }
            // properties
            val = System.getProperty(name);
            if (StringUtils.isNotEmpty(val)) {
                return val;
            }
            return "";
        }
    
        protected static String getConfigStr(String name) {
            if (cacheMap.containsKey(name)) {
                return (String) cacheMap.get(name);
            }
    
            String val = getConfig(name);
    
            if (StringUtils.isBlank(val)) {
                return null;
            }
    
            cacheMap.put(name, val);
            return val;
        }
    
        protected static int getConfigInt(String name, int defaultVal, int minVal) {
            if (cacheMap.containsKey(name)) {
                return (int)cacheMap.get(name);
            }
            int val = NumberUtils.toInt(getConfig(name));
            if (val == 0) {
                val = defaultVal;
            } else if (val < minVal) {
                val = minVal;
            }
            cacheMap.put(name, val);
            return val;
        }
    
        public static String getAuthUsername() {
            return getConfigStr(CONFIG_AUTH_USERNAME);
        }
    
        public static String getAuthPassword() {
            return getConfigStr(CONFIG_AUTH_PASSWORD);
        }
    
        public static int getHideAppNoMachineMillis() {
            return getConfigInt(CONFIG_HIDE_APP_NO_MACHINE_MILLIS, 0, 60000);
        }
        
        public static int getRemoveAppNoMachineMillis() {
            return getConfigInt(CONFIG_REMOVE_APP_NO_MACHINE_MILLIS, 0, 120000);
        }
        
        public static int getAutoRemoveMachineMillis() {
            return getConfigInt(CONFIG_AUTO_REMOVE_MACHINE_MILLIS, 0, 300000);
        }
        
        public static int getUnhealthyMachineMillis() {
            return getConfigInt(CONFIG_UNHEALTHY_MACHINE_MILLIS, DEFAULT_MACHINE_HEALTHY_TIMEOUT_MS, 30000);
        }
        
        public static void clearCache() {
            cacheMap.clear();
        }
    
        public static String getConfigNacosServerUrl() {
            return getConfigStr(CONFIG_NACOS_SERVER_URL);
        }
    
        public static String getConfigNacosServerNamespace() {
            return getConfigStr(CONFIG_NACOS_SERVER_NAMESPACE);
        }
    
        public static String getConfigNacosUsername() {
            return getConfigStr(CONFIG_NACOS_USERNAME);
        }
    
        public static String getConfigNacosPassword() {
            return getConfigStr(CONFIG_NACOS_PASSWORD);
        }
    }
    ```

- sidebar.html页面

`sentinel-dashboard/src/main/webapp/resources/app/scripts/directives/sidebar.html`并找到如下代码段后，并把注释放开

![](https://bytetrick.com/upload/2020/10/image-7bc469b8c8854cc1a61b3bd7e90fff4e.png)经过以上步骤就已经把流控规则改造成推模式持久化了。

- 修改请求接口

  src/main/webapp/resources/app/scripts/controllers/identity.js

  ![](http://wumu.rescreate.cn/image20210220152130.png)

**0x02：编译生成jar包**

执行命令

```javascript
mvn clean package -DskipTests
```

编译成功后，在项目的 target 目录可以找到sentinel-dashboard.jar ，执行以下命令可以启动控制台：

```javascript
java -jar sentinel-dashboard.jar
```

**0x03：改造微服务**

- 新建项目olive-nacos-sentinel-datasource

对应的pom.xml文件引入

```javascript
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.sentinel</groupId>
    <artifactId>olive-nacos-sentinel-datasource</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <name>olive-nacos-sentinel-datasource</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>

    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR3</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.1.0.RELEASE </version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
```

- 新建SpringBoot启动类

```javascript
package com.olive;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Hello world!
 *
 */
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

- 创建控制器

```javascript
package com.olive.controller;

import java.util.HashMap;
import java.util.Map;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/getUser")
    public Map<String, Object> getUser() {
        Map<String, Object> result = new HashMap<>();
        result.put("code", "000000");
        result.put("message", "ok");
        return result;
    }
}
```

- 修改配置文件application.yml

```javascript
spring:
  application:
    name: olive-nacos-sentinel-datasource
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      datasource:
        # 名称随意
        flow:
          nacos:
            server-addr: localhost:8848
            dataId: ${spring.application.name}-flow-rules
            groupId: SENTINEL_GROUP
            # 规则类型，取值见：
            # org.springframework.cloud.alibaba.sentinel.datasource.RuleType
            rule-type: flow

server:
  port: 8866
```

**0x04：验证**

**主要验证场景**

- 场景1：用Sentinel控制台【菜单栏的 流控规则 V1 】推送流控规则，规则会存储到Nacos；
- 场景2：直接在Nacos上修改流控规则，然后刷新Sentinel控制台，控制台上的显示也会被修改；
- 场景3：重启Sentinel控制台，并重启微服务；刷新控制台，可以发现规则依然存在。

**启动服务**

- Sentinel控制台
- Nacos
- olive-nacos-sentinel-datasource

**Nacos中创建限流规则的配置**

 http://127.0.0.1:8848/nacos/#/login

```javascript
[
    {
        "resource": "/getUser",
        "limitApp": "default",
        "grade": 1,
        "count": 5,
        "strategy": 0,
        "controlBehavior": 0,
        "clusterMode": false
    }
]
```

如下图

![](http://wumu.rescreate.cn/image20210219164855.png)**访问接口（olive-nacos-sentinel-datasource服务提供的接口）**

​     http://localhost:8866/getUser

**访问Sentinel控制台**

​     http://127.0.0.1:8080/#/login

![](http://wumu.rescreate.cn/image20210219164912.png)以上这条记录就是在Nacos中配置的限流规则。可以测试在**Sentinel控制台**修改规则是否同步到**Nacos，**或者在**Nacos**上修改规则是否同步到**Sentinel控制台**。



# 参考

> https://cloud.tencent.com/developer/article/1665816
>
> https://blog.csdn.net/EnjoyEDU/article/details/109587953

配置sentinel持久化nacos

> https://bytetrick.com/archives/sentinel-dashboard%E6%8C%81%E4%B9%85%E5%8C%96nacos
>
> https://blog.csdn.net/u014386444/article/details/112064291
>
> https://www.cnblogs.com/jian0110/p/14139044.html