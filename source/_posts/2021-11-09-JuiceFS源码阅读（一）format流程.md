---
title: JuiceFS源码阅读（一）format流程
date: 2021-11-09 23:30:30
categories:
- 源码阅读
tags: [文件存储, JuiceFS,format, ]
---

![](https://s4.ax1x.com/2021/12/08/ogfra8.png)

<!-- more -->

通过上一篇文章[JuiceFS性能怎么样](https://www.a-programmer.top/2021/11/07/JuiceFS性能怎么样/)，对JuiceFS的性能有了初步的了解，本文开始将会用一系列的文章来介绍JuiceFS的源码。本文是JuiceFS源码阅读的开篇，主要带大家了解一下JuiceFS的目录结构，知道各个文件大概功能是做什么的，然后介绍JuiceFS的format流程。

## JuiceFS的目录结构

![](https://s4.ax1x.com/2021/12/08/ogfoIU.png)
先看一眼目录结构:

* cmd 目录是总入口，如 juicefs format 命令的入口是 cmd/format.go。
* pkg 目录是具体实现，核心逻辑都在这里，其中：
* pkg/fuse/fuse.go 是 FUSE 实现的入口，基本是一个 wrapper，具体逻辑在下面。
* pkg/vfs 是具体的 FUSE 接口实现，元数据请求会调用 pkg/meta 中的实现，读请求会调用 pkg/vfs/reader.go，写请求会调用 pkg/vfs/writer.go。
* pkg/meta/redis.go 是 Redis 版本的元数据引擎，pkg/meta/sql.go 是 SQL 版本的元数据引擎。
* pkg/object 是与各种对象存储对接的实现
* sdk/java 是 Hadoop Java SDK 的实现，底层依赖 sdk/java/libjfs 这个库（通过 JNI 调用）。
pkg目录：
![](https://s4.ax1x.com/2021/12/08/ogfbRJ.png)
pkg目录下的各个目录基本都可以通过名字知道其大概功能。
juicefs是通过[https://github.com/urfave/cli](https://github.com/urfave/cli)这个go语言 commandline库来支持各种命令的。
程序的入口在cmd目录下的main.go文件中：

```go
#cmd/main.go
func main() {
    cli.VersionFlag = &cli.BoolFlag{
        Name: "version", Aliases: []string{"V"},
        Usage: "print only the version",
    }
    app := &cli.App{
        Name:                 "juicefs",
        Usage:                "A POSIX file system built on Redis and object storage.",
        Version:              version.Version(),
        Copyright:            "AGPLv3",
        EnableBashCompletion: true,
        Flags:                globalFlags(),
        Commands: []*cli.Command{
            formatFlags(),
            mountFlags(),
            umountFlags(),
            gatewayFlags(),
            syncFlags(),
            rmrFlags(),
            infoFlags(),
            benchFlags(),
            gcFlags(),
            checkFlags(),
            profileFlags(),
            statsFlags(),
            statusFlags(),
            warmupFlags(),
            dumpFlags(),
            loadFlags(),
        },
    }
    // Called via mount or fstab.
    if strings.HasSuffix(os.Args[0], "/mount.juicefs") {
        if newArgs, err := handleSysMountArgs(); err != nil {
            log.Fatal(err)
        } else {
            os.Args = newArgs
        }
    }
    err := app.Run(reorderOptions(app, os.Args))
    if err != nil {
        log.Fatal(err)
    }
}
```

可以看到main函数中包含了支持的各种命令：format, mount, umount, gateway, sync等等。

## format流程

使用过JuiceFS的朋友都知道，使用JuiceFS需要使用format和mount两个命令，这里就先介绍format流程，让大家都知道format究竟做了什么事情。我们先看format命令，这也是创建JuiceFS文件系统的第一步，通常我们会输入如下类似的命令：

```
$ juicefs format \
    --storage minio \
    --bucket http://127.0.0.1:9000/pics \
    --access-key minioadmin \
    --secret-key minioadmin \
    redis://127.0.0.1:6379/1 \
    pics
```

format整个过程的调用流程为：
![](https://s4.ax1x.com/2021/12/08/ogWRHK.png)
formatFlags()：这个函数没有什么好说的，就是按照```cli.Command```的结构封装了format命令，其中就注册了要执行的format()。
meta.NewClient(): 新建元数据服务的client，其中主要检查输入元数据相关参数的信息，根据传入的参数去调用具体的元数据服务，目前JuiceFS支持的元数据服务有redis/tikv/sql等,比如上面命令中是```redis://...```就会去调用redis的具体实现。

整个format过程中最重要的两点是**数据存储的创建**和**元数据服务的初始化**。这两点对应代码中函数为createStorage()和m.Init()。那么这两个函数究竟做了什么事情呢。

其实createStorage()主要是用传入的参数调用具体的对象存储，整个过程相当于创建了一个与对象存储连接的"client"，当然为了兼容市面上常见的对象存储，可以看到juicefs做了相当多的适配工作。这个createStorage的过程简单的说也就是根据参数去调用对应的对象存储，然后返回一个可以与对应的对象存储可以交互的"client"。

m.init()其实就是进行元数据的初始化，那么究竟需要初始化些什么呢。目前juicefs支持redis，tikv，sql等元数据服务，所以也对这几种服务进行相应的初始化实现。其实简单的说，就是在初始化的时候会去验证是不是已经创建过该juicefs的元数据了，如果没有就会生成一些关于juicefs的元数据信息，然后把这些信息写入对应的元数据服务中，比如使用Redis就会写入Redis中。如果有对应的信息，说明已经创建过juicefs，那么就会校验更新，如果有信息不匹配就会报“cannot update format from ”错误之类的。

## 参考资料

[JuiceFS源码](https://github.com/juicedata/juicefs)
[JuiceFS Office Hours 近期问题回顾](https://juicefs.com/blog/cn/posts/juicefs-office-hours-recap-2021-06-07/)