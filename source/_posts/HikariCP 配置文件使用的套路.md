
---
title: HikariCP 配置文件使用的套路
date: 2018-12-08 19:19:00
author: pleuvoir
img: /images/code.jpg
tags:
  - 开源
  - 套路
categories:
  - 技术
---

本文是对一个配置类的简要分析，其中隐藏了不易被关注的细节，并对作者使用的技巧进行学习。

`HikariCP` 中有一个面向对象的类 `HikariConfig` ，为什么说它是面向对象的，原因是因为常规对 `properties` 文件读取后，会直接加载到需要的位置，而此类会通过反射进行属性的转化。

## 一些方法

### loadProperties

其中有一个方法，是用来加载文件：

```java
private void loadProperties(String propertyFileName) {
	final File propFile = new File(propertyFileName);
	InputStream resourceAsStream = this.getClass().getResourceAsStream(propertyFileName);
	try (final InputStream is = propFile.isFile() ? new FileInputStream(propFile) : this.getClass().getResourceAsStream(propertyFileName)) {
		if (is != null) {
			Properties props = new Properties();
			props.load(is);
			// 这段代码使用了反射完成了 Properties 到本对象的转换
		    PropertyElf.setTargetFromProperties(this, props);
		} else {
			throw new IllegalArgumentException("Cannot find property file: " + propertyFileName);
		}
	} catch (IOException io) {
		throw new RuntimeException("Failed to read property file", io);
	}
}
```

