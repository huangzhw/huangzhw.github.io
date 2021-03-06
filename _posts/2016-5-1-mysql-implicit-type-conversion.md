---
layout: post
title: MySQL隐式类型转换
categories: MySQL
description: 介绍MySQL类型转换相关的坑
keywords: MySQL type conversion index
---

## 问题
```
mysql> show create table users;
+-------+--------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                         |
+-------+--------------------------------------------------------------------------------------------------------------------------------------+
| users | CREATE TABLE `users` (
  `user` varchar(11) DEFAULT NULL,
  `password` varchar(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+--------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from users;
+----------+----------+
| user     | password |
+----------+----------+
| everyday | hello1   |
| hello    | 1hello   |
+----------+----------+
2 rows in set (0.00 sec)

mysql> select * from users where user = 'everyday' and password = 0;
+----------+----------+
| user     | password |
+----------+----------+
| everyday | hello1   |
+----------+----------+
1 row in set, 1 warning (0.01 sec)
```
## 原因
```
mysql> show warnings;
+---------+------+--------------------------------------------+
| Level   | Code | Message                                    |
+---------+------+--------------------------------------------+
| Warning | 1292 | Truncated incorrect DOUBLE value: 'hello1' |
+---------+------+--------------------------------------------+
1 row in set (0.00 sec)
```
意思好像是说‘hello1’是被截断的双精度浮点型。不是很能理解，去官方文档查了一下，发现是由于隐式类型转换引起的。在查询中字符串类型和整型做了比较，他们都被转换为浮点型，‘hello1’被转换为0,所以比较的结果就是true。在字符串转换时，字符串前导都是数字将会进行截取，如果不是转换为0。

## 隐式类型转换规则
在官方文档里查到比较遵从以下规则：

* 两个参数至少有一个是NULL时，比较的结果也是NULL，但是使用<=>对两个NULL做比较时会返回true，这两种情况都不需要做类型转换;
* 两个参数都是字符串，会按照字符串来比较，不做类型转换;
* 两个参数都是整数，按照整数来比较，不做类型转换;
* 十六进制的值和非数字做比较时，会被当做二进制串;
* 有一个参数是TIMESTAMP或DATETIME，并且另外一个参数是常量，常量会被转换为TIMESTAMP;
* 有一个参数是DECIMAL类型，则比较依赖于另外一个参数，如果另外一个参数是DECIMAL或者INT，会以DECIMAL进行比较，如果另外一个参数是浮点数，则会以浮点数进行比较;
* 所有其它情况下，两个参数都会被转换为浮点数再进行比较。

### 日期相关类型的转换
由于日期的特殊性，所以有必要看一下日期的转换方式

```
mysql> select curtime(), curtime() + 0;
+-----------+---------------+
| curtime() | curtime() + 0 |
+-----------+---------------+
| 16:45:48  |        164548 |
+-----------+---------------+
1 row in set (0.00 sec)

mysql> select curdate(), curdate() + 0;
+------------+---------------+
| curdate()  | curdate() + 0 |
+------------+---------------+
| 2016-05-01 |      20160501 |
+------------+---------------+
1 row in set (0.00 sec)

mysql> select now(), now() + 0;
+---------------------+----------------+
| now()               | now() + 0      |
+---------------------+----------------+
| 2016-05-01 16:46:29 | 20160501164629 |
+---------------------+----------------+
1 row in set (0.00 sec)
```

日期转换为字符串对大家来说是比较直观的，其实平时我们在控制台写查询，都是默认用到了字符串转换为日期的。但是日期转换为数字，是直接取出了里面的数字。
所以在我们在写查询时，`select * from tablename where add_time = '2015-01-01 12:34:56'`或者`select * from tablename where add_time = 20150101123456`都能匹配同样的日期。

### DECIMAL FLOAT DOUBLE区别
在使用MySQL的过程中，并没有对这三个数据类型的区别有深入的了解，这里总结一下。

* DECIMAL[(M[,D])]

DECIMAL是一个准确的定点数。M是总的数字位数，N是小数点后的位数。M最大是65，D最大是30。M被省略就是10，D被省略就是0。

* FLOAT[(M,D)]

单精度浮点型，范围在 -3.402823466E+38 to -1.175494351E-38, 0, and 1.175494351E-38 to 3.402823466E+38，注意这是IEEE的理论标准，根据平台的实现不同有可能不同。

* DOUBLE[(M, D)]

双精度浮点型，范围在-1.7976931348623157E+308 to -2.2250738585072014E-308, 0, and 2.2250738585072014E-308 to 1.7976931348623157E+308，注意这是IEEE的理论标准，根据平台的实现不同有可能不同。

MySQL里还有可能使用FLOAT(p)，p是精度，如果p在0和24之间就使用单精度浮点型，如果在25和53之间就使用双精度浮点型。

## 隐式类型转换对性能的影响

```
mysql> show create table indextest;
+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table     | Create Table                                                                                                                                                                                                                                                                                                                                                                                      |
+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| indextest | CREATE TABLE `indextest` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(10) DEFAULT NULL,
  `age` tinyint(3) unsigned NOT NULL DEFAULT '0',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`),
  KEY `idx_age` (`age`),
  KEY `idx_create` (`create_time`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 |
+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from indextest;
+----+-------+-----+---------------------+
| id | name  | age | create_time         |
+----+-------+-----+---------------------+
|  1 | hello |  10 | 2016-05-01 17:09:42 |
|  2 | world |  20 | 2016-05-01 17:09:42 |
|  3 | 123   |  30 | 2016-05-01 17:09:42 |
+----+-------+-----+---------------------+
3 rows in set (0.00 sec)

mysql> explain select * from indextest where name = '123';
+----+-------------+-----------+------+---------------+----------+---------+-------+------+-----------------------+
| id | select_type | table     | type | possible_keys | key      | key_len | ref   | rows | Extra                 |
+----+-------------+-----------+------+---------------+----------+---------+-------+------+-----------------------+
|  1 | SIMPLE      | indextest | ref  | idx_name      | idx_name | 33      | const |    1 | Using index condition |
+----+-------------+-----------+------+---------------+----------+---------+-------+------+-----------------------+
1 row in set (0.00 sec)

mysql> explain select * from indextest where name = 123;
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | indextest | ALL  | idx_name      | NULL | NULL    | NULL |    3 | Using where |
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)
```


可以看到用字符串和name列做比较的时候使用了索引，而用数字和name做比较的时候用了全表扫描。原因是有很多不同的字符串可以转换为相同的数字，如'123a', '123'等都可以转换为123，所以这样就没法使用索引加快查询了。

## 总结

* MySQL隐式类型转换规则比较复杂，依赖MySQL隐式转换很容易出现各种想想不到的问题;
* MySQL隐式类型转换本身有可能非常耗费 MySQL服务器性能的;
* 不要依赖于隐式类型转换，尤其是规则比较复杂，有可能产生意想不到的行为;
* 在MySQL里可以用CAST强制类型转换，减少出错概率。
