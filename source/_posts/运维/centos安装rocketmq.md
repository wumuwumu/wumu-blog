---
title: centos安装rockermq
abbrlink: a0b1a962
date: 2020-09-19 18:00:00
---

###### 一、安装jdk 1.8

1. jdk1.8 资源下载

2. 上传至服务器目录，解压（以上传至root 目录为例）

```linux
tar -zxvf jdk-8u221-linux-x64.tar.gz
```

1. 将解压后的文件夹移动到/usr/local目录下

```linux
mv jdk1.8.0_221 /usr/local/
```

1. 编辑以下文件，配置java 环境

```linux
vim /etc/profile
```

1. 具体java 环境配置:

```linux
export JAVA_HOME=/usr/local/jdk1.8.0_221
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib/dt.JAVA_HOME/lib/tools.jar:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:${PATH}
```

###### 此处顺便配置rocketmq 环境

```bash
export NAMESRV_ADDR=127.0.0.1:9876
```

6.刷新文件，使配置立即生效

```linux
source /etc/profile
```

1. 查看是否安装成功

```linux
java -version
```

8.配置成功,将会看到以下类似信息

```css
java version "1.8.0_221"
Java(TM) SE Runtime Environment (build 1.8.0_221-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.221-b11, mixed mode)
```

###### 注意：使用openjdk 安装的话在配置rocketMq时候会出现（JAVA_HOME）问题，当时使用了很多方法，都没有成功，最好还是推荐使用这种方式吧。

###### 二、安装rocketMQ

1. 直接下载安装包（以4.5.1为例）
    官网：[https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.5.1/rocketmq-all-4.5.1-bin-release.zip](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.apache.org%2Fdyn%2Fcloser.cgi%3Fpath%3Drocketmq%2F4.5.1%2Frocketmq-all-4.5.1-bin-release.zip) 

###### 注意：不要下载源码包，否则是没有bin目录的

```ruby
wget http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.5.1/rocketmq-all-4.5.1-bin-release.zip
```

2.解压,将会得到 rocketmq-all-4.5.1-bin-release 文件夹

```css
unzip rocketmq-all-4.5.1-bin-release.zip
```

3.进入bin 目录 修改配置(分别修改runserver.sh 以及 runbroker.sh，因为默认配置内存过大，可能导致启动失败)

```bash
cd /root/rocketmq-all-4.5.1-bin-release/bin/
```

1. 修改 runserver.sh 文件



   ![img](https:////upload-images.jianshu.io/upload_images/12596656-c90d7cc4f81e1343.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1141/format/webp)

   修改位置

```bash
vim runbroker.sh
##使用快捷键 i 开启编辑模式
##找到以下配置，将xms/xmx/xmn 分别修改成以下数值（视机器配置而定）
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
##保存
wq
```

1. 修改 runbroker.sh



   ![img](https:////upload-images.jianshu.io/upload_images/12596656-50cf906fa3423e9d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/604/format/webp)

   修改位置

```bash
vim runbroker.sh
##使用快捷键 i 开启编辑模式
##具体数值视机器而定
JAVA_OPT="${JAVA_OPT} -server -Xms128m -Xmx256m -Xmn256m"
##保存
wq
```

修改配置文件

```css
vim broker.conf 
```

新增如下选项

```xml
brokerIP1=xxxxxx(你的服务器公网ip)
```

1. 分别后台启动 runserver.sh 以及 runbroker.sh

```bash
##启动runserver
nohup sh mqnamesrv &
##以配置文件启动runbroker
nohup sh mqbroker -n localhost:9876 -c /root/rocketmq-all-4.5.1-bin-release/conf/broker.conf &
```

7.查看启动是否成功

```undefined
jps
```

1. 启动成功(可以看到NamesrvStartup以及BrokerStartup)

```undefined
16065 Jps
9679 NamesrvStartup
7887 jar
11279 BrokerStartup
```

10.启动成功日志

```cpp
tail -f ~/logs/rocketmqlogs/namesrv.log
tail -f ~/logs/rocketmqlogs/broker.log
```

11.如果启动失败，请查看失败日志

```csharp
cat nohup.out
```

###### 三、关于防火墙以及安全组规则配置

**首先，请在你的云服务器配置安全组规则通道 9876 端口**
 **其次，centos7默认使用firewalld防火墙，而不是iptables，卸载firewalld，再安装iptables**

```csharp
##卸载firewalld
yum remove firewalld
##安装iptables
yum install iptables-services
##查看防火墙状态
service iptables status
##停止防火墙
service iptables stop
```

###### 四、SpringBoot整合监视台（rocketmq-externals插件）

[GITHUB地址](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapache%2Frocketmq-externals)
 下载[rocketmq-console](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapache%2Frocketmq-externals%2Ftree%2Fmaster%2Frocketmq-console)模块即可
 修改配置文件

```properties
rocketmq.config.namesrvAddr=你的公网IP:9876
##如果你版本小于3.5.8，下面应该配置为false
rocketmq.config.isVIPChannel=false
```

启动即可

```
Description=rockermq name service
Requires=network-online.target
After=network-online.target

[Service]
Type=simple
User=anonymous
WorkingDirectory=/opt/rocketmq
ExecStart=/opt/rocketmq/bin/mqnamesrv
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

