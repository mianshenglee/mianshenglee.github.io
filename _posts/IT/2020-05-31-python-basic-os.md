---
layout: post
title: python基本操作-文件、目录及路径
category: IT
tags: python os path
keywords: 
description: 
---

> 使用python的os模块，简单方便完成对文件夹、文件及路径的管理与访问操作。

##  1 前言

在最近开发中，经常需要对文件进行读取、遍历、修改等操作，想要快速、简单的完成这些操作，我选择用 python 。通过 python 的标准内置 os 模块，只需要几行代码，即可完成想要的操作。经过对 os 的使用，本文把 os 模块的常用的操作进行总结，主要分为以下几个划分：

- 文件夹操作：即文件夹的创建、修改（改名/移动），查询（查看、遍历）、删除等。
- 文件操作：即文件的创建、修改、读取、删除等。
- （文件夹/文件）路径操作：即文件夹或文件的路径操作，如绝对路径，文件名与路径分割，扩展名分割等



本文涉及常用 的 os 函数的使用展示，主要使用 python 交互模式下进行代码说明。后续操作默认已经引入 os 模块，如下：

```python
import os
```



## 2 文件夹操作

以本地 `E://pythontest` 目录作为演示目录，此目录下当前文件如下：

```shell
test
 │ test.txt
 └─test-1
     test-1.txt
```

`test`  及 `test-1`  是文件夹，`test.txt` 及 `test-1.txt` 是文件。

### 2.1 查询操作

熟悉 linux 同学应该对 `ls` / `pwd` / `cd` 等操作不陌生，对应的 python 也有对应的方法，主要包括：

- listdir ： 文件及目录列表
- getcwd ：获取当前目录
- chdir ：更换目录
- stat ：文件及目录基本信息
- walk ：递归遍历目录

```python
>>> os.chdir("E://pythontest")  # 更改目录
>>> os.getcwd()                 # 获取当前目录
'E:\\pythontest'
>>> os.listdir("test")          # 文件及目录列表，相对路径
['test-1', 'test.txt']          
>>> os.listdir("E://pythontest/test")  # 文件及目录列表，绝对路径
['test-1', 'test.txt']
>>> os.stat("test")             # 获取目录信息
os.stat_result(st_mode=16895, st_ino=4503599627377599, st_dev=266147611, st_nlink=1, st_uid=0, st_gid=0, st_size=0, st_atime=1590833033, st_mtime=1590832647, st_ctime=1590832207)
>>> os.stat("test/test.txt")    # 获取文件信息
os.stat_result(st_mode=33206, st_ino=2251799813692354, st_dev=266147611, st_nlink=1, st_uid=0, st_gid=0, st_size=4, st_atime=1590832653, st_mtime=1590832609, st_ctime=1590832598)
```

其中 stat 函数返回的是文件或者目录的基本信息，具体如下：

- **st_mode:** inode 保护模式
- **st_ino:** inode 节点号。
- **st_dev:** inode 驻留的设备。
- **st_nlink:** inode 的链接数。
- **st_uid:** 所有者的用户ID。
- **st_gid:** 所有者的组ID。
- **st_size:** 普通文件以字节为单位的大小
- **st_atime:** 上次访问的时间。
- **st_mtime:** 最后一次修改的时间。
- **st_ctime:** 创建时间。

日常使用中，我们一般使用 st_size 、st_ctime 及 st_mtime 获取文件大小，创建时间，修改时间。另外，我们看到输出的时间是秒数，在这里提一下，关于日期的转换处理。


（1）秒数转日期时间格式字符串

```python
>>> import time                              # 引入time模块
>>> timestruct = time.localtime(1590803070)  # 转换为时间结构体
>>> print(timestruct)
time.struct_time(tm_year=2020, tm_mon=5, tm_mday=30, tm_hour=9, tm_min=44, tm_sec=30, tm_wday=5, tm_yday=151, tm_isdst=0)
>>> time.strftime("%Y-%m-%d %H:%M:%S",timestruct)   # 格式化时间
'2020-05-30 09:44:30'
```


