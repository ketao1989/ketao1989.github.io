---
layout: post
categories: 
  - Java 
  - Thread 
title:  "Java Fork&Join框架使用和实现分析"
date: 2014-05-24 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

在并发编程网上,关于ForkJoin框架介绍得很好，推荐去看: [Fork/Join框架](http://ifeve.com/fork-join-1/) 本篇博文只是对一些地方进行补充说明(为了文章连续性，会借鉴一些介绍文字).

在上一篇博文: [Java 多线程线程池介绍](http://ketao1989.github.io/posts/Java-MultiThread-ThreadPool-Introduce.html) 中最后说明了，对于一个任务可以切割成多个小任务分别执行，然后把各个小任务的结果，组合成最终的结论。熟悉`MapReduce`的同学，肯定对此再熟悉不过了。

首先贴出一个很简单的代码demo，这段代码是对上篇博文中代码，用`ForkJoin` API方式来实现（实际上，这并不是一个好的介绍`ForkJoin`功能的例子，但是我们先用它来入门了）

>> `ForkJoin`任务，继承自`RecursiveAction`，因为我们不需要任务返回什么计算结果：

``` java

package io.github.ketao1989;

import java.util.List;
import java.util.concurrent.RecursiveAction;

/**
 * 很简单的一个操作，就是把字符串加一个后缀，然后放进队列里
 * 
 * @author: ketao Date: 14-5-24 Time: 下午10:16
 * @version: \$Id$
 */
public class ListTask extends RecursiveAction {

    private static final int THRESHOLD = 1;

    private List<String> processStr;
    private int start;
    private int end;

    public ListTask(List<String> processStr, int start, int end) {
        this.processStr = processStr;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        boolean isProcess = (end - start) == THRESHOLD;
        if (isProcess) {
            System.out.println(Thread.currentThread().getName());
            String newStr = processStr.get(start) + "-test";
            processStr.set(start, newStr);

        } else {
            System.out.println(Thread.currentThread().getName()+"----");
            int partPos = (start + end) / 2;
            ListTask taskl = new ListTask(processStr, start, partPos);
            ListTask taskr = new ListTask(processStr, partPos, end);
            invokeAll(taskl, taskr);
        }
    }

}
```

<!-- more -->

>> `ForkJoin`的DEMO主函数，最后如果任务正常结束，则打印`任务顺利完成`信息：

``` java

package io.github.ketao1989;

import java.util.List;
import java.util.concurrent.ForkJoinPool;

import com.alibaba.fastjson.JSON;
import com.google.common.collect.Lists;

/**
 * @author: ketao Date: 14-5-24 Time: 下午10:12
 * @version: \$Id: ForkJoinTest.java 6 2014-05-24 14:13:48Z ketao1989 $
 */
public class ForkJoinTest {

    private final static ForkJoinPool pool = new ForkJoinPool();

    public static void main(String[] args) {
        List<String> list = Lists.newArrayList("thread-11", "thread-21", "thread-31", "thread-41", "thread-51",
                "thread-61", "thread-71", "thread-81");
        ListTask listTask = new ListTask(list, 0, list.size());
        pool.invoke(listTask);
        System.out.println(JSON.toJSONString(list));
        pool.shutdown();
        if (listTask.isCompletedNormally()) {
            System.out.println("Task 任务顺利完成！");
        }
    }
}

```

>> 执行结果如下，如我们所期望的那样：
	
	ForkJoinPool-1-worker-1----
	ForkJoinPool-1-worker-1----
	ForkJoinPool-1-worker-1----
	ForkJoinPool-1-worker-1
	ForkJoinPool-1-worker-3----
	ForkJoinPool-1-worker-3
	ForkJoinPool-1-worker-3
	ForkJoinPool-1-worker-2----
	ForkJoinPool-1-worker-1
	ForkJoinPool-1-worker-4----
	ForkJoinPool-1-worker-4
	ForkJoinPool-1-worker-4
	ForkJoinPool-1-worker-2----
	ForkJoinPool-1-worker-2
	ForkJoinPool-1-worker-2
	["thread-11-test","thread-21-test","thread-31-test","thread-41-test","thread-51-test","thread-61-test","thread-71-test","thread-81-test"]
	Task 任务顺利完成！
	
上面的代码，其是同步执行任务，也就是说当任务开始执行时，主线程会阻塞执行任务，直到任务执行完成。和线程池一样，你也可以使用Future来完成异步执行任务。此外，对于需要返回结果的`ForkJoin`，Task类可以继承`RecursiveTask<T>`类。


## <a id="ForkJoin">ForkJoin框架介绍</a>

`ForkJoin`框架其本质就是将一个大任务分割成多个小任务来执行，然后将每个小任务执行的结果合并为我们需要的返回值。因此，和当前云计算框架`MapReduce`一样，其计算主要分两步：

	1. Fork操作：就是把一个大的任务分割成多个更小的子任务，然后执行这些小的子任务；
	
	2. Join操作：顾名思义就是等待所有任务完成后返回。

因此可以看出，命名意义和`Linux C`的`Thread`的API定义保持一致。借鉴网络上得一张图来形象描述下：

<img src="/images/2014/05/forkjoin-work.jpg" />

这个框架被设计用来解决可以使用分而治之技术将任务分解成更小的问题。在一个任务中，检查你想要解决问题的大小，如果它大于一个既定的大小，把它分解成更小的任务，然后用这个框架来执行。如果问题的大小是小于既定的大小，你直接在任务中解决这问题。它返回一个可选地结果。

Fork/Join 和Executor框架主要的区别是`work-stealing`算法，可以参考上一篇博文：[Java 多线程线程池介绍](http://ketao1989.github.io/posts/Java-MultiThread-ThreadPool-Introduce.html)。不像Executor框架，当一个任务正在等待它使用join操作创建的子任务的结 束时，执行这个任务的线程（工作线程）查找其他未被执行的任务并开始它的执行。通过这种方式，线程充分利用它们的运行时间，从而提高了应用程序的性能。

工作窃取算法，`work-stealing`算法存在可以帮助我们充分利用线程资源来减少执行时间。

>> Tips: 我们把一个大的任务分割成多个不相互依赖的小的子任务，并且把这些子任务分别放在不同的执行队列中，每个执行队列分别创建一个单独的线程来执行任务。默认线程数（队列数）为执行机器的CPU核数+1，具体可以看看上面DEMO执行的线程编号。每个队列分别有一个线程单独去执行，是为了避免或减少线程间的竞争。当某线程执行完队列中得所有任务时，而有其他线程没有完成对应队列中的任务时，线程会协助其他线程完成其对应队列中剩余的任务。为了避免线程间获取队列任务时产生竞争，显然会采取双端队列从而线程可以从队列尾部拿到还未被执行的任务，而真正执行队列任务的线程，则依然从队列头部获取任务。当然，该算法遇到队列只有一个任务时，也会产生竞争，并且多个队列和多个线程，也会消耗更多的系统资源。

为实现这个目标，Fork/Join框架执行的任务有以下局限性：

	1. 任务只能使用`fork()`和`join()`操作，作为同步机制。如果使用其他同步机制，工作线程不能执行其他任务，当它们在同步操作时。比如，在Fork/Join框架中，你使任务进入睡眠，正在执行这个任务的工作线程将不会执行其他任务，在这睡眠期间内。
	
	2. 任务不应该执行I/O操作，如读或写数据文件。

	3. 任务不能抛出检查异常，它必须包括必要的代码来处理它们。

Fork/Join框架的核心是由以下两个类：

	1. ForkJoinPool：它实现ExecutorService接口和work-stealing算法。它管理工作线程和提供关于任务的状态和它们执行的信息。
	
	2. ForkJoinTask： 它是将在ForkJoinPool中执行的任务的基类。它提供在任务中执行fork()和join()操作的机制，并且这两个方法控制任务的状态。通常， 为了实现你的Fork/Join任务，你将实现两个子类的子类的类：RecursiveAction对于没有返回结果的任务和RecursiveTask 对于返回结果的任务。

## <a id="API">ForkJoin API介绍</a>

一般地，你需要按照下面两种情况下使用`ForkJoin`框架的API：

>> `RecursiveAction`任务对应的API使用模型：

``` java

		If (problem size < default size){
            tasks=divide(task);
            execute(tasks);
        } else {
            resolve problem using another algorithm;
        }
```

>> `RecursiveTask<V>`类任务对应的API使用模型：

``` java

        If (problem size < size){
            tasks=Divide(task);
            execute(tasks);
            groupResults()
            return result;
        } else {
            resolve problem;
            return result;
        }
```

首先，看看`ForkJoinPool`类的构造函数，和一些重要的对外提供的方法：

``` java

	/**
	 * 创建线程数为当前系统CPU核数+1的{@code ForkJoinPool}对象，该对象使用{@linkplain
     * #defaultForkJoinWorkerThreadFactory default thread factory}，没有异常处理器和非异步的LIFO处理模式
     *
     * @throws SecurityException if a security manager exists and
     *         the caller is not permitted to modify threads
     *         because it does not hold {@link
     *         java.lang.RuntimePermission}{@code ("modifyThread")}
     */
    public ForkJoinPool() {
        this(Runtime.getRuntime().availableProcessors(),
             defaultForkJoinWorkerThreadFactory, null, false);
    }
    
    // 指定线程数
    public ForkJoinPool(int parallelism) {
        this(parallelism, defaultForkJoinWorkerThreadFactory, null, false);
    }
    
    // 原生的构造函数
    public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        Thread.UncaughtExceptionHandler handler,
                        boolean asyncMode)
    
    // 执行给定的task任务，直到执行完成之后返回它的结果
    public <T> T invoke(ForkJoinTask<T> task)
    
	// 异步执行给定的task任务
    public void execute(ForkJoinTask<?> task)
    
    // 提交一个 ForkJoinTask 任务去执行
    public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task)
    
    // 按照先前提交任务的顺序关闭，但是不在接收新的任务。对于已经关闭的pool，不会有副作用。
    public void shutdown()
```

接下来，看看`RecursiveAction`类的构造函数，以及相应地方法：

``` java

	// 抽象类，
	public abstract class RecursiveAction extends ForkJoinTask<Void>{

    /**
     * 任务需要执行的代码. 继承该类的子类，需要重写该方法
     */
    protected abstract void compute();
}
```

最后，看看`RecursiveTask`类的构造函数，以及相应地方法：

``` java
	// 抽象类，
	public abstract class RecursiveTask<V> extends ForkJoinTask<V>{
    /**
     * 任务需要执行的代码. 继承该类的子类，需要重写该方法
     */
    protected abstract V compute();
}
```

## <a id="Example">ForkJoin 使用示例</a>

在前言中已经给出了关于`RecursiveAction`的demo，下面来看看使用`RecursiveTask`来实现该问题的代码，一并说明异步返回：

``` java

package io.github.ketao1989;

import java.util.List;
import java.util.concurrent.RecursiveTask;

import com.google.common.collect.Lists;

/**
 * 很简单的一个操作，就是把字符串加一个后缀，然后放进队列里
 * 
 * @author: ketao Date: 14-5-24 Time: 下午10:16
 * @version: \$Id$
 */
public class ListTask extends RecursiveTask<List<String>> {

    private static final int THRESHOLD = 1;

    private List<String> processStr;

    private int start;
    private int end;

    public ListTask(List<String> processStr, int start, int end) {
        this.processStr = processStr;
        this.start = start;
        this.end = end;
    }

    @Override
    protected List<String> compute() {
        List<String> result = Lists.newArrayListWithCapacity(processStr.size());
        boolean isProcess = (end - start) == THRESHOLD;
        if (isProcess) {
            System.out.println(Thread.currentThread().getName());
            String newStr = processStr.get(start) + "-test";
            result.add(newStr);

        } else {
            System.out.println(Thread.currentThread().getName()+"---");
            int partPos = (start + end) / 2;
            ListTask taskl = new ListTask(processStr, start, partPos);
            ListTask taskr = new ListTask(processStr, partPos, end);

            taskl.fork(); //按序异步执行这个任务，会放到一个队列里
            taskr.fork();

            List<String> resultl = taskl.join(); //等待执行完成后返回，调用isDone 会返回true
            List<String> resultr = taskr.join();
            
            result.addAll(resultl);
            result.addAll(resultr);
        }



        return result;
    }
}

/**
 * 测试主函数
 * 
 * @author: ketao Date: 14-5-24 Time: 下午10:12
 * @version: \$Id: ForkJoinTest.java 6 2014-05-24 14:13:48Z ketao1989 $
 */
public class ForkJoinTest {

    private final static ForkJoinPool pool = new ForkJoinPool();

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        List<String> list = Lists.newArrayList("thread-11", "thread-21", "thread-31", "thread-41", "thread-51",
                "thread-61", "thread-71", "thread-81");
        ListTask listTask = new ListTask(list, 0, list.size());
        Future<List<String>> result = pool.submit(listTask);
        System.out.println(JSON.toJSONString(result.get()));
        pool.shutdown();
        if (listTask.isCompletedNormally()) {
            System.out.println("Task 任务顺利完成！");
        }
    }

}

```

执行结果，如下所示：

	ForkJoinPool-1-worker-1---
	ForkJoinPool-1-worker-2---
	ForkJoinPool-1-worker-3---
	ForkJoinPool-1-worker-4---
	ForkJoinPool-1-worker-5
	ForkJoinPool-1-worker-5---
	ForkJoinPool-1-worker-3
	ForkJoinPool-1-worker-3---
	ForkJoinPool-1-worker-2
	ForkJoinPool-1-worker-2
	ForkJoinPool-1-worker-5
	ForkJoinPool-1-worker-1---
	ForkJoinPool-1-worker-2
	ForkJoinPool-1-worker-2
	ForkJoinPool-1-worker-3
	["thread-11-test","thread-21-test","thread-31-test","thread-41-test","thread-51-test","thread-61-test","thread-71-test","thread-81-test"]
	Task 任务顺利完成！

>> demo代码很简单，这里不进行说明。

## <a id="Analyze">ForkJoin 实现剖析</a>

`ForkJoin`整体框架相对简单明了，实现起来，也就是`ForkJoinTask` 和`ForkJoinWorkerThread`两部分，其中Task负责存放需要执行的任务，而Thread负责执行任务即可。具体实现，如下分析。

### 5.1 ForkJoinPool实现分析

首先，看`ForkJoinPool`类的构造函数，代码如下：

``` java

	public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        Thread.UncaughtExceptionHandler handler,
                        boolean asyncMode) {
        checkPermission(); // 安全管理，检查操作是否有权限修改线程
        if (factory == null)
            throw new NullPointerException();
        if (parallelism <= 0 || parallelism > MAX_ID)
            throw new IllegalArgumentException();
        this.parallelism = parallelism;
        this.factory = factory;
        this.ueh = handler;
        this.locallyFifo = asyncMode;
        long np = (long)(-parallelism); // offset ctl counts
        this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);//ctl是整个池的核心控制技术变量，说明见下面
        this.submissionQueue = new ForkJoinTask<?>[INITIAL_QUEUE_CAPACITY]; // 提交任务队列
        // initialize workers array with room for 2*parallelism if possible
        int n = parallelism << 1;
        if (n >= MAX_ID)
            n = MAX_ID;
        else { // 当 n < (1 << 16)时，计算 n对应2进制的后面所有bit位为1，比如：6 = 110B --> 111B = 7 ；8 = 1000B --> 1111B = 15
            n |= n >>> 1; n |= n >>> 2; n |= n >>> 4; n |= n >>> 8;
        }
        workers = new ForkJoinWorkerThread[n + 1]; //执行任务的线程数组，n+1
        this.submissionLock = new ReentrantLock();
        this.termination = submissionLock.newCondition();
        StringBuilder sb = new StringBuilder("ForkJoinPool-");
        sb.append(poolNumberGenerator.incrementAndGet()); // pool 序数
        sb.append("-worker-");
        this.workerNamePrefix = sb.toString(); // 线程名前缀在demo中，结果中打印出来了
    }


```

>> `ForkJoinPool`代码中变量`volatile long ctl`包含了`forkjoinpool`几个核心的数值，使用bit位来表示。具体为： AC(16 bits)--活跃运行的`worker`数量减去当前系统`parallelism`值；TC(16 bits)--总的`worker`数减去当前系统`parallelism`值；ST（1 bits）-- `pool`是否结束；EC(15 bits) --等待线程组的头部的等待数；ID（16 bits）-- 正在等待的线程组栈顶的索引`poolIndex`. 

---

>> Tips: 在构造函数中，创建了两个对象，分别是大小为`8`的`ForkJoinTask`数组 和 大小为`n+1`（4核Cpu为8）的 `ForkJoinWorkerThread`。因此，可以知道**在初始化的时候，提交任务队列的大小 和 执行任务的线程数 很可能不相等**。

接下来需要说明的是，`ForkJoinPool`的`submit`方法，其会调用`forkOrSubmit(ForkJoinTask<T> task)`，实现代码如下：

``` java

 	private <T> void forkOrSubmit(ForkJoinTask<T> task) {
        ForkJoinWorkerThread w;
        Thread t = Thread.currentThread();
        if (shutdown)
            throw new RejectedExecutionException();
        if ((t instanceof ForkJoinWorkerThread) &&
            (w = (ForkJoinWorkerThread)t).pool == this)
            w.pushTask(task);//push 该任务到该线程对应的队列中
        else
            addSubmission(task); //把任务task 插入到submissionQueue队列中
    }
    
```

>> 因此，需要执行的任务task已经被放进了队列中，执行线程可以获取任务来进行执行了。`addSubmission`运行时会使用`this.submissionLock`锁，并且入队之后，会调用`signalWork()`方法，该方法会根据当前`pool`中`worker`数量和状态来决定 唤醒或者创建一个worker。

---

>> 在`pool`中有一个核心的顶层循环，所有的工作线程都会按照这个步骤执行：

``` java

    /**
     * 在每一步：如果上一步顺利通过所有的队列，并且发现没有了任务；或者有多余的线程，则可能会阻塞。此外，扫描scan，如果发现任务，则执行。
     * 当pool和 worker结束的时候，返回， 
     */
    final void work(ForkJoinWorkerThread w) {
        boolean swept = false;                // true on empty scans
        long c;
        while (!w.terminate && (int)(c = ctl) >= 0) { //当线程未结束，并且还有任务未完成执行
            int a;                            // active count
            if (!swept && (a = (int)(c >> AC_SHIFT)) <= 0)
                swept = scan(w, a); //扫描任务，发现，则执行
            else if (tryAwaitWork(w, c)) //把worker线程放入等待queue中，等待worker的eventCount改变。
                swept = false;
        }
    }
    
```

>> `Scan`方法的逻辑其实很简单，就是首先获取其线程内部的queue，执行任务；如果完了，则steal其他`worker`线程的任务；如果还没有，则执行pool中的`submissionQueue`。再没有，则返回true。

``` java

	/**
     * Scans for and, if found, executes one task. Scans start at a
     * random index of workers array, and randomly select the first
     * (2*#workers)-1 probes, and then, if all empty, resort to 2
     * circular sweeps, which is necessary to check quiescence. and
     * taking a submission only if no stealable tasks were found.  The
     * steal code inside the loop is a specialized form of
     * ForkJoinWorkerThread.deqTask, followed bookkeeping to support
     * helpJoinTask and signal propagation. The code for submission
     * queues is almost identical. On each steal, the worker completes
     * not only the task, but also all local tasks that this task may
     * have generated. On detecting staleness or contention when
     * trying to take a task, this method returns without finishing
     * sweep, which allows global state rechecks before retry.
     *
     * @param w the worker
     * @param a the number of active workers
     * @return true if swept all queues without finding a task
     */
    private boolean scan(ForkJoinWorkerThread w, int a) {
        int g = scanGuard; // mask 0 avoids useless scans if only one active
        int m = (parallelism == 1 - a && blockedCount == 0) ? 0 : g & SMASK;
        ForkJoinWorkerThread[] ws = workers;
        if (ws == null || ws.length <= m)         // staleness check
            return false;
        for (int r = w.seed, k = r, j = -(m + m); j <= m + m; ++j) {
            ForkJoinTask<?> t; ForkJoinTask<?>[] q; int b, i;
            ForkJoinWorkerThread v = ws[k & m];
            if (v != null && (b = v.queueBase) != v.queueTop &&
                (q = v.queue) != null && (i = (q.length - 1) & b) >= 0) {
                long u = (i << ASHIFT) + ABASE;
                if ((t = q[i]) != null && v.queueBase == b &&
                    UNSAFE.compareAndSwapObject(q, u, t, null)) {
                    int d = (v.queueBase = b + 1) - v.queueTop;
                    v.stealHint = w.poolIndex;
                    if (d != 0)
                        signalWork();             // propagate if nonempty
                    w.execTask(t);
                }
                r ^= r << 13; r ^= r >>> 17; w.seed = r ^ (r << 5);
                return false;                     // store next seed
            }
            else if (j < 0) {                     // xorshift
                r ^= r << 13; r ^= r >>> 17; k = r ^= r << 5;
            }
            else
                ++k;
        }
        if (scanGuard != g)                       // staleness check
            return false;
        else {                                    // try to take submission
            ForkJoinTask<?> t; ForkJoinTask<?>[] q; int b, i;
            if ((b = queueBase) != queueTop &&
                (q = submissionQueue) != null &&
                (i = (q.length - 1) & b) >= 0) {
                long u = (i << ASHIFT) + ABASE;
                if ((t = q[i]) != null && queueBase == b &&
                    UNSAFE.compareAndSwapObject(q, u, t, null)) {
                    queueBase = b + 1;
                    w.execTask(t);
                }
                return false;
            }
            return true;                         // all queues empty
        }
    }
    
```


### 5.2 ForkJoinWorkerThread实现分析

在`submit`方法中调用了`pushTask(ForkJoinTask<?> t)`方法，其实现在`ForkJoinWorkerThread`类中。`ForkJoinWorkerThread`类是用来被`ForkJoinPool`管理的线程类型，该类线程值执行`ForkJoinTask`类任务对象。

依然首先看看其构造方法：

``` java

    /**
     * 在给定的pool里面创建一个 ForkJoinWorkerThread 实例.
     */
    protected ForkJoinWorkerThread(ForkJoinPool pool) {
        super(pool.nextWorkerName()); // 使用Thread调用pool中指定的线程名前缀
        this.pool = pool;
        int k = pool.registerWorker(this); //注册线程到pool得worker数组中，获取在pool数组里对应的index索引
        poolIndex = k;
        eventCount = ~k & SMASK; // clear wait count
        locallyFifo = pool.locallyFifo;
        Thread.UncaughtExceptionHandler ueh = pool.ueh;
        if (ueh != null)
            setUncaughtExceptionHandler(ueh);
        setDaemon(true); //守护线程
    }

```

>> Tips: 在构造方法里面，新建的线程实例，会注册到`pool`的`worker`数组中去，当`worker`数组大小不够，会进行`CopyOf`操作，把大小扩大原来的一倍。此外，代码的实现被没有获取lock操作。此外，创建的线程被指定为`守护进程`。

接着来看看了`pushTask(ForkJoinTask<?> t)`方法的实现，该方法和`pool`的`addSubmission`方法基本一致，除了`addSubmission`会增加互斥锁操作。代码如下：

``` java

    /**
     * Pushes a task. Call only from this thread.
     *
     * @param t the task. Caller must ensure non-null.
     */
    final void pushTask(ForkJoinTask<?> t) {
        ForkJoinTask<?>[] q; int s, m;
        if ((q = queue) != null) {    // ignore if queue removed
            long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
            UNSAFE.putOrderedObject(q, u, t); // 把q数组偏移量为u的对应的值，置为t。不保证及时内存可见，如果field不为volatile
            queueTop = s + 1;         // or use putOrderedInt
            if ((s -= queueBase) <= 2)
                pool.signalWork(); //唤醒或者新建worker线程
            else if (s == m)
                growQueue(); //当s的值和队列值长度length-1一样时，即队列已满，则增加队列大小。
        }
    }


```

>> 关于`UNSAFE`的实现，底层实现的`native`方法是C++，具体代码可以参见：[UNSAFE 源码实现链接](http://www.oschina.net/code/explore/gcc-4.5.2/libjava/sun/misc)

---

>> 作为一个`Thread`的继承子类，必然需要实现`run`方法，实现细节如下：

``` java

	public void run() {
        Throwable exception = null;
        try {
            onStart(); // 该方法主要负责初始化Task 队列，和seed值
            pool.work(this); // 调用pool的work方法，在pool中说明
        } catch (Throwable ex) {
            exception = ex;
        } finally {
            onTermination(exception);// 清除该worker线程关于结束的一些操作，比如取消任务，解除在pool上的注册，状态为结束terminate
        }
    }

```


### 5.3 ForkJoinTask实现分析

在API接口描述中，可以看出`RecursiveAction`类和`RecursiveTask`类都继承自`ForkJoinTask`抽象类，唯一不同就是一个不返回执行结果。在`ForkJoinTask`中需要关注的就是`join`方法和`fork`方法。

首先是`fork`方法的实现：

``` java

    /**
     * 按序的异步执行这个任务.
     */
    public final ForkJoinTask<V> fork() {
        ((ForkJoinWorkerThread) Thread.currentThread())
            .pushTask(this);
        return this;
    }

```

>> `fork`方法实际上就是把新创建的子任务提交给当前线程，由当前线程push到它自身的队列数组中。

接下来看看`join`方法的实现：

``` java

   /**
     *当任务执行完成后，返回执行的结果，该方法和`Feture.get()`不同的地方时，其抛出的异常是`RuntimeException`和`Error`。
     *此外，也不会抛出`InterruptedException`。
     */
    public final V join() {
        if (doJoin() != NORMAL) // 任务没有正常完成
            return reportResult(); //处理非正常情况
        else
            return getRawResult(); // 返回结果
    }
```

>> `doJoin()`方法算是`ForkJoinTask`类主要方法之一，其他的方法`doInvoke`、`doExec`方法和`doJoin`一样，都会执行核心的任务自定义`compute`方法。

``` java

	/**
     * Primary mechanics for join, get, quietlyJoin.
     * @return status upon completion
     */
    private int doJoin() {
        Thread t; ForkJoinWorkerThread w; int s; boolean completed;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
            if ((s = status) < 0) // 如果任务已经完成，则直接返回
                return s;
            if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) { //从当前线程的任务数组中 pop 该任务，准备执行
                try {
                    completed = exec(); // 调用自定义任务的compute方法执行
                } catch (Throwable rex) {
                    return setExceptionalCompletion(rex);
                }
                if (completed)
                    return setCompletion(NORMAL); //如果顺利正常完成，则设置为正常完成状态
            }
            return w.joinTask(this); //当任务没有正常完成，可能阻塞什么的，则会给helpJoinTask stolen->joining 方式执行
        }
        else
            return externalAwaitDone();
    }

```

---

### 5.4 joinTask 方法实现分析

`joinTask`方法的具体实现在`ForkJoinWorkerThread`类中。但是由于其实现了 `ForkJoin`中关于`work-stealing`算法的实现，所以当初分析下。

``` java

	// helpJoinTask允许的最大stolen->joining 链深度，同时也是重试的最大次数
    private static final int MAX_HELP = 16;

    final int joinTask(ForkJoinTask<?> joinMe) {
        ForkJoinTask<?> prevJoin = currentJoin; //保存当前在执行的任务
        currentJoin = joinMe;
        for (int s, retries = MAX_HELP;;) {
            if ((s = joinMe.status) < 0) { //当joinMe任务正常完成，则执行原来正在执行的任务，返回执行状态
                currentJoin = prevJoin; 
                return s;
            }
            if (retries > 0) {
                if (queueTop != queueBase) { //当前队列中有任务未被执行
                    if (!localHelpJoinTask(joinMe)) //并且队列中还存在其他未取消的任务，则不重试，扔到pool.tryAwaitJoin中
                        retries = 0;           // cannot help
                }
                else if (retries == MAX_HELP >>> 1) { //这个值为什么这么判断呢？？为什么retries == 8 执行下面逻辑？？
                    --retries;                 // check uncommon case
                    if (tryDeqAndExec(joinMe) >= 0) // 当joinMe是一些worker 队列的base上面，则steal，并且执行，执行的状态为不正常完成时
                        Thread.yield();        // 则礼貌性的暂停任务
                       }
                else
                	// 尝试定位和执行给定任务的stealer的任务集，或者轮流执行他的所有stealers的一个。如果运行一个任务，则返回true
                    retries = helpJoinTask(joinMe) ? MAX_HELP : retries - 1; 
            }
            else {
                retries = MAX_HELP;           // restart if not done
                pool.tryAwaitJoin(joinMe);
            }
        }
    }
    

    private boolean helpJoinTask(ForkJoinTask<?> joinMe) {
        boolean helped = false;
        int m = pool.scanGuard & SMASK;
        ForkJoinWorkerThread[] ws = pool.workers; //获取pool所有的worker线程数组
        if (ws != null && ws.length > m && joinMe.status >= 0) { 
            int levels = MAX_HELP;              // remaining chain length
            ForkJoinTask<?> task = joinMe;      // base of chain
            outer:for (ForkJoinWorkerThread thread = this;;) {
                // Try to find v, the stealer of task, by first using hint
                ForkJoinWorkerThread v = ws[thread.stealHint & m];
                if (v == null || v.currentSteal != task) {
                    for (int j = 0; ;) {        // search array
                        if ((v = ws[j]) != null && v.currentSteal == task) {
                            thread.stealHint = j;
                            break;              // save hint for next time
                        }
                        if (++j > m)
                            break outer;        // can't find stealer
                    }
                }
                // Try to help v, using specialized form of deqTask
                for (;;) {
                    ForkJoinTask<?>[] q; int b, i;
                    if (joinMe.status < 0)
                        break outer;
                    if ((b = v.queueBase) == v.queueTop ||
                        (q = v.queue) == null ||
                        (i = (q.length-1) & b) < 0)
                        break;                  // empty
                    long u = (i << ASHIFT) + ABASE;
                    ForkJoinTask<?> t = q[i];
                    if (task.status < 0)
                        break outer;            // stale
                    if (t != null && v.queueBase == b &&
                        UNSAFE.compareAndSwapObject(q, u, t, null)) {
                        v.queueBase = b + 1;
                        v.stealHint = poolIndex;
                        ForkJoinTask<?> ps = currentSteal;
                        currentSteal = t;
                        t.doExec(); // 好了，这里获取到了steal到的task，可以执行了
                        currentSteal = ps;
                        helped = true; //执行了任务，这里设为true
                    }
                }
                // Try to descend to find v's stealer
                ForkJoinTask<?> next = v.currentJoin;
                if (--levels > 0 && task.status >= 0 &&
                    next != null && next != task) {
                    task = next;
                    thread = v;
                }
                else
                    break;  // max levels, stale, dead-end, or cyclic
            }
        }
        return helped;
    }


```

## <a id="Finally">小结</a>

本文只是简单地分析了Fork&Join 框架的用法和实现。由于JDK 中 关于多线程的代码，有些还涉及到native得实现，并且代码可读性不是太好，导致有些理解不是很清楚。不过知道大体框架和使用方法，应该就可以满足日常使用了。

Fork Join 框架的思想，在很多地方都可以体现，只是实现的繁简而已。大任务的切割，小任务的并发执行，然后Reuce 各个子结果，就是我们想要的最终值了。

