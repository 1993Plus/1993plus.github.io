---
title: dns service bind
date: 2017-09-27 09:10:21
category: Bind
tags:
	- bind
---

bind 搭建私有根域主从实验，涉及主从复制，转发，简单全安限制。

**网络示意图**

![网络示意图](http://ov2iiuul1.bkt.clouddn.com/DNS.png)

**准备工作：**

* 关闭 SELinux
* 关闭 IPTABLES
* 除 Client 端其它 Server DNS 地址全部指向自己 

**服务器安装 bind 包**

`bind` 主程序包

`bind-utils` 测试工具包

```sh
yum install -y bind bind-utils
```

将所有服务器的 `bind` 设置为开机自启动

```sh
systemctl enable named
```

**. (根) 域服务器配置**

修改 `named.conf` 配置文件

```sh
;options 中的设置为全局设置
options {
        ;监听的端口与地址
        listen-on port 53 { 192.168.1.100; 127.0.0.1; };
        ;bind 数据目录
        directory       "/var/named";
        ;允许查询解析库的地址
        allow-query     { any; };
        
        ;关闭递归解析
        recursion no;
        ;关闭 dns 安全扩展
        dnssec-enable no;
        dnssec-validation no;
};


;定义 .(根)域正向解析域
zone "." IN {
        type master;
        ;定义正向解析库 zone 文件
        file "root.zone";
        ;不允许任何地址更新该解析库
        allow-update { none; };
        ;不允许任何地址对该域做区域传送
        allow-transfer { none; };
};

;定义 .(根)域反向解析域
zone "1.168.192.in-addr.arpa" IN {
        type master;
        ;定义反向解析库 zone 文件
        file "rev.root.zone";
        allow-update { none; };
        allow-transfer { none; };
};
```



创建` . (根)`域正向解析库 `root.zone` 文件

```sh
;定义 dns 缓存最大可缓存 TTL 值为一天
$TTL 86400
;定义区域名称
$ORIGIN .
;起始授权记录( Start Of Authority)
@       IN      SOA     ns.root.        admin.weidong.com. (
                                        ;序列号,修改文件时需递增,否则从服务器不会立即同步
                                        20170927 ; serial
                                        ;刷新时间,从服务器每隔 1 天主动 pull 一次解析库
                                        1D       ; refresh
                                        ;重试时间,同步失败时 1 小时候重新尝试同步解析库
                                        1H       ; retry
                                        ;如果长达 1 周无法联系上主服务器从服务器停止服务
                                        1W       ; expire
                                        ;接收到不存在或者错误解析条目时将改条目缓存 3 小时 
                                        3H )     ; minimum
           IN      NS      ns.root.
com.    IN      NS      ns.com.
ns.root.        IN      A       192.168.1.100
ns.com.        IN      A       192.168.1.99
```



创建` . (根)`域反向解析库 `rev.root.zone` 文件

```sh
$TTL 86400
$ORIGIN 1.168.192.in-addr.arpa.
@       IN      SOA     ns.root.        admin.weidong.com. (
                                    20170927 ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns.root.
        IN      NS      ns.com.
100     IN      PTR     ns.root.
99      IN      PTR     ns.com.
```

修改解析库文件权限 (`为了安全将解析文件权限设置为 rw-r----- 所属组设置为 named`)

```sh
chmod 640 root.zone rev.root.zone 
chown :named root.zone rev.root.zone
```

启动 `bind` 服务

```sh
systemctl start named
```



**`com` 域服务器配置**

修改 `named.conf` 配置文件

```sh
options {
        listen-on port 53 { 192.168.1.99; 127.0.0.1; };
        directory       "/var/named";
        allow-query     { any; };
        
        recursion no;

        dnssec-enable no;
        dnssec-validation no;
};

;定义根区域文件
zone "." IN {
        ;定义类型为 hint
        type hint;
        ;定义文件,路径相对于 /var/named/
        file "named.ca";
};

zone "com" IN {
        type master;
        file "com.zone";
        allow-update { none; };
        allow-transfer { none; };
};

zone "1.168.192.in-addr.arpa" IN {
        type master;
        file "rev.com.zone";
        allow-update { none; };
        allow-transfer { none; };
};
```

导入 `.（根）` 域 NS 记录

```sh
dig -t NS . @192.168.1.100 > /var/named/named.ca
```

创建 `com` 域正向解析库 `com.zone` 文件

```sh
$TTL 86400
$ORIGIN com.
@       IN      SOA     ns.com.        admin.weidong.com. (
                                        20170926 ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns.com.
ns.com. IN      A       192.168.1.99
weidong.com.    IN      NS      ns1.weidong.com.
weidong.com.    IN      NS      ns2.weidong.com.
ns1.weidong.com.        IN      A       192.168.1.98
ns2.weidong.com.        IN      A       192.168.1.97
```

创建 `com` 域反向解析库 `rev.com.zone` 文件

```sh
$TTL 86400
$ORIGIN 1.168.192.in-addr.arpa.
@       IN      SOA     ns.com.        admin.weidong.com. (
                                        20170926 ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns.com.
99      IN      PTR     ns.com.
98      IN      PTR     ns1.weidong.com.
97      IN      PTR     ns2.weidong.com.
```

修改文件权限

```sh
chmod 640 com.zone rev.com.zone 
chown :named com.zone rev.com.zone
```



**`weidong.com master` 域主服务器配置**

修改 `named.conf` 配置文件

```sh

options {
        listen-on port 53 { 192.168.1.98; 127.0.0.1; };
        directory       "/var/named";
        allow-query     { any; };
        
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "weidong.com" IN {
        type master;
        file "weidong.com.zone";
        ;只允许指定的地址更新该解析库
        allow-update { 192.168.1.97; };
        ;只允许指定的地址对该域做区域传送
        allow-transfer { 192.168.1.97; };
};

zone "1.168.192.in-addr.arpa" IN {
        type master;
        file "rev.weidong.com.zone";
        allow-update { 192.168.1.97; };
        allow-transfer { 192.168.1.97; };
};
```

 导入 `.（根）` 域 NS 记录

```sh
dig -t NS . @192.168.1.100 > /var/named/named.ca
```

创建 `weidong.com` 域正向解析库文件 `weidong.com.zone`

```sh
$TTL 86400
$ORIGIN weidong.com.
@       IN      SOA     ns1.weidong.com.        admin.weidong.com. (
                                        20170926 ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns1.weidong.com.
        IN      NS      ns2.weidong.com.
ns1.weidong.com.        IN      A       192.168.1.98
ns2.weidong.com.        IN      A       192.168.1.97
sc.weidong.com.        IN      NS      ns1.sc.weidong.com.
sc.weidong.com.        IN      NS      ns2.sc.weidong.com.
ns1.sc.weidong.com.     IN      A       192.168.1.96
ns2.sc.weidong.com.     IN      A       192.168.1.95
```

 创建 `weidong.com` 域反向解析库文件 `rev.weidong.com.zone`

```sh
$TTL 86400
$ORIGIN 1.168.192.in-addr.arpa.
@       IN      SOA     ns1.weidong.com.        admin.weidong.com. (
                                        20170926 ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns1.weidong.com.
        IN      NS      ns2.weidong.com.
97      IN      PTR     ns2.weidong.com.
98      IN      PTR     ns1.weidong.com.
95      IN      PTR     ns1.sc.weidong.com.
96      IN      PTR     ns2.sc.weidong.com.
```

修改权限

```sh
chmod 640 weidong.com.zone rev.weidong.com.zone 
chown :named weidong.com.zone rev.weidong.com.zone
```
**配置 `weidong.com` 域从服务器**

修改 `named.conf` 配置文件 

```sh
options {
        listen-on port 53 { 192.168.1.97; 127.0.0.1; };
        directory       "/var/named";
        allow-query     { any; };
        
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;
        
zone "." IN {
        type hint;
        file "named.ca";
};

zone "weidong.com" IN {
        type slave;
        masters { 192.168.1.98; };
        file "slaves/weidong.com.zone";
        allow-update { none; };
        allow-transfer { none; };
};

zone "1.168.192.in-addr.arpa" IN {
        type slave;
        masters { 192.168.1.98; };
        file "slaves/rev.weidong.com.zone";
        allow-update { none; };
        allow-transfer { none; };
};
```

导入 `.（根）` 域 NS 记录

```sh
dig -t NS . @192.168.1.100 > /var/named/named.ca
```

启动 `bind` 服务

```sh
systemctl start named
```

检查解析库是否同步成功

```sh
ls -l slaves/
-rw-r--r-- 1 named named 528 Sep 27 13:53 rev.weidong.com.zone
-rw-r--r-- 1 named named 561 Sep 27 13:51 weidong.com.zone
```



**子域 `sc.weidong.com` 主服务器配置**

修改 `named.conf` 配置文件

```sh
options {
        listen-on port 53 { 192.168.1.96; 127.0.0.1; };
        directory       "/var/named";
        allow-query     { any; };
        
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "sc.weidong.com" IN {
        type master;
        file "sc.weidong.com.zone";
        allow-update { 192.168.1.95; };
        allow-transfer {192.168.195; };
};

zone "1.168.192.in-addr.arpa" IN {
        type master;
        file "rev.sc.weidong.com.zone";
        allow-update { 192.168.1.95; };
        allow-transfer {192.168.195; };
};

;对父域做转发
zone "weidong.com" IN {
        type forward;
        forward only;
        forwarders  { 192.168.1.98; };
};
```

导入 `.（根）` 域 NS 记录

```sh
dig -t NS . @192.168.1.100 > /var/named/named.ca
```

创建 `sc.weidong.com` 正向解析库文件 `sc.weidong.com.zone`

 ```sh
$TTL 86400
$ORIGIN sc.weidong.com.
@       IN      SOA     ns1.sc.weidong.com.     admin.weidong.com. (
                                       20170927  ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns1.sc.weidong.com.
        IN      NS      ns2.sc.weidong.com.
ns1.sc.weidong.com.     IN      A       192.168.1.96
ns2.sc.weidong.com.     IN      A       192.168.1.95
webserver.sc.weidong.com.       IN      A       192.168.1.96
webserver.sc.weidong.com.       IN      A       192.168.1.95
www.sc.weidong.com.     IN      CNAME   webserver.sc.weidong.com.
 ```



创建 `sc.weidong.com`  域反向解析库文件 `rev.sc.weidong.com.zone`

```sh
$TTL 86400
$ORIGIN 1.168.192.in-addr.arpa.
@       IN      SOA     ns1.sc.weidong.com      admin.weidong.com. (
                                        20170927 ; serial
                                        1D       ; refresh
                                        1H       ; retry
                                        1W       ; expire
                                        3H )     ; minimum
        IN      NS      ns1.sc.weidong.com.
        IN      NS      ns2.sc.weidong.com.
96      IN      PTR     ns1.sc.weidong.com.
95      IN      PTR     ns2.sc.weidong.com.
96      IN      PTR     webserver.sc.weidong.com.
95      IN      PTR     webserver.sc.weidong.com.
```

修改文件权限

```sh
chmod 640 sc.weidong.com.zone rev.sc.weidong.com.zone
chown :named sc.weidong.com.zone rev.sc.weidong.com.zone
```

启动 `bind` 服务

```sh
systemctl start named
```

**子域 `sc.weidong.com` 从服务器配置**

修改 `named.conf` 配置文件

```sh
options {
        listen-on port 53 { 192.168.1.95; 127.0.0.1; };
        directory       "/var/named";
        allow-query     { any; };
        
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "sc.weidong.com" IN {
        type slave;
        masters { 192.168.1.96; };
        file "slaves/sc.weidong.com.zone";
};

zone "1.168.192.in-addr.arpa" IN {
        type slave;
        masters { 192.168.1.96; };
        file "slaves/rev.sc.weidong.com.zone";
};

zone "weidong.com" IN {
        type forward;
        forward only;
        forwarders { 192.168.1.98; };
};
```

导入 `.（根）` 域 NS 记录

```sh
dig -t NS . @192.168.1.100 > /var/named/named.ca
```

启动 `bind` 服务

```sh
systemctl start named
```

检查解析库是否同步成功

```sh
-rw-r--r-- 1 named named 434 Sep 27 15:39 rev.sc.weidong.com.zone
-rw-r--r-- 1 named named 419 Sep 27 13:54 sc.weidong.com.zone
```



**配置 ISP DNS SERVER**

ISP DNS 只需要做 dns 缓存与递归，用户在向他请求时，他无法解析的地址需要从 `.(根)` 上去找，网络上的 DNS 服务器通常是不做递归查询的，当然用户端也是无法做递归的，所以需要由 ISP 的 DNS 服务器来为用户做递归解析查询。

修改 `named.conf` 配置文件

```sh
options {
        listen-on port 53 { 192.168.1.94; 127.0.0.1; };
        directory       "/var/named";
        allow-query     { any; };
        
        recursion yes;

        dnssec-enable no;
        dnssec-validation no;
};

zone "." IN {
        type hint;
        file "named.ca";
};
```

导入 `.（根）` 域 NS 记录

```sh
dig -t NS . @192.168.1.100 > /var/named/named.ca
```



**客户端测试**

客户端 DNS 服务指向 ISP DNS 使用 `dig` 命令做跟踪查询（ `trace`）

```sh
[root@client ~]# cat /etc/resolv.conf 
# Generated by NetworkManager
nameserver 192.168.1.94
[root@client ~]# dig www.sc.weidong.com +trace

; <<>> DiG 9.9.4-RedHat-9.9.4-37.el7 <<>> www.sc.weidong.com +trace
;; global options: +cmd
.                       74827   IN      NS      ns.root.
;; Received 64 bytes from 192.168.1.94#53(192.168.1.94) in 3 ms

com.                    86400   IN      NS      ns.com.
;; Received 80 bytes from 192.168.1.100#53(ns.root) in 3 ms

weidong.com.            86400   IN      NS      ns1.weidong.com.
weidong.com.            86400   IN      NS      ns2.weidong.com.
;; Received 115 bytes from 192.168.1.99#53(ns.com) in 5 ms

sc.weidong.com.         86400   IN      NS      ns1.sc.weidong.com.
sc.weidong.com.         86400   IN      NS      ns2.sc.weidong.com.
;; Received 115 bytes from 192.168.1.98#53(ns1.weidong.com) in 9 ms

www.sc.weidong.com.     86400   IN      CNAME   webserver.sc.weidong.com.
webserver.sc.weidong.com. 86400 IN      A       192.168.1.95
webserver.sc.weidong.com. 86400 IN      A       192.168.1.96
sc.weidong.com.         86400   IN      NS      ns1.sc.weidong.com.
sc.weidong.com.         86400   IN      NS      ns2.sc.weidong.com.
;; Received 171 bytes from 192.168.1.96#53(ns1.sc.weidong.com) in 0 ms

```



END!