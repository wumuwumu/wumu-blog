---
title: centos8安装docker
date: 2020-9-05 21:40:23
tags:
- linux
---

**背景简介**：

现在centos已经到了8 ，一直在接触容器方面，为了尝鲜，下载了CentOS8，并尝试安装docker&docker-ce，不料竟然还报了个错（缺少依赖），故及时记录一下，方便其他同学。

 

**安装步骤：**

1. 下载docker-ce的repo

```
curl https://download.docker.com/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
```

2. 安装依赖（这是相比centos7的关键步骤）

```
yum install https://download.docker.com/linux/fedora/30/x86_64/stable/Packages/containerd.io-1.2.6-3.3.fc30.x86_64.rpm
```

3. 安装docker-ce

```
yum install docker-ce
```

4. 启动docker

```
systemctl start docker
```

5. 开机启动docker

```
systemctl enable docker
```

6.安装docker-compose

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

7.添加操作权限

```
sudo chmod +x /usr/local/bin/docker-compose
```

8.设置快捷

```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

9.查看docker-compose 版本

```
docker-compose --version
```