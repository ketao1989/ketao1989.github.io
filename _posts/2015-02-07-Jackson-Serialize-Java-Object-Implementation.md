---
layout: post
categories: 
  - Java 
  - Json  
title:  "Jackson 自定义序列化Java对象实现"
date: 2015-02-07 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

关于Jackson工具类的相关学习和研究,先前已经写过一篇博文;但是,后来由于工作事情就草草结尾.此外,虽然对Jackson序列化和反序列化的实现机制进行了初步学习,但是现在看看,那时候的博客经验和技术水平决定了整篇博文结构并不是清晰明了,并且反序列化的整个简单的解析,对日常开发并没有直接帮助.对于Jackson这种模块化设计,轻巧灵活的扩展方法,多维度的性能效率优化,导致了最后博文的质量非常不好.因此,接下来几篇博文,将会以日常开发使用的一些API为入口,来分别介绍其使用方式和内部实现,然后在此基础上,对此博文进行分拆.

在开发中，前后端交互的项目，现在一般都是基于JSON文本格式给出API接口，因此对于java对象的序列化就是十分重要的了。基于Spring MVC的配置中，我们配置`org.springframework.http.converter.json.MappingJacksonHttpMessageConverter` bean时，就是需要配置`messageConverters`转换器。而针对json的转换器，就可以使用Jackson的`objectMapper`对象了。

返回给前端的`Response Body`一般需要良好的可读性，不能因为后端处理方便，就直接破坏前端展示。比如，我们后端使用枚举处理一些情况，但是如果把枚举英文或者数字返回给前端页面，对用户来说是非常不友好的。因此，这里就是涉及到了java对象自定义序列化的问题了。Jackson工具为我们提供了非常多的常见默认的序列化方法，以及一些扩展的接口；参考系统的序列化方法，我们就可以实现自定义的了。


##  <a id="CustomizeSerialize">Jackson自定义序列化方法</a>

在Jackson中，提供了多种方式去实现自定义的序列化方法。例如，`org.codehaus.jackson.map.JsonSerializableWithType`接口以及`org.codehaus.jackson.map.JsonSerializer`接口。

<!-- more -->

### JsonSerializableWithType 接口

JsonSerializableWithType 接口主要提供了两个方法。

``` java

/**
 * Interface that is to replace {@link JsonSerializable} to
 * allow for dynamic type information embedding.
 * 
 * @since 1.5
 * @author tatu
 */
@SuppressWarnings("deprecation")
public interface JsonSerializableWithType
    extends JsonSerializable
{
    public void serializeWithType(JsonGenerator jgen, SerializerProvider provider,
            TypeSerializer typeSer)
        throws IOException, JsonProcessingException;
}

@Deprecated
public interface JsonSerializable
{
    public void serialize(JsonGenerator jgen, SerializerProvider provider)
        throws IOException, JsonProcessingException;
}

```

在早期还没有`JsonSerializableWithType`接口的时候，官方推荐使用`JsonSerializable`接口。但是，从version 1.5之后，Jackson提供了JsonSerializableWithType来替换JsonSerializable，因为JsonSerializable接口不支持处理某些额外的类型信息。

但是，由于实现JsonSerializableWithType接口的方法比较麻烦，所以，Jackson对外提供了其他可以自定义序列化的抽象类`SerializerBase`和`ScalarSerializerBase`。其中ScalarSerializerBase是对SerializerBase的再次封装，它只针对输出了JSON 字符串，boolean或者数字类型才有用。

继承`SerializerBase`抽象类，主要实现`serialize(TypeB value, JsonGenerator jgen, SerializerProvider provider)`方法即可，实现起来非常简单。此外,Jackson还提供了另一种简单易用的自定义序列化方法，给开发者使用，这就是`JsonSerializer`接口。

### JsonSerializer 接口

JsonSerializer类就是一个抽象类，我们只需要简单实现其中的抽象方法就可以完成自定义的序列化工作。

先来看看`JsonSerializer`抽象类的主要方法实现：

``` java
/**
 * ObjectMapper类定义的抽象API类来给JsonGenerator类完成序列化任意对象到JSON功能。
 *<p>
 * 需要说明的是，官方推荐使用继承SerializerBase类完成自定义序列化工作，替代使用本类。
 * 主要是因为，SerializerBase类提供了更多的可选择的方法给开发者去定制会。
 * 当然，如果你只需要简单的序列化功能，则继承这个类足够了。
 */
public abstract class JsonSerializer<T>
{

    public abstract void serialize(T value, JsonGenerator jgen, SerializerProvider provider)
        throws IOException, JsonProcessingException;

    /**
     * Method that can be called to ask implementation to serialize
     * values of type this serializer handles, using specified type serializer
     * for embedding necessary type information.
     *<p>
     * Default implementation will ignore serialization of type information,
     * and just calls {@link #serialize}: serializers that can embed
     * type information should override this to implement actual handling.
     * Most common such handling is done by something like:
     *<pre>
     *  // note: method to call depends on whether this type is serialized as JSON scalar, object or Array!
     *  typeSer.writeTypePrefixForScalar(value, jgen);
     *  serialize(value, jgen, provider);
     *  typeSer.writeTypeSuffixForScalar(value, jgen);
     *</pre>
     */
    public void serializeWithType(T value, JsonGenerator jgen, SerializerProvider provider,
            TypeSerializer typeSer)
        throws IOException, JsonProcessingException
    {
        serialize(value, jgen, provider);
    }
}
```

