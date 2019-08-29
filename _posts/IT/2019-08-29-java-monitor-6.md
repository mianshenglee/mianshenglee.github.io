---
layout: post
title: java应用监测(6)-第三方内存分析工具MAT
category: IT
tags: java troubleshooting monitor mat
keywords: 
description: 
---

tags: java,troubleshooting,monitor,mat

---

> 一句话概括：`MAT`是一个强大的内存分析工具，可以快捷、有效地帮助我们找到内存泄露，减少内存消耗分析工具，下文将进行讲解。

# 1 引言
之前的文章有提过，内存中堆的使用情况是应用性能监测的重点，而对于堆的快照，可以dump出来进一步分析，总的来说，一般我们对于堆dump快照有三种方式：

- 添加启动参数发生OOM时自动dump：
java应用的启动参数一般最好都加上`-XX:+HeapDumpOnOutOfMemoryError`及`-XX:HeapDumpPath=logs/heapdump.hprof`，即在发生OOM时自动dump堆快照，但这种方式相当来说是滞后的（需要等到发生OOM后）。
- 使用命令按需手动dump：
我们也可以使用`jmap -dump:format=b,file=HeapDump.hprof <pid>`工具手动进行堆dump和线程dump
- 使用工具手动dump：
 如[上一篇文章](https://mianshenglee.github.io/2019/08/27/java-monitor-5.html)对`jconsole`及`jvisualvm`的讲解中讲到，jvisualvm有提供dump堆快照的功能，点击一下即可。

那么，对于dump出来的文件，如何对它们进行分析？`jvisualvm`可以直接装入快照文件进行分析，而本文所介绍的`MAT`，相对来说内存分析功能更强大，它可以自动检测有可能发生问题（特别是内存溢出、内存泄露）的地方，也可以详细查看类内存占用情况，方法级的调用情况等，是一个分析内存状况的不可多得的工具。

# 2 MAT工具介绍

MAT(Memory Analyzer tool)是一款内存分析器工具，它是一个快速且功能丰富的堆内存分析器，帮助我们查找内存泄漏和分析高内存消耗问题。[官方网站](https://www.eclipse.org/mat/)是：`https://www.eclipse.org/mat/`，有兴趣可以上去看一下。使用MAT，可以轻松实现以下功能：
- 找到最大的对象，因为MAT提供显示合理的累积大小（`retained size`）
- 探索对象图，包括`inbound`和`outbound`引用，即引用此对象的和此对象引出的。
- 查找无法回收的对象，可以计算从垃圾收集器根到相关对象的路径
- 找到内存浪费，比如冗余的String对象，空集合对象等等

# 3 MAT工具安装

`MAT`安装有两种方式，一种是以`eclipse`插件方式安装，一种是独立安装。在`MAT`的[官方文档](https://www.eclipse.org/mat/downloads.php)中有相应的安装文件下载，下载地址为：`https://www.eclipse.org/mat/downloads.php`

- 若使用eclipse插件安装，`help -> install new soft`点击`ADD`，在弹出框中添加插件地址：`http://download.eclipse.org/mat/1.9.0/update-site/`，也可以直接在下载页面下载离线插件包，以离线方式安装。
- 独立安装，windows下，根据系统直接下载`Windows (x86)`或`Windows (x86_64)`，下载时可以选择适合自己的镜像，双击安装即可。

# 4 MAT工具使用

`MAT`定位是内存分析工具，它的主要功能就是对内存快照进行分析，帮助我们找到有可能内存溢出或内存泄漏的地方，因此，找到占用内存大的对象和找出无法回收的对象是其主要目的。[MAT官方文档](https://help.eclipse.org/2019-06/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)，地址如下：`https://help.eclipse.org/2019-06/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html`，对`MAT`的使用进行描述，有兴趣的同学可以上去看看。下面主要对`MAT`的常用概念、常用的功能进行介绍。

> 下文中，以[`java-monitor-example`](https://github.com/mianshenglee/my-example/tree/master/java-monitor-example)`(https://github.com/mianshenglee/my-example/tree/master/java-monitor-example)`为例，此示例是一个简单的`spring boot`工程，里面的一个`controller`中的`user/oom`接口调用的`service`对象通过`List`成员不断地添加`User`对象，最终导致`OOM`的发生，应用的启动参数是`-Xms64m -Xmx64m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${APP_HOME}/logs/heapdump.hprof`。

## 4.1 MAT相关概念说明

### 4.1.1 内存泄漏与内存溢出

- 内存泄露：对象已经没用了（不被任何程序逻辑所需要），还存在被根元素引用的情况，无法通过垃圾收集器进行自动回收，需要通过找出泄漏的代码位置和原因，才好确定解决方案；
- 内存溢出：内存中的对象都还存活着，JVM的堆分配空间不足，需要检查堆设置大小（-Xmx与-Xms），代码是否存在对象生命周期太长、持有状态时间过长的情况。

### 4.1.2 引用（强引用，软引用，弱引用，虚引用）

- `Strong Ref`(强引用)：强可达性的引用，对象保存在内存中，只有去掉强可达，对象才被回收，通常我们编写的代码都是Strong Ref。

- `Soft Ref`(软引用)：对应软可达性，只要有足够的内存，就一直保持对象，直到发现内存吃紧且没有Strong Ref时才回收对象。一般可用来实现缓存，通过java.lang.ref.SoftReference类实现。

- `Weak Ref`(弱引用)：比Soft Ref更弱，当发现不存在Strong Ref时，立刻回收对象而不必等到内存吃紧的时候。通过java.lang.ref.WeakReference和java.util.WeakHashMap类实现。

- `Phantom Ref`(虚引用)：根本不会在内存中保持任何对象，你只能使用Phantom Ref本身。一般用于在进入finalize()方法后进行特殊的清理过程，通过 java.lang.ref.PhantomReference实现。

### 4.1.3 `shallow heap`及`retained heap`

- `shallow heap`：对象本身占用内存的大小，也就是对象头加成员变量（不是成员变量的值）的总和，如一个引用占用32或64bit，一个integer占4bytes，Long占8bytes等。如简单的一个类里面只有一个成员变量`int i`，那么这个类的`shallo size`是12字节，因为对象头是8字节，成员变量`int`是4字节。常规对象（非数组）的Shallow size有其成员变量的数量和类型决定，数组的shallow size有数组元素的类型（对象类型、基本类型）和数组长度决定。
- `retained heap`：如果一个对象被释放掉，那会因为该对象的释放而减少引用进而被释放的所有的对象（包括被递归释放的）所占用的heap大小，即对象X被垃圾回收器回收后能被GC从内存中移除的所有对象之和。相对于shallow heap，Retained heap可以更精确的反映一个对象实际占用的大小（若该对象释放，retained heap都可以被释放）。

### 4.1.4 `outgoing references`与`incoming references`

- `outgoing references` ：表示该对象的出节点（被该对象引用的对象）。
- `incoming references` ：表示该对象的入节点（引用到该对象的对象）。

### 4.1.5 `Dominator Tree`

将对象树转换成`Dominator Tree`能帮助我们快速的发现占用内存最大的块，也能帮我们分析对象之间的依赖关系。`Dominator Tree`有以下几个定义：

- 对象X `Dominator`（支配）对象Y，当且仅当在对象树中所有到达Y的路径都必须经过X
- 对象Y的直接`Dominator`，是指在对象树中距离Y最近的`Dominator`
- `Dominator tree`利用对象树构建出来。在`Dominator tree`中每一个对象都是他的直接`Dominator`的子节点。

对象树和`Dominator tree`的对应关系如下:

![](http://ww1.sinaimg.cn/large/72d660a7gy1g677abyxspj20ca083weg.jpg)

如上图，因为A和B都引用到C，所以A释放时，C内存不会被释放。所以这块内存不会被计算到A或者B的Retained Heap中，因此，对象树在转换成`Dominator tree`时，会A、B、C三个是平级的。

### 4.1.6 Garbage Collection Roots（GC root）

在执行GC时，是通过对象可达性来判断是否回收对象的，一个对象是否可达，也就是看这个对象的引用连是否和`GC Root`相连。一个`GC root`指的是可以从堆外部访问的对象，有以下原因可以使一个对象成为`GC root`对象。
- `System Class`: 通过bootstrap/system类加载器加载的类，如rt.jar中的java.util.*
- `JNI Local`: JNI方法中的变量或者方法形参
- `JNI Global`：JNI方法中的全局变量
- `Thread Block`：线程里面的变量，一个活着的线程里面的对象肯定不能被回收
- `Thread`：处于激活状态的线程
- `Busy Monitor`：调用了wait()、notify()方法，或者是同步对象，例如调用synchronized(Object) 或者进入一个synchronized方法后的当前对象
- `Java Local`：本地变量，例如方法的输入参数或者是方法内部创建的仍在线程堆栈里面的对象
- `Native Stack`：Java 方法中的变量或者方法形参.
- `Finalizable`：等待运行finalizer的对象
- `Unfinalized`：有finalize方法，但未进行finalized，且不在finalizer队列的对象。
- `Unreachable`：通过其它root都不可达的对象，MAT会把它标记为root以便于分析回收。
- `Java Stack Frame`：java栈帧
- `Unknown`

## 4.2 Leak Suspects自动分析泄漏

当发生`OOM`获取到`heapdump.hprof`文件或者手动dump出文件后，使用`MAT`打开文件。打开后默认会提示是否进行内存泄漏检测报告（如果打开Dump时跳过了的话，也可以从其它入口进入工具栏上的 `Run Expect System Test -> Leak Suspects`），如下图所示：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g679vasvomj20h50ayq33.jpg)

选择是后进入报告内容，此报告内容会帮我们分析的可能有内存泄露嫌疑的地方，它会把累积内存占用较大的通过饼状图显示出来。如下图所示：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g67b4hyxrlj20o10i4jsb.jpg)

如上图，报告已指出`UserService`占用了76.73%的内存，而这些内存都在Object[]这个数组中。因此，很大一种可能是这个对象中的数组数量过大。点击`Details`可以查看更详细的内容：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g67b9i1838j20ku0hhjsw.jpg)

在此详细而，`Shortest Paths To the Accumulation Point`可以显示出到`GC roots`的最短路径，由此可以分析是由于和哪个`GC root`相连导致当前`Retained Heap`占用相当大的对象无法被回收，本示例中，`GC root`是线程的本地变量(`java local`)。`Accumulated Objects in Dominator Tree`以`Dominator Tree`为视图，可以方便的看出受当前对象“支配”的对象中哪个占用Retained Heap比较大。图中可看出对象都在`ArrayList`中，而`ArrayList`下有`Object`数组，数组下是`User`对象。此可以知道问题出在哪里了。需要针对这个位置，查看代码，找出导致这个数组存储的`User`过量的原因。

> 注：在原始堆转储文件的目录下，`MAT`已经将报告的内容压缩打包到一个zip文件，命名为“xxx_Leak_Suspects.zip”，整个报告是一个HTML格式的文件，可以用浏览器直接打开查看，可以方便进行报告分发、分享。

## 4.3 histogram视图查看对象个数与大小

点击`Overview`页面`Actions`区域内的“Histogram视图”或点击工具栏的“histogram”按钮，可以显示直方图列表，它以Class类的维度展示每个Class类的实例存在的个数、 占用的`Shallow heap` 和 `Retained内存`大小，可以分别排序显示。如下图所示：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g67btbw36bj20mn07oaad.jpg)

`Shallow Heap`与`Retained Heap`的概念前面已经讲过。
对于java的对象，其成员基本都是引用。真正的内存都在堆上，看起来是一堆原生的byte[], char[], int[]，因此如果只看对象本身的内存，数量都很小，而多数情况下，在Histogram视图看到实例对象数量比较多的类都是一些基础类型（通常都排在前面），如char[]、String、byte[]，所以仅从这些是无法判断出具体导致内存泄露的类或者方法的。从上图中，可以直接看到`User`对象的数量很多，有时不容易看出来的，可以使用工具栏中的`group result by ` 可以以`super class`,`class loader`等排序，需要特别注意自定义的`classLoader`，如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g67byvu6xmj20mg09k3z1.jpg)

通常，如果Histogram视图展示的数量多的实例对象不是基础类型，而是用户自定义的类或者有嫌疑的类，就要重点关注查看。想进一步分析的，可以右键，选择使用`List objects` 或 `Merge Shortest Paths to GC roots`等功能继续钻取数据。其中`list objects`分别有`outgoing references`及`incoming references`，可以找出由这个对象出去的引用及通过哪些引用到这个对象的。`Merge Shortest Paths to GC roots`可以排除掉所有不是强引用的，找到这个对象的到`GC root`的引用路径。最终目的就是找到占用内存最大对象和无法回收的对象，计算从垃圾收集器根到相关对象的路径，从而根据这个对象路径去检查代码，找出问题所在。

## 4.4 `Dominator Tree`视图

点击`Overview`页面`Actions`区域内的“Dominator Tree视图”或点击工具栏的“open Dominator Tree”按钮，可以进入 `Dominator Tree`视图。该视图以实例对象的维度展示当前堆内存中`Retained Heap`占用最大的对象，以及依赖这些对象存活的对象的树状结构。如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g67cgfd7duj20qj075mxh.jpg)

