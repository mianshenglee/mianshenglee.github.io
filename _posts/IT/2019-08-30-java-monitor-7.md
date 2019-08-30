---
layout: post
title: java应用监测(7)-在线动态诊断神器BTrace
category: IT
tags: java troubleshooting monitor btrace
keywords: 
description: 
---

tags: java, troubleshooting, monitor,btrace

---


> 一句话概括：`BTrace`是一个是强大的java线上应用检测工具（动态追踪工具），可以在不修改应用代码，不停应用服务的前提下检测代码运行情况，进而诊断问题，是生产环境下必备神器，本文将对它的使用进行讲解。

# 1 引言

`BTrace`是一款[开源软件](https://github.com/btraceio/btrace)，`github`地址为：`https://github.com/btraceio/btrace`，官网的介绍是`BTrace is a safe, dynamic tracing tool for the Java platform.`，它是安全的动态追踪java应用的工具，即可以动态地向目标应用的字节码注入追踪代码。何为动态？我们都知道，即在java应用启动的时候会把`class`文件加载到`JVM`运行，此时`class`代码功能是确定、静态的（无法变更），要想修改，只能是修改代码，重新编译、部署、启动。

而在处理线上应用时，我们经常需要查看代码运行情况，参数值、返回值查看，或者添加自己需要调试的日志等，在开发阶段，添加日志，重新启动没有问题，但在生产环境就不适用了（生产环境一般不轻易关停服务，而且即使可以重启，可能发生问题的现场就破坏了，无法重现问题），那么是否有方法在java应用运行期间，不重启程序的情况，动态加入自己想要监测（追踪）的内容？`Btrace`就是这样一个动态追踪神器，可以在不用重启的情况下监控应用运行情况，可以获取程序运行时的数据信息，如方法参数、返回值、全局变量和堆栈信息等。本文就是对`BTrace`进行运行原理和使用进行描述。

# 2 BTrace运行原理

## 2.1 class文件的动态修改替换

`BTrace`是基于java的动态追踪技术来实现的。对于java开发人员，都清楚java程序的开发流程是写java代码，把它编译为class文件，然后在`JVM`中加载class运行。若此时想要在不停止应用的情况下对`class`进行修改来添加追踪内容，如在某个方法(`method`)中添加输出信息，主要是两件事情：

- (1)修改已经加载到`JVM`中的class，添加自定义输出
- (2)替换运行在JVM`中的class

第一步，修改，由于`JVM`运行的都是class文件，是不是可以直接修改字节码`class`文件就行了（当然，字节码文件的可读性远远没有Java代码高），但是已经有相应的框架可以做这件事，就是`ASM`，利用这框架，可以直接编辑字节码的框架，它也提供接口可以让我们方便地操作字节码文件，进行注入修改类的方法，动态创造一个新的类等等。`Spring`就是使用这种技术来实现动态代理的。

第二步，替换，如果对它进行替换，则需要用到java提供的`java.lang.instrument.Instrumentation`，它有两个接口`redefineClasses`和`retransformClasses`，`redefineClasses`是自己提供字节码文件替换掉已存在的class文件，`retransformClasses`是在已存在的字节码文件上修改后再替换。不过需要注意的是`instrument`的使用有限制的（不能添加、修改、删除已经有字段和方法，不能改变方法签名，改变继承属性等）：

```
The redefinition must not add, remove or rename fields or methods, change the signatures of methods, or change inheritance
```



## 2.2 BTrace的模块与运行流程

BTrace就是基于前面的技术来实现的，文章[《Java动态追踪技术探究》](<https://mp.weixin.qq.com/s/_hSaI5yMvPTWxvFgl-UItA>)（`https://mp.weixin.qq.com/s/_hSaI5yMvPTWxvFgl-UItA`）对动态追踪技术进行了详细说明，下面简要说明一下。

### 2.2.1 主要模块

- BTrace脚本：利用BTrace定义的注解，我们可以很方便地根据需要进行脚本的开发。

- Compiler：将BTrace脚本编译成BTrace class文件。

- Client：将class文件发送到Agent。

- Agent：基于Java的`Attach API`，Agent可以动态附着到一个运行的`JVM`上，然后开启一个`BTrace Server`，接收client发过来的BTrace脚本；解析脚本，然后根据脚本中的规则找到要修改的类；修改字节码后，调用`Java Instrument`的`retransform`接口，完成对对象行为的修改并使之生效。

### 2.2.2 运行流程

运行流程图如下：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g68i8ma6o7j20mf0hdtck.jpg)

