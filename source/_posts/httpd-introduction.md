---
title: Httpd 简介
date: 2017-10-06 22:07:32
category: HTTPD
tags:
	- web
	- httpd
	- apache
---

HTTPD

20 世纪 90 年代初，美国国家超级计算机应用中心 NCSA  开发
1995 年开源社区发布 Apache（a patchy server）
ASF: Apache Software Foundation
FSF: Free Software Foundation

特性：

- 高度模块化: core + modules
- DSO: Dynamic Shared Object 动态添加/卸载
- MPM: Multi-Processing Modules 多路处理模块


MPM 工作模式

- prefork: 多进程 I/O 模型，每个进程响应一个请求，默认模型
  一个主进程：生成和回收 N 个子进程，创建套接字，不响应请求
  多个子进程：工作 work 进程，每个子进程处理一个请求，系统初始时，预先生成多个空闲进程，等待请求，最多不超过 1024 个

- worker: 复用的多进程 I/O 模型，多进程多线程，IIS 使用此模型
  一个主进程：生成 M 个子进程，每个子进程负责生成 N 个线程，每个线程响应一个请求，并发响应请求 M*N

- event: 事件驱动模型（worker 模型的变种）
  一个主进程：生成 M 个子进程，每个进程直接响应 N 个请求，并发响应请求：M*N，有专门的线程来管理这些 keep-alive 类型的线程，当有真实响应请求时，将请求传递给服务线程，执行完毕后，又允许释放。这样增强了高并发场景下的请求处理能力

HTTPD 功能特性
虚拟主机
IP，Port，FQDN
CGI: Common Gateway Interface ，通用网关接口
反向代理
负载均衡
路径别名
丰富的用户认证机制
* basic
* digest

支持第三方模块



