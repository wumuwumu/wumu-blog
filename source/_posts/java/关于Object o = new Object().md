---
title: 关于Object o = new Object()
date: 2020-12-23 12:00:00
---

## 1、请解释一下对象的创建过程？（半初始化）

![](http://wumu.rescreate.cn/image20201222130815.png)

## 2、加问DCL与volatile问题？（指令重排）

volatile的作用：保持线程可见性，防止指令重排

DCL 是双重检查锁



3、对象在内存中的存储布局？（对象和数组的存储不同）





4、对象头具体包括什么？（markedword klasspointer）

synchronized锁信息







5、对象怎么定位？（直接  间接）







6、对象怎么分配？（栈上-线程本地-eden-old）







7、Object o = new Object()在内存中占用多少字节。