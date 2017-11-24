---
title: LVS 集群中的术语
date: 2017-10-28 21:34:42
category: LVS
tags:
	- lvs
---

**LVS 集群中的术语**

VS: Virtual Server， Director，Dispatcher(调度器)，Load Balancer

RS: Real Server(LVS)，upstream server(nginx)，backend server(haproxy)

CIP: Client IP

VIP: Virtual Server IP(VS 外网 IP)

DIP: Director IP(VS 内网 IP)

RIP: Real Server IP



`访问流程： CIP <—> VIP <===> DIP <—>  RIP`



**LVS: ipvsadm/ipvs**

ipvsadmin: 用户空间的命令行工具，规则管理器。用于管理集群服务及 Real Server

ipvs: 工作于内核弓箭 `netfilter` 的 `INPUT` 钩子上的框架



**LVS 集群类型**

* LVS-NAT: 修改请求报文的目标 IP，多目标 IP 的 DNAT
* LVS-DR: 操纵封装新的 MAC 地址
* LVS-TUN: 在原请求 IP 报文之外新加一个 IP 首部
* LVS-FULLNAT: 修改请求报文的源和目标 IP(kernel 默认不支持，需要自行编译)



**LVS-NAT**

本质是多目标 IP 的 DNAT，通过将请求报文中目标地址和目标端口修改为某选出的 RS 的 RIP 与 PORT 实现转发

1）RIP 和 DIP 必须同在一个 IP 网络，且应该使用私网地址；RS 的网关要指向 DIP

2）请求报文和响应报文都必须经由 Director 转发，Director 容易成为系统瓶颈

3）支持端口映射，可修改请求报文的目标 PORT

4）VS 必须是 Linux 系统，RS 可以是任意 OS



**LVS-DR**

LVS-DR(Direct Routing)，直接路由，LVS 默认模式，应用最广泛，通过为请求报文重新封装一个 MAC 首部进行转发，源 MAC 是 DIP 所在的接口的 MAC，目标 MAC 是某挑选出的 RS 的 RIP 所在接口的 MAC 地址；源 IP 和 PORT 以及目标 IP 和 PORT 均保持不变

`Director 和各个 RS 都配置有 VIP`

1）确保前段路由器将目标 IP 为 VIP 的请求报文发往 Director

前端网关做静态绑定 VIP 和 Director 和 MAC 地址

在 RS 上使用 arptables 工具

在 RS 上修改内核参数以及限制 arp 通告及应答级别

`arp_announce`

`arp_ignore`

2）RS 的 RIP 可以使用私网地址，也可以是公网地址；RIP 与 DIP 在同一 IP 网络；RIP 的网关不能指向 DIP，以确保响应报文不会经由 Director

3）RS 和 Director 要在同一物理网络

4）请求报文要经由 Director，但响应报文不经由 Director，而由 RS 直接返回给 Client

5）不支持端口映射（端口不能修改）

6）RS 可以使用大多数 OS



**LVS-TUN**

不修该请求报文的 IP 首部（源 IP 为 CIP，目标 IP 为 VIP），而是在原 IP 报文之外再分装一个 IP 首部（源 IP 为 DIP，目标 IP 为 RIP），将报文发给选出的目标 RS；RS 处理后直接返回给客户端（源 IP 为 VIP，目标 IP 为 CIP）

1）DIP，VIP，RIP 都应该是公网地址

2）RS 的网关不能，也不可能指向 DIP

3）请求报文要经由 Director，但响应不能经由 Director

4）不支持端口映射

5）RS 的 OS 必须支持隧道功能



**LVS-FULLNAT**

通过同时修改请求报文的源 IP 地址和目标 IP 地址进行转发

1）VIP 是公网地址，RIP 和 DIP 是私网地址，且通常不再同一 IP 网络；因此，RIP 的网关一般不会指向 DIP

2）RS 收到的请求报文源地址是 DIP ，因此，只需要响应给 DIP；但 Director 还要将其发往 Client

3）请求和响应报文都经由 Director

4）支持端口映射；