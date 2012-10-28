---
layout: post
title: "从wordpress到octopress"
date: 2012-10-08 00:28
comments: true
categories: 心情
---

#前记

兜兜转转，终于还是受不了wordpress的繁杂控制后台和二逼编辑器了，下定决心搬到octopress来了。相比wp，op更加清爽，更容易支配，更符合我的口味吧。比较喜欢op的组织结构，每篇文章都是一个文本文件，支持markdown格式，方便编辑和控制，好过面对冷冰冰的数据库。于是有了这个折腾之旅。

<!-- more -->

#octopress安装

octopress官网的文档写的够仔细了，照着倒腾很快就能让博客跑起来了，基本没什么问题。我是在本机搭的博客，采用的是ssh + Rsync。要发布出去的话，可以考虑github。

#文章迁移

搬窝首先要考虑的是以前的家当怎么处理。虽然说开博至今没憋出什么像样的文章，但终究是自己业余时间的一点心血，能搬就搬过来吧。而且，像我这样从wp折腾到op的人应该也不少，靠谱的方案应该也不少。

果不其然，google了一把，发现了不少成功案例。我采用的是[yorkxin](http://blog.yorkxin.org/2011/11/26/import-from-wpcom-to-octopress/)同学的方案，迁移脚本的代码在[这里](https://gist.github.com/1394128)。下面是主要的操作流程：

* 首先先从wp的控制后台导出wp的备份文件，保存为wordpress.xml，放到op的根目录下。
* 将上面gist里的脚本保存到op项目下的_import/wordpressdotcom.rb文件中。
* 在op根目录下执行
```
ruby -r "./_import/wordpressdotcom.rb" -e "Jekyll::WordpressDotCom.process"
```
这样就完成文章的迁移啦。

转移后有的地方格式还是不大对的，先把前面几篇文章放上来吧，后面的慢慢调好了以后再说～

#加入评论

op中没有自带评论功能，如果需要可以通过[DISQUS](http://disqus.com/)来提供。过程也很简单。

首先注册一个DISQUS帐号。剩下的就交给op吧，因为op本来就支持DISQUS。
在_config.yml配置文件找到disqus的配置，填上你的disqus_short_name，再重新生成一下博客文章，发布，然后就可以看到你的文章后面加入了非常漂亮的评论模块啦，是不是很简单呢。

#加入图片

op也没有上传图片的功能。这个也可以寻找一个第三方的图片服务就解决了。以前的博客图片大多是在本机上的，如果要放到网上，还得慢慢倒腾。

#美化

op的样式感觉比以前wp上的要大气许多，现在还没来得及好好倒腾，后面再一步步把把小博美化美化吧。以后迁到github上也很方便，op本身就是一个git管理的工程～:)