---
title: httpd(2.2)相关配置文件
date: 2017-10-07 15:07:42
category: HTTPD
tags:
	- httpd
---



**httpd-2.2 相关配置文件**

主程序文件

```sh
/usr/sbin/httpd
/usr/sbin/httpd.worker
/usr/sbin/httpd.event
```

主进程文件 PID 文件

```sh
/etc/httpd/run/httpd.pid
```

日志文件

```
/var/log/httpd/
access_log	;访问日志
error_log	;错误日志
```

配置文件

```
/etc/httpd/conf/httpd.conf
;配置文件组成
;Section 1:Global Environment
;Section 2:'Main' or 'Default' server configuration
;Section 3:Virtual Hosts
/etc/httpd/conf.d/*.conf
```

配置文件格式

```
directive	value
;directive	不区分字符大小写
;value	为路径时，是否区分大小写，取决于文件系统
```

检查配置语法

```
httpd -t
service httpd configtest
```

服务脚本

```
/etc/rc.d/init.d/httpd
;脚本配置文件
/etc/sysconfig/httpd
```

模块文件路径

```
/etc/httpd/modules/
/etc/lib64/httpd/modules/
```

站点网页文档根目录（默认）

```
/var/www/html/
```

帮助文档包

`httpd-manual`



END!