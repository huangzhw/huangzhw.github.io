---
layout: post
title: 一个MySQL混用utf8和utf8mb4产生的问题
categories: MySQL
description: 介绍MySQL混用utf8和utf8mb4产生的坑
keywords: MySQL utf8 utf8mb4 type conversion
---

### 前言

去年5月份我们将线上的MySQL升级到了5.7，所有的编码都采用了`utf8mb4`，旧的表依然使用了之前的`utf8`，但是新建的表都用了`utf8mb`。

因为`utf8mb4`是`utf8`的超集，`utf8`可以无损的转换为`utf8mb4`，客户端只需要设置成用`utf8mb4`连接服务端，便可完美兼容。

然而一次在线上执行查询的时候，发生了问题，一个应该很快结束的查询却花了很久。通过分析发现，就是`utf8`和`utf8mb4`混用造成的。

### 问题

执行以下查询，发现执行时间特别长。表`A`是一个小表，里面就几千条记录，`B`是一个大表，里面有几千万条记录。两个表`mobile`上都有唯一索引。

```
SELECT date(A.add_time) AS day, count(*) FROM A JOIN B on A.mobile = B.mobile GROUP BY day;
```

按照MySQL的执行计划来说，应该把表`A`作为外表，表`B`作为内表来连接，这样只要做几千次的索引查询，便可以得出结果。

表定义如下:

