---
layout: post
title: MySQL用户id超过最大值后的处理
categories: MySQL
description: 记一次线上用户ID超过int的最大值后的处理
keywords: MySQL int overflow bigint information_schema
---

## MySQL用户id超过最大值后的处理

### 起因

服务端日志里突然报错:

```
Traceback (most recent call last):
...
py2.7.egg/xxx/handler/__init__.py", line 150, in _update_session
    user_info['email'] = None
TypeError: 'NoneType' object does not support item assignment
```

这里的语句是:

```
if not user_info:
    nowtime = datetime.datetime.now()
    QS(self.db_writer).table(T.user).insert({
        'mobile': self.uid,
        'register_time': nowtime
    }, ignore=True)
    user_info = QS(self.db_writer).table(T.user).where(
        F.mobile == self.uid
    ).select_one('id')
    user_info['email'] = None
```
不可思议的事情出现了，照理说是先插入用户的，用户怎么都应该存在的。

我尝试在数据库里插入一条记录，发现报错：

```
ERROR 1062 (23000): Duplicate entry '2147483647' for key 'PRIMARY'
```

如果是执行`insert ignore`的话，没有报错，但是实际上没有插入记录。

一看，发现id值已经达到了`int`的最大值。但是`int`最大值有`2147483647`这么大，但是用户量并没有这么大，看了一下发现各个`id`间有非常大的间隙。

这么大间隙的原因是，每次用户登录都会执行`insert ignore`，但是大多数情况下并没有插入用户，但是还是浪费了一个id。

现在的影响就是新用户无法注册成功，但是老用户却没有影响，极度紧张中，吓尿了。

### 临时修复

考虑到用户`id`的大小或者顺序是不影响实际业务的，所以现在`id`用完了，我们可以强制指定一个用户`id`，这样用户就能插入了。所以添加了以下代码，考虑到间隙的大小，正常情况下1-2次就可以插入用户了:

```
if not user_info:
    max_count = 10
    while max_count > 0:
        max_count -= 1
        id_ = random.randint(1000000000, 2147483648)
        nowtime = datetime.datetime.now()
        QS(self.db_writer).table(T.user).insert({
            'id': id_,
            'mobile': self.uid,
            'register_time': nowtime
        }, ignore=True)
        user_info = QS(self.db_reader).table(T.user).where(
            F.mobile == self.uid
        ).select_one('id')
        if not user_info:
            continue
            user_info['email'] = None
        break
```

这样线上可以正确运行了，不过还是需要实际解决这个问题。

### 方案讨论

我们的MySQL有三台机器，`db1`和`db2`互为`master`，`db3`是`db1`的`slave`。我们的程序以`db1`为`master`进行写。

所以我们先把`db3`这台`slave`拿下来进行处理，保证线上有两台机器。

经过和SA的讨论，有三种方案可以处理：

* 将数据库的数据`dump`出来，然后重建数据库，将数据库里所有涉及到`user_id`的表都改成了`bigint`，重建数据库，然后将数据重新导入；
* 在线使用`ALTER`语句将所有`user_id`相关的表格的这个字段都改成`bigint`；
* 在线使用`ALTER`语句将所有`user_id`相关的表格的这个字段都改成`unsigned int`。

第一种方案，看了数据库的大小400多GB，估计是一个非常费时的操作，可能短期内无法完成。

使用第二种方案，还是第三种方案的唯一区别，就是第三种方案是否比第二种方案更加省时间。不然第二种方案是一劳永逸的解决方式。

经过讨论，我们选择了第二种方案。

### 初始尝试

首先对某一个表格执行转换的语句:

```
mysql> ALTER TABLE user_detail MODIFY user_id int(11);
ERROR 1832 (HY000): Cannot change column 'user_id': used in a foreign key constraint 'fk3_user_detail'
```

