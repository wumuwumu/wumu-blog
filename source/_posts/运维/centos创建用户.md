---
title: centos创建用户
date: 2020-9-05 21:40:23
tags:
- linux
---

# 创建用户

```bash
useradd wumu

## 给用户添加组，一定要加a
(FC4: usermod -G groupA,groupB,groupC user)
-a 代表 append， 也就是 将自己添加到 用户组groupA 中，而不必离开 其他用户组。
#命令的所有的选项，及其含义：
Options:
-c, --comment COMMENT         new value of the GECOS field
-d, --home HOME_DIR           new home directory for the user account
-e, --expiredate EXPIRE_DATE set account expiration date to EXPIRE_DATE
-f, --inactive INACTIVE       set password inactive after expiration
to INACTIVE
-g, --gid GROUP               force use GROUP as new primary group
-G, --groups GROUPS           new list of supplementary GROUPS
-a, --append          append the user to the supplemental GROUPS
mentioned by the -G option without removing
him/her from other groups
-h, --help                    display this help message and exit
-l, --login NEW_LOGIN         new value of the login name
-L, --lock                    lock the user account
-m, --move-home               move contents of the home directory to the new
location (use only with -d)
-o, --non-unique              allow using duplicate (non-unique) UID
-p, --password PASSWORD       use encrypted password for the new password
-s, --shell SHELL             new login shell for the user account
-u, --uid UID                 new UID for the user account
-U, --unlock                  unlock the user account

usermod -a -G wumugroup wumu
passwd wumu
```

# 添加sudo权限

```bash
visudo


#找到如下行数
root  ALL=(ALL)   ALL
#添加
username ALL=(ALL) ALL
```

# 免密码登录

```bash
ssh-keygen
ssh-copy-id -i .ssh/id_rsa.pub  用户名字@192.168.x.xxx
ssh 用户名字@192.168.x.xxx
```

# 使用pem登录

```bash
#在本地生成公钥私钥
ssh-keygen
#输入命令后，一路回车，即可。

#将本地的公钥传到服务器上
ssh-copy-id -i ~/.ssh/id_rsa.pub remote-host
#会提示你输入密码，成功之后，会帮助你把公钥放在服务器上，供登录使用。

#把本地的私钥转为 pem 格式，供windows上的 ssh 客户端使用
openssl rsa -in ~/.ssh/id_rsa -outform pem > id_rsa.pem
chmod 700 id_rsa.pem
#这样就导出了pem格式的私钥，因为公钥已经在服务器了，所以只要服务器上的公钥不删除，用这把私钥就能登录服务器,一般来说，经过这样设置之后，可以把ssh 密码登录的方式禁用掉，使得服务器更加安全。

#关闭 ssh 密码登录
vi /etc/ssh/sshd_config
#修改

PasswordAuthentication no
#重启 ssh 服务
service sshd restart
```

