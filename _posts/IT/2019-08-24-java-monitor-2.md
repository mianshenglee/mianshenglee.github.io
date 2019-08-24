---
layout: post
title: java应用监测(2)-java命令的秘密
category: IT
tags: java troubleshooting monitor
keywords: 
description: 
---

tags: java, troubleshooting, monitor

---


> 一句话概括：简单的java启动命令，原来藏着这么多秘密，本文为你揭晓。

# 1 引言

刚开始学java的同学，一定都不会忘记安装完jdk后，都会使用`java -version`命令来检测一下是否安装成功，那还有没有其它参数可以使用呢？平时开发和运行java应用时，经常看到一些`-D`的参数（如使用maven时，`package`时会使用`-Dmaven.test.skip`），这些参数是用来做什么的？还有经常说到调优，都会涉及到`-Xms`和`-Xmx`的设置，它是什么意思呢？这些，基本都是在使用`java`命令启动应用时所使用的参数，它的参数有很多，特别涉及到应用调优和问题诊断时会经常使用，学习java的同学都应该了解一下。本文将对java命令的启动参数进行详细描述，着重讲解常用的设置及用于调试监测的设置。

# 2 java应用启动

启动java应用使用的是java(class文件)或java -jar(jar或war包)命令，java命令其实就是生成一个JVM的实例，java应用则运行于此JVM实例中，JVM负责类加载，运行时区域堆栈分配等工作，当应用退出，JVM实例也会关闭。启动多个java应用，也会启动多个JVM实例，它们不会相互影响(但它们都共享同一系统的资源)，这也是为什么使用一个JDK，可以跑多个java应用的背后逻辑。使用java命令启动应用所使用的参数，基本是用于JVM的，JVM实例通过调用某个初始类的main()方法来运行一个Java程序，此方法将作为该程序初始线程的起点，任何其他的线程都是由这个初始线程启动的。在JVM内部有两种线程：守护线程（如垃圾回收线程）和非守护线程（`main`方法线程及用户使用`Thread`创建的线程），当该程序中所有的非守护线程都终止时，JVM实例将自动退出。

# 3 java应用启动参数说明

java命令究竟有哪些参数可以用，这些参数分别有什么作用，简单的不带参数使用`java`或`java -help`或`java -?`，即可看到此命令的使用方法及参数描述，如下所示：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g60etaku21j20de08v74u.jpg)

