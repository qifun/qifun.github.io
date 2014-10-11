---
author:
  name: 小绽
  email: yangbo@qifun.com
  url: "http://qforce.qifun.com/"
  github: qifun
  bio: 岂凡技术小站
  email_md5: 50993397b1b5ed2eae8e50e25d9663e8 
layout: post
title: "sbt-haxe，开源！"
categories: [岂凡开源项目]
tags: [岂凡,开源,sbt,haxe,java,scala]
description: 岂凡网络开源了sbt-haxe 1.0.0，一个把Haxe编译为Java的Sbt插件。
---
今天，岂凡网络开源了[sbt-haxe](https://bitbucket.org/qforce/sbt-haxe) 1.0.0，把Haxe编译为Java的Sbt插件。

## 用法

### 第一步：在你的Sbt项目中安装`sbt-haxe`

在`project/plugins.sbt`中加入以下代码：

    addSbtPlugin("com.qifun" % "sbt-haxe" % "1.0.0")

然后在`build.sbt`中增加`haxeSettings`：

    haxeSettings

### 第二步：创建Haxe源文件`src/haxe/yourPackage/YourHaxeClass.hx`

{% highlight haxe %}
package yourPackage;
import haxe.ds.Vector;
class YourHaxeClass
{
  public static function main(args:Vector<String>)
  {
    trace("Hello, World!");
  }
}
{% endhighlight %}

### 第三步：运行！

    $ sbt run
    [info] Loading global plugins from C:\Users\user\.sbt\0.13\plugins
    [info] Loading project definition from D:\Documents\sbt-haxe-test\project
    [info] Set current project to sbt-haxe-test (in build file:/D:/Documents/sbt-haxe-test/)
    [info] "haxe" "-cp" "D:\Documents\sbt-haxe-test\src\haxe" "-cp" "D:\Documents\sbt-haxe-test\target\scala-2.10\src_managed\haxe" "-java-lib" "C:\Users\user\.sbt\boot\scala-2.10.3\lib\scala-library.jar" "-java" "D:\cygwin\tmp\sbt_97a26bd9" "-D" "no-compilation" "yourPackage.YourHaxeClass"
    [info] Compiling 1 Java source to D:\Documents\sbt-haxe-test\target\scala-2.10\classes...
    [info] Running yourPackage.YourHaxeClass
    YourHaxeClass.hx:7: Hello, World!
    [success] Total time: 1 s, completed 2014-7-25 10:00:23

## 任务项和配置项

`sbt-haxe`提供了以下任务项和配置项：

 * haxe
 * dox
 * haxeCommand
 * haxelibCommand
 * doxPlatforms

欲知上述任务项和配置项的详情，请参见[src/main/scala/com/qifun/sbtHaxe/HaxePlugin.scala](https://bitbucket.org/qforce/sbt-haxe/src/master/src/main/scala/com/qifun/sbtHaxe/HaxePlugin.scala#cl-34)。

## 依赖项目

`sbt-haxe`需要Sbt 0.13、Haxe 3.1、[hxjava](http://lib.haxe.org/p/hxjava) 3.1.0、[Dox](http://lib.haxe.org/p/dox) 1.0.0。
