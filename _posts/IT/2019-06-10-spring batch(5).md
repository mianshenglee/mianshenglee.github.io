---
layout: post
title: 便捷的数据读写-spring batch(5)结合beetlSql进行数据读写
category: IT
tags: SpringBatch
keywords: 
description: 
---

# 便捷的数据读写-spring batch(5)结合beetlSql进行数据读写

tags： springbatch

---

# 1.引言
上一篇文章[《决战数据库-spring batch(4)数据库到数据库》][1]使用`Spring Batch`内置的读、写组件，对数据库间的数据进行同步。相对来说，这个数据读取和数据写入是基于`jdbc`进行读写的(数据对象的映射需要我们自己处理，如`UserRowMapper`)，我们现在开发一般都使用上层一点的`ORM`框架，如`Hibernate`,`MyBatis`,`BeetlSQL`。对于`Hibernate`,`Spring Batch`有默认的`HibernateCursorItemReader`和`HibernateItemWriter`，也可以实现自行使用`MyBatis`和`BeetlSQL`。个人感觉，从易用性和学习曲线上看，`BeetlSQL`会更容易上手，社区也挺活跃，因此，本文介绍一下使用`Spring Batch`结合`BeetlSQL`进行数据库到数据库的数据同步。

# 2.开发环境
- JDK: jdk1.8
- Spring Boot: 2.1.4.RELEASE
- Spring Batch:4.1.2.RELEASE
- BeetlSQL: 1.1.77.RELEASE
- 开发IDE: IDEA
- 构建工具Maven: 3.3.9
- 日志组件logback:1.2.3
- lombok:1.18.6

# 3.BeetlSQL简要说明
按官方文档，`BeetlSQL`是一个全功能`DAO`工具， 同时具有`Hibernate`优点 & `Mybatis`优点功能，适用于承认以`SQL`为中心，同时又需求工具能自动能生成大量常用的`SQL`的应用。详细可参考[官方文档][2]。从最近一段时间的使用过程中，感觉从开发效率、维护性、易用性，都比较优秀。

# 4.使用BeetlSQL读写数据库
本示例依然是基于上一篇文章的示例功能，从源数据库中读取`test_user`表的数据，然后经过处理，再写入到目标数据库的`test_user`表中。只是不是使用`Spring Batch`内置的`JdbcCursorItemReader`和`JdbcBatchItemWriter`，改为使用`BeetlSQL`进行读写。完整示例可参考[代码][3]

## 4.1 引入BeetlSQL依赖
`BeetlSQL`提供了`Spring Boot`的`starter`来实现自动配置，在`pom.xml`添加以下依赖即可：

```
<!-- orm框架: beetlsql -->
<dependency>
	<groupId>com.ibeetl</groupId>
	<artifactId>beetl-framework-starter</artifactId>
	<version>1.1.77.RELEASE</version>
</dependency>
```

添加后，会添加两个依赖，分别是`beetl-2.9.9`，和`beetlsql-2.11.2`，如下图：

![beetlsql][4]

## 4.2 编写多数据源的dao
### 4.2.1 添加配置文件
由于我们是使用多数据源进行读写，关于多数据源的配置，[上一篇文章][5]已经进行了描述，此处不再说明。`BeetlSQL`对多数据源有较好的支持，只需要简单的配置即可。具体可参考[官方文档][6]。下面对此做简单说明。在`application.properties`文件中添加以下配置：

```
#beetlsql配置
#默认/sql，可不设置
#beetlsql.sqlPath=/sql
#dao文件的后缀
beetlsql.daoSuffix=Repository
#自动加载和查找的dao文件所在包
beetlsql.basePackage=me.mason.springbatch
#默认org.beetl.sql.core.db.MySqlStyle，可不设置
#beetlsql.dbStyle=org.beetl.sql.core.db.MySqlStyle
#多数据源dao文件所在位置，以包区分读写数据源
beetlsql.ds.datasource.basePackage=me.mason.springbatch.dao.local
beetlsql.ds.originDatasource.basePackage=me.mason.springbatch.dao.origin
beetlsql.ds.targetDatasource.basePackage=me.mason.springbatch.dao.target
beetlsql.mutiple.datasource=datasource,originDatasource,targetDatasource
```