`java`执行类文件，`java -jar`执行`jar`或`war`文件。上面只是把参数简要的列了出来，更详细的参数说明，可参考官网的[`java`命令说明](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html#BABHDABI)（`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html`）。使用java命令启动应用所使用的参数，基本是用于JVM的，某种程度上也叫做JVM参数。总的来说，java启动参数共分为三大类，分别是：

- 标准参数(-)：相对稳定的参数，每个版本的JVM都可用。
- 非标准X参数(-X)：默认jvm实现这些参数的功能，但是并不保证所有jvm实现都满足，且不保证向后兼容。
- XX参数(-XX)：此类参数各个jvm实现会有所不同，将来可能会随时取消。

下面将会对这些参数进行说明。

## 3.1 标准参数(-)

从前面使用`java -?`可以看到，以`-`开头的参数，都属于标准参数，我们常用的`-help`，`-version`，`-classpath`，`-Dproperty=value`等均属于标准参数。参数详细说明如下：

```
-d32及-d64  分别表示应用运行在32位或64位的环境中，使用Java HotSpot Server VM的默认使用的是server模式，而server模式默认使用的是-d64，因此在没有使用此参数时，默认就是-d64。

-server       选择 "server" VM，默认 VM 是 server,表示是在服务器类计算机上运行。

-cp或-classpath <目录和 zip/jar 文件的类搜索路径>linux用":",windows用";"来分隔目录, JAR 档案和 ZIP 档案列表, 用于搜索类文件。
      使用-classpath后jvm将不再使用CLASSPATH中的类搜索路径，如果-classpath和CLASSPATH都没有设置，则jvm使用当前路径(.)作为类搜索路径。

-D<名称>=<值> 设置系统属性,运行在此jvm之上的应用程序可用System.getProperty(“property”)得到value的值。
      如果value中有空格，则需要用双引号将该值括起来，如-Dfoo=”foo bar”。该参数通常用于设置系统级全局变量值，如配置文件路径，以便该属性在程序中任何地方都可访问。

-verbose:[class|gc|jni] 启用详细输出，一般在调试和诊断时，都会把gc的详细信息输出
-version      输出产品版本并退出
-version:<值> 需要指定的版本才能运行
-showversion  输出产品版本并继续，即输出版本后，继续按java执行，这是跟-version的区别
-jre-restrict-search | -no-jre-restrict-search 在版本搜索中包括/排除用户专用 JRE
-? -help      输出此帮助消息
-X            输出非标准选项的帮助
-ea或-enableassertions [:<packagename>...|:<classname>] 按指定的粒度启用断言，默认jvm关闭断言机制
-da或-disableassertions [:<packagename>...|:<classname>] 禁用具有指定粒度的断言
-esa | -enablesystemassertions 启用系统断言
-dsa | -disablesystemassertions 禁用系统断言
-agentlib:<libname>[=<选项>] 加载本机代理库 <libname>, 例如 -agentlib:hprof
                  另请参阅 -agentlib:jdwp=help 和 -agentlib:hprof=help
-agentpath:<pathname>[=<选项>] 按完整路径名加载本机代理库
-javaagent:<jarpath>[=<选项>] 加载Java编程语言代理, 请参阅 java.lang.instrument
-splash:<imagepath> 使用指定的图像显示启动屏幕，一般用于图形编程。
```

由上面描述可，可知道我们常用的`-version`，`-classpath`，`-Dproperty=value`是用于做什么的了。特别提一下`-classpath`（以前遇到由于这个导致运行问题），jvm在加载类时，搜索的路径就是此路径，而它在linux及windows使用的分隔符是不一样的，linux用`:`,windows用`;`来分隔。

## 3.2 非标准X参数(-X)

使用命令`java -X`，即可把非标准参数输出，平时使用中，我们用得较多的就是`-Xloggc`，`-Xms<size>`,`-Xmx<size>`,`-Xss<size>`,`-Xmn<size>`了，详细说明如下所示：

```shell
-Xmixed  默认是mixed，使用它们来设置JVM的本地代码编译模式
-Xint    表示解释执行，所有的字节码将被直接执行，而不会编译成本地码
-Xcomp   表示第一次使用就编译成本地代码。
-Xbatch  禁止后台代码编译，强制在前台编译，编译完成之后才能进行代码执行，默认情况下，jvm在后台进行编译，若没有编译完成，则前台运行代码时以解释模式运行
-Xbootclasspath:    设置搜索路径以引导类和资源，让jvm从指定路径（可以是分号分隔的目录、jar、或者zip）中加载bootclass，用来替换jdk的rt.jar
-Xbootclasspath/a:  附加在引导类路径末尾
-Xbootclasspath/p:  置于引导类路径之前，让jvm优先于bootstrap默认路径加载指定路径的所有文件
-Xcheck:jni    对JNI函数进行附加check；此时jvm将校验传递给JNI函数参数的合法性，在本地代码中遇到非法数据时，jmv将报一个致命错误而终止；使用该参数后将造成性能下降，请慎用。
-Xfuture   让jvm对类文件执行严格的格式检查（默认jvm不进行严格格式检查），以符合类文件格式规范，推荐开发人员使用该参数
-Xincgc    开启增量gc（默认为关闭）；这有助于减少长时间GC时应用程序出现的停顿；但由于可能和应用程序并发执行，所以会降低CPU对应用的处理能力
-Xloggc:file   与-verbose:gc功能类似，只是将每次GC事件的相关情况记录到一个文件中，文件的位置最好在本地，以避免网络的潜在问题。若与verbose命令同时出现在命令行中，则以-Xloggc为准。
-Xms   指定jvm堆的初始大小，默认为物理内存的1/64，最小为1M；可以指定单位，比如k、m，若不指定，则默认为字节。
-Xmx   指定jvm堆的最大值，默认为物理内存的1/4或者1G，最小为2M；单位与-Xms一致。
-Xss   设置单个线程栈的大小，一般默认为512k。
-Xmn   设置堆(heap)的年轻代的初始值及最大值，单位与-Xms一致，年轻代是存储新对象的地址，也是GC发生得最频繁的地方，若设置过小，则会容易触发年轻代垃圾回收（minor gc），若设置过大，只触发full gc，则占用时间会很长，oracle建议是把年轻代设置在堆大小的四份之一到一半的。这命令同时设置了初始值和最大值，可以使用-XX:NewSize和-XX:MaxNewSiz来分别设置。
-XshowSettings    显示所有设置并继续
```
上述参数中，`-Xms<size>`,`-Xmx<size>`,`-Xss<size>`,`-Xmn<size>`都是我们性能优化中很重要的参数，`-Xloggc`是在没有专业跟踪工具情况下排错的好手。

## 3.3 XX参数(-XX)

此类参数非常丰富，包括高级运行时参数，高级JIT编译参数，高级维护参数和高级GC参数，在官网可以看到它全部的参数（`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html`），各个版本jvm实现有可能会有所不同。其中按设置格式，主要分为两类，一种是`boolean`类型，主要用于功能开关，一种是`key-value`类型，主要性能、调试参数等设置，下面列举一些主要使用的参数。

### 3.3.1 boolean类型

此类参数，格式：`-XX:[+-]<name>`，作为功能开关，表示启用或者禁用属性。以下列举一些：

```shell
-XX:+PrintFlagsFinal  输出参数的最终值
-XX:+PrintFlagsInitial 输出参数的默认值
-XX:-DisableExplicitGC  禁止调用System.gc()；但jvm的gc仍然有效
-XX:+MaxFDLimit 最大化文件描述符的数量限制
-XX:+ScavengeBeforeFullGC   新生代GC优先于Full GC执行
-XX:+UseGCOverheadLimit 在抛出OOM之前限制jvm耗费在GC上的时间比例
-XX:-UseConcMarkSweepGC 对老生代采用并发标记交换算法进行GC
-XX:-UseParallelGC  启用并行GC
-XX:-UseParallelOldGC   对Full GC启用并行，当-XX:-UseParallelGC启用时该项自动启用
-XX:-UseSerialGC    启用串行GC
-XX:+UseThreadPriorities    启用本地线程优先级
-XX:-UseG1GC    启用G1的GC
```

### 3.3.2 key-value类型

此类参数，格式：`-XX:<name>=<value>`表示属性name的值为value。在性能调优和调试监测时，会经常用到。

- 性能调优

性能调优时，主要是对JVM的内存分配情况的调优，包括堆大小，年轻代大小，年轻年老代比例等等。

```shell
-XX:LargePageSizeInBytes=4m 设置用于Java堆的大页面尺寸
-XX:MaxHeapFreeRatio=70 GC后java堆中空闲量占的最大比例
-XX:MaxNewSize=size 新生成对象能占用内存的最大值
-XX:MaxPermSize=64m 老生代对象能占用内存的最大值
-XX:MinHeapFreeRatio=40 GC后java堆中空闲量占的最小比例
-XX:NewRatio=2  新生代内存容量与老生代内存容量的比例
-XX:NewSize=2.125m  新生代对象生成时占用内存的默认值
-XX:ReservedCodeCacheSize=32m   保留代码占用的内存容量
-XX:ThreadStackSize=512 设置线程栈大小，若为0则使用系统默认值
-XX:+UseLargePages  使用大页面内存
```

- 调试监测

在需要对应用进行监测，特别是观察GC情况，OOM后检查问题等。

```shell
-XX:-CITime 打印消耗在JIT编译的时间
-XX:ErrorFile=./hs_err_pid<pid>.log 保存错误日志或者数据到文件中
-XX:-ExtendedDTraceProbes   开启solaris特有的dtrace探针
-XX:HeapDumpPath=./java_pid<pid>.hprof  指定导出堆信息时的路径或文件名
-XX:-HeapDumpOnOutOfMemoryError 当首次遭遇OOM时导出此时堆中相关信息
-XX:OnError="<cmd args>;<cmd args>" 出现致命ERROR之后运行自定义命令
-XX:OnOutOfMemoryError="<cmd args>;<cmd args>"  当首次遭遇OOM时执行自定义命令
-XX:-PrintClassHistogram    遇到Ctrl-Break后打印类实例的柱状信息，与jmap -histo功能相同
-XX:-PrintConcurrentLocks   遇到Ctrl-Break后打印并发锁的相关信息，与jstack -l功能相同
-XX:-PrintCommandLineFlags  打印在命令行中出现过的标记
-XX:-PrintCompilation   当一个方法被编译时打印相关信息
-XX:-PrintGC    每次GC时打印相关信息
-XX:-PrintGC Details    每次GC时打印详细信息
-XX:-PrintGCTimeStamps  打印每次GC的时间戳
-XX:-TraceClassLoading  跟踪类的加载信息
-XX:-TraceClassLoadingPreorder  跟踪被引用到的所有类的加载信息
-XX:-TraceClassResolution   跟踪常量池
-XX:-TraceClassUnloading    跟踪类的卸载信息
-XX:-TraceLoaderConstraints 跟踪类加载器约束的相关信息
```

# 4 常用java应用启动参数

经过前面几个章节的介绍，大家应该对java的启动参数（JVM参数）有一定的了解，但参数太多了，不可能把所有参数都得记住，有需要时，建议大家看`-help`或者看官网说明来查阅。很多时候，我们只需要记住几个常用的即可。下面总结一下常用的JVM参数。

## 4.1 常用标准参数

- `-version`，场景：想查看JDK版本，`java -version`。
- `-D<名称>=<值>`，场景:maven跳过单元测试，使用`java -Dmaven.test.skip=true`,
- `-cp或-classpath`, 场景：设置需要加载的jar包位置，使用`java -cp lib/test.jar com.test.TestMain`
- `-verbose:gc`, 场景：输出GC详细信息

## 4.2 常用X参数

- `-Xms<size>`和`-Xmx<size>`，场景：由于内存不足发生oom，调大堆大小，如设置为1G，可以`java -Xms1024m -Xmx1024m`，通常为了避免频繁发生GC，`-Xms`和`-Xmx`设置为一致。
- `-Xss<size>`，场景：线程操作数及局部变量多，把线程栈的大小调大，可以`java -Xss1024k`
- `-Xmn<size>`，场景：年轻代大小设置为512m，可以`java -Xmn512m`
- `-Xloggc:file`，场景：将每次GC事件的相关情况记录到一个文件中以便于后续分析，可以`java -Xloggc:logs/gc.log`

## 4.3 常用XX参数

打印GC相关的内容，包括堆情况，GC详情，GC时间，发生OOM时，生成快照，发生错误是记录错误日志等，如下：
- `-XX:+PrintHeapAtGC`
- `-XX:+PrintGCDetails`
- `-XX:+PrintGCDateStamps`
- `-XX:+PrintGCTimeStamps`
- `-XX:+PrintTenuringDistribution`
- `-XX:+PrintGCApplicationStoppedTime`
- `-XX:+HeapDumpOnOutOfMemoryError`
- `-XX:HeapDumpPath=logs/heapdump.hprof`，发生OOM时，dump出快照到文件`heapdump.hprof`中。
- `-XX:ErrorFile=logs/java_error_%p.log`，发生JVM错误时，把日志输出到`java_error_%p.log`中。

以上参数均是使用度很高的参数，在使用`java`命令启动应用时，可以把这些参数加上，以便于后续调优与问题诊断。

# 5 总结

简单的java启动命令，使用起来原来这么复杂，当然一般来说，只使用`java`或`java -jar`来按默认值启动应用，也不会有太大问题。只是涉及到调优、监测、诊断时，了解这些参数，无疑是高级程序员必要的技能。希望通过本文，大家对`java`命令及参数可以做到心中有数。

# 参考资料
- JDK工具参考文档：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/`

# 相关阅读

- 《[java应用监测(1)-java程序员应该知道的应用监测技术](https://mianshenglee.github.io/2019/08/23/java-monitor-1.html)》：`https://mianshenglee.github.io/2019/08/23/java-monitor-1.html`