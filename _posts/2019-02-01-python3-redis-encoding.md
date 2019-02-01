---
layout: post
title: Python3 连接Redis字符串和字节问题探究
categories: Python
description: Python3 连接Redis字符串和字节问题探究
keywords: Python3 Redis encode bytes str
---

### 前言

Python3将逐渐代替Python2，Python2和3还是有很多不同。在开发过程中报了错:

```
  File "/home/hzw/project/test/virtualenv/lib/python3.5/site-packages/test-1.0.0-py3.5.egg/test/templates/wechat_config.js", line 11, in top-level template code
    var config = {{ data|dumps }};
  File "/usr/lib/python3.5/json/__init__.py", line 230, in dumps 
    return _default_encoder.encode(obj)
  File "/usr/lib/python3.5/json/encoder.py", line 198, in encode
    chunks = self.iterencode(o, _one_shot=True)
  File "/usr/lib/python3.5/json/encoder.py", line 256, in iterencode
    return _iterencode(o, 0)
  File "/usr/lib/python3.5/json/encoder.py", line 179, in default
    raise TypeError(repr(o) + " is not JSON serializable")
TypeError: b'cf43e4f8b6ba463599b616d60cf0683e' is not JSON serializable
```

数据是缓存在Redis里面的，取出来然后变成一个JSON字符串，但是Python3 `bytes`类型是无法被串行化为JSON的。需要显示的将`bytes`转换为字符串，然后才能被串行化。如果每一次从Redis里取出来自己串行化岂不是很麻烦。

本篇将介绍：
* Python3和Python2字符串处理的区别;
* 如何配置Redis处理这种区别，能够让Redis更加透明的工作。

### Python2和Python3字符串处理的区别

