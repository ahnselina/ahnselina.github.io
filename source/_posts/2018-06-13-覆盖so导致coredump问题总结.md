---
layout: post
title: Linux下so动态链接库使用总结
categories: Linux
tags: [Linux, so, coredump, 日常积累]

---
本文主要总结在Linux环境下，使用so动态链接库经常遇到的问题，包括使用cp命令覆盖so导致进程coredump之类的问题。  
主要有以下内容：
* Linux下so动态链接库介绍
* ldconfig命令的使用
* so动态库使用的常见问题


---
### 一、Linux下so动态链接库介绍


在介绍动态库前，提一个问题：到底什么是库呢？

库从本质上来说是一种可执行代码的二进制格式，可以被载入内存中执行。库分静态库和动态库两种。

静态库：这类库的名字一般是libxxx.a，xxx为库的名字。利用静态函数库编译成的文件比较大，因为整个函数库的所有数据都会被整合进目标代码中，他的优点就显而易见了，即编译后的执行程序不需要外部的函数库支持，因为所有使用的函数都已经被编译进去了。当然这也会成为他的缺点，因为如果静态函数库改变了，那么你的程序必须重新编译。

动态库：这类库的名字一般是libxxx.M.N.so，同样的xxx为库的名字，M是库的主版本号，N是库的副版本号。当然也可以不要版本号，但名字必须有。相对于静态函数库，动态函数库在编译的时候并没有被编译进目标代码中，你的程序执行到相关函数时才调用该函数库里的相应函数，因此动态函数库所产生的可执行文件比较小。由于函数库没有被整合进你的程序，而是程序运行时动态的申请并调用，所以程序的运行环境中必须提供相应的库。动态函数库的改变并不影响你的程序，所以动态函数库的升级比较方便。linux系统有几个重要的目录存放相应的函数库，如/lib和/usr/lib。

本文主要介绍动态库。

Linux下动态库文件的文件名形如 **libxxx.so**，其中so是 **Shared Object** 的缩写，即可以共享的目标文件。

共享文件（*.so）也称为动态库文件，它包含了代码和数据（这些数据是在连接时候被连接器ld和运行时动态连接器使用的）。动态连接器可能称为ld.so.1，libc.so.1或者 ld-linux.so.1。我的CentOS6.0系统中该文件为：/lib/ld-2.12.so

在链接动态库生成可执行文件时，并不会把动态库的代码复制到执行文件中，而是在执行文件中记录对动态库的引用。
程序执行时，再去加载动态库文件。如果动态库已经加载，则不必重复加载，从而能节省内存空间。  
**程序动态链接的优点**是

1. 减少依赖相同动态库的多个进程同时运行时的内存的占用（不用每一个进程都加载一份动态库
2. 可扩展性在程序不用重启的情况下，动态的加载所需要的动态库，可实现对程序的扩展
3. 程序版本更新与动态链接库的分离  


***Linux下生成和使用动态库的步骤如下：***

1. 编写源文件
2. 使用命令： gcc -fPIC -shared -o libxxx.so xxx.c，将一个或几个源文件编译链接，生成共享库
3. 通过 -L<path> -lxxx 的gcc选项链接生成的libxxx.so。
4. 把libxxx.so放入链接库的标准路径，或指定 LD_LIBRARY_PATH，才能运行链接了libxxx.so的程序。

