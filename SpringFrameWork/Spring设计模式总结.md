Spring 框架中用到了哪些设计模式？

- 工厂设计模式 : Spring使用工厂模式通过 BeanFactory、ApplicationContext 创建 bean 对象。
    
- 单例设计模式 : Spring 中的 Bean 默认都是单例的。
    
- 代理设计模式 : cglib实现Spring AOP 功能
    
- 观察者模式: Spring 事件驱动模型就是观察者模式很经典的一个应用。
    
- 策略模式：Spring MVC中的策略模式
    
- 适配器模式和装饰者模式
    
- .....
    

# 工厂模式

Spring使用工厂模式可以通过 BeanFactory 或 ApplicationContext 创建 bean 对象。

两者对比：

- BeanFactory ：延迟注入(使用到某个 bean 的时候才会注入),相比于ApplicationContext 来说会占用更少的内存，程序启动速度更快。
    
- ApplicationContext ：容器启动的时候，不管你用没用到，一次性创建所有 bean 。BeanFactory 仅提供了最基本的依赖注入支持， ApplicationContext 扩展了 BeanFactory ,除了有BeanFactory的功能还有额外更多功能，所以一般开发人员使用 ApplicationContext会更多。
    

ApplicationContext的三个实现类：

1. ClassPathXmlApplication：把上下文文件当成类路径资源。
    
2. FileSystemXmlApplication：从文件系统中的 XML 文件载入上下文定义信息。
    
3. XmlWebApplicationContext：从Web系统中的XML文件载入上下文定义信息。
    

Example:

```Java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.FileSystemXmlApplicationContext;
 
public class App {
        public static void main(String[] args) {
                ApplicationContext context = new FileSystemXmlApplicationContext(
                                "C:/work/IOC Containers/springframework.applicationcontext/src/main/resources/bean-factory-config.xml");
 
                HelloApplicationContext obj = (HelloApplicationContext) context.getBean("helloApplicationContext");
                obj.getMsg();
        }
}
```

# 单例模式

在我们的系统中，有一些对象其实我们只需要一个，比如说：线程池、缓存、对话框、注册表、日志对象、充当打印机、显卡等设备驱动程序的对象。事实上，这一类对象只能有一个实例，如果制造出多个实例就可能会导致一些问题的产生，比如：程序的行为异常、资源使用过量、或者不一致性的结果。

使用单例模式的好处:

- 对于频繁使用的对象，可以省略创建对象所花费的时间，这对于那些重量级对象而言，是非常可观的一笔系统开销；
    
- 由于 new 操作的次数减少，因而对系统内存的使用频率也会降低，这将减轻 GC 压力，缩短 GC 停顿时间。
    

==Spring 中 bean 的默认作用域就是 singleton(单例)的==。 Spring 中 bean 共有下面五种作用域，可以通过@Scope注解设置：

- singleton ：将Bean放入 IOC容器的缓存池中，由Spring管理Bean的生命周期，此时Bean都是单实例的，对于单实例Bean对象来说，在Spring容器创建完成后就会对单实例Bean进行实例化
    
- prototype : 每次请求都会创建一个新的 bean 实例。 "prototype" 则将Bean返回给调用者， 由调用者管理Bean 的生命周期，此时Bean是多实例的，多实例Bean在使用getBean获取的时候才创建对象
    
- request : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
    
- session : 每一次HTTP请求都会产生一个新的 bean，该bean仅在当前 HTTP session 内有效。
    
- global-session： 全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话
    

Spring 实现单例的方式：

- xml方式: `<bean id="userService" class="top.snailclimb.UserService" scope="singleton"/>`
    
- 注解方式：`@Scope(value = "singleton")`
    

## Spring源码

Spring 通过 三级缓存 解决循环依赖问题，`DefaultSingletonBeanRegistry`类的`getSingleton(String beanName, boolean allowEarlyReference)`方法如下：

