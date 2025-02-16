---
title: 测试NVMe over RDMA上的ZFS遇到的问题
date: 2021-10-29 23:30:30
categories:
- Linux
tags: [NVMe, NVMeoF, RDMA, ZFS]
---

![](https://z3.ax1x.com/2021/11/17/IIVnhT.png)
<!-- more -->

本文主要介绍在测试构建在NVMe over RDMA上的ZFS遇到的问题及解决方法。

## 问题：contains a corrupt primary EFI label.
```
zpool create nvme_over_rdma_zfs_pool raidz2 nvme0n1 nvme0n2 nvme0n3 nvme0n4 nvme1n1 nvme1n2
invalid vdev specification
use '-f' to override the following errors:
/dev/nvme0n2 contains a corrupt primary EFI label.
```

**解决方法**：
```
wipefs -a /dev/nvme0n2
```

## 问题：使用nvme delete-ns /dev/nvme2n1删除该盘的ns之后，使用lsblk看不见该盘了
**解决方法**：
https://www.ibm.com/docs/en/linux-on-systems?topic=drive-deleting-stray-nvme-namespaces-nvme


## 问题：由于前面模拟拔盘和重新插盘后，那个盘符发生了变化，所以导致通过RDMA映射到本地客户端创池失败
**解决方法**：尝试了想修改nvme子系统中盘符，但是nvme子系统此时不让进行修改，想卸载nvmet也不行。
最后发现重启之后就可以了。


## 问题：重启后安装的nvme相关驱动不见了，自己做的设置也消失了（这类加载驱动后开机重启后没有驱动的解决方法）
**解决方法**：为了使命令行中执行insmod命令安装的驱动能在重启之后还有的解决方法大致有一下两种方法：
（1）直接编译内核，把想安装的驱动在编译内核的时候给编译进去，这种方法比较麻烦，花的时候也比较的多，一般不采用这种方法（这种方法网上有很多资料可以参考）。
（2）这种方法比较简单：就是在启动脚本中加载模块，这样每次开机启动都自动加载相应的驱动模块。具体的方式如下：
      在文件/etc/rc.local中加载你想要的模块程序即可，比如我想再系统启动的时候自动给我完成：卸载r8169驱动、然后安装r8168驱动、同时重启网络服务器的工作，那么我在该文件中的exit 0 之前写如下的语句：
```
rmmod r8169#（卸载相应的驱动）

insmod /usr/src/r8168.ko#(这个是r8168.ko文件放的绝对路径)，这步是安装相应的驱动

ifconfig eth0 down

ifconfig eth0 up
```
然后保存，重新启动reboot之后，系统就将自动完成我们想要的那几步工作。

在加载好驱动之后，还需要在脚本中执行自己想要做的设置！

加载NVME模块及NVME映射相关的配置：
```
#!/bin/bash

LOG="/var/log/nvme_over_rdma.log"
NVME_SUBSYSTEM_PATH="/sys/kernel/config/nvmet/subsystems/"
NVME_SUBSYSTEM_DIR="mon1_nvme"
NVME_PORT_PATH="/sys/kernel/config/nvmet/ports"
NAME_SPACE_NUM=11
PORT=4420

echo "$(date "+%Y-%m-%d %H:%M:%S")  start installing nvme driver..." | tee -a ${LOG}

# 加载模块
modprobe nvmet
modprobe nvmet-rdma
modprobe nvme-rdma

# 创建nvme的subsystem
echo "$(date "+%Y-%m-%d %H:%M:%S")  create the subsystem of nvme." | tee -a ${LOG}
mkdir /sys/kernel/config/nvmet/subsystems/$NVME_SUBSYSTEM_DIR
cd /sys/kernel/config/nvmet/subsystems/$NVME_SUBSYSTEM_DIR
echo 1 > attr_allow_any_host


j=1;
for i in `lsscsi | grep nvme | awk '{print $4}'`;
do
    # 创建命名空间
    echo "$(date "+%Y-%m-%d %H:%M:%S")  create namespace ${NAME_SPACE_NUM}." | tee -a ${LOG};
    mkdir namespaces/$NAME_SPACE_NUM;
    # 设置NVME盘的设备路径
    echo "$(date "+%Y-%m-%d %H:%M:%S")  set the NVMe disk device path: $i." | tee -a ${LOG};
    echo -n $i > namespaces/${NAME_SPACE_NUM}/device_path;
    echo 1 > namespaces/${NAME_SPACE_NUM}/enable;
    # 创建NVME的port
    echo "$(date "+%Y-%m-%d %H:%M:%S")  create the NVMe's port:${PORT}." | tee -a ${LOG};
    mkdir ${NVME_PORT_PATH}/${j};

    # 设置rdma的ip
    echo `ifconfig bond1  | grep inet | awk '{print $2}'` > ${NVME_PORT_PATH}/${j}/addr_traddr;
    echo rdma > ${NVME_PORT_PATH}/${j}/addr_trtype;
    echo ${PORT} > ${NVME_PORT_PATH}/${j}/addr_trsvcid;
    echo ipv4 > ${NVME_PORT_PATH}/${j}/addr_adrfam;
    ln -s ${NVME_SUBSYSTEM_PATH}/${NVME_SUBSYSTEM_DIR} ${NVME_PORT_PATH}/${j}/subsystems/${NVME_SUBSYSTEM_DIR};
    ((j++));
    ((PORT++));
    ((NAME_SPACE_NUM++));

done
echo "$(date "+%Y-%m-%d %H:%M:%S")  installed nvme success." | tee -a ${LOG}
```

## NVMe over RDMA的ZFS客户端重启后，ZFS存储池没了
**解决方法**：  
需要重新加载
```
modprobe nvme-rdma
```
然后尝试重新创池发现"is part of potentially active pool 'nvme_over_rdma_zfs_pool'"类型错误，其实是正常的，那些盘原本已经被我创建了存储池，不让创建是对的，所以解决办法是恢复原先创建的那个存储池。

恢复方法为：
```
zpool import -n nvme_over_rdma_zfs_pool -F
```
把上面的存储池的名字换成你自己的要恢复的存储池名字。
[[HELP] Trying to recreate an existing pool](https://www.reddit.com/r/zfs/comments/a5jdkp/help_trying_to_recreate_an_existing_pool/ebmxzlw/)

## 通过NVMe over RDMA映射组建的ZFS的硬盘盘符，在重启后发生变化
通过NVMe over RDMA映射到一个客户端上组建的ZFS，重启后端NVMe盘的机器后，盘符会发生改变，然后ZFS存储池中会检测到原来的那个盘符故障，然后也没法用"zpool replace"命令。

**解决方法**：
 
1. 重启客户端(即创建ZFS存储池的那台机器) 
2. 重新加载nvme-rdma 
3. **使用“nvme connect xxx”命令重新连接其他机器上的NVMe盘**
4.   **重新import存储池**
```
zpool import -n nvme_over_rdma_zfs_pool -F
```
在创建ZFS存储池的时候，最佳实践是不要使用/dev/sda这样的字盘符来创建存储池，可以使用"/dev/disk/by-id/xxx"这样字独一无二的标识符来创池。

## 参考资料
[加载驱动后开机重启后没有驱动的解决方法](https://blog.csdn.net/liangxanhai/article/details/7748271)

[[HELP] Trying to recreate an existing pool](https://www.reddit.com/r/zfs/comments/a5jdkp/help_trying_to_recreate_an_existing_pool/ebmxzlw/)

[zpool lost after reboot](https://www.reddit.com/r/zfs/comments/gh4svd/zpool_lost_after_reboot/)

[zpools don't automatically mount after boot](https://askubuntu.com/questions/404172/zpools-dont-automatically-mount-after-boot)
