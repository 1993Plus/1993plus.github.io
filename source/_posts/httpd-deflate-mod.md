---
title: httpd(2.2) deflate modules
date: 2017-10-08 21:15:31
category: HTTPD
tags:
	- httpd
	- gzip
	- deflate
---

**HTTPD(2.2)  deflate_module**

官方文档地址：http://httpd.apache.org/docs/2.2/mod/mod_deflate.html

`mod_deflate` 模块提供了一个压缩输出过滤器，它允许服务器的输出被压缩，然后再通过网络发送给客户端。

优点：节约带宽

缺点：消耗 CPU ，有些较老的浏览器不支持

支持压缩的文件类型

```
text/html text/plain text/xml text/css text/javascript application/javascript
```

确认模块是否加载

```sh
httpd -M | grep deflate
 deflate_module (shared)
Syntax OK
```

开启压缩指令

```sh
SetOutputFilter DEFLATE
```

如果要将压缩限制为一般的 `MIME` 类型，可以使用 `AddOutputFilterByType` 指令。

为了方便管理建议单独创建一个配置文件

```sh
vim /etc/httpd/conf.d/deflate.conf
SetOutputFilter DEFLATE
<Directory "/var/www/website1">		;压缩指定目录的文件
AddOutputFilterByType DEFLATE text/html		;指定要压缩的文件类型
</Directory>
```

重启服务

```sh
service httpd restart
```

测试文件原大小

```sh
-rw-r--r--. 1 root root 481918 Oct  5 11:49 /var/www/website1/test.html
```

浏览器访问测试

![压缩后大小](http://ov2iiuul1.bkt.clouddn.com/deflate_1.png)



可以看到原文件大小为 481918 个字节，压缩后大小只有 41751 个字节了。

**针对特定浏览器执行压缩**

官方示例

```sh
BrowserMatch ^Mozilla/4 gzip-only-text/html
BrowserMatch ^Mozilla/4\.0[678] no-gzip
BrowserMatch \bMSIE !no-gzip !gzip-only-text/html
```



**压缩级别**

压缩级别由 `DeflateCompressionLevel` 指令控制，压缩级别从 1-9 ，压缩等级越高，压缩比就越高，但会更消耗 CPU 资源；



END!