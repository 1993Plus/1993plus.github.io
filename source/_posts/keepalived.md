---
title: 关于 keepalived
date: 2017-11-04 15:55:13
category: keepalived
tags:
	- keepalived
---

**Keepalived**

Keepalived 是一个基于 VRRP 协议来实现的 LVS 服务高可用方案，可以利用 keepalived 来避免单点故障（Single Ponit of Failure）。

**工作原理：**

keepalived 是以 VRRP 协议为实现基础的， VRRP 全称 Virtual Router Redundancy Protocol，即虚拟冗余路由协议。虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将 N 台提供相同功能的路由器组成一个路由器组，这个组里面有一个 master 和 多个 backup，master 上面有一个对外提供服务器的 VIP ，master 会发组播，当 backup 收不到 vrrp 包时就认为 master 宕掉了，这是就需要根据 VRRP 的优先级来选举一个 backup 当 master。这样的话就可以保证路由器的高可用了。

**keepalived 的配置文件**

keepalived 只有一个配置文件 keepalived.conf（`/etc/keepalived/keepalived.conf`），里面主要包括以下几个配置区域。

`global_defs` 主要配置发生故障是的通知对象及主机标识

```
global_defs {
; 故障发送生时，邮件通知地址
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
; 通知邮件的发送地址
   notification_email_from Alexandre.Cassen@firewall.loc
; 通知邮件 smtp 地址
   smtp_server 192.168.200.1
; 连接 smtp 服务器的超时时间
   smtp_connect_timeout 30
; 节点标识符，通常为 hostname
   router_id LVS_DEVEL
; 组播地址
   vrrp_mcast_group4 255.255.255.255
}
```

`vrr_script` 自定义资源监控脚本，vrrp 示例根据脚本返回值，公共定义，可被多个示例调用，定义在 vrrp 实例之外

```
; 定义一个脚本名
vrrp_script SCRIPT_NAME {
; 脚本语句块或脚本完整路径
  script "script"
; 检测间隔
  intervl 1
; 当脚本执行结果条件满足时权重值为
  weight -20
; 检测多少次失败结果才算失败
  fall 2
; 检测多少次成功才算成功
  rise 1
}
```

`vrrp_instance` 用来定义对外提共服务器的 VIP 区域及相关属性

```
vrrp_instance VI_1 {
; 当前节点在此虚拟路由上的初始状态，master|backup
    state MASTER
; 绑定为当前虚拟路由使用的物理接口
    interface eth0
; 虚拟路由唯一标识符，范围 0-255
    virtual_router_id 51
; 当前物理节点在此虚拟路由中的优先级，范围 1-254
    priority 100
; VRRP 包发送间隔，默认 1s
    advert_int 1
; 认证机制
    authentication {
; 密码类型 AH|PASS
        auth_type PASS
; 密码(PASS 最多只能识别 8 位)
        auth_pass 1111
    }
; 虚拟 IP(VIP)
    virtual_ipaddress {
        192.168.200.17/24
        192.168.200.18/24 dev eth0 label eth0:1
    }
; 监控网卡列表，当一下列表中的网卡测试失败时，切换到 FALT 状态
    track_interface {
      eth1
    }
; 调用 vrr_script 定义的脚本
    track_script{
      SCRIPT_NAME
    }
; 定义工作模式为非抢占模式。允许一个 priority 较低的节点作为 master，即使有 priority 更高的节点
    nopreempt
; master 节点启动多久后进行接管资源
    preempt_delay 300
; 当前节点切换为 master 节点时触发脚本
    notify_master <STRING>|<QUOTED-STRING>
; 当前节点切换为 backup 节点时触发脚本
    notify_backup <STRING>|<QUOTED-STRING>
; 当前节点切换为失败状态时触发脚本
    notify_fault <STRING>|<QUOTED-STRING>
; 以上任意状态时触发脚本
    notify <STRING>|<QUOTED-STRING>
}
```

`virtual_server`

```
; 接收外部请求的 VIP 地址与端口(对应 vrrp_instance 中配置的 VIP 地址)
virtual_server 192.168.200.100 443 {
; 对后端服务器的检查间隔(单位为秒)
    delay_loop 6
; 对后端服务器的调度算法(rr|lc|wlc|lblc|sh|dh)
    lb_algo rr
; LVS 调度类型( NAT|DR|TUN)
    lb_kind NAT 
; 持续连接时长
    persistence_timeout 50
; 服务协议类型
    protocol TCP 

; 当后端所有 real_server 都不可用时由 sorry_server 接收请求
    sorry_server 192.168.200.200 1358

; 后端提供服务的真实服务器
    real_server 192.168.201.100 443 {
; 权重
        weight 1
; 健康状态检测方式(HTTP_GET|SSL_GET|TCP_CHECK|SMTP_CHECK|MISC_CHECK..)
        SSL_GET {
; 检测 URL
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }   
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }  
; 连接请求的超时时长
            connect_timeout 3
; 重试次数
            nb_get_retry 3
; 重试间隔
            delay_before_retry 3
; 向 real_server 指定 IP 地址发起健康状态检测请求
            connect_ip 192.168.200.100
; 向 real_server 指定端口发起健康状态检测请求
            connect_port 22
; 发出健康状态检测请求时使用的源地址
            bindto 192.168.200.100
; 发出健康状态检测请求时使用的源端口
            bind_port 22
        }   
    }   
}
```

在下面这个目录中还有很多模板文件可以作为参考

```
/usr/share/doc/keepalived-1.3.5/samples
```



END!
