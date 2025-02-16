---
title: NVMe namespace
date: 2021-10-30 23:30:30
categories:
- Linux
tags: [NVMe, namespace]
---

![](https://z3.ax1x.com/2021/11/19/IqmWqK.png)
<!-- more -->

本文主要介绍NVMe namespace。阅读完本文后，你会了解NVMe中的namespace概念，知道其作用及如何管理，并且可以了解常用的命令。
## 什么是NVMe namespace

NVMe 命名空间，**不是块的物理隔离**，而是可由主机软件寻址的逻辑块的隔离。

主机软件希望将NVMe SSD分解为多个名称空间的原因有很多:为了逻辑隔离、多租户、安全隔离(每个名称空间加密)、为了恢复目的对名称空间进行写保护、为了提高写性能和持久性而进行的overprovisioning等等。

## 命名空间(namespace)大小、容量和使用量
命名空间的数据结构包含相关的字段--大小，容量和使用量：
* 命名空间大小（NSZE）字段定义了命名空间在逻辑块中的总大小(LBA 0到n-1)。
* 命名空间容量（NCAP）字段定义了在任何时间点可以分配的逻辑块的最大数目。
* 命名空间使用量（NUSE）字段定义命名空间中当前分配的逻辑块的数量。这允许主机软件记录设备是如何响应deallocate命令的。在格式化之后，期望是这个值为零(没有使用量)，但在写入之后，使用量会增加，直到收到一个deallocate命令(有时称为“TRIM”)。

## nvme id命令包含了命名空间(namespace)能力的信息
除了了解命名空间(namespace)的大小和使用情况外，另一个重要属性是LBA格式。这有助于主机软件知道发送到命名空间(namespace)的命令的最佳大小，硬盘是否支持保护信息(端到端数据保护)等。
![](https://z3.ax1x.com/2021/11/19/Iqe53n.png)
<center>该图是linux上报告“nvme id 命名空间”命令的输出的屏幕截图 </center>



**注意**:命名空间可以附加到两个或多个控制器，称为**共享命名空间**。相反，命名空间可能只附加到一个控制器，称为**私有命名空间**。两者都由主机决定。

## 命名空间(namespace)如何管理
有两个命令集：**管理命令集和attachment命令集**  
<u>命名空间管理</u>：Create, Modify, or Delete  
<u>命名空间attachment</u>：Attach or detach


*  一旦主机创建了名称空间，它就不可见了。 
* 每个命名空间(namespace)由一个相应的NSID(命名空间ID)标识，每个命名空间(namespace)需要连接到子系统中的控制器。
![](https://z3.ax1x.com/2021/11/19/IqmA4H.png)
<center>图1:attached命名空间(Private和Shared)    NS A: Private，附加到控制器0   | NS B: Shared，附加到控制器0和1  | NS C z: Private，附加到控制器1 </center>


在图1中，主机创建了三个命名空间(namespace)。这不仅展示了NVMe子系统的抽象，还展示了如何利用私有和共享命名空间。每个命名空间被视为独立的目标设备。

## 多命名空间
### 多租户
一个用例可能涉及有两个或多个用户使用一个NVMe SSD。多个用户使用一个SSD会引起对服务一致性、专用性能和必须购买多个SSD的成本的担忧。这个时候namespace就起作用了，每个租户之间的逻辑隔离允许所有者根据租户的工作负载习惯处理每个命名空间。SSD仍然可以使用磨损级别，并在命名空间(namespace)之间共享用于垃圾收集的空闲区域。这与NVM Sets不同，在NVM Sets中，期望物理隔离，而不是命名空间(namespace)，后者是逻辑隔离。

![](https://z3.ax1x.com/2021/11/19/IqmWqK.png)
<center>图2:多租户用例 </center>

### Overprovisioning
SSD将使用未写入的LBA和一定数量的空闲区域进行垃圾收集、损耗均衡等。通过减少与设备上NAND闪存数量相关的命名空间(namespace)大小(称为预留空间)，这将提高耐久性、性能和服务质量。
![](https://z3.ax1x.com/2021/11/19/Iqn8eK.png)
<center> 例子：3.84TB NVMe SSD，典型的空闲区域和命名空间大小 </center>                   

* 主机不能直接访问未供应的空间，而是允许控制器在垃圾收集、TRIMs等期间使用该空闲区域。 
* 预留空间不仅对一个命名空间(namespace)有用，在多个名称空间的情况下也是如此。每个名称空间的预留空间的百分比可以增加或减少，以满足每个名称空间遇到的最频繁的工作负载。

使用nvme cli对命名空间(namespace)预留空间的步骤是：  
1.从每个控制器分离所有命名空间(namespace)(spec建议先分离，但也可以删除)。
```
nvme detach-ns /dev/nvme0 –namespace-id=1 –controllers=0
```
2.删除已分离的每个命名空间(namespace)
```
nvme delete-ns /dev/nvme0 –namespace-id=1
```
3.根据需要的容量，创建一个新的命名空间(对每个命名空间重复)
从7.68TB变到6.14TB
```
nvme create-ns /dev/nvme0 –nsze 11995709440 –ncap 1199570940 –flbas 0 –dps 0 –nmic 0
```
4.将新的命名空间附加到所需的控制器
```
nvme attach-ns /dev/nvme0 –namespace-id=1 –controllers=0
```
5.置设备以使目标对主机可见。
```
nvme reset /dev/nvme0
nvme list
```


## 参考资料
[https://nvmexpress.org/resources/nvm-express-technology-features/nvme-namespaces/](https://nvmexpress.org/resources/nvm-express-technology-features/nvme-namespaces/)