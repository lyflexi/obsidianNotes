Redis一般我们会把它用作于缓存，当然啦，日常有的应用场景比较简单，用个HashMap也能解决很多的问题了，没必要上Redis
# 实时消息打点
总体来讲消息链路追踪分为三个步骤：
1. 实时消息存储
2. 业务标识模板
3. 定义打点规则

我这边负责消息管理平台，简单来说就是发消息的，其中实时的数据我们就用Redis来进行存储（离线的数据我们是存储到MySQL或者Hive的）
发完消息后我们需要知道消息有没有下发成功，于是我们系统有一套针对Redis完整的链路追踪体系
## 业务标识

对消息进行实时链路追踪，我这边就用了Redis好几种的数据结构分别有Set、List和Hash
要在消息管理平台发消息，首先得在后台新建一个「模板」，有模板自然会有一个模板ID，再对模板ID进行扩展比如说加上日期和固定的业务参数，形成的ID可以唯一标识某个模板的下发链路，在系统上，我这边叫它为UMPID：Unified Multi-Purpose Identification唯一的多用途的身份ID
## 打点规则

在发送入口处会对所有需要下发的消息打上模板ID：UMPID，然后根据不同维度进行处理。比如说：

- Set结构`[key日期--Set<value>]`：我要看某一天下发的所有模板有哪些，就以key为日期，将对应UMPID扔到Set就好了
- List结构`[key用户--LinkList<value>]`：我要看某个用户当天下发的消息整体链路是如何。用的是List结构，Key是userId，Value则是UMPID+state(关键点位)+processTime（处理时间)
- Hash结构`[key模板--Hash<key状态--value>]`：我要看某一个模板的消息下发的整体链路情况，那我以UMPID为Key，Value是Hash结构，Key是state，Value则是人数，这里的state我们在下发的过程中打的关键点位，比如接收到消息打个51，消息被去重了打个61，消息成功下发了打个81…

# 排行榜功能实现
数据结构选取zset。
每个直播间都有粉丝的排行榜，可以通过`key+直播间id`来作为redis的key。例如`broadcast:20210108231`。每个直播间的观众按照点赞数排序。则观众刚刚进入直播间即可通过`ZADD`添加排行榜。
## 新增操作ZADD
```shell
ZADD [key] [score] [value]
```
张三观众进入直播间。
李四进入直播间
```shell
ZADD broadcast:20210108231 1 zhangsan
ZADD broadcast:20210108231 1 lisi
```
王五进入直播间。
赵六进入直播间。
```shell
ZADD broadcast:20210108231 1 wangwu
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
获取直播间前三名进行展示，按照分值排序
```shell
ZREVRANGE key start stop [WITHSCORES]
```
ZREVRANGE 命令返回有序集合，指定区间内的成员。其中成员的位置按分数值递减(从大到小)来排列。具有相同分数值的成员按字典序的逆序(reverse lexicographical order)排列。
```shell
127.0.0.1:6379> ZREVRANGE broadcast:20210108231 0 2
1) "zhangsan"
2) "lisi"
3) "wangwu"
```

## 查看直播间人数
```shell
ZCARD key 
```
返回集合数量
```shell
127.0.0.1:6379> zcard  broadcast:20210108231
(integer) 4
```
## 离开直播间
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
## 周榜
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
- 两个集合并集时，如果有相同的key，则可以通过 `SUM|MIN|MAX` 进行控制。默认为`SUM`
- 额外的，`WEIGHTS` 可以设置每个集合的权重，意为在原来集合分数乘权重得到输出集合的值。  
示例如下：
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

