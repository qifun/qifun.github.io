---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（二）欠条"
categories: [程序设计]
tags: [scala,stateless-future,异步,欠条,for,comprehension]
description: "<code>Future</code>对象，就是将来兑现的<strong>欠条</strong>。"
---

**Future**是指尚未完成的操作。在异步编程中，这些操作结果将来会在操作完成后，再传给回调函数。换句话说，一个[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package.html#Future[+AwaitResult]:Future[AwaitResult])对象，就是一张将来可能兑现的**欠条**。

在Stateless Future框架中，欠条对应类型为[com.qifun.statelessFuture.Future[AwaitResult]]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package.html#Future[+AwaitResult]:Future[AwaitResult])。其中，`AwaitResult`类型参数表示，欠条兑现时，将会给付的结果类型。

## 兑现欠条

有两种语法等待一张欠条兑现，[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)语法和`for`推导式[^ForComprehension]语法。

### 用[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)等待欠条兑现

[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)方法的语义是暂停执行流，一直等到欠条兑现才返回，其返回值就是欠条兑现的结果。我在前一节中讲过，[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)会被[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏替换成[onComplete]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#onComplete((AwaitResult)⇒TailRec[TailRecResult])(Catcher[TailRec[TailRecResult]]):TailRec[TailRecResult])，并不真正阻塞线程：

{% highlight scala %}
Nio2Future.connect(socket, new InetSocketAddress("www.qifun.com", 80)).await
{% endhighlight %}

{% highlight scala %}
readAll(socket, response).await
{% endhighlight %}

{% highlight scala %}
writeAll(socket, request).await
{% endhighlight %}

在上述例子中，[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)表示顺序等待操作，必须出现在[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)块中。如果在[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)外使用[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)，就会导致``` `await` must be enclosed in a `Future` block```的编译错误。

#### 嵌套函数

需要注意，在[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)块中的内嵌函数中，一般也不能用[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)：

{% highlight scala %}
val myFuture = Future {
  def nestedFunction() = {
    anotherFuture.await // 编译错误！
  }
  nestedFunction()
}
{% endhighlight %}

幸好[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package.html#Future[+AwaitResult]:Future[AwaitResult])可以嵌套，所以你可以这样写：

{% highlight scala %}
val myFuture = Future {
  def nestedFunction() = Future {
    anotherFuture.await // 编译通过
  }
  nestedFunction().await
}
{% endhighlight %}

### 用`for`等待欠条兑现

欠条兑现的另一语法借用了Scala的`for`关键字：

{% highlight scala %}
println("即将开始等待myFuture")
for (result <- myFuture) {
  println(s"结果是：$result")
}
println("正在等待myFuture……")
{% endhighlight %}

注意，此处的`for`语句并不是循环！`for`代码块内是欠条兑现后才触发的代码，通常只触发一次。而`for`代码块之后的代码则在开始等待`myFuture`以后，马上执行。


所以，大多数情况下，上述代码的输出顺序将会类似这样：

    即将开始等待myFuture
    正在等待myFuture……
    结果是：<myFuture兑现后的结果>

#### 等待多张欠条

还可以用`for`语句顺序等待多张欠条：

{% highlight scala %}
println("即将开始等待myFuture1")
for (result1 <- myFuture1; result2 <- getMyFuture2(result1)) {
  println(s"结果是：$result1、$result2")
}
println("正在等待myFuture1和getMyFuture2(result1)……")
{% endhighlight %}

上述代码中，两张欠条依次等待。其中第二张欠条`getMyFuture2(result1)`甚至是利用第一张欠条的兑现结果`result1`才能计算得到。

#### 欠条的映射和合并

你也可以用`for`/`yield`语法，把多张欠条合成一张：

{% highlight scala %}
case class MyResult(result1: Result1, result2: Result2)

val myFuture3: Future[MyResult] = for (result1 <- myFuture1; result2 <- getMyFuture2(result1)) yield {
  MyResult(result1, result2)
}
{% endhighlight %}

用`for`/`yield`创建的欠条，属于**无状态欠条**，支持**惰性执行**。创建这张新欠条时并不会发起`myFuture`或`getMyFuture2(result1)`所对应的异步操作，而要等到对新欠条调用[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)或者不含`yield`的`for`时，才会发起操作：

{% highlight scala %}
for (result3 <- myFuture3) {
  println(s"最终结果是：${result3.result1}、${result3.result2}")
}
{% endhighlight %}

当然，你也可以用前面学过的[Future]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)/[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)来合并欠条。以下代码与上方的`for`/`yield`示例，功能完全相同：

{% highlight scala %}
case class MyResult(result1: Result1, result2: Result2)

val myFuture3: Future[MyResult] = Future[MyResult] {
  val result1 = myFuture1.await
  val result2 = getMyFuture2(result1).await
  MyResult(result1, result2)
}
{% endhighlight %}

（待续。下一节将会讲解无状态欠条和有状态欠条的区别。）

---

[^ForComprehension]: **`for`推导式**即**For Comprehension**。Scala的`for`推导式语法类似命令式语言的`for`循环，但语义却更接近Ruby、Python等语言的列表推导式（List Comprehension）。
