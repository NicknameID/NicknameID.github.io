---
title: 基于ZooKeeper的分布式锁
date: 2020-09-09 14:59:15
categories: 
    - 分布式
tags: 
    - 分布式
---

# 基于ZooKeeper的分布式锁

这里介绍一下org.apache.curato的所有组件

| 组件                       | 介绍                                                         |
| -------------------------- | ------------------------------------------------------------ |
| curator-client             | 代替ZK官方提供的ZK客户端组件                                 |
| curator-framework          | 在Client基础上构建的高级API，更加方便的使用（依赖管理工具会自动引入底层的Client依赖） |
| curator-recipes            | Zookeeper 所有的典型应用场景的实现（除了两阶段提交外）该组件依赖 Client 和 Framework<br/>包括监听、各种分布式锁（可重入锁、排他锁、共享锁、信号锁等）、缓存、队列、选举、<br/>分布式 atomic（分布式计数器）、分布式Barrier 等等。 |
| curator-examples           | 各种高级的使用例子                                           |
| curator-x-discovery        | 在framework基础上的一个服务发现的实现                        |
| curator-x-discovery-server | RESTful风格的服务发现服务器                                  |

这里我们引入`curator-recipes`包含了Client和Framework组件

```xml
<!-- https://mvnrepository.com/artifact/org.apache.curator/curator-recipes -->
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>5.1.0</version>
</dependency>
```



### 可重入锁（InterProcessMutex）

```java
public void reentrantLock() throws Exception {
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();

        final String lockName = "/lock/reentrantLock";

        System.out.println("测试ZK可重入锁");
        System.out.println("============================================================");
        final InterProcessMutex lock = new InterProcessMutex(zkClient, lockName);
        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            executorService.submit(() -> {
                try {
                    if (lock.acquire(100, TimeUnit.SECONDS)) {
                        System.out.println("获得锁 Thread- " + finalI);
                        Thread.sleep(1000);
                    }
                }catch (Exception e) {
                    e.printStackTrace();
                }finally {
                    try {
                        lock.release();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    countDownLatch.countDown();
                }

            });
        }
        countDownLatch.await();
//        Revoker.attemptRevoke(zkClient, lockName);
        System.out.println("结束...");
        System.out.println();
    }
/* 输出
测试ZK可重入锁
============================================================
获得锁 Thread- 5
获得锁 Thread- 3
获得锁 Thread- 9
获得锁 Thread- 1
获得锁 Thread- 8
获得锁 Thread- 7
获得锁 Thread- 6
获得锁 Thread- 4
获得锁 Thread- 2
获得锁 Thread- 10
结束...
*/
```



### 可重入读写锁（InterProcessReadWriteLock）

```java
public void readWriteLock() throws Exception {
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();

        final String lockName = "/lock/readWriteLock";

        System.out.println("ZK读写锁");
        System.out.println("============================================================");
        final InterProcessReadWriteLock lock = new InterProcessReadWriteLock(zkClient, lockName);
        AtomicInteger radData = new AtomicInteger();

        boolean isRead = true;

        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            if (isRead) {
                executorService.submit(() -> {
                    try {
                        if (lock.readLock().acquire(30, TimeUnit.SECONDS)) {
                            System.out.println("获得Read锁 Thread-" + finalI);
                            Thread.sleep(1000);
                            System.out.println("Thread-" + finalI + " 读取到数据: " + radData.getOpaque());
                        }else {
                            System.out.println("没有获取到读锁 Thread-" + finalI);
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            lock.readLock().release();
                            System.out.println("释放写锁 Thread- " + finalI);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }finally {
                            countDownLatch.countDown();
                        }
                    }
                });
            }else {
                executorService.submit(() -> {
                    try {
                        if (lock.writeLock().acquire(30, TimeUnit.SECONDS)) {
                            System.out.println("获得Write锁 Thread-" + finalI);
                            Thread.sleep(1000);
                            final int value = radData.addAndGet(1);
                            System.out.println("Thread- "+ finalI +" 写成功：" + value);
                        }else {
                            System.out.println("没有获取到写锁 Thread-" + finalI);
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            lock.writeLock().release();
                            System.out.println("释放写锁 Thread- " + finalI);
                        } catch (Exception e) {
                            e.printStackTrace();
                        } finally {
                            countDownLatch.countDown();
                        }
                    }
                });
            }
            isRead = !isRead;
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
/* 输出
ZK读写锁
============================================================
获得Write锁 Thread-2
Thread- 2 写成功：1
获得Write锁 Thread-6
释放写锁 Thread- 2
Thread- 6 写成功：2
释放写锁 Thread- 6
获得Write锁 Thread-4
Thread- 4 写成功：3
释放写锁 Thread- 4
获得Read锁 Thread-1
Thread-1 读取到数据: 3
释放写锁 Thread- 1
获得Write锁 Thread-10
Thread- 10 写成功：4
释放写锁 Thread- 10
获得Read锁 Thread-7
Thread-7 读取到数据: 4
释放写锁 Thread- 7
获得Write锁 Thread-8
Thread- 8 写成功：5
释放写锁 Thread- 8
获得Read锁 Thread-5
获得Read锁 Thread-3
获得Read锁 Thread-9
Thread-5 读取到数据: 5
Thread-9 读取到数据: 5
Thread-3 读取到数据: 5
释放写锁 Thread- 5
释放写锁 Thread- 9
释放写锁 Thread- 3
结束...
*/
```





