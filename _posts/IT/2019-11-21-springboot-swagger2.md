---
layout: post
title: springboot+swagger接口文档企业实践（下）
category: IT
tags: springboot swagger doc
keywords: 
description: 
---

tags:  `springboot` `swagger` `doc`

---


> 一句话概括：基于springboot+swagger实现接口文档显示后，本文将给出在企业实践的进阶需求，包括接口按需过滤，前端mock数据，文档离线导出。

# 1.引言

在上一篇文章《[springboot+swagger接口文档企业实践（上）]( https://mianshenglee.github.io/2019/11/13/springboot-swagger1.html )》已对使用springboot+swagger的接口文档构建及配置进行了介绍，可以实时显示接口的输入输出，还可以调用接口调试。解决了后端开发人员写接口文档的难处。但在企业实践中，还有一些问题需要解决，如以下几种：

- 发布指定的接口：不希望把全部的接口都对外发布提供，只需要发布指定的接口。
- 发布指定版本接口：每一次版本迭代，只需要对变更和新增的接口进行说明，而不是每次都输出全部的接口（每次都全量发布接口，前端人员还得与后端沟通哪些是变更和新增接口，无疑会增加工作量）
- mock接口数据：有了接口文档，前后端可以并行开发，此时前端需要模拟接口的返回数据来显示效果，测试内容等。
- 离线导出接口文档：当前的接口文档是通过页面访问的，但有些企业（特别注重文档交付的企业）还是有一些特殊需求，需要离线的接口文档。

针对以上的情况，本文提供相应的解决方法，主要包含以下内容：

1. 接口过滤：通过包过滤、类注解过滤、方法注解过滤、分组过滤等方式，实现按需发布指定接口的功能。
2. 使用easy-mock+swagger实现mock数据。
3. 使用maven插件实现接口文档的离线导出。

本文配套的[示例工程地址]( https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/advance-swagger-demo )：` https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/advance-swagger-demo `，读者可fork或pull下来，结合学习。

# 2. swagger接口过滤

对于swagger的接口文档，在实践开发中，一般只需要发布指定的接口，按需发布即可，针对版本迭代，只需要对变更和新增的接口进行说明，而不是每次都输出全部的接口。这些需求都属于swagger的接口过滤功能。在springboot的swagger配置类(`Swagger2Config.java`)中，Docket提供了`apis()`和`paths()`两个方法用于接口过滤，下面详细说明一下。

## 2.1 按包过滤(package)

一般接口都是写在控制器(controller)中，而controller一般都统一放在一个package中，这样，可以通过包过滤显示指定包的接口。通过在配置文件`Swagger2Config.java`中使用`apis()`函数进行过滤，把需要显示的接口使用函数进行返回，注意，此处的参数`enableClassFilter`，`enableMethodFilter`和`groupsFilters`分别对应后面的过滤方法，定义如下：

```java
private List<Predicate<RequestHandler>> apisFilter(boolean enableClassFilter,boolean enableMethodFilter, String[]groupsFilters) {
    List<Predicate<RequestHandler>> apis = new ArrayList<>();
    String basePackageStr = swaggerInfo.getBasePackage();
    // 1.包过滤
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

此处代码作用是按配置项`swagger.basePackage`的包（多个包用`;`分隔），从接口所在类中获取包名，若包名是在配置的包内，则返回，并把匹配的内容使用`List`返回。然后在`apis()`调用时对返回值进行and操作`Predicates.and(apisFilter())`，这样，返回的内容就只是指定的包的接口描述。

## 2.2 按类注解过滤

接口一般是在Controller类中，对于springmvc的controller，都会使用`@Controller`进行注解，甚至前后端分离一般都是使用`@RestController`，因此，如果我们想只显示使用`@RestController`注解的类的接口，则可以对此进行过滤。在前面的`apisFilter()`方法中。使用`isAnnotationPresent`可判断是否有某注解，如下所示

```java
// 2.过滤被RestController注解的类
if(enableClassFilter){
    Predicate<RequestHandler> predicate = input -> {
        Class<?> declaringClass = input.declaringClass();
        return declaringClass.isAnnotationPresent(RestController.class);
    };
    apis.add(predicate);
} 
```

## 2.3 按方法注解过滤

按类注解是把整个类过滤掉，粒度较大，如果只想按方法过滤，可以使用swagger的`@ApiIgnore`注解对接口进行过滤，有此注解则不显示。也可以通过判断是否存在某个指定方法注解来过滤。swagger的接口描述一般都用`@ApiOperation`，因此，可以通过判断接口是否存在此注解，如果没有则不显示。如下：

```java
// 3.过滤被ApiOperation注解的方法
if(enableMethodFilter){
    apis.add(input -> input.isAnnotatedWith(ApiOperation.class));
}
```

## 2.4 按分组过滤

swagger的接口文档如果没有指定groupName，则会默认以`default`作为分组名，对应的接口文档是`v2/api-docs?group=default`，它会按`apis`过滤后的全部接口显示出来。在迭代版本时，有个显示的需求是只需要显示当前版本变更和新增的接口。其实也是使用注解过滤的方式，结合groupName进行设置即可。具体如下：

### 2.4.1 定义注解`ApiVersion`

此注解需自定义，用于指定版本号，注意可以是多个版本（多版本兼容的情况）。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ApiVersion {
    /**
     * 接口版本号(对应swagger中的group)
     * @return String[]
     */
    String[] group();
}
```

当开发新的版本，在变更和新增的接口中添加此注解，并把版本号写到对应的group中即可，如

```java
@ApiVersion(group = {"v1.0.0"})
```

### 2.4.2 添加`ApiVersion`过滤

在`apisFilter()`函数中，添加以下过滤代码：

```java
// 4.过滤组
if(groupsFilters !=null && groupsFilters.length >0){
    Predicate<RequestHandler> predicate = input -> {
        ApiVersion apiVersion = input.getHandlerMethod().getMethodAnnotation(ApiVersion.class);
        return apiVersion!=null && ArrayUtil.containsAny(apiVersion.group(),groupsFilters);
    };
    apis.add(predicate);
}
```

从接口方法中获取`ApiVersion`注解，并获取它的`group`参数，通过与指定的分组进行比较，存在即显示，否则不显示。

### 2.4.3 新建指定版本号分组的Docket

添加新的一个Docket Bean，指定`groupName`为当前需要显示的版本号，并输入需要过滤的分组数组。如下：

```java
@Bean
public Docket v100Api() {
    Docket docket = new Docket(DocumentationType.SWAGGER_2)
        .groupName(ApiVersionConstant.VERVION_100)
        ...//略
    ApiSelectorBuilder builder = docket.select();
    //指定需要过滤的版本号
    builder = builder.apis(Predicates.and(apisFilter(false,true,new String[]{ApiVersionConstant.VERVION_100})));
    ...//略
    return builder.build();
}
```

经过上面的处理，显示的接口界面有两个分组，一个是default，一个是指定的版本号(v1.0.0)，指定版本显示的接口即是当前版本变更或新增的接口描述，如下：

![按版本号分组]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/swagger-group.png )

