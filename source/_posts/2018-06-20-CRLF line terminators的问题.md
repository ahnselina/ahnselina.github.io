---
layout: post
title: CRLF line terminators的问题
categories: Linux
tags: [CRLF, 日常积累, 问题定位]

---
在定位一个问题时，发现问题是由于配置文件格式不对导致的。本文简单介绍下CRLF line terminators的问题的解决方法。
CRLF line terminators的问题导致的问题：
* shell脚本报错：command not found
* 如果是配置文件，由于格式是windows格式的，可能会导致解析配置参数错误

---
### CRLF line terminators的问题的根源

Linux和Windows文本文件的行结束标志不同。在Linux中，文本文件用"/n"表示回车换行，而Windows用"/r/n"表示回车换行。有时候在Windows编写shell脚本时需要注意这个，否则shell脚本会报"No such file or directory"或"command not found line x"之类的错误，如果不知晓前因后果，肯定会被这个折腾得相当郁闷。  
**区别方法** ： 用file命令或者“cat -v”命令查看文件，显示如下面示例的，则表示该文件是使用windows格式的换行符。
```
[root@DB-Server myscript]# file test.sh 
test.sh: ASCII text, with CRLF line terminators
[root@DB-Server myscript]# cat -v test.sh 
. /home/oracle/.bash_profile^M
echo ' '^M
date^M
echo ' '^M
^M
sqlplus test/test @/home/oracle/scripts/test.sql^M
^M
echo ' '^M
date^M
echo ' '^M
```


### 解决方法
三种解决方法：
* 直接在Linux环境上使用vi编辑器编写shell脚本，避免在windows下开发shell脚本
* 如果是在windows上开发的shell脚本，可以在Linux环境上创建shell脚本，然后从Windows的脚本里面拷贝内容过来；
* 最简单方便的是使用dos2unix将DOS格式文本文件转换成Unix格式或Linux格式。

---
### 参考资料
[处理文件CRLF line terminators的问题](http://www.hao32.com/unix-linux/565.html)  
[CRLF line terminators导致shell脚本报错：command not found](https://www.cnblogs.com/kerrycode/archive/2015/12/22/5065356.html)