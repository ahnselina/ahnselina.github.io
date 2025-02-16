---
title: JuiceFS的编译安装及Mac上使用JuiceFS
date: 2021-11-06 23:30:30
categories:
- 存储
tags: [文件存储, JuiceFS]
---

![](https://z3.ax1x.com/2021/11/30/oQzCRg.png)

<!-- more -->



本文主要记录JuiceFS在debian系统上的编译安装和升级，也记录了在Mac上的安装遇到的问题和解决办法。

## 安装git工具

```
apt-get install git -y
```

JuiceFS 客户端使用 Go 语言开发，因此在编译之前，你提前在本地安装好依赖的工具：
Go 1.15+
GCC 5.4+
提示：对于中国地区用户，为了加快获取 Go 模块的速度，建议通过 GOPROXY 环境变量设置国内的镜像服务器。例如：Goproxy China。

## 下载安装GO

```
wget https://golang.org/dl/go1.17.2.linux-amd64.tar.gz
```

使用sha256sum工具来验证下载的文件是否正确

```
sha256sum go1.17.2.linux-amd64.tar.gz
```

确保哈希值与[下载页面](https://golang.org/dl/)上的哈希值相同。

设置环境变量：

```
vim ~/.profile
```

添加

```
export PATH=$PATH:/usr/local/go/bin
```

保存文件，并将新的PATH环境变量应用于当前的shell会话：

```
source ~/.profile
```

验证是否安装成功，安装成功显示如下：

```
~# go version
go version go1.17.2 linux/amd64
```

## 安装gcc

机器上已经有gcc了，此步骤略过，可以看下网上的安装步骤，很简单。

## 执行编译

进入源代码目录:

```
$ cd juicefs
```

开始编译：

```
$ make
```

编译成功以后，可以在当前目录中找到编译好的 juicefs 二进制程序。

## 安装JuiceFS

编译好juicefs的二进制程序后，使用如下命令安装juicefs：

```
install juicefs /usr/local/bin
```



## 升级juicefs

升级juicefs很简单，直接使用新版本juicefs二进制替换旧版本即可。

## Mac上安装juicefs

Mac上安装JuiceFS可以直接参考文章:[macOS 系统使用 JuiceFS](https://juicefs.com/docs/zh/community/juicefs_on_macos) ，不再赘述。

## 遇到的问题

### 1.直接用linux设备上的juicefs二进制文件执行报错

**解决办法**：参考文章:[macOS 系统使用 JuiceFS](https://juicefs.com/docs/zh/community/juicefs_on_macos)进行操作

### 2.juicefs create /mnt/jfs: mkdir /mnt: read-only file system

/mnt是只读的。

 **解决办法：** 换一个可读写的挂载目录。

### 3.fail to mount after 10 seconds, please mount in foreground

**解决办法**：首先需要安装macFUSE，然后重新mount。

```
sudo juicefs mount redis://:password@x.x.x.x:6379/1 /tmp/jfs --debug
```

会弹出提示窗口显示:

> “Please open the Security & Privacy System Preferences pane, go to the General preferences and allow loading system software from developer "Benjamin Fleischer". A restart might be required before the system extension can be loaded.”

所以我们需要打开“系统偏好设置”->“安全与隐私”->“通用”->“allow loading system software from developer "Benjamin Fleischer".” ->“重启电脑”。

## 参考资料

[JuiceFS 编译安装和升级](https://github.com/juicedata/juicefs/blob/main/docs/zh_cn/client_compile_and_upgrade.md)

[macOS 系统使用 JuiceFS](https://juicefs.com/docs/zh/community/juicefs_on_macos)

[Apple M1 芯片如何使用 JuiceFS](https://www.modb.pro/db/156742)