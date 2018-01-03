---
layout: post
title: "深入浅出Java服务端原理之基础篇"
date: 2017-03-29 20:52:39 +0800
comments: true
categories: 
  - java
  - socket
  - thrift
---

>上篇文章介绍了[深入浅出RPC一些原理知识](http://ketao1989.github.io/2016/12/10/rpc-theory-in-action/)，这里将继续深入讲讲关于RPC网络服务技术。本文的目标，是让对Java网络服务端开发感兴趣的新人，可以一步步深入了解一个高性能服务所需要的相关知识体系。
>
> 当然，高性能高并发的服务端开发，涉及的知识点和手段众多，比如linux网络和内核参数调优等。本文只关注代码开发层级的介绍和深入。

## 前言

本文中，所谓 `服务端`指的是，基于client/server服务组件中的server端。在网络世界里，基本上任何一个用户交互都涉及客户端和服务器端网络通信；尤其是，当存在对服务端大量请求时，如果让服务器持续对外服务过程中，抗住更多的并发请求，是一个互联网开发同学十分关心和必备的技能。

本文中，首先会基于Java网络API开发一个`HelloWorld`版本的server端版本。  
接下来，会不断的在前一个server版本上进行优化和升级，使得其可以响应更多的网络连接请求。   
在此期间，会引入一些概念，比如IO同步异步、阻塞非阻塞。  
最后，我们要站在巨人的肩膀上，基于Netty来开发一个生产级别的server端应用。

<!-- more -->

## 网络协议介绍

> 服务器开发，是基于网络协议基础上的，虽然Java 网络API已经封装了协议细节，但是在分析程序运行中出现的一些网络问题，优化服务通信性能时，还是很有用的。
>
> 这里主要介绍传输层的两大协议：TCP和UDP协议。（深入的介绍，请参考 《TCP/IP 详解 卷1：协议》这本经典书籍。）一般应用定制所谓自定义的网络协议，基本上都是基于传输层来实现的。

### TCP 协议

>  传输层主要有两大协议：面向连接的TCP (Transmission Control Protocol)协议和面向无连接的UDP (User Datagram Protocol)协议。相对于UDP协议，TCP为了保证连接的可靠性，其协议实现细节是相当复杂的。

#### 三次握手/四次握手

既然TCP是面向连接的协议，那就存在如何建立连接以及如何断开连接的问题。

> 在介绍TCP连接协议前，需要特别指出：TCP所谓的处在连接状态`ESTABLISHED`，只是说明在通信双方建立的`连接状态表`中，当前的TCP链路是连接的。而且，通信双方的状态表，并不保证是一致的。
> 所以，在使用TCP协议的业务里，都需要有一个定时器来发送一个特殊的段给对等方检测当前链路是否依然处于连接状态，也就是`存活检测`。在后面的介绍中，还会有应用级别的`存活检测`。


首先，在所有的介绍TCP协议的文章里都会有的握手图（来自网络，参考文献1）：

![Alt text](/images/2017/tcp_open_close.jpeg)

所谓三次握手，就是在通信双方建立TCP连接的时候，需要双方共发送3个数据包来确认双方为通信都准备OK。

1. 第一次握手。客户端client首先初始化一个SYN序列号x，一般第一次客户端和服务端握手时，SYN初始化为0，随后每次都会加上传输数据的字节作为对端新的SYN序列号。Client将产生的SYN序列化发送给Server，并将自己的状态置为`SYN_SENT`。
2. 第二次握手。服务端Server接收到Client发送过来的连接请求包，由于在建立TCP连接的请求包中并没有有效数据(设置FIN和SYN这种标志位的包，是不计算有效数据的)，所以Server发送的握手包中，SYN序列号为0（Server对这次建立连接也是第一次），以及ACK确认号1=0+1。此外，服务器Server将自己的状态设置为`SYN_RCVD`。
3. 第三次握手。客户端Client接收到Server发送回来的确认包，根据服务端ACK号和SYN号，发送自己的最后一个连接数据包，其中和第二次握手相似，SYN序列号为1，ACK为1=0+1。并将自己的连接状态设置为`ESTABLISHED`。

在TCP通信过程中，ACK数据，是根据对方SYN序列号，再加上本次数据包有效数据大小来计算的，而下次对端发送SYN序列号的值就是其收到的ACK值。SYN 和 ACK的逻辑，保证了TCP重传机制。

> 在了解TCP建立连接进行的三次握手过程之后，肯定会有疑问，为什么是3次，而不是2次或者4次呢？
>  
>  3次考虑，是在满足日常应用场景中，所需要的最少次数。1次肯定不行，如果是2次呢？假设 C 向 S 发送连接请求，当时这个请求没有到达 S 。然后 C 重新发送一个连接请求，S接收到并且发送响应包，然后连接建立。正常数据交换完了之后，关闭连接。但是，这个时候，第一次的连接请求发送了过来，S 建立连接，一直处于等待数据交互中。 
>  或者，C 向 S 发送连接请求， 然后 S 会发送一个响应包，可能 C 可能没有收到这个响应包。这个时候，S 可能以为 C 收到了ACK包，然后S给C发送数据分组包，但是 C 一直在等待 S 的ACK应答分组包，并把 S 的数据分组报丢弃。
>  其实，2次握手，最重要的是，在双方都以为建立连接的情况下，可能存在无法对S的序列号达成一致。但是，3次握手，就可以确认好 C 和 S 的初始序列号一致。
>  
>  以上可以阅读参考文献2。

所谓的四次握手（也有叫做四次挥手的），就是处在连接状态的通信双方，正常断开连接所需要进行的四次数据包交互流程。由于TCP是全双工，因此通信双方都需要进行关闭，所以会进行4次握手。
1. 第一次握手。客户端将发送FIN包给服务端，关闭连接。FIN包中，FIN序列号和ACK值与前面数据组交互一样。此时，客户端处于`FIN_WAIT_1`状态。
2. 第二次握手。服务端收到客户端的FIN包之后，只会发回一个ACK包，表示收到关闭连接请求。此时，服务端处于`CLOSE_WAIT`状态
3. 第三次握手。作为全双工链路，服务端也需要发送FIN包来关闭自己这方的连接链路。由于ACK已经发送，所以这一次只需要发送FIN序列号。此时，服务端处于`LAST_ACK`状态。
4. 第四次握手。客户端现在收到了服务端发送的FIN包，客户端发送ACK表示自己收到了服务端的关闭请求。此时，客户端处于`TIME_WAIT`状态；而服务端收到客户端ACK包之后，则设置自己状态为`CLOSED`。客户端的`TIME_WAIT`状态会停留两倍的MSL时长。

> MSL指的是报文段的最大生存时间，如果报文段在网络通信中存活时间超过MSL时间，还没有被接收，那么会被丢弃。关于`TIME_WAIT`深入学习可以参考文献3

#### TCP 重传机制

> TCP重传机制，是TCP协议保证数据可靠性的一个重要的机制。在网络环境不好的情况下，重传机制可以对上层应用层，屏蔽掉因为各种网络问题导致超时数据丢失而进行重试的策略细节。但是，正式由于屏蔽，导致很多时间排查问题时会忽略掉这点。
>
> 此外，其实在很多系统设计过程中，重传重试相关的需求还是蛮多的。学习TCP重传机制，可以给我们在设计实现自己的应用系统时提供一些借鉴。

TCP重传机制，需要确认什么时候需要重传，以及重传哪些数据包。

TCP需要重传，肯定是在数据包丢失之后才需要重传，将发送端认为服务端未收到的报文重新发送给接收端。但是，在不可靠的网络环境下，你根本就无法确认数据包是否丢失，通过TCP ACK机制只能确保哪些数据报文被通信对方收到，但是由于接收报文是乱序的，所以当前未收到的报文可能在未来某个时候被ACK回来，也可能就真的被丢失了。

因此，TCP在发送报文之后，会开启一个定时器，然后如果在计算的超时时间内未收到ACK，则重传。所以，TCP重传机制，有的时候也被称为 TCP超时重传机制。

超时重传机制，包含两个重要的时间参数。
+ 往返时间RTT。其是发送端发送TCP包开始计算，到接收到该包ACK数据结束，这期间所消耗的时间。
+ 超时重传时间RTO。其是，发送端发送TCP包之后，确认包丢失，然后决定重新发送该包的时间。

显然，我们需要的是RTO时间，根据这个时间我们来确认是否应该超时重传了。但是RTO是动态计算出来的，也就是我们需要根据当前网络状况的不同，计算出RTO时间。因此，就说到了RTT，异常的超时时间，根据正常包往返时间来动态计算，是比较合理的。RTO初始的计算方式，就是取若干次RTT的平均时间，最小200ms，后面随着重传次数的变化，RTO也会调整。

> 因此，整个流程是这样子的：发送端发送TCP报文，并且启动该报文对应的重传计时器，如果收到了ACK报文，则结束计时器，并且获得最新一次的RTT，计算RTO；如果没有收到ACK报文，并且这时间已经超过当前RTO设置，则重传报文，并且RTO时间倍增。如果，倍增之后，ACK报文还未收到，则继续倍增，直到收到或者到达设置的最大RTO超时时间。
>  
>  在LINUX中，重传此时一般为15次。

此外，还有一种也会触发报文重传。比如，发送端先后发送了A：`seq=100,size=100`；B：`seq=200,size=200`；C：`seq=400,size=150` 三个报文，然后接收方，返回了A'：`ack=200`；B'=`ack=200`；C'=`ack=200`。那么这个时候，就不会等待RTO超时了，而是发送端会认为接收端明显没有收到seq=200的报文，立刻触发`快速重传机制`，发送丢失的数据报文。

#### TCP 滑动窗口

> 除了重传机制，TCP另一大亮点，就是滑动窗口。其滑动窗口可以很好的保护系统可靠性，避免大流量数据导致大量传输失败，限制传输速度以及网络带宽和服务器资源的有效利用。在实际应用系统设计中，也可以充分借鉴参考。

TCP 使用接收方接收窗口来控制通信的数据流速率，从而达到拥塞控制，避免通信过程中不必要的丢包处理。

首先来看下TCP报文头结构：

![Alt text](/images/2017/tcp_header.gif)

如上图所示，在20字节长的报文头里面，有16bit是接收方用来告诉客户端其可接收数据的大小，发送方根据这个数据，来控制发送给接收方的数据长度。

看一个简单的例子了解下：

![Alt text](/images/2017/tcp_window.png)

首先，在和接收端最后一次通信的时候，发送端从TCP报头中获取了剩余窗口为500，表示发送端可以发送500字节的数据给接收方；
然后，发送端发送400字节数据，其中seq=1到seq=200之间的数据已经ACK确认，seq=201到seq=400发送出去但是还未被确认。于是，可发送滑动窗口头部从`1-->201`，尾部从`500-->700`，如上图(b)；
接着发送端收到了接收端seq=201到seq=400的ACK报文，这个时候，滑动窗口继续向后滑动。

> 需要指出一种特殊的情形，就是当发送端发送数据过快，接收端还未来得及处理时，就会出现接收端返回报头中窗口为0的情形，这就是`Zero Window`。显然，这种情况，发送端会停止发送数据，知道窗口不为0。然后就有了`ZWP`技术，当自己的大小不为0，接收端ACK他的窗口大小给发送端 。

但是，基于滑动窗口会出现一个问题，就是当接收端处理速度比较慢，可能每次就只能处理很小字节的报文，然后告诉发送端，其剩余窗口为很小的数字。但是，一个TCP+IP头是40个字节，显然，如果窗口不为0就发送报文给接收端，是不可取的。

那如何避免在窗口很小的时候，频繁发送小传输报文给接收端呢？在TCP中，称为Silly Window Syndrome(糊涂窗口综合症)。解决方案可以是在特定时刻接收端ACK窗口为0或者发送端累计一部分报文之后在发送给接收端。

一般，我们在发送端控制比较方便。因此，就出现了`Nagle算法`。首先引入MSS概念：
+ MSS：Maximum Segment Size，也就是最大分段大小。为了达到最佳的传输效能，TCP协议在建立连接的时候通常要协商双方的MSS值，这个值TCP协议在实现的时候往往用MTU（最大数据包大小，以太网的MTU为1500）代替（需要减去IP数据包包头的大小20Bytes和TCP数据段的包头20Bytes），所以一般MSS值1460。

Nagle 算法的规则：
```
 [1]如果包长度达到 MSS ，则允许发送；
 [2]如果该包含有 FIN ，则允许发送；
 [3]设置了 TCP_NODELAY 选项，则允许发送；
 [4]设置 TCP_CORK 选项时，若所有发出去的小数据包（包长度小于 MSS ）均被确认，则允许发送；
 [5]上述条件都未满足，但发生了超时（一般为 200ms ），则立即发送。
```
> 从上面的规则可以看到，我们再开发应用层代码的时候，如果应用场景需要实时发送各种小报文数据，则需要将Socket的TCP_NODELAY设置为true，否则可能会由于报文太小，而导致数据一直未发送出去。

滑动窗口目前看来，控制发送端发送速率来保护接收端系统是足够了的；但是，滑动窗口，还需要满足解决网络拥塞控制，因此需要更进一步。

上文介绍过TCP通过采样了RTT时间然后计算出RTO，作为重传超时时间。但是，如果网络上的延时突然增加，那么，TCP对这个事做出的应对只有重传数据，显然，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，最后，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的TCP连接都这么行事，那么马上就会形成“网络风暴”，TCP这个协议就会拖垮整个网络。

因此，就出现TCP Reno算法，包含4个部分：
1. 慢热启动算法 – Slow Start。就是在发送端刚刚接入网络的时候，不会立刻将发送报文量提升到很大的值，而是慢慢试探网络，以每个RTT`X2`的指数来提速，或者当收到一个ACK则`+1`提速。在Linux 3.0 中，初始的发送大小为10个MSS。
2. 拥塞避免算法 – Congestion Avoidance。当指数增长到最大阈值ssthresh=65535，则会对发送速率进行调整ssthresh为当前速率/2。然后回到慢启动过程。此外，很可能在这个时候会出现RTO超时情况，然后进入快速重传。
3. 快速重传 - Fast Retransimit。调整ssthresh为当前速率/2，并且当前速率也调整为当前速率/2。然后进入拥塞避免阶段。
4. 快速恢复算法 – Fast Recovery。

基于以上的算法来调整发送端发送报文的速率，可以很好感知网络当前的负载情况，将网络延时导致重传超时作为一个影响因子，来控制网络拥塞。

> 其实，在使用很多工具的时候，都会明显感觉到`Reno`算法的存在。比如在下载文件的时候，下载速度都是逐步增加到一定速率。
>  关于TCP具体拥塞算法，可以参加文献4

#### TCP 存活检测

> 上面讲的东西，都比较理论，应用开发中相对用的比较少。这里，介绍一个在开发应用层网络程序时，需要考虑的知识点。

在前文中说过，TCP由于其内部只是维护了本身的状态表，并不能实时通知当前网络断开的消息。之所以，TCP不去提供这个实时通知网络变化的原因，有2个：
1. 这样会消耗掉大量的网络带宽，试想若存在着大量的不成熟的网络应用程序，网络带宽一定会消耗殆尽；
2. 在TCP设计之初，美国国防部设计TCP就是为了让在网络中断的情况下仍然通过其它途径维持通信的能力。
不过，在Linux中，提供了`KEEP_ALIVE`机制去检测TCP的存活状态。

这种KEEP_ALIVE机制下，TCP会在连接空闲一定时间间隔（一般时间为 7200 s，失败后重试 10 次，每次超时时间 75 s。）后发送一个特殊的段给通信对方，若对方系统依然运行，对应端口对外接收数据，则会响应并发送一个ACK消息。

KEEP_ALIVE 可以很好的检测连接是否存活。但是在实际应用中，其存在两个问题：
1. TCP默认的间隔时间有点过长，在我们日常环境中，使用这种默认时间来检测连接是否存活，是不行的。
2. TCP KEEP_ALIVE方式的存活检测，只是针对连接，而不是针对通信双方应用系统的。也就是，即使检测的连接是活跃的，但是可能对应的系统已经100%CPU，无法接收真实的应用处理了。

因此，一般的应用服务器实现中，都是使用自己的心跳机制来保证应用的存活。

> 应用心跳存活机制，是由应用定时周期发送心跳包给对方，这里就涉及到频率的问题了。如果时间设置的太长，则起不到实时感知网络是否端口。如果时间太短，则会导致数据量过大，增加网络和接收方负担。
> 此外，在移动智能设备占主导地位的今天，移动网络的不稳定，流量和电量的消耗，都要求心跳频率不能太频繁。
>  
>  因此，在APP端，一般如果应用正在前台使用，则心跳会相对频繁，比如30s一次，如果app被推到后台时，则心跳间隔时间可以调整大点，比如10分钟一次。

### UDP 协议

相对于TCP协议，UDP协议非常简单。UDP是面向无连接的，不需要在传送数据之前通过握手协议来建立连接；UDP是不可靠的协议，其不保证数据最后被交付到目的主机，因此也就没有ACK报文来告知发送端发送的数据是否成功。

此外，UDP如名字那样，面向报文的，也就是UDP包不会进行拆分或合并，没有TCP的拥塞控制和重传机制。重要的一点是，UDP报文头只占用8个字节，而TCP需要占用20个字节。

> 在TCP中，我们谈到，说TCP的连接其实只是在其双方内存表里面有个一对一连接状态的维护，并不是真的建立了一条链路。而，UDP无连接的，也只是说在UDP中不存在一个一对一连接链路状态的维护，UDP可以通过socket给多个UDP端发送数据，也可以接受多个UDP端发送过来的数据，如果UDP客户端和服务端都只接收和发送该一个socket的数据，那么可以说，其实“虚连接”的。
>  
>  不过需要注意的是，在TCP中，基于这个连接上有一堆的机制和算法来保证连接可靠性，这个是UDP没有的。所以TCP和UDP最大的不同，是一个是可靠的协议，一个是非可靠的。

UDP的不可靠，使得其适应一些特殊的应用场景。
1. 实时要求高的场景。比如视频通信。在视频直播过程中，用户是可以接受一些帧的数据包的丢失，相对而言，其无法接受一些过时的数据重放，因此业务基于UDP协议进行一些额外补充，满足一些实时性要求高的业务。
2. 在很早以前，由于国内的网络不是很好，导致TCP协议并不能很好的发挥其稳定性的特性，并且UDP无连接以及报头小等特性，使得服务器可以支持更高的用户量。具体可以参考文献5。

> 虽然，QQ使用的是UDP通信，TCP一个链路维护其他信息；但是对于现如今，这种设计模式并不一定性价比最高。所以，微信的通信，就是基于TCP来完成的，而不是UDP协议。


## Java Server入门

> 基于Java JDK提供的网络API开发简单的服务器是很便捷的，其封装的API接口对于开发者而已，非常易用。

### ECHO 服务端

和其他很多文章介绍网络编程一样，这里我们也首先实现一个echo服务器示例。

``` java
    public void startServer(int port) {
        try {
            //创建serverSocket来监听port
            ServerSocket serverSocket = new ServerSocket(port, 1);
            while (true) {

                // 建立一个socket连接
                Socket socket = serverSocket.accept();
                System.out.println("new connect......");

                try (InputStream is = socket.getInputStream(); OutputStream os = socket.getOutputStream()) {

                    // 连接成功，返回连接成功消息给客户端
                    os.write(("SERVER CONN OK!" + new Date().toString() + "\n\r").getBytes(Charset.defaultCharset()));
                    os.flush();

                    byte[] data = new byte[16];
                    int len;

                    while ((len = is.read(data)) != -1) {

                        // 读取数据，将数据打印出来
                        System.out.print(new String(data, 0, len, Charset.defaultCharset()));
                        // echo反馈给客户端
                        os.write(data, 0, len);
                        os.flush();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
> 代码展示的是一个单线程同步阻塞方式处理网络Socket IO操作的echo服务器版本，所以当客户端连接上服务器之后，其他客户端是不能同时操作网络IO的。
>  
>  此外，需要特别说明在 `public ServerSocket(int port, int backlog) throws IOException`方法中的backlog，在测试的时候，可以更清楚知道该参数含义：网络连接的等待队列的最大长度。

我们使用 Telnet模拟客户端进行测试，结果如下图所示：

![Alt text](/images/2017/socket_backlog.png)

> 在代码中，我们将backlog设为1，所以，如上图，当我们1个网络IO操作处理中，1个处在网络连接等待队列中，最后1个网络连接请求由于等待队列已满，从而`创建网络连接操作超时`而导致最终失败，所以会出现`Operation timed out`。

因此，我们看出来了这个版本的显著的缺点，就是无法同时处理多个网络请求。

### 多线程版本 ECHO服务端

显然，服务器只能同时处理单个客户端网络请求，肯定是不行的。从代码上分析，程序阻塞的地方，是服务器从socket流中获取数据，然后处理数据返回数据等同步阻塞的IO操作。由于是单线程来处理这些IO操作，所以线程阻塞，就会导致其他网络请求无法响应处理，因此，显然的，就会才有多线程来避免这个问题。

> 在实现多线程处理请求的时候，一般才有线程池来完成；并且，线程池的大小还需要进行一个限制，避免请求量过大，导致机器资源耗尽不可用。

``` java
public class ThreadPoolServer {

    private static final ExecutorService EXECUTOR_SERVICE = createDefaultExecutorService();

    public static void main(String[] args) {
        ThreadPoolServer server = new ThreadPoolServer();
        server.startServer(8080);
    }

    public void startServer(int port) {
        try {
            ServerSocket serverSocket = new ServerSocket(port, 4);

            // 设置超时无限大，保证在accept不阻塞
            serverSocket.setSoTimeout(0);

            for (; ; ) {
                Socket socket = serverSocket.accept();
               // 读超时设置无限大，保证不会断开client
                socket.setSoTimeout(0);
                try {
                    EXECUTOR_SERVICE.execute(new ProcessorRunnable(socket));
                } catch (Throwable t) {
                    socket.close();
                }
            }
        } catch (IOException e) {
        }
    }

    private static ExecutorService createDefaultExecutorService() {
        SynchronousQueue<Runnable> executorQueue = new SynchronousQueue<Runnable>();
        return new ThreadPoolExecutor(8, 16, 60, TimeUnit.SECONDS, executorQueue);
    }
    
    public class ProcessorRunnable implements Runnable {

        private Socket socket;

        public ProcessorRunnable(Socket socket) {
            this.socket = socket;
        }
        @Override
        public void run() {
            try (InputStream is = socket.getInputStream(); OutputStream os = socket.getOutputStream()) {
                // 同上文代码示例
                processData(is,os);
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                // 处理完成最后，如果socket是打开的，需要进行关闭
                if (socket != null && socket.isConnected()) {
                    socket.close();
                }
            }
        }}}
```

> 上面示例除了增加多线程支持多客户端同时访问外，实现的时候，对一些实现进行了优化和完善。比如线程池，socket资源关闭，超时设置等。

#### 设置 SO_TIMEOUT参数

**SO_TIMEOUT**是设置socket超时时间的。在server的代码中，ServerSocket.setSoTimeout(int) 和Socket.setSoTimeout(int)是不一样的。在ServerSocket中超时时间指的是accept新连接的超时时间，当服务器等待指定时间还未有新的连接请求过来，则会抛出SocketTimeout异常，并且停止接收新的socket连接请求，但是现有的socket连接读写操作服务器还会继续处理。如下所示：
``` java
java.net.SocketTimeoutException: Accept timed out
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
```
 与之不同，Socket的超时时间则是读read超时，当和服务器建立连接的客户端在超时时间内未写入新的数据到服务端，则服务器会抛出SocketTimeout异常，同时断开连接，客户端会出现`Connection closed by foreign host`类似的错误信息。抛出异常堆栈信息，如下所示：
```java
java.net.SocketTimeoutException: Read timed out
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.net.SocketInputStream.read(SocketInputStream.java:127)
```

> 在很多时候，我们都会给服务器的socket读设置一个超时时间，来避免由于客户端bug而导致一直占用服务器socket连接资源不释放问题，比如我们使用ssh客户端登录线上机器的时候，时间长了就会被断开连接。那如果客户端真的是期望和服务器一直保持连接该怎么办？定时发送心跳来维持连接，所以，我们一般通过修改ssh配置文件定时发送心跳包来保证不被服务器断开连接。

#### Executor使用说明

在服务器开发使用多线程模型的时候，服务器在收到客户端请求后，建立好socket连接，然后从线程池中获取一个线程来操作网络流IO。但是，在正常环境下，肯定不能无限的创建线程来支持客户端的请求，这个时候，就需要设置一个最大的线程数来保护服务器。

此外，当客户端请求处理逻辑发现线程池的初始线程数的线程都被消耗完，则依照线程池逻辑会临时放到Queue中等待线程释放，或者队列满进而在最大线程数之内创建新的线程来支持客户端请求处理。因此，就可能会有IO操作一直等待处理而不反馈给客户端（这种case对于需要及时响应的客户端请求，是很不推荐的，所以在HSF，thrift这些RPC框架中，都采用SynchronousQueue来避免这个问题）。

**SynchronousQueue**是一个阻塞队列，但是这个队列比较特殊，其每一次插入操作都必须等待另外一个线程去获取移除操作，才能结束阻塞返回。SynchronousQueue内部使用`TransferQueue`/`TransferStack`来实现，内部逻辑比较复杂，感兴趣的可以去看JDK源码。

> SynchronousQueue 其内部只会有一个元素（姑且这么认为，实际上其是不存储元素的）。当队列为空的时候，生产者将队列head指向新创建的node节点，在node上面waiter属性则设置为该生成者占用的线程，并且node的数据类型为DATA，阻塞线程等待消费；然后，需要消费数据的线程检查到head不为null，并且node类型为DATA，则消费该node，并且unpark掉处于阻塞的生成者线程。
> 当SynchronousQueue的head为null时，消费者需要获取node进行消费，则创建一个node节点，设置waiter为消费者线程，类型为REQUEST，将head指向node节点；然后生产者有生产数据的需求时，则unpark掉处于阻塞的消费者线程。

## 参考文献

1. TCP 的那些事儿([http://coolshell.cn/articles/11564.html](http://coolshell.cn/articles/11564.html))
2.  TCP 为什么是三次握手，为什么不是两次或四次？([https://www.zhihu.com/question/24853633](https://www.zhihu.com/question/24853633))
3.  再叙TIME_WAIT([https://huoding.com/2013/12/31/316](https://huoding.com/2013/12/31/316))
4.  从TCP三次握手说起--浅析TCP协议中的疑难杂症([http://wetest.qq.com/lab/view/81.html](http://wetest.qq.com/lab/view/81.html))
5. QQ 为什么以 UDP 协议为主，以 TCP 协议为辅？ ([https://www.zhihu.com/question/20292749](https://www.zhihu.com/question/20292749))
6. 微信对网络影响的技术试验及分析([http://www.52im.net/forum.php?mod=viewthread&tid=195&ctid=10](http://www.52im.net/forum.php?mod=viewthread&tid=195&ctid=10))
3.   thrift server整理后的项目源码([https://github.com/ketao1989/JavaThrift](https://github.com/ketao1989/JavaThrift))
4.  关于Java网络API中设置Socket连接超时分析 ([http://cuisuqiang.iteye.com/blog/1725348](http://cuisuqiang.iteye.com/blog/1725348))
5.  Java 半关闭导致Connect reset异常分析([http://xiaoz5919.iteye.com/blog/1685138](http://xiaoz5919.iteye.com/blog/1685138))


