---
title: mysql权限管理
date: 2019-03-29 16:55:22
tags:
- linux
---

# 用户管理

# 基本操作

```sql
create user zhangsan identified by 'zhangsan';

SELECT current_user();                                        ← 查看当前用户
SELECT user,host FROM mysql.user;                    ← 查看用户信息
SHOW GRANTS;                                                  ← 当前用户权限，会生成SQL语句
CREATE USER 'user'@'host' IDENTIFIED BY 'password';           ← 创建用户
DROP USER 'user'@'host';                                      ← 删除用户
RENAME USER 'user'@'host' TO 'fool'@'host';        
```

## 修改密码

```sql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new-password';   ← 修改密码(recommand)
mysql> SET PASSWORD FOR 'root'@'localhost'=PASSWORD('new-password'); ← 修改密码
mysql> UPDATE mysql.user SET password=PASSWORD('new-password')
       WHERE USER='root' AND Host='127.0.0.1';
mysql> UPDATE mysql.user SET password='' WHERE user='root';          ← 清除密码
mysql> FLUSH PRIVILEGES;
$ mysqladmin -uROOT -pOLD_PASSWD password NEW_PASSWD                 ← 通过mysqladmin修改
$ mysqladmin -uROOT -p flush-privileges
```

## 权限管理

```sql
mysql> GRANT ALL ON *.* TO 'user'@'%' [IDENTIFIED BY 'password'];
mysql> GRANT ALL  ON [TABLE | DATABASE] student,course TO user1,user2;
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, CREATE TEMPORARY, ALTER,
       DROP, REFERENCES, INDEX, CREATE VIEW, SHOW VIEW, CREATE ROUTINE,
       ALTER ROUTINE, EXECUTE
       ON db.tbl TO 'user'@'host' [IDENTIFIED BY 'password'];
mysql> GRANT ALL ON sampdb.* TO PUBLIC WITH GRANT OPTION;            ← 所有人，可以授权给其他人
mysql> GRANT UPDATE(col),SELECT ON TABLE tbl TO user;                ← 针对列赋值
mysql> SHOW GRANTS [FOR 'user'@'host'];                              ← 查看权限信息
mysql> REVOKE ALL ON *.* FROM 'user'@'host';                         ← 撤销权限
mysql> REVOKE SELECT(user, host), UPDATE(host) ON db.tbl FROM 'user'@'%';

```

# 权限

##  admin

```
mysql> CREATE USER 'admin'@'IP' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'admin'@'IP';

mysql> REVOKE ALL PRIVILEGES ON *.* FROM 'admin'@'IP';
mysql> DROP USER 'admin'@'IP';
```

## root

```sql
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
```

# 其他

## 重置root密码

```sql
----- 1. 停止mysql服务器
# systemctl stop mysqld
# /opt/mysql-5.7/bin/mysqladmin -uroot -p'init-password' shutdown
Shutting down MySQL..     done

----- 2. 获取跳过认证的启动参数
# mysqld --help --verbose | grep 'skip-grant-tables' -A1
    --skip-grant-tables Start without grant tables. This gives all users FULL
                          ACCESS to all tables.

----- 3. 启动服务器，跳过认证
# mysqld --skip-grant-tables --user=mysql &
[1] 10209

----- 4. 取消密码
mysql> UPDATE mysql.user SET password='' WHERE user='root';
Query OK, 2 rows affected (0.00 sec)
Rows matched: 2  Changed: 2  Warnings: 0
```

## 密码策略

### 参数解释

validate_password_dictionary_file
插件用于验证密码强度的字典文件路径。

validate_password_length
密码最小长度，参数默认为8，它有最小值的限制，最小值为：validate_password_number_count + validate_password_special_char_count + (2 * validate_password_mixed_case_count)

validate_password_mixed_case_count
密码至少要包含的小写字母个数和大写字母个数。

validate_password_number_count
密码至少要包含的数字个数。

validate_password_policy
密码强度检查等级，0/LOW、1/MEDIUM、2/STRONG。有以下取值：
Policy                 Tests Performed                                                                                                        
0 or LOW               Length                                                                                                                      
1 or MEDIUM         Length; numeric, lowercase/uppercase, and special characters                             
2 or STRONG        Length; numeric, lowercase/uppercase, and special characters; dictionary file      
默认是1，即MEDIUM，所以刚开始设置的密码必须符合长度，且必须含有数字，小写或大写字母，特殊字符。

validate_password_special_char_count
密码至少要包含的特殊字符数。

### 修改mysql参数配置

```mysql
mysql> set global validate_password_policy=0;
Query OK, 0 rows affected (0.05 sec)

mysql> set global validate_password_mixed_case_count=0;
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global validate_password_number_count=3;
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global validate_password_special_char_count=0;
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global validate_password_length=3;
Query OK, 0 rows affected (0.00 sec)
 
mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| validate_password_dictionary_file    |       |
| validate_password_length             | 3     |
| validate_password_mixed_case_count   | 0     |
| validate_password_number_count       | 3     |
| validate_password_policy             | LOW   |
| validate_password_special_char_count | 0     |
+--------------------------------------+-------+
6 rows in set (0.00 sec)
```



## MySQL 中 localhost 127.0.0.1 区别

`%` 是一个通配符，用以匹配所有的 IP 地址，但是不能匹配到 `locahost` 这个特殊的域名。

也就是说，如果要允许本地登录，单纯只配置一个 `%` 是不够的 (应该是说对这种方式是不够的)，需要同时配置一个 `locahost` 的账号。

```sql
mysql> GRANT ALL ON *.* TO 'foobar'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.01 sec)
mysql> SELECT user, host, password FROM mysql.user WHERE user like 'foobar%';
+--------+------+-------------------------------------------+
| user   | host | password                                  |
+--------+------+-------------------------------------------+
| foobar | %    | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
+--------+------+-------------------------------------------+
1 row in set (0.00 sec)

$ mysql -ufoobar -h127.0.0.1 -P3307 -p'123456'
ERROR 1045 (28000): Access denied for user 'foobar'@'localhost' (using password: YES)
```

https://jin-yang.github.io/post/mysql-localhost-vs-127.0.0.1-introduce.html

# 参考

https://jin-yang.github.io/post/mysql-users.html

https://www.cnblogs.com/Richardzhu/p/3318595.html