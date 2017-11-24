---
title: tomcat 通过 memcached 做会话共享
date: 2017-11-19 22:59:33
category: Tomcat
tags:
	- tomcat
	- msm
	- memcached-session-manager
---

**memcached-session-manager**

msmcached-session-manager 是一款开源的高可用 tomcat session 共享解决方案，他支持两种模式，粘性会话（sticky session）和非粘性会话（non-sticky-session），sticky 模式表示将会话全部存储到一台会话存储服务器上，客户端访问任意一台 web server 都能通过这台会话存储服务器找到会话。noo-sticky 模式表示将会话信息同步到所有会话存储服务器上，这样不管客户端访问哪台 web server 也能找到会话，就算其中一台宕机也毫无影响，因为他还可以在其他会话存储服务器上找到会话。

官方 Wiki：https://github.com/magro/memcached-session-manager/wiki

**实验环境**

前端负载均衡使用为 Nginx

**后端两台 Tomcat Server**

操作系统版本： CentOS 7.3
Tomcat 版本： 7.0.69
JDK 版本：openjdk 8.0
分别为 tomcat-node1、tomcat-node2

**后端两台 Memcached Server**

操作系统：CentOS 7.3

memcached 版本：1.4.15

分别为 memcached-node1、memcached-node2



**Memcached Server 配置**

安装 memcached 包

```
[root@memcached-node1 ~]# yum -y install memcached
```

启动 memcached 服务

```
[root@memcached-node1 ~]# systemctl start memcached.service
```

查看 memcached 端口 `11211` 是否处于监听状态

```
[root@memcached-node1 ~]# ss -tan | grep 11211
LISTEN     0      128          *:11211                    *:*                  
```

两台 memcached 执行相同的操作



**Tomat Server 配置**

msm 支持多种序列化工具，内置的 JAVA 序列化工具、kryo 序列化（据说效率最高）、javolution 序列化等。

安装 tomcat 包

```
[root@tomcat-node1 ~]# yum install tomcat
```

将 memcached-session-manager 的 jar 包拷贝到 `$CATALINA_HOME/lib/` 目录中

