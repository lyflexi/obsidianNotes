# 常规查询
语法如下：
```sql
/* SELECT */ ------------------
SELECT [ALL|DISTINCT] select_expr FROM -> WHERE -> GROUP BY [合计函数] -> HAVING -> ORDER BY -> LIMIT
```
## select_expr
```sql
a. select_expr
    -- 可以用 * 表示所有字段。
        select * from tb;
    -- 可以使用表达式（计算公式、函数调用、字段也是个表达式）
        select stu, 29+25, now() from tb;
    -- 可以为每个列使用别名。适用于简化列标识，避免多个列标识符重复。
        - 使用 as 关键字，也可省略 as.
        select stu+10 as add10 from tb;
```
## FROM 子句
```sql
b. FROM 子句
    用于标识查询来源。
    -- 可以为表起别名。使用as关键字。
        SELECT * FROM tb1 AS tt, tb2 AS bb;
    -- from子句后，可以同时出现多个表。
        -- 多个表会横向叠加到一起，而数据会形成一个笛卡尔积。
        SELECT * FROM tb1, tb2;
    -- 向优化符提示如何选择索引
        USE INDEX、IGNORE INDEX、FORCE INDEX
        SELECT * FROM table1 USE INDEX (key1,key2) WHERE key1=1 AND key2=2 AND key3=3;
        SELECT * FROM table1 IGNORE INDEX (key3) WHERE key1=1 AND key2=2 AND key3=3;
```

## WHERE 子句
```sql
c. WHERE 子句
    -- 从from获得的数据源中进行筛选。
    -- 整型1表示真，0表示假。
    -- 表达式由运算符和运算数组成。
        -- 运算数：变量（字段）、值、函数返回值
        -- 运算符：
            =, <=>, <>, !=, <=, <, >=, >, !, &&, ||,其中<=>与<>功能相同，<=>可用于null比较
            in (not) null, (not) like, (not) in, (not) between and, is (not), and, or, exist
            is/is not 加上ture/false/unknown，检验某个值的真假
```
## GROUP BY 子句，分组子句
```sql
d. GROUP BY 子句, 分组子句
    GROUP BY 字段/别名 [排序方式]
    分组后会进行排序。升序：ASC，降序：DESC
```
以下合计函数需配合 GROUP BY 使用：
```sql
    count 返回不同的非NULL值数目  count(*)、count(字段)
    sum 求和
    max 求最大值
    min 求最小值
    avg 求平均值
    group_concat 返回带有来自一个组的连接的非NULL值的字符串结果。组内字符串连接。
```
## HAVING 子句，条件子句
SQL标准要求HAVING必须引用GROUP BY子句中的列或用于合计函数中的列。
```sql
e. HAVING 子句，条件子句
    与 where 功能、用法相同，执行时机不同。
    where 在开始时执行检测数据，对原数据进行过滤。
    having 对筛选出的结果再次进行过滤。
    having 字段必须是查询出来的，where 字段必须是数据表存在的。
    where 不可以使用字段的别名，having 可以。因为执行WHERE代码时，可能尚未确定列值。
    where 不可以使用合计函数。一般需用合计函数才会用 having

```
## ORDER BY 子句，排序子句
```sql
f. ORDER BY 子句，排序子句
    order by 排序字段/别名 排序方式 [,排序字段/别名 排序方式]...
    升序：ASC，降序：DESC
    支持多个字段的排序。
```
## LIMIT 子句，限制结果数量子句
```sql
g. LIMIT 子句，限制结果数量子句
    仅对处理好的结果进行数量限制。将处理好的结果的看作是一个集合，按照记录出现的顺序，索引从0开始。
    limit 起始位置, 获取条数
    省略第一个参数，表示从索引0开始。limit 获取条数
```
## DISTINCT选项

```sql
 
h. DISTINCT, ALL 选项
    distinct 去除重复记录
    默认为 all, 全部记录

```

# 子查询-重要！
子查询需用括号包裹。
## from型
```sql
-- from型
    select * from (select * from tb where id>0) as subfrom where id>1;
```
## where型
```sql
-- where型
    select * from tb where money = (select max(money) from tb);
    -- 列子查询
        子查询结果返回的是一列
        in 或 not in 完成查询
        exists 和 not exists 条件，如果成立则返回true不成立则返回false，常用于判断条件
    -- 行子查询
        查询条件是一个行。
        select * from t1 where (id, gender) in (select id, gender from t2);
        行构造符：(col1, col2, ...) 或 ROW(col1, col2, ...)
        行构造符通常用于与对能返回两个或两个以上列的子查询进行比较。
```
## in和exist对比

