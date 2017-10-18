---
title: Linux 客户端基于 SSH key 认证登录 Linux Server
date: 2017-10-17 22:49:09
category: Linux
tags:
	- ssh
	- ssh key
---

**Linux 客户端基于 SSH key 认证登录 Linux Server**



**客户端生成密钥对**

```sh
;-t 指定加密类型，可用类型 dsa | ecdsa | ed25519 | rsa | rsa1
ssh-keygen -t rsa
Generating public/private rsa key pair.
;生成的密钥文件保存路径
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
;是否给私钥加密，如果不需要加密直接回车就可以了
Enter passphrase (empty for no passphrase): 
;再次输入密码
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:vyeCnEDO7d63PHHcDGrWOHLrtLj9Q8ZJZt+cDJ4kgJ0 root@localhost.localdomain
The key's randomart image is:
+---[RSA 2048]----+
|        o .      |
|       . E       |
|          .      |
|    .      .=o   |
|   + .  S  X+*+o.|
|    + . ..O Oo+oo|
|     + o =oB     |
|      =..==oo    |
|     .. +=B*..   |
+----[SHA256]-----+
```

更改私钥文件密码

```sh
;更改私钥密码使用 -p
ssh-keygen -p
确认要更改密码的私钥文件
Enter file in which the key is (/root/.ssh/id_rsa): 
;输入旧密码
Enter old passphrase: 
;输入新密码
Enter new passphrase (empty for no passphrase): 
;重新输入新密码
Enter same passphrase again: 
Your identification has been saved with the new passphrase.
```



将客户端生成的公钥文件拷贝到服务器上

```sh
 ;使用 ssh-copy-id 工具拷贝
 ; -i 指定公钥文件路径，跟上服务器登录用户名加主机地址
 ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.1.100
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '192.168.1.100 (192.168.1.100)' can't be established.
RSA key fingerprint is SHA256:tZu64oYqpse8To1kNaL+c/1+bvBFxgFZsFjvM1eqXBg.
RSA key fingerprint is MD5:85:bd:5d:3b:a5:35:fd:a7:43:e2:16:e3:d8:96:97:51.
;如果是第一次连接该主机会出现该提示，输入 yes
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
;输入服务器登录密码
root@192.168.1.100's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.1.100'"
and check to make sure that only the key(s) you wanted were added.

```

使用 `ssh-copy-id` 这个工具拷贝过去后在服务端用户 `home` 目录下会自动创建一个 `.ssh` 的隐藏文件夹，并且会将客户端的公钥文件改名为 `authorized_keys` ，如果这个文件原本就存在，那么该私钥只是会追加在这个文件中

```sh
cd .ssh/
ls -l
-rw------- 1 root root 408 Oct 11 16:05 authorized_keys
cat authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDAWLQik92axbFwYczZ/Z7xlaa3J0kTrqf8wzU8NQi07D4YkYAJVMqtotzIf1E4koOJ9v5yw4JbVLY/Fc9/KD3xvW8lHyoZkUdHr3XH9D8/AYQ92Va/tUlCfvxNR7ozo6MyfbAYrVnpHuwqdR6+q6347W/dJheTABkuh7uQ9n0pdEBWOZyuTSFanj8PydfU2aSGSYz7QBkaEDyJGsP5V7R9SH3YAVM1sQGWaaAzbBON2RuaQhrZH8DO4xZ+KPW+tifpuWyNIEL8jF18lb2qQH972lUW5VtHtMrPFJOPUFNs06NCcYoQUSmlLxKc78enMEsf0/kin3lOvWFsUusd2bxf root@localhost.localdomain
```

现在就可以使用 ssh 客户端工具来基于 key 登录服务器了

```sh
ssh root@192.168.1.100
;如果私钥文件是加密了的，那在登录的时候还需要输入私钥密码
Enter passphrase for key '/root/.ssh/id_rsa': 
```

**启用代理**

如果有多台服务器需要操作的话，那每次输入私钥密码也是一件很麻烦的事，解决这个问题的办法就是使用代理

它可以让你在退出客户端之前不再输入私钥密码，但下次还是需要输入密码的

验证代理（authentication agent）保密解密后的密钥

```sh
;启用代理(在 GNOME 中，代理被自动提供给 root 用户)
ssh-agent bash
;使用 ssh-add 将钥匙添加给代理
ssh-add 
;输入私钥密码
Enter passphrase for /root/.ssh/id_rsa: 
Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)
```



**注意事项**

对于私钥文件，如果落入到黑客手中将是一件非常可怕的事，所以一般情况下建议给私钥文件加密，来保护私钥文件。



END!