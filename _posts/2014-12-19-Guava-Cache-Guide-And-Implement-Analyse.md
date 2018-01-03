---
layout: post
title: Guava LocalCache 缓存介绍及实现源码深入剖析
date: 2014-12-19 22:21:35 +0800
comments: true
categories: 
  - Java
---

## <a id="Intro">前言</a>
Guava是Google开源出来的Java常用工具集库,包括集合|缓存|并发|字符串|IO操作等在Java开发过程中经常需要去实现的工具类.

显然,对于这种十分常见的需求,Guava提供了自己的工具类实现.GuavaCache 提供了一般我们使用缓存所需要的几乎所有的功能,主要有:

* 自动将entry节点加载进缓存结构中;

* 当缓存的数据已经超过预先设置的最大值时，使用LRU算法移除一些数据；

* 具备根据entry节点上次被访问或者写入的时间来计算过期机制；

* 缓存的key被封装在`WeakReference`引用内;

* 缓存的value被封装在`WeakReference`或者`SoftReference`引用内；

* 移除entry节点，可以触发监听器通知事件；

* 统计缓存使用过程中命中率/异常率/未命中率等数据。

此外，`Guava Cache`其核心数据结构大体上和`ConcurrentHashMap`一致，具体细节上会有些区别。功能上，ConcurrentMap会一直保存所有添加的元素，直到显式地移除.相对地,`Guava Cache`为了限制内存占用,通常都设定为自动回收元素.在某些场景下,尽管它不回收元素,也是很有用的,因为它会自动加载缓存.

<!-- more -->

## <a id="CacheGuide">Guava Cache 介绍</a>

在介绍`Guava Cache`使用之前，先需要引入下官方推荐的使用场景：

    * 愿意消耗一些内存空间来提升速度；
        
    * 能够预计某些key会被查询一次以上；
        
    * 缓存中存放的数据总量不会超出内存容量(`Guava Cache`是单个应用运行时的本地缓存)。

不管性能，还是可用性来说，`Guava Cache`绝对是本地缓存类库中首要推荐的工具类。其提供的`Builder模式`的CacheBuilder生成器来创建缓存的方式，十分方便，并且各个缓存参数的配置设置，类似于函数式编程的写法，也特别棒。

`Guava Cache`的官方文档地址：<http://code.google.com/p/guava-libraries/wiki/CachesExplained>. 该文档对`Cache`有详细的介绍。
<br/>

>> Tips：在官方文档中，提到三种方式加载`<key,value>`到缓存中。分别是:
>> 
>>   1. `LoadingCache`在构建缓存的时候，使用build方法内部调用`CacheLoader`方法加载数据；
>>  
>>   2. 在使用get方法的时候，如果缓存不存在该key或者key过期等，则调用`get(K, Callable<V>)`方式加载数据；
>>  
>>  3. 使用粗暴直接的方式，直接想缓存中put数据。
>>  
>>  需要说明的是，如果不能通过key快速计算出value时，则还是不要在初始化的时候直接调用`CacheLoader`加载数据到缓存中。


### 2.1 Guava Cache使用示例

``` java

import java.util.Date;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import com.google.common.cache.RemovalListener;
import com.google.common.cache.RemovalNotification;

/**
 * @author tao.ke Date: 14-12-20 Time: 下午1:55
 * @version \$Id$
 */
public class CacheSample {

    private static final Logger logger = LoggerFactory.getLogger(CacheSample.class);

    // Callable形式的Cache
    private static final Cache<String, String> CALLABLE_CACHE = CacheBuilder.newBuilder()
            .expireAfterWrite(1, TimeUnit.SECONDS).maximumSize(1000).recordStats()
            .removalListener(new RemovalListener<Object, Object>() {
                @Override
                public void onRemoval(RemovalNotification<Object, Object> notification) {
                    logger.info("Remove a map entry which key is {},value is {},cause is {}.", notification.getKey(),
                            notification.getValue(), notification.getCause().name());
                }
            }).build();

    // CacheLoader形式的Cache
    private static final LoadingCache<String, String> LOADER_CACHE = CacheBuilder.newBuilder()
            .expireAfterAccess(1, TimeUnit.SECONDS).maximumSize(1000).recordStats().build(new CacheLoader<String, String>() {
                @Override
                public String load(String key) throws Exception {
                    return key + new Date();
                }
            });

    public static void main(String[] args) throws Exception {

        int times = 4;
        while (times-- > 0) {

            Thread.sleep(900);

            String valueCallable = CALLABLE_CACHE.get("key", new Callable<String>() {
                @Override
                public String call() throws Exception {
                    return "key" + new Date();
                }
            });

            logger.info("Callable Cache ----->>>>> key is {},value is {}", "key", valueCallable);
            logger.info("Callable Cache ----->>>>> stat miss:{},stat hit:{}",CALLABLE_CACHE.stats().missRate(),CALLABLE_CACHE.stats().hitRate());

            String valueLoader = LOADER_CACHE.get("key");

            logger.info("Loader Cache ----->>>>> key is {},value is {}", "key", valueLoader);
            logger.info("Loader Cache ----->>>>> stat miss:{},stat hit:{}",LOADER_CACHE.stats().missRate(),LOADER_CACHE.stats().hitRate());

        }

    }
}

```

>> 上述代码，简单的介绍了`Guava Cache `的使用，给了两种加载构建Cache的方式。在`Guava Cache`对外提供的方法中， `recordStats`和`removalListener`是两个很有趣的接口，可以很好的帮我们完成统计功能和Entry移除引起的监听触发功能。
>> 
>> 此外，虽然在`Guava Cache`对外方法接口中提供了丰富的特性，但是如果我们在实际的代码中不是很有需要的话，建议不要设置这些属性，因为会额外占用内存并且会多一些处理计算工作，不值得。
>> 

## <a id="PrepareKnowledge">Guava Cache 分析前置知识</a>

`Guava Cache`就是借鉴Java的`ConcurrentHashMap`的思想来实现一个本地缓存，但是它内部代码实现的时候，还是有很多非常精彩的设计实现，并且如果对`ConcurrentHashMap`内部具体实现不是很清楚的话，通过阅读`Cache`的实现，对`ConcurrentHashMap`的实现基本上会有个全面的了解。

### 3.1 Builder模式 

设计模式之 Builder模式 在Guava中很多地方得到的使用。`Builder模式`是将一个复杂对象的构造与其对应配置属性表示的分离，也就是可以使用基本相同的构造过程去创建不同的具体对象。

Builder模式典型的结构图如：

<img src="/images/2014/12/builder.png" />

    Builder：为创建一个Product对象的各个部件制定抽象接口；

    ConcreteBuilder：具体的建造者，它负责真正的生产；

    Director：导演, 建造的执行者，它负责发布命令；

    Product：最终消费的产品

各类之间的交互关系如下图：

<img src="/images/2014/12/builder-relation.png" />

`Builder模式`的关键是其中的Director对象并不直接返回对象，而是通过（BuildPartA，BuildPartB，BuildPartC）来一步步进行对象的创建。当然这里Director可以提供一个默认的返回对象的接口（即返回通用的复杂对象的创建，即不指定或者特定唯一指定BuildPart中的参数）。

>> Tips：在`Effective Java`第二版中，`Josh Bloch`在第二章中就提到使用Builder模式处理需要很多参数的构造函数。他不仅展示了Builder的使用，也描述了相这种方法相对使用带很多参数的构造函数带来的好处。
>> 

下面给出一个使用Builder模式来构造对象，这种方式优点和不足（代码量增加）非常明显。

``` java

import org.apache.commons.lang3.builder.ToStringBuilder;
import org.apache.commons.lang3.builder.ToStringStyle;

/**
 * @author tao.ke Date: 14-12-22 Time: 下午8:57
 * @version \$Id$
 */
public class BuilderPattern {

    /**
     * 姓名
     */
    private String name;

    /**
     * 年龄
     */
    private int age;

    /**
     * 性别
     */
    private Gender gender;

    public static BuilderPattern newBuilder() {
        return new BuilderPattern();
    }

    public BuilderPattern setName(String name) {
        this.name = name;
        return this;
    }

    public BuilderPattern setAge(int age) {
        this.age = age;
        return this;
    }

    public BuilderPattern setGender(Gender gender) {
        this.gender = gender;
        return this;
    }

    @Override
    public String toString() {
        return ToStringBuilder.reflectionToString(this, ToStringStyle.SHORT_PREFIX_STYLE);
    }

    enum Gender {
        MALE, FEMALE
    }

    public static void main(String[] args) {

        BuilderPattern bp = BuilderPattern.newBuilder().setAge(10).setName("zhangsan").setGender(Gender.FEMALE);
        System.out.println(bp.toString());
    }

}

```


### 3.2 Java对象引用

对象引用之前需要先看看对象的访问定位。

当虚拟机执行时，遇到一条new指令时，首先会去检查这个指令在常量池中是否已经存在该类对应的符号引用，并且检查这个符号引用对应的类是否已经被加载，解析和初始化。如果没有，则执行相应的类加载过程。

然后虚拟机为新的对象分配内存。虚拟机根据我们配置的垃圾收集器规则采取不同的分配方式，包括：指针碰撞分配方式和空闲列表分配方式。

内存分配完成之后，开始执行init方法。init方法会按照代码的指定过程来初始化，对一些类属性进行赋值。

然后，我们需要访问这个对象，怎么办？在Java运行时内存区域模型中，线程拥有一个虚拟机栈，这个栈会有一个本地方法表，这个表内部就会存放一个引用地址，如下图所示（HotSpot虚拟机采用这种方式，还有另外一种形式这里不做介绍）：

