---
title: httpd(2.2) 常见配置
date: 2017-10-07 17:19:01
category: HTTPD
tags:
	- httpd
	- web
---


解决 `HTTPD` 服务启动错误提示

```
[root@localhost ~]# service httpd start
Starting httpd: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain for ServerName
                                                           [  OK  ]
```

修改配置文件

```
;/etc/httpd/conf/httpd.conf
;修改字段
#ServerName www.example.com:80
;修改后，地址可自行设置
ServerName 127.0.0.1:80
```

再次启动

```
service httpd start
Starting httpd:                                            [  OK  ]
```



**`ServerTokens`**

官方文档地址：http://httpd.apache.org/docs/2.2/mod/core.html#servertokens

该指令控制发送回给客户端的服务器响应头字段是否包括服务器的通用os类型的描述，以及关于被编译模块的信息。

```httpd
ServerTokens Prod[uctOnly]
Server sends (e.g.): Server: Apache
ServerTokens Major
Server sends (e.g.): Server: Apache/2
ServerTokens Minor
Server sends (e.g.): Server: Apache/2.0
ServerTokens Min[imal]
Server sends (e.g.): Server: Apache/2.0.41
ServerTokens OS
Server sends (e.g.): Server: Apache/2.0.41 (Unix)
ServerTokens Full (or not specified)
Server sends (e.g.): Server: Apache/2.0.41 (Unix) PHP/4.2.2 MyMod/1.2
```

为了安全考虑通常建议设置为 `Prod`

```
/etc/httpd/conf/httpd.conf
;默认值为 OS
ServerTokens OS
;修改为 Prod
ServerTokens Prod
;重新载入配置文件
service httpd reload
;使用 curl 工具测试；加上 -I 参数，只看头部
curl -I 127.0.0.1
HTTP/1.1 200 OK
Date: Wed, 04 Oct 2017 14:46:17 GMT
Server: Apache
Last-Modified: Sat, 02 Sep 2017 19:49:05 GMT
ETag: "14024f-16-5583a2b8666e9"
Accept-Ranges: bytes
Content-Length: 22
Connection: close
Content-Type: text/html; charset=UTF-8
```



**Listen**

官方文档地址：http://httpd.apache.org/docs/2.2/mod/mpm_common.html#listen

* Listen 指令配置 HTTPD 只监听特定的IP地址或端口；默认情况下，它响应所有IP接口的请求。Listen 是一个必需的指令。如果它不在配置文件中，服务器将无法启动。
* Listen 指令告诉服务器接受指定 Port 或 Address Port组合的传入请求。如果只指定了一个端口号，服务器将在所有接口上监听给定端口。如果给定一个IP地址和一个端口，服务器将只监听给定的端口和接口。
* 多个侦听指令可用于指定要侦听的多个地址和端口。服务器将响应来自任何列出的地址和端口的请求。

例如要同时监听在 80 与 8080 端口上

```
/etc/httpd/conf/httpd.conf
Listen 80
Listen 8080
;重新载入配置文件
service httpd reload
;查看服务器端口监听状态
ss -tan | grep '80\|8080'   
LISTEN     0      128                      :::8080                    :::*     
LISTEN     0      128                      :::80                      :::*     
```

监听在指定的 `IP PORT` 上

```
/etc/httpd/conf/httpd.conf
;现在服务器有两个 IP，分别为 192.168.1.200,192.168.1.201
;设置让它只监听 192.168.1.201 8080 上
Listen 192.168.1.201:8080
;重启 httpd 服务
service httpd restart
;测试结果
curl 127.0.0.1
curl: (7) couldn't connect to host
curl 192.168.1.200
curl: (7) couldn't connect to host
curl 192.168.1.201
curl: (7) couldn't connect to host
;现在除了 192.168.1.201:8080 可以访问，其它地址都无法访问了
curl 192.168.1.201:8080
<h1>Hello world!</h1>
```



**KeepAlive**

官方文档地址：http://httpd.apache.org/docs/2.2/mod/core.html#keepalive

Persistent Connection: 连接建立，每个资源获取完成后不会断开连接，而是继续等待其它的请求完成，默认是关闭的。

断开条件由 `MaxKeepAliveRequests ` 与 `KeepAliveTimeout` 决定

```
;默认设置
MaxKeepAliveRequests 100	;表示当请求数量达到 100 后断开
KeepAliveTimeout 15		;以秒(seconds)为单位，连接时间到达 15 秒后断开
```

