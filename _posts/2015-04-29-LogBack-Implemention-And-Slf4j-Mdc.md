---
layout: post
categories:
  - Log
  - Java
title:  "Slf4j MDC 使用和 基于 Logback 的实现分析"
date: 2015-04-29 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

如今,在 Java 开发中，日志的打印输出是必不可少的，`Slf4j + LogBack` 的组合是最通用的方式。

关于 `Slf4j` 的介绍,请参考本博客<http://ketao1989.github.io/posts/Java-slf4j-Introduce.html>

有了日志之后，我们就可以追踪各种线上问题。但是，在分布式系统中，各种无关日志穿行其中，导致我们可能无法直接定位整个操作流程。因此，我们可能需要对一个用户的操作流程进行归类标记，比如使用`线程+时间戳`，或者用户身份标识等；如此，我们可以从大量日志信息中grep出某个用户的操作流程，或者某个时间的流转记录。

因此，这就有了 `Slf4j MDC` 方法。


## <a id="MdcIntroduce">Slf4j MDC 介绍</a>

MDC ( Mapped Diagnostic Contexts )，顾名思义，其目的是为了便于我们诊断线上问题而出现的方法工具类。虽然，Slf4j 是用来适配其他的日志具体实现包的，但是针对 MDC功能，目前只有logback 以及 log4j 支持，或者说由于该功能的重要性，slf4j 专门为logback系列包装接口提供外部调用(玩笑～：）)。

> logback 和 log4j 的作者为同一人，所以这里统称logback系列。

先来看看 MDC 对外提高的接口：

``` java

public class MDC {
  //Put a context value as identified by key
  //into the current thread's context map.
  public static void put(String key, String val);

  //Get the context identified by the key parameter.
  public static String get(String key);

  //Remove the context identified by the key parameter.
  public static void remove(String key);

  //Clear all entries in the MDC.
  public static void clear();
}

```

> 接口定义非常简单，此外，其使用也非常简单。
> 

<!-- more -->

如上代码所示，一般，我们在代码中，只需要将指定的值put到线程上下文的Map中，然后，在对应的地方使用 get方法获取对应的值。此外，对于一些线程池使用的应用场景，可能我们在最后使用结束时，需要调用clear方法来清洗将要丢弃的数据。

然后，看看一个MDC使用的简单示例。

``` java

public class LogTest {
    private static final Logger logger = LoggerFactory.getLogger(LogTest.class);

    public static void main(String[] args) {

        MDC.put("THREAD_ID", String.valueOf(Thread.currentThread().getId()));

        logger.info("纯字符串信息的info级别日志");

    }

}
```

然后看看logback的输出模板配置：

``` xml

<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <property name="log.base" value="${catalina.base}/logs" />

    <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
        <resetJUL>true</resetJUL>
    </contextListener>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder charset="UTF-8">
            <pattern>[%d{yyyy-MM-dd HH:mm:ss} %highlight(%-5p) %logger.%M\(%F:%L\)] %X{THREAD_ID} %msg%n</pattern>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="console" />
    </root>
</configuration>
```

于是，就有了输出：

    [2015-04-30 15:34:35 INFO  io.github.ketao1989.log4j.LogTest.main(LogTest.java:29)] 1 纯字符串信息的info级别日志



> 当我们在web应用中，对服务的所有请求前进行filter拦截，然后加上自定义的唯一标识到MDC中，就可以在所有日志输出中，清楚看到某用户的操作流程。关于web MDC，会单独一遍博客介绍。
> 
> 此外，关于logback 是如何将模板中的变量替换成具体的值，会在下一节分析。
> 
> 在日志模板logback.xml 中，使用 `%X{ }`来占位，替换到对应的 MDC 中 key 的值。
> 

## <a id="PrepareKnowledge">前置知识介绍</a>

### InheritableThreadLocal 介绍

在代码开发中，经常使用 `ThreadLocal`来保证在同一个线程中共享变量。在 `ThreadLocal` 中，每个线程都拥有了自己独立的一个变量，线程间不存在共享竞争发生，并且它们也能最大限度的由CPU调度，并发执行。显然这是一种以空间来换取线程安全性的策略。

但是，`ThreadLocal`有一个问题，就是它只保证在同一个线程间共享变量，也就是说如果这个线程起了一个新线程，那么新线程是不会得到父线程的变量信息的。因此，为了保证子线程可以拥有父线程的某些变量视图，JDK提供了一个数据结构，`InheritableThreadLocal`。

