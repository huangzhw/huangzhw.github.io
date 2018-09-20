---
layout: post
title: MySQL Online DDL 探究
categories: MySQL
description: 探究MySQL Online DDL 的方式
keywords: MySQL online DDL gh-ost pt-osc
---

## MySQL Online DDL 探究

### 前言

如果项目要支持7*24小时服务，对MySQL表添加列或者添加索引等，无法在停机的情况下做了，需要Online DDL的支持，也就是在不影响服务的情况下，在线修改表的定义。

从MySQL 5.6开始，就有官方的Online DDL支持。

目前，MySQL Online DDL主要有三种方式：

* Percona pt-osc , Facebook osc 这种主要是用触发器实现，是一种比较古老的Online DDL方式；
* Github推出的gh-ost，放弃了触发器的实现，采用了`Binlog` 代替触发器来做增量数据同步，这样可以降低主库的负载，异步的执行；
* MySQL 5.6以来自带的Online DDL，主要对InnoDB表的支持。

### 官方Online DDL之殇

#### 官方Online DDL简介

Online DDL支持`INPLACE`的表的修改和并行的DML。据官网文档，Online DDL有以下优点：

* 在表一段时间不可用是无法接受的情况下，在繁忙的生产环境中提高响应能力和可用性；
* 使用`LOCK`子句在做DDL时控制性能和并发性；
* 比基于`COPY`的表修改方式更小的磁盘空间使用和`IO`负载。

正常情况下，使用`ALTER`修改表时，不需要做任何事，MySQL会使用尽可能少的`LOCK`。

当然也可以显示控制使用的`ALGORITHM`和`LOCK`，如下：

```
ALTER TABLE tbl_name ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;
``` 

在这种情况下，MySQL会使用相应的`ALGORITHM`和`LOCK`。对于每一种`ALTER`都有不同的最低的`ALGORITHM`和`LOCK`需求，如果指定的`ALGORITHM`和`LOCK`不够，就会终止语句的执行。

对于`ALGORITHM`有两种方式`INPLACE`和`COPY`，在大多数情况下都只需要`INPLACE`的方式，不需要拷贝表，可以减小磁盘使用和`IO`负载。

对于`LOCK`有4个参数`NONE`、`SHARED`、`DEFAULT`和`EXCLUSIVE`:

* `NONE`不加锁，在做DDL时可以允许并行的查询和DDL，这是在线修改表定义必须的参数；
* `SHARED`可以查询，但是不允许DML；
* `DEFAULT`对不同的`ALTER`语句采用所需的最小的`LOCK`；
* `EXCLUSIVE`阻塞查询和DML。