（2）格式日期时间字符串转秒数

```python
>>> import datetime               # 引入datetime模块
>>> timeobject = datetime.datetime.strptime("2020-05-23 10:00:00","%Y-%m-%d %H:%M:%S") #解析时间字符串为时间对象
>>> timeseconds=time.mktime(timeobject.timetuple())  # 获取时间秒数
>>> print(int(timeseconds))       # 转为int显示
1590199200
```

- 遍历操作

  walk 函数对目录进行递归遍历，返回 root，dirs，files，分别对应当前的遍历的目录，此目录中的子目录及文件。

```python
>>> data = os.walk("test")               # 遍历test目录
>>> for root,dirs,files in data:         # 递归遍历及输出
...    print("root:%s" % root)
...    for dir in dirs:
...       print(os.path.join(root,dir))
...    for file in files:
...       print(os.path.join(root,file))
...
root:test
test\test-1
test\test-2
test\test.txt
root:test\test-1
test\test-1\test-1.txt
root:test\test-2
test\test-2\test-2.txt
```



### 2.2 创建操作

- mkdir ：新建单个目录，若目录路径中父目录不存在，则创建失败

- makedirs ：新建多个目录，若目录路径中父目录不存在，则自动创建

```python
>>> os.mkdir("test")
>>> os.mkdir("test1/test1-1")          # 父目录不存在，报错
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
FileNotFoundError: [WinError 3] 系统找不到指定的路径。: 'test1/test1-1'
>>> os.makedirs("test1/test1-1")       # 父目录不存在，自动创建
>>> os.listdir("test1")
['test1-1']
```



### 2.3 删除操作

- rmdir ：删除单个空目录，目录不为空则报错
- removedirs ： 按路径删除递归多级空目录，目录不为空则报错

```python
>>> os.rmdir("test1")                         # 若目录不为空，报错
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
OSError: [WinError 145] 目录不是空的。: 'test1'
>>> os.rmdir("test1/test1-1")
>>> os.removedirs("test1/test1-1")            # 删除多级空目录
>>> os.listdir(".")
['test']
```

由于删除空目录的限制，更多的是使用 ` shutil ` 模块中的 ` rmtree ` 函数，可以删除不为空的目录及其文件。



### 2.4 修改操作 

- rename ：重命名目录或文件，可修改文件或目录的路径（即移动操作），若目标文件目录不存在，则报错。
- renames ：重命名目录或文件，若目标文件目录不存在，则自动创建

```python
>>> os.makedirs("test1/test1-1")
>>> os.rename("test1/test1-1","test1/test1-2")     # test1-1 修改为test1-2
>>> os.listdir("test1")
['test1-2']
>>> os.rename("test1/test1-2","test2/test2-2")     # 由于test2目录不存在，报错
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
FileNotFoundError: [WinError 3] 系统找不到指定的路径。: 'test1/test1-2' -> 'test2/test2-2'
>>> os.renames("test1/test1-2","test2/test2-2")    # renames可自动创建不存在的目录
>>> os.listdir("test2")
['test2-2']
```

如果目标路径文件已经存在，那么os.rename()和os.renames()都会报错：FileExistsError: [WinError 183] 当文件已存在时，无法创建该文件。



## 3 文件操作

### 3.1 查询操作

- open/read/close ：文件读取
- stat ：文件信息，详细见前面文件夹中的 stat 说明

```python
>>> f = os.open("test/test.txt", os.O_RDWR|os.O_CREAT)  # 打开文件
>>> str_bytes = os.read(f,100)                          # 读100字节
>>> str = bytes.decode(str_bytes)                       # 字节转字符串
>>> print(str)
test write data
>>> os.close(f)                                         # 关闭文件
```

注意 open/read/close 需要一起操作，其中 open 操作需要指定模式，上述是以读写模式打开文件，若文件不存在则创建文件。各模式具体如下：

