---
title: systemd 基础应用
date: 2017-09-15 17:37:40
category: Linux
tags:
	- systemd
---



**systemd**

从 CentOS7.x 以后，RedHat 系列的 distrbution 放弃了沿用多年的 System V 开机启动服务的流程，改用 systemd 服务管理机制。

**systemd 的特性**

* 平行处理所有服务，加速开机流程；旧的 init 启动脚本是一项一项任务依次启动模式，因此不相依的服务也是要一个一个的等待。而 systemd 可以让所有服务同时启动，加快系统启动速度。
* 按需启动模式；仅有 systemd 服务搭配 systemctl 命令来处理，无须其他额外的命令来支持，所有服务都采用非独立模式，由 systemd 来负责监听服务端口，当监听到服务端口有请求时 systemd 才会将任务交给 daemon。
* 服务相依性检查；systemd 可以自动对服务进行相依性检查，如果 A 服务是依赖于 B 服务的，如果在没有启动 B 服务的情况下启动 A 服务，systemd 会自动启动 B 服务。
* 根据 daemon 功能分类；systemd 定义了所有的服务为一个服务单位（unit），并将该 unit 归类到不同的服务类型（type）中。systemd 将 unit 区分为 service、socket、target、path、snapshot、timer 等多种不同的类型（type）。
* 将多个 daemons 集合成为一个群组；systemd 将许多的功能集合成为一个 target 项目，这个项目主要负责操作环境的创建，执行某个 target 就是执行了很多 daemons。
* 向下兼容 init 服务脚本；systemd 是可以兼容 init 的启动脚本的，所以就得 init 启动脚本也能通过 systemd 来管理，只是一些高级的 systemd 功能不支持。
* 在 runlevel 的对应上；大概仅有 runlevel 1、3、5 有对应的 systemd target 类型。
* 全部的 systemd 都用 systemctl 来管理；而 systemctl 所支持的语法有限制，systemctl 不可以自定参数。
* 如果某个服务是手动执行启动的，而不是使用 systemctl 启动的，那么 systemd 将无法侦测到该服务，所以不能使用 systemctl 对其进行管理。
* systemd 启动过程中无法与管理员通过 standard input（标准输入）进行互动。



**systemd 的配置文件存放目录**

systemd 将过去的 daemon 执行脚本通通称为一个服务单位（unit），而每种服务单位依据功能来区分时就分为不同的类型（type）。基本的类型有系统服务、监听与交换数据的 socket 服务、存储系统状态的快照类型、提供不同类型的操作环境（target）等，这些配置文件都存放在以下目录中；

* /usr/lib/systemd/system/：每个服务最主要的启动脚本设置，类似以前的 /etc/init.d 下面的文件
* /run/systemd/system/：系统执行过程中所产生的服务脚本，这些脚本的优先等级比 /usr/lib/systemd/system/ 高
* /etc/systemd/system/：管理员根据需求所创建的执行脚本，优先等级最高，其实在这个目录中存放的都是些链接文件，而实际都是存放在 /usr/lib/systemd/system/ 下面的。



**systemd 的 unit 类型分类**

在 /usr/lib/systemd/system/ 中有很多执行脚本，区分不同的类型主要看扩展名。

```sh
[root@localhost ~]# ll /usr/lib/systemd/system/ | grep '\(crond\|multi\)'
-rw-r--r--. 1 root root  284 Mar 31  2016 crond.service
-rw-r--r--. 1 root root  492 May 26 05:22 multi-user.target
drwxr-xr-x. 2 root root  279 Aug 15 17:40 multi-user.target.wants
lrwxrwxrwx. 1 root root   17 Aug 15 17:40 runlevel2.target -> multi-user.target
lrwxrwxrwx. 1 root root   17 Aug 15 17:40 runlevel3.target -> multi-user.target
lrwxrwxrwx. 1 root root   17 Aug 15 17:40 runlevel4.target -> multi-user.target
```

