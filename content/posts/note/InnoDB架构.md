---
title: InnoDB架构
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

#### InnoDB简介

`InnoDB` 是一种平衡高可靠性和高性能的通用存储引擎。 在MySQL 8.0中， `InnoDB` 是默认的MySQL存储引擎。 除非您配置了不同的默认存储引擎，否则发出 [`CREATE TABLE`](http://www.deituicms.com/mysql8cn/cn/sql-syntax.html#create-table) 不带 `ENGINE=` 子句的语句会创建 `InnoDB` 表。

#### innoDB优点

- 如果您的服务器因硬件或软件问题而崩溃，无论当时数据库中发生了什么，您都无需在重新启动数据库后执行任何特殊操作。 `InnoDB` [崩溃恢复会 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_crash_recovery)自动完成在崩溃之前提交的所有更改，并撤消正在进行但未提交的任何更改。 只需重新启动并继续您离开的地方。
- 该 `InnoDB` 存储引擎维护它自己的 [缓冲池 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_buffer_pool)，当数据被访问主内存中缓存表和索引数据。 经常使用的数据直接从内存中处理。 此缓存适用于许多类型的信息并加快处理速度。 在专用数据库服务器上，通常会将最多80％的物理内存分配给缓冲池。
- 如果将相关数据拆分到不同的表中，则可以设置 强制 [引用完整性的 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_referential_integrity)[外键 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_foreign_key)。 更新或删除数据，并自动更新或删除其他表中的相关数据。 尝试将数据插入到辅助表中，而不在主表中显示相应的数据，并且坏数据会自动被踢出。
- 如果数据在磁盘或内存中损坏， [校验和 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_checksum)机制会在您使用之前提醒您伪造数据。
- 使用 每个表的 相应 [主键 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_primary_key)列 设计数据库时 ，将自动优化涉及这些列的操作。 引用 [`WHERE`](http://www.deituicms.com/mysql8cn/cn/sql-syntax.html#select) 子句， [`ORDER BY`](http://www.deituicms.com/mysql8cn/cn/sql-syntax.html#select) 子句， [`GROUP BY`](http://www.deituicms.com/mysql8cn/cn/sql-syntax.html#select) 子句和 [连接 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_join)操作中 的主键列非常快 。
- 插入，更新和删除通过称为 [更改缓冲 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_change_buffering)的自动机制进行优化 。 `InnoDB` 不仅允许对同一个表进行并发读写访问，还可以缓存已更改的数据以简化磁盘I / O.
- 性能优势不仅限于具有长时间运行查询的巨型表。 当从表中反复访问相同的行时，称为 [自适应哈希索引的功能 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_adaptive_hash_index)会接管以使这些查找更快，就像它们来自哈希表一样。
- 您可以压缩表和关联的索引。
- 您可以创建和删除索引，而对性能和可用性的影响要小得多。
- 截断 [每个表 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_file_per_table)的 [文件表 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_file_per_table)空间非常快，并且可以释放磁盘空间以供操作系统重用，而不是释放 [系统表空间 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_system_tablespace)中只能 `InnoDB` 重用的[ 空间 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_system_tablespace)。
- [`BLOB`](http://www.deituicms.com/mysql8cn/cn/data-types.html#blob) 使用 [DYNAMIC ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_dynamic_row_format)行格式 ，表数据的存储布局对于 长文本字段 更有效 。
- 您可以通过查询 [INFORMATION_SCHEMA ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_information_schema)表 来监视存储引擎的内部工作方式 。
- 您可以通过查询 [性能架构 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_performance_schema)表 来监控存储引擎的性能详细信息 。
- 您可以自由地将 `InnoDB` 表与来自其他MySQL存储引擎的表 混合 ，甚至可以在同一语句中。 例如，您可以使用 [连接 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_join)操作来组合 单个查询中的 数据 `InnoDB` 和 [`MEMORY`](http://www.deituicms.com/mysql8cn/cn/storage-engines.html#memory-storage-engine) 表。
- `InnoDB` 专为处理大量数据时的CPU效率和最高性能而设计。
- `InnoDB` 表可以处理大量数据，即使在文件大小限制为2GB的操作系统上也是如此。

#### innoDB架构

![InnoDB architecture diagram showing in-memory and on-disk structures.](http://www.deituicms.com/mysql8cn/cn/images/innodb-architecture.png)

