---
layout: post
title: mongo同步-spring batch(8)的mongo读写组件使用
category: IT
tags: SpringBatch MongoDB
keywords: 
description: 
---

tags: springbatch mongodb

---

# 1.引言

之前对Spring Batch的通过实例的方式进行了介绍，有兴趣的可见以下文章：

- [数据批处理神器-Spring Batch(1)简介及使用场景][1]
- [快速了解组件-spring batch(2)之helloworld][2]
- [快速使用组件-spring batch(3)读文件数据到数据库][3]
- [决战数据库-spring batch(4)数据库到数据库][4]
- [便捷的数据读写-spring batch(5)结合beetlSql进行数据读写][5]
- [增量同步-spring batch(6)动态参数绑定与增量同步][6]
- [调度与监控-spring batch(7)结合xxl-job进行批处理][7]

除了文件及关系型数据库的数据同步，Spring Batch的读组件(`ItemReader`)，处理组件(`ItemProcessor`)，写组件(`ItemWriter`)支持丰富的数据类型，其中`MongoItemReader`及`MongoItemWriter`是针对mongo的读写组件，用户可以直接使用，进行`Mongodb`的数据读写操作。一种比较常用的情景是从关系型数据库（如`mysql`）把数据同步到`mongodb`中，下面通过实例对`mysql`到`mongodb`的数据同步进行讲解。本文主要讲解有关`Mongodb`的操作，对于`Spring Batch`使用`beetlsql`进行关系数据库数据读取的操作请见文章《[便捷的数据读写-spring batch(5)结合beetlSql进行数据读写][5]》。本文的示例代码见[github示例仓库][8]。

# 2.开发环境

- JDK: jdk1.8
- Spring Boot: 2.1.4.RELEASE
- Spring Batch:4.1.2.RELEASE
- 开发IDE: IDEA
- 构建工具Maven: 3.3.9
- 日志组件logback:1.2.3
- lombok:1.18.6
- MySQL: 5.6.26
- Mongodb:4.0.10

# 3.开发流程

## 3.1 示例数据库及目标数据库

本示例的流程如下所示：

![流程][9]

示例工程中的`sql`目录有相应的关系数据库脚本，`mytest.sql`脚本创建一个`test_user`表，并有相应的测试数据。`mongodb`的安装可见[官方文档][10]，建立相应的存放数据的`Collection`，本示例为`mytest`。

## 3.2 添加`maven`依赖及配置`mongodb`连接地址

由于需要使用`mongodb`的操作，因此需要添加它的依赖。如下所示：

```xml
<!-- mongodb -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

添加依赖后，`mongodb`的连接地址需配置在配置文件中，若有用户名密码，则同样需要配置。如下：

```java
spring.data.mongodb.uri=mongodb://192.168.222.10/mytest
# spring.data.mongodb.username=
# spring.data.mongodb.password=
```

## 3.3 编写`mongodb`的读写组件

按示例，共三个组件，需要的是一个读`mysql`数据库的组件，一个`mysql`数据库实体转化为`mongodb`的处理组件，一个写入`mongodb`的写组件，代码结构如下图所示：

![][11]



其中`ItemReader`组件和`ItemProcessor`组件无须多讲，可参考之前的文章，这里主要讲一下`mongodb`的`ItemWriter`，此写入组件通过继承`MongoItemWriter`，编写自己的逻辑即可，而`Spring Batch`提供的`mongodb`写操作，是在初始化`ItemWriter`时，通过`MongoOperations`引入的，因此，`MongoBatchConfig`文件中，添加以下代码：

```java
@Bean
public ItemWriter mongoWriter(MongoOperations mongoTemplate) {
    UserItemWriter userItemWriter = new UserItemWriter();
    userItemWriter.setTemplate(mongoTemplate);
    userItemWriter.setCollection("user");
    return userItemWriter;
}
```

其中，`MongoOperations`是在初始化时注入，在自定义的`UserItemWriter`中，设置`template`及`collection`即可。若逻辑简单，不写自定义的ItemWriter，也可以直接使用`MongoItemWriterBuilder `，直接构建`MongoItemWriter`,如下所示：

```java
return new MongoItemWriterBuilder<MongoUser>()
                .collection("user")
                .template(mongoTemplate)
                .build();
```

以上是写组件的构建，同理，对于`mongodb`的读组件，构建方式类似，只是需要注意一下动态参数的配置，如下示例代码是查询数据，并返回`map`，参数是在构建任务时动态传入的。

```java
@Bean
@StepScope
public MongoItemReader<Map> tweetsItemReader(MongoOperations mongoTemplate,@Value("#{jobParameters['hashTag']}") String hashtag) {
return new MongoItemReaderBuilder<Map>()
    .name("tweetsItemReader")
    .targetType(Map.class)
    .jsonQuery("{ \"entities.hashtags.text\": { $eq: ?0 }}")
    .collection("tweets_collection")
    .parameterValues(Collections.singletonList(hashtag))
    .pageSize(10)
    .sorts(Collections.singletonMap("created_at", Sort.Direction.ASC))
    .template(mongoTemplate)
    .build();
}
```

# 4.执行结果

编写单元测试或者在`Controller`编写启动任务，即可进行数据同步测试，执行结果如下所示：

![][12]

![][13]



# 5.总结

本文基于`Spring Batch`对数据从`mysql`到`mongodb`进行数据同步，通过结合示例代码，实现`mongodb`的读写组件进行编写及配置，希望需要使用`Spring Batch`进行关系数据库和`mongodb`进行批处理任务开发的人员有帮助。

[1]: https://mianshenglee.github.io/2019/06/04/springbatch(1).html
[2]: https://mianshenglee.github.io/2019/06/07/spring-batch(2).html
[3]: https://mianshenglee.github.io/2019/06/08/spring-batch(3).html
[4]: https://mianshenglee.github.io/2019/06/09/spring-batch(4).html
[5]: https://mianshenglee.github.io/2019/06/10/spring-batch(5).html
[6]: https://mianshenglee.github.io/2019/06/11/spring-batch(6).htm
[7]: https://mianshenglee.github.io/2019/06/12/spring-batch(7).html
[8]: https://github.com/mianshenglee/spring-batch-example
[9]: http://ww1.sinaimg.cn/large/72d660a7gy1g5ta97je59j20hm04uaa2.jpg
[10]: https://docs.mongodb.com/manual/installation/
[11]: http://ww1.sinaimg.cn/large/72d660a7gy1g5tbhzume7j207405e747.jpg
[12]: http://ww1.sinaimg.cn/large/72d660a7gy1g5tc2jz9qvj20n004l3yy.jpg
[13]: http://ww1.sinaimg.cn/large/72d660a7gy1g5tc3arqc4j20mv0ajt8w.jpg