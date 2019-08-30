
---
title: 并发编程-ReentrantLock
date: 2019-04-12 12:00:00
author: pleuvoir
tags:
  - java
  - 多线程
categories:
  - 技术
---


今天我们来看看并发编程的可重入锁`ReentrantLock`，它和关键字`synchronized`非常的相像。`ReentrantLock`可以完全替代关键字`synchronized`，JDK6以后`synchronized`进行了很多的优化，所以两者在性能上差距不大。建议能使用`synchronized`情况下优先使用，因为后续JVM还会对其实现进行持续优化。

那么它比`synchronized`有哪些优点呢？


1）支持公平锁和非公平锁：

在大多数情况下，锁的申请都是非公平的。当线程1请求了锁A，线程2之后也请求了锁A，当锁可用时，是谁获得这个锁呢？这是不一定的，系统会从锁的等待队列中随机挑选一个，因此不能保证公平性。开启公平锁模式必须要维护一个有序队列，所以会带来性能的损失。但是公平模式会保证有序性，并且不会产生饥饿现象。


2）支持响应中断以及限时获取锁：

对关键字`synchronized`而言，如果一个线程正在等待一把锁，那么结果只有两种：要么等待，要么获得锁。而`ReentrantLock`为我们提供了第三种可能，那就是响应中断（等着等着不等了），这种方式对处理死锁也是有帮助的。

限时获取锁指的是在一定时间内尝试获取锁，超时在不在获取返回false。

3）更灵活的api

这个也能算是它的一个优点，我们可以方便的控制临界区代码。


### 用法

#### lock

还是直接来看例子：

```java
static ReentrantLock lock = new ReentrantLock();

static int j = 0;

public static void main(String[] args) throws InterruptedException {
	Task first = new Task();
	Task second = new Task();
	first.start();
	second.start();
	first.join();
	second.join();
	System.out.println(j);
}

public static class Task extends Thread {
	@Override
	public void run() {
		try {
			lock.lock();
			for (int i = 0; i < 1000; i++) {
				j++;
			}
		} finally {
			lock.unlock();
		}
	}
}
```

可以看到，和`synchronized`相比，`ReentrantLock`加锁和释放锁的操作都是由开发人员手动控制的，所以它有更高的灵活性。需要注意的是，锁在使用完必须释放，否则其他线程没有机会访问到临界区了。一般我们会在`finally`中进行释放。

为什么叫可重入锁呢？看下面的代码就明白了：

```java
try {
	lock.lock();
	lock.lock();
	for (int i = 0; i < 1000; i++) {
		j++;
	}
} finally {
	lock.unlock();
	lock.unlock();
}
```

当获取到锁时再次进行获取是可以得到锁的，并不会因为自己已经持有了锁导致死锁。一定要注意，获取了几次便要释放几次锁。

#### lockInterruptibly中断响应

对关键字`synchronized`而言，如果一个线程正在等待一把锁，那么结果只有两种：要么等待，要么获得锁。而`ReentrantLock`为我们提供了第三种可能，那就是响应中断（等着等着不等了），这种方式对处理死锁也是有帮助的。


先演示一个死锁的发生：

```java
static ReentrantLock lock1 = new ReentrantLock();
static ReentrantLock lock2 = new ReentrantLock();

public static void main(String[] args) throws InterruptedException {
	Task t1 = new Task(1);
	Task t2 = new Task(2);
	t1.start();
	t2.start();
	t1.join();
}

public static class Task extends Thread {
	int lock; // 控制传入的锁

	public Task(int lock) {
		this.lock = lock;
	}

	@Override
	public void run() {

		if (lock == 1) {
			try {
				// 先获取lock1 再获取lock2
				lock1.lock();
				TimeUnit.SECONDS.sleep(1);
				lock2.lock();
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock1.unlock();
				lock2.unlock();
			}
		} else {
			try {
				// 先获取lock2 再获取lock1
				lock2.lock();
				TimeUnit.SECONDS.sleep(1);
				lock1.lock();
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock2.unlock();
				lock1.unlock();
			}
		}

	}
}
```

t1获得lock1的时候，t2同时获得了lock2；它们想获得的第二个锁被对方持有，因而死锁。我们将程序稍事修改，即可解决这个死锁问题。

