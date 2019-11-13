---
layout: post
title: springboot+swagger接口文档企业实践（上）
category: IT
tags: springboot swagger doc
keywords: 
description: 
---

tags:  springboot swagger doc

---


> 一句话概括：对于产品开发，特别是前后端分离的开发模式，接口文档是连接前后端的枢纽，本文对springboot+swagger在企业的实践进行详细描述。

# 1.引言

在软件开发过程中，接口设计与接口文档编写是重要的一环，特别是在前后端分离的情况下，接口说明文档是开发人员之间的连接点。接口文档的编写有很多方式，可以使用word，也可以使用编写工具如小幺鸡，这些基本属于脱离代码编写的文档，如果接口变化，需要额外修改文档，增加工作量。如何提高写接口文档效率，在springboot开发中，结合swagger来提供接口文档说明是一个很好的实践，通过配置与注解，在编写接口代码过程中，同时也把接口文档写好，接口需要变更时，文档也同时变更，具有工作量少，效率高，接口文档直观简洁，可实时调用验证等好处。本文基本springboot2+swagger2，结合在企业中的实践，对接口文档的编写进行详细说明，具体有如下内容：

- swagger介绍及文档生成说明
- 使用springboot2+swagger2构建示例工程及配置描述
- 使用注解添加文档内容说明
- 使用全局参数进行接口认证

如需看源码，本文[示例工程地址](https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/hello-swagger-demo)：`https://github.com/mianshenglee/my-example`

# 2.swagger简介

### 2.1 swagger 介绍

swagger[官网地址]( https://swagger.io/ )：` https://swagger.io `，从官网看出，它是一个规范 （OpenAPI Specification，OAS） 和完整的框架（如编辑器 Swagger Editor ，显示组件 Swagger Editor ，代码生成 Swagger Codegen ），用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。既然是规范，它定义了一系列与接口描述相关的规则，如文档格式，文件结构，数据类型，请求方法等等，可[参考官方规范说明]( https://swagger.io/specification/v2/ )：` https://swagger.io/specification/v2/ `，文件设计和编写时一般是 YAML格式 （方便编写），在传输时则会使用JSON（通用性强）。有了接口描述文件后，需要进行友好显示，swagger提供了[Swagger-UI]( https://swagger.io/tools/swagger-ui/ )：` https://swagger.io/tools/swagger-ui/ `。这个组件的作用就是把已经按规范写好的yaml文件，通过接口文档形式显示。我们可以下载这个组件本地运行，它是一个静态网页，放到web容器（如apache，nginx）来运行。也可以使用官网的在线版本。如下：

![Swagger-UI]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-ui.jpg )

至此，我们知道使用swagger进行接口文档编写步骤是：（1）编写符合OAS的接口文件，（2）使用swagger-ui进行显示。

### 2.2 springfox、swagger与springboot

从前面可知，编写符合OAS的接口文件还是得手工编写，swagger-ui显示也不方便（需要人工离线部署或使用在线服务），现在java领域的web开发，基本都使用springmvc开发，有没有更方便的方式来结合swagger进行接口文件编写，而不需要 手工编写，减少工作量？

