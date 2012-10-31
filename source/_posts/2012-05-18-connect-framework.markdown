---
layout: post
title: connect源码分析--基础架构
categories:
- nodejs
tags: [node.js]
published: true
comments: true
description: connect源代码分析, 分析基础框架设计。
keywords: connect, node.js, 源代码分析
---
connect是TJ tx给node.js社区贡献的一个热门的web基础框架。TJ的另一力作express框架便是在它基础之上构建的。与express不同，connect更加短小精悍，是一个偏向基础设施的框架。

正如名字所表达的一样，connect框架做的事情很简单，就是把一系列的组件连接到一起，然后让http的请求依次流过这些组件。这些被connect串联起来的组件被称为中间件（middlewire）。在connect中，http请求的处理流程被划分成一个个小片段，每一个小片段代表一项处理任务（如：请求body的解析，session的维护等），由一个中间件负责，前后片段之间靠request，response等对象传递中间数据。connect框架对这些处理细节并不关心，只知道将请求从一个中间件导向下一个中间件。connect的核心代码非常精简，加上注释，也就只有寥寥200来行代码。本文参考的connect版本为2.2.2（这个版本号好像有点特别-_-b）。

<!--more-->

先上个connect官网提供的一个Hello world例子吧：）

```javascript
  var connect = require('connect')
  , http = require('http');
  var app = connect()
  .use(connect.favicon())
  .use(connect.logger('dev'))
  .use(connect.static('public'))
  .use(connect.directory('public'))
  .use(connect.cookieParser('my secret here'))
  .use(connect.session())
  .use(function(req, res){
    res.end('Hello from Connect!\n');
  });
  http.createServer(app).listen(3000);
```

很简单，很清晰的一个链式调用代码。风格与express的很像，不，应该是说express的风格跟它很像，总得有个先来后到，是吧。那么下面开始对connect的核心代码进行解析吧～

##入口代码
connect的入口代码在lib/connect.js中，代码不多，主要定义了createServer函数。该函数最终被赋给module.exports，当成connect模块暴露出去，即对应于例子中的connect()函数。createServer的具体代码如下：

```javascript
  function createServer() {
    function app(req, res){ app.handle(req, res); }
    utils.merge(app, proto);
    utils.merge(app, EventEmitter.prototype);
    app.route = '/';
    app.stack = [];
    for (var i = 0; i &lt; arguments.length; ++i) {
      app.use(arguments[i]);
    }
    return app;
  };
```

这个方法里主要定义了一个名字叫app的函数，然后通过utils.merge方法，把proto（由lib/proto.js定义，后面会介绍）和EventEmitter的方法merge到了app函数上。在js中，function也是对象，这就相当与给app对象添加了proto和EventEmitter的特性。这个方式，有点像松本tx提到的Ruby中的mixin，是实现多继承的一种途径。与mixin所不同的是，这里的merge会把后面merge进来的属性覆盖掉原来已有的同名属性。

merge的结果，让我们拥有了一个继承了proto和EventEmitter的function对象，像app.handle, app.use等这些方法，其实都是从proto处继承而来的。这个function的函数体很简单，就是调用自己的handle方法来处理http请求。随后，这个function会被当作回调函数传递给node.js原生http模块的createServer方法，成为http请求处理的源头。

这里有个地方需要注意的，createServer函数返回的是一个叫app的function，期间并没有用new关键字，所以它不是由app作为构造函数而构建出来的一个纯粹的js对象，它依然还是一个function，所以它可以作为回调函数传递给http.createServer方法。

此外，createServer可以接受任意数量的中间件对象作为参数，在函数的最后会逐一加载这些传递进来的中间件。

lib/connect.js文件的最后，会遍历lib/middleware目录下的connect自带的中间件模块，并将这些模块挂到app和app.middleware之下，于是就有了例子里的connect.favicon, connect.logger等。

##中间件加载

connect的最大灵活性就在于它的中间件机制。connect的中间件加载代码定义在lib/proto.js中的app.use方法。use方法的定义如下：

```javascript
  app.use(route, fn)
```

route是中间件所使用的请求url的pattern，默认为'/'。connect会按照加载顺序，逐一执行pattern与请求url匹配的中间件处理函数。第二个参数fn即中间件处理函数，有两种定义形式。

第一种形式为：

```javascript
  function(req, resp, next)
```

第二种形式为：

```javascript
  function(err, req, resp, next)
```

第一种是正常的处理函数，第二种是异常处理函数。

req, resp为http模块的request和response对象。next是触发后续流程的回调函数，带一个err参数。通过传递给next传递一个err参数，告诉框架当前中间件处理出现异常。如果err为空，则会按顺序执行后面正常处理函数，忽略异常处理函数;相反，如果err非空，则会按顺序执行后续的异常处理函数，而忽略正常处理函数。

