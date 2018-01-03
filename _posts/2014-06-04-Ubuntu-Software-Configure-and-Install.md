---
layout: post
categories: 
  - Linux 
title:  "Ubuntu Subversion软件安装和配置"
date: 2014-06-04 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

最近，系统从 `windows`切换到`Ubuntu`，一些开发软件需要重新安装和配置。但是，众所周知，`windows`上的开发软件客户端比`ubuntu`等linux系系统使用要便捷和傻瓜得多，所以切换到`ubuntu`有很多软件需要安装，但是这些软件需要进行代码编译，就在这里记录下。

## <a id="Subversion">subversion安装和配置</a>

一般，如果对`subversion`的版本不限制，那些直接使用`sudo apt-get install subversion`命令就可以安装来，但是如果对版本有要求，比如由于`svn`的1.8版本在很多`svn`服务器端不被支持，所以需要安装低于1.8的svn客户端。这就需要我们在本地编译完了之后再安装。

### 2.1 subversion源码下载

点击进入下载页面：<http://subversion.apache.org/download/#supported-releases> , 选择当前最新的`1.7`的子版本下载；

或者，也可以直接在终端使用命令：`wget http://apache.fayea.com/apache-mirror/subversion/subversion-1.7.17.tar.gz`下载。

### 2.2 subversion 源码编译准备

在`Linux`编译安装`subversion`需要事先准备很多的工作，安装很多相关的工具包，否则代码无法编译通过。因此，在安装` subversion`之前，需要先做一些准备工作。

<!-- more -->

### 2.2.1 安装 autoconf 和 libtool

编译`subversion`首先需要安装`autoconf`和`libtool`两个工具，如果你的电脑上没有安装这两个工具包，很简单，直接执行安装命令就可以来了：

    1.  sudo apt-get install autoconf
    
    2.  sudo apt-get install libtool
    
但是，如果你就接下来运行`./configure`命令，则会出现下面错误:

``` 

configure: Apache Portable Runtime (APR) library configuration
checking for APR... no
configure: WARNING: APR not found
The Apache Portable Runtime (APR) library cannot be found.
Please install APR on this system and supply the appropriate
--with-apr option to 'configure'

or

get it with SVN and put it in a subdirectory of this source:

   svn co \
    http://svn.apache.org/repos/asf/apr/apr/branches/1.3.x \
    apr

Run that right here in the top level of the Subversion tree.
Afterwards, run apr/buildconf in that subdirectory and
then run configure again here.

Whichever of the above you do, you probably need to do
something similar for apr-util, either providing both
--with-apr and --with-apr-util to 'configure', or
getting both from SVN with:

   svn co \
    http://svn.apache.org/repos/asf/apr/apr-util/branches/1.3.x \
    apr-util

configure: error: no suitable apr found

```

### 2.2.2 安装 APR

因此，你还需要`APR`,首先，在安装`apr`之前需要`sqlite-autoconf`，因此`subversion`需要使用`sqlite`来存储数据。

1.  sqlite-autoconf：<http://www.sqlite.org/2014/sqlite-autoconf-3080403.tar.gz>

解压缩文件，然后在`subversion-1.7.17`目录下面，新建一个目录`sqlite-amalgamation`，然后在目录下面，从解压缩后的`sqlite-autoconf`目录里面复制一个文件`sqlite3.c`到该新建目录中。

>> Note：需要注意的是，新建目录名必须为`sqlite-amalgamation`，虽然下载的文件是`sqlite-autoconf`，这主要是因为`sqlite-autoconf`工具是由`sqlite-amalgamation`来的，后来改了名字了，但是`subversion`编译的时候，并没有改变相应代码配置，所以还是需要用原来的命名。

接下来，就可以下载apr.tar.gz和apr-util.tar.gz两个源码包：

1.  apr：<http://mirrors.cnnic.cn/apache//apr/apr-1.5.1.tar.gz>
    
2. apr-util：<http://mirrors.cnnic.cn/apache//apr/apr-util-1.5.3.tar.gz>

解压缩完了之后，分别在`subversion-1.7.17`目录下面新建`apr`目录和`apr-util`目录，然后把解压缩后的内容复制到对应的新建目录中，分别执行`./buildconf`

>> Note：这里的目录名字不能改变，必须为`apr`和`apr-util`，否则会编译失败。

当然，在这里运行`./configure`还是会出现问题，错误如下：

``` 

checking zlib.h usability... no
checking zlib.h presence... no
checking for zlib.h... no
configure: error: subversion requires zlib

```

### 2.2.3 安装 zlib

好吧，这里还需要`zlib`库，所以接下来，还需要下载`zlib`源码包：

1.  zlib：<http://cznic.dl.sourceforge.net/project/libpng/zlib/1.2.8/zlib-1.2.8.tar.gz>

然后解压缩，在`subversion-1.7.17`目录下面，新建一个`zlib`目录，然后把解压缩的内容复制到该目录下，执行`./configure --shared`，然后在`make`，OK了。

### 2.2.4 安装 neon

如果不安装`neon`库，在使用`svn co http://...` 的时候，则会出现错误。

```

svn: E170000: Unrecognized URL scheme for http...

```

因此，我们需要安装`neon`库来提供`HTTP`库给svn工具使用。在安装`neon`前需要安装`libxml2`和`libxml2-dev`，直接使用`sudo apt-get install `安装就可以了。

然后，下载neon，地址为：<http://www.webdav.org/neon/neon-0.30.0.tar.gz>，解压缩之后，进入目录，执行`./configure`、`make`、`sudo make install`。


### 2.3 subversion源码编译安装

准备工作做好了之后，就可以开始编译安装了。

``` bash

 ./configure --with-neon=/usr/local CPPFLAGS="-I/home/ketao/java/subversion-1.7.17/zlib/ -L/home/ketao/java/subversion-1.7.17/zlib/"
make
make install
```

>> Note：网上嗖的时候，说是`./configure CPPFLAGS="-Izlib/ -Lzlib/"`就可以，但是在执行的时候出现问题，找不到`zlib`目录，所以需要写绝对路径。

接下来，你可以在终端执行命名，查看安装版本是否正确。

``` bash

svnversion --version
```
然后会出现：

``` 

svnversion, version 1.7.17 (r1591372)
   compiled Jun  4 2014, 17:39:27

Copyright (C) 2014 The Apache Software Foundation.
This software consists of contributions made by many people; see the NOTICE
file for more information.
Subversion is open source software, see http://subversion.apache.org/

```

## <a id="Finally">小结</a>

Linux下面安装软件，通过编译安装实在是比较复杂，有时候，涉及到多个类库，需要一个个去下载编译安装，比较麻烦。所以，一般情况下，推荐使用`apt-get install `来打包安装。