因为外键约束的存在，无法转换。如果把外键约束检查关闭呢？
```
mysql> SET FOREIGN_KEY_CHECKS=0;
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER TABLE user_detail MODIFY user_id int(11);
ERROR 1025 (HY000): Error on rename of './mall/#sql-4e9_69632' to './mall/user_detail' (errno: 150 - Foreign key constraint is incorrectly formed)
```

发现语句执行了，但是最后一步重命名文件的时候还是出错了。

看样子直接修改是行不通的，只能把存在的外键约束先drop掉，然后再开始修改了，最后再加上外键约束。经过试验这样是可以的。

### 创建修改脚本

#### 初始脚本
对于去除外键、修改表定义和加上外键等操作都可以使用`ALTER`语句。如下对`user_detail`进行操作。
```
ALTER TABLE user_detail MODIFY user_id BIGINT(20) NULL;
ALTER TABLE user_detail MODIFY user_id BIGINT(20) NULL;
// 执行完所有修改操作后
ALTER TABLE user_detail ADD CONSTRAINT fk3_user_detail FOREIGN KEY (user_id) REFERENCES user(id);
```

不过涉及到的表格有40多张，如果每一张都这么搞，容易漏，而且费力。

这时候想到`information_schema`有一些表格里包括了列和外键的信息。

首先是`COLUMNS`表格，包括了每个表中每一列的信息:
```
mysql> SELECT * FROM COLUMNS WHERE TABLE_NAME = 'user_detail' and COLUMN_NAME = 'user_id'\G
*************************** 1. row ***************************
           TABLE_CATALOG: def
            TABLE_SCHEMA: mall
              TABLE_NAME: user_detail
             COLUMN_NAME: user_id
        ORDINAL_POSITION: 1
          COLUMN_DEFAULT: NULL
             IS_NULLABLE: YES
               DATA_TYPE: int
CHARACTER_MAXIMUM_LENGTH: NULL
  CHARACTER_OCTET_LENGTH: NULL
       NUMERIC_PRECISION: 10
           NUMERIC_SCALE: 0
      CHARACTER_SET_NAME: NULL
          COLLATION_NAME: NULL
             COLUMN_TYPE: int(11)
              COLUMN_KEY: UNI
                   EXTRA: 
              PRIVILEGES: select,insert,update,references
          COLUMN_COMMENT: 
1 row in set (0.00 sec)
```

然后是`KEY_COLUMN_USAGE`，包括了键的约束。
```
mysql> SELECT * FROM KEY_COLUMN_USAGE WHERE TABLE_NAME = 'user_detail' and COLUMN_NAME = 'user_id' and REFERENCED_TABLE_NAME is not null\G
*************************** 1. row ***************************
           CONSTRAINT_CATALOG: def
            CONSTRAINT_SCHEMA: mall
              CONSTRAINT_NAME: fk3_user_detail
                TABLE_CATALOG: def
                 TABLE_SCHEMA: mall
                   TABLE_NAME: user_detail
                  COLUMN_NAME: user_id
             ORDINAL_POSITION: 1
POSITION_IN_UNIQUE_CONSTRAINT: 1
      REFERENCED_TABLE_SCHEMA: mall
        REFERENCED_TABLE_NAME: user
       REFERENCED_COLUMN_NAME: id
1 row in set (0.01 sec)
```

通过这两张表格我们就可以通过SQL语句生成执行语句。语句如下
```
SELECT concat('ALTER TABLE ', TABLE_NAME, ' DROP FOREIGN KEY ', CONSTRAINT_NAME, ';') FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE WHERE CONSTRAINT_SCHEMA = 'mall' AND REFERENCED_TABLE_NAME = 'user' AND REFERENCED_COLUMN_NAME = 'id';

SELECT CONCAT('ALTER TABLE ', TABLE_NAME, ' MODIFY ', COLUMN_NAME, ' BIGINT(20) ', IF(IS_NULLABLE = 'YES', 'NULL', 'NOT NULL'), ';') FROM INFORMATION_SCHEMA.COLUMNS where TABLE_SCHEMA = 'mall' and COLUMN_NAME like '%user_id%' and COLUMN_TYPE like '%int%' ORDER BY TABLE_NAME;

SELECT CONCAT('ALTER TABLE ', TABLE_NAME ,' ADD CONSTRAINT ', CONSTRAINT_NAME, ' FOREIGN KEY (',COLUMN_NAME,') REFERENCES ', REFERENCED_TABLE_NAME ,'(id);') FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE WHERE CONSTRAINT_SCHEMA = 'mall' AND REFERENCED_TABLE_NAME = 'user' AND REFERENCED_COLUMN_NAME = 'id';
```

