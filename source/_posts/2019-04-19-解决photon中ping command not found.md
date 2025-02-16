---
title: 解决photon中ping command not found
date: 2019-04-19 16:11:30
categories:
- 云计算
tags: [harbor, photon]
---


![](https://z3.ax1x.com/2021/05/04/gmvtoQ.jpg)

<!-- more -->
  
本文主要解决photon中ping command not found的问题

## photon介绍
![](http://static.oschina.net/uploads/img/201512/20112919_P9RS.png)
VMware推出的Linux操作系统Photon，为vSphere优化的Container执行平台。
Photon是以提供Container执行环境为目的的轻量操作系统。

虚拟化厂商VMware在2015年4月推出了自家的Linux版本Photon，用于打造轻量级虚拟化Container环境的轻量级Linux操作系统。

Photon提供4种安装模式，而4种模式所占的镜像文件大小都不一样，如全安装占了1.7GB，最小安装（Minimal）则需303MB，微安装（Micro）需要462MB，至于最后一个自定义安装则介于全安装及最小安装两者之间。

VMware表示，Photon是一个为vSphere优化的Container执行平台，可以支持多项Container技术，如Docker、Rocket Container（rkt）和Pivotal Garden Container镜像文件。


Photon可部署于vSphere 5.5、vSphere 6.0、vCloud Air、VMware Fusion和VMware Workstation产品上，除了协助企业将Fusion、VMware Workstaion等测试环境上的应用程序，搬迁到使用vSphere上的正式环境，并通过Lightwave对Container进行访问控制、身份管理等功能。

企业也可以整合Lightwave和Photon，提供多租户目录服务，加强大量Container的管理，例如限定特定Container所能执行的主机，并且限制只有特定权限的使用者可以管理Container。
![](http://static.oschina.net/uploads/img/201512/20112919_RJTD.png)

Photon可部署在vSphere之上，并透过Lightwave对Container进行访问控制、身份管理等功能。

![](http://static.oschina.net/uploads/img/201512/20112920_Lt6d.png)

Photon可以提供4种安装模式：微安装（Micro）、最小安装（Minimal）、全安装及客制化安装，每种模式的映像文件大小都不一。


## 解决ping command not found问题
**VMware 的开源项目harbor使用的基础镜像也包括photon**  
不过在项目过程中，遇到使用photon的容器无法联通host上一个虚拟IP，而用Ubuntu的容器则可以，但是photon上面貌似什么命令也没有，包括ping命令。下面给出解决ping command not found问题的方法：
### photon 1.0版本
查看photon版本
```
cat /etc/photon-release
```
结果：
```
VMware Photon Linux 1.0
PHOTON_BUILD_NUMBER=62c543d
```

使用如下命令解决：
```
tdnf install iputils
```

### photon 2.0及以上版本
查看photon版本
```
cat /etc/photon-release
```
结果：
```
VMware Photon OS 2.0
PHOTON_BUILD_NUMBER=0922243
```

使用如下命令解决：
```
tdnf install pmd-cli
```

可以将这个安装添加到Dockerfile中：
```
FROM vmware/photon:2.0

RUN tdnf erase vim -y \
    && tdnf distro-sync -y || echo \
    && tdnf install -y sudo >> /dev/null \
    && tdnf install -y pmd-cli >> /dev/null \
    && tdnf clean all \
    && groupadd -r -g 10000 harbor && useradd --no-log-init -r -g 10000 -u 10000 harbor \
    && mkdir /harbor/
COPY ./make/dev/adminserver/harbor_adminserver ./make/photon/adminserver/start.sh /harbor/
HEALTHCHECK CMD curl --fail -s http://127.0.0.1:8080/api/ping || exit 1

RUN chmod u+x /harbor/harbor_adminserver /harbor/start.sh
WORKDIR /harbor/
ENTRYPOINT ["/harbor/start.sh"]
```


## telnet命令没有找到

解决方法： telnet(use SSH instead)
>The following Linux troubleshoot tools are neither installed on Photon OS by default nor available in the Photon OS repositories:  
iostat  
telnet (use SSH instead)  
Iprm  
hdparm  
syslog (use journalctl instead)  
ddd  
ksysmoops  
xev  
GUI tools (because Photon OS has no GUI)

## Why is netstat not working?
A. netstat is deprecated, ss or ip (part of iproute2) should be used instead.

For more information, see Use [ip and ss Commands](https://vmware.github.io/photon/assets/files/html/3.0/photon_admin/use-ip-and-ss-commands.html)

## 总结
可以多看看
[Photon OS Frequently Asked Questions](https://github.com/vmware/photon/wiki/Frequently-Asked-Questions)
这里面包含了许多常见问题

## 参考资料：
[harbor 官方github](https://my.oschina.net/JasonZhang/blog/548023) 

[Photon OS – no ping and no ICMP replies? Other quick hints on Photon too.](https://www.paluszek.com/wp/2018/03/27/photon-os-no-ping-and-no-ecmp-replies-other-quick-hints-on-photon-too/)  

[Linux Troubleshooting Tools](https://vmware.github.io/photon/assets/files/html/3.0/photon_troubleshoot/linux-troubleshooting-tools.html)

[Photon OS Frequently Asked Questions](https://github.com/vmware/photon/wiki/Frequently-Asked-Questions)


