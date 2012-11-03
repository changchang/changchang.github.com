---
layout: post
title: RedDwarf服务器数据模块分析
categories:
- Game
- Java
tags: [Game, Java, sgs]
published: true
comments: true
description: 分析RedDwarf游戏服务器的水平扩展原理和分布式事务实现
keywords: RedWarf, game server, horizontal scalability, distributed transaction
---

这段时间在分析RedDwarf服务器的代码，以下简称RD。RD最吸引我的地方就是水平扩展的能力了。一个游戏服务器，进行水平扩展，不可避免的一个问题就是分布式数据访问，即不同的服务器节点之间的数据共享问题。如：一个游戏服务器节点上的玩家要于另外一个节点上的玩家进行交互，中间的数据如何保证事务性，延时如何满足需求等。

<!--more-->

##RedDwarf的数据层的工作原理

RD将数据共享的重任委托给了一个强大的数据层。RD将需要共享的对象定义成ManagedObject（以下简称MO）。MO可以通过DataManager提供的接口持久化到统一的数据层，各个服务器节点再通过统一的数据层来访问共享对象。

RD的数据服务器层简单的来看可以分为三层。DataManager是一个接口，具体实现实际是DataServiceImpl。所以三层结构关系主要如下：DataService -> DataStore ->DB。

* DataService主要为开发者提供了数据层的操作接口;

* DataStore则为中间层，主要屏蔽主从节点的差异以及底层数据库实现的差异;

* DB层负责与具体的数据库进行交互，目前主要有两种实现BDB和JE。在目前RD版本中，只有一个中心节点负责运行数据库，其他的从属节点实际上都是通过远程调用委托中心节点进行数据库操作。

RD在多节点模式下运行时，有一个coreServer的中心节点需要首先起起来。coreServer的一项重要任务就是启动DataStoreImpl，后者将建立中心数据库（默认为BDB），并启动一个DataStoreServer实例，将代理DataStore的远程接口暴露出去。而之后的附属节点appServer启动之后，会建立一个DataStoreClient的实例来根据配置文件连接中心节点的DataStoreServer。DataClientClient实现了DataStore接口，所以它实际上充当的是DataStore的角色，将本节点的DataService的请求转交给中心节点的DataStoreServer。

就是通过这样的结构，RD将所有从节点的数据请求都转移到了中心节点上，再由中心节点进行统一的事务调度，避免了分布式事务。

##中心节点的事务

那么，在中心节点里，RD又是怎么处理数据库事务的呢？

在讲这个之前，需要先讲一下ManagedObject的原理。ManagedObject是一个标记接口，跟Serializable一样。RD要求所有MO都实现ManagedObject接口，同时也要实现Serializable接口，因为MO最终需要序列化再存到数据库里的。

每一个MO，在DataService内部实际上会有一个ManagedObjectReference（以下简称MOR）实例与之对应（无论是createReference和getBinding操作获得）。MOR主要维护的是objectId（即oid）到实际的java object的映射关系，底层数据库存的其实也就是oid->object的key/value对。MOR内部也会缓存一个与它关联的oid->object的映射。只有第一次调MOR的get方法时，才会真正到数据库里去取object的数据并反序列化，并把结果缓存到MOR内部。之后的所有get操作，都会从缓存里取，以提高速度和降低数据库负担。这就相当于本地有了个缓存。

那RD怎么保证这个缓存和DB的一致性呢？即如果别的节点改变了MO的内容，当前节点怎么得知呢？这也就是前面提到的数据库事务的问题了。

