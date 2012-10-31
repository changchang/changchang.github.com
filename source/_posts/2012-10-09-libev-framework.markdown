---
layout: post
title: libev源码分析 -- 整体设计
categories: C/C++
tags: [libev, C/C++, 源码分析]
published: true
comments: true
description: libev源代码分析，分析libev整体框架的设计思想
keywords: libev, 源代码分析, 设计思想
---

[libev](http://software.schmorp.de/pkg/libev.html)是Marc Lehmann用C写的高性能事件循环库。通过libev，可以灵活地把各种事件组织管理起来，如：时钟、io、信号等。libev在业界内也是广受好评，不少项目都采用它来做底层的事件循环。node.js也是其中之一。 学习和分析libev库，有助于理解node.js底层的工作原理，同时也可以学习和借鉴libev的设计思想。本文是最近在学习libev源码的一些心得总结吧。
<!-- more -->

#libev示例

先上一个例子，看看libev是怎么使用的吧。

```c
	// a single header file is required
	#include <ev.h>

	#include <stdio.h> // for puts

	// every watcher type has its own typedef'd struct
	// with the name ev_TYPE
	ev_io stdin_watcher;
	ev_timer timeout_watcher;

	// all watcher callbacks have a similar signature
	// this callback is called when data is readable on stdin
	static void
	stdin_cb (EV_P_ ev_io *w, int revents)
	{
		puts ("stdin ready");
		// for one-shot events, one must manually stop the watcher
		// with its corresponding stop function.
		ev_io_stop (EV_A_ w);

		// this causes all nested ev_run's to stop iterating
		ev_break (EV_A_ EVBREAK_ALL);
	}

	// another callback, this time for a time-out
	static void
	timeout_cb (EV_P_ ev_timer *w, int revents)
	{
		puts ("timeout");
		// this causes the innermost ev_run to stop iterating
		ev_break (EV_A_ EVBREAK_ONE);
	}

	int
	main (void)
	{
		// use the default event loop unless you have special needs
		struct ev_loop *loop = EV_DEFAULT;

		// initialise an io watcher, then start it
		// this one will watch for stdin to become readable
		ev_io_init (&stdin_watcher, stdin_cb, /*STDIN_FILENO*/ 0, EV_READ);
		ev_io_start (loop, &stdin_watcher);

		// initialise a timer watcher, then start it
		// simple non-repeating 5.5 second timeout
		ev_timer_init (&timeout_watcher, timeout_cb, 5.5, 0.);
		ev_timer_start (loop, &timeout_watcher);

		// now wait for events to arrive
		ev_run (loop, 0);

		// break was called, so exit
		return 0;
	}
```

这是libev官网文档的例子，其中libev的使用步骤还是比较清晰的。从main开始入手，可以发现代码中主要做了这么几件事情：

* 获取ev_loop实例。ev_loop，从名字上可以看出，它代表了一个事件循环，也是我们后面代码的主要组织者。

* 创建和初始化watcher。libev中定义了一系列的watcher，每类watcher负责一类特定的事件。一般可以通过ev_TYPE_init函数来创建一个watcher实例（TYPE是某一种watcher类型，如：io, timer等）。例子中分别创建了io和timer两个watcher，并绑定了相应的回调函数。当感兴趣的事件发生后，对应的回调函数将会被调用。

* 将watcher注册到ev_loop中。一般可以通过ev_TYPE_start函数来完成。注册成功后，watcher便和loop关联起来了，当loop中检测到感兴趣的事件发生，便会通知相关的watcher。

* 启动事件循环。 即后面的ev_run函数。事件循环启动后，当前线程/进程将会被阻塞，直到循环被终止。

在上面的例子中，在两个回调函数中的ev_break函数就是终止循环的地方。当5.5秒超时或是标准输入有输入事件，则会进入到相应的回调函数，然后会终止事件循环，退出程序。

#libev工作原理

总的来看，libev其实是实现了[Reactor模式](http://delivery.acm.org/10.1145/230000/226255/p65-schmidt.pdf?ip=112.10.101.170&acc=ACTIVE%20SERVICE&CFID=175798313&CFTOKEN=73864269&__acm__=1350399757_644772085f9a19ff9d2fdb35159272bf)。当中主要包含了这么几个角色：watcher， ev_loop和ev_run。

##watcher 
watcher是Reactor中的Event Handler。一方面，它向事件循环提供了统一的调用接口（按类型区分）;另一方面，它是外部代码的注入口，维护着具体的watcher信息，如：绑定的回调函数，watcher的优先级，是否激活等。

在ev.h中我们可以看到各种watcher的定义，如：ev_io, ev_timer等。其中，watcher的公共属性定义如下：

```c
	/* shared by all watchers */
	#define EV_WATCHER(type)			\
		int active; /* private */			\
		int pending; /* private */			\
		EV_DECL_PRIORITY /* private  int priority; */		\
		EV_COMMON /* rw  void *data; */				\
		EV_CB_DECLARE (type) /* private */
```
其中的宏定义如下：

```c
	# define EV_DECL_PRIORITY int priority;
	# define EV_COMMON void *data;
	# define EV_CB_DECLARE(type) void (*cb)(EV_P_ struct type *w, int revents);
```
* active: 表示当前watcher是否被激活。ev_TYPE_start调用后置位，ev_TYPE_stop调用后复位；

* pending: 表示当前watcher有事件就绪，等待处理。pending的值其实就是当前watcher在pendings队列中的下标；

* priority: 是当前watcher的优先级；

* data: 附加数据指针，用来在watcher中携带额外所需的数据；

* cb：是事件触发后的回调函数定义。

具体的watcher再在此基础上添加额外的属性。
开发者可以根据需要，选择特定类型的watcher，创建实例并进行初始化，然后将实例注册到loop中即可。libev中定义了[若干种类型的watcher](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#WATCHER_TYPES)，每类watcher负责解决某一特定领域的问题（如：io, timer, signal等），可以在ev.h中看到这些watcher的定义。

##ev_loop

ev_loop则是一个Reactor的角色，是事件循环的上下文环境，就像一根竹签，把前面的watcher实例像糖葫芦一样串起来。

###ev_loop的定义

ev_loop的定义在ev.c中，具体如下：

```c
	struct ev_loop
	{
		ev_tstamp ev_rt_now;
		#define ev_rt_now ((loop)->ev_rt_now)
		#define VAR(name,decl) decl;
			#include "ev_vars.h"
		#undef VAR
	};
```

ev_vars.h中定义了ev_loop的各种属性。在ev_wrap.h中则定义了与loop相关的各种宏，代码中大多都是以宏的形式进行操作。

###watcher的管理
在ev_loop中，watcher按各自的类型进行分类存储。如：io的loop->anfds，timer的loop->timers。ev_TYPE_start在激活watcher后，便将它加入到相应的存储结构中（具体的实现在后面介绍watcher的文章再分析）。

在事件循环中，有事件就绪的watcher会被挑拣出来，保存到ev_loop中。这些就绪的watcher主要由loop->pendings和loop->pendingcnt来维护（如下图所示）。这两个东西都是二维数组，第一维都是优先级。pendings中存的是ANPENDING实例，后者的做要作用就是维护就绪的watcher指针; 而pendingcnt中存的则是对应优先级上的pendings元素的数量。在每个就绪的watcher上也会有一个pending字段记录它在pendings列表中的下标，这样就可以通过watcher很方便的找到它在pendings列表中的位置了，这对删除操作很有帮助。

在一轮事件循环结束后，则会根据优先级，依次触发就绪的watcher。

<center>
	<img src="http://i.6.cn/cvbnm/f9/d3/73/334a86e7c2d386ed8c3e3174f3a543a3.png" alt="pendings结构图"/>
</center>

##bool ev_run(loop, flag)
ev_run函数是执行事件循环的引擎，即Reactor模式中的select方法。通过向ev_run函数传递一个ev_loop实例，便可以开启一个事件循环。ev_run实际上是一个巨大的do-while循环，期间会检查loop中注册的各种watcher的事件。如果有事件就绪，则触发相应的watcher。这个循环会一直持续到ev_break被调用或者无active的watcher为止。当然，也可以通过传递EVRUN_NOWAIT或EVRUN_ONCE等flag来控制循环的阻塞行为。

ev_run的工作内容，在[官方文档的API](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#FUNCTIONS_CONTROLLING_EVENT_LOOPS)中有详细说明，通过文档有助于对ev_run的理解。具体的代码有点长，在这里就不贴了，感兴趣的同学可以在ev.c中查看ev_run的实现代码。剔除掉条件检查和一些无关紧要的代码，主要的流程如下图所示。

<center>
	<img src="http://i.6.cn/cvbnm/4f/7f/ea/36f77813a07b89788a1ede622c3d34f9.png" alt="ev_run流程图"/>
</center>

可以看到，ev_run的主要工作就是按watcher的分类，先后检查各种类型的watcher上的事件，通过ev_feed_event函数将就绪的watcher加入到pending的数据结构中。最后调用ev_invoke_pending，触发pending中的watcher。完了以后会检查，是否还有active的watcher以及是否有ev_break调用过，然后决定是否要继续下一轮循环。

#总结
总的来看，libev的结构设计还是非常清晰。如果说，主循环ev_run是libev这棵大树的主干，那么功能强大，数量繁多的watcher就是这棵大树的树叶，而循环上下文ev_loop则是连接主干和树叶的树枝，它们的分工与职责是相当明确的。作为大树的主干，ev_run的代码是非常稳定和干净的，基本上不会掺杂外部开发者的逻辑代码进来。作为叶子的watcher，它的定位也非常明确：专注于自己的领域，只解决某一个类型的问题。不同的watcher以及watcher和主循环之间并没有太多的干扰和耦合，这也是libev能如此灵活扩展的原因。而中间的树枝ev_loop的作用也是不言而喻的，正式因为它在中间的调和，前面两位哥们才能活得这么有个性。

看到这里，libev的主体架构已经比较清楚了，但是似乎还没看到与性能相关的关键代码。与主干代码不一样，这些代码更多的是隐藏在实现具体逻辑的地方，也就是watcher之中的。虽然watcher的使用接口都比较相似，但是不同的watcher，底层的数据结构和处理策略还是不一样的。下面一篇文章我们就来探索一下libev中比较常用的几种watcher的设计与实现。