==exist需要在子查询中额外的判断一次where，因此exist不够直观，但有的时候使用exist可以优化sql查询==
- "IN" 子句用于在 WHERE 条件中指定多个可能的值。它允许你指定一个值列表，并检查列中的值是否匹配列表中的任何一个值。
语法示例：
```sql
SELECT column1, column2, ...  FROM table_name  WHERE column_name IN (value1, value2, ...);
```
==类似于以下Java代码：复杂度为N^2==
```java
```csharp
List resultSet={};
Array A=(select * from A);
//需要缓存B的所有结果
Array B=(select id from B);

for(int i=0;i<A.length;i++) {
  for(int j=0;j<B.length;j++) {
      if(A[i].id==B[j].id) {
        resultSet.add(A[i]);
        break;
      }
  }
}
return resultSet;
```
在这个例子中，`column_name` 是要检查的列名，`value1`, `value2`, ... 是可能的值列表。如果列中的值匹配列表中的任何一个值，则该行将被包含在结果集中。
- "EXISTS" 子句用于测试子查询是否返回任何结果。它用于确定是否存在至少一行与外部查询匹配的行
语法示例：
```sql
SELECT column1, column2, ...  FROM table_name1  WHERE EXISTS (SELECT column_name FROM table_name2 WHERE condition);
```
在这个例子中，`table_name1` 是外部查询的表名，`table_name2` 是子查询的表名，`column_name` 是要检查的列名，`condition` 是子查询中的条件。如果子查询返回至少一行结果，则 "EXISTS" 子句将返回 true，外部查询将返回匹配的结果。
因此，“EXISTS”一般是这么用的：
```sql
SELECT column1, column2, ...  FROM table_name1  WHERE EXISTS (SELECT 1 FROM table_name2 WHERE condition);
```
只需要确认子查询返回至少一行结果，而不需要子查询获取实际所有结果，这有助于减少不必要的数据处理和传输。类似于以下Java代码：复杂度为N
```java
```csharp
List resultSet={};
Array A=(select * from A);

//不需要缓存B的结果
for(int i=0;i<A.length;i++) {
   if(exists(A[i].id) {  //执行select 1 from B where B.id=A.id是否有记录返回
       resultSet.add(A[i]);
   }
}
return resultSet;
```
# 连接查询(join)

默认是内连接，可省略inner
```sql
-- 内连接(inner join)
    - 默认就是内连接，可省略inner。
    - 只有数据存在时才能发送连接。即连接结果不会出现空行。
    on 表示连接条件。其条件表达式与where类似。也可以省略条件（表示条件永远为真）
    也可用where表示连接条件。
    还有 using, 但需字段名相同。 using(字段名)
    -- 交叉连接 cross join
        即，没有条件的内连接。
        select * from tb1 cross join tb2;
-- 外连接(outer join)
    - 如果数据不存在，也会出现在连接结果中。
    -- 左外连接 left join
        如果数据不存在，左表记录会出现，而右表为null填充
    -- 右外连接 right join
        如果数据不存在，右表记录会出现，而左表为null填充
-- 自然连接(natural join)
    自动判断连接条件完成连接。
    相当于省略了using，会自动查找相同字段名。
    natural join
    natural left join
    natural right join
```
## 内连接默认

内连接，AB两表均不返回空

```sql

SELECT * FROM t_emp a INNER JOIN t_dept b ON a.deptId = b.id 
```
![[Pasted image 20240118093609.png]]
## 左外连接
左外连接，A全集展示，B没有则值显示为空

```sql
-- A的全集，B没有则值显示为空
SELECT *  FROM t_emp a LEFT JOIN t_dept b ON a.deptId = b.id
```
![[Pasted image 20240118093616.png]]
## 右外连接
右外连接，B全集展示，A没有则值显示为空

```sql
SELECT * FROM t_emp a RIGHT JOIN t_dept b ON a.deptId = b.id
```
![[Pasted image 20240118093623.png]]

