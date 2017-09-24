---
title: Cobbler 基本应用
date: 2017-09-18 14:47:59
category: Linux
tags:
	- cobbler
	- 自动化安装系统
---



Cobbler 建立在 kickstart 机制上，它使用 DHCP、TFTP、DNS、等服务来通过网络为多计算机自动化安装操作系统。

安装包：

* 主程序包 cobbler
* web 管理 cobbler-web



cobbler 服务集成

* pxe 服务
* DHCP
* rsync
* http
* dns
* kickstart
* IPMI 电源管理



配置文件

* `/etc/cobbler/settings` cobbler 主配置文件
* `/etc/cobbler/iso/` iso 模板配置文件
* `/etc/coobler/pxe/` pxe 模板配置文件
* `/etc/cobbler/power/` 电源管理模板配置文件
* `/etc/cobbler/user.conf` web 服务授权配置文件
* `/etc/cobbler/user.digest` web 访问用户密码配置文件
* `/etc/cobbler/dhcp.template` dhcp 服务模板配置文件
* `/etc/cobbler/dnsmasq.template` dns 服务模板配置文件
* `/etc/coobler/tftp.template` tftp 服务模板配置文件 
* `/etc/cobbler/modules.conf` 模块配置文件 



数据目录

* `/var/lib/cobbler/config/` 存放 distros、system、profiles 等信息配置文件
* `/var/lib/cobbler/triggers/` 存放用户定义的 cobbler 命令
* `/var/lib/cobbler/kickstart/` 存放默认 kickstart 文件

镜像目录

* `/var/www/cobbler/ks_mirror/` 导入的发行版系统的所有数据
* `/var/www/cobbler/images/` 导入的发行版的 kernel 和 initrd 文件用户网络启动
* `/var/www/cobbler/repo_mirror/` yum 仓库存储目录

日志目录

* `/var/log/cobbler/installing.log` 客户端安装日志
* `/var/log/cobbler/cobbler.log` cobbler 日志

> **cobbler 命令**

`cobbler check ` 检测当前设置是否存在问题

`cobbler list` 列出所有 `cobbler` 元素

`cobbler report` 列出元素的详细信息

`cobbler sync` 同步配置到数据目录

`cobbler reposync` 同步 `yum` 仓库

`cobbler distro` 查看导入的发行版系统信息

`cobbler system` 查看添加的系统信息

`cobbler profile` 查看配置信息



**`安装前请关闭 SELinux`**



安装主程序包

```sh
[root@Cobbler ~]# yum install -y cobbler
```

查看`cobbler`服务是否为开机自启动（`cobbler` 的服务脚本名称为 `cobblerd`）

```sh
[root@Cobbler ~]# chkconfig --list cobblerd
cobblerd        0:off   1:off   2:off   3:on    4:on    5:on    6:off
## 如果将 cobblerd 设置为开机自启动？
[root@Cobbler ~]# chkconfig cobblerd on
```

修改 `cobbler` 配置文件

```sh
## 配置文件路径 /etc/cobbler/settings
## 允许 cobbler 管理 DHCP 服务
manage_dhcp: 1
## 运行 cobbler 管理 rsync 服务
manage_rsync: 1
## 提供 PXE 启动镜像的的 TFTP 服务器地址
next_server: 192.168.1.200
## 开启 PXE 启动
pxe_just_once: 1
## cobbler 服务地址
server: 192.168.1.200
```

配置客户端系统密码（也就是客户机在安装完系统后的 root 密码）

```sh
## 使用 openssl 生成一个密码
[root@Cobbler ~]# openssl passwd -1
Password: 
Verifying - Password: 
$1$VPwt6vsK$Z7uHjzFul98wT/vsyls3u1
## 将配置文件中的默认密码改为自己的密码
default_password_crypted: "$1$VPwt6vsK$Z7uHjzFul98wT/vsyls3u1"
```

启动 `cobbler` 服务

```sh
[root@Cobbler ~]# service cobblerd start
```

检测配置是否有问题

