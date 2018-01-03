---
layout: post
categories: 
  - Java 
  - Spring
title:  "Spring MVC 项目构建入门指南"
date: 2014-05-26 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

走入社会工作接近一年了,感慨颇多.一年前的现在,自己对java知之甚少,更不知道怎么去创建一个满足`spring mvc`架构思想的web项目。最近，为了学习`Spring MVC`框架的实现原理，首先新建了一个web项目，该项目包含 `Spring + Mybatis`涉及数据库，DAO，service，API，业务，controller多个模块。

>> Note：这个项目代码完成的功能只是为了演示一个大项目应该具备的结构，而显然在实际工程中，这么简单地功能，是不需要如此繁杂的结构模块的。

此外，关于`Spring MVC`演示项目的代码，可以从`github`上`clone`一份到本地，项目为：[SimpleSpringProject](https://github.com/ketao1989/simpleSpringProject)。`git clone` 地址为：`https://github.com/ketao1989/simpleSpringProject.git`


## <a id="Framework">SimpleSpringProject 结构介绍</a>

`simpleSpringProject`项目各个模块分工明确，主要由8个模块组成：
	
	1. simple-spring-main模块：该模块主要是提供给外界访问的controller层所在。`controller`层可以对外提供html视图，也可以对外提供RPC调用，例如 alibaba 的 dubbo接口。

	2. simple-spring-api模块：该模块主要就是封装一些底层的方法接口给外部使用。比如，如果我们使用RPC接口调用服务，则只需要API包就可以了，具体实现调用方是不需要知道的。
	
	3. simple-spring-biz模块：该模块是业务模块，也就是具体业务需要的方法基本上都是在这里实现的。在demo 中，在这一层实现api提供的接口。该模块主要调用service层提供的基本服务，组装起来，实现各种不同的业务逻辑接口。
	
	4. simple-spring-service模块：该模块是基础服务模块，该模块会为biz业务模块提供基本的服务，这些服务功能都比较简单，业务逻辑单一。因此，在这一层进行单元测试，一般会取到比较好的效果。
	
	5. simple-spring-dao模块：该模块是数据库相关接口模块。`Mybatis`可以把interface 和 xml结合起来，使得开发者可以把数据库表中相关操作集成在 interface 代码中，而具体的`SQL`实现则写在xml文件中。这样子，可以让整个结构更清晰。
	
	6. simple-spring-config模块：该模块就是相关`sql`语句的配置文件所在地。一般地，会根据interface 来切分不同的配置文件，两种的关联关系是通过对于的`sql.xml`文件中`mapper namespace`来关联。
	
	7. simple-spring-common模块：该模块一般存放一些项目公用的工具类和常量值。比如，一些配置文件中需要配置的属性值，一些xxxUtils类实现等。
	
	8. simple-spring-model模块：该模块主要提供一些模型，各个类需要使用的对象。比如，我们需要获取一个学生个体信息，显然会作为一个对象类来实现。
	
>> Note：显然，对于各个模块的具体详细分工，其实还是可以调整的，比如可能有些地方会在`service`层来做稍微复杂的服务实现，而在`biz`层则稍微组合就可以了。这里，demo的模块分类，只是正常情况下，业务规模有一些大的情况下，才会进行多个模块的分工。

<!-- more -->

## <a id="Configure">SimpleSpringProject 配置介绍</a>

`Spring`作为一个主打配置的框架，配置文件的位置，十分重要。如果，一些配置文件的一个符号错误，都会导致整个`MVC` web 项目无法正常启动。

>> 首先，来看看web项目中，重中之重的配置文件`web.xml`，

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         id="WebApp_ID" version="2.5">

    <!--启动配置文件设置-->
    <display-name>simple-spring-main</display-name>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- 请求地址匹配映射 -->

    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>*.htm</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>*.json</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>*.xml</url-pattern>
    </servlet-mapping>

    <!-- 编码过滤器 -->
    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>*.htm</url-pattern>
    </filter-mapping>

    <!-- 指定404页面 -->
    <error-page>
        <error-code>404</error-code>
        <location>/404.html</location>
    </error-page>


```

>> 接下来默认`springmvc-servlet.xml`文件和`applicationContext.xml`文件。`springmvc-servlet.xml`配置文件，主要定义servlet相关配置，比如scan 基本包名，视图velocity配置，jsp配置设置等信息。由于这里只有一个 mapping servlet，所以只有一个对应的配置文件，当然也可以把这个文件放置其他地方，然后再`web.xml`中定义对于的servlet就可以了。`applicationContext.xml`配置文件，则是整个项目的公共配置，比如指定数据库配置，连接池相关信息，一些spring bean 注册信息，默认的视图解析器等等。

-

>> 接下来，就是`mybatis.xml`配置文件，这个配置主要是针对`Mybatis`而存在的，其指定项目中`Mybatis`设置，以及一些`typeHandler`，`mapper`实现的位置。下面给出demo中的配置示例：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <settings>
        <!-- 这个配置使全局的映射器启用或禁用缓存 -->
        <setting name="cacheEnabled" value="false"/>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        <!-- 全局启用或禁用延迟加载。当禁用时，所有关联对象都会即时加载 -->
        <!-- <setting name="lazyLoadingEnabled" value="true"/> -->
        <!-- 当启用时，有延迟加载属性的对象在被调用时将会完全加载任意属性。否则，每种属性将会按需要加载 -->
        <!-- <setting name="aggressiveLazyLoading" value="true"/> -->
        <!-- 允许或不允许多种结果集从一个单独的语句中返回（需要适合的驱动） -->
        <!-- <setting name="multipleResultSetsEnabled" value="true"/> -->
        <!-- 使用列标签代替列名。不同的驱动在这方便表现不同。参考驱动文档或充分测试两种方法来决定所使用的驱动 -->
        <!-- <setting name="useColumnLabel" value="true"/> -->
        <!-- 允许JDBC支持生成的键。需要适合的驱动。如果设置为true则这个设置强制生成的键被使用，尽管一些驱动拒绝兼容但仍然有效（比如Derby） -->
        <setting name="useGeneratedKeys" value="true"/>
        <!-- 指定MyBatis如何自动映射列到字段/属性。PARTIAL只会自动映射简单，没有嵌套的结果。FULL会自动映射任意复杂的结果（嵌套的或其他情况） -->
        <!-- <setting name="autoMappingBehavior" value="PARTIAL"/> -->
        <!-- 配置默认的执行器。SIMPLE执行器没有什么特别之处。REUSE执行器重用预处理语句。BATCH执行器重用语句和批量更新 -->
        <setting name="safeRowBoundsEnabled" value="false"/>
        <setting name="defaultExecutorType" value="REUSE"/>
        <!-- 设置超时时间，它决定驱动等待一个数据库响应的时间 -->
        <setting name="defaultStatementTimeout" value="600"/>
    </settings>

    <mappers>
        <mapper resource="mapper/version.xml"/>
    </mappers>

</configuration>
```

-

>> 最后，介绍下`pom.xml`文件，这个实际上，不算是MVC 所仅有的。但是，作为一个maven项目，很多配置都比较关键，一般模块的`pom.xml`都比较简单，但是`	main`模块由于涉及编译成war包，并且针对不同的运行环境，对应打包的配置文件不同，因此，其内部配置会比较复杂。具体参考demo项目代码。


## <a id="DataOp">SimpleSpringProject 数据操作实现介绍</a>

整个项目代码都非常简单，不需要过多的去说明。在这里，对于初学者，需要介绍下，数据库相关的访问代码实现逻辑。

>> 首先，定义一个接口，该接口里面会声明一些需要在	`sql`中去实现的方法名，如下所示：

``` java
/**
 * @author: tao.ke Date: 14-5-26 Time: 上午11:28
 * @version: \$Id$
 */
public interface VersionDao {

    /**
     * 根据id 查询对应的version信息
     * 
     * @param id
     * @return
     */
    Version queryVersionById(@Param("id") int id);

    /**
     * 查询所有的版本信息
     * 
     * @return
     */
    List<Version> queryAllVersions();

}

```

>> 当`dao`模块则存在了需要实现的接口，则接下来可以在`config`模块中去实现它，具体实现，如下所示：

``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//ibatis.apache.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="io.github.ketao1989.dao.VersionDao">
    <resultMap id="VersionMap" type="io.github.ketao1989.model.Version">
        <id column="id" property="id"/>
        <result column="id" property="id"/>
        <result column="version_num" property="versionNum"/>
        <result column="description" property="description"/>
        <result column="update_time" property="updateTime"/>
    </resultMap>

    <sql id="versionColumn">
        <trim suffix="" suffixOverrides=",">
            <if test="versionNum != null">version_num,</if>
            <if test="description != null">description,</if>
            <if test="updateTime != null">update_time</if>
        </trim>
    </sql>
    <sql id="versionValue">
        <trim suffix="" suffixOverrides=",">
            <if test="versionNum != null">#{versionNum},</if>
            <if test="description != null">#{description},</if>
            <if test="updateTime != null">#{updateTime}</if>
        </trim>
    </sql>

    <!-- 根据id查询Version记录 -->
    <select id="queryVersionById" parameterType="map" resultMap="VersionMap">
          select id,version_num,description,update_time
          from version
          where id = #{id}
    </select>

    <!-- 查询所有Version记录 -->
    <select id="queryAllVersions"  resultMap="VersionMap">
          select id,version_num,description,update_time
          from version
    </select>
</mapper>

```

>> Note：这样在`service`模块调用 `dao`模块的接口，就可以操作数据库了。当然，你在`applicationContext.xml`中需要配置下面一段代码：

``` xml
<!-- 创建SqlSessionFactory -->
    <bean id="sqlSessionFactory" name="sqlSessionFactory"
          class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <property name="configLocation" value="classpath:mybatis-config.xml" />
    </bean>

    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="io.github.ketao1989.dao" />
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>

```


## <a id="Finally">小结</a>

本文只是简单介绍了下`Spring MVC`项目的一些实现注意点，具体的代码实现，请参考本github上的[SimpleSpringProject](https://github.com/ketao1989/simpleSpringProject) 。在demo代码的实现里面，如果你是一个初学者，你会发现更多需要注意的地方。如果你对于本项目的各个地方都能理解，并且可以仿照新建一个项目，那么你对于`Spring MVC`就已经入门了，可以深入框架源码来进一步学习了。

当然，作为一个提供初学者使用的`Spring MVC` web工程项目，该demo只是供学习使用而已。你也可以继续在该demo上扩展，增加更多地类，更多地业务和功能，从而完成一个商业大项目。
