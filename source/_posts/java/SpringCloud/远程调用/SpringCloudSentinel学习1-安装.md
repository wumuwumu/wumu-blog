---
title: SpringCloudSentinel学习1-安装
tags:
  - SpringCloud
abbrlink: 4bd69169
date: 2021-02-19 12:00:00
---

# 手动安装

## 获取 Sentinel 控制台

您可以从 [release 页面](https://github.com/alibaba/Sentinel/releases) 下载最新版本的控制台 jar 包。

您也可以从最新版本的源码自行构建 Sentinel 控制台：

- 下载 [控制台](https://github.com/alibaba/Sentinel/tree/master/sentinel-dashboard) 工程
- 使用以下命令将代码打包成一个 fat jar: `mvn clean package`

## 启动

> **注意**：启动 Sentinel 控制台需要 JDK 版本为 1.8 及以上版本。

使用如下命令启动控制台：

```bash
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```

其中 `-Dserver.port=8080` 用于指定 Sentinel 控制台端口为 `8080`。

从 Sentinel 1.6.0 起，Sentinel 控制台引入基本的**登录**功能，默认用户名和密码都是 `sentinel`。可以参考 [鉴权模块文档](https://sentinelguard.io/zh-cn/docs/dashboard.html#鉴权) 配置用户名和密码。

> 注：若您的应用为 Spring Boot 或 Spring Cloud 应用，您可以通过 Spring 配置文件来指定配置，详情请参考 [Spring Cloud Alibaba Sentinel 文档](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Sentinel)。

# Docker安装

- Clone project

  ```
  git clone https://github.com/zhoutaoo/sentinel-dashboard-docker.git
  ```

- Build Image

  ```
  cd build
  docker build -t cike/sentinel-dashboard-docker .
  ```

- Run With docker

```
docker run -p 8021:8021 -it cike/sentinel-dashboard-docker
```

- Run With docker-compose

  ```
  docker-compose up
  ```

- Open the Sentinel Dashboard console in your browser

  link：http://127.0.0.1:8021/