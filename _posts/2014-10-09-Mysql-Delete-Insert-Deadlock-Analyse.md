---
layout: post
categories: 
  - Mysql
title:  "线上Mysql Delete 和 Insert 操作导致死锁问题分析"
date: 2014-10-09 22:21:35 +0800
comments: true
---

## <a id="Intro">前言</a>

今天项目发布，上线上跟日志的时候，发现一些死锁信息的出现，查询了一下，发现日志里面虽然死锁出现很少，但是都是同一个代码`sql`语句产生的，如下图所示：

<img src="/images/2014/10/deadlock-log.png" />

并且，一天产生死锁`31`次。

从DBA那边拿到了对应死锁的`Mysql`日志信息：

``` bash
------------------------
LATEST DETECTED DEADLOCK
------------------------
141009 12:54:59
*** (1) TRANSACTION:
TRANSACTION AEE50DCB, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 376, 1 row lock(s)
MySQL thread id 6055694, OS thread handle 0x7f4345c8d700, query id 2443700084 192.168.249.154 crm_w updating
DELETE FROM crm_business WHERE serial_number = 'CH01313320'
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 244 page no 817 n bits 824 index `uniq_serial_number_business_type` of table `crm`.`crm_business` trx id AEE50DCB lock_mode X waiting
*** (2) TRANSACTION:
TRANSACTION AEE50DCA, ACTIVE 0 sec inserting, thread declared inside InnoDB 500
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1248, 3 row lock(s), undo log entries 1
MySQL thread id 6055696, OS thread handle 0x7f4344941700, query id 2443700084 192.168.249.154 crm_w update
INSERT INTO crm_business(serial_number, business_type) values ('CH01313318', 2)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 244 page no 817 n bits 824 index `uniq_serial_number_business_type` of table `crm`.`crm_business` trx id AEE50DCA lock mode S
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 244 page no 817 n bits 824 index `uniq_serial_number_business_type` of table `crm`.`crm_business` trx id AEE50DCA lock_mode X locks gap before rec insert intention waiting
*** WE ROLL BACK TRANSACTION (1)
```

>> Note: 数据库中，`CH01313318`和`CH01313320`序列号是连着的两个记录。

## <a id="Problem">死锁问题定位</a>

之所以需要分析这个问题，主要原因是，这边代码并*没有*涉及到在一个事务内部sql操作导致死锁等常见的情况。

<!-- more -->

* 首先，我们看看设计问题的代码片段：

``` java

for (final C c : cs) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    processOneC(c);
                }
            });
        }


private void processOneC(C c) {


        if (Strings.isNullOrEmpty(c.getSeqNumber())) {
            return;
        }
        try {

            // 删除原合作业务
            cBusinessDAO.deleteBySN(c.getSerialNumber()); //delete操作

            //..........
            
            setTuanBusiness(c, hotelResult); //insert操作
            setDirectBusiness(c, hotelResult); //insert操作
        } catch (Exception e) {
        
            logger.error("CustomerBusinessProcess.processOneCustomer error, Exception", e);
        } finally {
            System.out.println("==========" + co.addAndGet(1));
        }
    }


```

上面代码的逻辑很简单，就是对一个列表对象，多线程来分别操作。每个对象的处理，删除原来的记录，然后添加新的记录。

* 其次，设计到的SQL语句：

``` sql

DELETE 
FROM crm_business 
WHERE serial_number = 'xxxx'

INSERT 
INTO crm_business(
serial_number, 
business_type
) values (
'xxxx', 
1
)
```

* 最后，看看相关表的建表sql语句：

``` sql

CREATE TABLE `crm_business` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `serial_number` varchar(50) NOT NULL COMMENT '商户编号',
  `business_type` tinyint(1) NOT NULL COMMENT '业务类型',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_serial_number_business_type` (`serial_number`,`business_type`)
) ENGINE=InnoDB AUTO_INCREMENT=34856774 DEFAULT CHARSET=utf8 COMMENT='合作业务'

