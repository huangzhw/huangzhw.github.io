---
layout: post
title: Mongodb Date 之坑探究
categories: Mongodb
description: 介绍Mongodb Date的坑
keywords: Mongodb Date
---

# Mongodb Date 之坑探究

### 前言

项目很多数据使用Mongodb来存储，字段里常用到`Date`类型。

简单情况下，我们`Python`经常使用`datetime.now()`生成一个当前日期存储在Mongodb里，然后根据`datetime`生成日期进行查询，这通常工作得很好。然而，当我们在mongo shell里执行查询时，问题就出现了。

我们现在`Python`里执行:

```
>>> db.test.insert({'insert_datetime': datetime.now()});
ObjectId('59ec517e23f73722de10cde2')
>>> db.test.find_one({'insert_datetime': {'$lte': datetime.now()}})
{u'_id': ObjectId('59ec517e23f73722de10cde2'), u'insert_datetime': datetime.datetime(2017, 10, 22, 16, 6, 22, 559000)}
```

可以看到这条记录马上被查到，接着我们马上在mongo shell里查询:

```
mongos> db.test.findOne({'insert_datetime': {$lte: new Date()}});
null
mongos> db.test.findOne();
{
    "_id" : ObjectId("59ec505f23f7371325258bdc"),
    "now" : ISODate("2017-10-22T16:01:35.915Z")
}
```

同样的查询条件，记录无法查到。

### Mongodb Date实现

#### Date的问题

在mongo shell里执行(现在时间是`2017-10-22 16:14:27`):

```
mongos> now = new Date()
ISODate("2017-10-22T08:14:27.326Z")
mongos> now instanceof Date
true
mongos> now instanceof ISODate
false
```

以上我们可以看出两个问题：

* 实际显示时间比我们的本地时间差八小时；
* 出现的日期显示的是一个`ISODate`，但是类型是`Date`类型。

#### Date实际如何存储

