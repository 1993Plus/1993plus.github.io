---
title: CentOS7 编译安装 httpd 2.4
date: 2017-10-09 21:58:57
category: HTTPD
tags:
	- httpd2.4
	- make
	- 编译安装
---

**CentOS 7 编译安装 httpd 2.4**

**创建所需要的用户与组**

```sh
useradd -r apache -s /sbin/nologin
; -r 创建系统用户
; -s 指定 shell 程序为 /sbin/nologin，不允许使用该账户登录系统
```

**安装必要依赖包**

```sh
yum install -y apr-devel apr-util-devel gcc-c++ pcre-devel openssl-devel wget
```

**下载源码包**

```sh
;切换到 /usr/local/src 下使用 wget 或其它方式将源码包放到这个目录中
cd /usr/local/src/
wget http://mirrors.shuosc.org/apache//httpd/httpd-2.4.28.tar.gz
;解压包
tar xvf httpd-2.4.28.tar.gz
;进入到解压出来的目录内
cd httpd-2.4.28/
```

**执行编译**

```sh
./configure \	;可以在这条命令后面使用 --help 查看帮助与一些编译参数
--prefix=/usr/local/httpd24 \	;编译安装路径
--enable-so \
--enable-cgi \
--with-zlib \
--with-pcre \
--enable-proxy \
--enable-ssl \
--enable-modules=most \
--enable-mpms-shared=all \
--enable-rewrite \
--with-mpm=prefork
make && make install
```

**修改守护进程运行用户与组**

```sh
vim /usr/local/httpd24/conf/httpd.conf
User apache
Group apache
```

**将 httpd 程序路径加入到 `PATH` 环境变量中**

```sh
echo export PATH=/usr/local/httpd24/bin/:$PATH > /etc/profile.d/httpd.sh
; 使用 source 重新读取改文件
source /etc/profile.d/httpd.sh
```

添加开机自启动

```sh
vim /etc/rc.d/rc.local
/usr/local/httpd24/bin/apachectl -k start
```

在 CentOS 7 中 rc.local 默认是没有执行权限的所以还要为它添加执行权限才行

```sh
chmod +x /etc/rc.d/rc.local
```

使用 `apachectl` 

```sh
apachectl -k start		;启动 httpd 服务
			 restart	;重启 httpd 服务
			 stop		;停止 httpd 服务
```

**访问测试**

```sh
curl 127.0.0.1
<html><body><h1>It works!</h1></body></html>
```



END!