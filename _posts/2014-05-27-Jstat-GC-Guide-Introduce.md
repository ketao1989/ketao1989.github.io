---
layout: post
categories: 
  - GC
title:  "Java 查看系统GC命令介绍"
date: 2014-05-27 22:21:35 +0800
comments: true
---

## <a id="Intro">前言</a>

使用	`JVM`的人都或多或少的了解`垃圾回收`机制，当系统的服务出现性能问题时，都会去服务器上查看下系统`GC`的情况。此外，如果有新的服务上线，也需要去服务器上查看下新服务的整体`GC`水平，这就可以使用`jstat`命令来查看了，当然你也可以使用其他方式。
	

## <a id="Jstat">Jstat 查看系统 GC 命令介绍</a>

`jstat`的命令查看系统`GC`情况，很简单，只需要先通过`jps`或者`ps -aux |grep tomcat`来查看对应服务所在的进程数，然后使用下面命令查看。

>> jstat 查看 gc 使用方法如下：  

``` bash

jstat -gcutil `线程值` `间隔时间数(ms)`
```

>> 执行命令`sudo jstat -gcutil 11694 3600`，结果会显示内存各个区域大小的情况，如图：

<img src="/images/2014/05/jstat-gc.jpg" />

>> Note: 结果里面列出每个区间的内存大小，新生代gc的次数和时间，老年代gc的次数和时间。	

	1. S0  — Heap上的 Survivor space 0 区已使用空间的百分比;
	2. S1  — Heap上的 Survivor space 1 区已使用空间的百分比;
	3. E   — Heap上的 Eden space 区已使用空间的百分比;
	4. O   — Heap上的 Old space 区已使用空间的百分比;
	5. P   — Perm space 区已使用空间的百分比;
	6. YGC — 从应用程序启动到采样时发生 Young GC 的次数;
	7. YGCT– 从应用程序启动到采样时 Young GC 所用的时间(单位秒);
	8. FGC — 从应用程序启动到采样时发生 Full GC 的次数;
	9. FGCT– 从应用程序启动到采样时 Full GC 所用的时间(单位秒);
	10. GCT — 从应用程序启动到采样时用于垃圾回收的总时间(单位秒).

<!-- more -->

## <a id="Jmap">Jmap 查看系统 GC 命令介绍</a>

当我们使用`jstat`命令查看`GC`时，发现`YoungGC`或者`FullGC`频率过高时，就需要分析服务的具体情况了。这个时候，可以使用`Jmap`来查看当前系统的实例使用内存大小，一般地，需要`dump`到本地进行分析，使用可视化工具可以使分析效率得到很大的提高。

>> `Jmap`的使用方法如下所示：  

``` bash
jmap -dump:format=b,file=log.bin `进程id`    
```

>> 使用`sudo jmap -dump:format=b,file=log.bin  -F 11694`命令，写入文件内容如下:
    
	JAVA PROFILE 1.0.1
	JAVA PROFILE 1.0.1
	Lcom/alibaba/dubbo/remoting/exchange/support/DefaultFuture$RemotingInvocationTimeoutScan;
	DefaultStatistics.java
	T8procedureNameRs
	StringUtils.10
	isCons
	(Ljava/lang/Integer;)Ljava/math/BigDecimal;
	HtheCloseBracketFor
	h([Ljava/lang/String;Lorg/codehaus/jackson/JsonGenerator;Lorg/codehaus/jackson/map/SerializerProvider;Lorg/codehaus/jackson/map/JsonSerializer;)V
	zC@roureq
	lastActiveFilter
	Ampersand
	......

>> Note：有很多可视化的工具来分析`dump`文件，为了简单，这里使用java自带的工具`jvisualvm`工具，在`文件`选项中`装入` dump 下来的 文件`log.bin`，就可以看到各个对象实例所占用的内存大小和实例数量了。如下图：

<img src="/images/2014/05/jmap-gc.jpg" />

## <a id="Finally">小结</a>

java 中 还有一个命令比较有用，就是`jstack`，该命令可以查看服务器上的某个服务的堆栈信息。从而分析线程是否处于死锁状态，从而导致服务不可用或者服务性能受到影响。使用命令`jstack 线程id`就可以获取所有堆栈信息了，非常简单便捷。
