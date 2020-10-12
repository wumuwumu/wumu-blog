---
title: nexus 使用
date: 2020-10-09 15:00:00
---

# 使用

## 账号密码

```xml
<!-- 在servers标签下配置server, 包括: 私服的用户名和密码, 在deploy项目时需要用到 -->
    <server>
        <id>releases</id>
        <username>admin</username>
        <password>admin123</password>
    </server>
    <server>
        <id>snapshots</id>
        <username>admin</username>
        <password>admin123</password>
    </server>
```

## 使用

```xml
<repository>
            <id>StongPublic</id>
            <name>StongCentral</name>
            <url>http://xxxx/repository/maven-public/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
```



# 发布

## 1.查看当前正在使用的settings.xml

```bash
mvn help:effective-settings
```

在pom文件中加入如下配置：

```xml
    <!--使用分发上传将项目打成jar包，上传到nexus私服上-->
    <distributionManagement>
        <!--发布版本仓库-->
        <repository>
            <!--nexus服务器中用户名：在settings.xml中和<server>的id一致-->
            <id>releases</id>
            <!--自定义名称-->
            <name>RELEASES PUBLISH</name>
            <!--仓库地址-->
            <url>http://xx.xx.xx.xx:xxxx/repository/maven-releases/</url>
        </repository>
        <!--快照版本仓库-->
        <snapshotRepository>
            <!--nexus服务器中用户名：在settings.xml中和<server>的id一致-->
            <id>snapshots</id>
            <!--自定义名称-->
            <name>SNAPSHOTS PUBLISH</name>
            <!--仓库地址-->
            <url>http://xx.xx.xx.xx:xxxx/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

```

## 2.在settings.xml文件中加入如下配置：

```xml
	<servers>
	 <server>
	  <id>releases</id>
	  <username>admin</username>	
	  <password>####@123</password>
	 </server>
	 <server>
	  <id>snapshots</id>
	  <username>admin</username>
	  <password>####@123</password>
	 </server>
  </servers>
123456789101112
```

## 3.发生如下错误可能是配置的账号信息有误的原因：

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy (default-deploy) on project demo-childA: Failed to deploy artifacts: Could not transfer artifact com.ecp:demo-childA:jar:1.0-20190625.082808-
1 from/to maven-snapshots (http://xx.xxx.xx.xx:xxxx/repository/maven-snapshots/): Failed to transfer file http://xx.xxx.xx.xx:xxxx/repository/maven-snapshots/com/ecp/demo-childA/1.0-SNAPSHOT/demo-childA-1.0-20190625.082808-1
.jar with status code 401 -> [Help 1]
123
```

可以使用：mvn help:effective-settings命令查看settings.xml配置文件查看配置信息

## 4.将本地项目发布到Nexus私服：

```bash
mvn clean deploy

## 跳过javadoc
mvn deploy  -Dmaven.javadoc.skip=true

```

