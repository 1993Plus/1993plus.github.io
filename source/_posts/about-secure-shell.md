---
title: 关于 SSH
date: 2017-10-17 22:44:14
category: Linux
tags:
	- ssh
	- secure shell
---
**SSH**

SSH( Secure Shell ) protocol ,22/tcp, 安全的远程登录

**具体的软件实现**

OpenSSH : SSH 协议的开源实现，CentOS 默认安装
dropbear : 另一个开源实现



**SSH 协议版本**

V1 : 基于 CRC-32 做 MAC，不安全；man-in-middle
V2 : 双方主机协议选择安全的 MAC 方式
基于 DH 算法做密钥交换，基于 RSA 或 DSA 实现身份认证

**两种用户登录认证方式**

基于 password
基于 key



**相关包**

openssh
openssh-clients
openssh-server



**工具**

基于 C/S 结构
Client：ssh，scp，sftp，slogin
Windows 客户端：xshll，putty，SecureCRT，sshsecureshellclient
Server：sshd



**客户端组件**

ssh
配置文件 `/etc/ssh/ssh_config`
Host PATTERN
StrictHostKeyChecking no 首次登录不显示检查提示

**openssh客户端**

```sh
 ssh [USER@]HOST[COMMAND]
 ssh [-l USER] HOST [COMMAND]
 -p port	;远程服务器监听的端口
 -b			;指定连接的源 IP
 -v			;调试模式
 -C			;压缩方式
 -X			;支持 X11 转发
 -y			;支持信任 X11 转发
 -t			;强制伪 tty 分配
 ssh -t RemoteServer1 ssh RemoteServer2
```

**允许实现对远程系统经验证地加密安全访问**

当用户远程连接 ssh 服务器时，会复制 ssh 服务器 `/etc/ssh/ssh_host*key.pub` (Centos7 默认是 `ssh_host_ecdsa_key.pub`) 文件中的公钥到客户机的 `~./ssh/know_hosts` 中；下次连接时，会自动匹配相应的私钥，不能匹配，将拒绝连接。

**ssh 服务登录验证方式**

用户/口令
基于密钥



**基于用户/口令登录验证**

1. 客户端发起 ssh 请求，服务器会把自己的公钥发送给用户
2. 用户会根据服务器发来的公钥对密码进行加密
3. 加密后的信息回传给服务器，服务器用自己的私钥解密，如果密码正确，则用户登录成功

**基于密钥的登录方式**

1. 首先在客户端生成一对密钥（ssh-keygen）
2. 并将客户端的公钥 ssh-copy-id 拷贝到服务端
3. 当客户端再次发送一个连接请求，包括 IP，用户名
4. 服务端得到客户端的请求后，会到 authorized_keys 中查找，如果有响应的 IP 和用户，就会随机生成一个字符串，例如：abcd
5. 服务端将使用客户端拷贝过来的公钥进行加密，然后发送给客户端
6. 得到服务端发送来的消息后，客户端会使用私钥进行解密，然后将解密后的字符串发送给服务端
7. 服务端接收到客户端发来的字符串后，与之前的字符串进行对比，如果一致，就运行免密码登录


![示意图](http://ov2iiuul1.bkt.clouddn.com/ssh1.png)



END!