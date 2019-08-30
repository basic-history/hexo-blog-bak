
---
title: 并发编程-synchronized
date: 2019-04-11 12:00:00
author: pleuvoir
tags:
  - java
  - 多线程
categories:
  - 技术
---


今天我们来看看并发编程的内置锁：synchronized

### 用法

`synchronized`是一种使用简单的锁关键字，简单来说用法大致分为三种：

- 指定加锁对象：给给定对象加锁，进入同步代码前要获得给定对象的锁。
- 直接作用于方法实例：相当于对当前实例加锁，进入同步代码前要获得当前实例的锁。
- 直接作用于静态方法：相当于对当前类进行加锁，进入同步代码前要获得当前类的锁。


#### 指定加锁对象

先看一个不安全的示例：

```java
static int j = 0;

public static void main(String[] args) throws InterruptedException {
	Task first = new Task();
	Task second = new Task();
	first.start();
	second.start();
	second.join();
	System.out.println(j);
}

public static class Task extends Thread {
	@Override
	public void run() {
		for (int i = 0; i < 1000; i++) {
			j++;
		}
	}
}
```

这段代码按理来说，第二个线程执行完毕后总之应该为2000，但实际上因为并发修改的原因，值总是<=2000。这个时候使用最简单的`synchronized`来对一个共享对象加锁，即可轻松的解决，如下示。

```java
static Object monitor = new Object();

public void run() {
	synchronized (monitor) { //线程必须获得monitor对象的锁才可以继续执行
		for (int i = 0; i < 1000; i++) {
			j++;
		}
	}
}
```

#### 直接作用于实例方法

还是先看一个例子：

```java
public static void main(String[] args) throws InterruptedException {
		Student student = new Student();
		Task first = new Task(student);
		Task second = new Task(student);
		first.start();
		second.start();
		first.join();
		second.join();
	}

	public static class Student {
		private void say() {
			System.out.println("你好啊");
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static class Task extends Thread {
		private Student student;
		public Task(Student student) {
			this.student = student;
		}
		@Override
		public void run() {
			student.say();
		}
	}
```

上面的程序执行效果是：同时输出你好啊，1秒后程序结束。这代表两个线程是并发执行的，怎么改成串行的呢？很简单，只需要在`say()`修饰`synchronized`即可。

```java
private synchronized void say() {
	System.out.println("你好啊");
	try {
		TimeUnit.SECONDS.sleep(1);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
}
```

这样程序执行效果为你好啊，一秒过后再次输出你好啊，再过一秒程序结束。这里需要注意的是，因为调用`say()`的对象是同一个`student`，所以锁才会生效。

正好我们复习一下第一种加锁方式（指定加锁对象）也是可以的：

```java
private void say() {
	synchronized (this) {
		System.out.println("你好啊");
		try {
			TimeUnit.SECONDS.sleep(1);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
这种写法的优势是可以有效的缩小临界区的范围，以后我们讲到锁的优化时会进行学习。


#### 直接作用于静态方法

还是以上面的例子为参考，可以看出上面加锁的都是同一个对象，如果换成2个不同的`student`对象呢？答案是不可以。

```java
Student student = new Student();
Student student2 = new Student();
Task first = new Task(student);   //synchronized锁的是调用方法的实例即student
Task second = new Task(student2); //synchronized锁的是调用方法的实例即student2
first.start();
second.start();
first.join();
second.join();

private synchronized void say() {
	System.out.println("你好啊");
	try {
		TimeUnit.SECONDS.sleep(1);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
}
```

如何改动可以生效呢？将`say()`改为静态方法，这样锁的就是类了。

```java
private static synchronized void say() {
	System.out.println("你好啊");
	try {
		TimeUnit.SECONDS.sleep(1);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
}
```


### 总结

在Java中，最基本的互斥同步手段就是`synchronized`关键字，`synchronized`关键字编译后，会在同步块的前后分别形成`monitorenter`和`monitorexit`这两个字节码指令，这两个字节码都需要一个`Reference`类型的参数来指明要锁定和解锁的对象。如果`synchronized`指定了加锁对象，那就是这个对象；如果没有则判断是作用于实例方法还是类方法(被static修饰)，去取对应的对象或者Class对象来作为`Reference`。

按照虚拟机规范，当执行`monitorenter`，先去获取对象的锁，如果这个对象没被锁定或者当前线程已经拥有这个对象的锁，那么锁计数+1；同样，`monitorexit`就是-1。当为0时锁就释放了。如果遇到了获取锁失败的情况，那么则等待，直到锁被别的线程释放。