```sh
[root@Cobbler ~]# cobbler check         
httpd does not appear to be running and proxying cobbler, or SELinux is in the way. Original traceback:
Traceback (most recent call last):
  File "/usr/lib/python2.6/site-packages/cobbler/cli.py", line 252, in check_setup
    s.ping()
  File "/usr/lib64/python2.6/xmlrpclib.py", line 1199, in __call__
    return self.__send(self.__name, args)
  File "/usr/lib64/python2.6/xmlrpclib.py", line 1489, in __request
    verbose=self.__verbose
  File "/usr/lib64/python2.6/xmlrpclib.py", line 1235, in request
    self.send_content(h, request_body)
  File "/usr/lib64/python2.6/xmlrpclib.py", line 1349, in send_content
    connection.endheaders()
  File "/usr/lib64/python2.6/httplib.py", line 967, in endheaders
    self._send_output()
  File "/usr/lib64/python2.6/httplib.py", line 831, in _send_output
    self.send(msg)
  File "/usr/lib64/python2.6/httplib.py", line 790, in send
    self.connect()
  File "/usr/lib64/python2.6/httplib.py", line 771, in connect
    self.timeout)
  File "/usr/lib64/python2.6/socket.py", line 567, in create_connection
    raise error, msg
error: [Errno 111] Connection refused
```

上面报了一堆吓人的错误，这是因为 `httpd` 服务没有启动

因为使用 `yum` 方式安装 `cobbler` 系统会自动安装 `httpd`服务，所有现在只需要启动它即可

 ```sh
[root@Cobbler ~]# service httpd start
## 将 httpd 服务设置为开机自启动
[root@Cobbler ~]# chkconfig --list httpd
httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
[root@Cobbler ~]# chkconfig httpd on
 ```

再次检测配置问题

```sh
[root@Cobbler ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : dhcpd is not installed
2 : change 'disable' to 'no' in /etc/xinetd.d/tftp
3 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
4 : change 'disable' to 'no' in /etc/xinetd.d/rsync
5 : file /etc/xinetd.d/rsync does not exist
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : ksvalidator was not found, install pykickstart
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
```

**系统提示有 8 个问题**

**问题1:** `dhcp` 服务没有安装，安装 `dhcp` 程序包

```sh
[root@Cobbler ~]# yum install -y dhcp
## 安装完后 dhcp 还不能启动，因为还没有配置文件
```

同步设置

同步设置后 `cobbler` 会根据自己的 `dhcp` 模板配置文件来修改 `dhcp` 配置文件
如果需要修改 `dhcp` 配置选项可以修改 `dhcp` 模板后再同步

```sh
[root@Cobbler ~]# cobbler sync
```

启动 `DHCP` 服务

```sh
[root@Cobbler ~]# service dhcpd start
```

**问题2：**修改 `xinetd` 配置文件将 `tftp` 的禁用选项设置为 no

```sh
# default: off
# description: The tftp server serves files using the trivial file transfer \
#       protocol.  The tftp protocol is often used to boot diskless \
#       workstations, download configuration files to network-aware printers, \
#       and to start the installation process for some operating systems.
service tftp
{
        disable                 = no
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -B 1380 -v -s /var/lib/tftpboot
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
```



**问题3：**缺少启动文件，系统提示可以运行 `cobbler get-loaders` 下载这些文件，也可以通过安装 `syslinux` 包获取这些文件，在安装 `cobbler` 的时候系统也会自动安装这个包（`如果是 CentOS 6 包名为 syslinux-nonlinux`）

如果你无法通过网络下载这些文件可以查询 syslinux(CentOS 6 为 syslinux-nonlinux)包列出这些文件的路径，然后将pxelinux.0 和 menu.c32 拷贝到 /var/lib/cobbler/loaders 目录

```sh
[root@Cobbler loaders]# rpm -ql syslinux-nonlinux | grep 'pxelinux.0\|menu.c32'
/usr/share/syslinux/gpxelinux.0
/usr/share/syslinux/menu.c32
/usr/share/syslinux/pxelinux.0
/usr/share/syslinux/vesamenu.c32
```

通过网络下载启动文件(下载的时候可能会出现失败的情况，多试几次也许会成功)

