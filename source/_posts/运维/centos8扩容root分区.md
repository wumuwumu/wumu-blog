---
title: centos8扩容root分区
tags:
  - linux
abbrlink: 7bc68cab
date: 2020-09-05 21:40:23
---

# 扩展磁盘

最近使用虚拟机的方式弄了个centos8的虚拟机，体验最新centos系统，分配了127g的空间，由于实际需要，发现home空间有好几十g的空间，而我都是使用root用户，无需home空间，因此找到在centos8中把home空间调整到root的方法，这里跟网上找到的centos7是有差别的。

步骤：

1. 使用usb系统进入修复
2. 使用df-h查看空间使用情况，备份home
3. 卸载home文件系统
4. 删除/home所在的lv
5. 扩展/root所在的lv
6. 扩展/root文件系统
7. 重新创建home lv并挂载home
8. 查看最终调整结果

## 使用df-lh查看空间使用情况，备份home

首先登陆ssh，使用df -lh查看空间使用情况

```bash
df -lh
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gireyleo0zj30kf07cjsm.jpg)

root已经不够了，而vps也就自己一个人用，根本不需要用到home，home设置1个g就够了，其余的都给root，这样就可以给root多出来73个g的空间。 这因为一开始没有截图，所以看到的是后面的1g大小，一开始home是74g大小的。 备份home文件到/tmp目录

```bash
tar cvf  /tmp/home.tar /home
# zip -r /tmp/home.zip /home
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gireyr28toj30go0c5dh3.jpg)

## 卸载home文件系统

```bash
fuser -km /home/
umount /home
```

解除home目录的占用，卸载home目录

## 删除/home所在的lv

这一步centos8有很大不同，因为centos7中目录是/dev/mapper/centos-home,而在centos8中为 /dev/mapper/cl-home，因此注意卸载设备名称

```bash
lvremove /dev/mapper/cl-home
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1girez61asuj30nf01xmxc.jpg)

## 扩展/root所在的lv

扩展root空间lv

```bash
lvextend -L +73G /dev/mapper/cl-root  
```

## 扩展/root文件系统

这一步是真正增加root空间，centos7和centos8具有非常大的差别，centos7中是使用xfs_growfs /dev/mapper/centos-root，按逻辑centos8就应该是 xfs_growfs /dev/mapper/cl-root，但是结果就是

```bash
xfs_growfs /dev/mapper/cl-root 
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1girezkj110j30li01gdfw.jpg)

经过摸索发现应该直接使用/就可以了

```bash
xfs_growfs / 
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1girf0736ygj30pp0810ub.jpg)

## 重新创建home lv并挂载home

创建1g空间的home

```bash
lvcreate -L 1G -n home cl
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1girf0kmo2hj30qq02rq39.jpg)文件系统类型设置

