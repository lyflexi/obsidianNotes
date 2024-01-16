# 在多线程中使用了事务
下面例子中，事务方法doOtherThing新起了线程执行。这样会导致内外事务不在同一个线程中，获取到的数据库连接不一样，从而是两个不同的事务。如果想doOtherThing方法中抛了异常，add方法也回滚是不可能的。

```Java
@Slf4j
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;
    @Autowired
    private RoleService roleService;

    @Transactional
    public void add(UserModel userModel) throws Exception {
        userMapper.insertUser(userModel);
        new Thread(() -> {
            roleService.doOtherThing();
        }).start();
    }
}

@Service
public class RoleService {

    @Transactional
    public void doOtherThing() {
        System.out.println("保存role表数据");
    }
}
```

Spring事务源码是通过数据库连接来实现的，当前线程中保存了一个map，key是数据源，value是数据库连接。在不同的线程中拿到的数据库连接也不同。我们说的同一个事务，其实是指同一个数据库连接，只有拥有同一个数据库连接才能同时提交和回滚。

```Java
private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");
```

# 使用了this调用事务方法
`updateStatus`方法拥有事务的能力是因为`spring aop`生成了代理对象，针对所有的Spring AOP注解（比如@Transcational或者@Async），Spring在扫描bean的时候如果发现有此类注解，那么会把目标方法所在的类动态构造一个代理对象返回，此后的目标方法调用相当于是代理对象在执行方法调用，Spring让代理对象暗中给目标方法做了增强


但是通过this获取到的当前对象不是代理对象，因此通过`this`调用另一个事务方法`updateStatus`，AOP是不生效的

```Java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Transactional
    public void add(UserModel userModel) {
        userMapper.insertUser(userModel);
        updateStatus(userModel);
    }

    @Transactional
    public void updateStatus(UserModel userModel) {
        doSameThing();
    }
}
```

## 解决方式1：新加一个Service类

这个方法非常简单，只需要新加一个Service类

```Java
@Servcie
public class ServiceA {
   @Autowired
   prvate ServiceB serviceB;

   public void save(User user) {
         queryData1();
         queryData2();
         serviceB.doSave(user);
   }
 }

 @Servcie
 public class ServiceB {

    @Transactional(rollbackFor=Exception.class)
    public void doSave(User user) {
       addData1();
       updateData2();
    }

 }
```

## 解决方式2：在该Service类中注入自己

如果不想再新加一个Service类，在该Service类中注入自己也是一种选择。Spring的三级缓存解决了循环依赖问题（自我依赖也是种循环依赖）

```Java
@Servcie
public class ServiceA {
   @Autowired
   prvate ServiceA serviceA;

   public void save(User user) {
         queryData1();
         queryData2();
         serviceA.doSave(user);
   }

   @Transactional(rollbackFor=Exception.class)
   public void doSave(User user) {
       addData1();
       updateData2();
    }
 }
```

## 解决方式3：通过AopContext类

使用aop-starter包中的工具类AopContext

```XML
<!-- 引入aop -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

使用`@EnableAspectJAutoProxy(exposeProxy=true)`开启AspectJAOP，设置true是为了对外暴露代理对象。

```Java
@EnableAspectJAutoProxy(exposeProxy = true)     //开启了aspect动态代理模式,对外暴露代理对象
@SpringBootApplication(exclude = GlobalTransactionAutoConfiguration.class)
public class GulimallOrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(GulimallOrderApplication.class, args);
    }
}
```

在该Service类中使用AopContext.currentProxy()获取代理对象

```Java
@Servcie
public class ServiceA {

   public void save(User user) {
         queryData1();
         queryData2();
         ((ServiceA)AopContext.currentProxy()).doSave(user);
   }

   @Transactional(rollbackFor=Exception.class)
   public void doSave(User user) {
       addData1();
       updateData2();
    }
 }
```