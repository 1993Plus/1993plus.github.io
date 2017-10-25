---
title: scp 命令简单用法
date: 2017-10-18 10:44:10
category: Linux
tags:
	- scp
	- linux
---

**scp 命令**

程序包名

`openssh-clients`

命令格式

```sh
scp [OPTION] SRC ... DEST/
-C	压缩数据流
-r	递归复制(拷贝目录时使用)
-p	保留原文件的属性信息
-q	静默模式
-P PORT	指定 Remote Host 的监听端口
```

两种使用方式

* 将文件从本地拷贝到远程服务器上

命令格式

```
scp [OPTION] [USER@]HOST:/SOURCEFILE /DESTPATH
;如果在登录时不填用户名那将以本地当前登录的用户名登录远程主机
```

示例

```sh
;如果目标主机后面没有跟路径，默认为该用户的 home 目录
scp testfile root@192.168.1.100:
;将本地的 testfile 文件拷贝到远程服务器的 /var/ 目录下
scp testfile root@192.168.1.100:/var/
```

* 从远程服务器上拷贝文件到本地

命令格式

```
scp [OPTION] /SOURCEFILE [USER@]HOST:/DESTPATH
;如果在登录时不填用户名那将以本地当前登录的用户名登录远程主机
```

示例

```sh
;从远程服务器 root 用户 home 目录下拷贝 testfile 文件到本地当前所在目录
scp root@192.168.1.100:testfile ./
;长远程服务器 /var/ 目录下拷贝 testfile 文件到本地 /var/ 目录下
scp root@192.168.1.100:/var/testfile /var/
```





END!