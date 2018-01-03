---
layout: post
categories: 
  - Java 
  - Thread 
title:  Java 多线程线程池分析
date: 2014-05-03 22:21:35 +0800
comments: true
---

## <a id="Intro">前言</a>

关于Java多线程的知识，看了很多博客书籍，对理论还是比较了解的。但是，最近写一个很简单的使用线程池对列表中任务进行处理，然后返回结果列表的功能，发现理论和实际操作还是有相当大的差距。

首先贴出一个很简单的代码demo：

``` java
/**
 * @author: ketao Date: 14-5-3 Time: 下午4:51
 * @version: \$Id$
 */
public class ThreadTest {

    private static final ExecutorService executors = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {

        List<String> list = Lists.newArrayList("thread-1", "thread-2", "thread-3", "thread-4");
        final List<String> results = Collections.synchronizedList(new ArrayList<String>());
        for (final String str : list) {
            executors.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    results.add(str+"test");
                    System.out.println(Thread.currentThread().getName());

                }
            });
        }

        System.out.println(JSON.toJSONString(results));
        executors.shutdown();
    }
}
```

执行结果如下，显然`results`的值并**不是我们想要的结果** ：

	[]
	pool-1-thread-1
	pool-1-thread-2
	pool-1-thread-2
	pool-1-thread-1

### 1.1 线程定义

