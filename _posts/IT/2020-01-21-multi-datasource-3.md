---
layout: post
title: 搞定SpringBoot多数据源(3)：参数化变更源
category: IT
tags: multi-datasource java springboot
keywords: 
description: 
---

![2020鼠年大吉](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/multi-datasource/2020-spring-festivel.jpg)

> 春节将至，今天放假了，在此祝小伙伴们新春大吉，身体健康，思路清晰，永远无BUG！

---



> 一句话概括：参数化变更源意思是根据参数动态添加数据源以及切换数据源，解决不确定数据源的问题。

# 1. 引言

经过前面两篇文章对于 Spring Boot 处理多个数据库的策略讲解，相信大家已经对多数据源和动态数据源有了比较好的了解。如需回顾，请见：

- 《[搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)》
- 《[搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)》

在前面文章中，留了一个思考题，无论是多套源还是动态数据源，相对来说还是固定的数据源（如一主一从，一主多从等），即在编码时已经确定的数据库数量，只是在具体使用哪一个时进行动态处理。如果数据源本身并不确定，或者说需要根据用户输入来连接数据库，这时，如何处理呢？可以想象现在我们有一个需求，需要对数据库进行连接管理，用户可以输入对应的数据库连接信息，然后可以查看数据库有哪些表。这就跟平时使用的数据库管理软件有点类似了，如 MySQL Workbench、Navicat、SQLyog，下图是SQLyog截图：

![SQLyog](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/multi-datasource/connect.png)

本文基于前面的示例，添加一个功能，根据用户输入的数据库连接信息，连接数据库，并返回数据库的表信息。内容包括动态添加数据源、动态代理简化数据源操作等。

本文所涉及到的[示例代码](https://github.com/mianshenglee/my-example/tree/master/multi-datasource):`https://github.com/mianshenglee/my-example/tree/master/multi-datasource`，读者可结合一起看。



# 2. 参数化变更源说明

## 2.1 解决思路

Spring Boot 的动态数据源，本质上是把多个数据源存储在一个 Map 中，当需要使用某个数据源时，从 Map 中获取此数据源进行处理。在动态数据源处理时，通过继承抽象类 `AbstractRoutingDataSource`  可实现此功能。既然是 Map ，如果有新的数据源，把新的数据源添加到此 Map 中就可以了。这就是整个解决思路。

但是，查看 `AbstractRoutingDataSource`  源码，可以发现，存放数据源的 Map `targetDataSources` 是 private 的，而且并没有提供对此 Map 本身的操作，它提供的是两个关键操作：`setTargetDataSources` 及 `afterPropertiesSet` 。其中 `setTargetDataSources` 设置整个 Map 目标数据源，`afterPropertiesSet` 则是对 Map 目标数据源进行解析，形成最终使用的 `resolvedDataSources`，可见以下源码：

```java
this.resolvedDataSources = new HashMap(this.targetDataSources.size());
this.targetDataSources.forEach((key, value) -> {
    Object lookupKey = this.resolveSpecifiedLookupKey(key);
    DataSource dataSource = this.resolveSpecifiedDataSource(value);
    this.resolvedDataSources.put(lookupKey, dataSource);
});
```

因此，为实现动态添加数据源到 Map 的功能，我们可以根据这两个关键操作进行处理。

## 2.2 流程说明

1. 用户输入数据库连接参数（包括IP、端口、驱动名、数据库名、用户名、密码）
2. 根据数据库连接参数创建数据源
3. 添加数据源到动态数据源中
4. 切换数据源
5. 操作数据库

# 3. 实现参数化变更源

> 说明，下面的操作基于之前文章的示例，基本的工程搭建及配置不再重复说明，有需要可参考文章。

## 3.1 改造动态数据源

### 3.1.1 动态数据源添加功能

为了可以动态添加数据源到 Map ，我们需要对动态数据源进行改造。如下：

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    private Map<Object, Object> backupTargetDataSources;

    /**
     * 自定义构造函数
     */
    public DynamicDataSource(DataSource defaultDataSource,Map<Object, Object> targetDataSource){
        backupTargetDataSources = targetDataSource;
        super.setDefaultTargetDataSource(defaultDataSource);
        super.setTargetDataSources(backupTargetDataSources);
        super.afterPropertiesSet();
    }

    /**
     * 添加新数据源
     */
    public void addDataSource(String key, DataSource dataSource){
        this.backupTargetDataSources.put(key,dataSource);
        super.setTargetDataSources(this.backupTargetDataSources);
        super.afterPropertiesSet();
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceContextHolder.getContextKey();
    }
}
```

> - 添加了自定义的 `backupTargetDataSources` 作为原 `targetDataSources` 的拷贝
> - 自定义构造函数，把需要保存的目标数据源拷贝到自定义的 Map 中
> - 添加新数据源时，依然使用 `setTargetDataSources` 及 `afterPropertiesSet` 完成新数据源添加。
> - 注意：`afterPropertiesSet` 的作用很重要，它负责解析成可用的目标数据源。

### 3.1.2 动态数据源配置

原来在创建动态数据源时，使用的是无参数构造函数，经过前面改造后，使用有参构造函数，如下：

```java
@Bean
@Primary
public DataSource dynamicDataSource() {
    Map<Object, Object> dataSourceMap = new HashMap<>(2);
    dataSourceMap.put(DataSourceConstants.DS_KEY_MASTER, masterDataSource());
    dataSourceMap.put(DataSourceConstants.DS_KEY_SLAVE, slaveDataSource());
    //有参构造函数
    return new DynamicDataSource(masterDataSource(), dataSourceMap);
}
```

## 3.2 添加数据源工具类

### 3.2.1 Spring 上下文工具类

在Spring Boot 使用过程中，经常会用到 Spring 的上下文，常见的就是从 Spring 的 IOC 中获取 bean 来进行操作。由于 Spring 使用的 IOC 基本上把 bean 都注入到容器中，因此需要 Spring 上下文来获取。我们在 context 包下添加 `SpringContextHolder` ，如下：

```java
@Component
public class SpringContextHolder implements ApplicationContextAware {
    private static ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextHolder.applicationContext = applicationContext;
    }
    /**
     * 返回上下文
     */
    public static ApplicationContext getContext(){
        return SpringContextHolder.applicationContext;
    }
}
```

通过 `getContext` 就可以获取上下文，进而操作。



### 3.2.2 数据源操作工具

通过参数添加数据源，需要根据参数构造数据源，然后添加到前面说的 Map 中。如下：

```java
public class DataSourceUtil {
    /**
     * 创建新的数据源，注意：此处只针对 MySQL 数据库
     */
    public static DataSource makeNewDataSource(DbInfo dbInfo){
        String url = "jdbc:mysql://"+dbInfo.getIp() + ":"+dbInfo.getPort()+"/"+dbInfo.getDbName()
                +"?useSSL=false&serverTimezone=GMT%2B8&characterEncoding=UTF-8";
        String driveClassName = StringUtils.isEmpty(dbInfo.getDriveClassName())? "com.mysql.cj.jdbc.Driver":dbInfo.getDriveClassName();
        return DataSourceBuilder.create().url(url)
                .driverClassName(driveClassName)
                .username(dbInfo.getUsername())
                .password(dbInfo.getPassword())
                .build();
    }

