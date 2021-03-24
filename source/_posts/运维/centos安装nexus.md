---
title: centos安装nexus
tag:
  - linux
abbrlink: 24033b1e
date: 2020-09-25 11:00:00
---

# 下载

```bash
 cd /opt
 sudo curl -O https://sonatype-download.global.ssl.fastly.net/nexus/3/nexus-3.3.1-01-unix.tar.gz
 sudo tar -xzvf nexus-3.3.1-01-unix.tar.gz
 sudo ln -s nexus-3.3.1-01 nexus

```

# 配置

```bash
 sudo useradd nexus
 sudo chown -R nexus:nexus /opt/nexus
 sudo chown -R nexus:nexus /opt/sonatype-work/
```

修改运行用户

```bash
 sudo vi /opt/nexus/bin/nexus.rc
 
 run_as_user="nexus"
```

# 开机启动

```bash
sudo ln -s /usr/local/nexus-3.13.0-01/bin/nexus /etc/init.d/nexus
#查看nexus服务状态、启动服务、停止服务等
service nexus status/start/stop
#设置为开机自启动/关闭等
chkconfig nexus on/off
```

