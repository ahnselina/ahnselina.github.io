---
layout: post
title: Python的缩进错误:unindent does not match any outer indentation level
categories: Python
tags: [bug, python]

---

### 解决方法

Python程序运行出现语法错误：IndentationError: unindent does not match any outer indentation level

运行环境是win7 x64 sublime text2，

这个错误最开始以为是缩进问题，看了很久，最后发现其实是由于有的地方使用了4个空格，有的地方使用了tab键。

代码区直接全选就会看到**有的地方是四个点有个地方是一个横线**，改一致了就好了

因此，以后在**使用tab键和空格键的时候需要注意统一**

***

