---
title: postgresql配置文件
tags:
  - postgresql
abbrlink: 3dcf42bf
date: 2019-12-09 16:29:39
---

# 1、配置文件

配置文件控制着一个PostgreSQL服务器实例的基本行为，主要包含postgresql.conf、pg_hba.conf、pg_ident.conf

（1）postgresql.conf

   该文件包含一些通用设置，比如内存分配，新建database的默认存储位置，PostgreSQL服务器的IP地址，日志的位置以及许多其他设置。9.4版引入了

   一个新的postgresql.auto.conf文件，任何时候执行Altersystem SQL命令，都会创建或重写该文件。该文件中的设置会替代postgresql.conf文件中的设置。

（2）pg_hba.conf

​    该文件用于控制访问安全性，管理客户端对Postgresql服务器的访问权限，内容包括：允许哪些用户连接到哪个数据库，允许哪些IP或者哪个网段的IP连

​    接到本服务器，以及指定连接时使用的身份验证模式

（3）pg_ident.conf

   pg_hba.conf的权限控制信息中的身份验证模式字段如果指定为ident方式，则用户连接时系统会尝试访问pg_ident文件，如果该文件存在，则系统会基于

​    文件内容将当前执行登录操作的操作系统用户映射为一个PostgreSQL数据库内部用户的身份来登录。

# 2、查看配置文件的位置：    

```bash
postgres=# selectname,setting from pg_settings where category='File Locations';
       name        |                 setting                 
-------------------+-----------------------------------------
 config_file       |/var/lib/pgsql/9.6/data/postgresql.conf
 data_directory    | /var/lib/pgsql/9.6/data
 external_pid_file | 
 hba_file          | /var/lib/pgsql/9.6/data/pg_hba.conf
 ident_file        | /var/lib/pgsql/9.6/data/pg_ident.conf
```



 

# 3、postgresql.conf

3.1、关键的设置

```bash
postgres=# selectname,context,unit,setting,boot_val,reset_val from pg_settings where namein('listen_addresses','max_connections','shared_buffers','effective_cache_size','work_mem','maintenance_work_mem')order by context,name;
         name         | context   | unit | setting |boot_val  | reset_val 
----------------------+------------+------+---------+-----------+-----------
 listen_addresses     | postmaster |      | *      | localhost | *
 max_connections      | postmaster |      | 100    | 100       | 100
 shared_buffers       | postmaster | 8kB  | 16384  | 1024      | 16384
 effective_cache_size | user       | 8kB | 524288  | 524288    | 524288
 maintenance_work_mem | user       | kB  | 65536   | 65536     | 65536
 work_mem             | user       | kB  | 4096    | 4096      | 4096
(6 rows)
```



 

context 设置为postmaster，更改此形参后需要重启PostgreSQL服务才能生效；

设置为user，那么只需要执行一次重新加载即可全局生效。重启数据库服务会终止活动连接，但重新加载不会。  

unit 字段表示这些设置的单位

setting是指当前设置；boot_val是指默认设置；reset_val是指重新启动服务器或重新加载设置之后的新设置

在postgresql.conf中修改了设置后，一定记得查看一下setting和reset_val并确保二者是一致，否则说明设置并未生效，需要重新启动服务器或者重新加载设置

3.2、postgresql.auto.conf与postgresql.conf区别

对于9.4版及之后的版本来说，Postgresql.auto.conf的优先级是高于postgresql.conf的，如果这两个文件中存在同名配置项，则系统会优先选择前者设定的值。

3.3、postgresql.conf以下网络设置，修改这些值是一定要重新启动数据库服务的

listen_addresses 一般设定为localhost或者local，但也有很多人会设为*，表示使用本机任一IP地址均可连接到Postgresql服务

port 默认值 为5432

max_connections

3.4、以下四个设置对系统性能有着全局性的影响，建议你在实际环境下通过实测来找到最优值

(1)share_buffers

​    用于缓存最近访问过的数据页的内存区大小，所有用户会话均可共享此缓存区

​    一般来说越大越好，至少应该达到系统总内存的25%，但不宜超过8GB，因为超过后会出现“边际收益递减”效应。

​    需重启postgreSQL服务

（2）effective_cache_size

一个查询执行过程中可以使用的最大缓存，包括操作系统使用的部分以及PostgreSQL使用部分，系统并不会根据这个值来真实地分配这么多内存，但是规划器会根据这个值来判断系统能否提供查询执行过程中所需的内存。如果将此设置设得过小，远远小于系统真实可用内存量，那么可能会给规划器造成误导，让规划器认为系统可用内存有限，从而选择不使用索引而是走全表扫描（因为使用索引虽然速度快，但需要占用更多的中间内存）。

在一台专用于运行PostgreSQL数据库服务的服务器上，建议将effective_cache_size的值设为系统总内存的一半或者更多。

此设置可动态生效，执行重新加载即可。

（3）work_mem

此设置指定了用于执行排序，哈希关联，表扫描等操作的最大内存量。

此设置可动态生效，执行重新加载即可。

   （4）mintenance_work_mem

​     此设置指定可用于vaccum操作（即清空已标记为“被删除”状态的记录）这类系统内部维护操作的内存总量。

​     其值不应大于1GB

此设置可动态生效，执行重新加载即可。

3.5修改参数命令

```bash
Alter system set work_mem=8192;
```



设置重新加载命令

```bash
Select pg_reload_conf();
```



3.6、遇到修改了postgresql.conf文件，结果服务器崩溃了这种情况

定位这种问题最简单的方法是查看日志文件，该文件位于postgresql数据文件夹的根目录或者pg_log子文件夹下。

# 4、pg_hba.conf

cat /var/lib/pgsql/9.6/data/pg_hba.conf

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD
 
# "local" isfor Unix domain socket connections only
local   all             all                                     peer
# IPv4 localconnections:
host    all             all             0.0.0.0/0               trust
# IPv6 localconnections:
host    all             all             ::1/128                 ident
# Allow replicationconnections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            ident
#host    replication     postgres        ::1/128                 ident
```



(1)   身份验证模式，一般以下几种常用选项：ident、trust、md5以及password

1. 1版本开始引入了peer身份验证模式。

Ident和peer模式公适用于Linux，Unix和Mac,不适用于windwos

Reject模式，其作用是拒绝所有请求。

(2)   如果你将+0.0.0./0 reject+规则放到+127.0.0.1/32 trust+的前面，那么此时本地用户全都无法连接，即使下面有规则允许也不行。

（3）各模式

trust最不安全的身份验证模式，该模式允许用户“自证清白”，即可以不用密码就连到数据库

md5该模式最常用，要求连接发起者携带用md5算法加密的密码

password 不推荐，因为该模式使用明文密码进行身份验证，不安全

ident：该身份验证模式下，系统会将请求发起的操作系统用户映射为PostgreSQL数据库内部用户，并以该内部用户的权限登录，且此时无需提供登录密码。操作系统用户与数据库内部用户之间的映射关系会记录在pg_ident.conf文件中。

peer使用发起端的操作系统名进行身份验证

# 5、配置文件的重新加载

```bash
/usr/pgsql-9.6/bin/pg_ctlreload -D /var/lib/pgsql/9.6/data/ 
systemctlreload postgresql-9.6.service 
selectpg_reload_conf();
```





 