来自Java 并发大家 Doug Lea 关于线程的描述（[中文版](http://ifeve.com/java-concurrency-constructs/)）：

>> 线程：其是一个独立执行的调用序列，同一个进程的线程在同一时刻共享一些系统资源（比如文件句柄等）也能访问同一个进程所创建的对象资源（内存资源）。

由于一般的系统，最小的基本调度单位是线程，因此如果一个程序中只有一个线程的话，当该线程因为远程调用或者数据库访问，或者其他大量数学计算导致IO/CPU阻塞时，就会导致整个处理性能大幅度的降低。即使没有这些阻塞，对于当前多核处理系统来讲，单线程也会导致资源的浪费。因此，多线程可以帮助我们很好地提高系统的处理能力和吞吐能力。

## <a id="Thread">Java线程API</a>

在Java中可以通过`java.lang.Thread`创建线程。一般，应用中包括两种类型的线程：用户线程和守护线程。当应用启动时，会创建main线程，然后main线程可以创建多个用户线程和守护线程。当所有的用户线程都终止的时候，则JVM会终止程序。
**相对于用户线程而言，守护线程是为用户线程服务的，当所有的用户线程都退出的时候，守护线程就会全部退出，而不管守护线程当前的执行任务是否完成。**

### 2.1 创建Thread

在java中，创建一个线程类对象很简单，有两种方式：其一，只需要继承`Thread`类，并且在子类中实现`run()`方法;其二，实现一个`Runnable`接口来创建线程。简单地demo如下：

``` java
/**
 * @author: ketao Date: 14-5-3 Time: 下午4:51
 * @version: \$Id$
 */
public class ThreadTest {

    public static void main(String[] args) {

        System.out.println(Thread.currentThread().getName()); // main

        Thread thread = new Thread(){
            public void run(){
                System.out.println("创建一个java线程");
                System.out.println(Thread.currentThread().getName()); // Thread-0
            }
        };

        thread.start();

        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("创建一个java线程");
                System.out.println(Thread.currentThread().getName()); // Thread==Runable=2
            }
        },"Thread==Runable=2");
        thread1.start();
    }
}
```

<!-- more -->

对于上面的两种创建线程的方法，推荐使用`Runnable`来实现，因为我们知道在java 线程池`ExecutorService`可以管理和使用`Runnable`接口的线程。
当请求超过线程池设置的大小后，新的请求会排队等待执行，直到所有的线程池空闲为止，如果通过`Thread 子类`来实现线程池，则会比较复杂。

>> Tip: 在demo中使用`thread.run()`也可以得到相同的输出结果，但是，**run() 的输出是由当前线程执行的，而不是新创建的线程**。

### 2.2 创建守护线程

守护线程，你可能没有注意过，但是在运行java服务的时候必然会遇到，因为一个典型的守护线程就是java垃圾回收线程。因此，当我们的java应用的所有用户线程都完成退出后，就不会再由内存垃圾产生，进而垃圾回收线程就不需要GC操作，对于只剩下守护线程时，JVM的操作就是退出，结束整个java应用环境。

参考网络上得一篇博文[Java中的Daemon线程--守护线程](#http://blog.csdn.net/lcore/article/details/12280027)，给出一个deamon示例：

``` java
/**
 * @author: ketao Date: 14-5-3 Time: 下午4:51
 * @version: \$Id$
 */
public class ThreadTest {

    public static void main(String[] args) {

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("创建一个守护线程Deamon");
                    System.out.println(Thread.currentThread().getName());
                }
            }
        }, "deamon-thread-1");

        thread.setDaemon(true);
        thread.start();

        System.out.println("守护线程：  " + thread.isDaemon());

        Thread.sleep(1000);

        // AddShutdownHook方法增加JVM停止时要做处理事件：

        // 当JVM退出时，打印JVM Exit语句.
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                System.out.println("JVM Exit!");
            }
        });

}}
```

>> Note: 守护线程不要去做一些文件、数据库等操作，因为一旦用户线程都完成操作退出后，守护线程也需要退出，这个时候可能会导致内存溢出等风险。

## <a id="ThreadPool">Java线程池API</a>

在前言中，引入的`ExecutorService`是对原生线程池`ThreadPoolExecutor`类的封装，提供了4种构造不同需求的线程池方法。首先，还是先介绍下`ThreadPoolExecutor`，API接口定义如下：

``` java
	/**
     * 根据给定的初始化参数创建一个新的 {@code ThreadPoolExecutor} 对象.
     *
     * @param corePoolSize 线程池中维持的线程数，即使所有线程都是空闲状态；除非设置了{@code allowCoreThreadTimeOut}。
     * @param maximumPoolSize 线程池中允许的最大数量的线程。
     * @param keepAliveTime 当线程数量比corePoolSize的值大时，这个变量指定了在结束之前，多余的线程等待新来任务时最长的时间。
     * @param unit 参数{@code keepAliveTime} 的时间单位
     * @param workQueue 在任务执行之前，存储这些任务的队列queue。这个队列只会保存通过{@code execute}提交的{@code Runnable}任务。
     * @param threadFactory 当创建一个新的线程时候，使用的工厂factory对象。
     * @param handler 当执行任务出现阻塞的时候，使用的处理器handler。一般，可能当前的线程上线和队列容量都已经饱和的时候，
     *        就需要对新进来的任务执行相应处理策略。
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

对于该构造函数的参数说明，已经对应的一些注意事项可以参考 [JDK 6 ThreadPoolExecutor API中文](http://dlc.sun.com.edgesuite.net/jdk/jdk-api-localizations/jdk-api-zh-cn/builds/latest/html/zh_CN/api/)。但是需要对其中`BlockingQueue<Runnable>`，`ThreadFactory`，`RejectedExecutionHandler`进行说明。

### 3.1 BlockingQueue<Runnable> 介绍

在jdk 6中对`BlockingQueue`接口进行了详细的说明，主要几点如下：

1. BlockingQueue 实现主要用于生产者-使用者队列，但它另外还支持 Collection 接口。因此，举例来说，使用 remove(x) 从队列中移除任意一个元素是有可能的。然而，这种操作通常不会有效执行，只能有计划地偶尔使用，比如在取消排队信息时。
	
2. BlockingQueue 实现是线程安全的。所有排队方法都可以使用内部锁或其他形式的并发控制来自动达到它们的目的。然而，大量的 Collection 操作（addAll、containsAll、retainAll 和 removeAll）没有 必要自动执行，除非在实现中特别说明。因此，举例来说，在只添加了 c 中的一些元素后，addAll(c) 有可能失败（抛出一个异常）。

3. BlockingQueue 实质上不 支持使用任何一种“close”或“shutdown”操作来指示不再添加任何项。这种功能的需求和使用有依赖于实现的倾向。例如，一种常用的策略是：对于生产者，插入特殊的 end-of-stream 或 poison 对象，并根据使用者获取这些对象的时间来对它们进行解释。

4. 此外，BlockingQueue 可以安全地与多个生产者和多个使用者一起使用。
	
在java 中默认实现了4种阻塞队列，提供四种不同的阻塞队列模型：

1. ArrayBlockingQueue, 底层由数组构成的有界阻塞队列。按照FIFO(先进先出)策略对元素进行排序。因此，队列的头部是当前队列中，最早进入队列的元素，而队尾则是最后进入队列的元素。并且，新来的任务元素，都插入到队列的尾部，执行任务的时候，从队列的头部取出任务元素。
	
2. LinkedBlockingQueue, 底层由链表组成的阻塞队列。同样是按照FIFO策略对元素进行排序。和ArrayBlockingQueue不同的是，基于链表的阻塞队列可以不设置队列的大小，从而构造一个无界队列；此外，LinkedBlockingQueue的吞吐量也要高于数组的阻塞队列，不过，它会造成部分元素插入顺序的不确定性。
	
3. SynchronousQueue，同步的阻塞队列，不存储元素，没有任何内部容量。因此，这决定了该队列模型是一个同步操作，即每一个生产者的任务消息都会直接给消费者处理，而不会先保存起来，让消费者从队列中FIFO来获取最老的消息元素。其特点就是：每个插入操作必须等待另一个线程的对应移除操作 ，反之亦然。适合传递性设计，设计中，在一个线程中运行的对象要将某些信息、事件或任务传递给在另一个线程中运行的对象，它就必须与该对象同步。
	
4. DelayQueue<E extends Delayed>，延迟的无界阻塞队列。队列中的元素只有在只有在延迟期满时才能从中提取元素。该队列的头部是延迟期满后保存时间最长的 Delayed 元素。如果延迟都还没有期满，则队列没有头部，并且 poll 将返回 null。
	
5. LinkedBlockingDeque，底层有双向链表构成的阻塞队列。和LinkedBlockingQueue一样，可以做无界队列，只是因为可以从两端插入和获取元素，所以时间消耗是单向链表的一半；当然这是一种空间换时的策略。该队列需要设置队列大小来防止过度膨胀。
	
6. PriorityBlockingQueue，一个无界的阻塞队列，它的使用和类 PriorityQueue 相同的顺序规则，并且提供了阻塞获取操作。虽然此队列逻辑上是无界的，但是资源被耗尽时试图执行 add 操作也将失败（导致 OutOfMemoryError）。iterator() 方法中提供的迭代器并不 保证以特定的顺序遍历 PriorityBlockingQueue 的元素。如果需要有序地进行遍历，则应考虑使用 Arrays.sort(pq.toArray())。此外，可以使用方法 drainTo 按优先级顺序移除 全部或部分元素，并将它们放在另一个 collection 中。
		
**ArrayBlockingQueue：**	  

>> Note: `ArrayBlockingQueue`队列是有界的队列，所以当队列满的时候，如果还向该队列插入元素，则会导致操作被阻塞住，当然，如果从空的队列中获取元素，该操作也会被阻塞。此外，构造`ArrayBlockingQueue`队列时，有一个参数为：` boolean fair` ：如果为 true，则按照 FIFO 顺序访问插入或移除时受阻塞线程的队列；如果为 false，则访问顺序是不确定的. 

**LinkedBlockingQueue：**  

>> Note: `LinkedBlockingQueue` 队列的吞吐量也要高于数组的阻塞队列，这主要是因为数组的特性和链表的特性决定的，链表在处理元素的offer队头元素和add队尾元素的速度要快于相应地数组操作。不过，显然这样会造成部分元素插入顺序的不确定性。

**`DelayQueue<E extends Delayed>`：** 

>> Note: `DelayQueue<E extends Delayed>`队列中的元素需要实现`Delayed`接口，该接口只有`long getDelay(TimeUnit unit);`方法即可使用延迟阻塞队列。此外，需要注意，可能存在的时间延时，即任务元素不一定会准时执行，会有一点点的延迟。

**LinkedBlockingDeque：**  

>> Note: `LinkedBlockingDeque`队列用的最多的地方，就是使用`工作窃取算法`的地方。工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。如下图（参考[工作窃取运行说明](http://ifeve.com/talk-concurrency-forkjoin/)）：

<img src="/images/2014/05/work-stealing.png" />

**PriorityBlockingQueue：** 

>> Note: `PriorityBlockingQueue`队列，默认情况下元素采取自然顺序排列，也可以通过比较器comparator来指定元素的排序规则。比较器可使用修改键断开主优先级值之间的联系。元素默认按照升序排列。

选择其中的`LinkedBlockingQueue`来简单分析下，其内部实现结构和细节：

``` java
   public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null); // Node是链表中一个节点，包含一个元素和下一个元素的引用
    }