# 备份与还原
备份，是指将数据的结构与表内数据保存起来。
利用 mysqldump（导出） 指令和source（导入）指令完成。
导出脚本格式如下：
```sql
-- 导出表
mysqldump [options] db_name [tables]
-- 导出库
mysqldump [options] ---database DB1 [DB2 DB3...]
mysqldump [options] --all--database
```
导出示例：
```sql
/* 备份与还原 */ ------------------
1. 导出一张表
　　mysqldump -u用户名 -p密码 库名 表名 > 文件名(D:/a.sql)
2. 导出多张表
　　mysqldump -u用户名 -p密码 库名 表1 表2 表3 > 文件名(D:/a.sql)
3. 导出所有表
　　mysqldump -u用户名 -p密码 库名 > 文件名(D:/a.sql)
4. 导出一个库
　　mysqldump -u用户名 -p密码 --lock-all-tables --database 库名 > 文件名(D:/a.sql)
可以-w携带WHERE条件
```
导入脚本：
```sql
-- 导入
1. 在登录mysql的情况下：
　　source  备份文件
2. 在不登录的情况下
　　mysql -u用户名 -p密码 库名 < 备份文件
```

# 删除操作

在MySQL中有三种删除操作，分别是delete、truncate、drop
上述三种删除操作属于不同的数据库语言
- `delete`语句是DML (数据库操作语言)语句，这个操作会放到`rollback segement`中，事务提交之后才生效
- `truncate`和`drop`属于 DDL(数据定义语言)语句，操作立即生效，原数据不放到`rollback segment`中不能回滚，操作不触发trigger
## delete
```sql
-- delete，用于删除某一行的数据，如果不加where子句则等价于truncate
delete from table where columnNmme=value
```
## truncate和drop
```sql
-- truncate（清空表）， 用于清空表中数据，重置auto_increment的值，新的插入主键将会从1开始
truncate table
-- drop，用于将数据以及表结构全部删除，整个表将不复存在
drop table
```

# SQL优化（慢查询优化）

## 开启慢查询日志slow_query_log
慢查询日志是MySQL提供的一种日志记录，用于记录在MySQL中响应时间超过阀值的语句。具体来说，慢查询日志会记录那些运行时间超过long_query_time值的SQL语句，其中long_query_time的默认值为10秒。

开启慢查询日志需要手动设置，因为默认情况下MySQL的慢查询日志是禁用的，因为开启慢查询日志会或多或少地带来一定的性能影响，它需要记录每个超时的SQL语句。要开启慢查询日志，可以通过设置slow_query_log的值来开启，例如使用以下命令：

```shell
SET GLOBAL slow_query_log = 1;
```

另外，还可以通过修改my.cnf文件来永久生效，例如在mysqld下添加以下行：

```sql
slow_query_log = 1  
slow_query_log_file = /var/lib/mysql/node-slow.log
```

其中，slow_query_log_file指定了慢查询日志文件的存储路径。

开启慢查询日志后，可以通过查看慢查询日志来分析哪些SQL语句超出了最大忍耐时间值，从而进行针对性的优化。

制造慢查询并执行。如下。

```sql
mysql> select sleep(1);
+----------+
| sleep(1) |
+----------+
|        0 |
+----------+
1 row in set (1.00 sec)
```

打开慢查询日志文件。可以看到上述慢查询的SQL语句被记录到这个特殊的日志中。

```sql
# Time: 180620 17:13:06
# User@Host: apsara[apsara] @ dc1487859883577.et2sqa [11.239.51.96]  Id:     3
# Query_time: 1.000246  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
SET timestamp=1529485986;
select sleep(1);
```
## 定位慢查询SQL语句
在生产环境中，如果要手工分析日志，查找、分析SQL，显然是个体力活。==因此除了逐行分析慢查询日志之外，MySQL还自带了分析慢查询的工具mysqldumpslow，该工具是Perl脚本。==
[常用参数]如下。
```shell
-s：排序方式，值如下
    c：查询次数
    t：查询时间
    l：锁定时间
    r：返回记录
    ac：平均查询次数
    al：平均锁定时间
    ar：平均返回记录书
    at：平均查询时间
-t：top N查询
-g：正则表达式
```

使用示例如下：
比如我们执行了多次类似如下的查询。

```sql
select * from db_user where name like 'zb%';
select * from db_user where name like 'aaa%';
select * from db_user where name like 'bc%';
......
```

通过命令mysqldumpslow -s c -t 5 /var/lib/mysql/slow-query.log获取访问次数最多的5个SQL语句

```shell
Reading mysql slow query log from /var/lib/mysql/slow-query.log
Count: 15  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=0.0 (0), apsara[apsara]@dc1487859883577.et2sqa
  # Query_time: N.N  Lock_time: N.N Rows_sent: N  Rows_examined: N
  SET timestamp=N;
  select * from db_user where name like 'S'

Count: 1  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=0.0 (0), apsara[apsara]@dc1487859883577.et2sqa
  # Query_time: N.N  Lock_time: N.N Rows_sent: N  Rows_examined: N
  use test;
  SET timestamp=N;
  select * from db_user where name like 'S'
```