javadoc 文档对 InheritableThreadLocal 说明：

>  该类扩展了 ThreadLocal，为子线程提供从父线程那里继承的值：在创建子线程时，子线程会接收所有可继承的线程局部变量的初始值，以获得父线程所具有的值。通常，子线程的值与父线程的值是一致的；但是，通过重写这个类中的 childValue 方法，子线程的值可以作为父线程值的一个任意函数。

> 当必须将变量（如用户 ID 和 事务 ID）中维护的每线程属性（per-thread-attribute）自动传送给创建的所有子线程时，应尽可能地采用可继承的线程局部变量，而不是采用普通的线程局部变量。

代码对比可以看出两者区别：

> ThreadLocal:

``` java

public class ThreadLocal<T> {

    /**
     * Method childValue is visibly defined in subclass
     * InheritableThreadLocal, but is internally defined here for the
     * sake of providing createInheritedMap factory method without
     * needing to subclass the map class in InheritableThreadLocal.
     * This technique is preferable to the alternative of embedding
     * instanceof tests in methods.
     */
    T childValue(T parentValue) {
        throw new UnsupportedOperationException();
    }

}


```

> InheritableThreadLocal:

``` java

public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * Get the map associated with a ThreadLocal.
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     * @param map the map to store.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}

// 这个是开发时一般使用的类，直接copy父线程的变量
public class CopyOnInheritThreadLocal extends
    InheritableThreadLocal<HashMap<String, String>> {

  /**
   * Child threads should get a copy of the parent's hashmap.
   */
  @Override
  protected HashMap<String, String> childValue(
      HashMap<String, String> parentValue) {
    if (parentValue == null) {
      return null;
    } else {
      return new HashMap<String, String>(parentValue);
    }
  }

}

```

> 为了支持InheritableThreadLocal的父子线程传递变量，JDK在Thread中，定义了`ThreadLocal.ThreadLocalMap inheritableThreadLocals` 属性。该属性变量在线程初始化的时候，如果父线程的该变量不为null，则会把其值复制到ThreadLocal。

> 从上面的代码实现，还可以看到，如果我们使用原生的 `InheritableThreadLocal`类则在子线程中修改变量，可能会影响到父线程的变量值，及其他子线程的值。因此，一般我们推荐没有特殊情况，最好使用`CopyOnInheritThreadLocal`类，该实现是新建一个map来保持值，而不是直接使用父线程的引用。

## <a id="Slf4jMdc">Log MDC 实现分析</a>

### Slf4j MDC 实现分析

Slf4j 的实现原则就是调用底层具体实现类，比如logback,logging等包；而不会去实现具体的输出打印等操作。因此，除了前文中介绍的门面(Facade)模式外，提供这种功能的还有适配器(Adapter)模式和装饰(Decorator)模式。

MDC 使用的就是`Decorator`模式，虽然，其类命名为M `MDCAdapter`。

Slf4j MDC 内部实现很简单。实现一个单例对应实例，获取具体的MDC实现类，然后其对外接口，就是对参数进行校验，然后调用 MDCAdapter 的方法实现。

实现源码如下：

``` java

public class MDC {

  static MDCAdapter mdcAdapter;

  private MDC() {
  }

  static {
    try {
      mdcAdapter = StaticMDCBinder.SINGLETON.getMDCA();
    } catch (NoClassDefFoundError ncde) {
      //......
  }


  public static void put(String key, String val)
      throws IllegalArgumentException {
    if (key == null) {
      throw new IllegalArgumentException("key parameter cannot be null");
    }
    if (mdcAdapter == null) {
      throw new IllegalStateException("MDCAdapter cannot be null. See also "
          + NULL_MDCA_URL);
    }
    mdcAdapter.put(key, val);
  }

  public static String get(String key) throws IllegalArgumentException {
    if (key == null) {
      throw new IllegalArgumentException("key parameter cannot be null");
    }

    if (mdcAdapter == null) {
      throw new IllegalStateException("MDCAdapter cannot be null. See also "
          + NULL_MDCA_URL);
    }
    return mdcAdapter.get(key);
  }
}  }

```

> 对于Slf4j的MDC 部分非常简单，MDC的核心实现是在logback方法中的。
> 
> 在 logback 中，提供了 `LogbackMDCAdapter`类，其实现了`MDCAdapter`接口。基于性能的考虑，logback 对于InheritableThreadLocal的使用做了一些优化工作。
> 

