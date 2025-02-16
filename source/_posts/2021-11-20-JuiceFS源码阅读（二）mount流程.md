---
title: JuiceFS源码阅读（二）mount流程
date: 2021-11-20 23:30:30
categories:
- 源码阅读
tags: [文件存储, JuiceFS, mount]
---

![](https://s4.ax1x.com/2021/12/20/TuUXMn.png)

<!-- more -->

注：本系列源码基于juicefs 0.17.2版本

前一篇文章介绍了[juicefs的format流程](https://www.a-programmer.top/2021/11/09/JuiceFS源码阅读（一）format流程/)，本文将简单介绍juicefs的mount流程。
使用juicefs的第二步需要mount，用如下类似的命令：

```
 sudo juicefs mount -d redis://127.0.0.1:6379/1 /mnt/jfs
```

然后就可以愉快地把juicefs当本地文件系统来使用了。
juicefs的fuse客户端是基于go-fuse开发的（https://github.com/hanwen/go-fuse/ ），文档可参考 https://pkg.go.dev/github.com/hanwen/go-fuse ，支持3种文件系统模式，juicefs用到的是RawFileSystem，全部操作需要自己实现，另外两种模式文档里有介绍。
此外，关于FUSE的介绍可以参考[FUSE 框架](https://zhuanlan.zhihu.com/p/378429806)。

## mount 流程

整个mount流程简单地说，就是执行juicefs mount命令之后，会检查输入的参数，接着初始化元数据连接、数据连接、初始化各种配置及vfs，然后创建FUSE文件系统，启动fuse server进程，接受用户文件操作请求。

mount流程图如下：
![](https://s4.ax1x.com/2021/12/20/TuUkHf.png)
main()->mountFlags()->mount_flags()->clientFlags()->mount()....，mount()中先设置日志级别，然后检查参数是否正确以及挂载点是否正常，然后新建元数据服务客户端，从元数据服务上获取元数据信息，注册普罗米修斯，初始化各种挂载点配置（如cache目录、写入策略write through或write back、预读取策略、内存buffer大小等），获取使用的对象存储，对对象存储设置上传和下载限速，新建cacheStore，注册特定消息类型的回调函数（比如DeleteChunk注册对应的store.Remove）， VFS初始化，检查挂载点，新建NewSession，installHandler中做好信号处理（比如kill信号，ctrl+c之类的处理），暴露数据采集接口，在fuse.Serve中新建文件系统fuse.RawFileSystem，在指定的挂载点创建FUSE文件系统，之后启动fuse server进程，随后开始接受用户文件操作请求。

其中NewSession()会调用元数据模块具体的实现（redis/sql等），大致做了如下事情：
（1）启动一个goroutine专门刷新空间及inode使用情况（每10秒一次）
（2）将本session id和session info(juicefs version, hostname, process id)记录到元数据服务(redis或sql等，具体看使用的什么来存元数据)中
（3）启动一个goroutine专门刷新session并会清理过期的session（每分钟一次）
（4）启动一个goroutine专门清理删除掉的文件（每分钟一次）
（5）启动一个goroutine专门清理slices（每小时一次，至于什么是slices，这个涉及到juicefs是如何存储文件的，本文不多做介绍，可以看文末参考资料中juicefs官方的《juicefs是如何存储文件的》），真正删除对象存储上对象时会调用前面注册的回调函数。
（6）还有一个flushStats goroutine，看起来是如果有新的空间和新的inodes则更新相应的信息（每秒一次）

本文主要介绍mount流程的主要逻辑，还有很多细节没有提，感兴趣的小伙伴可以自己去查阅源码。

## 参考资料

[JuiceFS源码](https://github.com/juicedata/juicefs)

[JuiceFS如何存储文件](https://juicefs.com/docs/zh/community/architecture#%E5%A6%82%E4%BD%95%E5%AD%98%E5%82%A8%E6%96%87%E4%BB%B6)

[FUSE 框架](https://zhuanlan.zhihu.com/p/378429806)

[JuiceFS框架介绍和读写流程解析](https://www.cnblogs.com/luohaixian/p/15374849.html)
[juicefs使用及关键流程源码分析](http://aspirer.wang/?p=1553)