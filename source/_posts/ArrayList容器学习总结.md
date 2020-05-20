---
title: ArrayList容器学习总结
date: 2020-05-20 14:48:26
tags:
    - Java
categories:
    - Java基础
cover: https://i.loli.net/2020/05/20/QwEJ3kxRnIahBz5.png
---

### 文章结构

![ArrayList容器](https://i.loli.net/2020/05/20/QwEJ3kxRnIahBz5.png)

ArrayList内部的核心数据结构为Object数组，通过`add(T e`), `get(int index)`, `set(T e)`, `remove(int index)`等增删改查方法来实现对Object数组的操作。自动扩容机制也能保证ArrayList容器在使用中能够自动的适应数据容量，方便应用程序编写者的使用

### 类结构

#### 核心数据结构

Object数组：Object[]

#### 父类

AbstractList

#### 实现的接口

- List：支持所有的List方法
- RandomAccess：可以根据数组角标进行快速的元素定位
- Cloneable：可以使用`clone()`方法进行元实例的克隆
- Serializable：能够实现对象的序列化

#### 线程的安全性

ArrayList的方法属于非线程安全性，所以要避免多个线程去修改共享ArrayList的应用场景。那么有什么可以替代的可以在多线程场景下使用的List实现类呢？目前有两个类可以代替ArrayList的多线程场景下的使用。

- Vector类
- CopyOnWriteArrayList类

Vector类是是JDK1.0版本就有用于解决有序容器的实现类，但是Vector类上用于解决线程安全方式是使用`synchronized`关键字修饰的Java同步锁。而同步的方法需要依靠JVM的协调，一个线程访问Vector代码相比于非同步的方法会话费较多的时间。在JDK1.5出来之后，在相同场景下会优先使用CopyOnWriteArrayList类。因为CopyOnWriteArrayList类对读写锁进行了优化，不锁定读操作，而对写操作时会在当前调用栈中拷贝一份原来的核心数组，在拷贝的内存上进行修改，修改结束后将原本的内存指针指向新的内存地址。由这个原理我们可以知道CopyOnWriteArrayList类在数据容量比较大的时候需要很多的时间进行数据的拷贝，所以在数据量较小的时候是属于CopyOnWriteArrayList的应用场景。

### ArrayList核心的算法

#### 扩容算法

下面代码拷贝自JDK11的ArrayList类

```java
// 扩容调用方法，可以看到直接调用扩容方法就是扩容当前的实际容量 size + 1
private Object[] grow() {
  return grow(size + 1);
}

// 目前需要的最小的容量
private Object[] grow(int minCapacity) {
  	// 获取真实需要扩容的内部Object数组大小，并将原数组进行拷贝
    return elementData = Arrays.copyOf(elementData, newCapacity(minCapacity));
}

// 确定需要扩容的容量
private int newCapacity(int minCapacity) {
  // overflow-conscious code
  int oldCapacity = elementData.length;
  
  // 这是扩容核心算法的核心，新的容量是 原容器容量 + 原容器容量的 * 0.5 => 也就是原容量的1.5倍
  // oldCapacity >> 1 右移动相当于 oldCapacity * 2^-1 => oldCapacity * 0.5
  // 这也是优化点，它这里问什么不用乘法或除法，而是用了位运算。因为jdk作为底层基础库必须考虑运行性能，而位运算的速度远远比乘除运算快
  int newCapacity = oldCapacity + (oldCapacity >> 1);
  if (newCapacity - minCapacity <= 0) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
      return Math.max(DEFAULT_CAPACITY, minCapacity);
    if (minCapacity < 0) // overflow
      throw new OutOfMemoryError();
    return minCapacity;
  }
  return (newCapacity - MAX_ARRAY_SIZE <= 0)
    ? newCapacity
    : hugeCapacity(minCapacity);
}

// 最大容量
private static int hugeCapacity(int minCapacity) {
  if (minCapacity < 0) // overflow
    throw new OutOfMemoryError();
  return (minCapacity > MAX_ARRAY_SIZE)
    ? Integer.MAX_VALUE
    : MAX_ARRAY_SIZE;
}

```

