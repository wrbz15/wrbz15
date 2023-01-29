---
title: InnoDB内存中结构
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

* Buffer Pool
* Change Buffer
* 自适应哈希索引
* Log BuFFer

#### Buffer Pool

缓冲池是主存储器中的一个区域，用于在访问时缓存表和索引数据。 缓冲池允许直接从内存处理常用数据，从而加快处理速度。 在专用服务器上，通常会将最多80％的物理内存分配给缓冲池。

为了提高大容量读取操作的效率，缓冲池被分成 可以容纳多行的 [页面 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_page)。 为了提高缓存管理的效率，缓冲池被实现为链接的页面列表; 使用 [LRU ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_lru)算法 的变体，很少使用的数据在缓存中老化 。

##### 缓冲池LRU算法

使用最近最少使用（LRU）算法的变体将缓冲池作为列表进行管理。 当需要空间将新页面添加到缓冲池时，最近最少使用的页面被逐出，并且新页面被添加到列表的中间。 此中点插入策略将列表视为两个子列表：

- 在头部， 最近访问过 的新（ “ 年轻 ” ）页面 的子列表
- 在尾部，是最近访问的旧页面的子列表


![Content is described in the surrounding text.](http://www.deituicms.com/mysql8cn/cn/images/innodb-buffer-pool-list.png)



该算法保留新子列表中查询大量使用的页面。 旧子列表包含较少使用的页面; 这些页面是 [驱逐的 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_eviction)候选人 。

默认情况下，算法操作如下：

- 3/8的缓冲池专用于旧子列表。
- 列表的中点是新子列表的尾部与旧子列表的头部相交的边界。
- 当 `InnoDB` 将页面读入缓冲池时，它最初将其插入中点（旧子列表的头部）。 可以读取页面，因为它是用户指定的操作（如SQL查询）所必需的，或者是 由自动执行的 [预读 ](http://www.deituicms.com/mysql8cn/cn/glossary.html#glos_read_ahead)操作的 一部分 `InnoDB` 。
- 访问旧子列表中的页面使其 “ 年轻 ” ，将其移动到缓冲池的头部（新子列表的头部）。 如果因为需要而读取页面，则会立即进行第一次访问，并使页面变得年轻。 如果由于预读而读取了页面，则第一次访问不会立即发生（并且在页面被逐出之前可能根本不会发生）。
- 随着数据库的运行，在缓冲池的页面没有被访问的 “ 年龄 ” 通过向列表的尾部移动。 新旧子列表中的页面随着其他页面的变化而变旧。 旧子列表中的页面也会随着页面插入中点而老化。 最终，仍然未使用的页面到达旧子列表的尾部并被逐出。

##### 重要指标：

* 内存命中率：执行show engine innodb status，得到返回值中Buffer pool hit rate。一般建议设置成可用物理内存的 60%~80%。

#### change Buffer

change Buffer是一种特殊的数据结构，当这些页面不在Buffer Pool中， 该缓存可缓存对二级索引页的更改。可能由DML导致的缓冲更改将在以后通过其他读取操作将页面加载到缓冲池中时合并。

![内容在周围的文字中描述。](https://dev.mysql.com/doc/refman/8.0/en/images/innodb-change-buffer.png)

* 与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引。同样，删除和更新可能会影响索引树中不相邻的二级索引页。当其他操作将受影响的页面读入缓冲池时，稍后合并缓存的更改可避免将二级索引页面从磁盘读入缓冲池所需的大量随机访问 I/O。

* 在系统大部分处于空闲状态或缓慢关闭期间运行的清除操作会定期将更新的索引页写入磁盘。与将每个值立即写入磁盘相比，清除操作可以更有效地为一系列索引值写入磁盘块。

* change buffer可以看成也是一个数据页，需要被持久化到 系统表空间（ibdata1），以及把这个change buffer页的改动记录在redo log里，事后刷进系统表空间（ibdata1）。
* 当有许多受影响的行和许多辅助索引要更新时，更改缓冲区合并可能需要几个小时。在此期间，磁盘 I/O 会增加，这可能会导致磁盘绑定查询的速度大大降低。提交事务之后，甚至在服务器关闭并重新启动之后，更改缓冲区合并也可能 continue 发生。
* change buffer的merge：将旧的二级索引页加载到内存中，合并change buffer生成新页的过程。

##### 重要指标

* innodb_change_buffer_max_size 变量允许将更改缓冲区的最大大小配置为缓冲池总大小的百分比
* innodb_change_buffering可以为插入、删除操作（当索引记录最初被标记为删除时）和清除操作（当索引记录被物理删除时）启用或禁用缓冲

#####  使用场景

* 写多读少： 页面在写完以后马上被访问到的概率比较小。

#### 自适应哈希索引

自适应哈希索引功能使`InnoDB`能够在具有适当的工作负载和足够的缓冲池内存的系统上，像内存数据库一样执行操作，而不会牺牲事务功能或可靠性。通过配置innodb_adaptive_hash_index启用

特点

- 哈希索引，查询消耗 O(1)
- 降低对二级索引树的频繁访问资源。
- 自适应

缺点

- hash自适应索引会占用innodb buffer pool；
- 自适应hash索引只适合搜索等值的查询，如select * from table where index_col='xxx'，而对于其他查找类型，如范围查找，是不能使用的；

#### log Buffer

日志缓冲区是存储区域，用于保存要写入磁盘上的日志文件的数据。

日志缓冲区的大小由innodb_log_buffer_size变量定义。默认大小为 16MB。日志缓冲区的内容会定期刷新到磁盘。较大的日志缓冲区使大型事务可以运行，而无需在事务提交之前将重做日志数据写入磁盘。因此，如果您有更新，插入或删除许多行的事务，则增加日志缓冲区的大小可以节省磁盘 I/O。

##### 重要参数

innodb_flush_log_at_trx_commit：

- innodb_flush_log_at_trx_commit  = 0， InnoDB 中的 Log Thread 每隔 1秒将 log buffer 中的数据写入文件，同时还会通知文件系统进行与文件同步的 flush操作，保证数据确实已经写入磁盘。
- innodb_flush_log_at_trx_commit = 1， InnoDB 默认设置。每次事务的结束都会出发 Log Thread 将 Log Buffer 中的数据写入文件、并通知文件系统同步文件。这个设置最安全，能够保证不论是 MySQL 崩溃、OS崩溃还是主机断电都不会丢失任何已经提交的数据。
- innodb_flush_log_at_trx_commit = 2， 每次事务结束的时候将数据写入事务日志，仅仅是调用了文件系统的文件写入操作。而文件系统都是有缓存机制的，所以 Log Thread 的写入并不能保证内容已经写入到物理磁盘完成持久化的动作。文件系统什么时候会将缓存中的数据同步到物理磁盘、文件， Log Thread 就完全不知道，所以，当设置 2 的时候， MySQL 崩溃并不会造成数据的丢失，但是 OS 崩溃或主机断电后可能丢失的数据量就完全控制在文件上了。

innodb_flush_log_at_timeout：控制redo log缓存写到redo log文件的频率。