同样，继承`JsonSerializer`类，实现serialize方法，就可以完成自定义序列化工作了。其实，和第一种方法相比，其本质上都是一样的，就是实现serialize序列化方法。

##  <a id="RegisterSerializer">自定义序列化方法注册到Jackson</a>

Jackson提供了多种方式注册自定义序列化类，虽然官方指出第一种继承JsonSerializableWithType的方式不需要注册，但是测试了下，发现貌似不起作用。

虽然注册的方式很多，但是主要是三种，并且最后一种在1.8版本以后已经标记为过时状态：

- 继承 `Module interface`接口，官方推荐使用模块接口来完成注册功能；

- 使用注解方式`@JsonSerialize`在方法或field上标记；

- 继承`CustomSerializerFactory`接口，通过addXxxMapping来自定义SerializerFactory。

### Module interface接口

模块接口，是Jackson在1.7以后的版本中给出的。Module接口，是Jackson专门提供了自定义序列化方法类注册到ObjectMapper实例上的接口，比如我们新增了一个数据类型。

在开发过程中，我们只需要直接使用`SimpleModule`类就可以完成注册了，一般没有必要去实现`Module`抽象类。

> Tips：其实Module也提供了反序列化来完成自定义类的注册工作，其他博文会再介绍。

``` java

/**
 * Simple {@link Module} implementation that allows registration
 * of serializers and deserializers, and bean serializer
 * and deserializer modifiers.
 * 
 * @since 1.7
 */
public class SimpleModule extends Module
{
    protected final String _name;
    protected final Version _version;
    
// ........

    public SimpleModule(String name, Version version)
    {
        _name = name;
        _version = version;
    }

// .......

        public SimpleModule addSerializer(JsonSerializer<?> ser)
    {
        if (_serializers == null) {
            _serializers = new SimpleSerializers();
        }
        _serializers.addSerializer(ser);
        return this;
    }
// .......
}

```

### @JsonSerialize 注解

通过注解的方式注册序列化方法，对使用体验来说非常友好。但是会存在两个问题：首先，提供的序列化类的功能比较简单，然后，就是提供的粒度比较小，然后你想在全局的某个类型上都是有自定义序列化，则需要对每个对象类型上都需要标记注解。

`@JsonSerialize` 注解的配置参数有很多种，但是我们只需要using这种来指明具体的自定义序列化类就可以了。

``` java

/**
 * Annotation used for configuring serialization aspects, by attaching
 * to "getter" methods or fields, or to value classes.
 * When annotating value classes, configuration is used for instances
 * of the value class but can be overridden by more specific annotations
 * (ones that attach to methods or fields).
 *<p>
 * An example annotation would be:
 *<pre>
 *  &#64;JsonSerialize(using=MySerializer.class,
 *    as=MySubClass.class,
 *    include=JsonSerialize.Inclusion.NON_NULL,
 *    typing=JsonSerialize.Typing.STATIC
 *  )
 *</pre>
 * (which would be redundant, since some properties block others:
 * specifically, 'using' has precedence over 'as', which has precedence
 * over 'typing' setting)
 *<p>
 * NOTE: since version 1.2, annotation has also been applicable
 * to (constructor) parameters
 *
 * @since 1.1
 */
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.TYPE, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@JacksonAnnotation
public @interface JsonSerialize
{
    // // // Annotations for explicitly specifying deserializer

    /**
     * Serializer class to use for
     * serializing associated value. Depending on what is annotated,
     * value is either an instance of annotated class (used globablly
     * anywhere where class serializer is needed); or only used for
     * serializing property access via a getter method.
     */
    public Class<? extends JsonSerializer<?>> using() default JsonSerializer.None.class;

    // .......
}

```

### CustomSerializerFactory接口

最后一种注册自定义序列化类的方法就是通过使用自定义序列化factory工厂，也就是继承SerializerFactory类。

首先，我们使用或者扩展已经存在的`CustomSerializerFactory`。然后调用实例对象实现的`addSpecificMapping`或者`addGenericMapping`方法增加mapping关系，把自定义的序列化方法add到序列化列表中。最后，我们可以把这个工厂set到ObjectMapper实例对象上。

##  <a id="JacksonCustomizeSample">Java对象序列化自定义示例</a>