```sh
[root@Cobbler ~]# cobbler get-loaders
task started: 2017-09-24_093837_get_loaders
task started (id=Download Bootloader Content, time=Sun Sep 24 09:38:37 2017)
path /var/lib/cobbler/loaders/README already exists, not overwriting existing content, use --force if you wish to update
downloading http://cobbler.github.io/loaders/COPYING.elilo to /var/lib/cobbler/loaders/COPYING.elilo
downloading http://cobbler.github.io/loaders/COPYING.yaboot to /var/lib/cobbler/loaders/COPYING.yaboot
downloading http://cobbler.github.io/loaders/COPYING.syslinux to /var/lib/cobbler/loaders/COPYING.syslinux
downloading http://cobbler.github.io/loaders/elilo-3.8-ia64.efi to /var/lib/cobbler/loaders/elilo-ia64.efi
downloading http://cobbler.github.io/loaders/yaboot-1.3.17 to /var/lib/cobbler/loaders/yaboot
downloading http://cobbler.github.io/loaders/pxelinux.0-3.86 to /var/lib/cobbler/loaders/pxelinux.0
downloading http://cobbler.github.io/loaders/menu.c32-3.86 to /var/lib/cobbler/loaders/menu.c32
downloading http://cobbler.github.io/loaders/grub-0.97-x86.efi to /var/lib/cobbler/loaders/grub-x86.efi
downloading http://cobbler.github.io/loaders/grub-0.97-x86_64.efi to /var/lib/cobbler/loaders/grub-x86_64.efi
*** TASK COMPLETE ***
```

**问题4：**修改 `xinetd` 配置文件将 `rsync` 的禁用选项设置为 no

```sh
[root@Cobbler ~]# vim /etc/xinetd.d/rsync 
# default: off
# description: The rsync server is a good addition to an ftp server, as it \
#       allows crc checksumming etc.
service rsync
{
        disable = no
        flags           = IPv6
        socket_type     = stream
        wait            = no
        user            = root
        server          = /usr/bin/rsync
        server_args     = --daemon
        log_on_failure  += USERID
}
```

再次同步并检测问题

```sh
[root@Cobbler ~]# cobbler sync
task started: 2017-09-24_101208_sync
task started (id=Sync, time=Sun Sep 24 10:12:08 2017)
running pre-sync triggers
cleaning trees
removing: /var/lib/tftpboot/pxelinux.cfg/default
removing: /var/lib/tftpboot/grub/grub-x86_64.efi
removing: /var/lib/tftpboot/grub/images
removing: /var/lib/tftpboot/grub/efidefault
removing: /var/lib/tftpboot/grub/grub-x86.efi
removing: /var/lib/tftpboot/s390x/profile_list
copying bootloaders
trying hardlink /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
copying: /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
trying hardlink /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
copying: /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
copying distros to tftpboot
copying images
generating PXE configuration files
generating PXE menu structure
rendering DHCP files
generating /etc/dhcp/dhcpd.conf
rendering TFTPD files
generating /etc/xinetd.d/tftp
cleaning link caches
rendering Rsync files
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running: dhcpd -t -q
received on stdout: 
received on stderr: 
running: service dhcpd restart
received on stdout: Shutting down dhcpd: [  OK  ]
Starting dhcpd: [  OK  ]

received on stderr: 
running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
[root@Cobbler ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : file /etc/xinetd.d/rsync does not exist
2 : debmirror package is not installed, it will be required to manage debian deployments and repositories
3 : ksvalidator was not found, install pykickstart
4 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.

```

* 问题1 这个文件是存在的但系统还是提示不存在，所以不用管它
* 问题2 如果你不使用 `debian` 系列的系统可以不用管
* 问题3 `pykickstart` 这是一个用来编写和阅读 `kickstart` 文件的程序，不需要的话可以不管
* 问题4 电源管理工具，用不着可以不管

挂载系统镜像并导入镜像

