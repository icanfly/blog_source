title: dubbo服务可用性监控手段
date: 2016-06-20
tags:
 - dubbo
 - 问题分析

categories:
 - 原创文章

---

# 前言

随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，分布式服务架构以及流动计算架构势在必行，亟需一个治理系统确保架构有条不紊的演进。在采用Dubbo作为服务化框架的过程中需要对服务接口可用性进行监控，程序需要通过监控平台去监控所有业务Dubbo服务接口的可用性。希望的是在1分钟内如果有服务挂掉会有监控报警发出，并且有监控页面显示监控数据，且有数据报表产出某个时间单位内的服务可用性。

服务可用性的定义： 在一段时间内，服务可用时间/（服务可用时间+服务不可用时间）* 100%

<!-- more -->

这里有几个问题需要解决：

- 监控平台监控怎么与业务平台松耦合？

首先监控平台不可能去引用所有一大堆的其它业务系统的Dubbo接口的SDK，这样无法保证稳定且无法扩展。

> 如果有新的业务需要监控，是不是需要新加入该业务的SDK接口包呢？
  如果某个业务平台的SDK升级了，是不是监控平台也需要跟着升级SDK呢？

所以首先想到的就是监控平台能够在不引入业务平台的SDK的前提下就可以监控到业务平台的服务状况。这里想到的就是使用Dubbo提供的泛化调用，可以在事先不知道业务方SDK的情况下调用业务接口。

- 监控平台怎么对业务判活？

如果要求业务方在自己的服务中强行嵌入一个健康检测的接口，对于业务方来说总有种被强奸的感觉。幸好Dubbo也提供了一个回声测试接口，什么是回声测试？就是你发给它什么东西，它返回给你什么东西。就像你对着大山喊了一声“hi”，大山回声一句“hi”一样的道理，证明对方是有能力回复你的，从某种程序上证明对方仍然处于存活状态。


在Dubbo中，泛接口调用方式主要用于客户端没有API接口及模型类元的情况，参数及返回值中的所有POJO均用Map表示，通常用于框架集成，比如：实现一个通用的服务测试框架，可通过GenericService调用所有服务实现。回声测试用于检测服务是否可用，回声测试按照正常请求流程执行，能够测试整个调用是否通畅，可用于监控。所有服务自动实现EchoService接口，只需将任意服务引用强制转型为EchoService，即可使用。


# dubbo回声测试

```xml
<dubbo:reference id="memberService" interface="com.xxx.MemberService" />
```

```java
MemberService memberService = ctx.getBean("memberService"); // 远程服务引用

EchoService echoService = (EchoService) memberService; // 强制转型为EchoService

String status = echoService.$echo("OK"); // 回声测试可用性

assert(status.equals("OK"));
```

# dubbo泛化调用

```xml
<dubbo:reference id="barService" interface="com.foo.BarService" generic="true" />
```

```java
GenericService barService = (GenericService) applicationContext.getBean("barService");
Object result = barService.$invoke("sayHello", new String[] { "java.lang.String" }, new Object[] { "World" });
```

```java
import com.alibaba.dubbo.rpc.service.GenericService; 
... 

// 引用远程服务 
ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>(); // 该实例很重量，里面封装了所有与注册中心及服务提供方连接，请缓存
reference.setInterface("com.xxx.XxxService"); // 弱类型接口名 
reference.setVersion("1.0.0"); 
reference.setGeneric(true); // 声明为泛化接口 

GenericService genericService = reference.get(); // 用com.alibaba.dubbo.rpc.service.GenericService可以替代所有接口引用 

// 基本类型以及Date,List,Map等不需要转换，直接调用 
Object result = genericService.$invoke("sayHello", new String[] {"java.lang.String"}, new Object[] {"world"}); 

// 用Map表示POJO参数，如果返回值为POJO也将自动转成Map 
Map<String, Object> person = new HashMap<String, Object>(); 
person.put("name", "xxx"); 
person.put("password", "yyy"); 
Object result = genericService.$invoke("findPerson", new String[]{"com.xxx.Person"}, new Object[]{person}); // 如果返回POJO将自动转成Map 

...

```
假设存在POJO如：
```java
package com.xxx;

public class PersonImpl implements Person {
    private String name;
    private String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```
则POJO数据：
```java
Person person = new PersonImpl(); 
person.setName("xxx"); 
person.setPassword("yyy");
```
可用下面Map表示：
```
Map<String, Object> map = new HashMap<String, Object>(); 
map.put("class", "com.xxx.PersonImpl"); // 注意：如果参数类型是接口，或者List等丢失泛型，可通过class属性指定类型。
map.put("name", "xxx"); 
map.put("password", "yyy");
```

