---
layout: post
title: "dubbo序列化机制处理-泛化&普通RPC"
date: 2017-03-29 20:52:39 +0800
comments: true
categories: 
  - java
  - dubbo
  - hessian
---

* content
{:toc}

## 1. 背景

当前，存在各种API网关，这些网关的定位之一就是管理域内各个服务需要对外的接口，统一做降级限流，权限控制，安全校验等工作。内部一般使用各种RPC服务，因此，基于微服务的不断扩展和升级，一般网关都是使用RPC 泛化机制来完成对内部服务的调用。

泛化机制，和普通的rpc调用有一些不同，它不需要引入各个依赖服务的jar包，调用方如果知道接口方法的具体信息：名称，版本，参数类型等，就可以完成rpc调用。

在开发过程中，使用泛化机制，发现一个"问题"，就是业务方调用我们内部的一个接口，接口中有一个参数是long类型，但是业务方使用string类型的参数(数字字符串，例如"12345678")请求过来，但是并没有出现错误。这个问题，在于后面另一个业务方，也按照string类型请求(非纯数字，例如"E12345678")，这个时候报错了。

<!-- more -->

## 2. dubbo 泛化调用

### 2.1 dubbo 泛化调用示例

先来看一个泛化调用的demo

``` java

// --------消费端泛化调用代码--------------
public class UserBizConsumer {  
    public static void main(String[] args) {

        ApplicationConfig applicationConfig = new ApplicationConfig("dubbo-provider");

        ReferenceConfig<GenericService> ref = new ReferenceConfig<>();

        RegistryConfig registryConfig = new RegistryConfig();
        registryConfig.setProtocol("zookeeper");
        registryConfig.setAddress("127.0.0.1:2181");

        ref.setTimeout(1000);
        ref.setProtocol("dubbo");
        ref.setConnections(2);
        ref.setInterface("io.github.ketao1989.dubbo.api.IUserBiz");
        ref.setApplication(applicationConfig);
        ref.setRegistry(registryConfig);
        ref.setGeneric(true);

        GenericService service = ref.get();
        Object o = service.$invoke("queryName",new String[]{"long"},new Object[]{12});
        System.out.println(o);
    }
}

// ------服务端代码--------
public class UserBizProvider {

    public static void main(String[] args) throws IOException {

        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("dubbo/dubbo-provider.xml");
        context.start();

        System.in.read();
    }
}

public interface IUserBiz {
    /**
     * 根据id获取用户名
     */
    public String queryName(long id);
}
```
> 上面是一个简单的测试rpc的代码。服务端对外提供的服务是一个普普通通rpc实现，参数是一个long类型。  
> 消费端的代码也是按照接口约定，传输long类型的12请求服务。和普通rpc调用不同，其没有引入jar api来调用，而是通过代码写配置来完成调用服务，这样就不需要在服务端加入和升级api的时候，还需要网关来升级版本了。

### 2.2 dubbo 泛化调用机制

虽然，从上面的demo来看，泛化机制最大的不同，在消费端需要代码写入大量的接口属性信息。但是，实际上，在底层实现上，并无大区别。看看普通的rpc调用配置：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans.xsd        http://code.alibabatech.com/schema/dubbo        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

       <!-- 提供方应用信息，用于计算依赖关系 -->
       <dubbo:application name="dubbo-provider"  />

       <!-- 使用multicast广播注册中心暴露服务地址 -->
       <dubbo:registry address="127.0.0.1:2181" protocol="zookeeper" id="provider-registry" />

       <!-- 用dubbo协议在20880端口暴露服务 -->
       <dubbo:protocol name="dubbo" port="20881" />

       <!-- consumer-->
       <dubbo:reference interface="io.github.ketao1989.dubbo.api.IUserBiz" id="userBiz"/>