第一句是选出所有参照`user`表`id`的列，然后组成语句去除外键约束。

第二句话根据根据`COLUMNS`表选出包含`user_id`名的列，这里就是所有的和`user_id`相关的列。

第三句刚好是第一句的相反的过程，创建语句重新加上外键约束。

执行以上三句话就可以生成脚本。不过因为这个依赖于字段名的命名，需要我们再仔细检查正确性。最后把下面这句加入生成外键约束之前:

```
ALTER TABLE user MODIFY id BIGINT(20) NOT NULL AUTO_INCREMENT;
```

#### 优化脚本

生成以上脚本后，开始执行脚本，发现对于大表，语句执行非常慢。对于一个有外键约束的表，有3个步骤，去除外键约束，修改字段定义和添加外键约束，每个阶段都非常得慢，是不是可以加快速度呢。发现第1个步骤和第2个步骤可以合并。

这样语句就变成了
```
ALTER TABLE user_detail DROP FOREIGN KEY fk3_user_detail, MODIFY user_id BIGINT(20) NULL;
```

这样修改后，执行速度就快了不少。

#### 执行脚本

对于大表的执行，需要很长时间，但是我们也无法获取执行的进度，这里有一个小技巧，就是查看数据库目录下的临时文件，这个临时文件就是当前的临时表，这个表的大小和原来表的大小大约比对，可以知道执行的进度。

对于执行脚本的机器，一定要有足够的磁盘空间。

### 后续工作

#### 同步

花了近一天时间后，脚本终于执行完毕了。现在就是最重要的步骤，修改后的`slave`是否能正确的从`master`同步，毕竟表定义有所不同，SA挂了上去，发现同步报错。

应该是slave上列定义是`bigint`，但是master上却是`int`的原因，查询了文档，发现有一个参数`slave_type_conversions`, 有以下取值：

* `ALL_LOSSY`：仅支持有损转换，什么叫有损？比如一个值本来是bigint存储为9999999999999，现在转换为int类型势必会要截断从而导致数据不一致。
* `ALL_NON_LOSSY`：仅支持无损转换，只能在无损的情况下才能进行转换
* `ALL_LOSSY,ALL_NON_LOSSY`：有损/无损转换都支持
* 空，即不设置这个参数：必须主从的字段类型一模一样。

对于我们的场景，需要支持无损转换，设置这个参数为`ALL_NON_LOSSY`后，同步可以完成。

#### 其它机器的修改

以上修改的过程繁琐，而且很花时间，不过有一个机子修改好后，其它的机子就快了。

* `db3`同步好后，将这台`slave`再次拿下来，将数据直接拷贝到另一台不使用的`master` `db2`上。
* 将`db3`再次挂上去，同步完成。
* 将`db2`拿下来，使用新的数据重启。
* 然后再将`db2`挂上去当成`slave`。

以上步骤，保证线上一直有两台机器，并且大约只用了三小时就处理好一台机器。

对于最后一个`db1`，我们选择了停服，来进行更新。

#### 修改代码
虽然`bigint`这辈子也不可能用完，不过我们还是需要修复原来的代码，进来减小`id`之间的间隙。

### 小结

* 在项目之初可能就是一句话的事情，但是在项目长时间运行后，这样就是一个修改起来非常复杂的事情了。
* 在MySQL中`insert ignore`和`auto_increment`组合起来的情况下需要慎用。
* MySQL的`information_schema`里提供了很多数据库信息的元数据。
