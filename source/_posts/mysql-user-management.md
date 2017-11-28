---
title: MariaDB 用户管理
date: 2017-11-26 21:16:19
category: MariaDB
tags:
	- mysql
	- mariadb
---



**MariaDB 用户名格式**

MySQL 的用户由 `user@host` 组成，user 为用户名 host 可以是一个具体的 IP 、主机名，表示该用户可以通过哪些客户端主机请求创建连接线程。对于一个网络的授权可以使用通配符来匹配，在 MariaDB 中 `%` 用来匹配任意字符，`_` 用来匹配任意单个字符。

例如要对一个网络授权

```
user@192.168.0.%
```

表示如果 host 通过 IP 地址前缀为 192.168.0 开头，后跟任意地址的客户端主机都可以请求创建连接线程。



**创建用户**

创建用户使用 `CREATE USER` 命令

```
CREATE USER 'user'@'host' [IDENTIFIED BY [PASSWORD] 'password']
```

在创建用户时密码是一个可选项，也就是不给密码也可以创建用户。

```
MariaDB [(none)]> CREATE USER 'testuser'@'192.168.0.1';
```



**重命名**

重命名用户使用 `RENAME USER` 命令

```
RENAME USER old_user TO new_user
```

需要注意的是重命名用户时也需要跟上主机名

```
MariaDB [(none)]> RENAME USER 'testuser'@'192.168.0.1' TO 'testu'@'192.168.0.%';
```



**删除用户**

删除用户使用 `DROP USER` 命令

```
DROP USER user [, user] ...
```

需要注意的是重命名用户时也需要跟上主机名

```
MariaDB [(none)]> DROP USER 'testu'@'192.168.0.%';
```



**修改用户密码**

修改用户密码有三种方法，可通过 `SET PASSWORD` 命令、直接修改 `mysql.user` 表、使用 `mysqladmin` 工具修改

使用 `SET PASSWORD` 命令修改

语法格式

```
SET PASSWORD [FOR user] =
    {
        PASSWORD('cleartext password')
      | OLD_PASSWORD('cleartext password')
      | 'encrypted password'
    }
```

示例

```
MariaDB [(none)]> SET PASSWORD FOR 'testu'@'192.168.1.%' = PASSWORD ('123456');
```

通过修改 mysql.user 表

```
UPDATE mysql.user SET Password = PASSWORD ('123456') WHERE User = 'testu';
```

如果改用户属于多个主机，但只想修改某个主机下的用户，那么在更新条件中还需要对主机进行限制

```
UPDATE mysql.user SET Password = PASSWORD ('123456') WHERE User = 'testu' AND Host = '192.168.1.%';
```

通过 mysqladmin 工具修改

```
mysqladmin -uroot -p password 'newpass'
Enter password: 输入旧密码
```

`通过修改表更改了密码需要使用  FLUSH PRIVIEGES 来刷新授权表`

**用户授权**

虽然用户已经创建好了，但他除了登录几乎不能做任何操作

查看其他用户权限

```
MariaDB [(none)]> SHOW GRANTS FOR 'testu'@'192.168.1.%'\G;
*************************** 1. row ***************************
Grants for testu@192.168.1.%: GRANT USAGE ON *.* TO 'testu'@'192.168.1.%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9'
```

对于 test@192.168.1.% 这个用于，由于没有对他进行授权，他现在只有一个 USAGE 的权限。

如果要查看当前登录用户的权限

```
MariaDB [(none)]> SHOW GRANTS FOR CURRENT_USER();
+---------------------------------------------------------------------+
| Grants for root@localhost                                           |
+---------------------------------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION |
| GRANT PROXY ON ''@'' TO 'root'@'localhost' WITH GRANT OPTION        |
+---------------------------------------------------------------------+
```

授权使用 `GRANT` 命令

命令格式

```
GRANT
    priv_type [(column_list)]
      [, priv_type [(column_list)]] ...
    ON [object_type] priv_level
    TO user_specification [, user_specification] ...
    [REQUIRE {NONE | ssl_option [[AND] ssl_option] ...}]
    [WITH with_option ...]

GRANT PROXY ON user_specification
    TO user_specification [, user_specification] ...
    [WITH GRANT OPTION]

object_type:
    TABLE
  | FUNCTION
  | PROCEDURE

priv_level:
    *
  | *.*
  | db_name.*
  | db_name.tbl_name
  | tbl_name
  | db_name.routine_name

user_specification:
    user
    [
        IDENTIFIED BY [PASSWORD] 'password'
      | IDENTIFIED WITH auth_plugin [AS 'auth_string']
    ]

ssl_option:
    SSL
  | X509
  | CIPHER 'cipher'
  | ISSUER 'issuer'
  | SUBJECT 'subject'

with_option:
    GRANT OPTION
  | MAX_QUERIES_PER_HOUR count
  | MAX_UPDATES_PER_HOUR count
  | MAX_CONNECTIONS_PER_HOUR count
  | MAX_USER_CONNECTIONS count
```

参数有很多

*priv_type 表示权限类型*

库和表级别

> CREATE, ALTER, DROP
>
> INDEX
>
> CREATE VIEW
>
> SHOW VIEW
>
> GRANT (可以将自己的权限生成一个副本转给其他用户，且无法被管理员收回)
>
> OPTION

数据操作

> INSERT
>
> DELETE
>
> UPDATE
>
> SELECT

字段操作

> SELECT(col1,col2,...)
>
> UPDATE(col1,col2,...)
>
> INSERT(col1,col2,...)

所有权限

> ALL
>
> ALL PRIVILEGES



例如给用户授权 `CREATE` 权限

```
MariaDB [(none)]> GRANT CREATE ON *.* TO 'testu'@'192.168.1.%';
```

这里在授权的时候并没有给 `object_type` 默认为 `TABLE` 

也可以使用 `GRANT` 命令直接创建用户并授权

```
MariaDB [(none)]> GRANT INSERT ON book.* TO 'tom'@'192.168.1.%' IDENTIFIED BY '123456';
```

创建一个一个名为 tom 的用户，给的权限为 book 数据库中所有表的 INSERT 权限



**撤回权限**

撤回权限使用 `REVOKE` 命令

命令格式

```
REVOKE
    priv_type [(column_list)]
      [, priv_type [(column_list)]] ...
    ON [object_type] priv_level
    FROM user [, user] ...

REVOKE ALL PRIVILEGES, GRANT OPTION
    FROM user [, user] ...

REVOKE PROXY ON user
    FROM user [, user] ...
```

撤回 tom 用户在 book 数据库上的 INSERT 权限

```
MariaDB [(none)]> REVOKE INSERT ON book.* FROM 'tom'@'192.168.1.%';
```