说明：

- `beetlsql.daoSuffix`表示`Dao`文件的后缀，`BeetlSql`会根据此后缀加载`Dao`。
- `beetlsql.mutiple.datasource`数据源名与数据源的配置一致。

### 4.2.2 添加dao文件
添加上述配置后，由于使用包名来区分读写数据源的`Dao`，因此需要创建对应的包，分别在工程中`me.mason.springbatch`下创建`dao.local`,`dao.origin`，`dao.target`三个包，分别存放需要读取的三个数据源对应的`Dao`。在本示例中，仅使用源数据库和目标数据库，因此，仅需要在`dao.origin`中添加`OriginUserRepository`进行源数据读操作，在`dao.target`中添加`TargetUserRepository`进行写操作即可。（注意，由于配置中指定是使用后缀`Repository`，因此此处的类名需要使用它作为后缀）。如下：

`OriginUserRepository.java`

```
@Repository
public interface OriginUserRepository extends BaseMapper<User> {
    List<User> getOriginUser(Map<String,Object> params);
}
```

`TargetUserRepository.java`

```
@Repository
public interface TargetUserRepository extends BaseMapper<User> {
}
```

说明：

- 使用注解`@Repository`标注是数据读写dao
- 继承`BaseMapper`，以使用`BeetlSQL`内置的增删改查能力
- 对于`OriginUserRepository`，`getOriginUser`是自定义的数据读取操作，此操作具体实现是使用写在`sql/user.md`中的`sql`语句（sql语句在后面将说明）。

## 4.3 编写sql文件
根据`BeetlSql`的功能，开发者可以自定义`sql`语句进行数据库操作，而`sql`语句是以markdown文件的形式保存，支持`beetl`的语法，支持参数化语句，逻辑判断等操作，这就有点像`Mybatis`中的xml语句，但显示和修改会更友好。具体sql文件更详细的使用功能，读者可到官文档查阅。

本示例中，上面`OriginUserRepository`自定义了`getOriginUser`函数，以此函数名即可编写`sql`进行读数据（当然，由于此`sql`比较简单，完全可以不写到`markdown`文件也可以实现，此处仅做示例展示此功能而已）。如下所示：

```
getOriginUser
===
* 查询user数据

select * from test_user
```

接着就是写数据需要用到的`sql`，`insertUser`是这条语句的名称，在插入数据时会使用到：

```
insertUser
===
* 插入数据

insert into test_user(id,name,phone,title,email,gender,date_of_birth
    ,sys_create_time,sys_create_user,sys_update_time,sys_update_user)
values (#id#,#name#,#phone#,#title#,#email#,#gender#,#dateOfBirth#
    ,#sysCreateTime#,#sysCreateUser#,#sysUpdateTime#,#sysUpdateUser#)
```

由上面的sql可见，这跟我们平时写`sql`没有区别，其中以`##`包括的是参数，即实体`User`的字段。

## 4.4 编写读组件ItemReader
经过上面的配置和添加的`dao`类，我们已经有了读和写数据的能力。使用`OriginUserRepository`，可以编写`Spring Batch`的`ItemReader`。读取数据后在内存中，在`read()`时返回。如下所示：

```
@Slf4j
public class UserItemReader implements ItemReader<User> {
    protected List<User> items;

    protected Map<String,Object> params;
    @Autowired
    private OriginUserRepository originUserRepository;

    @Override
    public User read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        if(Objects.isNull(items)){
            //使用beetlsql的md执行sql
            items = originUserRepository.getOriginUser(params);
            if(items.size() > 0){
                return items.remove(0);
            }
        }else{
            if (!items.isEmpty()) {
                return items.remove(0);
            }
        }
        return null;
    }

    public Map<String, Object> getParams() {
        return params;
    }

    public void setParams(Map<String, Object> params) {
        this.params = params;
    }
}
```

