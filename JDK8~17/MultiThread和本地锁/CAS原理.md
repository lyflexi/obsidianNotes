使用AtomicInteger之后，不需要对方法加锁，也可以实现i++线程安全。

`AtomicInteger`类部分源码：

```Java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
public final void lazySet(int newValue)//最终设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

`AtomicInteger`应用实例：

```Java

class AtomicIntegerTest {
    private AtomicInteger count = new AtomicInteger();
    //使用AtomicInteger之后，不需要对该方法加锁，也可以实现线程安全。
    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

`Atomic`原子类的原理就是乐观锁的实现CAS，`Atomic`原子类主要利用和 `UnSafe` 类当中的两个native 方法`objectFieldOffset`和`compareAndSwapInt`和一个自旋操作`getAndAddInt`来保证原子操作，从而避免 synchronized 的高开销，执行效率大为提升。

```Java
//传入反射类型Filed，返回要更新的变量V的内存地valueOffset。
public native long objectFieldOffset(Field var1);
/**
var1：待更新的变量V也即this，
var2：V的内存地址valueOffset，
var4：期望值E，
var5：更新值U
*/
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
//自旋操作
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

  

  

# 原子类源码分析

## AtomicX调用Unsafe的objectFieldOffset

```Java
//部分源码
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
   private volatile int value;//原子类声明了volatile的变量value（也即目标变量V），因此V在内存中可见。
   ....
}
```

## AtomicX调用Unsafe的compareAndSet

```Java
    ...
        public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    ...
```

CAS的全称是：比较并交换（Compare And Swap）。在CAS中，有这样四个值：

- this：当前变量var
    
- valueOffset：变量V的内存地。
    
- expect：期望值（旧值）
    
- update：新值
    

比较并交换的过程如下：

1. 判断`this`是否等于`expect`，如果等于，将`this`的值设置为`update`；
    
2. 如果不等，说明已经有其它线程更新了变量v，则当前线程放弃更新，什么都不做。
    

我们以一个简单的例子来解释这个过程：

如果有一个多个线程共享的变量i原本等于5，我现在在线程A中，想把它设置为新的值6。我们使用CAS来做这个事情：

cas成功：首先我们用i去与5对比，发现它等于5，说明没有被其它线程改过，那我就把它设置为新的值6，此次CAS成功，i的值被设置成了6；

cas失败：如果不等于5，说明i被其它线程改过了（比如现在i的值为2），那么我就什么也不做，此次CAS失败，i的值仍然为2。

那有没有可能我在判断了i为5之后，正准备更新它的新值的时候，被其它线程更改了i的值呢？

==不会的。因为CAS是一种原子操作，它是一种系统原语，是一条CPU的原子指令，从CPU层面保证它的原子性。==

当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作。

## AtomicX自旋执行compareAndSwapInt
```Java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

CAS是实现自旋锁的基础，CAS利用CPU指令保证了操作的原子性，以达到加锁的效果。

自旋看字面意思也很明白，自己旋转，是指尝试获取锁的线程不会立即阻塞而是采用循环的方式去尝试获取锁，当线程发现锁被占用时，会不断循环判断锁的状态，直到获取。但自旋成功的条件是CAS原语返回成功

- 这样的好处是减少线程上下文切换的消耗，
    
- 缺点是循环会消耗CPU。==解决思路是让JVM支持处理器提供的pause指令==。pause指令能让自旋失败时cpu睡眠一小段时间再继续自旋，从而使得读操作的频率低很多，为解决内存顺序冲突而导致的CPU流水线重排的代价也会小很多。
    

# 手写自旋锁SpinLockDemo

