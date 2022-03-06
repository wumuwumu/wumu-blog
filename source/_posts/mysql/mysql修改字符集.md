---
title: mysql修改字符集
tags:
  - mysql
abbrlink: 4e5c95b3
date: 2019-07-27 16:55:17
---

# 概念

1. 字符集（character set）：定义了字符以及字符的编码。
2. 字符序（collation）：定义了字符的比较规则。

# Mysql字符集

1. 一个字符集对应至少一种字符序（一般是1对多）。
2. 两个不同的字符集不能有相同的字符序。
3. 每个字符集都有默认的字符序。

```mysql
-- 第一种方式
SHOW CHARACTER SET;

-- 第二种方式
use information_schema;
select * from CHARACTER_SETS;

-- 例子
SHOW CHARACTER SET WHERE Charset="utf8";
SHOW CHARACTER SET LIKE "utf8%";
```

# Mysql字符序

```mysql
 -- 第一种方式
 SHOW COLLATION WHERE Charset = 'utf8';
 
 -- 第二种方式
 USE information_schema;
 SELECT * FROM COLLATIONS WHERE CHARACTER_SET_NAME="utf8";
```

## 命名规范

字符序的命名，以其对应的字符集作为前缀，如下所示。比如字符序`utf8_general_ci`，标明它是字符集`utf8`的字符序。

更多规则可以参考 [官方文档](https://dev.mysql.com/doc/refman/5.7/en/charset-collation-names.html)。

```mysql
[information_schema]> SELECT CHARACTER_SET_NAME, COLLATION_NAME FROM COLLATIONS WHERE CHARACTER_SET_NAME="utf8" limit 2; 
```

# 设置修改

1. 修改数据库字符集

   ```mysql
   ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];
   把表默认的字符集和所有字符列（CHAR,VARCHAR,TEXT）改为新的字符集：
   ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]
   如：ALTER TABLE logtest CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci;
   ```

2. 修改表的默认字符集

   ```mysql
   ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...];
   如：ALTER TABLE logtest DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
   ```

3. 修改字段的字符集

   ```mysql
   ALTER TABLE tbl_name CHANGE c_name c_name CHARACTER SET character_name [COLLATE ...];
   如：ALTER TABLE logtest CHANGE title title VARCHAR(100) CHARACTER SET utf8 COLLATE utf8_general_ci;
   ```

4. 查看数据库编码

   ```mysql
   SHOW CREATE DATABASE db_name;
   ```

5. 查看表编码

   ```mysql
   SHOW CREATE TABLE tbl_name;
   ```

6. 查看字段编码

   ```mysql
   SHOW FULL COLUMNS FROM tbl_name;
   ```

7. 查看系统的编码字符

   ```mysql
   SHOW VARIABLES WHERE Variable_name LIKE 'character\_set\_%' OR Variable_name LIKE 'collation%';
   ```

8. MySQL字符集设置

   系统变量：

   ```sh
   – character_set_server：默认的内部操作字符集
   
   – character_set_client：客户端来源数据使用的字符集
   
   – character_set_connection：连接层字符集
   
   – character_set_results：查询结果字符集
   
   – character_set_database：当前选中数据库的默认字符集
   
   – character_set_system：系统元数据(字段名等)字符集
   
   – 还有以collation_开头的同上面对应的变量，用来描述字符序。
   ```

   用introducer指定文本字符串的字符集：

   – 格式为：[_charset] ‘string’ [COLLATE collation]

   – 例如：

   ```sql
   • SELECT _latin1 ‘string’;
   
   • SELECT _utf8 ‘你好’ COLLATE utf8_general_ci;
   
   –-  由introducer修饰的文本字符串在请求过程中不经过多余的转码，直接转换为内部字符集处理。
   ```

   #### MySQL中的字符集转换过程

   1. MySQL Server收到请求时将请求数据从character_set_client转换为character_set_connection；
   2. 进行内部操作前将请求数据从character_set_connection转换为内部操作字符集，其确定方法如下：

   • 使用每个数据字段的CHARACTER SET设定值；

   • 若上述值不存在，则使用对应数据表的DEFAULT CHARACTER SET设定值(MySQL扩展，非SQL标准)；

   • 若上述值不存在，则使用对应数据库的DEFAULT CHARACTER SET设定值；

   • 若上述值不存在，则使用character_set_server设定值。

# 参考

> https://www.cnblogs.com/chyingp/p/mysql-character-set-collation.html
>
> https://www.cnblogs.com/qiumingcheng/p/10336170.html

