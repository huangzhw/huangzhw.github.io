---
layout: post
title: 记Python import遇到的坑
categories: Python
description: 记Python import遇到的坑
keywords: Python import 
---

## 问题
这几天，站点不时出现以下`exception`
```
File "/home/uu/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/handler/__init__.py", line 265, in _game_name_id_map
    return util.dbresult2dict(self.xxx_db.game_info.find(fields={
AttributeError: 'module' object has no attribute 'dbresult2dict'
```

这是看起来，很奇怪，因为检查了`util.py`，发现`dbresult2dict`明明在啊，重启uwsgi以后，这个问题就不存在，但是又会突然冒出来。

### 项目结构
项目文件布局如下:
```
xxx
|—- __init__.py
|—- url.py
|—- wsgi.py
|—- handler
    |—- __init__.py
    |—- util.py
    |—- ...
|—- base
    |—- __init__.py
    |—- util.py
|—- ...
```

各文件内容如下，只列出了和问题相关的代码：
```
# handler/__init__.py
from xxx.base import util
...
class BaseHandler(object):
    ...

# handler/util.py
from xxx import handler
...
class PicHandler(handler.BaseHandler):
    ...

# base/util.py
def dbresult2dict(dbresult, key, value=None, default_value=None):
    ...
```

## 分析
为什么`handler/__init__.py`里导入的`base/util.py`里`dbresult2dict`是存在的，但是却报了不存在的异常，最有可能的就是`util`这个模块被其它是东西取代了，变成了其它的模块，但是究竟是怎么变成其它模块的呢。

这个问题不是必现的，而是偶尔出现的，所以一定要让它复现。分析了一下项目的结构，发现有`base/util.py`和`handler/util.py`这两个重名文件，最有可能是这两个文件引起的冲突。

试着访问了一下`handler/util.py`里的`PicHandler`对应的url，再去访问之前出错的url，错误真的复现了。打印了日志，重新测试了一下，发现`handler/__init__.py`里面的util果然变成了`handler/util.py`这个模块。

但是这究竟是怎么出现的呢？有必要对`python`的`import`的流程做一个深入的了解。

## import探究

python 在导入包或者模块的时候,会做以下几件事情:

1. 检查系统的模块注册表(`sys.modules`)，看是否已经导入该模块，如果已经导入则直接使用该模块，如果没有则继续执行。
2. 创建一个新的空的`module`对象（它可能包含多个`module`)。
3. 把这个`module`对象插入`sys.module`中。
4. 装载`module`的代码（如果需要，首先必须编译）。
5. 执行新的`module`中对应的代码。

对于第5步，如果`module`为一个文件夹，那么文件夹下面必须要有一个文件名为`__init__.py`，在导入包的时候python会执行`__init__.py`里面的代码。但是总之，不论`module`为文件或者是文件夹，python每执行一段代码，该代码处定义的全局的变量或者函数都会被加载到`module`里面。之后python就可以使用这些导入的模块了。

所以我们可以看到，模块只会导入一次，然后缓存在`sys.modules`里，这样多个不同的文件访问相同的模块，不会让模块被导入多次。

那么`import a`，`import a.b`， `from a import c`究竟做了什么事情呢。

以下的讨论当模块不存在时，导入会出错，所以就不考虑这种情况了。

### import a
`import a`比较简单，如果模块`a`未导入，就导入模块，如果已经导入，就在当前作用域中绑定一个变量`a`，指向这个模块。举例如下：
```
>>> import os
>>> locals()
{'__builtins__': <module '__builtin__' (built-in)>, '__name__': '__main__', 'os': <module 'os' from '/usr/lib/python2.7/os.pyc'>, '__doc__': None, '__package__': None}
```

### import a.b
`import a.b`的行为可能和我们想得不一样，如果模块`a`未导入，就先导入模块`a`，然后如果模块`a.b`未导入就导入模块`a.b`，然后在当前作用域绑定一个变量`a`，指向模块`a`。举例如下:
```
>>> import sys
>>> import xxx.base
>>> sys.modules['xxx']
<module 'xxx' from '/home/hzw/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/__init__.pyc'>
>>> sys.modules['xxx.base']
<module 'xxx.base' from '/home/hzw/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/base/__init__.pyc'>
>>> locals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'sys': <module 'sys' (built-in)>, '__name__': '__main__', 'xxx': <module 'xxx' from '/home/hzw/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/__init__.pyc'>, '__doc__': None}
```

### from a import c
和上面不同，这里的c既可以是一个模块，也可以是一个模块里的变量。会优先导入变量，再考虑导入模块。

