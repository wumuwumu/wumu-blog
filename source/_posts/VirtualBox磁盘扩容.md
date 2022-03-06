---
title: VirtualBox磁盘扩容
tags:
  - web
abbrlink: 27ec5408
date: 2018-12-05 21:36:46
---

## 扩展磁盘文件

### VDI

```
VBoxManage modifyhd centos.vdi --resize 16000  # 单位M
```

### VMDK

```
VBoxManage clonehd "centos.vmdk" "centos.vdi" --format vdi     # vmdk是转换前的文件，vdi是转换之后的文件
VBoxManage modifyhd "centos.vdi" --resize 16000                # 这里的单位是M
VBoxManage clonehd "centos.vdi" "resized.vmdk" --format vmdk   #可以再转回来
```

## 使用克隆

本人在使用的时候，前面两种方式不能实现，采用第三种方式

```
VBoxManage createhd -filename centos7-main-64g -size 65536 -format VDI -variant Standard  # 创建一个新的磁盘，磁盘大小为想要的大小
VBoxManage clonemedium ../centos7-main\ Clone/centos7-main\ Clone.vdi centos7-main-64g.vdi --existing  # 将原有的磁盘复制到新磁盘上
```

## 磁盘扩容

这里可以使用gparted进行磁盘的扩容

1. 下载gparted-live镜像
2. 设置iso镜像开机启动
3. 进行分区的修改

## LVM扩容

如果你没有使用逻辑卷就可以跳过这节。如果使用逻辑卷也可以通过添加新磁盘的形式对文件系统进行扩容，这种方式更加简单方便。

### 创建PE、VG

### 扩展LV

```
sudo vgextend VolGroup /dev/sda4       # 通过新卷的方式扩展到卷组
lvresize -l +122 /dev/centos/root      # 直接扩容
```

### 刷新逻辑分区容量

```
xfs_growfs /devices/centos/root    # resize2fs是不能成功的
```