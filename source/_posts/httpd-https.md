---
title: httpd 实现 https
date: 2017-10-09 12:44:40
category:
tags:
---


**httpd 实现 https**

**向 CA 申请证书**

搭建私有 CA 与颁发证书：http://www.weidong.io/2017/09/29/create-private-ca/

为了方便管理可以将证书与私钥文件放到一个指定的位置

```sh
ls -l /etc/httpd/ssl/
-rw-r--r-- 1 root root 4614 Oct  5 14:17 cacert.pem		; 证书颁发机构的证书文件
-rw-r--r-- 1 root root 6182 Oct  5 14:15 weidong.io.crt	; 申请到的证书文件
-rw-r--r-- 1 root root 1756 Oct  5 14:10 weidong.io.csr	; 证书申请文件
-rw------- 1 root root 3243 Oct  5 14:06 weidong.io.key	; 私钥文件
```

**安装模块**

```
yum install -y mod_ssl
```

修改配置文件

```sh
SSLCertificateFile /etc/httpd/ssl/weidong.io.crt	; 证书文件路径
SSLCertificateKeyFile /etc/httpd/ssl/weidong.io.key	; 私钥文件路径
SSLCACertificateFile /etc/httpd/ssl/cacert.pem	; 证书颁发机构的证书文件路径
```

重启 `httpd` 服务

```sh
service httpd restart
```

如果你的客户端浏览器已经信任了该证书的根证书，那应该就能正常访问了，如果没有信任将会出现错误提示

![访问测试](http://ov2iiuul1.bkt.clouddn.com/httpd_https1.png)

查看证书

![查看证书](http://ov2iiuul1.bkt.clouddn.com/httpd_https2.png)

域名的主机位使用 “*” 表示使用泛域名，这将意味着在访问该域名时，只要是在 weidong.io 域中的域名都可以使用 https

但现在还有一个问题，那就是如果我在地址栏直接输入 www.weidong.io ，它是不会使用 https ，这时候就需要做 URL 重定向，当收到 http://www.weidong.io 的请求时就将 URL 重定向到 https://www.weidong.io 

详细信息可查看官方文档：http://httpd.apache.org/docs/2.2/mod/mod_alias.html#redirect
常用到的就是 301 永久重定向，302 临时重定向

**URL 重定向**

指令

```sh
Redirect
Redirect permanent / URL	; 代表 301 永久重定向
Redirect temp / URL	; 代表 302 临时重定向
```

具体实现

```sh
vim /etc/httpd/conf.d/vhost.conf 
Namevirtualhost *:80
<virtualhost *:80>
ServerName www.weidong.io
DocumentRoot "/var/www/website1"
Redirect temp / https://www.weidong.io/	; 使用临时重定向到 https
ErrorLog logs/website1_error_log
CustomLog logs/website1_access_log combined
</virtualhost>
```

 抓包查看跳转情况

![抓包查看跳转情况](http://ov2iiuul1.bkt.clouddn.com/httpd_https3.png)


试试 301 永久重定向

```sh
Redirect permanent / https://www.weidong.io/	; 使用临时重定向到 https
```

抓包查看跳转情况

![抓包查看跳转情况](http://ov2iiuul1.bkt.clouddn.com/httpd_https4.png)

**什么时候使用 301,302 呢？**

如果你有一个老地址已经确定不实用了话建议使用 301，像 http 转 https 这种，因为 http 协议的 URL 地址还会使用到，所以一般使用 302 重定向，因为这会关系到搜索引擎优化问题



END!