通过命令mysqldumpslow -s t -t 5 /var/lib/mysql/slow-query.log按照时间排的top 5个SQL语句
通过命令mysqldumpslow -s t -t 3 -g "like" /var/lib/mysql/slow-query.log按照时间排序且含有'like'的top 5个SQL语句
## 解释慢查询SQL执行过程
利用explain关键字可以分析SQL查询语句的执行过程，例如：执行
```sql
EXPLAIN SELECT * FROM res_user ORDER BYmodifiedtime LIMIT 0,1000
```

得到如下结果：显示结果分析：

table | type | possible_keys | key |key_len | ref | rows | Extra EXPLAIN列的解释：
重点关注：
- key（显示走的索引名称）：显示索引名称，比如I_name_age表示联合索引名称叫I_name_age
- key_len（数字，查看索引使用是否充分）：这里有个内置的计算公式
- type（显示走的索引类型）：
	- const：通过一次索引就能找到数据，一般用于主键索引或者联合索引减少了回表操作作为条件的查询sql中
	- eq_ref：常用于主键或唯一索引扫描。
	- ref：常用于普通索引和唯一索引扫描。
	- range：常用于范围查询，比如：between ... and 或 In ，like等操作
	- index：全索引扫描。执行sql如下
	- ALL：全表扫描，没走索引。
- Extra附加信息查看：
	- Using where：表示使用了where条件过滤。
	- Using temporary：表示是否使用了临时表，一般多见于order by 和 group by语句。
	- Using filesort：表示按文件排序，一般是SQL指定的排序字段和联合索引字段的排序顺序不一致的情况才会出现。比如我建立的是code和name的联合索引，顺序是code在前，name在后，这里直接按name排序，跟之前联合索引的顺序不一样。执行explain select code  from test1 order by name desc;附加信息中就会出现Using filesort
	- Using index：表示是否用了覆盖索引，说白了它表示是否所有获取的列都走了索引
	- Using index condition：默认开启了索引下推机制
## 慢查询优化措施举例
### 1.索引没起作用的情况

1. 使用LIKE关键字的查询语句
在使用LIKE关键字进行查询的查询语句中，如果匹配字符串的第一个字符为“%”，索引不会起作用。只有“%”不在第一个位置索引才会起作用。

2. 联合索引失效
MySQL可以为多个字段创建索引。一个索引最多可以包括16个字段。对于多列索引，只有查询条件使用了这些字段中的第一个字段时，索引才会被使用。

...还有很多种索引失效情况，这里不再赘述
### 2.优化表结构

合理的数据库结构不仅可以使数据库占用更小的磁盘空间，而且能够使查询速度更快。数据库结构的设计，需要考虑数据冗余、查询和更新的速度、字段的数据类型是否合理等多方面的内容。

1. 将字段很多的表分解成多个表

对于字段比较多的表，如果有些字段的使用频率很低，可以将这些字段分离出来形成新表。因为当一个表的数据量很大时，会由于使用频率低的字段的存在而变慢。

2. 避免频繁join操作，减少关联查询次数

将一个大的查询分解为多个小查询是很有必要的。

很多高性能的应用都会对关联查询进行分解，就是可以对每一个表进行一次单表查询，然后让查询结果在应用程序中进行关联，很多场景下这样会更高效，例如：

```javascript
SELECT * FROM tag 
        JOIN tag_post ON tag_id = tag.id
        JOIN post ON tag_post.post_id = post.id
        WHERE tag.tag = 'mysql';
分解为：
     SELECT * FROM tag WHERE tag = 'mysql';
     SELECT * FROM tag_post WHERE tag_id = 1234;
     SELECT * FROM post WHERE post.id in (123,456,567);
```