<img src="/images/2014/12/reference.png" />


在JDK 1.2之前，Java中关于引用的定义是：如果reference类型的数据中存储的数值表示的是另外一块内存的起始地址，就说明这块内存称为引用。这种定义表明对象只有两种：被引用的对象和没有被引用的对象。这种方式对于垃圾收集GC来说，效果并不是很好，因为很多对象划为为被引用和非被引用都不是很重要，这种现象就无法划分。垃圾收集的时候，就无法更好更精准的划为可GC的对象。

因此，在JDK 1.2之后，Java对引用的概念进行扩展，有如下四种类型的引用（按强度排序）：

    * 强引用(Strong Reference)

    * 软引用(SoftReference)

    * 弱引用(WeakReference)

    * 虚引用(PhantomReference)

1. *强引用*：强引用在程序代码中随处可见，十分普遍。比如： `Object object = new Object()` ，这类引用只要还存在，垃圾收集器就永远不会回收掉这类引用的对象。

2. *软引用*：软引用用来描述一些虽然有用但是并不是必须的对象。对于软引用关联的对象，在系统将可能发生内存溢出异常之前，垃圾收集器将会把这些引用的对象进行第二次回收。只有这次垃圾回收还没有足够的内存的时候，才会抛出内存溢出异常。

3. *弱引用*：弱引用是一种比软引用强度还要弱的引用，因此这些引用的对象也是非必须的。但是，对于弱引用的对象只能生存到下一次垃圾回收发生之前。当垃圾收集工作开始后，无论当前的内存是否够用，都会把这些弱引用的对象回收掉。

4. *虚引用*：虚引用是最弱的一种引用。一个对象是否被虚引用关联，完全不会对其生存时间构成影响，也无法通过虚引用获得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。

>> Notes：关于引用，最典型的使用就是对HashMap的自定义开发，包括JDK内部也存在。
>> 
>> 1. `Strong Reference`---> `HashMap`：默认情况下，HashMap使用的引用就是强引用，也就是说垃圾收集的时候，Map中引用的对象不会被GC掉。
>> 
>> 2. `Weak Reference`---> `WeakHashMap`：JDK中还有一种基于引用类型实现的HashMap，WeakHashMap。当节点的key不在被使用的时候，该entry就会被自动回收掉。因此，对于一个mapping映射，不能保证接下来的GC不会把这个entry回收掉。
>> 
>> 3. `Soft Reference`---> `SoftHashMap`：在JDK中没有提供基于软引用实现的HashMap，原因可能是一般大家都不能期待出现内存溢出，而当出现内存溢出，一点点的软引用GC余下的内存空间，肯定不会起到关键作用。但是，虽然不广泛，在`aspectj`提供的`ClassLoaderRepository`类中实现了SoftHashMap，作为一个基于ClassLoader字节码实现的方法，在OOM的时候，显然需要考虑通过GC释放内存空间，并且SoftHashMap在内部是作为缓存使用。


### 3.3 JMM可见性

