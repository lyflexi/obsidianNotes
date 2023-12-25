ThreadLocal提供了线程变量的本地副本，对于线程共享变量如果使用ThreadLocal则无需加锁，同时也更省事省心。可以使用 `get` 和 `set` 方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。

- ThreadLocalMap 的key是ThreadLocal ，多个线程Thread可以共用一个ThreadLocal 作为key，而不同的value才是最终的值。
    
- 相同的key并不会产生`ThreadLocalMap`当中Entry覆盖的问题，因为多个Thread并不会公用一个`ThreadLocalMap`，而是有几个Thread就会对应几个`ThreadLocalMap`
    
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

从上面Thread类 源代码可以看出Thread 类中有一个 threadLocals 和 一个 inheritableThreadLocals 变量，它们都是 ThreadLocalMap 类型的变量，我们可以把 ThreadLocalMap 理解为ThreadLocal 类实现的定制化的 HashMap。

默认情况下这两个变量都是 null，只有当前线程调用 ThreadLocal 类的 set或get方法时才创建它们。

## ThreadLocal类源码#初始化HashMap

ThreadLocal 在自己的getset方法内对ThreadLocalMap进行初始化 ，以及对value进行装填

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

//输出
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
```

### 避免造成内存泄露问题

每个Thread都有一个ThreadLocal.ThreadLocalMap类型的HashMap，该Map的key为ThreadLocal实例，他是一个弱引用，我们清楚弱引用有利于GC回收。当ThreadLocal作为Map的key为null的时候，GC就会回收这一部分空间。
![[Pasted image 20231225163736.png]]

但是value却不一定能够被回收，ThreadLocalMap还与Current Thread存在一个强引用关系（依赖关系）
![[Pasted image 20231225163743.png]]

因为对于线程池当中的线程会被复用，有可能会导致线程对象迟迟不会结束。假如我们不做任何措施的话，ThreadLocalMap 中就会出现很多 key 为 null 的 Entry，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。如下图所示：
![[Pasted image 20231225163748.png]]

ThreadLocalMap 实现中已经考虑了这种情况，使用完 ThreadLocal方法后最好手动调用remove()方法会清理掉 key 为 null 的记录。

# 强软弱虚四大引用扩展讲解
![[Pasted image 20231225163755.png]]
![[Pasted image 20231225163800.png]]

注意，是在gc将对象从内存中清除出去**之前**做的操作

```Java

protected void finalize() throws Throwable { }
```

## 强引用：最常见

当内存不足,JVM开始垃圾回收,对于强引用的对象,就算是出现了OOM也不会对该对象进行回收,机器死了都不收。

**强引用是我们最常见的普通对象引用**,只要还有强引用指向一个对象,就能表明对象还“活着”,垃圾收集器不会碰这种对象。 在Java 中最常见的就是强引用,把一个对象赋给一个引用变量,这个引用变量就是一个强引用。 当一个对象被强引用变量引用时,它处于可达状态,它是不可能被垃圾回收机制回收的, 即使该对象以后永远都不会被用到,JVM也不会回收。

因此强引用是造成Java内存泄漏的主要原因之一。

对于一个普通的对象,如果没有其他的引用关系,只要超过了引用的作用域或者显式地将相应(强)引用赋值为 null一般认为才是可以被垃圾收集的了(当然具体回收时机还是要看垃圾收集策略)。

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

## 软引用：看情况

软引用是一种相对强引用弱化了一些的引用,需要用java.lang.ref.SoftReference类来实现,可以让对象豁免一些垃圾收集。

对于只有软引用的对象来说：

- 当系统内存充足时它 不会 被回收
    
- 当系统内存不足时它 会被回收。
    

软引用通常用在对内存敏感的程序中,比如高速缓存就有用到软引用,内存够用的时候就保留,不够用就回收!
![[Pasted image 20231225163817.png]]

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
 /
public class ReferenceDemo
{
    public static void main(String[] args)
        SoftReference<MyObject> softReference = new SoftReference<>(new MyObject());
        //System.out.println("-----softReference:"+softReference.get());

        System.gc();
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("-----gc after内存够用: "+softReference.get());

        try
        {
            byte[] bytes = new byte[20  1024 * 1024];//20MB对象
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            System.out.println("-----gc after内存不够: "+softReference.get());
        }

    }


}
```

假如有一个应用需要读取大量的本地图片 :

我 如果每次读取图片都从硬盘读取则会严重影响性能,

- 如果一次性全部加载到内存中又可能造成内存溢出。
    

此时使用软引用可以解决这个问题。 设计思路是 : 用一个HashMap来保存图片的路径和相应图片对象关联的软引用之间的映射关系,在内存不足时, JVM会自动回收这些缓存图片对象所占用的空间,从而有效地避免了OOM的问题。

```Java
Map<StringSoftReference<Bitmap>> imageCache = new HashMap<String, SoftReference<Bitmap>>();
```

## 弱引用：无法豁免

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
 * @auther zzyy
 */
public class ReferenceDemo
{
    public static void main(String[] args)
    {
        WeakReference<MyObject> weakReference = new WeakReference<>(new MyObject());
        System.out.println("-----gc before 内存够用： "+weakReference.get());

        System.gc();
        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println("-----gc after 内存够用： "+weakReference.get());
    }
}
```

## 虚引用

- **引用队列：**虚引用PhantomReference不能单独使用，必须和引用队列 (ReferenceQueue)联合使用。与其他几种引用都不同，虚引用并不会决定对象的生命周期，也不能通过它访问对象。如果一个对象仅持有虚引用，那么它就和没有任何引用一样,在任何时候都可能被垃圾回收器回收。对象被回收之后，有东西将被装入引用队列 (ReferenceQueue)中。**gc之后！！！**
    
- **总是返回null：**PhantomReference的get方法总是返回null 虚引用的主要作用是跟踪对象被垃圾回收的状态。 PhantomReference的get方法总是返回null,因此无法访问对应的引用对象
    
- **通知机制：**虚引用仅仅是提供了一种确保对象被 finalize以后,做某些事情的通知机制。处理监控通知使用，换句话说,设置虚引用关联对象的唯一目的,就是在这个对象被收集器回收的时候收到一个系统通知或者后续添加进一步的处理,用 来实现比finalize机制更灵活的回收操作
![[Pasted image 20231225163829.png]]

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
 /
public class ReferenceDemo
{
    public static void main(String[] args)
    {
        MyObject myObject = new MyObject();
        ReferenceQueue<MyObject> referenceQueue = new ReferenceQueue<>();
        PhantomReference<MyObject> phantomReference = new PhantomReference<>(myObject,referenceQueue);
        //System.out.println(phantomReference.get());

        List<byte[]> list = new ArrayList<>();

        new Thread(() -> {
            while (true){
                list.add(new byte[1  1024 * 1024]);
                try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
                System.out.println(phantomReference.get()+"\t"+"list add ok");
            }
        },"t1").start();

        new Thread(() -> {
            while (true){
                Reference<? extends MyObject> reference = referenceQueue.poll();
                if(reference != null){
                    System.out.println("-----有虚对象回收加入了队列");
                    break;
                }
            }
        },"t2").start();

    }
}
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