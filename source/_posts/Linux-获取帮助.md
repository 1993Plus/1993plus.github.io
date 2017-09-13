---
title: Linux 获取帮助
date: 2017-07-17 19:08:09
category: Linux
tags: 
    - Linux
    - man
    - info
    - help
    
---

## 在 Linux(CentOS) 系统中获取帮助

### Linux 系统中指令的分类：
- 内部命令（指集成在 shell 中的指令）
- 外部指令（非集成在 shell 中的指令）
` 查看指令是否是内部指令或外部指令使用 #type`
> [root@localhost ~]# type cd
cd is a shell builtin   //系统告诉我们 cd 是一个 shell 内置的。
[root@localhost ~]# type ls
ls is aliased to `ls --color=auto'  //系统告诉我们 ls 指向的是一个命令别名（alias）。
[root@localhost ~]# which ls //使用 which 指令查看 ls 的完整路径。
alias ls='ls --color=auto'
	/bin/ls

### 获取内部命令的帮助信息

使用格式：` # help command `

> [root@localhost ~]# help cd
cd: cd [-L|-P] [dir]
    Change the shell working directory.
    
>    Change the current directory to DIR.  The default DIR is the value of the
    HOME shell variable.
    
>    The variable CDPATH defines the search path for the directory containing
    DIR.  Alternative directory names in CDPATH are separated by a colon (:).
    A null directory name is the same as the current directory.  If DIR begins
    with a slash (/), then CDPATH is not used.
    
>    If the directory is not found, and the shell option `cdable_vars' is set,
    the word is assumed to be  a variable name.  If that variable has a value,
    its value is used for DIR.
    
>    Options:
        -L	force symbolic links to be followed
        -P	use the physical directory structure without following symbolic
    	links
    
>    The default is to follow symbolic links, as if `-L' were specified.
    
>    Exit Status:
>    Returns 0 if the directory is changed; non-zero otherwise.
        

### 获取外部命令的帮助信息

- --help
使用格式： ` # command --help`
> [root@localhost ~]# date --help
 Usage: date [OPTION]... [+FORMAT]
  or:  date [-u|--utc|--universal] 
 [MMDDhhmm[[CC]YY][.ss]]
 Display the current time in the given FORMAT, or set the  system date.
...略...


- man

man 命令来自英文单词 manual 的简写。

man 常用的两个选项
` # man [-f] [-k] keyword`
-f：在系统中搜索与 keyword 完全匹配的文档。
-k：在系统中搜索所有与 keyword 相关的文档。
使用 `whatis` 效果相当于 man -f
在搜索出来的相关文档中关键字后面都会带有一个数字，这些数字都是有含义的。


> [root@localhost ~]# man -f passwd
passwd               (1)  - update user's authentication tokens
passwd               (5)  - password file
passwd [sslpasswd]   (1ssl)  - compute password hashes


#### man 文档代号说明

代号| 说明|
:--:|--|
1|  可执行程序或 shell 命令。                       |
2|  系统调用（由内核提供的函数）                    |
3|  库调用（程序库中的函数）                        |
4|  特殊文件（通常在 /dev 中）                      |
5|  文件格式和约定，如 /etc/passwd                  |
6|  游戏。                                          |
7|  杂项（包括宏包和约定）                          |
8|  系统管理命令（通常只用于 root ）                |
9|  内核相关的文件。                                |

如上例，如果我们是要查看与密码文件格式相关的文档那么我们应该用代号 5。

> [root@localhost ~]# man 5 passwd  


#### man 文档中各段落说明

段落名| 说明|
:--|--|
NAME       |  简短的指令、及说明。                    |
SYNOPSIS   |  简短的指令下达语法简介。                |
DESCRIPTION|  较为完整的说明。                        |
OPTIONS    |  相关举例说明                            |
COMMANDS   |  当这个指令在执行的时候可以在此程序中下达的指令。                                                  |
FILES      |  相关链接或文件。                        |
SEE ALSO   |  相关的其他说明。                        |
EXAMPLE    |  一些可以参考的范例                      |