```Java
package com.bilibili.juc.cas;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

/**
 * 题目：实现一个自旋锁,复习CAS思想
 * 自旋锁好处：循环比较获取没有类似wait的阻塞。
 *
 * 通过CAS操作完成自旋锁，A线程先进来调用myLock方法自己持有锁5秒钟，B随后进来后发现
 * 当前有线程持有锁，所以只能通过自旋等待，直到A释放锁后B随后抢到。
 */
public class SpinLockDemo
{
    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public void lock()
    {
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName()+"\t"+"----come in");
        while (!atomicReference.compareAndSet(null, thread)) {

        }
    }

    public void unLock()
    {
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread,null);
        System.out.println(Thread.currentThread().getName()+"\t"+"----task over,unLock...");
    }

    public static void main(String[] args)
    {
        SpinLockDemo spinLockDemo = new SpinLockDemo();

        new Thread(() -> {
            spinLockDemo.lock();
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
            spinLockDemo.unLock();
        },"A").start();

        //暂停500毫秒,线程A先于B启动
        try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            spinLockDemo.lock();

            spinLockDemo.unLock();
        },"B").start();


    }
}
```

# CAS存在的问题，ABA问题

所谓ABA问题： 比如说一个线程1从内存位置V中取出A

这时候另一个线程2也从内存中取出A，并且线程2进行了一些操作将值变成了B，然后线程2又将V位置的数据变成A放回去

这时候线程1进行CAS操作发现内存中仍然是A，预期OK，然后线程1操作成功

尽管线程1的CAS操作成功，但是不代表这个过程就是没有问题的。

段子：老刘出差媳妇在家，出差期间老刘的隔壁老王在练腰，竞争对手在磨刀，问老刘回家后其媳妇有没有发生变化？

```Java
package com.bilibili.juc.cas;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * @auther zzyy
 * @create 2022-02-25 11:40
 */
public class ABADemo {
    static AtomicInteger atomicInteger = new AtomicInteger(100);
    static AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        new Thread(() -> {
            atomicInteger.compareAndSet(100, 101);
            try {
                TimeUnit.MILLISECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicInteger.compareAndSet(101, 100);
        }, "t1").start();

        new Thread(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicInteger.compareAndSet(100, 2022) + "\t" + atomicInteger.get());
        }, "t2").start();

    }
}
//尽管线程1的CAS操作成功，但是不代表这个过程就是没有问题的。
```

# ABA问题的解决，AtomicStampedReference

ABA问题的解决思路是在变量前面追加上**版本号/时间戳/戳记流水，**择其一。

从JDK 1.5开始，JDK的atomic包里提供了一个类`AtomicStampedReference`类来解决ABA问题。

`AtomicStampedReference`中的的`compareAndSet`方法，不光先检查当前引用是否等于预期引用，并且会检查当前标志是否等于预期标志。如果二者都相等，就能证明对象没有被隔壁老王修改过。

```Java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}
```

`AtomicStampedReference`的每次`compareAndSet`操作，标志位都会改变。下面是代码用例：

```Java
package com.bilibili.juc.cas;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * @auther zzyy
 * @create 2022-02-25 11:40
 */
public class ABADemo {
    static AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        new Thread(() -> {
            int stamp = stampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t" + "首次版本号：" + stamp);

            //暂停500毫秒,保证后面的t4线程初始化拿到的版本号和我一样
            try {
                TimeUnit.MILLISECONDS.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            stampedReference.compareAndSet(100, 101, stampedReference.getStamp(), stampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t" + "2次流水号：" + stampedReference.getStamp());

            stampedReference.compareAndSet(101, 100, stampedReference.getStamp(), stampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t" + "3次流水号：" + stampedReference.getStamp());

        }, "t3").start();

        new Thread(() -> {
            int stamp = stampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t" + "首次版本号：" + stamp);

            //暂停1秒钟线程,等待上面的t3线程，发生了ABA问题
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            boolean b = stampedReference.compareAndSet(100, 2022, stamp, stamp + 1);

            System.out.println(b + "\t" + stampedReference.getReference() + "\t" + stampedReference.getStamp());

        }, "t4").start();

    }
}
//打印：
t3        首次版本号：1
t4        首次版本号：1
t3        2次流水号：2
t3        3次流水号：3
false        100        3，避免了ABA问题
```