牛人` Marty Pitt`编写了一个基于Spring的组件[swagger-springmvc](https://github.com/martypitt/swagger-springmvc-example):`https://github.com/martypitt/swagger-springmvc-example`，用于将swagger集成到springmvc中来。springfox则是从这个组件发展而来，它可以基于spring自动生成JSON的API文档，现在通过`https://github.com/martypitt/swagger-springmvc/`地址会直接跳转到[springfox的github页面]( https://github.com/springfox/springfox )，它的[官网]( http://springfox.io/ )：`http://springfox.io`。当前springfox已经是发布了多个版本，本文使用的是`springfox-swagger2`的2.7.0的版本。对于swagger-ui，它也提供了`springfox-swagger-ui`，集成接口文档显示页面。这样，前面swagger的两个步骤都可以自动完成。而现在我们开发java web应用基本都是使用springboot进行开发，使用它的自动配置与注解，更加方便开发。因此，基于swagger的规范，使用springfox的组件（swagger2和swagger-ui），结合springboot，可以让接口文档变得非常简单，有效。总的来说，springfox是对swagger规范的在springmvc和springboot开发中的自动化实现。本文后续就是以这种方式进行具体的使用描述。

# 3. 使用springboot+swagger构建接口文档

本章节将使用springboot2+swagger2构建示例工程，并对基本配置进行描述。

## 3.1 springboot示例工程搭建

（1） 创建项目：通过[ Spring Initializr 页面](https://start.spring.io/)生成一个空 Spring Boot 项目，添加web依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

（2）编写示例接口：

- 首先添加几个通用包：vo，controller，service，model，exception，分别存放视图层模型，控制层，服务层，数据模型层，异常对象等代码。
- 示例数据模型是User，对应有UserController，UserService，统一返回视图模型ResponseResult，统一异常类UniRuntimeException。
- 针对用户管理，使用restful风格的接口，定义了对用户的增删改查，对应(POST,DELETE,PUT,GET请求)，如下：

``` java
@RestController
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;
    
    /**
     * 查询单个用户
     */
    @GetMapping("/{userId}")
    public ResponseResult getUserById(@PathVariable long userId) {...}
    /**
     * 查询多个用户
     */
    @GetMapping()
    public ResponseResult getUsers() {...}
    /**
     * 添加多个用户
     */
    @PostMapping()
    public ResponseResult addUsers(@RequestBody List<User> users) {...}
    /**
     * 添加单个用户
     */
    @PostMapping("/user")
    public ResponseResult addUser(@RequestBody User user) {...}
    /**
     * 更新单个用户
     */
    @PutMapping("/user")
    public ResponseResult updateUser(@RequestBody User user) {...} 
    /**
     * 删除单个用户
     */
    @DeleteMapping("/user")
    public ResponseResult deleteUser(long userId) {...}
}
```

> 注意：此处我们只关注接口的定义、参数和返回值，具体实体不列出（详细可看[源码]( https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/hello-swagger-demo)：` https://github.com/mianshenglee/my-example`）。

至此，示例工程可以正常运行，具有以下接口：

1. `GET  /users/{userId}` 获取单个用户
2. `GET  /users` 获取多个用户
3. `POST /users `  添加多个用户
4. `POST /users/user` 添加单个用户
5. `PUT /users/user` 更新单个用户
6. `DELTE /users/user?userId=xx` 删除单个用户

> 注意：接口的RequestMapping，如果不指定httpMethod，springfox会把这个方法的所有动作GET/POST/PUT/PATCH/DELETE的方法都列出来，因此，写接口时，请指定实际的动作。

## 3.2 引入swagger2与基本配置

### 3.2.1 添加springfox-swagger依赖

为了可以自动化生成符合swagger的接口描述文件，需要添加springfox的依赖，如下：

``` xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>${swagger.version}</version>
</dependency>
```

### 3.2.2 配置swagger

添加config包用于存放配置文件，添加以下Swagger2Config.java配置，如下：

```java
@Configuration
@EnableSwagger2
public class Swagger2Config {
private ApiInfo apiInfo() {
	@Bean
	public Docket api(){
		return new Docket(DocumentationType.SWAGGER_2)
				.apiInfo(apiInfo())
				.select()
				.apis(RequestHandlerSelectors.any())
				.paths(PathSelectors.any())
				.build();
	}
    private ApiInfo apiInfo() {
		return new ApiInfoBuilder().title("接口说明文档")
				.description("API文档描述")
				.version("0.0.1-SNAPSHOT")
				.build();
	}
}
```

此文件配置了Docket的Bean，包括api的基本信息，需要显示的接口(apis)，需要过滤的路径(paths)，这样，就可以生成符合规范的接口文档。

### 3.2.3 查看swagger自动生成的描述文档

使用上面的配置集成swagger2后，启动项目，输入` http://localhost:8080/swaggerdemo/v2/api-docs  `即可查看JSON格式的接口描述文档。如下所示：

![1573465257350]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-api-doc.jpg )

此文档是符合OAS规范，但离可视化显示和使用还差一步，就是如何把这些信息展示在界面上。

## 3.3 添加swagger-ui界面交互

为了能可视化显示接口文档，springfox已提供界面显示组件，添加以下依赖：

``` xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>${swagger.version}</version>
</dependency>
```

添加此依赖后，引入的springfox-swagger-ui.jar包中有`swagger-ui.html`页面，使用地址` http://localhost:8080/swaggerdemo/swagger-ui.html `即可查看接口文档页面。在此文档中，可以查看接口定义、输入参数、返回值，也可以使用`try it out`调用接口进行调试。如下：

![1573465932646]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-ui2.jpg )