以上引用至Dubbo官方文档：[泛化引用](http://dubbo.io/user-guide/demos/%E6%B3%9B%E5%8C%96%E5%BC%95%E7%94%A8.html)

# 回声测试与泛化调用之冲突

通过泛化引用与回声测试两者结合，刚好能满足我们的监控需求，监控平台通过注册中心获取所有的服务接口，并通过泛化引用方式引用服务，并调用服务的回声接口测试可用性。

一切看起来都那么美好，但是现实总是那么残酷。对于Dubbo目前的处境来说，泛化引用和回声测试同时使用时会产生不兼容，究其原因是因为：`Dubbo的泛化引用调用和回声测试是两个不同的Filter，泛化调用Filter被用于客户端执行，而回声测试被用于服务端，在进行回声测试时并没有对泛化调用进行回应`


# 冲突分析

Dubbo回声测试的Filter：
```java
@Activate(group = Constants.PROVIDER, order = -110000)
public class EchoFilter implements Filter {

	public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
		if(inv.getMethodName().equals(Constants.$ECHO) && inv.getArguments() != null && inv.getArguments().length == 1 )
			return new RpcResult(inv.getArguments()[0]);
		return invoker.invoke(inv);
	}

}
```

Dubbo泛化引用调用的Filter：
```java
@Activate(group = Constants.CONSUMER, value = Constants.GENERIC_KEY, order = 20000)
public class GenericImplFilter implements Filter {
    
    private static final Logger logger = LoggerFactory.getLogger(GenericImplFilter.class);

    private static final Class<?>[] GENERIC_PARAMETER_TYPES = new Class<?>[] {String.class, String[].class, Object[].class};

    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        String generic = invoker.getUrl().getParameter(Constants.GENERIC_KEY);
        if (ProtocolUtils.isGeneric(generic)
                && ! Constants.$INVOKE.equals(invocation.getMethodName())
                && invocation instanceof RpcInvocation) {
            RpcInvocation invocation2 = (RpcInvocation) invocation;
            String methodName = invocation2.getMethodName();
            Class<?>[] parameterTypes = invocation2.getParameterTypes();
            Object[] arguments = invocation2.getArguments();
            
            String[] types = new String[parameterTypes.length];
            for (int i = 0; i < parameterTypes.length; i ++) {
                types[i] = ReflectUtils.getName(parameterTypes[i]);
            }

            Object[] args;
            if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                args = new Object[arguments.length];
                for(int i = 0; i < arguments.length; i++) {
                    args[i] = JavaBeanSerializeUtil.serialize(arguments[i], JavaBeanAccessor.METHOD);
                }
            } else {
                args = PojoUtils.generalize(arguments);
            }
            
            invocation2.setMethodName(Constants.$INVOKE);
            invocation2.setParameterTypes(GENERIC_PARAMETER_TYPES);
            invocation2.setArguments(new Object[] {methodName, types, args});
            Result result = invoker.invoke(invocation2);
            
            if (! result.hasException()) {
                Object value = result.getValue();
                try {
                    Method method = invoker.getInterface().getMethod(methodName, parameterTypes);
                    if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                        if (value == null) {
                            return new RpcResult(value);
                        } else if (value instanceof JavaBeanDescriptor) {
                            return new RpcResult(JavaBeanSerializeUtil.deserialize((JavaBeanDescriptor)value));
                        } else {
                            throw new RpcException(
                                new StringBuilder(64)
                                    .append("The type of result value is ")
                                    .append(value.getClass().getName())
                                    .append(" other than ")
                                    .append(JavaBeanDescriptor.class.getName())
                                    .append(", and the result is ")
                                    .append(value).toString());
                        }
                    } else {
                        return new RpcResult(PojoUtils.realize(value, method.getReturnType(), method.getGenericReturnType()));
                    }
                } catch (NoSuchMethodException e) {
                    throw new RpcException(e.getMessage(), e);
                }
            } else if (result.getException() instanceof GenericException) {
                GenericException exception = (GenericException) result.getException();
                try {
                    String className = exception.getExceptionClass();
                    Class<?> clazz = ReflectUtils.forName(className);
                    Throwable targetException = null;
                    Throwable lastException = null;
                    try {
                        targetException = (Throwable) clazz.newInstance();
                    } catch (Throwable e) {
                        lastException = e;
                        for (Constructor<?> constructor : clazz.getConstructors()) {
                            try {
                                targetException = (Throwable) constructor.newInstance(new Object[constructor.getParameterTypes().length]);
                                break;
                            } catch (Throwable e1) {
                                lastException = e1;
                            }
                        }
                    }
                    if (targetException != null) {
                        try {
                            Field field = Throwable.class.getDeclaredField("detailMessage");
                            if (! field.isAccessible()) {
                                field.setAccessible(true);
                            }
                            field.set(targetException, exception.getExceptionMessage());
                        } catch (Throwable e) {
                            logger.warn(e.getMessage(), e);
                        }
                        result = new RpcResult(targetException);
                    } else if (lastException != null) {
                        throw lastException;
                    }
                } catch (Throwable e) {
                    throw new RpcException("Can not deserialize exception " + exception.getExceptionClass() + ", message: " + exception.getExceptionMessage(), e);
                }
            }
            return result;
        }

        if (invocation.getMethodName().equals(Constants.$INVOKE)
            && invocation.getArguments() != null
            && invocation.getArguments().length == 3
            && ProtocolUtils.isGeneric(generic)) {

            Object[] args = (Object[]) invocation.getArguments()[2];
            if (ProtocolUtils.isJavaGenericSerialization(generic)) {

                for (Object arg : args) {
                    if (!(byte[].class == arg.getClass())) {
                        error(byte[].class.getName(), arg.getClass().getName());
                    }
                }
            } else if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                for(Object arg : args) {
                    if (!(arg instanceof JavaBeanDescriptor)) {
                        error(JavaBeanDescriptor.class.getName(), arg.getClass().getName());
                    }
                }
            }

            ((RpcInvocation)invocation).setAttachment(
                Constants.GENERIC_KEY, invoker.getUrl().getParameter(Constants.GENERIC_KEY));
        }
        return invoker.invoke(invocation);
    }

    private void error(String expected, String actual) throws RpcException {
        throw new RpcException(
            new StringBuilder(32)
                .append("Generic serialization [")
                .append(Constants.GENERIC_SERIALIZATION_NATIVE_JAVA)
                .append("] only support message type ")
                .append(expected)
                .append(" and your message type is ")
                .append(actual).toString());
    }

}
```
从以上两个源码中也可以看出，泛化调用中没有对回声测试进行处理。

# 冲突解决

解决冲突有两种方式：

- 修改回声测试调用Filter，并对泛化调用进行处理

- 增加新自定义的Provider端Filter，并且排在回声调用Filter之前对泛化调用中的回声测试进行处理返回。

由于Dubbo的扩展性做得非常棒，这里只需要自己定义一个Filter来实现就可以了：


GenericEchoFilter.java:
```java
@Activate(
    group = {"provider"},
    order = -999999
)
public class GenericEchoFilter implements Filter {
    private Logger logger = LoggerFactory.getLogger(GenericEchoFilter.class);

    public GenericEchoFilter() {
    }

    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if(invocation.getMethodName().equals("$invoke") && invocation.getArguments() != null && invocation.getArguments().length == 3 && !ProtocolUtils.isGeneric(invoker.getUrl().getParameter("generic"))) {
            Object[] arguments = invocation.getArguments();
            Object realMethod = arguments[0];
            String[] argsTypes = (String[])((String[])arguments[1]);
            Object[] args = (Object[])((Object[])arguments[2]);
            if("$echo".equals(realMethod) && argsTypes != null && argsTypes.length == 1 && args != null && args.length == 1) {
                return new RpcResult(args[0]);
            }
        }

        return invoker.invoke(invocation);
    }
}
```

注意以上的`order`属性值要大于回声测试Filter中的order值，这样才能保证先于回声测试就进行响应。

添加了这个后需要在自己的工程或者工具jar包META-INF/dubbo/下新增filter设定文件com.alibaba.dubbo.rpc.Filter，内容如下：
```
genericecho=com.xxx.dubbo.filter.GenericEchoFilter
```
这样操作过后，Dubbo在启动时会通过SPI方式自动扫描和加载自定义的Filter，这样我们的自定义Filter就自动生效了。

# 总结

Dubbo是阿里开源的一款精品服务化RPC框架，它的扩展点非常多，有着非常强的可定制化功能。本文中基于监控平台的远程泛化引用调用和回声测试中通过自定义的Filter完美地解决了服务化监控的问题。最多的Dubbo扩展特性还需要更多的探索与研究！