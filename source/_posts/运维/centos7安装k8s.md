---
title: centos7安装K8S
date: 2021-04-19 12:00:00
tags:
- centos
- k8s
---

# 安装准备（每台服务器）

### 关闭防火墙

```
systemctl stop firewalld
systemctl disable firewalld
```

## 关闭Selinux

- 临时禁用

```
setenforce 0
```

- 永久禁用

```
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

## 禁用交换分区

- 

```
swapoff -a
```

- 永久禁用，打开/etc/fstab注释掉swap那一行。

```
sed -i 's/.*swap.*/#&/' /etc/fstab
```

## 修改内核参数

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

sysctl --system
```

## 调整系统时区

### 设置系统时区为 中国/上海

```
timedatectl set-timezone Asia/Shanghai
```

### 将当前的 UTC 时间写入硬件时钟

```
timedatectl set-local-rtc 0
```

### 重启依赖于系统时间的服务

```
systemctl restart rsyslog
systemctl restart crond
```

# 安装docker(每台服务器)

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum update -y && yum install -y docker-ce
## 创建 /etc/docker 目录

grub2-set-default 'CentOS Linux (4.4.202-1.el7.elrepo.x86_64) 7 (Core)' && reboot
# 重新设置内核


systemctl restart docker && systemctl enable docker
# 设置开机自动

mkdir /etc/docker
# 配置 daemon.

cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m"
    }
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# 重启docker服务
systemctl daemon-reload && systemctl restart docker 


```



# 安装 Kubeadm(所有服务器)

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF 
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet-1.18.4 kubeadm-1.18.4 kubectl-1.18.4

systemctl enable kubelet.service

```

# 安装主节点

```bash
kubeadm config print init-defaults > kubeadm-config.yaml

# 修改(以及新增)kubeadm-config.yaml以下内容
localAPIEndpoint:
  advertiseAddress: 192.168.1.200
kubernetesVersion: v1.18.4
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12


```

## 下载初始化必备镜像
因为 Kubernetes 所需要的初始化必备镜像都是从谷歌官方拉取的，不会走 docker 的加速镜像服务器。由于谷歌被墙，所以我们需要自行下载必备镜像，怎么做呢？

首先列出使用的镜像以及版本号

```bash 
kubeadm config images list --config kubeadm-config.yaml
```

接着，我们通过国内的第三方镜像仓库下载完毕后再更改镜像名称与谷歌的镜像名称一致即可

我们编写一个 shell 脚本

```bash
#!/bin/bash

images=(
    kube-apiserver:v1.18.4
    kube-controller-manager:v1.18.4
    kube-scheduler:v1.18.4
    kube-proxy:v1.18.4
    pause:3.2
    etcd:3.4.3-0
    coredns:1.6.7
)

for imageName in ${images[@]} ; do
    docker pull mirrorgcrio/$imageName
    docker tag mirrorgcrio/$imageName k8s.gcr.io/$imageName
    docker rmi mirrorgcrio/$imageName
done
```


当然这里用什么版本，是由 Kubeadm 的版本节点的。通过上方的列出使用的镜像以及版本号我们可以很清楚的知道要下什么版本，下哪些的镜像了。

然后我们执行脚本，开始下载镜像 (注意哦，这个下载镜像，2 个 node 节点也要做的)

给予执行权限

chmod +x docker-download.sh

初始化,并且将标准输出同时写入至kubeadm-init.log文件

kubeadm init --config=kubeadm-config.yaml | tee kubeadm-init.log
完毕后，控制台输出的日志会告诉我们继续执行什么指令以及 node 节点如何加入

执行日志中的指令

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 部署网络

只要操作 master 节点即可噢～

现在我们执行 kubectl 相关指令已经会有了正常响应，但是此时节点处于 NotReady 的状态，这是因为我们还没有为 Kubernetes 指定它的网络模式。我们使用 flannel 来作为它的网络模式，这样就可以让不同节点上的容器跨主机通信。如果对这块感兴趣，可以自行搜索 flannel 的网络实现。

现在我们开始安装 flannel

### 创建文件夹

```bash
mkdir -p /usr/local/install-k8s/plugin/flannel
```

### 进入flannel文件夹

```bash
cd /usr/local/install-k8s/plugin/flannel
```

### 下载flannel配置文件

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 安装flannel

```bash
kubectl create -f kube-flannel.yml
```



## 常用命令

查看命名空间为kube-system的pod情况

```bash
kubectl get pod -n kube-system
```

查看更详细的信息

```bash
kubectl get pod -n kube-system -o wide
```

查看k8s所有节点连接情况

```
kubectl get node
```

## 加入子节点
根据 kubeadm-init.log 日志文件内容或者安装的时候的标准输出，在 node 节点执行指令

```bash
kubeadm join 192.168.1.200:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:7c2677754a3b09da10d5ffa6a7d6348ad63219cd69d2f5c3a27642d4b95ff15b
```


即可将 node 节点加入 master

# 子节点配置

## 下载docker相关组件

参考主节点配置

## 加入集群

## 清除加入

```
kubeadm reset
```

# 参考

> https://learnku.com/docs/go-micro-build/1.0/kubernetes-1804-cluster-installation-tutorial-based-on-centos7/8877#6077e2
>
> https://www.yinxiang.com/everhub/note/f420816c-2019-47a1-8dcd-7b3ade25ac1f
>
> https://www.jianshu.com/p/4a5e0de015a9