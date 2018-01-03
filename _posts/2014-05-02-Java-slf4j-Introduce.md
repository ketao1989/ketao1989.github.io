---
layout: post
categories: 
  - Java 
  - Log  
title:  "Java日志框架slf4j API介绍及异常接口实现分析"
date: 2014-05-02 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>
slf4j:简洁的java日志统一接口(Simple Logging Facade for Java),顾名思义,就是一个使用Facade设计模式实现的面向java Logging框架的接口开源包. 
其和java数据库连接工具包JDBC很像, 在JDBC框架中, 各个不同数据库连接器分别针对不同数据库系统来实现对应的连接操作, 而普通程序员只需要使用统一的JDBC接口而不需要关注具体底层使用的数据库类型, 或者针对不同的数据库系统写各种兼容代码.

>> Note: slf4j其实类似于适配器,但是这里不称呼适配器,是因为当底层log日志系统不支持slf4j扩展时,比如log4j,就需要在两者中间增加一个适配器层来完成slf4j调用相关日志系统的操作接口动作.例如,slf4j为log4j提高的slf4j-log412.jar类库,但是logback支持slf4J扩展,所以其不需适配层转换.

同样，slf4j 不参与具体的日志代码实现，它只是在代码编译的时候根据程序的配置来绑定具体的日志系统。这样，使用slf4j类库就可以让你的代码独立于任意一个特定的日志API。因此，如果编写一个对外开发的API活着一个同样的类库，那么为了不限制使用你类库的代码必须使用指定的日志系统，你应该使用slf4j。

