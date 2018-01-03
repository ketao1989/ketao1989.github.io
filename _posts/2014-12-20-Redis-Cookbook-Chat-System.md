---
layout: post 
categories: 
  - Redis
  - Redis Cookbook
title: "Redis Cookbook 之 基于Redis 实现一个聊天系统"
date: 2014-12-20 22:21:35 +0800
comments: true
---

##  <a id="Problem">问题</a>

想要借助 `Redis`的`PUB/SUB`功能，使用node.js和Socket.io实现一个轻量级的实时聊天系统。

##  <a id="Solution">解决方法</a>

由于Redis 天生就支持发布订阅(pub/sub)模式，所以我们可以很容易就使用`Node.js` 和 `Socket.IO`来快速创建一个实时的聊天系统。

发布订阅模式，其实就是接收者订阅某种特定模式的消息(比如，发送到某个指定channel的消息)，而发送者发送一个消息到消息云上。当一个消息到达云上的时候，订阅了这一种类的客户端就会获得消息。这中发布订阅模式，然后就可以允许发送者和接收客户端在不知道彼此的情况下，亲密结对交流。而他们仅仅需要以一种既定的模式发送消息和接收匹配类型的消息即可。

`Redis`直接支持`pub/sub`模式，意味着其可以让接收客户端订阅指定的匹配消息频道channel，以及发布消息到一个给定的频道channel。这意味着，我们可以很简单地创建像`chat:cars`的聊车频道；或者像`chat:sausage`这种关于食物的谈话。此外，频道channel的命名跟Redis 的keySpace无关，所以不用担心会存在某些冲突情况。下面给出，Redis支持的一些命令：

        * PUBLISH：发布消息到指定的频道；

        * SUBSCRIBE：订阅一个指定频道的消息；

        * UNSUBSCRIBE：取消订阅一个指定频道；

        * PSUBSCRIBE：订阅一个满足给定模式的频道集；

        * PUNSUBSCRIBE：取消订阅满足指定模式的频道集。

拥有上面这些知识，为在应用程序逻辑部分之间的终端用户或者流消息实现一个聊天和统计系统，其实还是很琐碎的。
`pub/sub`甚至可以被用来作为一个内建的强壮阻塞队列系统。接下来看看，如何去实现这么一个消息聊天系统吧。

在服务端，`Node.js` 和 `Socket.IO`将来实现网络层，然后Redis将作为一个在客户端之间递交消息的`pub/sub`功能的实现。在客户端，我们使用jQuery来处理消息，然后发送数据到服务器上。

<!-- more -->

##  <a id="Discussion">讨论</a>

由于本文使用Node.js来实现一个聊天系统，所以我们假设你已经安装了node.js，并且我们也希望你可以按顺序安装支持我们聊天系统所需要的`node库(Socket.IO and Redis)`。

### 初始设置
安装所需要的第三方库：

        npm install socket.io

        npm install redis

### 服务端代码实现

在服务端，我们正在运行`Redis`并且创建了一个运行node.js的javaScript文件。该代码主要负责建立到Redis服务之间的链接conn，然后一直监听来自clients端连接请求的端口。因此，我们创建一个javascript代码文件`chat.js`:

``` js

        
    var http = require('http'), 
    io = require('socket.io'), 
    redis = require('redis'), 
    rc = redis.createClient();

```

上面的代码，可以建立redis连接，和引入http,socket.io,redis库。接下来，我们需要设置一个简单地server，让客户端可以连接，请求数据：

``` js

        
    server = http.createServer(function(req, res){
        // we may want to redirect a client that hits this page 
        // to the chat URL instead
        res.writeHead(200, {'Content-Type': 'text/html'}); 
        res.end('<h1>Hello world</h1>');
    });
    
    // Set up our server to listen on 8000 and serve socket.io server.listen(8000);
    var socketio = io.listen(server);

```

接下来，建立连接了就可以开始使用node.js来完成开发连接redis，客户端订阅某个channel，接收到消息处理动作等功能。所以，接下来使用redis来完成订阅消息：

``` js

        
    // if the Redis server emits a connect event, it means we're ready to work, 
    // which in turn means we should subscribe to our channels. Which we will. rc.on("connect", function() {
        rc.subscribe("chat");
        // we could subscribe to more channels here 
    });
        // When we get a message in one of the channels we're subscribed to, // we send it over to all connected clients.
    rc.on("message", function (channel, message) {
        console.log("Sending: " + message);
        socketio.sockets.emit('message', message); 
    })

```

ok，如你所见，这段代码非常简单。其实现，就是我们在特定的channel监听消息，当有消息接收到的时候，服务端就广播给所有订阅该消息的客户端。

### 客户端代码实现

完成了server端部分的开发，接下来完成一个小页面来连接Node.js，建立客户端的Socket.IO，然后处理进来和出去的消息。所以我们创建了一个很简单的`HTML5`页面：

``` html

        
    <!doctype html> <html lang="en"> <head>
    <meta charset="utf-8">
    <title>Chat with Redis</title> </head>
    <body>
    <ul id="messages">
    <!-- chat messages go here --> </ul>
    </body> </html>

```

现在我们需要引入两个非常重要的库来获得想要的功能：jQuery 和 Socket.IO：

``` html

        
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.6.1/jquery.min.js"></script> 
    <script src="http://localhost:8000/socket.io/socket.io.js"></script>

```

现在我们准备好了从页面连接Node.js，然后开始监听处理消息。在页面的头部增加下面的代码：

``` html

        
    <script>
    var socket = io.connect('localhost', { port: 8000 });
    socket.on('message', function(data){
        var li = new Element('li').insert(data);
        $('messages').insert({top: li}); 
    });
    </script>

```

这个javascript代码片段表示，客户端使用`Socket.IO`连接我们的node.js实例8000端口，然后开始监听消息事件。当一个消息到达时，它创建一个新的list元素，并且把它添加到我们事先已经建好的未排序list中。

到这里，还剩下的，就是客户端发送消息了。和server端一样，我们使用`Socket.IO emit`方法，如下所示：

``` html

        
    <form id="chatform" action="">
        <input id="chattext" type="text" value="" /> 
        <input type="submit" value="Send" />
    </form>
    
    
    <script> 
    $('#chatform').submit(function() {
        socket.emit('message', $('chattext').val()); 
        $('chattext').val(""); // cleanup the field 
        return false;
    }); 
    </script>

```

当一个用户写东西到`form`中，然后点击`Send`，jQuery将会使用我们的socket变量emit发送一个消息事件到服务器端，服务器然后会广播这条消息给其他所有人。最后返回false表示消息事件真的被发送提交出去了。提交的这个动作是由`Socket.IO`完成的。


