---
title: nvc-server安装
date: 2018-12-05 21:39:00
tags:
- web
---

## centos 安装 vnc server

**没有实现多用户配置**

```
yum install tigervnc-server -y
cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service

## 然后打开这个配置文件/etc/systemd/system/vncserver@:1.service替换掉默认用户名 
## ExecStart=/sbin/runuser -l <USER> -c "/usr/bin/vncserver %i"
## PIDFile=/home/<USER>/.vnc/%H%i.pid
## 这里我直接用root 用户登录，所以我替换成
ExecStart=/sbin/runuser -l root -c "/usr/bin/vncserver %i"
PIDFile=/root/.vnc/%H%i.pid

## 如果是其他用户的话比如linoxide替换如下
ExecStart=/sbin/runuser -l linoxide -c "/usr/bin/vncserver %i"
PIDFile=/home/linoxide/.vnc/%H%i.pid

systemctl daemon-reload
vncpasswd
## 开放端口
## 重启服务
systemctl enable vncserver@:1.service
systemctl start vncserver@:1.service
```

## ubuntu 安装 vnc viewer

```
sudo dpkg -i VNC-Viewer-6.17.1113-Linux-x64.deb
```

# 参考

> https://my.oschina.net/huhaoren/blog/497394