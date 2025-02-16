---
title: NVMe over RDMA的安装配置
date: 2021-10-27 23:30:30
categories:
- Linux
tags: [NVMe, NVMeoF, RDMA]
---

![](https://z3.ax1x.com/2021/11/11/I0LqZn.png)
<!-- more -->

本文主要介绍NVMe over RDMA的安装和配置。关于什么是NVMe over Fabrics,什么是NVMe over RDMA，本文就不做介绍了，网上资料一大堆。
可以看看[什么是NVMe over Fabrics？](https://www.sdnlab.com/24838.html)  
RDMA（全称：Remote Direct Memory Access）是一种远程直接内存访问技术，通过在硬件中实现传输层协议，将内存/消息原语接口暴露至用户空间，通过绕过CPU和内核网络协议栈来实现高吞吐和低延迟的网络。RoCE（RDMA over Converged Ethernet）是一种允许通过以太网使用远程直接内存访问（RDMA）的网络协议。

## RDMA安装配置

### RDMA驱动安装
1.下载驱动包 debian10.5
下载页面：https://www.mellanox.com/products/infiniband-drivers/linux/mlnx_ofed

![](https://z3.ax1x.com/2021/11/11/I0LEvQ.png)
选好对应的OS版本等信息后就可以点击下载：
![](https://z3.ax1x.com/2021/11/11/I0L32F.png)

2.安装驱动   
```
./mlnxofedinstall  --add-kernel-support --with-nvmf --force
./mlnxofedinstall --force --without-dkms --with-nvmf --add-kernel-support update-initramfs -u
```

### 验证RDMA的联通性
target端和client端都安装驱动后，可以验证RDMA的联通性  
1.查询服务器的ib device
```
#target端
:~#ibdev2netdev
mlx5_4 port 1 ==> eth4 (Down)
mlx5_5 port 1 ==> eth5 (Down)
mlx5_bond_0 port 1 ==> bond0 (Up)
mlx5_bond_1 port 1 ==> bond1 (Up)
#client端
:~# ibdev2netdev
mlx5_bond_0 port 1 ==> bond0 (Up)
mlx5_bond_1 port 1 ==> bond1 (Up)

```

2.开启target端

```
# ib_send_bw --ib-dev=mlx5_bond_1

************************************
* Waiting for client to connect... *
************************************
```

3.连接target端

```
:~# ib_send_bw --ib-dev=mlx5_bond_1 target端的ip
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF		Device         : mlx5_bond_1
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x117f PSN 0x3a5e58
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:07:32:205:201
 remote address: LID 0000 QPN 0x126d PSN 0xf975cb
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:07:32:205:198
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      1000             2757.96            2757.93		   0.044127
---------------------------------------------------------------------------------------
```
查看网卡支持的RoCE版本
```
$ show_gids //看看网卡支持的roce版本, 
DEV     PORT    INDEX   GID                                     IPv4            VER     DEV
---     ----    -----   ---                                     ------------    ---     ---
mlx5_4  1       0       fe80:0000:0000:0000:ee0d:9aff:fe2f:c21a                 v1      p1p1
mlx5_4  1       1       fe80:0000:0000:0000:ee0d:9aff:fe2f:c21a                 v2      p1p1
mlx5_4  1       2       0000:0000:0000:0000:0000:ffff:c0a8:0101 192.168.1.1     v1      p1p1
mlx5_4  1       3       0000:0000:0000:0000:0000:ffff:c0a8:0101 192.168.1.1     v2      p1p1
```

show_gids是一个脚本，脚本内容为：
```shell
#!/bin/bash

black='\E[30;50m'
red='\E[31;50m'
green='\E[32;50m'
yellow='\E[33;50m'
blue='\E[34;50m'
magenta='\E[35;50m'
cyan='\E[36;50m'
white='\E[37;50m'
bold='\033[1m'

gid_count=0

# cecho (color echo) prints text in color.
# first parameter should be the desired color followed by text
function cecho ()
{
    echo -en $1
    shift
    echo -n $*

    tput sgr0
}

# becho (color echo) prints text in bold.
becho ()
{
    echo -en $bold
    echo -n $*

    tput sgr0
}

function print_gids()
{
    dev=$1
    port=$2

    for gf in /sys/class/infiniband/$dev/ports/$port/gids/* ; do

    gid=$(cat $gf);
    if [ $gid = 0000:0000:0000:0000:0000:0000:0000:0000 ] ; then
        continue
    fi

    echo -e $(basename $gf) "\t" $gid

    done
}

echo -e "DEV\tPORT\tINDEX\tGID\t\t\t\t\tIPv4 \t\tVER\tDEV"
echo -e "---\t----\t-----\t---\t\t\t\t\t------------ \t---\t---"

DEVS=$1
if [ -z "$DEVS" ] ; then
    DEVS=$(ls /sys/class/infiniband/)
fi

for d in $DEVS ; do
    for p in $(ls /sys/class/infiniband/$d/ports/) ; do
        for g in $(ls /sys/class/infiniband/$d/ports/$p/gids/) ; do
            gid=$(cat /sys/class/infiniband/$d/ports/$p/gids/$g);
            if [ $gid = 0000:0000:0000:0000:0000:0000:0000:0000 ] ; then
                continue
            fi

            if [ $gid = fe80:0000:0000:0000:0000:0000:0000:0000 ] ; then
                continue
            fi

            _ndev=$(cat /sys/class/infiniband/$d/ports/$p/gid_attrs/ndevs/$g 2>/dev/null)
            __type=$(cat /sys/class/infiniband/$d/ports/$p/gid_attrs/types/$g 2>/dev/null)
            _type=$(echo $__type| grep -o "[Vv].*")

            if [ $(echo $gid | cut -d ":" -f -1) = "0000" ] ; then
                ipv4=$(printf "%d.%d.%d.%d" 0x${gid:30:2} 0x${gid:32:2} 0x${gid:35:2} 0x${gid:37:2})
                echo -e "$d\t$p\t$g\t$gid\t$ipv4 \t$_type\t$_ndev"
            else
                echo -e "$d\t$p\t$g\t$gid\t\t\t$_type\t$_ndev"
            fi

            gid_count=$(expr 1 + $gid_count)

        done #g (gid)
    done #p (port)
done #d (dev)

echo
echo n_gids_found=$gid_count

```

### NVMe Initiator 和 target 配置
Initiator 和 target 分别为 2 台服务器，target 端 NVMe SSD能通过 RoCE 方式连接到 Initiator 端，连接至 Initiator 端的 SSD 与本地的块设备使用方式一致，例如 mkfs，mount等操作。可以理解为多个 target 端 NVMe 盘可以横向扩展至 Initiator 端，组成 NVMe 存储池使用。

其中nvme discover -q host nqn中nqn是指是NVMe 合格名称 (NVMe Qualified Name) 的缩写
#### NVMe target端配置
1. 安装nvme-cli（apt-get install nvme-cli）
2. 配置target端

```
# NVMe target configuration
# Assuming the following:
# IP is 192.168.13.147/24
# link is up
# using ib device eth2
# modprobe nvme and rdma module

modprobe nvmet
modprobe nvmet-rdma
modprobe nvme-rdma

# 1、config nvme subsystem
mkdir /sys/kernel/config/nvmet/subsystems/nvme-subsystem-name
cd /sys/kernel/config/nvmet/subsystems/nvme-subsystem-name

# 2、allow any host to be connected to this target
echo 1 > attr_allow_any_host

# 3、create a namesapce，example: nsid=10
mkdir namespaces/10
cd namespaces/10

# 4、set the path to the NVMe device
echo -n /dev/nvme0n1> device_path
echo 1 > enable

# 5、create the following dir with an NVMe port
mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1

# 6、set ip address to traddr
echo "192.168.13.147" > addr_traddr

# 7、set rdma as a transport type，addr_trsvcid is unique.
echo rdma > addr_trtype
echo 4420 > addr_trsvcid

# 8、set ipv4 as the Address family
echo ipv4 > addr_adrfam

# 9、create a soft link
ln -s /sys/kernel/config/nvmet/subsystems/nvme-subsystem-name /sys/kernel/config/nvmet/ports/1/subsystems/nvme-subsystem-name

# 10、Check dmesg to make sure that the NVMe target is listening on the port
dmesg -T| grep "enabling port"
[369910.403503] nvmet_rdma: enabling port 1 (192.168.13.147:4420)`
```

#### 配置initiator端
initiator 端，又称为host/client端，initiator 配置前提：RDMA基础环境已搭建。通过NVMe 互联命令探测和连接target 端 NVMe SSD 即可。
```
#探测 192.168.13.147 机器上 4420 端口 nvme ssd
nvme discover -t rdma -q nvme-subsystem-name -a 192.168.13.147 -s 4420

#连接 192.168.13.147 4420 端口 nvme ssd
nvme connect -t rdma -q nvme-subsystem-name -n nvme-subsystem-name  -a 192.168.13.147 -s 4420

#显示NVMe盘
lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0  1.1T  0 disk
├─sda1    8:1    0    8G  0 part /
├─sda2    8:2    0    1K  0 part
├─sda5    8:5    0    4G  0 part [SWAP]
├─sda6    8:6    0    8G  0 part /usr
├─sda7    8:7    0    8G  0 part /var
├─sda8    8:8    0    4G  0 part /tmp
└─sda9    8:9    0  1.1T  0 part /home
nvme0n1 259:1    0  1.5T  0 disk

#与target 端 nvme ssd 断开连接
nvme disconnect -n nvme-subsystem-name
```

## 安装配置过程中遇到的问题
### 1. 如何配置多个NVMe盘的问题
一般的配置文档都是只映射了一个NVMe盘到远端客户端，然后自己用如下两种思路尝试摸索：
* 按单个NVMe盘的操作思路来，多个盘就做多次，也就是这种思路创建了多个NVMe子系统，然后再客户端做多次nvme connect
* 只做单个子系统，在单个子系统下面创建多个命名空间（相当于每个命名空间对应一个NVMe盘）

实践发现第一个思路操作没有成功，按第二个思路操作成功了！不过在操作的时候需要注意，不同的盘需要对应不同的端口。
最后因为都是一个子系统，所以connect的时候就不用输端口了，命令如下：
```
nvme connect -t rdma -q ahnselina_test -n ahnselina_test -a x.x.x.x
```
请把上面命令中的"ahnselina_test"换成你自己创建的子系统名称，x.x.x.x替换成你那台机器上的IP。

### 2. 使用nvme connect遇到的问题
使用nvme connect遇到报"Failed to write to /dev/nvme-fabrics: Input/output error"错误。
```
nvme connect -t rdma -q ahnselina_test -n ahnselinatest -a x.x.x.x
Failed to write to /dev/nvme-fabrics: Input/output error
```
错误原因：通过看dmesg -T｜ grep nvme发现是自己把nqn名称打错了，尴尬，把名称该正确即可:
```
nvme connect -t rdma -q ahnselina_test -n ahnselina_test -a x.x.x.x
```






## 参考资料
[HowTo Install MLNX_OFED Driver](https://community.mellanox.com/s/article/howto-install-mlnx-ofed-driver)

[HowTo Configure NVMe over Fabrics](https://community.mellanox.com/s/article/howto-configure-nvme-over-fabrics)

[NVME OVER FABRIC 设备概述](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/managing_storage_devices/overview-of-nvme-over-fabric-devicesmanaging-storage-devices)


[NVMe over RoCE 初探](https://zhuanlan.zhihu.com/p/345609031)  
[浅析RoCE网络技术](https://www.sdnlab.com/24673.html)

