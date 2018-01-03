---
layout: post
category: Python
title: "Python 教程学习之 Python 函数和高级特性"
date: 2015-02-15 22:21:35 +0800
comments: true
---

## <a id="Intro">前言</a>

上一篇博客简单介绍了 Python 基本的概念和操作方法。但是，和其他编程语言一样，是缺少不了函数的。有了函数，代码才会被多次复用，程序才会简洁和易读。

同样，也是为了开发的简洁，Python 还为提供了 Slice切片等高级特性。切片可让我们更方便的操作列表数据结构，也使得代码更易开发和阅读。

## <a id="Function">Python 函数</a>

Python 函数和数学中函数的概念是一致的，都是对逻辑的一种抽象表示。在 Python 中，我们通过def来定义一个函数方法。

<!-- more -->

Python 的一些函数方法使用，如下所示：

``` python

#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2015-02-14 11:08:03
# @Author  : ketao1989
# @Version : $Id$


import sys
import os

reload(sys)
sys.setdefaultencoding('utf8')


def funTypeCheck(x):

    if not isinstance(x, int):
        raise TypeError('类型不一致')

    if x == 10:
        print 'x is my number'
    else:
        print 'x is not my number'


def moreReturns(x, y):
    if x > y:
        y = 101
    else:
        x = 101
    return x, y


def defaultParams(x, n=2, extra=0):
    multiply = 1
    while n > 0:
        n -= 1
        multiply = multiply * x
    return multiply + extra


def variableParams(*numbers):
    sum1 = 0
    for num in numbers:
        sum1 += num
    return sum1


def variableDictParams(name, **kw):
    print '----', name, '----'
    print kw

# @tail_call_optimized
def factFunction(s, n):

    if n < 1:
        return s
    return factFunction(s * n, n - 1)

if __name__ == '__main__':
    funTypeCheck(10)

    x, y = moreReturns(10, 20)
    print x, y

    multiply = defaultParams(5)
    print multiply
    multiply = defaultParams(5, 5, 3)
    print multiply
    multiply = defaultParams(5, extra=2)
    print multiply

    sum1 = variableParams(1, 2, 5)
    print sum1

    variableDictParams('ketao1989', web='ketao1989.github.com')

    print factFunction(1,100)
    print('-------------')
    funTypeCheck('error')

```

>> Notes：`isinstance` 方法判断是否是指定类型。
>> 
>> - `moreReturns(x, y)` 函数，返回多个结果。其本质上，就是返回一个tuple.
>>
>> - `defaultParams(x, n=2, extra=0)` 函数，设置默认的参数值。在调用的时候，我们可以替换任意一个默认的参数，比如extra，可以：`defaultParams(5, extra=2)`
>> 
>> - `variableParams(*numbers)` 函数，设置可变个值。
>>
>> - `variableDictParams(name, **kv)`函数，设置可变你的参数。其实，其类似Map来根据调用方确定参数。
>> 


## <a id="Advanced">Python 高级特性</a>

### 切片

切片，很简单。就是我们日常对列表进行sub操作的时候，一般需要`for`循环操作，而 Python 提供了一个更简单的方法来处理这些问题，就是切片。其，表示形式为:`[i:j:k]`，其中，i表示列表的开始位置索引，j表示列表的结束位置索引（不包含该文章）,k表示循环步长。

``` python

ll = [1,4,6,8]

# [4, 6],开始:结束:步长
print ll[1:3:1]

# (1, 3, 5)
print (1,2,3,4,5,6,)[0::2]

```

### 迭代

Python 的迭代，相对于 C 语言的迭代语法，有了很大的进步。使用 `for ... in` 方式来遍历一个集合或者 Map 数据，方便快捷。

``` python

ll = [1, 4, 6, 8, 11, 24, 23, 56]
for x in ll:
    print x
for i, value in enumerate(ll):
    print i, value

lt = [(1, 2), (5, 6), (7, 8)]
for x, y in lt:
    print x, y

dd = {'a': 'A', 'b': 'B', 'c': 'C', 'd': 'D'}

for key in dd:
    print key
for value in dd.itervalues():
    print value
for key, value in dd.iteritems():
    print key, value

```

### 列表生成器

迭代对于列表的优化还不够，Python 还提供了更牛逼的特性：让我们一行代码生成一个列表。

比如，生成一个 1 到 10 数字的平方数，组成一个列表，使用列表生成器，可以如下：

``` python

lg = [x * x for x in xrange(1, 10)]
print lg

# 产生偶数的平方数
lo = [x * x for x in xrange(1, 11) if x % 2 == 0]
print lo

```

>> Notes ：由于列表生成器，是一次生成所有的列表元素，对于很大的列表，如果一次生成完成，然后全部放在内存里，肯定是不好的，因此，Python 还提供了 `生成器`。
>> 

生成器，可以根据生成规则，在使用的时候，才会构造元素。构造简单的生成器，和列表生成器一样，只需要把`[ ]` 替换成`( )`即可。例如：

``` python

# <generator object <genexpr> at 0x107de44b0>
lg = [x * x for x in xrange(1, 10)]
print lg

```

>> Tips：如果需要打印，可以使用for，单个使用.next()方法依次调用，直到出现异常。

对于复杂一点的生成器，则需要使用 `yield` 关键字来完成。比如，在我们的一些函数中，某一个变量连续产生的数组，可以使用`yield x`来完成生成器设置。

例如，构造一个通俗的生成器，如下：

``` python

def generateFun():
    for x in xrange(1, 10):
        yield x * x
# <generator object generateFun at 0x102f5a5a0>
gf = generateFun()
print gf

for x in gf:
    print x

```