至此，通过前面示例，已经以最简单的方式实现了使用swagger对接口文档的自动生成和UI显示，但相对于企业中实际开发使用接口文档，还是有一定距离，包括：

- 不能根据环境选择显示文档（可能开发环境需要显示，正式环境不需要）
- 接口没有功能说明和详细描述
- 输入参数详细说明和示例说明
- 返回值的详细说明和示例说明
- 所有controller接口都返回了，没有按需过滤需要的接口
- 所有接口都可以调试，存在权限认证问题

针对这些问题，在企业实践中，需要进行处理。

# 4. 【企业实践】配置参数化与包过滤

## 4.1 配置参数化

一份接口文档，一般都会有关于此文档的基本信息，包括文档标题、描述、接口版本、联系人等信息，swagger已提供了对应的配置项，如上面的示例中的`apiInfo()`，把基本信息写在代码中进行配置，这样对修改不友好，因此，最好是把这些配置参数化，形成swagger的独立配置文件，避免修改基本信息导致代码的改动。

### 4.1.1 添加基本信息配置文件

在`resources/config`目录下，添加`swagger.properties`文件，把配置信息放到此文件中。

``` properties
swagger.basePackage =me.mason.helloswagger.demo.controller
swagger.title = 应用API说明文档
swagger.description = API文档描述
swagger.version = 0.0.1-SNAPSHOT
swagger.enable = true
swagger.contactName = mason
swagger.contactEmail =
swagger.contactUrl =
swagger.license =
swagger.licenseUrl =
```

对应的，在`config`包下，添加SwaggerInfo对象，对应上述配置文件：

``` java
@Component
@ConfigurationProperties(prefix = "swagger")
@PropertySource("classpath:/config/swagger.properties")
@Data
public class SwaggerInfo {
    private String basePackage;
    private String antPath;
    private String title = "HTTP API";
    private String description = "Swagger 自动生成接口文档";
    private String version ;
    private Boolean enable;
    private String contactName;
    private String contactEmail;
    private String contactUrl;
    private String license;
    private String licenseUrl;
}
```

官方建议对于这种自定义的配置，最好添加以下`spring-boot-configuration-processor`依赖，以生成配置元数据，否则在IDEA中，会有`"spring boot configuration annotation processor not found in classpath"`的警告：

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

### 4.1.2 集成配置swagger文档

有了`SwaggerInfo`配置类，在集成swagger2的`apiInfo()`可以注入使用，以后需要改动时修改配置文件即可。

```java
private ApiInfo apiInfo() {
    return new ApiInfoBuilder().title(swaggerInfo.getTitle())
        .description(swaggerInfo.getDescription())
        .version(swaggerInfo.getVersion())
        .licenseUrl(swaggerInfo.getLicenseUrl())
        .contact(
            new Contact(swaggerInfo.getContactName()
            ,swaggerInfo.getContactUrl()
            ,swaggerInfo.getContactEmail()))
        .build();
}
```

## 4.2 配置是否启用文档

对于不同环境，有可能需要启用和关闭接口文档，如在开发环境我们需要显示接口文档，但在正式线上环境，有时我们不希望对外公布接口。此需求可以使用`enable`配置即可，结合配置文件中有`swagger.enable`配置项(false关闭，true启用)。如下：

``` java
.enable(swaggerInfo.getEnable())
```

## 4.3 包过滤

swagger组件会自动扫描有`Controller`注解和`RequestMapping`注解的接口，转换为接口描述。在开发过程中，我们不希望项目中所有的接口都显示出来，而是想把特定的包下的接口显示即可。使用方法`apis()`，结合配置文件，可达到此效果。配置文件中`swagger.basePackage`配置（多个包使用";"分隔）。添加以下过滤函数：

