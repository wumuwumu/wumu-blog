---
title: Oracle表管理
tags:
  - oracle
abbrlink: d8f3fd68
date: 2018-12-29 21:45:47
---

## 数据类型

```
## 字符型
char 定长，后面空格补全
varchar2() 变长
clob 字符型大对象

## 数字类型
number
number(5，2) 标识5位有效数，2位小数-999.99-999.99
number(5) 5位整数

## 日期类型
date
timestramp
## 图片
blob 二进制4g,为了安全可以放入数据库
```

# 表操作

```
create table table_name(
)

drop table table_name;

rename table_name to other_table_name;

alter table table_name add ...;
alter table table_name modify ...;
alter table table_name drop column ...;
```

