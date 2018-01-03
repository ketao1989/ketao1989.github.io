---
layout: post
title:  "Dubbo provider 多组名注册问题分析"
date: 2014-10-22 22:21:35 +0800
comments: true
categories: 
  - Java
  - Dubbo
---

* content
{:toc}

## <a id="Intro">前言</a>

最近线上遇到一个很奇怪的现象，就是一个dubbo 服务被注册到了好几个`group`下面，并且这些`group`都是我们应用中，通过`dubbo:registry`来配置的。但是，显然这不是我们应用所期待的结果，因此，首先，我们需要修复这个问题；其次，我们需要找出原因。

### 1.1 问题定位

首先看配置：

``` xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <dubbo:registry id="crm-message-registry" group="crm-message"
                    address="${qunar-public.zookeeper.address}" protocol="zookeeper"/>

    <dubbo:protocol name="dubbo" port="${dubbo.protocol} "/>

    <dubbo:service interface="com.qxx.scm.message.api.IPushMessageBiz"
                   ref="pushMessageBiz" timeout="6000" version="1.0.1" />
                   
    </beans>
    
```

从上面的配置可以看到，我们实际上是有`dubbo:registry`，并且在其中也设置了`group`属性。

<!-- more -->

接下来，拿该xml配置和以前的 `dubbo provider`的配置进行比较，发现在`dubbo:service`的属性配置里面，缺少了指定`registry`的配置，猜想应该是该配置的缺失，导致应用把该`service`对应的服务，注册到了多个group下面。因此，为了正式推测，更改配置如下：

``` xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <dubbo:registry id="crm-message-registry" group="crm-message"
                    address="${qxx-public.zookeeper.address}" protocol="zookeeper"/>

    <dubbo:protocol name="dubbo" port="${dubbo.protocol} "/>
    
    <dubbo:service interface="com.qxx.scm.message.api.IPushMessageBiz"
                   ref="pushMessageBiz" timeout="6000" version="1.0.1" registry="crm-message-registry"/>
 
    </beans>
    
```

启动应用，发现果然问题解决了。

### 1.2 问题产生原因

在`dubbo:service`没有明确设置`registry`，不会导致启动失败，或者注册不上，而是会在应用配置中声明的所有组名下面注册对应服务。于是，去dubbo的github：<http://alibaba.github.io/dubbo-doc-static/Developer+Guide-zh.htm>  上去看看wiki有木有说明。关于注册说明，如：<http://alibaba.github.io/dubbo-doc-static/RegistryFactory+SPI-zh.htm>.

其中，文档关于配置说明如下：

``` xml

<dubbo:registry id="xxx1" address="xxx://ip:port" /> <!-- 定义注册中心 -->
<dubbo:service registry="xxx1" /> <!-- 引用注册中心，如果没有配置registry属性，将在ApplicationContext中自动扫描registry配置 -->
<dubbo:provider registry="xxx1" /> <!-- 引用注册中心缺省值，当<dubbo:service>没有配置registry属性时，使用此配置 -->

```

ok，到这里就明白了，`dubbo`在我们配置`<dubbo:service>`中没有指定registry时，会看看对应的`<dubbo:provider>`有木有配置`registry`,如果没有，则*在ApplicationContext中自动扫描registry配置*获取注册中心列表，然后进行注册。

## <a id="Registry">dubbo注册分析</a>

`dubbo`是阿里巴巴公司开源的一个高性能优秀的服务框架，使得应用可通过高性能的 RPC 实现服务的输出和输入功能，可以和 Spring框架无缝集成。当然，作为一个优秀的RPC框架，其涉及到服务注册和服务事件的订阅及发布。

### 2.1 dubbo配置解析

在Spring 中使用dubbo，会采用xml方式对dubbo相关属性进行配置，其加入了自己的命名空间，如`xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"`，因此，你会知道必然存在一个继承实现`org.springframework.beans.factory.xml.NamespaceHandlerSupport`来完成自定义配置的解析工作，部分代码如下：

``` java

public class DubboNamespaceHandler extends NamespaceHandlerSupport {

	static {
		Version.checkDuplicate(DubboNamespaceHandler.class);
	}

	public void init() {
	    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
    }

}

```

