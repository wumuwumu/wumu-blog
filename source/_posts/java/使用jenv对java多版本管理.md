---
title: 使用jenv对java多版本管理
tags:
  - java
abbrlink: 940fe8dd
date: 2019-10-25 10:42:43
---

- 配置JDK环境变量

打开 vim ~/.bash_profile 文件 进行添加

```bash
export JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_112.jdk/Contents/Home
export JAVA_7_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home
# 默认激活 jdk8
export JAVA_HOME=$JAVA_8_HOME
```





编辑完成，重新加载 .bash_profile

```
$ source ~/.bash_profile
```

#### jEnv安装

- 安装

```
$ brew install jenv
```

- 配置

安装了zsh，配置如下

```
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(jenv init -)"' >> ~/.zshrc
```



如果是默认的bash

```
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(jenv init -)"' >> ~/.bash_profilec
```

#### jEnv配置JDK

查看安装的java版本，如果我们一开始未添加jdk，执行jenv versions 应该是空的，* 号位置表示当前的jdk版本

```bash
$ jenv versions
  system
  1.7
* 1.7.0.80 (set by /Users/gulj/.java-version)
  1.8
  1.8.0.112
  oracle64-1.7.0.80
  oracle64-1.8.0.112
```

重启下terminal，为jEnv添加java版本

```
添加jdk7
$ jenv add /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home
添加jdk8
$ jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_112.jdk/Contents/Home
```

> 添加完jdk7和jdk8之后，再执行 **jenv versions** 命令就会看到我们添加的jdk

#### jEnv常用命令

- 移除指定版本jdk

```
$ jenv remove 1.8.0.111
```

- 选择一个jdk版本

```
$ jenv local 1.8.0.111
```

- 设置默认的jdk版本

```
$ jenv global 1.8.0.111
```

- 查看当前版本jdk的路径

```
jenv which java
```