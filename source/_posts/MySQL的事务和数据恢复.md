---
title: MySQL的事务和数据恢复
date: 2020-06-23 16:12:18
categories:
    - 数据库
tags: 
    - MySQL
---
# MySQL的事务和数据恢复

## 并发事务带来的问题

- **脏读**：某一个事务修改了数据，但未提交的情况下，这时另一个事务读取了该行数据。那么读取的这个事务读的数据称为脏数据。这种情况称为脏读
- **不可重复读**：在一个事务内，多次读取同一个数据，这个事务没有结束时，另一个事务修改了该数据，导致上一个事务中，多次读取的数据不一致的问题，因此称为不可重复读
- **幻读**：与不可重复读相似。一个事务读取了几行数据，在当前事务未提交的时候，另外一个事务插入了几条新的数据。前一个事务在随后的查询中发现新多了几条数据

不可重复读与幻读的区别：不可重复读出现在数据更新的事务中，如UPDATE语句中。而幻读出现在数据增删的情况中，如INSERT，DELETE语句中

## MySQL的事务隔离等级

- **(READ-UNCOMMITTED)读取未提交**: 允许读取未提交的数据,最低的隔离等级,可能会导致脏读,幻读,不可重复读。
- **(READ-COMMITTED)读取已提交**: 允许读取已提交的数据, 避免了脏读, 但会存在不可重复读和幻读。
- **(REPEATABLE-READ)可重复读**: MySQL默认的事务隔离等级,保证同一个字段的多次读取结果一致, 除非被本事务修改。避免了脏读，不可重复读，但有在幻读的可能。
- **(SERIALIZEABLE)串行化**：最高的隔离等级，所有事务逐个执行，事务之间不存在干扰。



#### 数据库事务隔离等级的通俗总结的很好

- 读未提交：别人改数据的事务尚未提交，我在我的事务中也能读到。
- 读已提交：别人改数据的事务已经提交，我在我的事务中才能读到。
- 可重复读：别人改数据的事务已经提交，我在我的事务中也不去读。
- 串行：我的事务尚未提交，别人就别想改数据。
  这4种隔离级别，并行性能依次降低，安全性依次提高。



#### Spring的事务隔离等级

```
package org.springframework.transaction.annotation;

public enum Isolation {
    DEFAULT(-1),
    READ_UNCOMMITTED(1),
    READ_COMMITTED(2),
    REPEATABLE_READ(4),
    SERIALIZABLE(8);

    private final int value;

    private Isolation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }
}
```

和MySQL的事务隔离等级相互对应。值多了一个默认枚举类型，`DEFAULT`表示遵循数据库的事务隔离等级



#### Spring的事务传递行为

什么是事务的传播行为？比如在方法A中开启了事务，在事务B中也开启了事务，在执行期间事务的维护情况，就是事务的传播行为。总共有下面七种

- REQUIRED：指定的方法必须在事务内执行。若当前存在事务，就加入到当前事务中；若当前没有事务，则创建一个新事务。这种传播行为是最常见的选择，也是 Spring 默认的事务传播行为。
- SUPPORTS：指定的方法支持当前事务，但若当前没有事务，也可以以非事务方式执行。
- MANDATORY：指定的方法必须在当前事务内执行，若当前没有事务，则直接抛出异常。
- REQUIRES_NEW：总是新建一个事务，若当前存在事务，就将当前事务挂起，直到新事务执行完毕。
- NOT_SUPPORTED：指定的方法不能在事务环境中执行，若当前存在事务，就将当前事务挂起。
- NEVER：指定的方法不能在事务环境下执行，若当前存在事务，就直接抛出异常。
- NESTED：指定的方法必须在事务内执行。若当前存在事务，则在嵌套事务内执行；若当前没有事务，则创建一个新事务。



#### MySQL中如何查看当前数据库系统的事务隔离等级

```mysql

mysql> show variables like 'transaction_isolation';

+-----------------------+----------------+

| Variable_name | Value |

+-----------------------+----------------+

| transaction_isolation | READ-COMMITTED |

+-----------------------+----------------+
```





## InnoDB存储引擎的锁的算法

- Record lock：单个行记录上的锁
  - 在查询存在唯一索引的时候用来锁定单条记录
- Gap lock：间隙锁，锁定一个范围，不包括记录本身
  - 设计的目的是解决多个事务将记录插入到同一个范围内，导致幻读问题
  - 如：查询条件中存在范围条件语句，如：between、> 、<、 <= 、>=等
- Next-key lock：record+gap 锁定一个范围，包含记录本身
  - 设计的目的是解决幻读问题



## MySQL是如何保证机器异常重启后不丢失数据的？

涉及到MySQL的数据更新机制。binlog（归档日志）和redolog（重做日志）。MySQL默认的InnoDB引擎会将更新数据先存在内存中，同时将记录redolog，该条redolog的状态是perpare，然后将操作提交至执行器。执行器收到任务后，写入binlog，同时调用引擎接口写入或更新数据，成功后将之前的redolog改为commit状态。

如果在更新或写入数据的过程中，机器出现崩溃。那么在机器在重启后，MySQL会首先去验证redolog的完整性，如果redolog中没有prepare状态的记录，则记录是完整的，就日记提交。如果redolog中存在prepare记录，那么就去验证这条redolog对应的binlog记录，如果这条binlog是完整的，那么完整提交redolog，否则执行回滚逻辑



## 参考资料

《MySQL实战45讲》---极客时间

[https://www.funtl.com/zh/spring-transaction/#spring-%E4%BA%8B%E5%8A%A1%E7%AE%A1%E7%90%86-api](https://www.funtl.com/zh/spring-transaction/#spring-事务管理-api)



