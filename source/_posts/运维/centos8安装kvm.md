---
title: centos8安装kvm
date: 2020-9-05 21:40:23
tags:
- linux
---

# 如何在CentOS/RHEL 8上安装KVM虚拟化

基于内核的虚拟机（简称KVM）是一种开源的标准虚拟化解决方案，已紧密集成到Linux中。它是一个可加载的内核模块，将Linux转换为Type-1（裸机）虚拟机管理程序，该虚拟机管理程序创建了用于运行虚拟机（VM）的虚拟操作平台。

### 精选回答

在KVM下，每个VM是一个Linux进程，由内核调度和管理，并具有专用的虚拟化硬件（即CPU，网卡，磁盘等）。它还支持嵌套虚拟化，使您可以在另一个VM内运行一个VM。

它的一些主要功能包括支持广泛的Linux支持的硬件平台（带有虚拟化扩展的x86硬件（Intel VT或AMD-V）），它使用SELinux和安全虚拟化（sVirt）提供增强的VM安全性和隔离，它继承了内核内存管理功能，并且支持脱机和实时迁移（在物理主机之间迁移正在运行的VM）。

在本文中，您将学习如何在CentOS 8和RHEL 8 Linux中安装KVM虚拟化，创建和管理虚拟机。

准备工作：

全新安装的CentOS 8[服务器](https://www.a5idc.net/)

全新安装的RHEL 8服务器

在RHEL 8服务器上启用了RedHat订阅

此外，通过运行以下命令，确保您的硬件平台支持虚拟化。

```
# grep -e 'vmx' /proc/cpuinfo #Intel systems
# grep -e 'svm' /proc/cpuinfo #AMD systems
```



另外，请确认内核中已加载KVM模块（默认情况下应为KVM模块）。

＃lsmod | grep kvm

这是基于英特尔的测试系统上的示例输出：

![img](https://tp.a5idc.net/wd/1a.png)

在以前的KVM指南系列中，我们展示了如何使用KVM（基于内核的虚拟机）在Linux中创建虚拟机，并展示了如何使用virt-manager GUI工具（根据RHEL已弃用）创建和管理VM。8个文档）。对于本指南，我们将采用不同的方法，我们将使用Cockpit Web控制台。

步骤1：在CentOS 8上设置Cockpit Web控制台

1.在Cockpit是一个易于使用的集成和可扩展的基于Web的界面在网页浏览器来管理Linux服务器。它使您能够执行系统任务，例如配置网络，管理存储，创建VM和使用鼠标检查日志。它使用系统的普通用户登录名和特权，但也支持其他身份验证方法。

它是预先安装的，并已在新安装的CentOS 8和RHEL 8系统上启用，如果尚未安装，请使用以下dnf命令进行安装。应安装cockpit-machines扩展程序以管理基于Libvirt的 VM 。

\# dnf install cockpit cockpit-machines

2.软件包安装完成后，启动座舱插座，使其在系统启动时自动启动，并检查其状态以确认其已启动并正在运行。

\# systemctl start cockpit.socket

\# systemctl enable cockpit.socket

\# systemctl status cockpit.socket

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr7689tj30p807nwej.jpg)

3.接下来，使用firewall-cmd命令将cockpit服务添加到默认启用的系统防火墙中，然后重新加载防火墙配置以应用新更改。

\# firewall-cmd --add-service=cockpit --permanent

\# firewall-cmd --reload

4.要访问CockpitWeb控制台，请打开Web浏览器并使用以下URL进行导航。

https://FQDN:9090/或者https://SERVER_IP:9090/

该Cockpit采用的是自签名证书启用HTTPS，只需使用该连接，当你在浏览器的警告。在登录页面上，使用您的服务器用户帐户凭据。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrar1fvj30wq0n4glu.jpg)

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrbov8uj30wq0nldgf.jpg)

步骤2：安装KVM虚拟化CentOS 8

5.接下来，如下安装虚拟化模块和其他虚拟化软件包。所述的virt安装包提供用于从所述命令行界面进行安装的虚拟机的工具，和一个的virt查看器用于查看虚拟机。

