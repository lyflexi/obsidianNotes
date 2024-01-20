从字面意义上直接理解，`redo`是重做，`undo`是撤销，`binlog`是二进制日志。

从MySQL体系结构上来看redolog和undolog都是innodb引擎层的日志，而binlog是server层的日志
![[Pasted image 20240118104741.png]]

# redolog和 undolog （引擎层日志）

比如某一时刻数据宕机了，有两个事务，一个事务已经提交，另一个事务正在处理。数据库重启的时候就要根据日志进行前滚`redo`及回滚`undo`：
- 把已提交事务的更改写到数据文件
- 把未提交事务的更改恢复到事务开始前的状态。

具体步骤如下：
通过 redolog 将所有已经在存储引擎内部提交的事务进行恢复，
存储引擎内部所有已经prepared但是没有commit的事务将会应用undolog做 rollback。

# binlog 和 relaylog（server层日志）

binlog记录了数据库所有DDL和DML操作并以二进制形式存储在磁盘中，binlog不包含SELECT和SHOW等命令，因为查询操作对数据本身并没有修改

默认情况下，二进制日志功能是关闭的。可以通过`SHOW VARIABLES LIKE 'log_bin';`命令查看二进制日志是否开启：

```SQL
mysql> SHOW VARIABLES LIKE 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | OFF   |
+---------------+-------+
```

bin log也被叫做归档日志，会一直记录日志。一个binlog日志文件默认最大容量1G，可以通过`max_binlog_size`参数修改，单个日志超过最大值，则会新创建一个新文件继续写。

通过`show binary logs;`命令来查看binlog日志

```SQL
mysql> show binary logs;
+-----------------+-----------+
| Log_name        | File_size |
+-----------------+-----------+
| mysq-bin.000001 |      8687 |
| mysq-bin.000002 |      1445 |
| mysq-bin.000003 |      3966 |
| mysq-bin.000004 |       177 |
| mysq-bin.000005 |      6405 |
| mysq-bin.000006 |       177 |
| mysq-bin.000007 |       154 |
| mysq-bin.000008 |       154 |
```

一般来说开启bin log都会给日志文件设置过期时间`expire_logs_days`，要不然日志的体量会非常庞大

```SQL
mysql> show variables like 'expire_logs_days';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 0     |
+------------------+-------+
1 row in set
 
mysql> SET GLOBAL expire_logs_days=30;
Query OK, 0 rows affected
```

## MySQL主从同步模式

binlog主要应用于MySQL主从同步模式`master-slave`

- 用户在主库master执行DDL和DML操作，修改记录顺序写入binlog;
- 从库slave的I/O线程连接上Master，检查到主库产生了binlog，于是请求读取指定位置position及之后的日志内容;
- Master收到从库slave请求后，将主库binlog文件的名称以及日志中的指定位置position推送给从库;
- slave的I/O线程接收到数据后，将接收到的日志内容依次写入到中继日志`relaylog`文件最末端，并将读取到的主库binlog文件名和位置position记录写到`master-info`文件中，以便在下一次读取用;
- slave的SQL线程检测到relay log中内容更新后，读取日志并解析成可执行的SQL语句，这样就实现了主从库的数据一致;

![[Pasted image 20240118105126.png]]

## binlog 记录格式
binlog有三种记录格式，分别是ROW、STATEMENT、MIXED
- ROW：基于变更的数据行进行记录，如果一个update语句修改一百行数据，那么这种模式下就会记录100行对应的记录日志。
- STATEMENT：基于SQL语句级别的记录日志，相对于ROW模式，STATEMENT模式下只会记录这一个update 的语句。所以此模式下会非常节省日志空间，也避免着大量的IO操作。
- MIXED：混合模式，此模式是ROW模式和STATEMENT模式的混合体，一般的语句修改使用STATEMENT格式保存binlog，==当遇到一些函数时STATEMENT无法完成主从复制的操作，则会切换为ROW格式来保存binlog==
    
这三种模式需要注意的是，使用 row 格式的 binlog 时，在进行数据同步或恢复的时候不一致的问题更容易被发现，因为它是基于数据行记录的。而使用mixed或者statement格式的binlog时，很多事务操作都是基于SQL逻辑记录，一个SQL在不同的时间点执行所产生的数据变化和影响是不一样的，所以这种情况下，数据同步或恢复的时候就容易出现不一致的情况。