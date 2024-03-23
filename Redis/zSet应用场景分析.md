## zSet底层数据结构分析
SortedSet是有序集合，集合中每个数据都包含element和score两个值。score是得分，element则是字符串值。SortedSet会根据每个element的score值排序，形成有序集合。
![[Pasted image 20240323103109.png]]
支撑zSort的数据结构：HashTable+Skiplist
- 根据element查询score值，要实现根据element查询对应的score值，就必须实现element与score之间的键值映射，SortedSet底层是基于HashTable来实现的。
- 要按照score值升序或降序查询element，并且查询效率还高，就需要有一种高效的有序数据结构，SortedSet是基于跳表Skiplist实现的。
SkipList（跳表）首先是链表，但与传统链表相比有几点差异：
- 先将跳表中的元素按照升序有序存储
- 节点可包含跨度不同的多级指针，指针跨度不同。
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
# zSet压缩存储
因为SortedSet底层需要用到上述两种数据结构，对内存占用比较高。因此当满足下述两个条件的时候：
1. 集合元素个数小于128个
2. 且每个元素都小于64字节
SortedSet底层会采用压缩列表ZipList来代替HashTable和SkipList，但是ZipList的缺陷也是有的：
- 不能保存过多的元素，否则查询效率就会降低；因此，Redis 对象（List 对象、Hash 对象、Zset 对象）包含的元素数量较少，或者元素值不大的情况才会使用压缩列表作为底层数据结构。
- 新增或修改某个元素时，压缩列表占用的内存空间需要重新分配，甚至可能引发连锁更新的问题。
## ZipList不适合存储大量元素
压缩列表是 Redis 为了节约内存而开发的，它是由连续内存块组成的顺序型数据结构，有点类似于数组。
压缩列表的表头有三个字段
- `_zlbytes_`，记录整个压缩列表占用对内存字节数；
- `_zltail_`，记录压缩列表「尾部」节点距离起始地址由多少字节，也就是列表尾的偏移量；
- `_zllen_`，记录压缩列表包含的节点数量；
- `_zlend_`，标记压缩列表的结束点，固定值 0xFF（十进制255）。
![[Pasted image 20240323105253.png]]
因此在压缩列表中，查找定位第一个元素和最后一个元素，可以通过偏移量直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N) 了，因此压缩列表不适合保存过多的元素。
## ZipList可能会造成连锁更新
压缩列表节点包含三部分内容：
- `_prevlen_`，记录了「前一个节点」的长度，目的是为了实现从后向前遍历；
- `_encoding_`，记录了当前节点实际数据的「类型和长度」，类型主要有两种：字符串和整数。
- `_data_`，记录了当前节点的实际数据，类型和长度都由 `encoding` 决定；
![[Pasted image 20240323105301.png]]

prevlen 属性的空间大小跟前一个节点长度值有关
- 如果前一个节点的长度小于 254 字，那么 prevlen 属性需要用 1 字节的空间来保存这个长度值；
- 如果前一个节点的长度大于等于 254 字节，那么 prevlen 属性需要用 5 字节的空间来保存这个长度值；
encoding 属性的空间大小跟实际数据data的类型有关
- 如果当前节点的数据是 int16 整数，则 encoding 会使用 1 字节的空间进行编码
- 如果当前节点的数据是字符串，根据字符串的长度大小，encoding 会使用 1 字节/2字节/5字节的空间进行编码，encoding 编码的前两个 bit 表示数据的类型，后续的其他 bit 表示字符串数据的实际长度。

上述根据数据大小和类型进行不同的空间大小分配的设计思想，正是 Redis 为了节省内存而采用的。但与此同时压缩列表新增某个元素或修改某个元素时，如果空间不不够，压缩列表占用的内存空间 prevlen 就需要重新分配。而当新插入的元素较大时，可能会导致后续元素的 prevlen 占用空间都发生变化，从而引起「连锁更新」问题，导致每个元素的空间都要重新分配，造成访问压缩列表性能的下降。
现在假设一个压缩列表中有多个连续的、长度在 250～253 之间的节点，如下图：
![[Pasted image 20240323110207.png]]
因为这些节点长度值小于 254 字节，所以 prevlen 属性需要用 1 字节的空间来保存这个长度值。

这时，如果将一个长度大于等于 254 字节的新节点加入到压缩列表的表头节点，即新节点将成为 e1 的前置节点，如下图：
![[Pasted image 20240323110214.png]]
因为 e1 节点的 prevlen 属性只有 1 个字节大小，无法保存新节点的长度，此时就需要对压缩列表的空间重分配操作，并将 e1 节点的 prevlen 属性从原来的 1 字节大小扩展为 5 字节大小。此时 e1 的长度就大于等于 254 了