### Logback MDC 实现分析 

Logback 中基于 MDC 实现了`LogbackMDCAdapter` 类，其 get 方法实现很简单，但是 put 方法会做一些优化操作。

关于 put 方法，主要有：

* 使用原始的`InheritableThreadLocal<Map<String, String>>`类，而不是使用子线程复制类 `CopyOnInheritThreadLocal`。这样，运行时可以大量避免不必要的copy操作，节省CPU消耗，毕竟在大量log操作中，子线程会很少去修改父线程中的`key-value`值。

* 由于上一条的优化，所以代码实现上实现了一个`写时复制版本的 InheritableThreadLocal`。实现会根据上一次操作来确定是否需要copy一份新的引用map，而不是去修改老的父线程的map引用。

* 此外，和 log4j 不同，其map中的val可以为null。

下面给出，get 和 put 的代码实现：

``` java

public final class LogbackMDCAdapter implements MDCAdapter {

  final InheritableThreadLocal<Map<String, String>> copyOnInheritThreadLocal = new InheritableThreadLocal<Map<String, String>();

  private static final int WRITE_OPERATION = 1;
  private static final int READ_OPERATION = 2;

  // keeps track of the last operation performed
  final ThreadLocal<Integer> lastOperation = new ThreadLocal<Integer>();

  private Integer getAndSetLastOperation(int op) {
    Integer lastOp = lastOperation.get();
    lastOperation.set(op);
    return lastOp;
  }

  private boolean wasLastOpReadOrNull(Integer lastOp) {
    return lastOp == null || lastOp.intValue() == READ_OPERATION;
  }

  private Map<String, String> duplicateAndInsertNewMap(Map<String, String> oldMap) {
    Map<String, String> newMap = Collections.synchronizedMap(new HashMap<String, String>());
    if (oldMap != null) {
        // we don't want the parent thread modifying oldMap while we are
        // iterating over it
        synchronized (oldMap) {
          newMap.putAll(oldMap);
        }
    }

    copyOnInheritThreadLocal.set(newMap);
    return newMap;
  }


  public void put(String key, String val) throws IllegalArgumentException {
    if (key == null) {
      throw new IllegalArgumentException("key cannot be null");
    }

    Map<String, String> oldMap = copyOnInheritThreadLocal.get();
    Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);

    if (wasLastOpReadOrNull(lastOp) || oldMap == null) {
      // 当上一次操作是read时，这次write，则需要new
      Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);
      newMap.put(key, val);
    } else {
      // 写的话，已经new了就不需要再new
      oldMap.put(key, val);
    }
  }

  /**
   * Get the context identified by the <code>key</code> parameter.
   * <p/>
   */
  public String get(String key) {
    Map<String, String> map = getPropertyMap();
    if ((map != null) && (key != null)) {
      return map.get(key);
    } else {
      return null;
    }
  }
}

```

> 需要注意，在上面的代码中，write操作即put会去修改 `lastOperation` ，而get操作则不会。这样就保证了，只会在第一次写时复制。

### MDC clear 操作

> Notes：对于涉及到ThreadLocal相关使用的接口，都需要去考虑在使用完上下文对象时，清除掉对应的数据，以避免内存泄露问题。

    因此，下面来分析下在MDC中如何清除掉不在需要的对象。

在MDC中提供了`clear`方法，该方法完成对象的清除工作，使用logback时，则调用的是`LogbackMDCAdapter#clear()`方法，继而调用`copyOnInheritThreadLocal.remove()`。

在ThreadLocal中，实现`remove()`方法：

``` java

     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }

```

> 这里，就是调用`ThreadLocal#remove`方法完成对象清理工作。
> 
> 所有线程的ThreadLocal都是以`ThreadLocalMap`来维护的，也就是，我们获取threadLocal对象时，实际上是根据当前线程去该Map中获取之前的设置。在清除的时候，从这个Map中获取对应的对象，然后移除map.

## <a id="LogbackPrint">Logback 日志输出实现</a>

MDC 的功能实现很简单，就是在线程上下文中，维护一个 `Map<String,String>` 属性来支持日志输出的时候，当我们在配置文件`logback.xml` 中配置了`%X{key}`，则后台日志打印出对应的 key 的值。

同样，`logback.xml`配置文件支持了多种格式的日志输出，比如`%highlight`、`%d`等等，这些标志，在`PatternLayout.java`中维护。

