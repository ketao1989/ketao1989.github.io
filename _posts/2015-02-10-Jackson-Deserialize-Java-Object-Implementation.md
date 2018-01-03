---
layout: post
categories: 
  - Java 
  - Json  
title:  "Jackson 自定义反序列化Java对象实现"
date: 2015-02-10 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

上一篇博客，介绍了关于[Jackson 自定义序列化](http://ketao1989.github.io/posts/Jackson-Serialize-Java-Object-Implementation.html)，本文将介绍关于 Jackson 自定义反序列化。

反序列化，Jackson 工具包中已经支持了开发中常用的 Java 类型的解析功能；但是还是会遇到一些我们需要自定义的解析转换工作。比如外部的一些非主流时间格式转换，再比如说对于一些类型转换，做一些额外数据校验和默认容错处理工作，再比如说前端的某种格式的字符串，我们想直接使用自定义反序列化类来完成到 Java Object 的转换工作等等，都需要我们去自己实现 Jackson 反序列化类。

和`Jackson 自定义序列化`相似，Jackson 为自定义的反序列化扩展，也提供了简单易用的接口。

## <a id="CustomizeDeserialize">Jackson 自定义反序列化方法</a>

在 Jackson 自定义序列化中提供了两种接口，但是在反序列的时候，只有一个接口使用。不过，和序列化一样，反序列化的扩展也很简单，接口为：`org.codehaus.jackson.map.JsonDeserializer`，Jackson还提供了一个通用的扩展子类`com.fasterxml.jackson.databind.deser.std.StdDeserializer`。

一般，我们可以直接继承`JsonDeserializer`抽象类就已经足够满足我们的自定义反序列化需求了。

### JsonDeserializer 接口

和序列化很像，这里我们也只需要实现一个反序列化方法`deserialize` 就足够了。当然，如果你有其他的更高的需求，可以进一步 override 其他的方法。

``` java

/**
 * Abstract class that defines API used by {@link ObjectMapper} (and
 * other chained {@link JsonDeserializer}s too) to deserialize Objects of
 * arbitrary types from JSON, using provided {@link JsonParser}.
 */
public abstract class JsonDeserializer<T>
{

    /**
     * 当请求将JSON内容反序列成java类型对象的时候，该方法将会被调用。然后，会返回一个
     * 由方法自己构造的对象实例。
     *
     * 需要注意的是，当JSON为null的时候，该方法该不会被调用，因此，方法不需要去
     * check 该值是否为null。
     */
    public abstract T deserialize(JsonParser jp, DeserializationContext ctxt)
        throws IOException, JsonProcessingException;

    /**
     * 和上面常用的方法不同，这个方法带有一个初始化的对象实例，该实例由反序列类配置的。
     * 
     * 当然，这个方法是不必须实现的。一般的，我们转换一个JSON为集合或者Map的时候，可以把
     * 通用的元素put到集合中，这样返回的时候，就不需要再操作了。
     */
    public T deserialize(JsonParser jp, DeserializationContext ctxt,
                         T intoValue)
        throws IOException, JsonProcessingException
    {
        throw new UnsupportedOperationException("Can not update object of type "
                +intoValue.getClass().getName()+" (by deserializer of type "+getClass().getName()+")");
    }

    /**
     * 这个方法主要是用来支持兼容老的代码，编码编译错误。
     *
     * 也就是说，当我们使用新的代码和反序列化工具的时候，可能还需要去兼容老的代码数据
     * 反序列化，这样子就可以尝试使用老的反序列化类进行解析工作。
     *
     */
    @SuppressWarnings("unchecked")
    public Object deserializeWithType(JsonParser jp, DeserializationContext ctxt,
            TypeDeserializer typeDeserializer)
        throws IOException, JsonProcessingException
    {
        // We could try calling 
        return (T) typeDeserializer.deserializeTypedFromAny(jp, ctxt);
    }

       // ........
}
```

> Tips：这个反序列化方法实现起来很简单，只需要实现核心的`deserialize`方法，使用`jp.getText`获取 value 然后对字符串进行我们需要的各种操作转换，赋值给创建的对象实例，就 OK 了。
> 

<!-- more -->

## <a id="RegisterDeserializer">自定义反序列化方法注册到Jackson</a>

和序列化一样，Jackson 也提供了三种方法来注册到 `ObjectMapper` 上。根据官方推荐，这里只介绍两种方式：

- 使用 `Module Interface`模式来注册反序列化方法（全局）

- 使用注解，细粒度到具体模型对象的属性。

### SimpleModule 类注册

`SimpleModule`类是1.8版本才提供的注册类，当我们想在全局的反序列环境下，对指定类型的每一次反序列化都会按照这个类方法进行解析，就可以在`SimpleModule`中添加指定的自定义反序列类。

``` java

public class SimpleModule extends Module
{
    protected final String _name;
    protected final Version _version;
    
    protected SimpleSerializers _serializers = null;
    protected SimpleDeserializers _deserializers = null;

        public <T> SimpleModule addDeserializer(Class<T> type, JsonDeserializer<? extends T> deser)
    {
        if (_deserializers == null) {
            _deserializers = new SimpleDeserializers();
        }
        _deserializers.addDeserializer(type, deser);
        return this;
    }

    public SimpleModule addKeyDeserializer(Class<?> type, KeyDeserializer deser)
    {
        if (_keyDeserializers == null) {
            _keyDeserializers = new SimpleKeyDeserializers();
        }
        _keyDeserializers.addDeserializer(type, deser);
        return this;
    }

    // ......

}
```

> Tips：显然，我们在`objectMapper.registerModule`中注册构造的simpleModule对象实例，这个实例调用`addDeserializer`把我们自定义的反序列化方法添加进去就可以了。
> 

### @JsonDeserialize 注解

使用注解的方式来注册自定义的反序列化方法，可以把自定义粒度控制到某一个类型对象属性上，但是这对于想全局反序列化某个类型，这种方式都会很繁琐。

`@JsonDeserialize`和序列化注解，完全一样。使用方式如下：

``` java

    @JsonDeserialize(using = CustomizeJsonDeserializer.class)
    public void setB(TypeB b) {
        this.b = b;
    }

```

## <a id="JacksonCustomizeSample">Java对象反序列化自定义示例</a>

和 Jackson 自定义序列化一样，这里主要把实现继承反序列化接口的代码和 mapper 注册贴出来，参考下：

``` java

public class CustomizeJsonDeserializer extends JsonDeserializer<TypeB> {

    @Override
    public TypeB deserialize(JsonParser jp, DeserializationContext ctxt) throws IOException, JsonProcessingException {

        String value = jp.getText();
        Map<String, String> properities = Splitter.on(",").trimResults().withKeyValueSeparator("=").split(StringUtils.substringBetween(value, "{", "}"));

        System.out.println(JsonUtils.encode(properities));

        TypeB typeB = new TypeB(properities.get("address"), Integer.valueOf(properities.get("code")));

        return typeB;
    }
}

public class JsonUtils {

    private static final Logger logger = LoggerFactory.getLogger(JsonUtils.class);

    public final static ObjectMapper objectMapper = new ObjectMapper();

    static {
        objectMapper.configure(JsonParser.Feature.ALLOW_COMMENTS, true);
        objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES, true);
        objectMapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);
        objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
        objectMapper.configure(JsonParser.Feature.INTERN_FIELD_NAMES, true);
        objectMapper.configure(JsonParser.Feature.CANONICALIZE_FIELD_NAMES, true);
        objectMapper.configure(DeserializationConfig.Feature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        objectMapper.setSerializationInclusion(JsonSerialize.Inclusion.NON_NULL);// JSON节点不包含属性值为NULL
    }

public static <T> T decode(String json, Class<T> valueType) {
        try {

            SimpleModule deserializeModule = new SimpleModule("DeserializeModule", new Version(1, 0, 0, null));
            deserializeModule.addDeserializer(TypeB.class,new CustomizeJsonDeserializer()); // assuming serializer declares correct class to bind to
            objectMapper.registerModule(deserializeModule);
            return objectMapper.readValue(json, valueType);
        } catch (JsonParseException e) {
            logger.error("decode(String, Class<T>)", e); //$NON-NLS-1$
        } catch (JsonMappingException e) {
            logger.error("decode(String, Class<T>)", e); //$NON-NLS-1$
        } catch (IOException e) {
            logger.error("decode(String, Class<T>)", e); //$NON-NLS-1$
        }
        return null;
    }
}
```

## <a id="End">总结</a>

- 官方序列化简要介绍：<http://wiki.fasterxml.com/JacksonHowToCustomDeserializers>

- 测试代码git地址:<https://github.com/ketao1989/JavaApiUtilsProject>
