---
layout: post
category: Python
title: "Python 教程学习之 Python 函数式编程"
date: 2015-02-17 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

函数式编程是一种比较抽象的编程范式,其将各种指令计算运行作为数学中函数计算一样,尽量避免状态和变量的概念,因此函数式编程,一大特点就是无副作用,这对于并发编程大行其道的今天,是非常优良的特点.

Python 作为一个动态脚本语言，其很早就支持了函数式编程范式。

## <a id="FunctionalProgram">Python 函数式编程</a>

Python 函数式编程，涉及太多的内容。简单介绍几个点。

### 高阶函数

所谓高阶函数，和数学上的高阶函数定义基本一样，就是对函数进行计算处理的函数。其处理对象是另一个函数。

*Map 和 Reduce 函数*

云计算中，关于 MapReduce 的并行计算框架，在其他很多编程语言中，都会提供 API 方法。即使在Java这种笨重的语言中，也已经提供了相关的方法给开发者使用。

Python 的 map 方法，表示分别对列表中的元素独立处理；Reduce 方法，则表示对map处理完的结果，按照指定的方式进行聚合处理。

<!-- more -->

看看示例代码：

``` python


def lowerLetter(c):
    if c >= 'a' and c <= 'z':
        return True
    else:
        return False


def firstUpper(s):
    ss = ''
    for i in xrange(0, len(s)):
        if i == 0:
            if lowerLetter(s[i]):
                ss = ss + chr(ord(s[i]) + ord('A') - ord('a'))
            else:
                ss = ss + s[i]

        else:
            if lowerLetter(s[i]):
                ss = ss + s[i]
            else:
                ss = ss + chr(ord(s[i]) + ord('a') - ord('A'))
    return ss


def mergeWords(strA, strB):
    return strA + '-' + strB

mm = map(firstUpper, ['abc', 'zfc', 'Azc'])
print mm

rr = reduce(mergeWords, mm)
print rr

```

>> Notes ：  
>> - map 方法，对每个列表的元素处理，开头字符大写，其他字母都小写。代码中使用了最简单的原始字符处理方式处理，主要是学习下python的字节操作方法。
>> 
>> - Reduce方法，则把所有的单词都聚合在一起，使用`-`相连。reduce 方法会自动依次取出上一次的reduce结果和接下来的元素，再调用reduce处理。
>> 

*filter 函数*

filter 函数用来按照我们的要求，过滤队列。我们首先，给出一个过滤条件来依次判断列表的元素是否满足，根据返回的`True`或则`False`来决定去留。

看看示例代码：

``` python


def contains(ss, subStr='z'):
    return ss.find(subStr) != -1
# ['zfc', 'Azc']
ff = filter(contains, ['abc', 'zfc', 'Azc'])
print ff

```

*sorted 函数*

排序方法，对列表按照比较函数的返回结果，进行排序，返回排序完之后的列表。

看看示例代码：

``` python

# ['Azc', 'abc', 'zfc']
sl = sorted(['abc', 'zfc', 'Azc'])
print sl


def revertSortCmp(x, y):
    if x > y:
        return -1
    elif x == y:
        return 0
    else:
        return 1

# ['zfc', 'abc', 'Azc']
rsl = sorted(['abc', 'zfc', 'Azc'], revertSortCmp)
print rsl

```

### 返回函数

所谓返回函数，其实这也是函数式编程的核心之一。因为有了返回函数，我们可以确保函数式编程方法在获取结果的时候，才会执行；而不是，赋值函数的时候，立即执行。

这种延迟计算的特性，使得很多时候，可以获得更好的计算性能。

看看示例代码：