``` java

public class PatternLayout extends PatternLayoutBase<ILoggingEvent> {

  public static final Map<String, String> defaultConverterMap = new HashMap<String, String>();
  public static final String HEADER_PREFIX = "#logback.classic pattern: ";
  
  static {
    defaultConverterMap.putAll(Parser.DEFAULT_COMPOSITE_CONVERTER_MAP);
    // 按照{}配置输出时间
    defaultConverterMap.put("d", DateConverter.class.getName());
    defaultConverterMap.put("date", DateConverter.class.getName());
    // 输出应用启动到日志时间触发时候的毫秒数
    defaultConverterMap.put("r", RelativeTimeConverter.class.getName());
    defaultConverterMap.put("relative", RelativeTimeConverter.class.getName());
    // 输出日志级别的信息
    defaultConverterMap.put("level", LevelConverter.class.getName());
    defaultConverterMap.put("le", LevelConverter.class.getName());
    defaultConverterMap.put("p", LevelConverter.class.getName());
    // 输出产生日志事件的线程名
    defaultConverterMap.put("t", ThreadConverter.class.getName());
    defaultConverterMap.put("thread", ThreadConverter.class.getName());
    // 输出产生log事件的原点的日志名=我们创建logger的时候设置的
    defaultConverterMap.put("lo", LoggerConverter.class.getName());
    defaultConverterMap.put("logger", LoggerConverter.class.getName());
    defaultConverterMap.put("c", LoggerConverter.class.getName());
    // 输出 提供日志事件的对应的应用信息
    defaultConverterMap.put("m", MessageConverter.class.getName());
    defaultConverterMap.put("msg", MessageConverter.class.getName());
    defaultConverterMap.put("message", MessageConverter.class.getName());
    // 输出调用方发布日志事件的完整类名
    defaultConverterMap.put("C", ClassOfCallerConverter.class.getName());
    defaultConverterMap.put("class", ClassOfCallerConverter.class.getName());
    // 输出发布日志请求的方法名
    defaultConverterMap.put("M", MethodOfCallerConverter.class.getName());
    defaultConverterMap.put("method", MethodOfCallerConverter.class.getName());
    // 输出log请求的行数
    defaultConverterMap.put("L", LineOfCallerConverter.class.getName());
    defaultConverterMap.put("line", LineOfCallerConverter.class.getName());
    // 输出发布日志请求的java源码的文件名
    defaultConverterMap.put("F", FileOfCallerConverter.class.getName());
    defaultConverterMap.put("file", FileOfCallerConverter.class.getName());
    // 输出和发布日志事件关联的线程的MDC
    defaultConverterMap.put("X", MDCConverter.class.getName());
    defaultConverterMap.put("mdc", MDCConverter.class.getName());
    // 输出和日志事件关联的异常的堆栈信息
    defaultConverterMap.put("ex", ThrowableProxyConverter.class.getName());
    defaultConverterMap.put("exception", ThrowableProxyConverter.class
        .getName());
    defaultConverterMap.put("rEx", RootCauseFirstThrowableProxyConverter.class.getName());
    defaultConverterMap.put("rootException", RootCauseFirstThrowableProxyConverter.class
        .getName());
    defaultConverterMap.put("throwable", ThrowableProxyConverter.class
        .getName());
    // 和上面一样，此外增加类的包信息
    defaultConverterMap.put("xEx", ExtendedThrowableProxyConverter.class.getName());
    defaultConverterMap.put("xException", ExtendedThrowableProxyConverter.class
        .getName());
    defaultConverterMap.put("xThrowable", ExtendedThrowableProxyConverter.class
        .getName());
    // 当我们想不输出异常信息时，使用这个。其假装处理异常，其实无任何输出
    defaultConverterMap.put("nopex", NopThrowableInformationConverter.class
        .getName());
    defaultConverterMap.put("nopexception",
        NopThrowableInformationConverter.class.getName());
    // 输出在类附加到日志上的上下文名字. 
    defaultConverterMap.put("cn", ContextNameConverter.class.getName());
    defaultConverterMap.put("contextName", ContextNameConverter.class.getName());
    // 输出产生日志事件的调用者的位置信息
    defaultConverterMap.put("caller", CallerDataConverter.class.getName());
    // 输出和日志请求关联的marker
    defaultConverterMap.put("marker", MarkerConverter.class.getName());
    // 输出属性对应的值，一般为System.properties中的属性
    defaultConverterMap.put("property", PropertyConverter.class.getName());
    // 输出依赖系统的行分隔符
    defaultConverterMap.put("n", LineSeparatorConverter.class.getName());
    // 相关的颜色格式设置
    defaultConverterMap.put("black", BlackCompositeConverter.class.getName());
    defaultConverterMap.put("red", RedCompositeConverter.class.getName());
    defaultConverterMap.put("green", GreenCompositeConverter.class.getName());
    defaultConverterMap.put("yellow", YellowCompositeConverter.class.getName());
    defaultConverterMap.put("blue", BlueCompositeConverter.class.getName());
    defaultConverterMap.put("magenta", MagentaCompositeConverter.class.getName());
    defaultConverterMap.put("cyan", CyanCompositeConverter.class.getName());
    defaultConverterMap.put("white", WhiteCompositeConverter.class.getName());
    defaultConverterMap.put("gray", GrayCompositeConverter.class.getName());
    defaultConverterMap.put("boldRed", BoldRedCompositeConverter.class.getName());
    defaultConverterMap.put("boldGreen", BoldGreenCompositeConverter.class.getName());
    defaultConverterMap.put("boldYellow", BoldYellowCompositeConverter.class.getName());
    defaultConverterMap.put("boldBlue", BoldBlueCompositeConverter.class.getName());
    defaultConverterMap.put("boldMagenta", BoldMagentaCompositeConverter.class.getName());
    defaultConverterMap.put("boldCyan", BoldCyanCompositeConverter.class.getName());
    defaultConverterMap.put("boldWhite", BoldWhiteCompositeConverter.class.getName());
    defaultConverterMap.put("highlight", HighlightingCompositeConverter.class.getName());
  }
}
```

