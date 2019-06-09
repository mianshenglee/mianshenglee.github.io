---
layout: post
title: 决战数据库-spring batch(4)数据库到数据库
category: IT
tags: SpringBatch
keywords: 
description: 
---

# 决战数据库-spring batch(4)数据库到数据库

tags：springbatch

---

# 1.引言
上一篇文章[《快速使用组件-spring batch(3)读文件数据到数据库》][1]对`Spring Batch`的读、处理、写组件进行了介绍，并且以实际案例使用了`FlatFileItemReader`读文本文件，并把每行数据映射为实体，然后使用`JdbcBatchItemWriter`把实体对象数据存储到`MySQL`中。但在数据集成的实际应用中，更多的工作是从A数据库到B数据库，数据库之间是异构的，数据与数据的字段定义是不尽相同的，因此，在数据同步、数据抽取时，需要从源数据库读取数据，经过校验、转换，过滤、清洗，然后把数据再写到目标数据库。本文将在[上一篇][2]的基础上，实现数据库到数据库的数据同步。简单来说，只需要把从文件读数据改为从数据库读即可。可下载完整[示例工程代码][3]参考

![数据库到数据库][4]

# 2.开发环境

- JDK: jdk1.8
- Spring Boot: 2.1.4.RELEASE
- Spring Batch:4.1.2.RELEASE
- 开发IDE: IDEA
- 构建工具Maven: 3.3.9
- 日志组件logback:1.2.3
- lombok:1.18.6

# 3.开发流程
上一篇文章中，已经把`User`数据存储在`mytest`数据库中，本文将以`mytest`数据库中的`test_user`表为源数据，使用`Spring Batch`把数据同步到目标数据库`my_test1`，实现`MySQL`到`MySQL`的同步。注，若是不同的数据库，在配置多数据源时可更改数据库驱动和连接信息即可。关键代码如下所示：

![关键代码][5]

## 3.1 创建目标数据库
在`MySQL`中创建`my_test1`数据库作为目标数据库，执行[示例工程][6]中的`sql/my_test1.sql`创建用户表，为简单起见，目标数据表与源数据表数据表结构一样。创建后如下：

![目标数据库][7]

## 3.2 配置多数据源
至此，我们的程序涉及三个数据库，分别是：

- 用于`Spring Batch`数据存储的`my_spring_batch`为
- 源数据库`mytest`
- 目标数据库`my_test1`

因此，需要先配置多数据源，配置方法跟之前一样，配置`properties`文件的数据库连接信息和使用注解进行配置即可。如下：

`application.properties`

```
# spring batch db
spring.datasource.jdbc-url=jdbc:mysql://localhost:3310/my_spring_batch?serverTimezone=GMT%2B8&useUnicode=true&characterEncoding=utf8&useSSL=false
spring.datasource.username=root
spring.datasource.password=111111
# origin db
spring.origin-datasource.jdbc-url=jdbc:mysql://localhost:3310/mytest?serverTimezone=GMT%2B8&useUnicode=true&characterEncoding=utf8&useSSL=false
spring.origin-datasource.username=root
spring.origin-datasource.password=111111
# target db
spring.target-datasource.jdbc-url=jdbc:mysql://localhost:3310/my_test1?serverTimezone=GMT%2B8&useUnicode=true&characterEncoding=utf8&useSSL=false
spring.target-datasource.username=root
spring.target-datasource.password=111111
```

然后使用注解注入多数据源，如下：

`DataSourceConfig.java`

```
@Bean("datasource")
@ConfigurationProperties(prefix="spring.datasource")
@Primary
public DataSource batchDatasource() {
    return DataSourceBuilder.create().build();
}

@Bean("originDatasource")
@ConfigurationProperties(prefix="spring.origin-datasource")
public DataSource originDatasource() {
    return DataSourceBuilder.create().build();
}

@Bean("targetDatasource")
@ConfigurationProperties(prefix="spring.target-datasource")
public DataSource targetDatasource() {
    return DataSourceBuilder.create().build();
}
```

## 3.3 添加读数据组件`JdbcCursorItemReader`
从数据库中读取数据，`Spring Batch`提供了组件`JdbcCursorItemReader`，通过它，可以把数据库的数据读取出来，然后映射为实体`User`，以供后续开发。创建方法如下：

```
@Bean
public ItemReader db2DbItemReader(@Qualifier("originDatasource") DataSource originDatasource) {
    String readSql = " select * from test_user";
    return new JdbcCursorItemReaderBuilder<User>()
            .dataSource(originDatasource).sql(readSql)
            .verifyCursorPosition(false).rowMapper(new UserRowMapper())
            .build();
}
```

说明：