视图展示了实例对象名、`Shallow Heap`大小、`Retained Heap`大小、以及当前对象的`Retained Heap`在整个堆中的占比。该图是树状结构的，当上一级的对象被回收，那么，它引用的子对象都会被回收，这也是`Dominator`的意义，当父节点的回收会导致子节点也被回收。通过此视图，可以很方便的找出占用`Retained Heap`内存最多的几个对象，并表示出某些objects的是因为哪些objects的原因而存活。本示例中，可以看出是由于`UserService`中的`ArrayList`引用的数组存储过量的`User`对象。

## 4.5 线程视图查看线程栈运行情况

点击工具栏的“齿轮”按钮，可以打开`Thread Overview`视图，可以查看线程的栈帧信息，包括线程对象/线程栈信息、线程名、`Shallow Heap`、`Retained Heap`、类加载器、是否Daemon线程等信息。结合内存Dump分析，看到线程栈帧中的本地变量，在左下方的对象属性区域还能看到本地变量的属性值。如按上面的示例（`shortest paths to GC root`），知道是由于线程`Thread-12`是`GC-root`占用内存大，在线程视图中，就可以着重看它的属性情况，如下图：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g67dbj4g6kj20qh0hsab3.jpg)

由上图可以看到此线程调用`WithOOM`方法，使用了变量`UserService`，而变量使用了`userList`，它包含了大量的`User`对象，占用`retained heap`很大。

