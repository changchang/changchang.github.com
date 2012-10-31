---
layout: post
title: 异步环境下的异常处理
categories:
- Java
- nodejs
tags: []
published: true
comments: true
description: 异步环境下异常处理的思路
keywords: node.js, java, tomcat, jetty, servlet3, 异步服务器, 异常
---

webgame容器的设计中遇到了一个问题，如何处理回调中未捕获的异常，维护系统状态的完整性。而这个问题的根源来自于node.js的异步回调模式。

先回想一下，熟悉的tomcat容器中是如何处理未捕获的异常的。

tomcat的模型比较简单，每个请求由一个单独的线程来处理，采用的是同步阻塞的模式。当一个请求到达后，请求会通过层层filter，最终到达servlet的service的方法。当请求从servlet的处理返回后，再按原路经过层层filter返回。整个处理的过程，都是在同一个线程，同一次函数调用周期中完成的。也就是说，完全可以在请求处理的入口处，用一个try...catch来捕获处理过程中抛出的所有的未捕获异常，并且在这个入口处，是可以获取到这个请求的上下文的，可以由此来给客户端返回错误的响应。

而在node.js中，采用的是异步回调的模式，处理函数返回，并不代表处理流程完成，可能还有后续的处理逻辑在回调函数中等待触发。而回调函数的触发可能会是在另外一个函数调用周期中，所以在当次请求的入口处包裹try...catch并不总是能捕捉到回调函数中抛出来的异常。但从另一面来看，回调函数最终一定是由node.js中处理任务的主线程来触发的，所以node可以帮我们捕捉到这些异常，反馈到开发者手里也就是我们所熟悉的uncaughtException事件。但在这种情况下，回调函数的执行环境是node的主线程，所以已无法获取到对应的请求上下文，也就无法给客户端反馈出错的信息了。

<!--more-->

作为对比，我简单的考察了一下java中的几个web容器提供异步请求处理接口的例子。

#tomcat comet

tomcat comet的处理手法比较简单粗暴，由容器管理线程的分配调度，只暴露给用户一个简单的event接口，让用户根据event的类型做处理。整个流程还是控制在容器的管理下的，即使event接口中，用户抛出了未捕获异常，容器还是可以捕捉到，并能知晓请求的上下文。代码框架如下：

```java
    /**
     * Process the given Comet event.
     *
     * @param event The Comet event that will be processed
     * @throws IOException
     * @throws ServletException
     */
    public void event(CometEvent event)
        throws IOException, ServletException {
        HttpServletRequest request = event.getHttpServletRequest();
        HttpServletResponse response = event.getHttpServletResponse();
        if (event.getEventType() == CometEvent.EventType.BEGIN) {
        } else if (event.getEventType() == CometEvent.EventType.ERROR) {
        } else if (event.getEventType() == CometEvent.EventType.END) {
        } else if (event.getEventType() == CometEvent.EventType.READ) {
        }
    }
```

#jetty continuation

jetty continuation的处理手法更加粗暴，通过Continuation实例的suspend方法来让当前请求挂起，并将处理线程归还给容器，等待超时或resume方法被调用时再继续执行该请求。

实现的手法是suspend方法内部会抛除一个RetryRequeset的RuntimeException，以此来终止当前请求的处理流程（具体参考SelectChannelConnector.RetryContinuation）。外层容器捕获该异常后，将请求加入超时队列，线程退还给容器。等到超时和resume方法被调用后，请求会从头再执行一次（包括前面的filter）。所以，continuation要求该类请求所经过的filter和servlet必须是幂等的（可以重复进入，不影响状态）。因为continuation其实是把request的流程再执行一遍，所以它实质上还是同步的模式，所以对未捕获的异常处理等同于tomcat的传统处理方式。

continuation的示例如下：

```java
private void doPoll(HttpServletRequest request, AjaxResponse response)
{
    HttpSession session = request.getSession(true);

    synchronized (mutex)
    {
        Member member = (Member)chatroom.get(session.getId());

        // Is there any chat events ready to send?
        if (!member.hasEvents())
        {
            // No - so prepare a continuation
            Continuation continuation = ContinuationSupport.getContinuation(request, mutex);
            member.setContinuation(continuation);

            // wait for an event or timeout
            continuation.suspend(timeoutMS);
        }
        member.setContinuation(null);

        // send any events
        member.sendEvents(response);
    }
}
```

#servlet 3.0
servlet 3.0中也加入了异步处理的支持，由ServletRequest对象通过startAsync方法获取一个AsyncContext，将请求处理变为异步化，当前的处理函数可以直接结束返回，开发者可以另外开一个线程，通过AsyncContext继续完成后续的响应。这个模型跟node.js是比较类似的。因为处理线程是用户派生的，如果有未捕获异常，容器是不知晓的。而因为异常，AsyncContext的complete或dispatch方法没有执行，将会导致请求无法返回，这点跟node也是类似的。

最后回头看看，在node的异步处理的模式下，容器对回调的控制较弱。一般的形式通过传递回调函数，由开发者代码调用来告知容器处理流程的真正结束，connect，nodeunit等都是用类似的手法来处理。如果回调函数中因为未捕获异常而退出，则后续流程会因此而丢失。做为容器来说，也有可能因为用户的不良代码，忘了调用回调函数而丢掉后面的流程。

暂时也没想到比较好的解决办法，目前的想法是设置回调超时。在每个需要用户代码调用回调的接口添加一个超时记录，用一个类似bitmap的东西来维护reqId和超时记录的映射关系。如果用户代码在指定时间内调用回调函数，则根据reqId清空超时记录;否则对该次请求做超时处理。