* .service；一般服务类型，系统服务与服务器本身所需要的本机服务以及网络服务等
* .socket；内部程序数据交换的 socket 服务，当有请求通过 socket 传递信息来需要连接服务时，就根据当时的状态将该用户的请求传送到对应的 daemon，若 daemon 未启动，则启动该 daemon 后再传送用户的请求
* .target；执行环节类型，其实就是一堆 unit 的集合，当执行一个 target 就是执行一堆其他 .service 或者 .socket 之类的服务
* .mount/.automount；文件系统挂载相关的服务（mount unit/automount unit）
* .path；侦测特定文件或目录类型（path unit），某些服务需要侦测某些特定的目录来听过伫列服务，比如打印服务
* .timer；循环执行的服务（timer unit）

> 其中 .service 类型的最常见，因为一堆网络服务都是通过这种类型来设计的



**使用 systemctl 管理单一服务（service unit）**

命令使用格式

```sh
systemctl [OPTIONS...] {UNIT} ...
```

命令常用选项

```sh
start			## 启动目标 unit
stop			## 停止目标 unit
restart			## 重启目标 unit(先执行 stop 再执行 start)
reload			## 在不停止目标 unit 的情况下重新载入配置文件让设置生效
enable			## 设置目标 unit 在系统开机时自动启动
disable			## 设置目标 unit 在系统开机时不自动启动
status			## 查看目标 unit 当前状态
is-active		## 查看目标 unit 是否在运行
is-enabled		## 查看目标 unit 开机是否启动
static			## 设置目标 unit 禁止自启动，但可以被其他依赖的服务启动
mask			## 设置目标 unit 禁止启动，此时目标 unit 的启动脚本会被链接到 /dev/null
umask			## 取消目标 unit 的禁止启动
```



示例

```sh
## 使用 status 查看 atd 服务状态
[root@localhost ~]# systemctl status atd.service
● atd.service - Job spooling tools
   Loaded: loaded (/usr/lib/systemd/system/atd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2017-09-10 17:38:41 CST; 1min 44s ago
 Main PID: 5781 (atd)
   CGroup: /system.slice/atd.service
           └─5781 /usr/sbin/atd -f

Sep 10 17:38:41 localhost systemd[1]: Started Job spooling tools.
Sep 10 17:38:41 localhost systemd[1]: Starting Job spooling tools...
## 使用 stop 停止 atd 服务
[root@localhost ~]# systemctl stop atd.service
## 使用 start 启动 atd 服务
[root@localhost ~]# systemctl start atd.service
## 使用 restart 重启 atd 服务
[root@localhost ~]# systemctl restart atd.service
## 使用 disable 禁止 atd 服务开机自动启动
[root@localhost ~]# systemctl disable atd.service
Removed symlink /etc/systemd/system/multi-user.target.wants/atd.service.
## 使用 enable 将 atd 服务设置为开机自动启动
[root@localhost ~]# systemctl enable atd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/atd.service to /usr/lib/systemd/system/atd.service.
## 使用 is-active 查看 atd 服务是否在运行中
[root@localhost ~]# systemctl is-active atd.service
active
## 使用 is-enabled 查看 atd 服务是否为开机自启动
[root@localhost ~]# systemctl is-enabled atd.service
enabled
## 使用 mask 将 atd 服务设置为禁止启动 
[root@localhost ~]# systemctl mask atd.service
Created symlink from /etc/systemd/system/atd.service to /dev/null.
## 再次重启 atd 服务
[root@localhost ~]# systemctl restart atd.service
Failed to restart atd.service: Unit is masked.
## 取消 atd 服务的 mask 设置
[root@localhost ~]# systemctl unmask atd.service
Removed symlink /etc/systemd/system/atd.service.
```



**使用 systemctl 观察系统中的所有服务**

常用选项

```sh
list-units		## 列出当前已启动的 unit 使用 --all 查看包括未启动的 unit
list-unit-files	## 根据 /usr/lib/systemd/system 中的文件列出 unit 列表
--type=TYPE		## 查看指定类型的 unit，如 service、socket、target 等

## 其实直接使用 systemctl 就是查看所有已启动的 unit
```



**使用 systemctl 管理 target unit**

查看当前已有的 target unit

