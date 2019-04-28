---
title: git基本操作
date: 2019-04-09 13:59:25
tags:
- git
---

# 简介

在实际开发中，会使用git作为版本控制工具来完成团队协作。因此，对基本的git操作指令进行总结是十分有必要的，本文对一些术语或者理论基础，不重新码字，可以[参考廖雪峰老师的博文](https://link.juejin.im?target=https%3A%2F%2Fwww.liaoxuefeng.com%2Fwiki%2F0013739516305929606dd18361248578c67b8067c8c017b000)，本文只对命令做归纳总结。

git的通用操作流程如下图（来源于网络）



![git操作通用流程](https://user-gold-cdn.xitu.io/2018/4/25/162fcc0987bf1c0a)



主要涉及到四个关键点：

1. 工作区：本地电脑存放项目文件的地方，比如learnGitProject文件夹；
2. 暂存区（Index/Stage）：在使用git管理项目文件的时候，其本地的项目文件会多出一个.git的文件夹，将这个.git文件夹称之为版本库。其中.git文件夹中包含了两个部分，一个是暂存区（Index或者Stage）,顾名思义就是暂时存放文件的地方，通常使用add命令将工作区的文件添加到暂存区里；
3. 本地仓库：.git文件夹里还包括git自动创建的master分支，并且将HEAD指针指向master分支。使用commit命令可以将暂存区中的文件添加到本地仓库中；
4. 远程仓库：不是在本地仓库中，项目代码在远程git服务器上，比如项目放在github上，就是一个远程仓库，通常使用clone命令将远程仓库拷贝到本地仓库中，开发后推送到远程仓库中即可；

更细节的来看：



![git几个核心区域间的关系](<assets/162fcc0e7e711dc7.png>)



日常开发时代码实际上放置在工作区中，也就是本地的XXX.java这些文件，通过add等这些命令将代码文教提交给暂存区（Index/Stage），也就意味着代码全权交给了git进行管理，之后通过commit等命令将暂存区提交给master分支上，也就是意味打了一个版本，也可以说代码提交到了本地仓库中。另外，团队协作过程中自然而然还涉及到与远程仓库的交互。

因此，经过这样的分析，git命令可以分为这样的逻辑进行理解和记忆：

1. git管理配置的命令；

   **几个核心存储区的交互命令：**

2. 工作区与暂存区的交互；

3. 暂存区与本地仓库（分支）上的交互；

4. 本地仓库与远程仓库的交互。

# 安装

[git安装](https://git-scm.com/book/zh/v1/%E8%B5%B7%E6%AD%A5-%E5%AE%89%E8%A3%85-Git)

https://git-scm.com/

# 配置

```bash
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"

$ git config --global core.editor emacs
$ git config --list
$ git config user.name
```

# 快速开始

```bash
$ git init  # 初始化工程
$ git add * # 将文件添加到暂存区
$ git commit -m  # 提交
$ git clone https://github.com/libgit2/libgit2
```

# 常用命令

## add

1. git add -A   保存所有的修改

2. git add .     保存新的添加和修改，但是不包括删除

3. git add -u   保存修改和删除，但是不包括新建文件。

## commit

1. git commit -m
2. git commit -ma   // -a是添加全部修改
3. git commit --amend

## checkout

1. git checkout — //使用暂缓区替换工作区
2. git checkout  切换分支
3. git checkout head — //直接使用本地参考的文件覆盖工作区文件

## rm

1. git rm  // 删除工作区，并且提交
2. git rm —cached  // 只删除暂存区
3. git rm -f   // 暂存区和工作区都删除

# reset

**谨慎使用！！！！！**

- --soft – 缓存区和工作目录都不会被改变
- --mixed – 默认选项。缓存区和你指定的提交同步，但工作目录不受影响
- --hard – 缓存区和工作目录都同步到你指定的提交

## revert

前提是已经提交，缺点：一次回滚过个记录会出现冲突。