### 信号量（InterProcessSemaphoreV2）

```java
public void semaphoreLock() throws Exception {
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();
        System.out.println("测试ZK信号量锁");
        System.out.println("============================================================");
        final InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(zkClient, "/lock/semaphore", 3);
        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            executorService.submit(() -> {
                Lease lease = null;
                try {
                    lease = semaphore.acquire(30, TimeUnit.SECONDS);
                    System.out.println("取得锁 Thread-" + finalI);
                    Thread.sleep(1000);
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    if (lease != null) {
                        semaphore.returnLease(lease);
                    }
                    System.out.println("释放锁 Thread- " + finalI);
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
/* 信号量
测试ZK信号量锁
============================================================
取得锁 Thread-4
取得锁 Thread-9
取得锁 Thread-7
释放锁 Thread- 4
取得锁 Thread-6
释放锁 Thread- 9
取得锁 Thread-8
释放锁 Thread- 7
取得锁 Thread-10
释放锁 Thread- 6
取得锁 Thread-5
释放锁 Thread- 8
取得锁 Thread-3
释放锁 Thread- 10
取得锁 Thread-1
释放锁 Thread- 5
取得锁 Thread-2
释放锁 Thread- 3
释放锁 Thread- 1
释放锁 Thread- 2
结束...
*/
```



### 栅栏（DistributedBarrier）

```java
public void barrierLock() throws Exception {
  final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
  final ExecutorService executorService = Executors.newCachedThreadPool();
  System.out.println("ZK栅栏同步锁");
  System.out.println("============================================================");
  final DistributedBarrier lock = new DistributedBarrier(zkClient, "/lock/barrier");
  lock.setBarrier();
  for (int i = 1; i <= THREAD_NUMBER; i++) {
    int finalI = i;
    executorService.submit(() -> {
      try {
        System.out.println("等待中 Thread-" + finalI);
        lock.waitOnBarrier();
        System.out.println("启动 Thread-" + finalI);
      } catch (Exception e) {
        e.printStackTrace();
      } finally {
        countDownLatch.countDown();
      }
    });
  }
  Thread.sleep(5000);
  lock.removeBarrier();
  countDownLatch.await();
  System.out.println("结束...");
  System.out.println();
}
/* 输出
ZK栅栏同步锁
============================================================
等待中 Thread-1
等待中 Thread-5
等待中 Thread-7
等待中 Thread-8
等待中 Thread-9
等待中 Thread-10
等待中 Thread-2
等待中 Thread-6
等待中 Thread-4
等待中 Thread-3
启动 Thread-9
启动 Thread-10
启动 Thread-2
启动 Thread-4
启动 Thread-6
启动 Thread-1
启动 Thread-3
启动 Thread-5
启动 Thread-8
启动 Thread-7
结束...
*/
```

