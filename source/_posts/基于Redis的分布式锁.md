---
title: 基于Redis的分布式锁
date: 2020-09-05 19:34:23
categories:
    - 应用总结
tags: 
    - 分布式
---

# 基于Redis的分布式锁

Redis使用`redisson`来实现分布式锁

```xml
<dependency>
  <groupId>org.redisson</groupId>
  <artifactId>redisson</artifactId>
  <version>3.13.4</version>
</dependency>
```

Redisson文档

https://github.com/redisson/redisson/wiki/1.-%E6%A6%82%E8%BF%B0





redisson配置

```java
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
                .setAddress("redis://localhost:6379")
                .setDatabase(0);
        return Redisson.create(config);
    }
}
```



## 常用的分布式锁的种类

### 可重入锁（ReentrantLock）

​	对于同一把锁，同一个申请者或者线程在已经获取到锁的情况下，可以重复申请锁，而不会导致死锁情况发生的一种锁

```java
@Component
public class LockTest {
    private static final int THREAD_NUMBER = 10;

    @Autowired
    private RedissonClient redissonClient;
  
  	// 可重入锁
  	public void reentrantLock() throws InterruptedException{
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();
        
        System.out.println("测试可重入锁");
        System.out.println("============================================================");
        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            executorService.submit(() -> {
                final RLock lock = redissonClient.getLock("reentrantLock");
                try {
                    if (lock.tryLock(100, 4, TimeUnit.SECONDS)) {
                        System.out.println("获得锁 Thread- " + finalI);
                        Thread.sleep(1000);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                    System.out.println("释放锁 Thread- " + finalI);
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
}
/* 输出（两两输出）
测试可重入锁
============================================================
获得锁 Thread- 2
释放锁 Thread- 2
获得锁 Thread- 1
释放锁 Thread- 1
获得锁 Thread- 7
释放锁 Thread- 7
获得锁 Thread- 6
释放锁 Thread- 6
获得锁 Thread- 10
释放锁 Thread- 10
获得锁 Thread- 4
释放锁 Thread- 4
获得锁 Thread- 8
释放锁 Thread- 8
获得锁 Thread- 9
释放锁 Thread- 9
获得锁 Thread- 5
释放锁 Thread- 5
获得锁 Thread- 3
释放锁 Thread- 3
结束...

*/
```

![可重入锁](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200905193016575-1595991545.gif)







### 公平锁（fairLock）

按申请前后的顺序获取到锁，先去申请的线程先获取到锁

```java
@Component
public class LockTest {
    private static final int THREAD_NUMBER = 10;

    @Autowired
    private RedissonClient redissonClient;
  
  	public void fairLock() throws InterruptedException{
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();

        System.out.println("测试公平锁");
        System.out.println("============================================================");
        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            executorService.submit(() -> {
                final RLock lock = redissonClient.getFairLock("fairLock");
                try {
                    if (lock.tryLock(100, 4, TimeUnit.SECONDS)) {
                        System.out.println("获得锁 Thread- " + finalI);
                        Thread.sleep(1000);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                    System.out.println("释放锁 Thread- " + finalI);
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
}

/* 输出（两两输出）
测试公平锁
============================================================
获得锁 Thread- 4
释放锁 Thread- 4
获得锁 Thread- 2
释放锁 Thread- 2
获得锁 Thread- 8
释放锁 Thread- 8
获得锁 Thread- 1
释放锁 Thread- 1
获得锁 Thread- 7
释放锁 Thread- 7
获得锁 Thread- 10
释放锁 Thread- 10
获得锁 Thread- 6
释放锁 Thread- 6
获得锁 Thread- 3
释放锁 Thread- 3
获得锁 Thread- 9
释放锁 Thread- 9
获得锁 Thread- 5
释放锁 Thread- 5
结束...

*/
```

![公平锁](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200905193043459-816471385.gif)





### 读写锁（RedWriteLock）

​	针对读操作不会引起资源争抢的情况，做的优化锁。它允许同时拥有多把读书和只有一把写锁

