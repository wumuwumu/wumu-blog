---
title: sudoer文件解析
date: 2020-9-05 21:40:23
tags:
- linux
---

- sudo的权限控制可以在`/etc/sudoers`文件中查看到。

  如果想要控制某个用户(或某个组用户)只能执行root权限中的一部分命令, 或者允许某些用户使用sudo时不需要输入密码,就需要对该文件有所了解。

  一般来说，通过`cat /etc/sudoers`指令来查看该文件, 会看到如下几行代码:

  ```bash
  root   ALL=(ALL:ALL) ALL
  %wheel ALL=(ALL) ALL
  %sudo  ALL=(ALL:ALL) ALL
  ```

  对`/etc/sudoers`文件进行编辑的代码公式可以概括为:

  ```bash
  授权用户/组 主机=[(切换到哪些用户或组)] [是否需要输入密码验证] 命令1,命令2,...
  ```

  凡是`[ ]`中的内容, 都能省略; 命令和命令之间用`,`号分隔;

  为了方便说明, 将公式的各个部分称呼为字段1 - 字段5:

  ```bash
  授权用户/组 主机  =[(切换到哪些用户或组)] [是否需要输入密码验证] 命令1,命令2,...
  字段1      字段2  =[(字段3)] [字段4] 字段5
  ```

  字段3、字段4，是可以省略的。

  在上面的默认例子中, "字段1"不以`%`号开头的表示"将要授权的用户", 比如例子中的`root`；
  以`%`号开头的表示"将要授权的组", 比如例子中的`%wheel`组 和 `%sudo`组。

  "字段2"表示允许登录的主机, ALL表示所有; 如果该字段不为ALL,表示授权用户只能在某些机器上登录本服务器来执行sudo命令. 比如:

  ```bash
  jack mycomputer=/usr/sbin/reboot,/usr/sbin/shutdown
  ```

  表示: 普通用户jack在主机(或主机组)mycomputer上, 可以通过sudo执行reboot和shutdown两个命令。"字段3"和"字段4"省略。

  "字段3"如果省略, 相当于`(root:root)`，表示可以通过`sudo`提权到root; 如果为`(ALL)`或者`(ALL:ALL)`, 表示能够提权到`(任意用户:任意用户组)`。

  请注意，"字段3"如果没省略,必须使用`( )`双括号包含起来。这样才能区分是省略了"字段3"还是省略了"字段4"。

  "字段4"的可能取值是`NOPASSWD:`。请注意NOPASSWD后面带有冒号`:`。表示执行sudo时可以不需要输入密码。比如:

  ```bash
  lucy ALL=(ALL) NOPASSWD: /bin/useradd
  ```

  表示: 普通用户lucy可以在任何主机上, 通过sudo执行`/bin/useradd`命令, 并且不需要输入密码.

  又比如:

  ```bash
  peter ALL=(ALL) NOPASSWD: ALL
  ```

  表示: 普通用户peter可以在任何主机上, 通过sudo执行任何命令, 并且不需要输入密码。

  "字段5"是使用逗号分开一系列命令,这些命令就是授权给用户的操作; ALL表示允许所有操作。

  你可能已经注意到了, 命令都是使用绝对路径, 这是为了避免目录下有同名命令被执行，从而造成安全隐患。

  如果你将授权写成如下安全性欠妥的格式:

  ```
  lucy ALL=(ALL) chown,chmod,useradd
  ```

  那么用户就有可能创建一个他自己的程序, 也命名为userad, 然后放在它的本地路径中, 如此一来他就能够使用root来执行这个"名为useradd的程序"。这是相当危险的!

  命令的绝对路径可通过`which`指令查看到: 比如`which useradd`可以查看到命令`useradd`的绝对路径: `/usr/sbin/useradd`

  ### 公式还要扩充

  例子1:

  ```bash
  papi ALL=(root) NOPASSWD: /bin/chown,/usr/sbin/useradd
  ```

  表示: 用户papi能在所有可能出现的主机上, 提权到root下执行/bin/chown, 不必输入密码; 但运行/usr/sbin/useradd 命令时需要密码.

  这是因为`NOPASSWD:`只影响了其后的第一个命令: 命令1.

  上面给出的公式只是简化版，完整的公式如下:

  ```bash
  授权用户/组 主机=[(切换到哪些用户或组)] [是否需要输入密码验证] 命令1, [(字段3)] [字段4] 命令2, ...
  ```

  在具有sudo操作的用户下, 执行`sudo -l`可以查看到该用户被允许和被禁止运行的命令.

  ### 通配符和取消命令

  例子2:

  ```
  papi ALL=/usr/sbin/*,/sbin/*,!/usr/sbin/fdisk
  ```

  用例子2来说明通配符`*`的用法, 以及命令前面加上`!`号表示取消该命令。

  该例子的意思是: 用户papi在所有可能出现的主机上, 能够运行目录/usr/sbin和/sbin下所有的程序, 但fdisk除外.

  ### 开始编辑

  “你讲了这么多,但是在实践中,我去编辑/etc/sudoers文件，系统提示我没权限啊，怎么办?”

  这是因为`/etc/sudoers`的内容如此敏感，以至于该文件是只读的。所以，编辑该文件前，请确认清楚你知道自己正在做什么。

  强烈建议通过`visudo`命令来修改该文件，通过`visudo`修改，如果配置出错，会有提示。

  不过，系统文档推荐的做法，不是直接修改`/etc/sudoers`文件，而是将修改写在`/etc/sudoers.d/`目录下的文件中。

  如果使用这种方式修改sudoers，需要在`/etc/sudoers`文件的最后行，加上`#includedir /etc/sudoers.d`一行(默认已有):

  ```
  #includedir /etc/sudoers.d
  ```

  注意了，这里的指令`#includedir`是一个整体, 前面的`#`号不能丢，并非注释，也不能在`#`号后有空格。

  任何在`/etc/sudoers.d/`目录下，不以`~`号结尾的文件和不包含`.`号的文件，都会被解析成`/etc/sudoers`的内容。

  文档中是这么说的:

  ```bash
  # This will cause sudo to read and parse any files in the /etc/sudoers.d
  # directory that do not end in '~' or contain a '.' character.
  
  # Note that there must be at least one file in the sudoers.d directory (this
  # one will do), and all files in this directory should be mode 0440.
  
  # Note also, that because sudoers contents can vary widely, no attempt is
  # made to add this directive to existing sudoers files on upgrade.
  
  # Finally, please note that using the visudo command is the recommended way
  # to update sudoers content, since it protects against many failure modes.
  ```

  ### 其他小知识

  #### 输入密码时有反馈

  当使用sudo后输入密码，并不会显示任何东西 —— 甚至连常规的星号都没有。有个办法可以解决该问题。

  打开`/etc/sudoers`文件找到下述一行:

  ```bash
  Defaults env_reset
  ```

  修改成:

  ```bash
  Defaults        env_reset,pwfeedback
  ```

  #### 修改sudo会话时间

  如果你经常使用sudo 命令，你肯定注意到过当你成功输入一次密码后，可以不用再输入密码就可以运行几次sudo命令。
  但是一段时间后，sudo 命令会再次要求你输入密码。默认是15分钟，该时间可以调整。添加`timestamp_timeout=分钟数`即可。
  时间以分钟为单位，-1表示永不过期，但强烈不推荐。

  比如我希望将时间延长到1小时，还是打开`/etc/sudoers`文件找到下述一行:

  ```bash
  Defaults env_reset
  ```

  修改成:

  ```bash
  Defaults        env_reset,pwfeedback,timestamp_timeout=60
  ```

