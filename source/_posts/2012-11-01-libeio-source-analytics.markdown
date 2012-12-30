---
layout: post
title: "libeio源码浅析"
date: 2012-11-01 18:31
comments: true
categories: C/C++
tags: [libeio, C/C++, 源码分析]
description: 分析libeio库的实现原理
keywords: libeio, aio, code analytics
---

[libeio](http://software.schmorp.de/pkg/libeio.html)是用C开发的高效异步IO库，填补了普通文件没有异步接口的空白。它的作者也是Marc Lehmann，所以它的整体设计风格和libev还是有几分相似的。本文主要记录一下近期对libeio源码分析的心得，作为分析libuv的基础。关于linux下另外一些aio的实现方案，如glibc和kernel native aio，以及libeio背后的一些八卦等，在[这篇文章](http://cnodejs.org/topic/4f16442ccae1f4aa270010a7)和[这篇文章](http://hi.baidu.com/_kouu/item/2b3cfecd49c17d10515058d9)都有比较深入的分析，这里就不赘述了。

<!-- more -->

#libeio示例
按照惯例，我们还是先上一个简单的例子，看一下libeio的基本工作流程吧。

```c
	#include<stdio.h>
	#include<eio.h>

	void want_poll () {
		printf ("I want poll\n");
	}

	void done_poll () {
		printf ("I done poll\n");
	}

	int res_cb (eio_req *req) {
	  printf ("res_cb(%d|%s) = %d\n", req->type, req->data ? req->data : "?", EIO_RESULT (req));

	  if (req->result < 0)
	    abort ();

	  return 0;
	}

	int main(void) {
		if (eio_init (want_poll, done_poll)) {
			printf ("fail to init eio.\n");
			abort ();
		}

		eio_mkdir ("eio-test-dir", 0777, 0, res_cb, "mkdir");

		printf ("\nentering event loop\n");
		while (eio_nreqs ()) {
			printf ("eio_poll () = %d\n", eio_poll ());
		}
		printf ("leaving event loop\n");

		return 0;
	}
```

这个例子是从官方demon简化而来的。所做的事情是在当前目录下创建一个eio-test-dir的目录，然后轮询等待它执行完毕。

#libeio工作原理

libeio主要是通过线程池来模拟异步操作，大致流程如下图所示：

<center>
	<img src="http://i.6.cn/cvbnm/dc/4b/1b/51f2c6bc78aaff156a5f09ee5c7dac65.png" title="libeio-arch" />
</center>

在主线程中可以通过libeio提供的异步接口发起请求，如例子中的eio_mkdir。之后请求的信息和完成后的回调函数会封装成eio_req对象，存放到req_queue队列中。线程池中的工作线程会从eio_queue中取出请求来处理。这里将会调用相关的系统接口，可能会发生阻塞。工作线程处理完毕后，将结果保存在eio_req对象中，并将其填入res_queue中。而主线程要做的就是调用eio_poll来处理res_queue中就绪的请求，触发其中的回调函数。

#主要参与者

###eio_init
eio_init是libeio库的初始化函数，它的主要工作就是初始化libeio中的一些全局结构，比如：req_queue，res_queue，以及各种互斥量等。req_queue和res_queue的结构是一样的，定义如下：

```c
	typedef struct {
		ETP_REQ *qs[ETP_NUM_PRI], *qe[ETP_NUM_PRI]; /* qstart, qend */
		int size;
	} etp_reqq;
```

这是一个二维的链表，第一维是优先级，第二维是eio_req对象的列表。工作线程和eio_poll会根据优先级先后来队列中的eio_req对象。这两个队列都会在多线程间共享，分别由reqlock和reslock两个互斥量来保证互斥访问。

eio_init的另一个工作就是保存外界传入的两个回调函数：want_poll和done_poll。这两个函数都是边缘触发函数，具体的作用和触发时机如下：

* want_poll 在res_queue从空转为非空的时候被触发，表示libeio中有异步请求已就绪，请求外界调用eio_poll来处理。want_poll一般是在工作线程内被调用。

* done_poll 则刚好相反，是在res_queue从非空转成空时被触发，表示libeio中所有就绪的异步请求都已处理完毕。done_poll在调用eio_poll的那个线程中被调用（一般是主线程）。

这两个函数是在已获取了互斥量的上下文中被调用了，所以不能在这两个函数中调用eio相关的函数，更多的是通过事件的形式通知另外的线程来做后续工作。

###eio_OPERATE

eio_OPERATE是libeio封装的异步操作[接口](http://pod.tst.eu/http://cvs.schmorp.de/libeio/eio.pod#HIGH_LEVEL_REQUEST_API)，如：eio_mkdir, eio_read等。

虽然这些异步接口功能各异，但主要的流程都是相似的，如eio_mkdir的代码：

```c
	eio_req *eio_mkdir (const char *path, mode_t mode, int pri, eio_cb cb, void *data)
	{
		REQ (EIO_MKDIR); PATH; req->int2 = (long)mode; SEND;
	}
```

前面的代码都是构造eio_req对象，最后的SEND宏则是调用etp_submit函数，将请求加入到req_queue队列中，关键代码如下：

```c
	X_LOCK (reqlock);
	++nreqs;
	++nready;
	reqq_push (&req_queue, req);
	X_COND_SIGNAL (reqwait);
	X_UNLOCK (reqlock);

	etp_maybe_start_thread ();
```

先获取reqlock，更新相应的计数，将请求对象入队，唤醒wait的工作线程等。最后的etp_maybe_start_thread函数则是检查是否要创建新的工作线程。工作线程的默认数量是4,可以通过eio_set_min_parallel和eio_set_max_parallel配置。

###请求对象

请求对象eio_req是对所有异步请求的抽象。所有上层的请求，都会统一归纳到eio_req的结构体上，并通过req_queue和res_queue在不同的线程之间流转。底层的代码只对eio_req进行处理。从而实现了上下层代码之间的解耦。

在这里eio_req的多态，实现手法有点粗暴。作者将libeio中所有请求所需的信息的并集归纳成了eio_req，所以这个结构体有点庞大，有的字段在某些请求中是不需要的。另外为了使字段适用于所有类型的请求，也采用了比较泛的字段命名，诸如：ptr1, ptr2, int1, int2, int3等。作者也在这些字段旁边加了比较详细的注释作为补救。所幸的是eio_req对象都是由libeio内部创建，内部维护，对外基本上只是对其errorno和result等字段进行只读操作。具体的代码就不贴了，可以在eio.c中查看定义。

###处理线程
处理线程的代码在etc_proc中定义。基本上就是一个无限循环，不断地从req_queue中取任务出来处理。如果没有任务则在reqwait上wait。处理请求的代码如下：

```c
	ETP_EXECUTE (self, req);

	X_LOCK (reslock);

	++npending;

	if (!reqq_push (&res_queue, req) && want_poll_cb)
		want_poll_cb ();

	etp_worker_clear (self);

	X_UNLOCK (reslock);
```

ETP_EXECUTE是处理具体请求的地方，也是发生阻塞的主要地方。主体代码不复杂，但挺长。主要是一个巨大无比的switch/case，根据eio_req中的type判断请求类型，之后就是执行相应的同步接口，并将结果保存到eio_req中返回。

接下来的代码就是获取相关的互斥量，并将处理完毕的eio_req存放入res_queue中。如果有需要，则触发want_poll回调。

###eio_poll

主要是完成res_queue中就绪的eio_req对象的处理。ETP_FINISH宏中会调用eio_req中绑定的回调函数。

```c
	for (;;)
	{
		ETP_REQ *req;

		etp_maybe_start_thread ();

		X_LOCK (reslock);
		req = reqq_shift (&res_queue);

		if (req)
		{
			--npending;

			if (!res_queue.size && done_poll_cb)
				done_poll_cb ();
		}

		X_UNLOCK (reslock);

		if (!req)
			return 0;

		X_LOCK (reqlock);
		--nreqs;
		X_UNLOCK (reqlock);

		// ...

		int res = ETP_FINISH (req);
		if (ecb_expect_false (res))
			return res;

		//...
	}
```

#总结

总的来看，libeio的实现并不复杂，通过线程池模拟异步，通过eio_req和req_queue，res_queue对代码进行解耦。一般要设计一个异步库的实现大致思路也是如此了吧。但同时它也很实用。它基本支持了POSIX的所有文件操作接口，没有引入过多的复杂性，底层的依赖的也只有pthread而已。所以它具有较好的可移植性，使用非常方便。