下面给出使用自定义序列化类来完成Java对象到JSON的转换工作。

``` java

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

    public static String encode(Object obj) {
        try {

//            SimpleModule SerializeModule = new SimpleModule("SerializeModule", new Version(1, 0, 0, null));
//            SerializeModule.addSerializer(new CustomizeJsonSerializerBaser(TypeB.class)); // assuming serializer declares correct class to bind to
//            objectMapper.registerModule(SerializeModule);

//            CustomSerializerFactory customSerializerFactory = new CustomSerializerFactory();
//            customSerializerFactory.addSpecificMapping(TypeB.class,new CustomizeJsonSerializerBaser(TypeB.class));
//            objectMapper.setSerializerFactory(customSerializerFactory);

            return objectMapper.writeValueAsString(obj);
        } catch (JsonGenerationException e) {
            logger.error("encode(Object)", e); //$NON-NLS-1$
        } catch (JsonMappingException e) {
            logger.error("encode(Object)", e); //$NON-NLS-1$
        } catch (IOException e) {
            logger.error("encode(Object)", e); //$NON-NLS-1$
        }
        return null;
    }
}

```

> Notes：JsonUtils类封装了Jackson序列化操作。代码中，提供了通过Module接口方式和创建`CustomSerializerFactory`对象来完成注册加载。
> 

``` java

public class CustomizeJsonSerializer extends JsonSerializer<List> {

    @Override
    public void serialize(List value, JsonGenerator jgen, SerializerProvider provider) throws IOException, JsonProcessingException {
        jgen.writeStartArray();

        for (Object va :value){
            jgen.writeString(va.toString()+"TestSerialize");
        }

        //jgen.writeString(Joiner.on(";").join(value));
        jgen.writeEndArray();
    }
}

public class CustomizeJsonSerializerBaser extends SerializerBase<TypeB> {

    // 注解必须有默认构造韩式
    public CustomizeJsonSerializerBaser(){
        super(TypeB.class);
    }

    protected CustomizeJsonSerializerBaser(Class<TypeB> t) {

        super(t);
    }

    protected CustomizeJsonSerializerBaser(JavaType type) {
        super(type);
    }

    protected CustomizeJsonSerializerBaser(Class<?> t, boolean dummy) {
        super(t, dummy);
    }

    @Override
    public void serialize(TypeB value, JsonGenerator jgen, SerializerProvider provider) throws IOException, JsonGenerationException {

        jgen.writeString(value.toString());
       // System.out.println("======================");
    }
}

```

> Notes：这里提供了两种不同的方式开发自定义序列化方法。目前，从1.8开始，官方推荐使用第二种方式实现自定义序列化类，但是如果只需要简单的序列化功能，使用第一种完全可以满足需求。
> 

``` java

public class ClassA {

    String name;

    int age;

    List<String> hobby;

    TypeB b;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @JsonSerialize(using = CustomizeJsonSerializerBaser.class)
    public TypeB getB() {
        return b;
    }

    public void setB(TypeB b) {
        this.b = b;
    }

    @JsonSerialize(using = CustomizeJsonSerializer.class)
    public List<String> getHobby() {
        return hobby;
    }

    public void setHobby(List<String> hobby) {
        this.hobby = hobby;
    }

    @Override
    public String toString() {
        return "ClassA{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", hobby=" + hobby +
                '}';
    }
}

public class TypeB {

    String address;

    int code;

    public TypeB(String address, int code) {
        this.address = address;
        this.code = code;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    @Override
    public String toString() {
        return "TypeB{" +
                "address='" + address + '\'' +
                ", code=" + code +
                '}';
    }
}

public class JsonSerializeTest {

    public static void main(String[] args) {

        ClassA a = new ClassA();

        a.setAge(10);
        a.setName("ketao1989");
        a.setHobby(Lists.newArrayList("dinner","swimming","music","programming"));
        a.setB(new TypeB("jiangxi",320000));

        String aJsonStr = JsonUtils.encode(a);
        System.out.println(aJsonStr);

        ClassA aa = new ClassA();

        aa.setAge(19);
        aa.setName("kexiaoxiaoxi");
        aa.setHobby(Lists.newArrayList("MacOs", "Linux", "Windows", "Unknown"));
        aa.setB(new TypeB("beijing", 100000));

        String aaJsonStr = JsonUtils.encode(aa);
        System.out.println(aaJsonStr);


        String bb = JsonUtils.encode(new TypeB("beijing",100000));
        System.out.println(bb);

    }
}

```

> Notes：测试相关类。目前都是通过注解的方式，可以把相关的注释取消，测试其他实现方式完成自定义序列化功能。

## <a id="End">总结</a>

- 官方序列化简要介绍：<http://wiki.fasterxml.com/JacksonHowToCustomSerializers>

- 测试代码svn地址:<http://code.taobao.org/svn/dubbocli/trunk>


