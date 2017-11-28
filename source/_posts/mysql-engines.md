---
title: MySQL Innodb MyISAM engine
date: 2017-11-26 17:45:19
category: MySQL
tags:
	- mysql
	- innodb
	- myisam
---
MySQL 存储引擎


**MySQL 结构体系**
![mysql-arch](http://ov2iiuul1.bkt.clouddn.com/mysql-arch.png )

​									MySQL 结构体系

MySQL 由以下几部分组成

* 连接池组件
* 管理服务器和工具组件
* SQL 接口组件
* 查询分线器组件
* 优化器组件
* 缓冲组件
* 插件式存储引擎
* 物理文件



MySQL 数据库区别于其它数据库的最重要的一个特点就是其插件式的表存储引擎。MySQL 插件式的存储引擎架构提供了一系列的管理和服务支持，这些标准与存储引擎本身无关，可能是每个数据库系统都必须的，如 MySQL 分析器和优化器等，而存储引擎是地产物理结构的实现，每个存储引擎开发者都可以按照自己的意愿来进行开发。存储引擎是基于表的，而不是数据库。

由于 MySQL 数据库开源的特性，存储引擎可以分为 MySQL 官方存储引擎和第三方存储引擎。有些第三方存储引擎很强大，如 InnoDB 存储引擎，最早是第三方存储引擎，后被 Oracle 收购，其应用及其广泛，甚至是 MySQL 数据库 OLTP (Online Transaction Processing 在线事务处理)应用中使用最广泛的存储引擎。

**InnoDB 存储引擎**

InnoDB 存储引擎支持事务，其设计的目标主要面向在线事务处理（OLTP）的应用。特点是行级别锁，支持外键，并支持类似 Oracle 的非锁定读，也就是默认读取操作不会产生锁。从 MySQL 数据库 5.5.8 版本开始，InnoDB 存储引擎是默认的存储引擎。

InnoDB 存储引擎将数据放在一个逻辑的表空间中，这个表空间就像黑盒一样由 InnoDB 存储引擎自身进行管理。从 MySQL 4.1 版本开始，它可以将每个 InnoDB 存储引擎的表单独存放在一个独立的 ibd 文件中。此外，InnoDB 存储引擎支持裸设备 （row disk）用来建立表空间。

InnoDB 通过使用 MVCC（Multiversion Concurrency Controll，多版本并发控制）来获得高并发，并且实现了 SQL 标准的 4 种隔离级别，默认为 REPEATABLE（可重复读）级别。同时，使用一种被称为 netx-key locking 的策略来避免幻读（phantom）现象的产生。除此之外，InnoDB 存储引擎还提供了插入缓冲（instert buffer）、二次写（double write）、自适应哈希索引（adaptive hash index）、预读（read ahead）等高性能和高可用的功能。

对于表中数据的存储，InnoDB 存储引擎采用了聚集（clustered）的方式，因此每张表的存储都是按主键的顺序进行存放。如果没有显示地在表定义时指定主键，InnoDB 存储引擎会为每一行生成一个 6 字节的 ROWID ，并以此作为主键。



**MyISAM 存储引擎**

MyISAM 存储引擎不支持事务、表级别锁，支持全文索引，主要面向一些 OLAP （On-Line Analytical Processing ，联机分析处理） 数据库应用。在 MySQL 5.5.8 版本之前 MyISAM 存储引擎是默认的存储引擎。数据库系统与文件系统很大的一个不同之处在于对事务的支持，然而 MyISAM 存储引擎是不支持事务的，在数据仓库中，如果没有 ETL (Extract Transform and Load，提取转换加载) 这些操作，只是简单的报表查询，可能对事务要求没那么高。MyISAM 另外一个与众不同的地方是它的缓冲池只缓存索引文件，而不缓存数据文件，这点和大多数的数据库都非常不同。

MyISAM 存储引擎表由 MYD 和 MYI 组成，MYD 用来存储数据文件，MYI 用来存放索引文件。可以通过使用 `myisampack` 工具来进一步压缩数据文件，因为 `myisampack` 工具使用赫夫曼（Hufffman）编码静态算法来压缩数据，因此使用 `myisampack` 工具压缩后的表是只读的，也可以通过 `myisampack` 来解压数据文件。

在 MySQL 5.0 版本之前，MyISAM 默认支持的表大小为 4GB，如果需要支持大于 4GB 的 MyISAM 表时，则需要指定 MAX_ROWS 和 AVG_ROW_LENGTH 属性。从 MySQL 5.0 版本开始，MyISAM 默认支持 256TB 的单表数据，这足够一般应用需求。

对于 MyISAM 存储引擎表，MySQL 数据库只缓存其索引文件，数据文件的缓存交由操作系统本身来完成，这与其他使用 LRU 算法缓存数据的大部分数据库大不相同。此外，在 MySQL 5.1.23 版本之前，无论是在 32 位还是 64 位操作系统环境下，缓存索引的缓冲区最大只能设置 4GB。在之后的版本中，64 为操作系统可以支持大于 4GB 的索引缓冲区。 