</beans>
```


基于api调用的消费端，框架会对 `dubbo:reference` 进行动态代理，实际调用具体的方法的时候，会拦截，然后解析出方法，入参属性等数据，然后和泛化调用蕾西，将解析完成之后的数据，包装成`ReferenceConfig`来传递给服务端，收到结果，完成调用。

既然，消费端在底层实现上并没有什么区别，那就说明，在服务端肯定有些不同的。

在框架的服务端，存在对于泛化调用的逻辑分支：
``` java
 public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
        if (inv.getMethodName().equals(Constants.$INVOKE) 
                && inv.getArguments() != null
                && inv.getArguments().length == 3
                && ! invoker.getUrl().getParameter(Constants.GENERIC_KEY, false)) {
                    // 通用的泛化调用
                    // 获取实际需要执行的方法，参数类型，参数等
                    String name = ((String) inv.getArguments()[0]).trim();
                    String[] types = (String[]) inv.getArguments()[1];
                    Object[] args = (Object[]) inv.getArguments()[2];
                    Method method = ReflectUtils.findMethodByMethodSignature(invoker.getInterface(), name, types);
                    Class<?>[] params = method.getParameterTypes();
                    if (args == null) {
                        args = new Object[params.length];
                    }
                    String generic = inv.getAttachment(Constants.GENERIC_KEY);
                
                    args = PojoUtils.realize(args, params, method.getGenericParameterTypes());
                    Result result = invoker.invoke(new RpcInvocation(method, args, inv.getAttachments()));return new RpcResult(PojoUtils.generalize(result.getValue()));
                }

        return invoker.invoke(inv);
 }

```

> 泛化调用和普通调用,如上代码，实现分支是不同的。基于泛化机制的调用，其首先会对入参进行解析，构造和普通调用一样的`RpcInvocation`对象，然后执行具体实现方法，包装结果返回。  
> 判断泛化调用的前置条件是方法名为`$invoke`。

和普通rpc调用不同的地方是，泛化调用会存在多一次序列化的操作，如代码:

`args = PojoUtils.realize(args, params, method.getGenericParameterTypes());`

这个地方就是一个关键。将泛化调用的参数按照规则进行解析，我们拿到了method，拿到了实际方法类型列表，以及具体参数。但是，对于具体参数，dubbo框架解析的时候，会进行一些兼容处理，如下代码块(只截取我们关系的代码段)：

``` java
    /**
     * 兼容类型转换。null值是OK的。如果不需要转换，则返回原来的值。
     * 进行的兼容类型转换如下：（基本类对应的Wrapper类型不再列出。）
     */
    @SuppressWarnings({ "unchecked", "rawtypes" })
	public static Object compatibleTypeConvert(Object value, Class<?> type) {
        if(value == null || type == null || type.isAssignableFrom(value.getClass())) {
        	return value;
        }
        if(value instanceof String) {
            String string = (String) value;
            if (char.class.equals(type) || Character.class.equals(type)) {
                if(string.length() != 1) {
                    throw new IllegalArgumentException(String.format("CAN NOT convert String(%s) to char!" +
                            " when convert String to char, the String MUST only 1 char.", string));
                }
                return string.charAt(0);
            } else if(type.isEnum()) {
                return Enum.valueOf((Class<Enum>)type, string);
            } else if(type == BigInteger.class) {
                return new BigInteger(string);
            } else if(type == BigDecimal.class) {
                return new BigDecimal(string);
            } else if(type == Short.class || type == short.class) {
                return new Short(string);
            } else if(type == Integer.class || type == int.class) {
                return new Integer(string);
            } else if(type == Long.class || type == long.class) {
                return new Long(string);
            } else if(type == Double.class || type == double.class) {
                return new Double(string);
            } else if(type == Float.class || type == float.class) {
                return new Float(string);
            }  else if(type == Byte.class || type == byte.class) {
                return new Byte(string);
            } else if(type == Boolean.class || type == boolean.class) {
                return new Boolean(string);
            } else if(type == Date.class) {
                try {
                    return new SimpleDateFormat(DATE_FORMAT).parse((String) value);
                } catch (ParseException e) {
                    throw new IllegalStateException("Failed to parse date " + value + " by format " + DATE_FORMAT + ", cause: " + e.getMessage(), e);
                }
            } else if (type == Class.class) {
                try {
                    return ReflectUtils.name2class((String)value);
                } catch (ClassNotFoundException e) {
                    throw new RuntimeException(e.getMessage(), e);
                }
            }
        }
    }

