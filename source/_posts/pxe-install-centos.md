---
title: CentOS 7 搭建操作系统自动化安装服务
date: 2017-09-17 18:04:33
category: Linux
tags:
	- 自动化安装操作系统
---
**环境**

* 服务端使用 CentOS 7.3
* 客户端提供镜像版本 CentOS 7.3、CentOS 6.9
* SELinux 关闭
* iptables 关闭
* 服务端 IP 192.168.1.250 ，客户端 IP 范围 192.168.1.10-192.168.1.100
* dhcp 提供客户端 ip 地址分配服务
* tftp 机器启动资源加载服务
* httpd 提供 yum 源



以上服务全部为一台计算机提供

![拓扑图](http://ot6jjpn4h.bkt.clouddn.com/net_install_system.png)

**安装必要软件包**

```sh
yum install -y dhcp
yum install -y tftp-server
yum install -y httpd
yum install -y syslinux
```



**配置 tftp server**

启动 tftp 服务

```sh

[root@localhost ~]# systemctl start tftp.service
## 检查 tftp 端口 69 是否活动
[root@localhost ~]# ss -unl | grep '69'
UNCONN     0      0           :::69                      :::*                  
```

拷贝必要文件到 tftp 工作目录( `/var/lib/tftpboot/` )

从 CentOS 7.3 光盘镜像中将 `initrd.img、vmlinuz、isolinux.cfg` 拷贝到 `/var/lib/tftpboot/` 中

```sh
[root@localhost ~]# cp /mnt/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot/ 
[root@localhost ~]# cp /mnt/isolinux/isolinux.cfg /var/lib/tftpboot/
```

现在还需要一个菜单背景文件( `mune.c32` )和启动文件 ( `pxelinux.0` )

这两个文件在刚才安装的 `syslinux`  包中

列出 `syslinux` 的文件查看这两个文件的路径并复制到 `/var/lib/tftpboot/` 中

```sh
[root@localhost ~]# rpm -ql syslinux | grep 'menu.c32\|pxelinux.0'
/usr/share/syslinux/gpxelinux.0
/usr/share/syslinux/menu.c32
/usr/share/syslinux/pxelinux.0
/usr/share/syslinux/vesamenu.c32
[root@localhost ~]# cp /usr/share/syslinux/{menu.c32,pxelinux.0} /var/lib/tftpboot/
```

切换至 `/var/lib/tftpboot/` 列出当前文件列表

```sh
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# ll
total 47716
-rw-r--r-- 1 root root 43372552 Sep 17 17:49 initrd.img
-rw-r--r-- 1 root root     3068 Sep 17 17:54 isolinux.cfg
-rw-r--r-- 1 root root    55012 Sep 17 18:02 menu.c32
-rw-r--r-- 1 root root    26764 Sep 17 18:02 pxelinux.0
-rwxr-xr-x 1 root root  5392080 Sep 17 17:49 vmlinuz
```

创建文件夹 `pxelinux.cfg` 

将 `isolinux.cfg` 移动到 `pexlinux.cfg` 文件夹中并改名为 `default`

```sh
[root@localhost tftpboot]# mkdir pxelinux.cfg
[root@localhost tftpboot]# mv isolinux.cfg pxelinux.cfg/default
```

当前目录结构

```sh
[root@localhost tftpboot]# tree
.
├── initrd.img
├── menu.c32
├── pxelinux.0
├── pxelinux.cfg
│   └── default
└── vmlinuz

1 directory, 5 files
[root@localhost tftpboot]# 
```



**配置 httpd server **

启动 httpd 服务

```sh
[root@localhost ~]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:httpd(8)
           man:apachectl(8)
[root@localhost ~]# systemctl start httpd
[root@localhost ~]# ss -tan | grep '80'
LISTEN     0      128         :::80                      :::*                  
[root@localhost ~]#
```

切换到 httpd 工作目录 `/var/www/html`

 创建 `yum` 文件存放目录与 `kickstart` 文件存放目录

```sh
[root@localhost ~]# cd /var/www/html/
[root@localhost html]# mkdir -p {centos/7,ksfile}
[root@localhost html]# tree
.
├── centos
│   └── 7
└── ksfile

3 directories, 0 files
```

配置`kickstart` 文件

可以把系统安装时自动生成 的`kickstart` 文件拷贝一份改一下就可以了，一般在 `root` 家目录一个名为`anaconda-ks.cfg` 的文件，还可以安装 `system-config-kickstart` 这个程序包来制作，不过需要图形界面的支持，而且不能添加 `lvm` 分区

```sh
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
# old format: keyboard us
# new format:
keyboard --vckeymap=us --xlayouts='us'
# Root password
rootpw --iscrypted $1$iUbShR9k$BCT3THf737s5Zs5MEREgg.
# Use network installation
url --url=http://192.168.1.250/centos/7
# System language
lang en_US --addsupport=zh_CN.UTF-8
user --name=weidong --password=$6$zQqnWSZohBYO8Zuf$1EYd/nebaz4r0PlSwXxoY8TWM5idE9uU4N3ANGLVcRp/n72OwOIUHm7hZ93zOrw3KS4dYR1N9BD6pF7a0.tAR/ --iscrypted --gecos="weidong"
# Firewall configuration
firewall --disabled
repo --name="Server-HighAvailability" --baseurl=file:///run/install/repo/addons/HighAvailability
repo --name="Server-ResilientStorage" --baseurl=file:///run/install/repo/addons/ResilientStorage
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use text mode install
text
firstboot --disable
# SELinux configuration
selinux --disabled

# System services
services --enabled="chronyd"
ignoredisk --only-use=sda
# Network information
network  --bootproto=dhcp --device=ens33
network  --bootproto=dhcp --device=None
# Reboot after installation
reboot
# System timezone
timezone Asia/Shanghai
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel --drives=sda
# Disk partitioning information
part pv.338 --fstype="lvmpv" --ondisk=sda --size=101899
part /boot --fstype="xfs" --ondisk=sda --size=500
volgroup lvmdisk --pesize=4096 pv.338
logvol /  --fstype="xfs" --size=51200 --name=root --vgname=lvmdisk
logvol /local  --fstype="xfs" --size=46600 --name=local --vgname=lvmdisk
logvol swap  --fstype="swap" --size=4096 --name=swap --vgname=lvmdisk

%packages
@^minimal
@core
vim
lrzsz
tree
-NetworkManager
-NetworkManager-team
-NetworkManager-tui
-NetworkManager-wifi
-aic94xx-firmware
-alsa-firmware
-biosdevname
-dracut-config-rescue
-ivtv-firmware
-iwl100-firmware
-iwl1000-firmware
-iwl105-firmware
-iwl135-firmware
-iwl2000-firmware
-iwl2030-firmware
-iwl3160-firmware
-iwl3945-firmware
-iwl4965-firmware
-iwl5000-firmware
-iwl5150-firmware
-iwl6000-firmware
-iwl6000g2a-firmware
-iwl6000g2b-firmware
-iwl6050-firmware
-iwl7260-firmware
-iwl7265-firmware
-kernel-tools
-libsysfs
-linux-firmware
-microcode_ctl
-postfix

%end
```

确认文件是否有读权限，如果没有读权限客户端在读取文件的时候会找不到 `kickstart` 文件而报错

```sh
[root@localhost ksfile]# ll
total 4
-rw-r--r-- 1 root root 2360 Sep 17 18:43 centos-7-ks.cfg
```

把`CentOS 7.3`光盘中的文件拷贝到 `/var/www/html/centos/7` 目录下

```sh
[root@localhost ksfile]# cp -r /mnt/* /var/www/html/centos/7/
```

修改启动菜单，把本地启动选项设置为默认选择项，避免出现误选择导致数据丢失 

在选项字符前加`^` 表示该字符设置为快捷键

```sh
[root@localhost ksfile]# vim /var/lib/tftpboot/pxelinux.cfg/default
[root@localhost ksfile]# cat /var/lib/tftpboot/pxelinux.cfg/default
default menu.c32
timeout 600

menu title CentOS Linux 7

label linux
  menu label ^Install CentOS Linux 7
  kernel vmlinuz
  append initrd=initrd.img ks=http://192.168.1.250/ksfile/centos-7-ks.cfg

label local
  menu label Boot from ^local drive
  menu default
  localboot 0xffff

menu end
```

**配置 dhcp server**

此时 `dhcp` 还不能启动，因为还没有配置文件

```sh
## 打开默认的配置文件会有提示,参考这个示例文件来修改配置文件
[root@localhost ~]# cat /etc/dhcp/dhcpd.conf 
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
```

修改好的配置文件

```sh
[root@localhost ~]# cat /etc/dhcp/dhcpd.conf 
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#

# option definitions common to all supported networks...
option domain-name "test.local";
option domain-name-servers 192.168.1.1;

default-lease-time 600;
max-lease-time 7200;
 
subnet 192.168.1.0 netmask 255.255.255.0 {
        range 192.168.1.10 192.168.1.100;
        next-server 192.168.1.250;
        filename "pxelinux.0";
}
```

启动 dhcp 服务

```sh
[root@localhost ~]# systemctl start dhcpd
[root@localhost ~]# systemctl is-active dhcpd
active
```



**启动客户端测试**

客户端开机选择网络启动（需要网卡支持 PXE）

![客户机启动设置](http://ot6jjpn4h.bkt.clouddn.com/cli1.png)



读取菜单文件成功

![客户端成功读取到菜单文件](http://ot6jjpn4h.bkt.clouddn.com/cli2.png)



######多安装源

现在已经能成功安装 `CentOS 7.3` 了，再添加` CentOS 6.9 `的安装源

添加 `CentOS 6.9` 的 `kickstart` 文件

```sh
[root@localhost ksfile]# cat centos-6-ks.cfg 
# Kickstart file automatically generated by anaconda.

#version=DEVEL
install
url --url=http://192.168.1.250/centos/6
text
lang en_US.UTF-8
keyboard us
network --onboot yes --device eth0 --bootproto dhcp --noipv6
rootpw  --iscrypted $6$8ls6kGd3I0YGF6Hf$JRsujR1hDeUC9MNYAe4CWM3uUiU9Z8YTayoRDbSKGMULX7yFJbiQlDQtlCUrDkUb2vlbTr07lvKHeH3/VEHaV0
firewall --service=ssh
authconfig --enableshadow --passalgo=sha512
selinux --disabled
timezone Asia/Shanghai
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto rhgb quiet"
# The following is the partition information you requested
# Note that any partitions you deleted are not expressed
# here so unless you clear all partitions first, this is
# not guaranteed to work
zerombr
clearpart --all
part pv.008003 --grow --size=200
volgroup lvmdisk --pesize=4096 pv.008003
logvol /local --fstype=ext4 --name=local --vgname=lvmdisk --size=159240
logvol / --fstype=ext4 --name=root --vgname=lvmdisk --size=40960

part /boot --fstype=ext4 --size=500
part swap --size=4096


reboot
%packages
@core
@server-policy
@workstation-policy
vim
tree
lrzsz
%end
```

添加 `CentOS 6.9` 源

```sh
[root@localhost ksfile]# cd ../centos/
[root@localhost centos]# mkdir 6
[root@localhost centos]# cp /mnt/* ./6/
```

在 `/var/lib/tftpboot` 目录中分别创建`Centos 6`与`CentOS7 `  的内核文件目录

```sh
[root@localhost centos]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# mkdir centos{6,7}
```

将 `CentOS 7` 的 initrd.img 与 vmlinuz 文件移动到 `CentOS 7` 文件夹中

```sh
[root@localhost tftpboot]# cp initrd.img vmlinuz centos7/ 
```

拷贝`CentOS 6.9` 安装光盘中的 `initrd.img、vmlinuz`两个文件到 `CentOS 6`目录中

```sh
[root@localhost tftpboot]# cp /mnt/images/pxeboot/{initrd.img,vimlinuz} centos6/
```

此时的目录结构

```sh
[root@localhost tftpboot]# tree
.
├── centos6
│   ├── initrd.img
│   └── vmlinuz
├── centos7
│   ├── initrd.img
│   └── vmlinuz
├── menu.c32
├── pxelinux.0
└── pxelinux.cfg
    └── default

3 directories, 7 files
```

修改菜单选项

```sh
[root@localhost tftpboot]# cat pxelinux.cfg/default
default menu.c32
timeout 600

menu title CentOS Linux 7

label centos6
  menu label Install CentOS Linux ^6
  kernel centos6/vmlinuz
  append initrd=centos6/initrd.img ks=http://192.168.1.250/ksfile/centos-6-ks.cfg

label centos7
  menu label Install CentOS Linux ^7
  kernel centos7/vmlinuz
  append initrd=centos7/initrd.img ks=http://192.168.1.250/ksfile/centos-7-ks.cfg

label local
  menu label Boot from ^local drive
  menu default
  localboot 0xffff

menu end
```

客户端测试

读取菜单项成功

![读取菜单选项成功](http://ov2iiuul1.bkt.clouddn.com/cli3.png)



![读取内核文件成功](http://ov2iiuul1.bkt.clouddn.com/cli4.png)

**FAQ:**

* 如果是从 `root` 目录拷贝的 `kickstart` 文件一定要将权限加上读(r)的权限
* 客户机内存必须大于 1G 以上，否则会出现挂载出错导致无法安装



END!