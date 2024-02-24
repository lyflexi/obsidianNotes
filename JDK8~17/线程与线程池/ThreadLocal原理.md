ThreadLocal提供了线程变量的本地副本，对于线程共享变量如果使用ThreadLocal则无需加锁，从而避免了线程安全问题。可以使用 `get` 和 `set` 方法来获取默认值或将其值更改为当前线程的副本值。

- Thread中持有ThreadLocalMap引用，其key是ThreadLocal ，而不同的value才是最终的值。必要的话，你可以同时申请多个ThreadLocal本地变量。
- 线程池会复用池中的线程，因此当前线程中的ThreadLocal可能会被复用，所以使用完ThreadLocal之后要及时手动remove，避免影响后续业务逻辑
- 多个线程Thread也有可能共用一个ThreadLocal 作为key。但是相同的key并不会覆盖`ThreadLocalMap`当中Entry，因为多个Thread并不会公用一个`ThreadLocalMap`，而是有几个Thread就会对应几个`ThreadLocalMap`

![[Pasted image 20231225163711.png]]

# ThreadLocal 源码

## Thread类源码#仅仅定义HashMap

```Java
public class Thread implements Runnable {
     ......
    //与此线程有关的ThreadLocal值。由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
    //与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}
```

从上面Thread类 源代码可以看出Thread 类中有一个 threadLocals 和 一个 inheritableThreadLocals 变量，它们都是 ThreadLocalMap 类型的变量，而ThreadLocalMap为ThreadLocal 类中定制化的 HashMap（内部类）。

默认情况下这两个变量都是 null，只有当前线程调用 ThreadLocal 类的 set或get方法时才创建它们。

## ThreadLocal类源码#初始化HashMap

ThreadLocalMap懒加载：ThreadLocal 在自己的getset方法内对ThreadLocalMap进行初始化 ，以及对value进行装填
```Java
   public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }


    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }   

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }

    protected T initialValue() {
        return null;
    }
```

## 线程复用导致ThreadLocal可能存在的问题

【强制】必须回收自定义的ThreadLocal变量，尤其在线程池场景下，线程经常会被复用，尽量在try-finally块中使用remove回收ThreadLocal变量。避免影响后续业务逻辑或者造成内存泄露问题

### 避免影响后续业务逻辑

```Java
package com.bilibili.juc.tl;


import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

class MyData
{
    ThreadLocal<Integer> threadLocalField = ThreadLocal.withInitial(() -> 0);
    public void add()
    {
        threadLocalField.set(1 + threadLocalField.get());
    }
}

/**
 * @auther zzyy
.【强制】必须回收自定义的 ThreadLocal 变量，尤其在线程池场景下，线程经常会被复用，如果不清理
自定义的 ThreadLocal 变量，可能会影响后续业务逻辑和造成内存泄露等问题。尽量在代理中使用
try-finally 块进行回收。
 */
public class ThreadLocalDemo2
{
    public static void main(String[] args) throws InterruptedException
    {
        MyData myData = new MyData();

        ExecutorService threadPool = Executors.newFixedThreadPool(3);

        try
        {
            for (int i = 0; i < 10; i++) {
                threadPool.submit(() -> {
                    try {
                        Integer beforeInt = myData.threadLocalField.get();
                        myData.add();
                        Integer afterInt = myData.threadLocalField.get();
                        System.out.println(Thread.currentThread().getName()+"\t"+"beforeInt:"+beforeInt+"\t afterInt: "+afterInt);
                    } finally {
                        myData.threadLocalField.remove();
                    }
                });
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            threadPool.shutdown();
        }

    }
}

//忘记执行remove的控制台输出
com.bilibili.juc.tl.ThreadLocalDemo2
pool-1-thread-3        beforeInt:0         afterInt: 1
pool-1-thread-2        beforeInt:0         afterInt: 1
pool-1-thread-1        beforeInt:0         afterInt: 1
pool-1-thread-3        beforeInt:1         afterInt: 2
pool-1-thread-1        beforeInt:1         afterInt: 2
pool-1-thread-2        beforeInt:1         afterInt: 2
pool-1-thread-1        beforeInt:2         afterInt: 3
pool-1-thread-1        beforeInt:3         afterInt: 4
pool-1-thread-3        beforeInt:2         afterInt: 3
pool-1-thread-2        beforeInt:2         afterInt: 3

Process finished with exit code 0
//执行remove的控制台输出
pool-1-thread-1	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-2	beforeInt:0	 afterInt: 1
pool-1-thread-3	beforeInt:0	 afterInt: 1
pool-1-thread-1	beforeInt:0	 afterInt: 1
```