RD很聪明，把这个艰巨的任务交给了底层的DB。RD底层依赖的是BDB和JE数据库，它采用的默认锁模式是，READ_COMMITED=false和READ_UNCOMMITED（具体可以参考[BDB LockMode](http://docs.oracle.com/cd/E17277_02/html/java/com/sleepycat/je/LockMode.html)和BdbEnvironment的构造函数）。简单的来说，RD对底层的DB的读操作会hold住该数据的一个读锁，直至事务结束。后面其他的事务如果也申请读锁是可以的，但申请写锁，则需要等待读事务结束后才能继续或者抛出TimeOut异常。从而保证了数据的事务性访问。我们所熟悉的DataManager的MarkForUpdate方法，实际上也是向数据库申请了一个写锁，直至事务结束后释放。

这样带来的一个问题就是，热门的数据可能会有多个事务竞争，产生瓶颈。不过RD的模型本来就建议采用短事务，默认事务时长是100ms以内，数据访问冲突的超时时间是事务超时时间的1/10,所以一般的游戏，除了FPS或即时的MMORPG，应该还是可以满足需求的。

##代码实验

说了那么多，都是从代码里看的，木得实际依据，为了证实我们的猜测，上点代码吧。

测试环境，两台RD服务器节点（一个coreServer和一个appServer）。

coreServer配置app.properties

```java
com.sun.sgs.app.name=Hello
com.sun.sgs.app.listener=test.Hello
com.sun.sgs.app.root=data
com.sun.sgs.services=test.MyServiceImpl
com.sun.sgs.txn.timeout=300000
com.sun.sgs.node.type=coreServerNode
com.sun.sgs.server.host=localhost
```

appServer配置app.properties

```java
com.sun.sgs.app.name=Hello1
com.sun.sgs.app.listener=test.Hello
com.sun.sgs.app.root=data
com.sun.sgs.services=test.MyServiceImpl
com.sun.sgs.txn.timeout=300000
com.sun.sgs.node.type=appNode
com.sun.sgs.server.host=localhost
```

为了展现访问冲突，有个事务需要中途sleep一阵子，所以把事务超时设为300s。

在coreServer的MyServiceImpl的doReady方法里加入以下代码

```java
	this.scheduler.scheduleTask(new AbstractKernelRunnable(name) {
		@Override
		public void run() throws Exception {
			String name = "chang";

			MyData bd = (MyData)txnProxy.getService(DataService.class).getBinding(name);
			System.out.println(new Date() + " before:" + bd.getA());
			Thread.sleep(10000);
			MyData ad = (MyData)txnProxy.getService(DataService.class).getBinding(name);
			System.out.println(new Date() + " after:" + ad.getA());
		}

	}, this.txnProxy.getCurrentOwner());
```

主要做的事情就是先读一个事先绑定好了的对象，观察它的值。然后休眠10s，再取一遍，观察值有没有变。

appServer的MyServiceImpl的doReady方法里加入以下代码

```java
	this.scheduler.scheduleTask(new AbstractKernelRunnable(name) {
		@Override
		public void run() throws Exception {
			System.out.println(new Date() + " begin.");
			String name = "chang1";
			MyData data = (MyData)txnProxy.getService(DataService.class).getBinding(name);
			txnProxy.getService(DataService.class).markForUpdate(data);
			data.setA(data.getA() + 1);
			System.out.println(new Date() + " finish.");
		}

	}, this.txnProxy.getCurrentOwner());
```

主要做的事情是取同一个对象，然后markForUpdate一下，然后修改对象的字段。
运行后，从输出结果的时间序列上看，appServer的事务会一直被堵着，直到coreServer的after输出完毕后，才能接着往下执行（这里因为加长了超时时间，所以不会冲突超时）。

那如果不做markForUpdate，直接修改会怎么样呢？其实效果也一样的，RD默认开启了自动检测修改，会将没有markForUpdate的MO设为MAYBE_MODIFIED状态，在事务提交时，会检测该MO是否有被修改过（主要原理是，get的时候做一下序列化，commit的时候再做一下序列化，两次序列化的结果进行比较看是否有改变），如果有改变的话就提交数据库，这时就会涉及到数据库写操作，一样会被阻塞住。
那么把appServer的

```java
txnProxy.getService(DataService.class).markForUpdate(data);
```

语句去掉。

执行的结果可以看到appServer上的begin和finish都在coreServer的after之前打印了，难道是绕过了RD的事务了？其实不是的，因为appServer上的begin和finish都是在transaction task的run方法里的，而数据提交是在transaction task的run方法执行完成后进行的，所以当finish打印时，实际上还没开始MO的flush，所以还不会堵塞。为了验证这一点，可以在coreServer上的DataStoreServerImpl的
```java
public void setObjects(long tid, long[] oids, byte[][] dataArray)
```
方法里，
```java
store.setObjects(txn, oids, dataArray);
```

语句前后加上日志，打印一下设置相应oid的请求的开始和结束时间，就可以直到，set请求其实是一直阻塞着，直到coreServer的事务完成后，才能往下执行的。

##总结

简单的看，RedDwarf分布式事务主要分三步来实现。第一步先将所有节点的操作请求集中到中心节点，将原来并发的请求串行化。第二步则是为请求分配读写锁，授权给各个节点。第三部，各个节点按读写约定执行事务逻辑，commit后同步到中心节点，完成事务。使用的前提是所有事务都是短事务，否则将会影响后面事务的执行。