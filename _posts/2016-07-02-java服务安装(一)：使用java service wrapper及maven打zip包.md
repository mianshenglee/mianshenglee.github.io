---
layout: default
title: java服务安装(一)：使用java service wrapper及maven打zip包
---

## 1、概述
使用java开发程序，在windows平台下，一般有web应用，后台服务应用，桌面应用：

>* web应用多数打成war包在web容器（如**tomcat,jetty**等）中运行
>* 桌面应用一般打成jar包或exe文件运行
>* 后台服务应用一般打成jar包，然后使用命令行（如java -jar xxx.jar）运行

前面两种运行方式在本文不作讨论，主要描述java开发的后台服务程序（如定时任务程序，文件处理，数据备份等）。

### 1.1、为什么要用服务形式运行
若使用命令行方式运行java程序，把命令写成脚本（如bat脚本）运行即可，但命令行方式有其不方便之处，如命令行窗口不能关闭，关闭即停止，因此维护人员容易误操作（关闭窗口使程序停止）;若服务器宕机或其它原因，程序往往无法在服务器重启时自动启动。在windows下，很多程序都是以服务的形式运行，这也符合windows的管理。因此，建议使用服务形式运行，操作方便。

### 1.2、如何让java程序以服务形式运行
有几种方法可以让java程序以服务形式运行：

> * [Java Service Wrapper][1]目前业界最知名、最成熟的解决方案，添加任何额外的代码即可使用，不足之处是收费、64位版本需要购买Licence，不过在64位使用32位的也关系不大（除非你的程序需要很大的运算量）。
> * [Yet Another Java Service Wrapper][2]类似JSW的开源实现版本，不过官方支持不怎么好。
> * [Apache Commons Daemon][3]著名的Apache Commons工具包的成员，按规则添加启动程序，再编写脚本实现。
> * 其它的（如WinRun4J，Launch4j）未使用过，更多可参考[java开源打包工具][4]

本文主要讲解使用java service wrapper把java程序作为windows的服务运行，它不需要添加任何代码，配置即可。

### 1.3、打包需求
java程序打包一般打成jar包，如果是供其它调用打包成一个jar包上传到仓库，其它人可以引用，这种方式可行，如果它是独立的程序，有其它的额外文件（如配置文件，本文中说的wrapper文件），打成jar包就比较难处理了，因此，更多的情况是把程序把成zip包以便传输，并规定好程序包目录结构，打包时打成一个zip包，解压即用。如下是我经常用到的一种包结构：
![程序包结构][5]
说明：

> * classes目录：存放所有java编译文件及资源配置文件
> * lib目录：存放所有程序使用到所有依赖jar
> * wrapper：所有存放wrapper相关的文件，包括运行脚本bin，wrapper的配置文件conf，wrapper使用的依赖lib，及日志存放目录
 
因此，需要使用maven把程序打成zip包，解压出来后就是上述的目录结构，可直接运行。

## 2、程序示例
按前面所说的要求，本文以下面的一个示例进行讲解，示例使用标准maven archetype结构，只实现了一个简单的文件写入内容的功能，使用jsw对程序进行包装，并把它使用maven打包成zip，解压后直接使用jsw的bin下的脚本安装或卸载服务。
![源码结构][6]
从图中可见，程序很简单，仅一个java类FileLogger用于写日志到文件，使用log4j(1.2.16版本，现在流行slf4j和logback了)输出日志内容。日志路径是当前的classpath下的logs目录。log4j使用参考它的[官网][7]，当前我们主要关注以下两点：

- wrapper文件夹:当前只配置windows，存放jsw的文件，以便把程序包装为服务安装。若是linux，可自行添加文件夹。
- pom.xml及assembly文件夹：用于maven配置按需打包成zip包，zip包内容如上面"打包需求"所示。

## 3、maven打zip包
### 3.1、maven-assembly-plugin介绍
maven-assembly-plugin是maven中用于构建发布包的插件，“assembly”是把一组文件、目录、依赖元素组装成一个归档文件，不仅支持创建二进制归档文件，也支持创建源码归档文件。目前Assembly插件支持如下格式的归档文件: 

- zip 打zip包
- tar.gz 打tar.gz包
- tar.bz2 打tar.bz2包
- jar 打jar包
- dir 直接打包目录
- war 打war包

使用方法也比较简单，可参考[maven官网的assembly][8]，一般是三个步骤

 1. 工程的pom.xml里配置Assembly插件。
 2. 自定义打包格式的描述符
 3. 运行"mvn package"或"mvn assembly:assembly"命令即可