在connect中，请求的处理流程是一个异步的过程，fn函数的返回并不代表处理流程的结束，所以在这里需要用next回调的形式通知框架执行后续的流程。但如果某一个中间件函数认为请求的流程到它那里已经处理完毕，无需再执行后面的流程，则可以直接返回，而不用再调用next回调函数了（包括正常和异常处理函数）。如果请求遍历完中间件列表后仍在调用next函数，connect则会认为这个请求没有中间件认领。这时，如果next的err参数非空，则会给页面返回500错误，表示server出现了内部错误;如果err为空，则返回404错误，即访问的资源不存在。

依照中间件函数的定义，我们也可以编写自己的中间件函数，然后通过use方法加入到connect的处理流程中。

在这里，我们也可以看出来，各个middleware之间其实并没有直接的依赖。request和response就成为它们在connect中传递信息的唯一桥梁。前面的中间件处理完毕后，把结果附到request或response之上，后面的中间件便可以从中获取到这些数据。所以，中间件的加载顺序在connect中就显得格外重要，必须将被依赖的中间件放在依赖它的模块之前。比如说，解析cookie的中间件应该放在处理session的中间件之前，因为一般session id是通过cookie来传递的。

##处理流程

经过use之后，我们就拥有了一个中间件和pattern的列表，接下来就是怎么让http请求流经这个列表了。

前面也提到过，http请求的处理源头就是app.handler方法。这个方法也比较简单，主要定义了一个next的函数，也就是上一节提到的next回调函数。next的调用其实是一个尾递归过程。每一次调用next，都会从列表中取出一个中间件，进行pattern匹配检查。这里的pattern匹配，其实是简单的startWith检查。也就是说，请求的url是以pattern开头，并且紧接着下一个字符是/或.的话，才认为是pattern匹配上了，然后才会将请求流入当前这个中间件的处理函数中。pattern匹配的代码如下：

```javascript
  path = utils.parseUrl(req).pathname;
  if (undefined == path) path = '/';
  // skip this layer if the route doesn't match.
  if (0 != path.indexOf(layer.route)) return next(err);
  c = path[layer.route.length];
  if (c && '/' != c && '.' != c) return next(err);
```

其中，layter是保存中间件信息的数据结构，结构为：{route: route, handle: fn}。

在handler方法中，我们也可以看到，上一节提到的两类中间件函数的处理判断，代码如下：

```javascript
  var arity = layer.handle.length;
  if (err) {
    if (arity === 4) {
      layer.handle(err, req, res, next);
    } else {
      next(err);
    }
  } else if (arity &lt; 4) {
    layer.handle(req, res, next);
  } else {
    next();
  }
```

可以看出来，connect是根据中间件处理函数的参数列表长度来区分中间件种类的。

##总结

###设计模式

总的来看，connect其实是典型的chain of responsibility模式的实现。各个中间件就是责任链上的一个节点，route pattern就是决定中间件是否参与请求处理的判断条件。链只知道请求的传递，而中间件只知道自己的处理逻辑，这样对象之间的耦合是非常松散的，任务的指派过程也是相当灵活的。第三方按照中间件的接口约定即可开发自己的中间件并加入到connect中，同时还可以调整链的构建顺序来控制请求处理的流程。但由于http处理流程的一些特殊性，有的中间件之间也存在有一些隐含的依赖关系（如前面提到的cookie和session），所以connect对中间件的加载顺序也有着一定的要求。

###filter
由于connect采用的是链式的结构，所以我们可以很方便地实现filter机制，将业务逻辑放在一个中间件中，而一些前期准备工作和后期的清理工作放到另外的中间件中，使得代码模块更加清晰，更容易复用。但正如前面所提到的，connect中，中间件之间是以异步的形式来进行衔接的，中间件函数的返回并不代表处理的结束，而需要通过next回调函数来执行后续流程。所以，connect中可以分别提供before和after的filter，但无法提供类似Java中的around风格的filter。这对于一些在after filter中需要访问before filter中上下文信息的情况来说并不是很方便。

###八卦
按照TJ tx的最初构想，connect应该只是一个基本的工具框架，供其他上层的框架（如：express）所使用，而一般情况下不应该直接使用connect。但connect是如此的好用，以至于很多人直接就把connect用到自己的应用之中了，这也许并不是TJ的本意。从前面的分析里也可以看到，connect的url匹配规则其实是很简单的，而具体的url匹配，请求处理工作是由router中间件负责的。熟悉connect的tx应该也有这种感觉，router绝对是当中重量级的一个中间件。但从TJ对connect和express的布局上来看，这个中间件似乎应该更靠近express一些。所以，TJ tx最终做了一个艰难的决定，把router从connect迁移到了express里。具体的讨论可以到[这里](https://github.com/senchalabs/connect/issues/262)围观。
