---
title: MySQL锁机制
date: 2017-06-10 17:55:11
tags: 数据库
categories: 数据库

---
本篇将介绍MySQL锁机制的特点，主要拿MyISAM，InnoDB引擎进行分析。

# MyISAM表锁
MyISAM引擎只支持表锁，这种锁的特点是加锁快，开销小，锁发生冲突的概率比较高，并行度较低。

## 锁模式
表级锁有两种模式：共享读锁和独占写锁，类似Java中的ReadWriteLock。多个线程可以共享一把读锁，但是会堵塞写线程的请求。当写锁被一个线程占有时，其他线程的读和写请求都会被堵塞。
## 如何加表锁
### 自动加表锁
MyISAM存储引擎在执行SELECT, UPDATE, INSERT, DELETE之前，会自动获取相应的所有的表锁，执行完毕之后，自动释放所有的表锁，这个过程不需要显式指定，用起来比较方便。
### 手动加锁
手动加锁一般是为了模拟事务操作，要注意以下几点。

1. 在执行Lock Tables之后，只能访问加锁的表，自动更新也是如此。
2. 获取读锁之后不能进行更新和插入操作。
3. 同一个表在SQL中出现多少次，就需要对相同的别名锁定多少次。

## 并发插入
MyISAM允许读写操作并发进行，存在系统变量concurrent_insert，专门控制并发插入的行为，其值有0，1，2。

- 当设置为0时，不允许并发插入。
- 当设置为1时，如果表的中间没有被删除，那就允许其他用户在表的末尾并发插入。
- 当设置为2时，无论表是否存在空洞，都可以在表的末尾进行插入。

## 锁调度
MyISAM默认给予写操作更高的优先级，这意味着如果一个读线程和一个写线程同时分别获得读锁和写锁，那写线程将会优先进行。即便读请求先到锁的等待队列，之后到达的写请求也会“插队”。要想降低写线程的优先级，可以采取以下办法：

- 制定启动参数low-priority-updates，使存储引擎给写线程以更低的优先级。
- 设置参数LOW_PRIORITY_UPDATES = 1。
- 通过制定INSERT, UPDATE, DELETE语句的LOW_PRIORITY属性，降低语句的优先级。

# InnoDB锁
InnoDB引擎支持事务，因此请先查看[数据库事务](http://zsrr.coding.me/2017/05/15/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1/)。

InnoDB的锁类型这里不再赘述，有关内容请查看：[InnoDB锁模式](http://www.zrray.com/art/247)

在这里主要记录以下几点：

## 共享锁(S)
通过**SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE**获得共享锁，主要是确保一个事务在读取这些记录的时候，另一个事务对这些数据集的修改需等到前一个事务提交或回滚后进行。

## 排他锁(X)
通过**SELECT * FROM table_name WHERE ... FOR UPDATE**来获取。一个事务对一个数据集进行加锁之后，另一个事务对这个数据集的更新，删除，以及试图获取其共享锁的操作都会被阻塞，但是普通的SELECT操作不会被阻塞（返回的只是数据集的快照）。

## InnoDB行锁的实现方式
主要有三种实现方式：

- **Record Lock** 对索引项进行加锁。
- **Gap Lock** 对索引项的“间隙”，第一条记录前的“间隙”或者最后一条记录后的“间隙”进行加锁。
- **Next-key Lock** 前两种的组合。

InnoDB这种行锁实现方式意味着，如果不通过索引检索数据，那么InnoDB将对所有记录进行加锁，实际上的效果和表锁一样。

## 行锁的注意事项
### 不通过索引检索数据，InnoDB会锁定表中的所有记录
### InnoDB行锁针对索引加锁，不是根据记录加的锁
访问不同行的记录，但是使用的是相同的索引键，会发生锁竞争，如下例所示：

	create table tab_with_index( id int not null, name varchar(30));
	create index index_id on tab_with_index(id);
	
	insert into tab_with_index values(1, '1'), (1, '2');
	
	...
	
	# session1
	select * from tab_with_index where id = 1 and name = '1' for update;
	
	...
	
	# session2
	select * from tab_with_index where id = 1 and name = '2' for update;
	# 等待
在上例中，由于id为1（id为索引）的记录在事务1中被加锁，因此尽管事务2访问的是不同行，也会被阻塞。
### 不同的事务可以根据不同的索引获得不同的锁
如下例所示：

	create index name on tab_with_index(name);
	
	insert into tab_with_index values(2, '4');
	
	# session1
	select * from tab_with_index where id = 1 for update;
	
	# session2
	select * from tab_with_index where name = '4' for update;
	# 立即返回数据，不会被阻塞。
	
	select * from tab_with_index where name = '2' for update;
	# 被阻塞，事务1对此数据集加了排他锁。
### Next-key “间隙”加锁
“间隙”(Gap)的定义为：键值在条件范围内但是并不存在的记录，用在对键值进行范围条件查找时。

举个例子，假如test表中只有100条记录，其id值为1，2，3...100。这时在一个事务中执行如下语句：

	select * from test where id >= 100 for update;
这时InnoDB不仅对id为100的实际存在的记录进行加锁，而且对id大于100但是实际上不存在的记录进行加锁。