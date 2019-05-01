
---
title: 如何实现一个LRUCache
date: 2018-06-19 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - java
  - 缓存
categories:
  - 技术
---

如何实现一个LRUCache

## 基于链表LRUCache实现

思路：

维护一个固定大小的单向链表，保证热数据永远在最后
 
1.新增的元素增加到链表的末尾，如果空间不足则删除第一个元素节点后再增加

2.修改的元素移动到末尾，并删除原节点
 
3.查找时如果之前已经有这个元素，则移动到末尾

具体的实现可参考笔者以前写的[单向链表实现](https://github.com/pleuvoir/Data-Structure-and-Algorithms/blob/master/source/data-structure-and-algorithms/src/main/java/io/github/pleuvoir/datasructure/linkedlist/Lru.java)

## 基于LinkedHashMap的实现

其实 `LinkedHashMap` 本身已经为我们的实现铺好了路。


```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

注释：

```java
 * Returns <tt>true</tt> if this map should remove its eldest entry.
 * This method is invoked by <tt>put</tt> and <tt>putAll</tt> after
 * inserting a new entry into the map.  It provides the implementor
 * with the opportunity to remove the eldest entry each time a new one
 * is added.  This is useful if the map represents a cache: it allows
 * the map to reduce memory consumption by deleting stale entries.
 *
 * <p>Sample use: this override will allow the map to grow up to 100
 * entries and then delete the eldest entry each time a new entry is
 * added, maintaining a steady state of 100 entries.
 * <pre>
 *     private static final int MAX_ENTRIES = 100;
 *
 *     protected boolean removeEldestEntry(Map.Entry eldest) {
 *        return size() &gt; MAX_ENTRIES;
 *     }
 * </pre>
 ``` 

该方法的意思是当返回true时，将从容器中移除最老的元素。该方法会在调用`put`和`putAll`时触发。

其中还有一个关键点即访问顺序：

```java
/**
 * The iteration ordering method for this linked hash map: <tt>true</tt>
 * for access-order, <tt>false</tt> for insertion-order.
 *
 */
final boolean accessOrder;
```

如果`accessOrder`为true的话(这个值默认是false)，每次访问会把元素放在链表后面，举个例子：

```java
@Test 
public void fun2() throws Exception { 
	LinkedHashMap<String, String> accessOrderTrue = new LinkedHashMap<>(16, 0.75f, true); 
	accessOrderTrue.put("1","1"); 
	accessOrderTrue.put("2","2");
	accessOrderTrue.put("3","3"); 
	accessOrderTrue.put("4","4"); 
	System.out.println("acessOrderTure"+accessOrderTrue); 
	accessOrderTrue.get("2"); 
	accessOrderTrue.get("3"); 
	System.out.println("获取了数据"+accessOrderTrue); 
} 
//控制台输出 
acessOrderTure{1=1, 2=2, 3=3, 4=4} 
获取了数据{1=1, 4=4, 2=2, 3=3}
```

基于如上的两种特性，我们就可以快速的实现自己的LRUCache。

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {

    private static final long serialVersionUID = -5167631809472116969L;

    private static final float DEFAULT_LOAD_FACTOR = 0.75f;

    private static final int DEFAULT_MAX_CAPACITY = 1000;	// 容器最大缓存量
    private final Lock lock = new ReentrantLock();
    private volatile int maxCapacity;

    public LRUCache() {
        this(DEFAULT_MAX_CAPACITY);
    }

    public LRUCache(int maxCapacity) {
        super(16, DEFAULT_LOAD_FACTOR, true);
        this.maxCapacity = maxCapacity;
    }

    @Override
    protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {
        return size() > maxCapacity;
    }

    @Override
    public boolean containsKey(Object key) {
        lock.lock();
        try {
            return super.containsKey(key);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public V get(Object key) {
        lock.lock();
        try {
            return super.get(key);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public V put(K key, V value) {
        lock.lock();
        try {
            return super.put(key, value);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public V remove(Object key) {
        lock.lock();
        try {
            return super.remove(key);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public int size() {
        lock.lock();
        try {
            return super.size();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void clear() {
        lock.lock();
        try {
            super.clear();
        } finally {
            lock.unlock();
        }
    }

    public int getMaxCapacity() {
        return maxCapacity;
    }

    public void setMaxCapacity(int maxCapacity) {
        this.maxCapacity = maxCapacity;
    }

}

```