可以看到`RegistryConfig.class`类，包含关于`注册中心`相关的属性变量配置；`ServiceBean.class`类，包含关于`对外服务`相关的属性变量配置。也就是说，通过`DubboBeanDefinitionParser`解析器会把对应xml节点解析成对应的对象实例。

Spring的自定义标签代码实现和内部原理，可以google一下。

>> 说明：`DubboNamespaceHandler`在初始化的时候，会把所有，针对不同xml节点的对应解析其注册到Spring `NamespaceHandlerSupport`的 `BeanDefinitionParser` Map上来。这样，在Spring 初始化解析xml配置时，就可以完成对自定义标签的兼容和实例化了。

对于Dubbo 自定义解析器`DubboBeanDefinitionParser`的说明，如[附录](#End)所示。

### 2.2 dubbo注册解析

阿里的dubbo提供了多种注册机制，比如：`Redis`注册中心，`ZooKeeper`注册中心，或者使用`广播`的方式等。而目前使用的比较多的是使用`ZookeeperRegistry`方式。

从代码结构可以看出，对于不同的注册机制，其主要流程还是一样的，只是在注册url的具体操作中，需要根据不同的部署实体来采用不同的注册操作。比如，对于`zookeeper`，则使用`zkclient`去连接zk注册中心；而对于`redis`，则直接使用`redis.clients.jedis.Jedis`在数据库上注册保持对应url。

在抽象类的`AbstractRegistryFactory`中，提供了连接注册中心的一些方法：

``` java

    // 注册中心集合 Map<RegistryAddress, Registry>
    private static final Map<String, Registry> REGISTRIES = new ConcurrentHashMap<String, Registry>();

    /**
     * 获取所有注册中心
     * 
     * @return 所有注册中心
     */
    public static Collection<Registry> getRegistries() {

        // 这里获取应用中配置的所有注册中心，如果dubbo:service里面没有配置registry属性时，应该会调用这个方法
        return Collections.unmodifiableCollection(REGISTRIES.values());
    }
    
     /**
     * 连接注册中心
     * @param url 注册中心地址，不允许为空
     * @return
     */
    public Registry getRegistry(URL url) {
    	url = url.setPath(RegistryService.class.getName())
    			.addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
    			.removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    	String key = url.toServiceString();
        // 锁定注册中心获取过程，保证注册中心单一实例
        LOCK.lock();
        try {
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            // 这里会把本应用所有不同的成功连接的注册配置放在全局concurrentMap中
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // 释放锁
            LOCK.unlock();
        }
    }

    protected abstract Registry createRegistry(URL url);

```

>> Note：在Factory中会创建Registery对象，该对象中，会实现注册，取消注册，订阅，取消订阅等方法来提供给`service`和`reference`完成需要的注册和订阅服务。

``` java

public interface RegistryService {

    /**
     * 注册数据，比如：提供者地址，消费者地址，路由规则，覆盖规则，等数据。
     * 
     * 注册需处理契约：<br>
     * 1. 当URL设置了check=false时，注册失败后不报错，在后台定时重试，否则抛出异常。<br>
     * 2. 当URL设置了dynamic=false参数，则需持久存储，否则，当注册者出现断电等情况异常退出时，需自动删除。<br>
     * 3. 当URL设置了category=routers时，表示分类存储，缺省类别为providers，可按分类部分通知数据。<br>
     * 4. 当注册中心重启，网络抖动，不能丢失数据，包括断线自动删除数据。<br>
     * 5. 允许URI相同但参数不同的URL并存，不能覆盖。<br>
     * 
     * @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void register(URL url);

    /**
     * 取消注册.
     * 
     * 取消注册需处理契约：<br>
     * 1. 如果是dynamic=false的持久存储数据，找不到注册数据，则抛IllegalStateException，否则忽略。<br>
     * 2. 按全URL匹配取消注册。<br>
     * 
     * @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void unregister(URL url);

    /**
     * 订阅符合条件的已注册数据，当有注册数据变更时自动推送.
     * 
     * 订阅需处理契约：<br>
     * 1. 当URL设置了check=false时，订阅失败后不报错，在后台定时重试。<br>
     * 2. 当URL设置了category=routers，只通知指定分类的数据，多个分类用逗号分隔，并允许星号通配，表示订阅所有分类数据。<br>
     * 3. 允许以interface,group,version,classifier作为条件查询，如：interface=com.alibaba.foo.BarService&version=1.0.0<br>
     * 4. 并且查询条件允许星号通配，订阅所有接口的所有分组的所有版本，或：interface=*&group=*&version=*&classifier=*<br>
     * 5. 当注册中心重启，网络抖动，需自动恢复订阅请求。<br>
     * 6. 允许URI相同但参数不同的URL并存，不能覆盖。<br>
     * 7. 必须阻塞订阅过程，等第一次通知完后再返回。<br>
     * 
     * @param url 订阅条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件监听器，不允许为空
     */
    void subscribe(URL url, NotifyListener listener);

    /**
     * 取消订阅.
     * 
     * 取消订阅需处理契约：<br>
     * 1. 如果没有订阅，直接忽略。<br>
     * 2. 按全URL匹配取消订阅。<br>
     * 
     * @param url 订阅条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件监听器，不允许为空
     */
    void unsubscribe(URL url, NotifyListener listener);

    /**
     * 查询符合条件的已注册数据，与订阅的推模式相对应，这里为拉模式，只返回一次结果。
     * 
     * @see com.alibaba.dubbo.registry.NotifyListener#notify(List)
     * @param url 查询条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @return 已注册信息列表，可能为空，含义同{@link com.alibaba.dubbo.registry.NotifyListener#notify(List<URL>)}的参数。
     */
    List<URL> lookup(URL url);

}

```

## <a id="Service">dubbo服务分析</a>

从上面的解析配置文件，可以看到对于`<dubbo:service>`的解析，是使用`serviceBean`类来实现的。在`serviceConfig`（serviceBean的父类）类中对于service需要的一些属性进行说明，但是对于一些必须的属性没有设置会怎么样呢？！dubbo，首先会去上下文中，通过查找构造出对应的属性；当然如果构造不了，则返回null，这样在后期的时候就会抛出异常。

### 3.1 dubbo服务配置处理分析

对于一些容错处理，也就是未配置的自动完善的方法，在`serviceBean`中完成。部分代码如下：

``` java

/**
     * 应用事件监听器ApplicationListener的接口方法，用来对监听到的事件进行处理。
     *
     * 显然的观察者设计模式。事件驱动。
     *
     * @param event
     */
    public void onApplicationEvent(ApplicationEvent event) {
        if (ContextRefreshedEvent.class.getName().equals(event.getClass().getName())) {
            if (isDelay() && !isExported() && !isUnexported()) {
                if (logger.isInfoEnabled()) {
                    logger.info("The service ready on spring started. service: " + getInterface());
                }
                export();// 暴露服务
            }
        }
    }
    
     /**
     * 这个方法很重要！
     *
     * 这里会构造 服务提供方相关的服务设置
     *
     * @throws Exception
     */
    @SuppressWarnings({ "unchecked", "deprecation" })
    public void afterPropertiesSet() throws Exception {

        // 在dubbo:service中没有配置Provider属性
        if (getProvider() == null) {

            // 从applicationContext中构造ProviderConfig类对象
            Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils
                    .beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
            if (providerConfigMap != null && providerConfigMap.size() > 0) {
                // 从applicationContext中构造ProtocolConfig类对象
                Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils
                        .beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);

                // 获取应用上下文中的所有providerConfig，然后保存到对应的service服务的protocol中
                if ((protocolConfigMap == null || protocolConfigMap.size() == 0) && providerConfigMap.size() > 1) { // 兼容旧版本
                    List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                    for (ProviderConfig config : providerConfigMap.values()) {
                        if (config.isDefault() != null && config.isDefault().booleanValue()) {
                            providerConfigs.add(config);
                        }
                    }
                    if (providerConfigs.size() > 0) {
                        setProviders(providerConfigs);
                    }
                } else {
                    ProviderConfig providerConfig = null;

                    // 难以详细这个逻辑，for多次竟然只set一个变量，多个则又抛异常
                    for (ProviderConfig config : providerConfigMap.values()) {
                        if (config.isDefault() == null || config.isDefault().booleanValue()) {// 缺省状态，多次设置，则抛异常？？
                            if (providerConfig != null) {
                                throw new IllegalStateException("Duplicate provider configs: " + providerConfig
                                        + " and " + config);
                            }
                            providerConfig = config;
                        }
                    }
                    if (providerConfig != null) {
                        setProvider(providerConfig);
                    }
                }
            }
        }

        // 对Application为空进行判断，并且只有provider为空或者provider的application为空，才会从应用的上下文去构造
        if (getApplication() == null && (getProvider() == null || getProvider().getApplication() == null)) {
            Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils
                    .beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
            if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
                ApplicationConfig applicationConfig = null;
                for (ApplicationConfig config : applicationConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (applicationConfig != null) {
                            throw new IllegalStateException("Duplicate application configs: " + applicationConfig
                                    + " and " + config);
                        }
                        applicationConfig = config;
                    }
                }
                if (applicationConfig != null) {
                    setApplication(applicationConfig);
                }
            }
        }
        if (getModule() == null && (getProvider() == null || getProvider().getModule() == null)) {
            Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils
                    .beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
            if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
                ModuleConfig moduleConfig = null;
                for (ModuleConfig config : moduleConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (moduleConfig != null) {
                            throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and "
                                    + config);
                        }
                        moduleConfig = config;
                    }
                }
                if (moduleConfig != null) {
                    setModule(moduleConfig);
                }
            }
        }

        // ok,这里就是注册中心的配置处理，对于service中没有配置registry属性的，dubbo会自己来完善
        // 还记得上面的dubbo wiki文档的说明吗？会查看provider是否配置了registry，如果没有才会调用全局应用上下文的配置。
        // 所以这里，会首先判断service是否配置registry，然后判断provider是否配置，然后在判断application是否配置registry，
        // 然后才会从applicationContext中获取已经在spring 中注册了的registry列表，作为本service的配置。
        if ((getRegistries() == null || getRegistries().size() == 0)
                && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries()
                        .size() == 0)
                && (getApplication() == null || getApplication().getRegistries() == null || getApplication()
                        .getRegistries().size() == 0)) {
            Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils
                    .beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
            if (registryConfigMap != null && registryConfigMap.size() > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                for (RegistryConfig config : registryConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        registryConfigs.add(config);
                    }
                }
                // 这里设置到了注册中心配置中，list哦。
                if (registryConfigs != null && registryConfigs.size() > 0) {
                    super.setRegistries(registryConfigs);
                }
            }
        }
        
        // ....................
        
         if (!isDelay()) {
            // 不延迟，可以开始暴露服务了
            export();
        }
  }

```

具体，dubbo怎么把服务暴露出去，消费者怎样通过dubbo rpc调用服务等，不在本文所描述之内。后续会单独进行分析说明。

### 3.2 问题原因解析

从上面的代码逻辑，可以明白之所以出现多组注册的问题原因了。

回到代码中，在配置的dubbo xml文件里面，有三个不同的节点，`<dubbo:registry>`，`<dubbo:protocol>`和`<dubbo:service>`。由于`<dubbo:service>`中没有配置`registry`属性，所以按照dubbo的逻辑，会先查找`<dubbo:provider>`节点配置，但是因为配置文件中没有配置，所以接下来会查找`<dubbo:application>`节点，看看该节点下面是否有配置`registry`属性，依然没有配置该节点。

这样，就走进了dubbo来配置相关registry属性的代码内部。

代码会从spring的`applicationContext`中构造出`Map<String, RegistryConfig> `类型变量，然后获取`map.values()`从而拿到所有在spring中已经存在上下文中的注册信息了。这样，把结果（所有注册的registry信息）放到service配置中。

>> Note：那么，为什么可以获取应用上下文环境中所有的`RegistryConfig`呢？还记得解析顺序吗！？最开始就是解析注册节点的数据哦！`registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));`早于`registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));`，此外，判断逻辑里面，其他节点数据解析，也是早于serviceBean的。

## <a id="End">附录</a>

dubbo 关于`DubboBeanDefinitionParser` 部分核心代码的注释说明：

``` java

public class DubboBeanDefinitionParser implements BeanDefinitionParser {

    private static final Logger logger = LoggerFactory.getLogger(DubboBeanDefinitionParser.class);

    private final Class<?> beanClass;

    private final boolean required;

    public DubboBeanDefinitionParser(Class<?> beanClass, boolean required) {
        this.beanClass = beanClass;
        this.required = required;
    }

    public BeanDefinition parse(Element element, ParserContext parserContext) {
        return parse(element, parserContext, beanClass, required);
    }

    @SuppressWarnings("unchecked")
    private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass,
            boolean required) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
        // 解析 id标识
        String id = element.getAttribute("id");

        // 如果id标识为空，则获取name标识
        if ((id == null || id.length() == 0) && required) {
            String generatedBeanName = element.getAttribute("name");
            if (generatedBeanName == null || generatedBeanName.length() == 0) {

                // 如果使用ProtocolConfig 为bean ，则为dubbo，配置为dubbo:protocol
                if (ProtocolConfig.class.equals(beanClass)) {
                    generatedBeanName = "dubbo";
                } else {

                    // ok，这里是没有id和name标识，并且正常config bean，则使用interface的标识value值
                    // 比如，一般我们写dubbo:service 可能会走到这个配置上。
                    generatedBeanName = element.getAttribute("interface");
                }
            }
            if (generatedBeanName == null || generatedBeanName.length() == 0) {
                generatedBeanName = beanClass.getName();
            }
            id = generatedBeanName;
            int counter = 2;
            // 如果存在该BeanDefinition，则重置id
            while (parserContext.getRegistry().containsBeanDefinition(id)) {
                id = generatedBeanName + (counter++);
            }
        }

        // ok，这里再次判断，检查是否重复id在bean里面
        if (id != null && id.length() > 0) {
            if (parserContext.getRegistry().containsBeanDefinition(id)) {
                throw new IllegalStateException("Duplicate spring bean id " + id);
            }

            // 注册到spring 的bean Map中去，其中id为key
            parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
            // 然后把id和对应value 作为Property放入bean里面
            beanDefinition.getPropertyValues().addPropertyValue("id", id);
        }

        // 如果使用ProtocolConfig的配置类，dubbo:protocol
        if (ProtocolConfig.class.equals(beanClass)) {
            // 拿到注册在parser环境里所有getBeanDefinitionNames，他实际上是bean map的key部分
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    Object value = property.getValue();
                    // 如果id 为protocol
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        }
        // dubbo:service 对应的配置解析
        else if (ServiceBean.class.equals(beanClass)) {
            String className = element.getAttribute("class");
            if (className != null && className.length() > 0) {
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
                beanDefinition.getPropertyValues().addPropertyValue("ref",
                        new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        }
        // provider配置
        else if (ProviderConfig.class.equals(beanClass)) {
            // tag:service ;property:provider;ref:id
            // parserContext
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        }

        // consumer配置，对于consumer，找到对应的reference 标识
        else if (ConsumerConfig.class.equals(beanClass)) {
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }

        // ok，以上都没有，接下来，正常解析了
        Set<String> props = new HashSet<String>();
        ManagedMap parameters = null;
        for (Method setter : beanClass.getMethods()) {
            String name = setter.getName();

            // 找到set方法来设置beanclass对象的值
            if (name.length() > 3 && name.startsWith("set") && Modifier.isPublic(setter.getModifiers())
                    && setter.getParameterTypes().length == 1) { // 判断set方法
                Class<?> type = setter.getParameterTypes()[0];

                // 获取属性的名称，并且把名称第一个大写字母 --> 小写，并且各个驼峰使用‘_’连接
                String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4),
                        "-");
                props.add(property);
                Method getter = null;

                // 获取get或者is等取值方法
                try {
                    getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e) {
                    try {
                        getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                    } catch (NoSuchMethodException e2) {
                    }
                }

                // 如果没有get等取值方法，则不放到结果中，对于处理registry的bean，需要解析各个属性参数，比如address，port，timeout等
                if (getter == null || !Modifier.isPublic(getter.getModifiers()) || !type.equals(getter.getReturnType())) {
                    continue;
                }
                if ("parameters".equals(property)) {
                    parameters = parseParameters(element.getChildNodes(), beanDefinition);
                } else if ("methods".equals(property)) {
                    parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                } else if ("arguments".equals(property)) {
                    parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
                } else {

                    // ok，来解析每个属性参数了
                    String value = element.getAttribute(property);
                    if (value != null) {
                        value = value.trim();
                        if (value.length() > 0) {
                            // N/A配置，则采用new 新配置对象
                            if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                                RegistryConfig registryConfig = new RegistryConfig();
                                registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                                beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                            }
                            // 多注册中心配置？？
                            else if ("registry".equals(property) && value.indexOf(',') != -1) {
                                parseMultiRef("registries", value, beanDefinition, parserContext);
                            } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                                parseMultiRef("providers", value, beanDefinition, parserContext);
                            } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                                parseMultiRef("protocols", value, beanDefinition, parserContext);
                            } else {
                                Object reference;
                                if (isPrimitive(type)) {
                                    if ("async".equals(property) && "false".equals(value) || "timeout".equals(property)
                                            && "0".equals(value) || "delay".equals(property) && "0".equals(value)
                                            || "version".equals(property) && "0.0.0".equals(value)
                                            || "stat".equals(property) && "-1".equals(value)
                                            || "reliable".equals(property) && "false".equals(value)) {
                                        // 兼容旧版本xsd中的default值
                                        value = null;
                                    }
                                    reference = value;
                                } else if ("protocol".equals(property)
                                        && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                        && (!parserContext.getRegistry().containsBeanDefinition(value) || !ProtocolConfig.class
                                                .getName().equals(
                                                        parserContext.getRegistry().getBeanDefinition(value)
                                                                .getBeanClassName()))) {
                                    if ("dubbo:provider".equals(element.getTagName())) {
                                        logger.warn("Recommended replace <dubbo:provider protocol=\"" + value
                                                + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                    }
                                    // 兼容旧版本配置
                                    ProtocolConfig protocol = new ProtocolConfig();
                                    protocol.setName(value);
                                    reference = protocol;
                                } else if ("monitor".equals(property)
                                        && (!parserContext.getRegistry().containsBeanDefinition(value) || !MonitorConfig.class
                                                .getName().equals(
                                                        parserContext.getRegistry().getBeanDefinition(value)
                                                                .getBeanClassName()))) {
                                    // 兼容旧版本配置
                                    reference = convertMonitor(value);
                                } else if ("onreturn".equals(property)) {
                                    int index = value.lastIndexOf(".");
                                    String returnRef = value.substring(0, index);
                                    String returnMethod = value.substring(index + 1);
                                    reference = new RuntimeBeanReference(returnRef);
                                    beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
                                } else if ("onthrow".equals(property)) {
                                    int index = value.lastIndexOf(".");
                                    String throwRef = value.substring(0, index);
                                    String throwMethod = value.substring(index + 1);
                                    reference = new RuntimeBeanReference(throwRef);
                                    beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
                                } else {
                                    if ("ref".equals(property)
                                            && parserContext.getRegistry().containsBeanDefinition(value)) {
                                        BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                        if (!refBean.isSingleton()) {
                                            throw new IllegalStateException("The exported service ref " + value
                                                    + " must be singleton! Please set the " + value
                                                    + " bean scope to singleton, eg: <bean id=\"" + value
                                                    + "\" scope=\"singleton\" ...>");
                                        }
                                    }
                                    reference = new RuntimeBeanReference(value);
                                }
                                beanDefinition.getPropertyValues().addPropertyValue(property, reference);
                            }
                        }
                    }
                }
            }
        }

        // 获取所有属性
        // ok,现在解析每一个节点对应的属性参数，比如 address，loadbalance等
        NamedNodeMap attributes = element.getAttributes();
        int len = attributes.getLength();
        for (int i = 0; i < len; i++) {
            Node node = attributes.item(i);
            String name = node.getLocalName();
            if (!props.contains(name)) {
                if (parameters == null) {
                    parameters = new ManagedMap();
                }
                String value = node.getNodeValue();
                // 整了这么多，到这里终于放进ManagedMap里面去了
                parameters.put(name, new TypedStringValue(value, String.class));
            }
        }
        if (parameters != null) {
            // 这里放到beanDefinition属性里面，进而在Context中就可以拿到了
            beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
        }
        return beanDefinition;
    }
    
    //.............................
    }

```