```

从上面的代码可以看到，`LinkedBlockingQueue`实质上就是一个链表结构。作为阻塞的队列，在插入和移出元素的时候，肯定会加一个特殊的操作控制。在代码中，可以很清楚的看到，其消费者和生产者是通过singal来维护的，包括`notFull`和`notEmpty`两个信号变量。

``` java
   /**
     * 插入指定元素到队列的尾部，如果没有空间的话，等待。
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node(e); //创建插入链表的节点node
        final ReentrantLock putLock = this.putLock; // 使用自旋锁，确保插入时线程安全
        final AtomicInteger count = this.count; // 原子类型整型
        putLock.lockInterruptibly();// 可中断加锁
        try {
            /*
             * 在这里的count并没有使用锁来保护，这是因为这里只有递减操作，并且我
             * 们在容量大小更改的时候将会发送信号，这和在其他等待guard计数相似。 
             */
            while (count.get() == capacity) {
                notFull.await(); //等待，直到有空间插入元素
            }
            enqueue(node); // 插入元素
            c = count.getAndIncrement(); //插入成功
            if (c + 1 < capacity)
                notFull.signal(); //释放信号，队列未满
        } finally {
            putLock.unlock(); //释放锁
        }
        if (c == 0)
            signalNotEmpty(); //发送信号，表明当前队列为空。使用全局takeLock 自旋锁来加锁设置发送信号
    }
    
    
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull(); //发送信号告知当前队列已满，使用全局putLock 自旋锁来加锁发送信号。
        return x;
    }

