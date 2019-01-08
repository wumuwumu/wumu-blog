---
title: Oracle基本管理
date: 2018-12-29 20:37:37
tags:
- oracle
---

# 用户管理

```sql
## 创建用户
create user test identified by test;
show user;

## 删除用户
delete test (cascade);

## 修改用户
alter user test identified by wumu;
alter user test expire;

## 用户口令
## 密码输错三次就密码锁定2天
create profile lock_account limit failed_login_attempts 3 password_lock_time 2;
alter user tea profile lock_account;
## 解锁
alter user tea account unlock;
## 每10天需要修改密码，宽限期为两天
create profile myprofile limit password_life_time 10 password_grace_time 2;
alter user tea profile myprofile;
## 口令10天后可以重用
create profile password_history limit password_lift_time 10 password_grace_time 2 password_reuse_time 10

## 撤销profile
drop profile my_profile CASCADE；

```

# 权限管理

```
## 授权
grant system_privilege|all privileges to {user identified by password |role|}
[with admin option]

grant object_privileage | All
on schema.object
to user | role
[with admin option]
[with the grant any object]

grant select on test to wumu with grant option;
grant connect tp wumu with admin option;

## create session 用于登录
## dba 管路员
## resource 可以建表
## desc table_name
## 撤销权限
## 如果授权者的权限被撤回，那么它的被授予者也会失去相关的权限
invoke system_privilege from user|role
invoke object_privilege|All on scheme.object from user|role [cascade contraints]

## 查询权限
## 系统权限放在DBA_SYS_PRIVS
## 对象权限放在数据字典DBA_TAB_PRIVS

```