```Java
    @Nullable
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                synchronized(this.singletonObjects) {
                    singletonObject = this.singletonObjects.get(beanName);
                    if (singletonObject == null) {
                        singletonObject = this.earlySingletonObjects.get(beanName);
                        if (singletonObject == null) {
                            ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                            if (singletonFactory != null) {
                                singletonObject = singletonFactory.getObject();
                                this.earlySingletonObjects.put(beanName, singletonObject);
                                this.singletonFactories.remove(beanName);
                            }
                        }
                    }
                }
            }
        }

        return singletonObject;
    }
```

## 双重校验锁实现对象单例（线程安全）

  

当被问到要实现一个单例模式时，很多人的第一反应是写出如下的代码，包括教科书上也是这样教我们的。

```Java
public class Singleton {
    private static Singleton instance;
    private Singleton (){}

    public static Singleton getInstance() {
     if (instance == null) {
         instance = new Singleton();
     }
     return instance;
    }
}
```

这段代码简单明了，而且使用了懒加载模式，但是却存在致命的问题。当有多个线程并行调用 getInstance() 的时候，就会创建多个实例。也就是说在多线程下不能正常工作。为了解决上面的问题，最简单的方法是将整个 getInstance() 方法设为同步（synchronized）。

```Java
public class Singleton {
    private static Singleton instance;
    private Singleton (){}

    public static synchronized Singleton getInstance() {//封死了
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
}
```

虽然做到了线程安全，并且解决了多实例的问题，但是它并不高效。因为在任何时候只能有一个线程调用 getInstance() 方法。但是同步操作只需要在第一次调用时才被需要，即第一次创建单例实例对象时。这就引出了双重检验锁。双重检验锁模式（double checked locking pattern），是一种使用同步块加锁的方法。程序员称其为双重检查锁，因为会有两次检查 instance == null，一次是在同步块外，一次是在同步块内。为什么在同步块内还要再检验一次？因为可能会有多个线程一起进入同步块外的 if，如果在同步块内不进行二次检验的话就会生成多个实例了。

```Java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {//Single Checked
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {//Double Checked
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

另外，需要注意 uniqueInstance 采用 volatile 关键字修饰也是很有必要。

uniqueInstance 采用 volatile 关键字修饰也是很有必要的， uniqueInstance = new Singleton(); 这段代码其实是分为三步执行：

1. 为 uniqueInstance 分配内存空间
    
2. 初始化 uniqueInstance
    
3. 将 uniqueInstance 指向分配的内存地址
    

但是由于 JVM 具有指令重排的特性，执行顺序有可能变成 1->3->2。指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。例如，线程 T1 执行了 1 和 3，此时 T2 调用 getUniqueInstance() 后发现 uniqueInstance 不为空，因此返回 uniqueInstance，但此时 uniqueInstance 还未被初始化。

使用 volatile 可以禁止 JVM 的指令重排，保证在多线程环境下也能正常运行。

# 代理模式介绍

代理模式是一种设计模式，提供了对目标对象额外的访问方式，即通过代理对象访问目标对象，这样可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。简言之，代理模式就是设置一个中间代理来控制访问原目标对象，以达到增强原对象的功能和简化访问方式。举个例子，我们生活中经常到火车站去买车票，但是人一多的话，就会非常拥挤，于是就有了代售点，我们能从代售点买车票了。这其中就是代理模式的体现，代售点代理了火车站对象，提供购买车票的方法。

代理类和目标类是否需要实现业务接口：

| 代理方式 | 代理类是否需要实现业务接口 | 目标类是否实现业务接口 |
| ---- | ---- | ---- |
| 静态代理 | √ | √ |
| 动态代理jdk | × | √ |
| 动态代理cglib | × | × |

## 静态代理

静态代理代理模式UML类图
![[Pasted image 20240108155242.png]]

这种代理方式需要代理对象和目标对象实现一样的接口。

优点：可以在不修改目标对象的前提下扩展目标对象的功能。

缺点：

1. 冗余。由于代理对象要实现与目标对象一致的接口，会产生过多的代理类。
    
2. 不易维护。一旦接口增加方法，目标对象与代理对象都要进行修改。
    

举例：保存用户功能的静态代理实现

- 接口类： IUserDao
    

```Java
package com.proxy;