相对于其他日志框架，slf4j日志类库的优点和推荐使用的缘由，可以参见 ImportNew 的译文【 [为什么要使用SLF4J而不是Log4J](#http://www.importnew.com/7450.html) 】


## <a id="Facade">Facade设计模式简介</a>

Facade模式，或者叫做外观模式，顾名思义就是封装各个底层子系统的提供的同一类功能接口，统一成一个更易操作使用的上层接口进而对外提供交互。有了这个上层封装的接口，接口调用方只需要调用这个接口，而不需要关于各个子系统的具体逻辑实现。

Facade设计模式的官方定义是：Facade模式定义了一个更高层的接口，使子系统更加容易使用。

<!-- more -->

关于Facade模式的实例，日常生活中很多这样子的例子。比如，5、1回家，可以有好几种方式：飞机、火车、长途汽车。在实际生活中，你回家的路线应该是：

		1. 坐车去机场（火车站/长途汽车站）；
		2. 坐飞机（火车/长途汽车）到家乡；
		3. 从家乡飞机场（火车站/长途汽车站）到家里。 

一般来说，上面的流程是毫无问题的。但是，如果做成一个系统，你需要对外暴露3个步骤中得3个不同的接口，外界需要根据不同的交通方式选择不同的调用接口，这无疑加大了接口调研的复杂度，以及系统的复杂度。如下图所示：

<img src="/images/2014/05/facade.png" />

使用Facade模式，封装各个子系统的实现，对外提供3个接口：

		1. 坐车其站点；
		2. 做主交通工具到家乡；
		3. 从家乡的站点回家里。

因此，接口使用方不需要知道子系统具体是什么样的业务逻辑，其主要要在配置中，或者一开始指定交通工具，就可以让facade系统来完成下面的一系列操作。这样，除了让我们的系统对外暴露接口少了，最重要的是可以让第三方以最低的成本使用我们的接口。


## <a id="Bind">slf4j绑定日志</a>

### 3.1 slf4j 设计模式说明

为了说明slf4j采用的Facade模式，也就是如果只引入slf4j-api包，日志系统将无法正常使用。例如在pom.xml文件这只有：

``` xml
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.5</version>
        </dependency>       
```

而<a id="BindCode">测试代码</a>为：

``` java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author: ketao Date: 14-5-3 Time: 上午1:03
 * @version: \$Id$
 */
public class LogTest {
    private static final Logger logger = LoggerFactory.getLogger(LogTest.class);

    public static void main(String[] args){
        logger.info("Hello world");
        logger.error("ERROR");
    }
}       
```

执行上面的代码会出现提示：

	SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
	SLF4J: Defaulting to no-operation (NOP) logger implementation
	SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.

而如果我们引入logback日志系统，并且配置logback.xml日志配置文件：

``` xml
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.0.13</version>
        </dependency>        
```

接下来执行上面的测试代码，则会打印日志信息：

	[2014-05-03 01:27:11 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:17)] Hello world
	[2014-05-03 01:27:11 [1;31mERROR[0;39m com.qunar.dubbo.LogTest.main(LogTest.java:18)] ERROR
	
### 3.2 slf4j 日志绑定流程

如[3.1](#BindCode)中的代码所示，首先调用`LoggerFactory.getLogger`的方法，这个方法会在编译的时候，绑定系统设置的真正的日志框架，如下代码所示：

``` java
  /**
   * Return a logger named according to the name parameter using the statically
   * bound {@link ILoggerFactory} instance.
   *
   * @param name The name of the logger.
   * @return logger
   */
  public static Logger getLogger(String name) {
    ILoggerFactory iLoggerFactory = getILoggerFactory(); // 这里先获取ILoggerFactory对象
    return iLoggerFactory.getLogger(name); // 根据获取的ILoggerFactory对象，调用其对应的日志对象
  }

   /**
   * Return the {@link ILoggerFactory} instance in use.
   * <p/>
   * <p/>
   * ILoggerFactory instance is bound with this class at compile time.
   *
   * @return the ILoggerFactory instance in use
   */
  public static ILoggerFactory getILoggerFactory() {
    if (INITIALIZATION_STATE == UNINITIALIZED) {
      INITIALIZATION_STATE = ONGOING_INITIALIZATION;
      performInitialization();
    }
    switch (INITIALIZATION_STATE) {
      case SUCCESSFUL_INITIALIZATION:
        return StaticLoggerBinder.getSingleton().getLoggerFactory();// 这里就可以获取底层日志系统的单例对象了
      case NOP_FALLBACK_INITIALIZATION:
        return NOP_FALLBACK_FACTORY;
      case FAILED_INITIALIZATION:
        throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
      case ONGOING_INITIALIZATION:
        // support re-entrant behavior.
        // See also http://bugzilla.slf4j.org/show_bug.cgi?id=106
        return TEMP_FACTORY;
    }
    throw new IllegalStateException("Unreachable code");
  }
}
```

而绑定是在`getILoggerFactory()`中调用的，在该方法的实现里，会调用`performInitialization()`，该方法调用`bind()`方法（部分代码）：

``` java
private final static void bind() {
    try {
      Set staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();// 寻找程序配置的日志系统集，具体见下面代码
      reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);// 验证多于1个日志系统时，输出警告信息
      // the next line does the binding
      StaticLoggerBinder.getSingleton();// 测试是否可以获取该静态绑定类单例，可以，则置为成功状态，如下行；否则，会打出3.1中的NOP异常信息。
      INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
      reportActualBinding(staticLoggerBinderPathSet);// 绑定，打印绑定具体日志系统的日志
      emitSubstituteLoggerWarning();// 提交给临时日志factory 打印的日志，不是重点
    } catch (NoClassDefFoundError ncde) {
      String msg = ncde.getMessage();
      if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
        INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
        Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
        Util.report("Defaulting to no-operation (NOP) logger implementation");
        Util.report("See " + NO_STATICLOGGERBINDER_URL
                + " for further details.");
      } 
    }
  }        
```

下面看看，slf4j是如何获取系统中指定的真正底层日志系统：

``` java
// We need to use the name of the StaticLoggerBinder class, but we can't reference
  // the class itself.
  private static String STATIC_LOGGER_BINDER_PATH = "org/slf4j/impl/StaticLoggerBinder.class";

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

slf4j在适配器层或者在兼容slf4j扩展的log-api 中会有一个`org/slf4j/impl/StaticLoggerBinder.class`类，这样就获取了系统中真正的日志系统。然后获取该日志的单例，打印相关的日志信息就可以了。比如，打印slf4j的`log.info`则调用logback中`Logger.info()`方法来打印日志信息。

## <a id="API">slf4j API使用</a>

slf4j的打印日志基本一致，主要分为：`trace`,`debug`,`info`,`warn`,`error`,比log4j少了`fatal`级别日志。由于每个级别对于的API方法级别一致，因此，这里选用info来介绍不同输入参数的API使用。

>> Tip: SLF4J 认为 ERROR 与 FATAL 并没有实质上的差别，所以拿掉了 FATAL 等级，只剩下其他五种。

``` java
 /**
   * 纯字符串形式的日志
   */
  public void info(String msg);

  /**
   * 指定一个参数和位置格式的info级别的日志输出形式。这个形式避免了多个object对象的创建。
   */
  public void info(String format, Object arg);

  /**
   * 指定2个参数和对于位置格式的info级别的日志输出。
   */
  public void info(String format, Object arg1, Object arg2);

  /**
   * 根据指定的参数和日志格式来输出info级别的日志信息。
   * 但是，需要指出这种形式虽然避免的字符串拼接的成本，但是它会私底下创建一个`Object[]`对象在调用info方法之前，即使info级别的日志不打印。
   * 因此，如果不是必须3个及以上参数的话，推荐使用两个参数和一个参数的info日志。
   */
  public void info(String format, Object... arguments);

  /**
   * 打印抛出异常信息的info 级别日志
   */
  public void info(String msg, Throwable t);
```
 
此外，需要介绍的是在slf4j中还提供了含有Marker对象的日记输出API接口。Marker是常常被用来丰富log状态的对象。遵守slf4j的日志系统实现，决定了信息怎样在使用的Marker之间传达。实际上，很多遵守规范的日志系统会忽视掉marker数据,所以，我们不介绍Marker相关API接口。

下面给出各个接口的使用示例代码：

``` java
import java.io.IOException;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author: ketao Date: 14-5-3 Time: 上午1:03
 * @version: \$Id$
 */
public class LogTest {
    private static final Logger logger = LoggerFactory.getLogger(LogTest.class);

    public static void main(String[] args) {
        logger.info("纯字符串信息的info级别日志");
        logger.info("一个参数:{}的info级别日志", "agr1");
        logger.info("二个参数:agrs1:{};agrs2:{}的info级别日志", "args1", "args2");
        // 下面两种方式都可以，一般使用上面一种就可以了
        logger.info("三个参数:agrs1:{};agrs2:{};args3:{} 的info级别日志", "args1", "args2", "args3");
        logger.info("三个参数:agrs1:{};agrs2:{};args3:{} 的info级别日志", new Object[] { "args1", "args2", "args3" });

        logger.info("======================异常相关====================================");
        // 测试异常相关日志
        logger.info("抛出异常,e:", new IOException("测试抛出IO异常信息"));

        logger.info("二个参数:agrs1:{};agrs2:{}的info级别日志", "args1", new IOException("测试抛出IO异常信息"));
        logger.info("二个参数:agrs1:{};agrs2:{}的info级别日志", "args1", "args2", new IOException("测试抛出IO异常信息"));

        // 下面两种方式都可以，一般使用上面一种就可以了
        logger.info("三个参数:agrs1:{};agrs2:{};args3:{} 的info级别日志", "args1", "args2", new IOException("测试抛出IO异常信息"));
        logger.info("三个参数:agrs1:{};agrs2:{};args3:{} 的info级别日志", "args1", "args2", "agrs3", new IOException(
                "测试抛出IO异常信息"));

        logger.info("三个参数:agrs1:{};agrs2:{};args3:{} 的info级别日志", new Object[] { "args1", "args2", "args3",
                new IOException("测试抛出IO异常信息") });
        logger.info("三个参数:agrs1:{};agrs2:{};args3:{} 的info级别日志", new Object[] { "args1", "args2",
                new IOException("测试抛出IO异常信息") });

    }

}
```

对应输出日志信息：

	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:20)] 纯字符串信息的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:21)] 一个参数:agr1的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:22)] 二个参数:agrs1:args1;agrs2:args2的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:24)] 三个参数:agrs1:args1;agrs2:args2;args3:args3 的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:25)] 三个参数:agrs1:args1;agrs2:args2;args3:args3 的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:27)] ======================异常相关====================================
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:29)] 抛出异常,e:
	java.io.IOException: 测试抛出IO异常信息
		at com.qunar.dubbo.LogTest.main(LogTest.java:29) [classes/:na]
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.7.0_45]
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57) ~[na:1.7.0_45]
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_45]
		at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_45]
		at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120) [idea_rt.jar:na]
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:31)] 二个参数:agrs1:args1;agrs2:java.io.IOException: 测试抛出IO异常信息的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:32)] 二个参数:agrs1:args1;agrs2:args2的info级别日志
	java.io.IOException: 测试抛出IO异常信息
		at com.qunar.dubbo.LogTest.main(LogTest.java:32) [classes/:na]
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.7.0_45]
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57) ~[na:1.7.0_45]
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_45]
		at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_45]
		at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120) [idea_rt.jar:na]
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:35)] 三个参数:agrs1:args1;agrs2:args2;args3:java.io.IOException: 测试抛出IO异常信息 的info级别日志
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:36)] 三个参数:agrs1:args1;agrs2:args2;args3:agrs3 的info级别日志
	java.io.IOException: 测试抛出IO异常信息
		at com.qunar.dubbo.LogTest.main(LogTest.java:36) [classes/:na]
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.7.0_45]
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57) ~[na:1.7.0_45]
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_45]
		at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_45]
		at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120) [idea_rt.jar:na]
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:38)] 三个参数:agrs1:args1;agrs2:args2;args3:args3 的info级别日志
	java.io.IOException: 测试抛出IO异常信息
		at com.qunar.dubbo.LogTest.main(LogTest.java:38) [classes/:na]
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.7.0_45]
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57) ~[na:1.7.0_45]
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_45]
		at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_45]
		at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120) [idea_rt.jar:na]
	[2014-05-03 03:47:14 [34mINFO [0;39m com.qunar.dubbo.LogTest.main(LogTest.java:39)] 三个参数:agrs1:args1;agrs2:args2;args3:java.io.IOException: 测试抛出IO异常信息 的info级别日志

>> Note: 从代码调用可以看到，throwable 异常信息单独作为一个参数输入，因此，如果把异常信息作为`{}`占位符中的字符串，则会调用其对应toString方法，而无法打印异常堆栈信息。可以看看下面的截取源码：  

``` java
  public void info(String msg, Throwable t) {
    filterAndLog_0_Or3Plus(FQCN, null, Level.INFO, msg, null, t);
  }

    // 当日志的参数string 大于1，并且包含 Throwable类型参数，则调用下面的方法
  public void info(String format, Object[] argArray) {
    filterAndLog_0_Or3Plus(FQCN, null, Level.INFO, format, argArray, null);
  }
  
   /**
   * 在logback代码中，作者表明如果不使用Object[]创建参数数组对象，则会减少20 纳秒的时间开销。
   */
  private void filterAndLog_0_Or3Plus(final String localFQCN,
      final Marker marker, final Level level, final String msg,
      final Object[] params, final Throwable t) {

    final FilterReply decision = loggerContext
        .getTurboFilterChainDecision_0_3OrMore(marker, this, level, msg,
            params, t);

    if (decision == FilterReply.NEUTRAL) {
      if (effectiveLevelInt > level.levelInt) {
        return;
      }
    } else if (decision == FilterReply.DENY) {
      return;
    }

    buildLoggingEventAndAppend(localFQCN, marker, level, msg, params, t);// 在这个方法里面，会LoggingEvent方法构架日志信息，而对于Throwable非空时，则会创建一个ThrowableProxy对象，具体代码见下面。
  }


  
  // 下面的代码，是对日志中的异常打印信息。显然，在messageFormat里面，使用String来处理，是无法获得这么丰富的异常堆栈信息的。
  public ThrowableProxy(Throwable throwable) {
   
    this.throwable = throwable;
    this.className = throwable.getClass().getName();
    this.message = throwable.getMessage();
    this.stackTraceElementProxyArray = ThrowableProxyUtil.steArrayToStepArray(throwable
        .getStackTrace());
    
    //下面构建详细异常的堆栈信息，这也就是我们在代码输出时，看到的一大坨at... 输出错误代码位置等。
    Throwable nested = throwable.getCause();
    
    if (nested != null) {
      this.cause = new ThrowableProxy(nested);
      this.cause.commonFrames = ThrowableProxyUtil
          .findNumberOfCommonFrames(nested.getStackTrace(),
              stackTraceElementProxyArray);
    }
    if(GET_SUPPRESSED_METHOD != null) {
      // this will only execute on Java 7
      try {
        Object obj = GET_SUPPRESSED_METHOD.invoke(throwable);
        if(obj instanceof Throwable[]) {
          Throwable[] throwableSuppressed = (Throwable[]) obj;
          if(throwableSuppressed.length > 0) {
            suppressed = new ThrowableProxy[throwableSuppressed.length];
            for(int i=0;i<throwableSuppressed.length;i++) {
              this.suppressed[i] = new ThrowableProxy(throwableSuppressed[i]);
              this.suppressed[i].commonFrames = ThrowableProxyUtil
                  .findNumberOfCommonFrames(throwableSuppressed[i].getStackTrace(),
                      stackTraceElementProxyArray);
            }
          }
        }
      } catch (IllegalAccessException e) {
        // ignore
      } catch (InvocationTargetException e) {
        // ignore
      }
    }

  }

```

>> Note: 上面代码只是一般的步骤，对于调用`Object[]`形式的方法，则`ThrowableProxy`之前，还会对`Object[]`中的元素进行过滤处理，提取出最后一个元素判断是不是 `Throwable`类型的对象。代码参考如下：

``` java

 final public static FormattingTuple format(final String messagePattern,
      Object arg1, Object arg2) {
    return arrayFormat(messagePattern, new Object[] { arg1, arg2 });
  }

	//这方法会对输入参数进行特殊处理和过滤
 final public static FormattingTuple arrayFormat(final String messagePattern,
      final Object[] argArray) {

    Throwable throwableCandidate = getThrowableCandidate(argArray);

	......

	if (L < argArray.length - 1) {
      return new FormattingTuple(sbuf.toString(), argArray, throwableCandidate);// 如果元素中有Throwable类型，则size会减少，因此，对应的Throwale参数位置为 置提取出来的异常对象
    } else {
      return new FormattingTuple(sbuf.toString(), argArray, null);
    }

}



  static final Throwable getThrowableCandidate(Object[] argArray) {
    if (argArray == null || argArray.length == 0) {
      return null;
    }

    final Object lastEntry = argArray[argArray.length - 1];
    if (lastEntry instanceof Throwable) {
      return (Throwable) lastEntry;
    }
    return null;
  }
```

## <a id="End">后记</a>
slf4j的日志，打印抛出异常的信息时，如果只需要message，则需要在log api接口中的String 里面对应位置添加`{}`符号；否则，如果想要打印全量<font color="red">异常栈信息，则**不能也不可以**</font>在string字符串中添加`{}`，不然会大失所望。




