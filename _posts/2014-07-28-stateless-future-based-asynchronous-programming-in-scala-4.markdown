---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（四）<code>await</code>、<code>for</code>和<code>yield</code>"
categories: [程序设计]
tags: [scala,stateless-future,异步,for,yield,await]
description: "<code>futureSeq</code>函数可以解决<code>await</code>与<code>for</code>的冲突。"
---

## `for`与[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)的冲突

Scala提议标准`scala.async`有个缺陷：不支持`for`循环[^Issue32]。比如，类似这样的代码无法编译：

{% highlight scala %}
import scala.async.Async._
val xs = Seq(1, 2, 3)
async {
  for (x <- xs) {
    await(async(x * 2)) // 编译错误
  }
}
{% endhighlight %}

我们的Stateless Future也存在类似的问题，以下代码也不能编译：

{% highlight scala %}
import com.qifun.statelessFuture.Future
val xs = Seq(1, 2, 3)
Future {
  for (x <- xs) {
    Future(x * 2).await // 编译错误
  }
}
{% endhighlight %}

### 冲突的原因

这是因为，Scala语言的`for`语句会被转换成`foreach`、`map`、`flatMap`、`withFilter`函数调用。而`for`的内部语句块会转换成一个函数对象。比如上述代码相当于

{% highlight scala %}
import com.qifun.statelessFuture.Future
val xs = Seq(1, 2, 3)
Future {
  xs.map { x =>
    Future(x * x).await // 编译错误
  }
}
{% endhighlight %}

由于上述代码中的[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)位于匿名函数`{ x => ... }`中，所以`await`与外层`Future`的关联就会阻断，最终导致编译错误。

## [futureSeq]({{ site.baseurl }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/AwaitableSeq$.html#futureSeq%5BA%5D%28TraversableOnce%5BA%5D%29%3AAwaitableSeq%5BA%2CUnit%5D)包装函数

幸好，Stateless Future的工具库中，有个[futureSeq]({{ site.baseurl }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/AwaitableSeq$.html#futureSeq%5BA%5D%28TraversableOnce%5BA%5D%29%3AAwaitableSeq%5BA%2CUnit%5D)函数可以解决[await]({{ site.baseurl }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/Awaitable.html#await:AwaitResult)与`for`的冲突。只要给`xs`包装一层[futureSeq]({{ site.baseurl }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/AwaitableSeq$.html#futureSeq%5BA%5D%28TraversableOnce%5BA%5D%29%3AAwaitableSeq%5BA%2CUnit%5D)就可以了。

{% highlight scala %}
import com.qifun.statelessFuture.Future
import com.qifun.statelessFuture.util.AwaitableSeq.futureSeq
val xs = Seq(1, 2, 3)
Future {
  val xs2 = for (x <- futureSeq(xs)) {
    Future(x * x).await // 编译通过
  }
}
{% endhighlight %}

[futureSeq]({{ site.baseurl }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/AwaitableSeq$.html#futureSeq%5BA%5D%28TraversableOnce%5BA%5D%29%3AAwaitableSeq%5BA%2CUnit%5D) 也支持`for`/`yield`推导式：

{% highlight scala %}
import com.qifun.statelessFuture.Future
import com.qifun.statelessFuture.util.AwaitableSeq.futureSeq
val xs = Seq(1, 2, 3)
Future {
  val xs2 = for (x <- futureSeq(xs)) yield {
    Future(x * x).await // 编译通过
  }
}
{% endhighlight %}


（待续）

---

[^Issue32]:
    Github上的相关问题讨论：[scala/async#32](https://github.com/scala/async/issues/32)