```sh
[root@Cobbler ~]# mount /dev/sr0 /mnt/
mount: block device /dev/sr0 is write-protected, mounting read-only
## import 表示导入镜像
## --name 为导入的系统设置名称
## --path 需要导入的镜像路径
## --arch 导入的系统架构
[root@Cobbler ~]# cobbler import --name=CentOS-6.9 --path=/mnt/ --arch=x86_64
task started: 2017-09-24_101802_import
task started (id=Media import, time=Sun Sep 24 10:18:02 2017)
Found a candidate signature: breed=redhat, version=rhel6
Found a matching signature: breed=redhat, version=rhel6
Adding distros from path /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64:
creating new distro: CentOS-6.9-x86_64
trying symlink: /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64 -> /var/www/cobbler/links/CentOS-6.9-x86_64
creating new profile: CentOS-6.9-x86_64
associating repos
checking for rsync repo(s)
checking for rhn repo(s)
checking for yum repo(s)
starting descent into /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64 for CentOS-6.9-x86_64
processing repo at : /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64
need to process repo/comps: /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64
looking for /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64/repodata/*comps*.xml
Keeping repodata as-is :/var/www/cobbler/ks_mirror/CentOS-6.9-x86_64/repodata
*** TASK COMPLETE ***
```

列出已导入的发行版

```sh
[root@Cobbler ~]# cobbler distro list
   CentOS-6.9-x86_64
```

查看发行版详细报告

```sh
[root@Cobbler ~]# cobbler distro report --name=CentOS-6.9-x86_64
Name                           : CentOS-6.9-x86_64
Architecture                   : x86_64
TFTP Boot Files                : {}
Breed                          : redhat
Comment                        : 
Fetchable Files                : {}
Initrd                         : /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64/images/pxeboot/initrd.img
Kernel                         : /var/www/cobbler/ks_mirror/CentOS-6.9-x86_64/images/pxeboot/vmlinuz
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart Metadata             : {'tree': 'http://@@http_server@@/cblr/links/CentOS-6.9-x86_64'}
Management Classes             : []
OS Version                     : rhel6
Owners                         : ['admin']
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Template Files                 : {}
```

列出已存在的配置文件

```sh
[root@Cobbler ~]# cobbler profile list
   CentOS-6.9-x86_64
```

查看配置文件详细信息

```sh
[root@Cobbler ~]# cobbler profile report --name=CentOS-6.9-x86_64
Name                           : CentOS-6.9-x86_64
TFTP Boot Files                : {}
Comment                        : 
DHCP Tag                       : default
Distribution                   : CentOS-6.9-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : ['cobbler_server']
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 : 
Internal proxy                 : 
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      : 
Virt RAM (MB)                  : 512
Virt Type                      : kvm
```

创建 `kickstart` 文件

```sh
[root@Cobbler ~]# vim /var/lib/cobbler/kickstarts/centos-6.9.ks
# Kickstart file automatically generated by anaconda.

#version=DEVEL
install
url --url=http://192.168.1.200/cobbler/ks_mirror/CentOS-6.9-x86_64/
lang en_US.UTF-8
keyboard us
network --onboot yes --device eth0 --mtu=1500 --bootproto dhcp --noipv6
rootpw  --iscrypted $6$8ls6kGd3I0YGF6Hf$JRsujR1hDeUC9MNYAe4CWM3uUiU9Z8YTayoRDbSKGMULX7yFJbiQlDQtlCUrDkUb2vlbTr07lvKHeH3/VEHaV0
# Reboot after installation
reboot
firewall --service=ssh
authconfig --enableshadow --passalgo=sha512
selinux --disabled
timezone Asia/Shanghai
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto rhgb rhgb quiet quiet"
# Clear the Master Boot Record
zerombr
# The following is the partition information you requested
# Note that any partitions you deleted are not expressed
# here so unless you clear all partitions first, this is
# not guaranteed to work
clearpart --all
part pv.008003 --grow --size=200
volgroup lvmdisk --pesize=4096 pv.008003
logvol /local --fstype=ext4 --name=local --vgname=lvmdisk --size=159240
logvol / --fstype=ext4 --name=root --vgname=lvmdisk --size=40960

part /boot --fstype=ext4 --size=500
part swap --size=4096


%packages
@Base
@Core
@core
@server-policy
@workstation-policy
lrzsz
tree
vim

%end
```

关联 `kickstart` 文件

```sh
cobbler profile edit --name=CentOS-6.9-x86_64 --kickstart=/var/lib/cobbler/kickstarts/centos-6.9.ks
```

客户端测试

![客户端读取菜单列表成功](http://ov2iiuul1.bkt.clouddn.com/cobbler_cli1.png)

![开始安装](http://ov2iiuul1.bkt.clouddn.com/cobbler_cli2.png)



