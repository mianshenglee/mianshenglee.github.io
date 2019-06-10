---
layout: post
title: 调度与监控-spring batch(7)结合xxl-job进行批处理
category: IT
tags: SpringBatch
keywords: 
description: 
---

# 调度与监控-spring batch(7)结合xxl-job进行批处理

tags: springbatch

---

# 1.引言

经过前面几篇文章对`Spring Batch`的介绍，同时结合示例，从最简单的`helloworld`字符串输出，到读取文件到数据库的数据同步，然后是数据库到数据库，接着结合`BeetlSql`进一步简化数据库读写，再通过动态参数绑定实现增量同步，由浅到深，已经可以基本满足数据抽取，数据同步的工作了，下面是之前的文章列表：

- [数据批处理神器-Spring Batch(1)简介及使用场景][1]
- [快速了解组件-spring batch(2)之helloworld][2]
- [快速使用组件-spring batch(3)读文件数据到数据库][3]
- [决战数据库-spring batch(4)数据库到数据库][4]
- [便捷的数据读写-spring batch(5)结合beetlSql进行数据读写][5]
- [增量同步-spring batch(6)动态参数绑定与增量同步][6]

批处理任务，一般做为后台服务定时运行，服务运维工作则是批处理服务之外的一项重要工作，包括服务运行如何监控，任务运行统计，任务执行日志查看等等。前面已说过，`Spring Batch`是批处理框架，并不是调度框架。调度框架,简单点可以使用`quartz`，`crontab`，而市面上已经有比较多功能完善的调度框架，经过比较，个人感觉`xxl-job`是比较容易上手，而且功能比较齐全，因此，这里介绍一下，使用`xxl-job`对`Spring Batch`的批处理任务进行调度，实现监控功能。

# 2.xxl-job介绍

[`xxl-job`][7]是一个轻量级的分布式任务调度平台,其核心设计目标是开发迅速、学习简单、轻量级、易扩展。系统架构设计合理，把调度系统与执行器分离，调度系统负责调度相关的逻辑，执行器则是具体任务的实现逻辑，开发者可以直接使用调度系统，然后实现自己任务逻辑来做为执行器即可，做到开箱即用。具体使用方式及详细说明，可见其[官方文档][8]

按照`xxl-job`的文档，先安装`xxl-job`的数据库，然后把`xxl-job-admin`启动起来，启动起来的页面如下：

![xxl-job界面][9]


# 3.编写Spring Batch执行器

本示例是基于上一篇文章所用到的[数据库增量同步的示例][10]，只需要在原来的基础上添加一个执行器即可，本示例为`spring-batch-xxl-executor`。按照`xxl-job`的官方文档，里面有说明如何[配置部署“执行器项目”][11]。具体到本实例，配置，实现如下：

## 3.1 添加maven依赖
添加`xxl-job`的核心依赖，如下：

```
<!-- xxl-job-core -->
<dependency>
	<groupId>com.xuxueli</groupId>
	<artifactId>xxl-job-core</artifactId>
	<version>2.0.2</version>
</dependency>
```

## 3.2 添加执行器配置文件`executor.properties`
执行器配置需要填写包括执行器名称、调度系统地址等，如下：

```
### xxl-job admin address list, such as "http://address" or "http://address01,http://address02"
xxl.job.admin.addresses=http://127.0.0.1:8089/xxl-job-admin

### xxl-job executor address
xxl.job.executor.appname=${spring.application.name}
xxl.job.executor.ip=
xxl.job.executor.port=9999

### xxl-job, access token
xxl.job.accessToken=

### xxl-job log path
xxl.job.executor.logpath=logs
### xxl-job log retention days
xxl.job.executor.logretentiondays=-1
```

## 3.3 执行器组件配置
添加`java`配置文件，获取以上的`executor.properties`配置。如下：

```
@Configuration
@ConfigurationProperties(prefix = "xxl.job")
@PropertySource("classpath:/config/executor.properties")
@Slf4j
public class JobExecutorConfig {
    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;
    
    ......略

    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        log.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }
}
```

## 3.4 编写执行器 
执行器，其实就是继承了`IJobHandler`类的实现，在`execute`方法实现任务实现逻辑即可。本示例中，只需要按之前测试的启动任务的逻辑实现即可。如下：

```
@JobHandler(value="incrementUserJobHandler")
@Component
public class JobIncrementUserHandler extends IJobHandler {
    @Autowired
    private JobLauncherService jobLauncherService;

    @Autowired
    private IncrementService incrementService;

    @Autowired
    private Job incrementJob;
    @Override
    public ReturnT<String> execute(String s) throws Exception {
        return JobUtil.runJob4Executor("incrementUser",incrementService,jobLauncherService,incrementJob);
    }
}
```

# 4.使用xxl-job进行Spring Batch任务调度
把`xxl-job-admin`和`spring-batch-xxl-executor`进行部署后（注意配置正确），就可以在`xxl-job-admin`界面中对任务进行调度。具体添加流程可参照`xxl-job`文档即可。简单说就是添加执行器`spring-batch-xxl-executor`，然后添加任务`incrementUserJobHandler`，指定执行的`cron`表达式，启动即可。执行任务后输出如下：

![执行结果][12]

# 5. 总结
本文基于数据库增量同步的示例，使用`Spring Batch`结合`xxl-job`，实现可监控的任务调度，使得批处理任务可维护性更强，希望需要使用`Spring Batch`进行批处理任务开发的人员有帮助。



[1]: https://mianshenglee.github.io/2019/06/04/springbatch(1).html
[2]: https://mianshenglee.github.io/2019/06/07/spring-batch(2).html
[3]: https://mianshenglee.github.io/2019/06/08/spring-batch(3).html
[4]: https://mianshenglee.github.io/2019/06/09/spring-batch(4).html
[5]: https://mianshenglee.github.io/2019/06/10/spring-batch(5).html
[6]: https://mianshenglee.github.io/2019/06/11/spring-batch(6).html
[7]: http://www.xuxueli.com/xxl-job
[8]: http://www.xuxueli.com/xxl-job/#/?id=%E3%80%8A%E5%88%86%E5%B8%83%E5%BC%8F%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0xxl-job%E3%80%8B
[9]: http://ww1.sinaimg.cn/large/72d660a7gy1g3q9oxy81jj222813agt8.jpg
[10]: https://github.com/mianshenglee/spring-batch-example/tree/master/spring-batch-increment
[11]: http://www.xuxueli.com/xxl-job/#/?id=_24-%E9%85%8D%E7%BD%AE%E9%83%A8%E7%BD%B2%E6%89%A7%E8%A1%8C%E5%99%A8%E9%A1%B9%E7%9B%AE
[12]: http://ww1.sinaimg.cn/large/72d660a7gy1g3qap9gh8vj20qu0clglp.jpg