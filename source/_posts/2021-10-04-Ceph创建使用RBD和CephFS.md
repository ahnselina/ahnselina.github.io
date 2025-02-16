---
title: Ceph之RBD及CephFS
date: 2021-10-04 23:30:30
categories:
- 分布式存储
tags: [Ceph, RBD, CephFS]
---

![](https://z3.ax1x.com/2021/10/04/4OHOYR.png)

<!-- more -->

本文主要介绍如何在搭建好的Ceph环境上创建RBD以及如何创建CephFS。  

## 创建RBD并将其挂载到操作系统

### 1、创建rbd使用的pool
```
#32 32: PG数和PGP数(PG内包含的对象);PG会随着容量的增加也会动态的扩容；生产上需要提前规划好数量
ceph osd pool create rbd  32 32
#给rbd使用的pool标识成rbd
ceph osd pool application enable rbd rbd 

#查看rbd的pool
[root@cephnode01 my-cluster]# ceph osd pool ls detail
pool 5 'rbd' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode warn last_change 68 flags hashpspool stripe_width 0
```

### 2、创建一个10G的块设备
```
rbd create --size 10240 image01 
```

### 3、查看块设备
```
rbd ls
rbd info image01
rados -p rbd ls --all
```

### 4、禁用当前系统内核不支持的feature
```
rbd feature disable image01 exclusive-lock, object-map, fast-diff, deep-flatten
```

### 5、将块设备映射到系统内核
```
rbd map image01 
rbd showmapped
```

### 6、格式化块设备镜像
```
mkfs.xfs /dev/rbd0
```

### 7、mount到本地
```
mount /dev/rbd0 /mnt
```

### 8、创建文件并查看
```
[root@cephnode01 my-cluster]# touch /mnt/{a..g}lcx.txt
[root@cephnode01 my-cluster]# rados -p rbd ls --all
    rbd_data.28209e4598ff7.0000000000000500
    rbd_id.image01
    rbd_header.28209e4598ff7
    rbd_data.28209e4598ff7.0000000000000820
    rbd_directory
    rbd_data.28209e4598ff7.0000000000000460
    rbd_data.28209e4598ff7.00000000000006e0
    rbd_info
    rbd_data.28209e4598ff7.00000000000000a0
    rbd_data.28209e4598ff7.00000000000009ff
    rbd_data.28209e4598ff7.00000000000005a0
    rbd_data.28209e4598ff7.0000000000000000
    rbd_data.28209e4598ff7.0000000000000780
...
```

### 9、取消块设备和内核映射
```
umount /mnt
rbd unmap image01 
```

### 10、删除RBD块设备
```
rbd rm image01
```
>如果想在其他机器上使用ceph块设备需要下载ceph-common客户端


## 创建CephFS
### 启动MDS（MDS是CephFS使用的元数据管理服务）
创建CephFS需要MDS，首先介绍MDS的启动：

#### 1. 创建mds存放目录
```
mkdir -p /var/lib/ceph/mds/{cluster-name}-{id}
例如：
mkdir -p /var/lib/ceph/mds/ceph-jchuan-ceph-1
```

#### 2. 创建keyring
```
ceph-authtool --create-keyring /var/lib/ceph/mds/{cluster-name}-{id}/keyring --gen-key -n mds.{id}
例如：
ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-jchuan-ceph-1/keyring --gen-key -n mds.jchuan-ceph-1
```

#### 3. 导入keyring并设置权限
```
ceph auth add mds.{id} osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/{cluster}-{id}/keyring
chmod 644 /var/lib/ceph/mds/{cluster}-{id}/keyring
例如：
ceph auth add mds.jchuan-ceph-1 osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-jchuan-ceph-1/keyring
```

#### 4. 添加到ceph.conf
```
[mds.{id}]
host = {id}
```

#### 5. 安装ceph-mds(原文档少了这一步，执行下一步就会启动不成功，因为找不到ceph-mds)
```
apt-get install ceph-mds
```

#### 6. 手动启动进程或者直接用systemctl启动
```
systemctl start ceph-mds@jchuan-ceph-1
或者
ceph-mds --cluster {cluster-name} -i {id} -m {mon-hostname}:{mon-port} [-f]
```



### 创建启用CephFS
Ceph文件系统至少需要两个RADOS池，一个用于数据，一个用于元数据
#### 1. 创建数据pool
```
ceph osd pool create cephfs_data 256 256
```

#### 2. 创建Metadata池
```
ceph osd pool create cephfs_metadata 128 128
```

#### 3. 启用pool
```
ceph fs new cephfs cephfs_metadata cephfs_data
```

#### 4. 挂载CephFS
挂载CephFS可以使用FUSE client或者kernel client(挂载cephfs有两种方式，kernel driver和fuse):
FUSE client是最容易访问的，也是最容易升级到存储集群使用的Ceph版本的，而kernel client能够提供更好的性能。
当遇到bug或者性能问题时，尝试使用其他客户端通常是有指导意义的，以便查明bug是否属于特定的客户端(然后让开发人员知道)。   
 
**挂载CephFS前的通用先决条件：**  
在挂载CephFS前，确保client机器（将挂载和使用CephFS的那台机器）有一份Ceph配置文件的复本（即ceph.conf）以及一个CephX用户有权限访问MDS的keyring。这两个文件必须已经存在于Ceph MON所在的主机上。   
（1）为客户端主机生成一个最小的conf文件，并将其放在一个标准位置(如果已经存在ceph.conf就忽略该步骤):
```
# on client host
mkdir -p -m 755 /etc/ceph
ssh {user}@{mon-host} "sudo ceph config generate-minimal-conf" | sudo tee /etc/ceph/ceph.conf
```

（2）调整配置文件的权限
```
chmod 644 /etc/ceph/ceph.conf
```

(3) 创建一个CephX用户并获取其密钥
```
ssh {user}@{mon-host} "sudo ceph fs authorize cephfs client.foo / rw" | sudo tee /etc/ceph/ceph.client.foo.keyring
```

在上面的命令中，用你自己的CephFS的名字取代“cephfs”


```
创建完成后可以通过ceph auth list 查看keyring跟文件里面是否一致，权限是否正常
chmod 600 /etc/ceph/ceph.client.foo.keyring
```

(4) 用kernel driver挂载CephFS
```
mkdir /mnt/mycephfs
mount -t ceph :/ /mnt/mycephfs -o name=foo
```
使用ceph-fuse挂载参考文档：https://docs.ceph.com/en/latest/cephfs/mount-using-fuse/

#### 遇到的错误及解决办法
1. 遇到错误“unable to get monitor info from DNS SRV with service name: ceph-mon”   
**原因**：配置文件ceph.conf中没有相关的配置信息  
**解决办法**：去其他正常节点拷贝一份配置文件过来即可

2. 遇到错误“[errno 2] error connecting to the cluster”  
**解决办法**：在其他可以执行该命令的节点上找到ceph.client.admin.keyring
find / -name ceph.client.admin.keyring
把该文件拷贝一份到报错节点即可。


## 参考资料
[Ceph—RBD块设备介绍与创建](https://www.jianshu.com/p/712e58d36a77)  
[探索RBD块存储接口](https://cloud.tencent.com/developer/article/1592961)  
[分布式文件系统CephFS](https://durantthorvalds.top/2021/01/17/2021117-「核心」Ceph学习三部曲之八：分布式文件系统CephFS/)  
[ceph-fuse挂载参考文档](https://docs.ceph.com/en/latest/cephfs/mount-using-fuse/)