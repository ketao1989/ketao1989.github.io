---
layout: post  
categories: 
  - Redis
  - Redis Cookbook
title: "Redis Cookbook 之 Redis 键值对存储服务"
date: 2014-08-10 22:21:35 +0800
comments: true
---

## <a id="Problem">问题</a>

许多的应用服务都需要存储关于使用，配置或者其他相关的信息的临时数据，这些数据结构并不适合存储在关系数据库中。
通常情况下，开发者会是MySQL或者其他RDBMS来存储这种数据。在这里，我们将使用Redis和它内建的数据结构，以一种更轻量级、更快速、更宽松的方式完成存储应用临时数据的功能。

## <a id="Solution">解决方法</a>

Redis本身的定位并不仅仅是`key/value`存储服务，同时也可以作为存储数据结构的服务器。这意味着，在传统的`key/value`存储功能之上，它还提供给你一些存储和操作应用数据的方式。
我们将使用这些数据结构和命令来存储应用示例数据：比如，我们在一些`regular keys`里面存储有用的计数器，在`Redis hashes`里面存储用户对象，以及使用`sets`实现朋友圈（像Google+）。

## <a id="Discussion">讨论</a>

### 存储应用使用计数器

首先，让我们开始存储一些非常基础的事情：计数器。想象我们运营一个商业的社交网络，然后想要追踪`profile/page`的访问数据。我们可以在RDBMS的存储我们页数据的表里面增加一列，
但是希望我们的流量足够高以至于每次更新这一列会带来一些麻烦。因此，我们需要更快的工具来更新和查询，所以我们使用Redis来代替。

由于Redis命令的原子性，我们知道如果我们存储一个计数器的key，则我们可以使用命令，比如：`INCR(或者INCRBY)和DECR(或者DECRBY)`，来增加增加或者减少它包含的值。所以，通过为我们的数据设计一个合适的命名保证我们的计数器单次操作成本微乎其微。

Redis系统内实际上并没有约定的方法来组织keys，但是许多人（包括作者）都喜欢使用`冒号：`来区分关键字从而创建keys，因此在这里也这样约定。为了存储我们的社交网络页面访问数据，我们可以有一个像`visits:pageid:totals`的命名，比如页面id为635，则命名为`visits:635:totals`。如果我们已经在一些地方存储访问数据，我们可以首先根据这些数据产生redis的keys，并且设置对应的值，比如：

		
		SET visits:1:totals 21389 
		SET visits:2:totals 1367894 
		(...)

当访问一个给定的页面，一个简单地`INCR`命令将更新Redis中的计数器：
	
		INCR visits:635:totals
		
然后我们可以获取任何页面在任何时候的访问次数，这只需要通过简单地`GET`命令：
		
		GET visits:635:totals
		
你也可以让你的命令更智能，比如你可以在人们查看页面的时候，显示当前该页面被访问的次数给他看。当然，你也可以计算他自己访问的次数，所以你甚至不需要执行最后的`GET`命令：你可以利用`INCR`命令返回值的特点，因为`INCR`命令会返回自增之后的计数值。一个简单地关于访问和计数器的伪代码如下所示：

		1. 访问者访问页面.
		2. 我们 INCR 相关页面的访问计数器 (比如：INCR visits:635:totals）		3. 我们获取INCR命令的返回值。
		4. 我们展示返回的值在用户页面上。

这种方式我们保证用户在查看页面的时候，常常可以看见实时计数值，以及他自己访问的计数--着所有都可以使用Redis命令。

<!-- more -->

### 存储对象数据在hashes内

Redis关于hashes的实现对于存储对象数据应用来说是一个很好的方案。在下面的例子里，我们将调查我们怎样使用`hashes`来在给定系统上存储用户相关的信息。

我们将开始设计一个key命名来存储我们的用户。在此之前，我们使用分号来分割我们的关键字，进而产生意义丰富的key。为了达到这个案例的目的，我们将构建一个简单的key，形如：`users:alias`，其中`alias`是二进制安全的字符串。所以，为了存储一个叫做`John Doe`的用户，我们将建一个hash叫做`users:jdoe`。

我们也假设我们将要存储关于用户的许多属性，比如全名、邮件地址、电话号码、以及访问我们英语的次数。我们将使用Redis的`hash管理命令`--像`HSET`、`HGET`和`HINCRBY`--存储用户的这些信息。

		redis> hset users:jdoe name "John Doe" 
		(integer) 1
		
		redis> hset users:jdoe email "jdoe@test.com" 
		(integer) 1
		
		redis> hset users:jdoe phone "+1555313940" 
		(integer) 1
		
		redis> hincrby users:jdoe visits 1 
		(integer) 1


随着我们hash的建立，我们可以在合适的地方通过`HGET`获取简单地属性值或者通过`HGETALL`命令所有hash对象数据，如同下面的示例：


		redis> hget users:jdoe email 
		"jdoe@test.com"

		redis> hgetall users:jdoe
		1) "name"
		2) "John Doe"
		3) "email"
		4) "jdoe@test.com" 5) "phone"
		6) "+1555313940" 7) "visits"
		8) "1"

