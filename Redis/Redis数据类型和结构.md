# 数据类型
Redis数据类型指的是value的类型，它有五种：

string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

还有一些高级数据类型，比如Bitmap、HyperLogLog、GEO等，其底层都是基于上述5种基本数据类型。因此在Redis的源码中，其实只有5种数据类型。

## String（字符串）

string 是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。

string 类型是二进制安全的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象。

string 类型是 Redis 最基本的数据类型，string 类型的值最大能存储 512MB。

```Shell
redis 127.0.0.1:6379 SET runoob "菜鸟教程"
OK
redis 127.0.0.1:6379 GET runoob
"菜鸟教程"
```

## Hash（哈希）

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

## List（列表）

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

## Set（集合）

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

## zset(sorted set：有序集合+score)

Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。

不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。

zset的成员是唯一的，但分数(score)却可以重复。

zadd 命令：添加元素到集合，元素在集合中存在则更新对应score，`zadd key score member`

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

# 数据结构
## RedisObject

不管是任何一种数据类型，最终都会封装为RedisObject格式，它是一种结构体，C语言中的一种结构，可以理解为Java中的类。

结构大概是这样的：
![[Pasted image 20240127134744.png]]
可以看到整个结构体中并不包含真实的数据，仅仅是对象头信息，内存占用的大小为4+4+24+32+64 = 128bit，也就是16字节，所以RedisObject的内存开销是很大的。

==其中的指针`ptr`指针指向的才是真实数据存储的内存地址。==

属性中的`encoding`就是当前对象底层采用的编码方式，不同的编码方式意味着不同的数据结构，可选的有11种之多：

| **编号** | **编码方式** | **说明** |
|---|---|---|
|0|OBJ_ENCODING_RAW|raw编码动态字符串|
|1|OBJ_ENCODING_INT|long类型的整数的字符串|
|2|OBJ_ENCODING_HT|==hashTable表（也叫dict）== |
|3|OBJ_ENCODING_ZIPMAP|已废弃|
|4|OBJ_ENCODING_LINKEDLIST|双端链表|
|5|OBJ_ENCODING_ZIPLIST|压缩列表|
|6|OBJ_ENCODING_INTSET|整数集合|
|7|OBJ_ENCODING_SKIPLIST|==跳表== |
|8|OBJ_ENCODING_EMBSTR|embstr编码的动态字符串|
|9|OBJ_ENCODING_QUICKLIST|快速列表|
|10|OBJ_ENCODING_STREAM|Stream流|
|11|OBJ_ENCODING_LISTPACK|紧凑列表|

Redis中的5种不同的数据类型采用的底层数据结构和编码方式如下：

|**数据类型** | **编码方式** |
|---|---|
|STRING|`int`、`embstr`、`raw`|
|LIST|`LinkedList和ZipList`(3.2以前)、`QuickList`（3.2以后）|
|SET|`intset`、`HT`|
|ZSET|`ZipList`（7.0以前）、`Listpack`（7.0以后）、`HT`、`SkipList`|
|HASH|`ZipList`（7.0以前）、`Listpack`（7.0以后）、`HT`|
## SkipList升序排列

SkipList（跳表）首先是链表，但与传统链表相比有几点差异：
- ==跳表中的元素按照升序有序存储==
- 节点可能包含多个指针，指针跨度不同。

传统链表只有指向前后元素的指针，因此只能顺序依次访问。如果查找的元素在链表中间，查询的效率会比较低。而SkipList则不同，它内部包含跨度不同的多级指针，可以让我们跳跃查找链表中间的元素，效率非常高。

其结构如图：
![[Pasted image 20240127135014.png]]

  我们可以看到1号元素就有指向3、5、10的多个指针，查询时就可以跳跃查找。例如我们要找大小为14的元素，查找的流程是这样的：
  ![[Pasted image 20240127135039.png]]
- 首先找元素1节点最高级指针，也就是4级指针，起始元素大小为1，指针跨度为9，可以判断出目标元素大小为10。由于14比10大，肯定要从10这个元素向下接着找。
- 找到10这个元素，发现10这个元素的最高级指针跨度为5，判断出目标元素大小为15，大于14，需要判断下级指针
- 10这个元素的2级指针跨度为3，判断出目标元素为13，小于14，因此要基于元素13接着找
- 13这个元素最高级级指针跨度为2，判断出目标元素为15，比14大，需要判断下级指针。
- 13的下级指针跨度为1，因此目标元素是14，刚好于目标一致，找到。

这种多级指针的查询方式就避免了传统链表的逐个遍历导致的查询效率下降问题。==在对有序数据==做随机查询和排序时效率非常高。

跳表SkipList的结构体如下：
可以看到SkipList主要属性是header和tail，也就是头尾指针，因此它是支持双向遍历的。
```C
typedef struct zskiplist {
    // 头尾节点指针
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 最大的索引层级
    int level;
} zskiplist;
```

跳表中zskiplistNode节点的结构体如下：
```C
typedef struct zskiplistNode {
    sds ele; // 节点存储的字符串
    double score;// 节点分数，排序、查找用
    struct zskiplistNode *backward; // 前一个节点指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 下一个节点指针
        unsigned long span; // 索引跨度
    } level[]; // 多级索引数组
} zskiplistNode;
```
## SortedSet数据结构分析
**面试题**：Redis的`SortedSet`底层的数据结构是怎样的？Redis源码中`zset`，也就是`SortedSet`的结构体如下：

**答**：SortedSet是有序集合，底层的存储的每个数据都包含element和score两个值。score是得分，element则是字符串值。SortedSet会根据每个element的score值排序，形成有序集合。

根据score的排序原理如下：
- 根据element查询score值，要实现根据element查询对应的score值，就必须实现element与score之间的键值映射，SortedSet底层是基于==HashTable==来实现的。
- 要按照score值升序或降序查询element，并且查询效率还高，就需要有一种高效的有序数据结构，SortedSet是基于==跳表Skiplist==实现的。

加分项：因为SortedSet底层需要用到两种数据结构，对内存占用比较高。因此Redis底层会对SortedSet中的元素大小做判断。如果元素大小小于128且每个元素都小于64字节，SortedSet底层会采用ZipList，也就是压缩列表来代替HashTable**和**SkipList

不过，`ZipList`存在连锁更新问题，因此而在Redis7.0版本以后，`ZipList`又被替换为Listpack（紧凑列表）。