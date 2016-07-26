---
layout: default
title: java服务安装(二)：使用commons-daemon
category: java
---

## 1、概述

### 1.1、为什么使用commons daemon
[上一篇][1]已对使用java service wrapper工具与java程序集成进行了讲解，java service wrapper使用简单，集成方法简单，不修改任何代码，一般情况下已满足需求。  

但是，java service wrapper只对java程序的开启及关闭进行操作，  若需要对程序启动前及关闭前进行一些自定义的操作（如启动时初始化工作，关闭时释放某些资源或进行特殊操作），此时就可以使用apache commons daemon了。

### 1.2、commons daemon介绍
Apache common deamon是用来提供java服务的安装，实现将一个普通的 Java 应用变成系统的一个后台服务，在linux下部署为后台运行程序，在windows部署为windows服务（著名的tomcat就是使用它实现启动及停止的。提供启动、停止、卸载等操作）。详细介绍可见[commons-daemon官网][2]。相对java service wrapper，commons daemon需要自己写少许代码，即按规定要求编写程序启动及关闭的入口类，仅此而已。

### 1.3、本文主要内容
本文主要讲解两部分内容：

> * linux下使用commons-daemon把java安装为后台运行程序
> * windows下使用commons-daemon把java安装为windows服务

## 2、程序示例
本文示例程序与[上一篇][3]的示例（log4j+java service wrapper）放在同一工程（因此上一篇的示例本程序也适用），不过改为使用logback+commons-daemon。程序结构如下：
![程序结构][4]  
说明：

> * 主要DaemonMainClassForLinux、DaemonMainClassForWindows及LogbackFileLogger类及logback.xml，其它的（WrapperMainClassForWindows、FileLogger及log4j.properties）是属于上一篇的示例。
> * install目录是用于对程序封装为服务的，即存放wrapper或daemon的，daemon目录下有linux及windows的安装内容
> * src/main/assembly下存放打包成自定义格式的maven配置（分别是linux及windows）
> * pom.xml配置打包插件信息

使用assembly打包，打包出windows包及linux包，包结构是bin,classes和lib三个目录，分别是安装目录，java程序，依赖包，如下所示：  
![打包结果][5]  
![包结构][6]

## 3、linux下使用commons-daemon
linux下使用commons-daemon主要通过commons-daemon主程序及jsvc实现。本示例环境是centos6.5。

### 3.1、下载commons-daemon
到[commons-daemon官网][7]下载，其中需要下载commons-daemon主程序和jsvc包（源码包）。如下图：  
![commons daemon下载][8]

 - 下载commons-daemon-1.0.15-bin.tar.gz，解压出commons-daemon-1.0.15.jar放到程序目录中（本示例为install/daemon），以便使用。
 - 下载commons-daemon-1.0.15-src.tar.gz源码包，此程序用于在linux下使用源码方式安装jsvc。（**提醒：在[官网的jsvc][9]只讲解了如休安装及使用，源码需要在这里下载**）。

### 3.2、安装jsvc
可先查看[官网的jsvc][10]，本示例中安装如下：把commons-daemon源码包放到/opt目录下。操作如下：

```sh
[root@localhost]# cd /opt/jsvc
[root@localhost jsvc]# unzip commons-daemon-1.0.15-src.zip
[root@localhost jsvc]# cd /opt/jsvc/commons-daemon-1.0.15-src/src/native/unix
[root@localhost unix]# ./configure --with-java=/usr/java/jdk1.8.0_51
*** All done ***
Now you can issue "make"
[root@localhost unix]# make
gcc   jsvc-unix.o libservice.a -ldl -lpthread -o ../jsvc
make[1]: 离开目录“/opt/jsvc/commons-daemon-1.0.15-src/src/native/unix/native”
```

说明

> * 确保linux上已安装java，安装需依赖java，如上述操作中`--with-java=/usr/java/jdk1.8.0_51`
> * make完后，会在commons-daemon-1.0.15-src/src/native/unix下生成jsvc文件如下图：
![jsvc][11]
> * 记住此jsvc路径，安装时使用。

### 3.3、编写程序入口类
程序实际的功能很简单，如下（见`LogbackFileLogger`类）：

