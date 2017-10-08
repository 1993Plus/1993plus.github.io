---
title: httpd(2.2) server status modules
date: 2017-10-08 14:55:43
category: HTTPD
tags:
	- httpd
	- server_status_mod
---

HTTPD(2.2) 服务状态监控模块 `status_module`

`status_module` 模块是 httpd 中一个用来监控 httpd 服务端各种状态信息的模块

查看模块是否已加载

```sh
httpd -M | grep status
status_module (shared)	;有表示已加载
Syntax OK
```

修改配置文件

```sh
;默认该配置是被注释了，需要将前面的 "#" 号去掉
<Location /server-status>	;"/" 后面的字段可以自定义修改，访问的时候将它放到地址后面
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from 192.168.1.0/24	;允许访问的地址
</Location>
```

重启服务测试

在浏览器地址栏中输入 http://SERVER_IP/server-status 

例如我现在服务端的 IP 地址是 192.168.1.101，那地址就是 http://192.168.1.101/server-status

![访问测试](http://ov2iiuul1.bkt.clouddn.com/server-status1.png)



如果想查看更详细的信息，那就将 `ExtendedStatus On` 取消注释即可