### 3.2、maven-assembly-plugin配置
如下所示，在pom.xml文件中的build>plugins元素下配置assembly插件，请看注释说明：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<!-- assembly版本 -->
	<version>2.2.1</version>
	<executions>
		<!-- 若要同时打多个包（如windows和linux不同系统的包），可配置多个execution，此处只打zip，因此配置一个 -->
		<execution>
			<!-- id标识，唯一即可 -->
			<id>make-wrapper-win-zip</id>
			<!-- 设置package阶段进行 -->
			<phase>package</phase>
			<goals>
				<!-- 只运行一次 -->
				<goal>single</goal>
			</goals>
			<configuration>
				<!-- 输出的最终名称，自动添加格式后缀（如zip），当前示例为jsw-test.zip -->
				<finalName>jsw-test</finalName>
				<!-- 配置是否添加id到名称中，若为true，当前示例中，则为jsw-test-zip.zip，false即不添加，只是jsw-test.zip；
				若同时打多个包，则可设为true，分别添加id以作区分-->
				<appendAssemblyId>true</appendAssemblyId>
				<!-- 打包的输出目录，可自定义，${project.build.directory}为编译输出目录，即target目录 -->
				<outputDirectory>${project.build.directory}</outputDirectory>
				<descriptors>
					<!-- 使用的描述符，按此描述进行打包，此处配置一个zip.xml表示打zip包 -->
					<descriptor>src/main/assembly/wrapper-win-zip.xml</descriptor>
				</descriptors>
			</configuration>
		</execution>
	</executions>
</plugin>
```

assembly插件在pom.xml的配置比较简单，回答几个问题即可：

- 在什么时候打包：phase，
- 打包出来的名称是什么:finalName
- 是否添加id到名称后缀:appendAssemblyId
- 打包后输出到哪里:outputDirectory
- 使用哪个描述符进行打包操作:descriptor

可参考[maven官网的assembly][9]

### 3.3、描述符wrapper-win-zip.xml配置
前面讲到要使用一个描述符进行打包操作，即wrapper-win-zip.xml，此类文件可统一存放在目录src/main/assembly中，以便统一管理。wrapper-win-zip.xml的格式如下所示：

```xml
<assembly>
	<!-- id标识，唯一即可，若pom中的appendAssemblyId设置为true，则会添加此id作为后缀 -->
	<id>wrapper-win</id>
	<formats>
		<!-- 打包的格式 -->
		<format>zip</format>
	</formats>
	<!-- 打包的文件不包含项目目录，压缩包下直接是文件 -->
	<includeBaseDirectory>false</includeBaseDirectory>
	<!-- 配置依赖的输出 -->
	<dependencySets>
		<dependencySet>
			<!-- 是否把当前项目的输出jar包并使用，true则会把当前项目输出为jar包到输出目录,false不输出 -->
			<useProjectArtifact>false</useProjectArtifact>
			<!-- 依赖输出目录，相对输出目录的根目录，当前示例把依赖输出到lib目录 -->
			<outputDirectory>/lib</outputDirectory>
		</dependencySet>
	</dependencySets>
	<!-- 文件输出 -->
	<fileSets>
		<fileSet>
			<!-- 源目录，此处是把编译出来的class文件都输出到根目录下的classes目录-->
			<directory>${project.build.directory}/classes</directory>
			<!-- 输出目录 -->
			<outputDirectory>/classes</outputDirectory>
		</fileSet>
		<fileSet>
			<!-- 此处是把wrapper文件全部输出到根目录下的wrapper目录-->
			<directory>install/wrapper/windows</directory>
			<outputDirectory>/wrapper</outputDirectory>
		</fileSet>
	</fileSets>
</assembly>
```

详细参考[官网assembly的配置说明][10]
说明一下，按上述的配置，使用maven命令进行打包（`mvn package`），在target目录会输出的是一个**jsw-test-wrapper-win**包，当前此包名称不影响程序运行，读者可自行个性，包下面直接是三个文件夹（classes,lib,wrapper）。至此，即可以使用maven打出自定义的zip包。

## 4、jsw集成java后台服务
jsw在不添加任何代码的情况下可以直接使用，把java程序安装为windows服务，这样就可以随着系统的运行而自动运行。

### 4.1、jsw介绍与下载
到[java service wrapper官网][11]下载，它支持各种操作系统，按系统下载即可，这里讲解windows的，下载32位(64位的收费)。
![jsw下载][12]
下载解压后，内容如下：
![jsw解压内容][13]

- bin：wrapper运行文件及安装脚本
- conf：配置文件目录
- doc：说明文档
- lib：wrapper本身要用到的包和dll文件
- logs：日志目录
- src：wrapper提供的模板文件（包括bin脚本和conf文件），用户直接复制这里的再修改为自己的脚本即可。

### 4.2、添加jsw到java程序
java程序中添加jsw的步骤很简单，主要以下两步：

 1. 复制必要的wrapper文件到程序需要的目录中;只有四个目录是必要的："bin","conf","lib","logs"，如当前示例中，在main目录下新建wrapper目录，复制上面wrapper的的"bin","conf","lib","logs"这四个文件夹到此目录。去掉jsw的测试文件，最后结构如下：
 ![jsw结构][14]
 2. 修改conf/wrapper.conf文件
一般会把经常修改的作为变量放在前面，以便后面配置使用，如当前示例，会先设置以下变量

```
rem 程序目录位置
set.APP_HOME=../..
rem java目录位置
set.JAVA_HOME=E:/Program Files/Java/jdk1.8.0_51
rem 服务英文名称
set.SERVICE_EN_NAME=jsw-test
rem 服务中文名称
set.SERVICE_CH_NAME=jsw测试
rem 服务描述
set.SERVICE_DESCRIPTION=jsw测试
rem 你的Java应用程序的运行类（主类）
set.USER_MAIN_CLASS=service.FileLogger

