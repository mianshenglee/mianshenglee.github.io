---
layout: post
title: 利用 MinIO 轻松搭建静态资源服务
category: IT
tags: minio static-resource-server
keywords: 
description: 
---

> 如果想搭建一个优雅、简单、功能完备的静态资源服务，那就用MinIO吧。

# 1 引言

开发 Web 项目过程中，经常需要处理静态资源（如图片、视频、音频，js库 ，css 库等），一般来说，若项目中需要用到这些资源文件，我们常用的有以下几种方法：

- 本地存储：即在项目工程中的 static 目录下，建立 js/css/icon/font/image/lib/audio/video 等目录，分别存放对应格式的资源文件。使用时，在 html 文件中使用相对位置引用进行。
- 使用代理搭建静态资源服务：即把资源存放于某一文件目录，使用代理服务器（如 nginx ，apache等）对目录进行映射，构建成资源服务。使用时，在 html 文件中使用代理服务的 url 地址进行引用。
- 使用第三方工具搭建静态资源服务：使用第三方开源的文件存储或对象存储工具，或者自己写个程序实现可以获取文件的接口。使用时，使用对应的 url 地址或接口地址。
- 使用在线静态资源服务：如阿里云、CDN等服务。

对于本地存储，缺点就很明显，资源与代码文件混合一起，没有必要，而且不方便扩展。对于本地内部部署的应用，显然是自己搭建静态资源服务比较稳妥。对于使用代理服务和第三方工具，相比起来，代理服务仅做映射，虽然可用，但功能单一，仅做映射，没有其它管理功能，也不方便扩展。使用第三方文件或对象存储工具，可以对文件进行管理、也能考虑高扩展，高性能、高可用等因素，因此是很好的选择，其中，MinIO 就是这样一款好用的对象存储工具，简单，快捷而且功能完备。

本文则是通过对 MinIO 的安装、配置与使用，构建静态资源服务，从而把图片、视频、音频，第三方 js 库等资源独立部署，访问；还会对 MinIO 提供的 Java API 进行使用简单介绍，以便于进一步开发。

# 2 MinIO 简介

