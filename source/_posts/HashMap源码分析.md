---
title: HashMap源码分析
date: 2020-05-25 22:03:17
tags:
    - Java
categories:
    - Java基础
cover: https://i.loli.net/2020/05/25/LrZQy1ASqxsongB.png
---
# HashMap源码解析

![HashMap源码解析文章结构](https://i.loli.net/2020/05/25/LrZQy1ASqxsongB.png)

## 1. 类结构

![继承实现关系图](https://i.loli.net/2020/05/25/LNKus7G16cgbo3X.png)

上图可以看到，HashMap继承了AbstractMap，实现的接口有，Map、Cloneable、Serializable。

HasMap的核心数据类型是链表或红黑树的数组，数组和List结构一样可以实现扩容。并且有实现相对应的用于通过计算key对象的hash值定位数组索引位置的hash()方法，当数组同一位置的链表长度大于触发阈值的时候，链表会转化为红黑树，提高遍历的效率。HashMap的数据结构如下图。

![HashMap核心数据结构图](https://i.loli.net/2020/05/25/YmSLJGAnVx5qUsz.png)

### 1.1.内部类

#### 1.1.1.数据节点类型

- Node

  HashMap最基本的数据结构单元，核心的数据结构就是Node数组（Node<K, V> table）

- TreeNode

  当HashMap的核心数据结构table数组在同一个位置的链表长度，大于树化触发阈值的时候，同一个位置的链表将会变成红黑树的结构。他们的都是Map.Entry<K,V>的实现类

#### 1.1.2. 访问器类

- KeySet

  KeySet是提供给外部用于迭代访问Map的键的访问器类型

- Values

  Values是提供给外部用于迭代访问Map的值的访问器类型

- EntrySet

  EntrySet是提供给外部用于迭代访问Map健值对的访问器类型

上面三个分别和下面的三个迭代器类型配合使用

#### 1.1.2. 普通迭代器类型

- KeyIterator
- ValueIterator
- EntryIterator

迭代器类的主要作用是提供给HashMap的使用者可以通过循环访问的方式迭代访问HashMap的每个存放的元素。顾名思义，KeyIterator是用于迭代访问Map类型的键值，在HashMap中KeyIterator没有直接的暴露给外部调用，而是给KeySet

类内部使用，来访问Map的键。同理，valueIterator是给Values类使用的，用于迭代访问Map的值。EntryIterator是给EntrySet类提供的内部类，用于迭代访问Map的健值对。

#### 1.1.3. 分割迭代器类型

- KeySpliterator
- ValueSpliterator
- EntrySpliterator

作用普通的的迭代器是相同的，但是分割迭代器是用于在多线程的情况下，同时遍历访问同一个HashMap对象。



### 1.2. 静态的类属性

**桶**：table数组同一个位置的一组以链表或红黑树形式存在的元素集合

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    // 序列号
    private static final long serialVersionUID = 362498820763181265L;    

	  // 默认的初始容量是16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   
  
  	// 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30; 
  
	  // 默认的填充因子，当Map的实际元素个数大于当前Map的容量的0.75倍时候Map会进行扩容
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
  
  	// 当桶(bucket)上的结点数大于这个值时会转成红黑树
    static final int TREEIFY_THRESHOLD = 8; 
  
	  // 当桶(bucket)上的结点数小于这个值时树转链表
    static final int UNTREEIFY_THRESHOLD = 6;
  
  	// 桶中结构转化为红黑树对应的table的最小大小
    static final int MIN_TREEIFY_CAPACITY = 64;
  
	  // 存储元素的数组，总是2的幂次倍
    transient Node<k,v>[] table; 
  
  	// 存放具体元素的集
    transient Set<map.entry<k,v>> entrySet;
  
	  // 存放元素的个数，注意这个不等于数组的长度。
    transient int size;
  
  	// 每次扩容和更改map结构的计数器
    transient int modCount;   
  
	  // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;

  	// 自定义的加载因子，没有指定的话后面会使用默认的DEFAULT_LOAD_FACTOR作为填充因子
    final float loadFactor;
}

```

### 1.3. 核心方法

#### 1.3.1. hash()方法

```java
// 用于计算键的Hash值来确定存放table数组的索引位置
static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### 1.2.3. resize()方法

```java
// Map扩容方法
final Node<K,V>[] resize() {
  // 暂存老的核心table数组
  Node<K,V>[] oldTab = table;
  // 老数组的容量
  int oldCap = (oldTab == null) ? 0 : oldTab.length;
  // 老的扩容阈值
  int oldThr = threshold;
  int newCap, newThr = 0;
  // 老容量的大于0的情况
  if (oldCap > 0) {
    // 防止数组越界的处理
    if (oldCap >= MAXIMUM_CAPACITY) {
      threshold = Integer.MAX_VALUE;
      return oldTab;
    }
    // 老容量的两倍小于最大容量，并且老容量是大于等于默认容量的情况
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
      // 新容量是老容量的2倍，扩容1倍
      newThr = oldThr << 1; // double threshold
  }
  // 老的扩容阈值大于0
  else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;
  // 初始化HashMap后，没有任何元素的时候调用resize
  else {               // zero initial threshold signifies using defaults
    // 使用默认容量作为新的容量
    newCap = DEFAULT_INITIAL_CAPACITY;
    // 计算新的扩容阈值
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
  }
  if (newThr == 0) {
    float ft = (float)newCap * loadFactor;
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
  }
  threshold = newThr;
  @SuppressWarnings({"rawtypes","unchecked"})
  // 船新的扩容后的空table数组
  Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
  table = newTab;
  if (oldTab != null) {
    // 循环遍历老数组
    for (int j = 0; j < oldCap; ++j) {
      Node<K,V> e;
      // 取老的元素，并赋个新的e
      if ((e = oldTab[j]) != null) {
        // 老数组位置赋值空指针，待GC回收内存
        oldTab[j] = null;
        // 只有一个元素的情况
        if (e.next == null)
          // 将元素赋值给新数组的新的索引位置上
          newTab[e.hash & (newCap - 1)] = e;
        else if (e instanceof TreeNode) // 是红黑树的情况
          // 迁移红黑树数据
          ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
        else { // preserve order
          // 链表的情况，迁移数据
          Node<K,V> loHead = null, loTail = null;
          Node<K,V> hiHead = null, hiTail = null;
          Node<K,V> next;
          // 循环迭代链表
          do {
            next = e.next;
            if ((e.hash & oldCap) == 0) {
              if (loTail == null)
                loHead = e;
              else
                loTail.next = e;
              loTail = e;
            }
            else {
              if (hiTail == null)
                hiHead = e;
              else
                hiTail.next = e;
              hiTail = e;
            }
          } while ((e = next) != null);
          if (loTail != null) {
            loTail.next = null;
            newTab[j] = loHead;
          }
          if (hiTail != null) {
            hiTail.next = null;
            newTab[j + oldCap] = hiHead;
          }
        }
      }
    }
  }
  return newTab;
}
```



## 2.健值对存放逻辑

1. 判断当前容量是否触发扩容，如果大于容量的负载因子，进行扩容
2. 通过HashMap的hash()方法取得hashCode
3. 通过 (n -1) & hashCode，获取table数组的索引位置，n为table的length
4. 如果当前元素的位置是空的话，直接存入。
5. 如果位置不为空，判断kv对的hash值与已存在的元素的是否相同，如果相同，则进行覆盖处理，不相同进行生成链表
6. 如果相同位置的链表长度符合树化阈值，则将链表转化为红黑树



## 3.常用的方法源码解析

### 3.1. put()方法

```java
// 存放元素的方法
public V put(K key, V value) {
  return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
  Node<K,V>[] tab; Node<K,V> p; int n, i;
  // 如果table数组没有初始化或者为空的情况
  if ((tab = table) == null || (n = tab.length) == 0)
    // 进行扩容
    n = (tab = resize()).length;
  // 确定元素的要存放的索引位置，并取其原始的值
  if ((p = tab[i = (n - 1) & hash]) == null)
    // 原来的位置是空的，直接新建一个新的Node来存放
    tab[i] = newNode(hash, key, value, null);
  // 已存在元素
  else {
    Node<K,V> e; K k;
    // 比较第一个元素的hash和键是否相等
    if (p.hash == hash &&
        ((k = p.key) == key || (key != null && key.equals(k))))
      // 相等的情况直接覆盖
      e = p;
    // 结构是红黑树的情况
    else if (p instanceof TreeNode)
      // 放入树中
      e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
    else {
      // 普通的链表结构，循环到链表的最后一个元素
      for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
          // 新元素链接到链表的最后一个元素上
          p.next = newNode(hash, key, value, null);
          // 判断是否需要转换为红黑树
          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
          // 结束循环
          break;
        }
        // 比较最后的元素，与插入的元素是否相等
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          break;
        p = e;
      }
    }
    // 表示在桶中找到key值、hash值与插入元素相等的结点
    if (e != null) { // existing mapping for key
      V oldValue = e.value;
      // onlyIfAbsent为false的话，用新的值取代旧的值
      if (!onlyIfAbsent || oldValue == null)
        e.value = value;
      afterNodeAccess(e);
      return oldValue;
    }
  }
  // 记录结构修改次数
  ++modCount;
  // 判断是否需要扩容
  if (++size > threshold)
    resize();
  afterNodeInsertion(evict);
  return null;
}
```

### 3.2. get()方法

```java
public V get(Object key) {
  Node<K,V> e;
  return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
  Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
  // 将table指针指向本地变量，并且定位到的数组元素的第一个不是null
  if ((tab = table) != null && (n = tab.length) > 0 &&
      (first = tab[(n - 1) & hash]) != null) {
    // 查找的hash和第一个元素的hash相等
    if (first.hash == hash && // always check first node
        // hash相等，key也是相等的情况下，就返回第一个元素
        ((k = first.key) == key || (key != null && key.equals(k))))
      return first;
    // 不是第一个节点的情况，查看下一个节点
    if ((e = first.next) != null) {
      // 是红黑树的情况
      if (first instanceof TreeNode)
        // 通过红黑树查找得到元素
        return ((TreeNode<K,V>)first).getTreeNode(hash, key);
      // 遍历链表
      do {
        // 找到hash和key匹配的元素返回
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          return e;
      } while ((e = e.next) != null);
    }
  }
  // 没有找到元素，返回null
  return null;
}
```

## 4. 常见问题

#### 1. 为什么默认的容量定义为16个？

可以看到HashMap用于定位数组索引的方法是`(n -1) & hash`，原理同十进制的取模运算。位运算直接对内存数据进行操作，不需要转成十进制，所以位运算要比取模运算的效率更高，所以HashMap在计算元素要存放在数组中的index的时候，使用位运算代替了取模运算。之所以可以做等价代替，前提是要求HashMap的容量一定要是2^n。至于为什么不是2，4，8而是16。因为2，4，8的容量太小，造成过早的扩容，而16以上32太大，浪费内存空间。



#### 2.为什么在JDK1.8之后加入红黑树数据结构？

之前是链表结构，搜索的时间复杂度是O(N)，之后的红黑树结构的搜索时间复杂度是O(logN)明显的红黑树的搜索效率更高。



#### 3.为什么使用与运算(n - 1) & has代替常规的百分号取模运算？

在上面的问题中已经说明，为了提高运算效率，不需要再转成十进制来运算。

## 参考资料

- 《[JavaGuide](https://snailclimb.gitee.io/javaguide/#/docs/java/collection/HashMap)》
- https://blog.csdn.net/hollis_chuang/article/details/103452727


