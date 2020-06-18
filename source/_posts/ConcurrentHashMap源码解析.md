---
title: ConcurrentHashMap源码解析
date: 2020-06-18 18:21:27
tags:
    - Java
categories:
    - Java基础
---
# ConcurrentHashMap源码解析

## ConcurrentHashMap是什么？

它是对HashMap线程安全性的增强类，保证了Map对象在多线程环境下的读写的线程安全性。在使用方法上和HashMap保持一致，都是Map接口的实现类。

##  类结构
![ConcurrentHashMap](https://i.loli.net/2020/06/18/9b2daqDHfTItKMc.png)



## 核心数据结构

核心数据结构和HashMap相同，都是采用数组链表或数组红黑树的结构
![](https://i.loli.net/2020/05/25/YmSLJGAnVx5qUsz.png)



## 核心源码解读（源码来自JDK11）

### initTable方法

initTable()方法用于核心数据结构对象数组的初始化操作

```java
// Table初始化和扩容时的控制，-1为正在有线程对其进行初始化；-N 说明有N-1个线程正在进行扩容；大于等于0的情况表示table的容量
private transient volatile int sizeCtl;

private static final Unsafe U = Unsafe.getUnsafe();

// 初始化操作
private final Node<K,V>[] initTable() {
  Node<K,V>[] tab; int sc;
  while ((tab = table) == null || tab.length == 0) {
    // 判断是否有其他线程在进行初始化操作
    if ((sc = sizeCtl) < 0)
      // 放弃线程竞争，仅空转
      Thread.yield(); // lost initialization race; just spin
    
    // 使用Java内部的本地方法，比较SIZECTL与预期值sc是否相同，相同的话，将sizeCtl更新为-1，成功返回true
    else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
      try {
        if ((tab = table) == null || tab.length == 0) {
          // 算得初始化容量n
          int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
          // 初始化Node数组
          @SuppressWarnings("unchecked")
          Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
          // 将初始化好的Node数组赋值给table成员变量
          table = tab = nt;
          // sc = n * (3/4); sc变为原来的0.75倍，对应默认的负载因子。表示下次扩容的触发的真实元素个数
          sc = n - (n >>> 2);
        }
      } finally {
        // 设置sizeCtl为sc
        sizeCtl = sc;
      }
      break;
    }
  }
  return tab;
}
```

### put方法

```java
public V put(K key, V value) {
  return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
  if (key == null || value == null) throw new NullPointerException();
  // 计算当前键的hash
  int hash = spread(key.hashCode());
  int binCount = 0;
  // 循环阻塞当前执行栈，直到满足跳出条件
  for (Node<K,V>[] tab = table;;) {
    // 定位到的头元素f
    Node<K,V> f; int n, i, fh; K fk; V fv;
    // 如果当前的table是空数组
    if (tab == null || (n = tab.length) == 0)
      // 自旋CAS初始化数组
      tab = initTable();
    // 使用hash与数组长度取模，获取指定位置的桶
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
      // 桶的位置是空的情况，直接放入新元素
      if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
        // 加完后直接跳出
        break;                   // no lock when adding to empty bin
    }
    else if ((fh = f.hash) == MOVED)
      // 桶元素的hash是否为-1，表示当前对象正在被其他线程扩容，当前线程协助其他线程加快扩容速度
      tab = helpTransfer(tab, f);
    // 桶元素的hash、key是否和待put的元素是否相等，如果相等就没必要加了，直接返回
    else if (onlyIfAbsent // check first node without acquiring lock
             && fh == hash
             && ((fk = f.key) == key || (fk != null && key.equals(fk)))
             && (fv = f.val) != null)
      return fv;
    else {
      V oldVal = null;
      // 对桶元素进行同步操作协调
      synchronized (f) {
        if (tabAt(tab, i) == f) {
          if (fh >= 0) {
            // 该桶为一个链表
            binCount = 1;
            // 遍历该链表的操作，和待put的元素的hash和key进行比较
            for (Node<K,V> e = f;; ++binCount) {
              K ek;
              // 如果存在则替换
              if (e.hash == hash &&
                  ((ek = e.key) == key ||
                   (ek != null && key.equals(ek)))) {
                oldVal = e.val;
                if (!onlyIfAbsent)
                  e.val = value;
                break;
              }
              Node<K,V> pred = e;
              // 不存在的话，用在链表的末尾加入当前元素
              if ((e = e.next) == null) {
                pred.next = new Node<K,V>(hash, key, value);
                break;
              }
            }
          }
          // 是红黑树的结构
          else if (f instanceof TreeBin) {
            Node<K,V> p;
            binCount = 2;
            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                  value)) != null) {
              oldVal = p.val;
              if (!onlyIfAbsent)
                p.val = value;
            }
          }
          else if (f instanceof ReservationNode)
            throw new IllegalStateException("Recursive update");
        }
      }
      if (binCount != 0) {
        // 如果链表节点长度大于等于树化阈值将链表转换为红黑二叉树
        if (binCount >= TREEIFY_THRESHOLD)
          treeifyBin(tab, i);
        if (oldVal != null)
          return oldVal;
        break;
      }
    }
  }
  addCount(1L, binCount);
  return null;
}

final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果 table 不是空 且 node 节点是转移类型，数据检验
    // 且 node 节点的 nextTable（新 table） 不是空，同样也是数据校验
    // 尝试帮助扩容
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        // 根据 length 得到一个标识符号
        int rs = resizeStamp(tab.length);
        // 如果 nextTab 没有被并发修改 且 tab 也没有被并发修改
        // 且 sizeCtl  < 0 （说明还在扩容）
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            // 如果 sizeCtl 无符号右移  16 不等于 rs （ sc前 16 位如果不等于标识符，则标识符变化了）
            // 或者 sizeCtl == rs + 1  （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
            // 或者 sizeCtl == rs + 65535  （如果达到最大帮助线程的数量，即 65535）
            // 或者转移下标正在调整 （扩容结束）
            // 结束循环，返回 table
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 如果以上都不是, 将 sizeCtl + 1, （表示增加了一个线程帮助其扩容）
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 进行转移
                transfer(tab, nextTab);
                // 结束循环
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### get方法

```java
public V get(Object key) {
  Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
  // 计算key的hash值
  int h = spread(key.hashCode());
  if ((tab = table) != null && (n = tab.length) > 0 &&
      (e = tabAt(tab, (n - 1) & h)) != null) {
    if ((eh = e.hash) == h) {
          // 桶元素的hash等于当前key的hash, key也相等的情况下找到目标元素
      if ((ek = e.key) == key || (ek != null && key.equals(ek)))
        return e.val;
    }
    // 正在扩容或者是红黑树
    else if (eh < 0)
      return (p = e.find(h, key)) != null ? p.val : null;
    // 沿链表寻找hash和key匹配的元素，匹配到就返回
    while ((e = e.next) != null) {
      if (e.hash == h &&
          ((ek = e.key) == key || (ek != null && key.equals(ek))))
        return e.val;
    }
  }
  return null;
}
```

## 参考

- https://snailclimb.gitee.io/javaguide/#/docs/java/collection/ConcurrentHashMap
- https://www.jianshu.com/p/39b747c99d32
