---
title: redis基本操作
abbrlink: 2a3c893b
date: 2021-01-29 11:00:00
---

[Redis 基本(basic)命令](https://blog.csdn.net/wangmx1993328/article/details/90339509#t0)

[Redis 键(key)命令](https://blog.csdn.net/wangmx1993328/article/details/90339509#t1)

[Redis 数据类型概述](https://blog.csdn.net/wangmx1993328/article/details/90339509#t2)

[Redis 字符串(String)](https://blog.csdn.net/wangmx1993328/article/details/90339509#t3)

[Redis 哈希（Hash）](https://blog.csdn.net/wangmx1993328/article/details/90339509#t4)

[Redis 列表(List)](https://blog.csdn.net/wangmx1993328/article/details/90339509#t5)

[Redis 集合(Set)](https://blog.csdn.net/wangmx1993328/article/details/90339509#t6)

[Redis 有序集合(sorted set)](https://blog.csdn.net/wangmx1993328/article/details/90339509#t7)

------

# Redis 基本(basic)命令

1、Redis 命令用于在 redis 服务上执行操作，要在 redis 服务上执行命令需要一个 redis 客户端。安装目录下的 redis-cli 就是自带的测试客户端。

命令行启动自带的 redis-cli 客户端连接到本地的 redis 服务：redis-cli连接远程 redis 服务器：redis-cli -h host -p port -a password

| **PING**      | 用于检测 redis 服务是否启动,连接是否正常，连接成功时返回 PONG |
| ------------- | ------------------------------------------------------------ |
| select index  | Redis Select 命令用于切换到指定的数据库，数据库索引号 index 用数字值指定，以 0 作为起始索引值。 |
| exit          | 退出 redis-cli                                               |
| auth password | 当 redis 服务器开启密码验证，客户端连接时没有指定密码时，连接后必须使用 "auth 密码" 先进行授权，否则其它命令会使用不了。 |
| set key value | 往 redis 数据库设置数据                                      |
| get key       | 从 redis 数据库读取数据。key 不存在时，返回 nil              |
| keys *        | 查询 redis 数据库中的所有 key 值                             |
| del key       | 删除指定的 key 的内容                                        |

![img](https://img-blog.csdnimg.cn/20190519102237597.gif)

# Redis 键(key)命令

1、Redis 键命令用于管理 redis 的键。

2、Redis 键命令的基本语法：command KEY_NAME

| 序号 | 命令                                                         | 描述                                                         |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | [del key](https://www.redis.net.cn/order/3528.html)          | 删除指定的 key。key 不存在时不影响。可以同时删除多个，如 del key1 key2 ...。list、set、zset、hash 中的元素全部删除后，key 也会自动被删除。 |
| 2    | [dump key](https://www.redis.net.cn/order/3529.html)         | 序列化给定 key ，并返回被序列化的值。                        |
| 3    | [exists key](https://www.redis.net.cn/order/3530.html)       | 检查给定 key 是否存在。返回 1 表示存在，返回 0 表示不存在。  |
| 4    | [expire key](https://www.redis.net.cn/order/3531.html) seconds | 为给定 key 设置过期时间。单位 秒。如果 key 后续被重新设置值，比如 set key value，则 key 过期时间失效。 |
| 5    | [expireat key timestamp](https://www.redis.net.cn/order/3532.html) | EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置过期时间。 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。如果 key 后续被重新设置值，比如 set key value，则 key 过期时间失效。 |
| 6    | [pexpire key milliseconds](https://www.redis.net.cn/order/3533.html) | 设置 key 的过期时间亿以毫秒计。如果 key 后续被重新设置值，比如 set key value，则 key 过期时间失效。 |
| 7    | [pexpire](https://www.redis.net.cn/order/3533.html)at[ key milliseconds-timestamp](https://www.redis.net.cn/order/3534.html) | 设置 key 过期时间的时间戳(unix timestamp) 以毫秒计。如果 key 后续被重新设置值，比如 set key value，则 key 过期时间失效。 |
| 8    | keys[ pattern](https://www.redis.net.cn/order/3535.html)     | 查找所有符合给定模式( pattern)的 key 。* 表示1个或多个，？ 表示一个任意字符。keys * ：查找所有key，keys user*：查找以 user 开头的 key，keys ag?：查找 ag 开头，且后面只有一个字符的 key。 |
| 9    | [move key db](https://www.redis.net.cn/order/3536.html)      | 将当前数据库的 key 移动到给定的数据库 db 当中。              |
| 10   | [persist key](https://www.redis.net.cn/order/3537.html)      | 移除 key 的过期时间，key 将持久保持。                        |
| 11   | [pttl key](https://www.redis.net.cn/order/3538.html)         | 以毫秒为单位返回 key 的剩余的过期时间。如果没有对 key 设置超时，则返回 -1；-1 表示超时不存在。正常情况返回大于0的正数。 |
| 12   | [ttl key](https://www.redis.net.cn/order/3539.html)          | 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。 |
| 13   | randomkey                                                    | 从当前数据库中随机返回一个 key 。                            |
| 14   | [rename key newkey](https://www.redis.net.cn/order/3541.html) | 修改 key 的名称。key 不存在时会报错：(error) ERR no such key。如果 newkey 已经存在时，则会删除旧值。 |
| 15   | [renamenx key newkey](https://www.redis.net.cn/order/3542.html) | 仅当 newkey 不存在时，将 key 改名为 newkey 。key 不存在时报错。 |
| 16   | [type key](https://www.redis.net.cn/order/3543.html)         | 返回 key 所储存的值的类型。有 string、list、set、zset、hash。如果 key 不存在，则返回 none |

在线命令演示源码：[Redis 基本命令、键（key）命令、数据类型概述.sql](https://gitee.com/wangmx1993/my-document/blob/master/redis/Redis 基本命令、键（key）命令、数据类型概述.sql)

# Redis 数据类型概述

1、Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

2、这里暂时先做个概述，后续会详细说明。

3、在线命令演示源码：[Redis 基本命令、键（key）命令、数据类型概述.sql](https://gitee.com/wangmx1993/my-document/blob/master/redis/Redis 基本命令、键（key）命令、数据类型概述.sql)

# Redis 字符串(String)

1、string 是 redis最基本的类型，一个key对应一个value。一个键最大能存储512MB。

2、string 类型是二进制安全的，可以包含任何数据，比如 jpg 图片或者序列化的对象 。

3、Redis 字符串(String)官网文档：https://www.redis.net.cn/order/3544.html

| 序号 | 命令                                                         | 描述                                                         |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | set [key value](https://www.redis.net.cn/tutorial/8669.html) | 设置指定 key 的值。key 存在时，覆盖其值。总是返回ok。设置的数字会自动转为字符串存储 |
| 2    | [get key](https://www.redis.net.cn/tutorial/8670.html)       | 获取指定 key 的值。如果 key 不存在，则返回 (nil) 相当于 null。如果 key 的类型不是 string ，则报错。 |
| 3    | [getrange key start end](https://www.redis.net.cn/tutorial/8671.html) | [range](https://www.redis.net.cn/tutorial/8671.html)：范围、界限。返回 key 中字符串值的子字符。索引 [start ,end] 从 0开始。可以为负数，如 -1表示倒数第一位，-2 表示倒数第二位。 |
| 4    | [getset key value](https://www.redis.net.cn/tutorial/8672.html) | 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。key 不存在时返回为(nil)，同时创建新值。 |
| 5    | [getbit key offset](https://www.redis.net.cn/tutorial/8673.html) | 对 key 所储存的字符串值，获取指定偏移量上的位(bit)。         |
| 6    | [mget key1 [key2..\]](https://www.redis.net.cn/tutorial/8674.html) | 获取所有(一个或多个)给定 key 的值。不存在的 key 返回 (nil)   |
| 7    | [setbit key offset value](https://www.redis.net.cn/tutorial/8675.html) | 对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。   |
| 8    | [setex key seconds value](https://www.redis.net.cn/tutorial/8676.html) | 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。 |
| 9    | [setnx key value](https://www.redis.net.cn/tutorial/8677.html) | 只有在 key 不存在时设置 key 的值。                           |
| 10   | setrange[ key offset value](https://www.redis.net.cn/tutorial/8678.html) | 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。 |
| 11   | strlen[ key](https://www.redis.net.cn/tutorial/8679.html)    | 返回 key 所储存的字符串值的长度。不存在的 key 返回 0         |
| 12   | [mset key value [key value ...\]](https://www.redis.net.cn/tutorial/8680.html) | 同时设置一个或多个 key-value 对。key 存在时，覆盖其值。      |
| 13   | [msetnx key value [key value ...\]](https://www.redis.net.cn/tutorial/8681.html) | 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在才设置。 |
| 14   | [psetex key milliseconds value](https://www.redis.net.cn/tutorial/8682.html) | 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。 |
| 15   | [incr key](https://www.redis.net.cn/tutorial/8683.html)      | 将 key 中储存的数字值增一。increment：ˈɪŋkrəmənt 增量、增加。如果不是数值，则报错。如果 key 不存在，则新建，incr 后，值为1。 |
| 16   | incrby [key increment](https://www.redis.net.cn/tutorial/8684.html) | 将 key 所储存的值加上给定的增量值（increment） 。如果不是数值，则报错。如果 key 不存在，则新建。 |
| 17   | incrbyfloat[ key increment](https://www.redis.net.cn/tutorial/8685.html) | 将 key 所储存的值加上给定的浮点增量值（increment） 。如果不是数值，则报错。如果 key 不存在，则新建。[increment](https://www.redis.net.cn/tutorial/8685.html) 不能是变量。 |
| 18   | [decr key](https://www.redis.net.cn/tutorial/8686.html)      | 将 key 中储存的数字值减一。如果不是数值，则报错。如果 key 不存在，则新建，decr 后，值为 -1。如果 key 不存在，则新建。 |
| 19   | [decrby key decrement](https://www.redis.net.cn/tutorial/8687.html) | key 所储存的值减去给定的减量值（decrement） 。如果 key 不存在，则新建。[increment](https://www.redis.net.cn/tutorial/8685.html) |
| 20   | [append key value](https://www.redis.net.cn/tutorial/8688.html) | 如果 key 已经存在并且是一个字符串， APPEND 命令将 value 追加到 key 原来的值的末尾。如果 key 不存在，则新建。value 不能是变量。 |

4、在线命令演示：[Redis 字符串(String)命令演示.sql](https://gitee.com/wangmx1993/my-document/blob/master/redis/Redis 字符串(String)命令演示.sql)

![img](https://img-blog.csdnimg.cn/20201010201027478.gif)

# **Redis 哈希（Hash）**

1、Redis hash 是一个键值对集合，值可以看成一个 Map。

2、Redis hash 是一个string类型的field和value的映射表，hash特别适合用于存储对象。

3、每个 hash 可以存储 40多亿键值对。

hmset key filed value [filed2 value2 filed3 value3 ...]：同时为 key 指定多个 filed 与 valuehgetall key：获取 key 中的所有 filed-value

4、Redis 哈希(Hash)官网文档：https://www.redis.net.cn/order/3564.html

| 序号 | 命令                                                         | 描述                                                         |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | [hdel key field2 [field2\]](https://www.redis.net.cn/order/3564.html) | 删除一个或多个哈希表字段。返回值成功删除的个数。key 或 field 不存在时会自动忽略。 |
| 2    | [hexists key field](https://www.redis.net.cn/order/3565.html) | 查看哈希表 key 中，指定的字段是否存在。返回1表示有，返回0表示没有。key 不存在时也返回0. |
| 3    | [hget key field](https://www.redis.net.cn/order/3566.html)   | 获取存储在哈希表中指定字段的值。key 或 field 不存在时，返回 (nil)。 |
| 4    | [hgetall key](https://www.redis.net.cn/order/3567.html)      | 获取在哈希表中指定 key 的所有字段和值                        |
| 5    | [hincrby key field increment](https://www.redis.net.cn/order/3568.html) | 为哈希表 key 中的指定字段的整数值加上增量 increment 。field 必须是数值，否则报错。key 不存在时会自动新建。field 不存在时也会自动新建。 |
| 6    | [hincrbyfloat key field increment](https://www.redis.net.cn/order/3569.html) | 为哈希表 key 中的指定字段的浮点数值加上增量 increment 。field 必须是数值，否则报错。key 不存在时会自动新建。field 不存在时也会自动新建。 |
| 7    | [hkeys key](https://www.redis.net.cn/order/3570.html)        | 获取所有哈希表中的字段                                       |
| 8    | [hlen key](https://www.redis.net.cn/order/3571.html)         | 获取哈希表中字段的数量。key 不存在时返回0.                   |
| 9    | [hmget key field1 [field2\]](https://www.redis.net.cn/order/3572.html) | 获取所有给定字段的值。key 或 field 不存在时，返回 (nil)。    |
| 10   | [hmset key field1 value1 [field2 value2 \]](https://www.redis.net.cn/order/3573.html) | 同时将多个 field-value (域-值)对设置到哈希表 key 中。field 存在时，覆盖 value。 |
| 11   | [hset key field value](https://www.redis.net.cn/order/3574.html) | 将哈希表 key 中的字段 field 的值设为 value 。field 存在时，覆盖 value。 |
| 12   | [hsetnx key field value](https://www.redis.net.cn/order/3575.html) | 只有在字段 field 不存在时，设置哈希表字段的值。              |
| 13   | [hvals key](https://www.redis.net.cn/order/3576.html)        | 获取哈希表中所有值                                           |
| 14   | HSCAN key cursor [MATCH pattern] [COUNT count]               | 迭代哈希表中的键值对。                                       |

5、命令在线演示：[Redis 哈希（Hash）命令演示.sql](https://gitee.com/wangmx1993/my-document/blob/master/redis/Redis 哈希（Hash）命令演示.sql)

![img](https://img-blog.csdnimg.cn/20201010201056850.gif)

# Redis 列表(List)

1、Redis 列表是简单的字符串列表，按照插入顺序排序，可以添加一个元素导列表的头部（左边）或者尾部（右边）。

2、每个列表最多可存储 4294967295 个元素（约40多亿)

lpush key value1 value2 value3 ...：在指定的 key 关联的 lsit 的头部插入所有的 value，如果 key 不存在，则会先创建一个与该 key 关联的空链表，之后向链表的头部插入数据，插入成功，返回插入的个数。lrange key start end：获取链表中 [start,end] 之间的元素值，从0开始计数。可以为负数，如 -1 表示链表尾部的元素，-2 表示倒数第二个。

3、Redis 列表(List)官网文档：https://www.redis.net.cn/order/3577.html

| 序号 | 命令                                                         | 描述                                                         |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | [blpop key1 [key2 \] timeout](https://www.redis.net.cn/order/3577.html) | 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。 |
| 2    | [brpop key1 [key2 \] timeout](https://www.redis.net.cn/order/3578.html) | 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。 |
| 3    | [brpoplpush source destination timeout](https://www.redis.net.cn/order/3579.html) | 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。 |
| 4    | [lindex key index](https://www.redis.net.cn/order/3580.html) | 通过索引获取列表中的元素                                     |
| 5    | [linsert key BEFORE\|AFTER pivot value](https://www.redis.net.cn/order/3581.html) | 在 pivot 元素前/后插入 value 元素。成功时返回列表中元素的个数。key 不存在时返回0。pivot 不存在时返回-1。 |
| 6    | [llen key](https://www.redis.net.cn/order/3582.html)         | 获取列表长度。key 不存在时返回0；                            |
| 7    | [lpop key](https://www.redis.net.cn/order/3583.html)         | 返回并弹出指定 key 关联的列表中的第一个元素（头部元素）。如果 key 不存在，则返回(nil)。弹出之后，列表中的此元素也就不存在了。 |
| 8    | [lpush key value1 [value2\]](https://www.redis.net.cn/order/3584.html) | 将一个或多个值插入到列表头部。如果 key 不存在，则先创建一个与该 key 关联的空列表，然后向列表的头部插入数据，返回插入成功的个数。因为有索引，所以可以插入重复的元素。返回 list 中的元素个数。 |
| 9    | [lpushx key value](https://www.redis.net.cn/order/3585.html) | 将一个或多个值插入到已存在的列表头部                         |
| 10   | [lrange key start stop](https://www.redis.net.cn/order/3586.html) | 获取链表中 [start,end] 之间的元素值。索引从0开始，可以为负数，如 -1 表示倒数第一个元素，-2 表示倒数第二个元素...。end 可以超出列表的整个大小，此时多余的会自动忽略。 |
| 11   | [lrem key count value](https://www.redis.net.cn/order/3587.html) | 删除 count 个值为 value 的元素。count > 0，则从头向尾遍历并删除 count 个值为 value 的元素，count < 0 ，则从尾向前遍历进行删除。count =0，则删除链表中所有的 value 元素。返回删除成功的个数。value 不存在时返回0。key 不存在时返回0。 |
| 12   | [lset key index value](https://www.redis.net.cn/order/3588.html) | 设置列表中索引为 index 的元素值，0 表示首元素，-1表示尾元素。如果 index 不存在，则抛出异常。如果 key 不存在，也抛出异常。 |
| 13   | [ltrim key start stop](https://www.redis.net.cn/order/3589.html) | 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。 |
| 14   | [rpop key](https://www.redis.net.cn/order/3590.html)         | 移除并获取列表最后一个元素                                   |
| 15   | [rpoplpush source destination](https://www.redis.net.cn/order/3591.html) | 将 resource 链表的尾部元素弹出并添加到 destination 链表的头部。成功时返回操作的元素。如果resource不存在，则返回（nil）。如果 destination 不存在，则自动会新建。 |
| 16   | [rpush key value1 [value2\]](https://www.redis.net.cn/order/3592.html) | 在列表尾部添加一个或多个值                                   |
| 17   | [rpushx key value](https://www.redis.net.cn/order/3593.html) | 为已存在的列表的尾部添加值                                   |

rpoplpush 使用场景：

  Redis 链表经常会被用于消息队列的服务，已完成多程序之间的消息交互。   假设一个应用程序正在执行 lpush 操作向链表头部插入新的元素，通常将这样的程序称之为"生产者(Producer)"，   而另一个应用程序正在执行 rpop 操作从链表的尾部取出元素，通常称之为"消费者（Consumer）"。   如果此时消费者程序取出消息后突然崩溃了，由于该消息已经被取出且没有被正常处理，那么就认为此消息已经丢失，由此可能导致业务数据丢失。   然而通过 rpoplpush 命令，消费者程序在主消息队列中取出消息之后再将其插入到备份队列中，直到消费者程序完成正常的处理后，再将该消息从备份列表中删除。   同时还可以提供一个守护进程，当发现备份队列中的消息过期时，可以重新将其再放回到主消息队列中，以便其它消费者程序继续处理。

 ![img](https://img-blog.csdnimg.cn/20201010200802602.gif)

# Redis 集合(Set)

1、Redis 的 Set 是 string 类型的无序集合。和 java 一样，集合中不会有重复的元素。

2、集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。

3、每个集合中最大的成员数为 4294967295（40多亿个成员)。

sadd key value1 value2 ...：向集合 key 中添加元素，key 不存在时会自动新建，value 存在时，后一次的会被忽略。smembers key：获取集合 key 中的所有元素。

4、Redis 集合(Set)官网文档：https://www.redis.net.cn/order/3594.html

| 序号 | 命令                                                         | 描述                                                         |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | [sadd key member1 [member2\]](https://www.redis.net.cn/order/3594.html) | 向集合添加一个或多个成员。如果 value 已经存在，则不会再添加。返回插入成功的个数。 |
| 2    | [scard key](https://www.redis.net.cn/order/3595.html)        | 获取集合的成员数。key 不存在时，返回0。                      |
| 3    | [sdiff key1 [key2\]](https://www.redis.net.cn/order/3596.html) | 返回给定所有集合的差集。求 key1 与 key2 key3 ...的差集，即 key1 中有，但 key2 key3 ...都没有的元素。 |
| 4    | [sdiffstore destination key1 [key2\]](https://www.redis.net.cn/order/3597.html) | 将 key1 集合与其它集合的差集放入到 destination 集合中。如果 destination 已经存在且有值，则会被全部清除，不存在时会新建。 |
| 5    | [sinter key1 [key2\]](https://www.redis.net.cn/order/3598.html) | 返回给定所有集合的交集。求 key1 与 key2 key3 ...集合的交集。 |
| 6    | [sinterstore destination key1 [key2\]](https://www.redis.net.cn/order/3599.html) | 将 key1 与其它集合的交集存放到 destination 集合中。如果 destination 集合已经有值，则会先被清理。 |
| 7    | [sismember key member](https://www.redis.net.cn/order/3600.html) | 判断 member 元素是否是集合 key 的成员。返回1表示存在，返回0表示不存在。key 不存在时也返回0。 |
| 8    | [smembers key](https://www.redis.net.cn/order/3601.html)     | 返回集合中的所有成员                                         |
| 9    | [smove source destination member](https://www.redis.net.cn/order/3602.html) | 将 member 元素从 source 集合移动到 destination 集合          |
| 10   | [spop key](https://www.redis.net.cn/order/3603.html)         | 移除并返回集合中的一个随机元素                               |
| 11   | [srandmember key [count\]](https://www.redis.net.cn/order/3604.html) | 返回集合中一个或多个随机数。key 不存在时返回（nil）          |
| 12   | [srem key member1 [member2\]](https://www.redis.net.cn/order/3605.html) | 移除集合中一个或多个成员。返回成功删除的个数。 当然也可以使用 del key 直接删除整个集合。 |
| 13   | [sunion key1 [key2\]](https://www.redis.net.cn/order/3606.html) ... | 返回所有给定集合的并集。求 key1 与集合 key2 key3 ...的并集。 |
| 14   | [sunionstore destination key1 [key2\]](https://www.redis.net.cn/order/3607.html) ... | 所有给定集合的并集存储在 destination 集合中。将并集结果存放到 destination 集合中。如果 destination 已经有值，则会被清除。 |
| 15   | [sscan key cursor [MATCH pattern\] [COUNT count]](https://www.redis.net.cn/order/3608.html) | 迭代集合中的元素                                             |

![img](https://img-blog.csdnimg.cn/20201010200903246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdteDE5OTMzMjg=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20201010200914979.gif)

# Redis 有序集合(sorted set)

1、Redis zset 和 set 一样也是 string 类型元素的集合，且不允许重复的成员。

2、不同的是每个元素都会关联一个 double 类型的分数，redis 正是通过分数来为集合中的成员进行从小到大的排序。

3、zset 的成员是唯一的，但分数(score)却可以重复。

zadd key score1 member1 score2 member2 ...：添加元素到集合，元素在集合中存在则更新对应 score：zrangebyscore key min max ：返回分数在 [mix,max]之间的成员，并按照分数由低到高排序。

4、Redis 有序集合(sorted set)官网文档：https://www.redis.net.cn/order/3609.html