```
> 上面的方法入参type，是从获取到的具体method中拿到的，value则是消费端传输过来的。因此，这个方法，就是在如果存在请求参数类型和具体执行method的参数不一致时，则进行兼容转换到正确的参数类型，完成调用。  
> 所以，当我们通过泛化传来"12345678"的时候，可以自动转换为long类型，但是如果参数是"E12345678",则自动转换就会失败，抛出异常。

### 2.3 dubbo泛化调用之参数类型1

2.2节，我们知道如果具体的参数值和实际参数类型不一致的时候，dubbo框架会尝试一次自动类型转换，来保证尽可能成功。但是，如果我们在demo中，修改如下：
``` java
        Object o = service.$invoke("queryName",new String[]{"java.lang.String"},new Object[]{"12"});
```
会怎样？
看demo执行结果如下：
``` 
com.alibaba.dubbo.rpc.RpcException: io.github.ketao1989.dubbo.api.IUserBiz.queryName(java.lang.String)
	at com.alibaba.dubbo.rpc.filter.GenericFilter.invoke(GenericFilter.java:107)
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
	at com.alibaba.dubbo.rpc.filter.ClassLoaderFilter.invoke(ClassLoaderFilter.java:38)
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
	at com.alibaba.dubbo.rpc.filter.EchoFilter.invoke(EchoFilter.java:38)
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
	at com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol$1.reply(DubboProtocol.java:108)
	at com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.handleRequest(HeaderExchangeHandler.java:84)
	at com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.received(HeaderExchangeHandler.java:170)
	at com.alibaba.dubbo.remoting.transport.DecodeHandler.received(DecodeHandler.java:52)
	at com.alibaba.dubbo.remoting.transport.dispatcher.ChannelEventRunnable.run(ChannelEventRunnable.java:82)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.NoSuchMethodException: io.github.ketao1989.dubbo.api.IUserBiz.queryName(java.lang.String)
	at java.lang.Class.getMethod(Class.java:1786)
	at com.alibaba.dubbo.common.utils.ReflectUtils.findMethodByMethodSignature(ReflectUtils.java:816)
	at com.alibaba.dubbo.rpc.filter.GenericFilter.invoke(GenericFilter.java:57)
	... 13 more

```
抛出了异常，因此，如果消费端的类型都不对，是不可能执行成功的。

看代码逻辑，请求执行会到服务端，然后在运行到如下代码时出错。
``` java
Method method = ReflectUtils.findMethodByMethodSignature(invoker.getInterface(), name, types);

	/**
	 * 根据方法签名从类中找出方法。
	 * 
	 * @param clazz 查找的类。
	 * @param methodName 方法签名，形如method1(int, String)。也允许只给方法名不参数只有方法名，形如method2。
	 * @return 返回查找到的方法。
	 * @throws NoSuchMethodException
	 * @throws ClassNotFoundException  
	 * @throws IllegalStateException 给定的方法签名找到多个方法（方法签名中没有指定参数，又有有重载的方法的情况）
	 */
	public static Method findMethodByMethodSignature(Class<?> clazz, String methodName, String[] parameterTypes) throws NoSuchMethodException, ClassNotFoundException {
        Class<?>[] types = new Class<?>[parameterTypes.length];//java.lang.String
        for (int i = 0; i < parameterTypes.length; i ++) {
            types[i] = ReflectUtils.name2class(parameterTypes[i]);
        }
        method = clazz.getMethod(methodName, types);
    }
    // Class反射拿到method，就会出错
    public Method getMethod(String name, Class<?>... parameterTypes) throws NoSuchMethodException, SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        Method method = getMethod0(name, parameterTypes, true);
        if (method == null) {
            throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
        }
        return method;
    }
