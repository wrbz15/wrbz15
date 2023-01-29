---
title: MYSQL架构
date: 2023-01-20T19:17:58+08:00
lastmod: 2023-01-27T19:17:58+08:00
author: ["wrbz15"]
categories: 
- 数据库
tags: 
- 数据库
- MYSQL
description: ""
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

## 服务器处理客户端查询请求流程

不论客户端进程和服务器进程是采用哪种方式进行通信，最后实现的效果都是：客户端进程向服务器进程发送一段文本（MySQL语句），服务器进程处理后再向客户端进程发送一段文本（处理结果）。

![mysql架构图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWctbXkuY3Nkbi5uZXQvdXBsb2Fkcy8yMDEyMTEvMTUvMTM1Mjk1OTg0N185MTQ3LmpwZw?x-oss-process=image/format,png)

大体来说， MySQL 可以分为 Server 层和存储引擎层两部分。

Server 层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

而存储引擎层负责数据的存储和提取。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。 现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始成为了默认存储引擎。

### 连接器

连接器负责跟客户端建立连接、获取权限、维持和管理连接。

建立链接方式有： TCP/IP`、`命名管道或共享内存`、`Unix域套接字等。

建立连接后MySQL服务器会为每一个连接进来的客户端分配一个线程。

用户名密码认证通过，连接器会到权限表里面查出你拥有的权限。之后，这个连接里面的权限判断逻辑，都将依赖于此时读到的权限。

连接使用的方式有：长连接、短连接

短连接缺点：建立连接的过程成本较高

长连接缺点：MySQL 在执行过程中临时使用的内存是管理在连接对象里面的，这些资源会在连接断开的时候才释放。（解决方案：mysql_reset_connection

### 查询缓存

`MySQL`服务器程序处理查询请求的过程也是这样，会把刚刚处理过的查询请求和结果`缓存`起来

使用限制：

1. MySQL的缓存系统会监测涉及到的每张表，只要该表的结构或者数据被修改，如对该表使用了`INSERT`、 `UPDATE`、`DELETE`、`TRUNCATE TABLE`、`ALTER TABLE`、`DROP TABLE`或 `DROP DATABASE`语句，那使用该表的所有高速缓存查询都将变为无效并从高速缓存中删除！
2. 如果两个查询请求在任何字符上的不同（例如：空格、注释、大小写），都会导致缓存不会命中。
3. 如果查询请求中包含某些系统函数、用户自定义变量和函数、一些系统表，如 mysql 、information_schema、 performance_schema 数据库中的表，那这个请求就不会被缓存。

### 分析器

分析器先会做“词法分析”。你输入的是由多个字符串和空格组成的一条 SQL 语句， MySQL 需要识别出里面的字符串分别是什么，代表什么。

做完了这些识别以后，就要做“语法分析”。 根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个 SQL 语句是否满足 MySQL 语法。

### 优化器

优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。

### 执行器 

执行器阶段，开始执行语句。

1. 首先会判断一下你对这个表 T 有没有执行查询的权限，如果没有，就会返回没有权限的错误；

2. 打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口

tips: rows_examined是执行器每次调用引擎获取数据行的时候累加的







