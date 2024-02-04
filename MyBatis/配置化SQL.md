
# if-test

```xml
<select id="getSendDetailPage"
		resultType="com.iwhalecloud.aiFactory.aiResource.aimessage.middle.MsgTaskSendDetailMiddle">
	SELECT b.user_code,b.user_name,c.send_time,a.send_state,a.msg_state
	FROM msg_log a LEFT JOIN msg_user b ON a.user_id = b.msg_user_id
	LEFT JOIN msg_task c ON a.msg_task_id = c.msg_task_id
	<where>

		<if test="sendChannel != null and sendChannel != ''">
			and a.msg_channel = #{sendChannel}
		</if>
		<if test="sendStatus != null and sendStatus != ''">
			and a.send_state = #{sendStatus}
		</if>

	</where>
	ORDER BY
	a.CREATE_TIME DESC
	LIMIT ${(current-1)*size},#{size}
</select>
```

# choose (when, otherwise)标签

有时候我们并不想应用所有的条件，而只是想从多个选项中选择一个。而使用if标签时，只要test中的表达式为 true，就会执行 if 标签中的条件。MyBatis 提供了 choose 元素。if标签是与(and)的关系，而 choose 是或(or)的关系。

choose标签是按顺序判断其内部when标签中的test条件出否成立，如果有一个成立，则 choose 结束。当 choose 中所有 when 的条件都不满则时，则执行 otherwise 中的sql。类似于Java 的 switch 语句，choose 为 switch，when 为 case，otherwise 则为 default。

例如下面例子，同样把所有可以限制的条件都写上，方面使用。choose会从上到下选择一个when标签的test为true的sql执行。安全考虑，我们使用where将choose包起来，防止关键字多余错误。

```xml
<!--  choose(判断参数) - 按顺序将实体类 User 第一个不为空的属性作为：where条件 -->  
<select id="getUserList_choose" resultMap="resultMap_user" parameterType="com.yiibai.pojo.User">  
    SELECT *  
      FROM User u   
    <where>  
        <choose>  
            <when test="username !=null ">  
                u.username LIKE CONCAT(CONCAT('%', #{username, jdbcType=VARCHAR}),'%')  
            </when >  
            <when test="sex != null and sex != '' ">  
                AND u.sex = #{sex, jdbcType=INTEGER}  
            </when >  
            <when test="birthday != null ">  
                AND u.birthday = #{birthday, jdbcType=DATE}  
            </when >  
            <otherwise>  
            </otherwise>  
        </choose>  
    </where>    
</select>
```

# foreach模板

你可以将任何可迭代对象（如 List、Set 等）、Map 对象或者数组对象作为集合参数传递给 foreach。
- 当使用可迭代对象或者数组时，index 是当前迭代的序号，item 的值是本次迭代获取到的元素。
- 当使用 Map 对象（或者 Map.Entry 对象的集合）时，index 是键，item 是值。

## separator=" ，"

属性separator 为逗号，前段传过来的myList 参数是list集合


```Java
<if test="myList != null">
    AND dm in
    <foreach collection="myList " item="item" open="(" separator="," close=")">
        #{item , jdbcType=VARCHAR }
    </foreach>
</if>

最后渲染为sql语句为

AND dm in ( '03' , '04')
```

## separator=" OR "

属性separator 为or，前端传过来的myList 参数是list集合

```Java
<if test="myList != null">
    AND
    <foreach collection="myList " index="index" item="item" open="(" separator="or" close=")">
        dm  = #{item , jdbcType=VARCHAR }
    </foreach>
</if>

最后渲染为sql语句为
AND ( dm  = '01'or dm   = '02' or dm   = '03') 
```
## 查询条件为行匹配

SQL:

```Ruby
select * from user where (user_id,type) in ((568,6),(569,6),(600,8));
```


MyBatis XML:

```xml
select date_format(create_Time,'%Y-%m-%d %H:%i:%s') as createTime ,reason,operator,remarks,operation 
from refund_order_log
where (order_no,channel) in
<foreach collection="list" item="item" open="(" close=")" separator=",">
	(#{item.orderNo},#{item.channel})
</foreach>
and status =1 
order by create_time  desc
```
# 返回嵌套的POJO

一直对association和collection有点混淆，现整理一篇文章，用于加强记忆。

现在有2个表book表、bookshelf书架表。

|   |   |   |
|---|---|---|
|BOOK|   |   |
|字段名称|类型|备注|
|id|int|主键|
|name|varchar|书名|
|type|int|类型|
|shelf_id|int|书架id|

|   |   |   |
|---|---|---|
|Book_shelf|   |   |
|字段名称|类型|备注|
|id|int|主键|
|number|varchar|书架编号|
|num|int|可存放数量|

## association【嵌套对象】

association用于当前POJO关联了另一个POJO的场景使用。

现有需求：查询根据书籍id查询书籍信息和所在书架编号。

Book.PoJo

```Java
public class Book{
	private Integer id;
	private String name;
	private String type;
	private Integer shelfId;
	private BookShelf bookShelfDto;
}
```

BookShelf.Pojo

```Java
public class BookShelf {
	private Integer id;
	private String number;
	private String num;
}
```

mapper：type="com.abc.Book"Bo

```xml
<resultMap id="bookResultMap" type="com.abc.Book"> 
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <result property="type" column="type"/>
    <!--关联属性-->
    <association property="bookShelfDto" ofType="com.abc.BookShelf">
        <id property="id" column="shelf_id"/>
        <result property="number" column="number"/>
        <result property="num" column="num"/>
    </association>
</resultMap>
 
 
<select id="getBookInfo" resultMap="bookResultMap">
	select book.id,book.name,book.type,book.shelf_id,shelf.number,shelf.num
	from book left join book_shelf shelf on book.shelf_id = shelf.id 
	where book.id = #{id}
</select>
```

## collection【嵌套对象集合】

collection用于当前POJO关联了其他POJO集合的场景使用。

表不变

现有需求：根据书架ID查询书架信息及书架存放的书籍信息。

Book.POJO

```Java
public class Book{
	private Integer id;
	private String name;
	private String type;
	private Integer shelfId;
}
```

BookShelf.Pojo

```Java
public class BookShelf {
	private Integer id;
	private String number;
	private String num;
	private List<Book> bookList;
}
```

mapper

```xml
<resultMap id="bookShelfResultMap" type="com.abc.BookShelf">
    <id property="id" column="shelf_id"/>
    <result property="number" column="number"/>
    <result property="num" column="num"/>
    <!--关联属性-->
    <collection property="bookList" javaType="com.abc.Book">
         <id property="id" column="id"/>
         <result property="name" column="name"/>
         <result property="type" column="type"/>
    </collection>
</resultMap>
 
 
<select id="getBookShelfInfo" resultMap="bookShelfResultMap">
	select book.id,book.name,book.type,book.shelf_id,shelf.number,shelf.num
	from book left join book_shelf shelf on book.shelf_id = shelf.id 
	where shelf.id = #{id}
</select>
 
Mapper.java
BookShelf getBookShelfInfo(Integer id);
```