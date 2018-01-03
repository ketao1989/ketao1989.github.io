---
layout: post 
categories: 
  - Redis
  - Redis Cookbook
title: "Redis Cookbook 之 使用Redis存储基于时间序列的数据和进行分析"
date: 2014-11-29 22:21:35 +0800
comments: true
---

* content
{:toc}

##  <a id="Problem">问题</a>

存储分析或者其他基于时间序列的数据，对于传统的存储系统（比如RDBMS）来说，是有一点挑战的。可能你想要对输入流量的速率进行限制（要求快速和高并发更新）或者简单地追踪网站访问者（或者其他更复杂的度量指标），然后以图表的形式画出来。

虽然当前在其他系统中，有很多的方式存储这类数据；但是，Redis是一个非常优秀的候选者，由于它强大的数据结构。

## <a id="Solution">解决方法</a>

Redis 理念上非常适合存储这类数据，以及跟踪某种特定的事件。具有原子性的，并且非常快的（O(1)时间复杂度）`HINCR`和`HINCRBY`命令，结合快速数据查找，使得它非常适合这类场景。

在Redis中一种好的高效内存存储这类数据的方式是使用hash来存储统计值，使用`HINCRBY`增加它们，然后使用`HGET`和`HMGET`来获取这些数据。查找位于top位置的元素通过`SORT`命令也是很容易做到的。

<!-- more -->

## <a id="Discussion">讨论</a>

为了简单起见，在这个实例中，我们将只追踪网页点击率数据。这也可以很简单地扩展到其他任务类型的事件。

``` ruby

require 'rubygems'
require 'active_support/time'

# 增加访问者的点击数，id表，date键，field数值
def add_hit(id)
    $redis.sadd("clients", id)
    $redis.hincrby("stats/client:#{id}", "total", 1) 
    $redis.hincrby("stats/client:#{id}", Date.today.to_s(:number), 1)
end

```

我们在这里把用户（如果我们追踪网站的访问者，那么可以只简单地根据IP地址来区分用户）的ID添加到访问者列表中，然后记录在两个不同时间空挡中的点击数："total"总数和日常的数。因此，这就允许我们追踪每天的网页点击数和一段时间内的全局总数。

``` ruby

# 获取某id的key对应的值
def hits(id, day = Date.today) 
    $redis.hget("stats/client:#{id}", day.to_s(:number)).to_i
end

# 判断是否超过阈值
def over_limit?(id, limit) 
    hits(id) > limit
end

```

这允许我们通过简单地检查访问者访问，是否超过了我们设置的在一段时间区间内的阈值，来执行速率限制功能。

获取一个给定时间区间内的数据，也是一项琐碎但是高效的操作，我们可以用来画图表或者以其他方式展示这些数据：

``` ruby

# 计算给定开始时间和结束时间对应的key值
def keys(beg_p, end_p) 
    keys = []
    while beg_p <= end_p
        keys << if block_given? 
            yield(beg_p.to_s(:number))
        else 
            beg_p.to_s(:number)
        end
        beg_p += 1.day 
    end
    keys 
end

def stats_for_period(id, beginning_of_period, end_of_period) 
    beg_p = Date.parse(beginning_of_period)
    end_p = Date.parse(end_of_period)

    # 获取id表中key对应的数据集
    $redis.hmget "stats/client:#{id}", *keys(beg_p, end_p) 
end

```

我们也可以获取我们存储数据中在任何时间空挡的位于top的用户，可以使用`SORT`命令完成。SORT允许我们排序一个集合set，有序的集合sorted set,，或者本例中得列表list，访问者可以选择使用外键-我们时间片，然后指定order，offset，limit等参数：

``` ruby
# 按照key为period进行排序，默认DESC，前0-limit个元素
def top_clients(period = "total", limit = 5)
$redis.sort("clients", :by => "stats/client:*->#{period}", :order => "DESC",:get => ["#", "stats/client:*->#{period}"], :limit => [0, limit])
end

```

使用hash的实现方式，对于存储，检索和更新都是高度优化的（所有都是O(1)操作），但是对于计算top用户而言则不是（尤其是一个时间区间内）。你需要要求这些操作-比如当你显示一个高分值表格，你可以重新使用有序集合sorted set来完成排序，这样可以保证你拿到的数据是有序的：

