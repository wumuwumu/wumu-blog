---
title: centos8安装docker
tags:
  - linux
abbrlink: a00f9d53
date: 2020-09-05 21:40:23
---

1. 下载docker-ce的repo

   ```bash
   dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
   ```

2. 安装

   ```bash
   dnf install docker-ce --nobest -y
   ```

3. 运行

   ```bash
   systemctl start docker
   systemctl enable docker
   docker --version
   
   ```

4. 安装docker-compose

   ```bash
   dnf install curl -y
   curl -L https://github.com/docker/compose/releases/download/1.25.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
   
   chmod +x /usr/local/bin/docker-compose
   docker-compose --version
   ```