```
> findMethodByMethodSignature方法会根据clazz类，methodName方法名和方法参数类型来反射拿到具体的方法，但是因为不存在对应给的method，所以抛异常失败。

### 2.4 dubbo泛化调用之参数类型2

在业务中，我们的dubbo调用方式，会存在复杂对象，不是原始类型。那么，如果我对象类型是正确的，但是对象内部的类型不对呢？

先看一下demo：

``` java
public class UserRequest {

  private long id;
  private String name;
}

public interface IUserBiz {

    public String addUser(UserRequest request);
}

public class UserBizConsumer {

    public static void main(String[] args) {
        // ...... 省略一些重复代码
        Map<String,Object> value = new HashMap<>();
        value.put("id","1234567");
        value.put("name","kk123");
        Object o2 = service.$invoke("addUser",new String[]{"io.github.ketao1989.dubbo.api.UserRequest"},new Object[]{value});
        System.out.println(o2);
        
    }
}

```

> 如上代码，可以看到，id是long类型，但是参数里面，id是一个字符串，但是id是在mapl里面。这个case，在执行的时候，并不会报错，而是返回值，例如：`{"id":1234567,"name":"kk123"}`

因此，我们可以看到泛化调用，即使type是个对象，对象里面的属性类型存在问题，也不一定会出错。在最底层，和2.2节一样，会对方法类型和值类型不一致的情况，进行兼容处理。

由于请求参数值得类型，不是Primitives类型，所以会进行特殊处理，例如我们这个demo里面是一个Map，所以走如下代码逻辑：

``` java
  private static Object realize0(Object pojo, Class<?> type, Type genericType, final Map<Object, Object> history) {

    if (pojo instanceof Map<?, ?> && type != null) {

      Map<Object, Object> map;
      // 返回值类型不是方法签名类型的子集 并且 不是接口类型
      if (!type.isInterface() && !type.isAssignableFrom(pojo.getClass())) {
        try {
          map = (Map<Object, Object>) type.newInstance(); // 由于类型不对，一般这里会抛异常
        } catch (Exception e) {
          //ignore error
          map = (Map<Object, Object>) pojo; // 执行这里的逻辑，强转map
        }
      }
      Object dest = newInstance(type);//反射构造函数，new个对象
      
      for (Map.Entry<Object, Object> entry : map.entrySet()) { // 遍历每个map元素
        Object key = entry.getKey();
        if (key instanceof String) { // 支持map，获取属性名称
          String name = (String) key;
          Object value = entry.getValue();
          if (value != null) {
            Method method = getSetterMethod(dest.getClass(), name, value.getClass()); //根据字段名，获取setter方法
            if (method != null) {
              if (!method.isAccessible()) {
                method.setAccessible(true);
              }
              Type ptype = method.getGenericParameterTypes()[0];
              // 对请求值进行兼容改造处理，例如string转成需要的long，所以，转换成功就不会报错
              value = realize0(value, method.getParameterTypes()[0], ptype, history);
              try {
                method.invoke(dest, value);// set方法来设置value到dest对象中
              } catch (Exception e) {
              }}}}
        return dest;
      }}
    return pojo;
  }

```

因此，如果我们对复杂对象中的某一个属性类型，在构造map的时候没有设置对，但是在dubbo中，这两个类型之间可以进行兼容转换，则不会出错。当然，如果是"E123456"，由于无法转成long类型，所以还是会报错的。

## 3. dubbo 序列化

### 3.1 不兼容的hessian2序列化

最后，简单聊聊dubbo的序列化。如果，我们没有特别的进行配置，则dubbo默认使用自定义的hessian2进行序列化操作。
``` java
public class CodecSupport {

     public static final String  DEFAULT_REMOTING_SERIALIZATION     = "hessian2";

    public static Serialization getSerialization(URL url) {
        return ExtensionLoader.getExtensionLoader(Serialization.class).getExtension(
            url.getParameter(Constants.SERIALIZATION_KEY, DEFAULT_REMOTING_SERIALIZATION));
    }
}
```

关于hessian2的序列化，我们来一个demo看看：

``` java
import com.alibaba.dubbo.common.serialize.ObjectInput;
import com.alibaba.dubbo.common.serialize.ObjectOutput;
import com.alibaba.dubbo.common.serialize.support.hessian.Hessian2Serialization;

