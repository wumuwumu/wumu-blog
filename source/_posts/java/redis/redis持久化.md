---
title: redis持久化
abbrlink: dcfaa904
date: 2021-01-29 12:00:00
---

[Redis](http://redis.io/)有两种持久化的方式：快照（`RDB`文件）和追加式文件（`AOF`文件）：

- RDB持久化方式会在一个特定的间隔保存那个时间点的一个数据快照。
- AOF持久化方式则会记录每一个服务器收到的写操作。在服务启动时，这些记录的操作会逐条执行从而重建出原来的数据。写操作命令记录的格式跟Redis协议一致，以追加的方式进行保存。
- Redis的持久化是可以禁用的，就是说你可以让数据的生命周期只存在于服务器的运行时间里。
- 两种方式的持久化是可以同时存在的，但是当Redis重启时，AOF文件会被优先用于重建数据。

## RDB

### 工作原理

- Redis调用fork()，产生一个子进程。
- 子进程把数据写到一个临时的RDB文件。
- 当子进程写完新的RDB文件后，把旧的RDB文件替换掉。

### 优点

- RDB文件是一个很简洁的单文件，它保存了某个时间点的Redis数据，很适合用于做备份。你可以设定一个时间点对RDB文件进行归档，这样就能在需要的时候很轻易的把数据恢复到不同的版本。
- 基于上面所描述的特性，RDB很适合用于灾备。单文件很方便就能传输到远程的服务器上。
- RDB的性能很好，需要进行持久化时，主进程会fork一个子进程出来，然后把持久化的工作交给子进程，自己不会有相关的I/O操作。
- 比起AOF，在数据量比较大的情况下，RDB的启动速度更快。

### 缺点

- RDB容易造成数据的丢失。假设每5分钟保存一次快照，如果Redis因为某些原因不能正常工作，那么从上次产生快照到Redis出现问题这段时间的数据就会丢失了。
- RDB使用`fork()`产生子进程进行数据的持久化，如果数据比较大的话可能就会花费点时间，造成Redis停止服务几毫秒。如果数据量很大且CPU性能不是很好的时候，停止服务的时间甚至会到1秒。

### 文件路径和名称

默认Redis会把快照文件存储为当前目录下一个名为`dump.rdb`的文件。要修改文件的存储路径和名称，可以通过修改配置文件`redis.conf`实现：

```
# RDB文件名，默认为dump.rdb。
dbfilename dump.rdb

# 文件存放的目录，AOF文件同样存放在此目录下。默认为当前工作目录。
dir ./
```

### 保存点（RDB的启用和禁用）

你可以配置保存点，使Redis如果在每N秒后数据发生了M次改变就保存快照文件。例如下面这个保存点配置表示每60秒，如果数据发生了1000次以上的变动，Redis就会自动保存快照文件：

```
save 60 1000
```

保存点可以设置多个，Redis的配置文件就默认设置了3个保存点：

```
# 格式为：save <seconds> <changes>
# 可以设置多个。
save 900 1 #900秒后至少1个key有变动
save 300 10 #300秒后至少10个key有变动
save 60 10000 #60秒后至少10000个key有变动
```

如果想禁用快照保存的功能，可以通过注释掉所有"save"配置达到，或者在最后一条"save"配置后添加如下的配置：

```
save ""
```

### 错误处理

默认情况下，如果Redis在后台生成快照的时候失败，那么就会停止接收数据，目的是让用户能知道数据没有持久化成功。但是如果你有其他的方式可以监控到Redis及其持久化的状态，那么可以把这个功能禁止掉。

```
stop-writes-on-bgsave-error yes
```

### 数据压缩

默认Redis会采用`LZF`对数据进行压缩。如果你想节省点CPU的性能，你可以把压缩功能禁用掉，但是数据集就会比没压缩的时候要打。

```
rdbcompression yes
```

### 数据校验

从版本5的RDB的开始，一个`CRC64`的校验码会放在文件的末尾。这样更能保证文件的完整性，但是在保存或者加载文件时会损失一定的性能（大概10%）。如果想追求更高的性能，可以把它禁用掉，这样文件在写入校验码时会用`0`替代，加载的时候看到`0`就会直接跳过校验。

```
rdbchecksum yes
```

### 手动生成快照

Redis提供了两个命令用于手动生成快照。

#### SAVE

[SAVE](http://redis.io/commands/save)命令会使用同步的方式生成RDB快照文件，这意味着在这个过程中会阻塞所有其他客户端的请求。因此不建议在生产环境使用这个命令，除非因为某种原因需要去阻止Redis使用子进程进行后台生成快照（例如调用`fork(2)`出错）。

#### BGSAVE

[BGSAVE](http://redis.io/commands/bgsave)命令使用后台的方式保存RDB文件，调用此命令后，会立刻返回`OK`返回码。Redis会产生一个子进程进行处理并立刻恢复对客户端的服务。在客户端我们可以使用[LASTSAVE](http://redis.io/commands/lastsave)命令查看操作是否成功。

```
127.0.0.1:6379> BGSAVE
Background saving started
127.0.0.1:6379> LASTSAVE
(integer) 1433936394
```

> 配置文件里禁用了快照生成功能不影响`SAVE`和`BGSAVE`命令的效果。

## AOF

快照并不是很可靠。如果你的电脑突然宕机了，或者电源断了，又或者不小心杀掉了进程，那么最新的数据就会丢失。而AOF文件则提供了一种更为可靠的持久化方式。每当Redis接受到会修改数据集的命令时，就会把命令追加到AOF文件里，当你重启Redis时，AOF里的命令会被重新执行一次，重建数据。

### 优点

- 比RDB可靠。你可以制定不同的fsync策略：不进行fsync、每秒fsync一次和每次查询进行fsync。默认是每秒fsync一次。这意味着你最多丢失一秒钟的数据。
- AOF日志文件是一个纯追加的文件。就算是遇到突然停电的情况，也不会出现日志的定位或者损坏问题。甚至如果因为某些原因（例如磁盘满了）命令只写了一半到日志文件里，我们也可以用`redis-check-aof`这个工具很简单的进行修复。
- 当AOF文件太大时，Redis会自动在后台进行重写。重写很安全，因为重写是在一个新的文件上进行，同时Redis会继续往旧的文件追加数据。新文件上会写入能重建当前数据集的最小操作命令的集合。当新文件重写完，Redis会把新旧文件进行切换，然后开始把数据写到新文件上。
- AOF把操作命令以简单易懂的格式一条接一条的保存在文件里，很容易导出来用于恢复数据。例如我们不小心用`FLUSHALL`命令把所有数据刷掉了，只要文件没有被重写，我们可以把服务停掉，把最后那条命令删掉，然后重启服务，这样就能把被刷掉的数据恢复回来。

### 缺点

- 在相同的数据集下，AOF文件的大小一般会比RDB文件大。
- 在某些fsync策略下，AOF的速度会比RDB慢。通常fsync设置为每秒一次就能获得比较高的性能，而在禁止fsync的情况下速度可以达到RDB的水平。
- 在过去曾经发现一些很罕见的BUG导致使用AOF重建的数据跟原数据不一致的问题。

### 启用AOF

把配置项`appendonly`设为`yes`：

```
appendonly yes
```

### 文件路径和名称

```
# 文件存放目录，与RDB共用。默认为当前工作目录。
dir ./

# 默认文件名为appendonly.aof
appendfilename "appendonly.aof"
```

### 可靠性

你可以配置Redis调用fsync的频率，有三个选项：

- 每当有新命令追加到AOF的时候调用fsync。速度最慢，但是最安全。
- 每秒fsync一次。速度快（2.4版本跟快照方式速度差不多），安全性不错（最多丢失1秒的数据）。
- 从不fsync，交由系统去处理。这个方式速度最快，但是安全性一般。

推荐使用每秒fsync一次的方式（默认的方式），因为它速度快，安全性也不错。相关配置如下：

```
# appendfsync always
appendfsync everysec
# appendfsync no
```

### 日志重写

随着写操作的不断增加，AOF文件会越来越大。例如你递增一个计数器100次，那么最终结果就是数据集里的计数器的值为最终的递增结果，但是AOF文件里却会把这100次操作完整的记录下来。而事实上要恢复这个记录，只需要1个命令就行了，也就是说AOF文件里那100条命令其实可以精简为1条。所以Redis支持这样一个功能：在不中断服务的情况下在后台重建AOF文件。

工作原理如下：

- Redis调用fork()，产生一个子进程。
- 子进程把新的AOF写到一个临时文件里。
- 主进程持续把新的变动写到内存里的buffer，同时也会把这些新的变动写到旧的AOF里，这样即使重写失败也能保证数据的安全。
- 当子进程完成文件的重写后，主进程会获得一个信号，然后把内存里的buffer追加到子进程生成的那个新AOF里。
- Redis

我们可以通过配置设置日志重写的条件：

```
# Redis会记住自从上一次重写后AOF文件的大小（如果自Redis启动后还没重写过，则记住启动时使用的AOF文件的大小）。
# 如果当前的文件大小比起记住的那个大小超过指定的百分比，则会触发重写。
# 同时需要设置一个文件大小最小值，只有大于这个值文件才会重写，以防文件很小，但是已经达到百分比的情况。

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

要禁用自动的日志重写功能，我们可以把百分比设置为0：

```
auto-aof-rewrite-percentage 0
```

> Redis 2.4以上才可以自动进行日志重写，之前的版本需要手动运行[BGREWRITEAOF](http://redis.io/commands/bgrewriteaof)这个命令。

### 数据损坏修复

如果因为某些原因（例如服务器崩溃）AOF文件损坏了，导致Redis加载不了，可以通过以下方式进行修复：

- 备份AOF文件。

- 使用`redis-check-aof`命令修复原始的AOF文件：

  ```
  $ redis-check-aof --fix
  ```

- 可以使用`diff -u`命令看下两个文件的差异。

- 使用修复过的文件重启Redis服务。

### 从RDB切换到AOF

这里只说Redis >= 2.2版本的方式：

- 备份一个最新的`dump.rdb`的文件，并把备份文件放在一个安全的地方。

- 运行以下两条命令：

  ```
  $ redis-cli config set appendonly yes
  $ redis-cli config set save ""
  ```

- 确保数据跟切换前一致。

- 确保数据正确的写到AOF文件里。

> 第二条命令是用来禁用RDB的持久化方式，但是这不是必须的，因为你可以同时启用两种持久化方式。

> 记得对配置文件`redis.conf`进行编辑启用AOF，因为命令行方式修改配置在重启Redis后就会失效。

# 具体方案

## 持久化配置

- RBD和AOF建议同时打开（Redis4.0之后支持）
- RDB做冷备，AOF做数据恢复（数据更可靠）
- RDB采取默认配置即可，AOF推荐采取everysec每秒策略

AOF和RDB还不懂的，请转移到如下几篇：

[看完这篇还不懂Redis的RDB持久化，你们来打我！](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247492839&idx=1&sn=c205eb385b0b1255f1ef9c8b3e3b4b6d&chksm=ebd5dbcbdca252dd677f470e4f2df3578c952015c312246d0171b126e3b1fd007d277e96f602&scene=21#wechat_redirect)

[天天在用Redis，那你对Redis的AOF持久化到底了解多少呢？](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247492865&idx=1&sn=00904394aa074df14bb65a14a56e1ffe&chksm=ebd5da2ddca2533ba1e3d20fd82b8acee93fc75199e9c10e38b62387ca0579c80390f72e3bbd&scene=21#wechat_redirect)

## 数据备份方案

### 需求

我们需要定时备份rdb文件来做冷备，为什么？不是有aof和rbd了吗为什么还要单独写定时任务去备份？因为Redis的aof和rdb是仅仅有一个最新的，比如谁手贱再Redis宕机的时候执行`rm -rf aof/rdb`了，那不就GG了吗？或者rdb/aof文件损坏了等不可预期的情况。所以我们需要单独备份rdb文件以防万一。

为什么不定时备份aof而是rdb？定时备份aof没意义呀，**定时**本身就是冷备份，不是实时的，rdb文件又小恢复又快，她哪里不香？

### 方案

- 写crontab定时调度脚本去做数据备份。
- 每小时都copy一份redis的rdb文件到一个其他目录中，这个目录里的rdb文件仅仅保留48小时内的。也就是每小时都做备份，保留2天内的rdb，只保留48个rdb。
- 每天0点0分copy一份redis的rdb文件到一个其他目录中，这个保留一个月的。也就是按天备份。
- 每天半夜找个时间将当前服务上的所有rdb备份都上传到云服务上。

### 实现

#### 按小时

> 每小时copy一次备份，删除48小时前的数据。

```shell
crontab -e
# 每小时都执行/usr/local/redis/copy/redis_rdb_copy_hourly.sh脚本
0 * * * * sh /usr/local/redis/copy/redis_rdb_copy_hourly.sh


# redis_rdb_copy_hourly.sh脚本的内容如下：

#!/bin/sh 
# +%Y%m%d%k == 年月日时
cur_date=`date +%Y%m%d%k`
rm -rf /usr/local/redis/rdb/$cur_date
mkdir /usr/local/redis/rdb/$cur_date
# 拷贝rdb到目录
cp /var/redis/6379/dump.rdb /usr/local/redis/rdb/$cur_date
# date -d -48hour +%Y%m%d%k == 48小时前的日期，比如今天2020060214，这个结果就是2020053114
del_date=`date -d -48hour +%Y%m%d%k`
# 删除48小时之前的目录
rm -rf /usr/local/redis/rdb/$del_date
```

### 按天

> 每天copy一次备份，删除一个月前的数据。

```shell
crontab -e
# 每天0点0分开始执行/usr/local/redis/copy/redis_rdb_copy_daily.sh脚本
0 0 * * * sh /usr/local/redis/copy/redis_rdb_copy_daily.sh

# redis_rdb_copy_daily.sh脚本的内容如下：

#!/bin/sh 
# 年月日
cur_date=`date +%Y%m%d`
rm -rf /usr/local/redis/rdb/$cur_date
mkdir /usr/local/redis/rdb/$cur_date
# 拷贝rdb到目录
cp /var/redis/6379/dump.rdb /usr/local/redis/rdb/$cur_date

# 获取一个月前的时间，比如今天是20200602，那么del_date就是20200502
del_date=`date -d -1month +%Y%m%d`
# 删除一个月前的数据
rm -rf /usr/local/redis/rdb/$del_date
```

### 传到云

没法演示，最终目的就是磁盘备份完上传到云，云保留多少天等策略自己看需求。

## 数据恢复方案

### redis挂了

如果仅仅是redis进程挂了，那么直接重启redis进程即可，Redis会按照持久化配置直接基于持久化文件进行恢复数据。

如果有AOF则按照AOF，AOF和RDB一起开的话也走AOF。

### 持久化文件丢了

如果持久化文件（rdb/aof）损坏了，或者直接丢失了。那么就要采取我们上面所做的rdb备份来进行恢复了。

> 不要脑子一热想着很简单，就以为直接把rdb拖过来重启redis进程就完事了，这种想法有很多问题。慢慢道来。

#### 问题

问题一：直接把备份的rdb扔到redis持久化目录下然后重启redis不行的原因在于：redis是按照先aof后rdb进行恢复的，所以都是开启aof的，redis启动后会重新生成新的aof文件，里面是空的。所以不会进行任何数据恢复，也就是说虽然你把rdb丢给redis了，但是redis会按照aof来恢复，而aof是redis启动的时候新生成的空文件，所以不会有任何数据进行恢复。

问题二：那么我们把rdb文件丢给redis后，先将redis的aof关闭再启动redis进程不就能按照rdb来进行恢复了吗？是这样的，没毛病！但是新的问题来了，我们aof肯定要开的，aof对数据保障更可靠。那什么我们按照rdb文件恢复完后再修改redis配置文件开启aof然后重启redis进程不就得了嘛？大哥…你打开aof然后重启redis，这时候redis又会生成一个空的aof文件，这时候恢复的时候又是啥数据都没了。

> 因为数据是存到内存里，你重启后肯定没了，需要持久化文件来恢复。这时候aof是空的，我恢复个鸡毛啊。

### 具体方案

> 可能有人想到方案了，但是耐心看完，看看我的文采如何。

我不管你是持久化文件丢了还是坏了，我都先`rm -rf *` 给他删了。

- 停止redis进程
- 删除坏掉的rdb和aof持久化文件。
- 修改配置文件关闭redis的aof持久化。
- 找到最新备份的rdb文件扔到redis的持久化目录里。（这里最新的肯定是按照小时备份的最后一个）
- 启动Redis进程
- 执行`set appendonly yes`动态打开aof持久化。

> 也就是说打开aof的操作不是修改配置文件然后重启，而是先热修改让他生成aof，这次生成肯定是会带着内存中完整的数据的。然后再修改配置文件重启。

- 等aof文件生成后再修改redis配置文件打开aof。
- 重启redis进程。
- 完美收官。