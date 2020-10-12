---
title: maven修改默认的jdk版本
date: 2020-10-09 25:00:00
---

> 当系统中装有多个 JDK 版本时，如果控制 Maven 能够指定到正确的版本

## 出现的问题

1. 在使用 `mvn clean package -Dmaven.test.skip=true` 对项目进行打包时
2. 发现进度一直卡在编译无法继续执行
3. 待编译项目 **pom.xml** 中指定的 JDK 版本是 1.8
4. 通过 `mvn -version` 发现 Maven 获取的 JDK 版本是 11
5. 这就是导致 Maven 无法顺利编译项目的根本原因

## 查看当前 Maven 版本

1. 下图中可以看到，Maven 当前指定的是 JDK 11

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjj22t62pjj30hx02vaaq.jpg)

## 查看当前 JDK 版本

1. 下图中可以看到，当前 JDK 版本是 1.8 ，很明显和上图 Maven 获取到的版本不一致
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjj23j7g3yj30cy020zkg.jpg)


## 查看当前系统配置的所有 JDK 版本

1. 下图中可以看到，当前系统配置了三个版本的 JDK ，而系统默认的是 JDK 11 ，与 Maven 获取到的一致

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjj23v44u8j30e003fdgd.jpg)

## 查看通过 JENV 管理的 JDK 版本

1. 下图中可以看到，通过 JENV 切换到的 JDK 版本是 1.8 ，与 `java -version` 获取到的一致

2. 这就说明 JENV 

   虽然可以管理当前 JDK 版本，但是无法切换当前系统的默认 JDK



![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjj244e81dj30cm02g74e.jpg)

## 终极解决方案

1. 在终端输入以下脚本，强制指定 

   ```
   JAVA_HOME
   ```

    的默认版本是 1.8 

   - 这样虽然无法改变当前系统的默认 JDK 版本
   -  **但是可以控制其他软件获取到的 JDK 版本**，这就已经满足需求了

```sh
echo export "JAVA_HOME=\$(/usr/libexec/java_home -v 1.8)" >> ~/.bash_profile
source ~/.bash_profile
```

### 再次查看当前 Maven 版本

1. 下图中可以看到，Maven 当前指定的 JDK 已经替换成了 1.8

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjj24erfomj30he02xjs2.jpg)

### 再次查看当前系统配置的所有 JDK 版本

1. 下图中可以看到，当前系统默认的 JDK 版本依旧是 11

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjj24n80daj30dr03dq3h.jpg)