public static void main(String[] args) throws IOException, ClassNotFoundException {

        // UserRequestOth 和 UserRequest 一致，只是id是string
        UserRequestOth request = new UserRequestOth("11111","rpc serial");
        
        ByteArrayOutputStream bos = new ByteArrayOutputStream(512);
        Hessian2Serialization hessian2Serialization = new Hessian2Serialization();

        ObjectOutput output = hessian2Serialization.serialize(null, bos);
        output.writeObject(request);
        output.flushBuffer();

        ByteArrayInputStream inputStream = new ByteArrayInputStream(bos.toByteArray());
        ObjectInput input = hessian2Serialization.deserialize(null, inputStream);
        UserRequest res  = input.readObject(UserRequest.class);

        System.out.println(res);

    }

```

运行的时候，会抛出异常，如：

```
Exception in thread "main" com.alibaba.com.caucho.hessian.io.HessianFieldException: io.github.ketao1989.dubbo.api.UserRequest.id: expected long at 0x5 java.lang.String (11111)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.logDeserializeError(JavaDeserializer.java:671)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer$LongFieldDeserializer.deserialize(JavaDeserializer.java:515)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:233)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:157)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2067)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1592)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1576)
	at com.alibaba.dubbo.common.serialize.support.hessian.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:94)
	at io.github.ketao1989.serial.SerialTest.main(SerialTest.java:63)
Caused by: com.alibaba.com.caucho.hessian.io.HessianProtocolException: expected long at 0x5 java.lang.String (11111)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.error(Hessian2Input.java:2720)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.expect(Hessian2Input.java:2691)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readLong(Hessian2Input.java:888)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer$LongFieldDeserializer.deserialize(JavaDeserializer.java:511)
	... 7 more

```

> 综上，如果我们client端的jar包和server端的jar包中，关于某个入参对象中的一个属性对象不一致的话，由于接收到请求后，首先就是使用自定义hessian2进行反序列，如果数据类型匹配兼容不了，dubbo调用会抛出异常返回失败。

### 3.2 简单兼容的dubbo处理

如果，将上面示例代码中 `UserRequest`和`UserRequestOth`互换下，会发现dubbo调用，并不会发送失败。为什么呢？因为，long 到 Sting类型的转换是兼容的，也就是说，任何一个类型是long的数据，都可以被转换成String类型，而不出现数据不正确的情况。

当然，我们可以看看dubbo自定义的hessian2代码中这块逻辑的处理。

``` java

  public Object readObject(AbstractHessianInput in,
			   Object obj,
			   String []fieldNames)
    throws IOException
  {
    try {
      int ref = in.addRef(obj);

      for (int i = 0; i < fieldNames.length; i++) {
        String name = fieldNames[i];
        
        FieldDeserializer deser = (FieldDeserializer) _fieldMap.get(name);

        if (deser != null)
	  deser.deserialize(in, obj);
        else
          in.readObject();
      }

      Object resolve = resolve(obj);

      if (obj != resolve)
	in.setRef(ref, resolve);

      return resolve;
    } catch (IOException e) {
      throw e;
    } catch (Exception e) {
      throw new IOExceptionWrapper(obj.getClass().getName() + ":" + e, e);
    }
  }
  
  static class StringFieldDeserializer extends FieldDeserializer {
    private final Field _field;

    StringFieldDeserializer(Field field)
    {
      _field = field;
    }
    
    void deserialize(AbstractHessianInput in, Object obj)
      throws IOException
    {
      String value = null;
      
      try {
	value = in.readString();
	
	_field.set(obj, value);
      } catch (Exception e) {
        logDeserializeError(_field, obj, value, e);
      }
    }
  }