> Notes：日志模板配置，使用 `%`为前缀让解析器识别特殊输出模式，然后以`{}`后缀结尾，内部指定相应的参数设置。

### 初始化

所谓初始化，就是我们构建`logger`的时候。在`LoggerFactory.getLogger()`，调用的是 slf4j 的方法，而底层使用的是`logback`的实现。因此，初始化的重点就是找到底层具体的实现接口，然后构建具体类。

``` java

  public static Logger getLogger(String name) {
    ILoggerFactory iLoggerFactory = getILoggerFactory();
    return iLoggerFactory.getLogger(name);
  }

  public static ILoggerFactory getILoggerFactory() {
    if (INITIALIZATION_STATE == UNINITIALIZED) {
      INITIALIZATION_STATE = ONGOING_INITIALIZATION;
      performInitialization();
    }
    switch (INITIALIZATION_STATE) {
      case SUCCESSFUL_INITIALIZATION:
        return StaticLoggerBinder.getSingleton().getLoggerFactory();
      // ......
      case ONGOING_INITIALIZATION:
        // support re-entrant behavior.
        // See also http://bugzilla.slf4j.org/show_bug.cgi?id=106
        return TEMP_FACTORY;
    }
    // .....
  }
}

  private final static void bind() {
    try {
      Set staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
      // the next line does the binding
      // 这里并没有使用上面的返回set进行反射构建类，这里实际上才是各种初始化的地方
      StaticLoggerBinder.getSingleton();
      INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
      // ......
    } catch (Exception e) {
      // ......
    }
  }

  private static Set findPossibleStaticLoggerBinderPathSet() {
    // use Set instead of list in order to deal with  bug #138
    // LinkedHashSet appropriate here because it preserves insertion order during iteration
    Set staticLoggerBinderPathSet = new LinkedHashSet();
    try {
      ClassLoader loggerFactoryClassLoader = LoggerFactory.class
              .getClassLoader();
      Enumeration paths;
      if (loggerFactoryClassLoader == null) {
        paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
      } else {
        paths = loggerFactoryClassLoader
                .getResources(STATIC_LOGGER_BINDER_PATH);
      }
      while (paths.hasMoreElements()) {
        URL path = (URL) paths.nextElement();
        staticLoggerBinderPathSet.add(path);
      }
    } catch (IOException ioe) {
      Util.report("Error getting resources from path", ioe);
    }
    return staticLoggerBinderPathSet;
  }

```

> 上面的部分代码，可以很明显看出，slf4j 会去调用classloader获取当前加载的类中，实现了指定的接口`org/slf4j/impl/StaticLoggerBinder.class`的类，如果多余1个，则会抛出异常。
> 
> 以上，依然可以从代码中看出这个只是检测是否存在符合接口的实现类，而没有像正常情况那样，通过反射构建类，返回给调用方。如何实现呢？
> 
> 直接在自己的包中实现一个和 `slf4j`要求路径一样的类，实现对应的接口，然后就可以调用了。不明白，看代码吧。:)

