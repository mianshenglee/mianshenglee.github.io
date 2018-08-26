---
layout: post
title: java服务安装(三)：使用appassembler
category: 技术
tags: Java
keywords: 
description: 
---

## 1、概述

### 1.1、为什么使用appassembler
java程序打成jar包后，简单的使用命令行`java -jar`直接来启动应用，但这样很不优雅，部署人员需要自己写脚本，而且也不符合工程化要求。至少这个包打出来后，部署或者安装人员只需要根据实际情况修改配置，直接`start/stop`来启动即可。

之前介绍过对于java服务程序，使用jsw和commons-daemon进行服务安装（见我的两篇文章[《java服务安装(一)：使用java service wrapper及maven打zip包》][1]和[《java服务安装(二)：使用commons-daemon》][2]），这两种方法对于简单的java服务，相对来讲还是有点繁琐，对于spring boot而言，[官方网站][3]推荐`linux`脚本和`winsw`的方式，也比较繁琐。

后来经过查阅，发现有个appassembler的maven插件，完美解决了服务安装部署包的处理，配置好插件后，`mvn package`即可生成完整部署包，并且适用于linux和windows两种平台。

### 1.2、appassembler介绍
[appassembler官网][4]有介绍，总的来说，它是一个maven插件，用于生成启动关闭应用的脚本，同时集成了相应的需要启动应用的依赖（当然，jdk还是要自己安装的）。它适用于linux平台、windows平台。使用的也是jsw的支持。

### 1.3、本文主要内容
本文通过一个简单的java应用示例和`appassembler`的`maven`配置，实现部署安装包的导出。此示例配置也适用于基于spring boot的应用。

## 2、程序示例
本示例是使用java生成一个文件并写入内容。生成安装部署包后，在`linux`下可使用直接`start`及`stop`启动和关闭。在`windows`平台，可以注册为服务，从而以服务的形式启动和关闭。项目结构如下图：

![工程目录.png-11.9kB][5]

共三个文件`App`是启动入口，`FileGenerator`用于生成文件并输出内容，pom.xml是相应的配置。`App`和`FileGenerator`比较简单，读者下载示例，看一下知道，不再细说，主要说一下pom.xml的配置项以及打包出来的内容。

## 3、打包说明
最终打出来部署包的结果如下：

![部署包结构.png-3.6kB][6]

说明：部署包主要分为`bin`,`etc`,`lib`,`logs`，这是比较通用的目录了，分别是运行脚本文件目录，配置文件目录，依赖包目录，日志目录。当前目录名称可以修改。

### 3.1 appassembler配置说明

```
<plugin>
				<groupId>org.codehaus.mojo</groupId>
				<artifactId>appassembler-maven-plugin</artifactId>
				<version>2.0.0</version>
				<executions>
					<execution>
						<id>generate-jsw-scripts</id>
						<phase>package</phase>
						<goals>
							<goal>generate-daemons</goal>
						</goals>
						<configuration>
							<repositoryLayout>flat</repositoryLayout>
							<includeConfigurationDirectoryInClasspath>true</includeConfigurationDirectoryInClasspath>
							<!--指定日志目录-->
							<logsDirectory>logs</logsDirectory>
							<daemons>
								<daemon>
									<!-- 指定服务名称 -->
									<id>app1</id>
									<!-- 指定服务主入口类 -->
									<mainClass>me.mason.test.app_assembler.App</mainClass>
									<platforms>
										<platform>jsw</platform>
									</platforms>
									<generatorConfigurations>
										<generatorConfiguration>
											<generator>jsw</generator>
											<includes>
												<include>linux-x86-32</include>
												<include>linux-x86-64</include>
												<include>macosx-universal-64</include>
												<include>macosx-universal-32</include>
												<include>windows-x86-32</include>
												<include>windows-x86-64</include>
											</includes>
											<configuration>
												<!-- 指定配置文件目录 -->
												<property>
													<name>configuration.directory.in.classpath.first</name>
													<value>etc</value>
												</property>
												<!-- 指定依赖包目录 -->
												<property>
													<name>set.default.REPO_DIR</name>
													<value>lib</value>
												</property>
												<!-- 指定wrapper日志文件位置 -->
												<property>
													<name>wrapper.logfile</name>
													<value>logs/wrapper.log</value>
												</property>
											</configuration>
										</generatorConfiguration>
									</generatorConfigurations>
								</daemon>
							</daemons>
						</configuration>
					</execution>
				</executions>
			</plugin>
```

如上配置所示的注释，打包配置相对来说比较简单，指定`appassembler-maven-plugin`插件，在`package`阶段生成JSW运行脚本。其中，需要指定服务名称、服务主入口类，配置文件目录和依赖文件目录等。

### 3.2 生成部署包

- 使用maven打包，命令`mvn package`，即可生成安装包目录，位置是`target\generated-resources\appassembler\jsw`。
- 安装时，进入到`bin`目录下，linux下，`chmod +x`修改脚本运行权限，执行`app1 start`即可启动应用。
- windows下，需要以管理员身份运行`app1.bat`，使用如下：
``` 
rem 安装应用为服务
app1.bat install 

rem 启动服务
app1.bat start 

rem 关闭服务
app1.bat stop

rem 卸载服务
app1.bat remove 
```

## 4、下载示例
链接: https://pan.baidu.com/s/1An8nJcAIxvtN7ocAnrpW9w 密码: c2ae

  [1]: https://mianshenglee.github.io/2016/07/02/java_service%281%29-java_service_wrapper_and_maven_zip.html
  [2]: https://mianshenglee.github.io/2016/07/04/java_service%282%29-commons-daemon.html
  [3]: https://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html
  [4]: http://www.mojohaus.org/appassembler/appassembler-maven-plugin/
