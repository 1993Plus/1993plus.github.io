---
title: CentOS7.3 编译安装 PHP7.1 FPM
date: 2017-10-13 20:20:49
category: PHP
tags:
	- php7.1
	- 编译安装
	- php-fpm
---

**CentOS7.3 编译安装 PHP 7.1 FPM 模式**

拓扑结构

![拓扑结构](http://ov2iiuul1.bkt.clouddn.com/php_arch.png)

安装必要依赖包

```sh
yum install -y wget gcc-c++ systemd-devel libxml2-devel openssl-devel libcurl-devel libjpeg-devel libpng-devel freetype-devel libmcrypt-devel openldap-devel
```

创建 php 网页程序文件存放目录

```sh
mkdir /local/www/
```

创建 php 守护进程用户与组，并将其 `home` 目录指向 php 网页程序文件存放目录

```sh
 groupadd php-fpm -r -g 502
 useradd php-fpm -r -u 502 -g 502 -m -d /local/www/ -s /sbin/nologin
```

下载源码包到 `/usr/local/src` 目录下

```sh
cd /usr/local/src/
wget http://docs.php.net/distributions/php-7.1.10.tar.xz
```

解压源码包并切换进该目录

```sh
tar xf php-7.1.10.tar.xz
cd php-7.1.10
```

*`FAQ`*

如果不需要 `ldap` 模块可以取消，因为在编译过程中可能会出现两个错误 

1) 提示找不到 `ldap` 库文件，解决办法就是将 `/usr/lib64/` 目录下的 `libldap` 库文件拷贝到 `/usr/lib/` 目录下

```sh
cp -a /usr/lib64/libldap* /usr/lib
```

2) 进行编译时系统抛出错误提示

```sh
/usr/bin/ld: ext/ldap/.libs/ldap.o: undefined reference to symbol 'ber_scanf'
/usr/lib64/liblber-2.4.so.2: error adding symbols: DSO missing from command line
collect2: error: ld returned 1 exit status
make: *** [sapi/cli/php] Error 1
```

解决办法编辑 `Makefile` (`vim Makefile`) 文件，找到 `EXTRA_LIBS =` 字段，在尾部追加 `-llber` 保存退出，继续执行 `make && meke install` 

编译参数

```sh
./configure \
--prefix=/local/php7 \
--enable-fpm \
--with-config-file-path=/etc \
--with-config-file-scan-dir=/etc/php.d \
--with-fpm-user=php-fpm \
--with-fpm-group=php-fpm \
--with-mcrypt --with-mhash \
--with-openssl \
--enable-mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd --with-gd \
--with-iconv \
--with-zlib \
--enable-zip \
--enable-inline-optimization \
--disable-debug \
--disable-rpath \
--enable-shared \
--enable-xml \
--enable-bcmath \
--enable-shmop \
--with-fpm-systemd \
--enable-mbregex \
--enable-mbstring \
--enable-gd-native-ttf \
--enable-pcntl \
--enable-sockets \
--with-xmlrpc \
--enable-soap \
--without-pear \
--with-gettext \
--enable-session --with-curl \
--with-jpeg-dir \
--with-freetype-dir \
--enable-opcache \
--without-gdbm \
--enable-maintainer-zts \
--with-ldap \
--disable-fileinfo

make && make install
```

拷贝配置 ` php.ini` 文件

```sh
cp php.ini-production /etc/php.ini
```

拷贝并修改 `php fpm` 配置文件

```sh
cp /usr/local/php7/etc/php-fpm.conf.default /etc/php-fpm.conf
;修改最后一行
include=/etc/php-fpm.d/*.conf
```

创建连接池配置文件目录

```sh
mkdir /etc/php-fpm.d/
```

拷贝并修改连接池配置文件

```sh
cp /usr/local/php7/etc/php-fpm.d/www.conf.default /etc/php-fpm.d/www.conf
;监听 IP 与端口,如果是监听在所有 IP 上只填端口
listen = 9000
;允许连接的 IP
listen.allowed_clients = 192.168.1.101
```

拷贝服务控制脚本

```sh
cp /usr/local/src/php-7.1.10/sapi/fpm/php-fpm.service /etc/systemd/system/
```

修改服务控制脚本

```sh
vim /etc/systemd/system/php-fpm.service
;将路径改为实际路径
[Unit]
Description=The PHP FastCGI Process Manager
After=network.target

[Service]
Type=notify
PIDFile=/usr/local/php7/var/run/php-fpm.pid
ExecStart=/usr/local/php7/sbin/php-fpm --nodaemonize --fpm-config /etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

修改执行权限

```sh
chmod +x /etc/systemd/system/php-fpm.service
```

加入开机自启动

```sh
systemctl enable php-fpm
```

启动服务

```sh
systemctl start php-fpm
```

修改 httpd 配置

```sh
;添加主页文件
DirectoryIndex index.php
;添加 php 支持
AddType application/x-httpd-php .php
AddType application/x-httpd-php-source .phps
;关闭正向代理
Proxyrequests Off
;开启 FCGI 反向代理
ProxyPassMatch ^/(.*\.php)$ fcgi://192.168.1.101:9000/var/www/php/$1
```

重启 httpd 服务

```
systemctl restart httpd
```

在 PHP 服务器 `/var/www/php` 目录下添加 测试页面

```sh
vim /var/www/php/index.php

<?php
phpinfo();
?>
```

在浏览器地址栏输入 httpd 服务器 IP 地址访问测试（本次测试的 httpd 服务器地址是 192.168.1.100 ）

![测试页面](http://ov2iiuul1.bkt.clouddn.com/php_test1.png)



END!