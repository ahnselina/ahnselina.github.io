---
title: ZFS系列（三）ZFS intent log
date: 2021-10-09 23:30:30
categories:
- Linux
tags: [Linux, 文件系统, ZFS, ZIL]
---

![](https://z3.ax1x.com/2021/10/09/5kRU9e.png)

<!-- more -->

本文介绍ZFS intent log（ZIL），SLOG，以及如何往zpool添加SLOG虚拟设备。

## 术语
在开始之前，我们需要弄清楚一些术语，这些术语出现在论坛、博客文章、邮件列表和一般讨论中似乎会让人们感到困惑。直到写这篇文章之前，尽管我明白最终目标是什么，这些术语有时也让我很困惑。所以，让我们来看看这些术语:
* ZFS Intent Log或者叫ZIL，一种日志记录机制，其中存储所有要写入的数据，然后作为事务性写入刷新。功能类似于日志文件系统(如ext3或ext4)的日志。通常存储在盘片磁盘上。包含一个ZIL头，它指向一个记录列表、ZIL块和一个ZIL trailer。对于不同的写入，ZIL的行为是不同的。对于小于64KB(默认情况下)的写操作，ZIL直接存储写入的数据。对于较大的写操作，写操作不存储在ZIL中，而ZIL维护指向存储在日志记录中的同步数据的指针。
* Separate Intent Log, or SLOG，一个单独的日志记录设备，在将ZIL的同步部分刷新到较慢的磁盘之前缓存它们。这设备是一个电池支持的DRAM驱动器或者是快速SSD。SLOG只缓存同步数据，不缓存异步数据。异步数据将直接刷新到旋转磁盘。而且，块是一次写入块，而不是作为同步事务写入SLOG。如果已经存在SLOG，则ZIL将被移动到它上面，而不是驻留在硬盘上。SLOG中的所有内容都将始终保存在系统内存中。

当你在网上读到有人提到“添加一个SSD的ZIL到池中”，他们是指添加一个SSD的SLOG，ZIL将驻留在SSD盘上。在本例中，ZIL是SLOG的一个子集。**SLOG是设备，ZIL是设备上的数据**。此外，并不是所有的应用程序都使用了ZIL。数据库(MySQL、PostgreSQL、Oracle)、NFS和iSCSI targets等应用都使用了ZIL。典型的文件系统数据复制不会使用ZIL。最后，通常不会读取ZIL，除非在引导时查看是否有丢失的事务。ZIL基本上是“只写”的，并且是非常密集的写。

## SLOG设备
哪种设备最适合SLOG?按照最快到最慢的顺序如下：
1. NVRAM，  由电池支持的DRAM驱动器，如STEC的ZeusRAM SSD。最快也是最可靠的，也是最昂贵的。
2. SSD，带有损耗均衡算法的NAND闪存芯片。比如PCI-Express OCZ固态硬盘或英特尔。最好是SLC，尽管SLC和MLC ssd之间的差距正在缩小。
3. 10k+的SAS盘， 企业级，旋转盘片。SAS和光纤通道驱动器推动IOPS超过吞吐量，通常是消费级SATA的两倍。在列出来的这三种中是最慢也是最不可靠的。也最便宜。

重要的是要确定上面列出的所有三个设备在断电期间都能保持数据持久性。SLOG和ZIL在将数据传输到旋转盘过程中是非常关键的。如果发生停电，并且您有一个volatile SLOG，最糟糕的情况是新数据没有刷新，只剩下旧数据。然而，重要的是要注意，在断电的情况下，您不会有损坏的数据，只有丢失的数据，您的数据在磁盘上仍然是一致的。

## SLOG性能
因为SLOG是快速磁盘，所以我期望看到的应用程序或系统的性能是怎样的呢?您将看到改进的磁盘延迟、磁盘利用率和系统负载。您不会看到吞吐量的提高。请记住，SLOG设备仍然每5秒将数据刷新到磁盘。因此，在添加了SLOG设备之后对磁盘进行基准测试没有多大意义，除非基准测试的目标是测试同步磁盘写延迟。所以，我没有足够的数据供你参考。不过，我有一些图表。

我有一个磁盘写密集的虚拟机。它是ZFS数据集上的GlusterFS复制文件系统上的磁盘映像。我在管理程序中有足够的RAM，一个快速的CPU，但很慢的SATA磁盘。由于这个虚拟机上的应用程序想要频繁地将许多图写入磁盘，随着图的数量的增加，我看到了大约5-10秒的磁盘延迟。磁盘上的吞吐量非常高。因此，在VM中进行任何写操作都很痛苦。系统升级，修改配置文件，甚至登录，一切都非常非常缓慢。
因此，我对ssd进行了分区，并添加了SLOG。立即，我的磁盘延迟下降到大约200毫秒。磁盘利用率从繁忙的50%左右下降到5%左右。系统负载从1-2下降到几乎不存在。与磁盘相关的一切都处于更加健康的状态，VM也更加高兴。为了证明这一点，请看下面从http://zen.ae7.st/munin/保存下来的图表。

第一张图片从系统管理程序的角度显示了我的磁盘。请注意，每个设备的吞吐量大约是800KBps。添加SSD SLOG后，吞吐量降至400KBps。这意味着池中的底层磁盘执行的工作更少，因此持续时间更长。
![](https://z3.ax1x.com/2021/10/09/5kWF8e.md.png)

下一张图片从虚拟机的角度显示了我的磁盘。注意磁盘延迟和利用率是如上所述下降的，包括系统负载。

![](https://z3.ax1x.com/2021/10/09/5kWuUf.md.png)

几天前，我在http://pthree.org/2012/12/03/how-a-zil-improves-disk-latencies/上写了一篇关于这个的博客。


## 添加一个SLOG
**警告**:一些主板在重新启动时不会以一致的方式向Linux内核提供磁盘。因此，在一次引导时标识为/dev/sda的磁盘可能在下一次引导时标识为/dev/sdb。对于存储数据的主池，这不是问题，因为ZFS可以基于元数据几何结构重建VDEVs。然而，对于您的L2ARC和SLOG设备，不存在这样的元数据。那么，与其通过它们的/dev/sd?名字，将它们添加到池中，还不如用/dev/disk/by-id/*名称将其加入池中，因为这些是符号指针，指向不断变化的/dev/sd?文件。如果您没有注意到这个警告，那么您的SLOG设备可能根本就不会被添加到您的混合池中，您需要稍后重新添加它。这可能会极大地影响应用程序的性能，这取决于是否存在一个快速的SLOG。

 将SLOG添加到现有的zpool中并不困难。然而，对SLOG做镜像被认为是SLOG的最佳实践。因此，在本例中，我将遵循最佳实践。假设我的池中有4个普通硬盘，一个OCZ Revodrive SSD向系统提供两个60GB驱动器。我将在SSD上对驱动器进行分区，分区大小为5 GB，然后将分区镜像为我的SLOG。这就是如何将SLOG添加到池中。在这里，我使用GNU parted首先创建分区，然后添加ssd。“/dev/disk/by-id/”中的设备分别指向“/dev/sda”和“/dev/sdb”。仅供参考。
```
# parted /dev/sda mklabel gpt mkpart primary zfs 0 5G
# parted /dev/sdb mklabel gpt mkpart primary zfs 0 5G
# zpool add tank log mirror \
/dev/disk/by-id/ata-OCZ-REVODRIVE_OCZ-69ZO5475MT43KNTU-part1 \
/dev/disk/by-id/ata-OCZ-REVODRIVE_OCZ-9724MG8BII8G3255-part1
# zpool status
  pool: tank
 state: ONLINE
 scan: scrub repaired 0 in 1h8m with 0 errors on Sun Dec  2 01:08:26 2012
config:

        NAME                                              STATE     READ WRITE CKSUM
        pool                                              ONLINE       0     0     0
          raidz1-0                                        ONLINE       0     0     0
            sdd                                           ONLINE       0     0     0
            sde                                           ONLINE       0     0     0
            sdf                                           ONLINE       0     0     0
            sdg                                           ONLINE       0     0     0
        logs
          mirror-1                                        ONLINE       0     0     0
            ata-OCZ-REVODRIVE_OCZ-69ZO5475MT43KNTU-part1  ONLINE       0     0     0
            ata-OCZ-REVODRIVE_OCZ-9724MG8BII8G3255-part1  ONLINE       0     0     0
```

##  SLOG预期生命周期
因为您可能会在GNU/Linux服务器中为您的SLOG使用消费级SSD，所以我们需要提到SSD在写密集型场景中的损耗。当然，这将在很大程度上取决于制造商，但我们可以设置一些概括性。

首先，ZFS拥有先进的磨损均衡算法，可以均匀地磨损SSD上的每个芯片，这不需要TRIM支持。由于文件系统的写时复制特性，ZFS的损耗是固有的。

其次，不同的硬盘采用不同的纳米工艺。纳米过程越小，SSD的寿命就越短。例如，Intel 320是一个25纳米的MLC 300GB SSD，额定P/E周期约为5000。这意味着如果使用磨损均衡算法，您可以写入整个SSD 5000次。这将产生总计1500000 GB的写入数据，即1500TB。我的ZIL每秒维护大约3 MB的数据。因此，我每年可以维护大约95 TB的写入数据。这给了我这个英特尔固态硬盘大约15年的寿命。

然而，英特尔335是一个20纳米MLC 240GB SSD，额定约3000 P/E周期。在损耗均衡的情况下，这意味着您可以写入整个SSD 3000次，这将产生总写入数据的720TB。对于3MBps 的ZIL来说，这仅仅是能用7年，这还不到Intel 320预期寿命的1/2。重点是，在规划存储池zpool的时候，你需要注意这些事情。

现在，如果您使用的是电池支持的DRAM驱动器，那么磨损水平不是问题，内存可能跟您的服务器的耐用时间差不多的。同样的说法也适用于10k以上的SAS或FC硬盘。

## 容量
只是一个简短的说明，您可能不需要一个大的ZIL。我将我的ZIL分区为仅4GB的可用空间，它几乎只占用了一到两MB的空间。我将所有虚拟机都放在同一个管理程序上，运行操作系统升级，同时它们也在做大量的工作，但也仅仅看到ZIL达到大约100 MB的缓存数据。我无法想象您需要什么样的工作负载才能使您的ZIL超过1 GB已用空间，更不用说4 GB了。这里有一个命令，你可以运行来检查你的ZIL的大小:
```
# zpool iostat -v tank
                                                     capacity     operations    bandwidth
tank                                              alloc   free   read  write   read  write
------------------------------------------------  -----  -----  -----  -----  -----  -----
tank                                               839G  2.81T     76      0  1.86M      0
  raidz1                                           839G  2.81T     73      0  1.81M      0
    sdd                                               -      -     52      0   623K      0
    sde                                               -      -     47      0   620K      0
    sdf                                               -      -     50      0   623K      0
    sdg                                               -      -     47      0   620K      0
logs                                                  -      -      -      -      -      -
  mirror                                          1.46M  3.72G     20      0   285K      0
    ata-OCZ-REVODRIVE_OCZ-69ZO5475MT43KNTU-part1      -      -     20      0   285K      0
    ata-OCZ-REVODRIVE_OCZ-9724MG8BII8G3255-part1      -      -     20      0   285K      0
------------------------------------------------  -----  -----  -----  -----  -----  -----
```

## 总结
快速SLOG可以为需要较低同步事务延迟的应用程序提供惊人的好处。这对于数据库服务器或其他时间敏感的应用程序很有效。然而，将SLOG添加到您的池中会增加成本。电池驱动的DRAM芯片非常非常昂贵。通常是每8GB DDR3内存2500美元，而40GB MLC SSD只需要100美元，600GB 15k SAS需要200美元。不过，容量确实不是问题，性能才是。我希望SSD上的IOPS更快，容量更小。除非你想分区它，并在同一个驱动器上共享L2ARC，这是个好主意，我将在下一篇文章中讨论。

## 补充

1. ZIL是ZFS的写入日志，即使没有添加独立高速ZIL，它也存在于储存池内。
2. ARC和L2ARC才是缓存，读和写的缓存。
3. **不添加ZIL也可以设置后实现极高的写入性能**。
4. 非要配ZIL，至少16G傲腾起步，SATA固态都是废物。

ZIL起到的作用是，对于**同步写**而言，写入ZIL就视为落盘写，返回写成功。然后ZFS再视情况把ZIL的写入同步到数据盘上去。因此高速ZIL可以有效加速写入。此外ZIL还必须要高可靠，起到在意外断电等情况下保护还未写入数据的作用。

但是家用NAS的使用场景（如SMB共享等）绝大部分是**异步写**，这时候写入会直接进内存ARC排队，而无需关心是否落盘。问题是一旦停电，那么还在内存里排队的写入就全都凉了，因此不够安全。

至于性能上，在不同步时有没有ZIL几乎无影响；强制同步时即使有傲腾900P做独立ZIL，随机写入速度也惨不忍睹。

## 参考资料
[https://pthree.org/2012/12/06/zfs-administration-part-iii-the-zfs-intent-log/](https://pthree.org/2012/12/06/zfs-administration-part-iii-the-zfs-intent-log/)  
[ZIL不是ZFS的写缓存](https://zhuanlan.zhihu.com/p/63991068)