# 3. 接口mock数据

有了接口文档，前后端可以并行开发，此时前端需要模拟接口的返回数据来显示效果，测试内容。虽然swagger提供`example`属性，可以在返回结果中提供示例，但对于前端而言，单一的示例不足以满足显示和测试的需求。需要对数据按接口情况进行mock。easy-mock是一个很好的选择，它可以连接swagger，自动生成mock数据，也可以自定义mock规则。下面对easy-mock+swagger的使用进行描述。

## 3.1 easy-mock安装

根据[easy-mock的官网]( https://www.easy-mock.com/ )介绍（官网经常挂掉，建议直接使用它的github私有部署）， Easy Mock 是一个可视化，并且能快速生成模拟数据的持久化服务。 可以使用它的在线服务，也可以私有部署。它是开源项目，[github地址]( https://github.com/easy-mock/easy-mock )：` https://github.com/easy-mock/easy-mock `，具有以下特征：

![easy-mock特征]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/easy-mock.png )

它的[官方文档]( https://easy-mock.com/docs )和[github文档]( https://github.com/easy-mock/easy-mock/blob/dev/README.zh-CN.md )中，已经对easymock的安装和使用进行详细描述，读者可参考官方文档，此处只列出安装的关键点和需要注意的地方（本文使用的easy-mock及相关工具是在centos7上安装的）。

1. 安装easy-mock前，需先安装了 [Node.js](https://nodejs.org/)（**v8.x**）、 [MongoDB](https://www.mongodb.com/)（**>= v3.4**）、 [Redis](https://redis.io/)（**>= v4.0**） ，且已正常运行。关于这几个软件的安装与运行，请读者自行查看官网教程。
2.  安装 easy-mock。

```shell
$ git clone https://github.com/easy-mock/easy-mock.git
$ cd easy-mock && npm install
```

3. 更改 easy-mock\config 文件夹下的配置文件 default.json，注意修改host为"0.0.0.0"，修改连接mongodb的地址以及redis的host，端口等信息。
4.  启动 easy-mock，这种不是在后台运行的方式，直接`ctrl+c`可关闭。

```shell
$ npm run dev
# 启动后访问 http://127.0.0.1:7300
```

5. 使用pm2进行后台启动， 启动成功后，在命令行中输入 `netstat -ntlp` 查看正在使用的端口（如mongodb的27017,redis的6379,easy-mock的7300等）

```
$ [sudo] npm install pm2 -g
$ pm2 start app.js
```

easy-mock启动后，通过浏览器可以访问， 输入用户名和密码（如果用户不存在会自动注册）。

![easy-mock界面](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/easy-mock2.png )

## 3.2 easy-mock + swagger实现mock数据

easy-mock是使用项目来进行接口管理，可以创建个人项目，也可以加入到其它人创建的项目，项目即是需要mock的接口。

### 3.2.1 创建项目并添加swagger地址

在创建项目时，可以设置swagger的接口文档地址，以此导入swagger的接口进行管理与模拟。前面提到，使用分组过滤可以按版本号来显示接口，因此，创建项目时，可以使用版本号作为项目名称，一个版本对应一个项目。这样，前端在mock数据时就可以针对当前版本进行处理。如下为示例项目中的v1.0.0版本(填写的swagger接口文档地址为`v2/api-docs?group=v1.0.0`)：

![添加项目]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/easy-mock-create.png )



填写swagger地址后，会自动导入对应的接口，与前端看到`swaggger-ui.html`的接口一致，执行测试时，也会按swgger的返回数据类型或example进行返回。

![详细]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/easy-mock-detail.png )

