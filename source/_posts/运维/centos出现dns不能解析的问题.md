---
title: 域名不能解析
abbrlink: dd239655
date: 2020-05-28 15:00:00
---

DNS有问题，之前手动配置DNS导致，执行如下内容(8.8.8.8是谷歌提供的)

echo 'nameserver 8.8.8.8'>>/etc/resolv.conf

也可使用阿里巴巴提供的DNS域名解析

nameserver 223.5.5.5

nameserver 223.6.6.6

`阿里巴巴DNS介绍` <https://opsx.alibaba.com/service?lang=zh-CN>

![img](https://img2018.cnblogs.com/blog/1114349/201910/1114349-20191026203755691-995379198.png)