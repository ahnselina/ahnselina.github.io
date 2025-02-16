---
layout: post
title: 实际项目中对Linux的profile.d目录的使用
categories: Linux 
tags: [环境变量, profile profile.d, 日常积累]

---
在Linux环境下，/etc/profile文件中/etc/profile.d目录下均可以设置环境变量，本文主要介绍/etc/profile文件中/etc/profile.d目录；并举一个示例，展示在实际项目中如何使用的/etc/profile.d目录。
主要有以下内容：
* /etc/profile 文件
* /etc/profile.d 目录中的脚本文件
* /etc/profile和/etc/profile.d 的区别和用法
* 项目中使用的实例


---
## 一、/etc/profile 文件
当一个用户登录Linux系统或使用su -命令切换到另一个用户时，也就是Login shell 启动时，首先要确保执行的启动脚本就是 /etc/profile 。
关于linux下 /etc/profile、~/.bash_profile ~/.profile的执行过程，请参考：[linux下 /etc/profile、~/.bash_profile ~/.profile的执行过程](https://blog.csdn.net/caiwenfeng_for_23/article/details/44242961)  


**注意：只有Login shell 启动时才会运行 /etc/profile 这个脚本，而Non-login shell 不会调用这个脚本。**  
如何区分Login shell 与Non-login shell，可以在shell下运行echo $0:
```shell
# echo $0
```
假设输出为该shell名字，加上一个**'-'**前缀。则说明该shell为login shell。   
详细介绍请参考：[/etc/profile与/etc/bashrc、交互式与非交互式、login与non-login shell的差别](https://www.cnblogs.com/wzzkaifa/p/7169640.html)

一些重要的变量就是在这个脚本文件中设置的，含义如下：

PATH： 预设可执行文件或命令的搜索路径。

USER: 用户登录时使用的用户名。

LOGNAME: 其值为$USER。

HOSTNAME: 所使用的主机名。

MAIL： 存放用户电子邮件的邮箱（实际上是一个ASCII码文件）。

HISTSIZE： 历史记录的行数。

INPUTRC: 存放的是针对键盘热键的信息（也是一个ASCII码文件）。

要注意的是**在/etc/profile 文件中设置的变量是全局变量**。


## 二、/etc/profile.d 目录中的脚本文件
在/etc/profile.d 目录中存放的是一些应用程序所需的启动脚本，其中包括了颜色、语言、less、vim及which等命令的一些附加设置。

这些脚本文件之所以能够 被自动执行，是因为**在/etc/profile 中使用一个for循环语句来调用这些脚本**。而这些脚本文件是用来设置一些变量和运行一些初始化过程的。  
/etc/profile中循环调用/etc/profile.d/目录下脚本的代码片段：
```shell
if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi
```

## 三、/etc/profile和/etc/profile.d 的区别和用法
**区别**：  
1. 两个文件都是设置环境变量文件的，/etc/profile是永久性的环境变量,是全局变量，/etc/profile.d/设置所有用户生效  
2. /etc/profile.d/比/etc/profile好维护，不想要什么变量直接删除/etc/profile.d/下对应的shell脚本即可，不用像/etc/profile需要改动此文件  

**用法**  
Linux每个用户都要指定各自的环境变量，其中包括可执行的path路径，这些路径决定了每个用户在执行时的命令工具。 

一般情况下，可以在每个用户的环境变量里设定各自的PATH变量值，然后再执行export PATH使其生效，但如果用户比较多，安装命令工具也原来越多，且除了本身用户可以执行这些工具，root用户或其他用户也可以执行命令，这时在每个用户环境变量里添加就比较复杂了。

所以可以用另外一种方法：

可以在/etc/profile.d/ 目录下创建一个 path.sh 脚本，脚本内容如下：

```shell
# cat /etc/profile.d/path.sh

export PATH="/bin:/usr/bin:/sbin:/usr/sbin:/usr/local/sbin"
```

可以将需要各个用户需要执行的命令路径都写在该命令中，这样每次操作系统启动后，会自动执行path.sh脚本，将所有的环境变量生效，让各个用户都可以直接执行各自的命令。

以后安装了新软件后，只需要将软件相关路径加入 /etc/profile.d/path.sh 脚本中，不需要再多个地方重复添加，使脚本生效后，各个用户都可以使用了。

后面实际项目的示例也是采用的这种方法。

## 四、项目中使用的实例
**背景** ：一个整体的服务内部分为A、B、C三个微服务，其中B和C都需要调用微服务A提供的接口，其接口以so的形式提供。那就需要我们在安装好微服务A时，就能告诉B和C，我们提供的so路径在哪。这种情况该如何做呢？


**实际做法**：
参考上面说的/etc/profile.d/的用法，比如我们提供的so路径在/opt/new_test/lib/目录下，还有可执行工具在/opt/new_test/tool下；那么在我们安装部署微服务A的时候就需要指定这两个路径。

安装的脚本,比如叫install_A.sh就需要包含如下类似的内容来设置环境变量：
```shell
#!/bin/bash

#do something install A
...

env_file="/etc/profile.d/set_A_env.sh"

echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/new_test/lib/' > $env_file

echo 'export PATH=$PATH:/opt/new_test/tool' >> $env_file

exit 0
```
通过如上设定，微服务A安装完成之后，环境变量已经设置OK，等微服务B和C安装起来之后就能从环境变量中找到我们指定的so的路径。

卸载时的脚本中比如叫uninstall_A.sh中需要包含如下类似的内容来清除环境变量：
```shell
#!/bin/bash

#do something for uninstall...
...
env_file="/etc/profile.d/set_A_env.sh"

rm -f ${env_file}

source /etc/profile

exit 0
```



---
## 参考资料
[etc/profile 文件和/etc/profile.d](https://blog.csdn.net/u011277123/article/details/72864826)  
[Linux 环境下/etc/profile和/etc/profile.d 的区别和用法](https://blog.csdn.net/efine_dxq/article/details/54601117)  
[linux 配置全用户的环境变量，profile.d文件夹的作用](https://blog.csdn.net/justweb/article/details/61221567)  
[linux下 /etc/profile、~/.bash_profile ~/.profile的执行过程](https://blog.csdn.net/caiwenfeng_for_23/article/details/44242961)  
[/etc/profile与/etc/bashrc、交互式与非交互式、login与non-login shell的差别](https://www.cnblogs.com/wzzkaifa/p/7169640.html)