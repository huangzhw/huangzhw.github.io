---
layout: post
title: redis-py里的Sentinel到底是如何工作的 
categories: Redis Python
description: redis-py Sentinel 源码分析
keywords: Redis Python Sentinel redis-py
---

## redis-py里的Sentinel到底是如何工作的

### 前言

为了高可用性，在使用Redis时，我们往往使用Sentinel模式。我们会以如下方式使用:

```
from redis.sentinel import Sentinel
conf = {
    'sentinel': [('10.160.84.01', 26379), ('10.160.85.02', 26379), ('10.160.86.03', 26379)],
    'master_group_name': 'test',
    'connection_conf': {
        'socket_timeout': 0.5,
        'retry_on_timeout': True,
        'socket_keepalive': True,
        'max_connections': 10,
        'db': 0,
        'encoding': 'utf8',
        'decode_responses': True,
    }
}
sentinel = Sentinel(conf['sentinel'], **conf['connection_conf'])
sentinel.discover_master(conf['master_group_name'])
cli = sentinel.master_for(conf['master_group_name'])
cli.set('hello', 'word')
cli.get('hello')
```

这时候我们往往会有几个问题：

* 查询时是否使用到了线程池；
* 如何从Sentinel拿到真实的服务器地址；
* 每次查询前，是不是都要走一遍`discover_master`和`master_for`的流程；
* 如果Redis服务器发生了主从切换，是不是要重新执行`discover_master`和`master_for`的流程。

尤其是对问题3、4，大家在使用的时候，往往很疑惑，是否应用需要重启，或者重新连接才能连到新的master去，或者需要在执行命令前不断的调用`discover_master`和`master_for`。

在这篇文章中，我们通过探索redis-py的源码，来解答这些问题，代码基于redis-py 3.2.1。

### Sentinel简介

首先我们来简介一下Sentinel的工作原理，Sentinel里使用了Raft来进行宕机时的选主。一般至少三台机器，我们假设一个集群里有三台机器。

* 3台机器在`6379`端口启动Redis服务器；
* 3台机器中一台机器选为master，另外两台机器选为slave，从master同步数据；
* 3台机器在`26379`端口启动Sentinel进程；
* 每一个Sentinel进程都监听3台服务器上的Redis进程；
* 当master宕机时，3个Sentinel通过Raft协议来进行选主；
* 客户端可以连接到Sentinel获取到maser和slave的ip，然后连接到相应的Redis服务器。

### 单机时redis-py是如何工作的

我们首先来看看`redis-py`单机时是如何工作的，如何使用线程池。

单机时的连接代码一般是这样的:

```
import redis
conf = {
    'host': '10.20.30.40',
    'socket_timeout': 0.5,
    'retry_on_timeout': True,
    'socket_keepalive': True,
    'max_connections': 10,
    'db': 0,
    'encoding': 'utf8',
    'decode_responses': True,
}
cli = redis.Redis(connection_pool=redis.ConnectionPool(**conf))
cli.set('hello', 'world')
cli.get('hello')
```

`cli`是一个`Redis`的实例，初始化时为它提供了一个连接池。我们来看看`get`如何执行的。

首先调用`client.py/Redis`的`get`方法：

```
def get(self, name):
    """
    Return the value at key ``name``, or None if the key doesn't exist
    """
    return self.execute_command('GET', name)
```

可以看到`get`只是调用了`execute_command`方法。

```
def execute_command(self, *args, **options):
    "Execute a command and return a parsed response"
    pool = self.connection_pool
    command_name = args[0]
    connection = pool.get_connection(command_name, **options) # 从连接池里拿到一个连接
    try:
        connection.send_command(*args)  # 向连接发送请求
        return self.parse_response(connection, command_name, **options)  # 解析响应
    except (ConnectionError, TimeoutError) as e:
        connection.disconnect()  # 断开连接
        if not (connection.retry_on_timeout and
                isinstance(e, TimeoutError)):  # 如果是超时，并且需要重试，再重试一次
            raise
        connection.send_command(*args)
        return self.parse_response(connection, command_name, **options)
    finally:
        pool.release(connection)  # 将连接放回到连接池
```

这个函数看到后一目了然，需要执行命令时，从连接池拿出一个连接，用这个连接执行命令，命令执行后放回连接。所以这里无缝使用了连接池，实现了线程安全性。

连接池管理了连接，如果需要连接时，若连接池中还有连接，就拿出一个连接。如果连接池为空，但是当前连接数小于配置，就再新建一个连接，否则报错。

基本上是这样一个流程:

`get` -> `execute_command` -> `get_connection` -> `send_command` -> `parse_response`

### redis-py Sentinel如何工作

再看到了单机的Redis的工作方式后，我们再来看看Sentinel的工作方式。

`sentinel.py/Sentinel`的`discover_master`的作用比较简单，就是去Sentinel查询`master`是哪台机器，每次调用这个函数，都会用`sentinel`命令去Sentinel查询。

#### master_for做了什么

