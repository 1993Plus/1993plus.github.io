---
title: 使用Percona XtraBackup备份MariaDB数据库
date: 2017-12-05 09:03:01
category: MariaDB
tags:
	- MariaDB
	- MySQL
	- xtrabackup
---

XtraBackup

XtraBackup 是由著名的 MySQL 咨询服务公司 Percona 公司所开发，他其实是 Oracle 官方所研发的 InnoDB Host Backup 的升级版。

Percona 官方做了一个与 InnoDB Hot Backup 的对比表：https://www.percona.com/software/mysql-database/percona-xtrabackup/feature-comparison

> 官方的介绍

Percona XtraBackup 是一个免费，开源，支持完全在线备份的解决方案，他所支持的 MySQL 版本有 Percona MySQL Server（Percona 的 MySQL 版本分支），MySQL，MariaDB。他能在事务性存储引擎上执行在线非阻塞，压缩，高度安全的备份（在 MyISAM 存储引擎上只能执行阻塞式备份）。Percona XtrBackup 拥有超过 210 万的下载量（2017年12月）。



XtraBackup 官方下载地址：https://www.percona.com/downloads/XtraBackup/LATEST/

XtraBackup 官方在线文档：https://www.percona.com/doc/percona-xtrabackup/LATEST/index.html

在下载列表中有 4 个包，第一个是后三个的集合打包，分为 dubuginfo 包，主包，test 包，通常只需要下载主包就可以了。

