---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（三）欠条的状态"
categories: [程序设计]
tags: [scala,stateless-future,异步,无状态,有状态,欠条]
description: "欠条可以分为两类：无状态欠条和有状态欠条。"
---

欠条可以分为两类：无状态欠条和有状态欠条。

### 有状态欠条

有状态欠条的类型为`com.qifun.statelessFuture.Future.Stateful[AwaitResult]`，从欠条派生。

有状态欠条表示**正在进行**中的操作。可以通过调用`Stateful.value`和`Stateful.isCompleted`查询操作是否已经成功完成。

有状态欠条可以和`scala.concurrent.Future`相互隐式转换，但需要提供`ExecutionContext`隐式参数：

{% highlight scala %}
val myStatelessFuture:Future.Stateless[Unit] = Future[Unit] {}
import scala.concurrent.ExecutionContext.Implicits.global
val myConcurrentFuture:scala.concurrent.Future = myStatelessFuture
val myStatelessFuture2:Future.Stateless[Unit] = myResponder
{% endhighlight %}

之所以需要提供`ExecutionContext`隐式参数，是因为`scala.concurrent.Future`不支持“线程无关”模型，必须要指定线程池才能运转。

### 无状态欠条

无状态欠条的类型为`com.qifun.statelessFuture.Future.Stateless[AwaitResult]`，也从欠条派生。

无状态欠条是Stateless Future框架的核心特性，Scala标准库中没有类似的概念。Stateless Future框架之所以叫这个名字，也正是因为其特有的无状态欠条。

[初印象]({{ site.BASE_PATH }}/2014/06/26/stateless-future-based-asynchronous-programming-in-scala-1)一节的例子中出现的`Nio2Future`，就属于无状态欠条。此外，`Future`宏生成的也是无状态欠条，例如：

{% highlight scala %}
val myFuture:Future.Stateless[Unit] = Future[Unit] {}
{% endhighlight %}

无状态欠条中所有API签名都与有状态欠条一致，但是语义却有所不同。而且，相比有状态欠条，无状态欠条中缺少了`value`和`isCompleted`方法，无法查询欠条可否兑现（即操作是否完成）。

这是因为，顾名思义，无状态欠条其实根本不保存任何状态，也就没法提供操作进度供查询了。

其实，无状态欠条是懒人开的欠条。懒人在创建无状态欠条时，根本不会发起任何操作。无状态欠条所对应的操作，要到持有欠条一方调用`foreach`或者`onComplete`时，懒人才会开始执行。这个特性就是**惰性求值**。惰性求值正是纯函数式语言的特征之一。尽管无状态欠条的方法签名与有状态欠条大多雷同，但语义却不同于有状态欠条或`scala.concurrent.Future`，而更像Scala 2.8的Continuation插件或Haskell中的Monad。

无状态欠条的另一特性是**重复执行**。如果多次调用同一无状态欠条实例中的`foreach`或者`onComplete`，那么每次调用都会发起一次新的操作，每次调用所传入的回调函数，都会收到相互独立的操作结果。

无状态欠条可以和`scala.Responder`相互隐式转换：

{% highlight scala %}
val myStatelessFuture:Future.Stateless[Unit] = Future[Unit] {}
val myResponder:Responder = myStatelessFuture
val myStatelessFuture2:Future.Stateless[Unit] = myResponder
{% endhighlight %}

### 两类欠条的相互转换

要想把无状态欠条转换成有状态欠条，应使用`Promise`，而要想把有状态欠条转换成无状态欠条，应使用`Future`宏进行包装：

{% highlight scala %}
val myStatelessFuture:Future.Stateless[Unit] = Future[Unit] {}
val myStatefulFuture:Future.Stateful[Unit] = Promise.completeWith(myStatelessFuture)
val myStatelessFuture2:Future.Stateless[Unit] = Future[Unit] { myStatefulFuture.await }
{% endhighlight %}