Python2和Python3字符串处理的区别，可以用[six](https://six.readthedocs.io/)(six是处理Python2和Python3兼容的一个库)里的一段代码来描述:

```
if PY3:
    string_types = str
    text_type = str
    binary_type = bytes
else:
    string_types = basestring
    text_type = unicode
    binary_type = str
```

也就是在Python3中不管是普通的字符串还是`unicode`字符串都是用`str`这种类型来表示的，而字节类型是用`bytes`来表示。而在Python2中，字符串的基类是`basestring`，他有两个子类，`str`代表单字节的字符串，而`unicode`代表`unicode`字符串，`str`和`bytes`是等同的。

总结来说：
```
Python3：
unicode == str  # unicode被去除了
str != bytes    # 两者无法自动转换

Python2:
unicode != str  # unicode表示unicode字符串，str表示单字节字符串
str == bytes    # str和bytes相同, 其实并没有bytes这种类型
```

```
Python 2.7.13 (default, Nov 24 2017, 17:33:09) 
[GCC 6.3.0 20170516] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> type('你好'), type(b'你好'), type(u'你好')
(<type 'str'>, <type 'str'>, <type 'unicode'>)
>>> isinstance('你好', basestring), isinstance(b'你好', basestring), isinstance(u'你好', basestring)
(True, True, True)
>>> '你好' == b'你好'
True
>>> u'你好' == '你好'
False
>>> u'hello' == 'hello'
True
>>> u'你好' == '你好'.decode('utf8')
True
>>> u'你好'.encode('utf8') == '你好'
True
```

可以看出Python2中`str`和`bytes`是完全等价的，所以`b`前缀的写法是多余的。`str`和`unicode`都是`basestring`的子类。

在书写字符串时，如果加前缀`u`，在内存中就是一个`unicode`类型的字符。 但是如果不加前缀，就是一个单字节的`str`，这个是根据文件的编码转换过来的。

`str`可以通过`decode`转换为`unicode`， `unicode`可以通过`encode`转换为`str`。

```
Python 3.5.3 (default, Jan 19 2017, 14:11:04) 
[GCC 6.3.0 20170118] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> type('你好'), type(u'你好'), type(b'hello')
(<class 'str'>, <class 'str'>, <class 'bytes'>)
>>> u'你好' == '你好'
True
>>> '你好'.encode('utf8')
b'\xe4\xbd\xa0\xe5\xa5\xbd'
>>> b'\xe4\xbd\xa0\xe5\xa5\xbd'.decode('utf8')
'你好'
>>> '\xe4\xbd\xa0\xe5\xa5\xbd'.decode('utf8')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'str' object has no attribute 'decode'
```

可见Python3中`str`和`unicode`无区别，都是用`str`来表示。而字节串用`bytes`表示，多字节的字符串无法自动转换为`bytes`。

`u`前缀变得多余了，但是`b`前缀却变得有必要了。`str`和`bytes`严格区分，无法自动转换。

`str`可以通过`encode`转换为`bytes`， `bytes`可以通过`decode`转换为`str`。


### Redis的问题

在Python3下面

```
Python 3.5.3 (default, Jan 19 2017, 14:11:04) 
[GCC 6.3.0 20170118] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from redis import StrictRedis
>>> r = StrictRedis('localhost', 6380)
>>> r.set('hello', '你好')
True
>>> r.get('hello')
b'\xe4\xbd\xa0\xe5\xa5\xbd'
```

可以看见，字符串输入被编码成utf8存储在Redis里了。而取出来的时候还是被编码后的`bytes`，需要显示的`decode`才能变成字符串。

### 解决方案

Redis建立连接时有两个参数，一个是`encoding`指定编码，默认是`utf8`。一个是`decode_responses`，默认为`False`，如果是`True`，会以`encoding`方式解码，然后返回字符串。如果是字符串，就根据`encoding`编码成`bytes`。

```
Python 3.5.3 (default, Jan 19 2017, 14:11:04) 
[GCC 6.3.0 20170118] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from redis import StrictRedis
>>> r = StrictRedis('localhost', 6380, encoding='utf8', decode_responses=True)
>>> r.set('hello', '你好')
True
>>> r.get('hello')
'你好'
```

可以看出加了这个步骤后，我们就不要显示的进行解码转换成字符串了。

### 更多问题

然后用这个连接后，出现了新的问题。

```
trace        Sat, 08 Dec 2018 15:09:38 ERROR    Traceback (most recent call last):
... 
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x80 in position 0: invalid start byte
```

因为`session`的存取使用了pickle来串行化session的存取。

```
r.set(sid, pickle.dumps(data))
pickle.loads(r.get(sid))
```

当`pickle.dumps(data)`存进去的是`bytes`，然后拿出来的进行`decode`的时候，这个不是正确的`utf8`编码，所以就出错了。

这种情况的话下，因为存储的时候是字节串，不会进行编码，如果我们把字节串变成字符串，就可以解决这个问题了。`latin1`用来编码字节是再好不错的事情了。

```
r.set(sid, pickle.dumps(data).decode('latin1'))
pickle.loads(r.get(sid).encode('latin1'))
```

这样存的话就是 Python对象 -> 字节串 -> latin字符串 -> utf8字节串存储在Redis里
取出来就是 utf8字节串 -> 解码变成字符串 -> 通过latin1编码成字节串 -> Python对象

### Python2的情况

```
Python 2.7.13 (default, Nov 24 2017, 17:33:09) 
[GCC 6.3.0 20170516] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from redis import StrictRedis
>>> r = StrictRedis('localhost', 6380)
>>> r.set('hello', '你好')
True
>>> r.get('hello')
'\xe4\xbd\xa0\xe5\xa5\xbd'
>>> r.set('hello', u'你好')
True
>>> r.get('hello')
'\xe4\xbd\xa0\xe5\xa5\xbd'
>>> r = StrictRedis('localhost', 6380, encoding='utf8', decode_responses=True)
>>> r.set('hello', '你好')
True
>>> r.get('hello')
u'\u4f60\u597d'
>>> r.set('hello', u'你好')
True
>>> r.get('hello')
u'\u4f60\u597d'
>>> import cPickle
>>> r.set('hello', cPickle.dumps({'a': u'你好'}).decode('latin1'))
True
>>> cPickle.loads(r.get('hello').encode('latin1'))
{'a': u'\u4f60\u597d'}
```

可见以上的方式对于Python2也是兼容的。

### 更进一步

如果，用`decode_responses=False`去连接Redis呢。这时候`dumps`是没有问题的，因为没有改变。但是`loads`就会有问题，此时Redis返回的是`bytes`,  执行`r.get(sid).encode('latin1')`时会报错，因为`bytes`没有`encode`方法。

这时我们需要做兼容，我们只需要`pickle.loads`的输入参数是一个`bytes`就行，可以通过`isinstance`来判断。不过有更好的办法，就是用`six`这个库。

```
import six
pickle.loads(six.ensure_binary(s, encoding='latin1'))
```

这样就能保证不管在Python2还是Python3中，`pickle.loads`的输入都是一个`bytes`。

### 小结

* 理解字符串相关的区别，是Python3程序必备;
* Redis连接时要理解`decode_responses`这个参数带来的影响;
* 要写出兼容Python2和Python3代码，可以使用`six`库，这个库只有900行，可以读一读。
