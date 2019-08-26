---
layout: post
title: java应用监测(4)-线上问题排查套路
category: IT
tags: java troubleshooting monitor
keywords: 
description: 
---

tags: java, troubleshooting, monitor

---

> 一句话概括：java应用线上问题如CPU过高，内存溢出，IO过高等问题如何排查，本文为你详细讲述。

# 1 引言

java应用上线运行后，免不了会有各种问题，总的来说问题会分为四大类：
- （1）CPU相关问题
- （2）内存相关问题
- （3）磁盘及IO相关问题
- （4）业务代码问题。

针对这些问题，线上如何进行监测与问题排查，是一个java开发人员的必要技能。下面将结合前面提到的java命令行工具，对这几个问题的排查套路进行说明。

# 2 CPU问题排查套路

如果发现系统变慢变卡，应用响应变慢，首先要查的就是CPU使用情况，一般是进程占用CPU过高，因此需要监测CPU的占用情况，而java应用中与CPU相关的主要是线程的运行，因此具体到java应用，需要监测线程的运行状态，对应就是命令行工具`jstack`。因此，总结CPU占用过高问题可按下面套路：

```shell
# (1) 查询CPU占用高的进程ID(PID)
top -c

# (2) 了解此进程的启动参数
ps -ef|grep  PID
或者
jinfo -flags PID

# (3) 打印线程堆栈信息并输出文件
jstack -l PID > PID.dump

# (4) 根据进程查找线程ID（TID）
top -H -p PID

# (5) 获取TID的16进制数
printf "%x\n" TID

# (6) 结合TID和线程堆栈信息文件查找问题
- 可以使用文本工具直接查看
- 可以使用 grep TID -A20 PID.dump 来查看
- 需要配合线程状态来检查

```

关于`jstack`工具和线程状态可查看文章《java应用监测(3)-这些命令行工具你掌握了吗》

# 3 内存问题排查套路

内存问题主要是java应用在运行过程中发生OOM（out of memory），因此需要建议在java应用启动时，添加几个参数，包括`-Xloggc:file -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=logs/heapdump.hprof -XX:ErrorFile=logs/java_error_%p.log`。这样当发生oom时，可以从dump出来的文件来分析oom的原因。与内存问题相关的java命令行工具包括`jmap`,`jstat`，因此内存OOM问题排查套路如下：

```shell
# (1)找到java应用进程(PID)
jps -lvm
或者
top -c

# (2)了解此进程启动参数（特别是-Xms，-Xmx等）
ps -ef|grep  PID
或者
jinfo -flags PID

# (3) 确认内存情况
jmap -heap PID

# (4) 查找占内存的大对象
jmap -histo:live PID 

# (5) dump出堆文件，以便使用工具分析
jmap -dump:file=./heap.hprof PID

# (6) 查看GC变化情况，如下每秒打印一次
jstat -gc PID 1000 

# (7) 结合日志文件出错信息及dump出来的堆文件分析OOM和GC情况
- 内存分配小，适当调整内存
- 对象被频繁创建，且不释放，优化代码
- young gc频率太高，查看-Xmn、-XX:SurvivorRatio等参数设置是否合理
```

关于OOM，[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)有关于OOM的说明（`https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html`），
主要分为以下几大类：

- `java.lang.OutOfMemoryError: Java heap space`，堆的内存占用已经达到`-Xmx`设置的最大值，无法创建新对象，简单的可以考虑通过调整`-Xmx`参数来解决。
- `java.lang.OutOfMemoryError: GC Overhead limit exceeded`，表示GC一直在执行且java进程运行很慢，通常会抛出此异常，java堆的分配的空间很小以至于新数据无法放到堆中。考虑调整堆大小，如果想关闭此输出，可用参数来关闭`-XX:-UseGCOverheadLimit`。
- `java.lang.OutOfMemoryError: Requested array size exceeds VM limit`，java应用尝试分配大于堆大小的数组，如堆大小是256M，却要分配512M的数组，则会报错。考虑调整堆大小或者修改代码
- `java.lang.OutOfMemoryError: Metaspace`，当类元数据所需的本机内存量超过时MaxMetaSpaceSize时报出，考虑调整MaxMetaSpaceSize。
- `java.lang.OutOfMemoryError: request size bytes for reason. Out of swap space?`当来自本机堆的分配失败并且本机堆可能接近耗尽时会报此错误，需要查看日志来处理。
- `java.lang.OutOfMemoryError: Compressed class space`，JVM的非堆结构中，类指针存放空间不足，考虑使用`CompressedClassSpaceSize`来调整。
- `java.lang.OutOfMemoryError: reason stack_trace_with_native_method`，JVM的本地方法区不足，在Java本机接口（JNI）或本机方法中检测到分配失败，需要查找对应堆栈信息来查询。