### 避免造成内存泄露问题

每个Thread都有一个ThreadLocal.ThreadLocalMap类型的HashMap，该Map的key为ThreadLocal实例，他是一个弱引用，我们清楚弱引用有利于GC回收。当ThreadLocal作为Map的key为null的时候，GC就会回收这一部分空间。
![[Pasted image 20231225163736.png]]

但是value却不一定能够被回收，ThreadLocalMap还与Current Thread存在一个强引用关系（关联关系）
![[Pasted image 20231225163743.png]]

因为对于线程池需要复用池中的线程，所以会导致线程对象迟迟不会结束。假如我们不做任何措施的话，ThreadLocalMap 中就会出现很多 key 为 null 的 Entry，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。如下图所示：
![[Pasted image 20231225163748.png]]

ThreadLocalMap 实现中已经考虑了这种情况，使用完 ThreadLocal方法后最好手动调用remove()方法会清理掉 key 为 null 的整个Entry。

# 强软弱虚四大引用扩展讲解
![[Pasted image 20231225163755.png]]
![[Pasted image 20231225163800.png]]

==注意，finalize()是在gc将对象从内存中清除出去**之前**做的操作==

```Java

protected void finalize() throws Throwable { }
```

## 强引用：最常见
强引用是我们最常见的普通对象引用，只要还有强引用指向一个对象,就能表明对象还“活着”，垃圾收集器不会碰这种对象。 在Java 中最常见的就是强引用是把一个对象赋给一个引用变量，那么这个引用变量就是一个强引用。 当一个对象被强引用变量引用时，它处于可达状态，它是不可能被垃圾回收机制回收的，即使该对象以后永远都不会被用到，JVM也不会回收。

因此强引用是造成Java内存泄漏的主要原因之一。

对于一个普通的对象，只能是超过了引用的作用域或者显式地将相应(强)引用赋值为 null才是可以被垃圾收集的了(当然具体回收时机还是要看垃圾收集策略)。

```Java
package com.bilibili.juc.tl;

import java.lang.ref.*;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

class MyObject
{
    //这个方法一般不用复写，我们只是为了教学给大家演示案例做说明
    @Override
    protected void finalize() throws Throwable
    {
        // finalize的通常目的是在对象被不可撤销地丢弃之前执行清理操作。
        System.out.println("-------invoke finalize method~!!!");
    }
}


/**
 * @auther zzyy
 */
public class ReferenceDemo
{
    public static void main(String[] args)
    {
        MyObject myObject = new MyObject();
        System.out.println("gc before: "+myObject);

        myObject = null;
        System.gc();//人工开启GC，一般不用

        //暂停毫秒
        try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println("gc after: "+myObject);

    }
}
```
打印信息：
```shell
gc before: org.lyflexi.reference.MyObject@5cad8086
-------invoke finalize method~!!!
gc after: null

Process finished with exit code 0
```

## 软引用SoftReference：看剩余内存

软引用是一种相对强引用弱化了一些的引用,需要用java.lang.ref.SoftReference类来实现,可以让对象豁免一些垃圾收集。

对于只有软引用的对象来说，当系统内存充足时它不会被回收，只有当系统内存不足时它 会被回收。

`-Xmx`、`-Xms` 和 `-Xmn` 是 Java 虚拟机（JVM）参数，用于调整 JVM 的堆内存大小和分配策略。它们的作用和区别如下：
1. `-Xmx`: 这个参数设置了 Java 堆的最大内存大小。
2. `-Xms`: 这个参数设置了 JVM 启动时分配给 Java 堆的初始内存大小。
3. `-Xmn`: 这个参数设置了新生代（Young Generation）的大小。
![[Pasted image 20240224103801.png]]

```Java
package com.bilibili.juc.tl;

import java.lang.ref.*;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

class MyObject
{
    //这个方法一般不用复写，我们只是为了教学给大家演示案例做说明
    @Override
    protected void finalize() throws Throwable
    {
        // finalize的通常目的是在对象被不可撤销地丢弃之前执行清理操作。
        System.out.println("-------invoke finalize method~!!!");
    }
}


public class SoftReferenceTest {  
  
    public static void main(String[] args) {  
        SoftReference<MyObject> softReference = new SoftReference<>(new MyObject());  
        System.out.println("-----softReference:"+softReference.get());  
  
        System.gc();  
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }  
        System.out.println("-----gc after内存够用: "+softReference.get());  
  
        try  
        {  
            byte[] bytes = new byte[20 * 1024 * 1024];//20MB对象  
        }catch (Exception e){  
            e.printStackTrace();  
        }finally {  
            System.out.println("-----gc after内存不够: "+softReference.get());  
        }  
    }  
}
```
打印信息：
```shell
-----softReference:org.lyflexi.reference.MyObject@5cad8086
-----gc after内存够用: org.lyflexi.reference.MyObject@5cad8086
-----gc after内存不够: null
-------invoke finalize method~!!!
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at org.lyflexi.reference.SoftReferenceTest.main(SoftReferenceTest.java:22)
```
## 弱引用WeakReference：无法豁免如ThreadLocal

