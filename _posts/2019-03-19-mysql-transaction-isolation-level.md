---
title: MySQL事务隔离级别总结
tags: 
    - MySQL 
    - summary
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
---

事务隔离是数据库处理的基础。隔离是ACID中的i（isolation）。当有多个事务在更改数据同时查询数据时，事务隔离级别调整的是数据库性能、可靠性、一致性和结果的可重复性之间的平衡。

MySQL的InnoDB存储引擎提供4种事务隔离级别：READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, and SERIALIZABLE。InnoDB的默认事务隔离级别时REPEATABLE_READ。

InnoDB中，对这4中事务隔离级别使用不同的锁策略。如果对于关键数据，ACID的一致性时十分重要的，你可以用默认的REPEATABLE_READ级别来保证数据的强一致性。但是，如果对比起大幅减少锁带来的性能提升，数据的强一致性不是那么重要，例如批量导出数据时，你可以将数据库的事务隔离级别改为READ_COMMITTED甚至是READ_UNCOMMITTED。SERIALIZABLE隔离级别是比REPEATABLE_READ更严格的隔离级别，通常用在一些特殊场合，例如：[XA事务](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_xa)，和解决并发中存在的死锁。

### 设置事务隔离级别

  1. 设置全局的事务隔离级别

  这个sql语句会影响接下来的所有会话的事务隔离级别，已经开始的会话不会受影响。

  ```sql
  SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
  SET GLOBAL TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  ```

  2. 设置当前会话期间的事务隔离级别

  这个sql语句会影响当前会话所有接下来的事务。
  可以在一个事务中间执行，不会影响当前执行中的事务隔离级别。
  如果在两个事务之间执行该sql语句，会以该事务隔离级别覆盖两个事物之间执行的sql语句。

  ```sql
  SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
  SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  ```

  3. 设置下一个事务的事务隔离级别

  这个sql语句只会影响当前会话的下一个事务。
  下一个事务之后的事务的隔离级别会回到该sql语句之前的隔离级别。
  该sql语句是不能在一个事务中间执行的。

  ```sql
  SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
  SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  ```

  ```shell
  mysql> START TRANSACTION;
  Query OK, 0 rows affected (0.02 sec)

  mysql> SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  ERROR 1568 (25001): Transaction characteristics can't be changed
  while a transaction is in progress
  ```

### REPEATTABLE_READ

这个是InnoDB的默认隔离级别。REPEATABLE READ是用一致性读(consistent reads，也叫快照读)的方法来实现可重复读的。在同一个事务里面，所有个select语句都是基于第一个select语句建立的快照。这意味着如果在同一事务中发起的多个非锁定(nonlocking)select语句，则这些select语句相互之间是一致的。

而对于锁读(locking reads，即select for update / for share)，update和delete语句，锁的范围取决于这些sql语句是使用了唯一搜索条件的唯一索引，还是一个范围搜索。

* 如果是使用了唯一搜索条件的唯一索引，InnoDB只会锁住哪一行。

* 否则，InnoDB会锁住它扫描的区间，在这个区间内，InnoDB会用gap locks或者next-key locks锁住，不让其他session的事务往这个区间插入数据。

### READ_COMMITTED



### READ_UNCOMMITTED

### SERIALIZABLE

#### 扩展

* consistent reads

  这是一种基于某一个时间戳的数据库快照的数据读取方式，也叫快照读。如果查询的数据被其他事务修改了，则基于撤销日志的内容重建原始数据。这种读取数据的方式可以避免在读取数据时要等待其他并发的事务释放锁之后才能读取导致性能降低的问题。

  如果数据库的事务隔离级别时REPEATABLE READ，那快照基于的时间戳是事务开始之后的第一个读取操作的时间。如果事务隔离级别时READ COMMITTED，那么每次读取操作都会重置快照。

  在InnoDB中，当隔离级别是REPEATABLE READ或READ COMMITTED时，consistent read(快照读)是select语句的默认模式。这是因为快照读不会对它读取的表上锁，其他会话可以修改快照读正在读取的数据。

* consistent nonlocking reads

  

* InnoDB locking




#### 参考

[https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