# 5 总结

对于java应用的内存分析，需要对java应用的内存进行dump操作，生成内存快照后，使用`MAT`进行分析，找出大对象，找到内存泄漏或溢出的地方，从而分析代码，解决问题。本文对`MAT`的使用场景，基本概念，安装、使用进行了详细介绍，大家可以自己安装一下，写个小示例或者拿本文中的示例，实践一下。通过本文，希望可以帮助大家更方便，更有效率地分析内存，解决java应用的内存故障问题。

# 参数资料

- `MAT`官网：`https://www.eclipse.org/mat/`
- `MAT`下载：`http://www.eclipse.org/mat/downloads.php`
- `MAT`使用手册：`https://help.eclipse.org/2019-06/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html`
- Shallow and retained sizes：`https://www.yourkit.com/docs/java/help/sizes.jsp`
- 使用Eclipse Memory Analyzer Tool（MAT）分析线上故障(一) - 视图&功能篇：`https://www.cnblogs.com/trust-freedom/p/6744948.html`
- 示例代码地址：`https://github.com/mianshenglee/my-example/tree/master/java-monitor-example`

# 相关阅读

- [java应用监测(1)-java程序员应该知道的应用监测技术](https://mianshenglee.github.io/2019/08/23/java-monitor-1.html)
- [java应用监测(2)-java命令的秘密](https://mianshenglee.github.io/2019/08/24/java-monitor-2.html)
- [java应用监测(3)-这些命令行工具你掌握了吗](https://mianshenglee.github.io/2019/08/25/java-monitor-3.html)
- [java应用监测(4)-线上问题排查套路](https://mianshenglee.github.io/2019/08/26/java-monitor-4.html)
- [java应用监测(5)-可视化监测工具](https://mianshenglee.github.io/2019/08/27/java-monitor-5.html)
