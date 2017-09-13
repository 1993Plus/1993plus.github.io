---
title: MACOS 中删除 nodejs
date: 2017-09-13 16:30:18
category: MACOS
tags:
	- nodejs
	- uninstall nodejs
	- macos
---



在 node 官网上下载的安装包，用安装包安装的 node 应该可以用一下命令行卸载：

在终端输入以下命令

```sh
sudo rm -rf /usr/local/{bin/{node,npm},lib/node_modules/npm,lib/node,share/man//node.}
```

* 删除/usr/local/lib中的所有node和node_modules
* 删除/usr/local/lib中的所有node和node_modules的文件夹
* 如果是从brew安装的, 运行brew uninstall node
* 检查~/中所有的local, lib或者include文件夹, 删除里面所有node和node_modules
* 在/usr/local/bin中, 删除所有node的可执行文件
* 最后运行以下代码:(可能具体安装路径会有区别 ,find ~ -name "node"   可以找到所有



**删除残余文件**


```sh
sudo rm /usr/local/bin/npm
sudo rm /usr/local/share/man/man1/node.1
sudo rm /usr/local/lib/dtrace/node.d
sudo rm -rf ~/.npm
sudo rm -rf ~/.node-gyp
sudo rm /opt/local/bin/node
sudo rm /opt/local/include/node
sudo rm -rf /opt/local/lib/node_modules
```