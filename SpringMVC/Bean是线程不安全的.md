Spring 的 bean 作用域（scope）类型有5种：

- singleton：单例，默认作用域。==对于单例Bean，所有线程都共享一个单例实例Bean，因此有可能会存在资源的竞争。==
    
- prototype：原型，每次创建一个新对象。对于原型Bean，每次创建一个新对象，也就是线程之间并不存在Bean共享，自然是不会有线程安全的问题。
    
- request：请求，每次Http请求创建一个新对象，适用于WebApplicationContext环境下。
    
- session：会话，同一个会话共享一个实例，不同会话使用不同的实例。
    
- global-session：全局会话，所有会话共享一个实例。
    

Spring的根本就是通过大量这种单例Bean构建起系统，如果单例Bean是一个无状态Bean，那么这个单例Bean是线程安全的，因为Java虚拟机栈是线程私有的，每个方法在执行的同时都会在Java虚拟机上创建一个栈帧，栈帧用于存储方法的局部变量表（非全局共享）、操作数栈、动态链接、方法出口等信息。==比如Mapper一般就是无状态的，因为我们不会在Mapper里添加全局共享变量==

> 无状态就是不会保存数据
> 
> 有状态就是有数据存储功能

但如果单例Bean是一个有状态Bean，比如我们很可能会在Controller和Service中或者在UserBean中创建全局的共享变量，那么此时Controller和Service或者UserBean就会变得不安全！



但很多场景下我们要操作单实例Bean中的共享变量！！！因此很多场景下Bean将会是有状态的，那就需要开发人员自己来进行线程安全的保证，解决办法有两种：

- 改变bean的作用域 把 `singleton` 改为 `protopyte`， 这样每次请求Bean就相当于是 new Bean() 这样就可以保证线程的安全了。
    
- 或者声明ThreadLocal类型的共享变量

# @Controller和Service不安全分析

在单例模式下我们在Controller中定义五种类型的变量：

- 普通变量
    
- 静态变量
    
- 配置变量
    
- ThreadLocal变量
    
- 注入对象User中的变量age
    
## 声明安全的ThreadLocal变量
```Java
  
/**  
 * @Author: ly  
 * @Date: 2024/1/8 16:52  
 */@RestController  
//@Scope(value = "prototype") // prototype singleton  
public class UnsafeBeanController {  
    private int var = 0; // 定义一个普通变量  
  
    private static int staticVar = 0; // 定义一个静态变量  
  
    @Value("${test-int}")  
    private int testInt; // 从配置文件中读取变量  
  
    ThreadLocal<Integer> tl = new ThreadLocal<>(); // 用ThreadLocal来封装变量  
  
    @Autowired  
    private User user; // 注入一个对象来封装变量  
  
  
    @GetMapping(value = "/test_var")  
    public String test() {  
  
        System.out.println("Before"+"普通变量var:" + var + "===静态变量staticVar:" + staticVar + "===配置变量testInt:" + testInt  
                + "===ThreadLocal变量tl:" + tl.get() + "===注入变量user:" + user.getAge());  
        tl.set(1);  
        user.setAge(1);  
        ++var;  
        ++staticVar;  
        ++testInt;  
  
  
        System.out.println("After"+"普通变量var:" + var + "===静态变量staticVar:" + staticVar + "===配置变量testInt:" + testInt  
                + "===ThreadLocal变量tl:" + tl.get() + "===注入变量user:" + user.getAge());  
  
  
  
        return null;  
    }  
}


/**  
 * @Author: ly  
 * @Date: 2024/1/8 16:53  
 */@Configuration  
  
public class MyConfig {  
    @Bean  
//    @Scope(value = "prototype")  
    public User user(){  
        return new User();  
    }  
}

/**  
 * @Author: ly  
 * @Date: 2024/1/8 16:53  
 */@Data  
public class User {  
    private Integer age;  
}

//application.properties
test-int=0
```

我暂时能想到的定义变量的方法就这么多了，两次http请求结果如下：可以看到，在单例模式下Controller中的2次请求结果里面
- ThreadLocal变量值每次都是从0+1=1的，
- 普通变量、配置变量、静态变量都是累加的，
- user对象的age默认值是0，第二次取值的时候就已经是1了，说明每次请求调用的都是同一个user对象。

```Java

Before普通变量var:0===静态变量staticVar:0===配置变量testInt:0===ThreadLocal变量tl:null===注入变量user:null
After普通变量var:1===静态变量staticVar:1===配置变量testInt:1===ThreadLocal变量tl:1===注入变量user:1

Before普通变量var:1===静态变量staticVar:1===配置变量testInt:1===ThreadLocal变量tl:null===注入变量user:1
After普通变量var:2===静态变量staticVar:2===配置变量testInt:2===ThreadLocal变量tl:1===注入变量user:1
```

## 将Bean作用域都改为prototype
下面将TestController 和MyConfig配置类中User 上的@Scope注解的属性都改成多实例的：@Scope(value = "prototype")

```Java
  
/**  
 * @Author: ly  
 * @Date: 2024/1/8 16:52  
 */@RestController  
@Scope(value = "prototype") // prototype singleton  
public class UnsafeBeanController {  
    private int var = 0; // 定义一个普通变量  
  
    private static int staticVar = 0; // 定义一个静态变量  
  
    @Value("${test-int}")  
    private int testInt; // 从配置文件中读取变量  
  
    ThreadLocal<Integer> tl = new ThreadLocal<>(); // 用ThreadLocal来封装变量  
  
    @Autowired  
    private User user; // 注入一个对象来封装变量  
  
  
    @GetMapping(value = "/test_var")  
    public String test() {  
  
        System.out.println("Before"+"普通变量var:" + var + "===静态变量staticVar:" + staticVar + "===配置变量testInt:" + testInt  
                + "===ThreadLocal变量tl:" + tl.get() + "===注入变量user:" + user.getAge());  
        tl.set(1);  
        user.setAge(1);  
        ++var;  
        ++staticVar;  
        ++testInt;  
  
  
        System.out.println("After"+"普通变量var:" + var + "===静态变量staticVar:" + staticVar + "===配置变量testInt:" + testInt  
                + "===ThreadLocal变量tl:" + tl.get() + "===注入变量user:" + user.getAge());  
  
  
  
        return null;  
    }  
}


/**  
 * @Author: ly  
 * @Date: 2024/1/8 16:53  
 */@Configuration  
  
public class MyConfig {  
    @Bean  
    @Scope(value = "prototype")  
    public User user(){  
        return new User();  
    }  
}
```

再次请求结果如下：
- 普通变量从不安全变为安全
- 配置变量从不安全变为安全
- 每次请求赋值前取user都是null，说明每次请求都是新的user。

至于静态变量staticVar，虽然每次都是单独创建一个prototype Controller但是扛不住他变量本身是static的呀，所以说呢，即便是加上@Scope注解也不一定能保证Controller 100%的线程安全。

```Java
Before普通变量var:0===静态变量staticVar:0===配置变量testInt:0===ThreadLocal变量tl:null===注入变量user:null
After普通变量var:1===静态变量staticVar:1===配置变量testInt:1===ThreadLocal变量tl:1===注入变量user:1

Before普通变量var:0===静态变量staticVar:1===配置变量testInt:0===ThreadLocal变量tl:null===注入变量user:null
After普通变量var:1===静态变量staticVar:2===配置变量testInt:1===ThreadLocal变量tl:1===注入变量user:1
```