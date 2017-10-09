---
title: httpd(2.4) 常用配置
date: 2017-10-09 18:54:57
category:
tags:
---

**httpd 2.4 常用配置**

**切换 MPM**

在 `conf.modules.d` 目录下默认有 7 个文件，其中 `00-mpm.conf` 这个配置文件就是配置 `MPM` 模块的

```sh
ls -l /etc/httpd/conf.modules.d/
-rw-r--r-- 1 root root 3739 Nov 15  2016 00-base.conf
-rw-r--r-- 1 root root  139 Nov 15  2016 00-dav.conf
-rw-r--r-- 1 root root   41 Nov 15  2016 00-lua.conf
-rw-r--r-- 1 root root  742 Nov 15  2016 00-mpm.conf
-rw-r--r-- 1 root root  957 Nov 15  2016 00-proxy.conf
-rw-r--r-- 1 root root   88 Nov 15  2016 00-systemd.conf
-rw-r--r-- 1 root root  451 Nov 15  2016 01-cgi.conf
```

打开 `00-mpm.conf` 其中主要的就是以下 3 行，默认使用的是 `prefork` 模式

```sh
LoadModule mpm_prefork_module modules/mod_mpm_prefork.so	; prefork 模式
#LoadModule mpm_worker_module modules/mod_mpm_worker.so		; worker 模式
#LoadModule mpm_event_module modules/mod_mpm_event.so		; event 模式
; 这三种模式只能开其中一个，不使用的模式就直接注释掉
```



**访问控制**

没有明确授权的目录，默认拒绝

允许所有主机访问

```sh
Require all granted
```

拒绝所有主机访问

```sh
Require all denied
```

控制特定的 IP 访问

```sh
Require ip IP-Address	; 允许指定 IP 的访问
Require not ip IP-Address	; 拒绝指定 IP 的访问
```

控制特定的主机访问

```sh
Require host HOSTNAME{FQDN|Domin.tld}	; 允许指定主机的访问
Require not host HOSTNAME{FQDN|Domin.tld}	; 拒绝指定主机的访问
```

允许所有 IP 后，拒绝指定 IP 访问

```sh
<RequireAll>
	Require all granted
	Require not ip 192.168.1.1	; 这个 IP 将被拒绝访问
</RequireAll>
```

拒绝所有 IP 后，允许特定 IP 访问

```sh
<RequireAny>
	Require all denied
	Require ip 192.168.1.1	; 只允许这个 IP 访问
</RequireAny>
```

现在添加一个新的 `DocumentRoot` 路径

```sh
vim website.conf
DocumentRoot "/website"
```

使用 curl 来测试

```sh
curl -I 127.0.0.1
HTTP/1.1 403 Forbidden	; 403 因为权限问题被拒绝访问了
Date: Mon, 09 Oct 2017 09:31:37 GMT
Server: Apache/2.4.6 (CentOS)
Last-Modified: Thu, 16 Oct 2014 13:20:58 GMT
ETag: "1321-5058a1e728280"
Accept-Ranges: bytes
Content-Length: 4897
Content-Type: text/html; charset=UTF-8
```

允许所有主机访问

```sh
vim website.conf
DocumentRoot "/website"
<Directory "/website">
Require all granted
</Directory>
```

使用 curl 来测试

```sh
curl 127.0.0.1   
Hello World!
```

拒绝所有的 IP 访问，只允许 192.168.1.101 这个 IP 访问

```sh
DocumentRoot "/website"
<Directory "/website">
<RequireAny>
Require all denied
Require ip 192.168.1.101
</RequireAny>
</Directory>
```

由于这里只允许 192.168.1.101 这个地址访问，这将意味着本机也无法访问了

```sh
curl  -I 127.0.0.1
HTTP/1.1 403 Forbidden		; 403 权限被拒绝
Date: Mon, 09 Oct 2017 09:47:38 GMT
Server: Apache/2.4.6 (CentOS)
Last-Modified: Thu, 16 Oct 2014 13:20:58 GMT
ETag: "1321-5058a1e728280"
Accept-Ranges: bytes
Content-Length: 4897
Content-Type: text/html; charset=UTF-8
```

在 192.168.1.101 主机上访问

```sh
curl -I 192.168.1.98
HTTP/1.1 200 OK		; 状态码 200，访问成功
Date: Mon, 09 Oct 2017 09:48:06 GMT
Server: Apache/2.4.6 (CentOS)
Last-Modified: Mon, 09 Oct 2017 09:24:49 GMT
ETag: "d-55b19c32359ed"
Accept-Ranges: bytes
Content-Length: 13
Content-Type: text/html; charset=UTF-8
```



**虚拟主机**

在 httpd 2.4 中配置基于 FQDN 的虚拟主机取消了 `NameVirtualHost` 指令，而是直接使用 `VirtualHost` 指令

```sh
<VirtualHost *:80>
        ServerName www.weidong.io
        DocumentRoot "/website"
        <Directory "/website">
                Require all granted
        </Directory>
</VirtualHost>
```

访问测试

```sh
curl www.weidong.io    
Hello World!
```



END!