``` java
private List<Predicate<RequestHandler>> apisFilter() {
    List<Predicate<RequestHandler>> apis = new ArrayList<>();
    String basePackageStr = swaggerInfo.getBasePackage();
    //包过滤
    if (StrUtil.isNotEmpty(basePackageStr)) {
        //支持多个包
        String[] basePackages = basePackageStr.split(";");
        if (null != basePackages && basePackages.length > 0) {
            Predicate<RequestHandler> predicate = input -> {
                // 按basePackage过滤
                Class<?> declaringClass = input.declaringClass();
                String packageName = declaringClass.getPackage().getName();
                return Arrays.asList(basePackages).contains(packageName);
            };
            apis.add(predicate);
        }
    }
    return apis;
}
```

上述功能是通过扫描到的接口与配置的包比较，在配置的包下则返回，否则过滤掉。然后在配置Docket时，添加此过滤。

``` java
builder = builder.apis(Predicates.or(apisFilter()));
```

此处使用`Predicates.or`，表示对返回的进行or操作，true的即显示。

# 5.【企业实践】使用注解添加文档内容

经过上面的配置化，我们已经可以灵活构建文档了。但文档的内容还不丰富，对接口的描述不详细，`springfox`把对接口的描述都通过注解的方式来完成，主要包括以下几种注解：

1. 数据模型注解：`@ApiModel`，实体描述；`@ApiModelProperty`，实体属性描述
2. 接口注解：`@ApiOperation`，接口描述；`@ApiIgnore`，忽略此接口
3. 控制器注解：`@Api`，控制器描述
4. 输入参数注解：`@ApiImplicitParams`，接口多个参数；`@ApiImplicitParam`，单个参数；`@ApiParam`，单个参数描述
5. 响应数据注解：`@ResponseHeader`，响应值header；`@ApiResponses`，响应值集；`@ApiResponse`，单个响应值

下面我们对注解的使用进行说明，并用注解添加示例中的接口描述。

## 5.1 数据模型(model)注解说明

开发过程中，对于MVC模式的接口，使用M（Model）进行数据通信，因此，需要对此数据模型进行有效的描述。对应的注解有：

- `@ApiModel`：实体描述
- `@ApiModelProperty`：实体属性描述。

下表为`@ApiModelProperty`的使用说明：

| **注解属性**    | **类型** | **描述**                 |
| :-------------- | :------- | :----------------------- |
| value           | String   | 字段说明。               |
| name            | String   | 重写字段名称。           |
| dataType        | Stirng   | 重写字段类型。           |
| required        | boolean  | 是否必填。               |
| example         | Stirng   | 举例说明。               |
| hidden          | boolean  | 是否在文档中隐藏该字段。 |
| allowEmptyValue | boolean  | 是否允许为空。           |
| allowableValues | String   | 该字段允许的值。         |

本示例中，即`User`类，使用如下注解进行描述：

```java
@ApiModel
public class User {
    @ApiModelProperty(value="id",required = true,example = "1")
    private long id;
    @ApiModelProperty(value="姓名",required = true,example = "张三")
    private String name;
    @ApiModelProperty(value="年龄",example = "10")
    private int age;
    @ApiModelProperty(value="密码",required = true,hidden = true)
    private String password;
}
```

这样，在接口文档中，使用`User`作为输入参数时，可以看到这个模型的描述，如下：

![1573550660883]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-model.jpg )

## 5.2 控制器及接口注解说明

springfox 在自动化生成文档时，如果不使用注解进行描述，控制器和接口基本是把代码的类名，方法名，接口映射作为显示的内容。对于控制器和具体某个接口的描述，分别有：

- `@ApiIgnore`: Swagger 文档不会显示拥有该注解的接口 

-  `@Api`: 对控制器的描述

| **注解属性** | **类型** | **描述**                           |
| :----------- | :------- | :--------------------------------- |
| tags         | String[] | 控制器标签。                       |
| description  | String   | 控制器描述（该字段被申明为过期）。 |

-   `@ApiOperation`: 对接口的描述。

| **注解属性** | **类型** | **描述**       |
| :----------- | :------- | :------------- |
| value        | String   | 接口说明。     |
| notes        | String   | 接口发布说明。 |
| tags         | Stirng[] | 标签。         |
| response     | Class<?> | 接口返回类型。 |
| httpMethod   | String   | 接口请求方式。 |

在本示例中，在`UserController`中进行相应描述，如下：

```java
@Api(tags = "用户管理接口", description = "User Controller")
public class UserController {
    @ApiOperation(value = "根据ID获取单个用户信息", notes = "根据ID返回用户对象")
    ...//省略
 @ApiOperation(value = "添加多个用户", notes = "使用JSON以数组形式添加多个用户")
    ...//其它
}
```

