---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（一）初印象"
categories: [程序设计]
tags: [scala,stateless-future,异步]
description: "假如你要编写I/O密集的网络服务器，使用Stateless Future + Java NIO.2的组合，可以轻易达到Java平台的理论最高性能。"
---

[Stateless Future](https://github.com/qifun/stateless-future)是我为岂凡开发的异步编程框架，是岂凡下一代QForce游戏引擎的服务器基础库。

前些日子，岂凡已经将Stateless Future及其相关的工具库开源。Stateless Future用途广泛，我们认为，在网络和文件IO、Actor模型的并行计算、基于状态机的人工智能领域中，如果应用Stateless Future的异步模型，都能比传统的Java/Scala的同步风格或事件风格的代码，更为简练，更易于学习和维护。

## 初印象

Scala是一门静态类型语言，融合了函数式和面向对象的编程范式[^ScalaOverview]。Stateless Future贯彻了这一思路。

 * Stateless Future提供了命令式[^Imperative_programming]的语法风格。
 * Stateless Future生成的底层Java字节码大都属于纯函数式[^Functional_programming]风格，定义不可变数据结构，靠大量函数相互组合来完成操作。
 * Stateless Future遵守Scala的**静态类型**系统规则。其中的表达式都会在编译期执行类型检查和类型推断。

Stateless Future的核心语法是[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)/[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)宏。请看以下代码[^HttpClientExample]：

{% highlight scala %}
// 用Stateless Future编写的异步HTTP客户端
val httpFuture = Future[String] {
  val socket = AsynchronousSocketChannel.open()
  try {
    Nio2Future.connect(socket, new InetSocketAddress("www.qifun.com", 80)).await
    val request = ByteBuffer.wrap("GET / HTTP/1.1\r\nHost:www.qifun.com\r\nConnection:Close\r\n\r\n".getBytes)
    writeAll(socket, request).await
    val response = ByteBuffer.allocate(100000)
    readAll(socket, response).await
    response.flip()
    Charset.forName("UTF-8").decode(response).toString
  } finally {
    socket.close()
  }
}
{% endhighlight %}

[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏会把源代码中每一处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)方法替换成[onComplete]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])调用，把[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)之后的源代码“捕获”成一个函数对象，作为参数传给[onComplete]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])。最终生成一个[Future.Stateless]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html#Stateless[+AwaitResult]:Stateless[AwaitResult])，也从[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package.html#Future[+AwaitResult]:Future[AwaitResult])的派生类，其中的Java字节码不会包含任何对[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)方法的调用。比如：

{% highlight scala %}
readAll(socket, response).await
response.flip()
Charset.forName("UTF-8").decode(response).toString
{% endhighlight %}

会转换成

{% highlight scala %}
val readFuture = readAll(socket, response)
readFuture onComplete { _ => // 定义一个匿名回调函数，传给readFuture.onComplete方法
  response.flip()
  handler(Charset.forName("UTF-8").decode(response).toString)
}
{% endhighlight %}

另外两处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)也会被[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏以类似的方式转换成了[onComplete]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])。所以，整段代码最后会变成类似这样：

{% highlight scala %}
val httpFuture = new Future.Stateless[String] {
  override final def onComplete(handler: String => TailRec[Unit])(implicit catcher: Catcher[TailRec[Unit]]): TailRec[Unit] = {
    val socket = AsynchronousSocketChannel.open()
    val tryCatchFinallyFuture = new com.qifun.statelessFuture.ANormalForm.TryCatchFinally[String, Unit](
      new Future.Stateless[String] {
        override final def onComplete(handler: String => TailRec[Unit])(implicit catcher: Catcher[TailRec[Unit]]): TailRec[Unit] = {
          val connectFuture = Nio2Future.connect(socket, new InetSocketAddress("www.qifun.com", 80))
          connectFuture onComplete { _ =>
            val request = ByteBuffer.wrap("GET / HTTP/1.1\r\nHost:www.qifun.com\r\nConnection:Close\r\n\r\n".getBytes())
            val writeFuture = writeAll(socket, request)
            writeFuture onComplete { _ =>
              val response = ByteBuffer.allocate(100000)
              val readFuture = readAll(socket, response)
              readFuture onComplete { _ =>
                response.flip()
                handler(Charset.forName("UTF-8").decode(response).toString)
              }
            }
          }
        }
      }, PartialFunction.empty, socket.close())
    tryCatchFinallyFuture.onComplete(handler)
  }
}
{% endhighlight %}

