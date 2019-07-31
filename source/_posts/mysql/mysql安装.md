---
title: centos安装mysql
date: 2019-03-29 15:45:32
tags:
- linux
---

# 添加 MySQL YUM 源

```
$wget 'https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm'
$sudo rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
$yum repolist all | grep mysql
mysql-connectors-community/x86_64 MySQL Connectors Community                  36
mysql-tools-community/x86_64      MySQL Tools Community                       47
mysql57-community/x86_64          MySQL 5.7 Community Server                 187
```

# 安装MySQL

```
## 安装最新版
$sudo yum install mysql-community-server
$ sudo yum install mysql   ## 安装客户端
## 安装老版本
## 1. yum-config-manager
$ sudo dnf config-manager --disable mysql57-community
$ sudo dnf config-manager --enable mysql56-community
$ yum repolist | grep mysql
mysql-connectors-community/x86_64 MySQL Connectors Community                  36
mysql-tools-community/x86_64      MySQL Tools Community                       47
mysql56-community/x86_64          MySQL 5.6 Community Server                 327
## 2. 直接修改 /etc/yum.repos.d/mysql-community.repo
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

# 启动Mysql

```
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

## 修改密码

```
## 获取临时密码
sudo grep 'temporary password' /var/log/mysqld.log
$ mysql -uroot -p  #输入查看到的密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

mysql的密码存在安全等级

```
shell> mysql_secure_installation
```

```
mysql> SHOW VARIABLES LIKE 'validate_password%';
```

**validate_password_number_count**参数是密码中至少含有的数字个数，当密码策略是MEDIUM或以上时生效。

**validate_password_special_char_count**参数是密码中非英文数字等特殊字符的个数，当密码策略是MEDIUM或以上时生效。

**validate_password_mixed_case_count**参数是密码中英文字符大小写的个数，当密码策略是MEDIUM或以上时生效。

**validate_password_length**参数是密码的长度，这个参数由下面的公式生成

validate_password_number_count+ validate_password_special_char_count+ (2 * validate_password_mixed_case_count)

**validate_password_dictionary_file**参数是指定密码验证的字典文件路径。

**validate_password_policy**这个参数可以设为0、1、2，分别代表从低到高的密码强度，此参数的默认值为1，如果想将密码强度改弱，则更改此参数为0。



## 修改密码策略

更改密码策略为LOW  

```
mysql> set global validate_password_policy=0;
```

更改密码长度  

```
mysql> set global validate_password_length=0;
```

## 安全设置

```
## 会提示设置5个关键位置
## 设置 root 密码
## 禁止 root 账号远程登录
## 禁止匿名账号（anonymous）登录
## 删除测试库
## 是否确认修改
$ mysql_secure_installation
```

# 安装三方插件

```
yum --disablerepo=\* --enablerepo='mysql*-community*' list available
```

# 修改编码

```
## /etc/my.cnf
[client]
default-character-set = utf8
[mysqld]
default-storage-engine = INNODB
character-set-server = utf8
collation-server = utf8_general_ci #不区分大小写
collation-server =  utf8_bin #区分大小写
collation-server = utf8_unicode_ci #比 utf8_general_ci 更准确
```

# 修改服务器时间

```
## mysql 中默认的时间戳是 UTC 时间，需要改为服务器时间的话官网提供了 3 种方式
$ mysql_tzinfo_to_sql tz_dir
$ mysql_tzinfo_to_sql tz_file tz_name
$ mysql_tzinfo_to_sql --leap tz_file
## tz_dir 代表服务器时间数据库，CentOS 7 中默认的目录为 /usr/share/zoneinfo ，tz_name 为具体的时区。如果设置的时区需要闰秒，则使用 --leap，具体的用法如下：
$ mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root -p mysql
$ mysql_tzinfo_to_sql tz_file tz_name | mysql -u root mysql
$ mysql_tzinfo_to_sql --leap tz_file | mysql -u root mysql
> set global time_zone = '+8:00';  ##修改mysql全局时区为北京时间，即我们所在的东8区
> set time_zone = '+8:00';  ##修改当前会话时区
> flush privileges;  #立即生效
## 通过修改my.cnf配置文件来修改时区
# vim /etc/my.cnf  ##在[mysqld]区域中加上
default-time_zone = '+8:00'
# /etc/init.d/mysqld restart  ##重启mysql使新时区生效
```

