---
title: linux压缩
date: 2019-09-02 22:46:45
tags: linux
---

# tar

```bash
# 打包
tar -cvf xx.tar dirName
# 解包
tar -xvf  xx.tar

# .gz
# 解压
gunzip fileName.gz
gzip -d fileName.gz
# 压缩
gzip fileName

# .tar.gz 和.tgz
# 解压
tar zxvf fileName.tar.gz
# 压缩
tar zcvf filename.tar.gz dirName

# bz2
# 解压
bzip2 -d fileName.bz
bunzip2 fileName.bz

# .tar.bz
# 解压
tar jxvf fileName.tar.bz
# 压缩
tar jcvf fileName.tar.bz dirName

```

# zip

```bash
# 安装
yum install zip unzip

# 解压
unzip mydata.zip -d mydatabak

# 压缩
zip -r abc123.zip abc 123.txt
```

# rar

```bash
# 安装
wget http://www.rarlab.com/rar/rarlinux-x64-5.3.0.tar.gz
tar -zxvf rarlinux-x64-5.3.0.tar.gz // 对应64位下载的
cd rar
make

# 解压
rar x fileName.rar

# 压缩
rar fileName.rar dirName
```

# 7z

```bash
# 安装
yum install p7zip p7zip-plugins

# 压缩
7za a 压缩包.7z 被压缩文件或目录

# 解压
#将压缩包解压到指定目录，注意：指定目录参数-o后面不要有空格
7za x 压缩包.7z -o解压目录
#将压缩包解压到当前目录
7za x 压缩包.7z
```

