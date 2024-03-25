`@Transactional`注解源码如下，里面包含了Spring事务的配置参数：

```Java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

        @AliasFor("transactionManager")
        String value() default "";

        @AliasFor("value")
        String transactionManager() default "";

//事务的传播行为
        Propagation propagation() default Propagation.REQUIRED;
//事务的隔离级别
        Isolation isolation() default Isolation.DEFAULT;
//事务的超时时间，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则强制回滚事务。其单位是秒,默认值为-1，这表示事务的超时时间取决于底层事务系统或者没有超时时间。

        int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
//设置只读事务，如果你一次执行单条查询语句，则没有必要启用只读事务支持，数据库默认支持 SQL 执行期间的读一致性；如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询SQL必须保证整体的读一致性，否则在前条SQL查询之后，后条SQL查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用只读事务支持
        boolean readOnly() default false;
        
//用于指定能够触发事务回滚的异常类型，并且可以指定多个异常类型。
        Class<? extends Throwable>[] rollbackFor() default {};
        
        String[] rollbackForClassName() default {};

        Class<? extends Throwable>[] noRollbackFor() default {};

        String[] noRollbackForClassName() default {};

}
```
# 隔离级别Isolation

枚举类`Isolation`定义了Sptring当中的五种隔离级别，其中DEFAULT代表使用数据库的设置

MySQL默认的可隔离级别是REPEATABLE-READ，所以对于MySQL而言`DEFAULT`等价于`ISOLATION_REPEATABLE_READ`

```Java
public enum Isolation {
  
    DEFAULT(TransactionDefinition.ISOLATION_DEFAULT),
    
    READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),
    
    READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),
    
    REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),
    
    SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);
    
    private final int value;
    
    Isolation(int value) {
    this.value = value;
    }
    
    public int value() {
    return this.value;
    }

}
```

# 嵌套事务传播行为propagation
枚举类`Propagation`定义了事务的七种传播行为

```Java
package org.springframework.transaction.annotation;

import org.springframework.transaction.TransactionDefinition;

public enum Propagation {
    //七大事务传播行为
    REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),

    SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),

    MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),

    REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),

    NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),

    NEVER(TransactionDefinition.PROPAGATION_NEVER),

    NESTED(TransactionDefinition.PROPAGATION_NESTED);

    private final int value;

    Propagation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }

}
```

下面重点讲解三类传播行为：
- REQUIRED
- REQUIRES_NEW
- NESTED

## REQUIRED

REQUIRED：Spring事务默认传播行为，默认情况下内外部事务都被REQUIRED修饰，内外部事务属于同一事务，一起回滚

## REQUIRES_NEW

REQUIRES_NEW：内部事务被REQUIRES_NEW修饰

- 外部事务回滚不影响内部REQUIRES_NEW事务回滚，内部REQUIRES_NEW事务属于独立新事务
- 当内部REQUIRES_NEW事务出现异常不仅自己的内容回滚，也会导致外层事务也回滚，此时REQUIRES_NEW退化为REQUIRED

如果某个特殊的业务方法和其他业务不关联，我们可以给它单独设置REQUIRES_NEW，这样就能保证其他业务有异常时，它也不会被回滚。

## NESTED

NESTED：内部事务被NESTED修饰
- 内部NESTED事务失败不影响外部事务，整体事务仅回滚到内部事务的SavePoint点
- 外部事务需要手动`try-catch`异常来保证外部事务自身不回滚

伪代码如下：

```Java
ServiceA {  
    /** 
* 事务属性配置为 PROPAGATION_REQUIRED 
*/  
    void methodA() {  
        try {
            DML......
            ServiceB.methodB();  
        } catch(Exppection e) {
            DML....
            }  
        } 
}

ServiceB {  
            /** 
* 事务属性配置为 PROPAGATION_NESTED 
*/   
    void methodB() {
        DML......
        throw new RuntimeExppection("");
    }  

}     
```

这里有人会说，外部事务没有回滚是因为手动吃了异常，没有被Spring事务捕获到，是不是和嵌套事务没关系？

那我们改一下内部NESTED事务行为，也使用默认方式，伪代码如下
```Java
ServiceA {  
    /** 
* 事务属性配置为 PROPAGATION_REQUIRED 
*/  
    void methodA() {  
        try {
            DML......
            ServiceB.methodB();  
        } catch(Exppection e) {
            DML....
            }  
        } 
}

ServiceB {  
            /** 
* 事务属性配置为 PROPAGATION_REQUIRED 
*/   
    void methodB() {
        DML......
        throw new RuntimeExppection("");
    }  

}     
```

