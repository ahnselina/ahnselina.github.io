---
title: lizardfs在同一个服务器上运行多个chunkserver的方法
date: 2019-01-27 21:21:20
categories: 
- 分布式存储
tags: [LizardFS, lizardfs, 分布式存储, 文件存储]
---
有时候设备不够，需要在一台服务器上运行多个chunkserver，本文介绍这种场景下如何用LizardFS在同一个服务器上运行多个chunkserver。

### 运行多个chunkserver的方法
1. 为新增的chunkserver准备一个新的 mfschunkserver.cfg配置文件，比如/etc/chunkserver2.cfg
2. 为新增的chunkserver准备一个新的mfshdd.cfg 配置文件，比如/etc/hdd2.cfg
3. 把  /etc/chunkserver2.cfg  文件中的HDD_CONF_FILENAME值 设置为新添加的mfshdd.cfg配置文件路径，在本例中HDD_CONF_FILENAME值为 /etc/hdd2.cfg
4. 把/etc/chunkserver2.cfg文件中的CSSERV_LISTEN_PORT项的值改为非默认值，设置为未被使用的端口（比如9522）
5. 运行第二个chunkserver用如下命令：
 mfschunkserver -c /etc/chunkserver2.cfg  
6. 如果你需要在同一个服务器上创建更多的chunkservers可以重复上述步骤（然而并不推荐这样做）

### 遇到的问题

#### 按照上面的操作执行之后没有看到多个chunkserver进程
按照上面方法操作遇到一个常见的问题是，启动一个新的chunkserver进程后，原有的chunkserver进程就自动退出了，也就是说，通过ps -ef | grep mfschunkserver查看进程，始终只有一个chunkserver进程并且就是刚刚启动的那个！

经过多方尝试找到原因为：
配置文件中/etc/chunkserver2.cfg与/etc/mfs/mfschunkserver.cfg的**DATA_PATH项**的值相同，而DATA_PATH是表示存使用统计文件和锁文件的目录，所以要启动多个chunkserver进程，除上述操作外，需要确保chunkserver配置文件中的DATA_PATH项的值是不相同，以免起冲突。  
**解决方法：**  
为每个chunkserver的配置文件中**DATA_PATH项**都配置一个不同的值。


#### 权限错误
除上述问题外，还需要注意 DATA_PATH项的值的目录需要是属于mfs:mfs的，否则会报权限错误
如果不是，可以使用“chown -R mfs:mfs 目录” 更改其用户组使得有权限

### 参考资料
[Running multiple chunkservers on the same machine.](https://lizardfs.com/running-multiple-chunkservers-on-the-same-machine/)