``` java

package org.slf4j.impl;

import ch.qos.logback.core.status.StatusUtil;
import org.slf4j.ILoggerFactory;
import org.slf4j.LoggerFactory;
import org.slf4j.helpers.Util;
import org.slf4j.spi.LoggerFactoryBinder;

import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.classic.util.ContextInitializer;
import ch.qos.logback.classic.util.ContextSelectorStaticBinder;
import ch.qos.logback.core.CoreConstants;

public class StaticLoggerBinder implements LoggerFactoryBinder {

  private static StaticLoggerBinder SINGLETON = new StaticLoggerBinder();

  static {
    SINGLETON.init();
  }

  // 这里就是创建logback的LoggerContext实例， 包含了log所需的环境配置
  private LoggerContext defaultLoggerContext = new LoggerContext();
  private final ContextSelectorStaticBinder contextSelectorBinder = ContextSelectorStaticBinder
      .getSingleton();

  private StaticLoggerBinder() {
    defaultLoggerContext.setName(CoreConstants.DEFAULT_CONTEXT_NAME);
  }

  public static StaticLoggerBinder getSingleton() {
    return SINGLETON;
  }

    void init() {
    try {
      try {
        // 这里会初始化配置文件和对应的模板，logback.xml解析
        new ContextInitializer(defaultLoggerContext).autoConfig();
      } 
      // ......
  }
  }}
```

> 从 package 和 import 的信息，可以看出，logback 中实现了一个 `org.slf4j.impl.StaticLoggerBinder` 类，而这个类，在slf4j 的 API 包中直接使用，所以使用slf4j时，必须还要引入其他具体的第三方包来实现相应的接口方法。
> 
> 此外，接下来的核心逻辑就是解析logback下各种配置文件信息，以及初始化配置。

### 输出日志模板解析

如上所见，其实关于logback.xml的解析工作，也是在初始化的时候完成的。但是，由于其重要性，所以这里重点介绍下。

在 logback 中，解析xml的工作，都是交给 Action 和其继承类来完成。在 Action 类中提供了三个方法`begin`、`body`和`end`三个方法，这三个抽象方法中：

* begin 方法负责处理ElementSelector元素的解析；
* body 方法，一般为空，处理文本的；
* end 方法则是处理模板解析的，所以我们的logback.xml的模板解析实在end方法中。具体是在 `NestedComplexPropertyIA`类中来解析。其继承Action类，并且其会调用具体的模板解析工具类：`PatternLayoutEncoder`类和`PatternLayout`类。

`PatternLayoutEncoder`会创建一个`PatternLayout`对象，然后获取到logback.xml中配置的模板字符串，即`[%d{yyyy-MM-dd HH:mm:ss} %highlight(%-5p) %logger.%M\(%F:%L\)] %X{THREAD_ID} %msg%n`，如配置的节点名一样，其在代码中同样赋值给pattern变量。

接下来，PatternLayoutEncoder 会调用相关方法对pattern进行解析，然后构建一个节点链表，保存这个链表会在日志输出的时使用到。

