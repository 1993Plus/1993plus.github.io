---
title: CentOS6 源码安装 httpd 2.4
date: 2017-10-10 20:36:03
category: HTTPD
tags:
	- httpd2.4
	- centos6
---

CentOS 6 编译安装 httpd 2.4

在 CentOS 6 系统上 yum 仓库是没有提供 httpd 2.4 ，只有 2.2 版本的，这时如果要想在 CentOS 6 系统上使用 httpd 2.4 就只有自己编译安装了，在编译安装 httpd 2.4 时又会遇到一个问题，因为 httpd 是依赖于 apr，apr-util 这两个运行库的，在 CentOS 6 yum 仓库中提供的这两个库版本安装包是 1.3.9 的，而 httpd 2.4 依赖这两个库的版本是 1.4 .0 以上的，所以这两个库也要自行解决。

`关闭 SELinux`

`关闭 iptables`

到 Apache 官方网站下载 apr，apr-util 的源码包（http://apr.apache.org/）

```sh
;将两个包放到 /usr/local/src/ 目录下
cd /usr/local/src/
wget http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-1.6.2.tar.gz
wget http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-util-1.6.0.tar.gz
```

下载 https 2.4 源码包，本次测试下载版本为`httpd-2.4.28`  (http://httpd.apache.org/)

```sh
wget http://mirrors.shuosc.org/apache//httpd/httpd-2.4.28.tar.gz
```

解压 httpd 源码包

```sh
tar xf httpd-2.4.28.tar.gz
```

将 apr 源码包解压，并移动改名放到` httpd-2.4.28/srclib` 目录下

```sh
mv apr-1.6.2 httpd-2.4.28/srclib/apr
```

将 apr-util 源码包解压，并移动改名放到 `httpd-2.4.28/srclib` 目录下

```sh
mv apr-util-1.6.0 httpd-2.4.28/srclib/apr-util
```

进入 `httpd-2.4.28` 目录内开始执行编译安装

```sh
cd httpd-2.4.28/
;编译参数
./configure \
--prefix=/usr/local/httpd24 \
--enable-so \
--enable-cgi \
--with-zlib \
--with-pcre \
--enable-proxy \
--enable-ssl \
--enable-modules=most \
--enable-mpms-shared=all \
--enable-rewrite \
--with-included-apr \
--with-mpm=prefork

make && make install
```

创建守护进程运行用户

```sh
;将 home 目录指向 httpd documentroot 目录，当然也可以修改配置文件来更改一个目录
useradd apache -r -m -d /usr/local/httpd24/htdocs/ -s /sbin/nologin
```

修改 httpd 配置文件，将运行用户改为 apache

```sh
vim /usr/local/httpd24/conf/httpd.conf
User apache
Group apache
```

创建服务控制脚本

```sh
vim /etc/init.d/httpd

#!/bin/bash
#
# httpd        Startup script for the Apache HTTP Server
#
# chkconfig: - 85 15
# description: The Apache HTTP Server is an efficient and extensible  \
#              server implementing the current HTTP standards.
# processname: httpd
# config: /etc/httpd/conf/httpd.conf
# config: /etc/sysconfig/httpd
# pidfile: /var/run/httpd/httpd.pid
#
### BEGIN INIT INFO
# Provides: httpd
# Required-Start: $local_fs $remote_fs $network $named
# Required-Stop: $local_fs $remote_fs $network
# Should-Start: distcache
# Short-Description: start and stop Apache HTTP Server
# Description: The Apache HTTP Server is an extensible server 
#  implementing the current HTTP standards.
### END INIT INFO

# Source function library.
. /etc/rc.d/init.d/functions

if [ -f /etc/sysconfig/httpd ]; then
        . /etc/sysconfig/httpd
fi

# Start httpd in the C locale by default.
HTTPD_LANG=${HTTPD_LANG-"C"}

# This will prevent initlog from swallowing up a pass-phrase prompt if
# mod_ssl needs a pass-phrase from the user.
INITLOG_ARGS=""

# Set HTTPD=/usr/sbin/httpd.worker in /etc/sysconfig/httpd to use a server
# with the thread-based "worker" MPM; BE WARNED that some modules may not
# work correctly with a thread-based MPM; notably PHP will refuse to start.

# Path to the apachectl script, server binary, and short-form for messages.
apachectl=/usr/local/httpd24/bin/apachectl
httpd=${HTTPD-/usr/local/httpd24/bin/httpd}
prog=httpd
pidfile=${PIDFILE-/usr/local/httpd24/logs/httpd.pid}
lockfile=${LOCKFILE-/var/lock/subsys/httpd}
RETVAL=0
STOP_TIMEOUT=${STOP_TIMEOUT-10}

# The semantics of these two functions differ from the way apachectl does
# things -- attempting to start while running is a failure, and shutdown
# when not running is also a failure.  So we just do it the way init scripts
# are expected to behave here.
start() {
        echo -n $"Starting $prog: "
        LANG=$HTTPD_LANG daemon --pidfile=${pidfile} $httpd $OPTIONS
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && touch ${lockfile}
        return $RETVAL
}

# When stopping httpd, a delay (of default 10 second) is required
# before SIGKILLing the httpd parent; this gives enough time for the
# httpd parent to SIGKILL any errant children.
stop() {
        status -p ${pidfile} $httpd > /dev/null
        if [[ $? = 0 ]]; then
                echo -n $"Stopping $prog: "
                killproc -p ${pidfile} -d ${STOP_TIMEOUT} $httpd
        else
                echo -n $"Stopping $prog: "
                success
        fi
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && rm -f ${lockfile} ${pidfile}
}

reload() {
    echo -n $"Reloading $prog: "
    if ! LANG=$HTTPD_LANG $httpd $OPTIONS -t >&/dev/null; then
        RETVAL=6
        echo $"not reloading due to configuration syntax error"
        failure $"not reloading $httpd due to configuration syntax error"
    else
        # Force LSB behaviour from killproc
        LSB=1 killproc -p ${pidfile} $httpd -HUP
        RETVAL=$?
        if [ $RETVAL -eq 7 ]; then
            failure $"httpd shutdown"
        fi
    fi
    echo
}

# See how we were called.
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  status)
        status -p ${pidfile} $httpd
        RETVAL=$?
        ;;
  restart)
        stop
        start
        ;;
  condrestart|try-restart)
        if status -p ${pidfile} $httpd >&/dev/null; then
                stop
                start
        fi
        ;;
  force-reload|reload)
        reload
        ;;
  graceful|help|configtest|fullstatus)
        $apachectl $@
        RETVAL=$?
        ;;
  *)
        echo $"Usage: $prog {start|stop|restart|condrestart|try-restart|force-reload|reload|status|fullstatus|graceful|help|configtest}"
        RETVAL=2
esac

exit $RETVAL
```

添加执行权限

```sh
chmod +x /etc/init.d/httpd
```

添加到开机启动项中

```sh
chkconfig httpd on
```

启动服务

```sh
service httpd start
```

访问测试

```sh
curl 127.0.0.1
<html><body><h1>It works!</h1></body></html>
```



END!