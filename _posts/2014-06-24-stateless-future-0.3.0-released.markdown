---
author:
  name: 小绽
  email: yangbo@qifun.com
  url: "http://qforce.qifun.com/"
  github: qifun
  bio: 岂凡技术小站
  email_md5: 50993397b1b5ed2eae8e50e25d9663e8 
layout: post
title: "Stateless Future 0.3.0发布"
categories: [岂凡开源项目]
tags: [岂凡,开源,stateless-future,generator,nio,async,await,yield]
description: 岂凡网络发布了Stateless Future 0.3.0和Stateless Future工具库0.4.0，为Scala提供<code>await</code>异步语法。
---
今天，岂凡网络发布了Stateless Future 0.3.0和Stateless Future工具库 0.4.0，新增功能：

 * 新增`JumpInto`，提供了在`Future`中切换`Executor`的功能。
 * 新增`Sleep`，提供了在`Future`中异步等待的功能。
 * 把`Promise`从Stateless Future移到Stateless Future 工具库，精简了核心库。
 * 增加了一些测试用例和示例。
 * 把`Nio2`类改名为`Nio2Future`。

## 关于Stateless Future

Stateless Future是岂凡网络开发的异步框架，主要功能有：

 * 为Scala提供了类似C# `async`/`await`的`Future`/`await`异步语法。
 * 为Scala提供了类似C# `yield`语法的`Generator`。
 * 解决了Scala标准库`scala.concurrent.Future`和提议标准`scala.async`的问题：无法处理异步代码中的异常。
 * 解决了Scala标准库`scala.concurrent.Future`的问题：无法跨线程模型进行异步编程。
 * 整合了Java NIO.2 API。
 * 整合了Akka Actor API。

### 示例

{% highlight scala %}
// 用Stateless Future编写的异步TCP客户端
Future {
  val socket = AsynchronousSocketChannel.open()
  try {
    Nio2Future.connect(socket, new InetSocketAddress("localhost", 8000)).await
    val buffer = ByteBuffer.wrap("Hello, World!".getBytes)
    Nio2Future.write(socket, buffer).await
  } finally {
    socket.close()
  }
}
{% endhighlight %}

### 相关链接

 * [Stateless Future项目首页](https://github.com/qifun/stateless-future)
 * [核心库API文档](http://central.maven.org/maven2/com/qifun/stateless-future_2.11/0.3.0/stateless-future_2.11-0.3.0-javadoc.jar)
 * [工具库API文档](http://central.maven.org/maven2/com/qifun/stateless-future-util_2.11/0.4.0/stateless-future-util_2.11-0.4.0-javadoc.jar)
 * [Akka整合库API文档](http://central.maven.org/maven2/com/qifun/stateless-future-akka_2.11/0.1.1/stateless-future-akka_2.11-0.1.1-javadoc.jar)
