---
layout: post
title: BSON内部剖析
categories: Mongodb
description: 介绍BSON格式
keywords: Mongodb BSON JSON
---

# BSON内部剖析

### 背景
NoSQL一度如雨后春笋搬冒出来，最常用的是两种NoSQL，一种键值对数据库，代表就是Redis，而另一种就是文档型数据库，代表无疑是Mongodb。

Mongodb作为一种文档型数据库，数据库是存储为一个一个的文档的，支持内嵌文档和数组，数据格式和JSON很像，但是Mongodb并没有采用原始的JSON，而是对JSON做了一定的改进，衍化出一种新的数据格式，叫Binary JSON，简称BSON。

和名字一样，BSON是二进制编码的，和JSON存储为字符串不一样。为什么要采用BSON，而不直接使用JSON呢？ BSON相比JSON又有什么优势呢？ 本文将从BSON的编码方式来得到这些问题的答案。

### BSON的特点

* 轻量级，BSON编码很简单，就比JSON稍微复杂一点，空间开销比较小;
* 可遍历的，比如在存储字符串时，BSON会牺牲额外的空间，在字符串前加上字符串的长度，这样遍历速度就会加快，作为Mongodb的主存储数据，这个非常重要;
* 有效的，BSON的编码和解码都非常快;
* BSON相比JSON增加了一些新的数据类型。

作为Mongodb的文档的存储格式，有以上特点是非常重要的。

### BSON编码

#### 基本数据类型

BSON的基本数据类型主要有

