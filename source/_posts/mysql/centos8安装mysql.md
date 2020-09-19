---
title: centos8安装mysql
date: 2020-09-17 16:55:17
tags:
- mysql
---

以root身份或[具有sudo特权的用户身份使用CentOS软件包管理器安装MySQL 8.0服务器](https://www.myfreax.com/create-a-sudo-user-on-centos/)：

```bash
sudo dnf install @mysql
```

`@mysql`模块安装MySQL及其所有依赖项。

安装完成后，通过运行以下命令来启动MySQL服务并使其在启动时自动启动：

```bash
sudo systemctl enable --now mysqld
```

要检查MySQL服务器是否正在运行，请输入：

```bash
sudo systemctl status mysqld
```

```bash
● mysqld.service - MySQL 8.0 database server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2019-10-17 22:09:39 UTC; 15s ago
   ...
```

## 保护MySQL

运行`mysql_secure_installation`脚本，该脚本执行一些与安全性相关的操作并设置MySQL根密码：

```bash
sudo mysql_secure_installation
```

将要求您配置[ `VALIDATE PASSWORD PLUGIN` ](https://dev.mysql.com/doc/refman/8.0/en/validate-password.html)，该工具用于测试MySQL用户密码的强度并提高安全性。密码验证策略分为三个级别：低，中和强。如果您不想设置验证密码插件，请按`ENTER`。

在下一个提示符下，将要求您为MySQL根用户设置密码。完成此操作后，脚本还将要求您删除匿名用户，限制root用户对本地计算机的访问，并删除测试数据库。您应该对所有问题回答“是”。

要从命令行与MySQL服务器进行交互，请使用MySQL客户端实用程序，它作为依赖项安装。通过键入以下内容测试根访问权限：

```bash
mysql -u root -p
```

出现提示时输入[ root密码](https://www.myfreax.com/how-to-reset-a-mysql-root-password/)，将为您提供MySQL shell，如下所示：

```bash
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 8.0.17 Source distribution
```

就是这样！您已经在CentOS服务器上安装并保护了MySQL 8.0，并准备使用它。

## 身份验证方法

由于CentOS 8中的某些客户端工具和库与`caching_sha2_password`方法不兼容，CentOS 8存储库中包含的MySQL 8.0服务器被设置为使用旧的`mysql_native_password`身份验证插件。上游MySQL 8.0版本。

`mysql_native_password`方法适用于大多数设置。但是，如果您想将默认身份验证插件更改为`caching_sha2_password`，这会更快并提供更好的安全性，请打开以下配置文件：

```bash
sudo vim /etc/my.cnf.d/mysql-default-authentication-plugin.cnf
```

将`default_authentication_plugin`的值更改为`caching_sha2_password`：

```bash
[mysqld]
default_authentication_plugin=caching_sha2_password
```

[关闭并保存文件](https://www.myfreax.com/how-to-save-file-in-vim-quit-editor/)，然后重新启动MySQL服务器以使更改生效：

```bash
sudo systemctl restart mysqld
```