---
layout: post
title: MinIO 的分布式部署
category: IT
tags: minio distributed storage
keywords: 
description: 
---

> 高可用分布式对象存储，MinIO 轻松实现。

# 1 前言

[上一篇文章]( https://mp.weixin.qq.com/s/rzaqPmpTOUJJsKISr6eF9Q)介绍了使用对象存储工具 MinIO 搭建一个优雅、简单、功能完备的静态资源服务，可见其操作简单，功能完备。但由于是单节点部署，难免会出现单点故障，无法做到服务的高可用。MinIO 已经提供了分布式部署的解决方案，实现高可靠、高可用的资源存储，同样的操作简单，功能完备。本文将对 MinIO 的分布式部署进行描述，主要分以下几个方面：

- 分布式存储的可靠性
- MinIO 的分布式的存储机制

- 分布式部署实践



# 2 分布式存储可靠性常用方法

分布式存储，很关键的点在于数据的可靠性，即保证数据的完整，不丢失，不损坏。只有在可靠性实现的前提下，才有了追求一致性、高可用、高性能的基础。而对于在存储领域，一般对于保证数据可靠性的方法主要有两类，一类是冗余法，一类是校验法。

## 2.1 冗余

冗余法最简单直接，即对存储的数据进行副本备份，当数据出现丢失，损坏，即可使用备份内容进行恢复，而副本 备份的多少，决定了数据可靠性的高低。这其中会有成本的考量，副本数据越多，数据越可靠，但需要的设备就越多，成本就越高。可靠性是允许丢失其中一份数据。当前已有很多分布式系统是采用此种方式实现，如 Hadoop 的文件系统（3个副本），Redis 的集群，MySQL 的主备模式等。

## 2.2 校验

校验法即通过校验码的数学计算的方式，对出现丢失、损坏的数据进行校验、还原。注意，这里有两个作用，一个校验，通过对数据进行校验和( checksum )进行计算，可以检查数据是否完整，有无损坏或更改，在数据传输和保存时经常用到，如 TCP 协议；二是恢复还原，通过对数据结合校验码，通过数学计算，还原丢失或损坏的数据，可以在保证数据可靠的前提下，降低冗余，如单机硬盘存储中的 RAID 技术，纠删码（Erasure Code）技术等。MinIO 采用的就是纠删码技术。

# 3 MinIO存储机制

## 3.1 概念理解

在部署分布式 MinIO 前，需要对下面的概念进行了解：

- 硬盘（Drive）：即存储数据的磁盘，在 MinIO 启动时，以参数的方式传入。
- 组（ Set ）：即一组 Drive 的集合，分布式部署根据集群规模自动划分一个或多个 Set ，每个 Set 中的 Drive 分布在不同位置。一个对象存储在一个 Set 上。
- 桶（Bucket）：文件对象存储的逻辑位置，对于客户端而言，就相当于一个存放文件的顶层文件夹。

## 3.2 纠删码EC（Erasure Code）

MinIO 使用纠删码机制来保证高可靠性，使用 highwayhash 来处理数据损坏（ Bit Rot Protection ）。关于纠删码，简单来说就是可以通过数学计算，把丢失的数据进行还原，它可以将n份原始数据，增加m份数据，并能通过n+m份中的任意n份数据，还原为原始数据。即如果有任意小于等于m份的数据失效，仍然能通过剩下的数据还原出来。举个最简单例子就是有两个数据(d1, d2)，用一个校验和y（`d1 + d2 = y`）即可保证即使丢失其中一个，依然可以还原数据。如丢失 d1 ，则使用 `y - d2 = d1` 还原，同理，d2 丢失或者y丢失，均可通过计算得出。

EC 的具体应用实现中， RS（Reed-Solomen）是 EC 的一种更简单快捷的实现，可以通过矩阵运算，还原数据。MinIO 将对象拆分成N/2数据和N/2 校验块 。具体的数学矩阵运算及证明，可以参考文章《[Erasure-Code-擦除码-1-原理篇]( https://blog.openacid.com/storage/ec-1/ )》及《[EC纠删码原理]( https://blog.csdn.net/shelldon/article/details/54144730 )》。

## 3.3 存储形式

文件对象上传到 MinIO ，会在对应的数据存储磁盘中，以 Bucket 名称为目录，文件名称为下一级目录，文件名称下是 part.1 和 xl.json，前者是编码数据块及检验块，后者是元数据文件。如有4个磁盘，当文件上传后，会有2个编码数据块，2个检验块，分别存储在4个磁盘中。如下图，`bg-01.jpg` 是上传的文件对象：

![存储结构]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200331-minio/store-cotent.png )

# 4 部署实践

## 4.1 单节点部署多磁盘

在启动 MinIO 时，若传入参数是多个目录，则会以纠删码的形式运行，即具备高可靠性意义。即在一个服务器（单节点）上对，多个磁盘上运行 MinIO。

运行命令也很简单，参数传入多个目录即可：

`MINIO_ACCESS_KEY=${ACCESS_KEY} MINIO_SECRET_KEY=${SECRET_KEY} nohup ${MINIO_HOME}/minio  server --address "${MINIO_HOST}:${MINIO_PORT}" /opt/min-data1 /opt/min-data2 /opt/min-data3 /opt/min-data4 > ${MINIO_LOGFILE} 2>&1 & `

注意替换命令中的变更，运行后输出信息如下：

![启动]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200331-minio/start-ok.png )

可见 MinIO 会创建一个1个 set，set 中有4个 drive ，其中它会提示一个警告，提示一个节点的 set 中存在多于2个的drive，如果节点挂掉，则数据都不可用了，这与 EC 码的规则一致。

