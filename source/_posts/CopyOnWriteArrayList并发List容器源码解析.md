---
title: CopyOnWriteArrayList并发List容器源码解析
date: 2020-06-03 13:05:30
tags:
    - Java
    - 并发容器
categories:
    - Java基础
cover: https://i.loli.net/2020/06/03/DMNYK4aiehxZw2j.png
---
# CopyOnWriteArrayList并发List容器源码解析
![CopyOnWriteArrayList源码分析](https://i.loli.net/2020/06/03/DMNYK4aiehxZw2j.png)


> 备注：下面的源码拷贝自JDK11

## 类结构
![CopyOnWriteArrayList](https://i.loli.net/2020/06/03/hDuVYLAiBd74gln.png)

### 实现的接口

- Serializable：支持对象的序列化
- Cloneable：支持对象的复制
- RandomAccess：支持通过索引的随机访问
- List：支持List的所有操作



### 核心数据结构

由下面的源码实现上来看，内部还是使用和ArrayList相同的普通对象数组。而且可以看到，相较于ArrayList增加了一个对象的实例锁`lock`，使用`synchronized`关键字给增删改上锁，后面的源码样例可以更加详细的看到

```java
public class CopyOnWriteArrayList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
  /** The array, accessed only via getArray/setArray. */
  private transient volatile Object[] array;
  
  
  /**
  * The lock protecting all mutators.  (We have a mild preference
  * for builtin monitors over ReentrantLock when either will do.)
  */
  final transient Object lock = new Object();
  
  
  public CopyOnWriteArrayList() {
    setArray(new Object[0]);
  }
}
```





### 内部类

#### COWIterator

CopyOnWriteArrayList的内部迭代器类

```java
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;

    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    COWIterator(Object[] es, int initialCursor) {
       cursor = initialCursor;
       snapshot = es;
    }
  //.........
}

// CopyOnWriteArrayList的iterator方法返回一个迭代器实例就是COWIterator类的实例
public Iterator<E> iterator() {
  	return new COWIterator<E>(getArray(), 0);
}
```



#### COWSubList

CopyOnWriteArrayList内部子列表视图

用于返回内部某一个时刻的数组的一个字视图，子列表视图的部分的，增删改查都用到了CopyOnWriteArrayList的实例锁

```java
private class COWSubList implements List<E>, RandomAccess {
  	private final int offset;
  	private int size;
  	private Object[] expectedArray;

    COWSubList(Object[] es, int offset, int size) {
        // assert Thread.holdsLock(lock);
        expectedArray = es;
        this.offset = offset;
        this.size = size;
    }
    //.........
 
  	// 可以看到子列表的改操作用到实例锁
  	public E set(int index, E element) {
      synchronized (lock) {
        rangeCheck(index);
        checkForComodification();
        E x = CopyOnWriteArrayList.this.set(offset + index, element);
        expectedArray = getArray();
        return x;
      }
    }

  	// 读操作用到实例锁
    public E get(int index) {
      synchronized (lock) {
        rangeCheck(index);
        checkForComodification();
        return CopyOnWriteArrayList.this.get(offset + index);
      }
    }

  	// 读操作用到实例锁
    public int size() {
      synchronized (lock) {
        checkForComodification();
        return size;
      }
    }

    // 加操作用到实例锁
    public boolean add(E element) {
      synchronized (lock) {
        checkForComodification();
        CopyOnWriteArrayList.this.add(offset + size, element);
        expectedArray = getArray();
        size++;
      }
      return true;
    }
  
  	// 删操作用到了实例锁
    public E remove(int index) {
      synchronized (lock) {
        rangeCheck(index);
        checkForComodification();
        E result = CopyOnWriteArrayList.this.remove(offset + index);
        expectedArray = getArray();
        size--;
        return result;
      }
    }
    //.........
}
```



#### COWSubListIterator

子列表视图迭代器，可以看到实现了ListIterator接口，作用和COWIterator类似，用于迭代访问数组某一个时刻的切片

```java
private static class COWSubListIterator<E> implements ListIterator<E> {
  private final ListIterator<E> it;
  private final int offset;
  private final int size;

  COWSubListIterator(List<E> l, int index, int offset, int size) {
    this.offset = offset;
    this.size = size;
    it = l.listIterator(index + offset);
  }
}
```



## 读写线程安全的实现原理

先给出语言描述，CopyOnWriteArrayList线程安全的实现原理简单来说就是：当进行增删改操作时候，通过`synchronized`修饰操作方法中的对象锁lock。然后对原数组进行复制，复制出一份和原数组一模一样的拷贝，然后在拷贝上进行增删改操作。操作完成后用新的拷贝代替原数组。然后解锁，就实现了多线程间安全的操作同一个List实例。



对于读操作，不上锁，随便读。这就是CopyOnWriteArrayList适合读多写少的场景的原因



### 读操作源码

可以看到，对于读操作是最简单的实现。没有上任何的锁机制

```java
public E get(int index) {
   return elementAt(getArray(), index);
}

final Object[] getArray() {
  return array;
}

// 没错就是你看到的这么简单，直接通过索引读取对应位置的元素
@SuppressWarnings("unchecked")
static <E> E elementAt(Object[] a, int index) {
  return (E) a[index];
}

```



### 增加操作源码

```java
public boolean add(E e) {
  // 使用synchronized对lock对象锁上锁，所以多线程的修改操作都是互斥的。同时也避免了多线程写时复制出多个副本出来
  synchronized (lock) {
    // 1.取原数组
    Object[] es = getArray();
    int len = es.length;
    
    // 2.拷贝一份新的数组实例，长度+1
    es = Arrays.copyOf(es, len + 1);
    
    // 3.将新的元素放在数组的最后面
    es[len] = e;
    
   	// 4.用新的数组代替老数组
    setArray(es);
    return true;
  }
}

final void setArray(Object[] a) {
  array = a;
}
```



### 替换操作源码

替换操作的逻辑和增加操作是一样的，只是做了一定的优化。先判断被替换的元素和新元素是不是一样。如果一样就没必要替换了

```java
public E set(int index, E element) {
  synchronized (lock) {
    Object[] es = getArray();
    E oldValue = elementAt(es, index);

    // 如果老元素和新元素是同一个，就没必要替换了
    if (oldValue != element) {
      // 拷贝一份新的数组
      es = es.clone();
      
      // 替换对应位置的元素
      es[index] = element;
      setArray(es);
    }
    return oldValue;
  }
}
```



### 删除操作源码

替换操作相对复杂一点，涉及到元素的两次拷贝

```java
public E remove(int index) {
  synchronized (lock) {
    // 取原数组
    Object[] es = getArray();
    // 原数组长度
    int len = es.length;
    
    // 要移除位置的元素
    E oldValue = elementAt(es, index);
    
    // 删除位置元素之后的剩余数组右侧需要复制的长度
    int numMoved = len - index - 1;
    
    // 新的空数组
    Object[] newElements;
    
    // 移除的是原始数组的最后一个元素
    if (numMoved == 0)
      // 直接复制到倒数第二个元素
      newElements = Arrays.copyOf(es, len - 1);
    else {
      // 长度减1的新空数组
      newElements = new Object[len - 1];
      
      // 将原始数组从0开始复制index个元素到新数组从0位置开始的数组中
      System.arraycopy(es, 0, newElements, 0, index);
      
      // 将原始数组跳过要移除的元素复制numMoved个元素到新数组从index开始的数组中
      System.arraycopy(es, index + 1, newElements, index, numMoved);
    }
    setArray(newElements);
    return oldValue;
  }
}
```



## CopyOnWriteArrayList的特点

- 同时读读不加锁：读高性能
- 同时写写互斥：写同步
- 同时读写不阻塞：读写分离，互相不干预
- 没有像ArrayList一样的扩容机制：因为所有的修改操作都是进行对象的复制，所以没有必要引入扩容机制



## CopyOnWriteArrayList 使用场景以及注意点

从CopyOnWriteArrayList的源码实现中可以看出，为了提高列表的读写性能，对读操作是不加锁的，可以任意读。换句话说，其实多线程读同一个元素，也没必要加锁。而对增删改操作使用实例的对象锁，进行操作的同步，只需要写入和写入之间需要同步，提高整体的读写性能。

但是从实现机理上也能看出，每次的写操作都会进行数组的复制操作。如果数组长度很大的话，效率会降低。

#### 适合的应用场景

- 读多写少的情况
- 列表规模不大的情况



## 参考
- [https://snailclimb.gitee.io/javaguide/#/docs/java/Multithread/并发容器总结](https://snailclimb.gitee.io/javaguide/#/docs/java/Multithread/并发容器总结?id=_32-copyonwritearraylist-是如何做到的)



