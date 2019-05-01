
---
title: 【源码分析】dubbo的SPI
date: 2018-04-19 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - java
  - dubbo
  - 源码
categories:
  - 技术
---

## SPI

SPI全称为（Service Provider Interface），是JDK提供的一种服务发现机制。一般常用于开发者对框架的扩展。具体使用是在Jar包的`META-INF/services` 目录下新建一个无格式文件（文件格式为必须UTF-8），文件名称为接口的全限定名，文件内容可以有多行，每一行都是该接口具体实现类的全限定类名。在JAVA中这样获取实现类：

```java
ServiceLoader<DynamicConfig> factories = ServiceLoader.load(DynamicConfig.class);
	for (DynamicConfig dynamicConfig : factories) {
		System.out.println(dynamicConfig);
	}
```

需要注意的是，实现类必须提供默认的构造函数，否则无法实例化。

具体参考[Service Provider](https://docs.oracle.com/javase/1.5.0/docs/guide/jar/jar.html#Service%20Provider)

## dubbo对SPI的扩展

### 介绍

dubbo也用了SPI思想，不过没有用JDK的SPI机制，自己实现了一套。看看官方文档的介绍：

> Dubbo 的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。
> 
> Dubbo 改进了 JDK 标准的 SPI 的以下问题：
>
> JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
> 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 getName() 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户
> 执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
> 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。
> 
> 约定：
> 在扩展类的 jar 包内，放置扩展点配置文件 META-INF/dubbo/接口全限定名，内容为：配置名=扩展实现类全限定名，多个实现类用换行符分隔。
> 
> 示例：
> 以扩展 Dubbo 的协议为例，在协议的实现 jar 包内放置文本文件：META-INF/dubbo/org.apache.dubbo.rpc.Protocol，内容为：
>
> xxx=com.alibaba.xxx.XxxProtocol
> 实现类内容：
> 
> package com.alibaba.xxx;
>  
> import org.apache.dubbo.rpc.Protocol;
>  
> public class XxxProtocol implements Protocol { 
>     // ...
> }
> 
> 配置模块中的配置
> 
> Dubbo 配置模块中，扩展点均有对应配置属性或标签，通过配置指定使用哪个扩展实现。比如：
> <dubbo:protocol name="xxx" />


`Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();`

`Protocol` 接口，`Dubbo` 要判断一下，在系统运行的时候，应该选用这个 `Protocol` 接口的哪个实现类来实例化对象来使用呢？

微内核，可插拔，大量的组件，`Protocol`负责rpc调用的东西，你可以实现自己的rpc调用组件，实现`Protocol`接口，给自己的一个实现类即可。

这行代码就是dubbo里大量使用的，就是对很多组件，都是保留一个接口和多个实现，然后在系统运行的时候动态根据配置去找到对应的实现类。如果你没配置，那就走默认的实现好了，没问题。

### 特性

dubbo的SPI主要有以下几种特性：

* 支持 `key-value pair` 形式配置的扩展式JDK SPI
* 扩展点自动包装
* 扩展点自动装配
* 扩展点自适应
* 扩展点自动激活


1）支持 `key-value pair` 形式配置的扩展式JDK SPI？

什么意思，就是以K-V的形式保存实现类，以 `Protocol` 接口举例：

```java
@SPI("dubbo")  // 表示这个注解支持SPI扩展，dubbo做为key去配置文件找对应实现
public interface Protocol {  
      
    int getDefaultPort();  
  
    @Adaptive  
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;  
  
    @Adaptive  
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;  

    void destroy();  
  
} 
```

在dubbo自己的jar里，在`/META_INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol`文件中：

```java
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
http=com.alibaba.dubbo.rpc.protocol.http.HttpProtocol
hessian=com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol
```

`@SPI("dubbo")`说的是，通过SPI机制来提供实现类，实现类是通过dubbo作为默认key去配置文件里找到的，配置文件名称与接口全限定名一样的，通过dubbo作为key可以找到默认的实现了就是com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol。

dubbo的默认网络通信协议，就是dubbo协议，用的DubboProtocol

2）扩展点自动包装？

当dubbo发现我们某个扩展点的实现类拥有一个拷贝构造函数时，变会认为这个类是包装类。什么是拷贝构造函数，如下：

```java
package com.alibaba.xxx;
 
import org.apache.dubbo.rpc.Protocol;
 
public class XxxProtocolWrapper implements Protocol {
    Protocol impl;
 
    public XxxProtocolWrapper(Protocol protocol) { // 我是拷贝构造函数
		 impl = protocol; 
	}
 
    // 接口方法做一个操作后，再调用extension的方法
    public void refer() {
        //... 一些操作
        impl.refer();
        // ... 一些操作
    }
 
    // ...
}

```

`XxxProtocolWrapper`类实现了`Protocol`接口，并且构造方法入参为 `Protocol`，即可认为是包装类。


3） 扩展点自动装配？

这个功能类似于Spring的IOC，具体机制是当dubbo加载扩展点时，发现这个类中的set方法是另外一个扩展点，便会获取并自动注入。如下：


示例：有两个为扩展点 CarMaker（造车者）、WheelMaker (造轮者)

```java
public interface CarMaker {
    Car makeCar();
}
 
public interface WheelMaker {
    Wheel makeWheel();
}
```

CarMaker 的一个实现类：

```java
public class RaceCarMaker implements CarMaker {
    WheelMaker wheelMaker;
 
    public setWheelMaker(WheelMaker wheelMaker) {  //此方法会被dubbo发现并尝试注入WheelMaker的扩展点实现
        this.wheelMaker = wheelMaker;
    }
 
    public Car makeCar() {
        // ...
        Wheel wheel = wheelMaker.makeWheel();
        // ...
        return new RaceCar(wheel, ...);
    }
}
```

这里带来另一个问题，ExtensionLoader 要注入依赖扩展点时，如何决定要注入依赖扩展点的哪个实现。在这个示例中，即是在多个WheelMaker 的实现中要注入哪个。

这个问题在下面一点扩展点自适应中说明。

4） 扩展点自适应？

dubbo的扩展点依赖于统一抽象模型URL，在所有的扩展点中均有此参数，如下：

```java
public interface CarMaker {
    Car makeCar(URL url);
}
 
public interface WheelMaker {
 	@Adaptive({"circular", "square"})
    Wheel makeWheel(URL url);
}
```

实现类：

```java
public class RaceCarMaker implements CarMaker {
    WheelMaker wheelMaker;
 
    public setWheelMaker(WheelMaker wheelMaker) {
        this.wheelMaker = wheelMaker;
    }
 
    public Car makeCar(URL url) {
        // ...
        Wheel wheel = wheelMaker.makeWheel(url);
        // ...
        return new RaceCar(wheel, ...);
    }
}
```

当上面的代码执行到`wheelMaker.makeWheel(url);`时，会从url中提取指定的key找到对应的实现类，这个key是设置在扩展点接口方法注解上的。

makeWheel()方法，Adaptive实现先查找circular key，如果该Key没有值则找square key值，来决定代理到哪个实际扩展点。当然，由于`@Adaptive`也支持为空的情况，如果为空，那么根据当前扩展点接口的名称做转换，即如果扩展点接口为`RaceCarMaker`，那么会变为`race.car.maker`


5） 扩展点自动激活？


对于集合类扩展点，比如：Filter, InvokerListener, ExportListener, TelnetHandler, StatusChecker 等，可以同时加载多个实现，此时，可以用自动激活来简化配置，如：

```java
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.rpc.Filter;
 
@Activate // 无条件自动激活
public class XxxFilter implements Filter {
    // ...
}
```

或：

```java
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.rpc.Filter;
 
@Activate("xxx") // 当配置了xxx参数，并且参数为有效值时激活，比如配了cache="lru"，自动激活CacheFilter。
public class XxxFilter implements Filter {
    // ...
}
```

或：

```java
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.rpc.Filter;
 
@Activate(group = "provider", value = "xxx") // 只对提供方激活，group可选"provider"或"consumer"
public class XxxFilter implements Filter {
    // ...
}
```



### 实现分析

基本上所有类型的动态导入都是使用adaptive，所以这里以`Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();`分析：

使用注解标记该类是扩展SPI

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {

    /**
     * default extension name
     */
    String value() default "";
}
```

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
	// 1. 检查
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

    // 2. 尝试从缓存中获取，没有则创建新的并返回
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type)); // 触发构造方法
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

先检查有没有带SPI的注解，没有带，直接报错，从缓存中根据这个类型查询对应的ExtensionLoader，查不到就创建一个，再放入缓存中。dubbo中的SPI部分大量利用了本地缓存，后续出现，不再着重讲解了。我们可以看一下他的创建该类型的ExtensionLoader的方法。

```java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    /**
     * type如果是ExtensionFactory类型，那么objectFactory是null,否则是ExtensionFactory类型的适配器类型
     */
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

获取适配的实现：

```java
 public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                    	// 从缓存中取，取不到就创建
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```

接下来是获取到适配器类的Class，利用反射创建适配器类的实例。injectExtension是dubbo的DI，依赖注入。如果适配器类有属性即其他扩展点的set方法，会自动注入（扩展点自动装配）。

```java
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance()); //反射创建实例
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```


```java
private Class<?> getAdaptiveExtensionClass() {
	// 扫描SPI，并将可能的实现放入缓存 读取文件加载放入缓存等等
    getExtensionClasses();
    // 如果通过上面的步骤可以获取到cachedAdaptiveClass直接返回，如果不行的话，就得考虑自己进行利用动态代理创建一个了
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 利用动态代理创建一个适配类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```



扫描SPI的代码：

```java
// 加载文件 extensionClasses 会保存 key=value的映射
private Map<String, Class<?>> loadExtensionClasses() {
	// 设置SPI名称中的类key为默认实现
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```

放入缓存的关键代码：

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    // 判断这个加载的类上，有没有Adaptive的注解，如果有缓存这个类的实现，这个缓存决定了动态替换的实现
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        cacheAdaptiveClass(clazz);
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    } else {
        ...
    }
}
```

如果cachedAdaptiveClass不为空就返回，什么情况下不为空？当扩展类上打上@Adaptive注解的时候，就会将这个类直接返回。如果没有上注解，怎么办，就得自己生成了，也就是createAdaptiveExtensionClass

```java
private Class<?> createAdaptiveExtensionClass() {
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```



接口Ext6生成的动态代理code大致如下：
```
@SPI
public interface Ext6 {
    @Adaptive
    String echo(URL url, String s);
}
```

```java
package org.apache.dubbo.common.extension.ext6_inject;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Ext6$Adaptive implements org.apache.dubbo.common.extension.ext6_inject.Ext6 {

	public java.lang.String echo( org.apache.dubbo.common.URL arg0, java.lang.String arg1 ){
		if ( arg0 == null )
			throw new IllegalArgumentException( "url == null" );
		org.apache.dubbo.common.URL	url	= arg0;
		String	extName = url.getParameter( "ext6" );
		// 这里是关键，需要增加参数ext6，value即为具体实现类的key，这样这个动态替换便实现了
		if ( extName == null )
			throw new IllegalStateException( "Failed to get extension (org.apache.dubbo.common.extension.ext6_inject.Ext6) name from url (" + url.toString() + ") use keys([ext6])" );
		org.apache.dubbo.common.extension.ext6_inject.Ext6 extension = (org.apache.dubbo.common.extension.ext6_inject.Ext6)ExtensionLoader.getExtensionLoader( org.apache.dubbo.common.extension.ext6_inject.Ext6.class ).getExtension( extName );
		return(extension.echo( arg0, arg1 ) );
	}
}
```

这里的关键是需要增加参数ext6，value即为具体实现类的key，这样这个动态替换便实现了。如何增加参数？

```java
Ext6 ext = ExtensionLoader.getExtensionLoader(Ext6.class).getAdaptiveExtension();
URL url = new URL("p1", "1.2.3.4", 1010, "path1");
url = url.addParameters("ext6", "impl1");  //增加关键参数
// 这样其实已经指定实现类 key 为 impl1 的类了
impl1=org.apache.dubbo.common.extension.ext6_inject.impl.Ext6Impl1
impl2=org.apache.dubbo.common.extension.ext6_inject.impl.Ext6Impl2
```

如果是 `@Adaptive({"test","test2","test3"})` 这种情况生成的代码：

```java
String extName = url.getParameter("test", url.getParameter("test2", url.getParameter("test3")));
```

就比较有意思了，`getParameter(String key, String defaultValue)`，先获取test，没找到则test2，最后test3，这也印证了上面的规则。


所以，我们可以简单的看出一些关于`@Adaptive`的规则，当标注在实现类上时优先级最高，当标注在方法上时次之，最后才是不标注默认使用扩展接口类名并以.分割作为key.


### 简单玩法

如果想要动态替换掉默认的实现类，需要使用@Adaptive接口，Protocol接口中，有两个方法加了@Adaptive注解，就是说那俩接口会被代理实现。

比如这个Protocol接口搞了俩@Adaptive注解标注了方法，在运行的时候会针对Protocol生成代理类，这个代理类的那俩方法里面会有代理代码，代理代码会在运行的时候动态根据url中的protocol来获取那个key，默认是dubbo，你也可以自己指定，你如果指定了别的key，那么就会获取别的实现类的实例了。

通过这个url中的参数不同，就可以控制动态使用不同的组件实现类。

好吧，那下面来说说怎么来自己扩展dubbo中的组件

自己写个工程，要是那种可以打成jar包的，里面的src/main/resources目录下，搞一个META-INF/services，里面放个文件叫：com.alibaba.dubbo.rpc.Protocol，文件里搞一个my=io.github.pleuvoir.MyProtocol。自己把jar弄到nexus私服里去。

然后自己搞一个dubbo provider工程，在这个工程里面依赖你自己搞的那个jar，然后在spring配置文件里给个配置：

<dubbo:protocol name="my" port="20000" />

这个时候provider启动的时候，就会加载到我们jar包里的my=io.github.pleuvoir.MyProtocol这行配置里，接着会根据你的配置使用你定义好的MyProtocol了，这个就是简单说明一下，你通过上述方式，可以替换掉大量的dubbo内部的组件，就是扔个你自己的jar包，然后配置一下即可。

dubbo里面提供了大量的类似上面的扩展点，就是说，你如果要扩展一个东西，只要自己写个jar，让你的consumer或者是provider工程，依赖你的那个jar，在你的jar里指定目录下配置好接口名称对应的文件，里面通过key=实现类。

然后对应的组件，用类似<dubbo:protocol>用你的哪个key对应的实现类来实现某个接口，你可以自己去扩展dubbo的各种功能，提供你自己的实现。


## 参考资料

[http://dubbo.apache.org/zh-cn/docs/dev/SPI.html](http://dubbo.apache.org/zh-cn/docs/dev/SPI.html "dubbo扩展点加载官方文档")