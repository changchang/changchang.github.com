---
layout: post
title: websocket的压缩
categories:
- Basic
- Javascript
- nodejs
tags: []
published: true
comments: true
---

websocket目前的版本只支持文本格式进行传输，预留有标志位供以后的版本扩展。而传输常常考虑的一个问题就是传输的效率。那么websocket是否跟http协议一样，能对传输的内容进行压缩来降低通讯所占用的带宽呢？

<!--more-->

从协议上看，websocket是支持压缩的。websockete里的压缩是以扩展的形式加进来的，参考：
[这里](http://tools.ietf.org/html/draft-tyoshino-hybi-websocket-perframe-deflate-00)

目前扩展只支持DEFLATE压缩，客户端和服务器端协商的过程也很简单，双方在握手阶段互发一个扩展头（客户端先发起）即可，然后后续所发的包的全部数据，包括header和body内容全部用DEFLATE算法进行压缩。（applies DEFLATE to everything including header octets such as opcode and payload length.）
Sec-WebSocket-Extensions: deflate-application-data

正如前面提到的，因为这个扩展的压缩会连header一起压缩，会导致一些代理无法直接解析header的信息，而需要先将整个数据解压。另外，websocket里客户端向服务器发的数据，需要先经过mark即用header里的marking-key对payload的字节进行异或，使包里数据尽量均匀，而这也将对DEFLATE压缩算法的效率造成影响。目前这个扩展争议还是比较大的。
[参考](http://www.lenholgate.com/blog/2011/07/websockets---the-deflate-stream-extension-is-broken-and-badly-designed.html)

不过目前websocket api也没有提供设置该参数的接口，支持的浏览器和服务器估计也不多吧。先记录下来，留待以后观察。

----------2011-10-09更新
上面的草案老了，今天找了个新的。

websocket协议支持对传输内容进行压缩。压缩设置以extension的形式提供，目前仅定义了deflate压缩，需要浏览器和服务器的支持。如果浏览器支持，在websocket握手的时候会进行extension协商，最简单的形式就是添加协议头
Sec-WebSocket-Extensions: deflate-frame
如果服务器也支持则在响应中添加同样的协议头即可。
之后传递的数据包的payload部分都经过deflate压缩。具体的协议内容参看[这里](http://tools.ietf.org/html/draft-tyoshino-hybi-websocket-perframe-deflate-04)

websocket协议也内置了一个压缩的扩展deflate-stream，具体可以参考[这里](http://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-10#section-9.2.1) 。但这个扩展存在不少争议，如：它会连同协议的头也一起压缩，导致中间的代理需要先解压然后才能解析websocket协议的头信息。参考WebSockets - [The deflate-stream extension is broken and badly designed](http://www.lenholgate.com/blog/2011/07/websockets---the-deflate-stream-extension-is-broken-and-badly-designed.html)

目前貌似还没有浏览器支持deflate-frame。对于deflate-stream，主要观察了一下firefox和chrome。firefox6支持deflate-stream压缩扩展，可以在about:config里network.websocket.extensions.stream-deflate设置。截包来看，能看到相应的扩展头。chrome 14.0.835.186没找到配置的地方，截包看默认没加压缩扩展头。

附:[各个浏览器对websocket版本的支持情况](https://developer.mozilla.org/en/WebSockets)
