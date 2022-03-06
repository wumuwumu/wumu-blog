---
title: Docker或Podman容器内无法解析DNS问题多种解决方案
abbrlink: e16a0684
date: 2020-10-23 15:00:00
---

## Docker或Podman容器内无法解析DNS问题多种解决方案

## 开机防火墙IP地址伪装（IP地址转发）功能

如果使用的是Centos(RHEL)，并且没有关闭`Firewalld`防火墙，你需要留意是否开启了IP转发功能。

`sudo firewall-cmd --query-masquerade`返回的结果为no，则没有开启。yes则为已经开启了IP地址转发，如果问题没有解决，请继续往下看。

`sudo firewall-cmd --add-masquerade --permanent && sudo firewall-cmd --reload`开启IP地址转发并生效。开启后请再次尝试容器内是否可以正常解析域名。

如果你想了解更多关于`firewalld`，请看我的另一篇博文[这可能是最全的firewalld防火墙常用指令教程](https://blog.yeefire.com/2020_02/Linux_Firewalld.html)

## 开启内核IP地址转发

`cat /proc/sys/net/ipv4/ip_forward`查看是否已经开启，0为关闭状态，1为开启状态。如果已经开启请尝试其他解决方案。

如果返回值为0，则为关闭状态。切换到root用户执行`echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf`，继续执行`sysctl -p /etc/sysctl.conf`使之永久生效。

之后需要重新启动网络服务来使IP地址转发功能生效：

如果你是Centos(RHEL)系，需要重新启动`network`服务。执行`systemctl restart network`重启服务后生效。

如果是Debian/Ubuntu系列的发行版，执行`/etc/init.d/procps restart`或`/etc/init.d/procps.sh restart`生效。

## 配置一个正确的DNS

正常情况下，运行的容器会与主机使用相同的`resolv.conf`文件来进行DNS解析，也就是说如果在本机上可以正常解析域名，那么只要开启了IP地址转发的情况下，容器中也可以正常解析。

你可以先查看你的`resolv.conf`文件文件配置的是否正确。

```
cat /etc/resolv.conf
```

如果发现DNS服务器地址有误，你可以手动编辑此文件，但是NetworkManager会在下次启动时网卡时对该文件还原为它的配置。所以，我们直接使用`NetworkManager`来更改我们的DNS比较妥当些。如果你的网络不是由`NetworkManager`管理。你可以试试看直接修改`/etc/resolv.conf`文件。

执行`sudo nmcli connection`来查看当前所有网络连接。

```
sudo nmcli connection modify ethernet-eth0(你的网卡连接名) ipv4.dns=114.114.114.114
```

`sudo nmcli connection up ethernet-eth0(你的网卡连接名)`重启网卡，使手动配置的DNS生效。

## 强制Docker使用自定义的DNS地址

`vim /etc/docker/daemon.json`修改该文件，如果没有该文件的话直接新建即可。

```
# 修改该文件内容为如下全部文本，注意花括号也包括在内。DNS服务器我使用的是114DNS，你也可以进行更换。

{
  "dns": ["114.114.114.114"]
}
```

希望这篇文章能解决你的问题！如果还是不行，请在评论区留言，将你的大致情况说说看，我们一起研究研究看。





> https://blog.yeefire.com/2020_03/docker_DNS_resolve.html



```bash
## 不知道有没有用
[root@RicenOS ~]# nmcli connection modify docker0 connection.zone trusted
[root@RicenOS ~]# systemctl stop NetworkManager.service
[root@RicenOS ~]# firewall-cmd --permanent --zone=trusted --change-interface=docker0
[root@RicenOS ~]# systemctl start NetworkManager.service
[root@RicenOS ~]# nmcli connection modify docker0 connection.zone trusted
[root@RicenOS ~]# systemctl restart docker.service
```