```java
public class LogbackFileLogger {
	private static Logger logger = LoggerFactory.getLogger(LogbackFileLogger.class);

	public void logInfo2file() {
		for (int i = 0; i < 50; i++) {
			logger.info("我的测试：my test{}", i);
			try {
				//睡眠一秒以维持服务运行
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				logger.error(e.getMessage(),e);
			}
		}
	}
}
```

入口类只需要实现`init`,`destroy`,`start`,`stop`方法即可，安装时指定此类，jsvc会根据这些方法进行初始化、启动、关闭等操作。程序如下：

```java
/**
 * Linux服务安装入口类
 * 
 * @author mason
 *
 */
public class DaemonMainClassForLinux {
	public static void init(String[] args) {
		//日志输出到程序根目录(classpath)
		MainClass.initWorkDir();
	}

	public static void destroy() {
	}

	public static void start() {
		LogbackFileLogger logger = new LogbackFileLogger();
		logger.logInfo2file();
	}

	public static void stop() {
		System.exit(0);
	}
}
```

说明：

> * init方法有参数，destroy、start及stop方法无参。
> * 本示例中的初始化`MainClass.initWorkDir();`用于指定日志输出的目录。
> * `start`方法作启动入口，`stop`方法为关闭，本示例简单退出程序，若需要自定义操作，可在此方法编写即可。

### 3.4、编写安装脚本
安装脚本在示例install/daemon/linux下，主要设置程序名称，路径，jsvc路径，java路径，程序入口类，日志输出目录即可。详细如下：

```sh
#!/bin/sh
# description: jsw-test
# processname: jsw-test
# chkconfig: 234 20 80

# 程序名称
SERVICE_NAME=jsw-test
#程序路径获取相对路径,可填写绝对路径，如APP_HOME=/usr/local/bingo_cdp/deploy/bingo-cdp-timerjob-linux
APP_HOME=$(dirname $(pwd))
EXEC=/opt/jsvc/commons-daemon-1.0.15-src/src/native/unix/jsvc
JAVA_HOME=/usr/java/jdk1.8.0_51

#依赖路径
cd ${APP_HOME}
CLASS_PATH="$PWD/classes":"$PWD/lib/*"

#程序入口类
CLASS=main.DaemonMainClassForLinux

#程序ID文件
PID=${APP_HOME}/${SERVICE_NAME}.pid
#日志输出路径
LOG_OUT=${APP_HOME}/logs/${SERVICE_NAME}.out
LOG_ERR=${APP_HOME}/logs/${SERVICE_NAME}.err

#输出
echo "service name: $SERVICE_NAME"
echo "app home: $APP_HOME"
echo "jsvc: $EXEC"
echo "java home: $JAVA_HOME"
echo "class path: $CLASS_PATH"
echo "main class: $CLASS"

#执行
do_exec()
{
    $EXEC -home "$JAVA_HOME" -Dsun.jnu.encoding=UTF-8 -Dfile.encoding=UTF-8 -Djcifs.smb.client.dfs.disabled=false -Djcifs.resolveOrder=DNS -Xms512M -Xmx1024M -cp $CLASS_PATH -outfile $LOG_OUT -errfile $LOG_ERR -pidfile $PID $1 $CLASS
}

#根据参数执行
case "$1" in
    start)
        do_exec
        echo "${SERVICE_NAME} started"
            ;;
    stop)
        do_exec "-stop"
        echo "${SERVICE_NAME} stopped"
            ;;
    restart)
        if [ -f "$PID" ]; then
            do_exec "-stop"
            do_exec
            echo "${SERVICE_NAME} restarted"
        else
            echo "service not running, will do nothing"
            exit 1
        fi
            ;;
    status)
		ps -ef | grep jsvc
		;;
    *)
	    echo "usage: service ${SERVICE_NAME} {start|stop|restart|status}" >&2
	    exit 3
	    ;;
esac
```

一般还需要把程序作为系统服务开机启动，因此本示例中提供了`set_auto_start.sh`文件，注意修改相应的文件名称：

```sh
#!/bin/bash

DAEMON_HOME=$(pwd)
AUTO_RUN_FILE=${DAEMON_HOME}/jsw-test.sh
AUTO_RUN_FILE_NAME=jsw-test

echo "设置${AUTO_RUN_FILE_NAME}开机启动......"

#若文件不存在，报错、否则设置开机启动
if [ ! -f "${AUTO_RUN_FILE}" ]; then 
	echo "${AUTO_RUN_FILE} 不存在，请检查文件!"
else
	\cp -f ${AUTO_RUN_FILE} /etc/init.d/${AUTO_RUN_FILE_NAME}
	chmod 777 /etc/init.d/${AUTO_RUN_FILE_NAME} 
	chkconfig --add ${AUTO_RUN_FILE_NAME} 
	chkconfig ${AUTO_RUN_FILE_NAME} on 
fi

echo "设置${AUTO_RUN_FILE_NAME}开机启动结束"

```

