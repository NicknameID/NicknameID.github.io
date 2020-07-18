---
title: AQS源码解析
date: 2020-07-18 14:16:08
categories:
    - Java
    - 并发
tags: 
    - Java基础
---
# AQS源码解析



## AQS是什么？

全称是`AbstractQueuedSynchronizer`，位于java.util.concurrent.locks包下面。AbstractQueuedSynchronizer是一个抽象类，其常见的派生子类有，ReentrantLock.Sync内部类。



### 申请锁入口方法

acquire方法为AQS中用于申请锁定的入口方法

```java
// 先尝试使用去获取锁，如果失败，则尝试加入申请队列
public final void acquire(int arg) {
  // 尝试申请失败，并且加入等待线程队列的后线程状态为中断状态的情况
  // addWaiter也是自旋操作
  // acquireQueued方法中拥有自旋操作
  // 所以 addWaiter 和 acquireQueued只要执行完成说明已经获取到了锁
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    // 进行自我中断
    selfInterrupt();
}
```



tryAcquire 方法是是AQS中用于尝试申请锁的方法，这里用到的模板方法，如果子类中没有重写改方法，那么直接调用会出错

```java
// 尝试锁定成功会返回true，失败返回返回false
protected boolean tryAcquire(int arg) {
  throw new UnsupportedOperationException();
}
```



addWaiter()方法会生成一个等待队列的节点，并且会自旋的去加入等待队列的尾部

```java
private Node addWaiter(Node mode) {
  Node node = new Node(mode);

  // 自旋加入
  for (;;) {
    // 获取当前的队尾元素
    Node oldTail = tail;
    // 如果存在队尾元素
    if (oldTail != null) {
      // 将当前节点添加到队尾
      node.setPrevRelaxed(oldTail);
      // 用CAS算法设置当前AQS的队尾元素为当前节点
      if (compareAndSetTail(oldTail, node)) {
        // 当前队尾元素的的后置节点设置为当前节点，进行两个节点的双向绑定
        oldTail.next = node;
        // 返回当前的节点
        return node;
      }
    } else {
      // 如果不存在队尾元素，就执行队列初始化操作
      initializeSyncQueue();
      // 执行完成后，但下一个for循环进入后，初始化完毕了
    }
  }
}
```



initializeSyncQueue()初始化等待队列

```java
private final void initializeSyncQueue() {
  Node h;
  // 用CAS算法生成一个头节点
  if (HEAD.compareAndSet(this, null, (h = new Node())))
    // 生成头节点成功后，将头节点也指定为尾节点
    tail = h;
}
```



acquireQueued()调度申请线程队列的方法

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean interrupted = false;
  try {
    // 自旋处理等待的线程队列
    for (;;) {
      // 取当前队列的前置节点，p
      final Node p = node.predecessor();
      // 如果p是AQS的头节点，那先去尝试获取AQS的锁如果成功
      if (p == head && tryAcquire(arg)) {
        // 设置当前的节点为头节点
        setHead(node);
        // 节点已被使用，将元素指定为null，帮助GC回收内存
        p.next = null; // help GC
        // 返回当前线程的是的中断状态
        return interrupted;
      }
      // 不是头节点或者不能成功申请到锁的情况
      // 当前线程是否需要暂停
      if (shouldParkAfterFailedAcquire(p, node))
        // 等同于 interrupted = interrupted | parkAndCheckInterrupt()
        // 暂停当前线程，并获取当前线程的中断状态
        interrupted |= parkAndCheckInterrupt();
    }
  } catch (Throwable t) {
    cancelAcquire(node);
    if (interrupted)
      selfInterrupt();
    throw t;
  }
}
```



shouldParkAfterFailedAcquire()判断当前等待的线程节点是否需要阻塞并停止调度，传入当前节点的前置节点和当前节点作为参数

waitStatus的种类
-  初始化状态：0
-   SIGNAL：-1
-   CANCELLED: 1
-   CONDITION: -2
-   PROPAGATE: -3

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  // 前置的节点的等待状态
  int ws = pred.waitStatus;
  // 如果前置节点为SIGNAL状态，说明当前节点为下一个启动节点，可以暂停当前节点线程的调度
  if (ws == Node.SIGNAL)
    /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
    return true;
  // 如果大于0，说明前置节点已被取消
  if (ws > 0) {
    /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
    // 跳过当前的前置节点，往前寻找下一个不是取消状态的节点
    do {
      // 当前前置节点的前一个节点作为前置节点，并且将其指定为当前节点的前置节点
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    // 跳过前置节点后寻找到的非取消节点的后置节点为当前节点
    pred.next = node;
  } else {
    /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
    // 如果是初始状态没有设置过waitStatus的情况或者是共享节点用到的需要传播状态时
    // 使用CAS算法将前置节点的状态变为SIGNAL状态
    pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
  }
  return false;
}
```



parkAndCheckInterrupt() 阻塞当前线程并停止其调度，以节约系统的资源。

```java
private final boolean parkAndCheckInterrupt() {
  // 暂停当前线程
  LockSupport.park(this);
  // 返回当前线程的中断状态，true表示当前线程已经被中断
  return Thread.interrupted();
}
```

