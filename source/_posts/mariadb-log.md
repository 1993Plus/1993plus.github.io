---
title: MariaDB 日志类型
date: 2017-11-28 22:08:57
category: MariaDB
tags: 
	- mariadb
	- mariadb-log
---

**MariaDB 日志**

**查询日志**

主要记录查询操作

可以记录在文件中，也可以记录在表中。默认设置 general_log 为关闭的。文件名为当前主机名以".log"结尾。存储在相对路径（数据库路径）中。

查询是否开启，值为 “0” 表示关闭状态，为 “1” 表示开启。

```
MariaDB [(none)]> SELECT @@GLOBAL.general_log;
+----------------------+
| @@GLOBAL.general_log |
+----------------------+
|                    0 |
+----------------------+
```

临时设置为开启状态

```
MariaDB [(none)]> SET @@GLOBAL.general_log = ON;
```

在终端中修改的设置都只是临时有效，因为只要重启 Mariadb 服务器，他又会还原为默认状态。如果需要永久生效，需要在配置文件 “my.cnf” 中修改

```
general_log = ON
```

可以可以使用 SHOW 命令来查询

```
MariaDB [(none)]> SHOW GLOBAL VARIABLES LIKE '%general_log%';
+------------------+---------------+
| Variable_name    | Value         |
+------------------+---------------+
| general_log      | ON            |
| general_log_file | localhost.log |
+------------------+---------------+
```

对于记录在文件中还是表中通过 log_output 来设置，可以设置为 “FILE”,"TABLE"，也可以设置 "FILE,TABLE" 表示在文件和表中都存储，也可以设置 NONE 表示不存储日志。

```
MariaDB [(none)]> SELECT @@GLOBAL.log_output;
+---------------------+
| @@GLOBAL.log_output |
+---------------------+
| FILE                |
+---------------------+
```

修改为 “FILE,TABLE”，赋值时需要加引号。

```
MariaDB [(none)]> SET @@GLOBAL.log_output = 'FILE,TABLE';

MariaDB [(none)]> SELECT @@GLOBAL.log_output;
+---------------------+
| @@GLOBAL.log_output |
+---------------------+
| FILE,TABLE          |
+---------------------+
```

这样查询日志就可以同时记录到文件和表中了，可以通过查看 "mysql.general_log" 表查询

```
MariaDB [(none)]> SELECT * FROM mysql.general_log;
```

一般在生产环境中不建议开启该设置，因为记录他并没有什么意义，并且会产生大量的 IO。

**慢查询日志**

当执行查询时因为各种原因，比如出现死锁等导致查询时长超出指定时长的操作。

这个时长由 `long_query_time` 变量决定，以秒计算。

```
MariaDB [(none)]> SELECT @@long_query_time;
+-------------------+
| @@long_query_time |
+-------------------+
|         10.000000 |
+-------------------+
```

修改该参数的值

```
MariaDB [(none)]> SET @@GLOBAL.long_query_time = 60; 

MariaDB [(none)]> SELECT @@GLOBAL.long_query_time;
+--------------------------+
| @@GLOBAL.long_query_time |
+--------------------------+
|                60.000000 |
+--------------------------+
```

默认设置该日志记录也是关闭状态的，由 `slow_query_log` 变量控制

```
MariaDB [(none)]> SELECT @@GLOBAL.slow_query_log;
+-------------------------+
| @@GLOBAL.slow_query_log |
+-------------------------+
|                       0 |
+-------------------------+
```

如果使用的是 MariaDB 5.6 以前的版本，这个开关是由 `log_slow_queries` 这个变量控制的，如果是 5.5 版本，那这两个变量会同时存在。

临时开启

```
MariaDB [(none)]> SET @@GLOBAL.slow_query_log = ON;
```

存放路径由 `slow_query_log_file` 变量控制

```
MariaDB [(none)]> SELECT @@GLOBAL.slow_query_log_file;
+------------------------------+
| @@GLOBAL.slow_query_log_file |
+------------------------------+
| localhost-slow.log           |
+------------------------------+
```

可以将他的存放位置修改为其它路径，也可以设置为 "TABLE" 将他保存到 mysql.slow_log 表中

```
MariaDB [(none)]> SET @@GLOBAL.slow_query_log_file = '/tmp/localhost-slow.log';
```

> 其它参数

log_slow_filter 慢查询日志过滤器（需要记录的日志类型），

默认值：admin,filesort,filesort_on_disk,full_join,full_scan,query_cache,query_cache_miss,tmp_table,tmp_table_on_disk

log_slow_rate_limit 记录速率
log_slow_verbosity 详细级别

**错误日志**

会记录 MariaDB 服务启动与关闭过程中输出的事件信息，运行过程中产生的错误信息，event scheduler 运行一个 event 时产生的日志信息，主从复制架构中的从服务器上启动从服务器线程时产生的信息。

日志文件保存路径由 `log_error` 变量控制。

```
MariaDB [(none)]> SELECT @@GLOBAL.log_error;
+------------------------------+
| @@GLOBAL.log_error           |
+------------------------------+
| /var/log/mariadb/mariadb.log |
+------------------------------+
```

控制警告信息的输出由 `log_warnings` 变量控制，开启后他会将警告信息也记录到错误日志文件中，“1” 表示要记录 “0” 表示不记录。

