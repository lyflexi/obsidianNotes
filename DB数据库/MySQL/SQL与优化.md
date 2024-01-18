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

# 子查询
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

==exist需要在子查询中额外的判断一次where，因此exist不够直观==
- "IN" 子句用于在 WHERE 条件中指定多个可能的值。它允许你指定一个值列表，并检查列中的值是否匹配列表中的任何一个值。
语法示例：
```sql
SELECT column1, column2, ...  FROM table_name  WHERE column_name IN (value1, value2, ...);
```
类似于：
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
- "EXISTS" 子句用于测试子查询是否返回任何结果。它用于确定是否存在至少一行与外部查询匹配的行，==有的时候使用exist可以优化sql查询==
语法示例：
```sql
SELECT column1, column2, ...  FROM table_name1  WHERE EXISTS (SELECT column_name FROM table_name2 WHERE condition);
```
在这个例子中，`table_name1` 是外部查询的表名，`table_name2` 是子查询的表名，`column_name` 是要检查的列名，`condition` 是子查询中的条件。如果子查询返回至少一行结果，则 "EXISTS" 子句将返回 true，外部查询将返回匹配的结果。
因此，“EXISTS”一般是这么用的：
```sql
SELECT column1, column2, ...  FROM table_name1  WHERE EXISTS (SELECT 1 FROM table_name2 WHERE condition);
```
只需要确认子查询返回至少一行结果，而不需要子查询获取实际所有结果，这有助于减少不必要的数据处理和传输。类似于：
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

# 慢查询日志
慢查询日志是MySQL提供的一种日志记录，用于记录在MySQL中响应时间超过阀值的语句。具体来说，慢查询日志会记录那些运行时间超过long_query_time值的SQL语句，其中long_query_time的默认值为10秒。

开启慢查询日志需要手动设置，因为默认情况下MySQL的慢查询日志是禁用的。要开启慢查询日志，可以通过设置slow_query_log的值来开启，例如使用以下命令：

```sql
sql复制代码SET GLOBAL slow_query_log = 1;
```

另外，还可以通过修改my.cnf文件来永久生效，例如在[mysqld]下添加以下行：

```makefile
makefile复制代码slow_query_log = 1  slow_query_log_file = /var/lib/mysql/node-slow.log
```

其中，slow_query_log_file指定了慢查询日志文件的存储路径。

开启慢查询日志后，可以通过查看慢查询日志来分析哪些SQL语句超出了最大忍耐时间值，从而进行优化。但需要注意的是，开启慢查询日志会或多或少地带来一定的性能影响，因为它需要记录每个超时的SQL语句。因此，如果不是调优需要的话，一般不建议开启慢查询日志。