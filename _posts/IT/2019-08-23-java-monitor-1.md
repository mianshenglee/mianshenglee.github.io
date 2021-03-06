---
layout: post
title: java应用监测(1)-java程序员应该知道的应用监测技术
category: IT
tags: java troubleshooting monitor
keywords: 
description: 
---

tags: java, troubleshooting, monitor

---


> 一句话概括：java应用监测，为什么？监测什么？如何监测？本文为你解答。

# 1 引言：为什么需要监测java应用

java开发人员都知道，启动java应用使用的是`java`(`class`文件)或`java -jar`(`jar`或`war`包)命令。而`java`命令其实就是启动一个java虚拟机（`JVM`），程序就是运行在`JVM`上，`JVM`负责类加载，运行时区域堆栈分配等工作，而堆栈分别用于程序中的对象及线程使用，分别影响的系统的cpu及内存，如果程序涉及文件或数据读写，还会影响系统的IO。因此，一个java应用启动后，如果不对它所占用的资源情况进行监测，无疑于一架飞机起飞了，却没有仪表盘，这种飞机估计没有人敢坐。因此，作为开发人员，得清楚应用启动后的运行情况，并能够对应用运行状况进行监测，如此，才能及时预测有可能发生的问题进行及时修正或发生问题后可以更快，更好地找到原因所在，进而解决。可喜的是，程序本身，java工具和第三方工具，都为我们提供了很多监测java应用的方法，因此，作为java开发人员，有必要对它们做一个系统的了解，以便于在实际应用中（特别是在生产环境中）更从容，更有效率的处理问题。

# 2 监测什么

一个java应用运行起来，若出现问题，我们第一反应肯定是查看日志，查看一下具体是报什么错，若没有报错，只是运行很慢，或者只是暂时还没有报错，那我们需要监测什么内容呢？这一节，对比我们平时使用操作系统，来看看对于java应用，需要监测什么内容。

## 2.1 系统监测内容

对于我们平时使用的操作系统，如`windows`，`linux`系统，出现应用打开缓慢，系统响应缓慢，一般就是先查看系统的cpu运行和内存使用情况，找出占用资源高的进程，定位原因，在`windows`，直接在`任务管理器-->性能`中查看，如下图。

