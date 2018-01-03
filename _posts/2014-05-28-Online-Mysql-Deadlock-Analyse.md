---
layout: post
categories: 
  - Mysql
title:  "线上Mysql死锁问题分析"
date: 2014-05-28 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

前段时间，查看线上`Tomcat`日志，发现多台服务器出现`Mysql`死锁情况，虽然死锁问题没有影响到正常业务，但是毕竟数据库死锁还是得需要好好分析原因去修复和开发过程中极力需要避免的。服务器上`Mysql死锁`日志如下：

<img src="/images/2014/05/deadlock-log.png" />

由于我们的服务使用了连接池，所以接着让 DBA 查询下`Mysql`的数据库操作日志信息，如下图所示：

<img src="/images/2014/05/mysql-log.png" />

## <a id="Problem">死锁问题定位</a>

查看了建表语句，其实很简单：

``` sql
CREATE TABLE `room_rate_plan_id` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT, 
  `value` varchar(100) NOT NULL ,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_value` (`value`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=175663 DEFAULT CHARSET=utf8  
```

而使用的地方只在一个类代码里用过，使用的方式是`select for update`，并且该查询事务中包含着，如果不存在，则`INSERT INTO room_rate_plan_id (value) VALUES(?)`语句，那么，接着就得了解`select for update`来确定是不是这里导致数据库死锁现象。主要有两篇博客：

<!-- more -->

* <http://stackoverflow.com/questions/5432370/mysql-innodb-dead-lock-on-select-with-exclusive-lock-for-update>

* <http://dev.mysql.com/doc/refman/5.0/en/innodb-deadlocks.html>

`SELECT FOR UPDATE` 是`SELECT` 的升级版，其在查询时会上锁。一般地，当数据量特别大时，可能有大量的并发，这些并发会导致在`SELECT`时，状态已经变更了，因此需要上锁。InnoDB默认是Row-Level Lock，因此`SELECT FOR UPDATE`只有在`WHERE` 判断条件内明确指定主键时，才会执行`Row lock`，否则会`lock`整个数据表。当有多个请求同时加上`for update`的意向锁时，如果`select`没有时，第一个请求会接着`insert`而去尝试获取排它锁，B锁保持等待；而第二个请求也`select`为空时，并且刚好第二条请求插入的数据和第一个请求一样的时候，则会导致死锁。

## <a id="Reproduce">问题重现</a>

> 执行下面语句：

* `session 1`执行：

``` sql
START TRANSACTION;
SELECT id FROM qunar.room_rate_plan_id where value='elong_abcd'for update;
```

* `session 2`执行：

``` sql
START TRANSACTION;
SELECT id FROM qunar.room_rate_plan_id where value='elong_abcd'for update;
```

* 在`session 1`上执行 `insert` 语句：

``` sql
insert into qunar.room_rate_plan_id(value) values ('elong_abcd');
```

> 执行上面的操作，则在`Mysql`控制台上打出死锁日志信息：

<img src="/assets/img/2014/5/28/mysql-dead.png" />

## <a id="Solution">解决方法</a>

1. 采用官网上指出的方法：If you are using locking reads (SELECT ... FOR UPDATE or SELECT ... LOCK IN SHARE MODE), try using `a lower isolation level` such as READ COMMITTED.

2.  由于`select for update`只有不存在记录时才会去加一个意向锁，所以可以采用下面方法解决：select---> if(id) is empty --->insert;如果不为空，则select for update。不过这样就多了一次select了。

3. 使用`insert ignore` 插入，然后select ，由于是value是`unique key`，所以select可以获取正确id，并且还可以不需要添加事务。可能会多一次insert ignore。这种方法之所以这样子，是因为在业务中没有update操作，只有insert和select，并且value是唯一的，这样子就可以只采用select而不需要采用select for update，并且还可以不需要事务，提供效率。

> 如果采用第3中解决方案，不会出现死锁问题，但是当锁一直被占用时，会出现等待超时，当然，如果不使用事务，则肯定不会有锁的问题了。

	* session 1：
        ``` sql
		START TRANSACTION;
		INSERT IGNORE 	INTO room_rate_plan_id(value) VALUES ('elong_abcs');
		SELECT id FROM room_rate_plan_id where value='elong_abcs';
		```

	* session 2：
        ``` sql
		START TRANSACTION;
		INSERT IGNORE 	INTO room_rate_plan_id(value) VALUES ('elong_abcs');
		```
	* 出现超时异常信息：

<img src="/images/2014/05/timeout.png" />
