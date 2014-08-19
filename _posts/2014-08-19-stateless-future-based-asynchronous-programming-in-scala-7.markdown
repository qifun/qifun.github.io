---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（七）<code>JumpInto</code>和<code>Sleep</code>"
categories: [程序设计]
tags: [scala,stateless-future,异步,thread,线程,sleep,线程池,thread-pool]
description: "基于<code>Future</code>宏的线程约定，Stateless Future的工具库额外提供了几个类，帮助程序员控制线程模型。"
---

基于[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏的线程约定，Stateless Future的工具库额外提供了几个类，帮助程序员控制线程模型。

## [JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)

[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)让[Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)宏中的代码切到指定的线程池中运行。

{% highlight scala %}
import java.util.concurrent.Executors
val executor = Executors.newSingleThreadExecutor // 创建单线程的线程池

import com.qifun.statelessFuture.Future
val future1 = Future[String] {
  println(s"Current thread name is ${Thread.currentThread.getName} before JumpInto.")
  import com.qifun.statelessFuture.util.JumpInto
  JumpInto(executor).await // 切换到线程池executor
  println(s"Current thread name is ${Thread.currentThread.getName} after JumpInto.")
  "RESULT"
}

println("Before starting future1")

implicit def catcher = PartialFunction.empty
for (result <- future1) {
  println(s"Result of future1 is $result.")
}

println("After starting future1")
{% endhighlight %}

以上代码将会输出

	Before starting future1
	Current thread name is run-main-0 before JumpInto.
	After starting future1
	Current thread name is pool-15-thread-1 after JumpInto.
	Result of future1 is RESULT.

从输出可以知道，调用[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)以前的代码运行在`run-main-0`线程，而调用[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)以后的代码运行在`pool-15-thread-1`线程。

用[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)配合单线程的线程池，就构成了一个轻量级的消息队列，可以保证跳入了同一个线程池的代码都顺序进行，不会相互打断。[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)常常可以用来代替`synchronized`锁。

## [Sleep]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/Sleep$.html)

[Sleep]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/Sleep$.html)类似[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)，也能切换线程池。但[Sleep]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/Sleep$.html)会延后一段时间才继续执行后续代码，而不像[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)那样立即切换线程。


{% highlight scala %}
import java.util.concurrent.Executors
// 创建支持延后执行服务的单线程线程池
val executor = Executors.newSingleThreadScheduledExecutor

import com.qifun.statelessFuture.Future
val future1 = Future[String] {
  println(s"Current thread name is ${Thread.currentThread.getName} before Sleep.")
  import com.qifun.statelessFuture.util.Sleep
  import scala.concurrent.duration._
  Sleep.apply(executor, 2.seconds).await // 2秒后切换到线程池executor
  println(s"Current thread name is ${Thread.currentThread.getName} after Sleep.")
  "RESULT"
}

println("Before starting future1")

implicit def catcher = PartialFunction.empty
for (result <- future1) {
  println(s"Result of future1 is $result.")
}

println("After starting future1")
{% endhighlight %}

输出和[JumpInto]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/JumpInto.html)的例子差不多


	Before starting future1
	Current thread name is run-main-0 before Sleep.
	After starting future1
	Current thread name is pool-13-thread-1 after Sleep.
	Result of future1 is RESULT.


区别在于[Sleep]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/Sleep$.html)的例子会在输出`After starting future1`后停顿两秒，然后再继续输出剩下的信息。
