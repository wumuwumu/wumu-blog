---
title: jvm回收算法
date: 2020-12-14 12:00:00
---

## 定位垃圾

## 引用计数算法

每个对象有一个计数器，每次有一个引用就添加1，清除一个引用就删除1。

## 根可达算法

局部变量

静态变量

常量池

JNI引用

# 垃圾清除算法

## 标记清除算法

位置不连续，垃圾碎片

## 拷贝算法（复制算法）

浪费空间

## 标记压缩算法

效率偏低

## JVM分代回收

新的垃圾回收期不在分代了。

1、new、young:

- 存活数量少
- 使用复制算法效率高

1. 新生代=Eden+2个suvivor区 YGC回收之后大部分的对象会被回收。

2. YGC回收之后，先放到eden区，回收之后就放到s0区域。

3. 再次YGC,活着的对象eden+s0 -> s1

4. 年龄足够进入old区
5. s区装不下直接进入老年代

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glnb9qrxeuj30p50btk14.jpg)

2、old:

1. FGC :full gc
2. 顽固分子
3. 老年代满了FGC

3、permanent(1.7 永久代)/Metaspace(1.8 元数据区)：

- 永久代 元数据-class
- 永久代必须指定大小限制，元数据可以设置，也可以不设置，无上限（受限于物理内存）
- 字符串常量， 1.7-永久代，1.8- 堆

4、GC Tuning(Generation)

1. 尽量减少FGC
2. MinorGC = YGC 
3. MajorGC = FGC

# 常见的垃圾回收器

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glog8gi2mij30oe0bx141.jpg)



1、Serial:串行回收，用于年轻代

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glogbm81npj30nb0bg12g.jpg)

2、Parallel Scavenage：并行回收，年轻代

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glogfjddfrj30lf0abth0.jpg)

3、ParNew:配合cms的并行回收，年轻代

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gloggmpg66j30k60b8n5w.jpg)

4、SerialOld：老年代

5、ParallelOld:老年代

6、ConcurrentMarkSweep:老年代，垃圾回收和应用同时运行，降低STW的时间



7、G1(10ms)

8、ZGC（1ms）

9、Shenandoah

10、Eplison

1.8默认的垃圾回收器：PS+ParallelOld

## JVM 调优

1、了解生产环境下的垃圾回收器组合

JVM的命令行参数参考：

- JVM命令参数分类：

  - 标准命令：-开头，所有的HotSpoot都支持
  - 非标准：-X开头，特定版本HotSpot支持的特定命令
  - 不稳定：-XX开头，下一个版本取消，

​       ` -XX:+PrintCommandLineFlags`

​       `-XX:+PrintFlagsFinal`最终参数值

​		`-XX+PrintFlagsInitial`默认参数



# 总结

![](http://wumu.rescreate.cn/image20201216232212.png)

## 垃圾回收器

![](https://tva1.sinaimg.cn/large/0081Kckwgy1glog8gi2mij30oe0bx141.jpg)

1. 垃圾回收的发展路线，是随着内存越来越大的过程演进

   从分代算法演变到不分代算法

   Serial算法 几十兆

   Parallel算法  几十G

   CMS 几十G 承上启下 开始并发回收-- 

   > 三色标记 标错 increament update -remark   + 写屏障

   

   G1 上百G内存-逻辑分代，物理不分代

   > 三色标记+STAB  + 写屏障

   ZGC-Shenandoah -- 4T 逻辑物理都不分代

   > ColoredPointers（颜色指针 着色指针）

   Epsilon 什么都不干，调试，确认不用GC参与就能干完活

2. JDK诞生Serial追随，提高效率，诞生了ps,为了配合CMS，诞生了PN,CMS是1.4后期引入，CMS是里程碑的GC,但是CMS的问题很多，没有哪一款JDK默认使用CMS,并发垃圾回收是因为无法忍受STW.

## 调优的基础概念

1. 吞吐量：用户代码时间（用户代码执行时间+垃圾回收时间）
2. 响应时间：STW阅读，