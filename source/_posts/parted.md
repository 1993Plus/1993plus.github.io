---
title: 使用 parted 创建磁盘分区
date: 2017-09-15 17:48:49
category: Linux
tags:
	- parted
---



linux 分区工具 parted

`man  page `中 `parted `的介绍

\##  `parted` 一个分区操作工具

```sh
parted - a partition manipulation program
```

\## parted 是一个用来操作磁盘分区的程序，它支持多种分区表格式，包括 msdos 和 GPT 。

**使用格式**

```sh
parted [options] [device [command [options...]...]]
```

\## 如果后面没有跟参数将以交互模式运行.

**命令选项**

>```sh
>   -h, --help
>          displays a help message
>          ## 显示帮助信息.
>   -l, --list
>          lists partition layout on all block devices
>          ## 列出所有块设备的分区表.
>   -s, --script
>          never prompts for user intervention
>          ## 不显示用户提示.
>   -v, --version
>          displays the version
>          ## 显示版本.
>```



**操作命令**



> help [command]
>
> \## 打印普通帮助信息,或者指定命令的帮助信息.
>
> mklabel
>
> \## 创建分区表.
>
> mkpart
>
> \## 创建一个分区.
>
> print
>
> \## 显示分区表与分区信息.
>
> rm
>
> \## 删除分区.
>
> select 
>
> \## 选择设备.
>
> unit
>
> \## 设置单位.
>
> align-check
>
> \## 检测分区是否最优或者最小对齐.
>
> quit
>
> \## 退出.

**操作示例**

直接输入 `parted` 进入交互式模式

```sh
[root@localhost ~]# parted
GNU Parted 3.1			## 版本号.
Using /dev/sda			## 当前操作的磁盘名称(默认选择第一块磁盘)
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)
```

 如果需要操作的磁盘不是这块磁盘,那输入 `seclet` 来选择其它设备.

```sh
(parted) select /dev/sdb                                                  
Using /dev/sdb			## 以成功切换到了 sdb 磁盘上.
(parted) 
```

使用 `print` 来查看一下当前磁盘分区表信息.

```sh
(parted) print                                                            
Error: /dev/sdb: unrecognised disk label
Model: VMware, VMware Virtual S (scsi)                                    
Disk /dev/sdb: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
(parted)
```

\## 由于这块磁盘是第一次使用没有创建分区表,所以会提示“为发现分区表”

\## 那现在来为这块磁盘创建分区表

使用 `mklabel`命令创建 `GPT` 分区表

```sh
(parted) mklabel gpt
```

再打印一下磁盘信息看看

```sh
(parted) print                                                            
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  Flags

(parted)
```

\## 可以看到`Partition Table` 显示为 `gpt` 说明已经创建成功了

好了,现在可以对磁盘创建分区了,来创建一个 10GB 的分区

```sh
(parted) mkpart
Partition name?  []? part1 
File system type?  [ext2]? 		## 这里直接默认即可,后期再重新格式化.
Start? 1
End? 10240MB
```

打印一下分区信息看看

```sh
(parted) print                                                            
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name   Flags
 1      1049kB  10.2GB  10.2GB  xfs          part1

(parted) 
```

如果现在不想要这个分区了,使用 `rm` 命令删除

```sh
(parted) rm 1		## 输入 rm 后面跟上分区编号即可.
(parted) 
```

将磁盘分区表更改为 `msdos` 分区表

```sh
(parted) mklabel msdos                                                    
Warning: The existing disk label on /dev/sdb will be destroyed and all data on this disk will be lost. Do
you want to continue?
Yes/No? y
(parted)
```

\## 由于之前磁盘已经存在 `gpt` 分区表了,所以系统会发出警告.如果确认要这么做的话输入 `y` 后按回车键即可.

再看一下分区表信息

```sh
(parted) print                                                            
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start  End  Size  Type  File system  Flags

(parted)
```

\## 可以看到 `Partition Table` 已经显示为 `msdos` 了.

现在了划分一块 10GB 的主分区

```sh
(parted) mkpart                                                           
Partition type?  primary/extended? primary	## 在这里选择 primary 表示主分区.
File system type?  [ext2]?                                                
Start? 1                                                                  
End? 10240MB
```

使用 `unit` 命令设置按百分百划分分区

```sh
(parted) unit %  
```

删除分区使用百分百来划分分区

```sh
(parted) rm 1 
(parted) print free		## print free 查看未划分分区的空间.
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 100%
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start  End   Size  Type  File system  Flags
        0.00%  100%  100%        Free Space
```

把剩余的空间全部划分为 `extended` 分区

```sh
(parted) mkpart 
Partition type?  primary/extended? extended                               
Start? 0                                                                  
End? 100%  
```

继续划分逻辑分区

```sh
(parted) mkpart                                                           
Partition type?  primary/logical? logical                                 
File system type?  [ext2]?                                                
Start? 0                                                                  
End? 20%  
(parted) mkpart 
Partition type?  primary/logical? logical                                 
File system type?  [ext2]?                                                
Start? 20%                                                                
End? 40%                                                                  
(parted)
```

\## 当前这个磁盘的大小是 100GB 划分 20% 也就是 20GB,继续划分第一块分区,那就是从 20% 开始划分到整块磁盘的 40%.至此,这个磁盘被划分了两个 20GB 的逻辑空间.

使用 `quit` 退出操作.

使用 `lsblk` 查看块设备情况.

```sh
[root@localhost ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  100G  0 disk 
├─sda1   8:1    0    1G  0 part /boot
├─sda2   8:2    0    4G  0 part [SWAP]
└─sda3   8:3    0   50G  0 part /
sdb      8:16   0  100G  0 disk 
├─sdb1   8:17   0  512B  0 part 
├─sdb5   8:21   0   20G  0 part 
└─sdb6   8:22   0   20G  0 part 
[root@localhost ~]# 
```

\## 可以看到分区以被成功创建.


END!