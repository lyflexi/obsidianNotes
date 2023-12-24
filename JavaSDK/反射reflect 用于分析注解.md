>我有新的感悟！反射突破了方法的传统调用方式，正向调用如`obj.methodName(arg1...)`。
但是反射不需要提供对象引用，只需提供方法名称+参数就能invoke，这种反向调用方法的机制被称作反射。

Java代码的运行过程是从`.java`文件到 `.class`文件再到 `.class`被jvm加载，再到运行的过程，共有三个关键节点：
1. Java源代码：`.java`文件
2. 编译器：生成`.class`文件
3. 虚拟机，加载`.class`文件。解析执行`.class`文件（运行时）

`Java`反射机制的场景指的是在程序运行时加载、探索以及执行编译期间完全未知的`.class`文件。
- 在运行时对`Class`类型扫描，对于任意一个类都能够知道并调用这个类的所有属性和方法
- 避免直接`NEW`对象，利用反射方法`newInstance`才创建出实例化对象，可以让咱们的代码更加灵活

这种在运行时**动态的解析对象信息**以及**动态调用对象的方法**的功能称为`Java`的反射机制
# 运行时分析Class

获取`Class`类信息的五种API：
```Java
Class<?> clazz = Object.class // 类名.class 
Class<?> clazz = object.getClass() //对象.getClass()     
Class<?> clazz = Class.forName("ClassName") // 在编译时不能确定需要的是哪个类时使用 
Class clazz = ClassLoader.loadClass("cn.javaguide.TargetObject"); //传入类路径
packageScan//包扫描
```

`Class`类生成实例的API：`newInstance()`
```Java
object.getClass().newInstance()// 使用无参构造函数创建实例 
clazz.getConstructor(String.class).newInstance(“233”) //使用有参构造函数创建实例 
```

获取`Class`类中信息的API如下，于`java.lang.reflect`类库中之中
```Java
Field[] getFields() // 获取成员变量 
Method[] getMethods() // 获取方法 
Constructor[] getConstructors() // 获取构造函数 
```

操作`Class`类中信息：
- 设置成员变量
- 执行（唤醒）成员方法
```Java
//操作Field：参数1：对象  参数2：要修改的值 
public void set(Object obj, Object value) //可以修改任意对象的某成员变量值
//操作Method：参数1：为实现类 参数2： 为方法参数 
Public Object invoke(Object implicitPara,Object[] explicitPara) // 调用某类的任意方法     
```

# 反射应用场景

## 数据库驱动加载

比如你手上有3种不同DB的`Driver`类
- `com.mysql.Driver`
- `org.psql.Driver`
- `com.microsoft.msql.Driver`

你有个配置文件配置了具体的数据库信息(这里用json举个例子）：
ok，这里就有个问题，当你读取到了"com.mysql.Driver"这个字符串后，怎么把它变成实际的mysql那个Driver的对象？

```JSON
{
  "db": {
       "driver": "com.mysql.Driver",
       "uri": "jdbc:mysql://database_server:3306/foo",
       "username": "xxxxx",
       "password": "yyyyy"
  }
}
```

你当然可以这么干，下面这个叫Hard-coding，在编译期可以确定类型的写法。
这么写是可行的，只不过每次增加driver的种类时，就得改这个代码增加一行else if
```Java
// load json，得到dbconfig
if (dbconfig.dbDrvier.equals("com.mysql.Driver")) {
  Driver d = new com.mysql.Driver(dbconfig.dbUri, dbconfig.dbUsername, dbconfig.dbPassword);
} else if (dbconfig.dbDriver.equals("org.psql.Driver")) {
  // ...
}
```

反射处理方式如下，如果`dbDriver`这个`class`找不到，又或者`clz`不是个`Driver`，抛异常就好了。这就相当于让`Java`在运行时帮你找名字是这个字符串的那个类在不在，如果在就创建一个。但为了写这个程序，你必须知道未来可能处理什么数据的大概范围。比如这里会要求，一定得是个`dbDriver`，而不是发射核弹的客户端。

```Java
Class clz = Class.forName(dbconfig.dbDriver);
try{
    Driver d = (Driver)clz.newInstance(dbconfig.dbUri, dbconfig.dbUsername, dbconfig.dbPassword);
}catch{
    // 如果dbDriver这个class找不到，又或者clz不是个Driver，抛异常就可以了
}
```

## JDK动态代理
比如下面是通过 JDK 实现动态代理的示例代码，其中就使用了反射类 Method 来调用指定的方法。
```Java
public class DebugInvocationHandler implements InvocationHandler {
    /**
     * 代理类中的真实对象
     */
    private final Object target;

    public DebugInvocationHandler(Object target) {
        this.target = target;
    }

    @override
    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        System.out.println("before method " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("after method " + method.getName());
        return result;
    }
}
```

## Spring注解生效原理

为什么你使用 Spring 的时候 ，一个@Component注解就声明了一个类为Spring Bean呢？
为什么你通过一个@Value注解就读取到配置文件中的值呢？究竟是怎么起作用的呢？
我们自定义一个注解GPService
```Java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface GPService {
    String value() default "";
}
```

业务类QuerySerivce，这里就是模拟Spring的Service注解，我们用一个自定义注解。
```Java
@GPService
public class QuerySerivce implements IQueryService {
    public String query(String name) {
        return "query service result";
    }
}
```

这样我们在容器初始化的时候就可以通过包扫描将框架功能组件存入`Spring`容器当中，大概的代码如下：
1. 包扫描，扫描出标有`@Service`注解的类，
2. 然后通过反射实例化后放进容器内。

```Java
//拿到全类名，用于定位类，这一步一般Spring是通过扫描项目路径来获取，这一步是动态获取的
String className = "com.demo.QueryService";
//反射获取类的Class对象
Class<?> clazz = Class.forName(className);
//如果该类标注有GPService注解，我们就通过反射实例化这个类
if(clazz.isAnnotationPresent(GPService.class){
   Object instance = clazz.newInstance();
   //用map来模拟容器
   map.put(clazz.getSimpleName(),instance);
}
```

我们知道，Java注解只有被声明为运行时`@Retention(RetentionPolicy.RUNTIME)`，才会被`JVM`保留，才能够在运行时环境使用！

因此，如果我们要去分析方法上的注解，也必须在运行时分析，这也就必须**基于反射**才能获取到类/属性/方法/上的注解信息。你获取到注解之后做一些判断，就可以做进一步的业务处理。

## SpringAOP原理

SpringAOP使用了动态代理，动态代理的基础就是反射机制。
- 前置方法
- 反射执行目标方法
- 后置方法
# 反射机制产生弊端

反射之所以被称为框架的灵魂，主要是因为它赋予了我们在运行时分析类以及执行类中方法的能力。但与此同时也存在一定的弊端
- 增加了安全隐患。比如可以无视泛型参数的安全检查，因为泛型参数的安全检查仅发生在编译时。
- 反射的性能也要稍差点，通知 JVM 要做的事情，性能比直接的java代码要慢很多。不过，对于框架来说实际是影响不大的。