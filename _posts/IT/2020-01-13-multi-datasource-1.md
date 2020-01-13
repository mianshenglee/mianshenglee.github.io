---
layout: post
title: 搞定SpringBoot多数据源(1)：多套源策略
category: IT
tags: multi-datasource java springboot
keywords: 
description: 
---

> 一句话概括：Spring Boot开发中连接多个数据库进行读写操作，使用多套数据源是最直接、简单的方式。

# 1. 引言

在开发过程中，避免不了需要同时操作多个数据库的情况，通常的应用场景如下 ：

- 数据库高性能场景：主从，包括一主一从，一主多从等，在主库进行增删改操作，在从库进行读操作。
- 数据库高可用场景：主备，包括一往一备，多主多备等，在数据库无法访问时可以切换。
- 同构或异构数据的业务处理：需要处理的数据存储在不同的数据库中，包括同构（如都是 MySQL ），异构（如一个MySQL ，另外是 PG 或者 Oracle ）。

使用 Spring Boot 该如何处理多个数据库的读写，一般有以下几种策略：

- 多套数据源：即针对一个数据库建立一套数据处理逻辑，每套数据库都包括数据源配置、会话工厂（ sessionFactory ）、连接、SQL 操作、实体。各套数据库相互独立。
- 动态数据源：确定数量的多个数据源共用一个会话工厂，根据条件动态选取数据源进行连接、SQL 操作。
- 参数化变更数据源：根据参数添加数据源，并进行数据源切换，数据源数量不确定。通常用于对多个数据库的管理工作。

本系列文章“搞定SpringBoot多数据源”将针对以上几个策略进行描述，本文是第一篇：“多套数据源”，主要以主从场景为实例，结合代码，对多套数据源的实现进行描述，内容包括搭建 Spring Boot + MyBatis Plus 工程、多数据源配置、多数据源处理与使用逻辑。

