---
layout: post
title: "Nginx 反向代理配置和工作原理"
date: 2015-08-30 18:52:03 +0800
comments: true
categories: 
  - Nginx
  - Script
  - Linux
---

* content
{:toc}

## <a id="Intro">前言</a>

`Nginx`是一款面向性能设计的HTTP服务器,其性能相对于其他服务器表现优异。内部使用异步的事件处理模型，比如linux平台的`epoll`事件模型，unix平台的`kqueue`事件模型等。在Nginx源码的`src/event/modules`目录下，其对各个平台不同的异步模型进行了二次封装。此外，Nginx在代码实现的时候，会考虑到众多细节优化。比如：根据CPU亲缘性来分配进程和事件，避免CPU级的缓存失效；比如字符串比较时，四字节转换为整数来进行快速指令级比较，等等。

本博文主要目的不是Nginx源码分析，所以，对源码及其独特优秀的代码设计不会去详细介绍。

在最近的一些项目中，涉及到nginx的反向代理配置，然后花了一些时间了解下关于Nginx的整体请求处理流程和返现代理的实现机制。

Nginx虽然代码整洁，模块清晰，但是代码量毕竟还是很多，而且注释实在是太少，所以把一些学习的资料和心得整理一下，以便以后查看。

## <a id="Proxy">Nginx 反向代理配置说明</a>

反向代理指以代理服务器来接受Internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给Internet上请求连接到客户端，此时代理服务器对外就表现为一个服务器，而此种工作模式类似于LVS-NET模型。

反向代理也可以理解为web服务器加速，它是一种通过在繁忙的web服务器和外部网络之间增加的 一个高速web缓冲服务器，用来降低实际的web服务器的负载的一种技术。反向代理是针对web服务器提高加速功能，所有外部网络要访问服务器时的所有请求都要通过它，这样反向代理服务器负责接收客户端的请求，然后到源服务器上获取内容，把内容返回给用户，并把内容保存在本地，以便日后再收到同样的信息请求时，它会将本地缓存里的内容直接发给用户，已减少后端web服务器的压力，提高响应速度。因此Nginx还具有缓存功能。

<!-- more -->

了解nginx的反向代理如何实现之前，先看看我们一般配置nginx反向代理的设置：

``` 

    upstream cc_001 {
        server 192.168.1.101:80;
        server 192.168.1.102:80;

        healthcheck_enabled;
        healthcheck_delay 3000;
        healthcheck_timeout 1000;
        healthcheck_failcount 2;
        healthcheck_send 'GET /healthcheck.html HTTP/1.0' 'Host: local.com' 'Connection: close';
    }

    server {

        listen       192.168.1.100:80;
        server_name  cc.local.com;

        proxy_buffers 64 4k;

        location = / {
            proxy_pass http://cc_001/bm/index.htm;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

        location / {
            proxy_pass http://cc_001;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
    }

```

上面的code主要列出来nginx 反向代理基本配置。

>> Tips：上面的配置选项都是最最基本的，一般涉及到反向代理都会使用到这些配置。对于其中的设置项，理解起来也很简单。

*upstream配置块*

其实在nginx中，`upstream`是一个非常重要的配置。nginx所有对于动态请求的处理，基本上都需要使用`upstream`配置模块。nginx的两个很重要的功能，反向代理和负载均衡，都需要通过配置对应的`upstream`来完成。

>> 其实在nginx中，有一个基础模块叫handler，这个模块可以接受来自客户端/用户端的请求，然后处理并产生对应的响应内容返回过去。因此，我们那些静态资源，前端页面什么的，都是使用handler模块来完成响应的。但是，众所知周，一般的核心服务都是后台动态产生的，这些资源就不可以方便使用handler去完成内容的生成和响应动作（当然也是可以使用开发自定义handler来完成的，比如各种xxxcgi之流，但是一般还是用来处理静态资源）。
>> 
>> 那么，upstream就出现了。其接收到用户的请求，然后转发到后端服务器拿到对应的响应资源，再返回给请求端。在整个处理过程中，其本身不会产生自己的响应内容，这是和`handler`模块唯一的区别。
>> 
>> upstream的特性，决定了在其配置块中，设置一些后端服务器的地址和端口，就ok了。
>> 

配置项说明：

*  upstream中的server项：表明后台的一台服务器地址和端口。当客户端有请求到`nginx`服务器的时候，upstream模块根据这里配置的server，该对应的请求转发到这些server服务上，由这些server来处理请求，然后把响应结果告知upstream模块。

*  healthcheck_enabled项：healthcheck健康监控功能，并不是原生nginx自带的。所以如果使用这个功能，必须要安装第三方插件：`ngx_http_healthcheck_module`。healthcheck_enabled表示启动健康检查模块功能。

*  healthcheck_delay项：对同一台后端服务器两次检测之间的时间间隔，单位毫秒，默认为1000。

*  healthcheck_timeout项：进行一次健康检测的超时时间，单位为毫秒，默认值2000。

*  healthcheck_failcount项：对一台后端服务器检测成功或失败多少次之后方才确定其为成功或失败，并实现启用或禁用此服务器。