``` python

def lazy_tranform(ll):
    resTotal = []

    def tranform():
        res = []
        for x in ll:
            xf = x + '-tranform'
            res.append(xf)
            resTotal.append(xf)
        return res, resTotal
    return tranform

lf = lazy_tranform(['abc', 'zfc', 'Azc'])
#<function tranform at 0x1006309b0>
print lf
#(['abc-tranform', 'zfc-tranform', 'Azc-tranform'], ['abc-tranform', 'zfc-tranform', 'Azc-tranform'])
print lf()
#(['abc-tranform', 'zfc-tranform', 'Azc-tranform'], ['abc-tranform', 'zfc-tranform', 'Azc-tranform', 'abc-tranform', 'zfc-tranform', 'Azc-tranform'])
print lf()

```

>> Notes ：可以看到上面，对于`resTotal`变量，当函数多次运行，结果会重复 append 到列表中。 也就是说，对于返回函数，这种变量`resTotal`定义相对于是全局的，所有使用该函数的地方，都会这个变量进行变更。
>> 
>> 上面的代码，如果不了解，是很难定位到错误的。下面的代码，告诉你，如果你非要这样子写，应该如何处理这种涉及到`闭包`的问题。
>> 

``` python

def closure_tranform(ll):
    res = []
    for x in ll:
        def tranform():
            xf = x + '-tranform'
            return xf
        res.append(tranform)
    return res

# <function tranform at 0x100630aa0> <function tranform at 0x100630b18> <function tranform at 0x100630b90>
# Azc-tranform Azc-tranform Azc-tranform
lf1, lf2, lf3 = closure_tranform(['abc', 'zfc', 'Azc'])
print lf1, lf2, lf3
print lf1(), lf2(), lf3()


def closure_tranform_opz(ll):
    res = []
    for x in ll:
        def tranform(y):
            def tranform_closure():
                xf = y + '-tranform'
                return xf
            return tranform_closure
        res.append(tranform(x))
    return res
# <function tranform_closure at 0x100630cf8> <function tranform_closure at 0x100630c80> <function tranform_closure at 0x100630d70>
# abc-tranform zfc-tranform Azc-tranform
lfo1, lfo2, lfo3 = closure_tranform_opz(['abc', 'zfc', 'Azc'])
print lfo1, lfo2, lfo3
print lfo1(), lfo2(), lfo3()
```

>> Notes ：出现这种现象，就是因为我们的方法都是延迟加载，也就是说，最后的变量已经变更了，上面的代码中`i`就像`resTotal` 一样，是个全局变量，所以会在多次调用函数中彼此影响。
>> 

## <a id="Decorator">Python 装饰器</a>

熟悉 AOP 面向方面编程的同学，都了解切点，切面等概念。重要的是，使用 AOP 编程，可以让核心的业务代码和运维调试等代码切分开，也可以让老代码在一定程度上更容易升级。比如最通用的场景：日志记录，事务管理，权限拦截，自定义数据管理等。

在 Python 中，完成 AOP 编码范式的就是`Python 装饰器`。使用 `@`标记，就可以完成 AOP 简单功能了。

看看示例代码：

``` python

def logger(user_name='ketao1989'):
    def log(func):
        @functools.wraps(func)
        def wrapper(*args, **kv):
            print 'Welcome,', user_name, func.__name__
            func(*args, **kv)
            print 'Goodbye,', user_name, func.__name__
        return wrapper
    return log


def log(func):
    if isinstance(func, str):
        return logger(func)

    @functools.wraps(func)
    def wrapper(*args, **kv):
        print 'Welcome,', func.__name__
        func(*args, **kv)
        print 'Goodbye,', func.__name__
    return wrapper


#@log('ketao1989')
@log
def logger_test(params):
    print 'execute core function (%s) ......' % params

logger_test(['test_param1', 'test_param2'])

```

>> Notes：代码示例中，标记`@functools.wraps(func)`，是因为在装饰器中可能原始的函数的`__name__`等被覆盖，而该标注，可以保证原始携带的func的内置属性依然保持。
>> 
>> 此外，在代码的开始，需要`import functools`引入模块。
>> 

关于 Python 装饰器参考：<http://www.cnblogs.com/huxi/archive/2011/03/01/1967600.html>
