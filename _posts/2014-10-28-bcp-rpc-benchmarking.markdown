---
author:
  name: 方里权
  email: fangliquan@qq.com
  url: 
  github: chank
  bio: 岂凡 软件工程师
  email_md5: 5851b66b5708deba39cb20fca4adf312
layout: post
title: "岂凡游戏服务器引擎性能评测"
categories: [性能评测]
tags: [scala,benchmarking]
description: "我们岂凡开发的游戏服务器引擎是一款高性能的引擎，可支持上万客户端同时在线！"
---

本文详细分析了我们岂凡开发的[游戏服务器引擎](https://github.com/qifun)的性能评测数据和修改的bug。

## 一、scala跟csharp单机性能评测

### 1. 测试方法

用scala做服务端，csharp做客户端单机连调测试，客户端每隔一段随机时间往服务端发一个ping请求，服务端返回一个pong响应。

### 2. 测试数据

<table border="1">
		<tr>
				<td>登录新客户端时间(毫秒)</td>
				<td>最大思考时间[^MaxThinkTime](毫秒)</td>
				<td>开始内存(千兆)</td>
				<td>结束内存(千兆)</td>
				<td>现象</td>
		</tr>
		<tr>
				<td>100</td>
				<td>2000</td>
				<td>4.53</td>
				<td>6</td>
				<td>5400个左右的客户端开始出现连接被关闭等异常</td>
		</tr>
		<tr>
				<td>10</td>
				<td>2000</td>
				<td>4.57</td>
				<td>5.86</td>
				<td>5900个左右的客户端开始出现连接被关闭等异常</td>
		</tr>
		<tr>
				<td>10</td>
				<td>1000</td>
				<td>4.61</td>
				<td>5.95</td>
				<td>4400个左右的客户端开始出现连接被关闭等异常</td>
		</tr>
		<tr>
				<td>10</td>
				<td>10000</td>
				<td>4.62</td>
				<td>5.29</td>
				<td>10900个左右的客户端开始出现连接被关闭等异常</td>
		</tr>
</table>

### 3. 测试结果

使用jvisualvm 性能分析工具发现，scala服务端的IO、HaxeException、resetHeartBeatTimer占用CPU较高，HaxeException大概占用了6%左右的CPU时间，而resetHeartBeatTimer大概占用了10%左右的CPU时间。

HaxeException占用CPU较高是因为Haxe底层解析的问题，经更改底层Haxe代码后HaxeException几乎不占CPU时间。

resetHeartBeatTimer占用CPU较高说明频繁取消并创建定时器是比较耗时的操作，以后可通过让所有BcpSession共用一个HeartBeat定时器来优化性能。

上述测试的IOPS并不高主要是因为这是单机连调并且由于csharp那边实现的性能有点低导致的（后来发现是由于许多bug导致的，修复之后性能就有了质的提升了）。通过看CPU占用发现csharp的CPU占用大概是scala的两倍左右。

## 二、scala跟scala单机性能评测

### 1. 测试方法

用scala做服务端跟客户端，客户端每隔一段固定时间往服务端发一个ping请求，服务端返回一个pong响应。

### 2. 高压测试并修正bug

在高压力下做性能评测时，发现并修复了很多bug。

由于对ScalaSTM的使用不当，在高压力下，ScalaSTM触发了很多bug。 ScalaSTM为了让嵌套的atomic事务更加高效廉价，ScalaSTM会试图把所有嵌套的事务整合成一个事务。如果内层的atomic事务发生了异常回滚，在这种机制下，外层的atomic事务也会被回滚，可以看出ScalaSTM是不会只回滚内部嵌套的atomic事务的。这时候如果atomic事务里面的回调函数也嵌套了atomic事务，在高压下就会很危险了。

而且，在ScalaSTM中做I/O操作和使用定时器都得很小心。在高压测试时，就发现了由于在发送request请求时没有把I/O放到afterCommit钩子中来做而导致服务器性能提高不上去的情况。并且由于在ScalaSTM中没有把定时器的定义和启动分开，也导致了服务器的性能提高不上去。后来把定时器的定义放在atomic事务内做，定时器的启动在atomic事务提交后来做服务器性能就好很多了。因此，对ScalaSTM的使用必须得格外地小心。

在压测时，发现客户端数量在高压下计数会出现错误。因为之前使用定时器和ScalaSTM时出了很多问题，所以刚开始查该问题还是把重点放在了定时器和ScalaSTM的使用上，后来在写stateless-future-util的单元测试用例时才发现定时器没有问题。问题在于在启动一个client并把该client加到clients中时clients忘了加锁导致的。

在压测时，由于主要是调服务端的性能，对客户端的性能就没怎么在意。后来真的感觉客户端性能有点差，然后用jvisualvm一看，吓了一跳，客户端最多竟然开了700多个线程，难怪性能低了。查看线程在做什么之后发现，是由于AsynchronousSocketChannel.open没有使用ChannelGroup引起客户端大量创建线程，导致性能提不上去的。

通过做性能评测，也修复了其他很多bug。例如，BcpClient状态设置错误bug、RetrasmissionFinish会重复清理数据bug和接收数据在收到异常时没有Interrupt掉低层链接bug等等。

### 3. 测试数据

经过上面漫长艰苦的修bug之后，服务器稳定了很多并且性能有了非常大的提升。下面图表就是单机性能评测的结果：

<table border="1">
		<tr>
				<td>新登客户端时间(毫秒)</td>
				<td>发送数据间隔(毫秒)</td>
				<td>请求数据包大小(字节)</td>
				<td>响应数据包大小(字节)</td>
				<td>心跳间隔(秒)</td>
				<td>现象</td>
		</tr>
		<tr>
				<td>50</td>
				<td>500</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多6200个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>1000</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多11600个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多22700个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>3000</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多26400个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>1000</td>
				<td>164</td>
				<td>71</td>
				<td>6</td>
				<td>最多11500个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>0.5</td>
				<td>最多8180个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>1</td>
				<td>最多11500个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>1.5</td>
				<td>最多13500个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>1000</td>
				<td>356</td>
				<td>151</td>
				<td>3</td>
				<td>最多10800个左右的客户端</td>
		</tr>
		<tr>
				<td>50</td>
				<td>2000</td>
				<td>356</td>
				<td>151</td>
				<td>3</td>
				<td>最多19100个左右的客户端</td>
		</tr>
</table>

可以看出，IOPS是可以达到12000左右的。每个客户端发送数据间隔在1s时可以支持上万客户端链接，而2s时可以支持两万以上的客户端链接。

在发送数据的频率大于心跳频率时，减小心跳频率并不能提升服务器性能。因为定时器的性能瓶颈是在创建和取消定时器时，如果数据发送频率大，我们现在是会频繁地创建和取消定时器，减小心跳频率也就没有意义了。后面会优化掉我们的心跳机制，到时服务器性能应该还会有8%~10%的提升的。

在发送数据的频率小于心跳频率时，减小心跳频率才可以提升服务器的性能。

加大数据包的大小性能会稍微下降一点点，这是因为解包花了更多的CPU时间。

## 三、scala跟scala多机性能评测

### 1. 测试方法

用scala做服务端跟客户端，使用了一台机器做服务端，两台机器做客户端。客户端每隔一段固定时间往服务端发一个ping请求，服务端返回一个pong响应。

### 2. 测试数据

多机性能评测结果：

<table border="1">
		<tr>
				<td>发送数据间隔(毫秒)</td>
				<td>请求数据包大小(字节)</td>
				<td>响应数据包大小(字节)</td>
				<td>心跳间隔(秒)</td>
				<td>现象</td>
		</tr>
		<tr>
				<td>500</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多6100个左右的客户端</td>
		</tr>
		<tr>
				<td>1000</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多10800个左右的客户端</td>
		</tr>
		<tr>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>3</td>
				<td>最多19400个左右的客户端</td>
		</tr>
		<tr>
				<td>1000</td>
				<td>164</td>
				<td>71</td>
				<td>6</td>
				<td>最多9700个左右的客户端</td>
		</tr>
		<tr>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>0.5</td>
				<td>最多8200个左右的客户端</td>
		</tr>
		<tr>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>1</td>
				<td>最多12100个左右的客户端</td>
		</tr>
		<tr>
				<td>2000</td>
				<td>164</td>
				<td>71</td>
				<td>1.5</td>
				<td>最多15100个左右的客户端</td>
		</tr>
		<tr>
				<td>1000</td>
				<td>356</td>
				<td>151</td>
				<td>3</td>
				<td>最多10050个左右的客户端</td>
		</tr>
		<tr>
				<td>2000</td>
				<td>356</td>
				<td>151</td>
				<td>3</td>
				<td>最多19500个左右的客户端</td>
		</tr>
</table>

可以看出，在联网的情况下也可以达到11000左右的IOPS，CPU基本上都能跑满。性能比单机下稍微差了一点点，主要是耗在网络IO上了。

另外在测网络流量时，如果有5000个客户端，每个客户端1s发送一次请求，那么服务端接收数据会稳定在759000 byte/s，发送数据会稳定在282000 byte/s。

平均接收一个包的大小为：( 759000 byte/s * 1s ) / 5000 = 151.8 byte

平均发送一个包的大小为：( 282000 byte/s * 1s ) / 5000 = 56.4 byte

因此网络流量正常。

[^MaxThinkTime]:
		最大思考时间
		: 客户端发送请求的时间为10m到最大思考时间之间的随机间隔

