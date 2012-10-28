---
layout: post
title: libev源码分析 -- 常用watcher
categories: C/C++
tags: [libev, C/C++, 源码分析]
published: true
comments: true
---

在上一篇文章里，我们分析了libev整体设计思想和主循环的工作原理，也提到了watcher是衔接开发者代码的主要入口。watcher与开发者最接近，也与具体事件处理逻辑最接近。所以，watcher的具体实现，与性能的关系也相当密切。下面，我们就来分析一下，libev中常用的几种watcher的设计与实现。
<!-- more -->

#ev_io

##ev_io与底层io
ev_io的主要使命就是监听并响应指定文件描述fd上的读写事件。对fd的监听工作，主要委托给底层的io库来完成。libev对目前比较流行的io库都提供了支持，如：select, epoll以及windows的iocp等。在这里libev使用了Adaptor模式，通过统一的适配层隐藏了底层io库的细节。在loop初始化的时候（loop_init），会根据配置将函数指针绑定到底层的io库函数对应的适配代码上。所以，开发者可以很方便的把代码切换到不同的底层实现上。相关的函数有：backend_modify，向底层库注册fd事件，如：epoll的epoll_ctl；backend_poll，向底层库轮询fd上是否有感兴趣的事件发生，如：epoll的epoll_wait。适配器实现的代码可以在ev_LIB.c中看到，LIB是io库的名字，如：ev_epoll.c，ev_win32.c等。

<center>
	<img src="http://i.6.cn/cvbnm/c6/ef/2a/ac177b3435994c7088ef9eb3d32394f2.png" alt="ev_io层次结构图" />
</center>

##ev_io的结构

```c
	typedef struct ev_io
	{
	  EV_WATCHER_LIST (ev_io)

	  int fd;     /* ro */
	  int events; /* ro */
	} ev_io;
```
其中，EV_WATCHER_LIST是EV_WATCHER结构的链表节点结构。fd是监听的文件描述符，events是感兴趣的事件。
ev_io从它诞生的那一刻起，便于文件描述符紧密结合在一起了。ev_io的实例被存储在loop->anfds的结构中。anfds的结构如下图所示：

<center>
	<img src="http://i.6.cn/cvbnm/9c/37/cc/91f3e08d93a0d2f41b936c6f2f4895f5.png" alt="anfds结构图"/>
</center>

anfds其实是一个数组，它使用fd作为下标，数组中的元素是一个ANFD的结构。ANFD是一个维护fd信息的结构体。其中，events记录了当前fd上感兴趣的事件的记录。head是watcher列表的头指针，这个列表就是在这个fd上注册的watcher列表。当fd的大小超出了anfds的容量，anfds会进行相应的扩展。anfds可以理解成一个简易的map，记录了fd与ANFD结构的映射关系。虽然，fd的申请不一定是连续的，可能会导致数组中出现空洞，但通过fd可以迅速获取到相应的watcher列表，这也许是用空间换取时间的一个考量吧。

##ev_io的插入

从前面的介绍我们知道，要通过libev来监听fd上的事件，得要先插入一个ev_io到libev的loop中。ev_io的插入操作被封装在ev_io_start函数中。毫无疑问，libev首先会根据fd找到对应的watcher列表，并将新watcher加入到列表中。接下来，会调用fd_change函数，将fd加入到loop->fdchanges中。fdchanges是一个简单的数组，记录的是当前注册事件有发生改变的fd。到此为止，新ev_io的插入完成，上面的所有操作时间代价都是O(1)。fdchanges的作用在下一个小节中进行分析。

##ev_io的选取

前面我们已经向libev的loop中插入了一个ev_io，那么libev是怎么把这个ev_io注册到底层io并响应底层的io事件的呢？ 要回答这个问题，我们得先回到[上一篇文章](/blog/2012/10/09/libev-framework/)。从ev_run流程图中可以看到，ev_io的选取由fd_reify和backend_poll这两个步骤来完成。

fd_reify函数的工作主要是遍历fdchanges，将对应列表的watcher的events字段合并到ANFD结构的events字段。ANFD上如果新的events与原来监听的events不一致，则表示在这个fd上监听的事件集发生了变化，需要将fd和事件集合通过backend_modify注册到底层的io库。

在循环的后面，则会调用backend_poll来检查fd上是否有注册的事件发生。如果有事件发生，则通过fd_event函数，遍历fd对应的watcher列表，比较每个watcher上的events字段与发生的事件event值，找出就绪的watcher添加到pendings中。

最后，这些pendings中的watcher会在循环结束前调用ev_invoke_pending来统一触发。

##ev_io的移除

ev_io的移除由ev_io_stop来完成。首先，会先检查该watcher是否在pendings列表中，如果是，则先从pendings中删除该watcher。pendings是一个数组，libev中的数组一般是以数组和元素数量来维护的。删除数组中的一个元素，只要把数组末尾的元素替换掉被删除的元素，并把元素数量减一就可以了，操作的时间复杂度是O(1) 。

接下来就是通过fd找到watcher列表，从中删除这个watcher。这个操作需要遍历列表找到待删除的watcher，所以平均时间复杂度是O(n)。其中n是注册在fd上的watcher数量，一般这个数量不会太大。

然后是把watcher的active标志位复位，并减少全局active的watcher计数。

