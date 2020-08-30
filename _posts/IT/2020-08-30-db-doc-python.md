---
layout: post
title: 还在手工生成数据库文档？3个步骤自动完成了解一下
category: IT
tags: db doc mysql python excel
keywords: 
description: 
---

> 自动化生成数据库文档，简单的3个步骤即可完成，了解一下。

## 1 前言

平时工作中，大家应该都会遇到需要导出数据库说明文档（也叫数据字典）的情况，即把各数据表的字段信息整理成一个个的表说明，然后用 excel/word/html/md 等文档格式进行保存。很多小伙伴还在用原始的手工方式，复制粘贴数据库的字段说明（名称、类型、长度、注释等），不得不说这种方式效率实在太低。作为程序员，能用编程解决的问题，就不是问题。下面介绍的方法很简单，只需要3个步骤。本文将对这3个步骤使用 python 进行编码实现，把数据表信息说明输出到 excel 文档中。因此，主要包含以下内容：

- 生成数据库文档的3步骤说明
- 获取数据库表元信息
- 获取数据表列的元信息
- 生成数据库说明 excel 文档
- （可选）设置 excel 文档格式

## 2 生成数据库说明文档的3步骤

由于数据库都会保存相应的元数据信息（即描述数据库、数据表、数据字段本身的信息，如表名，字段表、类型等等），因此，总的来说，生成数据库说明文档的思路很简单，分为3步：

- 1）根据数据库名，从数据库中获取数据表元信息，主要是表名，表注释等
- 2）根据数据表名，获取数据字段的元信息，主要是字段名、字段类型、是否可空、字段注释等
- 3）根据元数据信息生成文档

根据这个思路，把这3个步骤通过编码即可自动生成文档。获取元数据信息，各种数据库会有不同的查询语句，具体可以查询相关官方文档，下面简单列一下 mysql 及 oracle 的：

```sql
# mysql 查询表信息及字段信息
SELECT * FROM information_schema.`TABLES` WHERE TABLE_SCHEMA = %db_name%
SELECT * FROM information_schema.`COLUMNS` WHERE TABLE_SCHEMA = %db_name% AND TABLE_NAME = %table_name%

# oracle 查询表信息及字段信息
SELECT * FROM all_tables WHERE where owner= %db_name%
SELECT * FROM all_COL_COMMENTS WHERE owner = %db_name%  and TABLE_NAME=%table_name%
```