```

数据表里面，字段很简单，但是里面涉及一个唯一键索引。

>> Note: 上面的代码，`delete`和`insert`操作不在一个事务里面，因为代码上没有添加@transaction 注解，也就是单纯delete`和`insert`操作在多线程环境下，竟然会产生死锁。此外，需要注意，操作唯一键相关字段，经常会不注意就产生死锁问题。

## <a id="Analyse">死锁问题分析</a>

首先，给出一篇博客，mysql大神写的<http://hedengcheng.com/?p=844>,一下分析，借鉴使用该博客的分析。

>> Tips: 有兴趣，请直接阅读该博客的分析。

### Mysql日志分析

以上Mysql产生的死锁日志，包含两个事务。

*事务1*：当前正在操作一张表（mysql tables in use 1），持有两把锁(2 lock structs，一个表级意向锁，一个行锁(1 row lock))，这个事务，当前正在处理的语句是一条delete语句。同时，这唯一的一个行锁，处于等待状态(WAITING FOR THIS LOCK TO BE GRANTED)。

事务1等待中的行锁，加锁的对象是唯一索引`uniq_serial_number_business_type`上页面号为`817`页面上的一行(注：具体是哪一行，无法看到。但是能够看到的是，这个行锁，一共有824个bits可以用来锁824个行记录，n bits 824：lock_rec_print()方法)。同时，等待的行锁模式为next key锁(lock_mode X)。简单来说，next key锁有两层含义，一是对当前记录加X锁，防止记录被并发修改，同时锁住记录之前的GAP，防止有新的记录插入到此记录之前。

*事务2*：和事务1一样，事务2上面有三个行锁，三个行锁，`undo log entries 1`，两个锁都是唯一索引`uniq_serial_number_business_type`上页号`817`上的某一条记录。其中，一个锁处于持有状态，锁模式为S mode，即共享锁，同时，另外一把锁处于等待状态，锁模式为X mode，即互斥锁。因此，拥有锁S模式不代表可以获取接下来的X模式的锁。

事务1正在等待事务2释放锁的S模式，从而获取X模式的锁；但是事务2已经获取了S模式，但是其等待继续获取锁的X模型,这个模式事务1优先申请获取，因此就导致死锁。

>> Note: 事务1优先去等待获取锁的X模式，是因为在Mysql中为了公平竞争，杜绝事务1发生饥饿现象。这样会导致上述死锁出现。

那么为什么会出现`delete`操作抛出死锁失败，事务回滚。

Mysql实现，会根据死锁冲突的两个事务的权重，事务1的权重会更低，然后被选为抛弃的对象，回滚该操作。

### Delete操作的加锁逻辑

``` sql
DELETE 
FROM crm_business 
WHERE serial_number = 'xxxx'
```

` serial_number`是数据表里面的唯一键索引，一个二级索引键值。

` serial_number`是unique索引，而主键是id列。因此delete语句会选择走`serial_number`列的索引进行where条件的过滤，在找到`serial_number = 'xxxx'`的记录后，首先会将unique索引上的`serial_number = 'xxxx'`索引记录加上X锁，同时，会根据读取到的`id`列，回主键索引(聚簇索引)，然后将聚簇索引上的`id = 10` 对应的主键索引项加X锁。为什么聚簇索引上的记录也要加锁？试想一下，如果并发的一个SQL，是通过主键索引来更新：`update crm_business set serial_number = xyy where id = 10`; 此时，如果delete语句没有将主键索引上的记录加锁，那么并发的update就会感知不到delete语句的存在，违背了同一记录上的更新/删除需要串行执行的约束。

>> Note：和delete操作类似的加锁操作，还有：  

``` sql

    select * from table where ? lock in share mode;  
    select * from table where ? for update;  
    insert into table values ('xxx');  
    update table set ? where ?;  
    delete from table where ?;  
```

>> 上述操作都是当前读，需要读取记录的最新操作，读取之后，为了保证其他并发事务不能修改当前记录，会对读的记录加锁，可能是S模式锁，或者X模式锁。


