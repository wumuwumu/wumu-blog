---
title: centos7修改网卡
tags:
  - linux
abbrlink: d3821118
date: 2018-12-05 21:40:23
---

# 修改mac

使用virtualbox导入一个虚拟机时mac地址是一样的，此时需要修改。 修改mac地址直接在virtualBox的`setting>network`配置中进行修改。

# 修改网卡名称 

## 修改网卡的配置文件

```
vim /etc/sysconfig/network-scripts/ifcfg-eno16777736 //修改NAME，DEVICE 成希望的（不要加ifcfg）

mv ifcfg-eno16777736 ifcfg-eth0 //修改配置文件的名字
```

## 禁用可预测命名规则

```
vim /etc/default/grub
```

添加内核参数： net.ifnames=0 biosdevname=0

```
[root@ansheng network-scripts]# vi /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet net.ifnames=0 biosdevname=0"
GRUB_DISABLE_RECOVERY="true"
```

## 用 grub2-mkconfig 命令重新生成GRUB配置并更新内核

```
[root@ansheng network-scripts]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-327.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-327.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-4dd6b54f74c94bff9e92c61d669fc195
Found initrd image: /boot/initramfs-0-rescue-4dd6b54f74c94bff9e92c61d669fc195.img
done
```

重启系统

