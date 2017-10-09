---
title: httpd(2.4) 新特性
date: 2017-10-09 16:28:23
category: HTTPD
tags:
	- httpd
	- httpd2.4
---

**HTTPD-2.4 **


HTTPD 2.4 新特性

* MPM 支持运行为 DSO （动态共享对象）机制；以模块形式按需加载
* event MPM 生产环境可用
* 异步读写机制
* 支持没个模块及每个目录的单独日志级别定义
* 每个请求相关的专用配置
* 增强版的表达式分析
* 毫秒级别持久连接时长定义
* 基于 FQDN 的虚拟主机不需要 NameVirtualHost 指令
* 新指令，AllowOverrideList
* 支持用户自定义变量
* 更低的内存消耗

修改了一些配置机制

不再支持使用 Order，Deny，Allow，来做基于 IP 的访问控制

新模块

* mod_proxy_fcgi（FastCGI Protocol backend for mod_proxy）
* mod_remoteip（Replaces the apparent client remote IP address and hostname for the request with the  IP address list presented by a proxies or a load balancer via the request headers）
* mod_retelimit（Provides Bandwidth Rate Limiting for clients）



配置文件

```sh
/etc/httpd/conf/httpd.conf
/etc/httpd/conf.d/*.conf
```

模块相关配置文件

```sh
/etc/httpd/conf.modules.d/*.conf
```

systemd unit fiel

```sh
/usr/lib/systemd/system/httpd.service
```

主程序文件

```sh
/usr/sbin/httpd
; 与之前版本不同的是主程序只有一个，切换 MPM 模式使用模块的形式切换
```

日志文件

```sh
/var/log/httpd/
access_log	;访问日志
error_log	;错误日志
```

默认站点文档路径

```sh
/var/www/html
```

模块文件路径

```sh
/usr/lib64/modules
```



END!