``` java
  public Parser(String pattern, IEscapeUtil escapeUtil) throws ScanException {
    try {
      TokenStream ts = new TokenStream(pattern, escapeUtil);
      this.tokenList = ts.tokenize();
    } catch (IllegalArgumentException npe) {
      throw new ScanException("Failed to initialize Parser", npe);
    }
  }

  enum TokenizerState { LITERAL_STATE,  FORMAT_MODIFIER_STATE, KEYWORD_STATE, OPTION_STATE,  RIGHT_PARENTHESIS_STATE}


  List tokenize() throws ScanException {
    List<Token> tokenList = new ArrayList<Token>();
    StringBuffer buf = new StringBuffer();

    while (pointer < patternLength) {
      char c = pattern.charAt(pointer);
      pointer++;

      switch (state) {
        case LITERAL_STATE:
          handleLiteralState(c, tokenList, buf);
          break;
        case FORMAT_MODIFIER_STATE:
          handleFormatModifierState(c, tokenList, buf);
          break;
        // ......

        default:
      }
    }

    // EOS
    switch (state) {
      case LITERAL_STATE:
        addValuedToken(Token.LITERAL, buf, tokenList);
        break;
     // ......

      case FORMAT_MODIFIER_STATE:
      case OPTION_STATE:
        throw new ScanException("Unexpected end of pattern string");
    }

    return tokenList;
  }

// 构建head链表
  Converter<E> compile() {
    head = tail = null;
    for (Node n = top; n != null; n = n.next) {
      switch (n.type) {
        case Node.LITERAL:
          addToList(new LiteralConverter<E>((String) n.getValue()));
          break;
        case Node.COMPOSITE_KEYWORD:
          CompositeNode cn = (CompositeNode) n;
          CompositeConverter<E> compositeConverter = createCompositeConverter(cn);
          if(compositeConverter == null) {
            addError("Failed to create converter for [%"+cn.getValue()+"] keyword");
            addToList(new LiteralConverter<E>("%PARSER_ERROR["+cn.getValue()+"]"));
            break;
          }
          compositeConverter.setFormattingInfo(cn.getFormatInfo());
          compositeConverter.setOptionList(cn.getOptions());
          Compiler<E> childCompiler = new Compiler<E>(cn.getChildNode(),
                  converterMap);
          childCompiler.setContext(context);
          Converter<E> childConverter = childCompiler.compile();
          compositeConverter.setChildConverter(childConverter);
          addToList(compositeConverter);
          break;
        case Node.SIMPLE_KEYWORD:
          SimpleKeywordNode kn = (SimpleKeywordNode) n;
          DynamicConverter<E> dynaConverter = createConverter(kn);
          if (dynaConverter != null) {
            dynaConverter.setFormattingInfo(kn.getFormatInfo());
            dynaConverter.setOptionList(kn.getOptions());
            addToList(dynaConverter);
          } else {
            // if the appropriate dynaconverter cannot be found, then replace
            // it with a dummy LiteralConverter indicating an error.
            Converter<E> errConveter = new LiteralConverter<E>("%PARSER_ERROR["
                    + kn.getValue() + "]");
            addStatus(new ErrorStatus("[" + kn.getValue()
                    + "] is not a valid conversion word", this));
            addToList(errConveter);
          }

      }
    }
    return head;
  }
```

> 代码很简单，就是依次遍历pattern字符串，然后把符合要求的字符串放进tokenList中，这个list就维护了我们最终需要输出的模板的格式化模式了。
> 
> 在每个case里面，都会对字符串进行特定的处理，匹配具体的字符。
> 
> 在随后的处理中，会将这个tokenList进行转换，成为我们需要的Node类型的拥有head 和 tail 的链表。
> 

### 日志输出分析

构建了各种需要的环境参数，打印日志就很简单了。在需要输出日志的时候，根据初始化得到的Node链表head来解析，遇到%X的时候，从MDC中获取对应的key值，然后append到日志字符串中，然后输出。

> 在配置文件中，我们使用Appender模式，在日志输出类中，显然会调用append类似的方法了。:)
> 
> 其调用流程

``` java
public class OutputStreamAppender<E> extends UnsynchronizedAppenderBase<E> {
  @Override
  protected void append(E eventObject) {
    if (!isStarted()) {
      return;
    }

    subAppend(eventObject);
  }

  // 这里就是日志输出实际的操作，一般如果有需要，可以重写这个方法。
  protected void subAppend(E event) {
    if (!isStarted()) {
      return;
    }
    try {
      // this step avoids LBCLASSIC-139
      if (event instanceof DeferredProcessingAware) {
        // 这里虽然是为输出准备，在检查的同时，把把必要的信息解析出来放到变量中
        ((DeferredProcessingAware) event).prepareForDeferredProcessing();
      }
      // the synchronization prevents the OutputStream from being closed while we
      // are writing. It also prevents multiple threads from entering the same
      // converter. Converters assume that they are in a synchronized block.
      synchronized (lock) {
        // 避免多线程的问题，这里加了锁。而写日志的核心也是在这里
        writeOut(event);
      }
    } catch (IOException ioe) {
      // as soon as an exception occurs, move to non-started state
      // and add a single ErrorStatus to the SM.
      this.started = false;
      addStatus(new ErrorStatus("IO failure in appender", this, ioe));
    }
  }
  // 这里将会调用前面我们提到过的模板类，有该类对解析出来的模板按照当前环境进行输出
  protected void writeOut(E event) throws IOException {
    this.encoder.doEncode(event);
  }
}
```

