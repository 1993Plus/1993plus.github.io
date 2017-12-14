---
title: MySQL 主从复制
date: 2017-12-14 21:33:21
category: MariaDB
tags:
	- mysql
	- mariadb
	- replication
---

##### MySQL 复制架构

 MySQL 主从复制机制：在主从（Master/Slave）架构中，主从节点都存有相同的数据集。但只有主节点接受来自客户端读写请求，其余从节点只接受读请求。从节点通过扮演成客户端的形式从主节点上复制二进制日志到本地进行 replay，实现将数据写入本地数据库中。 实现了 MySQL 向外扩展（Scale Out）的解决方案。完成了数据分布，负载均衡的功用，但只能实现读请求的负载均衡。

MySQL 主主复制机制：在主主（Master/Master）架构中，各个节点互为主从，所以所有节点都需要开启二进制日志与中继日志。但这样会面临一个问题就是，当从节点 replay 二进制日志时自己也会记录二进制日志事件，会形成一个死循环。比如 node1 与 node2 工作在主主模式下，他们互为主从，当 node1 写入数据后 node2 会向他请求二进制日志事件，node2 在 replay 二进制日志事件后自己也记录了二进制日志，当 node1 在想 node2 请求二进制日志事件时会收到他之前发送过去的事件，这样一来就会造成数据重复写入了。在 MySQL 中通过 server id 来解决该问题。 MySQL 在记录二进制日志时会将自己的 server id 记录到二进制日志事件中，当收到的二进制日志事件中的 server id 是自己是那就不对该事件进行 replay。这种架构起到了数据冗余与高可用的效果。

##### 面临的问题

在主从复制中前端调度器所面临的问题，他需要解码识别来自客户端的 SQL 请求，对 SQL 语句作精确分析，判断客户端发来的请求是读请求还是写请求。如果是写请求那么就只能发送给 Master 节点，如果是读请求就基于负载均衡的方式发送 Slave 节点。要完成这些功能前端调度器就必须要工作在七层协议上，该调度器通常被称为 R/W Spliter（读写分离器）或 SQL 语句路由器。

##### MySQL 主从复制组件

> Slave

`I/O Thread` 向 Master 请求二进制日志事件，并保存在中继日志中，并记录下已读取到的位置。

`第一次请求二进制日志信息时如果不指定读取位置，那将从第一个二进制日志的的起始位置开始读取，在数据很多的情况下应避免此操作。`

`SQL Thread` 从中继日志中读取二进制日志事件，在本地完成重放

> Master

`Dump Thread` 为每个 Slave 的 `I/O Thread` 启动一个 Dump 线程，用于向请求者发送 binary log events。



##### 级联复制

通常对于从节点来说，保存二进制日志是没有意义的，而且会增加 I/O ，所以一般在从节点没有必要开启二进制日志。在另外一种结构中从节点也可以作为其他从节点的主节点，他的从节点想要从他那儿获取到数据那他就需要开启二进制日志的记录。这种结构就叫做级联复制。在 Slave 节点较多造成 Master 节点压力过大的情况下会用到。中继节点如果不需要存储数据时只需要将默认存储引擎设置为 `BLACKHOLE` 即可。



##### 复制机制

> 异步复制

在主从复制中写入数据时 Master 节点只需要保证本地写入成功即可，而不需要知道 Slave 节点是否复制完成。

> 半同步复制

半同步复制是基于 Google 为 MySQL 所研发的插件实现的。他的实现方式是要求 Slave 节点中必须有一个 Slave 节点在复制完数据后向 Master 节点回应“复制完成”， 才返回给客户端成功的信息。对于其他 Slave 节点仍然为异步复制。



##### 主从复制配置

`在配置前将所有节点时间同步为一致`

> Master 节点配置

修改配置文件

启用二进制日志、设置全局唯一 server id

```
server_id = 1
log-bin = master-binlog
```

创建仅有复制权限的用户

创建用户

```
MariaDB [(none)]> CREATE USER 'repluser'@'172.18.39.%' IDENTIFIED BY '123456';
```

撤回该账号所有权限

```
MariaDB [(none)]> REVOKE ALL PRIVILEGES,GRANT OPTION FROM 'repluser'@'172.18.39.%';
```

重新赋予 `REPLICATION SLAVE` , `REPLICATION CLIENT` 权限

```
MariaDB [(none)]> GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO 'repluser'@'172.18.39.%';
```

刷新权限表

```
MariaDB [(none)]> FLUSH PRIVILEGES;
```

> Slave 节点配置

修改配置文件

启用中继日志、设置全局唯一 server id、

```
server_id = 2
relay-log = relay-log
```

设置 Slave 节点指向 Master 节点使用 `CHANGE MASTER TO` 命令