使用这些注解后，显示的文档如下：

![1573551666969]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-ui3.jpg )



## 5.3 接口输入参数注解说明

### 5.3.1 SpringMVC的控制器参数使用

在介绍Swagger的接口输入参数注解说明前，有必要对SpringMVC的接口参数有一个清晰的了解。我们知道SpringMVC是通过`@RequestMapping`把方法与请求URL对应起来，而调用接口方法时需要把参数传给控制器。一般来说，SpringMVC的参数传递包括以下几种：

- 无注解参数
- `@RequestParam`注解参数
- 数组参数
- JSON对象参数
- REST风格参数

为方便解释，在本示例中，提供了`ParamDemoController`文件，里面对应给出了各种参数的示例，读者可参考查看，下面对这几种参数进行描述。

**（1）无注解参数**

无注解下获取参数，要求参数名称与HTTP请求参数名称一致，参数可以为空。如下接口：

```java
@GetMapping("/no/annotation")
public Map<String,Object> noAnnotation(Integer intVal, Long longVal, String str){}
```

调用时，可使用`http://localhost:8080/no/annotation?inVal=10&longVal=100`，参数`str`可为空。

**（2）`@RequestParam`注解参数**

使用注解RequestParam获取参数，可以指定HTTP参数和方法参数名的映射，而且不能为空（但可通过required设置false来允许为空）。如下接口：

```java
@GetMapping("/annotation")
public Map<String,Object> annotation(@RequestParam("int_val") Integer intVal,@RequestParam("long_val") Long longVal,@RequestParam(value="str_val", required = false) String str){}
```

调用方法跟上面无注解的一样。但如果没有设置`required=false`，则必须传入`str`参数。

**（3）数组参数**

使用数组做为参数，调用接口时，数组元素使用逗号(`,`)分隔。如下接口：

```java
@GetMapping("/array")
public Map<String,Object> requestArray(int[]intArr,long[]longArr,String[] strArr){}
```

调用此接口，可使用`http://localhost:8080/array?intArr=10,11,12&longArr=100,101,102&strArr=a,b,c`。

**（4）JSON对象参数**

当传输的数据相对复杂，或者与定义的模型相关，则需要把参数组装成JSON进行传递，当前端调用接口时使用`JSON`请求体，SpringMVC可通过`@RequestBody`注解，实现JSON参数的映射，并进行合适的对象转换。如下：

```java
@PostMapping("/add/user")
public User addUser(@RequestBody User user){}
```

调用接口时，使用JSON传递参数，通过RequestBody注解得到JSON参数,映射转换为User对象。如果是数组（List）对象，同样支持。

**（5）REST风格参数**

