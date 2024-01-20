Redis数据类型指的是value的类型，它有五种：

string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

# String（字符串）

string 是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。

string 类型是二进制安全的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象。

string 类型是 Redis 最基本的数据类型，string 类型的值最大能存储 512MB。

```Shell
redis 127.0.0.1:6379 SET runoob "菜鸟教程"
OK
redis 127.0.0.1:6379 GET runoob
"菜鸟教程"
```

# Hash（哈希）

DEL runoob 用于删除前面测试用过的 key，不然会报错：(error) WRONGTYPE Operation against a key holding the wrong kind of value

```Shell
redis 127.0.0.1:6379 DEL runoob
redis 127.0.0.1:6379 HMSET runoob field1 "Hello" field2 "World"
"OK"
redis 127.0.0.1:6379 HGET runoob field1
"Hello"
redis 127.0.0.1:6379 HGET runoob field2
"World"
```

# List（列表）

Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。

```Shell
redis 127.0.0.1:6379 DEL runoob
redis 127.0.0.1:6379 lpush runoob redis
(integer) 1
redis 127.0.0.1:6379 lpush runoob mongodb
(integer) 2
redis 127.0.0.1:6379 lpush runoob rabbitmq
(integer) 3
redis 127.0.0.1:6379 lrange runoob 0 10
1) "rabbitmq"
2) "mongodb"
3) "redis"
redis 127.0.0.1:6379
```

# Set（集合）

Redis 的 Set 是 string 类型的无序集合。

集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

sadd 命令：添加一个 string 元素到 key 对应的 set 集合中，成功返回 1，如果元素已经在集合中返回 0。

```Shell
redis 127.0.0.1:6379 DEL runoob
redis 127.0.0.1:6379 sadd runoob redis
(integer) 1
redis 127.0.0.1:6379 sadd runoob mongodb
(integer) 1
redis 127.0.0.1:6379 sadd runoob rabbitmq
(integer) 1
redis 127.0.0.1:6379 sadd runoob rabbitmq
(integer) 0
redis 127.0.0.1:6379 smembers runoob

1) "redis"
2) "rabbitmq"
3) "mongodb"
```

以上实例中 rabbitmq 添加了两次，但根据集合内元素的唯一性，第二次插入的元素将被忽略。

# zset(sorted set：有序集合)

Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。

不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。

zset的成员是唯一的,但分数(score)却可以重复。

**zadd 命令：**添加元素到集合，元素在集合中存在则更新对应score，`zadd key score member`

```Shell
redis 127.0.0.1:6379 DEL runoob
redis 127.0.0.1:6379 zadd runoob 0 redis
(integer) 1
redis 127.0.0.1:6379 zadd runoob 0 mongodb
(integer) 1
redis 127.0.0.1:6379 zadd runoob 0 rabbitmq
(integer) 1
redis 127.0.0.1:6379 zadd runoob 0 rabbitmq
(integer) 0
redis 127.0.0.1:6379 ZRANGEBYSCORE runoob 0 1000
1) "mongodb"
2) "rabbitmq"
3) "redis"
```