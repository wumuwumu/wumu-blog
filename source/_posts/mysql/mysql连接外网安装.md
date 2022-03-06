---
title: mysql连接外网安装
tags:
  - mysql
abbrlink: 91583e3e
date: 2019-12-09 15:24:23
---

## 添加 MySQL YUM 源

根据自己的操作系统选择合适的[安装源](https://link.jianshu.com?t=http://dev.mysql.com/downloads/repo/yum/)，和其他公司一样，总会让大家注册账号获取更新，注意是 Oracle 的账号，如果不想注册，下方有直接[下载的地址](https://link.jianshu.com?t=https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm)，下载之后通过 `rpm -Uvh` 安装。

```bash
$wget 'https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm'
$sudo rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
$yum repolist all | grep mysql
mysql-connectors-community/x86_64 MySQL Connectors Community                  36
mysql-tools-community/x86_64      MySQL Tools Community                       47
mysql57-community/x86_64          MySQL 5.7 Community Server                 187
```

先解释下为什么下载的是 5.7 版本的，现在最新的是 5.7 版本的，当然官网默认都是最新版本的，但是下载的页面也有说明

> The MySQL Yum repository includes the latest versions of:
>  MySQL 8.0 (Development)
>  MySQL 5.7 (GA)
>  MySQL 5.6 (GA)
>  MySQL 5.5 (GA - Red Hat Enterprise Linux and Oracle Linux Only)
>  MySQL Cluster 7.5 (GA)
>  MySQL Cluster 7.6 (Development)
>  MySQL Workbench
>  MySQL Fabric
>  MySQL Router (GA)
>  MySQL Utilities
>  MySQL Connector / ODBC
>  MySQL Connector / Python
>  MySQL Shell (GA)

也就是说这个安装源包含了上面列举的这些版本，当然包括 5.6 版本的。

## 选择安装版本

如果想安装最新版本的，直接使用 yum 命令即可

```bash
$sudo yum install mysql-community-server
```

如果想要安装 5.6 版本的，有2个方法。命令行支持 `yum-config-manager` 命令的话，可以使用如下命令：

```ruby
$ sudo dnf config-manager --disable mysql57-community
$ sudo dnf config-manager --enable mysql56-community
$ yum repolist | grep mysql
mysql-connectors-community/x86_64 MySQL Connectors Community                  36
mysql-tools-community/x86_64      MySQL Tools Community                       47
mysql56-community/x86_64          MySQL 5.6 Community Server                 327
```

或者直接修改 `/etc/yum.repos.d/mysql-community.repo` 这个文件

```ruby
# Enable to use MySQL 5.6
[mysql56-community]
name=MySQL 5.6 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.6-community/el/7/$basearch/
enabled=1 #表示当前版本是安装
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

[mysql57-community]
name=MySQL 5.7 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/
enabled=0 #默认这个是 1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```

通过设置 `enabled` 来决定安装哪个版本。

设置好之后使用 `yum` 安装即可。

## 启动 MySQL 服务

启动命令很简单

```php
$sudo service mysqld start 
$sudo systemctl start mysqld #CentOS 7
$sudo systemctl status mysqld
● mysqld.service - MySQL Community Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2017-05-27 12:56:26 CST; 15s ago
  Process: 2482 ExecStartPost=/usr/bin/mysql-systemd-start post (code=exited, status=0/SUCCESS)
  Process: 2421 ExecStartPre=/usr/bin/mysql-systemd-start pre (code=exited, status=0/SUCCESS)
 Main PID: 2481 (mysqld_safe)
   CGroup: /system.slice/mysqld.service
           ├─2481 /bin/sh /usr/bin/mysqld_safe --basedir=/usr
           └─2647 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/...
```

说明已经正在运行中了。

对于 MySQL 5.7 版本，启动的时候如果数据为空的，则会出现如下提示

> The server is initialized.
>  An SSL certificate and key files are generated in the data directory.
>  The validate_password plugin is installed and enabled.
>  A superuser account 'root'@'localhost' is created. A password for the superuser is set and stored in the error log [file.To](https://link.jianshu.com?t=http://file.To) reveal it, use the following command:
>  `sudo grep 'temporary password' /var/log/mysqld.log`

简单的说就是服务安装好了，SSL 认证的文件会在 data 目录中生存，密码不要设置的太简单了，初始密码通过下面的命令查看，赶紧去改密码吧。
 安装提示，查看密码，登录数据库，然后修改密码：

```ruby
$ mysql -uroot -p  #输入查看到的密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

## MySQL 5.6 的安全设置

由于 5.7 版本在安装的时候就设置好了，不需要额外设置，但是 5.6 版本建议从安全角度完善下，运行官方脚本即可

```ruby
$ mysql_secure_installation
```

会提示设置5个关键位置

1. 设置 root 密码
2. 禁止 root 账号远程登录
3. 禁止匿名账号（anonymous）登录
4. 删除测试库
5. 是否确认修改

## 安装第三方组件

查看 yum 源中有哪些默认的组件：

```php
$ yum --disablerepo=\* --enablerepo='mysql*-community*' list available
```

需要安装直接通过 `yum` 命令安装即可。

## 修改编码

在 `/etc/my.cnf` 中设置默认的编码

```csharp
[client]
default-character-set = utf8

[mysqld]
default-storage-engine = INNODB
character-set-server = utf8
collation-server = utf8_general_ci #不区分大小写
collation-server =  utf8_bin #区分大小写
collation-server = utf8_unicode_ci #比 utf8_general_ci 更准确
```

## 创建数据库和用户

创建数据库

```bash
CREATE DATABASE <datebasename> CHARACTER SET utf8;
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
GRANT privileges ON databasename.tablename TO 'username'@'host';
SHOW GRANTS FOR 'username'@'host';
REVOKE privilege ON databasename.tablename FROM 'username'@'host';
DROP USER 'username'@'host';
```

其中

- username：你将创建的用户名
- host：指定该用户在哪个主机上可以登陆，如果是本地用户可用 localhost，如果想让该用户可以从任意远程主机登陆，可以使用通配符 %
- password：该用户的登陆密码，密码可以为空，如果为空则该用户可以不需要密码登陆服务器
- privileges：用户的操作权限，如 SELECT，INSERT，UPDATE 等，如果要授予所的权限则使用ALL
- databasename：数据库名
- tablename：表名，如果要授予该用户对所有数据库和表的相应操作权限则可用 * 表示，如 *.*

# 参考

<https://www.jianshu.com/p/7cccdaa2d177>