> [memcached-session-manager-${version}.jar](http://repo1.maven.org/maven2/de/javakaffee/msm/memcached-session-manager/)
>
> [memcached-session-manager-tc6-${version}.jar](http://repo1.maven.org/maven2/de/javakaffee/msm/memcached-session-manager-tc6/) (tomcat 6)
>
> [memcached-session-manager-tc7-${version}.jar](http://repo1.maven.org/maven2/de/javakaffee/msm/memcached-session-manager-tc7/) (tomcat 7)
>
> [memcached-session-manager-tc8-${version}.jar](http://repo1.maven.org/maven2/de/javakaffee/msm/memcached-session-manager-tc8/) (tomcat 8)

另外用于 memcached 的 jar 包

> [spymemcached-${version}.jar](http://repo1.maven.org/maven2/net/spy/spymemcached/)

例如我这里安装的是 tomcat 7 就把上面对应的三个包放到下面这个目录中

```
/usr/share/tomcat/lib
```

下载序列化工具并将它们放到你的程序的 `WEB-INF/lib` 目录中

例如 kryo 需要下载一下 jar 包

> [msm-kryo-serializer](http://repo1.maven.org/maven2/de/javakaffee/msm/msm-kryo-serializer/)
>
> [kryo-serializers-0.34+](http://repo1.maven.org/maven2/de/javakaffee/kryo-serializers/)
>
> [kryo-3.x](http://repo1.maven.org/maven2/com/esotericsoftware/kryo/)
>
> [minlog](http://repo1.maven.org/maven2/com/esotericsoftware/minlog/)
>
> [reflectasm](http://repo1.maven.org/maven2/com/esotericsoftware/reflectasm/)
>
> [asm-5.x](http://repo1.maven.org/maven2/org/ow2/asm/asm/)
>
> [objenesis-2.x](http://repo1.maven.org/maven2/org/objenesis/objenesis/)

例如我这里的程序目录是在下面这个路径中

```
/var/lib/tomcat/webapps/testapp
```

那你的程序目录中应该创建一个 `WEB-INF`

```
[root@tomcat-node1 ~]# ll /var/lib/tomcat/webapps/testapp
drwxr-xr-x 3 root root  32 Nov 20 01:13 WEB-INF
```

然后在 `WEB-INF` 目录中创建一个 `lib` 目录，结构如下

```
[root@tomcat-node1 webapps]# tree testapp/
testapp/
└── WEB-INF
    └── lib

```

然后将上面下载 6 个 jar 放到 `WEB-INF/lib` 目录中，结构应该像下面这样

```
[root@tomcat-node1 webapps]# tree testapp/
testapp/
└── WEB-INF
    └── lib
        ├── asm-6.0.jar
        ├── kryo-4.0.1.jar
        ├── kryo-serializers-0.42.jar
        ├── minlog-1.3.0.jar
        ├── msm-kryo-serializer-2.1.1.jar
        ├── objenesis-2.6.jar
        └── reflectasm-1.11.3.jar

```

在你的程序对应的 `<Context>...</Context>` 中添加 ` Manager` 

```
<Context path="/testapp" docBase="testapp">
   <Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager"
    memcachedNodes="mnode1:192.168.1.114:11211,mnode2:192.168.1.115:11211"
    failoverNodes="mem-node2"
    requestUriIgnorePattern=".*\.(ico|png|gif|jpg|css|js)$"
    transcoderFactoryClass="de.javakaffee.web.msm.serializer.kryo.KryoTranscoderFactory"/>
</Context>
```

上面是一个粘性会话的配置

`memcachedNodes` 表示 memcached 节点，填上对于的 IP 地址，IP 地址前面的名称为自定义名称

`failoverNodes` 它让 msm 把 会话存入 mnode1 中，只有当 mnode2 无效的时候才会把会话放到 mem-node2 上

`requestUriIgnorePattern` 表示要忽略的 URI



对于粘性会话的配置

```
<Context path="/testapp" docBase="testapp">
<Manager className="de.javakaffee.web.msm.MemcachedBackupSessionManager"
   memcachedNodes="mnode1:192.168.1.114:11211,mnode2:192.168.1.115:11211"
   sticky="false"
   sessionBackupAsync="false"
   lockingMode="auto"
   requestUriIgnorePattern=".*\.(ico|png|gif|jpg|css|js)$"
   transcoderFactoryClass="de.javakaffee.web.msm.serializer.kryo.KryoTranscoderFactory" />
</Context>
```

`sticky` 赋值为 `false` 表示不使用粘性会话

`sessionBackupAsync` 表示是否采用异步的方式备份 session

`lockingMode` 设置 session 的锁定模式，auto 表示对于只读请求 session 将不会锁定，如果包含写入请求，则 session 会被锁定

添加测试页面

```
<%@ page language="java" %>
<html>
  <head><title>tomcat-node1</title></head>
  <body>
    <h1><font color="red">tomcat-node1</font></h1>
    <table align="centre" border="1">
      <tr>
        <td>Session ID</td>
    <% session.setAttribute("weidong.io","weidong.io"); %>
        <td><%= session.getId() %></td>
      </tr>
      <tr>
        <td>Created on</td>
        <td><%= session.getCreationTime() %></td>
     </tr>
    </table>
  </body>
</html>
```

启动 tomcat 服务

```
[root@tomcat-node1 ~]# systemctl start tomcat.service 
```



tomcat-node2 的配置与 tomcat-node1 基本一样

添加 tomcat-node2 的测试页面

```
<%@ page language="java" %>
<html>
  <head><title>tomcat-node2</title></head>
  <body>
    <h1><font color="blue">tomcat-node2</font></h1>
    <table align="centre" border="1">
      <tr>
        <td>Session ID</td>
    <% session.setAttribute("weidong.io","weidong.io"); %>
        <td><%= session.getId() %></td>
      </tr>
      <tr>
        <td>Created on</td>
        <td><%= session.getCreationTime() %></td>
     </tr>
    </table>
  </body>
</html>
```

 

**前端 Nginx 配置**

安装 Nginx 软件包

```
yum install nginx
```

配置 Nginx 反向代理

在 `nginx.conf` 的 `http` 段落中添加 server 组，将默认 server 代理到后端的 tomcat server 组

```
upstream tcser {
        server 192.168.1.112:8080;
        server 192.168.1.113:8080;
        
server {
        listen       80 default_server;
        location / {
                proxy_pass http://tcser;
        }
```

测试配置文件

```
nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

启动 Nginx

```
systemctl start nginx.service
```

访问测试

![node1](http://ov2iiuul1.bkt.clouddn.com/tomcat-node1.png)

多次刷新

![node2](http://ov2iiuul1.bkt.clouddn.com/tomcat-node2.png)

可以清楚的看到，不管客户端被调度到哪台 tomcat server 上，它的会话 ID 始终保持着不会改变，因为他已经被存入 后端的 memcached 中了，不管他调度到哪台 tomcat server 上，msm 都能在 memcached 中找到他



END!