如果一个对象只具有弱引用，那就类似于**可有可无的生活用品**。弱引用与软引用的区别在于：弱引用比软引用的生存期更短。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。

```Java
package com.bilibili.juc.tl;

import java.lang.ref.*;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

class MyObject
{
    //这个方法一般不用复写，我们只是为了教学给大家演示案例做说明
    @Override
    protected void finalize() throws Throwable
    {
        // finalize的通常目的是在对象被不可撤销地丢弃之前执行清理操作。
        System.out.println("-------invoke finalize method~!!!");
    }
}


/**  
 * @Author: ly  
 * @Date: 2024/2/24 10:41  
 */public class WeakReferenceTest {  
    public static void main(String[] args) {  
        WeakReference<MyObject> weakReference = new WeakReference<>(new MyObject());  
        System.out.println("-----gc before 内存够用： "+weakReference.get());  
  
        System.gc();  
        //暂停几秒钟线程  
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }  
  
        System.out.println("-----gc after 内存够用： "+weakReference.get());  
    }  
}
```
打印信息：
```shell
-----gc before 内存够用： org.lyflexi.reference.MyObject@5cad8086
-------invoke finalize method~!!!
-----gc after 内存够用： null

Process finished with exit code 0
```

## 虚引用PhantomReference：搭配引用队列
虚引用PhantomReference不能单独使用，必须和引用队列 (ReferenceQueue)联合使用。与其他几种引用都不同，虚引用并不会决定对象的生命周期，也不能通过它访问对象。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。对象被回收之后，有东西将被装入引用队列 (ReferenceQueue)中。gc之后！！！
- PhantomReference的get方法总是返回null：无法访问对应的引用对象，虚引用的主要作用是跟踪对象被垃圾回收的状态。
- 通知机制：虚引用仅仅是在对象被gc以后，做某些事情的通知机制。对比finalize()，finalize()是gc之前生效
同样配置jvm参数，-Xms10 -Xmx10m
![[Pasted image 20240224113244.png]]
测试用例：
```Java
public class PhantomReferenceTest {  
    private static final List<Object> TEST_DATA = new LinkedList<>();  
    private static final ReferenceQueue<MyObject> QUEUE = new ReferenceQueue<>();  
  
    public static void main(String[] args) {  
        MyObject obj = new MyObject();  
        PhantomReference<MyObject> phantomReference = new PhantomReference<>(obj, QUEUE);  
  
        // 该线程不断读取这个虚引用，并不断往列表里插入数据，以促使系统早点进行GC  
        new Thread(() -> {  
            while (true) {  
                TEST_DATA.add(new byte[1024 * 1000]);//1m  
                try {  
                    Thread.sleep(1000);  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                    Thread.currentThread().interrupt();  
                }  
                System.out.println(phantomReference.get());  
            }  
        }).start();  
  
        // 这个线程不断读取引用队列，当弱引用指向的对象呗回收时，该引用就会被加入到引用队列中  
        new Thread(() -> {  
            while (true) {  
                Reference<? extends MyObject> poll = QUEUE.poll();  
                if (poll != null) {  
                    System.out.println("--- 虚引用对象被jvm回收了 ---- " + poll);  
                    System.out.println("--- 回收对象 ---- " + poll.get());  
                }  
            }  
        }).start();  
  
        obj = null;  
  
        try {  
            Thread.currentThread().join();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
            System.exit(1);  
        }  
    }  
  
  
}
```
打印信息：
```shell
null
null
null
null
null
-------invoke finalize method~!!!
null
null
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at org.lyflexi.reference.PhantomReferenceTest.lambda$main$0(PhantomReferenceTest.java:30)
	at org.lyflexi.reference.PhantomReferenceTest$$Lambda$1/1452126962.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:750)
--- 虚引用对象被jvm回收了 ---- java.lang.ref.PhantomReference@25f209f5
--- 回收对象 ---- null
```

![[Pasted image 20231225163839.png]]

# ThreadLocal实践

## 应用案例一：SimpleDateFormat

SimpleDateFormat 是线程不安全的类，阿里开发手册推荐SimpleDateFormat 的正确用法是配合ThreadLocal类保证线程安全：