根据官方文档[https://docs.mongodb.com/manual/reference/method/Date/](https://docs.mongodb.com/manual/reference/method/Date/), 在Mongodb内部存储时，`Date`被存储为一个64bit的整数，代表从`Unix epoch (Jan 1, 1970)`开始的毫秒数，简单说`Date`被存储为一个整数时间戳，而在shell里则被转换为一个js `Date`类型。

时间戳是无关时区的，对于0时区就是和`1970-01-01 00:00:00`之差的毫秒数，而对于我们东八区，则是与`1970-01-01 00:08:00`之差的毫秒数。也就是不管在哪个时区，同一时间的时间戳是一样的。

所以看上面的第一个问题，时间实际显示的UTC时间，所以和我们本地时间相差八小时。也就是不管输入的是什么，实际显示的都是UTC时间，而存储的是时间戳。

所以我们在mongo shell里构造时间时，就要注意了：

```
mongos> new Date('2017-10-22 16:11:00');
ISODate("2017-10-22T08:11:00Z")
mongos> new Date('2017-10-22 16:11:00Z');
ISODate("2017-10-22T16:11:00Z")
mongos> new Date('2017-10-22 16:11:00+08:00');
ISODate("2017-10-22T08:11:00Z")
```
可以看出，普通输入时间时，表示这是一个本地时间，会被转换为UTC时间存储，我们也可以后面加一个`Z`表示这就是一个UTC时间，也可以指明这个时间所在的时区，最后也是转换为UTC时间。

#### ISODate 是什么？

`ISODate`其实只是一个shell帮助函数，看Mongodb里面的源代码：

```
ISODate = function(isoDateStr) {
    if (!isoDateStr)
        return new Date();

    // 时间字符串的正则表示
    var isoDateRegex =
        /^(\d{4})-?(\d{2})-?(\d{2})([T ](\d{2})(:?(\d{2})(:?(\d{2}(\.\d+)?))?)?(Z|([+-])(\d{2}):?(\d{2})?)?)?$/;
    var res = isoDateRegex.exec(isoDateStr);

    if (!res)
        throw Error("invalid ISO date: " + isoDateStr);

    var year = parseInt(res[1], 10);
    var month = (parseInt(res[2], 10)) - 1;
    var date = parseInt(res[3], 10);
    var hour = parseInt(res[5], 10) || 0;
    var min = parseInt(res[7], 10) || 0;
    var sec = parseInt((res[9] && res[9].substr(0, 2)), 10) || 0;
    var ms = Math.round((parseFloat(res[10]) || 0) * 1000);

    var dateTime = new Date();

    // 一开始默认得到一个UTC时间
    dateTime.setUTCFullYear(year, month, date);
    dateTime.setUTCHours(hour);
    dateTime.setUTCMinutes(min);
    dateTime.setUTCSeconds(sec);
    var time = dateTime.setUTCMilliseconds(ms);

     // 如果包含不包含Z，说明这是一个本地时间，或者含有时区信息，转换成UTC时间
    if (res[11] && res[11] != 'Z') {
        var ofs = 0;
        ofs += (parseInt(res[13], 10) || 0) * 60 * 60 * 1000;  // hours
        ofs += (parseInt(res[14], 10) || 0) * 60 * 1000;       // mins
        if (res[12] == '+')                                    // if ahead subtract
            ofs *= -1;

        time += ofs;
    }
    
    // ...

    return new Date(time);
```

以上可以看出`ISODate`只是`Date`类型的一个帮助函数， 返回一个`Date`，在shell显示一个`Date`的时间。

### python操作Mongodb Date

#### python datetime 插入mongo表现

从前言里的操作可以看出来，用`datetime.now()`生成一个本地时间，是没有时区信息的，存储到Mongodb后，好像这个时间是一个UTC时间，所以和真实的UTC时间差了八小时。简单来说，我们用python生成一个本地时间`2017-10-22T16:01:35.915`，但是mongo服务器把它当做了一个UTC时间`2017-10-22T16:01:35.915Z`，所以在mongo shell里取出后，这个时间就变成了`2017-10-22T16:01:35.915Z`，也就是比我们的本地时间还要相差八个小时。

如果我们使用`datetime.utcnow()`存储时间：

```
>>> db.test1.insert({'insert_datetime': datetime.utcnow()});
ObjectId('59ec5db923f73722de10cde4')
>>> db.test1.find_one();
{u'_id': ObjectId('59ec5db923f73722de10cde4'), u'insert_datetime': datetime.datetime(2017, 10, 22, 8, 58, 33, 956000)}
```

然后在mongo shell里取出：

```
mongos> db.test1.findOne();
{
    "_id" : ObjectId("59ec5db923f73722de10cde4"),
    "insert_datetime" : ISODate("2017-10-22T08:58:33.956Z")
}
```

这时候两个地方表现就统一了。

#### 原因分析

为什么会这样呢？

我们通过`pymongo 3.5`源码来进行分析。

最重要的就是，时间戳和`datetime`如何转换。

```
EPOCH_AWARE = datetime.datetime.fromtimestamp(0, utc)
EPOCH_NAIVE = datetime.datetime.utcfromtimestamp(0)
  
def _millis_to_datetime(millis, opts):
    """Convert milliseconds since epoch UTC to datetime."""
    diff = ((millis % 1000) + 1000) % 1000
    seconds = (millis - diff) / 1000
    micros = diff * 1000
    // 可以看到连接时，如果需要考虑时区，就会根据存储的时间戳，加上一个时区信息
    if opts.tz_aware:
        dt = EPOCH_AWARE + datetime.timedelta(seconds=seconds,
                                                microseconds=micros)
        if opts.tzinfo:
            dt = dt.astimezone(opts.tzinfo)
        return dt
    // 如果不考虑时区，就会`1970-01-01 00:00:00`加上时间戳得到本地时间
    else:
        return EPOCH_NAIVE + datetime.timedelta(seconds=seconds,
                                                  microseconds=micros)


def _datetime_to_millis(dtm):
    """Convert datetime to milliseconds since epoch UTC."""
    // 如果有时区信息转换成UTC时间
    if dtm.utcoffset() is not None:
        dtm = dtm - dtm.utcoffset()
    // 返回和`1970-01-01 00:00:00`的差值
    return int(calendar.timegm(dtm.timetuple()) * 1000 +
               dtm.microsecond / 1000)
```

由上面的源码可见，在我们默认连接mongodb时，没有设置考虑时区，存储时间时，把时间转换为实际的时间戳时，这时候就会计算本地时间和`1970-01-01 00:00:00`的差值，而不是计算和`1970-01-01 00:08:00`的差值，所以这时候实际的时间戳时多了八个小时的。但是当我们使用`datetime.utcnow()`时，时间是本来就减了八小时的，所以就是插入了正确的时间。

而当我们从Mongodb取出时间时，会将时间戳和`1970-01-01 00:00:00`相加，所以得到的时间和真实的实际少了八小时。

上面两点，简单来说，就是python在时间戳转换时好像看不到时区，就把本地直接当做UTC时间来进行操作。所以得到的时间戳和真实的时间戳相差了八个小时。

而如果连接时，指定了时区，则会将时区计算进去，那么和Mongodb里就兼容了。

### 小结

* 在Python里，在默认情况下，我们直接使用本地时间去存储时间，然后取得时间，是不会出问题的，得到的也是当前的本地时间，但是实际存储的时间戳，比真实的时间戳大了八个小时。但是如果用mongo shell或者其它驱动程序读取数据时，就需要考虑这个问题了，比如使用mongo shell时需要比较时，要加上这八个小时，才能正确的比较，这就回答了一开始提出的问题。
* 当然默认连接Mongodb的情况下，也可以使用正确的存储时间戳，也就是python里都使用UTC时间存储，当前时间使用`datetime.utcnow()`，而其它时间要减去当前的时区存储，不过这样也比较麻烦，容易出错。
* 或者，连接Mongodb时使用`tz_aware`设定时区，这样就可以用本地时间了，而实际存储在Mongodb里的时间就是正确的时间戳了，而不会有八个小时的偏移了。
* 不管使用哪种方法，最重要的就是使用统一的方式，否则就有可能出错。