在这种REQUIRED的情况下内外公用一个事务，随后内层事务出现异常，Spring会把内部事务标记为==`rollback-only`==；同时外部事务手动捕获并处理异常，直到外部事务结束时，这时外部事务以为catch了万事大吉最后想要commit。但是内外部事务是同一个事务，事务已经被内层方法标记为`rollback-only`必须回滚，从而导致外部事务无法commit，这时Spring就会抛出`org.springframework.transaction.UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only`异常

可见在内部事务为NESTED的情况下，外部事务手动try-catch并不是多余的操作。

SavePoint是数据库事务中的一个概念，可以将整个事务切割为不同的小事务，可以选择将状态回滚到某个小事务发生时的样子
SavePoint定义脚本如下：

```sql
//保存保存点
SAVEPOINT 保存点的名称;
//回滚到某个保存点
ROLLBACK [WORK] TO 保存点的名称
//删除保存点
RELEASE SAVEPOINT 保存点名称;
```

一个事务中可以设置多个保存点并且无限制，如下代码所示：
- 回滚点无法随意跳转，例如下面，如果跳转到第一个保存点名字1，就无法再到保存点名字2
- 当事务整体提交、回滚时，保存点也随之释放


```Plain
开启事务: begin
....DML语句....
设置保存点: savepoint 保存点名字1
....DML语句....
设置保存点: savepoint 保存点名字2
....DML语句....
回滚保存点: rollback to 保存点名字2 (此时保存点2后面操作的状态都将回滚直保存点2时样子)
```

接下来用mysql演示一下，思路是向mysql insert三条数据，每次insert设置一个保存点，在第三次insert后回滚到第一个保存点

```Plain
// 开始查询空表
mysql> select * from t_x;
Empty set
 
// 开启事务
mysql> begin;
Query OK, 0 rows affected (0.01 sec)
 
// 插入第一条数据
mysql> insert into t_x value(1, '1');
Query OK, 1 row affected (0.01 sec)

// 设置保存点
mysql> savepoint a1;
Query OK, 0 rows affected (0.01 sec)
 
// 此时查询一条数据
mysql> select * from t_x;
+----+------+
| id | name |
+----+------+
|  1 | 1    |
+----+------+
1 row in set (0.01 sec)

// 插入第二条数据
mysql> insert into t_x value(2, '2');
Query OK, 1 row affected (0.01 sec)

// 设置保存点2
mysql> savepoint a2;
Query OK, 0 rows affected (0.01 sec)

// 查询两条数据
mysql> select * from t_x;
+----+------+
| id | name |
+----+------+
|  1 | 1    |
|  2 | 2    |
+----+------+
2 rows in set (0.01 sec)

// 插入第三条数据
mysql> insert into t_x value(3, '3');
Query OK, 1 row affected (0.01 sec)

// 共三条
mysql> select * from t_x;
+----+------+
| id | name |
+----+------+
|  1 | 1    |
|  2 | 2    |
|  3 | 3    |
+----+------+
3 rows in set (0.01 sec)

// 回滚到第一个保存点
mysql> rollback to a1;
Query OK, 0 rows affected (0.01 sec)

// 此时查询，只有第一个保存点时的一条数据
mysql> select * from t_x;
+----+------+
| id | name |
+----+------+
|  1 | 1    |
+----+------+
1 row in set (0.00 sec)

// 尝试回滚到第二个保存点, 出错
mysql> rollback to a2;
1305 - SAVEPOINT a2 does not exist
mysql> 
```

其他四种传播行为（了解即可）

MANDATORY
MANDATORY：内部事务被MANDATORY修饰
MANDATORY属于强制性事务，当外层方法没有被事务修饰时，将抛异常`No existing transaction found for transaction marked with propagation 'mandatory'`


SUPPORTS
SUPPORTS：内部事务被SUPPORTS修饰
- 如果外部方法被事务修饰，则内部事务加入到外部事务
- 如果外部方法没有被事务修饰，就以内部事务就以非事务方式执行


NOT_SUPPORTED
NOT_SUPPORTED：内部事务被NOT_SUPPORTED修饰

