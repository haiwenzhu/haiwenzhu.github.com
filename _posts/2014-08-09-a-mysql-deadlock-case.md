---
layout: post
title: "MySQL死锁案例分享"
comments: true
categories:
  - PHP
  - MySQL
---

以前觉得INNODB的row lock机制很难碰到死锁，开启事务就更不用担心，之前的确没有碰到过死锁的问题。不过最近在项目中却碰到了死锁的问题。

问题出现的场景简化如下：

DB的表结构：

	CREATE TABLE `test` (
  		`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  		`name` char(50) NOT NULL DEFAULT '',
  		PRIMARY KEY (`id`)，
		UNIQUE KEY `name` (`name`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8

业务逻辑简化为：

	mysqli_begin_transation()
	for () {
		mysqli_query("REPLACE INTO test (name) values ('xxx')");
	}
	mysqli_commit();
其实就是会循环往test表里插入数据。针对这样的场景做了如下的测试
script1.php

	<?php
	$db = new mysqli('localhost', 'root', '', 'test');
	$db->begin_transaction();
	$db->query("REPLACE INTO test (name) value ('a')");

	sleep(10);

	$db->query("REPLACE INTO test (name) value ('a')");
	if (!empty($db->error)) {
	    echo $db->error . PHP_EOL;
	}

	$db->commit();

script2.php

	<?php
	$db = new mysqli('localhost', 'root', '', 'test');
	$db->begin_transaction();
	$db->query("REPLACE INTO test (name) value ('b')");

	if (!empty($db->error)) {
	    echo $db->error . PHP_EOL;
	}

	$db->commit();

script2.php紧接着script1.php运行，script2.php会报错：Deadlock found when trying to get lock; try restarting transaction。`show engine innodb status`看到的确发生了死锁，死锁的原因是因为script1的第一条语句的锁和script2中的replace语句发生了锁冲突，script1的第二条语句又再次申请锁时发生了死锁。
一开始很难理解为什么script1中的第二条语句还会再次申请锁，而且锁的类型和前一条语句锁的类型不一直，于是google了一下，发现MySQL对replace的实现其实是delete+insert。让我们先忘掉replace看另外一个例子：
script3.php

	<?php
	$db = new mysqli('localhost', 'root', '', 'test');
	$db->begin_transaction();
	
	$db->query("UPDATE test SET name='a' WHERE name = 'a'");
	sleep(5);
	$db->query("INSERT IGNORE INTO test (name) value ('a')");
	
	if (!empty($db->error)) {
	    echo $db->error;
	}
	
	$db->commit();

script4.php

	<?php
	$db = new mysqli('localhost', 'root', '', 'test');
	$db->begin_transaction();
	
	$db->query("UPDATE test SET name='a' WHERE name = 'a'");
	
	if (!empty($db->error)) {
	    echo $db->error;
	}
	
	$db->commit();

scirpt3.php和script4.php先后执行也会出现死锁，产生死锁的原因为：


1. script3中的update语句会获得一个排他的锁
2. script4中的update语句想要获得一个排它锁，这个加锁请求会被排队
3. script3中的insert ignore语句想要获得一个共享锁，这个锁请求也会被排队，死锁出现

script3中的insert会加共享锁是因为当出现重复键的时候，insert会加一个共享锁到相应的记录。如果把name字段上的unique key去掉变不会出现死锁。

个人理解产生死锁的根本原因是因为update和insert加锁的类型是不一样的，update和delete会加 exclusive next-key lock，而insert加的是index-record lock。第二个问题很久很久之前有人曾提过[bug](http://bugs.mysql.com/bug.php?id=1866)，不过对bug的解释不是很明白。

回到第一个问题，如果把replace into改成insert on dumplicate key update,死锁也会消失。很多人给出的建议也似尽量避免使用replace into。

关于这个问题其实我解释的也不是很清楚，有人如果能更清楚的解释这个问题欢迎留言交流。

关于MySQL锁机制可以看[这里](http://dev.mysql.com/doc/refman/5.5/en/innodb-record-level-locks.html)和[这里](http://dev.mysql.com/doc/refman/5.5/en/innodb-locks-set.html)
