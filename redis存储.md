## overview

redis不是一个普通的key-value存储，而是一个支持多种不同value的`datas tructure server`

- string
- list：基于`linked list`的有序list，按照插入顺序进行排序
- set：唯一、无序的集合
- 有序集合，每一个element都和一个叫`score`的联系在一起，所有的element按照score进行排序
- hash：就像map一样拥有键值对，key-value都是string
- Bit arrays (or simply bitmaps)：
- HyperLogLogs: 
- Streams: 

## redis keys

redis允许用二进制序列作为一个key，从string到JPEG 这样的file都是可以的，另外empty string也是一个有效的key，关于key的几个小贴士：

- key不应该太长，key太长不好记忆，并且lookup的时候在key比较上的开销会很大。
- key不应该太短，太短节省下来的空间相对于可读性来说不那么重要，所以平衡好key的长短比较重要。
- key的最大size为512M
- key应该有一定的规则，比如"object-type:id" is a good idea

## redis list

### overview

redis list是有序的，其基本内部实现是linked-list，也就是说当element数量达到百万级别的时候，在头插和尾插的时候都是常量级别的时间。redis基于linked-list的原因在于，作为`database system`在很长 的lsit中添加数据是要求快速的。另外一个优势就是，在常数级的时间内截取list中的特定长度。

>  个人感觉就是两端可用的栈

###  常用场景

- 社交网站中的user最近的几次update，可以理解，每次lpush后，用`LRANGE 0 9` 来获取最近的10个
- 消费者-生产者模型

### capped list 有限制的列表

在有一些场景下，我们只存储 the latest items。只记住最新的n个items并且discard the oldesd 用`LTRIM` 命令。

`LTRIM`和`LRANGE`命令一样，但是它起到的是一个截取的作用，并且范围之外的elements 会被移除

```shell
➜  ~ redis-cli -h localhost
localhost:6379> RPUSH mylist 1 2 3 4 5
(integer) 5
localhost:6379> LRANGE mylist 0 -1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
localhost:6379> LTRIM mylist 0 2
OK
localhost:6379> LRANGE mylist 0 -1
1) "1"
2) "2"
3) "3"
localhost:6379> LPUSH mylist a
(integer) 4
localhost:6379> LRANGE mylist 0 -1
1) "a"
2) "1"
3) "2"
4) "3"
```

**注意**：这边的``LTRIM``只是暂时性的调整list，而不是永久的将list设置为3个，`LPUSH`一个，右边的一个不会因此被挤掉。

### Blocking operations on list

假设一个生产者和消费者的场景，如果此时list中是NULL，但是消费者仍然从里面`RPOP`结果一定是NULL，消费者此时等待一段时间retry`RPOP`，这种叫做轮询的方式是有一些drawbacks的。

- 明明list中没有数据，但是redis不得不去process这些无用的commands。
- 客户端中，我们正常会间隔一点时间去call `RPOP`，这就造成了许多对redis无用的调用。

So Redis implements commands called [BRPOP](https://redis.io/commands/brpop) and [BLPOP](https://redis.io/commands/blpop) which are versions of [RPOP](https://redis.io/commands/rpop) and [LPOP](https://redis.io/commands/lpop) able to block if the list is empty: they'll return to the caller only when a new element is added to the list, or when a user-specified timeout is reached.

例子1:

```shell
Connection1:
localhost:6379> BRPOP tasks 5
(nil)
(5.02s)
```

这个连接向tasks中pop一个element，并且等待5s，意思是：5s内有element就return element 数据给我，否则返回NULL。

例子2: 假设在规定时间内，有数据插入：

```shell
Connection1:
localhost:6379> BRPOP tasks 10
1) "tasks"
2) "1"
(6.12s)

Connection2: 
localhost:6379> lpush tasks 1
(integer) 1
```

先运行connection1，在阻塞的时候运行connection2，push一条数据。

NOTE: 如果等待时间设置为0代表一直等待，你也可以指定多个list去等待，在这些list中，第一个得到元素的list会将element进行返回。

A few things to note about [BRPOP](https://redis.io/commands/brpop):

- 几个client一起等待一个list，会遵循先来先服务的原则，第一个等待的第一个获得element。
- 返回值：不同于`RPOP`，`BRPOP`返回值是一个包含两个element的array。这个array包含了一对key，value。包含key对原因是，一个client可以等待多个list，是一个一对多的关系。
- If the timeout is reached, NULL is returned.

### Automatic creation and removal of keys

redis会在你想push element的时候去创建这个list，在没有element的时候自动去delte这个list，Streams, Sets, Sorted Sets and Hashes 也是这样的。