```

>> Note: `put`在插入的时候，会一直等待插入成功；如果需要设置等待超时时间，需要使用`offer(E e, long timeout, TimeUnit unit)`来插入元素。
此外，`take`方法和`put`方法整体流程基本一样。

### 3.2 ThreadFactory 介绍

`ThreadFactory`，线程工厂，顾名思义，就是采用工厂模式来创建线程实例。使用`ThreadFactory`方式构建线程，可以不调用`{@link Thread#Thread(Runnable) new Thread}`方法来new 一个新的线程，这样可以更方便的让应用使用定制好了的线程子类，属性等。
`ThreadFactory`接口，只有一个需要实现的方法，接口定义为：

``` java
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```

我们一般使用`Executors`类中提供的`DefaultThreadFactory`对接口进行了简单地实现，我们在代码中使用`Executors`来创建线程池时，会用到这个默认线程工厂类。

``` java
/**
     * The default thread factory
     */
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1); //线程池序号
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1); //线程所在池中的序号
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup(); // 当前线程组名
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-"; // 线程前缀组合名
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0); // 封装了new 对象的方法
            if (t.isDaemon())
                t.setDaemon(false); // 设置为非deamon 线程
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY); //设置为默认优先级 
            return t;
        }
    }
```

>> Note: 除了`Executors`使用的默认的线程工厂类之外，还提供了一个线程工厂类：`PrivilegedThreadFactory`类。该类继承了`DefaultThreadFactory`，增加了访问控制上下文和类加载器，会检查类的调用者是否有相关权限。例如：`System.getSecurityManager().checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);`以及`System.getSecurityManager().checkPermission(new RuntimePermission("setContextClassLoader"));`。

### 3.3 RejectedExecutionHandler 介绍

`RejectedExecutionHandler`类是对线程池中不能被执行的任务，所需要采用的处理策略的指定。当 `executor` 不能接受某个任务时，可以由 `ThreadPoolExecutor` 调用`RejectedExecutionHandler`指定的处理方法。这种不能接受任务的情况，很容易就发生了，比如当超出其界限而没有更多可用的线程或队列池时，或者关闭 Executor 时。默认情况下，`private static final RejectedExecutionHandler defaultHandler = new AbortPolicy()`。

``` java
public interface RejectedExecutionHandler {

    /**
     * <p>In the absence of other alternatives, the method may throw
     * an unchecked {@link RejectedExecutionException}, which will be
     * propagated to the caller of {@code execute}.
     *
     * @param r the runnable task requested to be executed
     * @param executor the executor attempting to execute this task
     * @throws RejectedExecutionException if there is no remedy
     */
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

JDK 6 提供了 4 种处理拒绝执行任务的策略：

	1. `AbortPolicy`类，该策略很简单，如果出现任务要被拒绝处理，则会抛出`RejectedExecutionException`异常，该策略为默认处理方式。
	
	2. `CallerRunsPolicy`类，该策略会直接在`execute`方法的调用线程中运行该呗拒绝执行的任务；如果执行程序已经关闭，则直接丢弃该任务。
	
	3. `DiscardOldestPolicy`类，该策略会在出现拒绝执行任务的时候，放弃队列中最老的未被处理的请求，然后重试execute；如果执行程序关闭，同样直接丢弃该任务。
	
	4. `DiscardPolicy`类，该策略同样很简单，就是如果出现被拒绝执行的任务，则直接丢弃该任务。
	
比如`DiscardOldestPolicy`策略的实现，其把任务阻塞队列中得队头元素丢弃掉，然后重新执行该任务。

``` java
	/**
     * A handler for rejected tasks that discards the oldest unhandled
     * request and then retries {@code execute}, unless the executor
     * is shut down, in which case the task is discarded.
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll(); // 丢弃最老的元素
                e.execute(r); //重试执行该拒绝任务
            }
        }
    }
```

## <a id="Executors">Java Executors类介绍</a>

虽然`Executors`类只是对`ThreadPoolExecutor`的一些属性进行组合封装，但是，一般地，我们只需要使用该工具类完成创建线程池，就可以基本上满足我们的需求。
`Executors`类提供了创建4种不同属性的线程池，分别为：

``` java
	/**
     * 创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程，在需要时使用提供的 ThreadFactory 创建新线程。
     * 在任意点，大多数 nThreads 线程会处于处理任务的活动状态。如果在所有线程处于活动状态时提交附加任务，则在有可用线程之前，
     * 剩余任务将在队列中一直等待。如果在关闭前的执行期间由于失败而导致任何线程终止，那么一个新线程将代替它执行后续的任务（如果需要）。
     * 在某个线程被显式地关闭{@link ExecutorService#shutdown shutdown}之前，池中的线程将一直存在。
     *
     * @param nThreads the number of threads in the pool
     * @param threadFactory 默认使用Executors.defaultThreadFactory()线程工厂，使用抛出异常的AbortPolicy处理策略
     * @throws IllegalArgumentException if {@code nThreads <= 0}
     */
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
    
    /**
     * 创建一个使用单个 worker 线程的 Executor，以无界队列方式来运行该线程，
     *（注意，如果因为在关闭前的执行期间出现失败而终止了此单个线程，那么如果需要，一个新线程将代替它执行后续的任务）。
     * 并在需要时使用提供的 ThreadFactory 创建新线程。与其他等效的 newFixedThreadPool(1, threadFactory) 不同，
     * 可保证无需重新配置此方法所返回的执行程序即可使用其他的线程。
     */
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
    
    /**
     * 创建一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们。
     * 对于执行很多短期异步任务的程序而言，这些线程池通常可提高程序性能。调用 execute 将重用以前构造的线程（如果线程可用）。
     * 如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。
     * 因此，长时间保持空闲的线程池不会使用任何资源。注意，可以使用 ThreadPoolExecutor 构造方法创建具有类似属性但细节不同（例如超时参数）的线程池。
     */
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
    
    /**
     * 创建一个单线程执行程序，它可安排在给定延迟后运行命令或者定期地执行。
     *（注意，如果因为在关闭前的执行期间出现失败而终止了此单个线程，那么如果需要，一个新线程会代替它执行后续的任务）。
     * 可保证顺序地执行各个任务，并且在任意给定的时间不会有多个线程是活动的。
     * 与其他等效的 newScheduledThreadPool(1, threadFactory) 不同，可保证无需重新配置此方法所返回的执行程序即可使用其他的线程。
     */
    public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1, threadFactory));
    }
    
    /**
     * 创建一个线程池，它可安排在给定延迟后运行命令或者定期地执行。
     * @param corePoolSize 池中所保存的线程数，即使线程是空闲的也包括在内。
     */
    public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }


