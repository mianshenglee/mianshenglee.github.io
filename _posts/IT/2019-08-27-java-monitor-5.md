---
layout: post
title: java应用监测(5)-可视化监测工具
category: IT
tags: java troubleshooting monitor jvisualvm jconsole
keywords: 
description: 
---

tags: java, troubleshooting, monitor,jvisualvm,jconsole

---

> 一句话概括：jdk本身自带的监控工具jconsole和jvisualvm可以更方便，更直观地对java应用进行性能监测，下文为你讲解如何使用它们。

# 1 引言
前面几篇文章(见下文“相关阅读”)已经对jdk的命令行工具进行了介绍，但它们使用起来相对还是不够直观，而且一般都需要在本机上使用，有没有更方便，更直观的方式来对java应用进行监测？其实，jdk本身已经提供了java监测的GUI工具，分别是`jconsole`和`jvisualvm`，下面对这两款工具的功能及使用进行描述。

# 2 jconsole使用

jconsole是jdk一个内置Java性能分析器，在JDK安装目录下的bin目录，在windows下，可以从命令行(`jconsole.exe`)或直接双击jconsole.exe启动运行。读者有兴趣可以参考jconsole工具的[官方使用文档](https://docs.oracle.com/javase/8/docs/technotes/guides/management/jconsole.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/management/jconsole.html`

## 2.1 两种连接方式

jconsole启动时，会提供两种连接方式，分别是连接本地进程和连接远程进程。它会直接列出本地java进程来选择。若是需要监测远程的java进程，则勾选远程进程，然后输入<hostname>:<port>。这需要远程的java进程启动时，设置JMX的远程连接参数，否则是无法连接的，关于远程连接的JMX技术，可以参考[官方文章](https://docs.oracle.com/javase/8/docs/technotes/guides/management/agent.html)（`https://docs.oracle.com/javase/8/docs/technotes/guides/management/agent.html`）参数分别是以下几个：

```shell
# 需要监控的服务器IP
-Djava.rmi.server.hostname=192.168.222.10

# 提供监控的java进程端口
-Dcom.sun.management.jmxremote.port=9004 

# 指定后续的通讯端口，与上面一致
-Dcom.sun.management.jmxremote.rmi.port=9004 

# 不使用ssl登录，若有安全需求，可设置
-Dcom.sun.management.jmxremote.ssl=false 

# 不验证，若有安全需求，可设置
-Dcom.sun.management.jmxremote.authenticate=false 

```

如下图所示：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g665fimoa9j20av0ast91.jpg)

注意，由于不使用`ssl`，会提示"不安全连接"，点击它即可。

## 2.2 jconsole功能使用

启动并连接java进程后，jconsole的界面是比较简洁的，分为6个模块：

- 概述:对java进程的总体概览，包括堆，线程数，类，CPU占用率的变化折线图。
- 内存:显示堆及非堆的内存使用信息，类似jmap和jstat
- 线程:显示线程使用信息，类似jstack
- 类:显示类装载信息
- VM摘要:显示JVM信息，类似jinfo
- MBeans:显示MBeans信息（用得比较少）

### 2.2.1 概览

概览主要显示堆，线程数，类，CPU占用率的变化折线图，基本上可以直接根据折线图来查看应用概况。如果堆占用内存很高，活动线程数很多，CPU占用率很高，那就可以直接进入到相应的区域看详细内容来查找原因了。另外，右击对应的图，可以把数据导出到csv文件来分析。如图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g665z47m52j20on0ja0tn.jpg)

### 2.2.2 内存

内存是我们监测的重点区域，可以参看堆内存，非堆内存，内存池的状况，GC次数和时间，可以手动进行GC查看内存变化。如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g6666uuv0wj20oj0j7wf7.jpg)

其中，图的上方可以选择查看哪个内存的变化（堆、非堆，old区，eden区，survivor区，metaspace区等），也可以手动执行GC查看变化情况。图下方有显示内存的大小及使用大小，GC的次数和使用时间，同时以柱状图的方式来显示堆和非堆的变化。因此，对于有内存溢出，OOM这些问题，看这里的监测数据非常适合。

### 2.2.3 线程

线上应用，线程长时间停顿的主要原因主要有：等待外部资源（数据库连接、网络资源、设备资 
源等）、死循环、锁等待（活锁和死锁），这些都需要监测java应用的线程运行状况，如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g666a3znbfj20oa0j3q3i.jpg)

线程数变化情况及点击某个线程查看运行状态来分析，其中还可以点击“检测死锁”功能来处理死锁问题。

### 2.2.4 类

此功能主要用于查看加载的类总数，如果加载的类一直在增加，就得查看代码是否有不断产生类的代码了。如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g666fo8za7j20oo0jd3yq.jpg)

### 2.2.5 VM概要

当我们对java应用添加了启动参数（ `JAVA_OPTS`），若想在线上查看此应用的实际使用情况，参数是否生效，命令行工具我们是用jinfo，现在在这里可以直接看到，而且包括了系统信息，类信息，堆信息及相关的VM参数。如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g666j8gtr0j20oi0j3q3i.jpg)


