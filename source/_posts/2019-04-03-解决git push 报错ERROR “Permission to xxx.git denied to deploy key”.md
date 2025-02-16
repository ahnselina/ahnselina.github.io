---
title: 解决git push 报错ERROR “Permission to xxx.git denied to deploy key”
date: 2019-04-03 23:30:30
categories:
- git
tags: [git使用, git push error, git报错]
---


![](https://z3.ax1x.com/2021/05/04/gmxCTg.jpg)
<!-- more -->
   本文记录解决git push 报错ERROR “Permission to xxx.git denied to deploy key”的过程和方法。

## 遇到的问题
```
 git push -u origin master
ERROR: Permission to ahnselina/data-structure.git denied to deploy key
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

Please make sure you have the correct access rights
```
![](https://z3.ax1x.com/2021/05/04/gmxEpn.jpg)
还遇到过：
![](https://z3.ax1x.com/2021/05/04/gmxZ60.jpg)


## 查阅资料和解决办法
### deploy key
deploy key 每个 Repository 都有一个，主要用于 push 代码时使用。每个 Repo 的 deploy key 都是单独设置的，不能多个 Repo 使用相同的 deploy key。如果 Repository 没有添加 deploy key 时直接 push 代码，会出现权限错误：
```
$ git push origin master
Permission denied (publickey).
fatal: Could not write to remote repository.
Please make sure you have the correct access rights
and the repository exists.
```
生成 deploy key

deploy key 和 ssh key 本质是一样的，都可以用 ssh-keygen 命令生成。
```
$ ssh-keygen -f ~/.ssh/deploy_key_repo1
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/kang/.ssh/test.
Your public key has been saved in /Users/kang/.ssh/test.pub.
The key fingerprint is:
SHA256:Z//9gcDS40LheYjxpZOuDgbG5wRQ9X7O7BTXfo134es kang@ruikye
The key's randomart image is:
+---[RSA 2048]----+
| .....           |
|  .   .          |
|   .   o . .     |
|  . . . = X .    |
|   + o oS&oB . . |
|  . =   BoB.+ o.o|
|     +   B ..o.++|
|    . . + .  ..o+|
|      .o .    oE+|
+----[SHA256]-----+
```
>注意： 因为在生成 ssh-key 使用了默认 id_rsa.pub 文件存储，所以 deploy key 要放在其他的文件中。

由于 deploy key 不是放在默认文件，所以 deploy key 不能直接使用，需要添加到SSL的认证列表中：
```
# 添加默认的 id_rsa
$ ssh-add ~/.ssh/id_rsa
# 添加 deploy key
$ ssh-add ~/.ssh/deploy_key_repo1
# 查看所有 add 的 keys
$ ssh-add -l
2048 SHA256:kqQTYy......dsp+8 /Users/your/.ssh/id_rsa (RSA)
2048 SHA256:0b/vdS......62tok /Users/your/.ssh/deploy_key_repo1 (RSA)
```
执行ssh-add时出现Could not open a connection to your authentication agent

![](https://z3.ax1x.com/2021/05/04/gmxeXV.jpg)
若执行ssh-add /path/to/xxx.pem是出现这个错误:Could not open a connection to your authentication agent，则先执行如下命令即可：
```
　　ssh-agent bash
```
更多关于ssh-agent的细节，可以用 man ssh-agent 来查看


添加 deploy key
```
# 复制 deploy key
$ cat ~/.ssh/deploy_key_repo1.pub | pbcopy
```
打开 Repository 的设置页，在 Deploy keys -> Add deploy key 添加公钥并勾选 Allow write access。

## 解决办法
试了多种方法未果，最终发现用如下方法有效：
第一步：将ssh改为https，打开对应项目目录的 .git/config文件
```
vim .git/config
```
改之后如下图：
![](https://z3.ax1x.com/2021/05/04/gmxnmT.jpg)

第二步：修改之后会让输入github的用户名密码，输入之后可以采用如下命令来缓存，以免每次都输入
```
git config --global credential.helper wincred
```


第三步：解决遇到的新错误
修改之后继续git push，遇到如下的错误
![](https://z3.ax1x.com/2021/05/04/gmxu0U.md.jpg)
```
git push
Fatal: HttpRequestException encountered.
fatal: Out of memory, malloc failed (tried to allocate 2813163520 bytes)
error: failed to push some refs to 'https://github.com/ahnselina/data-structure.git'
```
然后继续解决该问题：
在对应项目目录的 .git/config文件中加入如下内容
```
[http]
  postbuffer = 1024m
```
然后git push发现问题解决:)


## 参考资料：
[GIT: fatal: Out of memory, malloc failed (tried to allocate 889192448 bytes)](https://stackoverflow.com/questions/41120920/git-fatal-out-of-memory-malloc-failed-tried-to-allocate-889192448-bytes)  

[Error: Permission denied (publickey)](https://help.github.com/en/articles/error-permission-denied-publickey)  

[git push Out of memory, malloc failed](https://stackoverflow.com/questions/8855317/git-push-out-of-memory-malloc-failed)  

[Caching your GitHub password in Git](https://help.github.com/en/articles/caching-your-github-password-in-git)


