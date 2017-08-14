title: Presto内存溢出(OutOfMemory)问题排查
date: 2017-08-08
tags:
 - presto
 - OOM
 - GC
 - 大数据
categories:
 - 原创文章
---

# 问题

最近公司大数据查询引擎Presto运行非常不稳定，容易出现掉节点的问题（一些Presto节点会在莫名其妙的情况出现宕机），从日志上看到是内存溢出了。公司线上的Presto集群JVM使用的垃圾回收器是G1，这也是Presto引擎推荐的一款GC收集器，G1对于超大内存（6G+）的垃圾收集有着良好的效率，能够高效的进行垃圾收集，并控制垃圾收集导致的停顿时间。

<!-- more -->

# 追踪

在出现宕机后，发现线上运行的JVM一直没有加GC日志打印，于是果断加上GC日志参数以便下次宕机收集GC情况：

```
-server
-Xmx16G
-XX:+UseG1GC
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:OnOutOfMemoryError=kill -9 %p
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=5M
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/lib/presto/var/log/gc.log
```
GC日志以5M的大小进行滚动，保留最近5个日志文件，便于GC日志文件的保留，同时JVM最大内存设置为16G。

# 分析

在加上GC日志后不久又出现了宕机，这时收集得到了程序最后宕机时刻的GC日志文件，这里推荐一款GC日志的分析工具软件[GCViewer](https://github.com/chewiebug/GCViewer)，这款软件优于其它GC日志分析工具的地方在于它支持G1日志的分析，这是非常重要的一点。

由于GCViewer github上并没有打包好的现成品下载，所以需要自己下载后进行maven编译和打包，具体过程就不再叙述了。

通过命令行运行`java -jar ./gcviewer-1.36-SNAPSHOT.jar`就可以启动图形界面，并载入自己的GC日志文件进行分析。

{% asset_img 8C65BD39-ECDC-49BC-8687-98C527892C25.png %}

图示：
 - 粉红区域：Tenured Generation
 - 黄色区域：Young Generation
 - 蓝色线：  Used Heap
 - 紫色线：  Used Tenured Heap
 - 绿色线：  GC Lines

1、从上图可以看到老年代占用了比较多的内存，基本在2G-15G之间波动。而年轻代则内存相对来说占用较少一些。老年代内存溢出时基本上占据了绝大部分内存。同时在GC日志中也找到了`2017-08-08T09:31:11.187+0800: 47413.801: [GC pause (GCLocker Initiated GC) (young) (to-space exhausted), 0.2758880 secs]`这样的日志打印，证明确实年轻代已经找不到内存可以分配使用了

2、从图中看GC时间比较正常，都基本维持在1s以下。

3、Presto在进行任务计算的时候会统计当前的内存使用量，并保证使用的内存不会超出限制，然而这个比较正常的GC日志内存溢出了。

# 可能的原因

 - 正常使用的情况下，因为提交的任务过多，导致内存不足引发溢出
 - Presto存在内存泄漏

对于第一种情况，Presto提供了一些解决方案，比如限制单个任务的总数据量大小，单个任务每个节点的内存使用最大限制，以及可以采用Presto内置的队列排队等限制并发任务数等方式
对于第二种情况，这种情况就比较麻烦了，需要确定是具体哪里的代码引起的内存泄漏

当在排查第一种情况的时候，检查Presto server日志的时候发现了一个隐藏在正常日志深处的一个错误日志：

{% asset_img 3EB7D5CC-1626-4282-A646-252E18D57E68.png %}

拿着这个日志在Presto的GITHUB官方网站ISSUE列表上搜索，还真发现了一个和我这个错误日志一样的ISSUE: [ISSUE-5688](https://github.com/prestodb/presto/issues/5688)

根据ISSUE中的讨论初步认定该ISSUE存在内存泄漏的可能，并且在0.161版本上仍然存在，而我们线上使用的版本是0.160，恰好落在这个有问题的区间上。修复的提交是[PULL-7099](https://github.com/prestodb/presto/pull/7099)

{% asset_img D37C02A0-1A21-4809-B03C-32A03391F1C9.png %}

根据tag历史可以知道，该问题是在0.166版本及以后才被进行了修复。

这比较肯定了我们线上内存溢出是因为这个bug导致的判断。

# 总结

从目前跟踪到的情况来说，线上的内存溢出与旧版本Presto存在的内存泄漏有关。

接下来要着手处理的就是线上Presto版本的升级，这个升级我会另外再开篇文章来说明如何在Ambari管理软件中对已经存在的Presto进行升级处理。

