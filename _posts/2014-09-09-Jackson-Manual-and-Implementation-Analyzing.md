---
layout: post
categories: 
  - Java 
  - Json  
title:  "Jackson 工具类使用及配置指南"
date: 2014-09-09 22:21:35 +0800
comments: true
---

## <a id="Intro">前言</a>

Json数据格式这两年发展的很快，其声称相对XML格式有很对好处:

* 容易阅读；
* 解析速度快；
* 占用空间更少。

不过,JSON 和 XML两者纠结谁优谁劣,这里不做讨论,可以参见知乎上[为什么XML这么笨重的数据结构仍在广泛应用？](http://www.zhihu.com/question/20738607)

最近在项目中，会有各种解析JSON文本的需求，使用第三方`Jackson`工具来解析时，又担心当增加会减少字段，会不会出现非预期的情况，因此，研究下`jackson`解析json数据格式的代码，很有需要。

## <a id="JsonUtils">Jackson使用工具类</a>

通常，我们对json格式的数据，只会进行解析和封装两种，也就是`json字符串--->java对象`以及`java对象---> json字符串`。

``` java
       
    public class JsonUtils {
    /**
     * Logger for this class
     */
    private static final Logger logger = LoggerFactory.getLogger(JsonUtils.class);

    private final static ObjectMapper objectMapper = new ObjectMapper();

    static {
        objectMapper.configure(JsonParser.Feature.ALLOW_COMMENTS, true);
        objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES, true);
        objectMapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);
        objectMapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
        objectMapper.configure(JsonParser.Feature.INTERN_FIELD_NAMES, true);
        objectMapper.configure(JsonParser.Feature.CANONICALIZE_FIELD_NAMES, true);
        objectMapper.configure(DeserializationConfig.Feature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    private JsonUtils() {
    }

    public static String encode(Object obj) {
        try {
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

    /**
     * 将json string反序列化成对象
     *
     * @param json
     * @param valueType
     * @return
     */
    public static <T> T decode(String json, Class<T> valueType) {
        try {
            return objectMapper.readValue(json, valueType);
        } catch (JsonParseException e) {
            logger.error("decode(String, Class<T>)", e);
        } catch (JsonMappingException e) {
            logger.error("decode(String, Class<T>)", e);
        } catch (IOException e) {
            logger.error("decode(String, Class<T>)", e);
        }
        return null;
    }

    /**
     * 将json array反序列化为对象
     *
     * @param json
     * @param jsonTypeReference
     * @return
     */
    @SuppressWarnings("unchecked")
    public static <T> T decode(String json, TypeReference<T> typeReference) {
        try {
            return (T) objectMapper.readValue(json, typeReference);
        } catch (JsonParseException e) {
            logger.error("decode(String, JsonTypeReference<T>)", e);
        } catch (JsonMappingException e) {
            logger.error("decode(String, JsonTypeReference<T>)", e);
        } catch (IOException e) {
            logger.error("decode(String, JsonTypeReference<T>)", e);
        }
        return null;
    }

}
             
```

## <a id="JsonConfig">Jackson配置属性</a>

如果上面的工具类实例，在Jackson中存在一些属性配置，这些配置决定了最后在解析或者编码后数据视图。因此，在分析Jackson之前，先了解下，Jackson具有的一些配置含义。

<!-- more -->

### JsonParser解析相关配置属性

`JsonParser`将JSON 数据格式的String字符串，解析成为Java对象。Jackson在解析的时候，对于一些非JSON官方文档支持的属性，则需要通过一些配置才可以被Jackson工具解析成对象。

``` java

	/**
     * Enumeration that defines all togglable features for parsers.
     */
    public enum Feature {

        // // // Low-level I/O handling features:

        /**
         * 这个特性，决定了解析器是否将自动关闭那些不属于parser自己的输入源。 如果禁止，则调用应用不得不分别去关闭那些被用来创建parser的基础输入流InputStream和reader；
         * 如果允许，parser只要自己需要获取closed方法（当遇到输入流结束，或者parser自己调用 JsonParder#close方法），就会处理流关闭。
         *
         * 注意：这个属性默认是true，即允许自动关闭流
         *
         */
        AUTO_CLOSE_SOURCE(true),

        // // // Support for non-standard data format constructs

        /**
         * 该特性决定parser将是否允许解析使用Java/C++ 样式的注释（包括'/'+'*' 和'//' 变量）。 由于JSON标准说明书上面没有提到注释是否是合法的组成，所以这是一个非标准的特性；
         * 尽管如此，这个特性还是被广泛地使用。
         *
         * 注意：该属性默认是false，因此必须显式允许，即通过JsonParser.Feature.ALLOW_COMMENTS 配置为true。
         *
         */
        ALLOW_COMMENTS(false),

        /**
         * 这个特性决定parser是否将允许使用非双引号属性名字， （这种形式在Javascript中被允许，但是JSON标准说明书中没有）。
         *
         * 注意：由于JSON标准上需要为属性名称使用双引号，所以这也是一个非标准特性，默认是false的。
         * 同样，需要设置JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES为true，打开该特性。
         *
         */
        ALLOW_UNQUOTED_FIELD_NAMES(false),

        /**
         * 该特性决定parser是否允许单引号来包住属性名称和字符串值。
         *
         * 注意：默认下，该属性也是关闭的。需要设置JsonParser.Feature.ALLOW_SINGLE_QUOTES为true
         *
         */
        ALLOW_SINGLE_QUOTES(false),

        /**
         * 该特性决定parser是否允许JSON字符串包含非引号控制字符（值小于32的ASCII字符，包含制表符和换行符）。 如果该属性关闭，则如果遇到这些字符，则会抛出异常。
         * JSON标准说明书要求所有控制符必须使用引号，因此这是一个非标准的特性。
         *
         * 注意：默认时候，该属性关闭的。需要设置：JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS为true。
         *
         */
        ALLOW_UNQUOTED_CONTROL_CHARS(false),

        /**
         * 该特性可以允许接受所有引号引起来的字符，使用‘反斜杠\’机制：如果不允许，只有JSON标准说明书中 列出来的字符可以被避开约束。
         *
         * 由于JSON标准说明中要求为所有控制字符使用引号，这是一个非标准的特性，所以默认是关闭的。
         *
         * 注意：一般在设置ALLOW_SINGLE_QUOTES属性时，也设置了ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER属性，
         * 所以，有时候，你会看到不设置ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER为true，但是依然可以正常运行。
         *
         * @since 1.6
         */
        ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER(false),

        /**
         * 该特性决定parser是否允许JSON整数以多个0开始(比如，如果000001赋值给json某变量，
         * 如果不设置该属性，则解析成int会抛异常报错：org.codehaus.jackson.JsonParseException: Invalid numeric value: Leading zeroes not
         * allowed)
         *
         * 注意：该属性默认是关闭的，如果需要打开，则设置JsonParser.Feature.ALLOW_NUMERIC_LEADING_ZEROS为true。
         *
         * @since 1.8
         */
        ALLOW_NUMERIC_LEADING_ZEROS(false),

        /**
         * 该特性允许parser可以识别"Not-a-Number" (NaN)标识集合作为一个合法的浮点数。 例如： allows (tokens are quoted contents, not including
         * quotes):
         * <ul>
         * <li>"INF" (for positive infinity), as well as alias of "Infinity"
         * <li>"-INF" (for negative infinity), alias "-Infinity"
         * <li>"NaN" (for other not-a-numbers, like result of division by zero)
         * </ul>
         */

        ALLOW_NON_NUMERIC_NUMBERS(false),

        // // // Controlling canonicalization (interning etc)

        /**
         * 该特性决定JSON对象属性名称是否可以被String#intern 规范化表示。
         *
         * 如果允许，则JSON所有的属性名将会 intern() ；如果不设置，则不会规范化，
         *
         * 默认下，该属性是开放的。此外，必须设置CANONICALIZE_FIELD_NAMES为true
         *
         * 关于intern方法作用：当调用 intern 方法时，如果池已经包含一个等于此 String 对象的字符串 （该对象由 equals(Object) 方法确定），则返回池中的字符串。否则，将此 String
         * 对象添加到池中， 并且返回此 String 对象的引用。
         *
         * @since 1.3
         */
        INTERN_FIELD_NAMES(true),

        /**
         * 该特性决定JSON对象的属性名称是否被规范化。
         *
         * @since 1.5
         */
        CANONICALIZE_FIELD_NAMES(true),

        ;

        final boolean _defaultState;

        /**
         * Method that calculates bit set (flags) of all features that are enabled by default.
         */
        public static int collectDefaults() {
            int flags = 0;
            for (Feature f : values()) {
                if (f.enabledByDefault()) {
                    flags |= f.getMask();
                }
            }
            return flags;
        }

        private Feature(boolean defaultState) {
            _defaultState = defaultState;
        }

        public boolean enabledByDefault() {
            return _defaultState;
        }

        public boolean enabledIn(int flags) {
            return (flags & getMask()) != 0;
        }

        public int getMask() {
            return (1 << ordinal());
        }
    };
```

>> Note: 在枚举最后有一个公共静态方法`collectDefaults()`，这个方法返回一个整形，整形包含的是所有枚举项对应位bit为初始默认值（true：1；false：0），如果默认属性为true，则通过对`1 << ordinal()`的值和flags进行亦或来置位。
	


### DeserializationConfig反序列化相关配置属性

将Java 对象序列化为Json字符串。Jackson在序列化Java对象的时候，对于有些不存在的属性处理，以及一些类型转换等，都可以通过配置来设置。

``` java


	/**
     * Enumeration that defines togglable features that guide the serialization feature.
     */
    public enum Feature implements MapperConfig.ConfigFeature {
        /*
         * /****************************************************** Introspection features
         * /******************************************************
         */

        /**
         * 该特性决定是否启动内部注解功能支持配置；如果允许，则使用AnnotationIntrospector扫描配置，否则，不考了注解配置。
         *
         * 默认启动该功能配置属性。
         * 
         * @since 1.2
         */
        USE_ANNOTATIONS(true),

        /**
         * 该特性决定是否使用“getter”方法来根据标准bean命名转换方式来自动检测。如果true，则所有公共的带有一个参数
         * 并且前缀为set的方法都将被当做setter方法。如果false，只会把显式注解的作为setter方法。
         *
         * 注意: 这个特性的优先级低于显式注解，并且只会在获取不到更细粒度配置的情况下。
         *
         */
        AUTO_DETECT_GETTERS(true),                        DETECT_IS_GETTERS(true),

        /**
         * 该特性决定是否使用creator方法来根据公共构造函数以及名字为“valueOf”的静态单参数方法自动检测。
         *
         * 注意：这个特性比每个类上注解的优先级要低。
         *
         */
        AUTO_DETECT_CREATORS(true),

        /**
         * 这个特性决定是否非静态field被当做属性。如果true，则所有公共成员field都被当做属性， 否则只有注解，才会被当做属性field。
         *
         */
        AUTO_DETECT_FIELDS(true),

        /**
         * 使用getter方法来作为setter方法（一般只处理集合和Maps，和其他没有setter的类型）。 该属性决定是否不需要setter方法，而只需要getter方法来修改属性。
         * 
         * 注意：该配置优先级低于setter。
         */
        USE_GETTERS_AS_SETTERS(true),

        /**
         * 该特性决定当访问属性时候，方法和field访问是否修改设置。 如果设置为true，则通过反射调用方法AccessibleObject#setAccessible 来允许访问不能访问的对象。
         * 
         */
        CAN_OVERRIDE_ACCESS_MODIFIERS(true),                GETTERS(false),

        /*
         * /****************************************************** /* Type conversion features
         * /******************************************************
         */

        /**
         * 该特性决定对于json浮点数，是否使用BigDecimal来序列化。如果不允许，则使用Double序列化。
         * 
         * 注意：该特性默认是关闭的，因为性能上来说，BigDecimal低于Double。
         *
         */
        USE_BIG_DECIMAL_FOR_FLOATS(false),

        /**
         * 该特性决定对于json整形（非浮点），是否使用BigInteger来序列化。如果不允许，则根据数值大小来确定 是使用Integer}, {@link Long} 或者
         * {@link java.math.BigInteger}
         *
         */
        USE_BIG_INTEGER_FOR_INTS(false),

        // [JACKSON-652]
        /**
         * 该特性决定JSON ARRAY是映射为Object[]还是List<Object>。如果开启，都为Object[]，false时，则使用List。
         *
         * @since 1.9
         */
        USE_JAVA_ARRAY_FOR_JSON_ARRAY(false),

        /**
         * 该特性决定了使用枚举值的标准序列化机制：如果允许，则枚举假定使用Enum.toString()返回的值作为序列化结构；如果禁止, 则返回Enum.name()的值。
         *
         * 注意：默认使用的时Enum.name()的值作为枚举序列化结果。这个的设置和WRITE_ENUMS_USING_TO_STRING需要一致。
         *
         * For further details, check out [JACKSON-212]
         * 
         * @since 1.6
         */
        READ_ENUMS_USING_TO_STRING(false),

        /*
         * /****************************************************** Error handling features
         * /****************************************************** 错误处理特性
         */

        /**
         * 该特性决定了当遇到未知属性（没有映射到属性，没有任何setter或者任何可以处理它的handler），是否应该抛出一个
         * JsonMappingException异常。这个特性一般式所有其他处理方法对未知属性处理都无效后才被尝试，属性保留未处理状态。
         *
         * 默认情况下，该设置是被打开的。
         *
         * @since 1.2
         */
        FAIL_ON_UNKNOWN_PROPERTIES(true),

        /**
         * 该特性决定当遇到JSON null的对象是java 原始类型，则是否抛出异常。当false时，则使用0 for 'int', 0.0 for double 来设定原始对象初始值。
         *
         * 默认情况下，允许原始类型可以使用null。
         *
         * @since 1.7
         */
        FAIL_ON_NULL_FOR_PRIMITIVES(false),

        /**
         * 该特性决定JSON 整数是否是一个有效的值，当被用来反序列化Java枚举值。如果false，数字可以接受，并且映射为枚举的值ordinal()；
         * 如果true，则数字不允许并且抛出JsonMappingException异常。后面一种行为原因是因为大部分情况下，枚举被反序列化为 JSON 字符串， 从而造成从整形到枚举的意外映射关系。
         *
         * Feature is disabled by default (to be consistent with behavior of Jackson 1.6), i.e. to allow use of JSON
         * integers for Java enums.
         * 
         * @since 1.7
         */
        FAIL_ON_NUMBERS_FOR_ENUMS(false),

        /**
         * 异常封装，不封装Error,catch异常之后，抛出IOException。默认封装异常。
         *
         * @since 1.7
         */
        WRAP_EXCEPTIONS(true),

        /*
         * /****************************************************** Structural conversion features
         * /****************************************************** 数据结构转换特性
         */

        /**
         * 该特性决定是否接受强制非数组（JSON）值到Java集合类型。如果允许，集合反序列化将尝试处理非数组值。
         *
         * Feature that determines whether it is acceptable to coerce non-array (in JSON) values to work with Java
         * collection (arrays, java.util.Collection) types. If enabled, collection deserializers will try to handle
         * non-array values as if they had "implicit" surrounding JSON array. This feature is meant to be used for
         * compatibility/interoperability reasons, to work with packages (such as XML-to-JSON converters) that leave out
         * JSON array in cases where there is just a single element in array.
         * 
         * @since 1.8
         */
        ACCEPT_SINGLE_VALUE_AS_ARRAY(false),

        /**
         * 该特征允许 unwrap根级别JSON 值，来匹配WRAP_ROOT_VALUE 序列化设置。
         *
         * @since 1.9
         */
        UNWRAP_ROOT_VALUE(false),

        /*
         * /****************************************************** Value conversion features
         * /****************************************************** 值转换特性
         */

        /**
         * 该特性可以允许JSON空字符串转换为POJO对象为null。如果禁用，则标准POJO只会从JSON null或者JSON对象转换过来；
         * 如果允许，则空JSON字符串可以等价于JSON null。
         * @since 1.8
         */
        ACCEPT_EMPTY_STRING_AS_NULL_OBJECT(false)

        ;

        final boolean _defaultState;

        private Feature(boolean defaultState) {
            _defaultState = defaultState;
        }

        @Override
        public boolean enabledByDefault() {
            return _defaultState;
        }

        @Override
        public int getMask() {
            return (1 << ordinal());
        }
    }

```


### SerializationConfig 序列化相关配置属性

``` java

	/**
     * 定义序列化对象所需配置的一些枚举.
     */
    public enum Feature implements MapperConfig.ConfigFeature
    {
        /*
        /******************************************************
        /*  Introspection features
        /******************************************************
         */
        
        /**
         * 注解扫描配置
         */
        USE_ANNOTATIONS(true),

        /**
         * 获取getter方法，前缀为get
         */
        AUTO_DETECT_GETTERS(true),

        /**
         * 获取getter方法，前缀为is
         */
        AUTO_DETECT_IS_GETTERS(true),

        /**
         * 将对象所有的field作为json属性
         */
         AUTO_DETECT_FIELDS(true),

        /**
         * 该特性决定当访问属性时候，方法和field访问是否修改设置。 如果设置为true，
         * 则通过反射调用方法AccessibleObject#setAccessible 来允许访问不能访问的对象。
         */
        CAN_OVERRIDE_ACCESS_MODIFIERS(true),

        /**
         * 获取的getter方法需要setter方法，否则，所有发现的getter都可以作为getter方法。
         */
        REQUIRE_SETTERS_FOR_GETTERS(false),
        
        /*
        /******************************************************
        /* Generic output features
        /******************************************************
         */

        /**
         * 属性对应的值为null，是否需要写出来，write out。
         */
        @Deprecated
        WRITE_NULL_PROPERTIES(true),

        /**
         * 特征决定是使用运行时动态类型，还是声明的静态类型。
         * 也可以使用{@link JsonSerialize#typing} 注解属性
         */
        USE_STATIC_TYPING(false),

        /**
         * 该特性决定拥有view注解{@link org.codehaus.jackson.map.annotate.JsonView}的属性是否在JSON序列化视图中。如果true，则非注解视图，也包含；
         * 否则，它们将会被排除在外。
         *
         */
        DEFAULT_VIEW_INCLUSION(true),
        
        /**
         * 在JAVA中配置XML root{@XmlRootElement.name}注解,最后xml数据中会出现对应root根name.
         */
        WRAP_ROOT_VALUE(false),

        /**
         * 该特性对于最基础的生成器，使用默认pretty printer {@link org.codehaus.jackson.JsonGenerator#useDefaultPrettyPrinter}
         * 这只会对{@link org.codehaus.jackson.JsonGenerator}有影响.该属性值允许使用默认的实现。
         */
        INDENT_OUTPUT(false),

        /**
         * 是否对属性使用排序，默认排序按照字母顺序。
         */
        SORT_PROPERTIES_ALPHABETICALLY(false),
        
        /*
        /******************************************************
        /*  Error handling features
        /******************************************************
         */
        
        /**
         * 是否允许一个类型没有注解表明打算被序列化。默认true，抛出一个异常；否则序列化一个空对象，比如没有任何属性。
         *
         * Note that empty types that this feature has only effect on
         * those "empty" beans that do not have any recognized annotations
         * (like <code>@JsonSerialize</code>): ones that do have annotations
         * do not result in an exception being thrown.
         *
         * @since 1.4
         */
        FAIL_ON_EMPTY_BEANS(true),

        /**
         * 封装所有异常
         */
        WRAP_EXCEPTIONS(true),

        /*
        /******************************************************
        /* Output life cycle features
        /******************************************************
         */
        
         /**
          * 该特性决定序列化root级对象的实现closeable接口的close方法是否在序列化后被调用。
          * 
          * 注意：如果true，则完成序列化后就关闭；如果，你可以在处理最后，调用排序操作等，则为false。
          * 
          */
        CLOSE_CLOSEABLE(false),

        /**
         * 该特性决定是否在writeValue()方法之后就调用JsonGenerator.flush()方法。
         * 当我们需要先压缩，然后再flush，则可能需要false。
         * 
         */
        FLUSH_AFTER_WRITE_VALUE(true),
         
        /*
        /******************************************************
        /* Data type - specific serialization configuration
        /******************************************************
         */

        /**
         * 该特性决定是否将基于Date的值序列化为timestamp数字式的值，或者作为文本表示。
         * 如果文本表示，则实际格式化的时候会调用{@link #getDateFormat}方法。
         * 
         * 该特性可能会影响其他date相关类型的处理，虽然我们理想情况是只对date起作用。
         * 
         */
        WRITE_DATES_AS_TIMESTAMPS(true),

        /**
         * 是否将Map中得key为Date的值，也序列化为timestamps形式（否则，会被序列化为文本形式的值）。
         */
        WRITE_DATE_KEYS_AS_TIMESTAMPS(false),

        /**
         * 该特性决定怎样处理类型char[]序列化，是否序列化为一个显式的JSON数组，还是默认作为一个字符串。
         *
         */
        WRITE_CHAR_ARRAYS_AS_JSON_ARRAYS(false),

        /**
         * 该特性决定对Enum 枚举值使用标准的序列化机制。如果true，则返回Enum.toString()值，否则为Enum.name()
         *
         */
        WRITE_ENUMS_USING_TO_STRING(false),

        /**
         * 这个特性决定Java枚举值是否序列化为数字（true）或者文本值（false）.如果是值的话，则使用Enum.ordinal().
         * 该特性优先级高于上面的那个。
         * 
         * @since 1.9
         */
        WRITE_ENUMS_USING_INDEX(false),
        
        /**
         * 决定是否Map的带有null值的entry被序列化（true）
         *
         */
        WRITE_NULL_MAP_VALUES(true),

        /**
         * 决定容器空的属性（声明为Collection或者array的值）是否被序列化为空的JSON数组（true），否则强制输出。
         *
         * Note that this does not change behavior of {@link java.util.Map}s, or
         * "Collection-like" types.
         * 
         * @since 1.9
         */
        WRITE_EMPTY_JSON_ARRAYS(true)
        
            ;

        final boolean _defaultState;
        
        private Feature(boolean defaultState) {
            _defaultState = defaultState;
        }

        @Override
        public boolean enabledByDefault() { return _defaultState; }
    
        @Override
        public int getMask() { return (1 << ordinal()); }
    }

```

## <a id="JsonParser">Jackson解析JSON数据</a>

Jackson对外提供了多种解析json数据格式的方法，例如，`String context`、`Reader src`、`Url src`等，此外，对于Java POJO类型也提供了三种方式：`Class<T> valueType`和`TypeReference valueTypeRef`以及`JavaType valueType`。为了简单，这里分析通常应用最多的一种，即

``` java

public <T> T readValue(String content, Class<T> valueType)

```

`readValue()`和其他解析方法一样，内部都是通过构造`_readMapAndClose(JsonParser jp, JavaType valueType)`方法所需要的参数，来调用解析JSON数据的。

首先，来看下如何构造方法的两个参数：

``` java

	public JsonParser createJsonParser(String content)
        throws IOException, JsonParseException
    {
	// true -> we own the Reader (and must close); not a big deal（还记得上面的配置吗:)）
	Reader r = new StringReader(content);
	return _createJsonParser(r, _createContext(r, true));
    }
    
    // 构造实际使用的parser的工厂方法。
    // _parserFeatures就是上面默认的属性int，collectDefaults()方法返回值。
    // _objectCodec：实现在JAVA对象和JSON内容之间的转换功能，没有默认，需要显式去设置。一般我们直接使用MappingJsonFactory构造。
    protected JsonParser _createJsonParser(Reader r, IOContext ctxt)
	throws IOException, JsonParseException
    {
        return new ReaderBasedParser(ctxt, _parserFeatures, r, _objectCodec,
                _rootCharSymbols.makeChild(isEnabled(JsonParser.Feature.CANONICALIZE_FIELD_NAMES),
                    isEnabled(JsonParser.Feature.INTERN_FIELD_NAMES)));
    }
    
    // 直接调用构造函数构造一个JsonParser实现类实例
    public ReaderBasedParser(IOContext ioCtxt, int features, Reader r,
                             ObjectCodec codec, CharsToNameCanonicalizer st)
    {
        super(ioCtxt, features, r);
        _objectCodec = codec;
        _symbols = st;
    }
    
```

在上面实现中，还可以看看`_createContext(r, true)`的实现，它会构造出一个`IOConetxt`对象。

``` java

	public IOContext(BufferRecycler br, Object sourceRef, boolean managedResource)
    {
        _bufferRecycler = br;
        _sourceRef = sourceRef;
        _managedResource = managedResource;
    }
    
    //  _recyclerRef 是一个定义全局的ThreadLocal<SoftReference<BufferRecycler>>
     public BufferRecycler _getBufferRecycler()
    {
        SoftReference<BufferRecycler> ref = _recyclerRef.get();
        BufferRecycler br = (ref == null) ? null : ref.get();

        if (br == null) {
            br = new BufferRecycler();
            _recyclerRef.set(new SoftReference<BufferRecycler>(br));
        }
        return br;
    }

```

构造函数有三个参数，`managedResource`，我们在配置上讲过，Jackson对于外部的资源会默认自动关闭流，但是对`自己拥有的流Reader`，会自动关闭，无论设置与否。`Object sourceRef`参数其实就是我们通过`content`构造出来的Reader引用。`BufferRecycler br`这是一个很重要的参数，涉及到内存分配优化。

>> Note： `BufferRecycler`其实就是一个小的工具类，主要负责初始字节/字符缓存的重复使用。在Jackson中，主要用来通过引用该类的`SoftReference`形式作为`ThreadLocal`成员，这样可以达到低负载GC循环的效果，显然是使用流reader所期待的结果。
其主要定义四种类型缓存：
		
``` java

      public enum CharBufferType {
        TOKEN_BUFFER(2000) // Tokenizable input
            ,CONCAT_BUFFER(2000) // concatenated output
            ,TEXT_BUFFER(200) // Text content from input
            ,NAME_COPY_BUFFER(200) // Temporary buffer for getting name characters
            ;
        
 private final int size;

 CharBufferType(int size) { this.size = size; }
    }
    
```


上面的`JsonParser`就完成的参数构造，接下来就是`JavaType`了，两种类型都最后调用`_constructType`来构造`JavaType`类型。

``` java

	public JavaType constructType(Type type) {
        return _constructType(type, null);
    }
    
    
    /**
     * Factory method that can be used if type information is passed
     * as Java typing returned from <code>getGenericXxx</code> methods
     * (usually for a return or argument type).
     */
    public JavaType _constructType(Type type, TypeBindings context)
    {
        JavaType resultType;

        // simple class?
        if (type instanceof Class<?>) {
            Class<?> cls = (Class<?>) type;
            /* 24-Mar-2010, tatu: Better create context if one was not passed;
             *   mostly matters for root serialization types
             */
            if (context == null) {
                context = new TypeBindings(this, cls);
            }
            resultType = _fromClass(cls, context);
        }
        // But if not, need to start resolving.
        else if (type instanceof ParameterizedType) {//带有参数化的类型，比如Collection<String>
            resultType = _fromParamType((ParameterizedType) type, context);
        }
        else if (type instanceof GenericArrayType) {//表示一个数组类型，成员有参数化类型或者type变量
            resultType = _fromArrayType((GenericArrayType) type, context);
        }
        else if (type instanceof TypeVariable<?>) {//其他类型的父接口
            resultType = _fromVariable((TypeVariable<?>) type, context);
        }
        else if (type instanceof WildcardType) {//通配符类型，比如? 或者 ? extends Number
            resultType = _fromWildcard((WildcardType) type, context);
        } else {
        
        // 最后类型都不符合，就会抛出非法参数异常Unrecognized Type
          throw new IllegalArgumentException("Unrecognized Type: "+type.toString());
        }
        /* 
         * 目前只会被 simple types调用 (i.e. not for arrays, map or collections).
         */
        if (_modifiers != null && !resultType.isContainerType()) {
            for (TypeModifier mod : _modifiers) {
                resultType = mod.modifyType(resultType, type, context, this);
            }
        }
        return resultType;
    }

```

>> 代码逻辑很简单，就是通过对参数类型进行不同的处理构造，最后返回`JavaType`某一具体的实现类实例。和其他处理一样，一般都是从精确类型开始匹配，慢慢抽象。

接下来，为了简单起见，我们不对非常用设置进行分析，看下面代码：

``` java

	protected Object _readMapAndClose(JsonParser jp, JavaType valueType)
        throws IOException, JsonParseException, JsonMappingException
    {
        try {
            Object result;
            JsonToken t = _initForReading(jp);
            
            // 省略一部分非重点代码。。。。。。
            
            	DeserializationConfig cfg = copyDeserializationConfig();//创建一个反序列化配置的副本，内部采用copy-on-write模式，所以使用副本。
                DeserializationContext ctxt = _createDeserializationContext(jp, cfg);//反序列化上下文
                JsonDeserializer<Object> deser = _findRootDeserializer(cfg, valueType);
                if (cfg.isEnabled(DeserializationConfig.Feature.UNWRAP_ROOT_VALUE)) {
                    result = _unwrapAndDeserialize(jp, valueType, ctxt, deser);
                } else {
                    result = deser.deserialize(jp, ctxt);
                }
            
            // Need to consume the token too
            jp.clearCurrentToken();
            return result;
        } finally {
            try {
                jp.close();//关闭资源
            } catch (IOException ioe) { }
        }
    }

```

上述代码中有几点重要的方法：


* `JsonToken _initForReading(JsonParser jp)`方法:

该方法主要是在反序列化开始的时候，获取JsonToken标识。

``` java

	protected JsonToken _initForReading(JsonParser jp)
        throws IOException, JsonParseException, JsonMappingException
    {
        /* 首先：必须只想一个token；没有token的情况只可以出现在第一次，和当前token被清除。
         * 此外，这里的JsonParser具体是指ReaderBaseParser实例，（还记得把String Reader处理吗？！）
         */
        JsonToken t = jp.getCurrentToken(); //内部使用了简单地缓存ConcurrentHashMap实现
        if (t == null) {
            // and then we must get something...
            t = jp.nextToken(); // 判断当前是什么类型的JSON标识，比如对象JSON刚开始为"START_OBJECT"
            if (t == null) {
                /* [JACKSON-99] Should throw EOFException, closest thing
                 *   semantically
                 */
                throw new EOFException("No content to map to Object due to end of input");
            }
        }
        return t;
    }

```

* `_findRootDeserializer(DeserializationConfig cfg, JavaType valueType)`方法：

该方法在反序列化root-level值时调用。该方法使用了ConcurrentHashMap缓存来优化，只有root-level的反序列化才会被缓存，这主要是因为反序列化的输入和解决方案都是静态的，但是序列化却是动态的，所以做缓存只对反序列化。

``` java

 	/**
     * Method called to locate deserializer for the passed root-level value.
     */
    protected JsonDeserializer<Object> _findRootDeserializer(DeserializationConfig cfg, JavaType valueType)
        throws JsonMappingException
    {
        // First: have we already seen it?
        JsonDeserializer<Object> deser = _rootDeserializers.get(valueType);//从缓存中获取对应类型的反序列化实例
        if (deser != null) {
            return deser;
        }
        // Nope: need to ask provider to resolve it
        deser = _deserializerProvider.findTypedValueDeserializer(cfg, valueType, null);//使用StdDeserializationContext默认的DeserializationContext实现。反序列化类实际上就是对某一类型进行详细描述，如下图。
        if (deser == null) { // can this happen?
            throw new JsonMappingException("Can not find a deserializer for type "+valueType);
        }
        _rootDeserializers.put(valueType, deser);//放入缓存中
        return deser;
    }

```

反序列化类结构如图：  
<img src="/images/2014/09/deser.png" />

* `deserialize(JsonParser jp, DeserializationContext ctxt)`方法:

由于解析对象为简单地Bean，所以调用`BeanDeserializer`转换为POJO对象。

``` java


 public final Object deserialize(JsonParser jp, DeserializationContext ctxt)
        throws IOException, JsonProcessingException
    {
        JsonToken t = jp.getCurrentToken();
        // common case first:
        if (t == JsonToken.START_OBJECT) {
            jp.nextToken();
            return deserializeFromObject(jp, ctxt);
        }
        // and then others, generally requiring use of @JsonCreator
        switch (t) {
        case VALUE_STRING:
            return deserializeFromString(jp, ctxt);
        case VALUE_NUMBER_INT:
            return deserializeFromNumber(jp, ctxt);
        case VALUE_NUMBER_FLOAT:
	    return deserializeFromDouble(jp, ctxt);
        case VALUE_EMBEDDED_OBJECT:
            return jp.getEmbeddedObject();
        case VALUE_TRUE:
        case VALUE_FALSE:
            return deserializeFromBoolean(jp, ctxt);
        case START_ARRAY:
            // these only work if there's a (delegating) creator...
            return deserializeFromArray(jp, ctxt);
        case FIELD_NAME:
        case END_OBJECT: // added to resolve [JACKSON-319], possible related issues
            return deserializeFromObject(jp, ctxt);
	}
        throw ctxt.mappingException(getBeanClass());
    }

```

下面是具体的反序列化方法,如下：

``` java
    
    public Object deserializeFromObject(JsonParser jp, DeserializationContext ctxt)
        throws IOException, JsonProcessingException
    {
        if (_nonStandardCreation) {
            if (_unwrappedPropertyHandler != null) {
                return deserializeWithUnwrapped(jp, ctxt);
            }
            if (_externalTypeIdHandler != null) {
                return deserializeWithExternalTypeId(jp, ctxt);
            }
            return deserializeFromObjectUsingNonDefault(jp, ctxt);
        }

        final Object bean = _valueInstantiator.createUsingDefault();// 构造一个默认的初始化类型对象，对象属性值使用初始化默认值。
        if (_injectables != null) {
            injectValues(ctxt, bean);
        }
        for (; jp.getCurrentToken() != JsonToken.END_OBJECT; jp.nextToken()) { //迭代输入流来设置各个属性的值
            String propName = jp.getCurrentName();
            // Skip field name:
            jp.nextToken();// 迭代到下一个输入流的token名字，即使在POJO中没有对应属性，则会调用_handleUnknown来处理。
            SettableBeanProperty prop = _beanProperties.find(propName); //具体实现如下
            if (prop != null) { // 如果在POJO中有对应属性，则返回kv
                try {
                    prop.deserializeAndSet(jp, ctxt, bean);//根据类型调用具体的方法设置属性值，比如StringDeserializer类对字符串。
                } catch (Exception e) {
                    wrapAndThrow(e, bean, propName, ctxt);
                }
                continue;
            }
            _handleUnknown(jp, ctxt, bean, propName);//这里会根据上面的配置来决定处理方案
        }
        return bean;
    }
    
    // BeanPropertyMap类是一个存储属性名到SettableBeanProperty实例之间的映射的工具类。该类主要代替HashMap，做了一些业务优化。
    public SettableBeanProperty find(String key)
    {
        int index = key.hashCode() & _hashMask;//hash找到bucket位置
        Bucket bucket = _buckets[index];//
        // Let's unroll first lookup since that is null or match in 90+% cases
        if (bucket == null) {//如果不存在，则返回null
            return null;
        }
        // Primarily we do just identity comparison as keys should be interned
        if (bucket.key == key) { // 当没有hash碰撞时，则会相等
            return bucket.value;
        }
        while ((bucket = bucket.next) != null) {//不能直接相等，则遍历找到bucket里面所有的key
            if (bucket.key == key) {
                return bucket.value;
            }
        }
        // Do we need fallback for non-interned Strings?
        return _findWithEquals(key, index);
    }

```


## <a id="JsonDeserializer">Jackson序列化Java对象</a>
`Jackson`工具包对外提供看好几种序列化接口，比如转换为String对象，Byte[]数组，或者直接写到stream中等等，但是不管如何，其内部实现的核心方法都是

``` java
protected final void _configAndWriteValue(JsonGenerator jgen, Object value)
```

因此，这里就选择常用的`writeValueAsString(Object value)`来解析器内部实现。

>> Note：虽然在`Jackson`中提供了`public void writeValue(Writer w, Object value)`的通用方法来序列化java对象，对于需要转换为String类型的需求，只需要`StringWriter`就可以调用writeValue，但是由于这种序列化对性能要求高，并且使用频繁，所以单独提供更高效的实现方式。而正由于在项目代码中AsString 场景十分多，所以这里选择`writeValueAsString`方法分析。

### 序列化方法处理流程

`writeValueAsString(Object value)`其处理和反序列化很相似，基本流程为：

1. *首先，构建通用序列化基础方法所需要的参数类型对象；*
2. *其次，对序列化类型进行分析，根据注解或者"get方法名(比如getXxx,isXxx)"等来构建需要序列化的属性*
3. *然后，通过反射机制分别对所有的序列化属性进行处理：通过发现拿到对应的值,getxxx方法等*
4. *拼接字符串：其内部是根据类型写入一些开始结束符号，例如{,[等，在其中嵌入步骤3的解析设值*
5. *返回最后得到的字符串内容*




