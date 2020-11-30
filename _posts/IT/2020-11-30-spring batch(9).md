---
layout: post
title: 数据共享-spring batch(9)上下文处理
category: IT
tags: SpringBatch
keywords: 
description: 
---

# 1 引言

本文是 Spring Batch 系列文章的第9篇，有兴趣的可见文章：

- [数据批处理神器-Spring Batch(1)简介及使用场景](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483738&idx=1&sn=13853ec98fa43e13df68278005c77116&chksm=fa24d18fcd535899b7b713d5d1a12c80d674efbd671a8e67d04d500521d29524a5914ebe4861&scene=21#wechat_redirect)
- [快速了解组件-spring batch(2)之helloworld](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483743&idx=1&sn=4e5f76932e8ab2ae20847fcb1a77682f&chksm=fa24d18acd53589c7a58ea05f3a011ce86088653e5d2ef13360d893c89310f2e03aa623785c8&scene=21#wechat_redirect)
- [快速使用组件-spring batch(3)读文件数据到数据库](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483748&idx=1&sn=986cb6f202a7a03048f4e7629e42a116&chksm=fa24d1b1cd5358a7be8e075ee359dc18b6452339ed21de4bf5c234af0f274e860b802d39a9d4&scene=21#wechat_redirect)
- [决战数据库-spring batch(4)数据库到数据库](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483755&idx=1&sn=4e1aaf706a5bcfbe94829075aa8dab76&chksm=fa24d1becd5358a8aeb2df4feda03c344a8972c36afb11c6012769925539b5a4ce54c0dac24f&scene=21#wechat_redirect)
- [便捷的数据读写-spring batch(5)结合beetlSql进行数据读写](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483760&idx=1&sn=a880d91912e3ede028ee5b81a419f866&chksm=fa24d1a5cd5358b37f1900eb9572054057ef98ad4377097b19dd5dd62066d480aa8636b2069a&scene=21#wechat_redirect)
- [增量同步-spring batch(6)动态参数绑定与增量同步](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483765&idx=1&sn=8436a83161b95987f3a5bc98c9401f2c&chksm=fa24d1a0cd5358b645359bdb75652b0e0cb32c4ffb17aa9c3f5350180097aba6aa8a6e08bcb4&scene=21#wechat_redirect)
- [调度与监控-spring batch(7)结合xxl-job进行批处理](http://mp.weixin.qq.com/s?__biz=MzUyNDk0NTg1MA==&mid=2247483772&idx=1&sn=3cefc5b1126c5da13ff7005c3ef2b62a&chksm=fa24d1a9cd5358bf678f9937189db8356ddfa19de0d84b87fecd4539a673aae1aa10d210a4ad&scene=21#wechat_redirect)
- [mongo同步-spring batch(8)的mongo读写组件使用](https://mp.weixin.qq.com/s/HvUbCdHeUIwqRX4PNPLebg)

前面文章以实例的方式对 Spring Batch 进行批处理进行详细说明，相信大家对字符串、文件，关系型数据库及 NoSQL 数据库的读取，处理，写入流程已比较熟悉。有小伙伴就问，针对这个任务流程，期间有多个步骤，从任务（ Job ）启动，到作业步（ Step ）的执行，其中又包含读组件、处理组件、写组件，那么，针对这个流程，若中间需要传递自定义的参数，该如何处理？本文将对 Spring Batch 进行参数传递的方法进行描述，依然会使用代码实例的方式进行讲解。包括以下几个内容：

- 基于 Mybatis-plus 集成多数据源的数据库访问
- 使用 ExecutionContext 共享数据
- StepScope 动态绑定参数传递

# 2 开发环境

- JDK环境: jdk1.8
- Spring Boot: 2.1.4.RELEASE
- Spring Batch:4.1.2.RELEASE
- 开发IDE: IDEA
- 构建工具Maven: 3.3.9
- 日志组件logback:1.2.3
- lombok:1.18.6
- MySQL: 5.6.26
- Mybatis-plus: 3.4.0

本示例源码已放至[github](https://github.com/mianshenglee/spring-batch-example/tree/master/spring-batch-param)：`https://github.com/mianshenglee/spring-batch-example/tree/master/spring-batch-param`，请结合示例代码进行阅读。



# 3 基于 Mybatis-plus 集成多数据源的数据库访问

本示例还是使用原来示例功能，从源数据库读取用户数据，处理数据，然后写入到目标数据库。其中会在任务启动时传递参数，并在作业步中传递参数。之前已经介绍过如何使用 beetlsql 进行多数据源配置（[便捷的数据读写-spring batch(5)结合beetlSql进行数据读写][5]），实现数据批处理。还有很多朋友使用 Mybatis 或 Mybatis-plus 进行数据库读写，因此，有必要提一下 Spring Batch 如何结合 Mybatis 或 Mybatis-plus 配置多数据源操作。本示例以 Mybatis-plus 为例。

示例工程中的`sql`目录有相应的数据库脚本，其中源数据库`mytest.sql`脚本创建一个`test_user`表，并有相应的测试数据。目标数据库 `my_test1.sql` 与 `mytest.sql`表结构一致，`spring-batch-mysql.sql`是 Spring Batch 本身提供的数据库脚本。

## 3.1 pom 文件中引入 Mybatis-plus

```xml
<!--mybatis-plus-->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.0</version>
</dependency>

```

## 3.2 配置及使用多数据源

本示例会涉及三个数据库，分别是 Spring Batch 本身数据库，需要批处理的源数据库，批处理的目标数据库。因此需要处理多个数据库，利用多套源策略，可以很简单就完成多套数据源的处理。简单来说主要分为以下几个步骤：

- 配置多数据源连接信息
- 根据不同数据源划分`mapper` 包，`entity`包，`mapper.xml`文件包
- 根据不同数据源配置独立的` SqlSessionFactory `
- 根据不同的应用场景，使用不同的 `mapper`

关于多数据源多套源策略的详细配置过程，可以参考我的另一篇文章《[搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)》

# 4 ExecutionContext 传递参数

关于 Spring Batch 的读数据（ ItemReader ）、处理数据（ ItemProcessor ）、写数据（ ItemWriter ）的配置流程，可以参考前面系列文章，本文不再详细描述。我们需要记住的是，当一个作业（ Job ）启动，Spring Batch 是通过作业名称（ Job name）及 作业参数（ JobParameters ）作为唯一标识来区分不同的作业。一个 Job 下可以有多个作业步（ Step ），每个 Step 中就是有具体的操作逻辑（读、处理、写）。在 Job 和 Step 下的各个操作步骤间，如何传递，，这里就需要理解 ExecutionContext 的概念。

## 4.1 ExecutionContext 概念

在 Job 的运行及 Step 的运行过程中，Spring Batch 提供 ExecutionContext 进行运行数据持久化，利用它，可以根据业务进行数据共享，如用来重启的静态数据与状态数据。如下图：

![Spring Batch 参数传递](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/springbatch/springbatch-9/spring%20batch-%E5%8F%82%E6%95%B0%E4%BC%A0%E9%80%92.png)

Execution Context 本质上来讲就是一个 `Map<String,Object>` ，它是Spring Batch 框架提供的持久化与控制的 key/value 对，可以让开发者在 Step 运行或Job 运行过程中保存需要进行持久化的状态，它可以。分为两类，一类是Job 运行的上下文（对应数据表：BATCH_JOB_EXECUTION_CONTEXT），另一类是Step Execution的上下文（对应数据表BATCH_STEP_EXECUTION_CONTEXT）。两类上下文关系：一个 Job 运行对应一个 Job Execution 的上下文（如上图中蓝色部分的 ExecutionContext ），每个 Step 运行对应一个 Step Execution 上下文（如上图中粉色部分的 ExecutionContext ）；同一个  Job 中的 Step Execution 共用 Job Execution 的上下文。也就是说，它们的作用范围有区别。因此，如果同一个 Job 的不同 Step 间需要共享数据时，可以通过 Job Execution 的上下文共享数据。根据 ExecutionContext 的共享数据特性，则可以实现在不同步骤间传递数据。

## 4.2 ExecutionContext 传递数据

一个 Job 启动后，会生成一个 JobExecution ，用于存放和记录 Job 运行的信息，同样，在 Step 启动后，也会有对应的 StepExecution 。如前面所说，在 JobExecution 和 StepExecution 中都会有一个 ExecutionContext ，用于存储上下文。因此，数据传递的思路就是确定数据使用范围，然后通过 ExecutionContext  传入数据，然后就可以在对应的范围内共享数据。如当前示例，需要 Job 范围内共享数据，在读组件（ ItemReader ）和写组件（ ItemWriter ）中传递读与写数据的数量（ size ），在 Job 结束时，输出读及写的数据量。实际上 Spring Batch 会自动计算读写数量，本示例仅为了显示数据共享功能。

![共享数据](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/springbatch/springbatch-9/spring%20batch-%E5%8F%82%E6%95%B0%E4%BC%A0%E9%80%92-2.png)

那么，如何获取对应的 Execution ？，Spring Batch 提供了 JobExecutionListener 和 StepExecutionListener 监听器接口，通过实现监听器接口，分别可以在开启作业前（ beforeJob ）和 完成作业后（ afterJob ）afterJob ），开启作业步前（ beforeStep）及 完成作业步后（ afterStep ）获取对应的 Execution ，然后进行操作。

### 4.2.1 实现监听器接口

在自定义的 UserItemReader 和 UserItemWriter 中，实现 StepExecutionListener 接口，其中使用 StepExecution 作为成员，从 beforeStep 中获取。如下：

```java
public class UserItemWriter implements ItemWriter<TargetUser>, StepExecutionListener {
    private StepExecution stepExecution;
    //...略
    @Override
    public void beforeStep(StepExecution stepExecution) {
        this.stepExecution = stepExecution;
    }
}
```

读组件（ UserItemReader ）也使用同样的方式。而在作业结束后，获取参数，则可以继承 JobExecutionListenerSupport ，实现自己感兴趣的方法，也从参数中获取 JobExecution，然后获取参数进行处理。

```java
public class ParamJobEndListener extends JobExecutionListenerSupport {
    @Override
    public void afterJob(JobExecution jobExecution) {}
}
```



### 4.2.2 设置用于传递的数据

由于我们需要在 Job 范围内传递参数，获取到 StepExecution 后，可以获得相应的 JobExecution ，进而获取 Job 对应的 executionContext，这样，就可以在 Job 范围内共享参数数据了。如下是在读组件中进行配置

```java
ExecutionContext executionContext = stepExecution.getJobExecution().getExecutionContext();
            executionContext.put(SyncConstants.PASS_PARAM_READ_NUM, items.size());
```

同样在写组件中，获取到 ExecutionContext 后，可以对参数进行处理。本示例中，是通过对 ItemReader 传递的处理数目参数进行累加处理，得到结果。

```java
@Override
public void write(List<? extends TargetUser> items) {
    ExecutionContext executionContext = stepExecution.getJobExecution().getExecutionContext();
    Object currentWriteNum = executionContext.get(SyncConstants.PASS_PARAM_WRITE_NUM);
    if (Objects.nonNull(currentWriteNum)) {
        log.info("currentWriteNum:{}", currentWriteNum);
        executionContext.put(SyncConstants.PASS_PARAM_WRITE_NUM, items.size()+(Integer)currentWriteNum);
    } else {
        executionContext.put(SyncConstants.PASS_PARAM_WRITE_NUM, items.size());
    }
```

最后在作业结束后，在实现 JobExecutionListenerSupport 的接口中，afterJob 函数中，对参数进行输出。

```java
public class ParamJobEndListener extends JobExecutionListenerSupport {
    @Override
    public void afterJob(JobExecution jobExecution) {
        ExecutionContext executionContext = jobExecution.getExecutionContext();
        Integer writeNum = (Integer)executionContext.get(SyncConstants.PASS_PARAM_WRITE_NUM);
        log.info(LogConstants.LOG_TAG + "writeNum:{}",writeNum);
    }
}
```

# 5 StepScope 动态绑定参数传递

## 5.1 StepScope及后期绑定

前面说到在 Job 及 Step 范围内，使用 ExecutionContext 进行数据共享，但，如果需要在 Job 启动前设置参数，并且每次启动输入的参数是动态变化的（比如增量同步时，日期是基于上一次同步的时间或者ID），也就是说，每次运行，需要根据参数新建一个操作步骤（如 ItemReader、ItemWriter等），我们知道，由于在 Spring IOC 中加载的Bean，默认都是单例模式的，因此，需要每次运行新建，运行完销毁，新建是在运行时进行的。这就需要用到StepScope 及后期绑定技术。

在之前的示例中，已出现过 StepScope，它的作用是提供了操作步骤的作用范围，某个 Spring Bean 使用注解StepScope，则表示此 Bean 在作业步（ Step ）开始的时候初始化，在 Step 结束的时候销毁，也就是说 Bean的作用范围是在 Step 这个生命周期中。而 Spring Batch 通过属性后期绑定技术，在运行期获取属性值，并使用 SPEL 的表达式进行属性绑定。而在 StepScope 中，Spring Batch 框架提供 JobParameters，JobExecutionContext，StepExecutionContext，当然也可以使用 Spring 容器中的 Bean ，如 JobExecution ，StepExecution。

## 5.2 作业参数传递及动态获取 StepExecution

一个 Job 是由 Job name 及 JobParameters 作为唯一标识的，也就是说只有 job name 和 JobParameters 不一致时，Spring Batch 才会启动一个新的 Job，一致的话就当作是同一个 Job ，若 此 Job 未执行过，则执行；若已执行过且是 FAILED 状态，则尝试重新运行此 Job ，若已执行过且是 COMPLETED 状态，则会报错。

本示例中，Job 启动时输入时间参数，在 ItemReader 中使用 StepScope 注解，然后把时间参数绑定到 ItemReader 中，同时绑定 StepExecution ，以便于在 ItemReader 对时间参数及 StepExecution 进行操作。

### 5.2.1 设置时间参数

在使用 JobLauncher 启动 Job 时，是需要输入 jobParameters 作为参数的。因此可以创建此对象，并设置参数。

```java
JobParameters jobParameters = new JobParametersBuilder()
                .addLong("time",timMillis)
                .toJobParameters();
```

### 5.2.2 动态绑定参数

在配置 Step 时，需要创建ItemReader 的 Bean，为了使用动态参数，在 ItemReader 中设置 Map 存放参数，并设置 StepExecution 为成员，以便于后面使用 ExecutionContext。

```java
public class UserItemReader implements ItemReader<User> {
    protected Map<String, Object> params;
    private StepExecution stepExecution;

    public void setStepExecution(StepExecution stepExecution) {
        this.stepExecution = stepExecution;
    }
}
```

使用 StepScope 进行配置：

```java
@Bean
@StepScope
public ItemReader paramItemReader(@Value("#{stepExecution}") StepExecution stepExecution,
                                  @Value("#{jobParameters['time']}") Long timeParam) {
    UserItemReader userItemReader = new UserItemReader();
    //设置参数
    Map<String, Object> params = CollUtil.newHashMap();
    Date datetime = new Date(timeParam);
    params.put(SyncConstants.PASS_PARAM_DATETIME, datetime);
    userItemReader.setParams(params);
    userItemReader.setStepExecution(stepExecution);

    return userItemReader;
}
```

注意：**此时 ItemReader 不可再使用实现 StepExecutionListener 的方式来对 stepExecution 赋值**，由于 ItemReader 是动态绑定的，StepExecutionListener 将不再起作用，因此需要在后期绑定中来绑定 stepExecution Bean 的方式来赋值。

### 5.2.3 设置及传递参数

ItemReader 获取到 StepExecution 后即可获取 ExecutionContext，然后可以像前面说的使用 ExecutionContext 方式进行数据传递。如下：

```java
ExecutionContext executionContext = stepExecution.getJobExecution().getExecutionContext();
//readNum参数
executionContext.put(SyncConstants.PASS_PARAM_READ_NUM, items.size());
//datetime参数
executionContext.put(SyncConstants.PASS_PARAM_DATETIME,params.get(SyncConstants.PASS_PARAM_DATETIME));
```



# 6.总结

在 Job 和 Step 不同的数据范围中，可使用 ExecutionContext 共享数据。本文以传递处理数量为例，使用 Mybatis-plus，基于 ExecutionContext ，结合 StepScope及后期绑定技术，实现在 Job 启动传入参数，然后在 ItemReader、ItemProcessor、ItemWriter 及 Job 完成后的数据共享及传递。如果你在使用 Spring Batch 过程中需要进行数据共享与传递，请试试这种方式吧。



# 往期文章

[还在手工生成数据库文档？3个步骤自动完成了解一下](https://mp.weixin.qq.com/s/eQpUD0tQ86B2QYZPoQUy3w)

[Python 处理 Excel 文件](https://mp.weixin.qq.com/s/neO8OuWu7Za41xcdmwu2wA)

[MinIO 的分布式部署](https://mp.weixin.qq.com/s/LX-4jhVV7FbB-TA8jcWYOA)

[利用MinIO轻松搭建静态资源服务](https://mp.weixin.qq.com/s/rzaqPmpTOUJJsKISr6eF9Q)

[搞定SpringBoot多数据源(3)：参数化变更源](https://mp.weixin.qq.com/s/ZzzPJZAhPiGCQjN3RNZJ3Q)

[搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)

[搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)

[java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)

[2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)



如果文章内容对你有帮助，欢迎转发分享~

我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![Mason技术记录](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)

