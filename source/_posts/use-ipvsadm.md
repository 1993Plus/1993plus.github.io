---
title: ipvsadm 命令的使用
date: 2017-10-28 22:32:17
category: LVS
tags:
	- LVS
	- ipvsadm
---



**ipvsadm 的使用**



程序包名：`ipvsadm`

主程序：/usr/sbin/ipvsadm

规则保存工具：/usr/sbin/ipvsadm-save

规则重载工具：/usr/sbin/ipvsadm-restore

配置文件：/etc/sysconfig/ipvsadm-config



**ipvsadm 命令**

核心功能

集群服务管理：增、删、改、查

集群服务器的 RS 管理：增、删、改、查



**集群服务管理**

增加、修改

```sh
ipvsadm -A|E -t|u|f SERVER-ADDRESS:PORT [-s scheduler] [-p [timeout]] 
; -A: 增加
; -E: 修改
; -t: TCP 协议
; -u: UDP 协议
; -f: firewall Mark(标记，一个数字)
---------------------------------------------------------------------
; SERVER-ADDRESS:PORT
; -t: TCP 协议端口，VIP: TCP PORT
; -u: UDP 协议端口，VIP: UDP PORT
---------------------------------------------------------------------
; -s: 指定集群的调度算法，默认为 wlc
; -p: 超时时间
```

删除

```sh
ipvsadm -D -t|u|f SERVER-ADDRESS
```



**管理集群上的 RS**

增加、修改

```sh
ipvsadm -a|e -t|u|f SERVER-ADDRESS:PORT -r SERVER-ADDRESS:PORT [-g|i|m] [-w weight]
; -a: 增加
; -e: 修改
; -r: Real Server IP(如省略 PORT，不作端口映射)
; -g: gateway,DR 类型，默认
; -i: ipip,TUN 类型
; -m: masquerade, NAT 类型
; -w: 权值
```

删除

```sh
ipvsadm -d -t|u|f SERVER-ADDRESS -r SERVER-ADDRESS
```



清空定义的所有规则

```sh
ipvsadm -C
```

 清空计数器

```sh
ipvsadm -Z [-t|u|f SERVER-IPADDRESS]
```

查看

```sh
ipvsadm -L [OPTION]
; --numeric,-n	以数字形式输出地址和端口号
; --exact	扩展信息，精确值
; --connection,-c	当前 IPVS 连接输出
; --stats	统计信息
; --rate	输出速率信息
```

ipvs 规则：`/proc/net/ip_vs`

ipvs 连接：`/proc/net/ip_vs_conn`



**保存与重载规则**

```sh
; 保存规则
ipvsadm-save > /PATH/IPVSADM_FILE
ipvsadm -S > /PATH/IPVSADM_FILE

;重载规则
ipvsadm-restore < /PATH/IPVSADM_FILE
ipvsadm -R < /PATH/IPVSADM_FILE
```



END!