# 3 jvisualvm使用

与`jconsole`类似，jdk在bin目录下还提供了`jvisualvm`工具，相对来说，`jvisualvm`更为强大，在windows下，可以从命令行(`jvisualvm.exe`)或直接双击`jvisualvm.exe`启动运行。读者有兴趣可以参考`jvisualvm`工具的[官方使用文档](https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html`

## 3.1 两种连接方式

跟`jconsole`一样，`jvisualvm`可以监测本地的java进程，也可以监测远程的java进程。本地进程直接选择即可，远程进程同样需要java进程添加JMX启动参数。

## 3.2 jvisualvm功能使用

jvisualvm的功能比较强大，主要包括以下几项功能：

- 显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）。
- 监视应用程序的CPU、GC、堆、方法区(1.7及以前)、元空间（JDK1.8及以后）以及线程的信息，相当于jmap,jstat,jstack。
- dump以及分析堆转储快照（jmap,jhat）。
- 方法级的程序运行性能分析，找出被调用最多、运行时间最长的方法。
- 离线程序快照：收集程序的运行时配置、线程dump、内存dump等信息建立一个快照

界面上的功能主要分为几大模块，分别是概述、监视、线程、抽样器。

### 3.2.1 概述

概述相当是java命令行工具中的`jps`及`jinfo`，可以显示进程以及进程的配置、系统属性，启动参数等，与`jconsole`的"VM概要"差不多。如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g6671o86p2j20tg0coaak.jpg)

### 3.2.2 监视

此功能相当于`jconsole`的"概览"功能，同样是以图形化的方式，显示CPU、堆变化、线程数变化及加载类情况，但它有一个功能是可以远程dump出堆转储快照（相当于`jmap -dump:file=./heap.hprof PID`），dump时会选择文件存储位置。

![](http://ww1.sinaimg.cn/large/72d660a7gy1g667bfgengj20mc0g2dgh.jpg)

dump出的堆快照，我们可以手动把文件下载下来，然后使用它的“装入快照”功能加载到`jvisualvm`（装入时需要选择文件类型是以"hprof"类型），进一步分析堆的内存情况。装入后，会包含概要信息，类和实例占用内存情况，双击类还可以看到具体的实例数，若是发现某些类的实例数很多，或者占用的内存大小比较高，则可以知道问题所在。如下所示：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g667fregrij20mm0fymxs.jpg)

### 3.2.3 线程

此此功能相当于`jconsole`的"线程"功能，但更丰富，它把每个线程的运行状态，运行时间都以图形化的方式显示，同时还可以进行远程线程dump，这个功能，其实就是`jstack -l`功能，dump出来后，直接显示到界面中。如下：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g667nilhd4j20mw0edt8z.jpg)

### 3.2.4 抽样器

抽样器是`jvisualvm`的独有功能，可以对CPU和内存进行抽样显示，每隔一段时间把内存信息，线程信息，可以很方便的集中精力分析某一段时间的数据变化（可以以层级方式显示细到方法的执行时间，类占用内存情况等），同时也提供执行GC、内存dump及线程dump功能。如下：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g667u8t9vuj20ms0g9q3l.jpg)


# 4 总结

有了`jconsole`和`jvisualvm`两款可视化工具，可以减少命令行的输入，以更方便，更直观的方式来监测java应用的内存、线程、CPU等信息，是处理java线上问题的好帮手。不过要注意一点的是，使用这两款工具之前，我们还是需要对JVM，Java程序运行机制，线程等知识有一定的积累。希望本文对大家有帮助。


# 参数资料

- [jconsole官方使用文档](https://docs.oracle.com/javase/8/docs/technotes/guides/management/jconsole.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/management/jconsole.html`
- [jvisualvm官方使用文档](https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/visualvm/index.html`
- [jvisualvm文档](https://visualvm.github.io/documentation.html)： `https://visualvm.github.io/documentation.html`
- [JMX远程监控技术](https://docs.oracle.com/javase/8/docs/technotes/guides/management/agent.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/management/agent.html`
- [示例代码地址](https://github.com/mianshenglee/my-example/tree/master/java-monitor-example)：`https://github.com/mianshenglee/my-example/tree/master/java-monitor-example`

# 相关阅读

- [java应用监测(1)-java程序员应该知道的应用监测技术](https://mianshenglee.github.io/2019/08/23/java-monitor-1.html):`https://mianshenglee.github.io/2019/08/23/java-monitor-1.html`
- [java应用监测(2)-java命令的秘密](https://mianshenglee.github.io/2019/08/24/java-monitor-2.html):`https://mianshenglee.github.io/2019/08/24/java-monitor-2.html`
- [java应用监测(3)-这些命令行工具你掌握了吗](https://mianshenglee.github.io/2019/08/25/java-monitor-3.html)：`https://mianshenglee.github.io/2019/08/25/java-monitor-3.html`
- [java应用监测(4)-线上问题排查套路](https://mianshenglee.github.io/2019/08/26/java-monitor-4.html)：`https://mianshenglee.github.io/2019/08/26/java-monitor-4.html`
