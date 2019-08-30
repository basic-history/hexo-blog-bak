---
title: mysql架构和存储引擎
date: 2019-02-19 12:00:00
author: pleuvoir
img: /images/mysql.jpg
tags:
  - mysql
categories:
  - 技术
---


本文是mysql学习的第一章，主要介绍mysql架构和存储引擎。

## mysql架构

* 连接层
* 服务层
* 引擎层
* 存储层


连接层：可以理解为JDBC层，用于和客户端进行通信，其中还包括连接池等。

1）当mysql启动，每一个客户端发出连接请求，服务端会新建一个线程进行处理，每一个线程都有自己的内存空间。如果是线程池的话那么是预先分配一个连接。如果存在多写的情况，修改同一块数据会出现同步问题。

2）其中连接到服务器进行密码等权限验证也是连接层完成的。

查询设置最大连接：

```sql
show VARIABLES like '%max_connections%'
set GLOBAL max_connections = 200
```


服务层：

其中服务层由以下4部分构成：

1. SQL处理层（负责一些sql语句的执行，存储过程，视图等）
2. 缓存
3. 解析（解析为msql认识的语句）
4. 优化（优化一些无谓的操作）

整体的流程是：

1)如果是查询语句，则查询缓存是否有相同的结果，无则进行下一步

2)解析查询语句，用于解析用户提交的sql为mysql自己认识的语句

3）优化sql语句，譬如`select * from user where 1=1`则会优化为`select * from user`，mysql会进行一些分析，从而避免一些不必要的执行语句。

关于缓存的使用：

一般情况下我们会选用外部的缓存中间件如redis，memcache，很少会用去mysql自带的缓存。如果需要开启，需要同时打开开关并设置缓存大小才会生效，如下：

```sql
show variables like '%query_cache_type%'
SET GLOBAL query_cache_size = 4000;
SET GLOBAL query_cache_size = 134217728;
select * from product_info where product_name ='iphone7'
```

当开启了缓存后，会发现第一次查询耗时较久，第二次查询几乎是瞬间返回。关于缓存的内容，它有可能会缓存执行计划，索引，也有可能会缓存数据；具体取决于存储引擎。



## 存储引擎

* MyISAM
* Innodb


```sql
--查看你的mysql现在支持什么存储引擎
show engines;
--查看默认的存储引擎
show variables like '%storage_engine%';
```

查看建表语句：

```sql
show create table 表名
```

一般建表语句中会指明表所使用的的存储引擎。

### MyISAM

特性：

并发性与锁级别-表级锁

支持全文检索

支持数据压缩
	myisampack -b -f testmysqm.MYI


如果使用myisam那么插入数据后会在mysql的安装目录下生成三个文件，假设我们的表名为`testmysqm`，文件名为分别是以下格式：

1. testmysqm.frm（和存储引擎无关，保存的是表结构信息）
2. testmysqm.MYD（数据信息DATA）
3. testmysqm.MYI(索引信息INDEX)


具体压缩指令如下（请修改为自己机器的实际目录）：

```sql
.\myisamchk.exe -b -f "C:\ProgramData\MySQL\MySQL Server 5.6\data
\mysqldemo\product_info.MYI"
```

如果我们压缩后出现了无法插入修改的问题，可以选择恢复：

```sql
CHECK table 表名
REPAIR table 表名
```

适用场景：

非事务型应用（数据仓库、报表、日志数据）

只读类应用

空间类如GIS应用（空间函数、坐标，必须选择myisam，只有它提供空间函数）

缓存：只缓存索引不缓存真实数据


### Innodb

特性：

ACID事务支持

Redo Log和Undo Log

支持行级锁（并发程度更高

适用场景：

大多数需要事务的应用


mysql5.5以后的默认引擎，需要注意的是可以检查表数据使用的表空间类型，建议使用独立的表空间以提高IO效率。

```sql
show VARIABLES like 'innodb_file_per_table'

on:独立的表空间 数据文件 表名.ibd
off:系统表空间 所有表的文件都在ibdataX这一个文件中

--on off
set global innodb_file_per_table=on
```

在mysql5.6以前默认为系统表空间，独立表空间的优点不言而喻，多表写数据操作更快，可收缩系统文件

```sql
OPTIMIZE TABLE product_info2 收缩优化表，可以理解为整理磁盘空间
```