* `byte`    1 byte (8-bits)
* `int32`   4 bytes (32-bit signed integer, two's complement)
* `int64`   8 bytes (64-bit signed integer, two's complement)
* `uint64`  8 bytes (64-bit unsigned integer)
* `double`  8 bytes (64-bit IEEE 754-2008 binary floating point)
* `decimal128`  16 bytes (128-bit IEEE 754-2008 decimal floating point)

所有的数据结构都是以这几种数据类型为基础组合出来的。

#### 基础数据结构

BSON里用到的基本数据类型有`double`、`int64`、`int32`，`decimal128`。其它的数据结构有:

* `string`，`string ::= int32 (byte*) "\x00"`，作为BSON里最常用的数据类型，字符串是存储为C字符串的，也就是以`\x00`结尾，所有的字符串都是`utf8`编码的。`string`最大的特点就是前面有一个整型记录字符串的长度(包括`\x00`)。这算是一种以空间换时间的方式，有了这个长度，找到整个字符串，遍历文档都会快很多。

* `cstring`，`cstring ::= (byte*) "\x00"`和`string`的区别就是没有前导的长度字段。

* `binary`，`int32 subtype (byte*)`，表示二进制数据，`int32`指的是后面`(byte*)`的长度，`subtype`指的这种二进制的子类型，包括一般二进制、函数、UUID等。

* `ObjectID`，`(byte*12)`，Mongodb里默认的`id`类型，由12个字节组成。

* `\x00`和`\x01`分别代表`false`和`true`。

* `UTC datetime`，用`int64`来表示。

* `JS code`，用`string`来表示。

* `Timestamp`，用`uint64`来表示。

#### 语法

根据以上对基础数据结构的描述，大家肯定会想，一个`string`或者`int`可以表示多种数据结构，而`\x00`即可以表示`cstring`的结尾，也可以表示`false`，如何区分它们。以下来描述BSON的语法。

一个文档其实就是一个有序键值对序列，定义为:

```
document ::= int32 e_list "\x00"
```

可以看到一个文档由一个`int32`开头，记录了文档的字节数(包括`int32`和`\x00`)，以`\x00`结尾。中间是一个`e_list`，表示的就是一个键值对序列，有如下定义:

```
e_list  ::= element e_list | ""
```

这表示`e_list`是一个`element`的列表，而`element`就是一个键值对，`element`的定义如下：

```
element ::= type e_name [value]
```

其中`e_name`是一个`cstring`，代表键值。而`type`是一个`byte`代表值的类型。

* 如`type`为`\x07`代表值是一个`ObjectId`， 也就是此时`element`为`"\x07" e_name (byte*12)`。

*  如`type`为`\x03`代表值是一个内嵌`document`。

*  如`type`为`0x08`代表值是布尔值，也就是`"\x08" e_name "\x00"`代表`false`。

`value`有可能不存在，代表了几种特殊类型：

* `"\x0A" e_name` 代表`null`。
* `"\xFF" e_name` 代表最小键。
* `"\x7F" e_name` 代表最大键。

引用官方文档，`element`的定义如下:

```
element ::= "\x01" e_name double    64-bit binary floating point
|   "\x02" e_name string    UTF-8 string
|   "\x03" e_name document  Embedded document
|   "\x04" e_name document  Array
|   "\x05" e_name binary    Binary data
|   "\x06" e_name   Undefined (value) — Deprecated
|   "\x07" e_name (byte*12) ObjectId
|   "\x08" e_name "\x00"    Boolean "false"
|   "\x08" e_name "\x01"    Boolean "true"
|   "\x09" e_name int64 UTC datetime
|   "\x0A" e_name   Null value
|   "\x0B" e_name cstring cstring   Regular expression - The first cstring is the regex pattern, the second is the regex options string. Options are identified by characters, which must be stored in alphabetical order. Valid options are 'i' for case insensitive matching, 'm' for multiline matching, 'x' for verbose mode, 'l' to make \w, \W, etc. locale dependent, 's' for dotall mode ('.' matches everything), and 'u' to make \w, \W, etc. match unicode.
|   "\x0C" e_name string (byte*12)  DBPointer — Deprecated
|   "\x0D" e_name string    JavaScript code
|   "\x0E" e_name string    Symbol. Deprecated
|   "\x0F" e_name code_w_s  JavaScript code w/ scope
|   "\x10" e_name int32 32-bit integer
|   "\x11" e_name uint64    Timestamp
|   "\x12" e_name int64 64-bit integer
|   "\x13" e_name decimal128    128-bit decimal floating point
|   "\xFF" e_name   Min key
|   "\x7F" e_name   Max key
```

嵌入文档和数组的编码方式和顶级的`document`一样。其中对于数组，BSON将它视为键为'0'，'1'，'2'...的文档，如`['hello', 'world']`被视为`{'0': 'hello', '1': 'world'}`。

#### 编码举例
`{"hi": "python"}`

* 键值被编码成`hi\x00`的C字符串;
* 值先是字符串长度，7个字节，然后就是C字符串，所以是`\x07\x00\x00\x00python\x00`;
* 因为值是一个`string`，所以`type`是`\x02`, 所以键值对就表示为`\x02hi\x00\x07\x00\x00\x00python\x00`;
* 键值对总共有15个字节，然后加上前导的4个字节和结尾的一个字节，共有20个字节，所以最后表示为`\x16\x00\x00\x00\x02hi\x00\x07\x00\x00\x00python\x00\x00`。

### 小结

通过以上对BSON编码的描述，我们可以知道BSON具有以下特点:

* JSON是像字符串一样存储的，BSON是按结构存储的；
* 因为BSON在头部存储了长度，所以可以很容易跳过一个文档或者字段，而JSON需要扫码整个数据结构，所以BSON的遍历速度快于JSON；
* 对于JSON来说，数据是无类型的，如果需要修改一个基本值，如99到100，字符就从2两个变成3个，可能需要内容移动，但是使用BSON，存储了类型值，文档长度不会变大，不需要移动，存储更加高效，也更不容易出现碎片；
* BSON对存储空间的消耗与JSON相比，可能更大，也可能更小。举例来说，如果值都是小数字，JSON消耗的空间更小，如果是比较大的数字，JSON的空间更大，总体来说，BSON的编码还是比较紧凑的，空间消耗较小；
* BSON相对JSON来说有丰富的数据类型。