首先来看看Sentinel得到的`cli`是什么类型的：

```
>>> type(cli)
<class 'redis.client.Redis'>
```

我们惊奇的发现，它同样是一个`Redis`类的实例。

我们来看`sentinel.py/Sentinel`的`master_for`方法做了什么:

```
def master_for(self, service_name, redis_class=Redis,
                connection_pool_class=SentinelConnectionPool, **kwargs):
    kwargs['is_master'] = True
    connection_kwargs = dict(self.connection_kwargs)
    connection_kwargs.update(kwargs)
    return redis_class(connection_pool=connection_pool_class(
        service_name, self, **connection_kwargs))
```

这个函数非常简单，它只是返回了一个`Redis`实例，但是与单机的Redis不同，它的连接池类型是`SentinelConnectionPool`。

所以Sentinel执行命令和单机Redis一样，都通过`execute_command`从连接池获取连接执行命令，然后放回连接池，都是`Redis`的实例，说明Sentinel和单机的不同主要是在连接池。

#### SentinelConnectionPool 有什么特别的

`SentinelConnectionPool`继承自`ConnectionPool`，它的默认连接是`SentinelManagedConnection`：

```
class SentinelConnectionPool(ConnectionPool):

    def __init__(self, service_name, sentinel_manager, **kwargs):
        kwargs['connection_class'] = kwargs.get(
            'connection_class', SentinelManagedConnection)
            
    def get_master_address(self):
        master_address = self.sentinel_manager.discover_master(
            self.service_name) # 通过discover_master获取master的地址
        if self.is_master:
            if self.master_address is None:
                self.master_address = master_address
            elif master_address != self.master_address: # 这次获取的地址和上次不一样，就断开连接池的所有连接
                # Master address changed, disconnect all clients in this pool
                self.disconnect()
        return master_address        
```

我们省略了其它相关性不大的代码，可以看到和普通`ConnectionPool`最大的不同就是`SentinelConnectionPool`的连接类型是`SentinelManagedConnection`。他还有一个`get_master_address`方法，这个方法通过连接到`Sentinel`进程通过`Sentinel`命令获取到master的地址。

`SentinelConnectionPool`非常简单，我们只列出了最重要的`connect`和`read_response`方法。

```
class SentinelManagedConnection(Connection):

    def connect(self):
        if self._sock:
            return  # already connected
        if self.connection_pool.is_master:  # 如果连接池是master，就连接到master的地址
            self.connect_to(self.connection_pool.get_master_address()) # 每次都通过get_master_address获取master地址
        else:
            for slave in self.connection_pool.rotate_slaves():
                try:
                    return self.connect_to(slave)
                except ConnectionError:
                    continue
            raise SlaveNotFoundError  # Never be here

    def read_response(self):
        try:
            return super(SentinelManagedConnection, self).read_response()
        # 服务端返回异常，表示自己不是master
        except ReadOnlyError:
            if self.connection_pool.is_master:
                # When talking to a master, a ReadOnlyError when likely
                # indicates that the previous master that we're still connected
                # to has been demoted to a slave and there's a new master.
                # calling disconnect will force the connection to re-query
                # sentinel during the next connect() attempt.
                self.disconnect()
                raise ConnectionError('The previous master is now a slave')  // 将异常转换为ConnectionError
            raise
```

我们发现通过`connect`创建连接时，每次都会先去Sentinel查询一次master的地址。

那么`read_response`方法是什么时候被调用的呢？就在`execute_command`的`parse_response`方法里调用。

master宕机发生主从切换时，有两种场景，第一种是原master宕机了，这时候这时候客户端无法连接到master, 会产生`ConnectionError`或者`TimeoutError`异常。第二种是原master会变成slave，这样就会返回`ReadOnlyError`异常，会被转换为`ConnectionError`。

无论哪种情况，都会在`execute_command`里调用`Connection`里的`disconnect`方法。

这样在下次再使用这个连接时，因为连接断开了，就会再次调用`SentinelManagedConnection`的`connect`创建连接，而`connect`调用`connection_pool`的`get_master_address`方法。我们又回到最初的起点，发现这里调用`get_master_address`获取master的地址，而且都是实时获取的，如果发现master地址变了，就会断开所有的连接，重新连接。

最后总结一下redis-py的Sentinel维持master的地址的方式是每次创建连接时都会去动态获取一次master的地址，而不是每次查询时都去获取一次master。不然的话，每一次请求都实际需要两次请求，吞吐量就下降了不少。而检查到master的异常后，会断开所有连接，然后从连。

### 小结

所以一开始提出的问题可以回答了。

* 只要配置了线程池，查询是使用线程池的;
* 从Sentinel拿到真实的地址，是先连接到Sentinel进程，然后执行`sentinel`命令获取到master的地址;
* 每次查询时并不需要执行`discover_master`和`master_for`，这些都会在连接的时候自动执行;
* 发生主从切换时，客户端会通过超时或者`ReadOnlyError`自动断开连接，然后重新连接的时候会通过再次获取master地址连接到新的master。