最后是把fd加入到fdchanges中，因为移除一个watcher，可能会改变fd上感兴趣的事件，所以要在下一轮循环中重新计算该fd上的事件集合。

#ev_timer

##ev_timer的管理

ev_timer watcher是主要负责处理超时事件的watcher。这类watcher被存储在loop->timers中，它们的特点是，超时时间小的watcher会被先触发。所以，timers其实是一个按触发时间升序排序的优先队列，底层的数据结构是一个用数组实现的二叉或四叉最小堆（关于堆的定义请google之）。ev_timer watcher的active字段，维护的其实是watcher在堆中的下标，通过它可以快速在堆中定位到watcher。

堆相关基本的堆操作有upheap和downheap。upheap操作是将一个节点上移到堆中合适的位置；downheap操作则刚好相反，将一个节点下移到堆中合适的位置。最小堆的特点是父节点的值比子节点的值都要小。通过这两个操作，可以调整堆中节点的位置，以满足最小堆的约束。它们的时间复杂度都是O(log(n))。有了这两个操作，便可以构建出watcher结构的常用操作了。

* 获取超时的watcher。因为timers是一个以触发时间排序的最小堆，根部的watcher总是最先要触发的watcher。所以这个操作的主要工作就是比较根部watcher的触发时间，如果可以触发，则加入pendings队列。然后检查该watcher是否是repeat的。如果是则更新下一次触发时间，调用downheap操作将这个节点下移至合适的位置；否则直接删除该watcher。

* 添加新的watcher。这无非就是一个入堆的操作。将新watcher添加到timers数组的末尾，再执行upheap操作，上升至合适的位置即可。

* 删除watcher。将堆尾节点替换掉待删除节点，再根据情况用upheap或downheap操作来调整替换后节点到合适的位置。

以上这些操作的时间代价都是O(log(n))。

##timer的使用策略

在实际应用中，可能会出现频繁使用大量timer的场景。比如：为每个请求设置一个超时时间，在指定时间内得不到响应，则报错。如果为每一个请求创建一个watcher，则将产生大量不必要的空间和计算开销。在libev的官方文档中，提供了一个比较高效的[方法](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#Be_smart_about_timeouts)。下面简单介绍一下这种方法的思路。

可以使用一个双向链表来维护超时时间相同的timer，这里的timer可以理解为定时器的记录结构，比如：超时时间和超时的回调函数等。因为大家的超时时间是一样的，所以新的timer进来后，肯定是添加到队尾的。

然后分配一个ev_timer watcher专门来处理这个链表的超时工作。watcher的超时时间设置的是链表第一个元素的超时时间。当超时发生后，按链表顺序触发超时的timer。如果timer是重复的，可以重新计算超时时间并加入到链表尾部；否则直接删除timer记录即可。然后再从链表首部获取下一次超时时间，重复上面的流程。

采用这种方案，只需要使用一个ev_timer watcher来处理相同timeout时间的timer。而timer的增删操作最后其实就是链表的插入和删除操作，所以操作的时间代价都是O(1)。而timeout时间相同的约束，主要是要保证链表里的元素都是有序的，插入操作都是发生在链表的尾端。如果要取消某一个timer，因为是双向链表，也可以在O(1)时间内从链表内移除掉指定的节点。

#ev_prepare, ev_check, ev_idle

从角色上来看，这三个类型的watcher其实都是事件循环的一个扩展点。通过这三个watcher，开发者可以在事件循环的一些特殊时刻获得回调的机会。

* ev_prepare 在事件循环发生阻塞前会被触发。

* ev_check 在事件循环阻塞结束后会被触发。ev_check的触发是按优先级划分的。可以保证，ev_check是同一个优先级上阻塞结束后最先被触发的watcher。所以，如果要保证ev_check是最先被执行的，可以把它的优先级设成最高。

* ev_idle 当没有其他watcher被触发时被触发。ev_idle也是按优先级划分的。它的语义是，在当前优先级以及更高的优先级上没有watcher被触发，那么它就会被触发，无论之后在较低优先级上是否有其他watcher被触发。

这三类watcher给外部的开发者提供了非常便利的扩展机制，在这个基础上，开发者可以做很多有意思的事情，也对事件循环有了更多的控制权。具体到底能做些什么，做到什么程度，那就要看开发者们的想象力和创造力了：）

#总结

ev_io和ev_timer应该是libev中使用的最多的watcher了，也是比较典型的watcher。从底层的实现上来看，处理得恰到好处，精明而干练，可谓独具匠心。好了，拍作者马屁的话就少说了，这一轮libev的代码分析也到此先告一段落了，在这说说感受吧。

看libev的代码，最大的障碍应该是非里面漫天飞舞的宏莫属了。但把握了libev的大概结构后，知道哪些家伙长得比较像宏（比如：一些看似全局的变量），知道哪些宏要到什么地方找定义（比如：ev_vars.h，ev_wrap.h），事情就变得简单了。再回头想想，宏也是个不错的选择。第一，它是一个不错的代码解耦手段。上层代码依赖于宏，而宏在不同的环境下可以绑定到不同的底层代码，切换底层的代码而不会影响到上层代码。第二，它也是提高性能的途径。宏的绑定在预处理阶段完成，不会有额外的动态查找和函数调用开销。第三，也可以偷偷懒，用短小的宏代替一大坨代码，写的人省力，看的人也舒服，正所谓他好我也好，一举多得～