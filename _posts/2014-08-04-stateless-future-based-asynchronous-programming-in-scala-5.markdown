---
author:
  name: 杨博
  email: pop.atry@gmail.com
  url: "http://www.ac.net.blog.163.com/"
  github: Atry
  bio: 岂凡 游戏架构师
  email_md5: 5b3c5026fee43baef3b15d7fef166a7e
layout: post
title: "基于Stateless Future的Scala异步编程（五）发生器"
categories: [程序设计]
tags: [scala,stateless-future,异步,for,yield,generator,发生器]
description: "发生器功能类似Scala的<code>for</code>/<code>yield</code>推导式，但发生器比<code>for</code>/<code>yield</code>推导式更灵活。"
---

Stateless Future支持发生器[^Generator]。发生器功能类似Scala的`for`/`yield`推导式，但发生器比`for`/`yield`推导式更灵活。你可以用Stateless Future生成很复杂的惰性求值容器。

## 用法

首先，想好你的容器中的元素类型，创建一个[Generator]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/Generator.html)：

{% highlight scala %}
import com.qifun.statelessFuture.util.Generator
val gen = Generator[Int] // 创建用来生成Seq[Int]的发生器
{% endhighlight %}

然后创建`gen.Future`：

{% highlight scala %}
val genFuture = gen.Future {
  for (i <- gen.futureSeq(0 until 3)) {
    gen(i * i).await // 生成容器中的一项，值是i*i
  }
}
{% endhighlight %}

`gen.Future`是一种特殊的`Future`，专门用于发生器。`gen.Future`与[com.qifun.statelessFuture.Future]({{ site.BASE_PATH }}/stateless-future/0.3.1-SNAPSHOT/api/com/qifun/statelessFuture/package$$Future$.html)的类型有微妙的区别，相互之间不能隐式转换。`gen.futureSeq`则类似[上一节]({{ site.BASE_PATH }}/2014/07/28/stateless-future-based-asynchronous-programming-in-scala-4/)中介绍的[futureSeq]({{ site.BASE_PATH }}/stateless-future-util/0.5.0-SNAPSHOT/api/com/qifun/statelessFuture/util/AwaitableSeq$.html#futureSeq%5BA%5D%28TraversableOnce%5BA%5D%29%3AAwaitableSeq%5BA%2CUnit%5D)，可以配合`gen.Future`使用。

最后把`genFuture`转换成`gen.OutputSeq`：

{% highlight scala %}
val genSeq: gen.OutputSeq = genFuture
{% endhighlight %}

`gen.OutputSeq`派生于`Seq`。`genSeq`是个惰性求值的容器，内容是`Seq(0, 1, 4, 9)`。

### 等价的`for`/`yield`推导式

这种惰性求值容器也可以用普通的`for`/`yield`推导式生成：

{% highlight scala %}
val forYieldSeq: Seq[Int] =
  for (i <- (0 until 10).view) yield { // view后缀使得for语句返回支持惰性求值的SeqView，而不是普通的Seq
    i * i
  }
{% endhighlight %}

`forYieldSeq`也是个惰性求值的容器，内容也是`Seq(0, 1, 4, 9)`。

## 进阶用法

虽然上例中的`for`/`yield`也能涵盖发生器最简单的用法，但是除此之外，发生器中还可以容纳任意控制语句，这种进阶用法就没办法靠`for`/`yield`实现了。

{% highlight scala %}
import com.qifun.statelessFuture.util.Generator
val gen = Generator[Int]
val genSeq2: gen.OutputSeq = gen.Future {
  gen(-1).await
  for (i <- gen.futureSeq(0 until 3)) {
    gen(i * i).await
    if (i == 2) {
      gen(i).await
      gen(i).await
      gen(i).await
    }
  }
  gen(-2, -3, -2).await // 在一行代码中生成多个元素
}
{% endhighlight %}

`genSeq2`的值是`Seq(-1, 0, 1, 4, 2, 2, 2, 9, -2, -3, -2)`，这样灵活的控制流程，比`for`/`yield`功能强。

（待续）

---

[^Generator]: 
    发生器
    : 即Generator，是用来控制循环迭代行为的一种特殊子例程。例如C#的`yield`功能就是发生器。参见[维基百科](http://en.wikipedia.org/wiki/Generator_%28computer_programming%29)。