还有辅助的命令，比如`HKEYS`，该命令返回一个特定的hash里面存储的所有keys；`HVALS`，该命令仅仅返回values。当从你应用的Redis中检索数据时，你可能会发现使用`HGETALL`或者其他命令很有用，这其实取决于你想怎样检索你的数据。

		redis> hkeys users:jdoe 
		1) "name"
		2) "email"
		3) "phone"
		4) "visits"

		redis> hvals users:jdoe 
		1) "John Doe"
		2) "jdoe@test.com"
		3) "+1555313940"
		4) "1"

关于我们用户hash的其他命令列表，你可以详细阅读 `Redis official documentation for hash commands`，这里面包含他自己的使用`hashes`管理数据的示例。


###  使用sets存储用户的朋友圈

为了完成Redis特有的方式存储数据，我们看看怎么样使用Redis支持的sets来创建和google+相似的朋友圈功能。sets天生就适应朋友圈，因为sets代码数据的集合，并且有一些天生的功能来做一些有趣的事情，比如交集和并集。

首先，让我们定义一个朋友圈的命名。我们想要为我们每一个用户存储一些朋友圈，所以我们的key需要包含用户和实际的朋友圈。举个例子，`John Doe`的家庭圈可能有一个key像`circle:jdoe:family`样子。相似地，他的足球训练朋友可能在key为`circle:jdoe:soccer`的set里面。这里没有key设计的设置规则，所以常常以一种对我们应用有意义的方式来设计它们。

现在，我们知道存储我们sets里面的keys，接下来，我们来创建`John Doe`的家庭集和足球朋友集。在集合本身里面，我们可以通过用户id来获取Redis里其他的keys。如果我们想要获取属于`John Doe`家庭圈用户的列表，和它们的用户信息，我们可以使用我们set的操作结果，然后为每一个用户提取真实的hashes（使用hashes来存储用户信息）。

		redis> sadd circle:jdoe:family users:anna 
		(integer) 1

		redis> sadd circle:jdoe:family users:richard
		(integer) 1 
		
		redis> sadd circle:jdoe:family users:mike
		(integer) 1 
		
		redis> sadd circle:jdoe:soccer users:mike
		(integer) 1 
		
		redis> sadd circle:jdoe:soccer users:adam
		(integer) 1 
		
		redis> sadd circle:jdoe:soccer users:toby
		(integer) 1 
		
		redis> sadd circle:jdoe:soccer users:apollo
		(integer) 1
		
需要记住的是，在上面的例子里，我们需要规范化我们set的成员通过使用实际的id数值而不是` users:name`。虽然上面的例子工作得很好，但它可能为了性能原因牺牲一些可读性而获取更快的速度和更高的内存使用效率。

现在我们有一个set叫做`circle:jdoe:family`,带有三个值（分别是：users:anna, users:richard, and users:mike），第二个叫做`	circle:jdoe:soccer`,带有四个值（分别是：users:mike, users:adam, users:toby, and users:apollo）。它们的值仅仅只是字符串，但是通过使用字符串对我们来说是有意义的（这和我们设计用户的hashes很相似），我们可以使用`SMEMBERS`命令的结果进而获取特点用户的信息。如下面的例子：

		redis> smembers circle:jdoe:family 
		1) "users:richard"
		2) "users:mike"
		3) "users:anna"

		redis> hgetall users:mike 
		(...)

现在，我们已经知道了怎么在sets里面存储信息了，我们可以扩展这个只是，做一些有趣的事情，比如获取属于这两个sets里面的人，或者获取一个`John Doe`添加到我们系统朋友圈中的完整的用户列表。

		redis> sinter circle:jdoe:family circle:jdoe:soccer 
		1) "users:mike"

		redis> sunion circle:jdoe:family circle:jdoe:soccer 
		1) "users:anna"
		2) "users:mike"
		3) "users:apollo" 
		4) "users:adam"
		5) "users:richard" 
		6) "users:toby"

根据我们的结果，`Mike`在`John Doe`的家庭圈和足球圈里面，通过两个圈的并集，我们也可以获取完整地成员列表。

如你所见，Redis sets使得RDBMS一些查询变得非常简单。Redis完成得非常的快，使得对于那些管理集合操作的应用来说，`Redis sets`是一个理想的候选者。朋友圈就是一个例子，此外像 推荐或者甚至文本搜素也很适合。在随后的案例里面，将会看到更深入的例子。


	SET key value
		Sets the key to hold the given value. Existing data is overwritten (even if of a dif- ferent data type).	

	GET key
		Returns the content held by the key. Works only with string values.
	
	INCR key
		Increments the integer stored at key by 1.
	
	INCRBY key value
		Performs the same operation as INCR, but incrementing by value instead.	
	DECR key
		Decrements the integer stored at key by 1.

	DECRBY key value
		Performs the same operation as DECR, but decrementing by value instead