*  healthcheck_send项：为了检测后端服务器的健康状态所发送的检测请求。然后根据各个服务器的响应情况来判断服务器是否存活。上面的配置表面，各个后台服务器上都存在`healthcheck.html`静态页面，然后nginx会get这个页面，根据是否status为200来判断是否服务器存活。

*server配置块*

在nginx中，不管怎么样的配置，都会有一个server配置块。http服务上支持若干虚拟主机。每个虚拟主机会有一个对应的server配置项，配置项里面包含该虚拟主机相关的配置。在提供mail服务的代理时，也可以建立若干server.每个server通过监听的地址来区分。

>> Server其实就是一个虚拟主机。因为在nginx中可以配置多个server，这样就使得nginx可以在一台服务器上配置多个域名。
>> 
>> 在nginx的Server虚拟主机中，它只会处理与之对应的域名请求。并且，如果在listen中设置了ip地址，则该虚拟主机只会处理从该服务器的指定ip端口进来的请求，才会去处理。关于一台服务器设置多个别名ip地址的方式，可以参考博客[在Nginx中部署基于IP的虚拟主机](http://www.cnblogs.com/mchina/archive/2012/05/21/2511824.html)

配置项说明：

*  listen项：监听ip和端口。当nginx服务器的该ip端口有请求访问，则调用该server的配置来处理该请求。

*  server_name项：域名。nginx对进入该虚拟主机的请求，检查其请求Host头是否匹配设置的server_name，如果是，则继续处理该请求。

*  location块选项：Location在nginx中是一个非常重要的指令。对于HTTP请求，其被用来详细匹配URI和设置的location path。一般这个uri path会是字符串或者正则表达式形式。

>>: 关于location匹配，存在一些语法规则，如下：
>>
>>       location [=|~|~*|^~|@] /uri/ { ... }
>>        =：表示精确匹配，如果找到，立即停止搜索并立即处理此请求。
>>        ~：表示区分大小写匹配。
>>        ~*：表示不区分大小写匹配。
>>        ^~：表示只匹配字符，串不查询正则表达式。
>>        @：指定一个命名的location，一般只用于内部重定向请求。

*  location中proxy_pass项：代理转发。配置了该项，当匹配location path的请求进来后，会根据upstream设置，请求后台服务器上的proxy_pass的请求。例如，上面的配置，当有请求`cc.local.com`时，由于精确匹配`=/`，则根据proxy_pass配置，则会反向代理，请求`192.168.1.101:80/bm/index.htm`。

*  location中proxy_set_header项：设置代理请求头。由于经过了反向代理服务器，所以后台服务器不能获取真正的客户端请求地址等信息，这样，就需要把这些ip地址，设置回请求头部中。然后，我们在后台服务上，可以使用`request.get("X-Real-IP")`或者`request.get("X-Forwarded-For")`获取真实的请求ip地址。获取host也是如此。具体可以参考博文：[ 使用nginx后如何在web应用中获取用户ip及原理解释](http://gong1208.iteye.com/blog/1559835).


## <a id="ProcessRequest">Nginx 架构和请求处理流程</a>

Nginx架构，在taobao的[《Nginx开发从入门到精通》](http://tengine.taobao.org/book/chapter_02.html)电子书中，写的比较详细。这里记录一些核心的细节。

Nginx在启动会以daemon形式在后台运行，采用`多进程+异步非阻塞IO事件模型`来处理各种连接请求。

Nginx主要包含一个master进行和多个worker进行，一般worker进程个数是根据服务器CPU核数来决定的。如下图：

<img src="/images/2015/08/nginx_process.png" />

>> Notes：从上图中可以很明显地看到，4个worker进程的父进程都是master进程，表明worker进程都是从父进程fork出来的，并且父进程的ppid为1，表示其为daemon进程。
>> 
>> 需要说明的是，在nginx多进程中，每个worker都是平等的，因此每个进程处理外部请求的机会权重都是一致的。
>> 

下面来介绍一个请求进来，进程模型的处理方式。

*首先*，master进程一开始就会根据我们的配置，来建立需要listen的网络socket fd，然后fork出多个worker进程。

*其次*，根据进程的特性，新建立的worker进程，也会和master进程一样，具有相同的设置。因此，其也会去监听相同ip端口的套接字socket fd。

*然后*，这个时候有多个worker进程都在监听同样设置的socket fd，意味着当有一个请求进来的时候，所有的worker都会感知到。这样就会产生所谓的`惊群现象`。为了保证只会有一个进程成功注册到listenfd的读事件，nginx中实现了一个`accept_mutex`类似互斥锁，只有获取到这个锁的进程，才可以去注册读事件。其他进程全部accept 失败。

*最后*，注册成功的worker进程，读取请求，解析处理，响应数据返回给客户端，断开连接，结束。因此，一个request请求，只需要worker进程就可以完成。


>> 进程模型的处理方式带来的一些好处就是：进程之间是独立的，也就是一个worker进程出现异常退出，其他worker进程是不会受到影响的；此外，独立进程也会避免一些不需要的锁操作，这样子会提高处理效率，并且开发调试也更容易。
>> 
>> 如前文所述，`多进程模型+异步非阻塞模型`才是胜出的方案。单纯的多进程模型会导致连接并发数量的降低，而采用异步非阻塞IO模型很好的解决了这个问题；并且还因此避免的多线程的上下文切换导致的性能损失。
>> 
>> 关于异步非阻塞IO模型：linux的epoll介绍，可以参考：[深入了解epoll ](http://www.cppblog.com/deane/articles/165218.html)

### Nginx 连接和请求处理

上一节介绍了，worker进程会竞争客户端的连接请求，这种方式可能会带来一个问题，就是可能所有的请求都被一个worker进程给竞争获取了，导致其他进程都比较空闲，而某一个进程会处于忙碌的状态，这种状态可能还会导致无法及时响应连接而丢弃discard掉本有能力处理的请求。这种不公平的现象，是需要避免的，尤其是在高可靠web服务器环境下。

针对这种现象，Nginx采用了一个是否打开accept_mutex选项的值`ngx_accept_disabled`。标识控制一个worker进程是否需要去竞争获取accept_mutex选项，进而获取accept事件。

>> ngx_accept_disabled值，nginx单进程的所有连接总数的八分之一，减去剩下的空闲连接数量，得到的这个ngx_accept_disabled。
>> 
>> 当ngx_accept_disabled大于0时，不会去尝试获取accept_mutex锁，并且将ngx_accept_disabled减1，于是，每次执行到此处时，都会去减1，直到小于0。不去获取accept_mutex锁，就是等于让出获取连接的机会，很显然可以看出，当空余连接越少时，ngx_accept_disable越大，于是让出的机会就越多，这样其它进程获取锁的机会也就越大。不去accept，自己的连接就控制下来了，其它进程的连接池就会得到利用，这样，nginx就控制了多进程间连接的平衡了。

接下来，看看连接处理流程（来自tengine.taobao.org）：

<img src="/images/2015/08/request_process.png" />

>> 关于处理流程的说明，参考: <http://tengine.taobao.org/book/chapter_02.html>
>> 

## <a id="ImplementationStudy">Nginx Upstream模块和Location配置</a>

*Nginx Upstream*

upstream模块实现反向代理的功能，将真正的请求转发到后端服务器上，并从后端服务器上读取响应，发回客户端。

从本质上说，upstream属于handler，只是他不产生自己的内容，而是通过请求后端服务器得到内容，所以才称为upstream（上游）。请求并取得响应内容的整个过程已经被封装到nginx内部，所以upstream模块只需要开发若干回调函数，完成构造请求和解析响应等具体的工作。

`upstream`模块逻辑实现的十分复杂，对于其具体实现，不分析。

`upstream`模块主要做两件事情：

* 当外部的客户端发送一个http请求后，如果涉及更后台服务，则会创建一个到后端服务的request请求；

* 请求到达后端，然后处理完成后，则upstream会将返回的数据接收过来，然后发送给外部请求的客户端。

*Nginx Location*

首先，介绍下存在的几种Location配置方式：

```

location  = / {
  # matches the query / only.
  [ configuration A ] 
}
location  / {
  # matches any query, since all queries begin with /, but regular
  # expressions and any longer conventional blocks will be
  # matched first.
  [ configuration B ] 
}
location /documents/ {
  # matches any query beginning with /documents/ and continues searching,
  # so regular expressions will be checked. This will be matched only if
  # regular expressions don't find a match.
  [ configuration C ] 
}
location ^~ /images/ {
  # matches any query beginning with /images/ and halts searching,
  # so regular expressions will not be checked.
  [ configuration D ] 
}
location ~* \.(gif|jpg|jpeg)$ {
  # matches any request ending in gif, jpg, or jpeg. However, all
  # requests to the /images/ directory will be handled by
  # Configuration D.   
  [ configuration E ] 
}

```

示例请求：

*      / -> configuration A
*     /index.html -> configuration B
*     /documents/document.html -> configuration C
*     /images/1.gif -> configuration D
*     /documents/1.jpg -> configuration E 

解析匹配规则为：

1. 字符串精确匹配到一个带 “=” 号前缀的location，则停止，且使用这个location的配置；

2. 字符串匹配剩下的非正则和非特殊location，如果匹配到某个带 "^~" 前缀的location，则停止；

3. 正则匹配，匹配顺序为location在配置文件中出现的顺序。如果匹配到某个正则location，则停止，并使用这个location的配置；否则，使用步骤2中得到的具有最大字符串匹配的location配置。

>> Notes：需要注意的是：`~ 开头`表示区分大小写的正则匹配；而`~*  开头`表示不区分大小写的正则匹配。`!~和!~*`分别为区分大小写不匹配及不区分大小写不匹配的正则

## <a id="End">后记</a>

Nginx 是一个十分优秀的服务器软件，其内部相当多的设计和实现都非常巧妙和高效。

关于Nginx的一些好的站点有：

- <http://tengine.taobao.org/book/>
- <http://www.pagefault.info/?cat=7>
- <http://nginx.org/en/docs/>
- <http://kxcoder.github.io/images/2015/08/nginx_stream.png>