
---
title: 并发编程-无锁CAS之原子变量
date: 2019-04-08 12:00:00
author: pleuvoir
tags:
  - java
  - 多线程
categories:
  - 技术
---


### 简述

无锁CAS（Compare and swap，比较和交换）是一种乐观的并发控制策略，它假设对资源的访问是没有冲突的，遇到冲突进行重试操作直到没有冲突为止。这种设计思路和数据库的乐观锁很相像。在硬件层面，大部分的处理器都支持原子化的CAS指令。也就是说比较和交换这个操作是有处理器来保证是原子操作的（在最坏的情况下，如果处理器不支持，JVM将使用自旋锁）。

简单来说，CAS需要你额外给出一个期望值，也就是你认为这个变量现在是什么样子的。如果不是你想象的那样，则说明它已经被别人修改过了。你就重新读取，再次尝试修改就好了。

优势：无锁更优的性能，没有死锁风险。

> 虽然Java语言锁定语法比较简洁，但JVM和操作在管理锁时需要完成的工作却并不简单。在实现锁定是需要遍历JVM一条非常复杂的代码路径，并可能导致操作系统的锁定、线程挂起和上下文切换等操作。在最好的情况下，在锁定时至少需要一次CAS，因此
> 虽然在使用锁时没有用到CAS，但实际上也无法节约任何执行开销。另一方面，在程序执行内部执行CAS时不需要执行JVM代码、系统调用或者线程调度操作。在应用级上看起来越长的代码路径，如果加上JVM和操作系统的代码调用，那么事实上却变得更短。
> CAS主要的缺点是，它将使调用者处理竞争问题（重试、回退、放弃），而在锁中能自动处理竞争问题（线程在获得锁之前将一直阻塞）。


缺点：它将使调用者处理竞争问题（重试、回退、放弃），而在锁中能自动处理竞争问题（线程在获得锁之前将一直阻塞）。

### JDK中的原子操作类

在JDK8中`java.util.concurrent.atomic`中展示了12个以`Atomic`开头的原子变量类，这些类比锁的粒度更细，量级更轻。原子变量相当于一种泛化的volatile，它支持原子的和有条件的 读改写 操作。

```
AtomicBoolean
AtomicInteger
AtomicIntegerArray
AtomicIntegerFieldUpdater
AtomicLong
AtomicLongArray
AtomicLongFieldUpdater
AtomicMarkableReference
AtomicReference
AtomicReferenceArray
AtomicReferenceFieldUpdater
AtomicStampedReference
DoubleAccumulator
DoubleAdder
LongAccumulator
LongAdder
Striped64
```

这12个原子变量类可分为4组：

1. 标量类
2. 更新器类
3. 数组类
4. 复合变量类

最常用的原子变量就是标量类：如`AtomicInteger`、`AtomicBoolean`、`AtomicLong`、`AtomicReference`。


这里举几个常用的类为例子，其他原子类操作也都是类似的。

#### AtomicInteger

```java
static int threadCount = 20;
static AtomicInteger count = new AtomicInteger();
static CountDownLatch lacth = new CountDownLatch(threadCount);

public static void main(String[] args) throws InterruptedException {

	for (int i = 0; i < threadCount; i++) {
		new Thread(() -> {
			for (int j = 0; j < 10000; j++) {
				count.incrementAndGet();
			}
			lacth.countDown();
		}).start();
	}

	lacth.await();
	System.out.println(count.get());
}
```

console output:

```
200000
```

看一下具体的实现：

```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;   //保存着value字段在当前对象中的偏移量（其实就是一个字段到对象头部的偏移量，通过这个偏移量可以快速定位字段）

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value; 

```

