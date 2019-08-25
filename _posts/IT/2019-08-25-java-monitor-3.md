---
layout: post
title: java应用监测(3)-这些命令行工具你掌握了吗
category: IT
tags: java troubleshooting monitor jvm
keywords: 
description: 
---

tags: java, troubleshooting, monitor,jvm

---


> 一句话概括：原来jdk自带的命令行工具如此好用，本文将详细介绍。

# 1 引言

监测java应用，最方便的就是直接使用jdk提供的现成工具，在jdk的安装的bin目录下，已经提供了多种命令行监测工具，以便于开发人员和运维人员监测java应用和诊断问题，因此，此类工具是java应用监测的重要手段。也是作为java开发人员需要掌握的基本技能。

# 2 常用监测命令行工具

一般来说，常用的命令行工具包括`jps`,`jinfo`,`jmap`,`jstack`,`jstat`，这些工具都在`JAVA_HOME/bin/`目录下，概要说明如下：

- `jps`查看java进程ID
- `jinfo`查看及调整虚拟机参数
- `jmap`查看堆(heap)使用情况及生成堆快照
- `jstack`查看线程运行状态及生成线程快照
- `jstat`显示进程中的类装载、内存、垃圾收集等运行数据。

通过这些工具，基本上可以了解java应用的内存变化状态，线程运行状态等信息，进而为应用监测及问题诊断提供依据。下面将结合实例对这些工具的使用进行详细讲解，文中所使用的示例代码`java-monitor-example`已上传到[我的github](https://github.com/mianshenglee/my-example/tree/master/java-monitor-example)，地址：`https://github.com/mianshenglee`。

# 3 进程查询工具`jps`

## 3.1 `jps`说明

要监测java应用，第一步就是先知道这个应用是哪个进程，它的运行参数是什么。`jps`就是可以查询进程的工具。熟悉linux的同学，大概都知道查询进程使用`ps -ef|grep java`这样的命令，jps也类似，但它不使用名称查找，而是查找全部当前jdk运行的java进程，而且只查找当前用户的Java进程，而不是当前系统中的所有进程。

## 3.2 `jps`使用

作为命令行工具，可以通过`-help`参数查看帮助，也可查阅[官方文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html`，如下：

```shell
[root@test bin]# jps -help
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]

参数解释：
-q：只显示java进程的pid
-m：输出传递给main方法的参数，在嵌入式jvm上可能是null
-l：输出应用程序main class的完整package名 或者 应用程序的jar文件完整路径名
-v：输出传递给JVM的参数
```

示例工程`java-monitor-example`在linux机器中运行起来，使用`jps`可输出以下信息：

- 只输出进程ID

```shell
[root@test bin]# jps -q
13680
14214
```

- 输出程序完整名称及JVM参数

```shell
[root@test bin]# jps -lv
13680 java-monitor-example-0.0.1-SNAPSHOT.jar -Xms128m -Xmx128m -Dserver.port=8083
14289 sun.tools.jps.Jps -Denv.class.path=.:/opt/jdk8/lib:/opt/jdk8/jre/lib -Dapplication.home=/opt/jdk8
```

输出的内容中，`java-monitor-example-0.0.1-SNAPSHOT.jar`是`-l`输出的完整名称，`-Xms128m -Xmx128m -Dserver.port=8083`是传给JVM的参数。

- 在shell脚本中使用命令获取java进程ID并作为变量使用

```shell
JAVA_HOME="/opt/jdk8"
APP_MAINCLASS=java-monitor-example
#初始化psid变量（全局）
psid=0

#查看进程ID函数
checkpid() {
   javaps=`$JAVA_HOME/bin/jps -l | grep $APP_MAINCLASS`
 
   if [ -n "$javaps" ]; then
      psid=`echo $javaps | awk '{print $1}'`
   else
      psid=0
   fi
}

#调用函数后通过psid进行业务逻辑操作，如根据进程id杀进程
checkpid
echo "(pid=$psid)"
```

上述脚本，比较适合运维人员对应用的开启和关闭，自动获取java进程ID，然后根据ID判断程序是否运行(start)，或者关闭应用(`kill -9`)。


# 4 配置信息工具`jinfo`

## 4.1 `jinfo`说明

知道java应用所属的进程号是第一步，在上一篇文章《java应用监测(2)-java命令的秘密》中，已经知道java的启动参数有很多，监测java应用前需要了解清楚它的启动参数是什么。这时就需要用到`jinfo`工具。`jinfo`可以输出JAVA应用的系统参数和JVM参数。jinfo还能够修改一部分运行期间能够调整的虚拟机参数，很多运行参数是不能调整的，如果出现"cannot be changed"异常，说明不能调整。不过官方文档指出，这个命令在后续的版本中可能不再使用，当前JDK8还是可以用的。

## 4.2 `jinfo`使用

通过`-help`参数查看帮助，也可查阅[`jinfo`官方文档说明](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html`，如下：

```shell
[root@test bin]# jinfo -help
Usage:
    jinfo [option] <pid>
        (to connect to running process)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message

```

使用`jps`获取到应用的进程ID后(示例的PID为13680)，如果直接`jps <pid>`则会输出全部的系统参数和JVM参数，其它参数说明在`help`中也说得很清楚了。下面还是结合示例代码`java-monitor-example`来实践一下：

- 获取java应用的堆初始值

```shell
[root@test bin]# jinfo -flag InitialHeapSize 13680
-XX:InitialHeapSize=134217728
```

- 查看全部的JVM参数

```shell
[root@test bin]# jinfo -flags 13680
Attaching to process ID 13680, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.51-b03
Non-default VM flags: -XX:CICompilerCount=2 -XX:InitialHeapSize=134217728 -XX:MaxHeapSize=134217728 -XX:MaxNewSize=44564480 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=44564480 -XX:OldSize=89653248 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Xms128m -Xmx128m -Dserver.port=8083

```

可见，由于我们启动时设置了`-Xms`和`-Xmx`，它们对应的就是`-XX:InitialHeapSize`及`-XX:MaxHeapSize`值。另外，参数`-Dserver.port`属于系统参数，使用`jinfo -sysprops 13680`就可以查看系统参数了。

# 5 堆内存查看工具`jmap`

## 5.1 `jmap`说明

java应用启动后，它在JVM中运行，内存是需要重点监测的地方，`jmap`就是这样的一个工具，它可以获取运行中的jvm的堆的快照，包括整体情况，堆占用情况的直方图，dump出快照文件以便于离线分析等。官方文档指出，这个命令在后续的版本中可能不再使用，当前JDK8还是可以用的。

## 5.2 `jmap`使用

通过`-help`参数查看帮助，也可查阅[`jmap`官方文档说明](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html`，帮助说明如下：

```shell
[root@test bin]# jmap -help
Usage:
    jmap [option] <pid>
        (to connect to running process)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -clstats             to print class loader statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system

```

如上所示，`jmap`参数常用的是`-heap`,`-histo`和`-dump`，结合示例`java-monitor-example`，说明如下：

- 打印jvm内存整体使用情况

```shell
[root@test bin]# jmap -heap 13680
Attaching to process ID 13680, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.51-b03

using thread-local object allocation.
Parallel GC with 2 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 134217728 (128.0MB)
   NewSize                  = 44564480 (42.5MB)
   MaxNewSize               = 44564480 (42.5MB)
   OldSize                  = 89653248 (85.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 31981568 (30.5MB)
   used     = 5306632 (5.060798645019531MB)
   free     = 26674936 (25.43920135498047MB)
   16.59278244268699% used
From Space:
   capacity = 6291456 (6.0MB)
   used     = 1081440 (1.031341552734375MB)
   free     = 5210016 (4.968658447265625MB)
   17.18902587890625% used
To Space:
   capacity = 6291456 (6.0MB)
   used     = 0 (0.0MB)
   free     = 6291456 (6.0MB)
   0.0% used
PS Old Generation
   capacity = 89653248 (85.5MB)
   used     = 16615680 (15.845947265625MB)
   free     = 73037568 (69.654052734375MB)
   18.533271655701753% used

18006 interned Strings occupying 2328928 bytes.

```

从以上信息，可以看出JVM中堆内存当前的使用情况，包括年轻代（`Eden`区，`From`区，`To`区）和年老代。

- 查看类名，对象数量，对象占用大小直方图

```shell

[root@test bin]# jmap -histo:live 13680|more

 num     #instances         #bytes  class name
----------------------------------------------
   1:         36536        6462912  [C
   2:         35557         853368  java.lang.String
   3:          7456         826968  java.lang.Class
   4:         20105         643360  java.util.concurrent.ConcurrentHashMap$Node
   5:          1449         469024  [B
   6:          6951         399280  [Ljava.lang.Object;
   7:          9311         297952  java.util.HashMap$Node
   8:          3122         274736  java.lang.reflect.Method
   9:          2884         269112  [I
  10:          6448         257920  java.util.LinkedHashMap$Entry
  11:          2994         255160  [Ljava.util.HashMap$Node;
  12:         15249         243984  java.lang.Object

.....
.....

```

如上所示，使用`-histo`输出包括序号，实例数，占用字节数和类名称。具体说明如下：

> 1. instances列：表示当前类有多少个实例。
> 2. bytes列：说明当前类的实例总共占用了多少个字节
> 3. class name列：表示的就是当前类的名称，class name 解读：
> 4. B代表byte 
> 5. C代表char 
> 6. D代表double 
> 7. F代表float 
> 8. I代表int 
> 9. J代表long 
> 10. Z代表boolean 
> 11. [代表数组，如[I相当于int[] 
> 12. 对象用[L+类名表示 

- 把内存情况dump内存到本地文件

```shell

[root@test bin]# jmap -dump:file=./heap.hprof 13680

```

如上所示，会把堆情况写入到当前目录的`heap.hprof`文件中，至于如何分析此文件，可以使用`jhat`，但一般实际开发中，很少使用jhat来直接对内存dump文件进行分析，因此不再对它进行讲述。更多的是使用工具`MAT`，以可视化的方式来查看，后续文章将会对`MAT`工具的使用进行详细讲解。

# 6 线程栈查询工具`jstack`

## 6.1 `jstack`说明

此命令打印指定Java应用的线程堆栈，对于每个Java帧，将打印完整的类名，方法名，字节代码索引（BCI）和行号，可以用于检测死锁，线程停顿，进程耗用cpu过高报警问题等排查。

## 6.2 `jstack`使用

使用`-help`参数查看帮助，也可查阅[`jstack`官方文档说明](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html`，帮助说明如下：

```shell

[root@test bin]# jstack -help
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>

Options:
    -F  强制dump线程堆栈信息. 用于进程hung住， jstack <pid>命令没有响应的情况
    -m  同时打印java和本地(native)线程栈信息，m是mixed mode的简写
    -l  打印锁的额外信息
```

结合示例`java-monitor-example`，可以打印线程信息（一般都会把打印的内容写入到文件然后再分析），如下：

- 打印当前线程堆栈信息

```shell
[root@test bin]# jstack 13680
2019-08-16 23:18:18
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.51-b03 mixed mode):
"http-nio-8083-Acceptor-0" #39 daemon prio=5 os_prio=0 tid=0x00007f7520698800 nid=0x359a runnable [0x00007f7508bb7000]
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)
	at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:422)
	at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:250)
	- locked <0x00000000f8c85380> (a java.lang.Object)
	at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:448)
	at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:70)
	at org.apache.tomcat.util.net.Acceptor.run(Acceptor.java:95)
	at java.lang.Thread.run(Thread.java:745)

```

## 6.3 线程dump分析

### 6.3.1 线程状态

java线程栈使用`jstack`dump出来后，可以看到线程的状态，线程状态一共分6种，可以参考[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr034.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr034.html`，下面是它的状态说明：

- NEW

线程已经new出来创建了，但是还没有启动(not yet started),`jstack`不会打印这个状态的线程信息。

- RUNNABLE 

正在Java虚拟机下跑任务的线程的状态，但其实它是只是表示线程是可运行的（ready）。对于单核的CPU，多个线程在同一时刻，只能运行一个线程，其它的则需要等CPU的调度。

- BLOCKED 

线程处于阻塞状态，正在等待一个锁，多个线程共用一个锁，当某线程正在使用这个锁进入某个synchronized同步方法块或者方法，而此线程需要进入这个同步代码块，也需要这个锁，则导致本线程处于阻塞状态。

- WAITING

等待状态，处于等待状态的线程是由于执行了3个方法中的任意方法。 1. `Object.wait`方法，并且没有使用timeout参数; 2. `Thread.join`方法，没有使用timeout参数 3. `LockSupport.park`方法。 处于waiting状态的线程会等待另外一个线程处理特殊的行为。一个线程处于等待状态(wait，通常是在等待其他线程完成某个操作（notify或者notifyAll）。注意，`Object.wait()`方法只能够在同步代码块中调用。调用了`wait()`方法后，会释放锁。

- TIMED_WAITING

线程等待指定的时间，对于以下方法的调用，可能会导致线程处于这个状态：1. Thread.sleep方法 2. `Object.wait`方法，带有时间 3. `Thread.join`方法，带有时间 4. `LockSupport.parkNanos`方法，带有时间 5. `LockSupport.parkUntil`方法，带有时间。注意，`Thread.sleep`方法调用后，它不会释放锁，仍然占用系统资源。

- TERMINATED

线程中止的状态，这个线程已经完整地执行了它的任务。

从下面这张图可以看出线程状态的变化情况:

![](http://ww1.sinaimg.cn/large/72d660a7ly1g61wl01qu2j20qo0l1aaq.jpg)

### 6.3.2 分析`jstack`后线程栈内容

从前面使用`jstack` dump出来信息，我们需要知道以下几个信息：

- `"http-nio-8083-Acceptor-0" #39`：是线程的名字，因此，一般我们创建线程时需要设置自己可以辩识的名字。
- `daemon` 表示线程是否是守护线程
- `prio` 表示我们为线程设置的优先级
- `os_prio` 表示的对应的操作系统线程的优先级，由于并不是所有的操作系统都支持线程优先级，所以可能会出现都置为0的情况
- `tid` 线程的id
- `nid` 线程对应的操作系统本地线程id，每一个java线程都有一个对应的操作系统线程，它是16进制的，因此一般在操作系统中获取到线程ID后，需要转为16进制，来对应上。
- `java.lang.Thread.State: RUNNABLE` 运行状态，上面已经介绍了线程的状态，若是WAITING状态，则括号中的内容说明了导致等待的原因，如parking说明是因为调用了LockSupport.park方法导致等待。通常的堆栈信息中，都会有lock标记，如`- locked <0x00000000f8c85380> (a java.lang.Object)`表示正在占用这个锁。
- 对于线程停顿，CPU占用等问题，可以重点看一下`wait`状态的线程
- 对于死锁，在Dump出来的线程栈快照可以直接报告出Java级别的死锁。

# 7 JVM统计数据工具`jstat`

## 7.1 `jstat`说明

`jstat`即`JVM Statistics Monitoring Tool`，即JVM统计监测工具，包括监测类装载、内存、垃圾收集、JIT编译等运行数据，在没有图形的服务器上，它是运行期定位虚拟机性能问题的首选工具。

## 7.2 `jstat`使用

使用`-help`参数查看帮助，也可查阅[`jstat`官方文档说明](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html`，帮助说明如下：

```shell
[root@test bin]# jstat -help
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
```

以上所示，`vmid`、`interval`、`count`分别是进程号，打印间隔时间（s或ms），打印次数，其中`option`参数主要是以下（也可以使用命令`jstat -option`查看）：

- `-class` 统计class loader行为信息 ，如总共加载了多少个类
- `-compile` 统计HotSpot Just-in-Time编译器的行为
- `-gc` 统计jdk gc时heap信息 
- `-gccapacity` 统计不同的generations相应的heap容量情况 
- `-gccause` 统计gc的情况，（同-gcutil）和引起gc的事件 
- `-gcnew` 统计gc时，新生代的情况 
- `-gcnewcapacity` 统计gc时，新生代heap容量 
- `-gcold` 统计gc时，老年区的情况 
- `-gcoldcapacity` 统计gc时，老年区heap容量 
- `-gcpermcapacity` 统计gc时，permanent区heap容量 
- `-gcutil` 统计gc时，heap情况 
- `-printcompilation` hotspot编译方法统计

一般我们使用`-class`，`-gc`，`-gccause`和`-gcutil`比较多，主要用于来分析类和堆使用情况及gc情况。

## 7.3 监测JVM的GC情况

以上述的示例工程`java-monitor-example`为例，里面包含了一个函数来测试内存溢出（使用一个数组，循环创建对象，直到内存溢出）。使用`jstat -gc 13680 1000`即每秒监测一次，调用`/monitor/user/oom`接口后，即看到堆和GC变化情况。为方便查看，我把输出放到sublime中显示，如下所示：

![](http://ww1.sinaimg.cn/large/72d660a7ly1g62uczm3clj20s305l74r.jpg)

日志输出OOM报错：

![](http://ww1.sinaimg.cn/large/72d660a7ly1g62trp5w6yj20um09gwfh.jpg)

以上输出的内容，每列的说明如下：

- `S0C` 当前年轻代中第一个survivor（s0）的总容量 (KB).
- `S1C` 当前年轻代中第一个survivor（s1）的总容量 (KB).
- `S0U` s0已使用的容量 (KB).
- `S1U` s1已使用的容量 (KB).
- `EC` 当前年轻代中eden区总容量 (KB).
- `EU` eden区已经使用的容量 (KB).
- `OC` 年老代的容量总容量 (KB).
- `OU` 年老代已使用容量(KB).
- `MC` 当前 Metaspace总容量(KB).
- `MU` 当前 Metaspace已使用容量 (KB).
- `CCSC`  Compressed class容量大小
- `CCSU`  Compressed class已使用容量
- `YGC` 从应用启动时到现在，年轻代young generation 发生GC Events的总次数.
- `YGCT` 从应用启动时到现在， 年轻代Young generation 垃圾回收的总耗时.
- `FGC` 从应用启动时到现在， full GC事件总次数.
- `FGCT` 从应用启动时到现在， Full sc总耗时.GCT     从应用启动时到现在， 垃圾回收总时间. - `GCT` GCT=YGCT+FGCT

从以上输出第6行可以看出，`EC`和`EU`，`OC`和`OU`表示年轻代、年老代的内存都已经用完(与容量数值相等)，发生OOM。这时，则需要采取措施，增大内存（-Xmx参数）或者找到导致OOM的代码进行修改。

# 8 总结

针对java应用的监测，本文对jdk提供自身提供的命令行工具进行了说明和使用的介绍，完整的描述了查看java应用进程，查看启动参数，查看内存情况，查看线程情况，查看内存统计情况等，主要是`jps`，`jinfo`，`jmap`，`jstack`，`jstat`5个工具，并结合实例，希望学习java开发人员都能掌握这些技术，在监测java应用时，可以从容面对如OOM，CPU高，线程停顿等问题。

# 参考资料
- [JDK工具参考文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix/`
- [示例代码`java-monitor-example`](https://github.com/mianshenglee/my-example/tree/master/java-monitor-example)：`https://github.com/mianshenglee/my-example/tree/master/java-monitor-example`
- [Java开发必须掌握的线上问题排查命令](https://www.hollischuang.com/archives/1561)：`https://www.hollischuang.com/archives/1561`
- [Java自带的性能监测工具](http://www.tianshouzhi.com/api/tutorials/jvm/346)：`http://www.tianshouzhi.com/api/tutorials/jvm/346`

# 相关阅读

- [《java应用监测(1)-java程序员应该知道的应用监测技术》](https://mianshenglee.github.io/2019/08/23/java-monitor-1.html)：`https://mianshenglee.github.io/2019/08/23/java-monitor-1.html`
- [《java应用监测(2)-java命令的秘密》](https://mianshenglee.github.io/2019/08/24/java-monitor-2.html)：`https://mianshenglee.github.io/2019/08/24/java-monitor-2.html`