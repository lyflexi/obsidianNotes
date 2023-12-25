当被问到要实现一个单例模式时，很多人的第一反应是写出如下的代码，包括教科书上也是这样教我们的。

# 单线程单例模式

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

这段代码简单明了，而且使用了懒加载模式，但是却存在致命的问题。当有多个线程并行调用 getInstance() 的时候，就会创建多个实例。也就是说在多线程下不能正常工作。

# 同步版单例模式

为了解决上面的问题，最简单的方法是将整个 getInstance() 方法设为同步（synchronized）。

>`synchronized`的三大特性可以保证有序性，整个方法加`synchronized`，同时也获取了对象实例锁`singleton`，保证了单实例创建的有序性
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

虽然做到了线程安全，并且解决了多实例的问题，但是它并不高效。因为在任何时候只能有一个线程调用 getInstance() 方法。其实同步操作只需要在`uniqueInstance == null`时才被需要，即第一次创建单例实例对象时。

# 双重检验锁单例模式

这就引出了双重检验锁。==双重检验锁模式（double checked locking pattern），是一种使用同步块加锁的方法==。
## DoubleCheck
程序员称其为双重检查锁，因为会有两次检查 `instance == null`，一次是在同步块外，一次是在同步块内。为什么在同步块内还要再检验一次？因为可能会有多个线程一起进入同步块外的 if，如果在同步块内不进行二次检验的话就会生成多个实例了。

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
## volatile
另外，uniqueInstance 采用 volatile 关键字修饰也是很有必要的， 因为`synchronized (Singleton.class)`加锁语句并没有加在成员变量`uniqueInstance`上 

`uniqueInstance = new Singleton(); `这段代码其实是分为三步执行：

1. 为 uniqueInstance 分配内存空间（堆）
    
2. 调用构造器方法，初始化uniqueInstance（栈上引用变量）
    
3. 将 uniqueInstance 指向分配的内存地址
    

>Java 语言规规定了线程执行程序时需要遵守intra-thread semantics（线程内语义）。保证重排序不会改变单线程内的程序执行结果。这个重排序在没有改变单线程程序的执行结果的前提下，可以提高程序的执行性能。虽然重排序并不影响单线程内的执行结果，但是在多线程的环境就带来一些问题。


因此多线程环境下的执行顺序有可能变成 1->3->2，但是并不会重排序 1 的顺序。也就是说 1 这个指令都需要先执行，因为 2,3 指令需要依托 1 指令执行结果。

例如，线程 T1 执行了 1 和 3，此时 T2 调用 getUniqueInstance() 后发现 uniqueInstance 不为空，因此返回 uniqueInstance，但此时 uniqueInstance 还未被初始化。

使用 volatile 可以禁止 JVM 的指令重排，保证在多线程环境下也能正常运行。