# 4 磁盘及IO问题排查套路

java应用运行过程中，会涉及日志产生，对磁盘的读写等操作，也有可能有各种问题，如磁盘不足（日志输出过多）、、磁盘读写IO比较慢、IO过于频繁等。一般来说，可以按以下套路进行排查：

```shell
# (1) 查看磁盘容量情况
df -h

# (2) 查看文件大小和目录大小
ls -l 或者直接ll
du -h --max-depth=1

# (3) 查看IO情况，找到IO读写频繁的进程PID
iotop -d 1 # 1秒打印一次
或者
iostat -d -x -k 1 #1秒打印一次

# (4) 使用stack打印线程堆栈信息，排查IO相关代码

# (5) 有时候若想测试磁盘的读写速度（特别是虚拟机），可以使用dd
# 示例：测数据卷挂载目录的纯写速度
dd if=/dev/zero of=/数据卷目录/test.iso bs=8k count=1000000

```

# 5 业务问题排查套路

业务问题，主要是涉及到代码逻辑层面的，主要是查询日志输出，方法是否按正确的逻辑执行，因此一般的排查套路如下：

```shell
# (1) 实时日志输出查询
tail -fn 100 log_file

# (2) 根据日志输出的关键字来定位问题
grep keyWord log_file # 关键字所在行
grep -C n keyWord log_file # 关键字所在前后n行

# (3) 日志文件使用可视化文本工具分析（notepad++，sublime，大文件查看如EmEditor）

# (4) 使用线上工具直接检测方法的参数、返回值，异常情况等等，如Btrace，arthas等。

```

关于java线上问题的诊断工具包括Btrace及arthas，本系列后续会有相应的文章进行介绍。


# 6 总结

本文对java应用线上遇到的问题分为四大类，分别是（1）CPU相关问题，（2）内存相关问题，（3）磁盘及IO相关问题，（4）业务代码问题。针对各种问题，按照一定的套路，结合java的命令行工具和线上诊断工具，可以很方便地给java应用进行排查。希望本文给相应的java开发人员有帮助。


# 参考资料

- [Understand the OutOfMemoryError Exception](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)：`https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html`

- [这部技术葵花宝典真的很硬核](https://mp.weixin.qq.com/s/NJPXFMgbwXWkzVLDK12Gfg)：`https://mp.weixin.qq.com/s/NJPXFMgbwXWkzVLDK12Gfg`
- [目前最全的Java服务问题排查套路](https://mp.weixin.qq.com/s/SuFPeWxtjHdXAcu6hkmBlA)：`https://mp.weixin.qq.com/s/SuFPeWxtjHdXAcu6hkmBlA`
- [JDK工具参考文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix)：`https://docs.oracle.com/javase/8/docs/technotes/tools/unix`
- [示例代码地址](https://github.com/mianshenglee/my-example/tree/master/java-monitor-example)：`https://github.com/mianshenglee/my-example/tree/master/java-monitor-example`

# 相关阅读

- [java应用监测(1)-java程序员应该知道的应用监测技术](https://mianshenglee.github.io/2019/08/23/java-monitor-1.html):`https://mianshenglee.github.io/2019/08/23/java-monitor-1.html`
- [java应用监测(2)-java命令的秘密](https://mianshenglee.github.io/2019/08/24/java-monitor-2.html):`https://mianshenglee.github.io/2019/08/24/java-monitor-2.html`
- [java应用监测(3)-这些命令行工具你掌握了吗](https://mianshenglee.github.io/2019/08/25/java-monitor-3.html):`https://mianshenglee.github.io/2019/08/25/java-monitor-3.html`