`Unsafe`类封装了指针操作，JDK中不能直接使用这个类。其中`openJDK`中`Unsafe`的实现可以参考[Unsafe](https://github.com/library-of-reading/OpenJDK/blob/master/jdk/src/share/classes/sun/misc/Unsafe.java)


接下来我们来模拟一下实现：

```java
/**
 * 模拟CAS操作
 * 
 */
public class SimulatedCAS {

	protected int value;

	public SimulatedCAS(int initialValue) { // Unsafe中通过offset定位字段，并且去内存中修改这个值，我们这里为方便起见直接使用初始化值
		this.value = initialValue;
	}

	public synchronized int get() {
		return value;
	}

	// 当期望值==旧值时成功
	public synchronized boolean compareAndSet(int expectedValue, int newValue) {
		return (expectedValue == compareAndSwap(expectedValue, newValue));
	}

	public synchronized int compareAndSwap(int expectedValue, int newValue) {
		int oldValue = value;
		if (oldValue == expectedValue) {
			value = newValue;
		}
		System.out.println("oldValue=" + oldValue + ", expectedValue=" + expectedValue + ", newValue=" + newValue);
		return oldValue;
	}
}

```

```java
/**
 * 模拟AtomicInteger
 */
public class SimulatedAtomicInteger extends SimulatedCAS {

	public SimulatedAtomicInteger(int initialValue) {
		super(initialValue);
	}

   // 自旋
	public int incrementAndGet() {
		for (;;) {
			int current = value;
			int next = current + 1;
			System.out.println("current=" + current + ", next=" + next);
			if (compareAndSet(current, next)) {
				return next;
			}
		}
	}
}
```

使用相同的测试代码，发现计数器能正常使用。


#### AtomicReference

`AtomicReference`和`AtomicInteger`非常相似，不过是对对象的引用进行封装。直接看代码：

```java

static AtomicReference<User> ar = new AtomicReference<>();

User oldUser = new User("pleuvoir", 18); //要修改的对象实例
ar.set(oldUser);

System.out.println("oldUser=" + oldUser + ", ar=" + ar.get());

oldUser.setAge(14); //这一步修改了对象属性，会发现原子引用中get的也变了

System.out.println("oldUser=" + oldUser + ", ar=" + ar.get());

User newUser = new User("duke", 27); 
boolean flag = ar.compareAndSet(oldUser, newUser); //交换成功后只有newUser的修改才会改变原子引用

System.out.println("更新成功？" + flag + ", oldUser=" + oldUser + ", ar=" + ar.get());

oldUser.setName("pleuvoir~");

System.out.println("oldUser=" + oldUser + ", newUser=" + newUser + ", ar=" + ar.get());

newUser.setName("duke~");
System.out.println("oldUser=" + oldUser + ", newUser=" + newUser + ", ar=" + ar.get());
```

console output：

```
oldUser=User [name=pleuvoir, age=18], ar=User [name=pleuvoir, age=18]
oldUser=User [name=pleuvoir, age=14], ar=User [name=pleuvoir, age=14]
更新成功？true, oldUser=User [name=pleuvoir, age=14], ar=User [name=duke, age=27]
oldUser=User [name=pleuvoir~, age=14], newUser=User [name=duke, age=27], ar=User [name=duke, age=27]
oldUser=User [name=pleuvoir~, age=14], newUser=User [name=duke~, age=27], ar=User [name=duke~, age=27]
```

可以看出`AtomicReference`中保存是对象本身的引用，当对象本身发生变化后`get()`所得也会变化。使用`compareAndSet(expected, update)`交换后，原来的`expected`对象将失效，和`AtomicReference`脱离关系，之后对`expected`对象的操作将不再影响`AtomicReference`所得。

### ABA问题

假设`expected = A , update = C`，那么当我们执行CAS时，如果有另外一几个线程将A改为了B，紧接着又改回了A，那么对于此次CAS操作而言也是成功的。对于某些场景而言，这种异常出现是无关紧要的，因为我们只关心最终结果。如果不仅需要关注结果而且还想关注过程，JDK为我们提供了2个类来解决ABA问题。它们分别是`AtomicStampedReference`和`AtomicMarkableReference`。个人推荐使用`AtomicStampedReference`，类似于数据库乐观锁。


#### AtomicStampedReference

```java
static AtomicStampedReference<String> asr = new AtomicStampedReference<>("A", 0);

public static void main(String[] args) throws InterruptedException {
	int oldStamp = asr.getStamp();
	String oldReference = asr.getReference();
	System.out.println("版本号=" + oldReference + "，当前变量值=" + oldStamp);

	Thread rightThread = new Thread(new Runnable() {
		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + "当前变量值=" + oldReference + "当前版本戳=" + oldStamp
					+ "更新成功？" + asr.compareAndSet(oldReference, "B", oldStamp, oldStamp + 1));
		}
	});

	rightThread.start();
	rightThread.join(); // 等正确的执行完

	Thread errorThread = new Thread(new Runnable() {
		@Override
		public void run() {
			
			int stamp = asr.getStamp();
			String reference = asr.getReference();
			
			System.out.println(Thread.currentThread().getName() + "当前变量值=" + reference + "当前版本戳="
					+ stamp + "更新成功？" + asr.compareAndSet(oldReference, "B", oldStamp, oldStamp + 1));

			
			// 这是正确的使用方式，上面的只是为了模拟失败才使用了一开始定义的旧的oldStamp
			System.out.println(Thread.currentThread().getName() + "当前变量值=" + reference + "当前版本戳="
					+ stamp + "更新成功？"
					+ asr.compareAndSet(reference, "B", stamp, stamp + 1));
		}
	});

	errorThread.start();
	errorThread.join();
}
```

console output:

```
版本号=A，当前变量值=0
Thread-0当前变量值=A当前版本戳=0更新成功？true
Thread-1当前变量值=B当前版本戳=1更新成功？false
Thread-1当前变量值=B当前版本戳=1更新成功？false
```

上面的例子演示了`AtomicStampedReference`在版本号不正确时可以正常工作。实际在并发程序中，**更新时记得从原子引用中拿最新的版本戳和数据即可**。

#### AtomicMarkableReference

```java
static AtomicMarkableReference<String> amr = new AtomicMarkableReference<String>("A", false);

public static void main(String[] args) throws InterruptedException {

	String oldReference = amr.getReference();

	Thread rightThread = new Thread(new Runnable() {

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + "当前变量值=" + oldReference + "更新成功？"
					+ amr.compareAndSet(oldReference, "B", false, true));
		}
	});
	
	Thread errorThread = new Thread(new Runnable() {

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + "当前变量值=" + oldReference + "更新成功？"
					+ amr.compareAndSet(oldReference, "B", false, true));
		}
	});
	

	rightThread.start();
	rightThread.join();
	
	errorThread.start();
	errorThread.join();
}
```

这是`AtomicStampedReference`的用法，个人觉得很鸡肋，不如直接使用带版本戳的`AtomicStampedReference`，其中` V get(boolean[] markHolder)`也不是很好用。

### 性能比较：锁与原子变量

如果基于锁和原子变量来实现一个计数器，那么哪个性能更优？

测试表明：当高度竞争的情况下，锁的性能>原子变量；在更真实的竞争情况下，原子变量>锁的性能。如果追求更高的性能，可以尝试使用`ThreadLocal`。如果以后有机会的话，会做专门的测试。

-end