按 [MinIO](https://min.io/) 官方介绍，MinIO 是高性能的对象存储（块存储、文件存储和对象存储的区别，可参考[架构师都知道的分布式对象存储解决方案](https://juejin.im/post/5cdc16e251882568651553f9)），兼容 Amazon S3 接口，充分考虑开发人员的需求和体验；支持分布式存储，具备高扩展性、高可用性；部署简单但功能丰富。官方的文档也很详细。它有多种不同的部署模式（单机部署，分布式部署）。为什么说 MinIO 简单易用，原因就在于它的启动、运行和配置都很简单。可以通过 docker 方式进行安装运行，也可以下载二进制文件，然后使用脚本运行。

本文以最简单的方式进行讲解，在 linux 机器中，单机部署，运行二进制文件。

# 3 MinIO 运行与静态资源使用

## 3.1 MinIO 获取

[MinIO 开发文档](https://docs.min.io)中，下载地址如下：

- linux: `https://dl.min.io/server/minio/release/linux-amd64/minio`
- windows: `https://dl.min.io/server/minio/release/windows-amd64/minio.exe`

本文在 linux 中运行。

## 3.2 MinIO 启动与运行

### 3.2.1 前台简单启动

把下载的 minio 文件存放到某个目录作为运行目录（如 /opt/minio），新建一个目录（如/opt/minio-data）作为 minio 数据存储位置，即可启动，如下脚本：

``` shell
cd /opt/minio
chmod +x minio
./minio server /opt/minio-data
```

启动后会输出访问地址 endpoint 和对应的 access_key 和 secret_key，使用浏览器访问 endpoint 地址，若可以访问，则表示 minio 已经安装成功。不过这样启动会有几个缺点：

- 不是后台运行，`ctrl+c` 就会结束进程，服务就停止
- 没有自定义访问的用户名密码
- 没有指定访问IP和端口
- 日志没有保留到文件

针对这些问题，建议使用下面的方式进行启动运行。

### 3.2.2 后台指定参数运行

使用 `nohup` 在后台运行程序，同时指定密码参数，访问地址参数和日志输出，如下所示。

```shell
MINIO_ACCESS_KEY=minio MINIO_SECRET_KEY=minio123 nohup /opt/minio/minio  server --address "${MINIO_HOST}:${MINIO_PORT}" /opt/minio-data  > /opt/minio/minio.log 2>&1 &
```

> `MINIO_ACCESS_KEY` 及 `MINIO_SECRET_KEY` 是访问密码
>
> `${MINIO_HOST}:${MINIO_PORT}` 分别是访问的 host 和端口，请按实际情况修改。

这样，通过浏览器访问地址 `${MINIO_HOST}:${MINIO_PORT}` ，使用指定的 `MINIO_ACCESS_KEY` 及 `MINIO_SECRET_KEY`  登录即可。

### 3.2.3 创建 bucket 并指定访问策略

在浏览器中登录到 MinIO 存储系统，点击右下角创建 bucket 来存储对象，分别创建对应的 bucket 以存放静态资源：image，video，audio。这样，就可以按资源类型在对应的 bucket 中进行文件上传了，上传后可以把文件分享，其它地方可以通过分享的 url 获取资源，如下图所示。

![分享文件](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200331-minio/share-object.png)

MinIO 默认的策略是分享地址的有效时间最多是7天，要突破这种限制，可以在 bucket 中进行策略设置。点击对应的 bucket ，`edit policy` 添加策略 `*.*`，`Read Only`，如下：

![edit policy](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200331-minio/edit-policy.png)

如此就放开了访问，没有时间限制，同时只需要按`http://${MINIO_HOST}:${MINIO_PORT}/${bucketName}/${fileName}` 则可直接访问资源（不需要进行分享操作）。

> 关于 MinIO 目录的误区
>
> - 其实对于对象存储来说，其实不区分文件还是目录，所有文件和目录都是对象，即 image/temp/xxx.jpg 和 image/temp/ 都是对象。它跟操作系统的文件系统的树状结构有本质区别。
> - 上传文件时，objectName 可以是 `/temp/xxx.jpg`，可以认为系统自动创建了temp目录。
> - MinIO 不会提供像删除目录，同时删除此目录下所有文件的操作（即 `rm -rf image/temp`），因此要想把目录 image/temp 删除，则需要先把以 image/temp 为前缀的所有文件删除。
> - 查询多个文件时，可以使用前缀匹配方式获取，见 API 文档 `listObjects(bucketName, prefix, recursive)`

## 3.3 在 html 文件中引用静态资源

通过上面的设置与运行，MinIO 作为静态资源服务器已经完成，可以写个 html 来引用 MinIO 中的静态资源。如下是测试的 html 里面的图片、视频、音频均使用 MinIO 的资源地址。

``` html
<div class="img-list">
        <img src="http://${MINIO_HOST}:${MINIO_PORT}/image/test.jpg" alt="图片">
    </div>
    <div class="audio-list">
        <audio src="http://${MINIO_HOST}:${MINIO_PORT}/audio/test.mp3"
        controls="controls"></audio>
    </div>
    <div class="video-list">
        <video src="http://${MINIO_HOST}:${MINIO_PORT}/video/test.mp4" controls="controls"></video>
    </div>
```

可发现资源是可以正常加载访问的。

# 4 Java 客户端 API 操作

MinIO 对开发者是非常友好的，提供了各种语言的 API 操作接口，具体可以参考 [MinIO开发文档](https://docs.min.io/)。下面以 Java 为例做一下测试。

## 4.1 添加依赖

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>6.0.13</version>
</dependency>
```



## 4.2 使用 Java API 进行文件操作

建立 MinIO 的操作客户端 `minioClient = new MinioClient(endpoint, accessKey, secretKey);`，参数中`endpoint` 是 MinIO 的访问地址，后面两对应启动时设置的密码。

### 4.2.1 上传文件

```java
/**
 * 上传文件
 * @param minioClient 操作客户端
 * @param bucketName 上传的bucket名称 
 * @param objectName 上传后存储在bucket中的文件名
 * @param filePath 上传的本地文件路径
 */
public void uploadFile(MinioClient minioClient, String bucketName, String objectName, String filePath) throws XmlPullParserException, NoSuchAlgorithmException, InvalidKeyException, IOException {
    try {
        // 若不存在bucket，则新建
        boolean isExist = minioClient.bucketExists(bucketName);
        if (!isExist) {
            minioClient.makeBucket(bucketName);
        }
        // 使用 putObject 上传文件
        minioClient.putObject(bucketName, objectName, filePath, null, null, null, null);
    } catch (MinioException e) {
        System.out.println("Error occurred: " + e);
    }
}
```



### 4.2.2 下载文件

```java
/**
 * 下载文件
 *
 * @param minioClient 操作客户端
 * @param bucketName 上传的bucket名称 
 * @param objectName 上传后存储在bucket中的文件名
 * @param downloadPath 下载文件保存路径
 */
public void downloadFile(MinioClient minioClient, String bucketName, String objectName, String downloadPath) throws XmlPullParserException, NoSuchAlgorithmException, InvalidKeyException, IOException {
    File file = new File(downloadPath);
    try (OutputStream out = new FileOutputStream(file)) {
        InputStream inputStream = minioClient.getObject(bucketName, objectName);
        byte[] tempbytes = new byte[1024];
        int byteread = 0;
        while ((byteread = inputStream.read(tempbytes)) != -1) {
            out.write(tempbytes, 0, byteread);
        }
    } catch (MinioException e) {
        System.out.println("Error occurred: " + e);
    }
}
```



### 4.2.3 删除文件

删除文件简单，使用`removeObject`即可。

```java
minioClient.removeObject(bucketName, objectName);
```



### 4.2.4 列出文件

```java
/**
  * 罗列文件
  * @param minioClient
  * @param bucketName
  */
public void listFile(MinioClient minioClient, String bucketName) throws XmlPullParserException, NoSuchAlgorithmException, InvalidKeyException, IOException {
    try {
        Iterable<Result<Item>> results = minioClient.listObjects(bucketName);
        Iterator<Result<Item>> iterator = results.iterator();
        while (iterator.hasNext()) {
            Item item = iterator.next().get();
            System.out.println(item.objectName() + ", " + item.objectSize() + "B");
        }
    } catch (MinioException e) {
        System.out.println("Error occurred: " + e);
    }
}
```



# 5 总结

由于有对静态资源进行独立访问的需求，进行动静分离，通过使用 MinIO ，可以快速简单的实现静态资源服务器，以供访问。本文通过对 MinIO 的下载、部署、启动、运行、配置等描述，并以 html 引用静态资源文件为例，讲解 MinIO 的使用，并提供 Java API 的简单使用。希望对大家有帮助。

# 资源下载

本文中使用了 nohup 对 MinIO 进行启动，但命令太长，一般我们都会写成脚本，以实现启动、关闭及状态查询，因此，我把脚本写成完整的 sh 文件，以供大家使用。另外，MinIO Java API 的测试，本示例使用的是 Spring Boot 项目，以单元测试的方式进行。

sh脚本文件及 Spring Boot 工程一起放在我的 [minio github 示例](https://github.com/mianshenglee/my-example/tree/master/minio-simple-demo) 中（脚本`minio-serviced.sh` 在 scripts 目录下），有需要的可下载参考。

> 脚本使用方法：
>
> - 根据实际情况修改sh脚本中的参数
> - 修改执行权限：chmod +x minio-serviced.sh
> - 按参数启动/关闭/重启/运行状态：./minio-serviced.sh start/stop/restart/status

# 参考资料

- [MinIO官网](https://min.io/): `https://min.io/`
- [MinIO开发文档](https://docs.min.io/): `https://docs.min.io/`
- [基于 Go 开源项目 MIMIO 的对象存储方案在探探的实践](https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487119&idx=1&sn=6e09abb32392e015911be3a1d7f066e5&source=41)：`https://mp.weixin.qq.com/s/MzA4ODg0NDkzOA==&mid=2247487119&idx=1&sn=6e09abb32392e015911be3a1d7f066e5&source=41`
- [架构师都知道的分布式对象存储解决方案](https://juejin.im/post/5cdc16e251882568651553f9)：`https://juejin.im/post/5cdc16e251882568651553f9`
- [使用minio搭建高性能对象存储](https://tonybai.com/2020/03/16/build-high-performance-object-storage-with-minio-part1-prototype)：`https://tonybai.com/2020/03/16/build-high-performance-object-storage-with-minio-part1-prototype`


# 往期文章

- [搞定SpringBoot多数据源(3)：参数化变更源](https://mp.weixin.qq.com/s/ZzzPJZAhPiGCQjN3RNZJ3Q)
- [搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)
- [搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)
- [java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)
- [2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)
- [springboot+apache前后端分离部署https](https://mp.weixin.qq.com/s/hiJdsjdDC07axk-_sAkyVQ)
- [springboot+logback 日志输出企业实践（下）](https://mp.weixin.qq.com/s/ha4LaR-E1gDxfUZ11neI-Q)
- [springboot+logback 日志输出企业实践（上）](https://mp.weixin.qq.com/s/Ti5i9vv9S1j4za5q11RWeA)


我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![mason](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)