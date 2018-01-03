---
layout: post
title: "聊聊 Java GC"
date: 2016-05-29 20:43:10 +0800
comments: true
categories: 
 - Java
 - GC
---

* content
{:toc}

## <a id="Preface">序</a>

GC 是每一个Java程序员不可绕过的话题。GC 是在`某些时候`对`内存`的垃圾`对象数据`进行`搜寻定位`，然后进行内存空间`回收`。根据这个定义，则学习GC相关知识，需要关注：对JVM整个内存结构中哪些区域进行垃圾回收；在这些内存区域中的类数据或者实例数据等数据结构是什么样子的；然后想想如何在JVM内存空间中分配内存给这些实例数据；在所有已分配了的实例里，怎么找出需要回收的数据。

综上，对于JVM GC知识体系来说，就是弄清楚JVM在什么时候，对什么对象，进行什么操作来回收内存空间。

## <a id="JVM_Memory">JVM 内存结构</a>

既然是对内存进行GC操作，那么首先需要了解 JVM 的内存结构了。

在Oracle的官方文档【[Java Garbage Collection Basics](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)】中，给出了JVM的整体架构图，如下所示：

![JVM架构图](/images/2016/05/jvm_conponent.png)

也就是说，对一个JVM来说，其主要由 `ClassLoader` 和 `Runtime Data 区域` ,`执行引擎`以及`本地方法`四大部分组成。(关键的性能优化集中在`Heap`,`JIT编译`和`GC`三大块)。

<!-- more -->

### ClassLoader

从架构图中，可以看出，`ClassLoader`是用来加载java class文件的，也就是说，在java的世界里，所有的`*.class`都必须通过`ClassLoader`加载到JVM虚拟机中，才能被执行。

对于每一个JVM来说，都会有一个默认的`ClassLoader`，用来加载二进制文件到内存中。当然，既然是默认，必然就存在自定义的`ClassLoader`了。

但是，在JVM中，存在自定义的`ClassLoader`，就需要考虑如果不让自定义的实现类加载逻辑，导致java基础类库的对象发生不可预知的安全问题，所以JVM 的`ClassLoader` 对类加载的顺序逻辑进行的限制。这就是所谓的`双亲委派模型`。

![ClassLoader架构](/images/2016/05/classloader.png)

