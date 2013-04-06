---
layout: post
title: RedDwarf服务器测试环境搭建
categories: [Game, Java]
tags: [Game, Java, sgs]
published: true
comments: true
description: RedDwarf游戏服务器开发环境配置
keywords: RedDwarf, game server, configure
---

分析RD服务器代码时，经常需要跑一跑，以验证分析的结果和猜测。在eclipse下搭了这个环境。简单点，直接上图吧，有图 有真相，说明基本上也能省了。

<!--more-->

入口代码设置
<center>
	<img src="http://pic.yupoo.com/changchang005/CLxhI645/4BJKI.jpg" title="sgs1"/>
</center>


参数设置
<center>
	<img src="http://pic.yupoo.com/changchang005/CLxhmXSm/no0tn.jpg" title="sgs2" />
</center>

代码入口是com.sun.sgs.impl.kernel.Kernel，配置文件主要在app.properties，指向它就可以了，bdb运行环境依赖的native包需要指明一下。如果用的是0.10.2的话，用的bdb的je（java edition）版本，native也免了。可以加个sgs-logging.properties文件来设一下日志level，都可以从SGS_HOME/conf目录下拷的。

app.properties文件内容

```java
	com.sun.sgs.app.name=Hello
	com.sun.sgs.app.listener=test.Hello
	com.sun.sgs.app.root=data
	com.sun.sgs.services=test.MyServiceImpl
	com.sun.sgs.txn.timeout=300000
	com.sun.sgs.node.type=coreServerNode
	com.sun.sgs.server.host=localhost
```

后面两个如果是单节点的模式的话可以不要。具体可以参考[config-properties](http://www.reddwarfserver.org/javadoc/current/server-all/com/sun/sgs/impl/kernel/doc-files/config-properties.html)。