跟java源码一样，先编写`Btrace`脚本（也是java文件），编译（`compiler`），通过client发送给`agent`，`agent`通过`attach api`添加到JVM并启动`agent server`来接收`client`发送过来的内容，然后底层是使用`ASM`修改字节码文件，之后使用`Java Instrument`的`retransform`接口替换修改后的class文件，运行后的输出再通过`agent`发送到`client`进行显示。

# 3 BTrace安装

知道了`BTrace`的运行原理，现在可以安装实践一下。本文用的示例还是`java-monitor-example`。`BTrace`的安装很简单，开箱即用。

- 下载地址（当前最新版本是`[v1.3.11.3]`）：`https://github.com/btraceio/btrace/releases`
- 解压到需要监测的java应用所在服务器中
- `btrace`的命令在`bin`目录 下
- 若需要在任意目录可执行，需要把`btrace`设置到环境变量中(`export`)

# 4 BTrace适用场景

基本上，`BTrace`只适用于动态追踪类的输出信息，不能添加属性、删除方法，修改继承等，这跟前面提到的`Instrument`的限制是一致的。一般来说，使用`Btrace`进行线上应用监测，基于都属于日志输出类，多数包括以下几大场景：

- 查看某一个方法中入参和返回值
- 查看某一个方法的响应时间
- 查看某行代码是否有执行到
- 打印系统参数或JVM启动参数
- 打印方法调用的线程堆栈
- 出现异常时打印出现异常信息

# 5 BTrace使用

`Btrace`作为一个独立运行的工具，默认只能在本地运行，也就是说，想要监测哪个正在运行的java应用，就需要把它解压到对应的服务器。本示例中运行的是`java-monitor-example`作为需要监测的java应用，然后就是根据监测业务需求，写脚本，运行脚本，查看输出了。

## 5.1 脚本编写

### 5.1.1 注解与BTraceUtils

`Btrace`的脚本与编写java代码无异，不过相对简单很多，主要是使用`Btrace`提供的注解和`BTraceUtils`，注解用于告诉`Btrace`需要拦截的类、拦截时机、拦截位置等，`BTraceUtils`用于提供打印输出种信息的功能。如官网给出的示例如下：

```java
package samples;

import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;

/**
 * This script traces method entry into every method of 
 * every class in javax.swing package! Think before using 
 * this script -- this will slow down your app significantly!!
 */
@BTrace public class AllMethods {
    @OnMethod(
        clazz="/javax\\.swing\\..*/",
        method="/.*/"
    )
    public static void m(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod) {
        print(Strings.strcat("entered ", probeClass));
        println(Strings.strcat(".", probeMethod));
    }
}
```

以上代码，表示，会拦截所有调用以`javax.swing`开头的方法，然后打印出类名和方法名。可以注意到注解有`@BTrace`、`@OnMethod`、`@ProbeClassName`，`@ProbeMethodName`，而`print`，`println`是`BTraceUtils`提供的静态方法。`BTraceUtils`还提供了很多打印方法(后面示例会提到)。另外，还要注意的是跟踪操作都需要在静态方法体内指定，因此都需要`static`方法。

