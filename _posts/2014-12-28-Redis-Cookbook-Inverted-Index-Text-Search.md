---
layout: post 
categories: 
  - Redis
  - Redis Cookbook
title: "Redis Cookbook 之 基于Redis 实现倒序索引全文搜索"
date: 2014-12-28 22:21:35 +0800
comments: true
---

* content
{:toc}

##  <a id="Problem">问题</a>

倒序索引是一种索引数据结构，该索引存储单词（或者其他内容）到它们位于文件，档案或者数据库等位置之间的映射关系。这个通常被用来实现全文搜素服务，但是这要求在搜索之前这些文档的相关倒序索引就必须建立好。

因此，我们想要事业Redis来作为背后的存储系统来实现全文搜索服务。

## <a id="Solution">解决方法</a>

我们的实现，将为每一个单词，准备一个set集合，这些集合包含对应的文档的ID。为了允许快速搜索，我们将在开始之前为所有的文档建立索引。

搜索服务本身先分割请求为各个单词，然后获取每个单词匹配的集合set的交集，最后就可以返回包含所有我们搜索的单词的文档ID集。

##  <a id="Discussion">讨论</a>

###  建立索引

首先，让我们假设我们有一百个允许我们搜索的文档或者网页，因此需要对它们建立倒序索引。为了建立索引，我们必须分割文本为分开的单词（分词操作），在此过程中，可能需要排除`stop word`以及长度小于3的单词。使用Ruby脚本，如下所示：

``` ruby

def id_for_document(filename)
    doc_id = $redis.hget("documents", filename) 
    if doc_id.nil?
        doc_id = $redis.incr("next_document_id") 
        $redis.hset("documents", filename, doc_id) 
        $redis.hset("filenames", doc_id, filename)
    end
    doc_id 
end

STOP_WORDS = ["the", "of", "to", "and", "a", "in", "is", "it", "you", "that"] f = File.open(filename)
doc_id = id_for_document(filename)
f.each_line do |l|
    l.strip.split(/ |,|\)|\(|\;|\./).each do |word|
        continue if word.size <= 3 || STOP_WORDS.include?(word) 
        add_word(word, doc_id)
    end
end

```

所以，我们将过滤掉这些已经被加入到索引的单词，然后为我们的文档生成唯一的ID。此外，我们仍然需要完成上面的索引方法：

<!-- more -->

``` ruby

def add_word(word, doc_id) 
    $redis.sadd("word:#{word}", doc_id)
end

```

因此，对于每一个我们在文档中发现的单词，我们都已经创建了一个新的集合set，该set包含被发现单词的文档ID集。

###  搜索

倒序索引的优势是查找的时候真的非常的快，因为绝大部分的工作在文档建索引的时候就已经完成了。为了搜索，我们仅仅需要，找到我们搜索查询里面单词对应的集合set的交集。下面的代码使用`redis-rb`接口完成查询redis服务器命令。

``` ruby

def search(*terms)
    # 对每一查询单词求对应的id集合，然后求集合的交集
    document_ids = $redis.sinter(*terms.map{|t| "word:#{t}"}) 
    # 根据id集合，查找对应文件名集合
    $redis.hmget("filenames", *document_ids)
end

```

> Notes: `sinter方法`：求指定多个集合的交集


###  排序计分

虽然前面的方法某种程度上是有限制的，并且非常简单；但是也是很容易扩展的。其中一件我们可以做的事情就是，当返回搜索结果的时候排序我们的文档，我们可以考虑计算一种分数：高分表示和我们搜索的查询更相关（比如查询单词位于文档的主题或者标题中）或者只是单纯地以为出现更高的次数。因此，我们将该索引方法如下所示：


``` ruby

def add_word(word, doc_id) 
    $redis.zincrby("word:#{word}", 1, doc_id)
end

```

搜索的结构会变得更加复杂一点点：

``` ruby

def search(*terms)
    document_ids = $redis.multi do
        $redis.zinterstore("temp_set", terms.map{|t| "word:#{t}"})
        $redis.zrevrange("temp_set", 0, -1) 
    end.last
    $redis.hmget("filenames", *document_ids) 
end

```

> Notes: 这里使用了前面代码中多个方法，这是因为我们在有序的`temp_set`集合中有一个潜在的竞争条件。当你必须在任务其他人也尝试访问他们改变的数据之前，都使用这两个或者更多命令（在`ZREVRANGE`命令之前完成`ZINTERSTORE`命令），就会有潜在的竞争条件存在。

为了避免在运行的时候出现竞争条件，当我们执行并发的搜索查询的时候，我们必须要么使用Redis的`MULTI/EXEC`命令，要么可能为每一个查询搜索产生一个唯一键。（在上例中，我们必须在我们自己之后清除并且删除临时的排序set集合）。