![](http://ww1.sinaimg.cn/large/72d660a7gy1g5zbn1nok5j20ia0fa75b.jpg)

在`linux`，可以使用`top`,`free`,`df`,`iostat`等命令进行查看，如下图。

![](http://ww1.sinaimg.cn/large/72d660a7gy1g5zbnc3hjzj20j3054jro.jpg)

在日常的运维过程中，也是对系统的磁盘使用、IO、CPU、内存、网络等情况进行监测，争取及时发现占用资源高的进程，定位问题然后处理。

## 2.2 java应用监测内容

相对于操作系统，java应用需要监测的内容也有类似的地方，如占用的cpu情况，内存情况，其中，会包含java应用的JVM参数，堆占用情况，线程运行状态，类加载情况，垃圾回收(`GC`)情况。至于为什么需要监测这些内容，就需要对java的虚拟机(JVM)有一定的了解，由于本文JVM涉及内容较多，后面有机会再对JVM进行详细讲解，这里可以对java的JVM体系作一个简要介绍，以便于读者对后面章节的理解。见下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g5zd3kupvuj20qo0k0wf3.jpg)

- 当java运行一个应用，就会生成一个JVM的实例，而java应用则运行于此JVM实例中，当应用退出，JVM实例也会关闭。启动多个java应用，也会启动多个JVM实例，它们不会相互影响(当然，它们都会占用系统的资源)。
- 虚拟机主要有三大模块，一个类加载子系统（`Class Loader Subsystem`,负责加载类），一个执行引擎（`Execution Engine`,负责执行类的方法指令和垃圾回收），一个运行时数据区（`Runtime Data Areas`,负责存放程序运行时的数据）。
- 其中运行时数据区分为方法区（存储如类信息，方法信息，引用，常量池等），堆（存储类实例对象和数组），java栈（以栈方式存放以帧为单位保存线程的运行状态帧），本地方法栈（跟本地方法相关的数据区），程序计数器（每一个线程都有它自己的程序计数器，表示下一条将被执行指令的“地址”）。
- java应用启动流程就是通过类加载子系统加载相关的类，然后把相关数据如类信息，方法等存到方法区的栈中，实例化相关的类，同时把实例对象存储在堆中，程序运行位置则是每个线程使用计数器来指定。方法区和堆是线程共享的，程序计数器及Java栈是线程私有的。

- 运行时数据区是java应用运行时的监测区域，其中各个区域的内存情况，特别是堆的内存使用情况，是重点区域。堆还会分年轻代、年老代及 `Metaspace`，垃圾回收器会进行分代回收。通过它的回收情况监测，可以检测到是否存在内存泄漏，而java栈则与线程有关，线程的运行状态又与CPU相关，因此java栈的监测可以知道CPU占用过大的问题，同时方法区和java栈的占用内存大小也是一个监测的指标。


# 3 如何监测

经过上面的描述，我们大概已经知道一个java应用启动后，我们为什么需要监测java应用以及需要监测哪些东西。那落实到实际应用中，该如何进行监测呢？下面通过对java应用监测工具及方法进行一个概要介绍，后续将会以系列文章的形式，对各个监测工具及方法进行详细介绍。

按监测工具的监测方式，主要分为以下四大类：

- 程序内置-日志及监控页面
- java自带命令行监测工具
- java自带可视化监测工具
- 第三方诊断工具

## 3.1 程序内置监测

程序内置监测就比较简单，在初学java时，最常用的就是使用`System.out.println()`把自己想要输出的内容输出，在开发阶段也有人喜欢这样，一方面可以输出业务内容，监测功能的正常与否，另一方面可以输出系统属性`System.getProperties()`及`System.getProperty()`。当然，现在更多的使用日志框架进行输出，并把日志按级别输出到文件中，如`log4j`及`logback`。对于`Spring Boot`应用，还可以使用`actuator`来监测程序运行情况。`tomcat`容器自身也带有监测页面。此类监测主要特点是程序内置，并且通过日志输出来监测，开发人员相对比较熟悉了，一般来说在开发和测试阶段比较常用，而生产环境下，日志一般是`error`级别或者由于安全考虑，自带的一些监测页面有可能会关闭。因此，此类监测不再细说。

## 3.2 java自带命令行监测工具

在jdk的安装目录下的的bin目录，已经提供了多种命令行监测工具，以便于开发人员和运维人员监测java应用，同时方便开发及运维人员对问题进行诊断，因此，此类工具是java应用监测的重要手段。一般来说，常用的命令行工具包括`jps`,`jinfo`,`jmap`,`jstack`,`jstat`，其中

- `jps`查看java进程ID
- `jinfo`查看及调整虚拟机参数
- `jmap`查看堆(heap)使用情况及生成堆快照
- `jstack`查看线程运行状态及生成线程快照
- `jstat`显示进程中的类装载、内存、垃圾收集等运行数据。

通过这些工具，基本上可以了解java应用的内存变化状态，线程运行状态等信息，进而为应用监测及问题诊断提供依据。后面的文章将会对这些工具进行详细描述。

## 3.3 java自带可视化监测工具

除了命令行工具，在windows平台下，jdk还提供了可视化监测工具，以更直观，更方便的方式对java应用运行状况进行监测。这两款工具分别是`jconsole`和`jvisualvm`，在jdk下的bin目录下可以找到。它们均可监测本地及远程的java应用，包括堆使用情况，线程使用，cpu使用，类加载情况，gc情况，`jvisualvm`还可以生成相应的堆和线程快照，同时还可以使用相应的插件，以便于进一步分析。后面的文章将会对这两款工具的使用进行详细描述。

## 3.4 第三方诊断工具

除了java自带的工具，一些第三方的工具也是监测及分析诊断，性能调优的利器，包括`MAT`，`BTrace`及`Arthas`。其中

- `MAT`是`eclipse`的内存分析插件，通过`MAT`可以对dump出来的堆快照进行分析，并且辅助分析内存泄露原因，快速的计算出在内存中对象的占用大小，垃圾收集器的回收工作情况，并可以通过报表直观的查看到可能造成这种结果的对象。
- `BTrace`是是sun推出的一款Java 动态、安全追踪（监控）工具，可以在不停机的情况下监控系统运行情况，并且做到最少的侵入，占用最少的系统资源。特别适用在生产环境下对java应用进行监测，问题排查。
- `Arthas`是阿里开源的在线Java诊断工具，同样可以在不停机情况监控系统，包括内存情况，线程情况，GC情况，运行时数据，也可以监测方法参数、返回值，异常返回等数据，堪称神器，在生产环境下使用非常方便。

后面的文章将会对这三款第三方工具的使用进行详细描述。

# 4 总结

希望通过本系列的文章，让大家可以对java的应用监测技术有所了解，熟悉各种监测工具，以便于在遇到线上问题（特别是性能问题，oom问题等），可以从容面对及处理。

# 参考资料

- JVM规范：`https://docs.oracle.com/javase/specs/jvms/se8/html/index.html`
- JDK工具参考文档：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/`
- JAVA虚拟机体系结构：`https://www.cnblogs.com/java-my-life/archive/2012/08/01/2615221.html`
- 书籍：《深入理解Java虚拟机-JVM高级特性与最佳实践》-周志朋