副作用：对并发访问量较大的服务器，持久连接功能会使有些请求得不到响应

修改配置

```
/etc/httpd/conf/httpd.conf
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 50
;使用 telnet 客户端测试
telnet 192.168.1.200 80
Trying 192.168.1.200...
Connected to 192.168.1.200.
Escape character is '^]'.
GET /index.html HTTP/1.1	;第一次请求资源
HOST: 192.168.1.200

HTTP/1.1 200 OK
Date: Wed, 04 Oct 2017 16:01:10 GMT
Server: Apache
Last-Modified: Sat, 02 Sep 2017 19:49:05 GMT
ETag: "14024f-16-5583a2b8666e9"
Accept-Ranges: bytes
Content-Length: 22
Content-Type: text/html; charset=UTF-8

<h1>Hello world!</h1>
GET /index2.html HTTP/1.1	;第二次请求资源
HOST: 192.168.1.200

HTTP/1.1 200 OK
Date: Wed, 04 Oct 2017 16:01:39 GMT
Server: Apache
Last-Modified: Wed, 04 Oct 2017 15:02:22 GMT
ETag: "141217-18-55ab9e5189d4b"
Accept-Ranges: bytes
Content-Length: 24
Content-Type: text/html; charset=UTF-8

<h1>Hello world!-2</h1>
Connection closed by foreign host.
```



**MPM (Multi-Processing Module) 多路处理模块**

* prefork

  prefork 模式默认配置

  ```
  <IfModule prefork.c>
  StartServers       8	;初始子进程数
  MinSpareServers    5	;最小空闲子进程数(当剩余的子进程数小于 5 个时新开子进程)
  MaxSpareServers   20	;最大空闲子进程数(客户端在访问结束后可存活的子进程数)
  ServerLimit      256	;最多进程数，最大 2000
  MaxClients       256	;最大并发，应该与 ServerLimit 的值相等
  MaxRequestsPerChild  4000	;子进程处理的最多请求数量，当达到这个值时将被父进程杀死，释放内存
  </IfModule>
  ;MaxRequestsPerChild 的值为 0 时将永远不释放内存
  ```

* worker

worker 模式默认配置

```
<IfModule worker.c>
StartServers         4	;初始进程数
MaxClients         300	;最大请求数
MinSpareThreads     25	;最小空闲线程数
MaxSpareThreads     75	;最大空闲线程数
ThreadsPerChild     25	;每个子进程可开启的线程数
MaxRequestsPerChild  0
</IfModule>
```

* event(测试阶段)

httpd 2.2 不支持同时编译多个模块，所以只能编译时选定一个；rpm 安装的包提供三个二进制程序文件，分别用于实现对不同 PMP 机制的支持

要查看当前使用的模式可以通过查看进程名称来确认

如果进程名为 `httpd` 即为 `prefork` 模式，如果是 `httpd.worker` 则为 `worker` 模式

可以通过修改配置文件 `vim /etc/sysconfig/httpd` 来更改模式

```
;找到这一行将前面的 “#” 去掉，重启 httpd 服务即可
#HTTPD=/usr/sbin/httpd.worker
```

**查看模块列表**

查看静态编译的模块

```
httpd -l
```

查看静态编译及动态装载的模块

```
httpd -M
```

动态模块加载不需要重启即可生效
动态模块路径

```
/usr/lib64/httpd/modules/
```



**DSO(Dynamic Shared Object)**

加载动态模块配置

格式

```
LoadModule		<mod_name>		<mod_path>
```

系统自带的一些模块

```
LoadModule auth_basic_module modules/mod_auth_basic.so
LoadModule auth_digest_module modules/mod_auth_digest.so
LoadModule authn_file_module modules/mod_authn_file.so
LoadModule authn_alias_module modules/mod_authn_alias.so
```

这些模块的路径使用的是一个相对路径，相对于 `ServerRoot ` 的值，默认 `/etc/httpd` 

另外一种方法就是单独创建一个配置文件来加载模块，这些配置文件应放在 `conf.d` 目录下，配置文件必须以 `".conf"` 结尾，因为这是由 `Include` 指令决定的



**定义 Main Server 的页面路径**

页面路径由 `DocumentRoot` 来定义，指定的路径为 URL 路径的起始位置

```
DocumentRoot "/var/www/html"
```

当更改了这个值时需要调整 SELinux



**定义站点主页面**

```
DirectoryIndex index.html index.html.var
```



END!