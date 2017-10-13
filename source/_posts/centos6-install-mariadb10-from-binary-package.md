---
title: CentOS6 使用二进制程序包安装 Mariadb
date: 2017-10-11 14:34:35
category: MySQL(MariaDB)
tags:
	- mariadb
	- 安装mariadb
---

CentOS6 使用二进制包安装 MariaDB 10

到 MariaDB 官方站点下载包 （https://downloads.mariadb.org/mariadb/+releases/）

```sh
wget http://mirrors.neusoft.edu.cn/mariadb//mariadb-10.2.9/bintar-linux-x86_64/mariadb-10.2.9-linux-x86_64.tar.gz
```

将包解压到 `/usr/local` 目录下

```sh
tar xf mariadb-10.2.9-linux-x86_64.tar.gz -C /usr/local/
```

对解压出来的目标目录创建 `mysql` 软连接

```sh
cd /usr/local/
ln -sv mariadb-10.2.9-linux-x86_64/ mysql
```

创建数据库存放目录

```sh
mkdir /local/mydata
```

创建 MariaDB 运行进程用户

```sh
; home 目录指定为数据库存放目录
useradd mysql -r -m -d /local/mydata/ -s /sbin/nologin
```

初始化数据库

```sh
cd mysql/
./scripts/mysql_install_db --datadir=/local/mydata/ --user=mysql
```

创建配置文件存放目录

```sh
mkdir /etc/mysql
```

拷贝配置文件到 `/etc/mysql` 目录下

```sh
cd support-files/
cp my-huge.cnf /etc/mysql/my.cnf
```

修改配置文件

```sh
vim /etc/mysql/my.cnf
;添加以下三行
;指定数据库存放目录
datadir = /local/mydata
;使用 innodb 引擎
innodb_file_per_table = ON
;跳过数据库连接地址反向解析
skip_name_resolv = ON
```

创建日志文件并将日志文件所有者改为 mysql 用户

```sh
touch /var/log/mysqld.log
chown mysql /var/log/mysqld.log
```

将 MariaDB 二进制程序路径添加到 PATH 变量中

```sh
vim /etc/profile.d/env.sh
export PATH=/usr/local/mysql/bin/:$PATH
```

读取该脚本

```sh
source /etc/profile.d/env.sh
```

拷贝服务控制脚本

```sh
cp mysql.server /etc/init.d/mysqld
```

添加到开机启动项

```sh
chkconfig mysqld on
```

启动 MariaDB 服务

```sh
service mysqld start
```

初始化数据库

```sh
mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

;输入 MariaDB root 用户当前密码，初始为空，直接回车
Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

;设置 root 密码
Set root password? [Y/n] y
;输入密码
New password: 
;再次输入
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

;移除匿名用户
Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

;禁止 root 用户远程登录
Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

;移除 test 数据库
Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

;重新载入权限设置
Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

登录 Mariadb

```sh
mysql -uroot -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 19
Server version: 10.2.9-MariaDB-log MariaDB Server

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```



END!