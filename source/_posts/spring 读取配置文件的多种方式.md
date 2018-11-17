
---
title: spring 读取配置文件的多种方式
date: 2017-05-24 19:19:00
author: pleuvoir
img: /images/code.jpg
tags:
  - spring
  - 配置文件
categories:
  - 技术
---


### 一、使用 @Value

注意：

项目中必须有一个配置类声明了 `@PropertySource(value = { "xxx.properties" })`, 当然如下这样也是可以的。

```xml
<context:property-placeholder
            location="classpath:config/dev/database-dev.properties,
                      classpath:config/dev/rabbitmq-dev.properties"/>
```

如果没有声明并且 `filed` 皆为 `string` 类型，项目启动的时候可能不会报错，但是使用时会发现对应的 `filed` 值并没有被解析，而是原始表达式如 `${example.name}` 。
如果 `filed` 有非 `string` 类型的值，则启动时会报类型转换异常的错。

使用如下所示：

```java
@Value(value = "${example.name}")
private String name;

@Value(value = "${example.version}")
private Integer version;

@Value(value = "${example.isfinish}")
private boolean isFinish;

@Value(value = "#{20-2}")
private Integer age;
```

如果属性过于多，建议进行封装，不要在需要用到值的地方声明太多，看着不清爽。

### 二、使用 Environment

spring 中提供的基础方法，这里按下不表；示例中提供了包装类的实现。

### 三、指定 location 读取

经常要写一些工具类，而这些工具类依赖于配置文件，则可以使用 spring 提供的工具类进行读取。

```java
PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
Resource resource = resolver.getResource(location);
Properties pro = PropertiesLoaderUtils.loadProperties(resource);
PropertiesWrap proWrap = new PropertiesWrap(pro);
```

示例： [https://github.com/pleuvoir/reference-samples/tree/master/spring-annotation-based-example/src/main/java/io/github/pleuvoir/chapter7](https://github.com/pleuvoir/reference-samples/tree/master/spring-annotation-based-example/src/main/java/io/github/pleuvoir/chapter7)