`from a import c`如果`a`中不存在`c`变量，那么`c`是一个模块，这时模块`a`未导入就导入模块`a`，如果模块`c`未导入，就导入模块`c`，然后在当前作用域绑定一个变量`c`指向模块`c`。举例如下:
```
>>> import sys
>>> from xxx import base
>>> sys.modules['xxx']
<module 'xxx' from '/home/hzw/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/__init__.pyc'>
>>> sys.modules['xxx.base']
<module 'xxx.base' from '/home/hzw/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/base/__init__.pyc'>
>>> locals()
{'__builtins__': <module '__builtin__' (built-in)>, '__package__': None, 'sys': <module 'sys' (built-in)>, 'base': <module 'xxx.base' from '/home/hzw/project/xxx/virtualenv/local/lib/python2.7/site-packages/xxx-1.0.0-py2.7.egg/xxx/base/__init__.pyc'>, '__name__': '__main__', '__doc__': None}
```

`from a import c`如果`a`中有`c`变量，则如果模块`a`未导入，则导入模块`a`，然后在本地绑定一个变量`c`指向导入的`c`。

### 导入一个父模块后，子模块的问题
假设模块如下结构
```
a
|—- __init__.py
|—- b.py
|—- c.py
```
做如下试验：
```
>>> import a
>>> dir(a)
['__builtins__', '__doc__', '__file__', '__name__', '__package__', '__path__']
>>> import a.b
>>> dir(a)
['__builtins__', '__doc__', '__file__', '__name__', '__package__', '__path__', 'b']
>>> import a.c
>>> dir(a)
['__builtins__', '__doc__', '__file__', '__name__', '__package__', '__path__', 'b', 'c']
```
可以看出导入一个父模块时，子模块是不被导入的，在父模块里并没有指向子模块的引用，但是当导入父模块里的子模块时，父模块里就增加了对子模块的引用。

### 举例说明
结构：
```
world
|—- __init__.py
|—- a.py
|—- b.py
```
文件内容：
```
#a.py
from b import D
class C(object):
    pass

#b.py
from a import C
class D(object):
    pass
```
执行语句：
```
^_^ /home/hzw/world $ python a.py
Traceback (most recent call last):
  File "a.py", line 4, in <module>
    from b import D
  File "/home/hzw/world/b.py", line 4, in <module>
    from a import C
  File "/home/hzw/world/a.py", line 4, in <module>
    from b import D
ImportError: cannot import name D
```
为什么找不到`D`？

1. 执行a.py中的`from b import D`，在`sys.modules`中并没有`<module b>`存在， 首先为`b.py`创建一个`module`对象， 这个`module`对象是空的，在 Python 内部创建这个对象后，就会解析执行`b.py`，填充`<module b>`的`__dict__`。
2. 执行`b.py`中的`from a import C`，首先检查`sys.modules`缓存中是否已经存在`<module a>`， 不存在所以为`a.py`创建一个`module`对象， 执行A.py中的语句。
3. 再次执行`a.py`中的`from b import D`这时，由于在第1步时，创建的`<module b>`对象已经缓存在了`sys.modules`中， 所以直接就得到了`<module B>`， 但是这时`<module b>`还是一个空的对象，所以获得符号`D`的操作就会抛出异常。


## 问题原因分析
有了以上关于模块的知识后，我们就可以来分析到底是什么原因造成如上的bug的。

1. 当访问某一个url时，webapp2就会通过`router`找到对应的`handler`类，然后导入这个`handler`类，这个`handler`类对应的模块里有一句话`from xxx import handler`，就会导入`handler`模块，执行`handler/__init__.py`，而这里面有一句`from xxx.base import util`，所以在`handler.util`里变量`util`指向模块`base/util.py`模块。
2. 而当访问`handler/util.py`里面handler对应的url时，比如`PicHandler`，会导入执行`from xxx.handler.util import PicHandler`，这是模块`xxx.handler.util`会被导入，父模块`xxx.handler`里会有对子模块的引用，所以`handler.util`就会被重新指向`handler/util.py`。
3. 经过以上两个步骤，`handler/__init__.py`里的util就不在指向`base/util.py`，而是指向`handler/util.py`，所以当调用这里不存在的方法时就会出错。

### 举例验证
有如下的项目布局：
```
hello
|—- __init__.py
|—- handler
    |—- __init__.py
    |—- util.py
```
其中文件内容如下：
```
# hello/__init__.py
handler = 'hello'
```
有如下结果：
```
>>> import hello
>>> hello.handler
'hello'
>>> import hello.handler
>>> hello.handler
<module 'hello.handler' from 'hello/handler/__init__.pyc'>
```
可以看出`hander`变量的绑定果然发生了改变。

## 解决方案
* 修改`handler/util.py`的命名，不过这是一种不太好的方式，很难阻止错误再次出现，不能从根本上解决问题。
* 应该保持`handler/__init__.py`的简单化，在这个文件里导入太多其它模块，会导致些被导入的模块都暴露在这个模块的接口中，引发刚才的冲突，修改修改这个文件，将这这个文件里的定义移到其它文件，然后在这个文件里导出应该暴露的接口。

## 小结
* 了解Python的一些底层运行机制，对写出优雅的少bug的代码是很有帮助的。
* 在编码中一定要遵循一定的规范，比如保持`__init__.py`的简单性，这样就可以避免这种错误的出现。
* 对外的接口一定要保持最小化，不要引入其它无用的东西。
