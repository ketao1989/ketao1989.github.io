---
layout: post
categories: 
  - Script
  - Linux
title:  "Linux Awk 使用学习笔记"
date: 2014-05-30 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

`linux Awk` 脚本语言，算来也有很长的一段历史了。在如今各种更全更简单的脚本语言遍地生成的年代，关注`Awk`的新人越来越少了。最近，在项目组同事的影响下，去学习了下`Awk`的基本使用方法。

> Note：本文只是学习笔记，很多示例和描述都来自最后推荐的文章里面。 

起意写关于`Awk`学习笔记的主要原因是，学习皓哥的[AWK 简明教程](http://coolshell.cn/articles/9070.html) 完了之后，完全还是云里雾里的，不知所以然。所以，找了个[AWK 手册](http://www.aslibra.com/doc/awk.htm)好好学习。但是，对于前者介绍太多浅，初学者对于`Awk`使用没有任何概念；后者又过于长，不方便我去查资料，所以合计起来，就写一个学习总结罢了。

## <a id="Console">Awk命令行</a>

关于`Awk`的历史，语言优势什么的，就不浪费时间介绍了。

### 2.1 Awk 程序结构

首先，我们在服务器上查看日志信息，大部分时候都希望使用一行命令搞定问题，因此，显然命令行方式使用`Awk`是很频繁的事情。

``` bash
awk 'awk程序主体' [操作的文本] 
```

在写一个`awk`脚本之前，首先需要的是了解 `awk`程序组成结构，如果你不了解它的程序一般的结构，那么你去看皓哥或者其他人写的入门级`Awk`小程序，你也无法真正去了解或者深入。

	Pattern1 {Actions1}
	Pattern2 { Actions2 }
	......
	Pattern3 { Actions3 }

<!-- more -->

> Note：`Pattern （模式）`，如同正则一样，就是判断`Pattern`是否满足，如果满足，则执行就下来的`Actions`动作行为。`awk`提供的比较运算符和`C`语言一样，不过新加了 ` ~ `(匹配) 及 ` !~ `(不匹配) 二个关系运算符.
	
	其用法与涵义如下:
	若 A 为一字符串, B 为一正则表达式(Regular Expression)
	A ~ B 判断 字符串A 中是否 包含 能匹配(match)B表达式的子字符串.
	A !~ B 判断 字符串A 中是否 不包含 能匹配(match)B表达式的子字符串.

> Note：`Action`，就是动作，但是需要指出的是，一个`Pattern`所对应的`Action`应该要放进`{ }`内。当然，存在一种情况，就是没有`Pattern`,直接就是`{Action}`，这表示，“无条件执行指定的Action动作”。

了解了`awk`程序的基本结构，下面给一个简单示例：

``` bash
awk ' $0>1 {print "hello world"}' 
```

> 这个程序意思就是：`$0>1`模式成立，则打印后面的`hello world`，在console上输入3，回车，则会打印`hello world`；如果输入0，则不会打印该消息。

### 2.2 Awk 程序说明

首先，需要了解`$0`这种格式： `$0`表示`awk`所读入的一行字符串；`$1`则表示所读入的字符串的第一个数据，默认安装`空格/tab`来分隔；`$2`则表示所读入的字符串的第二个数据；以此类推.....

其次，这个程序是从控制台，或者文本中读取变量值（需要在后面写操作的文本名）。如果你测试了这个小脚本，你会发现，它会一直读，知道遇到`ctrl+d`或者`EOF`才会结束。但是，如果你初始化一些变量或者其他只需要执行一次的逻辑，怎么办呢？你需要`BEGIN`和`END`。

	1. BEGIN ：程序一开始执行时执行的，仅被执行一次。
	2. END	 ：程序处理完所有数据即将返回的时候，执行一次。

比如上面的程序，可以写成：

``` bash
awk ' BEGIN{n=3} $0>1 && --n>0 {print "hello world"} END{print "EXIT AWK TEST"}'
```

执行完了结果如下：

	[tao.ke@l-rtools1.ops.cn6 ~]$ awk ' BEGIN{n=3} $0>1 && --n>0 {print "hello world"} END{print "EXIT AWK TEST"}'         
	3
	hello world
	2
	hello world
	1
	0 #（在这里的时候输入 ctrl+d）
	EXIT AWK TEST

最后，说说`Awk`中使用`linux shell`命令。和`linux`一样，使用`|`来操作管道流，使用`""`标识`linux`命令。

``` bash
awk ' BEGIN{while("who"|getline){print $0}} '  
```

> 这里可以打印出，所有的登陆在系统上的用户。	关于`while`的使用，和`c`语言一样；此外，其他控制语言`if`、`do-while`、`for`，也和`c`一样。需要说明的是，`for`可以使用类似`foreach`的形式，具体见下一节。


## <a id="Recommend">推荐教程</a>

* awk手册，<http://www.aslibra.com/doc/awk.htm>

* AWK 简明教程，<http://coolshell.cn/articles/9070.html>