具体示例请参考：[Linux动态库生成与使用指南](https://www.cnblogs.com/jiqingwu/p/linux_dynamic_lib_create.html)


**查看程序使用的动态库**  
基本上每一个linux 程序都会使用动态库，查看某个程序使用了那些动态库，可以使用ldd命令：  
```shell
# ldd /bin/ls
linux-vdso.so.1 => (0x00007fff597ff000)
libselinux.so.1 => /lib64/libselinux.so.1 (0x00000036c2e00000)
librt.so.1 => /lib64/librt.so.1 (0x00000036c2200000)
libcap.so.2 => /lib64/libcap.so.2 (0x00000036c4a00000)
libacl.so.1 => /lib64/libacl.so.1 (0x00000036d0600000)
libc.so.6 => /lib64/libc.so.6 (0x00000036c1200000)
libdl.so.2 => /lib64/libdl.so.2 (0x00000036c1600000)
/lib64/ld-linux-x86-64.so.2 (0x00000036c0e00000)
libpthread.so.0 => /lib64/libpthread.so.0 (0x00000036c1a00000)
libattr.so.1 => /lib64/libattr.so.1 (0x00000036cf600000)
```


### 二、ldconfig命令的使用
ldconfig命令是在Linux环境下使用so动态链接库时，经常会用到的命令，是Linux下动态链接库的管理命令，**该命令位于/sbin目录下**。  

ldconfig命令的用途主要是在默认搜寻目录/lib和/usr/lib以及动态库配置文件/etc/ld.so.conf内所列的目录下，搜索出可共享的动态链接库（格式如lib*.so*）,进而创建出动态装入程序(ld.so)所需的连接和缓存文件。缓存文件默认为/etc/ld.so.cache，此文件保存已排好序的动态链接库名字列表，为了让动态链接库为系统所共享，需运行动态链接库的管理命令ldconfig，此执行程序存放在/sbin目录下。

>ldconfig通常在系统启动时运行，而当用户安装了一个新的动态链接库时，就需要手工运行这个命令。

**使用ldconfig几个需要注意的地方：**

1. 往/lib和/usr/lib里面加东西，是不用修改/etc/ld.so.conf的，但是完了之后要调一下ldconfig，不然这个library会找不到。
2. 想往上面两个目录以外加东西的时候，一定要修改/etc/ld.so.conf，然后再调用ldconfig，不然也会找不到。
3. 比如安装了一个mysql到/usr/local/mysql，mysql有一大堆library在/usr/local/mysql/lib下面，这时就需要在/etc/ld.so.conf下面加一行/usr/local/mysql/lib，保存过后ldconfig一下，新的library才能在程序运行时被找到。
4. 如果想在这两个目录以外放lib，但是又不想在/etc/ld.so.conf中加东西（或者是没有权限加东西）。那也可以，就是export一个全局变量LD_LIBRARY_PATH，然后运行程序的时候就会去这个目录中找library。一般来讲这只是一种临时的解决方案，在没有权限或临时需要的时候使用,因为这样的export 只对当前shell有效，当另开一个shell时候，又要重新设置，当然可以把export LD_LIBRARY_PATH=xxx 语句写到 ~/.bashrc中，这样就对当前用户有效了，写到/etc/bashrc中就对所有用户有效了。


**程序执行时的搜索顺序**  
程序执行时按照下列顺序依次装载或者查找共享对象:  
0）最优先的是，如果在编译时通过-rpath选项指定了路径，便会优先搜索这个路径  
1）由环境变量 LD_LIBRARY_PATH指定的路径  
2）由路径缓存文件/etc/ld.so.cache指定的路径  
3）默认共享目录 /lib和/usr/lib  
其中LD_LIBRARY_PATH是一个环境变量，当指定某个程序的LD_LIBRARY_PATH时
动态链接器在查找共享库的时候，**会首先从指定的路径开始查找**


### 三、so动态库使用的常见问题
介绍完so动态链接库的常见使用之后，下面介绍两个常见的问题：  
一个是cp命令拷贝一个新的so去覆盖旧的so时，如果有进程或者程序正常使用这个so，那么该操作极有可能导致该进程coredump或者程序崩溃；  
另一个问题是多个进程或程序都在使用同一个so，但是这个so的路径和版本均不同，那么使用ldconfig命令可能导致另一个进程或者程序出错。

#### 1. 覆盖so导致coredump问题总结
出现问题的场景是升级，在升级流程的脚本中需要升级各个业务进程使用的so，但是有一个so文件是两个业务进程都在同时使用。比如有业务进程A、B、C，升级的过程是:A->B->C。其中有so是A和B都依赖，在升级A的过程中，先停掉A进程，升级其需要的so，这个时候升级so，使用的命令是cp，升级完so后，升级A进程使用的二进制，然后拉起A进程。在这一系列的过程中发现B进程coredump了，主要是没有考虑到B进程也在使用那个so。

**cp与mv/rm的区别：**

cp from to，则被覆盖文件 to的inode依旧不变（属性也不变），内容变为from的；

mv from to，则to的inode变为from的，相应的，to的属性也成了from的；rm类似；

关于为什么会coredump可参考：[关于so文件cp覆盖导致调用者core的研究](https://www.cnblogs.com/zhaoyl/p/4964811.html)

**解决方法**

>方法一：  
先删除旧的so，然后再把新的so拷贝过去，即：  
rm oldlib.so 然后 cp newlib.so oldlib.so
  
>方法二：  
mv oldlib.so oldlib.so_bak 然后 cp newlib.so oldlib.so


#### 2. 一次执行ldconfig导致别的模块进程挂掉的经历



**问题原因是：**
与我们服务共同部署在同一个Linux服务器的其他服务也使用了zk服务，需要用到zk的动态链接库，我们的业务进程也需要用到zk的动态链接库。本来最初相安无事，但是在执行一个脚本之后，发现他们的服务挂了，经定位发现是因为so使用有问题，用到了我们服务进程的路径下的zk的动态链接库。在那个shell脚本中，直接用了“ldconfig + 路径”的方式搜索指定路径的so，随后导致他们的服务链接到我们的zk动态链接库了，而这动态链接库是有区别的，最终导致他们的服务挂掉。

**解决方法**

方法一：检查使用ldconfig的地方，在多种服务共同使用的服务器上，不能直接用“ldconfig + 路径”的方式随意添加一些常用的so路径（诸如zk这样常用的服务），诸如多种服务共同部署的时候，要注意避免这种情况；如果要调用脚本使用，可以通过export LD_LIBRARY_PATH的方式临时添加。

方法二（推荐）：在编译的时候通过gcc -rpath 就指定动态库路径，这样就可以避免被其他路径下的不同的版本的so干扰。可以参考：[gcc -rpath 指定动态库路径](https://blog.csdn.net/v6543210/article/details/44809405)


---
### 参考资料
[有关Linux下库的概念、生成和升级和使用等](https://blog.csdn.net/poetic_vienna/article/details/51249660)  
[linux下so动态库一些不为人知的秘密（上）](http://blog.chinaunix.net/uid-27105712-id-3313293.html)  
[linux下so动态库一些不为人知的秘密（中）](http://blog.chinaunix.net/uid-27105712-id-3313327.html)  
[Linux共享库.so文件的命名和动态链接](https://blog.csdn.net/aganlengzi/article/details/44088239)  
[Linux动态库生成与使用指南](https://www.cnblogs.com/jiqingwu/p/linux_dynamic_lib_create.html)  
[ldconfig命令](http://man.linuxde.net/ldconfig)  
[Linux共享库(so)动态加载和升级](https://www.cnblogs.com/cnland/archive/2013/03/19/2969337.html)  
[gcc -rpath 指定动态库路径](https://blog.csdn.net/v6543210/article/details/44809405)  
[Linux系统中“动态库”和“静态库”那点事儿](http://blog.jobbole.com/107977/)  
[关于so文件cp覆盖导致调用者core的研究](https://www.cnblogs.com/zhaoyl/p/4964811.html)