如果外部方法有事务，则挂起外部事务，内部方法采取非事务模式操作数据库

NEVER
NEVER：内部事务被NEVER修饰

如果外部方法有事务，Spring直接报错

# 异常识别rollbackFor
Java所有的异常都有一个共同的祖先类`Throwable`，`Throwable`类有两个重要的子类Exception和Error
其中Exception 可以分为受检查异常`CheckException`和不受检查异常即`RuntimeException`
- `CheckException`：受检查代码比如IO操作没有被 catch/throw 处理的话，就没办法通过编译 ，有`IOException`、`ClassNotFoundException` 、`SQLException`等。
- `RuntimeException即UnCheckException`：不受检查异常，即使不处理也可以正常通过编译。RuntimeException及其子类都统称为非受检查异常，有空指针异常`NullPointerException`、字符串转换为数字异常`NumberFormatException`、数组越界异常`ArrayIndexOutOfBoundsException`、类型转换异常`ClassCastException`、算术异常`ArithmeticException`等
Error属于程序无法处理的错误无法通过catch来进行捕获 ，例如Java 虚拟机运行错误`Virtual MachineError`、虚拟机内存不够错误`OutOfMemoryError`、类定义错误`NoClassDefFoundError`等 。当Error发生时，JVM一般会选择线程终止
![[Pasted image 20240106163426.png]]

==Spring事务只有遇到`RuntimeException`和`Error`时才会回滚，但是在遇到检查型异常`CheckException`时不会回滚。==
![[Pasted image 20240325111016.png]]
需要注意的是，上面这句话的限制针对的是业务代码。我们习惯性的将rollbackFor设置的更高级一点，以兼容更多样的业务运行时异常
```java
@Transactional(rollbackFor = Exception.class)

OR

@Transactional(rollbackFor = Throwable.class)
```
## 错误的异常处置

1. 被try...catch手动捕获的事务不会回滚，比如：

```Java
@Slf4j
@Service
public class UserService {
    
    @Transactional
    public void add(UserModel userModel) {
        try {
            saveData(userModel);
            updateData(userModel);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }
}
```

2. 即使开发者没有手动捕获异常，但如果业务中抛出了Spring事务不支持的异常，则Spring事务也不会回滚。比如Spring事务默认情况下只会回滚RuntimeException和Error，对于RuntimeException的父类Exception默认并不会回滚。

```Java
@Slf4j
@Service
public class UserService {
    
    @Transactional
    public void add(UserModel userModel) throws Exception {
        try {
             saveData(userModel);
             updateData(userModel);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            throw new Exception(e);
        }
    }
}
```

3. rollbackFor声明的异常与业务异常不匹配，Spring事务也不会回滚。比如在执行下面这段代码保存和更新数据时，程序报错了，抛了SqlException、DuplicateKeyException等异常，报错异常不属于自定义异常BusinessException，所以事务也不会回滚。所以，建议一般情况下，将该参数设置成：`Exception`或`Throwable`

```Java
@Slf4j
@Service
public class UserService {
    
    @Transactional(rollbackFor = BusinessException.class)
    public void add(UserModel userModel) throws Exception {
       saveData(userModel);
       updateData(userModel);
    }
}
```

## 正确的异常处理

如果非要在业务方法中catch的话，有两种方式解决

1. 一定要抛出`throw new RuntimeException()`，否则Spring会将你的catch业务操作commit，这样就会产生脏数据
```Java
 @Override
 @Transactional
 public Json addOrder(TOrderAddReq tOrderAddReq) {
        try{
     //增删改方法
         } catch (Exception e) {
            //catch业务
                 throw new RuntimeException();
         }
    return json;
 }

```

2. 忽略声明式注解`@Transactional`，手动硬编码`TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`开启Spring事务强制回滚

```Java
     @Override
     @Transactional
     public Json addOrder(TOrderAddReq tOrderAddReq) {
     try{
     //增删改方法
         } catch (Exception e) {
             // 手动硬编码开启spring事务管理
             TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
             e.printStackTrace();}
// }
         return json;
     }
```

3. 为了使所有异常都触发事务控制，可以将回滚策略`rollbackFor`配置为`Exception.class`或者`Throwable.class`，最大化Spring事务识别能力

```java
 @Transactional(rollbackFor = Exception.class)
 @Transactional(rollbackFor = Throwable.class)
```