```java
static ReentrantLock lock1 = new ReentrantLock();
static ReentrantLock lock2 = new ReentrantLock();

public static void main(String[] args) throws InterruptedException {
	Task t1 = new Task(1);
	Task t2 = new Task(2);
	t1.start();
	t2.start();
	TimeUnit.SECONDS.sleep(2);
	t1.interrupt();
	System.out.println("t1.interrupt()");
}

public static class Task extends Thread {
	int lock; // 控制传入的锁

	public Task(int lock) {
		this.lock = lock;
		setName("lock-" + lock);
	}

	@Override
	public void run() {

		try {

			if (lock == 1) {
				// 先获取lock1 再获取lock2
				lock1.lockInterruptibly();
				System.out.println(Thread.currentThread().getName() + "已获得获取lock1");
				TimeUnit.SECONDS.sleep(1);
				System.out.println(Thread.currentThread().getName() + "尝试获取lock2");
				lock2.lockInterruptibly();
				System.out.println(Thread.currentThread().getName() + "获取lock2成功，执行完毕");
			} else {
				// 先获取lock2 再获取lock1
				lock2.lockInterruptibly();
				System.out.println(Thread.currentThread().getName() + "已获得获取lock2");
				TimeUnit.SECONDS.sleep(1);
				System.out.println(Thread.currentThread().getName() + "尝试获取lock1");
				lock1.lockInterruptibly();
				System.out.println(Thread.currentThread().getName() + "获取lock1成功，执行完毕");
			}

		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			if (lock1.isHeldByCurrentThread()) {
				lock1.unlock();
			}
			if (lock2.isHeldByCurrentThread()) {
				lock2.unlock();
			}
		}
	}
}
```

线程可以响应中断，中断的线程会让出持有的锁，从而解决死锁的问题。这里还有个细节释放锁的时候通过`lock.isHeldByCurrentThread()`来判断，字如其意，如果这个锁被当前线程持有则释放。为什么需要这样判断，直接释放不可以吗？对不起不可以，因为线程中断的时候，锁已经被释放了，这里不判断的话会多次释放触发`IllegalMonitorStateException`。

#### tryLock锁申请等待限时

```java
boolean tryLock();
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
```

这两个api看着都很直观，第一个尝试获取锁失败返回false，第二个则是在一定时间内尝试获取，超时后返回false。

同样，利用这种机制也可以减少死锁的发生。


#### 重入锁的好搭档Condition

类似于翻版的`wait-notify`，我们知道等待通知机制是配合`synchronized`先获取锁，然后才能`wait-notify`。在重入锁里也提供了这样类似实现。使用`Condition`来控制等待通知。

使用

```java
void await() throws InterruptedException;
void awaitUninterruptibly();
long awaitNanos(long nanosTimeout) throws InterruptedException;
boolean await(long time, TimeUnit unit) throws InterruptedException;
boolean awaitUntil(Date deadline) throws InterruptedException;
void signal();
void signalAll();
```

以上方法的含义如下：

* `await()`方法会使当前线程等待，并释放当前锁。当其他线程正确使用了`signal()`或者`signalAll()`会返回。当前线程被中断，也能跳出等待。这和`Object.wait()`类似。
*  `awaitUninterruptibly()`不会在等待过程中响应中断
*  `signal()`唤醒一个正在等待的线程，`signalAll()`唤醒所有。这和`Object.notify()/notifyAll()`类似。


来看一个示例：

```java
static ReentrantLock lock = new ReentrantLock();
static Condition condition = lock.newCondition();

public static void main(String[] args) throws InterruptedException {

	Task task1 = new Task();
	Task task2 = new Task();

	task2.start();
	task1.start();

	TimeUnit.SECONDS.sleep(3);
	try {
		lock.lock();
	//	condition.signal();
		condition.signalAll(); //通知所有
		System.out.println("已通知");
	} finally {
		TimeUnit.SECONDS.sleep(2);
		lock.unlock();
	}
}

public static class Task extends Thread {
	@Override
	public void run() {
		try {
			lock.lock();
			condition.await();
			System.out.println(getName() + " over..");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
}
```

console output:

```
已通知
Thread-1 over..
Thread-0 over..
```

程序运行的效果是3秒后控制台输出`已通知`，再过2秒，输出`Thread-0 over..和Thread-0 over..`。可以看出这点和我们使用等待通知是类似的（notify方法放在方法的最后一行）。通知时只有释放了锁，正在等待的线程才能恢复运行。


### 总结

今天我们学习了J.U.C中重要的可重入锁，它提供了更高的灵活性，可一定程度避免死锁，同时还有它的好搭档`Condition`。可以在很多软件中看见它的使用，下次我们将从源码层面分析其实现。