```
CREATE TABLE `A` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL,
  `mobile` varchar(45) DEFAULT NULL,
  `add_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `mobile` (`mobile`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `B` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `mobile` varchar(45) DEFAULT NULL,
  `register_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_mobile` (`mobile`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

系统的编码设置如下:

```
mysql> show variables like 'character_set_%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8mb4                    |
| character_set_connection | utf8mb4                    |
| character_set_database   | utf8mb4                    |
| character_set_filesystem | utf8mb4                    |
| character_set_results    | utf8mb4                    |
| character_set_server     | utf8mb4                    |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
```

### 分析

先使用`EXPLAIN`查看执行计划

```
mysql> EXPLAIN SELECT date(A.add_time) AS day, count(*) from A join B ON A.mobile = B.mobile GROUP BY day;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+----------+----------+----------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows     | filtered | Extra                                        |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+----------+----------+----------------------------------------------+
|  1 | SIMPLE      | B     | NULL       | index | NULL          | uk_mobile | 138     | NULL | 87654321 |   100.00 | Using index; Using temporary; Using filesort |
|  1 | SIMPLE      | A     | NULL       | ref   | mobile        | mobile    | 67      | func |        1 |   100.00 | Using index condition                        |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+----------+----------+----------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

发现竟然是把表`B`作为外循环，来连接表`A`，虽然查找表`A`的时候使用了索引，这样执行几千万次的索引查找，速度当然会很慢。照理说最优的情况下是把`A`作为外循环，来连接表`B`，这样只需要几千次的索引查询，速度会快很多。

原本以为是MySQL搜集的优化信息不够，所以给出了错误的执行计划，所以给一点提示，用`STRAIGHT_JOIN`强制表的连接顺序。

```
mysql> EXPLAIN SELECT date(A.add_time) AS day, count(*) FROM A STRAIGHT_JOIN B ON a.mobile = b.mobile  GROUP BY day;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+----------+----------+-----------------------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows     | filtered | Extra                                                           |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+----------+----------+-----------------------------------------------------------------+
|  1 | SIMPLE      | A     | NULL       | ALL   | mobile        | NULL      | NULL    | NULL |     9490 |   100.00 | Using temporary; Using filesort                                 |
|  1 | SIMPLE      | B     | NULL       | index | NULL          | uk_mobile | 138     | NULL | 87654321 |   100.00 | Using where; Using index; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+----------+----------+-----------------------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

发现虽然现在表按照指定的顺序连接了，但是在查找内层的表`B`时，使用的是索引扫描，就是要扫描`9490`次千万级的索引，这非常慢(实际上使用了`join buffer`减少了扫描的次数)。

所以为什么没有试用表`B`上的索引呢，我陷入了沉思，觉得很奇怪。

`EXPLAIN`可以通过`SHOW WARNINGS`查看语句被优化器如何转换。

对第一条语句进行`SHOW WARNINGS`查看

```
mysql> SHOW WARNINGS;
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                          |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | /* select#1 */ select cast(`test`.`A`.`add_time` as date) AS `day`,count(0) AS `count(*)` from `test`.`A` join `test`.`B` where (`test`.`A`.`mobile` = convert(`test`.`B`.`mobile` using utf8mb4)) group by `day` |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

在表连接的时候先把表`B`的`mobile`字段转换为`utf8mb4`然后再做连接的比较，所以就没法使用`B`的索引了。

所以最终结论是当`utf8`的列和`utf8mb4`的列进行比较时，需要都转换兼容的列，而`utf8`转换为`utf8mb4`是无损的，所以都转换为`utf8mb4`进行比较，那么`utf8`那一列的索引就没法使用了。所以第一个查询时，会将表`B`作为外表，因为表`B`的索引无法使用。

### 解决方法

#### 一劳永逸的方法

将所有的表都转换为统一的`utf8mb4`。这样就可以使用索引了。

但是转换工作量比较大，可能需要停服操作。但是这样转换后，以后就可以避免再次踩到这样的坑。


#### 转换回utf8

如果我们可以确定表`A`里面实际没有`4`字节的字符，也就是兼容`utf8`的情况，可以执行

```
mysql> ALTER TABLE A CONVERT TO CHARACTER SET utf8;
Query OK, 9522 rows affected (0.29 sec)
Records: 9522  Duplicates: 0  Warnings: 0
mysql> EXPLAIN SELECT date(A.add_time) AS day, count(*) from A join B ON A.mobile = B.mobile GROUP BY day;
+----+-------------+-------+------------+------+---------------+-----------+---------+------------------+------+----------+----------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref              | rows | filtered | Extra                                        |
+----+-------------+-------+------------+------+---------------+-----------+---------+------------------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | a     | NULL       | ALL  | mobile        | NULL      | NULL    | NULL             | 9490 |   100.00 | Using where; Using temporary; Using filesort |
|  1 | SIMPLE      | b     | NULL       | ref  | uk_mobile     | uk_mobile | 138     | uu_mall.a.mobile |    1 |   100.00 | Using where; Using index                     |
+----+-------------+-------+------------+------+---------------+-----------+---------+------------------+------+----------+----------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

#### 重写查询

如果我们可以确定表`A`里面实际没有`4`字节的字符，可以重写查询，将表`A`的`utf8mb4`转换为`utf8`。

```
mysql> EXPLAIN SELECT date(A.add_time) AS day, count(*) FROM A join B ON CONVERT(A.mobile using utf8) = B.mobile GROUP BY day;
+----+-------------+-------+------------+------+---------------+-----------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+------+---------------+-----------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | A     | NULL       | ALL  | NULL          | NULL      | NULL    | NULL | 9490 |   100.00 | Using temporary; Using filesort |
|  1 | SIMPLE      | B     | NULL       | ref  | uk_mobile     | uk_mobile | 138     | func |    1 |   100.00 | Using where; Using index        |
+----+-------------+-------+------------+------+---------------+-----------+---------+------+------+----------+---------------------------------+
```

这样转换后，表`A`的索引无法使用了，但是表`B`的索引就可以使用了，这时候表`A`就会作为外循环。

### 小结

* `uf8mb4`是`utf8`的超集这一事实，让数据库中两种编码混用在大多数情况下都没有问题;
* 但是`uf8mb4`和`utf8`需要比较的时候，就会进行隐式类型转换，这时候就可能出现索引无法使用的情况;
* 最好的方式还是同一个数据库就用一种编码，就肯定不会cai