### 3.5、程序打包及启动
 使用maven的assembly插件打包，流程可见[上一篇][12]，当前示例打linux的zip包，具体配置如下：
 pom.xml
 
```xml
<execution>
	<id>make-daemon-linux-zip</id>
	<phase>package</phase>
	<goals>
		<goal>single</goal>
	</goals>
	<configuration>
		<finalName>jsw-test</finalName>
		<appendAssemblyId>true</appendAssemblyId>
		<outputDirectory>${project.build.directory}</outputDirectory>
		<descriptors>
			<descriptor>src/main/assembly/daemon-linux-zip.xml</descriptor>
		</descriptors>
	</configuration>
</execution>
```
daemon-linux-zip.xml

```xml
<assembly>
	<id>deamon-linux</id>
	<formats>
		<format>zip</format>
	</formats>
	<includeBaseDirectory>false</includeBaseDirectory>
	<dependencySets>
		<dependencySet>
			<useProjectArtifact>false</useProjectArtifact>
			<outputDirectory>/lib</outputDirectory>
		</dependencySet>
	</dependencySets>
	<fileSets>
		<fileSet>
			<!-- 源目录，此处是把编译出来的class文件都输出到根目录下的classes目录-->
			<directory>${project.build.directory}/classes</directory>
			<outputDirectory>/classes</outputDirectory>
		</fileSet>
		<fileSet>
			<directory>install/daemon</directory>
			<includes>
		        <include>commons-daemon-1.0.15.jar</include>
		    </includes>
			<outputDirectory>/lib</outputDirectory>
		</fileSet>
		<fileSet>
			<!-- 此处是把daemon的脚本文件输出到根目录下的bin目录-->
			<directory>install/daemon/linux</directory>
			<includes>
		        <include>jsw-test.sh</include>
		        <include>set_auto_start.sh</include>
		    </includes>
			<outputDirectory>/bin</outputDirectory>
		</fileSet>
	</fileSets>
</assembly>
```

程序包(jsw-test-deamon-linux.zip)打出来后，即可放到linux下进行部署，部署过程如下：

```sh
[root@localhost opt]# unzip jsw-test-deamon-linux.zip -d jsw-test
[root@localhost opt]# cd jsw-test/bin
[root@localhost bin]# chmod 777 *.sh
[root@localhost bin]# ./jsw-test.sh start
```

说明：

> * 本示例中放到/opt下，解压，修改执行权限，执行。(`start`,`stop`,`restart`参数)。
> * 日志文件按脚本设置的目录，输出在/opt/jsw-test/logs，其下查看启动日志：jsw-test.err  jsw-test.out
> * /opt/jsw-test/classes/logs下查看程序使用logback的输出。

## 4、windows下使用commons-daemon
windows下使用commons-daemon安装服务跟linux下流程上差不多，差别是使用procrun，脚本的编写。

### 4.1、下载commons-daemon及procrun
跟linux一样，到[commons-daemon官网][13]下载。

 - 下载commons-daemon主程序（`commons-daemon-1.0.15-bin.zip`），解压出`commons-daemon-1.0.15.jar`，也可以使用前面下载的jar包。
 - 下载procrun，[官网的procrun页面][14]只对它的使用进行讲解，在哪里下载即没有提及，这里特别提醒一下，需要在[这里下载procrun][15]，下载`commons-daemon-1.0.15-bin-windows.zip`。解压出文件如下：
![procrun包解压][16]
复制`commons-daemon-1.0.15.jar`,prunmgr.exe及prunsrv.exe到安装目录（见本示例中为install及install/windows）。

### 4.2、编写程序入口类
跟linux下的入口类不同，入口类只需要实现`start`,`stop`方法，安装时指定此类，procrun会根据这些方法进启动、关闭操作。***注意：与linux的不同，此两个方法需要带参数，否则会报找不到start方法错误***，程序如下：

