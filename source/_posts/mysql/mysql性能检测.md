---
title: mysql性能检测
tags: mysql
abbrlink: 8ce03315
date: 2019-08-31 23:26:45
---

# 性能检测蝉蛹命令

1. show status
2. show processlist
3. show variables

# 瓶颈分析常用命令

## 获取mysql用户下的进程总数

```shell
ps -ef | awk '{print $1}' | grep "mysql" | grep -v "grep" | wc -l
```

## 主机性能状态

```shell
uptime
```

## CPU使用率

```shell
top
vmstat
```

## 磁盘IO量

```shell
vmstat
iostat
```

## swap进出量

```shell
free -m
```

# 数据库性能状态

## QPS

**方法一 基于 questions  计算qps,基于  com_commit  com_rollback 计算tps**

```sql
questions = show global status like 'questions';

uptime = show global status like 'uptime';

qps=questions/uptime
```

```sql
com_commit = show global status like 'com_commit';

com_rollback = show global status like 'com_rollback';

uptime = show global status like 'uptime';

tps=(com_commit + com_rollback)/uptime
```

**方法二  基于 com_\* 的status 变量计算tps ,qps**

使用如下命令：

```sql
show global status where variable_name in('com_select','com_insert','com_delete','com_update');

获取间隔1s 的 com_*的值，并作差值运算

del_diff = (int(mystat2['com_delete'])   - int(mystat1['com_delete']) ) / diff

ins_diff = (int(mystat2['com_insert'])    - int(mystat1['com_insert']) ) / diff

sel_diff = (int(mystat2['com_select'])    - int(mystat1['com_select']) ) / diff

upd_diff = (int(mystat2['com_update'])   - int(mystat1['com_update']) ) / diff


```

**总结：**

Questions 是记录了从mysqld启动以来所有的select，dml 次数包括show 命令的查询的次数。这样多少有失准确性，比如很多数据库有监控系统在运行，每5秒对数据库进行一次show 查询来获取当前数据库的状态，而这些查询就被记录到QPS,TPS统计中，造成一定的"数据污染".

如果数据库中存在比较多的myisam表，则计算还是questions 比较合适。

如果数据库中存在比较多的innodb表，则计算以com_*数据来源比较合适

## TPS

TPS = (Com_commit + Com_rollback) / seconds 

```sql
show status like 'Com_commit'; 
show status like 'Com_rollback';
```

## key Buffer 命中率

key_buffer_read_hits = (1-key_reads / key_read_requests) * 100% 
key_buffer_write_hits = (1-key_writes / key_write_requests) * 100%

```sql
show status like 'Key%';
```

## InnoDB Buffer命中率

innodb_buffer_read_hits = (1 - innodb_buffer_pool_reads / innodb_buffer_pool_read_requests) * 100%

```sql
show status like 'innodb_buffer_pool_read%';
```

## Query Cache命中率

Query_cache_hits = (Qcahce_hits / (Qcache_hits + Qcache_inserts )) * 100%;

```sql
 show status like 'Qcache%';
```

## Table Cache状态量

```sql
show status like 'open%';
```

## Thread Cache 命中率

Thread_cache_hits = (1 - Threads_created / connections ) * 100%

```sql
show status like 'Thread%';
show status like 'Connections';
```

## 锁定状态

```sql
show status like '%lock%';
```

## 复制延时量

```sql
show slave status;
```

## Tmp Table 状况(临时表状况)

```sql
show status like 'Create_tmp%';
```

## Binlog Cache 使用状况 

```sql
show status like 'Binlog_cache%';
```

## Innodb_log_waits

```SQL
show status like 'innodb_log_waits';
```







# 参考

<https://blog.csdn.net/li_adou/article/details/78791972>