其中该方法支持两种形式的入参，第一种是使用物理绝对路径，如 `F:\\git\\luxury-lunch\\src\\test\\resources\\HikariConfigTest.properties`；当然，我们也可以使用相对路径，如 `/HikariConfigTest.properties`，注意一定要有 `/`，`/`表示从 `classpath` 下寻找该文件。具体使用可以参考 [Java 中 getResourceAsStream 的用法](http://www.cnblogs.com/macwhirr/p/8116583.html)

显然，我们一般都会使用相对路径。同时，这段代码使用了 `try-resource` 风格，可以优雅的关闭文件流。所以，这段读取 `properties` 文件的代码现已加入豪华午餐。

### copyStateTo

复制状态，该方法通过反射将本类 `HikariConfig` 自身的属性值赋值给一个新创建的 `HikariConfig`

```java
public void copyStateTo(HikariConfig other) {
	for (Field field : HikariConfig.class.getDeclaredFields()) {
		if (!Modifier.isFinal(field.getModifiers())) {
			field.setAccessible(true);
			try {
				field.set(other, field.get(this));
			} catch (Exception e) {
				throw new RuntimeException("Failed to copy HikariConfig state: " + e.getMessage(), e);
			}
		}
	}
}
```

该示例经过测试发现，当属性为基本类型时，反射复制没有问题，即使修改原复制对象的属性，新对象也不会受到影响。注意：一旦原复制对象中有非基本类型，那么复制以后新对象的这些非基本类型也会被修改。

浅拷贝（Shallow Copy）：

1. 对于数据类型是基本数据类型的成员变量，浅拷贝会直接进行值传递，也就是将该属性值复制一份给新的对象。因为是两份不同的数据，所以对其中一个对象的该成员变量值进行修改，不会影响另一个对象拷贝得到的数据。

2. 对于数据类型是引用数据类型的成员变量，比如说成员变量是某个数组、某个类的对象等，那么浅拷贝会进行引用传递，也就是只是将该成员变量的引用值（内存地址）复制一份给新的对象。因为实际上两个对象的该成员变量都指向同一个实例。在这种情况下，在一个对象中修改该成员变量会影响到另一个对象的该成员变量值。

所以这个方法和浅拷贝的用处是一样的，同样实现浅拷贝的方法还有实现 `Cloneable` 接口，直接重写 `clone()` 方法，通过调用 `clone` 方法即可完成浅拷贝。

参考：[浅拷贝与深拷贝](https://www.cnblogs.com/shakinghead/p/7651502.html)

###  getPropertyNames

获取类中所有 `field` 的集合，过滤重复。

```java
private static final Pattern GETTER_PATTERN = Pattern.compile("(get|is)[A-Z].+");

public static Set<String> getPropertyNames(final Class<?> targetClass) {
	HashSet<String> set = new HashSet<>();
	Matcher matcher = GETTER_PATTERN.matcher("");
	for (Method method : targetClass.getMethods()) {
		String name = method.getName();
		if (method.getParameterTypes().length == 0 && matcher.reset(name).matches()) {  // 获取 get 或 is 开头的方法
			name = name.replaceFirst("(get|is)", "");	//  isAutoCommit 会被改为 AutoCommit
			try { 
				if (targetClass.getMethod("set" + name, method.getReturnType()) != null) {  //这里检查名称为 setAutoCommit 且入参类型为 boolean 的方法存在不存在，显然是存在的
					name = Character.toLowerCase(name.charAt(0)) + name.substring(1);		// 认为属性名是 autoCommit，所以是不对的
					set.add(name);
				}
			} catch (Exception e) {
				// fall thru (continue)
			}
		}
	}

	return set;
}
```

这个方法其实是有问题的，当属性值为 `boolean` 类型时，需要注意如果字段名为 `isAutoCommit` 或者 `autoCommit`，`IDE` 帮我们生成的 `get` 方法都是 `isAutoCommit`， `set` 方法都是 `setAutoCommit`。
所以，直接将方法名称中 `get 或者 set` 替换空是不严谨的。

所以，同时也是很多代码规范中的要求，不要给 `boolean` 类型的变量加 `is`，原因就在此。

### setTargetFromProperties

该方法是将 `properties` 文件中各个值反射设置给类的属性。但是原代码中不支持 `double` 类型，所以做了一下修改。

```java
// 尝试使用类中变量名进行转换，如果类型为布尔且以 is 开头，请配置 properties 文件时去除 is，否则会报错
private static void setProperty(final Object target, final String propName, final Object propValue,
		final List<Method> methods) {

  String methodName = "set" + propName.substring(0, 1).toUpperCase(Locale.ENGLISH) + propName.substring(1);
  Method writeMethod = methods.stream().filter(m -> m.getName().equals(methodName) && m.getParameterCount() == 1).findFirst().orElse(null);

  // 这里布尔类型会有问题，建议在类中设置布尔类型的变量时是不要以 is 开头，如果已经使用了 is 开头，那么 properties 文件中的 key 请去掉 is
  if (writeMethod == null) {
     String booleanMethodName =  "is" + propName.substring(0, 1).toUpperCase(Locale.ENGLISH) + propName.substring(1);
     writeMethod = methods.stream().filter(m -> m.getName().equals(booleanMethodName) && m.getParameterCount() == 1).findFirst().orElse(null);
  }

  if (writeMethod == null) {
     throw new RuntimeException(String.format("Property %s does not exist on target %s", propName, target.getClass()));
  }

  try {
	  // 根据参数类型尝试
     Class<?> paramClass = writeMethod.getParameterTypes()[0];
     if (paramClass == int.class) {
        writeMethod.invoke(target, Integer.parseInt(propValue.toString()));
     }
     else if (paramClass == long.class) {
        writeMethod.invoke(target, Long.parseLong(propValue.toString()));
     }
     else if (paramClass == boolean.class || paramClass == Boolean.class) {
        writeMethod.invoke(target, Boolean.parseBoolean(propValue.toString()));
     }
     else if (paramClass == String.class) {
        writeMethod.invoke(target, propValue.toString());
     }
	 else if (paramClass == double.class) {
		writeMethod.invoke(target, Double.valueOf(propValue.toString()));
	 }
     else {
        try {
           writeMethod.invoke(target, Class.forName(propValue.toString()).newInstance());
        }
        catch (InstantiationException | ClassNotFoundException e) {
           writeMethod.invoke(target, propValue);
        }
     }
  }
  catch (Exception e) {
     throw new RuntimeException(e);
  }
}
```

可以完成文件到对象的转换！简直 nice :blush:


## 特殊的思路

一个类需要使用另一个类的配置，我们一般都是注入，然而作者的思路是，这个类直接继承另一个类。如下：

```java
public class HikariDataSource extends HikariConfig {

	// 传统做法
	private HikariConfig hikariConfig;
	
	
	public HikariDataSource(HikariConfig configuration) {
	//	configuration.copyStateTo(this); // 这行会进行属性拷贝 ，这是作者的做法，优势是当属性值很多时可以不用再去每次都去用别的类点调用
		this.hikariConfig = configuration; // 传统做法
	}


	public void dosomething1() {
		System.out.println("驱动名称有了：" + hikariConfig.getDriverClassName()); // 这里就显得比较啰嗦
	}

	public void dosomething2() {
		System.out.println("驱动名称有了：" + getDriverClassName());  // 简单
	}

}

```

其实这段代码并不复杂，主要是我们能不能想到这么做很关键。


## 豪华午餐

```java
// 获得的豪华午餐
Properties props = PropLoaderUtil.loadProperties(propertyFileName);

// 反射拷贝属性，浅拷贝小心拷贝对象中引用类型的改变
User copyStateTo = CopyStateUtil.copyStateTo(source, target, User.class);
CopyStateUtil.copyStateTo(source, target);

Cloneable 接口的使用：

@Override
protected User clone() {
	User user = null;
	try {
		user = (User) super.clone();
	} catch (CloneNotSupportedException e) {
		e.printStackTrace();
	}
	return user;
}

// 完成文件到对象的转换，不存在的属性会报错，以后新建了对象然后将属性贴到文件中即可使用了
PropLoaderUtil.setTargetFromProperties(myHikariConfig, "/HikariConfigTest.properties");
```

最佳实践
```java
public class AppConfig {

private static final AppConfig INSTANCE = new AppConfig();

private String name;
private String name2;
private String name3;
private String name4;
private String name5;
private String name6;
private String name7;
private String name8;
private String version;
private boolean complete;


public static AppConfig load(String propPath) {
	return PropLoaderUtil.setTargetFromProperties(INSTANCE, propPath, "app.");
}

public void copyStateTo(Object target) {
	CopyStateUtil.copyStateTo(this, target);
}

// 省略 get and set
```

```java
public class Bootstrap extends AppConfig {

	public Bootstrap(AppConfig appConfig) {
		appConfig.copyStateTo(this);
		// 如此，appConfig 中属性值已经可以直接 getXXX 了，否则还得 appConfig.getXXX，当属性多时有特效
	}
}
```
```
//文件内容
app.name=luxury-lunch
app.name2=luxury-lunch2
app.name3=luxury-lunch3
app.name4=luxury-lunch4
app.name5=luxury-lunch5
app.name6=luxury-lunch6
app.name7=luxury-lunch7
app.version=20181208
app.complete=false
```

```java
// 使用
Bootstrap bootstrap = new Bootstrap(AppConfig.load("/config/appconfig.properties"));
```