```

然后主要设置以下配置（`%var%`为变量引用），其它配置按默认即可。如有个性化需求，可看[官方文档][15]

>* JVM位置：
wrapper.java.command=%JAVA_HOME%/bin/java
>* 你的Java应用程序的运行类（主类）
wrapper.app.parameter.1=%USER_MAIN_CLASS%
>* 你的Java程序所需的类路径：
wrapper.java.classpath.1=../lib/wrapper.jar
wrapper.java.classpath.2=%APP_HOME%/classes
wrapper.java.classpath.3=%APP_HOME%/lib/*
>* 你的Wrapper.DLL或wrapper.jar所在的目录
wrapper.java.library.path.1=../lib
>* 注册为服务的名称和显示名，你可以随意进行设置
wrapper.name=%SERVICE_EN_NAME%
wrapper.displayname=%SERVICE_CH_NAME%
wrapper.description=%SERVICE_DESCRIPTION%
>* 日志文件位置
wrapper.logfile=../logs/wrapper.log

配置完之后，使用bin下的脚本可进行相应的安装，卸载操作。

### 4.3、安装与卸载

 - 服务安装
运行InstallTestWrapper-NT.bat即可安装，在日志输出目录可查看日志检查是否正常启动。安装成功后可在控制面板－管理程序－服务中看到注册的服务名称（当前示例是jsw测试），并可进行启动、关闭等操作。若启动失败，则需根据日志输出检查（一般是配置问题）。
 - 服务卸载
 运行UninstallTestWrapper-NT.bat进行卸载服务。

### 4.4、打包并测试
使用maven打包，`mvn package`，按前面的配置，即可输出zip包，见前面的**程序示例**，把zip包放到服务，解压，即可使用wrapper安装服务。

## 5、附件

 - 源码：[jsw-test.zip][16]  ; 密码：`3hs5`


  [1]: http://wrapper.tanukisoftware.com/doc/english/download.jsp
  [2]: http://yajsw.sourceforge.net/
  [3]: http://commons.apache.org/proper/commons-daemon/index.html
  [4]: http://www.open-open.com/47.htm
  [5]: https://raw.githubusercontent.com/mianshenglee/dataStorage/master/md-photo/%E7%BC%96%E7%A8%8Bblog-jsw%E5%8F%8Amaven%E6%89%93zip/jsw-test-structure.png
  [6]: https://raw.githubusercontent.com/mianshenglee/dataStorage/master/md-photo/%E7%BC%96%E7%A8%8Bblog-jsw%E5%8F%8Amaven%E6%89%93zip/jsw-test-src-structure.png
  [7]: http://logging.apache.org/log4j/1.2/
  [8]: http://maven.apache.org/plugins/maven-assembly-plugin/usage.html
  [9]: http://maven.apache.org/plugins/maven-assembly-plugin/usage.html
  [10]: http://maven.apache.org/plugins/maven-assembly-plugin/assembly.html
  [11]: http://wrapper.tanukisoftware.com/doc/english/download.jsp
  [12]: https://raw.githubusercontent.com/mianshenglee/dataStorage/master/md-photo/%E7%BC%96%E7%A8%8Bblog-jsw%E5%8F%8Amaven%E6%89%93zip/download-jsw.png
  [13]: https://raw.githubusercontent.com/mianshenglee/dataStorage/master/md-photo/%E7%BC%96%E7%A8%8Bblog-jsw%E5%8F%8Amaven%E6%89%93zip/jsw-unzip.png
  [14]: https://raw.githubusercontent.com/mianshenglee/dataStorage/master/md-photo/%E7%BC%96%E7%A8%8Bblog-jsw%E5%8F%8Amaven%E6%89%93zip/jsw-structure.png
  [15]: http://wrapper.tanukisoftware.com/doc/english/introduction.html
  [16]: http://pan.baidu.com/s/1hs2QdJe