```

在前言部分使用了`Executors.newFixedThreadPool`来创建固定线程数的线程池。因此，我们就对这个代码的整个流程进行说明。

>> Note： 代码首先new 一个线程池，如上面代码所示，直接调用`ThreadPoolExecutor`构造函数即可。接下来就是创建任务放在线程池中执行了。

``` java
	/**
     * 在将来某个时间执行给定任务。可以在新线程中或者在现有池线程中执行该任务。 
     * 如果无法将任务提交执行，或者因为此执行程序已关闭，或者因为已达到其容量，则该任务由当前 RejectedExecutionHandler 处理。     
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. 如果比指定的corePoolSize值要少的线程在运行，则尝试着使用给定的factory来新建一个线程来运行该Runnable任务。
         * 这次addworker()方法的调用会自动检查运行状态和工作者worker数量，所以如果不允许增加worker则会返回false。具体实现参见下面分析。
         *
         * 2. 如果任务被插入队列，然后我们仍然需要再次检查是否我们应该增加一个线程（可能会有某一个线程在上一次检查完之后挂掉了），
         * 或者一进入该方法，线程池就down掉了。所以我们重复检查状态，并在如果需要，则回滚进入队列，或者开启新的线程。
         *
         * 3. 如果我们不可以插入任务到队列，则我们会尝试新加一个线程。如果增加失败，我们需要现在线程池已经关闭了或者饱和了，因此拒绝任务进入。
         */                  
        int c = ctl.get(); // 获取线程池中有效的线程数
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true)) //增加新的工作线程运行新的任务command
                return;
            c = ctl.get(); //增加新的失败，则获得有效线程数，进行再次尝试
        }
        if (isRunning(c) && workQueue.offer(command)) { // 如果worker正在运行任务，则把新的command放在queue中去。
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command)) //非running状态的线程是不接受任务的，所以从队列中移除任务
                reject(command); //并且执行拒绝操作
            else if (workerCountOf(recheck) == 0) //如果当前没有running线程是因为线程池没有线程，则增加非core线程。
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command); //增加线程失败，则调用对接的策略来执行拒绝该任务
    }
    
        public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate(); // In case SHUTDOWN and now empty
        return removed;
    }
