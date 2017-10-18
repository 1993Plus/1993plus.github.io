---
title:  SecureCRT 设置基于 SSH key 登录
date: 2017-10-18 09:46:26
category: Linux
tags:
	- SSH KEY
	- Linux
	- SecureCRT
---



**SecureCRT 基于 SSH KEY 登录 Linux Server**



在 SecureCRT 中生成密钥对

![生成密钥对](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key1.png)

点击 Continue 继续

![Continue](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key2.png)

选择加密方式，确认后点击 Continue

![选择加密方式](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key3.png)

1. 输入私钥文件密码
2. 输入备注信息

![设置密码与备注信息](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key4.png)

输入密钥加密长度，在 512 到 16384 之间

![输入密钥长度](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key5.png)

1. 选择密钥格式，默认就好了，因为到服务器上可以使用工具进行格式转换
2. 选择密钥文件保存路径

![选择密钥格式](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key7.png)

是否将该密钥加入到全局设置中，如果点了 yes 那所有主机默认都会使用该密钥

![](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key8.png)

将生成的公钥文件传到服务器上并进行格式转换

```sh
;在用户 home 目录创建公钥文件存放目录
mkdir .ssh
;公钥原格式
cat identity.pub 
---- BEGIN SSH2 PUBLIC KEY ----
Subject: Zhaoweidong
Comment: "Zhaoweidong@MacBook-Pro.local"
ModBitSize: 2048
AAAAB3NzaC1yc2EAAAADAQABAAABAQD0hak6HykrVOqZg61FlnCbyQXPXOW75Hm6
dCokhC3bqAmNkJMdvsRK5dQfaDNebpYogax/e/edc2TedS+RYA7I3eTkTuqcU302
+TAewcVrqel1b2qrSQ370iE/z4JQ6da6+q/FUDFfHp06Ww0CFIABcUBYLo3B4aG8
Q5VcMhmh2Z8hBV5Kjrjy2ALpytxiXvi36NRJiPTSg+W5QNnCUPgXdoWIVGuXfX7U
qp2ZOsHFuQoX2dydb5t+SLQDFTc7yf/aBBraSbqGxRklE+vztnv4RHda4ZjyjaYo
LpWsPv90ZAuRXYOJ95PVHTEaqIMlMd0MBnObgZjCp6eZmTQzGRh3
---- END SSH2 PUBLIC KEY ----

;使用 ssh-keygen 转换为 OpenSSH key 格式；
;将转换后的格式通过重定向保存到用户 home 目录下 .ssh/authorized_keys 文件中
ssh-keygen -i -f identity.pub >> /root/.ssh/authorized_keys

;转换后的格式
cat /root/.ssh/authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD0hak6HykrVOqZg61FlnCbyQXPXOW75Hm6dCokhC3bqAmNkJMdvsRK5dQfaDNebpYogax/e/edc2TedS+RYA7I3eTkTuqcU302+TAewcVrqel1b2qrSQ370iE/z4JQ6da6+q/FUDFfHp06Ww0CFIABcUBYLo3B4aG8Q5VcMhmh2Z8hBV5Kjrjy2ALpytxiXvi36NRJiPTSg+W5QNnCUPgXdoWIVGuXfX7Uqp2ZOsHFuQoX2dydb5t+SLQDFTc7yf/aBBraSbqGxRklE+vztnv4RHda4ZjyjaYoLpWsPv90ZAuRXYOJ95PVHTEaqIMlMd0MBnObgZjCp6eZmTQzGRh3
```

将公钥登录方式设置为首选项，如果你没有选择登录要使用的私钥文件，还需要点 `Properties` 选项来选择私钥文件

![设置登录首选项](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key9.png)



选择登录该主机要使用的私钥文件

![选择私钥文件](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key10.png)

现在就可以使用基于 KEY 的方式来登录服务器了，如果你的私钥文件设置了密码，还需要输入一次私钥文件的密码

![输入私钥文件密码](http://ov2iiuul1.bkt.clouddn.com/securecrt_ssh_key_13.png)



END!