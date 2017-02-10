---
title: 自定义域名解析到Github Page
date: 2017-02-09 17:25:53
tags: hexo
categories: hexo
---

今天刚刚搭建了Github page，但是始终没有一个自己的域名，感觉总是没那么爽快。于是马上在万网上注册一个顶级域名。根据github page的[官方文档](https://help.github.com/articles/using-a-custom-domain-with-github-pages/)我们可以很方便的进行解析。

# 一，添加CNAME文件到git master分支中
在source目录下新建CNAME文件，内容为你的域名，然后执行hexo g 和 hexo d
```
karosli.com
```

# 二，添加a记录
根据[文档](https://help.github.com/articles/setting-up-an-apex-domain/)添加a记录，这里我是用dnspod做域名解析的，前提是需要把万网的dns解析服务器设置成dnspod的服务器。

![粘贴图片.png](http://upload-images.jianshu.io/upload_images/1423834-8611f456892da1bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 最后
现在的话不管是[karosli.com](http://karosli.com), [www.karosli.com](http://www.karosli.com) 还是 [karosli.github.io](https://karosli.github.io) 都可以访问自己的博客了。

# 另外
下面这个博客对于next主题的使用，做了详细的介绍，大家可以[参考](http://theme-next.iissnan.com/getting-started.html)下。