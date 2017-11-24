---
title: Keepalived LVS DR 模型高可用实验
date: 2017-11-07 09:24:53
category: keepalived
tags:
    - keepalived
    - lvs
---



**Keepalived LVS DR 模型高可用实验**

**实验环境**

LVS Virtual Server 以下简称 VS，后端 Real Server 简称 RS。

操作系统：

> VS: CentOS 7.4
>
> RS: CentOS 7.4
>
> Router: RouterOS

IP 地址：

> VS1: VRRP 通信地址 192.168.11.101/24 ,DIP 192.168.12.101/24,{VIP 192.168.11.100/24}
>
> VS2: VRRP 通信地址 192.168.11.102/24 ,DIP 192.168.12.101/24,{VIP 192.168.11.100/24}
>
> RS1: RIP 192.168.12.121, VIP 192.168.11.100/32
>
> RS2: RIP 192.168.12.122, VIP 192.168.11.100/32
>
> Router：WAN eth0 192.168.10.254/24, LAN1 eth1 192.168.11.254/24, LAN2 eth2 192.168.12.254/24
>
> Client: 192.168.10.1/24



网络拓扑结构

![keepalived lvs dr](http://ov2iiuul1.bkt.clouddn.com/keepalived+lvs.png)

**VS keepalived 配置**

为了方便区分可以将主机名修改了

VS1

```
hostnamectl set-hostname vs-node1
```

VS2

```
hostnamectl set-hostname vs-node2
```

在两台 VS 上安装 keepalived 软件包

```
yum install -y keepalived
```

**MASTER 配置**

修改 keepalived 配置文件(`/etc/keepalived/keepalived.conf`)

```
! Configuration File for keepalived

global_defs {
   notification_email {
     1993plus@gmail.com
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   ; 节点名
   router_id vs-node1
   ; 使用自定义组播地址(两台 VS 必须在同一组播地址中)
   vrrp_mcast_group4 224.39.39.39
}

vrrp_instance VI_1 {
	; 默认状态
    state MASTER
    ; VIP 网卡名
    interface eth0
    ; 虚拟路由 ID (两台 VS 必须填一样的 ID)
    virtual_router_id 220
    ; 优先级 ( MASTER 的优先级应比 BACKUP 高)
    priority 100
    ; VRRP 包发送间隔
    advert_int 2
    authentication {
        auth_type PASS
        ; 认证密码 ( 两台 VS 必须填一样的密码)
        auth_pass weidong
    }
    virtual_ipaddress {
    	; VIP 地址，为了方便区分可以将该地址绑定在一个网卡别名上
        192.168.11.100/24 dev eth0 label eth0:1 broadcast 192.168.11.255
    }
}

virtual_server 192.168.11.100 80 {
	; 对 RS 的检测间隔
    delay_loop 3
    ; 调度算法
    lb_algo rr
    ; LVS 模式
    lb_kind DR
    ; 协议
    protocol TCP

	; 当后端所有 RS 宕机后又本机的 80 端口接管服务
    sorry_server 127.0.0.1 80

    real_server 192.168.12.121 80 {
        weight 1
        HTTP_GET {
            url {
              path /index.html
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.12.122 80 {
        weight 1
        HTTP_GET {
            url {
              path /index.html
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

启动  keepalived 服务

```
systemctl start keepalived.service
```

查看 IP 地址

```
[root@vs-node1 ~]# ifconfig eth0:1
eth0:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.11.100  netmask 255.255.255.0  broadcast 192.168.11.255
        ether 00:50:56:33:68:16  txqueuelen 1000  (Ethernet)

```

看到 VIP 已经被 keepalived 自动配置到网卡上了

在 BACKUP 上使用 tcpdump 抓包，看能否收到 MASTER 发送的 VRRP 包

```
[root@vs-node2 ~]# tcpdump -i eth0 -nn host 224.39.39.39
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
09:49:07.661233 IP 192.168.11.101 > 224.39.39.39: VRRPv2, Advertisement, vrid 220, prio 100, authtype simple, intvl 2s, length 20
09:49:09.662978 IP 192.168.11.101 > 224.39.39.39: VRRPv2, Advertisement, vrid 220, prio 100, authtype simple, intvl 2s, length 20
```

安装 ipvsadm 包

```
yum install -y ipvsadm
```

使用 ipvsadm 查看 ipvs 列表

```
[root@vs-node1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.11.100:80 rr
  -> 127.0.0.1:80                 Route   1      0          0  
```

由于后端 RS 还未配置所以现在是有本机接管的 80 端口

**BACKUP 配置**

修改 keepalived 配置文件(`/etc/keepalived/keepalived.conf`)

```
! Configuration File for keepalived

global_defs {
   notification_email {
     1993plus@gmail.com
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id vs-node2
   vrrp_mcast_group4 224.39.39.39
}

vrrp_instance VI_1 {
	; 默认状态为 BACKUP
    state BACKUP
    interface eth0
    virtual_router_id 220
    priority 90
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass weidong
    }
    virtual_ipaddress {
        192.168.11.100/24 dev eth0 label eth0:1 broadcast 192.168.11.255
    }
}

virtual_server 192.168.11.100 80 {
    delay_loop 6
    lb_algo rr 
    lb_kind DR
    protocol TCP

    sorry_server 127.0.0.1 80

    real_server 192.168.12.121 80 {
        weight 1
        HTTP_GET {
            url { 
              path /index.html
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.12.122 80 {
        weight 1
        HTTP_GET {
            url { 
              path /index.html
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

启动  keepalived 服务

```
systemctl start keepalived.service
```

因为 BACKUP 的优先级低于 MASTER，所以当它发现组播中有比它优先级更高的主机存在，那它就会停止发送 VRRP 包，当然也不会接管 VIP。如果在 advert_int 设置时间内没有收到比它优先级更高的 VRRP 包，那它就会向网络中发送 VRRP 包，如果网络中没有比它优先级更高的主机存在时，那它就会接管 VIP，当然，如果设置了密码的话，还需要验证通过了才能接管。

停止 MASTER 的 keepalived 服务，测试 BACKUP 是否能成功接管 VIP

VS1

```
[root@vs-node1 ~]# systemctl stop keepalived.service 
```

VS2

```
[root@vs-node2 ~]# ifconfig eth0:1
eth0:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.11.100  netmask 255.255.255.0  broadcast 192.168.11.255
        ether 00:50:56:25:48:96  txqueuelen 1000  (Ethernet)
```

可以看到 VS 现在已经成功接管了 VIP

而 VS1 上已经没有 VIP 了，当然，如果把 keepalived 服务启动的话，它还会接管 VIP，因为他的优先级最高 

```
[root@vs-node1 ~]# ifconfig eth0:1
eth0:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:50:56:33:68:16  txqueuelen 1000  (Ethernet)
```



**配置后端 RS**

`因为是 DR 模式所以 RS 的网关是不能指向 VS 的，这里演示的网关为 192.168.12.254/24`

RS1 配置

```
hostnamectl set-hostname web-server1
```

安装 Apache httpd

```
yum install -y httpd
```

创建测试页面

```
[root@web-server1 ~]# echo web-server1 > /var/www/html/index.html 
```

启动 httpd

```
systemctl start httpd
```

关闭网卡 ARP 响应

```
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore 
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce 
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore 
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce 
```

将 VIP 添加到回环网卡（lo）上，并将广播地址设置为自己

```
ifconfig lo:1 192.168.11.100 netmask 255.255.255.255 broadcast 192.168.11.100 up
```

RS2 配置

RS1 配置

```
hostnamectl set-hostname web-server2
```

安装 Apache httpd

```
yum install -y httpd
```

创建测试页面

```
[root@web-server2 ~]# echo web-server2 > /var/www/html/index.html 
```

启动 httpd

```
systemctl start httpd
```

关闭网卡 ARP 响应

```
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore 
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce 
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore 
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce 
```

将 VIP 添加到回环网卡（lo）上，并将广播地址设置为自己

```
ifconfig lo:1 192.168.11.100 netmask 255.255.255.255 broadcast 192.168.11.100 up
```

**测试**

在 VS 上查看 ipvs 列表

```
[root@vs-node1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.11.100:80 rr
  -> 192.168.12.121:80            Route   1      0          0         
  -> 192.168.12.122:80            Route   1      0          0  
```

因为 keepalived 在做健康性检查的时候测试后端服务器是正常响应的所以就会将它们加入到内核的 ipvs 列表中

你会发现 VIP 地址没在 VS2 上，但它的 ipvs 列表中也有 RS。这个是正常的，因为虽然它也有 ipvs 列表，但 VIP 不在它上面所以用户的请求是不会发到 VS2 上面的。

现在可以到客户端进行测试了（因为我这里是在路由器上对 VIP 做的端口映射，所以访问的时候需要访问 192.168.10.254 的 80 端口，但效果是一样的）

用一条循环语句来测试

![keepalived lvs client test](http://ov2iiuul1.bkt.clouddn.com/keepalived%20lvs%20client_test.png)

因为使用的是轮叫算法（Round-Robin）所以它会轮流访问两台 RS

通过路由器来观察流量

![keepalived lvs routeros](http://ov2iiuul1.bkt.clouddn.com/keepalived%20lvs%20router.png)



END!