对于REST风格的URL，参数是写在URL中，如`users/1`，其中`1`是用户ID参数。对于此类参数，SpringMVC使用`@PathVariable`进行实现，其中使用`{}`作为占位符。如下：

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable("id") Long id){}
```

### 5.3.2 swagger输入参数注解说明

对于上面提到的的SpringMVC参数，springfox已经做了自动处理，在`swagger-ui`文档中显示时，也会根据参数类型进行显示。在swagger中，参数类型分为：

- path：以地址的形式提交数据，根据 id 查询用户的接口就是这种形式传参
- query：Query string 的方式传参，对应无注解和`@RequestParam`注解参数
- header：请求头（header）中的参数
- form：以 Form 表单的形式提交，如上传文件，属于此类
- body：以JSON方式传参

在描述输入参数时，swagger在以下注解：

- `@ApiImplicitParams`: 描述接口的参数集。
- `@ApiImplicitParam`: 描述接口单个参数，与 `@ApiImplicitParams` 组合使用。
- `@ApiParam`：描述单个参数，可选

其中，`@ApiParam`与单个参数一起用，`@ApiImplicitParam`使用在接口描述中，两者的属性基本一致。`@ApiImplicitParam`的属性如下所示：

| **注解属性**  | **描述**                                                     |
| :------------ | :----------------------------------------------------------- |
| paramType     | 查询参数类型，表示参数在哪里，前面已提到，可取值： path，query，header，form，body。 |
| dataType      | 参数的数据类型， 只作为标志说明，并没有实际验证，可按实际类型填写，如String,User |
| name          | 参数名字                                                     |
| value         | 参数意义的描述                                               |
| required      | 是否必填：true/false                                         |
| allowMultiple | 是否有多个，在使用JSON数组对象时使用                         |
| example       | 数据示例                                                     |

以下是对使用JSON对象数组作为参数的接口，如下：

```java
@ApiOperation(value = "添加多个用户", notes = "使用JSON以数组形式添加多个用户")
@ApiImplicitParams({
    @ApiImplicitParam(name = "users", value = "用户JSON数组", required = true, dataType = "User",allowMultiple = true)
})
@PostMapping()
public ResponseResult addUsers(@RequestBody List<User> users) {}
```

此示例中，使用`@RequestBody`注解，传递`List<User>`的JSON数组参数，因此，使用了`allowMultiple`属性，dataType为`User`，而它的`paramType`为`body`。swagger文档如下：

![1573617046002]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-param.jpg )

## 5.4 接口响应数据注解说明

对于使用了`@RestController`注解的控制器，SpringMVC会自动把返回的数据转为json数据。我们在开发过程中，一般都会把返回数据做统一封装，返回状态，信息，实际数据等等。如本示例中，以`ResponseResult`作为统一返回结果，返回的数据使用泛型（若不使用泛型，后续swagger在显示返回结果时对于Model的属性则显示不出来），如下：

```java
@NoArgsConstructor
@AllArgsConstructor
@Data
public class ResponseResult<T> {
	@ApiModelProperty(value="返回状态",example = "true")
	private boolean success;
	@ApiModelProperty(value="错误信息代码")
	private String errCode;
	@ApiModelProperty(value="错误显示信息")
	private String errShowMsg;
	@ApiModelProperty(value="错误信息")
	private String errMsg;
	@ApiModelProperty(value = "返回数据",notes = "若返回错误信息，此数据为null或错误信息对象")
	private T resultData;
}
```

在接口返回值中，若不使用泛型指定具体Model，只返回`ResponseResult`，则只会显示此对象中的属性，而具体的resultData中的内容无法显示，如下：

![1573617582748]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-response.jpg )

因此，对于统一返回，需要指定具体返回的模型，这样，对于实际返回的模型就有对应的描述。如下：

```java
@ApiOperation(value = "获取所有用户", notes = "返回全部用户")
@GetMapping()
public ResponseResult<List<User>> getUsers() {}
```

![1573619148261]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-response2.jpg )

对于响应消息，swagger提供了默认的401,403,404的响应消息，也可以自己针对接口进行指定，有以下注解可使用：

- ` @ApiResponses`：响应消息集
- ` @ApiResponses`：响应消息， 描述一个错误的响应信息 ，与`@ApiResponses`一起使用
- `@ResponseHeader`：响应头设置

# 6.【企业实践】接口认证

对于对外发布的接口，一般都需要进行权限校验，需要登录的用户才可以调用，否则报权限不足。对于api的安全认证，一般有以下几种模式(可参考[swagger文档]( https://swagger.io/docs/specification/authentication/ ):` https://swagger.io/docs/specification/authentication/ `)：

- Basic 认证：使用`Authorization`请求头，用户名和密码使用Base64编码，如：`Authorization: Basic ZGVtbzpwQDU1dzByZA==`
- Bearer 认证：使用`Authorization`请求头，由服务端产生加密字符token，客户端发送请求时把此token的请求头发到服务端作为凭证，token格式是`Authorization: Bearer <token>`
- API Keys认证：使用请求头，参数或cookie，把只能客户端和服务端知道的密钥进行传输。
- OAuth2： 一个开放标准，允许用户授权第三方移动应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方移动应用或分享他们数据的所有内容 。

对于前后端分离的的springboot项目，现在很多采用jwt的认证方式（属于Bearer认证），需要先获取token，然后在调用接口时，添加类似`Authorization: Bearer <token>`的请求头来对接口进行认证。针对这种方式，在Swagger中有以下三种方法来实现：

1. 在接口参数描述中添加`@ApiImplicitParam`，其中参数类型是`ParamType=header`
2. 添加全局接口参数
3. 添加安全认证上下文

## 6.1 添加接口认证参数

