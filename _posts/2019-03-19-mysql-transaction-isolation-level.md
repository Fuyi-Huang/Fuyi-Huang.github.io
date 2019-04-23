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

这个是InnoDB的默认隔离级别。REPEATABLE READ是用一致性读(consistent reads，也叫快照读)的方法来实现可重复读的。在同一个事务里面，所有个select语句都是基于事务开始时建立的快照。这意味着如果在同一事务中发起的多个非锁定(nonlocking)select语句，则这些select语句相互之间是一致的。

而对于锁读(locking reads，即select for update / for share)，update和delete语句，锁的范围取决于这些sql语句是使用了唯一搜索条件的唯一索引，还是一个范围搜索，具体可以看何登成的这一篇博客[MySQL 加锁处理分析](http://hedengcheng.com/?p=771)。


### READ_COMMITTED

使用READ COMMITTED隔离级别会有以下效果：

  - 对于update或delete语句，InnoDB会锁住它更新或删除的行直到事务结束。而对于扫描过的不匹配的行，数据库会先上锁，比对过不匹配之后会把锁释放掉。这会大大减小死锁发生的可能性，但还是会存在。

  - 对于update语句，如果一行已经被锁定了，InnoDB会采取一种"semi-consistent"读的方式，读取最新已提交的数据，然后比对数据是否匹配where条件，如果条件匹配，mysql会再读一次，这次会锁住并等待锁的释放；如果不匹配则跳过。

请考虑以下示例：
```sql
CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
COMMIT;
```

在这种情况下，表没有索引，因此搜索和索引扫描会使用隐藏的聚集索引。

假设一个会话用下面的语句更新数据：
```sql
# Session A
START TRANSACTION;
UPDATE t SET b = 5 WHERE b = 3;
```

假设有另一个会话在上面的会话之后执行下面的更新操作：
```sql
# Session B
UPDATE t SET b = 4 WHERE b = 2;
```

每当InnoDB执行update语句，它会先去获取获取每一行的排他锁，然后判断是否更改它。如果InnoDB没有更改一行，它会释放掉锁；否则，InnoDB会一直持有这个锁直到事务结束。

当使用默认的REPEATABLE READ隔离级别时，上面的会话A事务会获取它读取过的每一行的x-lock而并不会释放：

  ```
  x-lock(1,2); retain x-lock
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); retain x-lock
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); retain x-lock
  ```

当上面的会话B执行update操作时，也会尝试去获取x-lock(会话A已经持有这些锁)，这时会话B会被阻塞直到会话A提交或者回滚。

  ```
  x-lock(1,2); block and wait for first UPDATE to commit or roll back
  ```

如果会话A和会话B执行是在READ COMMITTED隔离级别，那么会话A获取了每一行的x-lock之后，会释放掉判断不会更改的那些行的锁：

  ```
  x-lock(1,2); unlock(1,2)
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); unlock(3,2)
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); unlock(5,2)
  ```

对于会话B执行的update, InnoDB会用"semi-consistent"读，读取每一行的最近commit的版本，来判断是否满足update语句中where子句中的条件：

  ```
  x-lock(1,2); update(1,2) to (1,4); retain x-lock
  x-lock(2,3); unlock(2,3)
  x-lock(3,2); update(3,2) to (3,4); retain x-lock
  x-lock(4,3); unlock(4,3)
  x-lock(5,2); update(5,2) to (5,4); retain x-lock
  ```

考虑另一种情况，如果where子句中使用了索引行做判断，InnoDB会使用索引，索引行会被使用来记录行锁。
在下面的例子中，会话A的update操作会持有b=2的每一行的x-lock，因为InnoDB记录了索引b=2的锁被会话A获取了。之后，会话B的update操作会被阻塞，因为它也是需要获取索引行b=2的行锁。

```sql
CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2,3),(2,2,4);
COMMIT;

# Session A
START TRANSACTION;
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

# Session B
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
```


### READ_UNCOMMITTED

在这种事务隔离级别下，select语句的读取方式是非阻塞的，但是读取出来的数据是不可靠的，即脏读。



### SERIALIZABLE

这个隔离级别类似于REPEATABLE READ，但不同的是，如果禁止了数据库的autocommit，那InnoDB会隐式地将所有的select语句转为select ... for share；如果启用了autocommit，则每一个select语句就是一个事务。因此，如果只是用来读取数据库，则可以序列化而不用阻塞其他事务的执行。




#### 扩展

* consistent reads

  这是一种基于某一个时间戳的数据库快照的数据读取方式，也叫快照读。如果查询的数据被其他事务修改了，则基于撤销日志的内容重建原始数据。这种读取数据的方式可以避免在读取数据时要等待其他并发的事务释放锁之后才能读取导致性能降低的问题。

  如果数据库的事务隔离级别时REPEATABLE READ，那快照基于的时间戳是事务开始之后的第一个读取操作的时间。如果事务隔离级别时READ COMMITTED，那么每次读取操作都会重置快照。

  在InnoDB中，当隔离级别是REPEATABLE READ或READ COMMITTED时，consistent read(快照读)是select语句的默认模式。这是因为快照读不会对它读取的表上锁，其他会话可以修改快照读正在读取的数据。

* consistent nonlocking reads

  consistent reads(一致性读)，Innodb用的是基于一个时间点的快照的多版本。查询操作可以查询到快照时间点之前提交的事务更新的数据，查不到该时间点之后提交的事务更新的数据。但是，可以看到同一个事务里面前面的语句的更新结果。

  如果数据库的事务隔离级别是REPEATABLE READ(默认的)，那么同一个事务的所有一致性读操作都是基于事务的第一个读取操作的时间点(一般是start transaction;语句)建立的快照。可以通过提交事务的方式来获取一个新的快照。

  ```
                Session A              Session B

            SET autocommit=0;      SET autocommit=0;
  time
  |          SELECT * FROM t;
  |          empty set
  |                                 INSERT INTO t VALUES (1, 2);
  |
  v          SELECT * FROM t;
            empty set
                                    COMMIT;

            SELECT * FROM t;
            empty set

            COMMIT;

            SELECT * FROM t;
            ---------------------
            |    1    |    2    |
            ---------------------
  ```

  如果数据库的事务隔离级别是READ COMMITTED，那么每一个一致性读操作都会重置快照的时间点为最新。

  一致性读是READ COMMITTED和REPEATABLE READ事务隔离级别下select语句的默认数据读取方式。这种方式不会给读取的数据行上锁，其他的并行事务可以读取并更改。

  > 数据库的快照适用于事务中的select语句，不一定适用于DML语句。如果插入或者修改某些行然后提交事务，则从另一个并发的REPEATABLE READ事务发出的delete或update语句可能会影响那些刚刚提交的行，即使会话无法查询到它们。如果事务确实更新或者删除了由其他事务提交的行，则这些更改对当前事务可见。例如：

    ```sql
    SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
    -- 返回0：没有匹配的行.
    DELETE FROM t1 WHERE c1 = 'xyz';
    -- 这里有可能会删除其他事务刚刚提交的行

    SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
    -- 返回0：没有匹配的行
    UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
    -- Affects 10 rows: 将刚刚提交的10行的值改为了'abc'.
    SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
    -- 返回10：现在这个事务可以看到刚刚它更新的行.
    ```




* InnoDB locking



* dirty read

  又名脏读，是指读取出来的数据不是可靠数据的读取方式，这些不可靠数据一般是其他未提交的事务写的数据。脏读一般发生在事务隔离级别设置为READ UNCOMMITTED(读未提交)时。

  读取出来的是数据不可靠，这是因为这些数据的更改的那个事务可能rollback，或者在事务commit之前进一步更改。这样读取方式是违反了数据库设ACID设计原则的。在生产环境的数据库这样子用是有很大风险的，一般不建议。



#### 参考

[https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
