---
title: 编译安装 Zabbix 3.4.3
date: 2017-10-25 11:35:43
category: Zabbix
tags:
	- zabbix
	- monitor
	- 监控
---

**CentOS7.4 编译安装 Zabbix 3.4.3**

**环境**

zabbix server ip: 192.168.1.23

mariadb server ip: 192.168.1.22

web server ip: 192.168.1.21

**Zabbix server**

安装依赖软件包

```sh
yum install -y gcc-c++ mysql++-devel libxml2-devel net-snmp-devel libevent-devel curl-devel
```

到官方网站下载源码包(https://www.zabbix.com/download)

```sh
;切换目录将源码包下载到 /usr/local/src 目录下(非必要)
cd /usr/local/src/
;下载源码包
wget https://nchc.dl.sourceforge.net/project/zabbix/ZABBIX%20Latest%20Stable/3.4.3/zabbix-3.4.3.tar.gz
```

加压源码包

```sh
tar xf zabbix-3.4.3.tar.gz
```

进入解压目录

```sh
cd zabbix-3.4.3/
```

执行编译参数

```sh
./configure \
--prefix=/usr/local/zabbix \
--enable-server \
--enable-agent \
--with-mysql \
--enable-ipv6 \
--with-net-snmp \
--with-libcurl \
--with-libxml2

make && make install
```

创建守护进程运行账户

```sh
groupadd zabbix -r -g 997
useradd zabbix1 -r -u 995 -g zabbix -s /sbin/nologin
```

创建日志存放目录

```sh
mkdir /var/log/zabbix
;将日志存放目录所有者改为 zabbix 用户，或给 zabbix 用户写(w)权限
chown zabbix /var/log/zabbix
```

修改 `zabbix_server` 配置文件

```sh
vim /usr/local/zabbix/etc/zabbix_server.conf

;日志存放路径
LogFile=/var/log/zabbix/zabbix_server.log
;数据库服务器地址
DBHost=192.168.1.22
;连接的数据库名
DBName=zabbix
;登录数据库的用户名
DBUser=zabbix
;数据库用户密码
DBPassword=zabbix
;数据库监听端口
DBPort=3306
;Zabbix server 监听端口
ListenIP=192.168.1.23
;守护进程运行用户
User=zabbix
```

修改 `zabbix_agentd` 配置文件

```sh
vim /usr/local/zabbix/etc/zabbix_agentd.conf

;日志存放路径
LogFile=/var/log/zabbix/zabbix_agentd.log
;zabbix server 地址
Server=192.168.1.23
;zabbix agentd 守护进程运行用户
User=zabbix
```

拷贝服务启动脚本

```sh
;在源码包目录中
cp /usr/local/src/zabbix-3.4.3/misc/init.d/fedora/core5/* /etc/init.d/
```

修改脚本

```sh
vim /etc/init.d/zabbix_server
;修改 zabbix server 二进制程序路径为实际路径
ZABBIX_BIN="/usr/local/zabbix/sbin/zabbix_server"
-------------------------------------------------------
;修改 zabbix agentd 二进制程序路径为实际路径
vim /etc/init.d/zabbix_agentd
ZABBIX_BIN="/usr/local/zabbix/sbin/zabbix_agentd"
```

添加为开机自启动

```sh
chkconfig zabbix_server on
chkconfig zabbix_agentd on
```



**数据库服务器**

MariaDB 二进制方式安装：http://www.weidong.io/2017/10/11/centos6-install-mariadb10-from-binary-package/

将源码包中的 sql 文件拷贝到 数据库服务器上

```sh
cd /usr/local/src/zabbix-3.4.3/database/
;拷贝到了 root 用户 home 目录
scp -r mysql/ root@192.168.1.22:
```

创建数据库

```mysql
CREATE DATABASE zabbix CHARACTER SET 'utf8';
```

创建数据库用户并授权

```mysql
GRANT ALL ON zabbix.* TO 'zabbix'@'192.168.1.%' IDENTIFIED BY 'zabbix';
```

导入数据库

```sh
mysql -u zabbix -p -h 192.168.1.22 zabbix < ~/mysql/schema.sql
mysql -u zabbix -p -h 192.168.1.22 zabbix < ~/mysql/images.sql
mysql -u zabbix -p -h 192.168.1.22 zabbix < ~/mysql/data.sql
```

启动 zabbix 服务

```sh
systemctl start zabbix_server
systemctl start zabbix_agentd
```

**WEB 服务器**

HTTPD 2.4 安装：http://www.weidong.io/2017/10/09/centos7-source-install-httpd-2-4/

PHP FPM 安装：http://www.weidong.io/2017/10/13/centos7-install-php7-1-fpm-from-source/

拷贝源码包中的网页文件拷贝到 web 服务器工作目录

```sh
scp -r php/ root@192.168.1.21:/local/www/zabbix
```

打开浏览器输入 web 服务器地址访问

![setup1](http://ov2iiuul1.bkt.clouddn.com/zabbix_1.1.png)

参数检测，根据提示修改配置文件

![setup2](http://ov2iiuul1.bkt.clouddn.com/zabbix_2.1.png)

输入数据库地址，端口，数据库名，数据库用户，数据库密码

![setup3](http://ov2iiuul1.bkt.clouddn.com/zabbix_3.1.png)

输入 zabbix server 服务端地址，端口，名称

![setup4](http://ov2iiuul1.bkt.clouddn.com/zabbix_4.1.png)

确认参数

![setup5](http://ov2iiuul1.bkt.clouddn.com/zabbix_5.1.png)

点击 Finish 完成

![setup6](http://ov2iiuul1.bkt.clouddn.com/zabbix_6.1.png)

登录(用户名“Admin”，密码“zabbix”)

![setup7](http://ov2iiuul1.bkt.clouddn.com/zabbix_7.1.png)



**`登录后可能会显示服务端没有运行，只需要重新运行一下 zabbix 服务端就可以了`**



END!