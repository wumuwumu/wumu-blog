---
title: postgresql账号管理
date: 2020-09-16 18:37:46
tags:
- linux
---

**注意：**创建好用户（角色）之后需要连接的话，还需要修改2个权限控制的配置文件（pg_hba.conf、pg_ident.conf）。并且创建用户（user）和创建角色（role）一样，唯一的区别是用户默认可以登录，而创建的角色默认不能登录。创建用户和角色的各个参数选项是一样的。

**Tip：安装PostgreSQL会自动创建一个postgres用户，需要切换到该用户下访问PostgreSQL。**

- [创建用户/角色](https://www.postgresql.org/docs/11/sql-createuser.html)

  ```
  CREATE USER/ROLE name [ [ WITH ] option [ ... ] ]  : 关键词 USER,ROLE； name 用户或角色名； 
  
  where option can be:
  
        SUPERUSER | NOSUPERUSER      :超级权限，拥有所有权限，默认nosuperuser。
      | CREATEDB | NOCREATEDB        :建库权限，默认nocreatedb。
      | CREATEROLE | NOCREATEROLE    :建角色权限，拥有创建、修改、删除角色，默认nocreaterole。
      | INHERIT | NOINHERIT          :继承权限，可以把除superuser权限继承给其他用户/角色，默认inherit。
      | LOGIN | NOLOGIN              :登录权限，作为连接的用户，默认nologin，除非是create user（默认登录）。
      | REPLICATION | NOREPLICATION  :复制权限，用于物理或则逻辑复制（复制和删除slots），默认是noreplication。
      | BYPASSRLS | NOBYPASSRLS      :安全策略RLS权限，默认nobypassrls。
  ```

  ```
      | CONNECTION LIMIT connlimit   :限制用户并发数，默认-1，不限制。正常连接会受限制，后台连接和prepared事务不受限制。
      | [ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL :设置密码，密码仅用于有login属性的用户，不使用密码身份验证，则可以省略此选项。可以选择将空密码显式写为PASSWORD NULL。
                                                           加密方法由配置参数password_encryption确定，密码始终以加密方式存储在系统目录中。
      | VALID UNTIL 'timestamp'      :密码有效期时间，不设置则用不失效。
      | IN ROLE role_name [, ...]    :新角色将立即添加为新成员。
      | IN GROUP role_name [, ...]   :同上
      | ROLE role_name [, ...]       :ROLE子句列出一个或多个现有角色，这些角色自动添加为新角色的成员。 （这实际上使新角色成为“组”）。
      | ADMIN role_name [, ...]      :与ROLE类似，但命名角色将添加到新角色WITH ADMIN OPTION，使他们有权将此角色的成员资格授予其他人。
      | USER role_name [, ...]       :同上
      | SYSID uid                    :被忽略，但是为向后兼容性而存在。
  ```



​      **示例：**

1. 创建不需要密码登陆的用户zjy：

   ```
   postgres=# CREATE ROLE zjy LOGIN;
   CREATE ROLE
   ```

   创建该用户后，还不能直接登录。需要修改 **pg_hba.conf** 文件（后面会对该文件进行说明），加入：

   ①：本地登陆：local   all    all    **trust**②：远程登陆：host   all    all    192.168.163.132/32     **trust**

2. 创建需要密码登陆的用户zjy1：

   ```
   postgres=# CREATE USER zjy1 WITH PASSWORD 'zjy1';
   CREATE ROLE
   ```

   和ROLE的区别是：USER带LOGIN属性。也需要修改 **pg_hba.conf** 文件（后面会对该文件进行说明），加入：
   host    all     all     192.168.163.132/32    **md5**

3. 创建有时间限制的用户zjy2：

   ```
   postgres=# CREATE ROLE zjy2 WITH LOGIN PASSWORD 'zjy2' VALID UNTIL '2019-05-30';
   CREATE ROLE
   ```

   和2的处理方法一样，修改 **pg_hba.conf** 文件，该用户会的密码在给定的时间之后过期不可用。

4. 创建有创建数据库和管理角色权限的用户admin：

   ```
   postgres=# CREATE ROLE admin WITH CREATEDB CREATEROLE;
   CREATE ROLE
   ```

   注意：拥有创建数据库，角色的用户，也可以删除和修改这些对象。

5. 创建具有超级权限的用户：admin

   ```
   postgres=# CREATE ROLE admin WITH SUPERUSER LOGIN PASSWORD 'admin';
   CREATE ROLE
   ```

6. 创建复制账号：repl 

   ```
   postgres=# CREATE USER repl REPLICATION LOGIN ENCRYPTED PASSWORD 'repl';
   CREATE ROLE
   ```

7. 其他说明



8. 

- [授权，定义访问权限](https://www.postgresql.org/docs/11/sql-grant.html)



  ```
  GRANT { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
      [, ...] | ALL [ PRIVILEGES ] }
      ON { [ TABLE ] table_name [, ...]
           | ALL TABLES IN SCHEMA schema_name [, ...] }
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  ##单表授权：授权zjy账号可以访问schema为zjy的zjy表
  grant select,insert,update,delete on zjy.zjy to zjy;
  ##所有表授权：
  grant select,insert,update,delete on all tables in schema zjy to zjy;
  
  
  GRANT { { SELECT | INSERT | UPDATE | REFERENCES } ( column_name [, ...] )
      [, ...] | ALL [ PRIVILEGES ] ( column_name [, ...] ) }
      ON [ TABLE ] table_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  ##列授权，授权指定列(zjy schema下的zjy表的name列)的更新权限给zjy用户
  grant update (name) on zjy.zjy to zjy;
  ##指定列授不同权限，zjy schema下的zjy表，查看更新name、age字段，插入name字段
  grant select (name,age),update (name,age),insert(name) on zjy.xxx to zjy;
  
  
  GRANT { { USAGE | SELECT | UPDATE }
      [, ...] | ALL [ PRIVILEGES ] }
      ON { SEQUENCE sequence_name [, ...]
           | ALL SEQUENCES IN SCHEMA schema_name [, ...] }
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  ##序列（自增键）属性授权，指定zjy schema下的seq_id_seq 给zjy用户
  grant select,update on sequence zjy.seq_id_seq to zjy;
  ##序列（自增键）属性授权，给用户zjy授权zjy schema下的所有序列
  grant select,update on all sequences in schema zjy to zjy;
  
  
  GRANT { { CREATE | CONNECT | TEMPORARY | TEMP } [, ...] | ALL [ PRIVILEGES ] }
      ON DATABASE database_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  ##连接数据库权限，授权cc用户连接数据库zjy
  grant connect on database zjy to cc;
  
  
  GRANT { USAGE | ALL [ PRIVILEGES ] }
      ON DOMAIN domain_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  ##
  ```

  ```
  GRANT { USAGE | ALL [ PRIVILEGES ] }
      ON FOREIGN DATA WRAPPER fdw_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  ```

    \##

  ```
  GRANT { USAGE | ALL [ PRIVILEGES ] }
      ON FOREIGN SERVER server_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  ```

  ```
  ##
  ```

  ```
  GRANT { EXECUTE | ALL [ PRIVILEGES ] }
      ON { { FUNCTION | PROCEDURE | ROUTINE } routine_name [ ( [ [ argmode ] [ arg_name ] arg_type [, ...] ] ) ] [, ...]
           | ALL { FUNCTIONS | PROCEDURES | ROUTINES } IN SCHEMA schema_name [, ...] }
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  ```

  ```
  ##
  
  
  GRANT { USAGE | ALL [ PRIVILEGES ] }
      ON LANGUAGE lang_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  ```

    \##

  ```
  GRANT { { SELECT | UPDATE } [, ...] | ALL [ PRIVILEGES ] }
      ON LARGE OBJECT loid [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  ```

  ```
  ##
  
  GRANT { { CREATE | USAGE } [, ...] | ALL [ PRIVILEGES ] }
      ON SCHEMA schema_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  ##连接schema权限，授权cc访问zjy schema权限
  grant usage on schema zjy to cc;
  
  GRANT { CREATE | ALL [ PRIVILEGES ] }
      ON TABLESPACE tablespace_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  GRANT { USAGE | ALL [ PRIVILEGES ] }
      ON TYPE type_name [, ...]
      TO role_specification [, ...] [ WITH GRANT OPTION ]
  
  where role_specification can be:
  
      [ GROUP ] role_name
    | PUBLIC
    | CURRENT_USER
    | SESSION_USER
  
  GRANT role_name [, ...] TO role_name [, ...] [ WITH ADMIN OPTION ]
  ##把zjy用户的权限授予用户cc。
  grant zjy to cc;
  ```



  [权限说明](https://blog.51cto.com/riverxyz/1880795)：



  ```
  SELECT：允许从指定表，视图或序列的任何列或列出的特定列进行SELECT。也允许使用COPY TO。在UPDATE或DELETE中引用现有列值也需要此权限。对于序列，此权限还允许使用currval函数。对于大对象，此权限允许读取对象。
  
  INSERT：允许将新行INSERT到指定的表中。如果列出了特定列，则只能在INSERT命令中为这些列分配（因此其他列将接收默认值）。也允许COPY FROM。
  
  UPDATE：允许更新指定表的任何列或列出的特定列，需要SELECT权限。
  
  DELETE：允许删除指定表中的行，需要SELECT权限。
  
  TRUNCATE：允许在指定的表上创建触发器。
  
  REFERENCES：允许创建引用指定表或表的指定列的外键约束。
  
  TRIGGER：允许在指定的表上创建触发器。 
  
  CREATE：对于数据库，允许在数据库中创建新的schema、table、index。
  
  CONNECT：允许用户连接到指定的数据库。在连接启动时检查此权限。
  
  TEMPORARY、TEMP：允许在使用指定数据库时创建临时表。
  
  EXECUTE：允许使用指定的函数或过程以及在函数。
  
  USAGE：对于schema，允许访问指定模式中包含的对象；对于sequence，允许使用currval和nextval函数。对于类型和域，允许在创建表，函数和其他模式对象时使用类型或域。
  
  ALL PRIVILEGES：一次授予所有可用权限。
  ```



- [撤销权限
  ](https://www.postgresql.org/docs/11/sql-revoke.html)



  ```
  REVOKE [ GRANT OPTION FOR ]
      { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
      [, ...] | ALL [ PRIVILEGES ] }
      ON { [ TABLE ] table_name [, ...]
           | ALL TABLES IN SCHEMA schema_name [, ...] }
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  
   ##移除用户zjy在schema zjy上所有表的select权限
   revoke select on all tables in schema zjy from zjy;
  
  
  REVOKE [ GRANT OPTION FOR ]
      { { SELECT | INSERT | UPDATE | REFERENCES } ( column_name [, ...] )
      [, ...] | ALL [ PRIVILEGES ] ( column_name [, ...] ) }
      ON [ TABLE ] table_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  
   ##移除用户zjy在zjy schema的zjy表的age列的查询权限
   revoke select (age) on zjy.zjy from zjy;
  
  
  REVOKE [ GRANT OPTION FOR ]
      { { USAGE | SELECT | UPDATE }
      [, ...] | ALL [ PRIVILEGES ] }
      ON { SEQUENCE sequence_name [, ...]
           | ALL SEQUENCES IN SCHEMA schema_name [, ...] }
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##序列
  
  
  REVOKE [ GRANT OPTION FOR ]
      { { CREATE | CONNECT | TEMPORARY | TEMP } [, ...] | ALL [ PRIVILEGES ] }
      ON DATABASE database_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##库
  
  
  REVOKE [ GRANT OPTION FOR ]
      { USAGE | ALL [ PRIVILEGES ] }
      ON DOMAIN domain_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT]
  ##
  
  
  REVOKE [ GRANT OPTION FOR ]
      { USAGE | ALL [ PRIVILEGES ] }
      ON FOREIGN DATA WRAPPER fdw_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT]
  ##
  
  REVOKE [ GRANT OPTION FOR ]
      { USAGE | ALL [ PRIVILEGES ] }
      ON FOREIGN SERVER server_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT]
  ##
  
  
  REVOKE [ GRANT OPTION FOR ]
      { EXECUTE | ALL [ PRIVILEGES ] }
      ON { { FUNCTION | PROCEDURE | ROUTINE } function_name [ ( [ [ argmode ] [ arg_name ] arg_type [, ...] ] ) ] [, ...]
           | ALL { FUNCTIONS | PROCEDURES | ROUTINES } IN SCHEMA schema_name [, ...] }
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##
  ```

  ```
  REVOKE [ GRANT OPTION FOR ]
      { USAGE | ALL [ PRIVILEGES ] }
      ON LANGUAGE lang_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##
  
  
  REVOKE [ GRANT OPTION FOR ]
      { { SELECT | UPDATE } [, ...] | ALL [ PRIVILEGES ] }
      ON LARGE OBJECT loid [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##
  
  
  REVOKE [ GRANT OPTION FOR ]
      { { CREATE | USAGE } [, ...] | ALL [ PRIVILEGES ] }
      ON SCHEMA schema_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##schena权限
  
  
  REVOKE [ GRANT OPTION FOR ]
      { CREATE | ALL [ PRIVILEGES ] }
      ON TABLESPACE tablespace_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##
  
  
  REVOKE [ GRANT OPTION FOR ]
      { USAGE | ALL [ PRIVILEGES ] }
      ON TYPE type_name [, ...]
      FROM { [ GROUP ] role_name | PUBLIC } [, ...]
      [ CASCADE | RESTRICT ]
  ##
  ```

  ```
  REVOKE [ ADMIN OPTION FOR ]
      role_name [, ...] FROM role_name [, ...]
      [ CASCADE | RESTRICT ]
  ##
  ```



  注意：任何用户对public的schema都有all的权限，为了安全可以禁止用户对public schema

  ```
  ##移除所有用户（public），superuser除外，对指定DB下的public schema的create 权限。
  zjy=# revoke  create  on schema public from public;
  REVOKE
  ```

- [修改用户属性
  ](https://www.postgresql.org/docs/11/sql-alteruser.html)


  ```
  ALTER USER role_specification [ WITH ] option [ ... ]
  
  where option can be:
  
        SUPERUSER | NOSUPERUSER
      | CREATEDB | NOCREATEDB
      | CREATEROLE | NOCREATEROLE
      | INHERIT | NOINHERIT
      | LOGIN | NOLOGIN
      | REPLICATION | NOREPLICATION
      | BYPASSRLS | NOBYPASSRLS
      | CONNECTION LIMIT connlimit
      | [ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL
      | VALID UNTIL 'timestamp'
  
  ALTER USER name RENAME TO new_name
  
  ALTER USER { role_specification | ALL } [ IN DATABASE database_name ] SET configuration_parameter { TO | = } { value | DEFAULT }
  ALTER USER { role_specification | ALL } [ IN DATABASE database_name ] SET configuration_parameter FROM CURRENT
  ALTER USER { role_specification | ALL } [ IN DATABASE database_name ] RESET configuration_parameter
  ALTER USER { role_specification | ALL } [ IN DATABASE database_name ] RESET ALL
  
  where role_specification can be:
  
      role_name
    | CURRENT_USER
    | SESSION_USER
  ```



  **示例：**     注意：option选项里的用户都可以通过alter role进行修改

- - 修改用户为超级/非超级用户

    ```
    alter role caocao with superuser/nosuperuser;
    ```

  - 修改用户为可/不可登陆用户

    ```
    alter role caocao with nologin/login;
    ```

  - 修改用户名：

    ```
    alter role caocao rename to youxing;
    ```

  - 修改用户密码，移除密码用NULL

    ```
    alter role youxing with password 'youxing';
    ```

  - 修改用户参数，该用户登陆后的以该参数为准

    ```
    alter role zjy in database zjy SET geqo to 0/default;
    ```

- [控制访问文件](https://www.postgresql.org/docs/11/auth-pg-hba-conf.html) pg_hba.conf[
  ](https://www.postgresql.org/docs/11/auth-pg-hba-conf.html)



  ```
  local      database  user  auth-method  [auth-options]
  host       database  user  address  auth-method  [auth-options]
  hostssl    database  user  address  auth-method  [auth-options]
  hostnossl  database  user  address  auth-method  [auth-options]
  host       database  user  IP-address  IP-mask  auth-method  [auth-options]
  hostssl    database  user  IP-address  IP-mask  auth-method  [auth-options]
  hostnossl  database  user  IP-address  IP-mask  auth-method  [auth-options]
  ```



  **local**：匹配使用Unix域套接字的连接，如果没有此类型的记录，则不允许使用Unix域套接字连接。
  **host**：匹配使用TCP/IP进行的连接，主机记录匹配SSL或非SSL连接，需要配置listen_addresses。
  **hostssl**：匹配使用TCP/IP进行的连接，仅限于使用SSL加密进行连接，需要配置ssl参数。
  **hostnossl**：匹配通过TCP/IP进行的连接，不使用SSL的连接。
  **database**：匹配的数据库名称，all指定它匹配所有数据库。如果请求的数据库与请求的用户具有相同的名称则可以使用samerole值。复制（replication）不指定数据库，多个数据库可以用逗号分隔。
  **user**：匹配的数据库用户名，值all指定它匹配所有用户。 可以通过用逗号分隔来提供多个用户名。
  **address**：匹配的客户端计算机地址，可以包含主机名，IP地址范围。如：172.20.143.89/32、172.20.143.0/24、10.6.0.0/16、:: 1/128。 0.0.0.0/0表示所有IPv4地址，:: 0/0表示所有IPv6地址。要指定单个主机，请使用掩码长度32（对于IPv4）或128（对于IPv6）。all以匹配任何IP地址。
  **IP-address、IP-mask**：这两个字段可用作IP地址/掩码长度，如：127.0.0.1 255.255.255.255。
  **auth-method**：指定连接与此记录匹配时要使用的身份验证方法：trust、reject、scram-sha-256、md5、password、gss、sspi、ident、peer、ldap、radius、cert、pam、bsd。



  ```
  trust：允许无条件连接，允许任何PostgreSQL用户身份登录，而无需密码或任何其他身份验证。
  reject：拒绝任何条件连接，这对于从组中“过滤掉”某些主机非常有用。
  scram-sha-256：执行SCRAM-SHA-256身份验证以验证用户的密码。
  md5：执行SCRAM-SHA-256或MD5身份验证以验证用户的密码。
  password：要提供未加密的密码以进行身份验证。由于密码是通过网络以明文形式发送的，因此不应在不受信任的网络上使用。
  gss：使用GSSAPI对用户进行身份验证，这仅适用于TCP / IP连接。
  sspi：使用SSPI对用户进行身份验证，这仅适用于Windows。
  ident：通过联系客户端上的ident服务器获取客户端的操作系统用户名，并检查它是否与请求的数据库用户名匹配。 Ident身份验证只能用于TCP / IP连接。为本地连接指定时，将使用对等身份验证。
  peer：从操作系统获取客户端的操作系统用户名，并检查它是否与请求的数据库用户名匹配。这仅适用于本地连接。
  ldap：使用LDAP服务器进行身份验证。
  radius：使用RADIUS服务器进行身份验证。
  cert：使用SSL客户端证书进行身份验证。
  pam：使用操作系统提供的可插入身份验证模块（PAM）服务进行身份验证。
  bsd：使用操作系统提供的BSD身份验证服务进行身份验证。
  ```



  **auth-options**：在auth-method字段之后，可以存在name = value形式的字段，用于指定认证方法的选项。
  例子：



  ```
  # TYPE  DATABASE    USER   ADDRESS   METHOD
  local          all               all                         trust
  --在本地允许任何用户无密码登录
  local          all                all                        peer
  --操作系统的登录用户和pg的用户是否一致，一致则可以登录
  local          all                all                        ident
  --操作系统的登录用户和pg的用户是否一致，一致则可以登录
  host          all                all    192.168.163.0/24   md5
  --指定客户端IP访问通过md5身份验证进行登录
  host          all                all     192.168.163.132/32   password
  --指定客户端IP通过passwotd身份验证进行登录
  
  host    all             all     192.168.54.1/32         reject
  host    all             all     192.168.0.0/16           ident  
  host    all             all     127.0.0.1       255.255.255.255     trust
  ...
  ```



  设置完之后可以通过查看表来查看hba：



  ```
  zjy=# select * from pg_hba_file_rules;
   line_number | type  |   database    | user_name |    address    |                 netmask                 | auth_method | options | error 
  -------------+-------+---------------+-----------+---------------+-----------------------------------------+-------------+---------+-------
            87 | host  | {all}         | {all}     | 192.168.163.0 | 255.255.255.0                           | md5         |         | 
            92 | local | {all}         | {all}     |               |                                         | peer        |         | 
            94 | host  | {all}         | {all}     | 127.0.0.1     | 255.255.255.255                         | md5         |         | 
            96 | host  | {all}         | {all}     | ::1           | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | md5         |         | 
            99 | local | {replication} | {all}     |               |                                         | peer        |         | 
           100 | host  | {replication} | {all}     | 127.0.0.1     | 255.255.255.255                         | md5         |         | 
           101 | host  | {replication} | {all}     | ::1           | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | md5         |         | 
  ```



  当然，修改完pg_hba.conf文件之后，需要重新加载配置，不用重启数据库：

  ```
  postgres=# select pg_reload_conf();
   pg_reload_conf 
  ----------------
   t
  ```

- ### 日常使用

用户权限管理涉及到的东西很多，本文也只是大致说明了一小部分，大部分的还得继续学习。那么现在按照一个正常项目上线的流程来创建一个应用账号为例，看看需要怎么操作。

比如一个项目**zjy**上线：用管理账号来操作

- 创建数据库：

  ```
  postgres=# create database zjy;
  CREATE DATABASE
  ```

- 创建账号：账号和数据库名字保持一致（search_path）

  ```
  postgres=# create user zjy with password 'zjy';
  CREATE ROLE
  ```

- 创建schema：不能用默认的public的schma

  ```
  postgres=# \c zjy
  You are now connected to database "zjy" as user "postgres".
  zjy=# create schema zjy;
  CREATE SCHEMA
  ```

- 授权：

  [![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

  ```
  #访问库
  zjy=# grant connect on database zjy to zjy;
  GRANT
  #访问schmea
  zjy=# grant usage on schema zjy to zjy;
  GRANT
  #访问表
  zjy=# grant select,insert,update,delete on all tables in schema zjy to zjy;
  GRANT
  #如果访问自增序列，需要授权
  zjy=# grant select,update on all sequences in schema zjy to zjy;
  GRANT
  
  注意：上面的授权只对历史的一些对象授权，后期增加的对象是没有权限的，需要给个默认权限
  
  #默认表权限
  zjy=# ALTER DEFAULT PRIVILEGES IN SCHEMA zjy GRANT select,insert,update,delete ON TABLES TO zjy;
  ALTER DEFAULT PRIVILEGES
  
  #默认自增序列权限
  zjy=# ALTER DEFAULT PRIVILEGES IN SCHEMA zjy GRANT select,update ON sequences TO zjy;
  ALTER DEFAULT PRIVILEGES
  ```

- ### 常用命令

1. 查看当前用户javascript:void(0);)

   ```
   zjy=# \du
                                      List of roles
    Role name |                         Attributes                         | Member of 
   -----------+------------------------------------------------------------+-----------
    admin     | Superuser, Cannot login                                    | {}
    postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
    zjy       |                                                            | {}
   
   zjy=# select * from pg_roles;
          rolname        | rolsuper | rolinherit | rolcreaterole | rolcreatedb | rolcanlogin | rolreplication | rolconnlimit | rolpassword | rolvaliduntil | rolbypassrls | rolconfig |  oid  
   ----------------------+----------+------------+---------------+-------------+-------------+----------------+--------------+-------------+---------------+--------------+-----------+-------
    pg_signal_backend    | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  4200
    postgres             | t        | t          | t             | t           | t           | t              |           -1 | ********    |               | t            |           |    10
    admin                | t        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           | 16456
    pg_read_all_stats    | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3375
    zjy                  | f        | t          | f             | f           | t           | f              |           -1 | ********    |               | f            |           | 16729
    pg_monitor           | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3373
    pg_read_all_settings | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3374
    pg_stat_scan_tables  | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3377
   (8 rows)
   ```

2. 查看用户权限javascript:void(0);)

   ```
   zjy=# select * from information_schema.table_privileges where grantee='zjy';
    grantor  | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy 
   ----------+---------+---------------+--------------+------------+----------------+--------------+----------------
    postgres | zjy     | zjy           | zjy          | zjy        | INSERT         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy        | SELECT         | NO           | YES
    postgres | zjy     | zjy           | zjy          | zjy        | UPDATE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy        | DELETE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy1       | INSERT         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy1       | SELECT         | NO           | YES
    postgres | zjy     | zjy           | zjy          | zjy1       | UPDATE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy1       | DELETE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy2       | INSERT         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy2       | SELECT         | NO           | YES
    postgres | zjy     | zjy           | zjy          | zjy2       | UPDATE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy2       | DELETE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy3       | INSERT         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy3       | SELECT         | NO           | YES
    postgres | zjy     | zjy           | zjy          | zjy3       | UPDATE         | NO           | NO
    postgres | zjy     | zjy           | zjy          | zjy3       | DELETE         | NO           | NO
   ```


