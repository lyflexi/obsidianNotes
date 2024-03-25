# 表锁-lock tables
顾名思义，就是直接对表进行加锁，可以使用下面命令：
```sql
--加读锁  
lock tables table_name read;  
--加写锁  
lock tables table_name write;  
-- 释放当前会话的所有表锁  
unlock tables
```
如果加的是写锁，当对表进行写操作时也会被阻塞，直到写锁被释放。
不过尽量避免在使用 InnoDB 引擎的表使用表锁，因为表锁的颗粒度太大，会影响并发性能。
# 行锁-for update
行锁又称记录锁，对表中的记录加锁，记录锁属于排他锁X，加锁脚本是`FOR UPDATE`，==但前提是查询条件加了索引，否则就直接锁表了!==
特别的，如果给id=2这条记录加锁行锁也是可以生效的，因为这把锁是加在主键索引(聚簇索引)上的
```sql
-- 记录锁在 id=1 的记录上加上排他锁，以阻止其他事务插入，更新，删除 id=1 这一行。
SELECT * FROM `test` WHERE `id`=1 FOR UPDATE; 
```
另外对于update形如total_stocks = total_stocks - 1操作，MySQL会自动给涉及数据集加排它锁X，可以省略for update

但是update的查询条件也必须给查询条件加索引，否则也会退化为表锁
```sql
-- 对于update、delete、insert语句MySQL会自动给涉及数据集加排它行锁X，
update  goods set total_stocks = total_stocks - 1 ,update_time = now() where id = #{value} and total_stocks - 1 >= 0
```

# 临键锁，特指普通索引
Next-Key Lock是用来解决幻读问题的，记录锁和间隙锁的组合，注意临键锁只在查询条件为普通索引列时候才生效，在主键列和唯一索引列上不存在临键锁。
- 主键列不存在临键锁
- 唯一索引列不存在临键锁
下面举例，下表结构中age字段为普通索引
```sql
insert into sys_user (id, name, name_pinyin, id_card, phone, age)values (1, '小六', 'xiaoliu', 300000000, 13000008000, 10);
insert into sys_user (id, name, name_pinyin, id_card, phone, age)values (2, '小六', 'xiaoliu', 300000001, 13000008000, 11);
insert into sys_user (id, name, name_pinyin, id_card, phone, age)values (3, '小六', 'xiaoliu', 300000002, 13000008000, 13);
insert into sys_user (id, name, name_pinyin, id_card, phone, age)values (4, '小六', 'xiaoliu', 300000003, 13000008000, 20);
```
Next-key Lock 临键锁是锁的范围是左开右闭区间的数据（即在某条记录以及这条记录前面间隙上的锁），因此存在临键区间为
-  (负无穷大，10]
- (10, 11]
- (11, 13]
- (13, 20]
- (20, 正无穷大]

举例测试区间(负无穷大，10]：
```sql
-- 事务A:  
begin;
select * from sys_user where age=10 for update;
```
事务B间隙锁范围内插入被阻塞，无法被插入
```sql
begin;
insert into sys_user (id, name, name_pinyin, id_card, phone, age)values (5, '小六', 'xiaoliu', 300000004, 13000008000, 9);
```
事务B间隙锁范围之外插入正常插入，无阻塞
```sql
begin;
insert into sys_user (id, name, name_pinyin, id_card, phone, age)values (5, '小六', 'xiaoliu', 300000004, 13000008000, 11);
```


# AUTO-INC锁

字面意思是用来控制自动自增的锁？

是的，一般来说我们会在表中设置一个字段声明 AUTO_INCREMENT 的自增ID字段。

> AUTO-INC锁在自增字段起了个什么作用呢？

当使用INSERT语句插入一条新记录时，MySQL会自动为自增字段加锁，防止其他并发的插入操作同时获取相同的自增值。

其他事务要等待，直到执行完插入语句之后才会释放锁。

这就保证了数据表的 AUTO_INCREMENT 字段的值是连续递增。

好吧，原来这个AUTO_INC锁的作用是这样的，以前我还一直不知道呢！
# 共享锁
除了for update排他锁，MySQL还是先了共享锁lock in share mode，对于共享锁而言，对当前行加共享锁，不会阻塞其他事务对同一行的读请求，但会阻塞对同一行的写请求。只有当读锁释放后，才会执行其它写操作。
```sql
-- 加共享行锁（S）
select * from table_name where ... lock in share mode
```