- 使用`@Qualifier("originDatasource")`标识源数据库
- `JdbcCursorItemReaderBuilder`用于构建`JdbcCursorItemReader`
- 读数据的`sql`语句根据实际情况编写即可，此处是读取整个表数据。
- 需要把数据库映射为实体`User`，使用`UserRowMapper`，此`mapper`实现`RowMapper`接口，把从数据库读取的`ResultSet`映射为`User`的字段。

## 3.4 自定义处理组件`Db2DbItemProcessor`
读取到数据后，当前的处理是针对`title`字段，不为`null`的则转为大写即可。如下：

```
if(Objects.nonNull(title)){
    user.setTitle(title.toUpperCase());
}
```

## 3.5 添加写数据组件`JdbcBatchItemWriter`
写入数据，同样使用`JdbcBatchItemWriter`组合，编写插入`sql`语句，把实体`User`数据插入到数据库即可，如下：

```
@Bean
public ItemWriter db2DbWriter(@Qualifier("targetDatasource") DataSource targetDatasource) {
    String inserSql ="INSERT INTO test_user(id,name,phone,title,email,gender,date_of_birth,sys_create_time,sys_create_user,sys_update_time,sys_update_user) " +
            "VALUES (:id,:name,:phone,:title,:email,:gender,:dateOfBirth,:sysCreateTime,:sysCreateUser,:sysUpdateTime,:sysUpdateUser)";
    return new JdbcBatchItemWriterBuilder<User>()
            .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
            .sql(inserSql)
            .dataSource(targetDatasource)
            .build();
}
```

说明：

- 使用`JdbcBatchItemWriterBuilder`进行`JdbcBatchItemWriter`的创建，设置插入数据库的sql语句，同时指定数据源即可。
- `@Qualifier("targetDatasource") DataSource datasource`用于指定数据源
- 使用`BeanPropertyItemSqlParameterSourceProvider`可以直接把读取的数据实体的属性数据作为参数填充到`sql`语句中，从而实现数据插入操作。

## 3.6 组装完整任务
经过上面的操作，可以使用一个java配置，把读、写、处理组装成完整的`step`和`job`，如下所示（详细可见示例工程文件）：

```
@Bean
public Job db2DbJob(Step db2DbStep,JobExecutionListener db2DbListener){
    String funcName = Thread.currentThread().getStackTrace()[1].getMethodName();
    return jobBuilderFactory.get(funcName)
            .listener(db2DbListener)
            .flow(db2DbStep)
            .end().build();
}
@Bean
public Step db2DbStep(ItemReader db2DbItemReader ,ItemProcessor db2DbProcessor
        ,ItemWriter db2DbWriter){
    String funcName = Thread.currentThread().getStackTrace()[1].getMethodName();
    return stepBuilderFactory.get(funcName)
            .<User,User>chunk(10)
            .reader(db2DbItemReader)
            .processor(db2DbProcessor)
            .writer(db2DbWriter)
            .build();
}
```

## 3.7 测试 
参考上一文章的`File2DbJobTest`，编写`Db2DbJobTest`文件即可。如下：

```
@Test
public void testDb2DbJob() throws JobParametersInvalidException, JobExecutionAlreadyRunningException, JobRestartException, JobInstanceAlreadyCompleteException {
    //构建job参数
    JobParameters jobParameters = JobUtil.makeJobParameters();
    //运行job
    Map<String, Object> stringObjectMap = jobLauncherService.startJob(db2DbJob, jobParameters);
    //测试结果
    Assert.assertEquals(ExitStatus.COMPLETED,stringObjectMap.get(SyncConstants.STR_RETURN_EXITSTATUS));
}
```

经过此测试，可查看到源数据库`mytest`中的`test_user`表中的数据，已全部同步到目标库`my_test1`中的`test_user`中。完成数据库到数据库的数据同步。

# 4.总结
本文通过简单的示例，从源数据库中读取表数据，经过处理，写入到目标数据库，具体一定的通用性。希望让大家更深入的了解Spring Batch，并能用到实践中。


[1]: https://mianshenglee.github.io/2019/06/08/spring-batch(3).html
[2]: https://mianshenglee.github.io/2019/06/08/spring-batch(3).html
[3]: https://github.com/mianshenglee/spring-batch-example
[4]: http://ww1.sinaimg.cn/large/72d660a7gy1g3owxj2wtlj20ht04lmx6.jpg
[5]: http://ww1.sinaimg.cn/large/72d660a7gy1g3oybdxk4yj208205gmx3.jpg
[6]: https://github.com/mianshenglee/spring-batch-example
[7]: http://ww1.sinaimg.cn/large/72d660a7gy1g3oxoanfl7j206101c3ya.jpg