![xtrabackup-download](http://ov2iiuul1.bkt.clouddn.com/xtrabackup-download.png)

使用 wget 下载

```
[root@node1 pkg]# wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.9/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.9-1.el7.x86_64.rpm
```

因为安装的时候有一些依赖包需要安装，如果是 Red Hat 或 CentOS 系列的系统建议直接使用 yum 来安装(如果配置了 epel 源的话也可以直接使用 yum 安装 epel 源里面的版本)

```
[root@node1 pkg]# yum install -y percona-xtrabackup-24-2.4.9-1.el7.x86_64.rpm
```

安装完成后可以看看他生成的文件，可以发现他的文件非常少，是一个非常轻量的工具。

```
[root@node1 pkg]# rpm -ql percona-xtrabackup-24-2.4.9-1.el7.x86_64
/usr/bin/innobackupex
/usr/bin/xbcloud
/usr/bin/xbcloud_osenv
/usr/bin/xbcrypt
/usr/bin/xbstream
/usr/bin/xtrabackup
/usr/share/doc/percona-xtrabackup-24-2.4.9
/usr/share/doc/percona-xtrabackup-24-2.4.9/COPYING
/usr/share/man/man1/innobackupex.1.gz
/usr/share/man/man1/xbcrypt.1.gz
/usr/share/man/man1/xbstream.1.gz
/usr/share/man/man1/xtrabackup.1.gz
```

通常备份能用到的就是 `innobackupex`  和 `xtrabackup  `

`xtrabackup` 是一个 C 语言开发二进制程序，他链接至 InnoDB 库和 MySQL 客户端标准库。通过 InnoDB 库提供了将日志应用到数据文件的功能，而 Mysql 客户端标准库提供了命令行选项的解析、配置文件解析等功能。

`xtrabackup` 支持增量备份，也就是说他只能复制上一次完全备份以来发生了变化的数据。可以在每个完整备份之间执行许多增量备份，因此可以设置一个备份计划，例如每周进行一次完整备份，每天进行增量备份。

在备份的同时，xtrabackup 还会在备份目录下创建文件 `xtrabackup_checkpoints` ，记录备份类型（完全备份或增量备份），备份状态（是否已经为 perpared 状态），和 LSN (日志序列号) 范围信息。每个 InnoDB 页（通常为 16kb ）都会包含一个 LSN，LSN 是整个数据库系统的系统版本号，每个页面相关的 LSN 能够表明此页面最近是如果发生改变的。增量备份实际上并没有将数据文件与之前的备份数据文件进行比较，他只是读取 InnoDB 页，并将其 LSN 与最后一个备份的 LSN 进行比较。

`innobackupex` 这个程序在 2.3 (包括2.3) 版本之前是一个 perl 脚本，它对 `xtrabackup` 进行了封装，在备份 innodb 表是，这个脚本会调用 `xtrabackup` 。直接使用 `xtrabackup` 只能备份 Mysql 的 innodb 表和 MariaDB 的 xtraDB ，如果使用 `innobackupex` 就能备份 innodb 表和 xtradb 表，同时还能备份 myisam 表。在 2.4 版本以后官方使用 C 语言将 `innobackupex` 的功能整合进了 `xtrbackup` ，但为了兼容之前版本中老用户的使用习惯，在 2.4 版本中的 `innobackupex` 只是一个链接到 `xtrabackup` 的符号链接。他允许用户传递旧版本中 `innobackupex` 的参数选项。

```
[root@node1 ~]# ll /usr/bin/innobackupex
lrwxrwxrwx 1 root root 10 Dec  2 17:22 /usr/bin/innobackupex -> xtrabackup
```



备份时可以通过远程连接至 MariaDB 服务验证权限通过后进行备份。Percona XtraBackup 所执行的备份属于物理备份（通过服务数据文件），恢复时也是将数据文件拷贝到数据目录完成恢复，恢复时不能启动 MAriaDB  服务，所以恢复时只能在提供 MariaDB 服务的服务器上安装 Percona XtraBackup 进行恢复。

因为在备份那一刻可能有些事务并没有完成，所以 Percona XtraBackup 在备份时还会将 InnoDB 的事务日志一同备份，在还原时如果是没有做完的事务就直接回滚，如果是执行完了的事务，但是没有提交，那就直接提交，也就是崩溃后恢复。

这时候有个问题就是在做完一次完整备份后再做增量备份，他们直接可能也会存在事务不完整的情况。如果将第一个增量备份直接合并到完整备份上，并把一些不完整的事务给回滚了，这时在将第二个增量备份合并上来的时候就会出问题了。所以再进行恢复时不能随意将事务日志进行提交或回滚，只有将最后一个增量备份合并到完整备份后才能对事务做提交或回滚。这个过程在 Percona XtraBackup 中叫 prepare（准备）。

因为不管如何备份都是无法做到备份到系统宕机那一刻的，在恢复时还是需要通过二进制日志来做时间点的数据恢复，所以 Percona XtraBackup 在备份时会将备份那一刻正在使用的二进制日志文件名与二进制日志的结束位置记录下来。在恢复数据时只需要将记录的二进制日志之后的日志取出来进行 replay 就可以了。这需要在启动 MariaDB 服务后通过人为来操作。

 `innobackupex` 是一个为了能兼容 Oracle 的 Innodb backup 在 `xtrabackup`  之上做的二次封装。在 Percona XtraBackup 2.4 版本之前，这个程序是一个 Perl 脚本，从 2.4 版本开始官方使用 C 语言将他重写整合到了 `xtrabackup` 程序中，所以在 2.4 版本所看到的 `innobackupex` 其实是一个连接到 `xtrabackup`  的符号链接，但他依然能接受旧版本中 `innobackupex` 所支持的选项。通常也是使用 `innobackupex`。



> innoackupex 选项

```
-u, --user=name	# 连接 MariaDB 服务的用户名
-p, --password=name	#连接密码
-H, --host=name	#要连接的主机地址
-P, --port=#	#MariaDB 监听的端口
-S, --socket=name	#通过 socket 连接 MariaDB 服务
--databases=name	#指定要备份的库
--apply-log	#应用备份数据目录中的 “xtrabackup_logfile” 事务日志，做恢复前准备操纵
--redo-only	#在对有增量备份的数据做准备操作时跳过“回滚”，只做“重做”，直到合并最后一个增量时
--copy-back	#将准备好的备份数据复制到数据目录中
--move-back	#将准备好的备份数据直接移动到数据目录中。
--incremental	#表示本次备份类型为增量备份。需要使用 --incremental-basedir 指定基于哪个备份做增量
--incremental-basedir=name	#指定基于哪个数据目录做增量备份
--incremental-dir=name	#需要合并到完整备份的增量备份数据目录
--parallel=#	#备份时指定开启多少个线程来并行备份
--use-memory=#	#备份时能使用的内存大小，支持直接给定单位（如 1MB,1GB)
```



> 权限相关

在备份时应该使用一个最小权限的用户来进行备份

创建一个用户

```
MariaDB [(none)]> CREATE USER 'bakuser'@'localhost' IDENTIFIED BY '123456';
```

撤销他的所有权限

```
MariaDB [(none)]> REVOKE ALL PRIVILEGES,GRANT OPTION FROM 'bakuser'@'localhost';
```

重新授予权限（通常只需要 `RELOAD`, `LOCK TABLES`, `REPLCATION CLIENT` 这几个权限就可以了）

```
MariaDB [(none)]> GRANT RELOAD,PROCESS,LOCK TABLES,REPLICATION CLIENT ON *.* TO 'bakuser'@'localhost';
```

刷新表权限

```
MariaDB [(none)]> FLUSH PRIVILEGES;
```

> 准备一次完整备份

创建备份数据存放的目录

```
[root@node1 ~]# mkdir /tmp/databak
```

对所有库进行一次完整备份

```
[root@node1 ~]# innobackupex -ubakuser -p123456 /tmp/databak/
```

在开始备份后程序给出了一个提示，只有最后显示 `completed OK!` 时表示备份成功

```
IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".
```

备份完成后备份数据会存放在一个以当前时间命名的目录中

```
[root@node1 ~]# ll /tmp/databak/
drwxr-x--- 5 root root 175 Dec  3 22:29 2017-12-03_22-29-34
```

在备份目录中的 `xtrabackup_checkpoints` 中可以看到备份类型，及 LSN 先关信息

通过文件中的 backup_type 可以看到本次备份类型为 full-backuped 表示完整备份

```
[root@node1 ~]# cat /tmp/databak/2017-12-03_22-29-34/xtrabackup_checkpoints 
backup_type = full-backuped
from_lsn = 0
to_lsn = 1604969
last_lsn = 1604969
compact = 0
recover_binlog_info = 0
```

在备份数据目录中除了创建会 `xtrabackup_checkpoints` 还会创建以下文件

- backup-my.cn：备份时 MariaDB 服务器的必要配置信息
- xtrabackup_binlog_info ：备份那一刻所处的二进制日志位置信息
- xtrabacku_info ：备份时传递给 Percona XtraBackup 参数，及软件版本信息等
- xtrabackup_logfile ：备份时的事务日志信息

> 恢复备份数据

在恢复时，在恢复数据的系统上也要安装 Percona XtraBackup，恢复前需要先对备份数据做 Prepare （准备）操作，将备份数据中的事务应用到数据文件中。将数据复制的数据目录之前停止 MariaDB 服务，清空数据目录。

使用 `--apply-log` 选项并指定数据目录

```
[root@node2 ~]# innobackupex --apply-log 2017-12-03_22-29-34/
```

最后显示 `completed OK!` 表示成功

现在可以 `--copy-back` 将备份数据复制到数据目录。复制时 Percona XtraBackup 会读取 `/etc` 下 MariaDB 的配置文件 my.cnf 获取数据目录

```
[root@node2 ~]# innobackupex --copy-back 2017-12-03_22-29-34/
```

数据文件复制到数据目录后确认文件的所有者与所有组是否为 mysql，如果不是 mysql 将其改为 mysql

```
[root@node2 ~]# chown -R mysql:mysql /var/lib/mysql/
```

现在就可以正常启动 MariaDB 服务了

```
[root@node2 ~]# systemctl start mariadb.service
```

如果需要做时间点数据恢复，在启动 MariaDB 服务后将对二进制日志做 replay 就可以了。



> 基于完整备份做增量备份

当前在 stuinfo 数据库 student 表中有 5 条数据

```
MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+--------+
| s_id | s_name |
+------+--------+
|    1 | Stu01  |
|    2 | Stu02  |
|    3 | Stu03  |
|    4 | Stu04  |
|    5 | Stu05  |
+------+--------+
```

对当前状态做一次完整备份

```
[root@node1 ~]# innobackupex -uroot -p123456 /tmp/databak/
```

在 stuinfo 数据库 student 表中插入一条数据

```
MariaDB [(none)]> INSERT INTO stuinfo.student VALUE (6,'Stu06');

MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+--------+
| s_id | s_name |
+------+--------+
|    1 | Stu01  |
|    2 | Stu02  |
|    3 | Stu03  |
|    4 | Stu04  |
|    5 | Stu05  |
|    6 | Stu06  |
+------+--------+
```

通过 `xtrabackup_checkpoints` 查看备份类型与 LSN 信息

```
[root@node1 ~]# cat /tmp/databak/2017-12-04_17-07-33/xtrabackup_checkpoints 
backup_type = full-backuped
from_lsn = 0
to_lsn = 1626814
last_lsn = 1626814
compact = 0
recover_binlog_info = 0
```

现在基于完整备份对发生变化的数据做增量备份

```
[root@node1 ~]# innobackupex -uroot -p123456 --incremental /tmp/databak/ --incremental-basedir /tmp/databak/2017-12-04_17-07-33/
```

通过 `xtrabackup_checkpoints` 可以查看到 backup_type 为 incremental（增量备份）

他的起始 LSN 为完整备份的结束位置。

```
[root@node1 ~]# cat /tmp/databak/2017-12-04_17-12-54/xtrabackup_checkpoints 
backup_type = incremental
from_lsn = 1626814
to_lsn = 1627868
last_lsn = 1627868
compact = 0
recover_binlog_info = 0
```

再插入一条数据

```
MariaDB [(none)]> INSERT INTO stuinfo.student VALUE (7,'Stu07');

MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+--------+
| s_id | s_name |
+------+--------+
|    1 | Stu01  |
|    2 | Stu02  |
|    3 | Stu03  |
|    4 | Stu04  |
|    5 | Stu05  |
|    6 | Stu06  |
|    7 | Stu07  |
+------+--------+
```

现在基于上次的增量备份再对发生变化的数据做增量备份，如果每次都是基于完整备份做备份的话属于差异备份。

```
[root@node1 ~]# innobackupex -uroot -p123456 --incremental /tmp/databak/ --incremental-basedir /tmp/databak/2017-12-04_17-12-54/
```

他的 LSN 起始位置也是由上次备份的结束位置为起始点

```
[root@node1 ~]# cat /tmp/databak/2017-12-04_17-16-34/xtrabackup_checkpoints 
backup_type = incremental
from_lsn = 1627868
to_lsn = 1628922
last_lsn = 1628922
compact = 0
recover_binlog_info = 0
```

再向数据库中插入一条数据，用来模拟二进制日志重放

```
MariaDB [(none)]> INSERT INTO stuinfo.student VALUE (8,'Stu08');

MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+--------+
| s_id | s_name |
+------+--------+
|    1 | Stu01  |
|    2 | Stu02  |
|    3 | Stu03  |
|    4 | Stu04  |
|    5 | Stu05  |
|    6 | Stu06  |
|    7 | Stu07  |
|    8 | Stu08  |
+------+--------+
```



> 恢复增量备份

在另外一台 MariaDB 服务器上恢复这些增量备份。首先这台 MariaDB 也应该安装 Percona XtraBackup ，安装好 MariaDB 服务，但不能启动 MariaDB 服务，不能初始化数据库。

先对第一次的完整备份做 `apply-log` 操作，但不能回滚一些未完成的事务，所以要加上 `--redo-only` 选项

```
[root@node2 databak]# innobackupex --apply-log --redo-only 2017-12-04_17-07-33/
```

先合并第一个增量备份到完整备份上，他的事务也不能做回滚操作，所以也需要加上  `redo-only` 选项。除此之外还需要使用 `--incremental-dir` 指定增量备份的路径

```
[root@node2 databak]# innobackupex --apply-log --redo-only 2017-12-04_17-07-33/ --incremental-dir 2017-12-04_17-12-54/
```

再合并上最后一次的增量备份

```
[root@node2 databak]# innobackupex --apply-log --redo-only 2017-12-04_17-07-33/ --incremental-dir 2017-12-04_17-16-34/

```

现在所有增量备份都已经成功合并到了完整备份上。再对完整备份做一次 apply-log。

```
[root@node2 databak]# innobackupex --apply-log 2017-12-04_17-07-33/

```

可以通过查看完整备份中的 `xtrabackup_checkpoints` LSN 来确认

```
[root@node2 databak]# cat 2017-12-04_17-07-33/xtrabackup_checkpoints 
backup_type = full-prepared
from_lsn = 0
to_lsn = 1628922
last_lsn = 1628922
compact = 0
recover_binlog_info = 0

```

将他复制到数据目录中

```
[root@node2 databak]# innobackupex --copy-back 2017-12-04_17-07-33/

```

修改数据文件所有者与所属组为 mysql

```
[root@node2 ~]# chown -R mysql:mysql /var/lib/mysql/


```

启动 MariaDB 服务

```
[root@node2 ~]# systemctl start mariadb


```

登录 MariaDB 服务查看数据

```
MariaDB [(none)]> SELECT  * FROM stuinfo.student;
+------+--------+
| s_id | s_name |
+------+--------+
|    1 | Stu01  |
|    2 | Stu02  |
|    3 | Stu03  |
|    4 | Stu04  |
|    5 | Stu05  |
|    6 | Stu06  |
|    7 | Stu07  |
+------+--------+


```

可以看到第 8 条数据因为没有在增量备份中，所有还原回来的数据中只有 7 条数据。现在通过二进制日志来模拟做时间点还原。

通过查看最后一次增量备份中的 `xtrabackup_binlog_info` 来确认二进制日志的位置。

```
[root@node2 ~]# cat databak/2017-12-04_17-16-34/xtrabackup_binlog_info 
mariadb-binlog.000001	3527


```

可以看到最后一次增量备份时二进制日志结束位置在 mariadb-binlog.000001 这个文件的 3527。所以现在需要从 3527 这个位置开始读取二进制日志，然后对他做 replay。

```
[root@node2 ~]# mysqlbinlog --start-position 3527 mariadb-binlog.000001 > /tmp/binlog.sql


```

临时将他导出到 /tmp 下是为了让 Mariadb 能有权限读取这个 sql 脚本

登录 MariaDB 后在会话级别关闭二进制日志记录（在 replay 二进制日志数据时是没必要在记录二进制日志的）

```
MariaDB [(none)]> SET @@SESSION.sql_log_bin = OFF;


```

导入 sql 脚本

```
MariaDB [(none)]> SOURCE /tmp/binlog.sql


```

现在数据就已经全部恢复完成了

```
MariaDB [(none)]> SELECT * FROM stuinfo.student;
+------+--------+
| s_id | s_name |
+------+--------+
|    1 | Stu01  |
|    2 | Stu02  |
|    3 | Stu03  |
|    4 | Stu04  |
|    5 | Stu05  |
|    6 | Stu06  |
|    7 | Stu07  |
|    8 | Stu08  |
+------+--------+


```

到此一次完整的备份就完成了。

Percona XtraBackup 其他的一些高级功能可以通过查看官方文档。



END!