```
CHANGE MASTER TO
    MASTER_BIND = 'interface_name'	# 连接 Master 使用的网络接口
  | MASTER_HOST = 'host_name'	# Master 节点地址
  | MASTER_USER = 'user_name'	# 连接 Master 节点用于复制数据的用户
  | MASTER_PASSWORD = 'password'	# 用户密码
  | MASTER_PORT = port_num	# Master 监听的端口，默认为 3306
  | MASTER_CONNECT_RETRY = interval	# 连接失败重试间隔，单位为秒，默认 60 秒
  | MASTER_HEARTBEAT_PERIOD = interval	# 心跳检查间隔
  | MASTER_LOG_FILE = 'master_log_name'	# 要复制数据的二进制日志起始文件
  | MASTER_LOG_POS = master_log_pos	# 二进制日志起始位置
  | RELAY_LOG_FILE = 'relay_log_name'	# 中继日志文件
  | RELAY_LOG_POS = relay_log_pos	# 中继日志起始位置

# 使用 SSL 连接的相关参数
  | MASTER_SSL = {0|1}
  | MASTER_SSL_CA = 'ca_file_name'
  | MASTER_SSL_CAPATH = 'ca_directory_name'
  | MASTER_SSL_CERT = 'cert_file_name'
  | MASTER_SSL_KEY = 'key_file_name'
  | MASTER_SSL_CIPHER = 'cipher_list'
  | MASTER_SSL_VERIFY_SERVER_CERT = {0|1}
  
# 要忽略的 server id
  | IGNORE_SERVER_IDS = (server_id_list)
```

有很多选项，但大多数是可选的可以不用设置。选项之间使用英文逗号“,”分隔。

查看 Master 节点的二进制日志信息

```
MariaDB [(none)]> SHOW MASTER LOGS;
+----------------------+-----------+
| Log_name             | File_size |
+----------------------+-----------+
| master-binlog.000001 |       590 |
+----------------------+-----------+
```

在这个位置之前的数据都是一些 Master 节点配置产生的二进制日志事件，这些事件不需要被 Slave 节点复制那就从这个位置复制即可。

```
MariaDB [(none)]> CHANGE MASTER TO 
MASTER_HOST='172.18.39.41',
MASTER_USER='repluser',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='master-binlog.000001',
MASTER_LOG_POS=590,
MASTER_CONNECT_RETRY=10;
```

查看 Slave 状态

```
MariaDB [(none)]> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 172.18.39.41
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: master-binlog.000001
          Read_Master_Log_Pos: 590
               Relay_Log_File: relay-log.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: master-binlog.000001
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 590
              Relay_Log_Space: 245
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 0
```

现在配置完成了，但从 Slave 状态中可以看到 `Slave_IO_Running` 和 `Slave_SQL_Running` 为 NO，表示 Slave 节点现在还没有开始复制数据。

启动 IO 线程与 SQL 线程

要启动 IO 线程与 SQL 线程使用 `START SLAVE` 命令

```
START SLAVE [thread_types]
thread_type: IO_THREAD | SQL_THREAD
# 如果不知道 thread_types 则表示两个线程都启动
```

启动后 `Slave_IO_Running` 和 `Slave_SQL_Running` 将显示为 Yes ，Slave_IO_State 为等待 Master 发送事件

```
MariaDB [(none)]> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.39.41
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: master-binlog.000001
          Read_Master_Log_Pos: 590
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 533
        Relay_Master_Log_File: master-binlog.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

测试

在 Master 节点上创建一个数据库 “repltest”

```
MariaDB [(none)]> CREATE DATABASE repltest;
```

查看 Slave 节点上是否复制

```
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| repltest           |
| test               |
+--------------------+
```

可以看到 Master 节点上创建的 “repltest” 数据库已经成功复制到了 Slave 节点上了。

再次查看  Slave 状态，可以看到 “Read_Master_Log_Pos” 位置从 590 到 681

```
MariaDB [(none)]> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.39.41
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: master-binlog.000001
          Read_Master_Log_Pos: 681
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 624
        Relay_Master_Log_File: master-binlog.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

在 Slave 节点的数据目录下会生成 `master.info` 和 `relay-log.info` 两个文件，在 Slave 节点到 Master 节点请求二进制日志事件时非常重要

`master.info` 中记录了连接 Master 的信息

`relay-log.info` 记录了当前正在使用的中继日志文件与位置和从 Master 节点读取的二进制日志文件与位置

到此 MySQL 的主从配置就完成了。

##### 常见问题

- 如果配置主从复制前 Master 节点已经有很多数据了，这时使用二进制日志是没法进行复制的。应该先将 Master 节点上的数据库做完整备份，然后恢复到 Slave 节点上，在开启同步。
- 限制 Slave 节点为只读。可以在配置文件中增加 `read_only=ON` 。但该设置对 SUPER 用户无效。
- 事务安全。对于事务型存储引擎，如 InnoDB 在提交事务后二进制日志事件会在内存中缓冲一段时间后才会写入到磁盘上的二进制日志文件中。这样一来如果遇到服务器宕机内存中的二进制日志事件就会丢失，造成事件不完整，Slave 节点也就获取不到完整的数据了。为了避免这种情况的发送可以设置 `sync_binlog=ON` ，表示当产生新的二进制日志事件是立即写入二进制日志文件中。
- 对于 InnoDB 存储引擎还应该设置 `innodb_flush_logs_at_trx_commit` 为 `ON` ，表示当事务提交时立即将事务日志刷写到事务日志文件中，避免宕机造成事务丢失。设置 `innodb_support_xa` 为 `ON` 启动分布式事务提交。
- 对于 Slave 节点，可以设置 `skip_slave_start` 为 `ON` 表示启动 MySQL 服务时跳过自动启动 IO 线程与 SQL 线程，在启动服务确认 Master 节点正常后手动开启 IO 线程与 SQL 线程，避免将 Master 上的错误事件复制到 Slave 上。