### 3.优化深分页问题
先说一下limit分页语法
```sql
select * from user_address limit M,N
```
我们一起来看一个实战案例哈。假设现在有表结构如下，并且有200万数据.
```sql
CREATE TABLE account (  
id varchar(32) COLLATE utf8_bin NOT NULL COMMENT '主键',  
account_no varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '账号'  
amount decimal(20,2) DEFAULT NULL COMMENT '金额'  
type varchar(10) COLLATE utf8_bin DEFAULT NULL COMMENT '类型A，B'  
create_time datetime DEFAULT NULL COMMENT '创建时间',  
update_time datetime DEFAULT NULL COMMENT '更新时间',  
PRIMARY KEY (id),  
KEY `idx_account_no` (account_no),  
KEY `idx_create_time` (create_time)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='账户表'
```
业务需求是这样：获取最2021年的A类型账户数据，上报到大数据平台。
```java
    //查询上报总数量  
    Integer total = accountDAO.countAccount();  
  
//查询上报总数量对应的SQL  
<select id ='countAccount' resultType="java.lang.Integer">  
    seelct count(1)  
    from account  
    where create_time >='2021-01-01 00:00:00'  
    and  type ='A'  
</select>  
  
    //计算页数  
    int pageNo = total % pageSize == 0 ? total / pageSize : (total / pageSize + 1);  
  
//分页查询，上报  
for(int i = 0; i < pageNo; i++){  
        List<AcctountPO> list = accountDAO.listAccountByPage(startRow,pageSize);  
        startRow = (pageNo-1)*pageSize;  
        //上报大数据  
        postBigData(list);  
    }  
  
//分页查询SQL（可能存在limit深分页问题，因为account表数据量几百万）  
<select id ='listAccountByPage' >  
    seelct *  
    from account  
    where create_time >='2021-01-01 00:00:00'  
    and  type ='A'  
    limit #{startRow},#{pageSize}  
</select>
```
以上的实现方案，会存在limit深分页问题，limit语句会先扫描offset+n行，然后再丢弃掉前offset行，只返回后n行数据。
那怎么优化呢？
#### 标签记录法
试想，如果limit的下一次的查询能从前一次查询结束后标记的位置开始查找
```sql
do{
	lastId = list.get(list,size()-1).getId();  
	lastList = select id,name,balance FROM account where id > #{lastId} order by id limit 10; 
	list.add(lastList);
}while(lastList==null)
```
#### 使用between and
很多时候，可以将`limit`查询转换为已知位置的查询，这样MySQL通过范围扫描`between...and`，就能获得到对应的结果。
如果知道边界值为100000，100010后，就可以这样优化：
```sql
select  id,name,balance FROM account where id between 100000 and 100010 order by id;
```
这种方式也有个天然的缺点，很多时候我们并不知道精确的扫描范围
#### 避免二级索引回表
先看下表结构哈：

```sql
CREATE TABLE account (
  id int(11) NOT NULL AUTO_INCREMENT COMMENT '主键Id',
  name varchar(255) DEFAULT NULL COMMENT '账户名',
  balance int(11) DEFAULT NULL COMMENT '余额',
  create_time datetime NOT NULL COMMENT '创建时间',
  update_time datetime NOT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (id),
  KEY idx_name (name),
  KEY idx_update_time (update_time) //索引
) ENGINE=InnoDB AUTO_INCREMENT=1570068 DEFAULT CHARSET=utf8 ROW_FORMAT=REDUNDANT COMMENT='账户表';
```
假设深分页的执行SQL如下：
```sql
select id,name,balance from account where update_time> '2020-09-19' limit 100000,10;
```
这个SQL的执行时间如下：
![[Pasted image 20240229163128.png]]
SQL的执行分析流程：
1. 通过普通二级索引树idx_update_time，==通过update_time条件，找到满足条件的记录ID（二级索引中叶子节点存放的是主键的值）==。
2. 通过ID，回到主键索引树，找到满足记录的行，然后取出展示的列（回表）
3. 扫描满足条件的100010行，然后扔掉前100000行，返回。
执行计划如下，扫描更多的行数，也意味着回表更多的次数。
![[Pasted image 20240229163643.png]]
##### 子查询select id

以上的SQL，回表了100010次，实际上我们能不能不去回表呢？可以将查询压力直接转移到主键索引上
```sql
select id,name,balance FROM account where id >= (select a.id from account a where a.update_time >= '2020-09-19' limit 100000, 1) LIMIT 10;
```
查询效果一样的，执行时间只需要0.038秒！
![[Pasted image 20240229164057.png]]
##### 连接查询INNER JOIN
通过INNER JOIN可以实现延迟关联，SQL如下：也是将查询压力直接转移到主键索引上
```sql
SELECT  acct1.id,acct1.name,acct1.balance FROM account acct1 INNER JOIN (SELECT a.id FROM account a WHERE a.update_time >= '2020-09-19' ORDER BY a.update_time LIMIT 100000, 10) AS  acct2 on acct1.id= acct2.id;
```
查询效果也是杠杆的，只需要0.034秒
![[Pasted image 20240229164336.png]]

因此我们可以做出如下的优化方案：
```sql
-- 查询用时0.068S
select field1,field2,field3 from user_address where id >= (select id from user_address order by id limit 100000,1) limit 10  
-- 或者INNER JOIN 延迟关联
Select news.id, news.description from news inner join (select id from news order by title limit 50000,5) as myNew using(id);
```