```
MariaDB [(none)]> SELECT @@GLOBAL.log_warnings;
+-----------------------+
| @@GLOBAL.log_warnings |
+-----------------------+
|                     1 |
+-----------------------+
```



**二进制日志**

二进制日志（Binray log）这种日志对于 MySQL 来说非常重要，他在主从复制，数据恢复中会用到。通过重放（replay）日志文件中的事件来生成数据副本。

要用来记录会导致数据改变的 SQL 语句，他在文件中是以二进制格式存储。

> 记录格式

* 基于 SQL “语句” 的记录，STATEMENT

每一条会修改数据的 SQL 语句都会记录到 二进制日志中。slave 在复制的时候对 SQL 进程会解析成和 master 执行过的相同的 SQL 语句再次执行

优点：在 STATEMENT 模式下，他解决了 ROW 模式的缺点，不需要记录每一条数据的变化，减少了日志量，因为他只需要记录执行的语句的细节，以及执行语句的时候的上下文信息。

缺点：在 STATEMENT 模式下，由于他是记录的执行语句，所以，为了让这些语句能正确执行，那么他必须要记录每条语句在执行的时候的一些相关的上下文信息，以保证所有语句在执行的时候能够得到相同的结果。在使用一些特定的函数时，不能保证恢复的结果与原有值一直，比如 `now()` 函数。

* 基于 “行” 记录，ROW，日志中会记录成每一行数据被修改的形式，如果存在 slave 那 slave 也对相同的数据执行相同操作

优点：在 ROW 模式下，二进制日志中可以不记录执行的 SQL 语句的上下文相关的信息，只需要记录哪一条记录被修改了，修改成什么样了。所有 ROW 的日志内容会非常清楚的记录每一条记录修改的细节。并且不会出现某些特定情况下的存储过程或存储函数，以及 trigger 的调用和触发无法被正确复制的问题。

缺点：在 ROW 模式下，所有的执行语句当记录到日志中的时候，都将以每行记录的修改来记录，这样可能会产生大量的日志内容。

* 混合模式，MIXED，让系统自行判断该基于 STATEMENT 还是 ROW 模式进行存储日志。比如一般的插入语句就使用 STATEMENT 模式存储日志，遇到特定的函数时就使用 ROW 模式存储。



> 二进制日志文件的构成

两类文件

日志文件：mysql-bin.文件名后缀，二进制格式

索引文件：mysql_bin.index，文本格式，用来保存正在使用中的二进制日志文件列表。

> 相关变量

log_bin：用于控制是否开启二进制日志，如果设置为 “OFF” 表示不记录二进制日志，这个变量在会话中时为只读变量，是无法修改的，如果需要开启，那需要修改 MariaDB 配置文件，设置的值为二进制日志文件的前缀，而不是直接复制为 “ON”，如果直接赋值为 “ON” 那你会发现二进制日志文件名变成了"ON.000001" 类似的格式。

```
MariaDB [(none)]> SET @@GLOBAL.log_bin = 1;
ERROR 1238 (HY000): Variable 'log_bin' is a read only variable
```

例如将二进制日志文件名设置为 “master-bin” 为前缀的格式，修改 MariaDB 配置文件

```
log_bin = master-bin
```

重启 MariaDB 服务就能看到二进制日志了，使用 `SHOW BINARY LOGS`或者 `SHOW MASTER LOGS` 查看。

```
MariaDB [(none)]> SHOW BINARY LOGS;
+-------------------+-----------+
| Log_name          | File_size |
+-------------------+-----------+
| master-bin.000001 |       245 |
+-------------------+-----------+
```

也可以直接在 index 文件中查看

```
[root@localhost mysql]# cat master-bin.index 
./master-bin.000001
```

查看当前正在使用的日志文件

```
MariaDB [(none)]> SHOW MASTER STATUS;
```

要查看日志内容需要使用 mysqlbinlog 工具，或者在 mysql 客户端工具中使用命令查看

```
SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count]
```

如果命令后面没有给定日志文件名，那他会把第一个日志文件到最后一个日志文件全部打印一遍

`IN 'log_name'` ：给定要查看的日志文件名

`FROM post` ：给定要查看的日志的起始位置

`LIMIT offset` ：给定查看日志的偏移量

`LIMIT row_count` ：给定查看的行数

```
MariaDB [(none)]> SHOW BINLOG EVENTS IN 'master-bin.000001';
```



sql_log_bin：用于控制当前会话的二进制日志开关。当我们在使用二进制日志进行重放恢复数据时，是没有必要再记录二进制日志的，那就可以将该变量设置为 “OFF” 临时关闭二进制日志。

binlog_format ：二进制文件记录格式，（STATEMENT|ROW|MIXED） 

max_binlog_size：单个二进制文件大小的最大值（单位为字节），当接近该值时日志将自动滚动。

max_binlog_stmt_cache_size：用于缓存那些事务中非事务表产生的 SQL 语句的最大缓存。	

binlog_cache_size：日志缓存大小

sync_binlog：设置为 “0” 表示系统来控制什么时候将日志同步到磁盘上，如果出现突然宕机，可能会导致日志记录不完整，安全性比较差。如果设置为 "1" ，那每次提交一个事务就会立即将日志同步到磁盘上的日志文件中，为 “2” 时每提交两个事务才会同步一次，以此类推，设置的值越到可能丢失的记录就越多。

expire_logs_days：日志过期时间