**flags** -- 该参数可以是以下选项，多个使用 "|" 隔开：

- **os.O_RDONLY:** 以只读的方式打开
- **os.O_WRONLY:** 以只写的方式打开
- **os.O_RDWR :** 以读写的方式打开
- **os.O_NONBLOCK:** 打开时不阻塞
- **os.O_APPEND:** 以追加的方式打开
- **os.O_CREAT:** 创建并打开一个新文件
- **os.O_TRUNC:** 打开一个文件并截断它的长度为零（必须有写权限）
- **os.O_EXCL:** 如果指定的文件存在，返回错误
- **os.O_SHLOCK:** 自动获取共享锁
- **os.O_EXLOCK:** 自动获取独立锁
- **os.O_DIRECT:** 消除或减少缓存效果
- **os.O_FSYNC :** 同步写入
- **os.O_NOFOLLOW:** 不追踪软链接

### 3.2 创建操作

 前面已提到，使用 open ，指定模式， 若文件不存在，则创建。有点类似 linux 操作中的 touch。

```python
>>> f = os.open("test/test.txt", os.O_RDWR|os.O_CREAT)   # 若文件不存在，则创建
>>> os.close(f)
```

### 3.3 修改操作

- open/write/close ：写入文件内容
- rename ，renames ： 与前面介绍的修改名称、移动操作一致。

```python
>>> f = os.open("test/test.txt", os.O_RDWR|os.O_CREAT)     # 打开文件
>>> os.write(f,b"test write data")                         # 写入内容
15
>>> os.close(f)                                   # 关闭文件
```



### 3.4 删除

- remove ：删除文件，注意不能删除目录（使用 rmdir/removedirs）

```python
>>> os.remove("test/test-1")       # 删除目录报错
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
FileNotFoundError: [WinError 2] 系统找不到指定的文件。: 'test/test1'
>>> os.remove("test/test.txt")     # 删除文件
>>> os.listdir("test")
['test-1']
```



## 4 路径操作

在使用文件或目录过程中，经常需要对文件及目录路径进行处理，因此，os 中有一个子模块 path，专门就是处理路径操作的。主要有以下操作：

- abspath ：返回绝对路径

```python
>>> os.path.abspath("test")
'E:\\pythontest\\test'
```

- exists ：判断文件或目录是否存在

```python
>>> os.path.exists("test")
True
>>> os.path.exists("test/test.txt")
False
>>> os.path.exists("test/test-1/test-1.txt")
True
```

- isfile/isdir ：判断是否为文件/目录

```python
>>> os.path.isdir("test")
True
>>> os.path.isfile("test/test-1/test-1.txt")
True
```

- basename/dirname：获取路径尾部和路径头部。其实就是以路径中最后一个 `/` 为分割符，分为头（head） 和尾（tail）两部分，tail 是 basename 返回的内容，head 是 dirname 返回的内容。经常用于获取文件名，目录名等操作

```python
>>> os.path.basename("test/test-1/test-1.txt")   # 文件名
'test-1.txt'
>>> os.path.basename("test/test-1/")     # 空内容
''
>>> os.path.basename("test/test-1")      # 目录名
'test-1'
>>> os.path.dirname("test/test-1/test-1.txt")   # 文件所在目录路径
'test/test-1'
>>> os.path.dirname("test/test-1/")   # 目录路径
'test/test-1'
>>> os.path.dirname("test/test-1")   # 父目录路径
'test'
```

- join ：合成路径，即把两个参数使用系统路径分割符进行连接，形成完整路径。

```python
>>> os.path.join("test","test-1")   # 连接两个目录
'test\\test-1'
>>> os.path.join("test\\test-1","test-1.txt")   # 连接目录与文件名
'test\\test-1\\test-1.txt'
```

- split ：分割文件名和文件夹，即把 path 以最后一个斜线"/"为分隔符，切割为 head 和 tail ，以 (head, tail) 元组的形势返回。