本文所涉及到的[示例代码](https://github.com/mianshenglee/my-example/tree/master/multi-datasource):`https://github.com/mianshenglee/my-example/tree/master/multi-datasource`，读者可结合一起看。



# 2. 运行环境

- JAVA 运行环境:  `JDK1.8`
- Spring Boot:  `2.2.2.RELEASE`
- MyBatis Plus: `3.3.0`
- 开发IDE: `IDEA`
- 构建工具Maven: `3.3.9`
- MySQL : `5.6.26`
- [Lombok][3]: `1.18.10`



# 3. 多套数据源

多套数据源，顾名思义每一个数据库有一套独立的操作。从下往上，数据库、会话工厂、DAO操作，服务层都是独立的一套，如下所示：

![多套数据源](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/multi-datasource/multi-datasource-1.png)

本示例中，以一主一从两个数据库，两数据库的分别有一个表 `test_user`，表结构一致，为便于说明，两个表中的数据是不一样的。两个表结构可在[示例代码](https://github.com/mianshenglee/my-example/tree/master/multi-datasource)中的 `sql` 目录中获取。

## 3.1 搭建 Spring Boot 工程

### 3.1.1 初始化 Spring Boot 工程

使用 [spring.io](https://start.spring.io)构建初始 Spring Boot 工程，选用以下几个构件：

- Lombok: 用于简化操作
- Spring Configuration Processor：配置文件处理器
- Spring Web: 用于构建web服务
- MySQL Driver: 数据库驱动

### 3.1.2 添加 MyBatis Plus 依赖

[MyBatis Plus](https://mybatis.plus/) 是对 MyBatis 增强，简化 DAO 操作，提高数据库操作效率。依赖如下：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.3.0</version>
</dependency>
```

### 3.1.3 添加包结构

主要添加以下几外包：

```java
├─config ---------------------------------- // 数据源配置
├─controller ------------------------------ // web服务
├─entity ---------------------------------- // 实体类
│ ├─master 
│ └─slave 
├─mapper ---------------------------------- // dao操作类
│ ├─master 
│ └─slave 
└─vo -------------------------------------- // 视图返回对象  
```

> 注：
>
> - 由于示例简单，省略service层
> - 实体类及mapper均根据主从进行划分

## 3.2 多套数据源

### 3.2.1 独立数据库连接信息

Spring Boot 的默认配置文件是 `application.properties` ，由于有两个数据库配置，独立配置数据库是好的实践，因此添加配置文件 `jbdc.properties` ，添加以下自定义的主从数据库配置：

```properties
# master
spring.datasource.master.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.master.jdbc-url=jdbc:mysql://localhost:3306/mytest?useSSL=false&serverTimezone=GMT%2B8&characterEncoding=UTF-8
spring.datasource.master.username=root
spring.datasource.master.password=111111

# slave
spring.datasource.slave.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.slave.jdbc-url=jdbc:mysql://localhost:3306/my_test1?useSSL=false&serverTimezone=GMT%2B8&characterEncoding=UTF-8
spring.datasource.slave.username=root
spring.datasource.slave.password=111111
```

### 3.2.2 多套数据源配置

有了数据源连接信息，需要把数据源注入到 Spring 中。由于每个数据库使用独立的一套数据库连接，数据库连接使用的 `SqlSession` 进行会话连接，`SqlSession` 是由`SqlSessionFactory` 生成。因此，需要分别配置`SqlSessionFactory` 。以下操作均在 `config` 目录 下：

（1）添加 `DataSourceConfig` 配置文件，注入主从数据源

```java
@Configuration
@PropertySource("classpath:config/jdbc.properties")
public class DatasourceConfig {
    @Bean("master")
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource(){
        return DataSourceBuilder.create().build();
    }

    @Bean("slave")
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource(){
        return DataSourceBuilder.create().build();
    }
}
```

> - 注解 `PropertySource` 指定配置信息文件
> - 注解 `ConfigurationProperties` 指定主从配置前缀
> - 分别指定主从数据源的 bean 名称为 `master` ，`slave`

（2）添加 `MasterMybatisConfig` 配置文件，注入 Master 的`SqlSessionFactory`

```java
@Configuration
@MapperScan(basePackages = "me.mason.demo.basicmultidatasource.mapper.master", sqlSessionFactoryRef = "masterSqlSessionFactory")
public class MasterMybatisConfig {
    /**
     * 注意，此处需要使用MybatisSqlSessionFactoryBean，不是SqlSessionFactoryBean，
     * 否则，使用mybatis-plus的内置函数时就会报invalid bound statement (not found)异常
     */
    @Bean("masterSqlSessionFactory")
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("master") DataSource dataSource) throws Exception {
        // 设置数据源
        MybatisSqlSessionFactoryBean mybatisSqlSessionFactoryBean = new MybatisSqlSessionFactoryBean();
        mybatisSqlSessionFactoryBean.setDataSource(dataSource);
        //mapper的xml文件位置
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        String locationPattern = "classpath*:/mapper/master/*.xml";
        mybatisSqlSessionFactoryBean.setMapperLocations(resolver.getResources(locationPattern));
        //对应数据库的entity位置
        String typeAliasesPackage = "me.mason.demo.basicmultidatasource.entity.master";
        mybatisSqlSessionFactoryBean.setTypeAliasesPackage(typeAliasesPackage);
        return mybatisSqlSessionFactoryBean.getObject();
    }
}
```

> - 注解 `MapperScan` 指定那些包下的 `mapper` 使用本数据源，并指定使用哪个`SqlSessionFactory`，注意，此处的 `sqlSessionFactoryRef` 即本配置中的注入的 `SqlSessionFactory`。
> - 设置指定的数据源，此处是名为 `master` 的数据源，使用 `Qualifier` 指定。
> - MyBatis Plus 对应的 Mapper 若有自定义的 mapper.xml， 则使用 `setMapperLocations` 指定。
> - 若需要对实体进行别名处理，则使用 `setTypeAliasesPackage` 指定。

（3）添加 `SlaveMybatisConfig` 配置文件，注入Slave  的 `SqlSessionFactory` 

与（2）一致，把 master 改为 slave即可。

### 3.2.3 多套实体

在 MyBatis 配置中，实体设置 `typeAliases` 可以简化 xml 的配置，前面提到，使用 `typeAliasesPackage` 设置实体路径，在 `entity` 包下分别设置 `master` 和 `slave` 包，存放两个库对应的表实体，使用 Lombok 简化实体操作。如下：

```java
@Data
@TableName("test_user")
public class MasterTestUser implements Serializable {
    private static final long serialVersionUID = 1L;
    /** id */
    private Long id;
    /** 姓名 */
    private String name;
    ...
}
```

### 3.2.4 多套Mapper操作

在 `mapper` 包下，分别添加 `master` 和 `slave` 包，存放两个库对应的 Mapper ，由于 MyBatis Plus 本身已包含基本的 CRUD 操作，所以很多时候可以不用 xml 文件配置。若需要自定义操作，需要结合 xml文件，与此同时需要指定对应的 xml 文件所在目录。如下：

```java
@Repository
public interface MasterTestUserMapper extends BaseMapper<MasterTestUser> {
    /**
     * 自定义查询
     * @param wrapper 条件构造器
     */
    List<MasterTestUser> selectAll(@Param(Constants.WRAPPER)Wrapper<MasterTestUser> wrapper);
}
```

slave对应的Mapper与此类似



### 3.2.5 多套 mapper xml 文件

MyBatis Plus 的默认mapper xml 文件路径为 `classpath*:/mapper/**/*.xml`，即 `resources/mapper` 下，同样设置 `master` 及 `slave` 目录，分别存放对应的mapper xml 文件。以下是 master 的自定义操作：

``` xml
<mapper namespace="me.mason.demo.basicmultidatasource.mapper.master.MasterTestUserMapper">
    <select id="selectAll" resultType="masterTestUser">
        select * from test_user
        <if test="ew!=null">
          ${ew.customSqlSegment}
        </if>
    </select>
</mapper>
```



## 3.3 多数据源使用

经过上面的多套数据源配置，可知道，若需要操作哪个数据库，直接使用对应的 mapper 进行 CRUD 操作即可。如下为 Controller 中分别查询两个库，获取到的数据合在一起返回：

```java
@RestController
@RequestMapping("/user")
public class TestUserController {

    @Autowired
    private MasterTestUserMapper masterTestUserMapper;
    @Autowired
    private SlaveTestUserMapper slaveTestUserMapper;
    /**
     * 查询全部
     */
    @GetMapping("/listall")
    public Object listAll() {
        //master库，自定义接口查询
        QueryWrapper<MasterTestUser> queryWrapper = new QueryWrapper<>();
        List<MasterTestUser> resultData = masterTestUserMapper.selectAll(queryWrapper.isNotNull("name"));
        //slave库，mp内置接口
        List<SlaveTestUser> resultDataSlave = slaveTestUserMapper.selectList(null);
        //返回
        Map<String, Object> result = new HashMap<>();
        result.put("master" , resultData);
        result.put("slave" , resultDataSlave);
        return ResponseResult.success(result);
    }

}
```

> - 使用Autowired注解注入对应的mapper
>- 使用对应数据库的 mapper 进行业务操作
> - 根据业务在数据库中执行相应的操作，如主只做增删改操作、从只读操作

至此，多数据源的实现已完成，当前示例是两个同构的数据库，当然，若是异构的数据库，或者多于两个的数据库，处理方式是一样的，只不过是把数据源增加一套而已。

# 4. 优缺点

由上述说明，我们可以总结一下使用多套数据源的方法进行多数据库操作，它的优缺点是什么。

## 4.1 优点

- 简单、直接：一个库对应一套处理方式，很好理解。
- 符合开闭原则（ OCP ）：开发的设计模式告诉我们，对扩展开放，对修改关闭，添加多一个数据库，原来的那一套不需要改动，只添加即可。

## 4.2 缺点

- 资源浪费：针对每一个数据源写一套操作，连接数据库的资源也是独立的，分别占用同样多的资源。`SqlSessionFactory` 是一个工厂，建议是使用单例，完全可以重用，不需要建立多个，只需要更改数据源即可，跟多线程，使用线程池减少资源消耗是同一道理。
- 代码冗余：在前面的多数据源配置中可以看出，其实 master 和 slave 的很多操作是一样的，只是改个名称而已，因此会造成代码冗余。
- 缺乏灵活：所有需要使用的地方都需要引入对应的mapper，对于很多操作，只是选择数据源的不一样，代码逻辑是一致的。另外，对于一主多从的情况，若需要对多个从库进行负载均衡，相对比较麻烦。

正因为有上述的缺点，所以还有改进的空间。于是就有了动态数据源，至于动态数据源如何实现，下回分解。

# 5. 总结

本文对多个数据库的操作进行了初步探讨，并对使用多套源的策略进行讲解，结合主从代码示例，搭建了 Spring Boot + MyBatis Plus 工程，配置多数据源及使用 Mapper 进行多数据源操作，最后对多套数据源的优缺点进行总结。希望小伙伴们可以对多数据源操作有个初步印象。

本文有配套的[示例代码](https://github.com/mianshenglee/my-example/tree/master/multi-datasource)，有兴趣的可以跑一下示例来感受一下。



# 参考资料

- [Spring主从数据库的配置和动态数据源切换原理](https://www.liaoxuefeng.com/article/1182502273240832): `https://www.liaoxuefeng.com/article/1182502273240832`
- [多数据源与动态数据源的权衡](https://juejin.im/post/5b790a866fb9a019ea01f38c): `https://juejin.im/post/5b790a866fb9a019ea01f38c`
- [谈谈Spring Boot 数据源加载及其多数据源简单实现](https://juejin.im/post/5cb0023d5188250df17d4ffc): `https://juejin.im/post/5cb0023d5188250df17d4ffc`
- [Spring Boot 和 MyBatis 实现多数据源、动态数据源切换](https://juejin.im/post/5a927d23f265da4e7e10d740): `https://juejin.im/post/5a927d23f265da4e7e10d740`


# 往期文章

- [java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)

- [2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)

- [springboot+apache前后端分离部署https](https://mp.weixin.qq.com/s/hiJdsjdDC07axk-_sAkyVQ)

- [springboot+logback 日志输出企业实践（下）](https://mp.weixin.qq.com/s/ha4LaR-E1gDxfUZ11neI-Q)

- [springboot+logback 日志输出企业实践（上）]( https://mp.weixin.qq.com/s/Ti5i9vv9S1j4za5q11RWeA )

  

我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![mason](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)