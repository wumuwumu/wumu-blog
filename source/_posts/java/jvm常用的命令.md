---
title: jvm常用的命令
abbrlink: f6980397
date: 2020-12-20 12:00:00
---

# jvm基本命令

## jps

> 显示当前运行的java进程以及相关参数

jps参数：

```kotlin
jps -l pid

-q 只显示pid，不显示class名称,jar文件名和传递给main 方法的参数。
-l 输出应用程序main class的完整package名 或者 应用程序的jar文件完整路径名。
-m 输出传递给main方法的参数
-v 输出传递给JVM的参数
```

**备注：**也可以使用ps aux | grep 项目名 查看pid

## jstack

> 用于生成java虚拟机当前时刻的线程快照。

### 分析CPU利用率100%问题

1. top 查看占CPU最多的进程
2. top -Hp pid 查询进程下所有线程的运行情况（shift+p 按cpu排序，shift+m 按内存排序）
3. 用printf ‘%x’ pid 转换为16进制（加入查到的是a）
4. jstact查看线程快照，jstack 30316 | grep -A 20 a

[推荐阅读][http://jameswxx.iteye.com/blog/1041173](https://link.jianshu.com?t=http://jameswxx.iteye.com/blog/1041173)

## 死锁分析

java程序如下：

```java

public class JvmLock {
  public static Object obj1 = new Object();
  public static Object obj2 = new Object();

  public static void main(String[] args) {
    System.out.println("Default Charset=" + Charset.defaultCharset());
    System.out.println("file.encoding=" + System.getProperty("file.encoding"));

    LockA a = new LockA();
    LockB b = new LockB();
    new Thread(a).start();
    new Thread(b).start();
  }


}

class LockA implements Runnable {

  @Override
  public void run() {
    try {
      synchronized (JvmLock.obj1) {
        System.out.println("lockA 获取到obj1");

        Thread.sleep(1000);

        synchronized (JvmLock.obj2) {
          System.out.println("lockA 获取obj2");
          Thread.sleep(60 * 1000);
        }
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
}


class LockB implements Runnable {

  @Override
  public void run() {
    try {
      synchronized (JvmLock.obj2) {
        System.out.println("lockb 获取到obj2");
        Thread.sleep(1000);
        synchronized (JvmLock.obj1) {
          System.out.println("lockA 获取obj1");
          Thread.sleep(60 * 1000);
        }
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
}
```

获取到的堆栈信息，直接可以查看到死锁的存在

```bash

"DestroyJavaVM" #15 prio=5 os_prio=0 cpu=453.13ms elapsed=6111.89s tid=0x000002873420d800 nid=0x49c waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"VM Thread" os_prio=2 cpu=15.63ms elapsed=6112.33s tid=0x00000287582ba800 nid=0x2850 runnable

"GC Thread#0" os_prio=2 cpu=125.00ms elapsed=6112.35s tid=0x0000028734225800 nid=0x1a3c runnable

"G1 Main Marker" os_prio=2 cpu=15.63ms elapsed=6112.35s tid=0x0000028734296000 nid=0x4b0 runnable

"G1 Conc#0" os_prio=2 cpu=15.63ms elapsed=6112.35s tid=0x0000028734297000 nid=0x32a4 runnable

"G1 Refine#0" os_prio=2 cpu=0.00ms elapsed=6112.33s tid=0x00000287572b2000 nid=0x2ef8 runnable

"G1 Young RemSet Sampling" os_prio=2 cpu=203.13ms elapsed=6112.33s tid=0x00000287572b5000 nid=0x2ddc runnable
"VM Periodic Task Thread" os_prio=2 cpu=578.13ms elapsed=6112.27s tid=0x0000028758710800 nid=0x192c waiting on condition

JNI global refs: 9, weak refs: 0


Found one Java-level deadlock:
=============================
"Thread-0":
  waiting to lock monitor 0x00000287582e6280 (object 0x0000000711c39548, a java.lang.Object),
  which is held by "Thread-1"
"Thread-1":
  waiting to lock monitor 0x00000287582e6080 (object 0x0000000711c39538, a java.lang.Object),
  which is held by "Thread-0"

Java stack information for the threads listed above:
===================================================
"Thread-0":
        at top.sciento.wumu.jvm.LockA.run(JvmLock.java:35)
        - waiting to lock <0x0000000711c39548> (a java.lang.Object)
        - locked <0x0000000711c39538> (a java.lang.Object)
        at java.lang.Thread.run(java.base@11.0.2/Thread.java:834)
"Thread-1":
        at top.sciento.wumu.jvm.LockB.run(JvmLock.java:55)
        - waiting to lock <0x0000000711c39538> (a java.lang.Object)
        - locked <0x0000000711c39548> (a java.lang.Object)
        at java.lang.Thread.run(java.base@11.0.2/Thread.java:834)

Found 1 deadlock.
```

## jmap

> 用于打印指定Java进程(或核心文件、远程调试服务器)的共享对象内存映射或堆内存细节。

> 堆Dump是反应Java堆使用情况的内存镜像，其中主要包括系统信息、虚拟机属性、完整的线程Dump、所有类和对象的状态等。 一般，在内存不足、GC异常等情况下，我们就会怀疑有内存泄露。这个时候我们就可以制作堆Dump来查看具体情况。分析原因。

1. 查看java堆（heap）中的对象数量及大小：jmap -histo 31846
2. 将内存使用的详细情况输出到文件： jmap -dump:format=b,file=heapDump pid然后使用jhat -port 5000 heapDump在浏览器中访问：[http://localhost:5000/](https://link.jianshu.com?t=http://localhost:5000/)查看详细信息

## jinfo

> jinfo可以输出java进程、core文件或远程debug服务器的配置信息。可以使用jps -v替换

## jstat

> 是用于监控虚拟机各种运行状态信息的命令行工具。他可以显示本地或远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
 参数解释：

Option — 选项，我们一般使用 -gcutil 查看gc情况

vmid — VM的进程号，即当前运行的java进程号

interval– 间隔时间，单位为秒或者毫秒

count — 打印次数，如果缺省则打印无数次

例子：jstat -gc 5828 250 5

如下所示为jstat的命令格式

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]1
```

如下表示分析进程id为31736 的gc情况，每隔1000ms打印一次记录，打印10次停止，每3行后打印指标头部

```
jstat -gc -h3 31736 1000 101
```

###   jstat -gc

```
jstat -gc xxxx1
```

其对应的指标含义如下：

| 参数 | 描述                                                  |
| ---- | ----------------------------------------------------- |
| S0C  | 年轻代中第一个survivor（幸存区）的容量 (字节)         |
| S1C  | 年轻代中第二个survivor（幸存区）的容量 (字节)         |
| S0U  | 年轻代中第一个survivor（幸存区）目前已使用空间 (字节) |
| S1U  | 年轻代中第二个survivor（幸存区）目前已使用空间 (字节) |
| EC   | 年轻代中Eden（伊甸园）的容量 (字节)                   |
| EU   | 年轻代中Eden（伊甸园）目前已使用空间 (字节)           |
| OC   | Old代的容量 (字节)                                    |
| OU   | Old代目前已使用空间 (字节)                            |
| PC   | Perm(持久代)的容量 (字节)                             |
| PU   | Perm(持久代)目前已使用空间 (字节)                     |
| YGC  | 从应用程序启动到采样时年轻代中gc次数                  |
| YGCT | 从应用程序启动到采样时年轻代中gc所用时间(s)           |
| FGC  | 从应用程序启动到采样时old代(全gc)gc次数               |
| FGCT | 从应用程序启动到采样时old代(全gc)gc所用时间(s)        |
| GCT  | 从应用程序启动到采样时gc用的总时间(s)                 |

### jstat -gcutil

查看gc的统计信息

```
jstat -gcutil xxxx1
```

其对应的指标含义如下：

| 参数 | 描述                                                     |
| ---- | -------------------------------------------------------- |
| S0   | 年轻代中第一个survivor（幸存区）已使用的占当前容量百分比 |
| S1   | 年轻代中第二个survivor（幸存区）已使用的占当前容量百分比 |
| E    | 年轻代中Eden（伊甸园）已使用的占当前容量百分比           |
| O    | old代已使用的占当前容量百分比                            |
| P    | perm代已使用的占当前容量百分比                           |
| YGC  | 从应用程序启动到采样时年轻代中gc次数                     |
| YGCT | 从应用程序启动到采样时年轻代中gc所用时间(s)              |
| FGC  | 从应用程序启动到采样时old代(全gc)gc次数                  |
| FGCT | 从应用程序启动到采样时old代(全gc)gc所用时间(s)           |
| GCT  | 从应用程序启动到采样时gc用的总时间(s)                    |

### jstat -gccapacity

```
jstat -gccapacity xxxx1
```

其对应的指标含义如下：

| 参数  | 描述                                          |
| ----- | --------------------------------------------- |
| NGCMN | 年轻代(young)中初始化(最小)的大小 (字节)      |
| NGCMX | 年轻代(young)的最大容量 (字节)                |
| NGC   | 年轻代(young)中当前的容量 (字节)              |
| S0C   | 年轻代中第一个survivor（幸存区）的容量 (字节) |
| S1C   | 年轻代中第二个survivor（幸存区）的容量 (字节) |
| EC    | 年轻代中Eden（伊甸园）的容量 (字节)           |
| OGCMN | old代中初始化(最小)的大小 (字节)              |
| OGCMX | old代的最大容量 (字节)                        |
| OGC   | old代当前新生成的容量 (字节)                  |
| OC    | Old代的容量 (字节)                            |
| PGCMN | perm代中初始化(最小)的大小 (字节)             |
| PGCMX | perm代的最大容量 (字节)                       |
| PGC   | perm代当前新生成的容量 (字节)                 |
| PC    | Perm(持久代)的容量 (字节)                     |
| YGC   | 从应用程序启动到采样时年轻代中gc次数          |
| FGC   | 从应用程序启动到采样时old代(全gc)gc次数       |

**4 其他命令**

1) 查看年轻代对象的信息及其占用量。

```bash
jstat -gcnewcapacity xxxx1
```

2) 查看老年代对象的信息及其占用量。

```bash
jstat -gcoldcapacity xxxx1
```

3) 查看年轻代对象的信息

```bash
jstat -gcnew xxxx1
```

4) 查看老年代对象的信息

```bash
jstat -gcold xxxx
```

## javap

> 可以对代码反编译，也可以查看java编译器生成的字节码。

## jhsdb(java9以上)

```bash
 # jhsdb
    clhsdb       	command line debugger
    debugd       	debug server
    hsdb         	ui debugger
    jstack --help	to get more information
    jmap   --help	to get more information
    jinfo  --help	to get more information
    jsnap  --help	to get more information

```

- jhsdb是java9引入的，可以在JAVA_HOME/bin目录下找到jhsdb；它取代了jdk9之前的JAVA_HOME/lib/sa-jdi.jar
- jhsdb有clhsdb、debugd、hsdb、jstack、jmap、jinfo、jsnap这些mode可以使用
- 其中hsdb为ui debugger，就是jdk9之前的sun.jvm.hotspot.HSDB；而clhsdb即为jdk9之前的sun.jvm.hotspot.CLHSDB

### jhsdb jstack

```bash
 # jhsdb jstack --help
    --locks	to print java.util.concurrent locks
    --mixed	to print both java and native frames (mixed mode)
    --exe	executable image name
    --core	path to coredump
    --pid	pid of process to attach
```

> --pid用于指定JVM的进程ID；--exe用于指定可执行文件；--core用于指定core dump文件

### 异常

```bash
jhsdb jstack --mixed --pid 1
//......
Caused by: sun.jvm.hotspot.debugger.DebuggerException: get_thread_regs failed for a lwp
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.getThreadIntegerRegisterSet0(Native Method)
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$1GetThreadIntegerRegisterSetTask.doit(LinuxDebuggerLocal.java:534)
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.run(LinuxDebuggerLocal.java:151)
复制代码
```

> 如果出现这个异常表示是采用jdk版本的问题，可以尝试一下其他jdk编译版本

### debugger

```bash
# jhsdb jstack --locks --pid 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 12+33
Deadlock Detection:

No deadlocks found.

"DestroyJavaVM" #32 prio=5 tid=0x000055c3b5be0800 nid=0x6 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   JavaThread state: _thread_blocked

Locked ownable synchronizers:
    - None

"http-nio-8080-Acceptor-0" #30 daemon prio=5 tid=0x000055c3b5d71800 nid=0x2f runnable [0x00007fa0d13de000]
   java.lang.Thread.State: RUNNABLE
   JavaThread state: _thread_in_native
 - sun.nio.ch.ServerSocketChannelImpl.accept0(java.io.FileDescriptor, java.io.FileDescriptor, java.net.InetSocketAddress[]) @bci=0 (Interpreted frame)
 - sun.nio.ch.ServerSocketChannelImpl.accept(java.io.FileDescriptor, java.io.FileDescriptor, java.net.InetSocketAddress[]) @bci=4, line=525 (Interpreted frame)
 - sun.nio.ch.ServerSocketChannelImpl.accept() @bci=41, line=277 (Interpreted frame)
 - org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept() @bci=4, line=448 (Interpreted frame)
 - org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept() @bci=1, line=70 (Interpreted frame)
 - org.apache.tomcat.util.net.Acceptor.run() @bci=98, line=95 (Interpreted frame)
 - java.lang.Thread.run() @bci=11, line=835 (Interpreted frame)

Locked ownable synchronizers:
    - <0x00000000e3aab6e0>, (a java/util/concurrent/locks/ReentrantLock$NonfairSync)

"http-nio-8080-ClientPoller-0" #29 daemon prio=5 tid=0x000055c3b5c20000 nid=0x2e runnable [0x00007fa0d14df000]
   java.lang.Thread.State: RUNNABLE
   JavaThread state: _thread_in_native
 - sun.nio.ch.EPoll.wait(int, long, int, int) @bci=0 (Interpreted frame)
 - sun.nio.ch.EPollSelectorImpl.doSelect(java.util.function.Consumer, long) @bci=96, line=120 (Interpreted frame)
 - sun.nio.ch.SelectorImpl.lockAndDoSelect(java.util.function.Consumer, long) @bci=42, line=124 (Interpreted frame)
	- locked <0x00000000e392ece8> (a sun.nio.ch.EPollSelectorImpl)
	- locked <0x00000000e392ee38> (a sun.nio.ch.Util$2)
 - sun.nio.ch.SelectorImpl.select(long) @bci=31, line=136 (Interpreted frame)
 - org.apache.tomcat.util.net.NioEndpoint$Poller.run() @bci=55, line=743 (Interpreted frame)
 - java.lang.Thread.run() @bci=11, line=835 (Interpreted frame)

Locked ownable synchronizers:
    - None

"http-nio-8080-exec-10" #28 daemon prio=5 tid=0x000055c3b48d6000 nid=0x2d waiting on condition [0x00007fa0d15e0000]
   java.lang.Thread.State: WAITING (parking)
   JavaThread state: _thread_blocked
 - jdk.internal.misc.Unsafe.park(boolean, long) @bci=0 (Interpreted frame)
	- parking to wait for <0x00000000e3901670> (a java/util/concurrent/locks/AbstractQueuedSynchronizer$ConditionObject)
 - java.util.concurrent.locks.LockSupport.park(java.lang.Object) @bci=14, line=194 (Interpreted frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await() @bci=42, line=2081 (Interpreted frame)
 - java.util.concurrent.LinkedBlockingQueue.take() @bci=27, line=433 (Interpreted frame)
 - org.apache.tomcat.util.threads.TaskQueue.take() @bci=36, line=107 (Interpreted frame)
 - org.apache.tomcat.util.threads.TaskQueue.take() @bci=1, line=33 (Interpreted frame)
 - java.util.concurrent.ThreadPoolExecutor.getTask() @bci=147, line=1054 (Interpreted frame)
 - java.util.concurrent.ThreadPoolExecutor.runWorker(java.util.concurrent.ThreadPoolExecutor$Worker) @bci=26, line=1114 (Interpreted frame)
 - java.util.concurrent.ThreadPoolExecutor$Worker.run() @bci=5, line=628 (Interpreted frame)
 - org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run() @bci=4, line=61 (Interpreted frame)
 - java.lang.Thread.run() @bci=11, line=835 (Interpreted frame)
 //......

/ # jhsdb jstack --mixed --pid 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 12+33
Deadlock Detection:

No deadlocks found.

----------------- 47 -----------------
"http-nio-8080-Acceptor-0" #30 daemon prio=5 tid=0x000055c3b5d71800 nid=0x2f runnable [0x00007fa0d13de000]
   java.lang.Thread.State: RUNNABLE
   JavaThread state: _thread_in_native
0x00007fa0ee0923ad		????????
----------------- 46 -----------------
"http-nio-8080-ClientPoller-0" #29 daemon prio=5 tid=0x000055c3b5c20000 nid=0x2e runnable [0x00007fa0d14df000]
   java.lang.Thread.State: RUNNABLE
   JavaThread state: _thread_in_native
0x00007fa0ee05f3d0	epoll_pwait + 0x1d
0x00007fa0daa97810	* sun.nio.ch.EPoll.wait(int, long, int, int) bci:0 (Interpreted frame)
0x00007fa0daa91680	* sun.nio.ch.EPollSelectorImpl.doSelect(java.util.function.Consumer, long) bci:96 line:120 (Interpreted frame)
0x00007fa0db85f57c	* sun.nio.ch.SelectorImpl.lockAndDoSelect(java.util.function.Consumer, long) bci:42 line:124 (Compiled frame)
* sun.nio.ch.SelectorImpl.select(long) bci:31 line:136 (Compiled frame)
* org.apache.tomcat.util.net.NioEndpoint$Poller.run() bci:55 line:743 (Interpreted frame)
0x00007fa0daa91c88	* java.lang.Thread.run() bci:11 line:835 (Interpreted frame)
0x00007fa0daa88849	<StubRoutines>
0x00007fa0ed122952	_ZN9JavaCalls11call_helperEP9JavaValueRK12methodHandleP17JavaCallArgumentsP6Thread + 0x3c2
0x00007fa0ed1208d0	_ZN9JavaCalls12call_virtualEP9JavaValue6HandleP5KlassP6SymbolS6_P6Thread + 0x200
0x00007fa0ed1ccfc5	_ZL12thread_entryP10JavaThreadP6Thread + 0x75
0x00007fa0ed74f3a3	_ZN10JavaThread17thread_main_innerEv + 0x103
0x00007fa0ed74c3f5	_ZN6Thread8call_runEv + 0x75
0x00007fa0ed4a477e	_ZL19thread_native_entryP6Thread + 0xee
//......

```

> --locks或者--mixed花费的时间可能比较长(`几分钟，可能要将近6分钟`)，因而进程暂停的时间也可能比较长，在使用这两个选项时要注意

### jhsdb jmap

### jmap -heap pid

```bash
# jmap -heap 1
Error: -heap option used
Cannot connect to core dump or remote debug server. Use jhsdb jmap instead
```

> jdk9及以上版本使用jmap -heap pid命令查看当前heap使用情况时，发现报错，提示需要使用jhsdb jmap来替代

### jhsdb jmap pid

```bash
 # jhsdb jmap 1
sh: jhsdb: not found
```

> 发现jlink的时候没有添加jdk.hotspot.agent这个module，添加了这个module之后可以发现JAVA_HOME/bin目录下就有了jhsdb

### PTRACE_ATTACH failed

```bash
 # jhsdb jmap 1
You have to set --pid or --exe.
    <no option>	to print same info as Solaris pmap
    --heap	to print java heap summary
    --binaryheap	to dump java heap in hprof binary format
    --dumpfile	name of the dump file
    --histo	to print histogram of java object heap
    --clstats	to print class loader statistics
    --finalizerinfo	to print information on objects awaiting finalization
    --exe	executable image name
    --core	path to coredump
    --pid	pid of process to attach
/ # jhsdb jmap --heap --pid 1
Attaching to process ID 1, please wait...
ERROR: ptrace(PTRACE_ATTACH, ..) failed for 1: Operation not permitted
Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process: ptrace(PTRACE_ATTACH, ..) failed for 1: Operation not permitted
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process: ptrace(PTRACE_ATTACH, ..) failed for 1: Operation not permitted
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.execute(LinuxDebuggerLocal.java:176)
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach(LinuxDebuggerLocal.java:336)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.attachDebugger(HotSpotAgent.java:672)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.setupDebuggerLinux(HotSpotAgent.java:612)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.setupDebugger(HotSpotAgent.java:338)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:305)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:141)
	at jdk.hotspot.agent/sun.jvm.hotspot.tools.Tool.start(Tool.java:185)
	at jdk.hotspot.agent/sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
	at jdk.hotspot.agent/sun.jvm.hotspot.tools.JMap.main(JMap.java:176)
	at jdk.hotspot.agent/sun.jvm.hotspot.SALauncher.runJMAP(SALauncher.java:326)
	at jdk.hotspot.agent/sun.jvm.hotspot.SALauncher.main(SALauncher.java:455)
