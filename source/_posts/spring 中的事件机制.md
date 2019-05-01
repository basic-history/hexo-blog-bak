
---
title: spring 中的事件机制
date: 2018-09-08 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - spring
  - 事件
categories:
  - 技术
---


### 使用

#### spring 默认提供的系统事件

![](https://i.imgur.com/zlucVL0.png)

一般当容器启动后，我们需要加载某些资源或者执行操作，可以通过 `ContextRefreshedEvent` 完成。

示例:

```java
public class ApplicationStartupListener implements ApplicationListener<ContextStartedEvent> {

	@Override
	public void onApplicationEvent(ContextStartedEvent event) {
		// 可以拿到容器
		ApplicationContext applicationContext = event.getApplicationContext();
		// 在这里进行业务处理
	}

}
```

启动时预先注册进容器:

```java
AnnotationConfigApplicationContext app = new AnnotationConfigApplicationContext();
app.register(Config8.class);
app.addApplicationListener(new ApplicationStartupListener()); //注意这里
app.refresh();
app.start();
app.close();
```

在 `non-web` 的 `spring` 中注册可以采用这种方式，方便让开发者清楚的知道现在有哪些事件监听器是起作用的。实际上，只需要将事件监听器注册进 `spring` 容器即可生效。于此同时可以发现 `EventObject` 有非常多的事件实现，具体使用时可以查看有没有符合心意的。

#### 自定义实现

##### 实现接口

如上，我们看到名为 `PayloadApplicationEvent<T>` 的事件接口，可以利用此接口来实现自己的自定义事件。

```java
public class MessageListener implements ApplicationListener<PayloadApplicationEvent<Message>> {

	@Override
	public void onApplicationEvent(PayloadApplicationEvent<Message> event) {
		System.out.println("接收到事件：" + event.getPayload().getMessage());
	}

}
```

发布事件:

```java
AnnotationConfigApplicationContext app = new AnnotationConfigApplicationContext();
app.register(Config8.class);
app.addApplicationListener(new MessageListener());
app.refresh();
app.start();

app.publishEvent(new Message("消息"));
app.close();
```

这里也是同样的道理，只需要注册进容器即可。

##### 使用 @EventListener 注解

上面的用法是实现了接口，`spring` 也提供了注解的支持。

```java
@Service
public class EmailService {

	@EventListener
	public void send(String address) {
		System.out.println("发送邮件 -> " + address);
	}
}

```

```java
AnnotationConfigApplicationContext app = new AnnotationConfigApplicationContext();
app.register(Config8.class);
app.addApplicationListener(new ApplicationStartupListener());
app.refresh();
app.start();
app.publishEvent("pleuvior@foxmail.com");
app.close();
```

### 源码分析

![](https://i.imgur.com/70POWGo.png)

1. 初始化事件分发器

![](https://i.imgur.com/wE3jGAj.png)

新建一个简单的事件广播器 `SimpleApplicationEventMulticaster` ，注意此处的 `taskExecutor` 为 `null`

这里我们当然也可以自己制造一个。

```java
@Bean(name = AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME)
public ApplicationEventMulticaster initApplicationEventMulticaster(BeanFactory beanFactory) {
	
	SimpleApplicationEventMulticaster applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);

	ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("事件处理线程-%d").build();
	ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors() * 2,
			200, 60L, TimeUnit.SECONDS, new ArrayBlockingQueue<>(128), threadFactory);
	
	//threadPoolExecutor.prestartAllCoreThreads();
	applicationEventMulticaster.setTaskExecutor(threadPoolExecutor);
	
	applicationEventMulticaster.setErrorHandler(new ErrorHandler() {
		
		@Override
		public void handleError(Throwable t) {
				System.out.println("出错了");
				t.printStackTrace();
		}
	});
	return applicationEventMulticaster;
}
```

2. 注册监听器

先将静态的监听器注册进来，即实现了 `ApplicationListener` 接口的。

3. 容器启动

![](https://i.imgur.com/lbuf4Wz.png)

可以看到 `start` 和 `stop` 时分别发布了两个事件。点进去看到:

![](https://i.imgur.com/h96JtFh.png)

当我们发布的内容没有实现 `ApplicationEvent` 接口，则创建一个 `PayloadApplicationEvent` 类型的事件。

![](https://i.imgur.com/T7vG7MU.png)

![](https://i.imgur.com/gYD6Wk3.png)


如下标红的两处，即是找到了对应的监听器。其中第一处的 `defaultRetrieve.applicationListeners`的值是下图执行时添加的。可以注意到，这里的事件类型匹配是根据参数类型，所以监听器一旦出现多个形参是基本类型的方法，会发现这些事件都会被广播一次。

下图是给每个被 @EventListener 标记的方法创建一个新的事件监听器，并添加到广播器中:

![](https://i.imgur.com/zgjMDsU.png)


![](https://i.imgur.com/JbOqxZr.png)

这里 `supportEvent` 方法点进去看，会发现有判断当前监听器的类型，当我们的监听器是使用注解创建时类型为 `ApplicationListenerMethodAdapter`，实现接口的则为 `GenericApplicationListenerAdapter`。 这两种判断有些区别，如果是 `GenericApplicationListenerAdapter` 会去检查当前发布的事件对象是否为监听器的实现，它里面不会去判断 `payload` 的继承关系。而 `ApplicationListenerMethodAdapter` 则先判断是否为实现（基本上不会），它会再次判断 `payload` 的继承关系。这样就会出现子类继承父类，发布父类事件，子类父类事件都执行的问题。


发布事件的代码

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
	ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
	for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
		Executor executor = getTaskExecutor();
		if (executor != null) {		// 所以只要我们设置了线程池则可以以异步的形式执行
			executor.execute(() -> invokeListener(listener, event));
		}
		else {
			invokeListener(listener, event);
		}
	}
}

// 同样可以使用错误处理器
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}
```
### 总结

至此，整个流程已经分析完毕。使用默认的事件机制，可以实现编译期的解耦，但是不能实现运行时解耦。所以可以提供线程池，让广播时新启动线程，这样则达到了单应用中合理的解耦。如果项目中某行代码不想在声明式事务中执行，则可以使用此种方式。而它也比编程时事务的代码复杂度以及合理性更胜一筹。建议使用实现接口的形式，这样不同的业务代码是隔离的。并且不会出现父子通知的问题。


代码: [spring 事件](https://github.com/pleuvoir/reference-samples/blob/master/spring-annotation-based-example/src/main/java/io/github/pleuvoir/chapter08/Config8.java "spring 事件")