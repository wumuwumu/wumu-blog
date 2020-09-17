---
title: nvc-server安装
date: 2018-12-05 21:39:00
tags:
- web
---

## centos 安装 vnc server

VNC需要系统安装的有桌面，如果是生产环境服务器，安装时使用的最小化安装，那么进行下面操作按章GNOME 桌面。

```bash
# 列出的组列表里有GNOME Desktop。

yum grouplist  

#安装之

yum groupinstall -y "GNOME Desktop" 


# 安装完成后，修改默认启动方式为图形化界面

systemctl set-default graphical.target  //设置成图形模式 
# 如果要换回来 
systemctl set-default multi-user.target  //设置成命令模式 
#然后重启系统即可
```

第一步：安装VNC服务软件，使用root用户执行以下命令（以下操作没有特别说明均在root用户）：

```bash
yum install tigervnc-server -y
```

![img](https://img-blog.csdnimg.cn/2019011120374466.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

安装后可以使用如下命令来验证是否安装成功：

```bash
rpm -qa|grep tigervnc-server
```

![img](https://img-blog.csdnimg.cn/20190111204109802.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

第二步：复制vnc的启动操作脚本, vncserver@:1.service中的：1表示"桌面号"，启动的端口号就是5900+桌面号，即是5901，如果再有一个就是2啦，端口号加1就是5902，以此类推：

```bash
cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service
```

第三步：编辑 /etc/systemd/system/vncserver@:1.service

```bash
vim /etc/systemd/system/vncserver@\:1.service
```

![img](https://img-blog.csdnimg.cn/20190111205105834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)vnc配置文件修改前

找到其中的<USER> ，修改成自己的用户名，如果是root用户登录桌面就使用root用户，如果使用普通用户登录桌面使用普通用户，这里笔者使用用户名：cy

![img](https://img-blog.csdnimg.cn/20190111205309803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)vnc配置文件修改后

修改完毕后保存退出vim。

第四步：设置vnc密码，执行su cy，切换到刚配置文件设置的cy用户，执行（这一步是在cy用户下操作），输入两次密码，输入完成后会提示是否设置view-only password（*“View-only* *password”密码，*只允许查看,无控制权限。）这个可设可不设：

```bash
vncpasswd
```

![img](https://img-blog.csdnimg.cn/20190111205934208.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

第五步：启动服务：

```bash
systemctl start vncserver@\:1.service
```

第一次输入启动服务命令可能会要求输入（从新加载配置文件，新增和配置文件发生变化时都需要执行 daemon-reload 子命令）：

```bash
systemctl daemon-reload
```

执行完毕之后在执行启动命令就可以了：

![img](https://img-blog.csdnimg.cn/20190111205848408.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

可以加入开机启动，下次开机就会自动启动啦：

```bash
systemctl enable vncserver@\:1.service
```

第六步：查看端口是否监听：

```bash
netstat -lnpt|grep Xvnc
```

![img](https://img-blog.csdnimg.cn/20190111211103409.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)这里我们可以看到5901端口已经被监听

第七步：开放防火墙的5901端口：

```bash
firewall-cmd --zone=public --add-port=5901/tcp --permanent
```

如果防火墙没有启动需要先启动防火墙。

![img](https://img-blog.csdnimg.cn/2019011121160287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

当然也可以狠一点，直接停止防火墙：

```bash
systemctl stop firewalld.service
```

![img](https://img-blog.csdnimg.cn/20190111211803178.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

停止之后该需要禁止开机启动：

```
systemctl disable firewalld.service
```

第八步：关闭SELinux，编辑/etc/selinux/config 文件：

```bash
vim /etc/selinux/config
```

![img](https://img-blog.csdnimg.cn/20190111212208680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

将selinux设置为disabled

![img](https://img-blog.csdnimg.cn/2019011121253381.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

到这里vnc服务已经安装完毕，下面就可使用vnc客户端来连接。

第九步：在vnc客户端（vnc viewer）输入服务器IP:桌面号（如192.168.31.100:1），输入后回车：

![img](https://img-blog.csdnimg.cn/20190111213220647.png)

第十步：输入IP后会弹出确认，点击contiue即可：

![img](https://img-blog.csdnimg.cn/20190111213443826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

 

第十一步：输入vnc密码：

![img](https://img-blog.csdnimg.cn/20190111213532135.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

第十二步：登录成功，输入远程机器密码（登录成功后需要输入远程机器的用户的密码，如果没有密码就可以直接进入系统）：

![img](https://img-blog.csdnimg.cn/20190111214637416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

第十三步：成功进入远程桌面：

![img](https://img-blog.csdnimg.cn/20190111214719914.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25haGFuY3k=,size_16,color_FFFFFF,t_70)

至此整个CentOS7.x 的VNC服务安装完毕^_^。

小贴士：vnc服务只能在局域网使用，如果在外网，则需要有公网IP地址，VNC不仅具备内网穿透功能。

## ubuntu 安装 vnc viewer

vnc view的网站`https://www.realvnc.com/en/connect/download/viewer/`

```
sudo dpkg -i VNC-Viewer-6.17.1113-Linux-x64.deb
```

# 参考

> https://my.oschina.net/huhaoren/blog/497394