``` ruby

def add_hit(id)
    $redis.zincrby("stats/total", 1, id) 
    $redis.zincrby("stats/#{Date.today.to_s(:number)}", 1, id)
end

def hits(id, day = Date.today) 
    $redis.zrank("stats/#{day.to_s(:number)}", id)
end

def over_limit?(id, limit) 
    hits(id) > limit
end

def stats_for_period(id, beginning_of_period, end_of_period) 
    beg_p = Date.parse(beginning_of_period)
    end_p = Date.parse(end_of_period)

    keys(beg_p, end_p) { |k| $redis.zrank("stats/#{k}", id) } 
end

def top_clients(period = "total", limit = 5) 
    $redis.zrevrange("stats/#{period}", 0, limit, :withscores => true)
end

def top_for_period(beginning_of_period, end_of_period, limit = 5) 
    beg_p = Date.parse(beginning_of_period)
    end_p = Date.parse(end_of_period)

    result_key = "top/#{beg_p.to_s(:number)}/#{end_p.to_s(:number)}"
    return $redis.zrevrange(result_key, 0, limit, :withscores => true) if $redis.exists result_key

    $redis.multi do
        $redis.zunionstore result_key, keys(beg_p, end_p){|k| "stats/#{k}"} $redis.expire result_key, 10.minutes
        $redis.zrevrange result_key, 0, limit, :withscores => true
    end.last 
end

```

> Notes：我们保持了`ZUNIONSTORE`的结果，然后在它上面设置一个超时时间戳。这是一个通用的Redis模式：缓存一个计算昂贵的操作结果，然后每次有请求过来，都会在重新操作之前先检查缓存情况。
> 在上面的例子中，我们使用hash的地方，我们也可以存储SORT操作的结果，然后使用和EXISTS相似的方式检查它的缓存对象的存在性。
> 

当我们使用有序集合sorted sets时，这些top操作会更高效率的多（因为数据已经是排好序了），但是我们的内存使用率也会更高。

> Warns：这个特定的例子有一个竞争条件：如果缓存不存在，我们可能在结束之前会进行多次`ZUNIONSTORE`操作。因为我们最后期待的输出显然是相同或者更新的数值结果，因此存在竞争条件比使用`WATCH`，然后在我们在做客户端的计算时锁定其他访问者，效果可能会更好。

-

> *Redis 命令*:
> 
> - `HINCRBY hash-name field increment-value`
>
>      按照给定的increment-value值增加hash表中存储的对应整数。这个命令和INCRBY很相似，但是和增加字符串不一样，这个使用在hash表中。而且increment-value的值也允许为负数。
>
> - `HMGET hash-name field1 [field2 ...]`
>
>      从给定的hash表中获取一些field值。这个命令和HGET很相似，但是这个允许你在一个单操作中获取一些field值。
>
> - `SORT key [BY pattern] [LIMIT offset count] [GET pattern1 [GET pattern2 ...]] [ASC| DESC] [ALPHA] [STORE destination]`
>
>     允许你排序一个list,set,或者sorted set，比较他们的值。排序也可以是使用外键完成，使用来自字符串或者hashes的模式匹配查询，就像我们在上面的例子中那样：`SORT clients BY stats/client:*->20110407`。其中，通配符*可以被set中成员所替换，所以在这些hash表中排序是基于匹配field 为20110407的值来完成的。如果我们把分析数据存储在strings中而不是hash表，则我们可以提交命令：`SORT clients BY stats/client:*/20110407`。
>      使用相同的模式，你除了排好序的list也可以获取更多地数据（比如你用来排序的值）.可选择地，在list里SORT的输出也可以被排序。
>      
> - `ZRANK set-name member`
>      
>      返回在给定的有序集合中给定成员的排名。
>      
> - `ZUNIONSTORE destination number-of-keys sorted-set1 [sorted-set2 ...] [WEIGHTS weight1 [weight2 ...]] [AGGREGATE SUM|MIN|MAX]`
>      
>      聚合sorted sets集合，然后作为一个新的sorted set存储。可选择地，你可以为每一个set指定 weight，并且只需聚合函数：sum（默认）,maximum scores, 或者 minimum scores。
>
> - `EXISTS key`
>      
>      检查key是否存在。如果key存在则返回1；否则返回0.
>