Caused by: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process: ptrace(PTRACE_ATTACH, ..) failed for 1: Operation not permitted
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach0(Native Method)
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$1AttachTask.doit(LinuxDebuggerLocal.java:326)
	at jdk.hotspot.agent/sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.run(LinuxDebuggerLocal.java:151)
```

> 发现PTRACE_ATTACH被docker禁用了，需要在运行容器时启用PTRACE_ATTACH

### docker启用SYS_PTRACE

```bash
docker run --cap-add=SYS_PTRACE
```

之后就可以正常使用jhsdb如下：

```bash
# jhsdb jmap --heap --pid 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 12+33

using thread-local object allocation.
Shenandoah GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 523763712 (499.5MB)
   NewSize                  = 1363144 (1.2999954223632812MB)
   MaxNewSize               = 17592186044415 MB
   OldSize                  = 5452592 (5.1999969482421875MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   ShenandoahRegionSize     = 262144 (0.25MB)

Heap Usage:
Shenandoah Heap:
   regions   = 1997
   capacity  = 523501568 (499.25MB)
   used      = 70470552 (67.2059555053711MB)
   committed = 144441344 (137.75MB)
```

### jhsdb jinfo

```bash
 # jhsdb jinfo --help
    --flags	to print VM flags
    --sysprops	to print Java System properties
    <no option>	to print both of the above
    --exe	executable image name
    --core	path to coredump
    --pid	pid of process to attach
```

> 使用jhsdb显示jinfo的sysprops如下：

```bash
 # jhsdb jinfo --sysprops --pid 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 12+33
awt.toolkit = sun.awt.X11.XToolkit
java.specification.version = 12
sun.jnu.encoding = UTF-8
//......

```

> 这个命令其实跟jinfo -sysprops 1是等价的

### jhsdb jsnap

```
# jhsdb jsnap --pid 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 12+33
java.threads.started=27 event(s)
java.threads.live=24
java.threads.livePeak=24
java.threads.daemon=20
java.cls.loadedClasses=8250 event(s)
java.cls.unloadedClasses=1 event(s)
java.cls.sharedLoadedClasses=0 event(s)
java.cls.sharedUnloadedClasses=0 event(s)
java.ci.totalTime=18236958158 tick(s)
java.property.java.vm.specification.version=12
java.property.java.vm.specification.name=Java Virtual Machine Specification
java.property.java.vm.specification.vendor=Oracle Corporation
java.property.java.vm.version=12+33
java.property.java.vm.name=OpenJDK 64-Bit Server VM
java.property.java.vm.vendor=Azul Systems, Inc.
java.property.java.vm.info=mixed mode
java.property.jdk.debug=release
//......

```

> jhsdb jsnap的功能主要是由jdk.hotspot.agent模块中的sun.jvm.hotspot.tools.JSnap.java来提供的，它可以用于查看threads及class loading/unloading相关的event、JVM属性参数等，其中--all可以显示更多的JVM属性参数

### jhsdb与jcmd

[jhsdb: A New Tool for JDK 9](https://dzone.com/articles/jhsdb-a-new-tool-for-jdk-9)这篇文章中列出了jhsdb与jcmd的等价命令，如下图：

![img](https://user-gold-cdn.xitu.io/2019/3/27/169bdde54fd27641?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 小结

- 在java9之前，JAVA_HOME/lib目录下有个sa-jdi.jar，可以通过如上命令启动HSDB(`图形界面`)及CLHSDB(`命令行`)；sa-jdi.jar中的sa的全称为Serviceability Agent，它之前是sun公司提供的一个用于协助调试HotSpot的组件，而HSDB便是使用Serviceability Agent来实现的；HSDB就是HotSpot Debugger的简称，由于Serviceability Agent在使用的时候会先attach进程，然后暂停进程进行snapshot，最后deattach进程(`进程恢复运行`)，所以在使用HSDB时要注意
- jhsdb是java9引入的，可以在JAVA_HOME/bin目录下找到jhsdb；它取代了jdk9之前的JAVA_HOME/lib/sa-jdi.jar；jhsdb有clhsdb、debugd、hsdb、jstack、jmap、jinfo、jsnap这些mode可以使用；其中hsdb为ui debugger，就是jdk9之前的sun.jvm.hotspot.HSDB；而clhsdb即为jdk9之前的sun.jvm.hotspot.CLHSDB
- jhsdb在jdk.hotspot.agent这个模块中；对于jhsdb jstack的--locks或者--mixed命令花费的时间可能比较长(`几分钟，可能要将近6分钟`)，因而进程暂停的时间也可能比较长，在使用这两个选项时要注意；对于jdk9及以后的版本不再使用jmap -heap命令来查询heap内存情况，需要用jhsdb jmap --heap --pid来替代；使用jhsdb jmap需要在运行容器时启用PTRACE_ATTACH才可以

### doc

- [JVM信息查看](https://segmentfault.com/a/1190000004621417)
- [jhsdb](https://docs.oracle.com/en/java/javase/12/tools/jhsdb.html)
- [jdk.hotspot.agent jhsdb](https://docs.oracle.com/en/java/javase/12/docs/api/jdk.hotspot.agent/module-summary.html#jhsdb)
- [jhsdb: A New Tool for JDK 9](https://dzone.com/articles/jhsdb-a-new-tool-for-jdk-9)
- [jcmd: One JDK Command-Line Tool to Rule Them All](http://marxsoftware.blogspot.com/2016/02/jcmd-one-jdk-command-line-tool-to-rule.html)
- [JVM in Docker and PTRACE_ATTACH](https://jarekprzygodzki.wordpress.com/2016/12/19/jvm-in-docker-and-ptrace_attach/)
- [Serviceability in HotSpot](http://openjdk.java.net/groups/hotspot/docs/Serviceability.html)
- [The HotSpot™ Serviceability Agent: An out-of-process high level debugger for a Java™ virtual machine](https://www.usenix.org/legacy/events/jvm01/full_papers/russell/russell_html/index.html)



# 参考

> https://www.jianshu.com/p/bacc64527894
>
> https://juejin.cn/post/6844903808057753613