在[Java一些小Tips](http://ketao1989.github.io/posts/java-some-tips.html)博文中，简单地介绍了JMM模型，但是Java内存模型涉及了大量的规则内容指令。

*什么叫可见性？*

可见性就是，当程序中一个线程修改了某个全局共享变量的值之后，其他使用该值的线程都可以获知，在随后他们读该共享变量的时候，查询的都是最新的改改修改的值。

在上一篇博文中，我们给出了内存模型访问的图。根据图可以了解，一个线程上修改共享变量，这个变量的最新的值不会立刻写入到共享内存中，还是暂时存放在线程本地缓存，然后某一时刻触发写入到共享内存中。可见性就是，当我们对共享变量修改的时候，立刻把新值同步到主内存中，然后该变量被读的时候从主内存获取最新的值确保所有对该变量的读取操作，总是获取最新最近修改的值。

*为什么会有可见性问题？*

学过计算机组成原理的同学都知道，在现代CPU结构中，存在多级缓存架构，如下图所示：

<img src="/images/2014/12/cpu_cache.jpg" />

同样，在Java虚拟机中分为两种内存：

    >> 主内存(Main Memory)：所有线程共享的内存区域，虚拟机内存的一部分。
    
    >> 工作内存(Working Memory)：线程自己操作的内存区域，线程直接无法访问对方的工作内存区域。

之所以分为两部分内存区域，原因和CPU很类似。为了线程可以快速访问操作变量，当线程全部直接操作共享内存，则会导致大量线程之间竞争等问题出现，影响效率。

关于Java中线程，工作内存，主内存之间的交互关系如下图（深入理解Java虚拟机配图）：

<img src="/images/2014/12/java_mmm.png" />

为了保证共享变量可见性，除了上篇博文中介绍的`volatile`之外，还有`synchronized`和`final`关键字。

*synchronized*：执行synchronized代码块时，在对变量执行unlock操作之前，一定会把此变量写入到主内存中。*final*：该关键字修饰的变量在构造函数中初始化完成之后（不考虑指针逃逸，变量初始化一半的问题），其他线程就可以看到这个final变量的值，并且由于变量不能修改，所以能确保可见性。


>> Notes：*保证JMM可见性，并不代表确保变量的线程安全性！！！*
>> 

### 3.4 指令重排序

重排序通常是编译器或者运行时环境为了优化程序性能而采取的对指令进行重新排序执行的一种手段。重排序分为两类：编译期重排序和运行期重排序，分别对应编译时和运行时环境。

编译期重排序主要的原因是CPU导致的。在编译期的微指令翻译阶段，许多操作同时执行，并且执行的顺序是乱序的，所以有可能出现一条指令读一个寄存器的同时，另外一条指令正在对这个寄存器进行写操作。此外，翻译之后，就是重排序缓存阶段。不同的微指令在不同的执行单元中同时执行，而且每个执行单元都全速运行。只要当前微指令所需要的数据就绪，而且有空闲的执行单元，微指令就可以立即执行，有时甚至可以跳过前面还未就绪的微指令。通过这种方式，需要长时间运行的操作不会阻塞后面的操作，流水线阻塞带来的损失被极大的减小了。

运行期JVM会对指令进行重排序以提高程序性能，当然其会通过`happens-before`原则保证顺序执行语义，也就是不会随便对代码指令进行重排序。

借用一个例子说明（来源<http://www.infoq.com/cn/articles/java-memory-model-2>）：

``` java

class ReorderExample {
int a = 0;
boolean flag = false;

public void writer() {
    a = 1;                   //1
    flag = true;             //2
}

Public void reader() {
    if (flag) {                //3
        int i =  a * a;        //4
        ……
    }
}
}

```

上述的代码会造成很多的不同结果，由于数据的可见性问题，或者就是重排序。比如重排序后执行顺序如下，则会存在问题。

<img src="/images/2014/12/reorder.png" />

### 3.5 锁细化

*锁粒度细化，是所有保证线程安全的程序方法优化的必经之路*。

这两年十分火的用于线程间通信的高性能消息组件，其虽然有很多创新的设计，但是很多优化的基本就是，锁细化，包括核心数据结构 `Ringbuffer`。[剖析Disruptor:为什么会这么快？（一）Ringbuffer的特别之处](http://ifeve.com/dissecting-disruptor-whats-so-special/)

此外，在Linux内核2.6之后采用的RCU锁机制，本质上也是锁粒度细化。[Linux 2.6内核中新的锁机制--RCU](https://www.ibm.com/developerworks/cn/linux/l-rcu/)

在Java语言中，最经典的锁细化提高多线程并发性能的案例，就是`ConcurrentHashMap`，其采用多个`segment`，每个segment对应一个锁，来分散全局锁带来的性能损失。从而，当我们put某一个entry的时候，在实现的时候，一般只需要拥有某一个segment锁就可以完成。

关于普通的`HashTable`结构和`ConcurrentHashMap`结构，借用一张图来说明：

<img src="/images/2014/12/currentHashMap.jpg" />

从结构上，可以很显而易见的看出两者的区别。所以，就锁这个层面上，concurrentHashMap就会比HashTable性能好。

### 3.6 Guava ListenableFuture接口

我们强烈地建议你在代码中多使用`ListenableFuture`来代替JDK的 Future, 因为：

- 大多数Futures 方法中需要它。

- 转到`ListenableFuture` 编程比较容易。

- Guava提供的通用公共类封装了公共的操作方方法，不需要提供Future和`ListenableFuture`的扩展方法。

*创建ListenableFuture实例*

首先需要创建`ListeningExecutorService`实例，Guava 提供了专门的方法把JDK中提供`ExecutorService`对象转换为`ListeningExecutorService`。然后通过submit方法就可以创建一个ListenableFuture实例了。

代码片段如下：

``` java

ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));
ListenableFuture explosion = service.submit(new Callable() {
  public Explosion call() {
    return pushBigRedButton();
  }
});
Futures.addCallback(explosion, new FutureCallback() {
  // we want this handler to run immediately after we push the big red button!
  public void onSuccess(Explosion explosion) {
    walkAwayFrom(explosion);
  }
  public void onFailure(Throwable thrown) {
    battleArchNemesis(); // escaped the explosion!
  }
});

```

也就是说，对于异步的方法，我可以通过监听器来根据执行结果来判断接下来的处理行为。

*ListenableFuture 链式操作*

使用ListenableFuture 最重要的理由是它可以进行一系列的复杂链式的异步操作。

一般，使用AsyncFunction来完成链式异步操作。不同的操作可以在不同的Executors中执行，单独的ListenableFuture 可以有多个操作等待。

>>
>> Tips:  AsyncFunction接口常被用于当我们想要异步的执行转换而不造成线程阻塞时，尽管Future.get()方法会在任务没有完成时造成阻塞，但是AsyncFunction接口并不被建议用来异步的执行转换，它常被用于返回Future实例。
>> 
下面给出这个链式操作完成一个简单的异步字符串转换操作：

``` java

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import com.google.common.util.concurrent.AsyncFunction;
import com.google.common.util.concurrent.FutureCallback;
import com.google.common.util.concurrent.Futures;
import com.google.common.util.concurrent.ListenableFuture;
import com.google.common.util.concurrent.ListeningExecutorService;
import com.google.common.util.concurrent.MoreExecutors;

/**
 * @author tao.ke Date: 14-12-26 Time: 下午5:28
 * @version \$Id$
 */
public class ListenerFutureChain {

    private static final ExecutorService executor = Executors.newFixedThreadPool(2);

    private static final ListeningExecutorService executorService = MoreExecutors.listeningDecorator(executor);

    public void executeChain() {

        AsyncFunction<String, String> asyncFunction = new AsyncFunction<String, String>() {

            @Override
            public ListenableFuture<String> apply(final String input) throws Exception {
                ListenableFuture<String> future = executorService.submit(new Callable<String>() {
                    @Override
                    public String call() throws Exception {
                        System.out.println("STEP1 >>>" + Thread.currentThread().getName());
                        return input + "|||step 1 ===--===||| ";
                    }
                });

                return future;

            }
        };

        AsyncFunction<String, String> asyncFunction2 = new AsyncFunction<String, String>() {

            @Override
            public ListenableFuture<String> apply(final String input) throws Exception {
                ListenableFuture<String> future = executorService.submit(new Callable<String>() {
                    @Override
                    public String call() throws Exception {
                        System.out.println("STEP2 >>>" + Thread.currentThread().getName());
                        return input + "|||step 2 ===--===---||| ";
                    }
                });

                return future;

            }
        };

        ListenableFuture startFuture = executorService.submit(new Callable() {
            @Override
            public Object call() throws Exception {
                System.out.println("BEGIN >>>" + Thread.currentThread().getName());
                return "BEGIN--->";
            }
        });

        ListenableFuture future = Futures.transform(startFuture, asyncFunction, executor);
        ListenableFuture endFuture = Futures.transform(future, asyncFunction2, executor);

        Futures.addCallback(endFuture, new FutureCallback() {
            @Override
            public void onSuccess(Object result) {
                System.out.println(result);
                System.out.println("=======OK=======");
            }

            @Override
            public void onFailure(Throwable t) {
                t.printStackTrace();

            }
        });

    }

    public static void main(String[] args) {
        System.out.println("========START=======");
        System.out.println("MAIN >>>" + Thread.currentThread().getName());
        ListenerFutureChain chain = new ListenerFutureChain();
        chain.executeChain();
        System.out.println("========END=======");

        executor.shutdown();
        // System.exit(0);
    }
}

```

输出：

    ========START=======
    MAIN >>>main
    BEGIN >>>pool-2-thread-1
    ========END=======
    STEP1 >>>pool-2-thread-2
    STEP2 >>>pool-2-thread-1
    BEGIN--->|||step 1 ===--===||| |||step 2 ===--===---||| 
    =======OK=======

从输出可以看出，代码是异步完成字符串操作的。

## <a id="CacheBuilder">CacheBuilder实现</a>

写过Cache的，或者其他一些工具类的同学知道，为了让工具类更灵活，我们需要对外提供大量的参数配置给使用者设置，虽然这带有一些好处，但是由于参数太多，使用者开发构造对象的时候过于繁杂。

上面提到过参数配置过多，可以使用Builder模式。Guava Cache也一样，它为我们提供了CacheBuilder工具类来构造不同配置的Cache实例。但是，和本文上面提到的构造器实现有点不一样，它构造器返回的是另外一个对象，因此，这意味着在实现的时候，对象构造函数需要有Builder参数提供配置属性。

### 4.1 CacheBuilder构造LocalCache实现

首先，我们先看看Cache的构造函数：

``` java

/**
 * 从builder中获取相应的配置参数。 
 */

LocalCache(CacheBuilder<? super K, ? super V> builder, @Nullable CacheLoader<? super K, V> loader) {
    concurrencyLevel = Math.min(builder.getConcurrencyLevel(), MAX_SEGMENTS);

    keyStrength = builder.getKeyStrength();
    valueStrength = builder.getValueStrength();

    keyEquivalence = builder.getKeyEquivalence();
    valueEquivalence = builder.getValueEquivalence();

    maxWeight = builder.getMaximumWeight();
    weigher = builder.getWeigher();
    expireAfterAccessNanos = builder.getExpireAfterAccessNanos();
    expireAfterWriteNanos = builder.getExpireAfterWriteNanos();
    refreshNanos = builder.getRefreshNanos();

    removalListener = builder.getRemovalListener();
    removalNotificationQueue = (removalListener == NullListener.INSTANCE) ? LocalCache
                .<RemovalNotification<K, V>> discardingQueue() : new ConcurrentLinkedQueue<RemovalNotification<K, V>>();

    ticker = builder.getTicker(recordsTime());
    entryFactory = EntryFactory.getFactory(keyStrength, usesAccessEntries(), usesWriteEntries());
    globalStatsCounter = builder.getStatsCounterSupplier().get();
    defaultLoader = loader;

    int initialCapacity = Math.min(builder.getInitialCapacity(), MAXIMUM_CAPACITY);
    if (evictsBySize() && !customWeigher()) {
        initialCapacity = Math.min(initialCapacity, (int) maxWeight);
    }

    //.......
}

```

从构造函数可以看到，Cache的所有参数配置都是从Builder对象中获取的，Builder完成了作为该模式最典型的应用，多配置参数构建对象。

在Cache中只提供一个构造函数，但是在上面代码示例中，我们演示了两种构建缓存的方式：自动加载；手动加载。那么，一般会存在一个完成两者之间的过渡`adapter`组件，接下来看看Builder在内部是如何完成创建缓存对象过程的。

OK，你猜到了。在`LocalCache`中确实提供了两种过渡类，一个是支持自动加载value的`LocalLoadingCache` 和只能在键值找不到的时候手动调用获取值方法的`LocalManualCache`。

*LocalManualCache实现*

``` java

    static class LocalManualCache<K, V> implements Cache<K, V>, Serializable {
        final LocalCache<K, V> localCache;

        LocalManualCache(CacheBuilder<? super K, ? super V> builder) {
            this(new LocalCache<K, V>(builder, null));
        }

        private LocalManualCache(LocalCache<K, V> localCache) {
            this.localCache = localCache;
        }

        // Cache methods

        @Override
        @Nullable
        public V getIfPresent(Object key) {
            return localCache.getIfPresent(key);
        }

        @Override
        public V get(K key, final Callable<? extends V> valueLoader) throws ExecutionException {
            checkNotNull(valueLoader);
            return localCache.get(key, new CacheLoader<Object, V>() {
                @Override
                public V load(Object key) throws Exception {
                    return valueLoader.call();
                }
            });
        }

        //......

        @Override
        public CacheStats stats() {
            SimpleStatsCounter aggregator = new SimpleStatsCounter();
            aggregator.incrementBy(localCache.globalStatsCounter);
            for (Segment<K, V> segment : localCache.segments) {
                aggregator.incrementBy(segment.statsCounter);
            }
            return aggregator.snapshot();
        }

        // Serialization Support

        private static final long serialVersionUID = 1;

        Object writeReplace() {
            return new ManualSerializationProxy<K, V>(localCache);
        }
    }

```

从代码实现看出实际上是一个adapter组件，并且绝大部分实现都是直接调用LocalCache的方法，或者加一些参数判断和聚合。在它核心的构造函数中，就是直接调用LocalCache构造函数，对于loader对象直接设null值。

*LocalLoadingCache实现*

`LocalLoadingCache`实现继承了``类，其主要对get相关方法做了重写。

``` java

static class LocalLoadingCache<K, V> extends LocalManualCache<K, V> implements LoadingCache<K, V> {

        LocalLoadingCache(CacheBuilder<? super K, ? super V> builder, CacheLoader<? super K, V> loader) {
            super(new LocalCache<K, V>(builder, checkNotNull(loader)));
        }

        // LoadingCache methods

        @Override
        public V get(K key) throws ExecutionException {
            return localCache.getOrLoad(key);
        }

        @Override
        public V getUnchecked(K key) {
            try {
                return get(key);
            } catch (ExecutionException e) {
                throw new UncheckedExecutionException(e.getCause());
            }
        }

        @Override
        public ImmutableMap<K, V> getAll(Iterable<? extends K> keys) throws ExecutionException {
            return localCache.getAll(keys);
        }

        @Override
        public void refresh(K key) {
            localCache.refresh(key);
        }

        @Override
        public final V apply(K key) {
            return getUnchecked(key);
        }

        // Serialization Support

        private static final long serialVersionUID = 1;

        @Override
        Object writeReplace() {
            return new LoadingSerializationProxy<K, V>(localCache);
        }
    }
}

```

提供了这些adapter类之后，builder类就可以创建`LocalCache`，如下：

``` java
    
    // 获取value可以通过key计算出
   public <K1 extends K, V1 extends V> LoadingCache<K1, V1> build(CacheLoader<? super K1, V1> loader) {
        checkWeightWithWeigher();
        return new LocalCache.LocalLoadingCache<K1, V1>(this, loader);
    }

    // 手动加载
    public <K1 extends K, V1 extends V> Cache<K1, V1> build() {
        checkWeightWithWeigher();
        checkNonLoadingCache();
        return new LocalCache.LocalManualCache<K1, V1>(this);
    }

```

### 4.2 CacheBuilder参数设置

`CacheBuilder`在为我们提供了构造一个Cache对象时，会构造各个成员对象的初始值（默认值）。了解这些默认值，对于我们分析Cache源码实现时，一些判断条件的设置原因，还是很有用的。

*初始参数值设置*

在`ConcurrentHashMap`中，我们知道有个并发水平（CONCURRENCY_LEVEL），这个参数决定了其允许多少个线程并发操作修改该数据结构。这是因为这个参数是最后map使用的segment个数，而每个segment对应一个锁，因此，对于一个map来说，并发环境下，理论上最大可以有segment个数的线程同时安全地操作修改数据结构。那么是不是segment的值可以设置很大呢？显然不是，要记住维护一个锁的成本还是挺高的，此外如果涉及全表操作，那么性能就会非常不好了。

其他一些初始参数值的设置如下所示：

``` java

    private static final int DEFAULT_INITIAL_CAPACITY = 16; // 默认的初始化Map大小
    private static final int DEFAULT_CONCURRENCY_LEVEL = 4; // 默认并发水平
    private static final int DEFAULT_EXPIRATION_NANOS = 0; // 默认超时
    private static final int DEFAULT_REFRESH_NANOS = 0; // 默认刷新时间
    static final int UNSET_INT = -1;
    boolean strictParsing = true;
    int initialCapacity = UNSET_INT;
    int concurrencyLevel = UNSET_INT;
    long maximumSize = UNSET_INT;
    long maximumWeight = UNSET_INT;
    long expireAfterWriteNanos = UNSET_INT;
    long expireAfterAccessNanos = UNSET_INT;
    long refreshNanos = UNSET_INT;


```

*初始对象引用设置*

在Cache中，我们除了超时时间，键值引用属性等设置外，还关注命中统计情况，这就需要统计对象来工作。CacheBuilder提供了初始的null 统计对象和空统计对象。

此外，还会设置到默认的引用类型等设置，代码如下：

``` java
/**
     * 默认空的缓存命中统计类
     */
    static final Supplier<? extends StatsCounter> NULL_STATS_COUNTER = Suppliers.ofInstance(new StatsCounter() {
        
        //......省略空override

        @Override
        public CacheStats snapshot() {
            return EMPTY_STATS;
        }
    });
    static final CacheStats EMPTY_STATS = new CacheStats(0, 0, 0, 0, 0, 0);// 初始状态的统计对象

    /**
     * 系统实现的简单的缓存状态统计类
     */
    static final Supplier<StatsCounter> CACHE_STATS_COUNTER = new Supplier<StatsCounter>() {
        @Override
        public StatsCounter get() {
            return new SimpleStatsCounter();//这里构造简单地统计类实现
        }
    };

    /**
     * 自定义的空RemovalListener，监听到移除通知，默认空处理。
     */
    enum NullListener implements RemovalListener<Object, Object> {
        INSTANCE;

        @Override
        public void onRemoval(RemovalNotification<Object, Object> notification) {
        }
    }

    /**
     * 默认权重类，任何对象的权重均为1
     */
    enum OneWeigher implements Weigher<Object, Object> {
        INSTANCE;

        @Override
        public int weigh(Object key, Object value) {
            return 1;
        }
    }

    static final Ticker NULL_TICKER = new Ticker() {
        @Override
        public long read() {
            return 0;
        }
    };

    /**
     * 默认的key等同判断
     * @return
     */
    Equivalence<Object> getKeyEquivalence() {
        return firstNonNull(keyEquivalence, getKeyStrength().defaultEquivalence());
    }

    /**
     * 默认value的等同判断
     * @return
     */
    Equivalence<Object> getValueEquivalence() {
        return firstNonNull(valueEquivalence, getValueStrength().defaultEquivalence());
    }

    /**
     * 默认的key引用
     * @return
     */
    Strength getKeyStrength() {
        return firstNonNull(keyStrength, Strength.STRONG);
    }

    /**
     * 默认为Strong 属性的引用
     * @return
     */
    Strength getValueStrength() {
        return firstNonNull(valueStrength, Strength.STRONG);
    }

    <K1 extends K, V1 extends V> Weigher<K1, V1> getWeigher() {
        return (Weigher<K1, V1>) Objects.firstNonNull(weigher, OneWeigher.INSTANCE);
    }

```

其中，在我们不设置缓存中键值引用的情况下，默认都是采用强引用及相对应的属性策略来初始化的。此外，在上面代码中还可以看到，统计类`SimpleStatsCounter`是一个简单的实现。里面主要是简单地缓存累加，此外由于多线程下Long类型的线程非安全性，所以也进行了一下封装，下面给出命中率的实现：

``` java
public static final class SimpleStatsCounter implements StatsCounter {

    private final LongAddable hitCount = LongAddables.create();
    private final LongAddable missCount = LongAddables.create();
    private final LongAddable loadSuccessCount = LongAddables.create();
    private final LongAddable loadExceptionCount = LongAddables.create();
    private final LongAddable totalLoadTime = LongAddables.create();
    private final LongAddable evictionCount = LongAddables.create();

    public SimpleStatsCounter() {}
    @Override
    public void recordHits(int count) {
      hitCount.add(count);
    }
    @Override
    public CacheStats snapshot() {
      return new CacheStats(
          hitCount.sum());
    }

    /**
     * Increments all counters by the values in {@code other}.
     */
    public void incrementBy(StatsCounter other) {
      CacheStats otherStats = other.snapshot();
      hitCount.add(otherStats.hitCount());
    }
}

```

因此，CacheBuilder的一些参数对象等得初始化就完成了。可以看到这些默认的初始化，有两套引用：Null对象和Empty对象，显然Null会更省空间，但我们在创建的时候将决定不使用某特性的时候，就会使用Null来创建，否则使用Empty来完成初始化工作。在分析Cache的时候，写后超时队列和读后超时队列也存在两个版本。


## <a id="LocalCache">LocalCache实现</a>

在设计实现上，`LocalCache`的并发策略和`concurrentHashMap`的并发策略是一致的，也是根据分段锁来提高并发能力,分段锁可以很好的保证 并发读写的效率。因此，该map支持非阻塞读和不同段之间并发写。

如果最大的大小指定了，那么基于段来执行操作是最好的。使用页面替换算法来决定当map大小超过指定值时，哪些entries需要被驱赶出去。页面替换算法的数据结构保证Map临时一致性：对一个segment写排序是一致的；但是对map进行更新和读不能直接立刻 反应在数据结构上。 虽然这些数据结构被lock锁保护，但是其结构决定了批量操作可以避免锁竞争出现。在线程之间传播的批量操作导致分摊成本比不强制大小限制的操作要稍微高一点。

此外，`LoacalCache`使用LRU页面替换算法，是因为该算法简单，并且有很高的命中率，以及O(1)的时间复杂度。需要说明的是， LRU算法是基于页面而不是全局实现的，所以可能在命中率上不如全局LRU算法，但是应该基本相似。

最后，要说明一点，在代码实现上，页面其实就是一个段segment。之所以说page页，是因为在计算机专业课程上，CPU，操作系统，算法上，基本上都介绍过分页导致优化效果的提升。

### 5.1 总体数据结构

`LocalCache`的数据结构和`ConcurrentHashMap`一样，都是采用分segment来细化管理HashMap中的节点Entry。借用`ConcurrentHashMap`的数据结构图来说明Cache的实现：

<img src="/images/2014/12/segement.jpg" height="300" width="600" />

从图中可以直观看到cache是以segment粒度来控制并发get和put等操作的，接下来首先看我们的`LocalCache`是如何构造这些segment段的，继续上面初始化localCache构造函数的代码：

``` java

    // 找到大于并发水平的最小2的次方的值，作为segment数量
        int segmentShift = 0;
        int segmentCount = 1;
        while (segmentCount < concurrencyLevel && (!evictsBySize() || segmentCount * 20 <= maxWeight)) {
            ++segmentShift;
            segmentCount <<= 1;
        }
        this.segmentShift = 32 - segmentShift;//位 偏移数
        segmentMask = segmentCount - 1;//mask码

        this.segments = newSegmentArray(segmentCount);// 构造数据数组，如上图所示
        //获取每个segment初始化容量，并且保证大于等于map初始容量
        int segmentCapacity = initialCapacity / segmentCount;
        if (segmentCapacity * segmentCount < initialCapacity) {
            ++segmentCapacity;
        }

        //段Size 必须为2的次数，并且刚刚大于段初始容量
        int segmentSize = 1;
        while (segmentSize < segmentCapacity) {
            segmentSize <<= 1;
        }

        // 权重设置，确保权重和==map权重
        if (evictsBySize()) {
            // Ensure sum of segment max weights = overall max weights
            long maxSegmentWeight = maxWeight / segmentCount + 1;
            long remainder = maxWeight % segmentCount;
            for (int i = 0; i < this.segments.length; ++i) {
                if (i == remainder) {
                    maxSegmentWeight--;
                }
                //构造每个段结构
                this.segments[i] = createSegment(segmentSize, maxSegmentWeight, builder.getStatsCounterSupplier().get());
            }
        } else {
            for (int i = 0; i < this.segments.length; ++i) {
                //构造每个段结构
                this.segments[i] = createSegment(segmentSize, UNSET_INT, builder.getStatsCounterSupplier().get());
            }
        }

```

>> Notes：基本上都是基于2的次数来设置大小的，显然基于移位操作比普通计算操作速度要快。此外，对于最大权重分配到段权重的设计上，很特殊。为什么呢？为了保证两者能够相等（maxWeight==sumAll(maxSegmentWeight)）,对于remainder前面的segment maxSegmentWeight的值比remainder后面的权重值大1，这样保证最后值相等。
>> 

*map get 方法*

``` java

    @Override
    @Nullable
    public V get(@Nullable Object key) {
        if (key == null) {
            return null;
        }
        int hash = hash(key);
        return segmentFor(hash).get(key, hash);
    }

```

>> Notes：代码很简单，首先check key是否为null，然后计算hash值，定位到对应的segment，执行segment实例拥有的get方法获取对应的value值
>> 

*map put 方法*

``` java

@Override
    public V put(K key, V value) {
        checkNotNull(key);
        checkNotNull(value);
        int hash = hash(key);
        return segmentFor(hash).put(key, hash, value, false);
    }

```

>> Notes：和get方法一样，也是先check值，然后计算key的hash值，然后定位到对应的segment段，执行段实例的put方法，将键值存入map中。
>> 

*map isEmpty 方法*

``` java

    @Override
    public boolean isEmpty() {
        long sum = 0L;
        Segment<K, V>[] segments = this.segments;
        for (int i = 0; i < segments.length; ++i) {
            if (segments[i].count != 0) {
                return false;
            }
            sum += segments[i].modCount;
        }

        if (sum != 0L) { // recheck unless no modifications
            for (int i = 0; i < segments.length; ++i) {
                if (segments[i].count != 0) {
                    return false;
                }
                sum -= segments[i].modCount;
            }
            if (sum != 0L) {
                return false;
            }
        }
        return true;
    }

```

>> Notes：判断Cache是否为空，就是分别判断每个段segment是否都为空，但是由于整体是在并发环境下进行的，也就是说存在对一个segment并发的增加和移除元素的时候，而我们此时正在check其他segment段。
>> 
>> 上面这种情况，决定了我们不能够获得任何一个时间点真实段状态的情况。因此，上面的代码引入了sum变量来计算段modCount变更情况。modCount表示改变segment大小size的更新次数，这个在批量读取方法期间保证它们可以看到一致性的快照。`需要注意，这里先获取count，该值是volatile，因此modCount通常都可以在不需要一致性控制下，获得当前segment最新的值.`
>> 
>> 在判断如果在第一次check的时候，发现segment发生了数据结构级别变更，则会进行recheck，就是在每个modCount下，段仍然是空的，则判断该map为空。如果发现这期间数据结构发生变化，则返回非空判断。
>> 

*map 其他方法*

在Cache数据结构中，还有很多方法，和上面列出来的方法一样，其底层核心实现都是依赖segment类实例中实现的对应方法。

此外，在总的数据结构中，还提供了一些根据builder配制制定相应地缓存策略方法。比如：

* expiresAfterAccess：是否执行访问后超时过期策略；
* expiresAfterWrite：是否执行写后超时过期策略；
* usesAccessQueue：根据上面的配置决定是否需要new一个访问队列；
* usesWriteQueue：根据上面的配置决定是否需要new一个写队列；
* usesKeyReferences/usesValueReferences：是否需要使用特别的引用策略(非Strong引用).
* 等等......

### 5.2 引用数据结构

在介绍Segment数据结构之前，先讲讲Cache中引用的设计。

关于Reference引用的一些说明，在博文的上面已经介绍了，这里就不赘述。在Guava Cache 中，主要使用三种引用类型，分别是：`STRONG引用`，`SOFT引用` ，`WEAK引用`。和Map不同，在Cache中，使用`ReferenceEntry`来封装键值对，并且对于值来说，还额外实现了`ValueReference`引用对象来封装对应Value对象。

*ReferenceEntry节点结构*

为了支持各种不同类型的引用，以及不同过期策略，这里构造了一个ReferenceEntry节点结构。通过下面的节点数据结构，可以清晰的看到缓存大致操作流程。

``` java
    /**
     * 引用map中一个entry节点。
     *
     * 在map中得entries节点有下面几种状态：
     * valid：-live：设置了有效的key/value;-loading：加载正在处理中....
     * invalid：-expired：时间过期(但是key/value可能仍然设置了)；Collected：key/value部分被垃圾收集了，但是还没有被清除；
     * -unset：标记为unset，表示等待清除或者重新使用。
     *
     */
    interface ReferenceEntry<K, V> {
        /**
         * 从entry中返回value引用
         */
        ValueReference<K, V> getValueReference();

        /**
         * 为entry设置value引用
         */
        void setValueReference(ValueReference<K, V> valueReference);

        /**
         * 返回链中下一个entry（解决hash碰撞存在链表）
         */
        @Nullable
        ReferenceEntry<K, V> getNext();

        /**
         * 返回entry的hash
         */
        int getHash();

        /**
         * 返回entry的key
         */
        @Nullable
        K getKey();

        /*
         * Used by entries that use access order. Access entries are maintained in a doubly-linked list. New entries are
         * added at the tail of the list at write time; stale entries are expired from the head of the list.
         */

        /**
         * 返回该entry最近一次被访问的时间ns
         */
        long getAccessTime();

        /**
         * 设置entry访问时间ns.
         */
        void setAccessTime(long time);

        /**
         * 返回访问队列中下一个entry
         */
        ReferenceEntry<K, V> getNextInAccessQueue();

        /**
         * Sets the next entry in the access queue.
         */
        void setNextInAccessQueue(ReferenceEntry<K, V> next);

        /**
         * Returns the previous entry in the access queue.
         */
        ReferenceEntry<K, V> getPreviousInAccessQueue();

        /**
         * Sets the previous entry in the access queue.
         */
        void setPreviousInAccessQueue(ReferenceEntry<K, V> previous);

        // ...... 省略write队列相关方法，和access一样
    }

```

>> Notes：从上面代码可以看到除了和Map一样，有key、value、hash和next四个属性之外，还有访问和写更新两个双向链表队列，以及entry的最近访问时间和最近更新时间。显然，多出来的属性就是为了支持缓存必须要有的过期机制。
>> 
>> 此外，从上面的代码可以看出*cache支持的LRU机制实际上是建立在segment上的，也就是基于页的替换机制。*
>> 
>> 关于访问队列数据结构，其实质就是一个双向的链表。当节点被访问的时候，这个节点将会移除，然后把这个节点添加到链表的结尾。关于具体实现，将在segment中介绍。
>> 
>> 创建不同类型的ReferenceEntry由其枚举工厂类EntryFactory来实现，它根据key的Strength类型、是否使用accessQueue、是否使用writeQueue来决定不同的EntryFactry实例，并通过它创建相应的ReferenceEntry实例

*ValueReference结构*

同样为了支持Cache中各个不同类型的引用，其对Value类型进行再封装，支持引用。看看其内部数据属性：

``` java

    /**
     * A reference to a value.
     */
    interface ValueReference<K, V> {
        /**
         * Returns the value. Does not block or throw exceptions.
         */
        @Nullable
        V get();

        /**
         * Waits for a value that may still be loading. Unlike get(), this method can block (in the case of
         * FutureValueReference).
         * 
         * @throws ExecutionException if the loading thread throws an exception
         * @throws ExecutionError if the loading thread throws an error
         */
        V waitForValue() throws ExecutionException;

        /**
         * Returns the weight of this entry. This is assumed to be static between calls to setValue.
         */
        int getWeight();

        /**
         * Returns the entry associated with this value reference, or {@code null} if this value reference is
         * independent of any entry.
         */
        @Nullable
        ReferenceEntry<K, V> getEntry();

        /**
         * 为一个指定的entry创建一个该引用的副本
         * <p>
         * {@code value} may be null only for a loading reference.
         */
        ValueReference<K, V> copyFor(ReferenceQueue<V> queue, @Nullable V value, ReferenceEntry<K, V> entry);

        /**
         * 告知一个新的值正在加载中。这个只会关联到加载值引用。
         */
        void notifyNewValue(@Nullable V newValue);

        /**
         * 当一个新的value正在被加载的时候，返回true。不管是否已经有存在的值。这里加锁方法返回的值对于给定的ValueReference实例来说是常量。
         * 
         */
        boolean isLoading();

        /**
         * 返回true，如果该reference包含一个活跃的值,意味着在cache里仍然有一个值存在。活跃的值包含：cache查找返回的，等待被移除的要被驱赶的值； 非激活的包含：正在加载的值，
         */
        boolean isActive();
    }

```

>> Notes：value引用接口对象中包含了不同状态的标记，以及一些加载方法和获取具体value值对象。
>> 
>> 为了减少不必须的load加载，在value引用中增加了loading标识和wait方法等待加载获取值。这样，就可以等待上一个调用loader方法获取值，而不是重复去调用loader方法加重系统负担，而且可以更快的获取对应的值。
>> 
>> 此外，介绍下`ReferenceQueue`引用队列，这个队列是JDK提供的，在检测到适当的可到达性更改后，垃圾回收器将已注册的引用对象添加到该队列中。因为Cache使用了各种引用，而通过ReferenceQueue这个“监听器”就可以优雅的实现自动删除那些引用不可达的key了，是不是很吊，哈哈。
>> 
>> 在Cache分别实现了基于Strong,Soft，Weak三种形式的ValueReference实现。
>> 
>> 这里ValueReference之所以要有对ReferenceEntry的引用是因为在Value因为WeakReference、SoftReference被回收时，需要使用其key将对应的项从Segment段中移除；
>> copyFor()函数的存在是因为在expand(rehash)重新创建节点时，对WeakReference、SoftReference需要重新创建实例（C++中的深度复制思想，就是为了保持对象状态不会相互影响），而对强引用来说，直接使用原来的值即可，这里很好的展示了对彼变化的封装思想；
>> notifiyNewValue只用于LoadingValueReference，它的存在是为了对LoadingValueReference来说能更加及时的得到CacheLoader加载的值。

### 5.3 Segment 数据结构

`Segment`数据结构，是ConcurrentHashMap的核心实现，也是该结构保证了其算法的高效性。在`Guava Cache` 中也一样，`segment`数据结构保证了缓存在线程安全的前提下可以高效地更新，插入，获取对应value。

实际上，segment就是一个特殊版本的hash table实现。其内部也是对应一个锁，不同的是，对于get和put操作做了一些优化处理。因此，在代码实现的时候，为了快速开发和利用已有锁特性，直接`extends ReentrantLock`。

在segment中，其主要的类属性就是一个`LoacalCache`类型的map变量。关于segment实现说明，如下：

``` java

        /**
         * segments 维护一个entry列表的table，确保一致性状态。所以可以不加锁去读。节点的next field是不可修改的final，因为所有list的增加操作
         * 是执行在每个容器的头部。因此，这样子很容易去检查变化，也可以快速遍历。此外，当节点被改变的时候，新的节点将被创建然后替换它们。 由于容器的list一般都比较短（平均长度小于2），所以对于hash
         * tables来说，可以工作的很好。虽然说读操作因此可以不需要锁进行，但是是依赖
         * 使用volatile确保其他线程完成写操作。对于绝大多数目的而言，count变量，跟踪元素的数量，其作为一个volatile变量确保可见性（其内部原理可以参考其他相关博文）。
         * 这样一下子变得方便的很多，因为这个变量在很多读操作的时候都会被获取：所有非同步的（unsynchronized）读操作必须首先读取这个count值，并且如果count为0则不会 查找table
         * 的entries元素；所有的同步（synchronized）操作必须在结构性的改变任务bin容器之后，才会写操作这个count值。
         * 这些操作必须在并发读操作看到不一致的数据的时候，不采取任务动作。在map中读操作性质可以更容易实现这个限制。例如：没有操作可以显示出 当table
         * 增长了，但是threshold值没有更新，所以考虑读的时候不要求原子性。作为一个原则，所有危险的volatile读和写count变量都必须在代码中标记。
         */

        final LocalCache<K, V> map;

        /**
         * 该segment区域内所有存活的元素个数
         */
        volatile int count;

        /**
         * 改变table大小size的更新次数。这个在批量读取方法期间保证它们可以看到一致性的快照：
         * 如果modCount在我们遍历段加载大小或者核对containsValue期间被改变了，然后我们会看到一个不一致的状态视图，以至于必须去重试。
         * count+modCount 保证内存一致性
         * 
         * 感觉这里有点像是版本控制，比如数据库里的version字段来控制数据一致性
         */
        int modCount;

        /**
         * 每个段表，使用乐观锁的Array来保存entry The per-segment table.
         */
        volatile AtomicReferenceArray<ReferenceEntry<K, V>> table; // 这里和concurrentHashMap不一致，原因是这边元素是引用，直接使用不会线程安全
        /**
         * A queue of elements currently in the map, ordered by write time. Elements are added to the tail of the queue
         * on write.
         */
        @GuardedBy("Segment.this")
        final Queue<ReferenceEntry<K, V>> writeQueue;

        /**
         * A queue of elements currently in the map, ordered by access time. Elements are added to the tail of the queue
         * on access (note that writes count as accesses).
         */
        @GuardedBy("Segment.this")
        final Queue<ReferenceEntry<K, V>> accessQueue;
```

>> Notes：
>> 
>> 在segment实现中，很多地方使用count变量和modCount变量来保持线程安全，从而省掉lock开销。
>> 
>> 在本文上面的图中说明了每个segment就是一个节点table，和jdk实现不一致，这里为了GC，内部维护的是一个`AtomicReferenceArray<ReferenceEntry<K, V>>`类型的列表，可以保证安全性。
>> 
>> 最后，`LocalCache`作为一个缓存，其必须具有访问和写超时特性，因为其内部维护了访问队列和写队列，队列中的元素按照访问或者写时间排序，新的元素会被添加到队列尾部。如果，在队列中已经存在了该元素，则会先delete掉，然后再尾部add该节点，新的时间。这也就是为什么，对于`LocalCache`而言，其LRU是针对segment的，而不是全Cache范围的。
>> 

在本文的 5.2节中知道，cache会根据初始化实例时配置来创建多个segment（`createSegment`），然后该方法最终调用segment类的构造函数创建一个段。对于参数set，就不展示，看看构造方法中其主要操作：

``` java
// 构造函数
    Segment(LocalCache<K, V> map, int initialCapacity, long maxSegmentWeight, StatsCounter statsCounter) {
        initTable(newEntryArray(initialCapacity));
    }

    AtomicReferenceArray<ReferenceEntry<K, V>> newEntryArray(int size) {
        return new AtomicReferenceArray<ReferenceEntry<K, V>>(size);
    }

    void initTable(AtomicReferenceArray<ReferenceEntry<K, V>> newTable) {
        this.threshold = newTable.length() * 3 / 4; // 0.75
        if (!map.customWeigher() && this.threshold == maxSegmentWeight) {
            // prevent spurious expansion before eviction
            this.threshold++;
        }
        this.table = newTable;
    }
```

OK，这里我们已经构造好了整个localCache对象了，包括其内部每个segment中对应的节点表。这些节点table，决定了最后所有核心操作的具体实现和操作结果。

接下来，需要看看最核心的几个方法。

>> Tips：本文把这几个方法单独作为几节来说明，这也表示这几个方法的重要性。

### 5.4 GET方法实现

首先，如果我们从一个列表中查找对象，怎么做？

    1. 列表元素个数是否为0；
    
    2. 如果非0，则依次查询列表中元素是否是我们的对象。

然后，如果是考虑超时策略的缓存呢？

    1. 缓存列表元素个数是否为0；
    
    2. 如果非0，则依次查询列表中元素是否是我们的对象；
    
    3. 查看队列中该对象是否已过期，如果过期则考虑其他方式获取。
    
    4. 此外，为了线程安全，必须在获取的时候，锁住表不让更新缓存操作。

接下来是，`LocalCache`的缓存应该怎么做？

    1. 缓存中元素个数volatile count是否为0；
    
    2. 如果非0，则获取我们需要的对象引用【getEntry(key, hash)】；
    
    3. 如果对象引用不为null,则获取对应的value值；
    
    4. 如果value已经过期或者无效，则判断是否在Loading【scheduleRefresh(e, key, hash, value, now, loader)】,否则，判断是否到了refresh时间;
    
    5. 如果设置refresh，则异步刷新查询value，然后等待返回最新value【scheduleRefresh(e, key, hash, value, now, loader)】;
    
    6. ok，这里如果value还没有拿到，则查询loader方法获取对应的值(存在加锁)【lockedGetOrLoad(key, hash, loader)】。

上面就是get方法的主要流程，对于其中一些核心的方法进行分析解析：

*getEntry方法*

``` java

        @Nullable
        ReferenceEntry<K, V> getEntry(Object key, int hash) {
            // hash链表
            for (ReferenceEntry<K, V> e = getFirst(hash); e != null; e = e.getNext()) {
                if (e.getHash() != hash) {
                    continue;
                }

                // hash值相同的，接下来找key值也相同的ReferenceEntry
                K entryKey = e.getKey();
                if (entryKey == null) {
                    tryDrainReferenceQueues();//线程安全的清除搜集到的entries，使用lock机制。
                    continue;
                }

                if (map.keyEquivalence.equivalent(key, entryKey)) {
                    return e;
                }
            }

            return null;
        }

        /**
         * AtomicReferenceArray 可以确保原子的更新引用的元素。
         * 
         * 为给定的hash值返回第一个entry节点.
         */
        ReferenceEntry<K, V> getFirst(int hash) {

            // 复制到线程安全的数组中，形成一个快照，确保读的时候，数据一致性。只会读取这个域一次。
            // 此外，这样子可以提供读对于整个table的影响，因为全局的table并不会锁住。（猜测）
            AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
            return table.get(hash & (table.length() - 1));
        }

```

>> Notes：上面从缓存中直接获取key对应value，是完全没有加锁来完成的。

*scheduleRefresh方法*

如果配置refresh特性，到了配置的刷新间隔时间，而且节点也没有正在加载，则应该进行refresh操作。refresh操作比较复杂。

``` java
/**
         * 刷新和key关联的value值，除非另一个线程正在做这个。如果在内部刷新了，则返回和key关联的value，否则如果另一个线程正在
         * 刷新或者出现error则返回null
         */
        @Nullable
        V refresh(K key, int hash, CacheLoader<? super K, V> loader, boolean checkTime) {
            
            // loadingValueReference表明当前线程开始加载，获取key对于的value引用。
            final LoadingValueReference<K, V> loadingValueReference = insertLoadingValueReference(key, hash, checkTime);
            if (loadingValueReference == null) {
                return null;
            }

            // 如果说本线程启动加载，则开始异步调用，等待future返回get获取一个监听listenableFuture（见本文准备知识部分介绍），然后等待返回value值。loader相关方法随后介绍
            ListenableFuture<V> result = loadAsync(key, hash, loadingValueReference, loader);
            if (result.isDone()) {
                try {
                    return Uninterruptibles.getUninterruptibly(result);
                } catch (Throwable t) {
                    // don't let refresh exceptions propagate; error was already logged
                }
            }
            return null;
        }

        /**
         * 返回一个本线程新插入的LoadingValueReference对象，或者如果一个活跃的value引用已经被加载了，则返回null
         */
        @Nullable
        LoadingValueReference<K, V> insertLoadingValueReference(final K key, final int hash, boolean checkTime) {
            ReferenceEntry<K, V> e = null;
            // 加锁，保证只有一个线程对segment refresh操作
            lock();
            try {
                long now = map.ticker.read();
                preWriteCleanup(now);

                // 快照保证
                AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
                int index = hash & (table.length() - 1);
                ReferenceEntry<K, V> first = table.get(index);

                // 查找一个存在的entry节点，和上面的getEntry方法基本一致。
                for (e = first; e != null; e = e.getNext()) {
                    K entryKey = e.getKey();
                    if (e.getHash() == hash && entryKey != null && map.keyEquivalence.equivalent(key, entryKey)) {
                        // 如果存在我们想要的节点
                        ValueReference<K, V> valueReference = e.getValueReference();
                        if (valueReference.isLoading() || (checkTime && (now - e.getWriteTime() < map.refreshNanos))) {
                            // 如果loading正在处理，并且发现该节点引用的写时间未超期刷新周期，则返回null
                            return null;
                        }

                        // continue returning old value while loading
                        ++modCount;
                        LoadingValueReference<K, V> loadingValueReference = new LoadingValueReference<K, V>(
                                valueReference);//使用老的值引用
                        e.setValueReference(loadingValueReference);
                        return loadingValueReference;
                    }
                }

                ++modCount;
                LoadingValueReference<K, V> loadingValueReference = new LoadingValueReference<K, V>();
                e = newEntry(key, hash, first);//一个新的节点，存放的hash链头部
                e.setValueReference(loadingValueReference);
                table.set(index, e);// 插入到列表中
                return loadingValueReference;//返回新的值引用
            } finally {
                unlock();
                postWriteCleanup();
            }
        }
```

*lockedGetOrLoad方法*

如方法名所见，该方法是加锁加载key对应的值引用。

``` java

 /**
         * 这里开始从我们实现cacheLoader继承类中的load方法获取 key对应的值。
         * 
         * 加锁get或者load
         */
        V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
            ReferenceEntry<K, V> e;
            ValueReference<K, V> valueReference = null;
            LoadingValueReference<K, V> loadingValueReference = null;
            boolean createNewEntry = true;

            // 确保线程安全，使用加锁来确保加载。当然这个也是针对segment粒度来加的
            lock();
            try {
                // re-read ticker once inside the lock
                long now = map.ticker.read();
                preWriteCleanup(now);// 加锁清GC遗留引用数据和超时数据

                int newCount = this.count - 1;
                AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
                int index = hash & (table.length() - 1);// 根据hash和table长度来确定index索引
                ReferenceEntry<K, V> first = table.get(index);

                for (e = first; e != null; e = e.getNext()) {
                    K entryKey = e.getKey();
                    if (e.getHash() == hash && entryKey != null && map.keyEquivalence.equivalent(key, entryKey)) {
                        valueReference = e.getValueReference();
                        if (valueReference.isLoading()) {
                            createNewEntry = false;// 如果正在加载，则返回false，表示不需要新建entry
                        } else {
                            // 对value进行判断处理，
                            V value = valueReference.get();
                            if (value == null) {
                                // 相关通知操作，GC原因回收了
                                enqueueNotification(entryKey, hash, valueReference, RemovalCause.COLLECTED);
                            } else if (map.isExpired(e, now)) {
                                // This is a duplicate check, as preWriteCleanup already purged expired
                                // entries, but let's accomodate an incorrect expiration queue.
                                enqueueNotification(entryKey, hash, valueReference, RemovalCause.EXPIRED);
                            } else {
                                // cache存在value，命中缓存
                                recordLockedRead(e, now);
                                statsCounter.recordHits(1);
                                // we were concurrent with loading; don't consider refresh
                                return value;
                            }

                            // 最后写count，保证前面的变量操作，对内存立刻可见
                            writeQueue.remove(e);
                            accessQueue.remove(e);
                            this.count = newCount; // write-volatile
                        }
                        break;
                    }
                }

                // 处理需要新增entry，从load方法获取的逻辑
                if (createNewEntry) {
                    loadingValueReference = new LoadingValueReference<K, V>();

                    if (e == null) {
                        e = newEntry(key, hash, first);// segment神马都没有的时候，新建一个
                        e.setValueReference(loadingValueReference);
                        table.set(index, e);
                    } else {
                        e.setValueReference(loadingValueReference);
                    }
                }
            } finally {
                unlock();
                postWriteCleanup();
            }

            // ok,上面加锁部分建完了新的entry，设置完valueReference
            if (createNewEntry) {
                try {

                    // 在entry同步，但检测到递归load则会快速失败。当entry被copy时候可能绕行，但是绝大部分时间会快速失败
                    synchronized (e) {
                        return loadSync(key, hash, loadingValueReference, loader);
                    }
                } finally {
                    statsCounter.recordMisses(1);// 处理命中率
                }
            } else {
                // 如果正在加载，则等待加载完成
                // The entry already exists. Wait for loading.
                return waitForLoadingValue(e, key, valueReference);
            }
        }

```

>> Tips：不管是lockget还是refresh，最后都会调用不同的load方法，只不过refresh使用`loadingFuture.addListener`方式来异步加载值而已。其最后都会调用`getAndRecordStats`方法。
>> 
>> 

``` java
 V getAndRecordStats(K key, int hash, LoadingValueReference<K, V> loadingValueReference,
                ListenableFuture<V> newValue) throws ExecutionException {
            V value = null;
            try {
                value = getUninterruptibly(newValue);// 非中断方式调用future.get方法获取值
                if (value == null) {
                    throw new InvalidCacheLoadException("CacheLoader returned null for key " + key + ".");
                }
                statsCounter.recordLoadSuccess(loadingValueReference.elapsedNanos());
                //线程安全地把key和value存放到cache中。
                storeLoadedValue(key, hash, loadingValueReference, value);
                return value;
            } finally {
                if (value == null) {
                    statsCounter.recordLoadException(loadingValueReference.elapsedNanos());
                    removeLoadingValue(key, hash, loadingValueReference);
                }
            }
        }
```

>> Notes：上面代码会调用storeLoadedValue方法,这个方法和后面的put方法实现很相似.如下:
>> 

``` java
        /**
         * 首先，这里是线程安全的。把key和value存放到cache中。
         */
        boolean storeLoadedValue(K key, int hash, LoadingValueReference<K, V> oldValueReference, V newValue) {
            lock();
            try {
                long now = map.ticker.read();
                preWriteCleanup(now);// clean工作

                int newCount = this.count + 1;
                if (newCount > this.threshold) { // 保证大小够用ensure capacity
                    expand();
                    newCount = this.count + 1;// 扩容之后，count可能会变化
                }

                AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
                int index = hash & (table.length() - 1);
                ReferenceEntry<K, V> first = table.get(index);

                // 如果当前segment中已经存在了该key元素
                for (ReferenceEntry<K, V> e = first; e != null; e = e.getNext()) {
                    K entryKey = e.getKey();
                    // 找到hash链中对应的相等节点,则add操作;但是如果value是活跃的,则先移除
                    if (e.getHash() == hash && entryKey != null && map.keyEquivalence.equivalent(key, entryKey)) {
                        ValueReference<K, V> valueReference = e.getValueReference();
                        V entryValue = valueReference.get();

                        // 实现就有value引用的情况下
                        if (oldValueReference == valueReference || (entryValue == null && valueReference != UNSET)) {
                            ++modCount;
                            // 首先如果value引用活跃,则让放入等待GC回收队列中,等待被回收.
                            if (oldValueReference.isActive()) {
                                RemovalCause cause = (entryValue == null) ? RemovalCause.COLLECTED
                                        : RemovalCause.REPLACED;
                                // 如果监听类配置了,则这里会触发监听方法响应
                                enqueueNotification(key, hash, oldValueReference, cause);
                                newCount--;
                            }
                            // 更新新的值引用,如上所述,如果有老值,不直接删除,让GC回收.
                            // 这里会操作访问队列和写队列,还有其他对外的抽象监听方法调用等
                            setValue(e, key, newValue, now);
                            this.count = newCount; // write-volatile,确保modCount能及时写入共享内存中
                            evictEntries();// 移除操作,put方法也调用.
                            return true;
                        }

                        //那如果value引用已经没有了呢?!也就是value引用已经被回收了,而不只是value值为null
                        // 新建一个value引用就好了呀?为什么返回false呢???
                        valueReference = new WeightedStrongValueReference<K, V>(newValue, 0);
                        enqueueNotification(key, hash, valueReference, RemovalCause.REPLACED);
                        return false;
                    }
                }

                // 如果事先segment数组中没有该key,则新建一个节点entry
                ++modCount;
                ReferenceEntry<K, V> newEntry = newEntry(key, hash, first);
                setValue(newEntry, key, newValue, now);
                table.set(index, newEntry);
                this.count = newCount; // write-volatile
                evictEntries();
                return true;
            } finally {
                unlock();
                postWriteCleanup();
            }
        }
```

### 5.5 PUT方法实现

和 get方法相比，put方法则相对而言，简单了很多，直接上`Guava LocalCache`的实现。

    1. 加锁，对于更新操作，是需要加锁来确保线程安全的。
    
    2. put操作，所以需要确保当前的空间，足够存放；否则需要扩容【expand】
    
    3. 查看当前cache中是否已经存在该对象对应的key；
    3.1. 如果存在，则更新相关的value，并且更新相关的时间参数；
    3.2. 如果不存在，则创建一个新entry，然后放入table数组中。

    4. 在上面的一些步骤中，还涉及到移除一些参数。【evictEntries】.当我们put操作的时候, 对于map的segment容量就可能会有变更,这样子就需要调用evict方法,决定是否需要采取移除多余的元素.

*expand方法*

扩展table的大小空间。

``` java

        /**
         * 如果需要并且没到限制大小，则扩展表table。
         *
         */
        @GuardedBy("Segment.this")
        void expand() {
            AtomicReferenceArray<ReferenceEntry<K, V>> oldTable = table;// 原子引用
            int oldCapacity = oldTable.length();
            if (oldCapacity >= MAXIMUM_CAPACITY) {// 无法扩容
                return;
            }

            /**
             * 把每个list的nodes分类到新的map中。 因为我们这里使用的是2的指数次扩容，所以在每个bin的元素，要么还是同样的index中待着，
             * 要么移到2的指数个偏移。我们排除了不必要的节点创建（可以优化场景：因为老的节点们下一个fields不会被改变，所以老的节点可以被重复使用）。
             *
             * 以默认域设置来统计，当我们双倍扩展table时，仅仅只有六分之一的节点需要clone。这些节点将会被GC掉， 在他们不在被任务reader线程（这些线程可能正遍历在table的中间部分）引用的时候。
             *
             */

            int newCount = count;
            AtomicReferenceArray<ReferenceEntry<K, V>> newTable = newEntryArray(oldCapacity << 1);
            threshold = newTable.length() * 3 / 4;
            int newMask = newTable.length() - 1;
            for (int oldIndex = 0; oldIndex < oldCapacity; ++oldIndex) {
                // 我们必须保证任务对老Map的正在进行的读操作可以处理，所以我们不能每个bin设置null
                ReferenceEntry<K, V> head = oldTable.get(oldIndex);

                if (head != null) {
                    ReferenceEntry<K, V> next = head.getNext();
                    int headIndex = head.getHash() & newMask;

                    // hash链只有一个节点的情况
                    if (next == null) {
                        newTable.set(headIndex, head);
                    } else {
                        
                        // 这里可以重复使用链表，如注释所述，2的倍数扩展，很多引用hash值还是一样，所以把链表头直接set过去就可以了
                        ReferenceEntry<K, V> tail = head;
                        int tailIndex = headIndex;
                        for (ReferenceEntry<K, V> e = next; e != null; e = e.getNext()) {
                            int newIndex = e.getHash() & newMask;
                            if (newIndex != tailIndex) {
                                // 如果hash更变了，则引用改变。将需要复制前面的节点
                                tailIndex = newIndex;
                                tail = e;
                            }
                        }
                        newTable.set(tailIndex, tail);

                        // Clone nodes leading up to the tail.
                        for (ReferenceEntry<K, V> e = head; e != tail; e = e.getNext()) {
                            int newIndex = e.getHash() & newMask;
                            ReferenceEntry<K, V> newNext = newTable.get(newIndex);
                            ReferenceEntry<K, V> newFirst = copyEntry(e, newNext);
                            if (newFirst != null) {
                                // 设置新位置的节点链表
                                newTable.set(newIndex, newFirst);
                            } else {
                                // 移除节点相关操作
                                removeCollectedEntry(e);
                                newCount--;
                            }
                        }
                    }
                }
            }
            table = newTable;
            this.count = newCount;
        }

```

*evictEntries方法*

``` java
        /**
         * 如果segment满了，则执行evict操作。这个调用仅仅发生在增加一个新的entry并且增加了count的时候。
         */
        @GuardedBy("Segment.this")
        void evictEntries() {
            if (!map.evictsBySize()) { // 如果没有设置cache的权重，则不执行evict操作
                return;
            }

            //清除recencyQueue队列，按照指定的相关顺序来读取entries并且更新驱赶的元数据。
            // 把他们加到相关的evict列表 （这表明他们可以被移除出map中，由于被加到了recencyQueue队列中。）
            drainRecencyQueue();
            while (totalWeight > maxSegmentWeight) { // 当总的权重大于设置的最大段权重，才会执行remove操作
                ReferenceEntry<K, V> e = getNextEvictable();
                if (!removeEntry(e, e.getHash(), RemovalCause.SIZE)) {
                    throw new AssertionError();
                }
            }
        }
```

## <a id="CacheOverWrite">Guava Cache扩展</a>

Guava的CacheBuilder是一个final对象，不允许继承。但是，其提供看专门用来扩展的接口供重写部分方法。分别为`ForwardingCache`和`ForwardingLoadingCache`，对应着Cache类和LoadingCache类。

两个扩展类，采用委托模式和/或装饰模式，提供抽象实现。

委派模式（Delegate）是面向对象设计模式中常用的一种模式。这种模式的原理为类B和类A是两个互相没有任何关系的类，B具有和A一模一样的方法和属性；并且调用B中的方法，属性就是调用A中同名的方法和属性。B好像就是一个受A授权委托的中介。第三方的代码不需要知道A的存在，也不需要和A发生直接的联系，通过B就可以直接使用A的功能，这样既能够使用到A的各种公能，又能够很好的将A保护起来了。

Decorator装饰模式是一种结构型模式，它主要是解决：“过度地使用了继承来扩展对象的功能”，由于继承为类型引入的静态特质，使得这种扩展方式缺乏灵活性；并且随着子类的增多（扩展功能的增多），各种子类的组合（扩展功能的组合）会导致更多子类的膨胀（多继承）。继承为类型引入的静态特质的意思是说以继承的方式使某一类型要获得功能是在编译时。所谓静态，是指在编译时；动态，是指在运行时。

GoF《设计模式》中说道：动态的给一个对象添加一些额外的职责。就增加功能而言，Decorator模式比生成子类更为灵活。

两种模式其实很相近。委派模式的最终结果就是达到装饰模式的目的。

简单地来看看ForwardingCache抽象类的实现：

``` java
/**
 * 一个缓存将所有他的方法调用转到其他cache上。子类需要重写一个或者多个方法来改变背后cache的行为。
 * 因此，在该类里面会有一个delegate的成员，负责调用具体的cache类对象方法。
 */
@Beta
public abstract class ForwardingCache<K, V> extends ForwardingObject implements Cache<K, V> {

  /** Constructor for use by subclasses. */
  protected ForwardingCache() {}

  @Override
  protected abstract Cache<K, V> delegate();

  /**
   * @since 11.0
   */
  @Override
  @Nullable
  public V getIfPresent(Object key) {
    return delegate().getIfPresent(key);
  }

  /**
   * @since 11.0
   */
  @Override
  public V get(K key, Callable<? extends V> valueLoader) throws ExecutionException {
    return delegate().get(key, valueLoader);
  }

  /**
   * @since 11.0
   */
  @Override
  public ImmutableMap<K, V> getAllPresent(Iterable<?> keys) {
    return delegate().getAllPresent(keys);
  }

  /**
   * @since 11.0
   */
  @Override
  public void put(K key, V value) {
    delegate().put(key, value);
  }

  /**
   * @since 12.0
   */
  @Override
  public void putAll(Map<? extends K,? extends V> m) {
    delegate().putAll(m);
  }

  @Override
  public void invalidate(Object key) {
    delegate().invalidate(key);
  }

  /**
   * @since 11.0
   */
  @Override
  public void invalidateAll(Iterable<?> keys) {
    delegate().invalidateAll(keys);
  }

  @Override
  public void invalidateAll() {
    delegate().invalidateAll();
  }

  @Override
  public long size() {
    return delegate().size();
  }

  @Override
  public CacheStats stats() {
    return delegate().stats();
  }

  @Override
  public ConcurrentMap<K, V> asMap() {
    return delegate().asMap();
  }

  @Override
  public void cleanUp() {
    delegate().cleanUp();
  }

  /**
   * A simplified version of {@link ForwardingCache} where subclasses can pass in an already
   * constructed {@link Cache} as the delegete.
   *
   * @since 10.0
   */
  @Beta
  public abstract static class SimpleForwardingCache<K, V> extends ForwardingCache<K, V> {
    private final Cache<K, V> delegate;

    protected SimpleForwardingCache(Cache<K, V> delegate) {
      this.delegate = Preconditions.checkNotNull(delegate);
    }

    @Override
    protected final Cache<K, V> delegate() {
      return delegate;
    }
  }
}

```

## <a id="End">Guava Cache 总结</a>

Guava Cache的实现,核心数据结构和算法都是和JDK 1.6版本的`ConcurrentHashMap`一致.因此,如果你熟悉ConcurrentHashMap实现原理,对Cache是很容易明白的.

此外,Guava 还提供了相当多的优秀的工具类给开发者快速开发业务. 在后续的博客中, 会进一步介绍.

关于Guava 的源码学习, 博主都将一些注解和思考, 放在了源码解读中. 关于源码的地址, 请参考github: <https://github.com/ketao1989/cnGuava.git>

>> 源码被裁剪过, 删掉了一些不太关注的GWT相关源码等, 所以是无法编译的. 