#### 扩容逻辑

我们可以看到上面的核心算法，每次调用grow方法都会进行数组的复制，如果每次add元素，都调用grow方法话肯定是不现实的这将会浪费大量的运算资源。

那类内部是如何扩容的呢？

```java
public boolean add(E e) {
  modCount++;
  add(e, elementData, size);
  return true;
}

private void add(E e, Object[] elementData, int s) {
  // 我们可以看到当时间容量等于内部Object数组的长度时才会自信扩容操作
  if (s == elementData.length)
    elementData = grow();
  elementData[s] = e;
  size = s + 1;
}
```

从上面的例子中也可以看出JDK内部为了提高运行的效率，运用了批量申请内存的思想。这与池化思想根本目的也是相同的。就是减少运算资源的消耗。这种优秀的编程思想在JDK的内部也处处可见。

### 使用的经典案例

摘自《JavaGuide》ArrayList源码章节[https://snailclimb.gitee.io/javaguide/#/docs/java/collection/ArrayList](https://snailclimb.gitee.io/javaguide/#/docs/java/collection/ArrayList)

```java
package list;
import java.util.ArrayList;
import java.util.Iterator;

public class ArrayListDemo {

    public static void main(String[] srgs){
         ArrayList<Integer> arrayList = new ArrayList<Integer>();

         System.out.printf("Before add:arrayList.size() = %d\n",arrayList.size());

         arrayList.add(1);
         arrayList.add(3);
         arrayList.add(5);
         arrayList.add(7);
         arrayList.add(9);
         System.out.printf("After add:arrayList.size() = %d\n",arrayList.size());

         System.out.println("Printing elements of arrayList");
         // 三种遍历方式打印元素
         // 第一种：通过迭代器遍历
         System.out.print("通过迭代器遍历:");
         Iterator<Integer> it = arrayList.iterator();
         while(it.hasNext()){
             System.out.print(it.next() + " ");
         }
         System.out.println();

         // 第二种：通过索引值遍历
         System.out.print("通过索引值遍历:");
         for(int i = 0; i < arrayList.size(); i++){
             System.out.print(arrayList.get(i) + " ");
         }
         System.out.println();

         // 第三种：for循环遍历
         System.out.print("for循环遍历:");
         for(Integer number : arrayList){
             System.out.print(number + " ");
         }

         // toArray用法
         // 第一种方式(最常用)
         Integer[] integer = arrayList.toArray(new Integer[0]);

         // 第二种方式(容易理解)
         Integer[] integer1 = new Integer[arrayList.size()];
         arrayList.toArray(integer1);

         // 抛出异常，java不支持向下转型
         //Integer[] integer2 = new Integer[arrayList.size()];
         //integer2 = arrayList.toArray();
         System.out.println();

         // 在指定位置添加元素
         arrayList.add(2,2);
         // 删除指定位置上的元素
         arrayList.remove(2);    
         // 删除指定元素
         arrayList.remove((Object)3);
         // 判断arrayList是否包含5
         System.out.println("ArrayList contains 5 is: " + arrayList.contains(5));

         // 清空ArrayList
         arrayList.clear();
         // 判断ArrayList是否为空
         System.out.println("ArrayList is empty: " + arrayList.isEmpty());
    }
}

```



### 使用ArrayList的注意事项

- 非线程安全，切勿在夸多线程环境下修改ArrayList实例。请使用CopyOnWriteArrayList类来代替



### ArrayList的优化手段

- **实例化对象的时候指定数组的容量**

  在实例化ArrayList对象的时候，构造器参数可以指定初始化的容器容量。目的是一次性申请足够的容量，避免自动扩容的时候进行无意义的数组拷贝，减少运算资源的浪费

- **trimToSize()，减小内存占用**

  由于扩容的机制扩容都是原容量的1.5倍，在大容量实例的ArrayList实例中内存占用会越来越大，但业务上可以预知不再增加新的元素的时候，可以使用`trimToSize()`方法使ArrayList内存占用最小化

### 参考资料

- [JavaGuide](https://snailclimb.gitee.io/javaguide/#/docs/java/collection/ArrayList)