```java
/**
 * Windows服务安装入口类
 * 
 * @author mason
 *
 */
public class DaemonMainClassForWindows {

	public static void start(String[] args) {
		// 日志输出到程序根目录(classpath)
		MainClass.initWorkDir();
		LogbackFileLogger logger = new LogbackFileLogger();
		logger.logInfo2file();
	}

	public static void stop(String[] args) {
		System.exit(0);
	}
}
```

### 4.3、编写安装脚本
使用procrun安装成服务，需写bat脚本进行安装（见示例中的install/daemon/windows目录下的install.bat,uninstall.bat）。主要是设置服务名称，java路径，依赖类，入口类，prunsrv路径，日志路径。**注意jvm的大小参数请根据实际情况修改**详细如下：
install.bat：

```sh
@echo off

rem 设置程序名称
set SERVICE_EN_NAME=jsw-test
set SERVICE_CH_NAME=jsw测试程序

rem 设置java路径
set JAVA_HOME=E:\Program Files\Java\jdk1.8.0_51

rem 设置程序依赖及程序入口类
cd..
set BASEDIR=%CD%
set CLASSPATH=%BASEDIR%\classes;%BASEDIR%\lib\*
set MAIN_CLASS=main.DaemonMainClassForWindows

rem 设置prunsrv路径 
set SRV=%BASEDIR%\bin\prunsrv.exe

rem 设置日志路径及日志文件前缀
set LOGPATH=%BASEDIR%\logs

rem 输出信息
echo SERVICE_NAME: %SERVICE_EN_NAME%
echo JAVA_HOME: %JAVA_HOME%
echo MAIN_CLASS: %MAIN_CLASS%
echo prunsrv path: %SRV%

rem 设置jvm
if "%JVM%" == "" goto findJvm
if exist "%JVM%" goto foundJvm
:findJvm
set "JVM=%JAVA_HOME%\jre\bin\server\jvm.dll"
if exist "%JVM%" goto foundJvm
echo can not find jvm.dll automatically,
echo please use COMMAND to localation it
echo for example : set "JVM=C:\Program Files\Java\jdk1.8.0_25\jre\bin\server\jvm.dll"
echo then install service
goto end
:foundJvm

rem 安装
"%SRV%" //IS//%SERVICE_EN_NAME% --DisplayName="%SERVICE_CH_NAME%" "--Classpath=%CLASSPATH%" "--Install=%SRV%" "--JavaHome=%JAVA_HOME%" "--Jvm=%JVM%" --JvmMs=256 --JvmMx=1024 --Startup=auto --JvmOptions=-Djcifs.smb.client.dfs.disabled=false ++JvmOptions=-Djcifs.resolveOrder=DNS "--StartPath=%BASEDIR%" --StartMode=jvm --StartClass=%MAIN_CLASS% --StartMethod=start "--StopPath=%BASEDIR%" --StopMode=jvm --StopClass=%MAIN_CLASS% --StopMethod=stop --LogPath=%LOGPATH% --StdOutput=auto --StdError=auto
  
:end
```

uninstall.bat：

```sh
@echo off
cd..
set BASEDIR=%CD%
set SERVICE_NAME=jsw-test
set "SRV=%BASEDIR%\bin\prunsrv.exe"

%SRV% //DS//%SERVICE_NAME%

:end
```

一般安装好服务后，则可在控制面板－管理程序－服务中进行启动，关闭管理。若需要使用脚本启动，可使用prunmgr.exe进行管理，编写如下：

```sh
@echo off
cd..
set BASEDIR=%CD%
set SERVICE_NAME=prunmgr
set MONITOR_PATH=%BASEDIR%\bin\prunmgr.exe
echo start %SERVICE_NAME% 

%MONITOR_PATH% //MR//%SERVICE_NAME%

:end
```

### 4.4、程序打包及安装
使用maven的assembly插件打包，流程可见[上一篇][17]，当前示例打windows的zip包，具体配置如下：
 pom.xml
 
```xml
<execution>
	<id>make-daemon-win-zip</id>
	<phase>package</phase>
	<goals>
		<goal>single</goal>
	</goals>
	<configuration>
		<finalName>jsw-test</finalName>
		<appendAssemblyId>true</appendAssemblyId>
		<outputDirectory>${project.build.directory}</outputDirectory>
		<descriptors>
			<descriptor>src/main/assembly/daemon-win-zip.xml</descriptor>
		</descriptors>
	</configuration>
</execution>
```