说明：

- `originUserRepository.getOriginUser`执行的是`user.md`的`getOriginUser`查询语句。
- 查询后数据保存在`List<User>`，在`read()`时返回其中的数据。全部返回后会返回null，以表示结束。
- `BeetlSql`支持使用`Map`来传参，本示例暂时没有使用。

## 4.5 编写写组件ItemWriter
读取到数据后，使用`TargetUserRepository`，执行上面编写的`insertUser`来插入数据。如下所示

```
public class UserItemWriter implements ItemWriter<User> {
    @Autowired
    private TargetUserRepository targetUserRepository;

    @Override
    public void write(List<? extends User> items) throws Exception {
        targetUserRepository.getSQLManager().updateBatch("user.insertUser",items);
    }
}
```

说明：
- 使用`SQLManager`的`updateBatch`批量写数据
- `user.insertUser`中，`user`是`markdown`文件的名称，也是实体名称，`insertUser`是写的`sql`语句。

## 4.6 组装完整任务
新建`BeetlsqlBatchConfig.java`，作为`Spring Batch`的任务配置

### 4.6.1 注入读写组件
使用上面已写的`Reader`和`Writer`，使用`Bean`注解加入，如下：

```
@Bean
public ItemReader beetlsqlItemReader() {
    UserItemReader userItemReader = new UserItemReader();
    //设置参数，当前示例可不设置参数
    Map<String,Object> params = CollUtil.newHashMap();
    userItemReader.setParams(params);
    return userItemReader;
}

@Bean
public ItemWriter beetlsqlWriter() {
    return new UserItemWriter();
}
```

### 4.6.2 组装任务
使用`Step`和`Job`，实现完整的任务配置，如下：

```
@Bean
public Job beetlsqlJob(Step beetlsqlStep,JobExecutionListener beetlsqlListener){
    String funcName = Thread.currentThread().getStackTrace()[1].getMethodName();
    return jobBuilderFactory.get(funcName)
            .listener(beetlsqlListener)
            .flow(beetlsqlStep)
            .end().build();
}
@Bean
public Step beetlsqlStep(ItemReader beetlsqlItemReader ,ItemProcessor beetlsqlProcessor
        ,ItemWriter beetlsqlWriter){
    String funcName = Thread.currentThread().getStackTrace()[1].getMethodName();
    return stepBuilderFactory.get(funcName)
            .<User,User>chunk(10)
            .reader(beetlsqlItemReader)
            .processor(beetlsqlProcessor)
            .writer(beetlsqlWriter)
            .build();
}
```

## 4.7 测试
参考上一文章的`Db2DbJobTest`，编写`BeetlsqlJobTest`文件。测试前先把目标数据库中的`test_user`库清空，然后启动`Job`进行测试，结果跟`Db2DbJobTest`是一致的。日志输出如下：

![result][7]

在上面使用`BeetlSql`过程中，可见有几个好处：

- 不需要自己写`RowMapper`对数据进行映射，更简单。
- `sql`语句写在`markdown`文件中，修改更灵活。
- `sql`语句执行输出在日志中，更清晰。

# 5.总结
本文使用`Spring Batch`对数据库到数据库的示例做了一个改动，使用`BeetlSQL`进行多数据源的读写操作，以实现更简单、更灵活、更清晰的数据库读写。希望对想要用`Spring Batch`同时又想了解`BeetlSql`的读者有帮助。


[1]: https://mianshenglee.github.io/2019/06/09/spring-batch(4).html
[2]: http://ibeetl.com/guide/#beetlsql
[3]: https://github.com/mianshenglee/spring-batch-example/tree/master/spring-batch-beetlsql
[4]: http://ww1.sinaimg.cn/large/72d660a7gy1g3p4p76lkmj20b501wjr8.jpg
[5]: https://mianshenglee.github.io/2019/06/09/spring-batch(4).html
[6]: http://ibeetl.com/guide/#beetlsql
[7]: http://ww1.sinaimg.cn/large/72d660a7gy1g3p6h1un6xj20xx09habb.jpg