```
>> Tips: 在代码中，获取当前程序中运行的线程数，是一个很有趣的实现。核心代码如下：

``` java
	// 主线程池控制状态 ctl，表示当前有效地线程数，此外还可以指示是否是running、shutdown等状态
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3; //Integer.SIZE = 32
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;// 00011111111111111111111111111111
    
    /**
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     */
    
    // runState 一共5种，使用高位的3位即可完全表示
    private static final int RUNNING    = -1 << COUNT_BITS; // 11100000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS; // 00000000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS; // 00100000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS; // 01000000000000000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS; // 01100000000000000000000000000000
    
    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; } //获取线程的状态
    private static int workerCountOf(int c)  { return c & CAPACITY; } //获取线程池的线程数
    private static int ctlOf(int rs, int wc) { return rs | wc; } //获取某一状态的值
```



>> Note: addWorker方法，会检查如果增加一个新的worker后，是否满足当前线程池的状态和最初创建线程池设置的大小限制。这可能会创建一个新的工作线程worker来运行该任务。但是，如果线程池停止或者将要关闭，再或者使用线程工厂创建新线程失败，都将返回false。
> `Worker`的继承`AbstractQueuedSynchronizer`类，该类是实现基于FIFO等待队列的阻塞锁和相关同步器的一个基本的框架，其可以依靠单个原子int值来表示状态。因此，`Worker`类其实主要目的就是为了维持线程的运行的任务的状态而存在的工作者队列。

``` java
	/**
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c); //运行的状态

            // Check if queue empty only if necessary.非running状态
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            w = new Worker(firstTask); // 新建worker，并且指定第一个任务
            final Thread t = w.thread;
            if (t != null) {
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {//正常的线程状态
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w); //非core线程数时，加入到任务队列中
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start(); //执行worker任务，详细见下面分析
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

这段代码比较复杂，其主要就是判断新建worker线程的环境条件，如果可以创建，则执行相应地任务`w = new Worker(firstTask);final Thread t = w.thread; t.start()`，否则返回false；`execute`方法会执行相关拒绝策略的操作。

**Worker中任务的执行**  
但是，从上面的代码中，我们看到`worker`的新建流程，并且把新任务作为参数来初始化worker，但是执行worker有一个目的只是为了测试worker实例是否创建成功。根据上面的API介绍，应该猜到其实大部分的任务到达线程池的时候，显然不是都新建一个线程来处理，而是放进`queue`中，然后执行。在`Worker`类中，封装需要执行的Runnable任务，然后其重写了run方法，内部调用`runWorker`执行任务。

>> Note:`runWorker`是`worker`主要的工作。就是重复的从queue中获取任务，然后执行他们。整个流程大概如下：    
>1. 我们可能会从一个初始的任务开始，当然非core数创建的`worker`则没有第一个`task`。此外
    在pool运行期间，我们使用`getTask`方法来获取任务。如果返回null的时候，则退出worker线程。
    另外，如果执行的任务会抛出异常，也会导致worker突然地完成，进而会使用`processWorkerExit`来代替该线程。    
>2. 在执行任何任务task之前，需要需求`lock`锁和调用`clearInterruptsForTaskRun`方法，这是为了防止在任务正在执行的时候，其他线程池中断。  
>3. 每个任务在递交运行之前，都会调用`beforeExecute`。这个方法可能会抛出一个异常，这个异常可以导致线程down掉，而不需要执行任务task。  
>4. 假设	`beforeExecute`顺利完成了，则开始运行task。在此期间产生的任务异常都会抛给`afterExecute`方法。分别会处理`RuntimeException`,`Error`,以及任意的`Throwables`。由于我们不可以在run方法中重新抛出`Throwables`，所以我们封装它们在即将过时的`Errors`里给线程的`UncaughtExceptionHandler`方法来处理。任何抛出来得异常也会导致线程down掉。  
>5. 在run方法完成之后，就会调用`afterExecute`方法，这也会抛出一个异常，当然也会导致线程down掉。According to `JLS Sec 14.20`, this exception is the one that will be in effect even if `task.run` throws.


代码如下：

``` java

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts。使该worker状态为0，即可以运行新的任务。
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) { //如果task为null的时候，则从队列中获取任务
                w.lock(); //设置0为1，表示该worker不可用，原子操作。
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted()) // 在这些情况下，需要中断当前的线程。
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();//执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);//执行后处理异常等信息
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();//恢复当前worker可工作
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);//为脏worker执行清扫工作和记账工作，true时，方法会把worker的线程移出，或者替换worker等
        }
    }



    /**
     * 从queue中获取需要执行的任务
     * Performs blocking or timed wait for a task, depending on
     * current configuration settings, or returns null if this worker
     * must exit because of any of:
     * 1. There are more than maximumPoolSize workers (due to
     *    a call to setMaximumPoolSize).
     * 2. The pool is stopped.
     * 3. The pool is shutdown and the queue is empty.
     * 4. This worker timed out waiting for a task, and timed-out
     *    workers are subject to termination (that is,
     *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
     *    both before and after the timed wait.
     *
     * @return task, or null if the worker must exit, in which case
     *         workerCount is decremented
     */
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        retry:
        for (;;) {
            
            ........
            
             try {
             // 获取任务，超时设计判断获取逻辑
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }



```


## <a id="CountDownLatch">Java CountDownLatch类介绍</a>

`CountDownLatch`类，是一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或者多个线程一直等待。

用给定的`计数Count` 初始化 `CountDownLatch`。由于调用了 `countDown()` 方法，所以在当前计数到达零之前，`await` 方法会一直受阻塞。之后，会释放所有等待的线程，await 的所有后续调用都将立即返回。这种现象只出现一次——计数无法被重置。如果需要**重置计数，请考虑使用 `CyclicBarrier`**。

`CountDownLatch` 是一个通用同步工具，它有很多用途。将计数 1 初始化的 `CountDownLatch` 用作一个简单的开/关锁存器，或入口：在通过调用 `countDown()` 的线程打开入口前，所有调用 `await` 的线程都一直在入口处等待。用 `N` 初始化的 `CountDownLatch` 可以使一个线程在 N 个线程完成某项操作之前一直等待，或者使其在某项操作完成 N 次之前一直等待。

`CountDownLatch` 的一个有用特性是，它不要求调用 `countDown`方法的线程等到计数到达零时才继续， 而在所有线程都能通过之前，它只是通过一个 `await`阻止任何线程继续。

知道`CountDownLatch`类作用，我们就可以回到前言中说到的一个简单地多线程处理问题。我在一开始的时候，直接使用线程池执行多组任务，虽然考虑了多个线程在处理完任务之后，把结果add到list里面会有线程安全问题，但是放了一个非常大的`错误`，就是线程池创建完线程，分配给完所有任务之后，主线程Main会接着往下执行，即打印结果。而这时，非常大的可能是，线程全部都在执行，并没有结果add到list中，导致list可能并不是完整地结果集，甚至有些情况下list还会为空。

因此，这个时候就需要`CountDownLatch`上场了。经过修改的代码如下：

``` java
/**
 * @author: ketao Date: 14-5-3 Time: 下午4:51
 * @version: \$Id$
 */
public class ThreadTest {

    private static final ExecutorService executors = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {

List<String> list = Lists.newArrayList("thread-11", "thread-21", "thread-31", "thread-41");
        final List<String> results = Collections.synchronizedList(new ArrayList<String>());
        final CountDownLatch latch = new CountDownLatch(list.size());

        for (final String str : list) {
            executors.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000);

                        results.add(str + "-test");
                        System.out.println(Thread.currentThread().getName());

                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        latch.countDown();
                    }
                }
            });
        }
        latch.await();
        System.out.println(JSON.toJSONString(results));
        executors.shutdown();
    }
 }

```

## <a id="Finally">总结</a>

在最后，对于`CountDownLatch`类并没有详细的进行介绍，只是使用该类修复了前言中有问题的代码。其实在Java 7中，对于期待结果的多线程任务，推荐使用Fork & Join 方式来处理。关于多线程其他方面的介绍，将在以后慢慢给出。

