---
layout: post
categories:
  - Java
  - Spring
title:  "Spring 中实现动态数据源"
date: 2015-01-12 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

前段时间业务上有一个需求,这个需求需要查询数据库,由于单表数据比较大，导致出现超过5s的慢查询。随后，为了快速修复慢查询对整个系统带来的影响，将查询的数据源通过简单粗暴的修改配置切换到从库上。此后，增加了memcached来缓存一些case下的查询数据，但是对从库配置实现方式，并没有去调整。

最近，有其他业务的数据查询也需要切换到从库上，因此对上述简单的配置实现进行了思考。

动态数据源，其实就是根据我们的代码实现和配置来选择不同的数据源进行sql操作。一般地，我们会把读操作移到从库中，从而减轻主库的压力，也就是所谓的读写分离。

当然，对于一些使用数据库中间件来完成读写分离，而不需要业务层来做。这种方式，在大互联网公司中大量使用，比如360基于mysql-proxy的Atlas，阿里的DRDS(基于淘宝之前开源的TDDL）以及网易的分布式数据库中间件DDB等等。

对于一些未使用部署数据库中间件的公司，简单的方法就是在代码里面使用AOP方式通过对每个DAO层sql请求进行配置，来完成自定义的动态数据源。

## <a id="AbstractRoutingDataSource">Spring动态数据源接口</a>

Spring提供了一个抽象类`AbstractRoutingDataSource`，该类可以让开发人员快速实现数据源路由完成根据不同请求使用不同数据源的需求。

`AbstractRoutingDataSource`抽象类，继承关系如下图：

<img src="/images/2015/01/ards.png" />

> Notes: 抽象类最终继承`javax.sql.DataSource`类，该数据源类提供的一些接口就是我们最终需要实现的。
> 

DataSource接口主要提供了两个方法给开发者实现，因此实现动态数据源，我们只需要把这两个方法的实现，在调用数据库查询的时候，告知执行上下文，运行环境拿到对应的数据库连接，就可以连接到对应的数据库进行查询更新等操作。

``` java

public interface DataSource  extends CommonDataSource,Wrapper {

  Connection getConnection() throws SQLException;

  Connection getConnection(String username, String password)
    throws SQLException;

}

```

<!-- more -->

解析来，需要分析Spring提供给我们的抽象数据源路由类。既然`AbstractRoutingDataSource`简化了大家实现动态数据源功能的开发工作，那么该类必然会实现DataSource的两个接口方法。其需要决定，在什么情况下，使用哪个数据源的Connection连接。

> `AbstractRoutingDataSource`怎样来获取数据源连接呢？

使用Map数据结构存放所有配置中使用的数据源，value是数据源DataSource对象，key则是根据我们自己的爱好来取名的，比如：master，slave等。这样，我们可以根据具体Dao方法配置的数据源key来获取对应的DataSource对象，从而告知运行环境该sql查询使用哪一个connection连接。

下面给出`AbstractRoutingDataSource`的部分实现：

``` java
/**
 * 抽象的javax.sql.DataSource实现，可以完成基于一个查找key来路由 #getConnection()到某些特性目标DataSourcesd的一个。
 * 一般通过绑定线程事务上下文来决定。
 *
 */
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {

private Map<Object, Object> targetDataSources;

    private Object defaultTargetDataSource;

    private boolean lenientFallback = true;

    private DataSourceLookup dataSourceLookup = new JndiDataSourceLookup();

    private Map<Object, DataSource> resolvedDataSources;

    private DataSource resolvedDefaultDataSource;


    /**
     * 设置目标DataSources的map映射，其中查找key作为 map的key。
     * 这个映射的value可以是对象的DataSource实例，或者是一个数据源 name的字符串（可以被DataSourceLookup解析）。
     *
     * key可以是任意的类型，只要实现了普通的查找处理。
     * 具体的key表示形式，将会被resolveSpecifiedLookupKey和determineCurrentLookupKey处理
     *
     */
    public void setTargetDataSources(Map<Object, Object> targetDataSources) {
        this.targetDataSources = targetDataSources;
    }

    /**
     * 设置默认目标数据源。如果我们在map中找不到对应的key时，则会使用这里设置的默认数据源
     */
    public void setDefaultTargetDataSource(Object defaultTargetDataSource) {
        this.defaultTargetDataSource = defaultTargetDataSource;
    }

    /**
     * 指定默认的DataSource，当通过指定的查找key不能找到对应的DataSource。
     * 如果为false，则直接返回失败，如果为true，则使用默认的数据源。默认为true
     */
    public void setLenientFallback(boolean lenientFallback) {
        this.lenientFallback = lenientFallback;
    }

    /**
     * 设置DataSourceLookup的实现类，该实现类可以把字符串配置的数据源，解析成我们需要的DataSource类.默认使用JndiDataSourceLookup。
     *
     * JndiDataSourceLookup方法使用ref bean方式获取配置文件中配置的dataSource数据源，也就是我们一般使用xml中配置datasource的方式就是jndi。
     */
    public void setDataSourceLookup(DataSourceLookup dataSourceLookup) {
        this.dataSourceLookup = (dataSourceLookup != null ? dataSourceLookup : new JndiDataSourceLookup());
    }


    public void afterPropertiesSet() {
        if (this.targetDataSources == null) {
            throw new IllegalArgumentException("Property 'targetDataSources' is required");
        }
        this.resolvedDataSources = new HashMap<Object, DataSource>(this.targetDataSources.size());
        for (Map.Entry entry : this.targetDataSources.entrySet()) {
            Object lookupKey = resolveSpecifiedLookupKey(entry.getKey());
            DataSource dataSource = resolveSpecifiedDataSource(entry.getValue());
            this.resolvedDataSources.put(lookupKey, dataSource);
        }
        if (this.defaultTargetDataSource != null) {
            this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
        }
    }

    /**
     * 根据lookupKey获取map中存放的key值，一般无特性情况，两者是一样的
     */
    protected Object resolveSpecifiedLookupKey(Object lookupKey) {
        return lookupKey;
    }
    /**
     * 转换从获取map中存放的dataSource
     */
    protected DataSource resolveSpecifiedDataSource(Object dataSource) throws IllegalArgumentException {
        if (dataSource instanceof DataSource) {
            return (DataSource) dataSource;
        }
        else if (dataSource instanceof String) {
            return this.dataSourceLookup.getDataSource((String) dataSource);
        }
        else {
            throw new IllegalArgumentException(
                    "Illegal data source value - only [javax.sql.DataSource] and String supported: " + dataSource);
        }
    }

    // 这里就是抽象类给我们实现的接口方法，根据我们的配置上下文，抽象类决定实现哪个连接
    public Connection getConnection() throws SQLException {
        return determineTargetDataSource().getConnection();
    }

    public Connection getConnection(String username, String password) throws SQLException {
        return determineTargetDataSource().getConnection(username, password);
    }

    protected DataSource determineTargetDataSource() {
        Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
        Object lookupKey = determineCurrentLookupKey();
        DataSource dataSource = this.resolvedDataSources.get(lookupKey);
        if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
            dataSource = this.resolvedDefaultDataSource;
        }
        if (dataSource == null) {
            throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
        }
        return dataSource;
    }

    //这里是我们使用这个抽象类需要实现的方法，主要就是告诉该抽象类，当前需要使用的数据源的key是什么，这样抽象类就可以知道使用哪个数据库连接
    protected abstract Object determineCurrentLookupKey();
}
```

## <a id="Implementation">基于Spring动态数据源实现</a>

### 3.1 实现抽象路由数据源类

上一节介绍了抽象类`AbstractRoutingDataSource`，继承这个抽象类，我们实现动态数据源，只需要告诉抽象类，当前使用哪个key去获取数据源（determineCurrentLookupKey）。

在项目中，我们一般会指定哪些数据库操作需要使用哪个数据源，这个设置会存放在上下文中。也就是，我们可以使用ThreadLocal来存放当前数据操作使用的key。

因此，可以实现两个类，一个类实现`AbstractRoutingDataSource`抽象接口；一个来获取当前上下文中对应的key值。代码如下：

``` java
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {

        return DynamicDataSourceHolder.getDataSource();
    }
}

public class DynamicDataSourceHolder {

    private static final ThreadLocal<String> dsHolder = new ThreadLocal<String>();

    public static String getDataSource() {
        return dsHolder.get();
    }

    public static void putDataSource(String value) {
        dsHolder.set(value);
    }

    public static void clear(){
        dsHolder.remove();
    }
}

```

通过上面两个类，我们就可以从上下文中获取当前操作需要使用的key值，然后通过实现的抽象路由数据源类来找到配置的DataSource，这样spring上下文就知道具体使用哪个connection连接来操作数据库sql了。

> Tips: 这里需要注意ThreadLocal类中实现了clear方法，主要是在一个线程中会存在多个sql操作，可能设计不同的数据源，如果不清除当前sql的数据源，可能接下来的sql操作也会使用前一个操作设置的数据源连接，导致错误。

### 3.2 实现AOP简化配置

上一小节完成了怎样从上下文中获取设置的key，从而使用哪个数据源连接。但是，如何告知哪些操作使用哪个数据源key。

我们可以在每个需要使用动态数据源的地方，在具体业务代码的开始，把key值put到线程上下文中；然后在业务代码结束的地方，把上下文的设置清除掉。这样可以完成我们的需求，但是，对业务代码的侵入程度有点大哦。

上述这种场景，非常适合使用AOP技术完成。关于AOP介绍，可以参考：<http://oss.org.cn/ossdocs/framework/spring/zh-cn/aop.html>

采用AOP技术，我们需要在配置文件（或注解方式）中设置切点pointCut，然后我们需要实现切点前调用的方法（threadLocal中存入数据源key），和切点后调用的方法（清除threadLocal数据）。

为了使用方便，我们使用注解的方式配置数据源key。注解的实现代码：

``` java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface DBSource {

    /**
     * 指定数据源使用哪个配置
     */
    String value();
}

```

接下来，看看怎样实现AOP的before和after通知方法：

``` java
public class DataSourceAspect {

    private static final Logger logger = LoggerFactory.getLogger(DataSourceAspect.class);

    public void before(JoinPoint point) {

        Object target = point.getTarget();
        String method = point.getSignature().getName();

        Class<?>[] classz = target.getClass().getInterfaces();

        Class<?>[] parameterTypes = ((MethodSignature) point.getSignature()).getMethod().getParameterTypes();
        try {
            Method m = classz[0].getMethod(method, parameterTypes);
            if (m != null && m.isAnnotationPresent(DBSource.class)) {
                DBSource data = m.getAnnotation(DBSource.class);
                DynamicDataSourceHolder.putDataSource(data.value());
                logger.info("-------数据源：{}------",data.value());
            }

        } catch (Exception e) {
            logger.error("=======================AOP注册拦截失败了！",e);
        }
    }

    public void after(JoinPoint point){

        Object target = point.getTarget();
        String method = point.getSignature().getName();

        Class<?>[] classz = target.getClass().getInterfaces();

        Class<?>[] parameterTypes = ((MethodSignature) point.getSignature()).getMethod().getParameterTypes();
        try {
            Method m = classz[0].getMethod(method, parameterTypes);
            if (m != null && m.isAnnotationPresent(DBSource.class)) {
                DynamicDataSourceHolder.clear();
                logger.info("-------清除ThreadLocal------");
            }

        } catch (Exception e) {
            logger.error("=======================AOP注册拦截失败了！",e);
        }
    }
}

```

这样，我们接下来，只需要在xml文件中配置相关切点和通知方法，即完成了整个动态数据源功能。

``` xml
 <!-- 配置数据库注解aop -->
    <aop:aspectj-autoproxy/>
    <!-- 配置数据库注解aop -->
    <bean id="dataSourceAspect" class="io.github.ketao1989.simple.service.dataSource.DataSourceAspect"/>
    <aop:config>
        <aop:aspect id="dsa" ref="dataSourceAspect">
            <aop:pointcut id="pc" expression="execution(* io.github.ketao1989.dao.*.*(..))"/>
            <aop:before pointcut-ref="pc" method="before"/>
            <aop:after pointcut-ref="pc" method="after"/>
        </aop:aspect>
    </aop:config>

```

## <a id="End">后记</a>

本文的代码和spring源码注释可以在github上查看：

Spring源码注释：<https://github.com/ketao1989/cnSpring>

spring 动态数据源项目：<https://github.com/ketao1989/simpleSpringProject>

最后，本文借鉴参考了博客园中的一篇博客：[spring实现数据库读写分离](http://www.cnblogs.com/xiyangyang/p/3580625.html)。感谢！