```java
@Component
public class LockTest {
    private static final int THREAD_NUMBER = 10;

    @Autowired
    private RedissonClient redissonClient;
  
  	public void readWriteLock() throws InterruptedException {
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();
        System.out.println("读写锁");
        System.out.println("============================================================");
        final RReadWriteLock lock = redissonClient.getReadWriteLock("readWriteLock");

        AtomicInteger radData = new AtomicInteger();

        boolean isRead = true;

        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            if (isRead) {
                executorService.submit(() -> {
                        try {
                            if (lock.readLock().tryLock(100, 30, TimeUnit.SECONDS)) {
                                System.out.println("获得Read锁 Thread-" + finalI);
                                Thread.sleep(1000);
                                System.out.println("读取到数据: " + radData.getOpaque());
                            }else {
                                System.out.println("没有获取到读锁 Thread-" + finalI);
                            }
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } finally {
                            lock.readLock().unlock();
                            countDownLatch.countDown();
                        }
                });
            }else {
                executorService.submit(() -> {
                    try {
                        if (lock.writeLock().tryLock(100, 30, TimeUnit.SECONDS)) {
                            System.out.println("获得Writer锁 Thread-" + finalI);
                            Thread.sleep(1000);
                            final int value = radData.addAndGet(1);
                            System.out.println("写成功：" + value);
                        }else {
                            System.out.println("没有获取到写锁 Thread-" + finalI);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        lock.writeLock().unlock();
                        System.out.println("释放锁 Thread- " + finalI);
                        countDownLatch.countDown();
                    }
                });
            }
            isRead = !isRead;
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
}

/* 输出
读写锁
============================================================
获得Writer锁 Thread-10
写成功：1
释放锁 Thread- 10
获得Writer锁 Thread-6
写成功：2
释放锁 Thread- 6
获得Read锁 Thread-3
获得Read锁 Thread-5
获得Read锁 Thread-7
获得Read锁 Thread-1
获得Read锁 Thread-9
读取到数据: 2
读取到数据: 2
读取到数据: 2
读取到数据: 2
读取到数据: 2
获得Writer锁 Thread-4
写成功：3
释放锁 Thread- 4
获得Writer锁 Thread-2
写成功：4
释放锁 Thread- 2
获得Writer锁 Thread-8
写成功：5
释放锁 Thread- 8
结束...
*/
```


![读写锁](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200905193102664-299319743.gif)


### 信号量（Semaphore）

​	对于同一个资源的并发度做的锁限制，在实例化的时候可以指定许可的数量。每个并发线程要持有一个许可，当许可都被领取空时，后面的线程就会阻塞。常用于做并发度的流控

```java
@Component
public class LockTest {
    private static final int THREAD_NUMBER = 10;

    @Autowired
    private RedissonClient redissonClient;
  
  	public void semaphoreLock() throws InterruptedException {
        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();
        System.out.println("测试信号量锁");
        System.out.println("============================================================");
        final RSemaphore semaphore = redissonClient.getSemaphore("semaphoreLock");
        semaphore.trySetPermits(2);
        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            executorService.submit(() -> {
                try {
                    semaphore.acquire();
                    System.out.println("取得锁 Thread-" + finalI);
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                    System.out.println("释放锁 Thread- " + finalI);
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
}
/* 输出

只有两个并发度，两个线程同时执行

测试信号量锁
============================================================
取得锁 Thread-3
取得锁 Thread-1
释放锁 Thread- 1
释放锁 Thread- 3
取得锁 Thread-7
取得锁 Thread-4
释放锁 Thread- 4
释放锁 Thread- 7
取得锁 Thread-10
取得锁 Thread-2
释放锁 Thread- 2
释放锁 Thread- 10
取得锁 Thread-6
取得锁 Thread-5
释放锁 Thread- 5
释放锁 Thread- 6
取得锁 Thread-8
取得锁 Thread-9
释放锁 Thread- 8
释放锁 Thread- 9
结束...

*/
```

![信号量锁](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200905193122031-781079239.gif)




### 倒计锁（CountDownLatch）

​	常用于协调多个线程，互相等待的场景。在全部线程执行完成后，结束阻塞



```java
@Component
public class LockTest {
    private static final int THREAD_NUMBER = 10;

    @Autowired
    private RedissonClient redissonClient;
  
  	public void countDownLatchLock() throws InterruptedException {
//        final CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUMBER);
        final ExecutorService executorService = Executors.newCachedThreadPool();
        System.out.println("倒数同步锁");
        System.out.println("============================================================");
        final RCountDownLatch countDownLatch = redissonClient.getCountDownLatch("countDownLatchLock");
        countDownLatch.trySetCount(THREAD_NUMBER);
        for (int i = 1; i <= THREAD_NUMBER; i++) {
            int finalI = i;
            executorService.submit(() -> {
                try {
                    System.out.println("取得锁 Thread-" + finalI);
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    countDownLatch.countDown();
                    System.out.println("释放锁 Thread- " + finalI);
                }
            });
        }
        countDownLatch.await();
        System.out.println("结束...");
        System.out.println();
    }
}

/* 输出
倒数同步锁
============================================================
取得锁 Thread-1
取得锁 Thread-7
取得锁 Thread-8
取得锁 Thread-3
取得锁 Thread-6
取得锁 Thread-4
取得锁 Thread-5
取得锁 Thread-10
取得锁 Thread-9
取得锁 Thread-2
释放锁 Thread- 4
释放锁 Thread- 9
释放锁 Thread- 1
释放锁 Thread- 8
释放锁 Thread- 3
释放锁 Thread- 2
释放锁 Thread- 6
释放锁 Thread- 5
释放锁 Thread- 7
释放锁 Thread- 10
结束...
*/
```


![倒数同步锁](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200905193143607-151599541.gif)
