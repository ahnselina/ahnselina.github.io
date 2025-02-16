---
title: 博客从Jekyll迁移Hexo  
date: 2019-01-06 22:35:20  
tags: [随想, 总结, 问题定位]
---
### 遇到的问题

搭建教程参考：[GitHub+Hexo 搭建个人网站详细教程](https://zhuanlan.zhihu.com/p/26625249)   

#### npm install g hexo 没有反应
执行 npm config set registry "https://registry.npm.taobao.org" 将npm包源指向淘宝，就不需要翻墙安包了：[npm install g hexo 总是失败](https://www.oschina.net/question/2443995_2155065) 

可供参考的资料：
[使用Hexo+Github一步步搭建属于自己的博客（进阶）](https://www.cnblogs.com/fengxiongZz/p/7707568.html)

[Hexo+Next主题 文章添加阅读次数，访问量等](https://blog.csdn.net/xr469786706/article/details/78166227)

[hexo+github 搭建个人博客及美化](https://lruihao.cn/hexo%20+%20github%20%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2.html)

[设置 SSH 使用 hexo deploy 时免输用户名密码](https://blog.csdn.net/hhgggggg/article/details/77853665)

[细数用hexo搭建github博客踩过的坑（windows版）](https://yeyouluo.github.io/2017/10/16/%E7%BB%86%E6%95%B0%E7%94%A8hexo%E6%90%AD%E5%BB%BAgithub%E5%8D%9A%E5%AE%A2%E8%B8%A9%E8%BF%87%E7%9A%84%E5%9D%91/)

[hexo d 出错](https://lruihao.cn/hexo-d-error.html)

[Busuanzi 统计浏览失效解决方法点这](https://blog.csdn.net/ddydavie/article/details/83020549)

[Hexo使用攻略-添加分类及标签](https://linlif.github.io/2017/05/27/Hexo%E4%BD%BF%E7%94%A8%E6%94%BB%E7%95%A5-%E6%B7%BB%E5%8A%A0%E5%88%86%E7%B1%BB%E5%8F%8A%E6%A0%87%E7%AD%BE/)

[添加评论方法](https://blog.csdn.net/qq_32454537/article/details/79482879)

本来想用来必力，但是网页打开很慢，遂放弃。
[为你的Hexo加上评论系统-Valine](https://www.bluelzy.com/articles/use_valine_for_your_blog.html)

[Hexo Next下添加版权声明模块](http://stevenshi.me/2017/05/26/hexo-add-copyright/)


下面文章含有“提交百度谷歌站点验证出错”的解决方法：
[如何避免 Hexo 编译 HTML 文件](https://www.huangzz.xyz/how-to-avoid-hexo-compiling-html-files.html)
由于hexo会在生成编译文件的过程中，修改html文件内容，导致百度验证失败，因此，不建议再踩一遍这个坑。可参考下文：
[Hexo：Github部署站点的SEO优化教程](http://www.dadroid.cn/posts/undefined/)
[Hexo博客Next主题SEO优化方法](https://hoxis.github.io/Hexo+Next%20SEO%E4%BC%98%E5%8C%96.html)

另外一个错误是不要私自去往自己的博客仓库里面添加文件，导致hexo本地的库和远程库内容不一致，这样在新推送日志或其他东西时，会推送失败。


[显示每篇文章的更新时间](https://wuchenxu.com/2015/12/13/Static-Blog-hexo-github-7-display-updated-date/)



### 修改头像实现旋转

更换头像，打开站点配置文件,找到avatar字段，可以使用网络路径，也可以将头像存放在source/images/中。如果头像是椭圆的，是因为图片不是一个正方形的图片，找到一个宽高像素一样的的图片即可。

avatar: /images/head.jpg
打开\themes\next\source\css\_common\components\sidebar\sidebar-author.styl，在里面添加如下代码：

```http
.site-author-image {
  display: block;
  margin: 0 auto;
  padding: $site-author-image-padding;
  max-width: $site-author-image-width;
  height: $site-author-image-height;
  border: $site-author-image-border-width solid $site-author-image-border-color;
  /* 头像圆形 */
  border-radius: 80px;
  -webkit-border-radius: 80px;
  -moz-border-radius: 80px;
  box-shadow: inset 0 -1px 0 #333sf;
  /* 设置循环动画 [animation: (play)动画名称 (2s)动画播放时长单位秒或微秒 (ase-out)动画播放的速度曲线为以低速结束
    (1s)等待1秒然后开始动画 (1)动画播放次数(infinite为循环播放) ]*/

  /* 鼠标经过头像旋转360度 */
  -webkit-transition: -webkit-transform 1.0s ease-out;
  -moz-transition: -moz-transform 1.0s ease-out;
  transition: transform 1.0s ease-out;
}
img:hover {
  /* 鼠标经过停止头像旋转
  -webkit-animation-play-state:paused;
  animation-play-state:paused;*/
  /* 鼠标经过头像旋转360度 */
  -webkit-transform: rotateZ(360deg);
  -moz-transform: rotateZ(360deg);
  transform: rotateZ(360deg);
}
/* Z 轴旋转动画 */
@-webkit-keyframes play {
  0% {
    -webkit-transform: rotateZ(0deg);
  }
  100% {
    -webkit-transform: rotateZ(-360deg);
  }
}
@-moz-keyframes play {
  0% {
    -moz-transform: rotateZ(0deg);
  }
  100% {
    -moz-transform: rotateZ(-360deg);
  }
}
@keyframes play {
  0% {
    transform: rotateZ(0deg);
  }
  100% {
    transform: rotateZ(-360deg);
  }
}
```
[参考该文](http://www.shaoyance.com/2018/01/26/Hexo%E5%8D%9A%E5%AE%A2Next%E4%B8%BB%E9%A2%98%E4%BC%98%E5%8C%96%E6%80%BB%E7%BB%93/)

### hexo d 卡住半天没反应
首先不要作死，私自去往自己的博客仓库里面添加文件，这样会导致hexo本地的库和远程库内容不一致，这样再hexo d新推送日志或其他东西时，会失败。
如果没有作死，可以参考下文：
[hexo d 卡住问题](http://blog.2hao.cc/2018/07/29/question/)

如果使用hexo s都能正常预览网页,这种情况可能是网络问题，ping一下github.com如果超时，那就要用到下面的方法了。  

路径 C:\Windows\System32\drivers\etc\hosts

用记事本打开，在末尾添加内容：
```
192.30.253.113    github.com
192.30.252.131 github.com
185.31.16.185 github.global.ssl.fastly.net
74.125.237.1 dl-ssl.google.com
173.194.127.200 groups.google.com
192.30.252.131 github.com
185.31.16.185 github.global.ssl.fastly.net
74.125.128.95 ajax.googleapis.com
```
保存（以管理员身份），重新运行 cmd 再ping，可以通。





其他问题待续...


### 参考资料

[GitHub+Hexo 搭建个人网站详细教程](https://zhuanlan.zhihu.com/p/26625249) 
[npm install g hexo 总是失败](https://www.oschina.net/question/2443995_2155065)   
[迁移](https://hexo.io/zh-cn/docs/migration)  
[Github绑定域名](https://www.jianshu.com/p/0886c99f826a)  
[GitHub Pages 绑定来自阿里云的域名](https://blog.csdn.net/qq_29232943/article/details/52786603)