具体不同的`ALTER`操作需要的`LOCK`，可以参考文档：
[https://dev.mysql.com/doc/refman/5.7/en/innodb-create-index-overview.html#online-ddl-column-operations](https://dev.mysql.com/doc/refman/5.7/en/innodb-create-index-overview.html#online-ddl-column-operations)

对于大多数操作，包括添加索引和添加列等，这是我们日常最需要的操作，都可以使用`LOCK=NONE`来执行，所以看起来官方的ONLINE DDL很完美。

#### 在复制架构下的官方Online DDL的问题

在通读了MySQL Online DDL的文档后，我觉得这个工具非常适合我们，就使用他来加索引。

但是有一次对一个千万级别的大表加索引时，出现了用户购买成功但是没有获取到VIP的情况，但是购买成功了VIP是肯定加上了的。因为我们的架构读数据是在`Slave`读取的，所以只能是主从同步出问题了。查看了监控图标，发现`Slave`在一段时间内有大量的复制延迟。

在网上查阅了资料，发现原因如下：

1. 在`Master`上执行DDL语句时，这时候允许并行的DML操作没有什么问题；
2. 但是在`Master`的DDL语句没有执行完前，这条语句是不会同步到`Slave`的，执行完后，这条语句同步到`Slave`开始执行；
3. 在`Slave`上执行DDL时，在DDL之后的DML语句不会被执行，直到DDL执行完毕后，这些DML语句才会开始执行；
4. 然后`Slave`需要一段时间跟上`Master`。

`Slave`不能并行执行的原因是这些DML操作语句可能依赖于表的Schema的修改。

#### 实验证明

下面做一个实验证明。

用以下脚本定期往表`test`插入数据：

```
while True:
    QS(db).table(T.test).insert({'a': 1, 'b': 'b'})
    time.sleep(0.1)
```

在一个终端执行以下语句：

```
mysql> alter table cdkey2 add index(end_time), algorithm=inplace, lock=none;
```

我们在`Master`上查询：

```
mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1136 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1140 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1150 |
+----------+
1 row in set (0.00 sec)
```

再在`Slave`上查询：

```
mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1164 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1169 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1175 |
+----------+
1 row in set (0.00 sec)
```

可见数据都有插入到`Master`，并且同步到`Slave`里。

当`ALTER`语句完成的瞬间：

```
mysql> alter table cdkey2 add index(end_time), algorithm=inplace, lock=none;
Query OK, 0 rows affected (7.83 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

查询`Master`：

```
mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1258 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1263 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1269 |
+----------+
1 row in set (0.00 sec)
```

查询`Slave`：

```
mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1179 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1179 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from test;
+----------+
| count(*) |
+----------+
|     1179 |
+----------+
1 row in set (0.00 sec)
```

以上说明，在`Master`执行完成，将语句同步到`Slave`后，`Slave`后续的DML都卡主了。


### pt-osc和gh-ost的选择

在`Master-Slave`架构下，官方的Online DDL存在致命的缺陷，所以我们只能转向第三方工具。这里有Percona的公司的pt-osc，是基于触发器的Online DDL的代表。以及比较新的，Github的gh-ost，是基于Binlog的。

第三方的Online DDL，一般是以下步骤：

1. 根据原来的表结构执行 alter 语句，新建一个更新表结构之后的表，通常称为ghost表，对用户不可见；

2. 把原来表的已有数据分批次拷贝到ghost表；

3. 在拷贝的过程中，会有新的数据过来，这些数据通过同步到ghost表；

4. 拷贝和同步完成后，锁住源表，将ghost表替换为原表。


这其中比较重要的第三步，如何同步增量的数据。最开始办法就是使用触发器，在源表上增加几个触发器，例如当源表执行 `INSERT`，`UPDATE`，`DELETE` 语句，就把这些操作通过触发器同步到ghost表上，这样在ghost表上执行的语句和源表的语句就属于同一个事务，这样当主库负载大时，会严重影响性能。

后面出现了异步的模式，使用触发器把对源表的操作保存到一个 `Changelog` 表中，不真正的去执行，专门有一个后台的线程从`Changelog` 表读取数据应用到幽灵表上。这种方式一定程度上缓解了主库的压力，但是保存到 `Changelog` 表也同样是属于同一个事务中，对性能也有不小的影响。

在 gh-ost 的文档 中细数了触发器的不足之处，大致有以下几点:

* overhead：触发器是用存储过程的实现的，就无法避免存储过程本身需要的开销；
* locks：增大了同一个事务的执行步骤，更多的锁争抢；
* Trigger based migration, no pause: 整个过程无法暂停，假如发现影响主库性能，停止 Online DDL，那么下次就需要从头来过；
* multiple migrations: 他们认为多个并行的操作是不安全的；
* Trigger based migration, no reliable production test: 无法在生产环境做测试；
* Trigger based migration, bound to server: 触发器和源操作还是在同一个事务空间。

gh-ost放弃了触发器，采用`Binlog`同步。

gh-ost 作为一个伪装的备库，可以从主库/备库上拉取 `Binlog`，过滤之后重新应用到主库上去，相当于主库上的增量操作通过 `Binlog` 又应用回主库本身，不过是应用在ghost表上。

<img src="/images/posts/mysql/gh_osc.png" />

gh-ost 的执行步骤如下：

* 在 Master 中创建镜像表`_tablename_gho`和心跳表`_tablename_ghc`;
* 向心跳表中写入 Online DDL 的进度以及时间；
* 在镜像表上执行 `ALTER`操作；
* 伪装成 `Slave` 连接到 `Master` 的 `Slave` 上获取 `Binlog` 的信息（默认设置，也可以连 `Master`）；
* 在 `Master` 中完成镜像表的数据同步；
* 从源表中拷贝数据到镜像表；
* 依据 `Binlog` 信息完成增量数据的变更；
* 在源表上加锁；
* 确认心跳表中的时间，确保数据是完全同步的；
* 用镜像表替换源表；
* Online DDL 完成。

gh-ost有以下好处：
* 整个流程异步执行，对于源表的增量数据操作没有额外的开销，高峰期变更业务对性能影响小；
* 降低写压力，触发器操作都在一个事务内，gh-ost 应用 `Binlog` 是另外一个连接在做；
* 可停止，`Binlog`有位点记录，如果变更过程发现主库性能受影响，可以立刻停止拉`Binlog`，停止应用 `Binlog`，稳定之后继续应用；
* 可测试，gh-ost 提供了测试功能，可以连接到一个备库上直接做 Online DDL，在备库上观察变更结果是否正确，再对主库操作，心里更有底；
* 并行操作，对于 gh-ost 来说就是多个对主库的连接。

### 使用gh-ost

#### 权限

主要是在我们要改变的数据库上有一个用户可以修改表定义，并有复制的权限，需要有以下权限

```
REPLICATION CLIENT, REPLICATION SLAVE on *.* and ALL on `test`.*
```

#### 限制

gh-ost目前有以下限制：

* `Binlog`格式必须使用 `row`，且 `image` 必须是 `FULL`；
* 不支持外键，不论源表是主表还是子表，都无法使用，也就是不管是这表上有外键，或者其它表有外键参照这表都不行；
* 不支持触发器；
* 不支持包含 `JSON` 列的主键；
* 迁移表需要有显示定义的主键，或者有非空的唯一索引；
* 迁移工具不区分大小写英文字母，如果存在同名，但是大小写不同的表则无法迁移；
* 迁移表的主键或者非空唯一索引包含枚举类型时，迁移效率会大幅度降低。

#### 安装
可以在[https://github.com/github/gh-ost/releases/tag/v1.0.46](https://github.com/github/gh-ost/releases/tag/v1.0.46)下载。

对于debian只需执行`dpkg -i gh-ost_1.0.46_amd64.deb`即可。

#### 执行

在双主环境下，如下执行便可：

```
gh-ost --alter "add index (add_time)" --database="test" --table="test" --allow-master-master --allow-on-master --host="10.20.9.6"  --user="rpl" --password="hellworld" --assume-rbr --execute
```

基本选项：

* `--execute` 测试使用，当不加这个选项时，会检查是否可以执行，但是实际上不会执行；
* `--alter` 需要执行的`ALTER`操作，仅需部分`ALTER`语句；
* `--database`  指定数据库；
* `--table` 指定操作的表；
* `--host` 指定`Master`的`host`；
* `--user` 指定连接的用户；
* `--password` 指定用户的密码。

高级选项：

* `--assume-rbr` 如果用户没有`Super`权限的话，需要加上这个参数，这样gh-ost会认为`Binlog`本身就是`row`模式，不会再去修改；
* `--allow-master-master`对于主主架构需要加上这个选项；
* `--allow-on-master` 允许直接在`Master`库上使用，有些架构可能不支持在`Slave`上获取`binlog`，需要指定这个选项；
* `--chunk-size` 每次循环处理的数据行数 (allowed range: 100-100,000) (default 1000)

以下是我在线上执行为一个表添加字段的结果

```
(virtualenv) hzw@stat01-dc:~/script/bin$ gh-ost --alter "add column begin_time datetime, add column end_time datetime, add column add_time datetime" --database="test" --table="test" --allow-master-master --allow-on-master --host="10.19.2.12"  --user="rpl" --password="password" --assume-rbr --execute
2018/08/28 09:42:28 binlogsyncer.go:79: [info] create BinlogSyncer with config {99999 mysql 10.19.2.12 3306 rpl   false false <nil>}
2018/08/28 09:42:28 binlogsyncer.go:246: [info] begin to sync binlog from position (binlog.000557, 155595533)
2018/08/28 09:42:28 binlogsyncer.go:139: [info] register slave for master server 10.19.2.12:3306
2018/08/28 09:42:28 binlogsyncer.go:573: [info] rotate to (binlog.000557, 155595533)
# Migrating `test`.`test`; Ghost table is `test`.`_test_gho`
# Migrating mysql01-1234:3306; inspecting mysql01-1234:3306; executing on stat01-dc
# Migration started at Tue Aug 28 09:42:28 +0800 2018
# chunk-size: 1000; max-lag-millis: 1500ms; dml-batch-size: 10; max-load: ; critical-load: ; nice-ratio: 0.000000
# throttle-additional-flag-file: /tmp/gh-ost.throttle
# Serving on unix socket: /tmp/gh-ost.test.test.sock
......
Copy: 15451000/15555551 99.3%; Applied: 10; Backlog: 0/1000; Time: 5m59s(total), 5m59s(copy); streamer: binlog.000557:636908451; State: migrating; ETA: 2s
Copy: 15483000/15555551 99.5%; Applied: 10; Backlog: 0/1000; Time: 6m0s(total), 6m0s(copy); streamer: binlog.000557:638055904; State: migrating; ETA: 1s
Copy: 15513000/15555551 99.7%; Applied: 10; Backlog: 0/1000; Time: 6m1s(total), 6m1s(copy); streamer: binlog.000557:639028705; State: migrating; ETA: 0s
Copy: 15554000/15555551 100.0%; Applied: 10; Backlog: 0/1000; Time: 6m2s(total), 6m2s(copy); streamer: binlog.000557:640386863; State: migrating; ETA: 0s
Copy: 16375325/16375325 100.0%; Applied: 10; Backlog: 0/1000; Time: 6m21s(total), 6m21s(copy); streamer: binlog.000557:667803496; State: migrating; ETA: due
# Migrating `test`.`test`; Ghost table is `test`.`_test_gho`
# Migrating mysql01-1234:3306; inspecting mysql01-1234:3306; executing on stat01-dc
# Migration started at Tue Aug 28 09:42:28 +0800 2018
# chunk-size: 1000; max-lag-millis: 1500ms; dml-batch-size: 10; max-load: ; critical-load: ; nice-ratio: 0.000000
# throttle-additional-flag-file: /tmp/gh-ost.throttle
# Serving on unix socket: /tmp/gh-ost.test.test.sock
Copy: 16375325/16375325 100.0%; Applied: 10; Backlog: 0/1000; Time: 6m22s(total), 6m21s(copy); streamer: binlog.000557:667816599; State: migrating; ETA: due
2018/08/28 09:48:51 binlogsyncer.go:107: [info] syncer is closing...
2018/08/28 09:48:51 binlogstreamer.go:47: [error] close sync with err: sync is been closing...
2018/08/28 09:48:51 binlogsyncer.go:122: [info] syncer is closed
2018-08-28 09:48:51 ERROR Error 1146: Table 'test._test_ghc' doesn't exist
# Done
```

对一个1800万级别的表进行了修改，用了6分钟，说明速度还是不错的。

完成后，旧表被命名为`_tablename_del`，注意如果表太大，使用`DROP`或者`TRUNCATE`可能会卡主数据库，可以使用`DELETE`分多批删除。

### 小结

* 虽然官方的DDL存在一些问题，但是对于一些只修改元数据的操作，如删除索引，重命名字段，改变域的默认值、添加外键约束、删除外键约束等操作，执行起来非常迅速，使用官方的Online DDL没有问题；
* gh-ost有一些前置的限制，不过这些限制在实际使用中问题不是很大；
* gh-ost在执行前可以通过`--execute`来先测试；
* 尽管gh-ost的负载不是很大，但是对于大表，还是建议在非业务高峰期来做DDL操作。