public interface IUserDao {
    public void save();
}
```

- 目标对象：UserDao
    

```Java
package com.proxy;

public class UserDao implements IUserDao{

    @Override
    public void save() {
        System.out.println("保存数据");
    }
}
```

- 静态代理对象：UserDapProxy 需要实现IUserDao接口！
    

```Java
package com.proxy;

public class UserDaoProxy implements IUserDao{

    private IUserDao target;
    public UserDaoProxy(IUserDao target) {
        this.target = target;
    }
    
    @Override
    public void save() {
        System.out.println("开启事务");//扩展了额外功能
        target.save();
        System.out.println("提交事务");
    }
}
```

- 测试类：TestProxy
    

```Java
package com.proxy;

import org.junit.Test;

public class StaticUserProxy {
    @Test
    public void testStaticProxy(){
        //目标对象
        IUserDao target = new UserDao();
        //代理对象
        UserDaoProxy proxy = new UserDaoProxy(target);
        proxy.save();
    }
}
```

- 输出结果
    

```Java
开启事务
保存数据
提交事务
```

## 动态代理jdk

动态代理利用了JDK API，动态地在内存中构建代理对象，从而实现对目标对象的代理功能。动态代理又被称为JDK代理或接口代理。
静态代理与动态代理的区别主要在：

- 静态代理在编译时就已经实现，编译完成后代理类是一个实际的class文件
    
- 动态代理是在运行时动态生成的，即编译完成后没有实际的class文件，而是在运行时动态生成类字节码，并加载到JVM中
    

特点： 动态代理对象不需要实现接口，但是要求目标对象必须实现接口，否则不能使用动态代理。

JDK中生成代理对象主要涉及的类有

- `java.lang.reflect Proxy`，主要方法为
    

```Java
static Object    newProxyInstance(ClassLoader loader,  //指定当前目标对象使用类加载器

 Class<?>[] interfaces,    //目标对象实现的接口的类型
 InvocationHandler h      //事件处理器
) 
//返回一个指定接口的代理类实例，该接口可以将方法调用指派到指定的调用处理程序。
```

- `java.lang.reflect InvocationHandler`，主要方法为
    

```Java
 Object    invoke(Object proxy, Method method, Object[] args) 
// 在代理实例上处理方法调用并返回结果。
```

举例：保存用户功能的动态代理实现

- 接口类： IUserDao
    

```Java
package com.proxy;

public interface IUserDao {
    public void save();
}
```

- 目标对象：UserDao
    

```Java
package com.proxy;

public class UserDao implements IUserDao{

    @Override
    public void save() {
        System.out.println("保存数据");
    }
}
```

- 动态代理对象：UserProxyFactory
    

```Java
package com.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class ProxyFactory {

    private Object target;// 维护一个目标对象

    public ProxyFactory(Object target) {
        this.target = target;
    }

    // 为目标对象生成代理对象
    public Object getProxyInstance() {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(),
                new InvocationHandler() {

                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("开启事务");

                        // 执行目标对象方法
                        Object returnValue = method.invoke(target, args);

                        System.out.println("提交事务");
                        return null;
                    }
                });
    }
}
```

动态代理的时候为什么还是用了实现了接口的目标对象？

因为JDK代理`public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h) throws IllegalArgumentException{...}`即`Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new InvocationHandler() {...}`，需要用构造方法动态获取具体的接口信息，如果不实现接口的话，代理对象Proxy没法初始化

  

- 测试类：TestProxy
    

```Java
package com.proxy;

import org.junit.Test;

public class TestProxy {