```bash
mkfs.xfs /dev/cl/home 
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1girf0uu0ecj30pa07dq4c.jpg)

挂载到home目录

```bash
mount /dev/cl/home /home
```

恢复home目录下文件

```bash
mv /tmp/home.tar /home
cd /home
tar xvf  home.tar
mv home/* .
rm -rf home*
```

## 查看最终调整结果

查看各分区大小

```bash
df -lh
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1girf13w9foj30jt06qjsf.jpg)

## 总结：

本文主要介绍了在centos8系统下调整各分区大小，这里就是/home分区和/root分区，介绍在centos7和centos8下参数差异。熟悉linux系统下的文件系统的分区调整。对于刚装系统分区不合适需要调整centos各分区大小的用户起到指导作用，有疑问再邮件联系吧。



# lvm修改根分区大小

- 参考：
  1. 减小lvm根分区容量: <http://kwokchivu.blog.51cto.com/1128937/724128>
  2. CentOS 5 LVM逻辑卷管理: <http://sunshyfangtian.blog.51cto.com/1405751/860018>

## 目标

home、根各为50GB空间，根空间不足，需缩小home至10GB、扩大根为90GB。

```
lvm> lvscan
  ACTIVE            '/dev/vg_db/lv_root' [50.00 GiB] inherit
  ACTIVE            '/dev/vg_db/lv_home' [50.00 GiB] inherit
  ACTIVE            '/dev/vg_db/lv_swap' [9.83 GiB] inherit
```

## 缩小home、增大根分区

### 进入rescue模式

```
增大root分区是否可以在线完成、不用进rescue状态？找机会试试...
```

从Linux安装光盘启动进入rescue模式；

选择相关的语言，键盘模式，当系统提示启用网络设备时，选择“NO”；

然后在提示允许rescue模式挂载本地Linux系统到/mnt/sysimage下时选择“Skip”，文件系统必须不被挂载才可以对/分区减小容量操作。

最后系统会提示选择进入shell终端还是reboot机器，选择进入shell终端。

### 激活分区

输入lvm命令，进入lvm界面，依次输入pvscan、vgscan、lvscan三个命令扫描pv、vg、lv相关信息。

然后输入lvchange -ay /dev/vg_db/lv_root（上文提到的/分区名称）此命令是激活/分区所在的逻辑卷，输入 quit返回到bash shell界面。

```
lvchange -ay /dev/vg_db/lv_home
lvchange -ay /dev/vg_db/lv_root
```

### 缩小home分区

- 先检查下分区: e2fsck -f /dev/vg_db/lv_home

- 缩小文件系统大小：resize2fs /dev/vg_db/lv_home 10G

- 缩小逻辑卷

  - 输入lvm命令进入lvm模式
  - 缩小逻辑卷：lvreduce -L 10G /dev/vg_db/lv_home
  - 系统会询问是否缩小逻辑卷，输入 y 确定。

- 查看修改结果: vgdisplay，lvdisplay

  ```
  减小LVM中的文件系统必须离线操作(处于umount装态)，要减小文件系统和LV:
      # Unmount相应的文件系统
      # 运行磁盘检查确保卷的完整
      # 减小文件系统
      # 减小LV
  ```

### 扩大根分区

- 先检查下分区: e2fsck -f /dev/vg_db/lv_root
- 扩大逻辑卷:
  - 输入lvm命令进入lvm模式
  - 扩大逻辑卷：lvresize -L +40G /dev/vg_db/lv_root
- 更改文件系统大小
  - resize2fs -p /dev/vg_db/lv_root
- 查看修改结果: lvscan

## 其他操作

### 修改swap卷大小

- 取消激活swap空间: swapoff
- 修改swap分区大小: lvresize -L 4G /dev/vg_db/lv_swap
- 重新格区化: mkswap -f /dev/vb_db/lv_swap
- 激活swap空间: swapon

### 新建逻辑卷lv_develop

- 创建逻辑卷 : lvcreate -L 2.8G -n lv_develop /dev/vb_db
- 创建文件系统 : mkfs.ext3 /dev/vg_db/lv_develop

### 增加物理盘

- fdisk分区，并将分区类型为0×8e(Linux LVM)
- 创建物理卷PV: pvcreate /dev/hdb1
- 创建卷组VG: vgcreate vgtest /dev/hdb1
- 添加PV到VG: vgextend
- 创建逻辑卷LV: lvcreate -L 6000M -n mysql vgtest
- 创建文件系统: mkfs -t ext3 /dev/vgtest/mysql
- 建立新分区卷标: tune2fs –L /mysql /dev/vgtest/mysql
- 加载新分区: mount –t ext3 /dev/vgtest/mysql /mysql
- 卸载卷的顺序:
  1. umount
  2. 卸载逻辑卷:lvremove LVDEVICE
  3. 卸载卷组:vgremove VGNAME
  4. 卸载物理卷:pvremove PVDEVICE

# LVM分区在线扩容

2011-12-19 15:24:16

<http://share.blog.51cto.com/278008/745479>

今天对三台服务器的LV分区进行了一次扩容。本文有点标题党嫌疑，因为只有一台服务器是在线扩容，其它两台都是先卸载再扩容的。

在线扩容的这台服务器，LV分区格式为xfs，原大小1.2TB。增加了一块硬盘，大小为1.8TB。

```
`fdisk` `/dev/cciss/c0d1`                              `# 创建分区，并指定分区类型为LVM (8e) ``pvcreate ``/dev/cciss/c0d1p1`                         `# 创建pv``vgextend VolGroup00 ``/dev/cciss/c0d1p1`              `# 添加新创建的pv到原有vg``lvextend -L +1.8T ``/dev/mapper/VolGroup00-LogVol05`  `# 在线扩容指定lv分区``xfs_growfs ``/dev/mapper/VolGroup00-LogVol05`         `# 使扩容生效。注意xfs文件系统的生效命令！ `
```

其它两台服务器也是新增了一个1.8TB的硬盘，要扩容的LV分区格式为ext3。之所以没有进行在线扩容，是因为没有找到ext2online命令；后来发现，resize2fs也是支持在线扩容的！

```
`lvextend -l +100%FREE ``/dev/mapper/VolGroup00-LogVol05``umount` `-l ``/dev/mapper/VolGroup00-LogVol05``e2fsck -f ``/dev/mapper/VolGroup00-LogVol05`    `# 过程比较长 ``resize2fs ``/dev/mapper/VolGroup00-LogVol05`    `# 也要几分钟时间 ``mount` `/dev/mapper/VolGroup00-LogVol05` `/hdfs`
```

虽然resize2fs可以在线使用，但是对在线lv分区执行e2fsck有点风险！