    /**
     * 添加数据源到动态源中
     */
    public static void addDataSourceToDynamic(String key, DataSource dataSource){
        DynamicDataSource dynamicDataSource = SpringContextHolder.getContext().getBean(DynamicDataSource.class);
        dynamicDataSource.addDataSource(key,dataSource);
    }

    /**
     * 根据数据库连接信息添加数据源到动态源中
     * @param key
     * @param dbInfo
     */
    public static void addDataSourceToDynamic(String key, DbInfo dbInfo){
        DataSource dataSource = makeNewDataSource(dbInfo);
        addDataSourceToDynamic(key,dataSource);
    }
}
```

> - 通过 `DataSourceBuilder` 及相应的参数来构造数据源，注意此处只针对 MySQL 作处理，其它数据库的话，对应的 url 及 DriveClassName 需作相应的变更。
> - 添加数据源时，通过 Spring 上下文获取动态数据源的 bean，然后添加。

## 3.3 使用参数变更数据源

前面两步已实现添加数据源，下面我们根据需求（根据用户输入的数据库连接信息，连接数据库，并返回数据库的表信息），看看如何使用它。

### 3.3.1 添加查询数据库表信息的 Mapper

通过 MySQL 的 `information_schema` 可以获取表信息。

```java
@Repository
public interface TableMapper extends BaseMapper<TestUser> {
    /**
     * 查询表信息
     */
    @Select("select table_name, table_comment, create_time, update_time " +
            " from information_schema.tables " +
            " where table_schema = (select database())")
    List<Map<String,Object>> selectTableList();
}
```

### 3.3.2 定义数据库连接信息对象

把数据库连接信息通过一个类进行封装。

```java
@Data
public class DbInfo {
    private String ip;
    private String port;
    private String dbName;
    private String driveClassName;
    private String username;
    private String password;
}
```

### 3.3.3 参数化变更源并查询表信息

在 controller 层，我们定义一个查询表信息的接口，根据传入的参数，连接数据源，返回表信息：

```java
/**
 * 根据数据库连接信息获取表信息
 */
