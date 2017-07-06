---
title: SQL优化
date: 2017-06-09 14:41:57
tags: 数据库
categories: 数据库

---
本篇详细记录SQL优化的一般步骤，以MySQL为例。

# 通过EXPLAIN语句分析SQL
EXPLAIN通常用来分析select子句，返回的数据格式为：

|列名|含义|
|---|----|
|id|select标识符|
|select_type|select类型|
|type|访问类型|
|possible_keys|查询设计到的索引|
|key|实际使用的索引|
|key_length|使用的索引的长度|
|ref|哪些列或常量用于匹配索引的值|
|extra|关于查询的详细信息|

## select_type
此列记录了select的类型，主要有以下几种：

- **SIMPLE** 简单查询，不使用**UNION**和子查询。
- **PRIMARY** 若查询中包括子查询，则此项说明是最外边的查询。
- **UNION** UNION中第二个SELECT语句。
- **SUBQUERY** 子查询中第一个SELECT。
- **DERIVED** FROM子句中的子查询。

## type
访问类型，性能从好到坏排序可以分成以下几个部分：

- **system** 表中只有一行。
- **const** 单表中至多有一个数据项可以匹配，用在PRIMARY KEY和UNIQUE INDEX和常量进行"="比较的情况。
- **eq_ref** 在多表连接当中，使用PRIMARY KEY或者UNIQUE INDEX进行比较。
- **ref** 与eq_ref类似，但是不使用PRIMARY KEY和UNIQUE INDEX。
- **range** 根据索引进行范围内搜索。
- **index** 遍历索引就能获得相应的数据，通常比遍历整个表要快，但是也不一定，当数据量较小的时候MySQL可能仍会选择遍历整个表。
- **all** 遍历整个表，通常比较慢，当数据量较大的时候需要优化。

## ref
ref是记录了哪些列或者常量被用来和索引进行比较，如果值为func的话，那就是从某个函数返回的值进行比较。

# show profile
可以通过**show profile**语句来查看一个query语句消耗了多长时间。要想查询特定的查询语句，需要先执行**show profiles**（先将profiling变量设置为1），执行的结果如图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-09%20%E4%B8%8B%E5%8D%8810.49.40.png)
上图列出了之前查询语句的id，耗时和语句详情，要想查看一个具体的查询语句的耗时情况，执行show profile for query [*query_id*]:

	show profile for query 7;
执行结果如图所示：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-06-09%20%E4%B8%8B%E5%8D%8811.01.04.png)

**注：**上图中sending data过程是MySQL从表中读取数据并把结果返回给客户的过程，往往是整个过程最耗时的操作。
# 索引
## 使用索引的情况
### 匹配全值
即对where子句中所有的索引列指定一个具体值。
### 匹配值的范围查询
对索引的值能够进行范围查找（Hash索引不支持）。
### 匹配最左前缀
仅仅使用复合索引的最左边列进行查找，如有col1 + col2 + col3的复合索引，可以针对col1, col1 + col2, col1 + col2 + col3, col1 + col3进行查找，但是不能对col2, col3, col2 + col3进行查找，这是B-Tree索引使用时的首要准则。
### 仅仅对索引进行查询
当要查询的列都是索引的时候，MySQL会在索引中直接查找，因为索引的大小通常比原表小得多，所以查询的速度要比原表要快，此时Extra信息中会提示：Using index。
### 匹配列前缀
仅仅匹配索引第一列的部分，前缀索引不适合**order by group by**子句
### ICP特性
MySQL 5.6引入了Index Condition Pushdown特性，将操作过滤（下放）到存储引擎。

如有一复合索引col1 + col2，查询条件为：col1 = ... and col2 > ...，那么在MySQL 5.6之前的版本，数据库先根据复合索引的首字段col1来过滤出满足第一个条件的记录，然后回表查找记录（此时为Using where），最后根据第二个条件过滤之前查找的记录。MySQL 5.6之后，则把col2的过滤下放到底层存储引擎层来完成，这样能减少不必要的io操作，提高性能。
## 存在索引但是不能使用索引的情况
### 以%开头的like查询不能够使用B-Tree索引
由于B-Tree索引的结构，以%开头的like查询不能够使用B-Tree索引，一般采用[全文索引](http://www.cnblogs.com/tommy-huang/p/4483684.html)来解决此类问题。
### 数据出现隐式转换的时候不能使用索引
例如：

	select * from actor where name = 1;
name为varchar类型，但是给予的比较常量是一个int类型，这时候即便name是一个索引，查询仍不能使用此索引。
### 复合索引不满足最左原则的情况下不能够被使用
### 如果查找索引比查找表更慢，那么索引不会被使用
在数据量较小或者索引比表大的情况下（极少出现），扫描索引会比直接扫描表更慢，这时索引不会被使用。
### 用or分隔开的条件，前面有索引而后面没有，索引不被使用
# 常用的优化方法
## 定期优化表
如果表中绝大部分数据已经被删除，并且包含可变长度的列，那么可以通过OPTIMIZE TABLE [TABLE_NAME]进行优化，这个命令可以将表中空间碎片进行整合，消除由于删除更新而带来的空间浪费。
## SQL优化
### 大量导入数据
对于MyISAM的表来说，可以通过以下方式快速导入数据：

	ALTER TABLE tab_name DISABLE KEYS;
	#load data
	ALTER TABLE tab_name ENABLE KEYS;
再导入数据前先关闭非唯一索引的更新，导入完成后再打开。

对于InnoDB表来说，导入的数据按照主键进行排序，可以有效地提高数据导入的效率。

在导入数据之前，关闭唯一性检验（SET UNIQUE_CHECK = 0），导入完成后打开，可以提高数据导入的效率。
### 优化INSERT语句
当插入很多行是，应尽量使用多个值表的INSERT语句，比单个INSERT语句执行快得多。

当导入大量数据时，从文本中导入数据比INSERT语句快得多。
### 优化ORDER BY子句
#### MySQL排序方式
##### 索引顺序扫描
通过有序索引顺序扫描直接返回有效的数据，操作效率高。
##### Filesort排序
对返回数据进行排序，所有不是通过索引直接返回排序结果的排序都是Filesort排序，此时Extra信息中存在Using filesort。

所以，应尽量保证索引顺序和ORDER BY后面的顺序相同，避免Filesort排序，有关Filesort的详细信息及底层算法，参见：[MySQL排序原理与案例分析](http://www.cnblogs.com/cchust/p/5304594.html)

总结： 

下列情况可以使用索引：

	SELECT * FROM TABLE_NAME ORDER BY KET_PART_1, KEY_PART2...;
	SELECT * FROM TABLE_NAME WHERE KEY_PART_1 = XXX ORDER BY KEY_PART_2;
	SELECT * FROM TABLE_NAME WHERE KEY = XXX ORDER BY KEY;
	SELECT * FROM TABLE_NAME WHERE KEY_PART_1 = XXX ORDER BY KEY_PART_1 DESC, KEY_PART_2 DESC;
以下情况不可以：

	SELECT * FROM TABLE_NAME ORDER BY KEY_PART_1 DESC, KEY_PART_2 ASC;
	SELECT * FROM TABLE_NAME WHERE KEY1 = XXX ORDER BY KEY2;
	SELECT * FROM TABLE_NAME ORDER BY KEY1, KEY2;