\# dnf module install virt

\# dnf install virt-install virt-viewer

6.接下来，运行virt-host-validate命令以验证主机是否设置为运行libvirt系统管理程序驱动程序。

\# virt-host-validate

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr7hgijj30o506mt8o.jpg)

7.接下来，启动libvirtd守护程序（libvirtd），并使它在每次引导时自动启动。然后检查其状态以确认它已启动并正在运行。

\# systemctl start libvirtd.service

\# systemctl enable libvirtd.service

\# systemctl status libvirtd.service

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrd3s9zj30vz0bj74i.jpg)

步骤3：通过Cockpit设置网桥（虚拟网络交换机）

8.现在创建一个网桥（虚拟网络交换机），将虚拟机集成到与主机相同的网络中。默认情况下，一旦启动libvirtd守护程序，它将激活默认网络接口virbr0，该接口代表以NAT模式运行的虚拟网络交换机。

在本指南中，我们将以桥接模式创建名为br0的网络接口。这将使虚拟机可在主机网络上访问。

在座舱主界面中，单击“ 网络”，然后单击“ 添加网桥”，如以下屏幕截图所示。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrf0gggj30yd0n8dgd.jpg)

9.从弹出窗口中，输入网桥名称，然后选择网桥从站或端口设备（例如，代表以太网接口的enp2s0），如以下屏幕截图所示。然后单击“ 应用”。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr9t6d6j30ls0dedfu.jpg)

10.现在，当您查看“ 接口 ”列表时，新的网桥应显示在此处，几秒钟后，应禁用以太网接口（关闭）。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgraacb2j30ya0bz3yo.jpg)

步骤4：通过Cockpit Web控制台创建和管理虚拟机

11.在座舱主界面中，单击“ 虚拟机”选项，如以下屏幕快照中突出显示。在“ 虚拟机”页面上，单击创建虚拟机。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrejb9rj30wm0ckq30.jpg)

12.将显示一个带有用于创建新VM的选项的窗口。输入连接，名称（例如ubuntu18.04），安装源类型（在测试系统上，我们已将ISO映像存储在存储池下，即/ var / lib / libvirt / images /），安装源，存储，大小，内存如下图所示。输入安装源后，应自动选择OS供应商和操作系统。

还要选中立即启动VM的选项，然后单击“ 创建”。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr9d2jdj30hp0gedg2.jpg)

13.在上一步中单击“ 创建”后，应自动启动VM，并使用提供的ISO映像启动VM。继续安装客户机操作系统（在本例中为Ubuntu 18.04）。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr7y5kjj30wj0hst90.jpg)

如果你点击网络接口的的虚拟机，网络源应注明新建桥网络接口。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr8w9jwj30wm09o74g.jpg)

并且在安装过程中，在配置网络接口的步骤中，您应该能够注意到VM以太网接口从主机网络的DHCP服务器接收IP地址。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrcmjofj30wm0c1q33.jpg)

请注意，您需要安装OpenSSH软件包才能从主机网络上的任何计算机通过SSH访问来宾OS，如上一节所述。

14.客户机操作系统安装完成后，请重新引导VM，然后转到“ 磁盘”并分离/除去VM磁盘下的cdrom设备。然后单击“运行”以启动VM。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrb8getj30rs090wek.jpg)

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrc5he7j30s206mdfr.jpg)

15.现在，在Consoles（控制台）下，您可以使用在OS安装期间创建的用户帐户登录来宾OS。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgr8ep0jj30qu0il74f.jpg)

步骤5：通过SSH访问虚拟机访客操作系统

16.要通过SSH从主机网络访问新安装的来宾OS，请运行以下命令（将10.42.0.197替换为来宾的IP地址）。

$ ssh tecmint@10.42.0.197

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgrdqnrej30qj0ent91.jpg)

17.要关闭，重新启动或删除VM，请从VM列表中单击它，然后使用以下屏幕快照中突出显示的按钮。

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1girgre19rij30sy0b2dfw.jpg)

在本文中，介绍了如何安装KVM虚拟化软件包以及如何通过cockpit Web控制台创建和管理VM。