daemon-win-zip.xml

```xml
<assembly>
	<id>daemon-win</id>
	<formats>
		<format>zip</format>
	</formats>
	<includeBaseDirectory>false</includeBaseDirectory>
	<dependencySets>
		<dependencySet>
			<useProjectArtifact>false</useProjectArtifact>
			<outputDirectory>/lib</outputDirectory>
		</dependencySet>
	</dependencySets>
	<fileSets>
		<fileSet>
			<!-- 源目录，此处是把编译出来的class文件都输出到根目录下的classes目录-->
			<directory>${project.build.directory}/classes</directory>
			<outputDirectory>/classes</outputDirectory>
		</fileSet>
		<fileSet>
			<directory>install/daemon</directory>
			<includes>
		        <include>commons-daemon-1.0.15.jar</include>
		    </includes>
			<outputDirectory>/lib</outputDirectory>
		</fileSet>
		<fileSet>
			<!-- 此处是把daemon文件全部输出到根目录下的install目录-->
			<directory>install/daemon/windows</directory>
			<outputDirectory>/bin</outputDirectory>
		</fileSet>
	</fileSets>
</assembly>
```

程序包(jsw-test-deamon-win.zip)打出来后，即可放到windows目录下进行部署，部署过程如下：

 - 解压jsw-test-deamon-win.zip到jsw-test-deamon-win目录，打开bin目录，运行install.bat。**注意：运行弹出窗口，运行时间有点长，需等待。**
 - 成功安装后，可在控制面板－管理程序－服务中进行启动，关闭管理。
 - 日志文件按脚本设置的目录，输出在jsw-test-daemon-win/logs，其下查看启动日志：jsw-test-stderr.2016-07-04.log,jsw-test-stdout.2016-07-04.log
 - jsw-test-daemon-win/classes/logs下查看程序使用logback的输出

## 5、附件
[jsw-test.zip][18]; 密码: `uryd`


  [1]: https://mianshenglee.github.io/java%E6%9C%8D%E5%8A%A1%E5%AE%89%E8%A3%85(%E4%B8%80)-%E4%BD%BF%E7%94%A8java-service-wrapper%E5%8F%8Amaven%E6%89%93zip%E5%8C%85/
  [2]: http://commons.apache.org/proper/commons-daemon/index.html
  [3]: https://mianshenglee.github.io/java%E6%9C%8D%E5%8A%A1%E5%AE%89%E8%A3%85(%E4%B8%80)-%E4%BD%BF%E7%94%A8java-service-wrapper%E5%8F%8Amaven%E6%89%93zip%E5%8C%85/
  [4]: http://ww4.sinaimg.cn/large/72d660a7gw1f5itjvd631j209n0oj41l.jpg
  [5]: http://ww4.sinaimg.cn/large/72d660a7gw1f5iut0cygxj206t01caa2.jpg
  [6]: http://ww4.sinaimg.cn/large/72d660a7gw1f5iutek36tj20b70gtmy9.jpg
  [7]: http://commons.apache.org/proper/commons-daemon/index.html
  [8]: http://ww2.sinaimg.cn/large/72d660a7gw1f5itczwl24j20ec06yaaz.jpg
  [9]: http://commons.apache.org/proper/commons-daemon/jsvc.html
  [10]: http://commons.apache.org/proper/commons-daemon/jsvc.html
  [11]: http://ww2.sinaimg.cn/large/72d660a7gw1f5iubijyqrj20me01ewf9.jpg
  [12]: http://blog.csdn.net/masson32/article/details/51802732
  [13]: http://commons.apache.org/proper/commons-daemon/index.html
  [14]: http://commons.apache.org/proper/commons-daemon/procrun.html
  [15]: http://www.apache.org/dist/commons/daemon/binaries/windows/
  [16]: http://ww3.sinaimg.cn/large/72d660a7gw1f5ixdqnaxkj20bm04wmxk.jpg
  [17]: https://mianshenglee.github.io/java%E6%9C%8D%E5%8A%A1%E5%AE%89%E8%A3%85(%E4%B8%80)-%E4%BD%BF%E7%94%A8java-service-wrapper%E5%8F%8Amaven%E6%89%93zip%E5%8C%85/
  [18]: http://pan.baidu.com/s/1jIPgLTk
