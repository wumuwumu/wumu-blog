---
title: centos安装jenkins
date: 2020-9-28 16:00:00
---

# 下载

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum install jenkins
  
```

## 2.配置

```shell
vim /etc/sysconfig/jenkins

#监听端口
JENKINS_PORT="8080"
```

## 3.配置权限

为了不因为权限出现各种问题，这里直接使用root

修改用户为root

```shell
vim /etc/sysconfig/jenkins

#修改配置
$JENKINS_USER="root"
```

修改目录权限

```shell
chown -R root:root /var/lib/jenkins
chown -R root:root /var/cache/jenkins
chown -R root:root /var/log/jenkins
```

重启

```shell
service jenkins restart
ps -ef | grep jenkins
```

## 4.启动

```shell
systemctl start jenkins
```

我这里启动失败了：

![1531198978143](https://images2018.cnblogs.com/blog/668104/201807/668104-20180710201227396-1299962709.png)

错误信息为`Starting Jenkins bash: /usr/bin/java: No such file or directory`是java环境配置的问题。

找到你的jdk目录，我是在 `usr/local/java/jdk1.8.0_171/`下，创建软链接即可：

```shell
ln -s /usr/local/java/jdk1.8.0_171/bin/java /usr/bin/java
```

然后重新启动

![1531199078302](https://images2018.cnblogs.com/blog/668104/201807/668104-20180710201226959-451256225.png)

## 5.安装

访问jenkins地址 http:<ip或者域名>:8080

![1531204667345](https://tva1.sinaimg.cn/large/007S8ZIlgy1gj6iowwig6j30rq0p4myg.jpg)

执行命令查看密码：

```shell
cat /var/lib/jenkins/secrets/initialAdminPassword
```

插件安装选择推荐插件

![1531204844660](https://tva1.sinaimg.cn/large/007S8ZIlgy1gj6iox8ce3j30rq0p7tap.jpg)

安装进行中

![1531204864191](https://tva1.sinaimg.cn/large/007S8ZIlgy1gj6iozkb95j30rl0owdhe.jpg)

插件安装完成以后将会创建管理员账户

![1531205120250](https://tva1.sinaimg.cn/large/007S8ZIlgy1gj6ioyv8m5j30rs0p4aat.jpg)

安装完成：

![1531205170165](https://tva1.sinaimg.cn/large/007S8ZIlgy1gj6ioy5jyqj30rn0p8t9b.jpg)

运行截图：

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1gj6ioxpo29j31h80q80ul.jpg)