### 3.2.2 自定义模拟数据

模拟数据使用的是 [Mock.js](http://mockjs.com/) ，在其官网中有相应的文档、示例和代码。读者可以上去详细阅读。简单来说，mockjs提供了对`String`，`Number`，` Boolean `，` Object `，` Array `等数据的模拟规则，只需按规则编写，即可生成随机数据，达到模拟数据效果。生成规则有 7 种格式：

1. `'name|min-max': value`
2. `'name|count': value`
3. `'name|min-max.dmin-dmax': value`
4. `'name|min-max.dcount': value`
5. `'name|count.dmin-dmax': value`
6. `'name|count.dcount': value`
7. `'name|+step': value`

使用了自定义的模拟数据后，需要注意以下几点：

- 使用了自定义的模拟数据规则后，前端在使用此接口进行模拟数据时，原来在swagger中定义的示例和返回值就会被自定义的数据覆盖。
- 当后端接口有变化，需要前端在easy-mock中重新同步新的swagger接口，同步前需要备份原来的自定义数据规则，否则，重新同步后，原来自定义的数据会swagger的接口定义覆盖。因此，最好是先备份，然后针对有改动的地方进行规则修改即可。

# 4.接口文档离线导出

一般在开发过程中使用swagger文档，直接使用浏览器访问`swagger-ui.html`页面即可满足要求，若有需求是需要导出离线接口文档，总体可以按以下思路进行：

1. 使用`swagger`的`v2/api-docs`的url地址导出json文档（不导出也可以直接使用此url作为输入文档的地址）
2. 使用`swagger2markup`工具将json文档转为asciidoc格式文档。
3. 使用`asciidoctor`工具将asciidoc文件转html或pdf文件。

## 4.1 导出swagger的json文档

在浏览器中访问接口文档页面的地址是`/swagger-ui.html`，而swagger的规范文档的地址是`/v2/api-docs`，`ctrl+s`把此文件保存为json文件，如`api-docs.json`。在项目的根目录添加一个`docs`目录，用于存放离线文档相关内容，如下所示，分别建立对应文档格式的目录，并把`api-docs.json`放在swagger目录下：

```txt
├─asciidoc 
├─html 
├─pdf 
└─swagger
  └─api-docs.json 
```

## 4.2 导出asciidoc文档

有了`api-docs.json`文档，即可使用[swagger2markup](https://github.com/Swagger2Markup/swagger2markup)导出asciidoc文档，Swagger2Markup是Github上的一个开源项目。该项目主要用来将Swagger自动生成的文档转换成几种流行的格式以便于静态部署和使用，比如：AsciiDoc、Markdown、Confluence。使用swagger2markup导出asciidoc文档有两种方式：

- 引入swagger2markup的jar包，通过编码方式生成对应格式的文档输出。
- 使用swagger2markup的maven插件，以mvn命令生成对应格式文档输出。

第一种方式示例代码中有提供`SwaggerExportTest.java`，其中编写了相应的测试代码用于生成文档，请读者自行阅读。本文主要使用第二种方式，即maven插件进行asciidoc文档输出。

### 4.2.1 设置文档输出相关插件版本

以下插件版本在示例可正常运行，更高的版本有可能会出现不兼容的情况，因此请读者按本文设置的版本进行处理。文档输出的路径以前面指定的相关docs目录为准（`${basedir}`是项目的根目录）。

```xml
<!-- 文档输出插件版本 -->
<swagger.plugin.version>3.1.8</swagger.plugin.version>
<swagger2markup.version>1.3.1</swagger2markup.version>
<swagger2markup.plugin.version>1.3.3</swagger2markup.plugin.version>
<asciidoctor.plugin.version>1.5.7</asciidoctor.plugin.version>
<!-- 文档输出路径 -->
<docs.path>${basedir}/docs</docs.path>
<docs.asciidoc.path>${docs.path}/asciidoc</docs.asciidoc.path>
<docs.html.path>${docs.path}/html</docs.html.path>
<docs.pdf.path>${docs.path}/pdf</docs.pdf.path>
<docs.swagger.json.path>${docs.path}/swagger/api-docs.json</docs.swagger.json.path>

```

### 4.2.2 添加`swagger2markup-maven-plugin`插件

在`build/plugins`元素下，添加以下元素：

```xml
<!-- 1.swagger.json文件转asciidoc文件-->
<plugin>
    <groupId>io.github.swagger2markup</groupId>
    <artifactId>swagger2markup-maven-plugin</artifactId>
    <version>${swagger2markup.plugin.version}</version>
    <configuration>
        <!-- 访问url -->
        <!--<swaggerInput>http://localhost:8080/swaggerdemo/v2/api-docs?group=default</swaggerInput>-->
        <!-- 访问json文件-->
        <swaggerInput>${docs.swagger.json.path}</swaggerInput>
        <!-- 生成多个文档输出路径 -->
        <!--<outputDir>${docs.asciidoc.path}</outputDir>-->
        <!-- 生成单个文档输出路径 -->
        <outputFile>${docs.asciidoc.path}/all</outputFile>
        <config>
            <swagger2markup.pathsGroupedBy>TAGS</swagger2markup.pathsGroupedBy>
            <!-- 选择：ASCIIDOC/MARKDOWN/CONFLUENCE_MARKUP-->
            <swagger2markup.markupLanguage>ASCIIDOC</swagger2markup.markupLanguage>
        </config>
    </configuration>
</plugin>

```

说明：

- swaggerInput元素可以是导出的json文件路径，也可以使用访问url地址(`v2/api-docs`)，效果是一样的。但若使用url，则需要确保先把应用启动，能正常访问url。
- outputFile元素是指把全部内容输出到一个文件中，若使用outputDir，则会把各种类型的内容，分开成多个文件输出。
- swagger2markup.markupLanguage元素可以选择导出`ASCIIDOC/MARKDOWN/CONFLUENCE_MARKUP`三种格式的文档，此处使用`ASCIIDOC`即可。

### 4.2.3  使用命令输出文档

添加完此插件后，使用`mvn swagger2markup:convertSwagger2markup`命令即可以导出文件到指定的目录。如本示例中的输出是`docs/asciidoc/all.adoc`

## 4.3 导出html/pdf文档

导出asciidoc文档后，使用asciidoctor插件对其转换为html和pdf文档输出。

### 4.3.1 添加asciidoctor插件

如下所示，通过添加[asciidoctor-maven-plugin]( https://asciidoctor.org/docs/asciidoctor-maven-plugin/ )，并对它进行配置：

```xml
<!-- 2.asciidoc文件转html/pdf文件-->
<plugin>
    <groupId>org.asciidoctor</groupId>
    <artifactId>asciidoctor-maven-plugin</artifactId>
    <version>${asciidoctor.plugin.version}</version>
    <!-- 转换pdf使用的依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.asciidoctor</groupId>
            <artifactId>asciidoctorj-pdf</artifactId>
            <version>1.5.0-alpha.16</version>
        </dependency>
        <dependency>
            <groupId>org.jruby</groupId>
            <artifactId>jruby-complete</artifactId>
            <version>9.2.8.0</version>
        </dependency>
    </dependencies>
    <configuration>
        <sourceDirectory>${docs.asciidoc.path}</sourceDirectory>
        <doctype>book</doctype>
        <sourceHighlighter>coderay</sourceHighlighter>
        <headerFooter>true</headerFooter>
        <attributes>
            <!-- 菜单栏在左边 -->
            <toc>left</toc>
            <!-- 三级目录 -->
            <toclevels>3</toclevels>
            <!-- 数字序号 -->
            <sectnums>true</sectnums>
        </attributes>
    </configuration>
    <!-- 生成html和pdf两种格式 -->
    <executions>
        <execution>
            <id>output-html</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>process-asciidoc</goal>
            </goals>
            <configuration>
                <outputDirectory>${docs.html.path}</outputDirectory>
                <backend>html</backend>
            </configuration>
        </execution>
        <execution>
            <id>output-pdf</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>process-asciidoc</goal>
            </goals>
            <configuration>
                <outputDirectory>${docs.pdf.path}</outputDirectory>
                <backend>pdf</backend>
                <!-- 处理中文字符问题 -->
                <attributes>
                    <pdf-fontsdir>${docs.pdf.path}/fonts</pdf-fontsdir>
                    <pdf-stylesdir>${docs.pdf.path}/themes</pdf-stylesdir>
                    <pdf-style>cn</pdf-style>
                </attributes>
            </configuration>
        </execution>
    </executions>
</plugin>
```

此配置相对较长，其实主要分为三部分：

1. 转换pdf使用的依赖，即`<dependencies>`元素，直接使用即可。
2. 通用文档配置`<configuration>`元素，其中需要注意的是`<sourceDirectory>`，此处需要配置上一个步骤生成的`asiidoc`文件目录路径。
3. `<executions>`元素：由于需要生成html和pdf两种格式的文档，因此分别使用`executions`来实现。其中html相对简单，配置`outputDirectory`和`backend`元素指定输出目录路径和文件格式html即可，对应的pdf也一样配置。

对于pdf文档的转换，需要解决中文字体缺失的问题。

- 若不配置对应的中文字符支持（即配置中的`<attributes>`元素），则会出现中文乱码或文字缺失的情况，如下图，缺失了`户`，`对`这几个字：

![文字缺失]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/doc-pdf1.png )

- pdf中文字符支持，在github中，已有对[asciidoctor-pdf中文字体]( https://github.com/chloerei/asciidoctor-pdf-cjk-kai_gen_gothic/releases )的支持，在下载页面下载`KaiGenGothicCN`开头和`RobotoMono`开头的ttf字体，同时下载`cn-theme.yml`文件，分别放到`docs/pdf`目录下：

```txt
├─pdf 
│ ├─fonts 
│ │ ├─KaiGenGothicCN-Bold-Italic.ttf 
│ │ ├─KaiGenGothicCN-Bold.ttf 
│ │ ├─KaiGenGothicCN-Regular-Italic.ttf 
│ │ ├─KaiGenGothicCN-Regular.ttf 
│ │ ├─RobotoMono-Bold.ttf 
│ │ ├─RobotoMono-BoldItalic.ttf 
│ │ ├─RobotoMono-Italic.ttf 
│ │ └─RobotoMono-Regular.ttf 
│ └─themes 
│   └─cn-theme.yml 
```

- 使用`<attributes>`元素设置对应的`pdf-fontsdir`,`pdf-stylesdir`及`pdf-style`，指定下载好的fonts目录和themes目录。

### 4.3.2 使用命令输出文档

使用命令`mvn generate-resources`，即可生成对应的html和pdf文档到对应的目录，分别是`docs/html/all.html`及`docs/html/all.pdf`。效果如下图所示：

- html文档

![html]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/doc-html.png )

- pdf文档（缺失文字问题已解决）

![pdf]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/swagger/swagger2/doc-pdf2.png )

# 5. 总结

本篇文章针对接口过滤显示，前端mock数据和离线导出文档等问题，提供相应的解决方法，包括接口过滤（包过滤、类注解过滤、方法注解过滤、分组过滤等方式），实现按需发布指定接口的功能；使用easy-mock+swagger实现mock数据；使用maven插件实现接口文档的离线导出。希望对有需要的同学有所帮助。本文配套的[示例工程地址]( https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/advance-swagger-demo )：` https://github.com/mianshenglee/my-example/tree/master/springboot-swagger-demo/advance-swagger-demo `，读者可fork或pull下来，结合学习。

# 相关阅读

[springboot+swagger接口文档企业实践（上）]( https://mianshenglee.github.io/2019/11/13/springboot-swagger1.html )

# 参考资料

- [swagger2markup-maven-plugin]( https://github.com/Swagger2Markup/swagger2markup-maven-plugin )： ` https://github.com/Swagger2Markup/swagger2markup-maven-plugin `
- [asciidoctor-maven-plugin]( https://asciidoctor.org/docs/asciidoctor-maven-plugin/ )：` https://asciidoctor.org/docs/asciidoctor-maven-plugin/ `
- [asciidoctor-pdf中文字体]( https://github.com/chloerei/asciidoctor-pdf-cjk-kai_gen_gothic/releases ):` https://github.com/chloerei/asciidoctor-pdf-cjk-kai_gen_gothic/releases `
- [使用Swagger2Markup、asciidoctor-maven-plugin和asciidoctorj-pdf插件生成PDF格式的API文档中文问题解决]( https://blog.csdn.net/lihuaijun/article/details/79727863 ):` https://blog.csdn.net/lihuaijun/article/details/79727863 `


**关注我的公众号，获取更多技术记录**：

![mason技术记录](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)