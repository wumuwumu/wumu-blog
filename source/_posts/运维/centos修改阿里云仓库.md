---
title: centos修改成阿里云仓库
abbrlink: 68bf35c9
date: 2020-09-28 16:00:00
---

# DNF/YUM源配置文件替换为阿里家的

由于系统安装的包管理配置文件链接的国外的服务器，导致我们安装软件、升级内核和升级软件的时候会从国外的服务器下载相关文件。由于众所周知的原因，国外服务器的网速真的不敢恭维，所以我们要把他们替换为国内的服务器，这样安装和升级软件的速度就会提高，降低维护人员在等待上所花费的时间。
因为阿里源文件里面已经包含了AppStream、Base、centosplus、Extras和PowerTools的相关内容，所以需要把这些文件改名为bak，不让系统执行。

```bash
cd /etc/yum.repos.d/
mv /etc/yum.repos.d/CentOS-AppStream.repo /etc/yum.repos.d/CentOS-AppStream.repo.bak
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
mv /etc/yum.repos.d/CentOS-centosplus.repo /etc/yum.repos.d/CentOS-centosplus.repo.bak
mv /etc/yum.repos.d/CentOS-Extras.repo /etc/yum.repos.d/CentOS-Extras.repo.bak
mv /etc/yum.repos.d/CentOS-PowerTools.repo /etc/yum.repos.d/CentOS-PowerTools.repo.bak

```

做完以上修改以后，就可以下载新的阿里源文件了，因为默认没有装wget，我们可以用curl来执行以下命令：

```bash
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo

```

如果有wget也可以执行以下命令

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo

```

如果没有安装wget，运行这个命令会提示“bash: wget: 未找到命令”，那就用curl的那个命令来执行好了。或者你也可以先安装wget，很简单，只需要下面一个命令即可（前提是在将上面的文件改为“.bak”之前，如果已经改了，先改回去再执行下述命令）

```bash
yum -y install wget
```