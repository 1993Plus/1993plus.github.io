---
title: HTTPD VirtualHost 配置
date: 2017-10-07 22:13:22
category: HTTPD
tags:
	- httpd
	- virtualhost
---

**HTTPD 虚拟主机配置**

**基于 IP 地址创建**

添加三个 IP 地址

```sh
192.168.1.101/24
192.168.1.102/24
192.168.1.103/24
```

`DocumentRoot` 分别为

```sh
/var/www/website1	;对应 IP 地址 192.168.1.101/24
/var/www/website2	;对应 IP 地址 192.168.1.102/24
/var/www/website3	;对应 IP 地址 192.168.1.103/24
```

为了方便管理应该为虚拟主机创建单独的配置文件

```sh
vim /etc/httpd/conf.d/vhost.conf

<virtualhost 192.168.1.101:80>
DocumentRoot "/var/www/website1"
</virtualhost>

<virtualhost 192.168.1.102:80>
DocumentRoot "/var/www/website2"
</virtualhost>

<virtualhost 192.168.1.103:80>
DocumentRoot "/var/www/website3"
</virtualhost>
```

重载配置文件访问测试

```sh
service httpd reload

curl 192.168.1.101
Virtual Host 1
curl 192.168.1.102
Virtual Host 2
curl 192.168.1.103
Virtual Host 3
```

添加日志文件

```sh
vim /etc/httpd/conf.d/vhost.conf 

<virtualhost 192.168.1.101:80>
DocumentRoot "/var/www/website1"
ErrorLog logs/website1_error_log
CustomLog logs/website1_access_log common
</virtualhost>

<virtualhost 192.168.1.102:80>
DocumentRoot "/var/www/website2"
ErrorLog logs/website2_error_log
CustomLog logs/website2_access_log common
</virtualhost>

<virtualhost 192.168.1.103:80>
DocumentRoot "/var/www/website3"
ErrorLog logs/website3_error_log
CustomLog logs/website3_access_log common
</virtualhost>
```

查看日志

```sh
cat /var/log/httpd/website*
192.168.1.101 - - [05/Oct/2017:04:24:56 +0800] "GET / HTTP/1.1" 200 15
192.168.1.102 - - [05/Oct/2017:04:24:58 +0800] "GET / HTTP/1.1" 200 15
192.168.1.103 - - [05/Oct/2017:04:25:01 +0800] "GET / HTTP/1.1" 200 15
```



**基于端口创建**

使用同一个 IP 不同端口

监听端口分别为 81,82,83, IP 地址都使用 192.168.1.101

```sh
vim /etc/httpd/conf.d/vhost.conf 
Listen 192.168.1.101:81
Listen 192.168.1.101:82
Listen 192.168.1.101:83

<virtualhost 192.168.1.101:81>
DocumentRoot "/var/www/website1"
ErrorLog logs/website1_error_log
CustomLog logs/website1_access_log common
</virtualhost>

<virtualhost 192.168.1.101:82>
DocumentRoot "/var/www/website2"
ErrorLog logs/website2_error_log
CustomLog logs/website2_access_log common
</virtualhost>

<virtualhost 192.168.1.101:83>
DocumentRoot "/var/www/website3"
ErrorLog logs/website3_error_log
CustomLog logs/website3_access_log common
</virtualhost>
```

重启服务访问测试

```sh
service httpd restart

curl 192.168.1.101:81
Virtual Host 1
curl 192.168.1.101:82
Virtual Host 2
curl 192.168.1.101:83
Virtual Host 3
```



**基于 FQDN**

域名分别为

```
www.website1.com
www.website2.com
www.website3.com
```

虚拟主机配置文件

```sh
Namevirtualhost *:80
<virtualhost *:80>
ServerName www.website1.com
DocumentRoot "/var/www/website1"
ErrorLog logs/website1_error_log
CustomLog logs/website1_access_log common
</virtualhost>

<virtualhost *:80>
ServerName www.website2.com
DocumentRoot "/var/www/website2"
ErrorLog logs/website2_error_log
CustomLog logs/website2_access_log common
</virtualhost>

<virtualhost *:80>
ServerName www.website3.com
DocumentRoot "/var/www/website3"
ErrorLog logs/website3_error_log
CustomLog logs/website3_access_log common
</virtualhost>
```

如果没有配置 DNS 服务器可以通过修改 hosts 文件来测试

```sh
vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.101   www.website1.com
192.168.1.101   www.website2.com
192.168.1.101   www.website3.com
```

重启 httpd 服务访问测试

```sh
service httpd restart

curl www.website1.com
Virtual Host 1
curl www.website2.com
Virtual Host 2
curl www.website3.com
Virtual Host 3
```



END!