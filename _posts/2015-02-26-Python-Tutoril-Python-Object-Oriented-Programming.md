---
layout: post
category: Python
title: "Python 教程学习之 Python 面向对象编程"
date: 2015-02-26 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

在之前的学习中,基本上都是使用类似于面对过程的方式来写代码,学习各种 Python 的基础知识，接下来，我们需要继续学习面向对象的Python，也就是类对象继承等概念。

对象和属性，这些概念在任务编程语言中，其定义都是一致的。我们通过一个对象来表示一个物体，而这个物体的一些特征，就是对象中的属性。比如，一个人，作为一个对象 Person，则会有`姓名，身高，体重，性别，出生日期`等属性，这些属性就是这个人的对外特性了。

## <a id="BasicGuide">Python 面向对象编程介绍</a>

类，就是对一类物体进行抽象的表示，比如上面的 Person；而实例，则是具体化了每一个类的属性值，其可以决定某一个物体特征的对象。

在 Python 中，可以如下来定义一个类：

``` python

class Person(object):

    """docstring for Person"""

    def __init__(self, name):
        super(Person, self).__init__()
        self.__name = name

```

<!-- more -->

`class` 用来表示这是一个类，`object` 表示该类继承的父类，默认是 object，但是推荐是写上父类，即使其继承 `object`类。

> Tips：Python 中，默认规定，类下面的第一行注释，是该类的 doc 说明。
> 
> 此外，`__xx__`形式的属性和方法，在Python中都是有特殊用途的，比如 __len__ 方法返回长度，再比如 __init__ 方法是类对象初始化时调用的，__new__ 方法则是构造对象的时候调用的，即在 `__init__` 之前调用。
> 

接下来，需要说明的是，Python 的权限控制。

和 Java 完全不一样，Python 没有关键字来说明该变量属性或者方法是 私有的 还是受保护的，亦或者是公共的。

Python 使用一种约定的方式来，声明访问限制的粒度。

- xx ：这种方式命名的变量或者方法，都是 `public`粒度的；

- _xx ：这种方式命名的变量或者方法，都是 `protected`粒度的；

-  __xx ：私有属性的变量或者方法，外部调用会报错的。

Python 中拥有的一些特殊功能的方法：

- type() ：获取对象的类型，比如，type('abc')会返回 `<type 'str'>` .

- isinstance()：type 方法不能很好的判断拥有继承关系的对象，而 `isinstance` 则可以。例如：`isinstance(person,object) #person是Person的实例`.

- dir()：想要获取一个类下面所有的方法吗？使用 dir 就可以了，比如`dir('abc')`

- hasattr/getattr/setattr()：依次是判断一个对象是否有指定属性，获取该属性，设置某个属性

## <a id="AdvancedTech">Python 面向对象编程技巧</a>

Python 中还有一些比较高大上的技巧。

*`@property`装饰器*

装饰器概念，在 `Python 函数式编程` 中已经介绍了。我们在类的代码 coding 中，有一些可能想要把方法使用如同属性使用般方便，则可以使用`@property`装饰器。

注解方式，如下所示：

``` python

    @property
    def age(self):
        try:
            return self.__age
        except Exception, e:
            return 0

    @age.setter
    def age(self, value):
        self.__age = value

```

*metaclass 元类*

Python 中，元类是一个很神奇的东西，由于其涉及到了底层的类结构构造，所以全部深入理解起来，可能会比较复杂。

关于元类的资料，可以参考<http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python>，第一个 answer 回答的特别棒。

当我们定义了类以后，就可以根据这个类创建出实例，所以：先定义类，然后创建实例。

但是如果我们想创建出类呢？那就必须根据metaclass创建出类，所以：先定义metaclass，然后创建类。

连接起来就是：先定义metaclass，就可以创建类，最后创建实例。

所以，metaclass允许你创建类或者修改类。换句话说，你可以把类看成是metaclass创建出来的“实例”。

