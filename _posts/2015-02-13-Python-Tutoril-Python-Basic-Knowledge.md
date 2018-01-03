---
layout: post
category: Python
title: "Python 教程学习之Python基础知识"
date: 2015-02-13 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

2015年春节临近，提前请假回家，闲下来的功夫，系统学习下 python 语言，倒是也好。

Python 在日常工作中，主要是脚本语言的开发，由于开发快捷容易，并且在所有 Linux 版本都会安装 Python 的语言环境，不需要额外的配置工作。

## <a id="Sublime">Sublime Python 插件安装</a>

对于 Python 的基础知识的学习，已经工作中脚本语言的开发，使用 Sublime 编辑器可以非常好的满足我们的需求，并且 Sublime 提供了丰富的插件供使用，非常便捷。

开发 Python(Django) 语言，一般推荐安装 :

- `SublimeCodeIntel` 插件：一款代码自动提示的插件。

- `Python PEP8 Autoformat` 插件：一款针对 Python 的代码格式化插件，快捷键为`Ctrl + Shift + R`。

- `SublimeTmpl` 插件：一款自定义文件模板的插件，可以跟进我们的配置在创建文件时，跟进模板完成初始化填充。关于 Python 的代码模板如下所示：

``` python

#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : ${date}
# @Author  : ${author}
# @Version : \$Id\$


import sys
${1:import os}

reload(sys)
sys.setdefaultencoding('utf8')
$0

```

>> Notes：打开 `preferences --> pachages`，在打开的目录下依次找到`sublimeTmpl --> templates --> python.tmpl`，直接修改就好了。
>> 

<!-- more -->

## <a id="BasicKnowledge">Python 基础知识</a>

基础知识都非常简单，这里贴一下学习的测试代码，就好了。

``` python

#!/usr/bin/env python
# -*- coding: utf-8 -*-


import sys

reload(sys)
sys.setdefaultencoding('utf8')

# 100
a = -100
if a > 0:
    print a
elif a == 0:
    print 0
else:
    print -a

# 中文
print u'中文'

# 4
names = ['zhang', 'li', 'wang', 'er']
print len(names)

# 5
names.append('ma')
print len(names)

# ['zhang', 'li', 'wang', 'zi', 'er', 'ma']
names.insert(3, 'zi')
print names

# ['zhang', 'li', 'wang', 'zi', 'er']
names.pop()
print names

# ['ma', 'li', 'wang', 'zi', 'er']
names[0] = 'ma'
print names

# ['ma', 'li', 'wang', 'zi', 'er', ['subzhang', 'subwang']]
names.append(['subzhang', 'subwang'])
print names

# ('ma', 'li', 'wang', 'zi', 'er', ['subzhang', 'subwang'])
tumples = tuple(names)
print tumples

# ['ma', 'li', 'wang', 'zi', 'er', ['subzhang', 'subwang']]
names = list(tumples)
print names

# 分别打印元素，['subzhang', 'subwang']是一体的
for x in tumples:
    print x
# dict字典，ketao
maps = {'name': 'ketao', 'age': 25, 'job': 'dev'}
print maps['name']

# {'job': 'dev', 'age': 25, 'program': 'python', 'name': 'ketao'}
maps['program'] = 'python'
print maps

# {'job': 'dev', 'gender': 'male', 'age': 25, 'program': 'python', 'name': 'ketao'}
if 'gender' in maps:
    pass
else:
    maps['gender'] = 'male'

if maps.get('gender'):
    print maps

# set(['wang', 14, 'zhang'])
sets = set(['zhang', "wang", 14])
print sets

# set(['li', 'wang', 14, 'zhang'])
sets.add('li')
print sets

# set([14, 'zhang']); set(['ma', 'wang', 'zhang', 14, 'li'])
setss = set(['zhang', 14, 'ma'])
print setss & sets
print setss | sets

```

>> Notes : 需要说明的是，必须在python 脚本中，注入 utf-8 相关配置，不然会出现下面的编码问题。如果使用模板，请参考上一节。
>> 

``` 

Traceback (most recent call last):
  File "/Users/ketao/python/test.py", line 20, in <module>
    print u'中文'
UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-1: ordinal not in range(128)
[Finished in 0.0s with exit code 1]
[shell_cmd: python -u "/Users/ketao/python/test.py"]
[dir: /Users/ketao/python]
[path: /usr/bin:/bin:/usr/sbin:/sbin]

```