另外，关于`BTrace`提供的注解，详细可以参考[官方文档](https://github.com/btraceio/btrace/wiki/BTrace-Annotations)（`https://github.com/btraceio/btrace/wiki/BTrace-Annotations`）。主要包括以下：

```java
/**Class Annotations*/
@com.sun.btrace.annotations.DTrace
@com.sun.btrace.annotations.DTraceRef
@com.sun.btrace.annotations.BTrace
/**Method Annotations*/
@com.sun.btrace.annotations.OnMethod
@com.sun.btrace.annotations.OnTimer
@com.sun.btrace.annotations.OnError
@com.sun.btrace.annotations.OnExit
@com.sun.btrace.annotations.OnEvent
@com.sun.btrace.annotations.OnLowMemory
@com.sun.btrace.annotations.OnProbe
/**Argument Annotations*/
@com.sun.btrace.annotations.Self
@com.sun.btrace.annotations.Return
@com.sun.btrace.annotations.CalledInstance
@com.sun.btrace.annotations.CalledMethod
/**Field Annotations*/
@com.sun.btrace.annotations.Export
@com.sun.btrace.annotations.Property
@com.sun.btrace.annotations.TLS

```

其中，`@OnMethod`用得比较多，需要重点说明一下，它主要是三个属性`clazz`，`method`和`location`。



- `clazz`：类的全路径名，如`me.mason.monitor.controller.UserController`

- `method`：要监测的方法名，如`getUsers`

- `location`：拦截时机，使用`@Location`注解。

`@Location`又有以下几种：

- `Kind.ENTRY`：在进入方法时调用
- `Kind.RETURN`：方法执行完时调用，只有把拦截位置定义为`Kind.RETURN`，才能获取方法的返回结果`@Return`和执行时间`@Duration`
- `Kind.CALL`：方法中调用其它方法时调用
- `Kind.LINE`：通过设置line，可以监控代码是否执行到指定的位置
- `Kind.ERROR, Kind.THROW, Kind.CATCH`：异常情况的跟踪

### 5.1.2 关于编写

建议还是使用java的maven项目的开发环境进行编写，可以使用代码提示功能。写好后再放到对应需要监测的服务器中。不过编辑时需要引用对应的jar包(`btrace-agent`,`btrace-boot`,`btrace-client`)，对应的jar在下载的安装下的`build`目录下。通过`pom.xml`引入即可使用。如下所示：

```xml
<!-- BTrace -->
<dependency>
    <groupId>com.sun.btrace</groupId>
    <artifactId>btrace-agent</artifactId>
    <version>1.3.11.3</version>
    <type>jar</type>
    <scope>system</scope>
    <systemPath>E:/btrace-bin-1.3.11.3/build/btrace-agent.jar</systemPath>
</dependency>
<dependency>
    <groupId>com.sun.btrace</groupId>
    <artifactId>btrace-boot</artifactId>
    <version>1.3.11.3</version>
    <type>jar</type>
    <scope>system</scope>
    <systemPath>E:/btrace-bin-1.3.11.3/build/btrace-boot.jar</systemPath>
</dependency>
<dependency>
    <groupId>com.sun.btrace</groupId>
    <artifactId>btrace-client</artifactId>
    <version>1.3.11.3</version>
    <type>jar</type>
    <scope>system</scope>
    <systemPath>E:/btrace-bin-1.3.11.3/build/btrace-client.jar</systemPath>
</dependency>
```

## 5.2 脚本运行

打印帮助信息如下：

![](http://ww1.sinaimg.cn/large/72d660a7gy1g68lcglqfwj20n106edg7.jpg)

一般来说，在服务器上，直接是`btrace PID btraceFile.java`，然后查看输出（也可以把内容输出到文件中再查看，如`btrace PID btraceFile.java > info.txt`）。如果有使用到特定的jar包，则需要把参数`cp`或`classpath`加上。如下示例是把调用方法的返回值进行输出：



![](http://ww1.sinaimg.cn/large/72d660a7gy1g68lgadh3wj20p101ea9y.jpg)

## 5.3 脚本示例

下面通过几个常用的示例来说明一下`BTrace`脚本的使用，脚本在示例工程`java-monitor-example`中的`btrace`目录下。`java-monitor-example`中，分别是一个`controller`和`service`，有如下方法定义，下面会根据这些方法进行动态追踪。

```java
/**
  * UserController.java
  **/
@GetMapping("/user")
public ResponseResult<User> getUser() {
    User user = userService.getUser();
    return ResponseResult.ok(user);
}

@GetMapping("/users")
public ResponseResult<User> getUsers(int num) {
    List<User> users = userService.getUsers(num);
    return ResponseResult.ok(users);
}

/**
  * UserService.java
  * 根据ID获取用户
  *
  * @return
  */
public User getUser() {
    return mockUser();
}

/**
  * 获取用户数组
  *
  * @return
  */
public List<User> getUsers(int num) {
    userList.clear();
    for(int i=0 ; i < num; i++){
        userList.add(mockUser());
    }
    return userList;
}
```



### 5.3.1 打印方法相关信息

- 打印调用方法时的参数(调用`UserController`的`getUsers`方法时打印)

```java
@OnMethod(clazz = "me.mason.monitor.controller.UserController"
          ,method = "getUsers",location = @Location(Kind.ENTRY))
public static void readFunction(@ProbeClassName String className, @ProbeMethodName String methodName, AnyType[] args) {
    // 打印时间
    BTraceUtils.println(BTraceUtils.Time.timestamp("yyyy-MM-dd HH:mm:ss"));
    BTraceUtils.println("method controller");
    BTraceUtils.printArray(args);
    BTraceUtils.println(className + "," + methodName);
    BTraceUtils.println("==========================");
}
```

- 打印调用方法时的返回值

```java
@OnMethod(clazz = "me.mason.monitor.service.UserService"
,method = "getUsers",location = @Location(Kind.RETURN))
public static void printReturnData1(@Return AnyType result){
    BTraceUtils.println(BTraceUtils.Time.timestamp("yyyy-MM-dd HH:mm:ss"));
    BTraceUtils.printFields(result);
    BTraceUtils.println("==========================");
    BTraceUtils.println(BTraceUtils.str(result));
    BTraceUtils.println("==========================");
}
```
- 执行到的行数(查看是否执行到`UserService`的39行)

```java
@OnMethod(clazz = "me.mason.monitor.service.UserService"
,method = "getUsers",location = @Location(value = Kind.LINE,line = 39))
public static void printLineData(@ProbeClassName String className, @ProbeMethodName String methodName,int line){
    BTraceUtils.println(BTraceUtils.Time.timestamp("yyyy-MM-dd HH:mm:ss"));
    BTraceUtils.println(className + "," + methodName + ","+line);
    BTraceUtils.println("==========================");
 }
```
- 执行方法的用时(`UserController`的`getUsers`方法用时多长)

```java
@OnMethod(clazz = "me.mason.monitor.controller.UserController"
,method = "getUsers",location = @Location(Kind.RETURN))
public static void getUsersDuration(@Duration long duration){
    BTraceUtils.println(BTraceUtils.Time.timestamp("yyyy-MM-dd HH:mm:ss"));
    BTraceUtils.println("time(ns):" + duration);
    BTraceUtils.println("time(ms):" + BTraceUtils.str(duration / 1000000));
    BTraceUtils.println("time(s):" + BTraceUtils.str(duration / 1000000000));
    BTraceUtils.println("==========================");
}
```


### 5.3.2 打印系统属性及JVM属性

类似`JDK`的命令行工具`jinfo`，另外`jmap`及`jstatck`可查询官方示例。

```java
@BTrace
public class JInfo {
    static {
        println("System Properties:");
        printProperties();
        println("VM Flags:");
        printVmArguments();
        println("OS Enviroment:");
        printEnv();
        exit(0);
    }
}
```

### 5.3.3 打印异常输出

`java`开发人员应该都知道，`java`的异常分为`Error`和`Exception`，而它们都是`Throwable`的子类，即`java`中所有异常的父类都`Throwable`，因此追踪这个的构造函数，然后把堆栈打印出来即可。如下：

```java
//局部变量存储异常
@TLS static Throwable currentException;
//异常构造函数开始
@OnMethod(
    clazz="java.lang.Throwable",
    method="<init>"
)
public static void onthrow(@Self Throwable self) {
    currentException = self;
}
//异常构造函数结束，输出堆栈
@OnMethod(
    clazz="java.lang.Throwable",
    method="<init>",
    location=@Location(Kind.RETURN)
)
public static void onthrowreturn() {
    if (currentException != null) {
        Threads.jstack(currentException);
        println("=====================");
        currentException = null;
    }
}
```



## 5.4 脚本限制
`BTrace`对JVM来说是“只读的”，BTrace要做的是，虽然修改了字节码，但是主要是输出需要的信息，对整个程序的正常运行并没有影响。需要注意的是，由于是动态替换class文件，被修改的字节码是不会自动还原的。官方文档也有说明，`BTrace`脚本会有以下限制：

- 不允许创建对象

- 不允许创建数组

- 不允许抛异常

- 不允许catch异常

- 不允许随意调用其他对象或者类的方法，只允许调用com.sun.btrace.BTraceUtils中提供的静态方法（一些数据处理和信息输出工具）

- 不允许改变类的属性

- 不允许有成员变量和方法，只允许存在static public void方法

- 不允许有内部类、嵌套类

- 不允许有同步方法和同步块

- 不允许有循环

- 不允许随意继承其他类（当然，java.lang.Object除外）

- 不允许实现接口

- 不允许使用assert

- 不允许使用Class对象

  

# 6 一些经验

- 搭建使用java的maven项目的开发环境进行脚本编写，引入相应的jar，以提供代码提示功能。
- 查看官方提供的例子，在下载包中已提供例子，位置：`btrace-bin-1.3.11.3\samples`目录
- `BTrace`脚本中追踪的输入参数，返回值类型是简单类型直接使用（如int ,float等），复杂类型可以使用`AnyType`，但如果是使用自定义包中的类型（如User），则需要运行脚本时添加`cp`或`classpath`参数，指定自定义包。
- 一般简单类型或字符串，直接使用`print`或`println`，打印对象属性可使用`printFields`，打印`List`，可以使用`BTraceUtils.println(BTraceUtils.str(list))`
- 在探查方法的最后一行打印分隔，强烈建议。可能是由于输出有缓冲区延迟，如果不输出分隔，有可能会无法输出或者输出后内容没有分隔。分隔可使用`BTraceUtils.println`或`BTraceUtils.println("============")`。

# 7 总结

对于线上的java应用，如果想不停服务进行日志输出来诊断问题，动态追踪技术是必不可少的技术，而`Btrace`是使用此技术来实现动态追踪的有力工具。本文从`Btrace`的运行原理、安装、适用场景、脚本编写、运行等方面进行了详细描述，希望可以帮助大家加深`Btrace`的了解，更方便、有效率地解决线上问题。

# 参数资料

- `BTrace`官网：`https://github.com/btraceio/btrace`
- `BTrace`注解：`https://github.com/btraceio/btrace/wiki/BTrace-Annotations`
- 示例代码地址：`https://github.com/mianshenglee/my-example/tree/master/java-monitor-example`
- Java动态追踪技术探究：`https://mp.weixin.qq.com/s/_hSaI5yMvPTWxvFgl-UItA`
- BTrace原理浅析：`https://www.rowkey.me/blog/2016/09/20/btrace/`

# 相关阅读

- [java应用监测(1)-java程序员应该知道的应用监测技术](https://mianshenglee.github.io/2019/08/23/java-monitor-1.html)
- [java应用监测(2)-java命令的秘密](https://mianshenglee.github.io/2019/08/24/java-monitor-2.html)
- [java应用监测(3)-这些命令行工具你掌握了吗](https://mianshenglee.github.io/2019/08/25/java-monitor-3.html)
- [java应用监测(4)-线上问题排查套路](https://mianshenglee.github.io/2019/08/26/java-monitor-4.html)
- [java应用监测(5)-可视化监测工具](https://mianshenglee.github.io/2019/08/27/java-monitor-5.html)
- [java应用监测(6)-第三方内存分析工具MAT](https://mianshenglee.github.io/2019/08/29/java-monitor-6.html)
