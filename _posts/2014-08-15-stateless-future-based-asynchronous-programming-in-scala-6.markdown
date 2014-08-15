---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（六）<code>Future</code>宏的线程约定"
categories: [程序设计]
tags: [scala,stateless-future,异步,thread,线程]
description: "在线程无关模型中，使用<code>Future</code>宏时，程序员不需要关心JVM的线程模型，只需要关注于执行顺序和逻辑流程。不过，Stateless Future中确实有一些约定，决定了线程切换的行为。"
---

在本系列文章[第一节]({{ site.BASE_PATH }}/2014/06/26/stateless-future-based-asynchronous-programming-in-scala-1)中，我曾提及，Stateless Future采用**线程无关**模型。在线程无关模型中，使用[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏时，程序员不需要关心JVM的线程模型，只需要关注于执行顺序和逻辑流程。不过，Stateless Future中确实有一些约定，决定了线程切换的行为。

## [Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏的线程约定

[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏的功能是把源代码中每一处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)调用替换成[onComplete]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])调用。每一处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)调用以后的所有代码，都会被转换成回调函数，作为参数传给[onComplete]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])。

例如，给定以下代码：

{% highlight scala %}
val future1 = Future {
  beforeAwait()
  val resultOfFuture2 = future2.await
  afterAwait()
  "result of future1"
}
{% endhighlight %}

大体相当于

{% highlight scala %}
val future1 = new Future.Stateless {
  // 注意：真正的Future宏生成onComplete而非foreach。
  // 为了简化代码，本示例使用foreach代替onComplete，相当于不支持尾调用功能的onComplete
  def foreach(handler: String => Unit)(implicit catcher: Catcher[Unit]) = {
    beforeAwait()
    // 注意：真正的Future宏使用onComplete而非foreach。
    future2.foreach { resultOfFuture2 =>
      afterAwait()
      handler("result of future1")
    }
  }
}
{% endhighlight %}

要获取`future1`的结果时，类似`for (resultOfFuture1 <- future1) { println(resultOfFuture1) }`的代码会被编译器转换成`future1.foreach { resultOfFuture1 -> println(resultOfFuture1) }`[^for]。所以，`future1`内，`future2.await`上方的代码`beforeAwait()`，就会在当前线程立即执行。这样就得到了第一个结论：**[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏内的首处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)以前的所有代码，都在当前线程立即执行。**

而`afterAwait()`位于`future2.await`，所以会被[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏捕获到回调函数中，由于这个回调函数作为参数传给`future2`，所以回调函数什么时候触发，在哪个线程中触发，就由`future2`决定了。这样就得到了第二个结论：**[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏内每一处`xxxFuture.await`以后的代码，由`xxxFuture`决定在哪个线程中执行。**

最后，注意[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏生成了`handler("result of future1")`用来触发`handler`以告知`future1`的结果。这行代码和`afterAwait()`一样，也位于回调函数中。这样就得到了第三个结论：**[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏内最后一处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)决定了[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏的结果在哪个线程中处理。**

总结[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏对线程模型的约定如下：

 1. [Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏内的首处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)以前的所有代码，都在当前线程立即执行。
 2. [Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏内每一处`xxxFuture.await`以后的代码，由`xxxFuture`决定在哪个线程中执行。
 3. [Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏内最后一处[await]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)决定了[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏的结果在哪个线程中处理。

---

[^for]: `for`语句到`foreach`方法调用的转换详见[Scala语言规范](http://www.scala-lang.org/docu/files/ScalaReference.pdf) 6.19 For Comprehensions and For Loops。