如果你有Node.js的经验，你会感到生成的代码非常熟悉。

尽管在源代码中，[Nio2Future.connect]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/io/Nio2Future$.html#connect(AsynchronousSocketChannel,SocketAddress):Nio2Future[Void])与`ByteBuffer.wrap`两行代码相邻。然而，在转换后的代码及最终生成的Java字节码中，你会发现，[Nio2Future.connect]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/io/Nio2Future$.html#connect(AsynchronousSocketChannel,SocketAddress):Nio2Future[Void])在主线程中执行，但`ByteBuffer.wrap`却位于回调函数中。`ByteBuffer.wrap`要等TCP连接成功建立时，才会执行。而执行`ByteBuffer.wrap`的线程也不再是主线程，而是[AsynchronousSocketChannel.open()](http://docs.oracle.com/javase/7/docs/api/java/nio/channels/AsynchronousSocketChannel.html#open%28%29)所绑定的默认I/O线程池了。

这是因为，如果编写的[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)块内出现[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)，那么，[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)之后的代码运行在哪一线程，取决于[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)转换成的回调函数如何被调，而在上述例子中，[Nio2Future.connect]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/io/Nio2Future$.html#connect(AsynchronousSocketChannel,SocketAddress):Nio2Future[Void])返回了一个[Nio2Future]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/io/Nio2Future.html)的实例`connectFuture`，`connectFuture`会在默认I/O线程池中调用通过[onComplete]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/io/Nio2Future.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])传入的回调函数。

这一特点，我称之为**线程无关**模型。意思是，编写[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)的程序员不需要关心[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)前后的代码分别在哪个线程执行，他只需要知道前面的代码比后面的代码先执行，就够了。

线程无关模型，使得生成的代码，能够达到手写回调函数一样的性能，然而却不会带来任何额外的线程切换或等待的开销。

比如，上述示例代码虽然简单，但性能却极高，在2014年的主流桌面机和服务器上，能承受每秒数万次到数十万次的网络I/O操作。如果你用性能剖析工具评测性能，你会发现这些操作的瓶颈，都在Java NIO.2和操作系统的套接字实现中，相比之下，Stateless Future本身的开销几乎可以忽略不计。由于Java NIO.2 API是Java平台最高效的网络API，假如你要编写I/O密集的网络服务器，使用Stateless Future + Java NIO.2的组合，可以轻易达到Java平台的理论最高性能。

（待续）

---

[^ScalaOverview]: Martin Odersky, Philippe Altherr, Vincent Cremet, Iulian Dragos, Gilles Dubochet, Burak Emir, Sean McDirmid, Stéphane Micheloud, Nikolay Mihaylov, Michel Schinz, Erik Stenman, Lex Spoon, Matthias Zenger <cite>[An Overview of the Scala Programming Language, 2nd Edition](http://www.scala-lang.org/docu/files/ScalaOverview.pdf)</cite>

[^Imperative_programming]: 
    命令式编程
    : 组合对计算机下达的顺序指令的编程风格。C/C++/Java都是命令式编程语言。命令式变成的反义词是“声明式编程”。参见[维基百科](http://zh.wikipedia.org/wiki/%E6%8C%87%E4%BB%A4%E5%BC%8F%E7%B7%A8%E7%A8%8B)。

[^Functional_programming]: 
    函数式编程
    : 组合函数算子表达程序逻辑的编程风格，属于声明式编程。代表语言包括Lisp族、Haskell、ML族，但Python、JavaScript、Ruby、Lua等现代语言或多或少都具有函数式编程的部分特征。参见[维基百科](http://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B8%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80)。

[^HttpClientExample]: 该示例的完整代码在这里查看：[HttpClientExample.scala](https://github.com/Atry/stateless-future-example/blob/master/HttpClientExample.scala)。