    @Test
    public void testDynamicProxy (){
        IUserDao target = new UserDao();
        System.out.println(target.getClass());  //输出目标对象信息
        IUserDao proxy = (IUserDao) new ProxyFactory(target).getProxyInstance();
        System.out.println(proxy.getClass());  //输出代理对象信息
        proxy.save();  //执行代理方法
    }
}
```

- 输出结果
    

```Java
class com.proxy.UserDao
class com.sun.proxy.$Proxy4
开启事务
保存数据
提交事务
```

## 动态代理cglib

cglib is a powerful, high performance and quality Code Generation Library. It can extend JAVA classes and implement interfaces at runtime.

[cglib](https://link.segmentfault.com/?enc=iRS52zco5WOLgrbJodtmhw%3D%3D.6O1Bq5KhfKRRAnXVpx2cknbi4Fth3qHMPbLqx%2BGNc%2BU%3D) (Code Generation Library )是一个第三方代码生成类库，基于字节码class，运行时在内存中动态生成一个子类对象从而实现对目标对象功能的扩展。

如下图右图所示
![[Pasted image 20240108155303.png]]

jdk动态代理：代理类没有实现业务接口，只是目标对象实现了业务接口。上图JDK Proxy左边的那条implements是一个逻辑概念而不是实现方式

cglib特点

- JDK的动态代理有一个限制，就是使用动态代理的对象必须实现一个或多个接口。如果想代理没有实现接口的类，就可以使用CGLIB实现。
    
- CGLIB是一个强大的高性能的代码生成包，它可以在运行期扩展Java类与实现Java接口。它广泛的被许多AOP的框架使用，例如Spring AOP和dynaop，为他们提供方法的interception（拦截）。
    
- CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。不鼓励直接使用ASM，因为它需要你对JVM内部结构包括class文件的格式和指令集都很熟悉。
    

cglib与动态代理最大的区别就是

- 使用动态代理的对象必须实现一个或多个接口
    
- 使用cglib代理的对象则无需实现接口，达到代理类无侵入。
    

使用cglib需要引入[cglib的jar包](https://link.segmentfault.com/?enc=SiqrIHP%2FGoo6IkY6O4RorQ%3D%3D.tZ5JojnN%2BgG%2F5xHll5GoBAdglTQBcLBptMzRHyJN74%2BFRbqv7d1K%2B3XIFAEdf4JwsRYuuwjMJ3Qhlhinr%2FgoTf4s%2FG9SCKyCC3xU2QJmr4w%3D)，如果你已经有spring-core的jar包，则无需引入，因为spring中包含了cglib。

- cglib的Maven坐标
    

```Java
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.5</version>
</dependency>
```

举例：保存用户功能的动态代理实现

- 目标对象：UserDao
    

```Java
package com.cglib;

public class UserDao{

    public void save() {
        System.out.println("保存数据");
    }
}
```

- 代理对象：ProxyFactory
    

```Java
package com.cglib;

import java.lang.reflect.Method;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class ProxyFactory implements MethodInterceptor{

    private Object target;//维护一个目标对象
    public ProxyFactory(Object target) {
        this.target = target;
    }
    
    //为目标对象生成代理对象
    public Object getProxyInstance() {
        //工具类
        Enhancer en = new Enhancer();
        //设置父类
        en.setSuperclass(target.getClass());
        //设置回调函数
        en.setCallback(this);
        //创建子类对象代理
        return en.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("开启事务");
        // 执行目标对象的方法
        Object returnValue = method.invoke(target, args);
        System.out.println("关闭事务");
        return null;
    }
}
```

- 测试类：TestProxy
    

```Java
package com.cglib;

import org.junit.Test;

public class TestProxy {

