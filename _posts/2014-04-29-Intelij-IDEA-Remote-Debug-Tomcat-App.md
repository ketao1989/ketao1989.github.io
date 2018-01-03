---
layout: post  
categories: 
  - Java 
  - IDE  
title: "Intelij IDEA 远程调试Tomcat服务"
date: 2014-04-29 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

在很多情况下，我们在本地启动调试一些服务；或者说外部调用开发测试环境某些服务时，需要直接调试定位问题代码点；
这些问题都会让我们需要可以在本地IDE上面调试本地代码来查看线上情况。最近和其他业务部门联调的时候，
了解到原来真的可以debug本地代码同步控制线上运行流程。下面，记录一下具体的操作配置步骤。

## <a id="Server">线上服务配置</a>

目前线上的整个tomcat的服务脚本配置：

1. 一台机器上放一个全局脚本，比如放置在`/home/tomcat/bin`目录下；
1. 机器上的每一个tomcat实例目录里面都会有一些基本的设置，比如tomcat的`conf`目录，以及`startenv.sh`文件，
1. `startenv.sh`文件目前的配置为：  

``` bash
export TOMCAT_USER="tomcat"
export JAVA_OPTS="-Xms512m -Xmx1024m -XX:NewSize=256m -XX:PermSize=256m -server -XX:+DisableExplicitGC -Dqunar.logs=$CATALINA_BASE/logs -Dqunar.cache=$CATALINA_BASE/cache -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:$CATALINA_BASE/logs/gc.log"
chown -R tomcat:tomcat $CATALINA_BASE/logs
chown -R tomcat:tomcat $CATALINA_BASE/cache
chown -R tomcat:tomcat $CATALINA_BASE/conf
chown -R tomcat:tomcat $CATALINA_BASE/work
chown -R tomcat:tomcat $CATALINA_BASE/temp
```

<!-- more -->

因此，为了方便，我们只需要增加debug相关配置在JAVA_OPTS就可以了：

``` bash
-server -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,address=9999,server=y,suspend=n
```

> Note: 这里的端口指定为9999，你也可以自己指定。主要是JVM绑定端口使用。

	-Xdebug					|启用调试特性
	-Xrunjdwp				|启用JDWP实现，它包含若干子选项：
	transport=dt_socket		|JPDA front-end和back-end之间的传输方法。dt_socket表示使用套接字传输。
	address=9999			|JVM在9999端口上监听请求。
	server=y				|y表示启动的JVM是被调试者。如果为n，则表示启动的JVM是调试器。
	suspend=y				|y表示启动的JVM会暂停等待，直到调试器连接上。
	suspend=y这个选项很重要。如果你想从Tomcat启动的一开始就进行调试，那么就必须设置suspend=y。

接下来，重新启动线上服务，就可以在本地调试相关app了。

## <a id="Client">IDE本地配置</a>

本地使用的IDE是Intelij IDEA 开发工具，具体操作步骤为：

1、 在IDEA上面新建一个 tomcat remote server服务：
<img src="/images/2014/04/newremote.png" />
> Note:图片中的端口是web服务的端口号，而不是JVM监听绑定的端口号。

<img src="/images/2014/04/debugaddress.png" />
> Note:图片中的端口是JVM监听绑定的端口号，即我们在服务端设置绑定的address值。

2、 接下来就可以通过debug来启动本地服务，当出现下面字样时，表示连接成功，可以debug了。

	client：  
		Connected to server
		Connected to the target VM, address: 'l-hds2.h.dev.cn6.qunar.com:9999', transport: 'socket'  
	server：  
		Listening for transport dt_socket at address: 9999  
		Listening for transport dt_socket at address: 9999  
		Listening for transport dt_socket at address: 9999
