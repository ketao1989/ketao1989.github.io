---
layout: post
title: "Netty系列--NIO学习"
date: 2018-05-05 21:36:00 +0800
comments: true
categories: 
  - java
  - netty
---

* content
{:toc}

> Netty官方文档：Netty是一款异步的事件驱动的网络应用程序框架，支持快速地开发可维护的高性能的面向协议的服务器和客户端。

其实，在官网 [https://netty.io/](https://netty.io/) 中特别指出，netty是基于NIO的网络通信框架。因此，在学习Netty之前，先熟悉java NIO组件。一般，我们说NIO，指的是非阻塞IO (Non-blocking IO)。

# 1. I/O模型

在计算机开发领域中，一个服务的性能受制于两方面：CPU处理能力和I/O处理能力。也就是说，我们希望我们的服务，可以消耗更少的CPU计算单元，消耗更少的I/O处理等待时长。

因此，由于I/O性能对整个系统的影响如此重要，在很多系统里，尽量来避免IO问题。例如，Redis通过使用内存，异步刷磁盘来提高IO性能，保证系统的处理能力。

当然，在很多时候，我们是避免不了I/O性能的制约的。比如，网络通信，就是一个典型的I/O问题。在优化I/O问题之前，先来聊聊I/O模型。

在《UNIX网络编程卷1》中，将I/O模型分为5种：
- 阻塞式I/O；
- 非租塞式I/O；
- I/O复用(select，poll，epoll等)；
- 信号驱动式I/O(SIGIO)；
- 异步I/O(POSIX的aio_系列函数)

在区分阻塞和非租塞，异步和同步之前，先介绍下，一个输入操作主要包括两个阶段：
1. 等待数据准备好；
2. 从内核向进程复制数据。

对于一个网络I/O请求来说，第一步就是等待网络通信的数据传输到服务器上，系统内核会将所有传输过来的数据保存在对应位置的缓冲区中；第二步就是将网络的数据从内核区复制到用户进程的缓冲区中，然后应用就可以处理。

<!-- more -->

## 1.1 概念解释

### 1.1.1 同步和异步
所谓同步，就是系统中最小CPU调度单位在执行任务的过程中，前一个任务没有执行完成返回之前，是一直处于阻塞的状态，不行进行后续的动作。

与此相反，所以异步，就是前一个任务执行还未完成的时候，其也可以执行后续的动作，而不需一直处理等待过程中。

在I/O模型中：  
我们通常认为，同步就是在整个输入过程中，线程都是处理等待的状态，直到用户态的进程缓冲区中有了数据可以触发后续执行；  
而异步，执行发出数据请求之后，直接返回执行后续的动作，直到内核数据接收完成拷贝到进程缓冲区之后，触发预先设置的回调动作。因此，我们说nodeJs是异步的，就是因为nodejs都是通过各种回调函数来处理I/O请求的。

### 1.1.2 阻塞和非阻塞
所谓阻塞，就是系统中细小CPU调用单位在执行任务过程中，需要一些条件准备好之后，等待条件确认满足，才能返回继续执行。

与此相反，所谓非阻塞，就是执行任务的时候，如果需要一些条件准备好时，系统不会一直处在等待的状态，而是返回一个标识，然后，可能会循环check条件是否满足。

在I/O模型中：  
我们通常认为，阻塞就是在输入过程中，如果数据没有准备好，线程会一直处理等待准备数据的动作完成；  
而非阻塞，则不会盲目去等待所有数据完成，而是内核会立刻返回一个错误标识，系统根据这个标识来循环的去check数据是否准备好。等数据准备好了之后，再由进程把数据复制到进程缓冲区中。

### 1.1.3 总结
从上面的介绍，可以看出，阻塞和同步，非阻塞和异步，并没有什么必然的等号关系。完全是两个不同维度的定义。
比如说，非阻塞I/O也会是同步I/O；虽然说异步I/O必然是非阻塞的。

## 1.2 阻塞和多路复用I/O

虽然有5中I/O模型，但是实际上，我们用的最多的就是2种。阻塞I/O和多路复用I/O(Java中的非阻塞NIO)。

### 1.2.1 阻塞式I/O

上文介绍了，I/O分成两个阶段，而这两个阶段都是处理等待的时候，则是阻塞式I/O。如下图：

![阻塞式I/O模型](/images/2018/block_io.png)

在上图中，调用recvfrom，系统调用直到数据包到达并且数据被复制到应用的缓冲区时才返回，而在此之间，进程一直处理阻塞等待的状态。

### 1.2.2 多路复用I/O

如上面非租塞I/O所述，系统在知道数据未准备好的时候，会根据标识，不停的轮询内核拿到结果，其实这对于系统性能而言，并不是十分友好。

于是，多路复用I/O将其拆成两个方法调用，首先以“准非阻塞方式”等待数据准备好，然后再来copy数据。如下图所示：

![多路复用I/O模型](/images/2018/multi_io.png)

实际上，多路复用就是用一个统一的内核线程来管理所有需要获取数据的请求，当统一线程发现有数据准备好后，会通知相应的用户线程来copy数据。所以，上面说“准非阻塞方式”，当多个线程中的至少一个数据准备好后，统一线程就会返回。

### 1.2.3 总结

从如上的图可以发现，多路复用I/O和阻塞I/O相比，并没有什么优势，毕竟阻塞I/O还只需要一个接口。多路复用，顾名思义，就是针对多路请求的，如果请求量很少很少的时候，直接使用多线程+阻塞I/O才是一个更好的选择。

此外，当请求量很大，并且每个连接都很活跃的时候，多路复用并不推荐，因为这个时候，每个连接都会非常活跃，而使用一个统一的内核线程管理数据准备情况，会导致单点严重。

但是，从现实的网络应用模型来说，绝大部分的网络连接，都是处于非活跃状态，因此，多路复用模型才在网络应用上，效果会如此好，应用如此广泛。

## 1.3 多路复用方法

在linux系统中，主要有select，poll和epoll三个方法，而在linux内核2.6推出epoll之后，基本上多路复用指的就是epoll。

### 1.3.1 wakeup callback机制

在linux 2.6内核中，推出了所谓wakeup callback机制。linux内核通过睡眠队列来组织所有等待某个事件的任务，而wakeup方法则可以异步唤醒整个睡眠队列上的任务，而队列上的任务都会关联上一个callback回调方法。当我们的wakeup方法执行唤醒动作的时候，会遍历整个睡眠队列的节点，然后调用对应的callback方法。

#### 1.3.1.1 睡眠等待

``` sh
define sleep_list;  
define wait_entry;  
wait_entry.task= current_task;  
wait_entry.callback = func1;  
if (something_not_ready); then  
    // 如果条件没有准备好，则陷入内核阻塞路径  
    add_entry_to_list(wait_entry, sleep_list);  
go on:    
    // 进行循环等待中，直到关心的事件准备好
    schedule();  
    if (something_not_ready); then  
        goto go_on;  
    endif  
    // 将当前的任务从睡眠队列中移除
    del_entry_from_list(wait_entry, sleep_list);  
endif  
```

#### 1.3.1.2 唤醒动作

``` sh
something_ready;  
// 当某个事情准备好之后，唤醒整个睡眠队列
for_each(sleep_list) as wait_entry; do  
    // 对每个睡眠队列中的节点执行回调方法
    wait_entry.callback(...);  
    if(wait_entry.exclusion); then  
        break;  
    endif  
done  

```

一般而言，在callback中执行如下逻辑：

``` sh
common_callback_func(...)  
{  
    // 自定义逻辑
    do_something_private;  
    wakeup_common;  
}  
```
其中，do_something_private是wait_entry自己的自定义逻辑，而wakeup_common则是公共逻辑，旨在将该wait_entry的task加入到CPU的就绪task队列，然后让CPU去调度它。

### 1.3.2 select && poll

select函数允许进程指示内核等待多个事件中的任何一个发生，并只在有一个或者多个事件发生或者经历一段指定的时间后才会唤醒它，结束阻塞等待。

简单的看下，linux select函数：

``` c
/**
 * @params nfds         集合中所有文件描述符的范围，即所有文件描述符的最大值加1
 * @params readfds      需要监视的读变化的文件描述符集合
 * @params writefds     需要监视的写变化的文件描述符集合
 * @params exceptfds    需要监视的出现的错误异常的文件描述符集合
 * @params timeout      select方法阻塞等待的最长时间，超过该超时时间则结束阻塞，返回0
 * @return <0：表示方法执行出错；=0：表示等待超时，没有可读写或异常事件；>0：表示某fd可读写或出错
 */
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

```

上文已经介绍了多路复用IO的使用场景：网络连接需要处理大量网络连接，并且只会有部分是处于活跃状态，当应用调用select方法的时候，select函数会将需要监控的各个fd_set集合copy到系统的内核空间，然后自己会先遍历一次需要监控的所有fd_set集合，确认对应等待的数据是否满足条件。如果，不存在任何一个fd条件满足，则会进入schedule_timeout的内部循环。

在schedule_timeout中，在超时时间内，select会阻塞，将所有的set中的socket放入了`wakeup callback机制`中介绍的睡眠队列中，等待数据的状态的变化。当其中一个socket事件发现数据发生变化的时候，则会执行对应统一的callback方法，如下：

``` sh
for_each_N_socket as sk; do  
    event.evt = sk.poll(...);  
    event.sk = sk;  
    put_event_to_user; 
done; 

```
也就是说，由于wakeup中的callback并没有给出是哪个socket准备好了，所以select会遍历所有注册在其上的socket，执行poll函数，判断对应的socket数据是否准备好。

虽然select限制了最大只能有1024个socket可以服用，但是如果每次唤醒都只因为1个socket数据准备好了，而需要遍历1024个socket，对性能的影响也是非常大的。更不用说，我们还可以通过修改`FD_SETSIZE`配置，来增大这个大小限制。

``` c
/**
 * @params fds     所有需要监视变化的文件描述符集合
 * @params nfds    需要监视的文件描述符范围，即所有文件描述符的最大值加1
 * @params timeout 超时时间，超时返回0
 * @return <0：表示方法执行出错；=0：表示等待超时，没有可读写或异常事件；>0：表示某fd可读写或出错
 */
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

poll底层和select是基本上一致的，其主要是为了解决1024数量的限制，但是性能上的问题，并没有解决，甚至由于限制的大小增大，反而性能更差。

### 1.3.3 epoll

在`select&&poll`中介绍了，其对应的callback只是通知对应的fd_set中某个socket数据准备好了，并没有告知具体情况，这个时候，select方法只能遍历所有的fd_set来找到真正准备好的socket，来处理后续流程。

然而，我们显然可以做的更进一步，每个socket都有自己的callback方法，来完成自己对应的回调逻辑。

epoll 新增了一个链表ready_list，这个链表内的所有socket都是数据已经准备好了，应用进程可以处理了。这个时候，当等待在睡眠队列上的节点呗唤醒之后，将自己加入到epoll准备好的ready_lis中，然后epoll_wait返回的时候，只需要遍历ready_list链表即可。

``` c
epoll_wakecallback(...)  
{  
    // 将自己这个socket加入到ready_list中
    add_this_socket_to_ready_list;  
    // 唤醒epoll_wait等待函数
    wakeup_single_epoll_waitlist;  
}  

```
如上所述，因为epoll和select不同之处，在于每个socket fd 都有一个callback，因此在使用epoll的时候，也需要添加对应的socket callback。所以，epoll在使用上，会相对比较复杂。

``` c
/**
 * 创建一个epoll句柄，因此其返回时一个fd文件描述符
 *
 * @params size 监听的socket数目，显然和select方法的nfds是不同的。
 */
int epoll_create(int size);
```

``` c
/**
 * epoll事件注册函数。这个方法其实是核心方法，其告诉内核需要监听什么事件，并且在事件触发的时候，执行什么回调
 * 
 * @params epfd create返回的epoll句柄fd
 * @params op 需要执行的动作，比如EPOLL_CTL_ADD：注册新的fd到epoll句柄中
 * @params fd 要监听的fd
 * @params event 监听的事件，内核根据这个事件来决定是否触发callback，例如EPOLLIN读事件
 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

``` c
/**
 * 类似select方法一样，阻塞等待监听的所有fd中某个事件触发。
 *
 * @params epfd create返回的epoll句柄fd
 * @params event 内核中得到的fd集合，表示哪些fd已经准备好了
 * @params maxevents events的大小。
 * @params timeout 超时时间，如果阻塞直到超时，则返回0
 */
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

在epoll中还有一个很重要的概念：ET(Edge Triggered) 和 LT(Level Triggered)。

所谓ET：只有状态发生变化的时候，才会通知，比如数据缓冲去从无到有的时候(不可读-可读)，如果缓冲区里面有数据，便不会一直通知；

所谓LT：只要缓冲区里面有数据，而这些数据没有被应用处理的时候，就会一直通知。

一般而言，为了代码健壮，绝大部分框架都对外提供基于LT触发的epoll，从而避免代码bug导致事件丢失。java NIO Selector提供的就是基于LT触发的，而如果想使用基于ET的epoll，则可以使用Netty提供的Epoll类。

# 2. java NIO 介绍

上文，只是简单的介绍了linux下I/O模型，以及开发服务端尝尝接触到的多路复用I/O底层接口。

在java的世界里，都进行了很好的封装。如下，我们将继续学习nio相关的类功能。

java nio 三剑客：
+ Buffer(缓冲区)
+ Channel(通道)
+ Selector(多路复用选择器)

我们知道，传统的I/O接口是基于流（字节流/字符流）来进行操作的，内部所有的接口，都是在输入输出stream上操作。因为基于流，所以不需要额外的空间来存储数据，但是同样因为基于流，所以对于stream，是不能前后移动来获取流中的数据，除非先将所有数据保存下来。

而，java nio类接口是基于缓冲区buffer和通道channel来操作的。在java nio中，数据从channel读出到buffer中，然后应用程序操作buffer中的字节；或者应用程序将缓冲区buffer中的数据写入到channel中。

如上面讲的多路复用那样，应用线程在需要读写数据的时候，只需要把请求包装之后交给selector线程来阻塞等待，而业务自己的线程可以去做一些别的事情。等到事件触发之后，再由业务线程来处理数据即可。

## 2.1 Buffer

### 2.1.1 Buffer类介绍

Buffer是一个用于特定类型数据的容器，在java nio中，有多个缓冲区实现，如：
`ByteBuffer, CharBuffer, DoubleBuffer, FloatBuffer, IntBuffer, LongBuffer, ShortBuffer`。在网络应用开发中，用的最多的是ByteBuffer类。

Buffer除了数据内容之后，还会有一些基本的属性，例如：

+ 容量capacity：表示该缓冲区包含元素的数量；
+ 限制limit：第一个不应该读取或者写入的元素的索引；
+ 位置position：下一个要读取或者写入的元素的索引。

以上，三个属性特别重要，基于buffer上的所有方法都是操作以上三个属性来完成各种功能的。

> 需要特别说明下，buffer是非线程安全的，如果多个线程操作buffer，则需要使用同步加锁来进行约束。

对于Buffer读写数据，主要是5个步骤：
1. 对于Buffer容器，首先需要申请分配内存空间来保存数据；
2. 写入数据到Buffer容器中，Buffer容器会记录下写入的数据，以及更新position的值；
3. 调用flip()方法切换读写模式,这个时候Buffer类的三个属性会进行更新，例如position现在是读位置索引等；
4. 从Buffer容器中读取数据，Buffer容器讲数据操作了之后，会同步更新position值；
5. 调用clear()方法或者compact()方法来请客缓冲区。clear()方法会清空整个缓冲区。compact()方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。

### 2.1.2 ByteBuffer类使用

由于ByteBuffer是使用最多的容器类型，这里直接用这种类型来介绍buffer的使用。

``` java
    public static void main(String[] args) throws Exception {

        RandomAccessFile aFile = new RandomAccessFile("test.txt", "rw");
        FileChannel inChannel = aFile.getChannel();

        //create buffer with capacity of 48 bytes
        ByteBuffer buf = ByteBuffer.allocate(48);

        //read into buffer.
        int bytesRead = inChannel.read(buf);
        while (bytesRead != -1) {

            System.out.println("read byte size: " + bytesRead);

            //make buffer ready for read
            System.out.println("--------------------读写切换开始--------------------- ");
            System.out.println("before flip position: " + buf.position() + ", limit: " + buf.limit());
            buf.flip();
            System.out.println("after flip position: " + buf.position() + ", limit: " + buf.limit());
            System.out.println("--------------------读写切换结束--------------------- ");

            int readSize = 0;
            while (buf.hasRemaining()) {
                // read 1 byte at a time
                readSize++;
                // 如果这里不操作get方法，则这个while会死循环，因为buffer的position一直没变化
                buf.get();
            }
            System.out.println("this time read buffer data size: " + readSize);

            //make buffer ready for writing
            System.out.println("++++++++++++++++++++++清缓冲区开始+++++++++++++++++++++++ ");
            System.out.println("before clear position: " + buf.position() + ", limit: " + buf.limit());
            buf.clear();
            System.out.println("after clear position: " + buf.position() + ", limit: " + buf.limit());
            System.out.println("++++++++++++++++++++++清缓冲区结束+++++++++++++++++++++++ ");


            bytesRead = inChannel.read(buf);
        }
        aFile.close();
    }

``` 
代码其实并不重要，我们通过打印的数据，可以发现buffer对于三个属性的更新操作。如下：

``` java
read byte size: 48
--------------------读写切换开始--------------------- 
before flip position: 48, limit: 48
after flip position: 0, limit: 48
--------------------读写切换结束--------------------- 
this time read buffer data size: 48
++++++++++++++++++++++清缓冲区开始+++++++++++++++++++++++ 
before clear position: 48, limit: 48
after clear position: 0, limit: 48
++++++++++++++++++++++清缓冲区结束+++++++++++++++++++++++ 
read byte size: 29
--------------------读写切换开始--------------------- 
before flip position: 29, limit: 48
after flip position: 0, limit: 29
--------------------读写切换结束--------------------- 
this time read buffer data size: 29
++++++++++++++++++++++清缓冲区开始+++++++++++++++++++++++ 
before clear position: 29, limit: 29
after clear position: 0, limit: 48
++++++++++++++++++++++清缓冲区结束+++++++++++++++++++++++ 

```
通过返回结果，我们可以清晰的知道读写flip之后，三个属性的变化，以及clear之后，三个属性的变化。如下图，会有个形象的说明：

![Buffer数据模型](/images/2018/buffer.png)

### nio buffer API不足之处

如上用例的用例，我们会发现buffer API使用过程中的一些不足：

1.  `ByteBuffer buf = ByteBuffer.allocate(48);` 在申请的时候，需要明确指出buffer的大小，分配完成之后，则大小固定，如果后续的内容过大，则会抛异常，导致`BufferOverflowException`.

2. 另外，还有一个很不好的体验，就是我们做读写切换的时候，需要自己手动flip来切换position位置，一不小心就可以能到读写出错。


## 2.2 Channel

在java nio体系中，channel就类似一个stream流，但是其相对stream来说，在使用上有更多的进步：

1. 如上所述，Channel是基于buffer缓冲区的，所以其支持在同个channel中执行读和写操作, 然而同一个Stream在构造的时候，就需要明确指出其针对的是读还是写.

2. buffer的引入，让Channel可以提供异步的读写，但是Stream基于流，只支持同步阻塞的读写操作；

在java nio中，目前主要分为：

* FileChanel：用来读取、写入、映射和操作文件的通道。通过FileChannel从一个文件中读取数据, 也可以将数据写入到文件中。不过FileChannel是同步的，如果想要异步的文件Channel，则需要使用`AsynchronousFileChannel`。所谓AsynchronousFileChannel，其实就是在读写的时候，其直接返回一个future，后面只需要在需要的时候，执行future.get来感知读写进度即可。

* SocketChannel：用来操作连接套接字TCP流的通道，主要是针对客户端连接。

* ServerSocketChannel：用来操作连接套接字TCP流的通道，主要是针对服务端连接。

以上，是主要的channel，还会有些其他的，暂不说明。

### 2.2.1 SocketChannel && ServerSocketChannel介绍

`SocketChannel` 和 `ServerSocketChannel`对应Stream模式下的Socket和ServerSocket，其最大的不同就是支持非阻塞模式。

所以，我们如下使用ServerSocketChannel：

``` java
public class NioServer {
    public static void main(String[] args) throws IOException {

        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        serverSocketChannel.socket().bind(new InetSocketAddress(8888));

        // 设置成阻塞模式
        //serverSocketChannel.configureBlocking(false);

        while (true) {

            SocketChannel socketChannel = serverSocketChannel.accept();
            
            // 阻塞模式
            ByteBuffer buf = ByteBuffer.allocate(48);
            int bytesRead = 0;

            while ((bytesRead = socketChannel.read(buf)) != -1) {
                System.out.println("read socket data size: " + bytesRead);
                // 空操作，只是为了把数据读出来
                buf.get();
            }
            
        }
    }
}

```

> 特别说明，上面是一个使用serverSocketChannel的简单示例，读内容的逻辑，只试用于阻塞，开启非阻塞模式，则会出现：
> ``` java
> Exception in thread "main" java.lang.NullPointerException
>	at io.github.ketao1989.nio.NioServer.main(NioServer.java:32)
> ```
> 关于非阻塞模式，在Selector中说明。此外，实际上在ServerSocketChannelImpl中的实现非阻塞的时候，是会关联一个Selector的，而这个Selector在不同的系统环境，是不同的。
>
> | 系统 | Selector | 备注 |
> | - | :-: | :- | 
> | Unix(MacOs) | KQueueSelectorImpl| 类Unix系统的多路复用器 |
> |Linux| PollSelectorImpl|内部应该是使用的epoll多路复用|
> |windows|WindowsSelectorImpl|内部应该是有的是iocp| 
>
> 具体Selector的介绍，见下一节。

从上面的代码，可以明确看出，channel数据操作是基于buffer来的，其所有的接口都是基于Buffer对外暴露的。

而在，Buffer的介绍中，说过Buffer是缓冲区，可以读和写一起操作，不需要在构造一个对象。因此，基于其数据结构之上的channel也是可以读写一起操作的。

## 2.3 Selector

Selector，多路复用选择器。其可以通过一个线程来同步检测多个通道channel是都有相关事件准备好了，从而减少线程hang在IO等事件上。

通常情况下，使用一个线程监听事件，而不是多线程来处理，主要原因是线程上下文切换的开销会比较大，并且每个资源暂用的资源也会有一些，因此使用单线程对于系统来说，可能当前是比较友好的。

> 但是，需要说明的，由于CPU的技术发展以及操作系统的不断优化，现在多线程的开销会越来越小。如果有多核，而程序不去使用，那显然是浪费CPU能力。因此，我们会看到，有一种线程模型，是多个selector来并发执行。例如thrift的TThreadedSelectorServer，这个模型，以后有机会再详说。

### 2.3.1 Selector 介绍

在java中，linux环境下，selector使用epoll来实现多路复用。

下面通过代码了解selector的使用：

```  java

public class NioServer {


    public static void main(String[] args) throws IOException {

        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        serverSocketChannel.socket().bind(new InetSocketAddress(8888));

        Selector selector = Selector.open();

        // 设置成阻塞模式
        serverSocketChannel.configureBlocking(false);

        // 注册监听事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {

            int select = selector.select();

            if (select == 0) {
                System.out.println("啦啦啦啦啦，谁把我弄起来了");
            }

            if (select > 0) {

                Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();

                    // 接收连接请求
                    if (selectionKey.isAcceptable()) {

                        SocketChannel socketChannel = serverSocketChannel.accept();
                        socketChannel.configureBlocking(false);

                        System.out.println("开始处理连接:" + socketChannel.getRemoteAddress().toString());

                        //每接收请求之后，感兴趣的树读事件，所以在同一个selector中注册读事件
                        socketChannel.register(selector, SelectionKey.OP_READ);
                    }

                    if (selectionKey.isReadable()) {

                        System.out.println("开始读取数据......");
                        // 简单处理，不起线程了
                        ByteBuffer buf = ByteBuffer.allocate(48);
                        int bytesRead = 0;
                        SocketChannel socketChannel = (SocketChannel) (selectionKey.channel());
                        while ((bytesRead = socketChannel.read(buf)) > 0) {
                            System.out.println("read socket data size: " + bytesRead);
                            // 空操作，只是为了把数据读出来
                            buf.get();
                        }

                        // 这里一定需要判断，不然关闭client之后，会出现死循环
                        // 一直处在 '开始读取数据......'
                        if (bytesRead == -1) {

                            System.out.println("开始关闭会话......");
                            selectionKey.cancel();
                            socketChannel.close();
                        }
                    }
                    iterator.remove();
                }
            }
        }
    }
}

```
> 以上是实现一个很简单的selector的非阻塞服务器，在开发过程中，会存在很多小问题，例如如果我们不对bytesRead为-1的情况进行处理，则会一直处在有读事件没有处理，然后就会出现死循环。此外，基于多线程处理读数据，又会有很多地方修改和注意点，需要非常谨慎和细致。
> 因此，开发一个nio的服务器是非常需要技术的，一旦一个点遗漏，就会导致死循环。java nio 接口使用的不友好，造就了mina，netty网络库的出现。


### 2.3.2 Selector wakeup方法


在selector中，也有一个wakeup方法，虽然和上面的wakeup并不是一个东西，但是其目的是一样的。

在Selector中，我们知道其是一个线程阻塞的来处理所有事件监听的，如果selector不设置超时的话，就会一直处于阻塞状态，直到有时间发生。

但是，如果这个时候，其他线程需要这个线程返回，停止阻塞，就需要调用wakeup来唤醒处在select()中的线程。

举个例子，比如我服务端使用selector来处理事件，现在，我需要停止服务,但是因为没事件发生，导致selector的select方法处于阻塞状态，这个时候就需要使用wakeup唤醒。例如在thrift的TNonblockingServer，在停止server的时候，就会调用wakeup，代码如下：

``` java

  /**
   * Stop serving and shut everything down.
   */
  public void stop() {
    stopped_ = true;
    if (selectAcceptThread_ != null) {
      selectAcceptThread_.wakeupSelector();
    }
  }

    /**
     * If the selector is blocked, wake it up.
     */
    public void wakeupSelector() {
      selector.wakeup();
    }  

```
> wakeup的实现，其实很有意思，就是mock一个事件发生，然后select()方法就可以从阻塞中醒过来，从而完成wakeup的动作。
> 具体，可以参考：[http://jm.taobao.org/2010/10/22/380/](http://jm.taobao.org/2010/10/22/380/)

# 3. Netty ByteBuf 介绍

由于Nio的ByteBuffer的一些不足，在netty中提供了一个相同功能的类：ByteBuf。但是ByteBuf很好的简化了nio bytebuffer的使用，并且将一些常用的方法包装出来对外提供。

在介绍各个bytebuf特性之前，先看看netty bytebuf提供了哪些常用的类

![netty bytebuf主要实现类图](/images/2018/bytebuf_main_class.png)

+ EmptyByteBuf：一个空的bytebuf，其capacity为0，读写index也为0.
+ WrappedByteBuf：主要是对ByteBuf类进行一些包装的类型，也就是其只是对例如CompositeByteBuf进行一些装饰，额外加一些日志或者记录等。
    + SimpleLeakAwareByteBuf：对于一些方法，比如slice，duplicate，touch等等对buf对象会有引用的时候，会进行record引用记录。
    + AdvancedLeakAwareByteBuf：看实现跟上面的Simple是类似的，只不过对更多的操作会记录详细的操作计数的日志。
    + UnreleasableByteBuf：顾名思义，就是不允许release buf空间。调用release直接返回false，不做任何处理。一般，我们在定义一个常量的bytebuf会使用这种类型，比如说，我们对每一个文件都会加以java结尾，则将java定义成一个UnreleasableByteBuf，这样内容不会修改和释放，每一次文件路径过来，加上java buf 返回。

+ DuplicatedByteBuf：在底层数据共享的时候，该bytebuf可以对每一个Duplicated出来的对象单独提供readIndex和writeIndex，彼此之间互不影响，但是缓冲区是共享的，这样子可以节省内存占用。
+ ReadOnlyByteBuf/ReadOnlyByteBufferBuf：和上一个bytebuf类似，只不会新创建出来的bytebuf引用，不能操作writeIndex，也就意味着在很多为了数据安全，只能够共享读数据的地方，非常适用。
+ CompositeByteBuf：一个虚拟的buffer，实际上他本身并不额外分配buffer空间，而是引用组合的各个buffer为一个list和迭代器。有的时候，我们可能从好几个地方获取到buf数据，然后聚合起来返回。使用CompositeByteBuf可以避免申请新的空间，然后复制数据的过程。
> 需要说明下，SlicedByteBuf这种和CompositeByteBuf刚好相反，只截取bytebuf部分缓冲区数据的类型。现在这种类型之间使用bytebuf.slice方法构造。

+ PooledByteBuf：基于内存池的ByteBuf子类型，主要是重用ByteBuf对象，避免内存申请释放导致的性能问题。由于内存获取方式的不同，还有如下更优化的子类型bytebuf存在。
    + PooledDirectByteBuf：基于directByteBuffer来构造，直接使用堆外内存进行分配空间。堆外内存的时候，可以避免内存到JVM内存的一次拷贝，在和内核做数据交换的时候，对性能的提升很大。
    + PooledHeapByteBuf：使用JVM堆进行内存分配的byteBuf子类型。堆内分配，如果一些数据只是在java进程内部使用，则使用堆内存分配安全性 和速度上都会比较好，毕竟内存的回收有JVM来处理。
    + PooledUnsafeDirectByteBuf：和PooledDirectByteBuf类似，只不过其不是使用directByteBuffer来处理，而是自己通过PlatformDependent来根据不同系统平台使用不同的分配内存方式。
    如下代码：
    ``` java

    private static void getBytes(long inAddr, byte[] in, int inOffset, int inLen, OutputStream out, int outLen)
            throws IOException {
        do {
            int len = Math.min(inLen, outLen);
            PlatformDependent.copyMemory(inAddr, in, inOffset, len);
            out.write(in, inOffset, len);
            outLen -= len;
            inAddr += len;
        } while (outLen > 0);
    }
    ```
+ `UnpooledByteBuf`：在netty不存在UnpooledByteBuf这种类型，这里只是相对`PooledByteBuf`而言。
    + UnpooledHeapByteBuf：在堆内及时分配内存的bytebuf，内部代码直接基于ByteBuffer来实现的，不过增加了引用计数的内存回收。
    + UnpooledDirectByteBuf：在对外分配内存的bytebuf，每次需要就直接申请分配内存空间，在大量操作分配和回收内存额时候，会导致性能下降。
    + UnpooledUnsafeDirectByteBuf：基于平台优化的堆外内存分配的UnpooledDirectByteBuf。

因此，如上，我们可以大概了解，bytebuf主要基于两个维度：1.堆内&堆外；2.池化&非池化。所以在使用的时候，需要仔细考虑下。

## 3.1 Netty ByteBuf 接口

### 3.1.1 读写标记分离

在nio的ByteBuffer的实现中，是使用一个position来表示读位置和写位置，使得操作的时候，读写切换，需要在此之前把position进行更新，保证不会出现非预期结果。

在netty的实现中，ByteBuf申明了两个变量，分别表示读位置readerIndex，写位置writerIndex。读操作的时候，使用和更新readerIndexn；写操作的时候，使用和更新writerIndex。于是，在初始化之后，两者都为0。然后，写执行，writerIndex递增；读执行，readerIndex递增；约束条件就是readerIndex<= writerIndex.

如：

``` java

 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      |                   |     (CONTENT)    |                  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *
```
例如，我们读写一个字节时，则可以如下实现：

``` java

    @Override
    public byte readByte() {
        checkReadableBytes0(1);//check读空间
        int i = readerIndex;
        byte b = _getByte(i);
        readerIndex = i + 1;
        return b;
    }

    @Override
    public ByteBuf writeByte(int value) {
        ensureWritable0(1);//check写空间
        _setByte(writerIndex++, value);
        return this;
    }

```


如果，我们需要清理已读的空间，只需要将当前readIndex之前的空间clear即可，相应的index往前移，非常简单。

``` java

 * 
 *  BEFORE discardReadBytes()
 *
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *
 *
 *  AFTER discardReadBytes()
 *
 *      +------------------+--------------------------------------+
 *      |  readable bytes  |    writable bytes (got more space)   |
 *      +------------------+--------------------------------------+
 *      |                  |                                      |
 * readerIndex (0) <= writerIndex (decreased)        <=        capacity
 * 

```
代码简单实现如下：

``` java

    @Override
    public ByteBuf discardReadBytes() {
        ensureAccessible();
        if (readerIndex == 0) {
            return this;
        }

        if (readerIndex != writerIndex) {
            setBytes(0, this, readerIndex, writerIndex - readerIndex);
            writerIndex -= readerIndex;
            adjustMarkers(readerIndex);
            readerIndex = 0;
        } else {
            adjustMarkers(readerIndex);
            writerIndex = readerIndex = 0;
        }
        return this;
    }

```

### 3.1.2 动态扩容

java 的list相对于C，其提供的动态扩容。在使用list之前，开发同学不需要事先明确集合空间的大小，等到空间预期不够的时候，则执行动态扩容。

但是，在nio的ByteBuffer中，竟然不支持动态扩容，如果一不小心写多数据了，则会抛出BufferOverflowException异常，显然和java的风格不一致。因此，在netty中，其次需要解决的，就是支持动态扩容功能。

netty的实现原理，很简单。就是在写的时候，如果空间不足，则分配一个新的更大空间，将老数据复制过去。这涉及到两个点：
1. 什么时候扩容，扩容多少？
2. 怎么分配更大空间？

在netty中，当新写入的byte数组不够存放到byteBuf空间时，才会申请扩容。在分配器Allocator中，当整个内容大小(writeIndex+bytes.size)小于4M的时候，则从64开始，*2，直到可以存放整个内容。如果需要的空间大于4M，则按照4M的倍数来存放内容。代码如下：

``` java
    public int calculateNewCapacity(int minNewCapacity, int maxCapacity) {

        final int threshold = 1048576 * 4; // 4 MiB page

        if (minNewCapacity == threshold) {
            return threshold;
        }

        // If over threshold, do not double but just increase by threshold.
        if (minNewCapacity > threshold) {
            int newCapacity = minNewCapacity / threshold * threshold;
            // 如果离最大空间不足4M，则直接以最大空间来创建
            if (newCapacity > maxCapacity - threshold) {
                newCapacity = maxCapacity;
            } else {
                // 如果空间足够，则直接增加一个4M来申请新空间即可
                newCapacity += threshold;
            }
            return newCapacity;
        }

        // Not over threshold. Double up to 4 MiB, starting from 64.
        // 如果新空间少于4M，则以64开始，找到刚好大于新空间的2的幂次方数
        int newCapacity = 64;
        while (newCapacity < minNewCapacity) {
            newCapacity <<= 1;
        }

        return Math.min(newCapacity, maxCapacity);
    }
```

确认好了新空间的大小之后，就是开始分配空间内存了。由于ByteBuf有很多种实现，比如pool的，unpool的，heapbuffer的，directBuffer的。拿个最简单的实现示例UnpooledHeapByteBuf来看：

``` java

    @Override
    public ByteBuf capacity(int newCapacity) {
        checkNewCapacity(newCapacity);

        int oldCapacity = array.length;
        byte[] oldArray = array;
        if (newCapacity > oldCapacity) {
            // new 一个byte数组
            byte[] newArray = allocateArray(newCapacity);
            // 数据copy
            System.arraycopy(oldArray, 0, newArray, 0, oldArray.length);
            //替换底层数据结构byte array
            setArray(newArray);
            // 非pool的，这块等着gc回收好了
            freeArray(oldArray);
        } else if (newCapacity < oldCapacity) {
            //.......
        }
        return this;
    }

```

### 3.1.3 引用计数

引用计数是最简单的GC方式，其在循环引用的场景下，会存在内存泄漏的问题；但是由于其简单性，在某些确认不会存在彼此循环引用的情况下，是非常易于开发和理解的。

引用计数，算是ByteBuf提供的基本功能，在最基础的接口定义上，就把`ReferenceCounted`包含进去了，如：
```java
@SuppressWarnings("ClassMayBeInterface")
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf> {}
```
> 特别说明下，这个注解ClassMayBeInterface，表明其实ByteBuf是个接口，使用abstract class只是为了后期可能内部实现一些默认方法，或者其他一些工具性质方法，在jdk8之前，只能使用抽象类来解决。
> 此外，其实在netty3中，ByteBuf就是一个接口，将接口改为abstract class会有5%的性能提升。

和GC中的引用计数一样，当byteBuf对象被引用的时候，调用retain方法计数增一；调用release方法计数减一。当最终计数为1的时候，再执行release的时候，则会使用deallocate将buf对象释放。

``` java

    /**
     * 以下两个保证多线程安全可见的特性组合使用，可以学习下。AtomicIntegerFieldUpdater+refCnt来原子CAS更新计数.
     * 上面的结合主要是在不修改对外暴露方法的前提下，内部来保证线程安全，从netty提交记录也大概可以看出来。
     */
    private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater =
            AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");

    private volatile int refCnt;

    private ByteBuf retain0(final int increment) {
        int oldRef = refCntUpdater.getAndAdd(this, increment);
        // 以下判断负数，或者加完increment之后整数溢出（这种判断很有趣）
        if (oldRef <= 0 || oldRef + increment < oldRef) {
            // Ensure we don't resurrect (which means the refCnt was 0) and also that we encountered an overflow.
            refCntUpdater.getAndAdd(this, -increment);
            throw new IllegalReferenceCountException(oldRef, increment);
        }
        return this;
    }

    private boolean release0(int decrement) {
        int oldRef = refCntUpdater.getAndAdd(this, -decrement);
        if (oldRef == decrement) {
            deallocate();
            return true;
        } else if (oldRef < decrement || oldRef - decrement > oldRef) {
            // Ensure we don't over-release, and avoid underflow.
            refCntUpdater.getAndAdd(this, decrement);
            throw new IllegalReferenceCountException(oldRef, decrement);
        }
        return false;
    }
```

引用计数，在netty中，也主要根据对象使用，然后自动释放&回收空间。如上代码，具体释放的动作，是一个抽象方法，需要子类来实现。例如：

池化的PooledByteBuf的实现：
``` java
    /**
     * bytebuf上面的相关空间进行销毁，比如内存页的块chuck在堆外内存的时候，进行free，还有相关引用置为null
     */
    protected final void deallocate() {
        if (handle >= 0) {
            final long handle = this.handle;
            this.handle = -1;
            memory = null;
            tmpNioBuf = null;
            chunk.arena.free(chunk, handle, maxLength, cache);
            chunk = null;
            recycle();
        }
    }

    /**
     * 具体的实现回收，默认是将内存块push到stack栈中。
     */
    private void recycle() {
        recyclerHandle.recycle(this);
    }
```
而在非池化的UnpooledByteBuf则基本上不会做什么操作，如下：
``` java
    @Override
    protected void deallocate() {
        freeArray(array);//空方法
        array = EmptyArrays.EMPTY_BYTES;//对应buffer的底层数据结构设置为空byte[] EMPTY_BYTES = {}
    }

```

> 如上，对于需要进行内存管理的buf来说，retain & release来管理引用计数，然后再必要的时候，进行内存释放和回归，是很需要的。
> 

## 3.2 Netty ByteBuf高级特性

### 3.2.1 内存泄漏检测

上面有一个基于引用计数的内存回收。但是，由于retain和release的方法调用都是由开发者自己来主动触发的，这就导致可能哪里遗忘了release buf对象的操作，导致一直没办法进行主动内存释放的操作。

内存泄漏检测，主要针对PooledByteBuf，因为是自己管理buf池子，就需要判断什么时候buf可以释放，当前有没有存在未释放的buf。

当前，netty内存泄漏检测的级别主要有：
+ DISABLED – 完全禁用检查。不推荐。
+ SIMPLE – 检查1/128比例的缓冲区是否存在内存泄露。默认检测为该级别。
+ ADVANCED – 检查1/128比例的缓冲区，并提示发生内存泄露的位置。
+ PARANOID – 与ADVANCED等级一样，不同的是会检查所有的缓冲区。对于自动化测试很有用，你可以让构建测试失败 如果构建输出中包含'LEAK', 用JVM选项 -Dio.netty.leakDetectionLevel 来指定内存泄露检查等级

于是，开启检测之后，如果存在内存泄漏，则在日志里，会存在如下错误信息：
``` java
LEAK: ByteBuf.release() was not called before it's garbage-collected. " +
                "Enable advanced leak reporting to find out where the leak occurred. " +
                "To enable advanced leak reporting, " +
                "specify the JVM option '-Dio.netty.leakDetection.level=advanced' or call ResourceLeakDetector.setLevel() " +
                "See http://netty.io/wiki/reference-counted-objects.html for more information.
```

在PooledByteBufAllocator中，新建buf都会进行装饰，如：

``` java
    protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<byte[]> heapArena = cache.heapArena;

        final ByteBuf buf;
        if (heapArena != null) {
            buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }

```
当我们，在测试的时候，则可以调用open方法打开检测，然后执行track指令，如下代码：

``` java
   private DefaultResourceLeak track0(T obj) {
        Level level = ResourceLeakDetector.level;
        // .....
        // 为了测试，检测所有怀疑资源泄漏的地方
        if (level.ordinal() < Level.PARANOID.ordinal()) {
            // 由于对系统压力大，所以进行采样，大概1/128比例
            if ((PlatformDependent.threadLocalRandom().nextInt(samplingInterval)) == 0) {
                reportLeak();
                return new DefaultResourceLeak(obj, refQueue, allLeaks);
            }
            return null;
        }
        // 其他设置，主动触发就会出一份检测报告
        reportLeak();
        return new DefaultResourceLeak(obj, refQueue, allLeaks);
    }

```

此外，在内存泄漏检测的实现中，使用了：

``` java
/**
 * 资源泄漏的统计对象是一个弱引用，在内存紧张的时候，会进行GC回收
 */
private static final class DefaultResourceLeak<T>
        extends WeakReference<Object> implements ResourceLeakTracker<T>, ResourceLeak {
}

/**
 * 在refQueue中保存的是DefaultResourceLeak类型的弱引用，然后放到ReferenceQueue中，当GC的时候，会查看
 * ReferenceQueue中的元素，进行GC回收，而不管外部是否还有改元素对象的引用。
 */
private final ReferenceQueue<Object> refQueue = new ReferenceQueue<Object>();

```

### 3.2.2 零拷贝优化

一般，我们拷贝，主要存在两种情况：

+ 我们创建新buf的时候，当需要从某个已存在的buf对象的部分或全部类型来回写新buf时，就需要拷贝这部分内容。
+ 当我们需要把数据从不同空间转移的时候，例如内核区到用户区，就需要进行数据拷贝的操作。

针对以上两种情况，netty都存在一些针对使用场景的优化，来避免不必要的拷贝。这些操作，我们统一称之为零拷贝。

#### 3.2.2.1 零拷贝之ByteBuf创建

##### CompositeByteBuf

当我们从不同的解析方法里面返回了header bytebuf和body bytebuf，然后想组合成一个请求整体进行后续反序列化等操作的时候，就需要将不同的buf组合成新的bytebuf，于是，就有了CompositeByteBuf。

CompositeByteBuf并没有真的copy多个bytebuf的数据到最终的buf中，而只是维护各个bytebuf的对象引用。

``` java
public class CompositeByteBuf extends AbstractReferenceCountedByteBuf implements Iterable<ByteBuf> {
    
    private static final Iterator<ByteBuf> EMPTY_ITERATOR = Collections.<ByteBuf>emptyList().iterator();

    CompositeByteBuf(ByteBufAllocator alloc, boolean direct, int maxNumComponents, ByteBuf[] buffers, int offset, int len) {
        super(AbstractByteBufAllocator.DEFAULT_MAX_CAPACITY);

        this.alloc = alloc;
        this.direct = direct;
        this.maxNumComponents = maxNumComponents;
        components = newList(maxNumComponents);

        // 这里讲buffer的数据加入到components列表中
        addComponents0(false, 0, buffers, offset, len);
        //当前components size 超过指定maxSize，触发copy数据操作
        consolidateIfNeeded();
        setIndex(0, capacity());
    }

    private int addComponent0(boolean increaseWriterIndex, int cIndex, ByteBuf buffer) {

        boolean wasAdded = false;
        try {

            int readableBytes = buffer.readableBytes();

            Component c = new Component(buffer.order(ByteOrder.BIG_ENDIAN).slice());
            if (cIndex == components.size()) {
                wasAdded = components.add(c);
                // 维护各个offset:对应buf在list中基于字节的开始偏移量和结束偏移量
                if (cIndex == 0) {
                    c.endOffset = readableBytes;
                } else {
                    Component prev = components.get(cIndex - 1);
                    c.offset = prev.endOffset;
                    c.endOffset = c.offset + readableBytes;
                }
            } else {
                // 不同之处在于，更新offset将是一个On的操作了
                components.add(cIndex, c);
                wasAdded = true;
                if (readableBytes != 0) {
                    updateComponentOffsets(cIndex);
                }
            }
            if (increaseWriterIndex) {
                writerIndex(writerIndex() + buffer.readableBytes());
            }
            return cIndex;
        } finally {
            // 如果没有更新成功，则需要把对应的buffer引用-1
            if (!wasAdded) {
                buffer.release();
            }
        }
    }

}
```
> 从上面的代码，可以清晰的看到，正常情况下，完全没有copy每一个组合中的bytebuf数据，而是维护一个包含bytebuf的component list，每个bytebuf的偏移信息等维护在component中，便于后续对CompositeByteBuf对象的操作。

##### DuplicatedByteBuf

ByteBuf维护了writeIndex和readIndex，当多个地方同时操作ByteBuf的时候，就会导致index错乱，这个时候，我们就需要复制一份数据，构造一个全新的ByteBuf对象。但是，另外一个方案，就是我只重新构造index，对于底层的buf，还是沿用原始的bytebuf。这样子，我们就不需要copy bytebuf中的所有数据到新的对象中。

#### 3.2.2.2 零拷贝之数据迁移

在不同系统空间之间会存在数据的来回拷贝，比如网络数据过来时，会从内核区的内存区间，拷贝到用户区的内存区间。如果加上JVM的堆内存，则还需要从用户区的Native内存拷贝到JVM堆内内存中，因此，正常需要好多次内存拷贝操作。

因此，为了快速优化拷贝性能，则需要考虑：
1. socket数据 --> 内核区空间
2. 内核区空间 --> Native内存空间
3. Native内存 --> 堆内内存空间

一般情况下，用户态的内存空间和系统内核的内存空间之间是隔离的，不允许批次操作。



## 4. Netty ByteBuf 内存管理

### 4.1 堆内堆外内存



### 4.2 池化非池化内存


## 参考文献

1. [大话 Select、Poll、Epoll](https://cloud.tencent.com/developer/article/1005481)
2. [Java NIO 的 wakeup 剖析](http://goldendoc.iteye.com/blog/1152079)