## 4.2 多节点部署

### 4.2.1 部署脚本

为了防止单点故障，分布式存储自然是需要多节点部署，以达到高可靠和高可用的能力。MinIO 对于多节点的部署，也是在启动时通过指定有 Host 和端口的目录地址，即可实现。下面在单台机器上，通过不同的端口模拟在4台机器节点上运行，存储目录依然是 min-data1~4，而对应的端口是9001~9004。脚本如下：

```shell
RUNNING_USER=root
MINIO_HOME=/opt/minio
MINIO_HOST=192.168.222.10
#accesskey and secretkey
ACCESS_KEY=minio 
SECRET_KEY=minio123

for i in {01..04}; do
    START_CMD="MINIO_ACCESS_KEY=${ACCESS_KEY} MINIO_SECRET_KEY=${SECRET_KEY} nohup ${MINIO_HOME}/minio  server --address "${MINIO_HOST}:90${i}" http://${MINIO_HOST}:9001/opt/min-data1 http://${MINIO_HOST}:9002/opt/min-data2 http://${MINIO_HOST}:9003/opt/min-data3 http://${MINIO_HOST}:9004/opt/min-data4 > ${MINIO_HOME}/minio-90${i}.log 2>&1 &"
    su - ${RUNNING_USER} -c "${START_CMD}"
done
```

本示例中，minio 的启动命令运行了4次，相当于在四台机器节点上都分别运行一个minio实例，从而模拟四个节点。运行结果如下：

![分布式启动]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200331-minio/dis-start-log.png )

查看进程`ps -ef |grep minio`：

![分布式进程]( https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200331-minio/dis-ps.png )

### 4.2.2 部署注意点

- 所有运行分布式 MinIO 的节点需要具有相同的访问密钥和秘密密钥才能连接。建议在执行 MINIO 服务器命令之前，将访问密钥作为环境变量，MINIO access key 和 MINIO secret key 导出到所有节点上 。
-  Minio 创建4到16个驱动器的擦除编码集。
-  Minio 选择最大的 EC 集大小，该集大小除以给定的驱动器总数。 例如，8个驱动器将用作一个大小为8的 EC 集，而不是两个大小为4的 EC 集 。
-  建议所有运行分布式 MinIO 设置的节点都是同构的，即相同的操作系统、相同数量的磁盘和相同的网络互连 。
-  运行分布式 MinIO 实例的服务器时间差不应超过15分钟。

运行起来后，使用 `http://${MINIO_HOST}:9001` 到`http://${MINIO_HOST}:9004` 均可以访问到 MinIO 的使用界面。

### 4.2.3 使用 nginx 负载均衡

前面单独对每个节点进行访问显然不合理，通过使用 nginx 代理，进行负载均衡则很有必要。简单的配置如下：

```shell
upstream http_minio {
    server 192.168.222.10:9001;
    server 192.168.222.10:9002;
    server 192.168.222.10:9003;
    server 192.168.222.10:9004;
}

server{
    listen       8888;
    server_name  192.168.222.10;

    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_buffering off;

    location / {
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-Host  $host:$server_port;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  $http_x_forwarded_proto;
		proxy_set_header   Host $http_host;

        proxy_connect_timeout 300;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_ignore_client_abort on;

        proxy_pass http://http_minio;
    }
}
```

其中主要是 upstream 及 proxy_pass 的配置。如此，即可使用`http://${MINIO_HOST}:8888` 进行访问。



# 5 总结

对于分布式存储，高可靠必是首要考虑的因素，MinIO 已经提供了分布式部署的解决方案，实现高可靠、高可用的资源存储。本文对可靠性的实现方法进行描述，探讨了 MinIO 的存储机制，并通过脚本模拟实践 MinIO 的分布式部署，希望对大家有帮助。



# 参考资料

- [MinIO官网](https://min.io/): `https://min.io/`
- [MinIO开发文档](https://docs.min.io/): `https://docs.min.io/`
- [基于 Go 开源项目 MIMIO 的对象存储方案在探探的实践](https://mp.weixin.qq.com/s?__biz=MzA4ODg0NDkzOA==&mid=2247487119&idx=1&sn=6e09abb32392e015911be3a1d7f066e5&source=41)：`https://mp.weixin.qq.com/s/MzA4ODg0NDkzOA==&mid=2247487119&idx=1&sn=6e09abb32392e015911be3a1d7f066e5&source=41`
- [Minio 文件服务（1）—— Minio部署使用及存储机制分析]( https://www.jianshu.com/p/3e81b87d5b0b )：`https://www.jianshu.com/p/3e81b87d5b0b`
- [使用minio搭建高性能对象存储](https://tonybai.com/2020/03/16/build-high-performance-object-storage-with-minio-part1-prototype)：`https://tonybai.com/2020/03/16/build-high-performance-object-storage-with-minio-part1-prototype`


# 往期文章

- [利用MinIO轻松搭建静态资源服务](https://mp.weixin.qq.com/s/rzaqPmpTOUJJsKISr6eF9Q)
- [搞定SpringBoot多数据源(3)：参数化变更源](https://mp.weixin.qq.com/s/ZzzPJZAhPiGCQjN3RNZJ3Q)
- [搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)
- [搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)
- [java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)
- [2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)
- [springboot+logback 日志输出企业实践（下）](https://mp.weixin.qq.com/s/ha4LaR-E1gDxfUZ11neI-Q)
- [springboot+logback 日志输出企业实践（上）](https://mp.weixin.qq.com/s/Ti5i9vv9S1j4za5q11RWeA)


我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![mason](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)

