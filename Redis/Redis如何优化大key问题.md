### 什么是大Key，多大是大Key?

**注意**：Redis中的大key，实际上指的是key所关联的value值特别大，或者是某种数据结构（如hash, set, zset, list）中存储了过多的元素。

详情可参照《阿里Redis开发规范》
![[Pasted image 20240325101745.png]]

一般来讲，String类型控制在10KB以内，hash、list、set、zset元素个数不要超过5000。

### 为什么会产生BigKey？

大key的产生一般与业务方设计有关，对vaule的动态增长问题预估不足。造成大key问题的原因有：
- 数据结构设计不合理。在不适用的场景下使用Redis，易造成Key的value过大，如使用String类型的Key存放大体积二进制文件型数据；
- 业务规划设计不足。没有对Key中的成员进行合理的拆分将大key变成小key，从而造成个别Key中一直往value里面塞数据，没有删除机制，未定期清理无效数据，导致不断增加。
- 上线前期预估不足。如头条重大新闻，造成value值动态突增。如：百度热搜
![[Pasted image 20240325101807.png]]
- 汇总统计类，随着时间推移value逐渐增加

### 产生大Key会有什么问题？

- 内存不足（因为redis基于内存）
- 删除超时
- 网络阻塞
- 集群节点容量倾斜甚至宕机
因此需引起足够重视。
### 如何察觉redis变慢了？

1. Redis 基准性能测试，了解Redis 在生产环境服务器上的基准性能，才能进一步评估，当其延迟达到什么程度时，才认为Redis确实变慢了。例如：按自身硬件配置，可能延迟是`0.5ms` 时就可以认为Redis 变慢了。

执行以下命令`./redis-cli --intrinsic-latency 60  `，测试出这个实例60 秒内的最大响应延迟：
```shell
[root@bogon src]# ./redis-cli --intrinsic-latency 60  
Max latency so far: 1 microseconds.  
Max latency so far: 25 microseconds.  
Max latency so far: 220 microseconds.  
Max latency so far: 253 microseconds.  
Max latency so far: 351 microseconds.  
Max latency so far: 448 microseconds.  
Max latency so far: 514 microseconds.  
  
1706810010 total runs (avg latency: 0.0352 microseconds / 35.15 nanoseconds per run).  
Worst run took 14622x longer than the average latency
```


从输出结果可以看到，这60 秒内的最大响应延迟为514 微秒（0.514 毫秒）。

还可以使用以下命令`Redis-cli -h 127.0.0.1 -p 6379 --latency-history -i 1`，查看一段时间内Redis 的最小、最大、平均访问延迟。如下：redis-cli 每隔1秒向 Redis 服务器发送一个 PING 命令，并测量其往返时间.
```shell
[root@bogon src]# redis-cli -h 127.0.0.1 -p 6379 --latency-history -i 1  
min: 0, max: 1, avg: 0.15 (82 samples) -- 1.01 seconds range  
min: 0, max: 1, avg: 0.06 (80 samples) -- 1.00 seconds range  
min: 0, max: 1, avg: 0.12 (82 samples) -- 1.00 seconds range  
min: 0, max: 1, avg: 0.09 (81 samples) -- 1.01 seconds range  
min: 0, max: 1, avg: 0.07 (82 samples) -- 1.00 seconds range  
min: 0, max: 1, avg: 0.07 (82 samples) -- 1.01 seconds range
```

根据《阿里开发手册》如果你观察到的Redis 运行时延迟是其`基线性能的2倍及以上`，就可以认定Redis变慢了。

2. 使用Redis慢日志

Redis 提供了慢日志命令的统计功能，它记录了有哪些命令在执行时耗时比较久。

例如，设置慢日志的阈值为5毫秒，并且保留最近10条慢日志记录：
```shell
# 命令执行耗时超过 5 毫秒，记录慢日志   
CONFIG SET slowlog-log-slower-than 5000      
# 只保留最近 10 条慢日志   
CONFIG SET slowlog-max-len 10  
```
如果你查询慢日志发现，并不是复杂度过高的命令导致的，而都是SET/DEL这种简单命令出现在慢日志中，那么你就要怀疑你的实例否写入了bigkey。

### 如何定位BigKey？

使用命令`redis-cli --bigkeys`给出每种数据结构最大的bigkey，同时给出每种数据类型的键值个数和平均大小。
```shell
redis-cli --bigkeys  
  
[root@bogon src]# redis-cli --bigkeys  
  
# Scanning the entire keyspace to find biggest keys as well as  
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec  
# per 100 SCAN commands (not usually needed).  
  
[00.00%] Biggest string found so far '"key162116"' with 6 bytes  
[75.91%] Biggest string found so far '"key1000000"' with 7 bytes  
[100.00%] Sampled 1000000 keys so far  
  
-------- summary -------  
  
Sampled 1000000 keys in the keyspace!  
Total key length in bytes is 8888896 (avg len 8.89)  
  
Biggest string found '"key1000000"' has 7 bytes  
  
0 lists with 0 items (00.00% of keys, avg size 0.00)  
0 hashs with 0 fields (00.00% of keys, avg size 0.00)  
1000000 strings with 5888896 bytes (100.00% of keys, avg size 5.89)  
0 streams with 0 entries (00.00% of keys, avg size 0.00)  
0 sets with 0 members (00.00% of keys, avg size 0.00)  
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

> 注意：对线上实例进行bigkey扫描时，Redis 的OPS（Operation Per Second 每秒操作次数）会突增，扫描过程最好控制一下扫描的频率，指定-i 参数，命令：`redis-cli -h 127.0.0.1 -p 6379 --bigkeys -i 1`.它表示扫描过程中每次扫描后休息的时间间隔，单位是秒。

但是，如果想要获得一个 key 和它的值在 RAM 中所占用的字节数。需要使用以下命令：
```shell
redis 127.0.0.1:6379> MEMORY USAGE key [SAMPLES count]
````
例如：
```shell
127.0.0.1:6379> MEMORY usage key1000000   (integer) 56  
```
当我们发现生产的大Key后，那么如何进行删除？