`MULTI 和 EXEC`命令运行Redis中得事务行为。在`MULTI/EXEC`块中得命令保证运行的时候序列化串行，这意味着在块长度期间，没有其他的Redis客户端获取服务。在先前的例子里，它排除了在`temp_set`中的竞争条件，因为其他客户端不可能在`ZINTERSTORE`和`ZREVRANGE`操作之间修改值。在事务内部使用`DISCARD`就会放弃事务，丢弃所有的命令，然后返回一个正常状态。

由于命令只会在`EXEC`之后才会被调用执行，因此只有在那个时刻你才会接收事务内部所有命令做出的回答响应。因此，不可能会使用同一个事务的事务内部一个命令运行的响应结果。为了达到这点，你将需要使用`WATCH`.

`redis-rb`没有直接的`EXEC`调用。换句话说，在提交给你的`multi方法`的块的开始和结束，表明也是事务的开始和结束。在你块结束的时候，`redis-rb`内部会调用`EXEC`。

> *Redis 命令*:
> 
> - `ZINCRBY zset-name increment element`
>
>      添加或者增长在有序集合中元素的分数。而使用ZADD和SADD，则如果集合不存在则将会被创建。
>
> - `ZINTERSTORE destination-zset number-of-zsets-to-intersect zset1 [zset2 ...] [WEIGHTS weight1 [weight2 ...]] [AGGREGATE SUM | MIN | MAX]`
>
>      计算给定的一些ZSETS集合的交集，然后把结果存储在新的ZSET中。此外，也可以使用增长因子或者聚合方法来获取新的集合。默认情况下，它是所有集合中分数的和，但是它也可以是最大或者最小值。
>
> - `ZREVRANGE zset-name start-index stop-index [WITHSCORES]`
>
>      返回在有序集合中给定范围内的元素，以递减的顺序。这个命令也可以选择在返回结果中包含元素的分数。ZRANGE命令执行相同的操作，但是是以递增的顺序。

### 其他优化

对于搜索，还有许多地方可以被优化：

- *大小写敏感*

    我们可以在建立索引之前单词和查询之前的搜索项，使用单词的大小写敏感。

- *模糊搜索*

    可能你也感兴趣实现模糊搜索作为你的搜索应用的一部分。它考虑基于通常错误的拼写单词。例如，在我们的例子中，在建立索引的时候也一起考虑为拼写错误单词的项建立索引，要么从一个列表中查找，要么为这个目标使用专门的算法（例如语音学上的算法）

- *部分单词匹配*

    虽然这个非常有用，但是将会增加索引内存的使用，并且给出一些你不想要的搜索结果。为了达到这个目的，你不得不分解你的单词为子串，然后为它们建索引。例如，为单词`matching`建索引，你不得不增加下面这些：

        matching
        mat 
        matc 
        match 
        matchi 
        matchin

    假设设置的最小长度为3个字符，并且也假设我们只匹配单词的前缀。如果我们有兴趣建立所有可能的组合，你需要为这个单词其他的子串也建立索引。

    使用有序集合对于这个和前面的模糊查找增强技术都是很有用的。你可以根据部分单词匹配和错误拼写单词而让它们*获取更低的分数来提高你的搜索结果质量*。

##  <a id="InvertedIndex">倒序索引介绍</a>

如果不使用倒序索引技术，在每次进行检索时，搜索引擎必须遍历每一个网页，查找网页中是否包含你指定的关键词。这个工作量是十分巨大的，主要原因有二：

- 互联网的网页基数非常大；

- 在每一个网页中检索是否含有指定的关键词不是一件简单的事情，它需要遍历网页的每个字符。
为了更好的建立被搜索的关键字和含有这些关键字的页面之间的映射关系，倒序索引产生了。简单的说，倒序索引的倒序，指的是这个索引是从关键词中查找对应的源的，而不是从源中检索对应的关键词。

*举例如下*：为了检索关键词 A，首先从倒序索引的索引表中，找到关键词 A，然后查找 A 所在的页。由于倒序索引表排序后，在其中查找一个关键词可以使用二分查找，特别是在采用分布式数据、服务器集群、多线程技术等条件下，效率极高，所以，查找含有某个关键词的页变得非常简单。

假设数据库中含有1000000条记录，其中有 10 条记录符合搜寻条件，如果使用倒序索引，可以很快找到这些关键词，并且定位到含有这些关键词的十条记录；否则，需要遍历1000000条记录，效率的差异可想而知。

所以，倒序索引相当于一本出处大字典，查阅其中的每个词汇，都可以告诉你它的所有出处。

倒序索引中的关键词，一般是 *蜘蛛（Spider）*在网页爬行时对网页进行分词的结果。中文分词也是一件比较麻烦的事情。关于 *分词技术*，请查阅其他相关文章。