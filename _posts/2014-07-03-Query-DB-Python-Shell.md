---
layout: post  
categories: 
  - Python
  - Script  
title: "使用Python操作线上数据库脚本"
date: 2014-07-03 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

最近对于线上一些数据需要进行过滤导到本地来，或者对一些运营人员给的文本数据，需要在线上数据库跑出对应的结果出来，因此，需要通过一个脚本来执行。因为，现在服务器基本上会默认安装python，所以这里存一份脚本备用。


## <a id="Script">Python脚本</a>

``` python

#!/usr/bin/python
#coding=utf-8
import os;
import MySQLdb;
import sys;
reload(sys) 
sys.setdefaultencoding('utf-8')
file=open("hotelseq.txt")
outfile=open("customer.txt",'w+')
conn=MySQLdb.connect(host='127.0.0.1',user='root',passwd='123456',db='test',port=3306,charset="utf8")
cur=conn.cursor()
order_cur = conn.cursor()

while 1:
    line=file.readline().strip()
    print line
    if not line:
        break
    count=cur.execute("select cus.customer_state,cus.name from crm_customer_hotel as hotel join crm_customer as cus on hotel.customer_serial_number=cus.serial_number where hotel.hotel_seq='%s'" % line)
    result = cur.fetchone()
    print result
    if result:
        outfile.write(str(result[0])+"  "+result[1])
        outfile.write('\n')
cur.close()
file.close()
outfile.close()


```

<!-- more -->