```sh
[root@localhost ~]# systemctl list-units --type=target --all
  UNIT                   LOAD      ACTIVE   SUB    DESCRIPTION
  basic.target           loaded    active   active Basic System
  bluetooth.target       loaded    inactive dead   Bluetooth
  cryptsetup.target      loaded    active   active Encrypted Volumes
  emergency.target       loaded    inactive dead   Emergency Mode
  final.target           loaded    inactive dead   Final Step
  getty.target           loaded    active   active Login Prompts
  graphical.target       loaded    inactive dead   Graphical Interface
  local-fs-pre.target    loaded    active   active Local File Systems (Pre)
  local-fs.target        loaded    active   active Local File Systems
  multi-user.target      loaded    active   active Multi-User System
  network-online.target  loaded    active   active Network is Online
  network-pre.target     loaded    inactive dead   Network (Pre)
  network.target         loaded    active   active Network
  nfs-client.target      loaded    active   active NFS client services
  nss-lookup.target      loaded    inactive dead   Host and Network Name Lookups
  nss-user-lookup.target loaded    inactive dead   User and Group Name Lookups
  paths.target           loaded    active   active Paths
  remote-fs-pre.target   loaded    active   active Remote File Systems (Pre)
  remote-fs.target       loaded    active   active Remote File Systems
  rescue.target          loaded    inactive dead   Rescue Mode
  rpcbind.target         loaded    inactive dead   RPC Port Mapper
  shutdown.target        loaded    inactive dead   Shutdown
  slices.target          loaded    active   active Slices
  sockets.target         loaded    active   active Sockets
  swap.target            loaded    active   active Swap
  sysinit.target         loaded    active   active System Initialization
● syslog.target          not-found inactive dead   syslog.target
  time-sync.target       loaded    inactive dead   System Time Synchronized
  timers.target          loaded    active   active Timers
  umount.target          loaded    inactive dead   Unmount All Filesystems

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

30 loaded units listed.
To show all installed unit files use 'systemctl list-unit-files'.
```

**与操作环境相关的 target**

* multi-user.target；纯文本模式
* graphical.target；文字加图形界面，其中包含了 multi.user.target
* rescue.target；救援模式，这是一个额外的系统，如果需要操作原有的系统需要使用 chroot 的方式来切换到原有系统
* emergency.target;紧急救援模式，需要使用 root 用户登录，在无法使用 rescue.target 时可以尝试该模式
* shutdown.target；关机模式
* gettty.target；设置可登录的 tty 数量
* ​

获取当前默认 target

```sh
[root@localhost ~]# systemctl get-default
multi-user.target
```

设置默认 target

```sh
[root@localhost ~]# systemctl set-default graphical.target 
Removed symlink /etc/systemd/system/default.target.
Created symlink from /etc/systemd/system/default.target to /usr/lib/systemd/system/graphical.target.
[root@localhost ~]# 
```

临时切换 target

```sh
[root@localhost ~]# systemctl isolate graphical.target 
## 如果当前系统中有安装图形服务那系统就会开启图形界面
```



使用 systemctl 分析各服务之间的依赖关系

```sh
## 使用 list-dependencies 查看所有 unit 的依赖关系
[root@localhost ~]# systemctl list-dependencies 
default.target
● ├─auditd.service
● ├─brandbot.path
● ├─chronyd.service
● ├─crond.service
● ├─dbus.service
● ├─irqbalance.service
● ├─mdmonitor.service
● ├─network.service
● ├─NetworkManager.service
● ├─plymouth-quit-wait.service
● ├─plymouth-quit.service
● ├─postfix.service
● ├─rsyslog.service
● ├─sshd.service
● ├─systemd-ask-password-wall.path
● ├─systemd-logind.service
● ├─systemd-readahead-collect.service
● ├─systemd-readahead-replay.service
● ├─systemd-update-utmp-runlevel.service
● ├─systemd-user-sessions.service
● ├─tuned.service
● ├─vmtoolsd.service
● ├─basic.target
...略...

## 使用 --reverse 反向追踪
[root@localhost ~]# systemctl list-dependencies sshd.service --reverse
sshd.service
● └─multi-user.target
●   └─graphical.target
```

END!