上图（来自【[深入分析Java ClassLoader原理](http://blog.csdn.net/xyang81/article/details/7292380)】）展示了JVM ClassLoader 的三个模型的加载器以及应用可以自己实现的自定义加载器。从图上可以看出除了`Bootstrap ClassLoader`不存在父类之后，其他ClassLoader都有父类加载器。

`双亲委派模型`就是说，ClassLoader 在加载类的时候，其首先去父类里查找是否其已经加载了该类，一层层递归查找，没有的话，就依次自己尝试加载该类，如果没找到，则会抛出`ClassNotFoundException`，当最外层的子类依然没加载成功，则最终的依次将会抛出到应用中。

简单来说，就是首先向上(父类方向)查找类是否加载，然后如果没有已经被加载，则向下(子类方向)尝试去加载类，直到加载成功，或者抛出异常。

简单地源码如下：

``` java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);//向上查找
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);//自己加载

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```

### JVM 运行时内存空间

![JVM架构-运行时数据区](/images/2016/05/jvm_mm.png)

上图【[JVM 的 工作原理，层次结构 以及 GC工作原理](https://segmentfault.com/a/1190000002579346)】中可以看出线程和内存区域之间的关系。对于具体的内存空间结构，如下图所示：

![内存结构](/images/2016/05/jvm_mm2.png)

上图包含除`ClassLoader`之外，JVM架构的其他三个部分。执行引擎中JIT是对java程序执行时的优化操作，GC引擎则是本文的重点，后面会详细介绍。本地方法则是使用C/C++编写的底层代码程序，这里不作介绍。

下面重点来说说，绿色区域的运行时数据区。

#### 线程私有

+ 程序计数器：和系统中PC一样，就是执行下一条需要执行的指令；每一个线程都对应一个程序计数器；但是执行本地方法由于对JVM透明，所以对应程序计数器为空。
+ 虚拟机栈：如上图所示，一个虚拟机栈包含局部变量，操作数栈，返回地址等信息。对于一个线程中的每一个方法，都有对应的虚拟机栈。虚拟机栈中定义了两种异常，如果线程调用的栈深度大于虚拟机允许的最大深度，则抛出StatckOverFlowError（栈溢出）；不过多数Java虚拟机都允许动态扩展虚拟机栈的大小(有少部分是固定长度的)，所以线程可以一直申请栈，直到内存不足，此时，会抛出OutOfMemoryError（内存溢出）。
+ 本地方法栈：执行native方法时，所需要的栈空间。
 
>> 本地方法栈和虚拟机栈在HotSpot实现里，是合并在一起的，统称为栈。

#### 线程共享

+ 方法区：方法区是对所有类共享的，所以多线程操作需要进行同步处理。其内部主要存储加载类的相关信息（包括版本、field、方法、接口等信息）、final常量、静态变量、编译器即时编译的代码等。
>> 如上图，方法区中其中一个重要内容就是运行时常量池，但是这个在JDK7中已经移到堆中存储了。运行时常量池，包括编译后的字符常量，以及运行时的常量。因此，以前String.itern()导致的常量池爆掉抛出`java.lang.OutOfMemoryError: PermGen space`错误，现在没有了。
+ 堆：一般来说，JVM所有的对象实例都是存储在堆上的，因此其也是线程共享的。对于JVM GC来说，其主战场就是堆空间。因此，后面的GC 操作都是基于堆空间的回收策略分析的。

### 小结

JVM 的内存结构，实际上就是堆和栈，栈是线程私有的，因此上面的对象不需要同步操作，并且重点是由于栈帧是伴随着线程方法的生命周期的，所以其内部的垃圾回收不用考虑(这里说的是存储在栈上的数据，当然方法里的变量，实例数据等分配在堆上的，还是需要垃圾回收的)。

堆上面包含几乎所有交互操作数据，各个对象实例，变量等信息都是由堆来分配管理，因此，JVM的垃圾回收就是基于堆的垃圾回收。

## <a id="Class_Instace">JVM 对象实例结构</a>

垃圾回收是回收内存中标记为不再使用的对象实例，因此，需要了解对象的结构和对应的引用被访问的方式。

### 对象实例结构

在C的世界里，我们会经常计算一个数据结构所占用的内存空间；那么对于Java而言，一个对象的数据结构和内存占用是怎样的呢。

对象的实例内存布局主要包括三部分：对象头(Header)、实例数据(Instance Data)和对其数据(Padding)。

#### JVM 对象头
对于JDK默认的虚拟机HotSpot来说，其对象头主要包含两部分：
+ 所谓的`Mark Word`，用于存储对象自身的运行时数据，比如hashcode，`GC分代年龄`(这个在分代GC中很重要)以及锁状态标识和线程ID等相关信息。这部分在32bit和64bit的JVM上占用的空间是不同的，分别为4B 和 8B。如下图所示：  

32bit:  
![32bit](/images/2016/05/32_bits.png)  

64bit:  
![64bit](/images/2016/05/64_bits.png)

不管32还是64，其GC分代年龄都是4bit，因此，分代年龄最高为15。
+ 类型指针(kclass)，即对象指向其类元数据的指针，JVM通过这个指针可以确定该对象是哪个类的实例。比如，针对java对象调用 `ins instanceOf Clazz `方法，就是首先通过ins对象头中该指针来找到方法区中该类的相关信息，然后匹配Clazz以及其super链，匹配上了，则说明是其对应的实例对象。 

>> 此外，如果是数组类型的话，则在对象头中还有一块空间是用来存储数组长度，不管32还是64，这个长度占用的空间都是4B，因此，数组最大长度为`Integer.MAX_VALUE`。  

>> 一般的，对象头在32bit下，大小为4B+4B=8B，而64bit为8B+8B=16B。(开启指针压缩，则为8B+4B=12B)

#### 实例数据
顾名思义，就是对象真正存储有效信息的地方。无论数据是从父类继承下来的，还是子类中定义的，都在这里记录下来。java中基础类型的内存占用如下：


类型 | 占用字节bytes 
---|---:
boolean | 1 
byte    | 1
short   | 2
char    | 2
int     | 4
float   | 4
long    | 8
double  | 8
ref     | 4/8


>> 一般来说，一个对象实例原生数据和实例在一起，而最引用类型只是一个地址，具体的数据，在指向堆上其他对应的实例。

#### 对齐数据
在HotSpot JVM 中，内存中的数据都要求是按8B字节对其的，如果一个对象大小不是8B，则会自动填充对其到8B的倍数。

### 对象访问方式

如上文所述，一个对象的数据，可能会包含其他对象的引用，那么在使用的时候，就需要找到对应的对象数据信息。

【[HotSpot虚拟机对象探秘](http://www.infoq.com/cn/articles/jvm-hotspot)】
Java程序需要通过栈上的reference数据来操作堆上的具体对象。由于reference类型在Java虚拟机规范里面只规定了是一个指向对象的引用，并没有定义这个引用应该通过什么种方式去定位、访问到堆中的对象的具体位置，对象访问方式也是取决于虚拟机实现而定的。主流的访问方式有使用句柄和直接指针两种。

+ 如果使用句柄访问的话，Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据的具体各自的地址信息。如图1所示。

![indirect](/images/2016/05/indirect.png)

+ 如果使用直接指针访问的话，Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，reference中存储的直接就是对象地址，如图2所示。

![direct](/images/2016/05/direct.png)

这两种对象访问方式各有优势，使用句柄来访问的最大好处就是reference中存储的是稳定句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要被修改。

使用直接指针来访问最大的好处就是速度更快，它节省了一次指针定位的时间开销，由于对象访问的在Java中非常频繁，因此这类开销积小成多也是一项非常可观的执行成本。从上一部分讲解的对象内存布局可以看出，就虚拟机`HotSpot`而言，它是使用第二种方式进行对象访问，但在整个软件开发的范围来看，各种语言、框架中使用句柄来访问的情况也十分常见。

## <a id="Garbage_Search">JVM 垃圾对象判断</a>

垃圾对象查找，首先需要明确什么对象可以认为是垃圾对象。

### 垃圾对象

判断一个对象是不是垃圾对象，需要去回收其占用的内存空间，一般有两张方法，一种是基于引用计数，一种是可达性分析。

#### 引用计数

引用计数，操作起来非常简单。就是在全局维护一个Map，当引用一个对象的时候，就给对应的map entry 的value加1；当这个引用失效的时候，就将对应的value减1。当value的值为0的时候，就说明此刻该对象是垃圾对象，可以被回收掉。

因此，引用计数基本上可以做到实时去回收空间，但是它有一个大问题，就是无法处理循环引用的问题。此外，对每一个对象的申请或者销毁都会导致其内部引用的对象计数进行实时更改，这个操作量级还是相当大的，这种开销有时候对应用来说可能都不能容忍。比如，有一个对象内部包含了非常多的局部变量，并且引用的这些变量可能全局Map都剩1，则当这个对象销毁的时候，其会触发大量的引用计数为0，从而大量对象进行销毁回收操作，这些操作可能导致系统暂时无法响应其他请求。

因此，现在一些编译器即使使用引用计数自动回收垃圾，也会加上其他辅助算法，比如标记-清除。

#### 可达性分析

可达性分享，也就是 `Tracing GC`。Tracing GC的核心操作之一就是从给定的根集合出发去遍历对象图。对象图是一种有向图，该图的节点是对象，边是引用。遍历它有两种典型顺序：深度优先（DFS）和广度优先（BFS）。【[HotSpot VM Serial GC的一个问题](http://hllvm.group.iteye.com/group/topic/39376)】

>> 由于深度优先DFS一般采用递归方式实现，处理tracing的时候，可能会导致栈空间溢出，所以一般采用广度优先来实现tracing。

广度优先遍历其实很简单，就是将遍历到了的节点处理，然后将子节点放入到队列中，迭代处理每个队列节点，直到队列为空。如下图所示：

![image](https://upload.wikimedia.org/wikipedia/commons/4/46/Animated_BFS.gif)

对于HotSpot来说，其`GC Roots`为：
+ 虚拟机栈中引用的对象（栈帧中局部变量表中的对象）；
+ 方法区中静态属性和常量引用的对象；
+ 本地方法栈中的对象。
 
因此，在遍历之前，将这些对象放入扫描的队列中，然后依次迭代遍历。

下面给出了一个简化版的遍历算法实现：

``` cpp

void breadth_first_search(Graph* graph) {  
  // 记录灰色对象的队列  
  Queue<Node*> scanning;  
  
  // 1. 一开始对象都是白色的  
  
  // 2. 把根集合的引用能碰到的对象标记为灰色  
  // 由于根集合的引用有可能有重复，所以这里也必须  
  // 在把对象加入队列前先检查它是否已经被扫描到了  
  for (Node* node : graph->root_edges()) {  
    // 如果出边指向的对象还没有被扫描过  
    if (node != nullptr && !node->is_marked()) {  
      node->set_marked();        // 记录下它已经被扫描到了  
      scanning.enqueue(child); // 也把该对象放进灰色队列里等待扫描  
    }  
  }  
  
  // 3. 逐个扫描灰色对象的出边直到没有灰色对象  
  while (!scanning.is_empty()) {  
    Node* parent = scanning.dequeue();  
    for (Node* child : parent->child_nodes() { // 扫描灰色对象的出边  
      // 如果出边指向的对象还没有被扫描过  
      if (child != nullptr && !child->is_marked()) {  
        child->set_marked();       // 把它记录到黑色集合里  
        scanning.enqueue(child); // 也把该对象放进灰色队列里等待扫描  
      }  
    }  
  }  
}  

```

如上遍历完之后，如果某些对象没有被遍历标记到，则它就有`可能`会被回收了。

>> 可能被回收，则说明也有可能不会被回收。其实，在HotSpot的实现中，对象是有两次机会逃过被垃圾回收销毁的命运的。
>>
>> 如果对象进行可达性遍历之后，发现没有和GC Roots有相连的引用，则将该对象标记为不可达对象，并且进行筛选：对象是否覆盖finalize方法或者是否已经执行了该方法来决定是否接下来执行`finalize()`方法。
>>
>> 如果对象判定为需要执行finalize()方法，则将该对象放入`F-Queue`队列中，然后由一个虚拟机创建的低优先级Finalizer线程去触发执行队列中的finalize()方法，但是不一定会被执行（可能前面的finalize方法导致线程一直阻塞无法继续遍历执行下去）。因此，这种情况下，对象可能并不会被销毁回收。
>>
>> 稍后GC将对`F-Queue`队列进行第二次标记，如果这次标记还无效的话，则真的被回收销毁了。


### JVM GC 引用

【[这段 Java 代码中的局部变量能够被提前回收吗？编译器或 VM 能够实现如下的人工优化吗？](http://www.zhihu.com/question/34341582/answer/58444959)】  
【[找出栈上的指针/引用](http://rednaxelafx.iteye.com/blog/1044951)】

在前面章节中已经介绍了内存对象结构和访问方式，并且知道栈帧中的引用类型是GC Roots集合中的一个重要的部分，因此要获取这些引用类型数据就是GC的重要一步了。

#### 保守GC

如果JVM选择不记录任何这种类型的数据，那么它就无法区分内存里某个位置上的数据到底应该解读为引用类型还是原生类型。这种GC就是“保守式GC（conservative GC）”。在进行GC的时候，JVM开始从一些已知位置（例如说JVM栈）开始扫描内存，扫描的时候每看到一个数字就看看它“像不像是一个指向GC堆中的指针”。这里会涉及上下边界检查（GC堆的上下界是已知的）、对齐检查（通常分配空间的时候会有对齐要求，假如说是8字节对齐，那么不能被8整除的数字就肯定不是指针），之类的。然后递归的这么扫描出去。 

保守式GC的好处是相对来说实现简单些，而且可以方便的用在对GC没有特别支持的编程语言里提供自动内存管理功能。比如在C或者C++语言中自己实现。

保守式GC的缺点有：

1. 会有部分对象本来应该已经死了，但有疑似指针指向它们，使它们逃过GC的收集。这对程序语义来说是安全的，因为所有应该活着的对象都会是活的；但对内存占用量来说就不是件好事，总会有一些已经不需要的数据还占用着GC堆空间。具体实现可以通过一些调节来让这种无用对象的比例少一些，可以缓解（但不能根治）内存占用量大的问题。 

2. 由于不知道疑似指针是否真的是指针，所以它们的值都不能改写；移动对象就意味着要修正指针。换言之，对象就`不可移动`了。有一种办法可以在使用保守式GC的同时支持对象的移动，那就是增加一个间接层，不直接通过指针来实现引用，而是添加一层“句柄”（handle）在中间，所有引用先指到一个句柄表里，再从句柄表找到实际对象。这样，要移动对象的话，只要修改句柄表里的内容即可。但是这样的话引用的访问速度就降低了。Sun JDK的Classic VM曾经用过这种全handle的设计，但效果实在算不上好。 

由于JVM要支持丰富的反射功能，本来就需要让对象能了解自身的结构，而且这种信息GC的时候也可以利用上，所以JVM都会保留一些信息在对象上，而不会采用完全保守式的GC。

#### 半保守GC

JVM可以选择在栈上不记录类型信息，而在对象上记录类型信息。这样的话，扫描栈的时候仍然会跟上面说的过程一样，但扫描到GC堆内的对象时，因为对象带有足够类型信息，JVM就够判断出在该对象内什么位置的数据是引用类型了。这种是“半保守式GC”。 

为了支持半保守式GC，运行时需要在对象上带有足够的元数据。如果是JVM的话，这些数据可能在类加载器或者对象模型的模块里计算得到，但不需要JIT编译器的特别支持。

由于半保守式GC在堆内部的数据是准确的，所以它可以在直接使用指针来实现引用的条件下支持部分对象的移动，方法是只将保守扫描能直接扫到的对象设置为不可移动（pinned），而从它们出发再扫描到的对象就可以移动了。

完全保守的GC通常使用不移动对象的算法，例如mark-sweep。半保守方式的GC既可以使用mark-sweep，也可以使用移动部分对象的算法。半保守式GC对JNI方法调用的支持会比较容易：管它是不是JNI方法调用，是栈都扫过去，不需要对引用做任何额外的处理。当然代价跟完全保守式一样，会有“疑似指针”的问题。

#### 准确GC

与保守式GC相对的是“准确式GC”。也就是说给定某个位置上的某块数据，要能知道它的准确类型是什么，这样才可以合理地解读数据的含义；GC所关心的含义就是“这块数据是不是指针”。 要实现这样的GC，JVM就要能够判断出所有位置上的数据是不是指向GC堆里的引用，包括活动记录（栈+寄存器）里的数据。

实现方法是：从外部记录下类型信息，存成映射表。HotSpot把这样的数据结构叫做OopMap。其每次都遍历原始的映射表，循环的一个个偏移量扫描过去；这种用法也叫“解释式”；HotSpot采用的是这种方式。(示例可以参考safePoint例子)

在HotSpot中，对象的类型信息里有记录自己的OopMap，记录了在该类型的对象内什么偏移量上是什么类型的数据。所以从对象开始向外的扫描可以是准确的；这些数据是在类加载过程中计算得到的。 

每个被JIT编译过后的方法也会在一些特定的位置记录下OopMap，记录了执行到该方法的某条指令的时候，栈上和寄存器里哪些位置是引用。这样GC在扫描栈的时候就会查询这些OopMap就知道哪里是引用了。这些位置就是“安全点”（safepoint）。之所以要选择一些特定的位置来记录OopMap，是因为如果对每条指令（的位置）都记录OopMap的话，这些记录就会比较大，那么空间开销会显得不值得。选用一些比较关键的点来记录就能有效的缩小需要记录的数据量，但仍然能达到区分引用的目的。因为这样，HotSpot中GC不是在任意位置都可以进入，而只能在safepoint处进入。而仍然在解释器中执行的方法(非JIT优化的代码)则可以通过解释器里的功能自动生成出OopMap出来给GC用。 

平时这些OopMap都是压缩了存在内存里的；在GC的时候才按需解压出来使用。 

#### GC SafePoint位置

在HotSpot JVM中，解释执行的方法可以在任何字节码边界上执行GC，但是对于采用JIT编译执行的方法，则不能想GC就GC了，必须进入到GC SafePoint 位置。

+ 主动SafePoint：由方法里的代码主动轮询去发现需要进入SafePoint，两种情况：
    + 循环回跳处(loop backedge)；
    + 方法返回处(return)；
    + 可能抛出异常的问题(throw)。
+ 被动SafePoint：调用别的方法的调用点。之所以叫做“被动”是因为并不是该方法主动发现要进入safepoint的，而是某个被调用的方法主动进入了safepoint，导致其整条调用链上的调用者都被动的进入了safepoint。

比如下面的这个简单方法：

``` java

public static void main() {
  LargeObject lo = new LargeObject(); // safepoint 1/2: 被动safepoint
  // OopMap 1: 空。
  // 该栈帧里尚未有任何引用类型的局部变量是活跃的——new还没执行完
  // OopMap 2: 记录了一个变量：刚分配的对象的引用是活的，在OopMap里；不过这还不是局部变量lo而是求值栈上的一个slot
  lo.doSomeThing();                   // safepoint 3: 被动safepoint
  // OopMap 3: 空。局部变量lo的最后一次使用是作为上面方法的"this"参数传递出去；
  // 维持"this"的存活是被调用方法的责任而不是调用方法的责任。此后局部变量lo再也没有被使用过，所以对main()来说lo在此处已死。

  while (true) {
    whatever();                       // safepoint 4: 被动safepoint
    // OopMap 4: 空。

    // safepoint 5: 主动safepoint：循环回跳轮询（backedge poll）
    // OopMap 5: 空。
  }
  // 上面循环是无限循环，所以下面如果有代码都属于不可到达的代码（unreachable code）

  // 如果上面的循环不是无限循环的话，则：
  // safepoint 6: 主动safepoint：返回前轮询（return poll）
  // OopMap 6: 空
}

```

## <a id="JVM_Management">JVM 内存管理</a>

前面讲了这么多，都是在介绍JVM内存中一些区域的划分，对象在内存中的结构，以及HotSpot JVM 支持的精确式GC所采用的数据结构等等。

下面，进入重点。Java 中创建一个对象时，JVM 内存如何处理，当内存空间不够时，又是如何触发垃圾回收的。

### 内存分配

一般的，在java里面讨论内存分配都以为是建立在堆上，但这个不熟绝对的。细分的话，给一个新对象分配内存空间，可能是在栈上，TLAB(Thread Local Allocation Buffer) 或者堆上。

#### 逃逸分析之栈分配

在 JVM中提供了一种内存分配优化的方式，就是基于逃逸分析技术，将新的对象实例分配到栈上。所谓逃逸分析，就是分析指针`动态范围`的方法。简而言之，就是当一个对象的指针被多个方法或线程引用时，我们称这个指针发生了逃逸。而用来分析这种逃逸现象的方法，就称之为逃逸分析。

``` java

class Test{
    public static List<Integer> list ;
    
    public void setList(){
        list = new ArrayList<>(); // list是全局变量，发生逃逸
    }
    
    public List<Integer> getListValue(){
        return new ArrayList<>();;    // 返回了list，发生了逃逸
    }
    
    public void echoTmp(){
        TestClass clazz = new TestClass();//未发生逃逸
        clazz.echo();
        System.out.println('ok.');
    }
}

```

那么，当一个方法中有变量没有发生逃逸事件，则VM可以根据设置进行内存分配优化。

从上文的对象定位可以看到，一般情况下，java对象都是在堆上分配，然后其引用指针保存在调用栈对应位置，然后在访问的时候，需要两次查找才能获取完所有的需要的对象信息，而在栈上分配对象内存的话，就避免了这个耗时。此外，当对象不在使用的时候，由于只是方法内部使用，所以下次GC就需要回收这些对象，这又需要GC线程去遍历对象树来回收内存。

>> 逃逸分析的内存优化并不能在静态编译的时候进行，而需要在JIT动态编译的时候触发。逃逸分析是基于方法内部执行来的，当时在静态编译的时候，由于Java动态代理，反射等特性导致最终运行时候的逻辑是无法知道的。关于这个话题具体讨论参考【[逃逸分析为何不能在编译期进行？](https://www.zhihu.com/question/27963717)】
>>
>> 在测试的时候，你需要` -XX:+DoEscapeAnalysis`设置来打开逃逸分析的优化技术。但是由于基于逃逸分析开销会有一些，而这些未必能抵住获得的性能，所以需要仔细测试是否需要打开该项优化。


#### TLAB 分配区

在创建新的对象而申请内存的时候，则需要去堆上获取内存空间，在多线程的情况下，则需要进行同步锁操作，进而影响分配效率，因此，JVM在内存新生代`Eden Space`中开辟了一小块线程私有的区域，称作TLAB。默认设定为占用Eden Space的1%。

在Java程序中很多对象都是小对象且用完之后就需要回收，它们不存在线程共享也适合被快速GC，所以对于小对象通常JVM会优先分配在TLAB上，并且TLAB上的分配由于是线程私有所以没有锁开销。因此在实践中分配多个小对象的效率通常比分配一个大对象的效率要高。

>> TLAB是针对线程的小对象的，如果需要分配的对象过大，还是需要在堆上进行分配。`-XX:TLABWasteTargetPercent`来设置TLAB占用Eden区的比例。

#### 堆空间分配

如果优化全开，则JVM会先进行逃逸分析，如果未逃逸则在栈上进行分配；否则在TLAB是否有足够的空间分配，如果有，则在TLAB上分配空间；如果依然失败，则要在堆上进行分配了。

>> Notes：由于堆是所有线程共享的，所以分配时是需要加锁同步的。

Oracle官方参考文档【[Java Garbage Collection Basics](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html#t1s2)】

首先，来看看JVM 堆空间详细的分代划分：

![image](/images/2016/05/jvm_heap.png)

>> 一般，我们使用`-Xms`来设置JVM初始内存空间大小，`-Xmx`设计JVM最大内存空间大小，`-Xmn`来设置新生代的内存大小(推荐为整个堆的3/8)，`-Xss`来设置虚拟机栈的空间大小。默认，Eden区和Survivor区的内存空间比例为`8:1:1`，

然后，看一个新对象是如何创建的：

1. 一个对象分配到Eden区，此时survivor区还是一个空内存。
![image](/images/2016/05/allocate_1.png)

2. 然后，当Eden区逐渐增加，当Eden空间不足以分配新的内存申请操作的时候，就会触发minor GC。
![image](/images/2016/05/allocate_2.png)

3. 在Eden中仍存活的对象，则copy到survivor0区中，没有存活的对象则被清除掉。
![image](/images/2016/05/allocate_3.png)

4. 循环进行，当下一个minor GC发生的时候，不存活的对象被删除回收，存活的对象被copy到survivor1中。此外，上一次 minor GC被迁移到Survior0上的对象，存活的迁移到Survivor1上，并且对应的对象分代年龄上的value+1。因此，在S1上，就存在不同分代年龄的对象了。
![image](/images/2016/05/allocate_4.png)

5. 在下一次minor GC中，依然重复上面的动作，将Eden区和S1区的仍然存活的对象copy到S0上，并且对应对象的分代年龄+1。
![image](/images/2016/05/allocate_5.png)

6. 在minor GC 之后，如果存在对象的分代年龄超过设置的`-XX:MaxTenuringThreshold`值时，则copy到年老代Tenured区。
![image](/images/2016/05/allocate_6.png)

7. minor GC 继续，将会有越来越多的对象分配到Tenured区。
![image](/images/2016/05/allocate_7.png)

8. 最后，随着Tenured区的对象越来越多，从而触发了major GC，改GC会将年老代的不用对象进行清除，然后压缩年老代内存空间。
![image](/images/2016/05/allocate_8.png) 

>> 当需要分配的对象内存足够大时，则直接分配到Tenured区。Full GC和major GC 不一样，其GC时，包含minor GC 和 major GC。


### 内存回收

在介绍内存回收之前，需要对内存对象生命周期进行分析。IBM研究表明，绝大部分新生对象都是朝生夕死的，也就是一次GC之后就会把对应内存空间释放出来，最后存活下来的对象很少。如下图所示（显然绝大部分对象在第一次GC的时候就被回收掉了）：  
![image](/images/2016/05/gc_mm.png) 

上图还表明，不同的对象的生命周期不同，对不同的GC策略影响也是不同的。这种表现行为说明，需要对内存对象空间进行分类，因此，在HotSpot中，进行了分代：年轻代和年老代，简单来理解(不完全)：年轻代都是最近才分配内存空间的对象，而年老代则是在年轻代存活足够多的时间升级到年老代来的对象。它的分代内存空间可以参考上图(内存分配)。

常见的GC策略：

+ 标记-清除策略(Mark-Sweep)。       

  标记-清除策略应该是垃圾回收策略里面最基础最容易想到的算法。其核心思想，就是将需要进行回收的对象进行标记，然后再将没有标记的内存对象进行回收销毁。因此，这个算法存在两个阶段，首先需要从GC Rootes开始根据前面介绍的可达性分析来遍历整个内存对象树，把遍历过得对象进行标记；然后，GC算法将所有未marked的对象delete掉。整个算法很容易明白，但是对于内存分配和回收策略来说，其存在二个比较大的问题：
    + 内存碎片化问题。对内存进行大量的标记清除操作之后，会导致整个内存区域存在大量不连续内存碎片，最后如果需要分配大内存对象时，很可能需要进入下一次GC操作，甚至会导致分配失败，但是实际上内存空间总量是很充足的。针对碎片化，一般采取的策略是压缩整理方式，即将对象的内存碎片压缩移动到一起，然后将所有区域外的内存全部delete掉即可。
    + 效率问题。标记-清除，需要首先标记整个内存空间，然后回收未被标记的对象。都是比较影响性能的操作。

+ 复制策略(Copying)。   

  由于标记-清除策略效率不高，对于高并发实时服务来说，是很致命的。
  根据内存生命周期的特点，我们将新生的对象放在一个新生代的Eden内存区域中，然后GC存活下来的对象复制到另一块小的内存区域S0/S1中。此外，新生代可能内存不足或者一些需要长期存活的对象，这些对象就需要分配移动到年老代的内存区域中。  

  对于标记-清除两步骤的操作，复制策略由于采取的内存对象操作不同，导致其只需要一个步骤就能完成GC操作。在遍历GC Roots 对象树的时候，同时将标记到得对象复制到新生代小的S0/S1内存区域中，这样，当我们遍历完对象空间的时候，可以将整个新生代Eden区域内存分配的当前有效内存开始指针置为开始位置，这样，我们就操作完了对新生代对象的GC回收。

+ 标记-整理策略(Mark-Compact/Sweep)。

  复制策略效率非常高，但是其是针对新生代的对象特性而优化的，对于年老代的对象来说，其没有那么高的效率，此外主要的就是其浪费大量的内存空间用于复制GC下来的对象。因此，标记-清除策略对于老年代来说，比复制算法还是更好的，我们只需要解决内存碎片化问题即可。因此，这就是标记-整理算法了。

  也就是，我们在标记完整个内存区域的对象树之后，将存活下来的对象依次移动到内存的一端，然后将内存分配指针设置为存活对象的最后，从而完成内存的清理。

## <a id="GC_Algorithm">JDK GC 算法</a>

HotSpot VM 对外提供的垃圾回收：

![image](/images/2016/05/gc.png)

### 年轻代三大GC

如上图所示，对于年轻代的垃圾回收主要有三种：Serial GC，ParNew GC，ParallelScavenge(PS)。

#### 串行Serial GC

单线程垃圾回收。单线程串行的，所以在进行垃圾回收的时候，需要所有的应用线程都暂停(`Stop The World`)。全局单线程执行垃圾回收，会得到最好的单线程收集效率，因此对于client来说，其实默认的垃圾回收策略。

#### ParNew GC

顾名思义，就是Serial GC的多线程版本，同样也是`Stop The World`。由于ParNew GC 是目前唯一一个可以专注于老年代GC收集器CMS 配合的并行年轻代GC收集器，CMS优秀的GC性能，导致ParNew GC 被很多server应用采用。

>> 并发和并行收集器：并发指应用工作线程和GC线程可以同时执行，分别在不同地CPU上；并行指的是多个GC线程同时工作，此时的工作线程是STOP的状态。

#### Parallel Scavenge (吞吐量GC)

ParallelSvavenge 收集器也是采用复制算法的多线程年轻代收集器。和ParNew GC不同的是，其关注点在可控吞吐量上，而后者则重视的是GC时间。因此，ParNew收集器适用于用户交互场景，而PS则适用于CPU计算而少交互的场景。吞吐量指CPU用于运行用户代码的时间和CPU总运行时间的比例，即吞吐量=用户代码运行时间/(用户代码运行时间+GC时间)。

>> ParallelScavenge和ParNew都是并行GC，主要是并行收集年轻代，目的和性能其实都差不多。
>>
>>最明显的区别有下面几点： 
>> 1. 在JDK6u18之后，PS只用深度优先遍历。ParNew则是一直都只用广度优先顺序来遍历。 
>>
>> 2. PS完整实现了自适应大小策略，而ParNew内则没有实现完。所以千万别在用`ParNew + CMS`的组合下用UseAdaptiveSizePolicy，请只在使用UseParallelGC或UseParallelOldGC的时候用它。 
>> 3. 由于在"分代式GC框架"内，ParNew可以跟CMS搭配使用，而ParallelScavenge不能。当时ParNew GC被从Exact VM移植到HotSpot VM的最大原因就是为了跟CMS搭配使用。 

### 年老代三代GC

如上图所示，对于年轻代的垃圾回收主要有三种：Serial Old，Parallel Old，Concurrent Mark Sweep(CMS)。

#### 串行Serial Old

老年代单线程收集器，使用标记整理策略。Serial Old的标记-整理策略是先标记存活对象，然后先清理掉需要回收的对象，然后将存活的对象进行移动，保证一部分都是存活对象，一部分是空闲的内存空间。和SerialGC 一样，都需要暂停所有工作线程。

>> client模式下默认的Old GC。在Server模式下，当CMS GC时出现`Concurrent Mode Failure`，则回退使用Serial Old 收集器。

#### Parallel Old

Parallel Old 是多线程并行的老年代GC收集器，其主要是配合年轻代PS收集器的老年代版本。

>> 和Serial Old不同的整理策略，其首先汇总存活的对象复制到一个区域，然后压缩起来，而不是直接清除再压缩（为什么不直接将存活的对象移动到一侧呢？）。

*以上所有的收集器在执行GC的整个过程中都需要Stop The World，即不允许工作线程在GC的时候运行。*

#### CMS

>> 遇到的绝大部分server一般都是采用ParNew + CMS 收集器策略。 

CMS收集器目标是获取最短停顿时间的收集器，所以和上面的收集器不同，虽然其也会`Stop The World`，但是在有些阶段是可以并发进行的。因此，特别适合用户交互场景的服务系统。

CMS 主要分为4个阶段：

- 初始化标记(STW)：标记GC Roots可以直接关联到的对象，速度非常快。
- 并发标记：进行GC Roots Tracing的过程。
- 重新标记(STW)：修正在并发标记期间工作线程操作导致变更的部分对象的标记记录。
- 并发清除。

上面四个步骤中，1和3是需要暂停工作线程的。

由于在实际中，应用特别多，所以这里具体分析下CMS收集器。

【[Java系列笔记(3) - Java 内存区域和GC机制](http://www.cnblogs.com/zhguang/p/3257367.html)】

CMS收集的执行过程是：初始标记(CMS-initial-mark) -> 并发标记(CMS-concurrent-mark) -->预清理(CMS-concurrent-preclean)-->可控预清理(CMS-concurrent-abortable-preclean)-> 重新标记(CMS-remark) -> 并发清除(CMS-concurrent-sweep) ->并发重设状态等待下次CMS的触发(CMS-concurrent-reset)
具体的说，先2次标记，1次预清理，1次重新标记，再1次清除。 

1. 首先jvm根据-XX:CMSInitiatingOccupancyFraction，-XX:+UseCMSInitiatingOccupancyOnly来决定什么时间开始垃圾收集；
2. 如果设置了-XX:+UseCMSInitiatingOccupancyOnly，那么只有当old代占用确实达到了-XX:CMSInitiatingOccupancyFraction参数所设定的比例时才会触发cms gc；
3. 如果没有设置-XX:+UseCMSInitiatingOccupancyOnly，那么系统会根据统计数据自行决定什么时候触发cms gc；因此有时会遇到设置了80%比例才cms gc，但是50%时就已经触发了，就是因为这个参数没有设置的原因；
4. 当cms gc开始时，首先的阶段是初始标记(CMS-initial-mark)，是stop the world阶段，因此此阶段标记的对象只是从root集最直接可达的对象；
     CMS-initial-mark：961330K（1572864K），指标记时，old代的已用空间和总空间
5. 下一个阶段是并发标记(CMS-concurrent-mark)，此阶段是和应用线程并发执行的，所谓并发收集器指的就是这个，主要作用是标记可达的对象，此阶段不需要用户停顿。
       此阶段会打印2条日志：CMS-concurrent-mark-start，CMS-concurrent-mark
6. 下一个阶段是CMS-concurrent-preclean，此阶段主要是进行一些预清理，因为标记和应用线程是并发执行的，因此会有些对象的状态在标记后会改变，此阶段正是解决这个问题因为之后的Rescan阶段也会stop the world，为了使暂停的时间尽可能的小，也需要preclean阶段先做一部分工作以节省时间
     此阶段会打印2条日志：CMS-concurrent-preclean-start，CMS-concurrent-preclean
7. 下一阶段是CMS-concurrent-abortable-preclean阶段，加入此阶段的目的是使cms gc更加可控一些，作用也是执行一些预清理，以减少Rescan阶段造成应用暂停的时间
     此阶段涉及几个参数：
     -XX:CMSMaxAbortablePrecleanTime：当abortable-preclean阶段执行达到这个时间时才会结束
     -XX:CMSScheduleRemarkEdenSizeThreshold（默认2m）：控制abortable-preclean阶段什么时候开始执行，
      即当eden使用达到此值时，才会开始abortable-preclean阶段
     -XX:CMSScheduleRemarkEdenPenetratio（默认50%）：控制abortable-preclean阶段什么时候结束执行
      此阶段会打印一些日志如下：
     CMS-concurrent-abortable-preclean-start，CMS-concurrent-abortable-preclean，
      CMS：abort preclean due to time XXX
8. 再下一个阶段是第二个stop the world阶段了，即Rescan阶段，此阶段暂停应用线程，停顿时间比并发标记小得多，但比初始标记稍长。对对象进行重新扫描并标记；
       YG occupancy：964861K（2403008K），指执行时young代的情况
       CMS remark：961330K（1572864K），指执行时old代的情况
      此外，还打印出了弱引用处理、类卸载等过程的耗时
9. 再下一个阶段是CMS-concurrent-sweep，进行并发的垃圾清理
10. 最后是CMS-concurrent-reset，为下一次cms gc重置相关数据结构
 
有2种情况会触发CMS 的悲观full gc，在悲观full gc时，整个应用会暂停
       A，concurrent-mode-failure：预清理阶段可能出现，当cms gc正进行时，此时有新的对象要进行old代，但是old代空间不足造成的。其可能性有：1，O区空间不足以让新生代晋级，2，O区空间用完之前，无法完成对无引用的对象的清理。这表明，当前有大量数据进入内存且无法释放。
       B，promotion-failed：新生代young gc可能出现，当进行young gc时，有部分young代对象仍然可用，但是S1或S2放不下，因此需要放到old代，但此时old代空间无法容纳此。
 
影响cms gc时长及触发的参数是以下2个：
        -XX:CMSMaxAbortablePrecleanTime=5000
        -XX:CMSInitiatingOccupancyFraction=80
解决也是针对这两个参数来的，根本的原因是每次请求消耗的内存量过大
解决方式：
      A，针对cms gc的触发阶段，调整-XX:CMSInitiatingOccupancyFraction=50，提早触发cms gc，就可以缓解当old代达到80%，cms gc处理不完，从而造成concurrent mode failure引发full gc
     B，修改-XX:CMSMaxAbortablePrecleanTime=500，缩小CMS-concurrent-abortable-preclean阶段的时间
     C，考虑到cms gc时不会进行compact，因此加入-XX:+UseCMSCompactAtFullCollection
       （cms gc后会进行内存的compact）和-XX:CMSFullGCsBeforeCompaction=4（在full gc4次后会进行compact）参数
 
在CMS清理过程中，只有初始标记和重新标记需要短暂停顿，并发标记和并发清除都不需要暂停用户线程，因此效率很高，很适合高交互的场合。
CMS也有缺点，它需要消耗额外的CPU和内存资源，在CPU和内存资源紧张，CPU较少时，会加重系统负担（CMS默认启动线程数为(CPU数量+3)/4）。
另外，在并发收集过程中，用户线程仍然在运行，仍然产生内存垃圾，所以可能产生“浮动垃圾”，本次无法清理，只能下一次Full GC才清理，因此在GC期间，需要预留足够的内存给用户线程使用。所以使用CMS的收集器并不是老年代满了才触发Full GC，而是在使用了一大半（默认68%，即2/3，使用-XX:CMSInitiatingOccupancyFraction来设置）的时候就要进行Full GC，如果用户线程消耗内存不是特别大，可以适当调高-XX:CMSInitiatingOccupancyFraction以降低GC次数，提高性能，如果预留的用户线程内存不够，则会触发Concurrent Mode Failure，此时，将触发备用方案：使用Serial Old 收集器进行收集，但这样停顿时间就长了，因此-XX:CMSInitiatingOccupancyFraction不宜设的过大。
还有，CMS采用的是标记清除算法，会导致内存碎片的产生，可以使用-XX：+UseCMSCompactAtFullCollection来设置是否在Full GC之后进行碎片整理，用-XX：CMSFullGCsBeforeCompaction来设置在执行多少次不压缩的Full GC之后，来一次带压缩的Full GC。