现在，一个简单的示例，就是，对于该类所有的自定义属性，其name都大写。

``` python

class Female(Person.Person):

    """docstring for Female"""

    def log_construct(class_name, class_parent, class_attr):
        logging.info('==----==---contruct---===---')
        upper_attr = {}
        for key, value in class_attr.items():
            if key.startswith('__'):  # 不能改变class自带的属性
                upper_attr[key] = value
            else:
                upper_attr[key.upper()] = value

        return type(class_name, class_parent, upper_attr)

    def __init__(self, name, married):
        Person.Person.__init__(self, name)
        self.__married = married

    __metaclass__ = log_construct

if __name__ == '__main__':
    felame = Female('ketao1989', False)
    felame.age = 100
    print felame

    print hasattr(Female, 'MARRIED') # 返回Ture

```

## <a id="SourceCode">Python 面向对象编程代码示例</a>

这里主要是两个类，涉及到继承，和一些面向对象的使用方法。

``` python

#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2015-02-22 19:49:20
# @Author  : ketao1989
# @Version : $Id$


import sys
import os
import json

reload(sys)
sys.setdefaultencoding('utf8')


class Person(object):

    """docstring for Person"""

    def __init__(self, name):
        super(Person, self).__init__()
        self.__name = name

    @property
    def name(self):
        return self.__name

    @name.setter
    def name(self, value):
        self.__name = value

    def get_name(self):
        return self.__tranform_name(self.name)

    def __tranform_name(self, name):
        return name + "_test"

    @property
    def age(self):
        try:
            return self.__age
        except Exception, e:
            return 0

    @age.setter
    def age(self, value):
        self.__age = value

    def __str__(self):
        return "%s 's age is : %s" % (self.get_name(), self.age)

if __name__ == '__main__':
    person = Person('ketao1989')
    person.age = 10
    print person.name
    print person
    print os.uname()
    print os.getenv('PATH')
    print dir('abc')

#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2015-02-22 20:34:46
# @Author  : ketao1989
# @Version : $Id$


import sys
import Person
import logging
import json

reload(sys)
sys.setdefaultencoding('utf8')
logging.basicConfig(level=logging.INFO)


class Female(Person.Person):

    """docstring for Female"""

    def log_construct(class_name, class_parent, class_attr):
        logging.info('==----==---contruct---===---')
        upper_attr = {}
        for key, value in class_attr.items():
            if key.startswith('__'):  # 不能改变class自带的属性
                upper_attr[key] = value
            else:
                upper_attr[key.upper()] = value

        return type(class_name, class_parent, upper_attr)

    def __init__(self, name, married):
        Person.Person.__init__(self, name)
        self.__married = married

    __metaclass__ = log_construct

    @property
    def married(self):
        return self.__married

    @married.setter
    def married(self, value):
        self.__married = value

    def get_name(self):
        return self.name + "_Female____"

    def __str__(self):
        prefix = Person.Person.__str__(self)
        return prefix + ",married is : %s" % str(self.MARRIED)

if __name__ == '__main__':
    felame = Female('ketao1989', False)
    felame.age = 100
    print felame

    print hasattr(Female, 'MARRIED')

    print Female
    print hasattr(Female, 'job')
    felame.job = 'dev'
    print hasattr(felame, 'job')
    print felame.name

    felame1 = Female('ketao', True)
    print hasattr(Female, 'job')
    print hasattr(felame1, 'job')
    print hasattr(felame, 'job')
    print felame.__class__.__class__
    print json.dumps(felame, default=lambda obj: obj.__dict__)
    json_str = '{"_Person__name": "ketao1989", "_Person__age": 100, "job": "dev", "_Female__married": false}'
    felame2 = json.loads(json_str)
    print felame2


```

> Notes : 在Female类中，继承需要写成`Person.Person`的形式，前面的`Person`是module,后面的 `Person`才是类。
> 
> 此外，使用metaClass方式更改属性时，只会修改该类自己的属性，而不会更改其继承的父类的属性名。
