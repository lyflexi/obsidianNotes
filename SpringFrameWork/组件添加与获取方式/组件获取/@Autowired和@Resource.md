@Autowired和@Resource注解都是用来注入Bean的，@Autowired注解是Spring提供的，而@Resource注解是J2EE本身提供的
![[Pasted image 20231226152620.png]]
- @Autowird注解优先通过byType方式注入
- @Resource注解优先通过byName方式注入

# byType

首先有一个接口UserService和两个实现类UserServiceImpl1和UserServiceImpl2，并且这两个实现类已经加入到Spring的IOC容器中了

```Java
@Service
public class UserServiceImpl1 implements UserService
@Service
public class UserServiceImpl2 implements UserService
```

通过@Autowired注入使用

```Java
@Autowired
private UserService userService;
```

根据上面的步骤，可以很容易判断出，直接这么使用是会报错的

原因：首先通过byType注入，判断**UserService**类型有两个实现，无法确定具体是哪一个，

解决方式一：

```Java
// 方式一：改变变量名
@Autowired
private UserService userServiceImpl1;
```

解决方式二：

```Java
// 方式二：配合@Qualifier注解来显式指定name值
@Autowired
@Qualifier(value = "userServiceImpl1")
private UserService userService;
```

另外，使用@Autowired注解注入对象的前提是对象本身在IOC容器中存在，否则需要加上属性`required=false`，表示忽略当前要注入的Bean，如果有直接注入，没有则跳过，不会报错

# byName

@Resource优先通过byName注入，如果没有匹配则通过byType注入

```Java
@Service
public class UserServiceImpl1 implements UserService
@Service
public class UserServiceImpl2 implements UserService
```

通过@Resource注入使用

```Java
@Resource
private UserService userService;
```

通过byName匹配，首字母小写的变量名userService无法匹配IOC容器中任何一个id（这里指的userServiceImpl1和userServiceImpl2）报错

@Resource可以通过参数`name`显式指定所需的对象ID
```Java
// 1. 默认方式：byName，既没指定name属性，也没指定type属性：默认通过byName方式注入，如果byName匹配失败，则使用byType方式注入（也就是上面的那个例子）
@Resource  
private UserService userDao; 
// 2. 指定byName，指定name属性：通过byName方式注入，把name属性和IOC容器中的id去匹配，匹配失败则报错
@Resource(name="userService")  
private UserService userService; 
```

另外，@Resource可以通过参数`type`显式指定对象类型，以达到byType同样的注入效果
```java
// 1. 指定byType，指定type属性：通过byType方式注入，在IOC容器中匹配对应的类型，如果匹配不到或者匹配到多个则报错
@Resource(type=UserService.class)  
private UserService userService; 
// 2. 指定byName和byType，同时指定name属性和type属性：在IOC容器中匹配，名字和类型同时匹配则成功，否则失败
@Resource(name="userService",type=UserService.class)  
private UserService userService;
```