针对需要认证的接口，直接使用`@ApiImplicitParam`，其中参数类型是`ParamType=header`即可，参数的描述前面已有介绍。如下：

```java
@ApiImplicitParam(name = "Authorization", value = "token，格式: Bearer &lttoken&gt", required = false, dataType = "String",paramType = "header")
```

这种方式的缺点是，针对每一个接口，都需要添加这个参数描述，而描述都是一样的，重复工作。

## 6.2 添加全局接口参数

swagger配置中，方法`globalOperationParameters`可以设置全局的参数。

```java
//全局header参数
ParameterBuilder tokenPar = new ParameterBuilder();
List<Parameter> pars = new ArrayList<Parameter>();
tokenPar.name("Authorization").description("token令牌")
    .modelRef(new ModelRef("string"))
    .parameterType("header")
    .required(true).build();
pars.add(tokenPar.build());
docket.globalOperationParameters(pars);
```

这种方式缺点也明显，由于设置了全局参数，则所有接口都需要此参数，若有某些接口不需要，则需要进行特殊处理。

## 6.3 添加安全认证上下文

设置认证模式，并使用正则式对需要认证的接口进行筛选，这样swagger界面提供统一的认证界面。如下：

```java
docket.securitySchemes(securitySchemes())
		.securityContexts(securityContexts());
...//省略
private List<ApiKey> securitySchemes() {
		return Lists.newArrayList(
				new ApiKey("Authorization", "Authorization", "header"));
	}
private List<SecurityContext> securityContexts() {
    return Lists.newArrayList(
        SecurityContext.builder()
        .securityReferences(defaultAuth())
        //正则式过滤,此处是所有非login开头的接口都需要认证
        .forPaths(PathSelectors.regex("^(?!login).*$"))
        .build()
    );
}
List<SecurityReference> defaultAuth() {
    AuthorizationScope authorizationScope = new AuthorizationScope("global", "认证权限");
    return Lists.newArrayList(
        new SecurityReference("Authorization", new AuthorizationScope[]{authorizationScope}));
}
```

设置后，输入登录即可，效果如下：

![1573630877692]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger1/swagger-auth.jpg )

# 7. 总结

本篇文章针对swagger的使用和企业实践进行了详细描述，主要介绍了swagger的原理，如何使用springboot与swagger结合创建接口文档，对swagger进行参数化配置及包过滤，swagger注解的详细使用，接口认证方法等，本文中使用的示例代码已放在[github]( https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/hello-swagger-demo )：`https://github.com/mianshenglee/my-example`，有兴趣的同学可以pull代码，结合示例一起学习。

# 参考资料

[swagger官网]( https://swagger.io/ ): ` https://swagger.io/ `

[springfox官网]( http://springfox.github.io/springfox/ ): `http://springfox.github.io/springfox/`

[在 Spring Boot 项目中使用 Swagger 文档](https://www.ibm.com/developerworks/cn/java/j-using-swagger-in-a-spring-boot-project/index.html )：`https://www.ibm.com/developerworks/cn/java/j-using-swagger-in-a-spring-boot-project/index.html`

[SpringBoot2 配置swagger2并统一加入认证参数]( https://www.jianshu.com/p/7a24d202b395 )：`https://www.jianshu.com/p/7a24d202b395`

[一篇文章带你搞懂 Swagger 与 SpringBoot 整合]( https://mp.weixin.qq.com/s/1KuBFfMugJ4pf3cSEFfXfw ): `https://mp.weixin.qq.com/s/1KuBFfMugJ4pf3cSEFfXfw`

# 往期文章

- [查阅了十几篇学习资源后，我总结了这份AI学习路径]( https://mianshenglee.github.io/2019/09/06/ai-study-path.html )
- [java应用监测(8)-阿里诊断工具arthas]( https://mianshenglee.github.io/2019/08/31/java-monitor-8.html )
- [java应用监测(7)-在线动态诊断神器BTrace](https://mianshenglee.github.io/2019/08/30/java-monitor-7.html)
- [java应用监测(6)-第三方内存分析工具MAT](https://mianshenglee.github.io/2019/08/29/java-monitor-6.html)
- [java应用监测(5)-可视化监测工具](https://mianshenglee.github.io/2019/08/27/java-monitor-5.html)

# 我的公众号

关注我的公众号，获取更多技术文章。

![公众号](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)