多米诺牌的效应就此开始。原本 e2 保存 e1 的 prevlen 属性也必须从 1 字节扩展至 5 字节大小。
正如扩展 e1 引发了对 e2 扩展一样，扩展 e2 也会引发对 e3 的扩展，而扩展 e3 又会引发对 e4 的扩展.... 一直持续到结尾。
![[Pasted image 20240323110224.png]]
由于`ZipList`存在连锁更新问题，因此而在Redis7.0版本以后，`ZipList`又被替换为紧凑列表Listpack
## 紧凑列表listpack优化

listpack 采用了压缩列表的很多优秀的设计，比如还是用一块连续的内存空间来紧凑地保存数据，并且为了节省内存的开销，listpack 节点会采用不同的编码方式保存不同大小的数据。

listpack 头包含两个属性，分别记录了 listpack 总字节数和元素数量，然后 listpack 末尾也有个结尾标识。图中的 listpack entry 就是 listpack 的节点了。
![[Pasted image 20240323110548.png]]
每个 listpack 节点结构如下：
- encoding，定义该元素的编码类型，会对不同长度的整数和字符串进行编码；
- data，实际存放的数据；
- len，encoding+data的总长度；
![[Pasted image 20240323110552.png]]


可以看到，listpack 没有压缩列表中记录前一个节点长度的字段了，listpack 只记录当前节点的长度，当我们向 listpack 加入一个新元素的时候，不会影响其他节点的长度字段的变化，从而避免了压缩列表的连锁更新问题。
# 排行榜功能实现
每个直播间都有粉丝的排行榜，可以通过`key+直播间id`来作为redis的key，例如`broadcast:20210108231`。

使用zSet将直播间观众按照点赞数排序。


## 新增操作ZADD
通过`ZADD`新增用户。
```shell
ZADD [key] [score] [value]
```
规定首次进入直播间将score设置为初始值1
```shell
#张三观众进入直播间。
ZADD broadcast:20210108231 1 zhangsan
李四进入直播间
ZADD broadcast:20210108231 1 lisi
#王五进入直播间。
ZADD broadcast:20210108231 1 wangwu
#赵六进入直播间。
ZADD broadcast:20210108231 1 zhaoliu
```
## 加分值ZINCRBY
```shell
ZINCRBY [key] increment [member]
```
李四送了直播间两颗小红心。李四分值加2。
```shell
ZINCRBY broadcast:20210108231 2 lisi
```
## 展示榜单ZRANGE
查看[start,stop]区间排序结果，其中成员的位置按分数值递减来排列。
```shell
ZREVRANGE key start stop [WITHSCORES]
```
通过如上指令对直播间分值进行设置之后，得到redis的value如下：
```shell
127.0.0.1:6379> ZRANGE broadcast:20210108231 0 -1 WITHSCORES
1) "zhaoliu"
2) "2"
3) "wangwu"
4) "5"
5) "lisi"
6) "8"
7) "zhangsan"
8) "10"
```
又如，获取直播间前三名
```shell
127.0.0.1:6379> ZREVRANGE broadcast:20210108231 0 2
1) "zhangsan"
2) "lisi"
3) "wangwu"
```
## 查看直播间人数ZCARD
```shell
ZCARD key 
```
返回集合数量
```shell
127.0.0.1:6379> zcard  broadcast:20210108231
(integer) 4
```
## 离开直播间ZREM
```shell
ZREM [key] [value]
```
张三离开直播间，则删除对应key。
```shell
127.0.0.1:6379>  ZREM broadcast:20210108231 zhangsan
(integer) 1
127.0.0.1:6379>  ZREVRANGE broadcast:20210108231 0 2
1) "lisi"
2) "wangwu"
3) "zhaoliu"
```
## 周榜ZUNIONSTORE
真实场景中肯定会有时间段的划分，例如查看日榜、周榜、月榜。只需要按照最小的时间单位(如天)分成不同的集合，最后求出这些集合的并集即可。
![[Pasted image 20240303155805.png]]
如此，先创建两个zset集合，最后得到两个集合的并集。
```shell
ZADD hotnews:1 10 zhangsan 
ZADD hotnews:1 10 lisi
ZADD hotnews:2 5 zhangsan
ZADD hotnews:2 5 wangwu
```
ZUNIONSTORE语法如下：
```shell
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]
```
[参数设置]
- 两个集合并集时，如果有相同的key，则可以通过 `SUM|MIN|MAX` 进行控制。默认为`SUM`
- `WEIGHTS` 可以设置每个集合的权重，意为在原来集合分数乘权重得到输出集合的值。
合并周一和周二，示例如下：
```shell
127.0.0.1:6379> ZUNIONSTORE hotnews:week:1 2 hotnews:1 hotnews:2
(integer) 3
127.0.0.1:6379> ZRANGE hotnews:week:1 0 -1 WITHSCORES
1) "wangwu"
2) "5"
3) "lisi"
4) "10"
5) "zhangsan"
6) "15"
```

