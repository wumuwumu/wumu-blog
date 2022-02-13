---
title: kvm日常操作
date: 2021-05-18 12:00:00
tags
- linux
---

# 查看所有的虚拟机

```bash
virsh list 
virsh list --all
```

# 查看虚拟机配置

```bash
virsh dumpxml centos1708vm03
virsh dominfo centos7
```

# 进入虚拟机内部

```bash
virsh console centos7
```

# 扩大内存

## 当前内存小于最大内存

不需要关机

```bash
virsh setmem centos7 2048M
```

## 当前内存和最大内存一样

需要关闭虚拟机

```bash
# 1. 第一种
virsh setmem centos7 1024M --config

# 2.第二种
virsh edit centos7
###########################################
#  <memory unit='KiB'>2097152</memory>
#  <currentMemory unit='KiB'>2048576</currentMemory>
```

# 虚拟机仓用命令

### 虚拟机管理常用参数

```shell
virsh list                                # 获取当前主机所有虚拟机
virsh domstate <ID or Name or UUID>       # 获取虚拟机运行状态
virsh dominfo <ID or Name or UUID>        # 获取虚拟机基本信息
virsh domid <Name or UUID>                # 根据虚拟机名称或UUID获取ID
virsh domname <ID or UUID>                # 根据虚拟机ID或UUID获取名称
virsh dommemstat <ID or Name or UUID>     # 获取虚拟机内存使用情况
virsh setmem <ID or Name or UUID>         # 设置虚拟机内存大小，值不能超过最大分配内存，否则需要关闭虚拟机后设置   
virsh vcpuinfo <ID or Name or UUID>       # 获取vCPU基本信息
virsh vcpupin <ID or Name or UUID> <vCPU> <pCPU> # 将一个vCPU 绑定到物理CPU
virsh setvcpus <ID or Name or UUID> <vCPU num> # 设置虚拟机vCPU 个数
virsh vncdisplay <ID or Name or UUID> # 获取虚拟机的VNC连接参数
virsh create <dom.xml>                    # 根据XML文件创建虚拟机
virsh define <dom.xml>                    # 根据XML文件定义虚拟机，但不启动
virsh start <ID or Name or UUID>          # 启动(预定义的)虚拟机
virsh suspend <ID or Name or UUID>        # 暂停虚拟机
virsh resume <ID or Name or UUID>         # 唤醒虚拟机
virsh shutdown <ID or Name or UUID>       # 关闭虚拟机
virsh reboot <ID or Name or UUID>         # 重启虚拟机
virsh reset  <ID or Name or UUID>         # 强制重启虚拟机
virsh destory <ID or Name or UUID>        # 销毁虚拟机
virsh save <ID> <file.img>                # 保存运行中的虚拟机到一个文件
virsh migrate <ID or Name or UUID> <dst url> # 迁移虚拟机
virsh dump <ID or Name or UUID> <core.file> #coredump保存虚拟机到文件
virsh dumpxml <ID or Name or UUID>        # 输出虚拟机配置
virsh attach-device <ID or Name or UUID> <device.xml> # 添加设备
virsh detach-device <ID or Name or UUID> <device.xml> # 移除设备
virsh console <ID or Name or UUID>        # 连接到虚拟机 
virsh autostart <ID or Name or UUID>      # 设置虚拟机自动启动
virsh auotstart --disable <ID or Name or UUID>      # 取消虚拟机自动启动
```

### 宿主机和hypervisor管理参数

```shell
virsh version                          # 获取libvirt 和hypervisor版本信息
virsh sysinfo                          # 获取宿主机系统信息
virsh nodeinfo                         # 获取宿主机CPU，内存，核数等信息
virsh uri                              # 显示当前连接对象
virsh connect                          # 连接到指定对象
virsh hostname                         # 获取宿主机主机名
virsh capabilities                     # 获取宿主机和虚拟机的架构及特性
virsh freecell                         #  显示当前MUMA单元空闲内存
virsh nodememstats                     # 获取宿主机内存使用情况
virsh nodecpustats                     # 获取宿主机CPU使用情况
virsh qemu-attach                      # 根据PID添加一个QEMU进程到libvirt中
virsh qemu-monitor-command domain [--hmp] command     # 向QEMU monitor发送一个命令
```

### 宿主机网络及虚拟网络管理参数

```shell
virsh iface-list                                 # 获取宿主机网络接口列表
virsh iface-mac <iface name>                     # 获取网络接口mac地址
virsh iface-name <mac>                           # 获取网络接口名称
virsh iface-edit <iface name of uuid>            # 编辑网络接口XML配置文件
virsh iface-dumpxml <iface name of uuid>         # 输出网络接口XML配置
virsh iface-destory <iface name of uuid>         # 关闭网络接口
virsh net-list                                   # 获取libvirt管理的虚拟网络
virsh net-info <net name or uuid>                # 获取虚拟网络基本信息
virsh net-uuid <net name>                        # 获取虚拟网络UUID
virsh net-name <net uuid>                        # 获取虚拟网络名称
virsh net-create <net.xml>                       # 根据XML配置文件创建虚拟网络
virsh net-edit <net name or uuid>                # 编辑虚拟网络信息
virsh net-dumpxml  <net name or uuid>            # 输出虚拟网络XML配置
virsh net-destory <net name or uuid>             # 删除虚拟网络
```

### 存储池和存储卷管理参数

```shell
virsh pool-list                              # 获取libvirt管理的存储池
virsh pool-info <pool name>                  # 获取存储池信息
virsh pool-uuid <pool name>                  # 获取储存池UUID
virsh pool-create <pool.xml>                 # 根据XML配置文件创建存储池
virsh pool-edit <pool name or uuid>          # 编辑存储池配置
virsh pool-destory <pool name or uuid>       # 关闭存储池
virsh pool-delete <pool name or uuid>        # 删除存储池
virsh vol-list <pool name or uuid>           # 获取某个存储池的卷列表
virsh vol-name <vol key or path>             # 获取存储卷名称
virsh vol-path --pool <pool name or uuid> <vol name or key> # 获取存储卷路径
virsh vol-create <vol.xml>                   #  根据XML配置创建存储池
virsh vol-clone <vol name path> <name>       # 克隆存储卷
virsh vol-delete <vol name or key or path>   # 删除存储卷
```

### 其他常用参数

```shell
virsh pwd                            #  获取当前路径
virsh cd <dir>                       # 进入某个路径
virsh echo <param>                   # 输出param
virsh quit                           # 退出交互
virsh exit                           # 退出交互
```



