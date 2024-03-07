Redis现如今使用的场景越来越多？如何批量删除key呢？

有人说用`KEYS`命令，刚开始学Redis的时候就是用这个命令列出库中键。

# KEYS 命令

> Warning: consider KEYS as a command that should only be used in production environments with extreme care. It may ruin performance when it is executed against large databases. This command is intended for debugging and special operations, such as changing your keyspace layout. Don't use KEYS in your regular application code. If you're looking for a way to find keys in a subset of your keyspace, consider using sets.

上面是官方文档声明，`KEYS`命令不能用在生产的环境中，这个时候如果数量过大效率是十分低的。同时也不要用KEYS正则匹配，官方建议直接用集合类型。

`KEYS`相当于关系性数据的库的 `select *`，在生产环境几乎是要禁用的。
- `KEYS`命令的性能随着数据库数据的增多而越来越慢
- `KEYS`命令会引起阻塞，连续的 `KEYS`命令足以让 Redis 阻塞

然而，网上很多都是这么写的 `redis-cli --raw keys "key前缀*" | xargs redis-cli del`，千万别照炒，拿到生产环境上做实验。
其中xargs命令是给其他命令传递参数的一个过滤器，也是组合多个命令的一个工具。它擅长将标准输入数据转换成命令行参数

# SCAN 命令

Redis从2.8版本开始支持scan命令，SCAN命令的基本用法如下：
- 复杂度虽然也是 O(n)，通过游标分步进行不会阻塞线程;
- 同 keys命令 一样提供模式匹配功能;
- 服务器不需要为游标保存状态，游标的唯一状态就是 scan 返回给客户端的游标整数;
## scan用法
scan 命令提供[三个参数]：
```shell
SCAN cursor [MATCH pattern] [COUNT count]
```
- 第一个参数是cursor
- 第二个是要匹配的正则
- 第三个是单次遍历的槽位
返回值分为两个部分：
- 1）代表下一次迭代的游标，
- 2）代表本次迭代的结果集  
第一个遍历时候 cursor 值为0，然后将返回结果的第一个整数作为下一个遍历的游标，注意如果返回游标为0就代表全部匹配完成。
```bash
127.0.0.1:6379> scan 0 MATCH tony* 
1) "42"
2)  1) "tony25"
    2) "tony2519"
    3) "tony2529"
    4) "tony2510"
    5) "tony2523"
    6) "tony255"
    7) "tony2514"
    8) "tony256"
    9) "tony2511"
   10) "tony15"
127.0.0.1:6379> scan 42 MATCH tony* COUNT 1000
1) "0"
2)  1) "tony3513"
    2) "tony359"
    3) "tony4521"
    4) "tony356"
    5) "tony30"
    6) "tony320"
    7) "tony3"
    8) "tony312"
```

因为KEYS命令的时间复杂度为O(n)，而SCAN命令会将遍历操作分解成m次，然后每次去执行，从而时间复杂度为O(1)。也解决使用keys命令遍历大量数据而导致Redis服务器阻塞的情况。所以建议使用下边的指令进行批量的删除操作：
```shell
`redis-cli --scan --pattern "key前缀*" | xargs -L 1000 redis-cli del`
```



因为Redis是单线程的`KEYS`在某种情况下会阻塞。有个真实真案件小哥哥生产用KEYS，最终导致服务宕机。后果很严重，产生的经济损失就不说了。

切记严重会导致程序的雪崩，删除的时候用`SCAN`命令，看完这篇文章应该都记住了。