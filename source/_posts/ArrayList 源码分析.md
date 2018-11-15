
---
title: ArrayList 源码分析
date: 2018-03-19 12:00:00
author: pleuvoir
img: /images/code.jpg
tags:
  - java
categories:
  - 技术
---

## ArrayList 源码分析  
  
### 1. 概览  

```java  
public class ArrayList extends AbstractList  
        implements List, RandomAccess, Cloneable, java.io.Serializable  
```
  
实现了 RandomAccess 接口，也就是说支持随机访问，因为 ArrayList 是基于数组实现的。 实际上查看 RandomAccess 的实现发现为空实现标记接口。文档提及，如果一个 List 实现了 RandomAccess 接口，则使用 for 循环遍历的方式会比 Iterator 更快。在平时的开发中，在数据量比较大的时候通过判断List的类型从而选择更优的迭代方式。
  
```java
if (arrayList instanceof RandomAccess) {
	for (;;) {
		// .. ignored
	}
} else {
	Iterator<String> iterator = arrayList.iterator();
	while (iterator.hasNext()) {
		String s = (String) iterator.next();
		// .. ignored
	}
}
```

由于该类基于数组实现，保存元素的数组使用 transient 修饰，表明数组默认不会被序列化。想想 ArrayList 的动态扩容特性，因此保存元素的数组不一定都会被使用，那么就没必要全部进行序列化。ArrayList 重写了 writeObject() 和 readObject() 来控制只序列化数组中有元素填充那部分内容。  
  
```java  
transient Object[] elementData; // non-private to simplify nested class access  
```  

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```


数组的默认大小为 10。  
  
```java  
private static final int DEFAULT_CAPACITY = 10;  
```  
  
删除元素时需要调用 System.arraycopy() 对元素进行复制，因此删除操作成本很高。  
  
```java  
public E remove(int index) {  
    rangeCheck(index);  
  
    modCount++;  
    E oldValue = elementData(index);  
  
    int numMoved = size - index - 1;  
    if (numMoved > 0)  
        System.arraycopy(elementData, index+1, elementData, index, numMoved);  
    elementData[--size] = null; // clear to let GC do its work  
  
    return oldValue;  
}  
```  
  
添加元素时使用 ensureCapacity() 方法来保证容量足够，如果不够时，需要使用 grow() 方法进行扩容，使得新容量为旧容量的 1.5 倍（oldCapacity + (oldCapacity >> 1))。扩容操作需要把原数组整个复制到新数组中，因此最好在创建 ArrayList 对象时就指定大概的容量大小，减少扩容操作的次数。  
  
```java  
private void ensureExplicitCapacity(int minCapacity) {  
    modCount++;  
  
    // overflow-conscious code  
    if (minCapacity - elementData.length > 0)  
        grow(minCapacity);  
}  
  
private void grow(int minCapacity) {  
    // overflow-conscious code  
    int oldCapacity = elementData.length;  
    int newCapacity = oldCapacity + (oldCapacity >> 1);  
    if (newCapacity - minCapacity < 0)  
        newCapacity = minCapacity;  
    if (newCapacity - MAX_ARRAY_SIZE > 0)  
        newCapacity = hugeCapacity(minCapacity);  
    // minCapacity is usually close to size, so this is a win:  
    elementData = Arrays.copyOf(elementData, newCapacity);  
}  
```  
  
### 2. Fail-Fast  
  
modCount 用来记录 ArrayList 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，修改元素的值不算。    
  
在进行序列化 ，迭代等操作时，会比较操作前后 modCount 是否改变，如果改变了会抛出ConcurrentModificationException。   

在并行的情况下，可能会抛出该异常，文档中提及并不能保证该特性有效，所以不能依赖该异常来做判断，仅仅作为检测bug来使用是合理的。  
  
如果需要一个线程安全的实现，可以这样，本质上是用synchronized来处理的。  
  
```java  
List arrayList = new ArrayList();  
List synchronizedList = Collections.synchronizedList(arrayList);  
``` 

### 3. 和 LinkedList 的区别  
  
- ArrayList 基于动态数组实现，LinkedList 基于双向循环链表实现；  
- ArrayList 支持随机访问，LinkedList 不支持；  
- LinkedList 在任意位置添加删除元素更快。