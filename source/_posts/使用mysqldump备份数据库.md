---
title: 使用mysqldump备份数据库
date: 2017-12-05 08:58:33
category: MariaDB
tags:
	- MariaDB
	- MySQL
	- mysqldump
---

 MariaDB 备份恢复



> 备份类型

完全备份：备份真个数据集

部分备份：只备份数据子集



增量备份：仅备份最近一次完全备份或增量备份（如果存在增量）以来发生变化的数据，在还原时需要将所有备份合并到完全备份上，比较麻烦。

差异备份：仅备份最近一次完全备份以来发生变化的数据，还原时只需要将最后一次差异备份合并到完全备份上，这种备份比较浪费空间。

`以上两种备份还需要依赖 bin-log做时间点还原`



热备份：备份时读写操作均可执行，实现复杂，在 MariaDB 中只有 innodb 存储引擎支持热备

温备份：备份时读操作可执行，但写操作不能执行

冷备份：备份时读写操作均不可执行（服务停止）



物理备份：直接复制数据文件进行备份

逻辑备份：通过 mysql 客户端连接到 MariaDB 服务端从数据库中“导出”数据另存进行备份



> 需要备份的数据

数据

二进制日志、InnoDB 的事务日志

代码（存储过程、存储函数、触发器、事件调度器）

服务器的配置文件



> 备份时需要考虑的因素

持锁多久

备份过程的时长

备份负载

恢复过程的时长



> 备份工具

mysqldump：逻辑备份工具，适用于所有存储引擎，只支持温备，可以做完全备份或部分备份。对于 innodb 存储引擎支持热备，但如果数据集过大耗时会很久自然持锁时间也会很长。

lvm2 的快照：几乎热备，借助于文件系统管理工具进行备份

mysqlhotcopy：几乎冷备，仅适用于 MyISAM 存储引擎

xtrbackup：有 Percona 提供的支持对 innodb 做热备（物理备份）的工具，完全备份、增量备份。



> 通过 mysqldump 逻辑备份数据库

mysqldump 是一个 mysql 客户端工具，通过 mysql 协议连接至 mysql 服务器，向 mysql 服务器发起全量查询操作，将读取到的数据保存到文件中实现备份。

对于 MyISAM 支持温备，备份时先锁定需要备份的库，再启动备份操作，因为在备份时不锁定库，如果此时正好有用户在写入数据，备份出来的数据库会出现时间点不一致。

mysqldump 选项

```
shell> mysqldump [options] db_name [tbl_name ...]# 只备份其中的数据表，还原时需要手动创建数据库
shell> mysqldump [options] --databases db_name ...# 备份时会备份数据库，还原是不用手动创建数据库
shell> mysqldump [options] --all-databases# 备份时会备份数据库，还原是不用手动创建数据库

-x, --lock-all-tables	#备份时锁定所有表，温备，适用于 MyISAM 存储引擎
-l, --lock-tables	#备份时锁定正在备份的表，温备，适用于 MyISAM 存储引擎
--single-transaction	#备份时对目标库启动一个快照单一的事务。将事务的隔离级别设置为可重复读(REPEATABLE READ) ，这样能保证在 dump 期间一个事务中所有相同的查询读取到相同的数据。仅能用于 innodb 引擎热备。
-E, --events	#备份目标数据库相关的所有 event scheduler。
-R, --routines	#备份目标数据库相关的所有存储过程和存储函数。
--triggers	#备份目标数据库相关的触发器(默认启用)。
--skip-triggers		#不备份目标数据库相关的触发器。
--master-data[=#]	#备份时记录二进制日志位置。赋值为 1 备份文件中记录二进制日志位置，但不被注释，为 2 表示将二进制日志位置记录在备份文件中，但该语句会被注释。
--flush-logs	#备份时表锁定完成后，执行 flush logs 命令，滚动二进制日志(重新生成一个新的二进制日志文件)
```



**示例**

这里使用两台 MariaDB 测试，分别为 node1，node2，测试在 node1 上备份数据，到 node2 上进行还原。

创建测试数据库`stuinfo`（MariaDB 版本为 5.5.56，默认使用 innodb 存储引擎）

```
MariaDB [(none)]> CREATE DATABASE stuinfo CHARACTER SET = 'utf8';
```

在 `stuinfo` 数据库中创建 `student` 表

```
MariaDB [(none)]> CREATE TABLE stuinfo.student (s_id INT,s_name CHAR(30));
```

在其中插入两条数据

```
MariaDB [(none)]> INSERT INTO stuinfo.student VALUE (1,'xiaoming');
MariaDB [(none)]> INSERT INTO stuinfo.student VALUE (2,'xiaohong');
```

现在 `student` 表中有两条数据了

```
MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+----------+
| s_id | s_name   |
+------+----------+
|    1 | xiaoming |
|    2 | xiaohong |
+------+----------+
```

先使用 mysqldump 完全备份一次数据库

```
[root@node1 ~]# mysqldump -uroot -p123456 -h 127.0.0.1 --single-transaction --master-data=2 --databases stuinfo
```

在 sql 脚本中找到二进制信息

```
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000007', MASTER_LOG_POS=1050;
```

从该信息中可以看到日志文件为 mysql-bin.000007 起始位置为 1050。还原时作时间点还原就需要从 mysql-bin.000007 这个日志文件的 POS 1050 位置作还原。

现在将备份好的 sql 脚本传到 node2 上。

```
scp stuinfo.sql root@node2:/tmp
```

然后再在 node1 的 `student` 表中插入一条信息。

```
MariaDB [(none)]> INSERT INTO stuinfo.student VALUE (3,'xiaohua');
```

现在 node1 上的 `student` 数据库中有 3 条信息了

```
MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+----------+
| s_id | s_name   |
+------+----------+
|    1 | xiaoming |
|    2 | xiaohong |
|    3 | xiaohua  |
+------+----------+
```

如果此时 node1 上的数据库宕机了，能用的就只有二进制文件了（所以二进制文件不应该跟数据库文件放在同一路径）。

现在再把 node1 上从上次备份后的二进制文件拷贝到 node2 上

```
scp mysql-bin.000007 root@node2:/tmp
```

在 node2 上登录 MariaDB 临时关闭二进制日志的记录，因为现在是还原数据库，记录二进制日志是没有意义的

```
MariaDB [(none)]> SET @@SESSION.sql_log_bin = OFF;
```

首先导入全量备份的数据库

```
MariaDB [stuinfo]> source /tmp/stuinfo.sql;
```

但导入的数据库中只有两条数据

```
MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+----------+
| s_id | s_name   |
+------+----------+
|    1 | xiaoming |
|    2 | xiaohong |
+------+----------+
```

现在使用 mysqlbinlog 将备份的二进制文件从 1050 这个位置读取出来

```
[root@node2 /tmp]# mysqlbinlog --start-position=1050 mysql-bin.000007 > stubinlog.sql
```

导入二进制日志脚本

```
MariaDB [stuinfo]> source /tmp/stubinlog.sql;
```

查看数据信息

```
MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+----------+
| s_id | s_name   |
+------+----------+
|    1 | xiaoming |
|    2 | xiaohong |
|    3 | xiaohua  |
+------+----------+
```

可以看到现在数据都成功还原回来了。



END!