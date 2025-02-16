---
title: LevelDB源码阅读 -- Log Writer
date: 2020-01-25 23:31:30
categories:
- 源码阅读
tags: [源码阅读, leveldb]
---


![](https://s1.ax1x.com/2020/04/26/J6K7wT.png)

<!-- more -->

本文主要介绍leveldb的Log Writer


## LOG文件

LOG文件在leveldb中的主要作用是系统故障恢复时，能够保证不会丢失数据。因为在将记录写入内存的Memtable之前，会先写入LOG文件，这样即使系统发生故障，Memtable中的数据没有来得及Dump到磁盘的SSTable文件，LevelDB也可以根据LOG文件恢复内存的Memtable数据结构内容，不会造成系统丢失数据，在这点上LevelDb和Bigtable是一致的。(PS:实际代码实现时，p写文件也可以通过参数控制是立马刷盘还是交由操作系统自行控制，也就是说交由系统控制的话，也是可能会存在丢数据的情况)
![](https://s1.ax1x.com/2020/04/26/J6Mgnx.jpg)

通过上面的代码可以看出LOG文件的物理布局就是由连续的 32K 大小 Block 构成的。  
物理布局如下图：
![物理布局](https://s1.ax1x.com/2020/04/21/J3nBAs.md.png)  
在应用的视野里是看不到这些Block的，应用看到的是一系列的Key:Value对，在leveldb内部，会将一个Key:Value对看做一条记录的数据，另外在这个数据前增加一个记录头，用来记载一些管理信息，以方便内部处理，下图显示了记录头的结构：
![](https://s1.ax1x.com/2020/04/21/J3uVEj.png)

```
    checksum: uint32           // type及data[]对应的crc32值
    length:   uint16           // 数据长度
    type:     uint8            // FULL/FIRST/MIDDLE/LAST中的一种
    data:     uint8[length]    // 实际存储的数据
```
类型存在4种：

kFullType ： 顾名思义记录完全在一个block中
kFirstType ： 当前block容纳不下所有的内容，记录的第一片在本block中
kMiddleType ： 记录的内容的起始位置不在本block，结束未知也不在本block
kLastType ： 记录的内容起始位置不在本block，但 结束位置在本block

## log_writer.cc注释
![](https://s1.ax1x.com/2020/04/26/J6K0SA.png)

leveldb主要想解决的问题就是想尽量地把随机写改成顺序写。写入一个数据时，如果其大小超过32K，那么一个block是无法放下的，所以才有了记录的头部来进行管理，要不然也不知道哪个数据是哪个数据啊。具体代码逻辑见上面的注释。

## 参考资料：

[数据分析与处理之二（Leveldb 实现原理）](https://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html)   
[leveldb源码阅读（一）PUT流程](https://www.a-programmer.top/2020/01/20/leveldb%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%EF%BC%88%E4%B8%80%EF%BC%89PUT%E6%B5%81%E7%A8%8B/)   
[LevelDB源码解析7. 日志格式](https://zhuanlan.zhihu.com/p/35134533)  
[LevelDB源码解析11. 7个byte](https://zhuanlan.zhihu.com/p/44023605)