```Java
import java.text.SimpleDateFormat;
import java.util.Random;

public class ThreadLocalExample implements Runnable{
    private static final ThreadLocal<SimpleDateFormat> formatter = ThreadLocal.withInitial(
        () -> new SimpleDateFormat("yyyyMMdd HHmm")
    );

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample obj = new ThreadLocalExample();
        for(int i=0 ; i<10; i++) {
            Thread t = new Thread(obj, ""+i);
            Thread.sleep(new Random().nextInt(1000));
            t.start();
        }
    }

    @Override
    public void run() {
        System.out.println("Thread Name= "+Thread.currentThread().getName()+" default Formatter = "+formatter.get().toPattern());
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //formatter pattern is changed here by thread, but it won't reflect to other threads
        formatter.set(new SimpleDateFormat());

        System.out.println("Thread Name= "+Thread.currentThread().getName()+" formatter = "+formatter.get().toPattern());
    }

}
```

从输出中可以看出，Thread-0 已经改变了 formatter 的值，但仍然是 thread-1/thread-2/thread-3... 默认格式化程序与初始化值相同。

```Java
//输出
Thread Name= 0 default Formatter = yyyyMMdd HHmm
Thread Name= 0 formatter = yy-M-d ah:mm
Thread Name= 1 default Formatter = yyyyMMdd HHmm
Thread Name= 2 default Formatter = yyyyMMdd HHmm
Thread Name= 1 formatter = yy-M-d ah:mm
Thread Name= 3 default Formatter = yyyyMMdd HHmm
Thread Name= 2 formatter = yy-M-d ah:mm
Thread Name= 4 default Formatter = yyyyMMdd HHmm
Thread Name= 3 formatter = yy-M-d ah:mm
Thread Name= 4 formatter = yy-M-d ah:mm
Thread Name= 5 default Formatter = yyyyMMdd HHmm
Thread Name= 5 formatter = yy-M-d ah:mm
Thread Name= 6 default Formatter = yyyyMMdd HHmm
Thread Name= 6 formatter = yy-M-d ah:mm
Thread Name= 7 default Formatter = yyyyMMdd HHmm
Thread Name= 7 formatter = yy-M-d ah:mm
Thread Name= 8 default Formatter = yyyyMMdd HHmm
Thread Name= 9 default Formatter = yyyyMMdd HHmm
Thread Name= 8 formatter = yy-M-d ah:mm
Thread Name= 9 formatter = yy-M-d ah:mm
```

## 应用案例二：每个销售员可以出售多少套房子

```Java
package com.bilibili.juc.tl;

import lombok.Getter;
import sun.font.FontRunIterator;

import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

class House //资源类
{
    int saleCount = 0;
    public synchronized void saleHouse()
    {
        ++saleCount;
    }

    ThreadLocal<Integer> saleVolume = ThreadLocal.withInitial(() -> 0);
    public void saleVolumeByThreadLocal()
    {
        saleVolume.set(1+saleVolume.get());
    }
}

/**
 * @auther zzyy
 * @create 2021-12-31 15:46
 *
 * 需求1： 5个销售卖房子，集团高层只关心销售总量的准确统计数。
 *
 * 需求2： 5个销售卖完随机数房子，各自独立销售额度，自己业绩按提成走，分灶吃饭，各个销售自己动手，丰衣足食
 *
 *
 */
public class ThreadLocalDemo
{
    public static void main(String[] args) throws InterruptedException
    {

        House house = new House();

        for (int i = 1; i <=5; i++) {
            new Thread(() -> {
                int size = new Random().nextInt(5)+1;//每个销售随机卖1~5件商品
                try {
                    for (int j = 1; j <=size; j++) {
                        house.saleHouse();
                        house.saleVolumeByThreadLocal();
                    }
                    System.out.println(Thread.currentThread().getName()+"\t"+"号销售卖出："+house.saleVolume.get());
                } finally {
                    house.saleVolume.remove();
                }
            },String.valueOf(i)).start();
        };

        //暂停毫秒
        try { TimeUnit.MILLISECONDS.sleep(300); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println(Thread.currentThread().getName()+"\t"+"共计卖出多少套：saleCount "+house.saleCount);
        System.out.println(Thread.currentThread().getName()+"\t"+"共计卖出多少套：saleVolume "+house.saleVolume.get());
    }
}
```

输出：

```Java
com.bilibili.juc.tl.ThreadLocalDemo
4        号销售卖出：2
1        号销售卖出：5
2        号销售卖出：5
3        号销售卖出：2
5        号销售卖出：2
main        共计卖出多少套：saleCount 16
main        共计卖出多少套：saleVolume 0

Process finished with exit code 0
```