> Notes：在`prepareForDeferredProcessing`中，查询了一些常用值，比如当前线程名，比如mdc设置赋值到Map中。而这些信息，当准备结束没有出现问题时，则会给后面的输出日志时公用。
> 
> 这种方式，其实在我们的代码中，也可以参考。一般我们可能对当前上下文的入参检查会去查询数据库等耗费CPU或者IO的操作，然后check ok的时候，又会在正常的业务中再次做相同的重复工作，导致不必要的性能损失。
> 

接下来看看，针对模板进行按需获取属性值，然后输出日志的逻辑：

``` java

  // 这里的逻辑就是按照模板获取值然后转换成字节流输出到后台
  public void doEncode(E event) throws IOException {
    String txt = layout.doLayout(event);
    outputStream.write(convertToBytes(txt));
    if (immediateFlush)
      outputStream.flush();
  }

  public String doLayout(ILoggingEvent event) {
    if (!isStarted()) {
      return CoreConstants.EMPTY_STRING;
    }
    return writeLoopOnConverters(event);
  }

  // 初始化的时候，就介绍过最后会构建一个head链表，
  // 这里输出就是按照解析后的链表进行分析输出的。然后根据c类型不同，获取字符串方法也不同
  protected String writeLoopOnConverters(E event) {
    StringBuilder buf = new StringBuilder(128);
    Converter<E> c = head;
    while (c != null) {
      c.write(buf, event);
      c = c.getNext();
    }
    return buf.toString();
  }

```

> 在`writeLoopOnConverters`方法中，获取对应字符串是不同的，其根据不同的Converter，输出也不同。而Converter的判断，时就是根据我们配置的map映射来的，在初始化一节的时候，介绍的`PatternLayout`就包含各种映射关系。至于具体的convert方法，看看mdc的实现：
> 

``` java

public class MDCConverter extends ClassicConverter {

  private String key;
  private String defaultValue = "";

  @Override
  public void start() {
    String[] keyInfo = extractDefaultReplacement(getFirstOption());
    key = keyInfo[0];
    if (keyInfo[1] != null) {
      defaultValue = keyInfo[1];
    }
    super.start();
  }

  @Override
  public void stop() {
    key = null;
    super.stop();
  }

  @Override
  public String convert(ILoggingEvent event) {
    Map<String, String> mdcPropertyMap = event.getMDCPropertyMap();

    if (mdcPropertyMap == null) {
      return defaultValue;
    }

    if (key == null) {
      return outputMDCForAllKeys(mdcPropertyMap);
    } else {

      String value = event.getMDCPropertyMap().get(key);//获取key的值
      if (value != null) {
        return value;
      } else {
        return defaultValue;
      }
    }
  }

  /**
   * if no key is specified, return all the values present in the MDC, in the format "k1=v1, k2=v2, ..."
   */
  private String outputMDCForAllKeys(Map<String, String> mdcPropertyMap) {
    StringBuilder buf = new StringBuilder();
    boolean first = true;
    for (Map.Entry<String, String> entry : mdcPropertyMap.entrySet()) {
      if (first) {
        first = false;
      } else {
        buf.append(", ");
      }
      //format: key0=value0, key1=value1
      buf.append(entry.getKey()).append('=').append(entry.getValue());
    }
    return buf.toString();
  }
}

```

> 我们在MDC实现的时候看到的get方法，在这里就使用了。我们将key:value键值对存放到MDC之后，在logback.xml中配置%X{key}，没有直接调用get方法，logback会根据`X`判断是MDC类型，然后根据`key`拿到MDC中对应的value，然后返回给buf中，最后append到后台日志上。

## <a id="End">后记</a>

其实，本身的 MDC 使用很简单，实现原理也很简单。但是，这里为了分析从将 key:value put 进MDC，然后怎么获取，怎么打印到后台的逻辑，对整个从 SLF4J 到 logback 的运行流程进场了大体解析。而对不影响理解的一些枝节，进行了删减。因此，如果需要完全弄清楚整个逻辑，还需要进行详细分析源码。

在目前的代码中，我们在web.xml 中配置了 filter 来将一些用户个人访问特征存入了MDC中，这样可以获取一个用户的操作流程，根据某一个访问特征去grep的话。

下一次，将分享下这种实现细节背后的一些技术。虽然实现很简单，但是想深入分析下filter机制和web = tomcat + spring mvc 的请求处理流程，这些技术细节，是如何使一个MDC信息可以保存一个用户依次的访问流水记录。