    @Test
    public void testCglibProxy(){
        //目标对象
        UserDao target = new UserDao();
        System.out.println(target.getClass());
        //代理对象
        UserDao proxy = (UserDao) new ProxyFactory(target).getProxyInstance();
        System.out.println(proxy.getClass());
        //执行代理对象方法
        proxy.save();
    }
}
```

- 输出结果
    

```Java
class com.cglib.UserDao
class com.cglib.UserDao$$EnhancerByCGLIB$$552188b6
开启事务
保存数据
关闭事务
```

## 代理模式在 AOP 中的应用

Spring事务基于AOP实现，AOP又是基于动态代理实现

AOP(Aspect-Oriented Programming:面向切面编程)能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

Spring AOP 就是基于动态代理的，如果要代理的对象，实现了某个接口，那么Spring AOP会使用JDK Proxy，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用 Cglib 生成一个被代理对象的子类来作为代理
# 观察者模式（Spring监听器）

观察者模式是一种对象行为型模式。它表示的是一种对象与对象之间具有依赖关系，当一个对象发生改变的时候，这个对象所依赖的对象也会做出反应。Spring 事件驱动模型就是观察者模式很经典的一个应用。Spring 事件驱动模型非常有用，在很多场景都可以解耦我们的代码。比如我们每次添加商品的时候都需要重新更新商品索引，这个时候就可以利用观察者模式来解决这个问题。

## Spring 事件驱动模型中的四种角色

### 事件角色Event

ApplicationEvent (org.springframework.context包下)充当事件的角色,这是一个抽象类，它继承了java.util.EventObject并实现了 java.io.Serializable接口。Spring 中默认存在以下事件，他们都是对 `ApplicationContextEvent` 的实现(继承自`ApplicationContextEvent`)：

- ContextStartedEvent：ApplicationContext 启动后触发的事件;
    
- ContextStoppedEvent：ApplicationContext 停止后触发的事件;
    
- ContextRefreshedEvent：ApplicationContext 初始化或刷新完成后触发的事件;
    
- ContextClosedEvent：ApplicationContext 关闭后触发的事件。
    
![[Pasted image 20240108155436.png]]
![[Pasted image 20240108155443.png]]

### 事件监听者角色Listener

ApplicationListener 充当了事件监听者角色，它是一个接口，里面只定义了一个 onApplicationEvent（）方法来处理ApplicationEvent。ApplicationListener接口类源码如下，可以看出接口定义看出接口中的事件只要实现了 ApplicationEvent就可以了。所以，在 Spring中我们只要实现 ApplicationListener 接口的 onApplicationEvent() 方法即可完成监听事件

```Java
package org.springframework.context;
import java.util.EventListener;
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}
```
### 广播器Multicaster
向Multicaster中添加所有的监听器lintener，以便于将事件通知到每一个监听器
### 事件发布者角色Publisher

ApplicationEventPublisher 充当了事件的发布者，它也是一个接口。

```Java
@FunctionalInterface
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {
        this.publishEvent((Object)event);
    }

    void publishEvent(Object var1);
}
```

ApplicationEventPublisher 接口的publishEvent（）这个方法在AbstractApplicationContext类中被实现，阅读这个方法的实现，你会发现实际上事件真正是通过ApplicationEventMulticaster来广播出去的。具体内容过多，就不在这里分析了，后面可能会单独写一篇文章提到。

 Spring 的事件流程总结

1. 定义一个事件: 实现一个继承自 ApplicationEvent，并且写相应的构造函数；
    
2. 定义一个事件监听者：实现 ApplicationListener 接口，重写 onApplicationEvent() 方法；
    
3. 使用事件发布者发布消息: 可以通过 ApplicationEventPublisher 的 publishEvent() 方法发布消息。==AnnotationConfigApplicationContext默认实现了ApplicationEventPublisher，因此可以直接利用context来调用publishEvent() 方法发布消息==
    

Example:

```Java
// 定义一个事件,继承自ApplicationEvent并且写相应的构造函数
public class DemoEvent extends ApplicationEvent{
    private static final long serialVersionUID = 1L;

    private String message;

    public DemoEvent(Object source,String message){
        super(source);
        this.message = message;
    }

    public String getMessage() {
         return message;
          }
}
    
// 定义一个事件监听者,实现ApplicationListener接口，重写 onApplicationEvent() 方法；
@Component
public class DemoListener implements ApplicationListener<DemoEvent>{

    //使用onApplicationEvent接收消息
    @Override
    public void onApplicationEvent(DemoEvent event) {
        String msg = event.getMessage();
        System.out.println("接收到的信息是："+msg);
    }

}
// 发布事件，可以通过ApplicationEventPublisher  的 publishEvent() 方法发布消息。
@Component
public class DemoPublisher {

    @Autowired
    ApplicationContext applicationContext;

    public void publish(String message){
        //发布事件
        applicationContext.publishEvent(new DemoEvent(this, message));
    }
}
```

当调用 DemoPublisher 的 publish() 方法的时候，比如 demoPublisher.publish("你好") ，控制台就会打印出:接收到的信息是：你好 。