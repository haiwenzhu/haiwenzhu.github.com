---
layout: post
comments: true
title: "InnoDB锁漫谈"
summary: "InnoDB不同的锁类型说明"
categories:
  - MySQL
---

#### 前言
当我们谈论数据库，尤其是关系数据库时，锁是一个永远也避不开的话题，他是关系数据库保证ACID特性的基础。具体到InnoDB，它支持那些锁？不同的锁有什么不一样？本文尝试着回答这些问题。

#### 共享锁和排他锁(Shared and Exclusive Locks)
InnoDB的锁粒度是可到行级别的，有两种锁类型：
- 共享锁(S lock)：事务为了读某行而加的锁
- 排他锁(X lock)：事务为了更新或删除某行而加的锁

如果事务对某一行如果已经加了共享锁，其他事务还可以为这一行加共享锁，但是无法加排他锁。
如果事务对某一行加了排他锁，则其他事务不可以再对这一行加共享锁或排他锁。

#### 意向锁(Intention Locks)
所谓意向锁，简单点理解就是为了锁而锁，比如在一个事务里，读取某行数据的同时，不希望再有其它事务更新这行数据，可以用`SELECT ... FOR UPDATE`加上一个排他意向锁。
意向锁也有两种类型，共享意向锁(IS)和排他意向锁(IX)，各种类型的锁的互斥逻辑如下表：

 |  排他锁(X)  |  排他意向锁(IX)  |  共享锁(S)  |  共享意向锁(IS)
--- | ------ | -------------- | --------- | --------------
排他锁(X) | 冲突 | 冲突 | 冲突 | 冲突
排他意向锁(IX) | 冲突 | 不冲突 | 冲突 | 不冲突
共享锁(S) | 冲突 | 冲突 | 不冲突 | 不冲突
共享意向锁(IS) | 冲突 | 不冲突 | 不冲突 | 不冲突

#### 记录锁(Record Locks)
记录锁是一种加再索引上的锁，当我们想要修改、删除某一行时，会再对应的索引上加锁（注意，[即使没有显示的定义索引，InnoDB也会默认的为表加上一个隐藏的主键索引](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)）。
例如下面这张表`test`：

id  |  name  |  age
--- | --- | ---
1 | Tom | 10
2 | Jerry | 20

id是主键，age上有索引。
`SELECT name FROM test WHERE age=10 FOR UPDATE`会在id=1这一行上加上记录锁（注意：前面这段描述不够准确，下面会提到）。因为锁是加再对应行的索引上的，如果行不存在，是不会枷锁的，比如上面的sql，查询条件改为`age=15`是不会加锁的。

#### 范围锁(Gap Locks)
范围锁也是一种加在索引上的锁，但是和记录锁不一样的地方是，它锁的是一个范围。
还是上面的表，`SELECT * FROM test WHERE age<10 FOR UPDATE`会加一个范围锁从而阻止其它事务向test表中插入age<10的记录。

#### 临键锁(Next Key Locks)
临键锁是一种记录锁和范围锁的合体，它既锁了记录，又锁了范围。前面提到这条SQL`SELECT name FROM test WHERE age=10 FOR UPDATE`会在id=1上加上记录锁不够准确，是因为在不同的事务隔离级别下加的锁是不同的，在`READ COMMITTED`级别下，加的是记录锁，而在`REPEATABLE READ`级别下，加的是临键锁，MySQL默认的事务隔离级别是`REPEATABLE READ`。上面的SQl，在`REPEATABLE READ`级别下，会同时锁住`(-INF, 10)`和`(10, 20)`以阻止其它事务插入age在该范围的记录。
当记录锁是加在唯一索引上时，不会产生临检索，比如上面的例子，如果age上时唯一索引时，不会产生临键锁。

#### 插入意向锁(Insert Intention Locks)
当事务向某一个范围里插入一行记录时，不会阻塞其它事务往该范围内插入记录，前提是插入的记录不冲突。如果事务向表test插入age=15的记录，不会阻塞另一个事务插入age=16的记录，单位阻塞其它事务插入age=15的记录。

#### MVCC(Multiversion concurrency control)
SELECT语句默认时不加锁的，InnoDB是如何实现重复读的呢？答案是MVCC，MVCC简单点说就是为每一行维护了一个版本号，当前事务只能读到版本号在当前事务之前的记录。

#### 总结
1. InnoDB支持多种类型的锁（记录锁、范围锁、临键锁等）
2. 不同事务级别锁的类型不一样
3. InnoDB通过MVCC实现重复读

#### 备注
文中例子在`10.3.14-MariaDB`版本中验证，该版本可以简单理解为和`MySQL5.7`相互兼容。

#### 参考资料
1. [官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