```
> 代码是根据最后反序列化出的对象类型来进行deserialize的，因此，针对对象，默认使用`JavaDeserializer`类来操作。把long转换为String，因此，目标字段类型是String，所以，最终使用`StringFieldDeserializer`来操作id这个字段的反序列化。

那，由于调用方传来的是long，我们使用readString是怎么执行下去的呢？

dubbo 自定义重写了hessian2的 Hessian2Input，所以基本上，dubbo把hessian2的序列化代码重写了。下面看看Hessian2Input的readString是如何实现的（部分代码片段）：

``` java
  public String readString()
    throws IOException
  {
    int tag = read();

    switch (tag) {
    case 'N':
      return null;
    case 'T':
      return "true";
    case 'F':
      return "false";

      // direct long
    case 0xd8: case 0xd9: case 0xda: case 0xdb:
    case 0xdc: case 0xdd: case 0xde: case 0xdf:
      
    case 0xe0: case 0xe1: case 0xe2: case 0xe3:
    case 0xe4: case 0xe5: case 0xe6: case 0xe7:
    case 0xe8: case 0xe9: case 0xea: case 0xeb:
    case 0xec: case 0xed: case 0xee: case 0xef:
      return String.valueOf(tag - BC_LONG_ZERO);

    case 'L':
      return String.valueOf(parseLong());

    case 'S':
    case BC_STRING_CHUNK:
      _isLastChunk = tag == 'S';
      _chunkLength = (read() << 8) + read();

      _sbuf.setLength(0);
      int ch;

      while ((ch = parseChar()) >= 0)
        _sbuf.append((char) ch);

      return _sbuf.toString();

      // 0-byte string
    case 0x00: case 0x01: case 0x02: case 0x03:
    case 0x04: case 0x05: case 0x06: case 0x07:
    case 0x08: case 0x09: case 0x0a: case 0x0b:
    case 0x0c: case 0x0d: case 0x0e: case 0x0f:

    case 0x10: case 0x11: case 0x12: case 0x13:
    case 0x14: case 0x15: case 0x16: case 0x17:
    case 0x18: case 0x19: case 0x1a: case 0x1b:
    case 0x1c: case 0x1d: case 0x1e: case 0x1f:
      _isLastChunk = true;
      _chunkLength = tag - 0x00;

      _sbuf.setLength(0);

      while ((ch = parseChar()) >= 0)
        _sbuf.append((char) ch);

      return _sbuf.toString();

    case 0x30: case 0x31: case 0x32: case 0x33:
      _isLastChunk = true;
      _chunkLength = (tag - 0x30) * 256 + read();

      _sbuf.setLength(0);

      while ((ch = parseChar()) >= 0)
        _sbuf.append((char) ch);

      return _sbuf.toString();

    default:
      throw expect("string", tag);
    }
  }

```

因此，我们发现，如果是long数字类型，则会使用String.valueOf来进行转换为string类型，因此我们的代码处理值为long类型，目标为string类型时，没有出现任何错误，并且结果也是正确的原因所在。


## 4. 最后

虽然，现在大部分接口服务都是通过泛化走网关对外提供的，而dubbo的泛化调用会对某些参数类型进行兼容处理。
但是，最为服务化的接口规范，我们显然是不允许修改api包中的已存在属性的类型（dubbo hessian2做了某些类型转换的兼容，并不是一个允许的好借口）。
一般来说，对于某些字段的类型可能不再支持当前业务发展时，采取的方式都是新增字段，来支持新功能，并且通过业务逻辑来兼容老的版本。

之所以，这样子处理，主要是因为：

+ 调用方业务收到影响。如果存在业务方使用jar包调用，hessian2无法兼容，那将是灾难性的后果，调用的请求将全部失败，尤其是有的系统会将dubbo反序列化失败的日志，放在单独的目录文件下，导致观察业务日志时，并没有错误的地方。这个时候，除了业务方反馈，还有就是查看调用方请求量的监控来主动发现。

+ 存在某些不兼容的场景。比如，将string--->long调整，可能当前某些时候能正常运行。后期业务方，发现自己需要业务调整，新业务的参数值变成`New1234`，这个时候，就会发现调用失败了。

+ 最后，但是也是最重要的，这就是api的规范，遵循规范，可以避免自己和别人掉进坑里。