```python
>>> os.path.split("test/test-1")     # 分割目录
('test', 'test-1')
>>> os.path.split("test/test-1/")    # 以/结尾的目录分割
('test/test-1', '')
>>> os.path.split("test/test-1/test-1.txt")  # 分割文件
('test/test-1', 'test-1.txt')
```

- splitext ：分割路径名和文件扩展名，把path 以最后一个扩展名分隔符“.”分割，切割为 head 和 tail ，以 (head, tail) 元组的形势返回。注意与 split 的区别是分隔符的不同。

```python
>>> os.path.splitext("test/test-1")  
('test/test-1', '')
>>> os.path.splitext("test/test-1/") 
('test/test-1/', '')
>>> os.path.splitext("test/test-1/test-1.txt")  # 区分文件名及扩展名
('test/test-1/test-1', '.txt')
>>> os.path.splitext("test/test-1/test-1.txt.tmp") # 以最后的"."为分割点
('test/test-1/test-1.txt', '.tmp')
```



## 5 示例应用

下面以一些平时使用到的场景，对前面的操作函数进行综合使用。

### 5.1 批量修改文件名

```python
def batch_rename(dir_path):
    itemlist = os.listdir(dir_path)
    # 获取目录文件列表
    for item in itemlist:
        # 连接成完整路径
        item_path = os.path.join(dir_path, item)
        print(item_path)
        # 修改文件名
        if os.path.isfile(item_path):
            splitext = os.path.splitext(item_path)
            os.rename(item_path, splitext[0] + "-副本" + splitext[1])
```

![批量修改文件名](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200531-python/batch-modify.png)

### 5.2 遍历目录及子目录下所有指定扩展名的文件

```python

def walk_ext_file(dir_path,ext):
    # 遍历
    for root, dirs, files in os.walk(dir_path):
        # 获取文件名称及路径
        for file in files:
            file_path = os.path.join(root, file)
            file_item = os.path.splitext(file_path)
            # 输出指定扩展名的文件路径
            if ext == file_item[1]:
                print(file_path)
```

![遍历指定扩展名文件](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200531-python/list-file-ext.png)

### 5.3 按修改时间排序指定目录下的文件

```python
def sort_file(dir_path):
    # 排序前
    itemlist = os.listdir(dir_path)
    print(itemlist)
    # 正向排序
    itemlist.sort(key=lambda filename: os.path.getmtime(os.path.join(dir_path, filename)))
    print(itemlist)
    # 反向排序
    itemlist.sort(key=lambda filename: os.path.getmtime(os.path.join(dir_path, filename)), reverse=True)
    print(itemlist)
    # 获取最新修改的文件
    print(itemlist[0])
```

![排序文件](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200531-python/sort-file.png)

## 6 总结

在需要对文件或者目录进行操作时，python 是一个简单快速选择。本文通过 python 的标准内置 os 模块及子模块 os.path 的常用方法进行介绍，最后结合使用场景进行综合使用。相信已经满足大家对文件及目录操作的大部分需求。



## 参考资料

- [python之os模块](https://www.cnblogs.com/yufeihlf/p/6179547.html)：`https://www.cnblogs.com/yufeihlf/p/6179547.html`
- [Python OS 文件/目录方法](https://www.runoob.com/python/os-file-methods.html): `https://www.runoob.com/python/os-file-methods.html`
- [Python os.path() 模块](https://www.runoob.com/python/python-os-path.html): `https://www.runoob.com/python/python-os-path.html`



## 往期文章

- [MinIO 的分布式部署](https://mp.weixin.qq.com/s/LX-4jhVV7FbB-TA8jcWYOA)
- [利用MinIO轻松搭建静态资源服务](https://mp.weixin.qq.com/s/rzaqPmpTOUJJsKISr6eF9Q)
- [搞定SpringBoot多数据源(3)：参数化变更源](https://mp.weixin.qq.com/s/ZzzPJZAhPiGCQjN3RNZJ3Q)
- [搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)
- [搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)
- [java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)
- [2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)



我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![mason](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)