实现方式也可以根据各人喜欢的编程语言来实现。在本文中，以 MySQL 为例，使用 python 编程实现，把数据信息输出到 excel 文档（具体 excel 操作，可参考我上一篇文章《[ Python 处理 Excel 文件](https://mp.weixin.qq.com/s/GAoOOXebxshW1fGCLJc8BQ)》）。输出效果如下所示：

![带格式的数据库说明文档](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200830-python/db-excel.png)

下面就跟着我进行实现吧。

## 3 获取数据库表元信息

### 3.1 客户端 pymysql 基本使用

使用 python 进行 MySQL 读写操作，使用的是 [pymysql]( https://github.com/PyMySQL/PyMySQL )，读者可以访问它的[官方文档]( https://pymysql.readthedocs.io/en/latest )了解它的安装和使用。简单来说，对数据库进行读数据，需要以下几步：

- 连接数据库：`connect`
- 获取读数据的游标（ cursor ）：`connection.cursor()`
- 执行 SQL 语句获取数据：`cursor.execute(sql,args)`，`cursor.fetchall()`，`cursor.fetchone()`,`cursor.fetchmany()`
- 关闭游标和连接：`connection.close()`
- 获取正在使用的工作表：`workbook.active`，`cursor.close()`

因此，我们在类的初始化（ `__init__` ）和关闭（`__del__` ）时，进行数据库连接和关闭操作。代码如下：

```python
def __init__(self, host, port, user, password, db_name, charset):
    # 初始化数据库操作

    self.db = pymysql.connect(host=host, port=port, user=user,password=password, database=db_name, charset=charset)
    self.cursor = self.db.cursor()

def __del__(self):
    # 关闭数据库连接

    self.db.close()
    self.cursor.close()
```



### 3.2 获取数据库表元信息

根据上面说的 pymysql 的基本操作，只需要执行查询数据表元信息的 SQL 即可。前面已经提到，MySQL 的查询表元信息的 SQL 语句是`SELECT * FROM information_schema.TABLES  WHERE TABLE_SCHEMA = %db_name%`  ，而我们只需要表名及表注释即可。因此实现如下：

```python
def get_table_info(self, db_name):
    # 获取数据表信息

    sql = '''SELECT table_schema, table_name, table_comment FROM information_schema.`TABLES` WHERE TABLE_SCHEMA = %s order by table_name'''
    params = [db_name]
    # 查询数据
    self.cursor.execute(sql, params)
    return self.cursor.fetchall()
```

此函数功能：传入数据库名，返回所有表信息。



## 4 获取数据表字段的元信息

获取到表信息后，同样的道理，需要遍历每一个表，根据表名获取每个表的字段信息。前面已经到，MySQL 获取表的字段是查询表 `information_schema.COLUMNS`即可，而对于字段信息， 我们主要关注字段名、字段类型、 是否允许为空、字段的注释描述等信息。代码如下：

```python
def get_table_column_info(self, database_name, table_name):
    # 获取数据表列信息

    params = [database_name, table_name]
    sql = '''SELECT
                TABLE_SCHEMA AS '库名',TABLE_NAME AS '表名',
                COLUMN_NAME AS '列名',ORDINAL_POSITION AS '列的排列顺序',
                COLUMN_DEFAULT AS '默认值',IS_NULLABLE AS '是否为空',
                DATA_TYPE AS '数据类型',CHARACTER_MAXIMUM_LENGTH AS '字符最大长度',
                NUMERIC_PRECISION AS '数值精度(最大位数)',NUMERIC_SCALE AS '小数精度',
                COLUMN_TYPE AS '列类型',COLUMN_COMMENT AS '注释'
            FROM information_schema.`COLUMNS`
            WHERE TABLE_SCHEMA = %s AND TABLE_NAME = %s
            ORDER BY TABLE_NAME, ORDINAL_POSITION'''
    # 查询数据
    self.cursor.execute(sql, params)
    return self.cursor.fetchall()
```

此函数功能，根据数据库名及表名，获取此表的字段信息。



## 5 生成 excel 文档

### 5.1 输出表字段信息到 excel 文档

对于 excel 的操作，我们使用 openpyxl 进行读写操作，具体 excel 操作，可参考我上一篇文章《[ Python 处理 Excel 文件](https://mp.weixin.qq.com/s/GAoOOXebxshW1fGCLJc8BQ)》。而现在我们需要实现的功能是把每个表的字段信息，以表格的方式写入到 excel 表中，并按字段名、允许为空、字段类型、字段描述进行输出。

还有一点就是，我们经常在设计表的的过程中，基本都会有一些公共的字段，比如 id ，创建时间、创建人、修改时间、修改人等，这些我们在导出字典时，可以选择过滤掉。因此，使用如下代码进行实现：

```python
def create_file(self, file_path):
    # 获取文件，若文件不存在则创建，存在则删除后重新创建

    if os.path.exists(file_path):
        os.remove(file_path)
    wb = Workbook()
    wb.save(file_path)
    
def save_column_info_to_excel(self, table_name, table_comment, column_info, file_path, col_names_skip):
    # 写入表信息到excel文件

    workbook = openpyxl.load_workbook(file_path)
    # 根据下标获取（下标从0开始）
    sheet = workbook.worksheets[0]
    row_data = [table_name]
    if table_comment:
        row_data = [table_name + "(" + table_comment + ")"]
        sheet.append(row_data)
        rurrent_max_row = sheet.max_row
        # 空行分隔
        sheet.insert_rows(rurrent_max_row)
        # 列名
        col_name_data = ["字段名", "允许为空", "类型", "字段描述"]
        sheet.append(col_name_data)
        for row in column_info:
            # 需要过滤的
            if col_names_skip and row[2].lower() in col_names_skip:
                print("#" * 10, "跳过此字段：", row[2])
                continue
            print(row[2] + "," + row[5] + "," + row[10] + "," + row[11])
            row_data = [row[2], row[5], row[10], row[11]]
            sheet.append(row_data)
        # 保存文档
        workbook.save(file_path)
```

此处包含两个函数，`create_file` 功能主要是创建文档，若文档存在则先删除。`save_column_info_to_excel`功能是根据表字段信息及需要过滤的字段名，使用 for 语句按行输出到 excel 中，输出过程中，若有需要过滤的字段则跳过，最后把文档保存到指定的路径中。

### 5.2 把各功能连接起来

前面已经实现数据库表元信息获取、数据表字段元信息获取及字段信息输出到 excel 文档三个功能。现在把这三个功能连接起来，就可以形成完整的数据库文档导出功能了。思路是遍历生成的表元信息( get_table_info )，根据表元信息获取表的字段信息( gen_table_column_info )，然后输出 excel 文档( save_column_info_to_excel )，如下所示：

```python
def gen_db_table_info_skip_col(self, db_name, file_path, col_names_skip):
    # 过滤指定列，导出数据表信息到文档

    table_info_rows = self.get_table_info(db_name)
    for table_row in table_info_rows:
        print("\n", "*" * 10, "生成表信息：", table_row[1])
        self.gen_table_column_info(table_row, file_path, col_names_skip)

def gen_table_column_info(self, table_info_row, file_path, col_names_skip=None):
    # 导出字段信息表到文档

    database_name = table_info_row[0]
    table_name = table_info_row[1]
    table_comment = table_info_row[2]
    # 从数据库获取表信息
    column_info = self.get_table_column_info(database_name, table_name)
    # 写入excel文件
    self.save_column_info_to_excel(table_name, table_comment, column_info, file_path, col_names_skip)
```

此处包含两个函数，`gen_db_table_info_skip_col` 功能是根据数据库名、文件保存路径、需要过滤的字段名导出表元信息，然后使用 for 语句进行遍历。`gen_table_column_info` 是根据表信息及需要过滤的字段，先读表字段信息，然后写入到 excel 文档。注意，此处`col_names_skip` 默认值为 None ，即如果不需要过滤，不输入此参数即可。至此，我们自动生成数据库文档的功能已完成。在`__main__` 中执行看一下输出情况：

```python
if __name__ == '__main__':
    # 输出文档地址
    excel_path = "E:/pythontest/test_tableinfo.xlsx"
    # 数据库连接信息
    host = "localhost"
    port = 3306
    user = "root"
    password = "123456"
    db_name = "test"
    charset = 'utf8'
    # 需要过滤的字段
    col_names_to_skip = ["id", "sys_create_time", "sys_create_user", "sys_update_time", "sys_update_user", "record_version"]
	# 初始化类，创建文件，生成数据库说明文档
    dbInfoGenerator = DbInfoGenerator(host, port, user, password, db_name, charset)
    dbInfoGenerator.create_file(excel_path)
    dbInfoGenerator.gen_db_table_info_skip_col(db_name, excel_path, col_names_to_skip)
```

结果如下：

![无格式数据库说明文档](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/20200830-python/db-excel-without-format.png)

表的字段说明已输出到 excel 文档中，对应的字段也已过滤。只是格式不是很好看，因此，有需要的可以用 openpyxl 对 excel 文档的格式进行设置即可。

## 6 （可选）设置 excel 文档格式

如果需要对 excel 文档进行格式设置，以下是我的一个基本格式设置，有需要的可以参考一下，制作适合自己的文档格式。格式的设置思路主要如下：

- 设置各列的宽度
- 遍历 excel 表每一行，如果是表名，则合并为一行作为表头，设置表头格式为加粗、加黑色边框、居中对齐、填充背景色。
- 如果表字段信息内容，则设置黑色边框即可。

```python
def set_file_format(self, file_path):
    # 设置表格式

    if not os.path.exists(file_path):
        print("文件不存在，不处理")
        return
    workbook = openpyxl.load_workbook(file_path)
    sheet = workbook.worksheets[0]
    # 设置各列宽
    sheet.column_dimensions["A"].width = 16
    sheet.column_dimensions["B"].width = 10
    sheet.column_dimensions["C"].width = 20
    sheet.column_dimensions["D"].width = 40

    # 设置表名格式
    max_row = sheet.max_row
    for i in range(1, max_row + 1):
        col1_value = sheet.cell(i, 1).value
        col2_value = sheet.cell(i, 2).value
        # 首列有数据，第2列无数据，则为表名
        if col1_value and not col2_value:
            # 合并表名
            sheet.merge_cells(start_row=i, start_column=1, end_row=i, end_column=4)
            # 加粗字体
            font = Font(name="微软雅黑", size=12, bold=True, italic=False, color="000000")
            # 黑色边框
            side_style = Side(style="thin", color="000000")
            border = Border(left=side_style, right=side_style, top=side_style, bottom=side_style)
            # 居中对齐
            cell_alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)
            # 填充背景色
            p_fill = PatternFill(fill_type="solid", fgColor="BFBFBF")
            # 表名cell格式
            for j in range(1, 5):
                sheet.cell(i, j).font = font
                sheet.cell(i, j).border = border
                sheet.cell(i, j).alignment = cell_alignment
                sheet.cell(i, j).fill = p_fill
         # 若首列和第2列都有数据，则是表内容
         if col1_value and col2_value:
            # 黑色边框
            side_style = Side(style="thin", color="000000")
            border = Border(left=side_style, right=side_style, top=side_style, bottom=side_style)
            # 表名cell格式
            for j in range(1, 5):
                sheet.cell(i, j).border = border
    # 保存文档
    workbook.save(file_path)
```

当生成数据库说明文档后，调用此函数，即可修改其文档格式，效果如下：

![1598756250234](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1598756250234.png)

## 7 总结

本文主要针对数据库说明文档（数据字典）的自动化生成进行讲解。通过使用 SQL 读取数据库表及字段元信息，然后输出到 excel 文档的思路，以 python 的实现方式完成自动生成文档功能。如果你还在手工生成数据库说明文档，可以试试这种方法，一定让你效率大增。希望可以帮助到有需要的人。如果想看完整的代码，可到我 [github]( https://github.com/mianshenglee/my-example/tree/master/python/tool-gen-db-doc ) 地址中查看：` https://github.com/mianshenglee/my-example/tree/master/python/tool-gen-db-doc `

根据本文实现的思路，最后可以留几个思考题给大家，想想如何做：

- 不使用 python ，用其它你熟悉的语言来实现此功能。
- 如何只需要生成指定表的字段信息或者过滤指定表，怎么做？
- 数据库表名一般都会有前缀或后缀，能否根据前缀或后缀来过滤生成或者过滤，怎么做？
- 本文是生成 excel 文档，如果需要生成 word 、html 、md 、pdf 等格式的文档，怎么做？

## 参考资料

- [openpyxl官方文档]( https://openpyxl.readthedocs.io/ ): ` https://openpyxl.readthedocs.io/`
- [pymysql官方文档](  https://pymysql.readthedocs.io/en/latest/ ): ` https://pymysql.readthedocs.io/en/latest/`



## 往期文章

- [Python 处理 Excel 文件](https://mp.weixin.qq.com/s/GAoOOXebxshW1fGCLJc8BQ)
- [python基本操作-文件、目录及路径](https://mp.weixin.qq.com/s/neO8OuWu7Za41xcdmwu2wA)
- [MinIO 的分布式部署](https://mp.weixin.qq.com/s/LX-4jhVV7FbB-TA8jcWYOA)
- [利用MinIO轻松搭建静态资源服务](https://mp.weixin.qq.com/s/rzaqPmpTOUJJsKISr6eF9Q)
- [搞定SpringBoot多数据源(3)：参数化变更源](https://mp.weixin.qq.com/s/ZzzPJZAhPiGCQjN3RNZJ3Q)
- [搞定SpringBoot多数据源(2)：动态数据源](https://mp.weixin.qq.com/s/neIN3htjkn4bifPpdq5l7w)
- [搞定SpringBoot多数据源(1)：多套源策略](https://mp.weixin.qq.com/s/0J-FLYScYtEMnj0vZToX7g)
- [java开发必学知识:动态代理](https://mp.weixin.qq.com/s/a3x_pKUryb_at_4Xk48IiQ)
- [2019 读过的好书推荐](https://mp.weixin.qq.com/s/Wlbjhohb_HrqT67lstwVwA)



我的公众号（搜索`Mason技术记录`），获取更多技术记录：

![mason](https://gitee.com/mianshenglee/datastorage/raw/master/md-photo/myphoto/wx/wx-public.jpg)