@GetMapping("table")
public Object findWithDbInfo(DbInfo dbInfo) throws Exception {
    //数据源key
    String newDsKey = System.currentTimeMillis()+"";
    //添加数据源
    DataSourceUtil.addDataSourceToDynamic(newDsKey,dbInfo);
    DynamicDataSourceContextHolder.setContextKey(newDsKey);
    //查询表信息
    List<Map<String, Object>> tables = tableMapper.selectTableList();
    DynamicDataSourceContextHolder.removeContextKey();
    return ResponseResult.success(tables);
}
```

> - 访问地址 `http://localhost:8080/dd/table?ip=localhost&port=3310&dbName=mytest&username=root&password=111111` ，对应数据库连接参数。
> - 此处数据源的 key 是无意义的，建议根据实际场景设置有意义的值



# 4. 动态代理消除模板代码

前面已经完成了参数化切换数据源功能，但还有一点就是有模板代码，如添加数据源、切换数据源、对此数据源进行CURD操作、释放数据源，如果每个地方都这样做，就很繁琐，这个时候，就需要用到动态代理了，可参数我之前的文章：[java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)。此处，使用 JDK 自带的动态代理，实现参数化变更数据源的功能，消除模板代码。

## 4.1 添加 JDK 动态代理

添加 proxy 包，添加 `JdkParamDsMethodProxy` 类，实现 `InvocationHandler` 接口，在 `invoke` 中编写参数化切换数据源的逻辑即可。如下：

```java
public class JdkParamDsMethodProxy implements InvocationHandler {
    // 代理对象及相应参数
    private String dataSourceKey;
    private DbInfo dbInfo;
    private Object targetObject;
    public JdkParamDsMethodProxy(Object targetObject, String dataSourceKey, DbInfo dbInfo) {
        this.targetObject = targetObject;
        this.dataSourceKey = dataSourceKey;
        this.dbInfo = dbInfo;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //切换数据源
        DataSourceUtil.addDataSourceToDynamic(dataSourceKey, dbInfo);
        DynamicDataSourceContextHolder.setContextKey(dataSourceKey);
        //调用方法
        Object result = method.invoke(targetObject, args);
        DynamicDataSourceContextHolder.removeContextKey();
        return result;
    }

    /**
     * 创建代理
     */
    public static Object createProxyInstance(Object targetObject, String dataSourceKey, DbInfo dbInfo) throws Exception {
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader()
                , targetObject.getClass().getInterfaces(), new JdkParamDsMethodProxy(targetObject, dataSourceKey, dbInfo));
    }
}
```

> - 代码中，需要使用的参数通过构造函数传入
> - 通过 `Proxy.newProxyInstance` 创建代理，在方法执行时( `invoke` ) 进行数据源添加、切换、数据库操作、清除等

## 4.2 使用代理实现功能

有了代理，在添加和切换数据源时就可以擦除模板代码，前面的业务代码就变成：

```java
@GetMapping("table")
    public Object findWithDbInfo(DbInfo dbInfo) throws Exception {
        //数据源key
        String newDsKey = System.currentTimeMillis()+"";
        //使用代理切换数据源
        TableMapper tableMapperProxy = (TableMapper)JdkParamDsMethodProxy.createProxyInstance(tableMapper, newDsKey, dbInfo);
        List<Map<String, Object>> tables = tableMapperProxy.selectTableList();
        return ResponseResult.success(tables);
    }
```

通过代理，代码就简洁多了。

# 5. 总结

本文基于动态数据源，对参数化变更数据源及应用场景进行了说明，提出连接数据库，查询表信息的功能需求作为示例，实现根据参数构建数据源，动态添加数据源功能，对参数化变更数据源的使用进行讲解，最后使用动态代理简化操作。本篇文章偏重代码实现，小伙伴们可以新手实践来加深认知。

本文配套的示例，[示例代码](https://github.com/mianshenglee/my-example/tree/master/multi-datasource)，有兴趣的可以运行示例来感受一下。



# 参考资料

- [Spring主从数据库的配置和动态数据源切换原理](https://www.liaoxuefeng.com/article/1182502273240832): `https://www.liaoxuefeng.com/article/1182502273240832`
- [谈谈Spring Boot 数据源加载及其多数据源简单实现](https://juejin.im/post/5cb0023d5188250df17d4ffc): `https://juejin.im/post/5cb0023d5188250df17d4ffc`


# 往期文章

- [搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)

- [搞定SpringBoot多数据源(1)：多套源策略](https://mianshenglee.github.io/2020/01/13/multi-datasource-1.html)

- [java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)

- [2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)

- [springboot+apache前后端分离部署https](https://mp.weixin.qq.com/s/hiJdsjdDC07axk-_sAkyVQ)

- [springboot+logback 日志输出企业实践（下）](https://mp.weixin.qq.com/s/ha4LaR-E1gDxfUZ11neI-Q)

- [springboot+logback 日志输出企业实践（上）]( https://mp.weixin.qq.com/s/Ti5i9vv9S1j4za5q11RWeA )

  

我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![mason](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)