### 如何处理大Key?

我们按照不同数据类型，给出以下命令：

- String类型: `DEL/UNLINK`
    

删除Redis中String类型的大Key，你可以使用`DEL`命令：

```shell
DEL key [key ...]  
```

如果你使用的是Redis的集群模式，可以使用`redis-cli`的`-c`选项来启用集群模式，并执行删除命令。
```shell

redis-cli -c DEL key_name 
```

由于`DEL`命令会对Redis服务器造成阻塞，可以考虑使用`UNLINK`命令。Redis 4.0及以上版本中可用，它会异步地删除Key，避免阻塞。

```shell
`UNLINK key [key ...]   `
```

**注意:** 即使使用`UNLINK`命令，删除非常大的Key仍然可能会对Redis服务器造成一些影响，因为它仍然需要释放内存。因此，在生产环境中执行此类操作时，请务必谨慎，并考虑在低峰时段进行，同时监控Redis的性能指标。

- Hash类型:`HSCAN + HDEL`
```shell
HSCAN key cursor [MATCH pattern] [COUNT count]  
  
        127.0.0.1:6379> HSET myhash field1 value1  
        (integer) 1  
        127.0.0.1:6379> HSET myhash field2 value2  
        (integer) 1  
        127.0.0.1:6379> HSET myhash field3 value3  
        (integer) 1  
        127.0.0.1:6379> HSCAN myhash 0 MATCH * COUNT 10  
        1) "0"  
        2) 1) "field1"  
        2) "value1"  
        3) "field2"  
        4) "value2"  
        5) "field3"  
        6) "value3"  
        127.0.0.1:6379> HDEL myhash field2  
        (integer) 1  
        127.0.0.1:6379> HGETALL myhash  
1) "field1"  
        2) "value1"  
        3) "field3"  
        4) "value3"
```

- List类型:`LTRIM`渐进式删除
```shell
LTRIM key start stop  
  
redis> RPUSH mylist "one"  
        (integer) 1  
redis> RPUSH mylist "two"  
        (integer) 2  
redis> RPUSH mylist "three"  
        (integer) 3  
redis> LTRIM mylist 1 -1  
        "OK"  
redis> LRANGE mylist 0 -1  
        1) "two"  
        2) "three"  
redis>
```

- Set类型:使用`sscan`每次获取部分元素，再使用`srem`命令删除每个元素
    

`127.0.0.1:6379> SADD myset e1 e2 e3    (integer) 0   127.0.0.1:6379> SSCAN myset 1   1) "0"   2) 1) "e3"   127.0.0.1:6379> SMEMBERS myset   1) "e2"   2) "e1"   3) "e3"   127.0.0.1:6379> SREM myset e2   (integer) 1   127.0.0.1:6379> SMEMBERS myset   1) "e1"   2) "e3"   127.0.0.1:6379>` 

- Zset类型：使用`zscan`每次获取部分元素，再使用`ZREM`命令删除每个元素
```shell
127.0.0.1:6379> SADD myset e1 e2 e3  
        (integer) 0  
        127.0.0.1:6379> SSCAN myset 1  
        1) "0"  
        2) 1) "e3"  
        127.0.0.1:6379> SMEMBERS myset  
1) "e2"  
        2) "e1"  
        3) "e3"  
        127.0.0.1:6379> SREM myset e2  
        (integer) 1  
        127.0.0.1:6379> SMEMBERS myset  
1) "e1"  
        2) "e3"  
        127.0.0.1:6379>
```

### 生产BigKey如何调优？

采用惰性删除策略。具体在${redis_home}/redis.conf 文件配置修改
```shell
lazyfree-lazy-server-del yes
replica-lazy-flush yes   
lazyfree-lazy-user-del yes   
```

### 总结

  Redis中的大Key指的是占用内存特别大的Key，处理不当可能导致性能下降、内存消耗大等问题。

**解决方案**：

- `避免创建大Key`：设计数据结构时，尽量分散数据，避免单一Key过大。
    
- `分批次处理`：对于已存在的大Key，使用相关命令（如SCAN）分批次读取和删除。
    
- `设置过期时间`：为大Key设置TTL，让Redis自动清理。
    
- `监控与告警`：使用监控工具及时发现大Key，并设置告警通知。
    
- `优化网络`：如果删除大Key时网络压力大，考虑增加带宽或优化网络连接。
    

**注意事项**：

- 处理大Key时要谨慎，最好在低峰时段操作。