---
title: mysql自带的数据库
date: 2019-07-27 18:06:05
tags:
- mysql
---

#  information_schema

1. SCHEMATA表：提供了当前mysql实例中所有数据库的信息。是show databases的结果取之此表。

2. TABLES表：提供了关于数据库中的表的信息（包括视图）。详细表述了某个表属于哪个schema，表类型，表引擎，创建时间等信息。是show tables from schemaname的　　结果取之此表。

3. COLUMNS表：提供了表中的列信息。详细表述了某张表的所有列以及每个列的信息。是show columns from schemaname.tablename的结果取之此表。

4. STATISTICS表：提供了关于表索引的信息。是show index from schemaname.tablename的结果取之此表。

5. USER_PRIVILEGES（用户权限）表：给出了关于全程权限的信息。该信息源自mysql.user授权表。是非标准表。

6. SCHEMA_PRIVILEGES（方案权限）表：给出了关于方案（数据库）权限的信息。该信息来自mysql.db授权表。是非标准表。

7. TABLE_PRIVILEGES（表权限）表：给出了关于表权限的信息。该信息源自mysql.tables_priv授权表。是非标准表。

8. COLUMN_PRIVILEGES（列权限）表：给出了关于列权限的信息。该信息源自mysql.columns_priv授权表。是非标准表。

9. CHARACTER_SETS（字符集）表：提供了mysql实例可用字符集的信息。是SHOW CHARACTER SET结果集取之此表。

10. COLLATIONS表：提供了关于各字符集的对照信息。

11. COLLATION_CHARACTER_SET_APPLICABILITY表：指明了可用于校对的字符集。这些列等效于SHOW COLLATION的前两个显示字段。

12. TABLE_CONSTRAINTS表：描述了存在约束的表。以及表的约束类型。

13. KEY_COLUMN_USAGE表：描述了具有约束的键列。

14. ROUTINES表：提供了关于存储子程序（存储程序和函数）的信息。此时，ROUTINES表不包含自定义函数（UDF）。名为“mysql.proc name”的列指明了对应于　　　　　　　INFORMATION_SCHEMA.ROUTINES表的mysql.proc表列。

15. VIEWS表：给出了关于数据库中的视图的信息。需要有show views权限，否则无法查看视图信息。

16. TRIGGERS表：提供了关于触发程序的信息。必须有super权限才能查看该表。

# mysql



# performance_schema

 需要设置参数： performance_schema 才可以启动该功能

按照相关的标准对进行的事件统计表, 表也是只读的，只能turcate

　　events_waits_summary_by_instance             

　　events_waits_summary_by_thread_by_event_name 

　　events_waits_summary_global_by_event_name    

　　file_summary_by_event_name                   

　　file_summary_by_instance   

- setup_consumers 描述各种事件

- setup_instruments 描述这个数据库下的表名以及是否开启监控。

- setup_timers   描述 监控选项已经采样频率的时间间隔

- events_waits_current  记录当前正在发生的等待事件，这个表是只读的表，不能update ，delete ，但是可以truncate

- 性能历史表 ：events_waits_history  只保留每个线程（thread） 的最近的10个事件

- 性能历史表 ：events_waits_history_long 记录最近的10000个事件  标准的先进先出（FIFO) 这俩表也是只读表，只能truncate

# sakila

　　这是一个MySQL的一个样本数据库，里边都是一些例子表。

