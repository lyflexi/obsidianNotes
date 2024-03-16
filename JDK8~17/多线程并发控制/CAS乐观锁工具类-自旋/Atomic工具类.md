JUC 包涵盖了5大原子类型

1.基本原子类：使用原子的方式更新基本类型

- AtomicInteger：整形原子类
    
- AtomicLong：长整型原子类
    
- AtomicBoolean：布尔型原子类
    

2.引用原子类

- AtomicReference：引用类型原子类
    
- AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。
    
- AtomicMarkableReference ：原子更新带有标记位的引用类型
    

3.数组原子类

使用原子的方式更新数组里的某个元素

- AtomicIntegerArray：整形数组原子类
    
- AtomicLongArray：长整形数组原子类
    
- AtomicReferenceArray：引用类型数组原子类
    

4.对象的属性修改原子类

- AtomicIntegerFieldUpdater：原子更新整形字段的更新器
    
- AtomicLongFieldUpdater：原子更新长整形字段的更新器
    
- AtomicReferenceFieldUpdater：原子更新引用类型字段的更新器
    

5.原子操作增强原子类Java8

# 一、原子基本类型示例

AtomicInteger保证i++在多线程情况下的安全：

```Java
class AtomicIntegerTest {
    private AtomicInteger atomicInteger = new AtomicInteger();
    //使用AtomicInteger之后，不需要对该方法加锁，也可以实现线程安全。
    public int getNumber() {
        return count.get();
    }
    public void setNumber() {
        atomicInteger.incrementAndGet();
    }
}
```

# 二、原子引用类型示例

## AtomicReference

使用JDK 1.5开始就提供的AtomicReference类保证对象之间的原子性，把多个变量放到一个对象里面进行CAS操作；

```Java
package com.bilibili.juc.cas;


import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.ToString;

import java.util.concurrent.atomic.AtomicReference;

@Getter
@ToString
@AllArgsConstructor
class User
{
    String userName;
    int    age;
}

/**
 * @auther zzyy
 * @create 2022-02-24 14:50
 */
public class AtomicReferenceDemo
{
    public static void main(String[] args)
    {
        AtomicReference<User> atomicReference = new AtomicReference<>();

        User z3 = new User("z3",22);
        User li4 = new User("li4",28);

        atomicReference.set(z3);

        System.out.println(atomicReference.compareAndSet(z3, li4)+"\t"+atomicReference.get().toString());
        System.out.println(atomicReference.compareAndSet(z3, li4)+"\t"+atomicReference.get().toString());


    }
}
```

## AtomicStampedReference

解决CAS的ABA问题：ABA问题的解决思路是在变量前面追加上**版本号|时间戳|戳记流水，择其一**。从JDK 1.5开始，JDK的atomic包里提供了一个类`AtomicStampedReference`类来解决ABA问题。

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

这个类的**compareAndSet方法的作用，不光先检查当前引用是否等于预期引用，并且会检查当前标志是否等于预期标志**。如果二者都相等，才证明改对象没有被隔壁老王修改过。

`AtomicStampedReference`的每次`compareAndSet`操作，标志位都会自增。

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

## AtomicMarkableReference

它的定义就是将状态戳简化为true|false

- 标记位默认为false表示没人动过，
    
- 如果标记位被置为true，则compareAndSet就会失败，表明不可修改
    

AtomicMarkableReferenceDemo：

```Java
package com.bilibili.juc.atomics;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicMarkableReference;

/**
 * @auther zzyy
 * @create 2022-02-26 10:57
 */
public class AtomicMarkableReferenceDemo
{
    static AtomicMarkableReference markableReference = new AtomicMarkableReference(100,false);

    public static void main(String[] args)
    {
        new Thread(() -> {
            boolean marked = markableReference.isMarked();
            System.out.println(Thread.currentThread().getName()+"\t"+"默认标识："+marked);
            //暂停1秒钟线程,等待后面的T2线程和我拿到一样的模式flag标识，都是false
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            markableReference.compareAndSet(100,1000,marked,!marked);
        },"t1").start();

        new Thread(() -> {
            boolean marked = markableReference.isMarked();
            System.out.println(Thread.currentThread().getName()+"\t"+"默认标识："+marked);

            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
            boolean b = markableReference.compareAndSet(100, 2000, marked, !marked);
            System.out.println(Thread.currentThread().getName()+"\t"+"t2线程CASresult： "+b);
            System.out.println(Thread.currentThread().getName()+"\t"+markableReference.isMarked());
            System.out.println(Thread.currentThread().getName()+"\t"+markableReference.getReference());
        },"t2").start();
    }
}

/**
 *  CAS----Unsafe----do while+ABA---AtomicStampedReference,AtomicMarkableReference
 *
 *  AtomicStampedReference,version号，+1；
 *
 *  AtomicMarkableReference，一次，false，true
 *
 */
打印：
com.bilibili.juc.atomics.AtomicMarkableReferenceDemo
t1        默认标识：false
t2        默认标识：false
t2        t2线程CASresult： false
t2        true
t2        1000

Process finished with exit code 0
```

# 三、原子数组类型示例

```Java
package com.bilibili.juc.atomics;

import java.util.concurrent.atomic.AtomicIntegerArray;

/**
 * @auther zzyy
 * @create 2022-02-26 10:20
 */
public class AtomicIntegerArrayDemo
{
    public static void main(String[] args)
    {
        AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(new int[5]);
        //AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(5);
        //AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(new int[]{1,2,3,4,5});

        for (int i = 0; i <atomicIntegerArray.length(); i++) {
            System.out.println(atomicIntegerArray.get(i));
        }

        System.out.println();

        int tmpInt = 0;

        tmpInt = atomicIntegerArray.getAndSet(0,1122);
        System.out.println(tmpInt+"\t"+atomicIntegerArray.get(0));

        tmpInt = atomicIntegerArray.getAndIncrement(0);
        System.out.println(tmpInt+"\t"+atomicIntegerArray.get(0));
    }
}
```

# 四、对象的属性修改原子类示例

使用目的：以一种线程安全的方式操作非线程安全对象内的某些字段

思想：是否可以不要锁定整个对象，减少锁定的范围，只关注长期、敏感性变化的某一个字段，而不是整个对象，以达到精确加锁+节约内存的目的

使用要求：

- 更新的对象属性必须使用public volatile修饰符
    
- 这种原子类型是抽象类，所以每次使用都必须使用静态方法newUpdater( )创建一个更新器，并且需要设置想要更新的类和属性
    

## AtomicIntegerFieldUpdater

```Java
package com.bilibili.juc.atomics;


import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

class BankAccount//资源类
{
    String bankName = "CCB";

    //更新的对象属性必须使用 public volatile 修饰符。
    public volatile int money = 0;//钱数

    public void add()
    {
        money++;
    }

    //因为对象的属性修改类型原子类都是抽象类，所以每次使用都必须
    // 使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。

    AtomicIntegerFieldUpdater<BankAccount> fieldUpdater =
            AtomicIntegerFieldUpdater.newUpdater(BankAccount.class,"money");

    //不加synchronized，保证高性能原子性，局部微创小手术
    public void transMoney(BankAccount bankAccount)
    {
        fieldUpdater.getAndIncrement(bankAccount);
    }


}

/**
 * @auther zzyy
 * 以一种线程安全的方式操作非线程安全对象的某些字段。
 *
 * 需求：
 * 10个线程，
 * 每个线程转账1000，
 * 不使用synchronized,尝试使用AtomicIntegerFieldUpdater来实现。
 */
public class AtomicIntegerFieldUpdaterDemo
{
    public static void main(String[] args) throws InterruptedException
    {
        BankAccount bankAccount = new BankAccount();
        CountDownLatch countDownLatch = new CountDownLatch(10);

        for (int i = 1; i <=10; i++) {
            new Thread(() -> {
                try {
                    for (int j = 1; j <=1000; j++) {
                        //bankAccount.add();
                        bankAccount.transMoney(bankAccount);
                    }
                } finally {
                    countDownLatch.countDown();
                }
            },String.valueOf(i)).start();
        }

        countDownLatch.await();

        System.out.println(Thread.currentThread().getName()+"\t"+"result: "+bankAccount.money);
    }
}
```

## AtomicReferenceFieldUpdater

```Java
package com.bilibili.juc.atomics;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

class MyVar //资源类
{
    public volatile Boolean isInit = Boolean.FALSE;

    AtomicReferenceFieldUpdater<MyVar,Boolean> referenceFieldUpdater =
            AtomicReferenceFieldUpdater.newUpdater(MyVar.class,Boolean.class,"isInit");

    public void init(MyVar myVar)
    {
        if (referenceFieldUpdater.compareAndSet(myVar,Boolean.FALSE,Boolean.TRUE))
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"----- start init,need 2 seconds");
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"----- over init");
        }else{
            System.out.println(Thread.currentThread().getName()+"\t"+"----- 已经有线程在进行初始化工作。。。。。");
        }
    }
}


/**
 * @auther zzyy
 * 需求：
 * 多线程并发调用一个类的初始化方法，如果未被初始化过，将执行初始化工作，
 * 要求只能被初始化一次，只有一个线程操作成功
 */
public class AtomicReferenceFieldUpdaterDemo
{
    public static void main(String[] args)
    {
        MyVar myVar = new MyVar();

        for (int i = 1; i <=5; i++) {
            new Thread(() -> {
                myVar.init(myVar);
            },String.valueOf(i)).start();
        }
    }
}
//打印：
com.bilibili.juc.atomics.AtomicReferenceFieldUpdaterDemo
1        ----- start init,need 2 seconds
3        ----- 已经有线程在进行初始化工作。。。。。
2        ----- 已经有线程在进行初始化工作。。。。。
4        ----- 已经有线程在进行初始化工作。。。。。
5        ----- 已经有线程在进行初始化工作。。。。。
1        ----- over init

Process finished with exit code 0
```

# 五、原子操作增强原子类LongAdder

- LongAdder
    
- LongAccumulator
    

【参考】volatile 解决多线程内存不可见问题。对于一写多读，是可以解决变量同步问题，但是如果多写，涉及到并发问题，volatile不支持原子性，因此同样无法解决线程安全问题。

【说明】：如果是 count++操作，使用如下类AtomicInteger 实现

```Java
AtomicInteger count = new AtomicInteger(); 
count.addAndGet(1); 
```

上述代码在并发量比较低的情况下，线程冲突的概率比较小，自旋的次数不会很多。但是，高并发情况下，N个线程同时进行自旋操作，N-1个线程失败，导致CPU打满场景。此时AtomicLong的自旋会成为瓶颈

  

如果是 JDK8，推荐使用 LongAdder 对象，比 AtomicInteger性能更好（减少乐观锁的重试次数）。这就是LongAdder引入的初衷------解决高并发环境下AtomictLong的自旋瓶颈问题。下面我们针对`LongAdder`的常用API进行剖析，但`LongAdder` 只能做累加且只能从0开始累加

|   |   |
|---|---|
|方法|解释|
|void add(long x)|将当前的value加x|
|void increment( )|将当前的value加1|
|void decrement( )|将当前value减1|
|long sum( )|返回当前的值,特别注意在没有并发更新value的情况下sum会返回一个精确值,在存在并发的情况下,sum不保证返回精确值|
|void reset()|将value重置为0,可用于替换重新new一个LongAdder,但次方法只可以在没有并发更新的情况下使用|
|long sumThenReset()|获取当前value,并将value重置为0|

## 高并发点赞案例性能对比

```Java
package lock;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAccumulator;
import java.util.concurrent.atomic.LongAdder;

class ClickNumber //资源类
{
    int number = 0;

    public synchronized void clickBySynchronized() {
        number++;
    }

    AtomicLong atomicLong = new AtomicLong(0);

    public void clickByAtomicLong() {
        atomicLong.getAndIncrement();
    }

    LongAdder longAdder = new LongAdder();

    public void clickByLongAdder() {
        longAdder.increment();
    }

    LongAccumulator longAccumulator = new LongAccumulator((x, y) -> x + y, 0);

    public void clickByLongAccumulator() {
        longAccumulator.accumulate(1);
    }

}

/**
 * @auther zzyy
 * 需求： 50个线程，每个线程100W次，总点赞数出来
 */
public class AccumulatorCompareDemo {
    public static final int _1W = 10000;
    public static final int threadNumber = 50;

    public static void main(String[] args) throws InterruptedException {
        ClickNumber clickNumber = new ClickNumber();
        long startTime;
        long endTime;

        CountDownLatch countDownLatch1 = new CountDownLatch(threadNumber);
        CountDownLatch countDownLatch2 = new CountDownLatch(threadNumber);
        CountDownLatch countDownLatch3 = new CountDownLatch(threadNumber);
        CountDownLatch countDownLatch4 = new CountDownLatch(threadNumber);

        startTime = System.currentTimeMillis();
        for (int i = 1; i <= threadNumber; i++) {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= 100 * _1W; j++) {
                        clickNumber.clickBySynchronized();
                    }
                } finally {
                    countDownLatch1.countDown();
                }
            }, String.valueOf(i)).start();
        }
        countDownLatch1.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: " + (endTime - startTime) + " 毫秒" + "\t clickBySynchronized: " + clickNumber.number);

        startTime = System.currentTimeMillis();
        for (int i = 1; i <= threadNumber; i++) {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= 100 * _1W; j++) {
                        clickNumber.clickByAtomicLong();
                    }
                } finally {
                    countDownLatch2.countDown();
                }
            }, String.valueOf(i)).start();
        }
        countDownLatch2.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: " + (endTime - startTime) + " 毫秒" + "\t clickByAtomicLong: " + clickNumber.atomicLong.get());


        startTime = System.currentTimeMillis();
        for (int i = 1; i <= threadNumber; i++) {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= 100 * _1W; j++) {
                        clickNumber.clickByLongAdder();
                    }
                } finally {
                    countDownLatch3.countDown();
                }
            }, String.valueOf(i)).start();
        }
        countDownLatch3.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: " + (endTime - startTime) + " 毫秒" + "\t clickByLongAdder: " + clickNumber.longAdder.sum());

        startTime = System.currentTimeMillis();
        for (int i = 1; i <= threadNumber; i++) {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= 100 * _1W; j++) {
                        clickNumber.clickByLongAccumulator();
                    }
                } finally {
                    countDownLatch4.countDown();
                }
            }, String.valueOf(i)).start();
        }
        countDownLatch4.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: " + (endTime - startTime) + " 毫秒" + "\t clickByLongAccumulator: " + clickNumber.longAccumulator.get());

    }
}
```

打印信息：

```Java
----costTime: 1417 毫秒         clickBySynchronized: 50000000
----costTime: 1003 毫秒         clickByAtomicLong: 50000000
----costTime: 102 毫秒         clickByLongAdder: 50000000
----costTime: 75 毫秒         clickByLongAccumulator: 50000000

Process finished with exit code 0
```

# LongAdder增强原理

`LongAdder`继承自`Striped64`

## Striped64的数据结构

Striped64全局变量信息

```Java

    /** Number of CPUS, to place bound on table size 
    当前计算机CPU数量,Cell数组扩容时会使用到
    */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /**
     * Table of cells. When non-null, size is a power of 2.
     */
    transient volatile Cell[] cells;

    /**
     * Base value, used mainly when there is no contention, but also as
     * a fallback during table initialization races. Updated via CAS.
     类似于AtomicLong中全局的value值。再没有竞争情况下数据直接累加到base上,或者cells扩容时,也需要将数据写入到base上
     */
    transient volatile long base;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating Cells.
     初始化cells或者扩容cells需要获取锁,0表示无锁状态,1表示其他线程已经持有了锁
     */
    transient volatile int cellsBusy;
```

Striped64的内部类Cell信息

```Java

    /**
     * Padded variant of AtomicLong supporting only raw accesses plus CAS.
     *
     * JVM intrinsics note: It would be possible to use a release-only
     * form of CAS here, if it were provided.
     */
    @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

## Striped64分散热点技术剖析
![[Pasted image 20231225154955.png]]

### LongAdder#add

```Java
public void add(long x) {
//as是striped64中的cells数组
//b是striped64中的base
//v是当前线程hash到的cell中存储的值
//m是cells的长度减1,hash时作为掩码使用
//a时当前线程hash到的cell
Cell[] as; long b, v; int m; Cell a;
/**
    条件1:cells不为空,说明出现过竞争,cell[]已创建
    条件2:cas操作base失败,说明其他线程先一步修改了base正在出现竞争
    首次首线程(as = cells) != null)一定是false,此时走casBase方法,以CAS的方式更新base值,
    且只有当cas失败时,才会走到if中
*/
if ((as = cells) != null || !casBase(b = base, b + x)) {
    //true无竞争 fasle表示竞争激烈,多个线程hash到同一个cell,可能要扩容
    boolean uncontended = true;
    /*
    条件1:cells为空,说明正在出现竞争,上面是从条件2过来的,说明!casBase(b = base, b + x))=true
              会通过调用longAccumulate(x, null, uncontended)新建一个数组,默认长度是2
    条件2:默认会新建一个数组长度为2的数组,m = as.length - 1) < 0 应该不会出现,
    条件3:当前线程所在的cell为空,说明当前线程还没有更新过cell,应初始化一个cell。
              a = as[getProbe() & m]) == null,如果cell为空,进行一个初始化的处理
    条件4:更新当前线程所在的cell失败,说明现在竞争很激烈,多个线程hash到同一个Cell,应扩容
              (如果是cell中有一个线程操作,这个时候,通过a.cas(v = a.value, v + x)可以进行处理,返回的结果是true)
    **/
    if (as == null || (m = as.length - 1) < 0 ||
            //getProbe( )方法返回的时线程中的threadLocalRandomProbe字段
            //它是通过随机数生成的一个值,对于一个确定的线程这个值是固定的(除非刻意修改它)
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
        //调用Striped64中的方法处理
        longAccumulate(x, null, uncontended);
}
```

1. 最初无竞争时，直接通过casBase进行更新base的处理，跳过第一个if
    
2. 当casBase比较激烈，则进入第一个if，开始判断第二个if
    
    1. 如果更新casBase失败后,首次新建一个Cell[ ]数组(默认长度是2)
        
    2. 如果Cell数组当中的某一个槽位为空
        
    3. 当多个线程竞争同一个Cell比较激烈时,可能就要对Cell[ ]扩容
        
3. 调用longAccumulate：
	![[Pasted image 20231225155006.png]]

### Striped64#longAccumulate

`longAccumulate`方法首先得到线程hash值h：

```Java
final void longAccumulate(long x, LongBinaryOperator fn,
                                                  boolean wasUncontended) {
    //存储线程的probe值
    int h;
    //如果getProbe()方法返回0,说明随机数未初始化
    if ((h = getProbe()) == 0) { //这个if相当于给当前线程生成一个非0的hash值
        //使用ThreadLocalRandom为当前线程重新计算一个hash值,强制初始化
        ThreadLocalRandom.current(); // force initialization
        //重新获取probe值,hash值被重置就好比一个全新的线程一样,所以设置了wasUncontended竞争状态为true
        h = getProbe();
        //重新计算了当前线程的hash后认为此次不算是一次竞争,都未初始化,肯定还不存在竞争激烈
        //wasUncontended竞争状态为true
        wasUncontended = true;
    }
}
```
![[Pasted image 20231225155033.png]]

#### CASE2：刚刚初始化Cell[ ]数组(首次新建)

```Java
        //CASE2:cells没有加锁且没有初始化,则尝试对它进行加锁,并初始化cells数组
        /*
        cellsBusy:初始化cells或者扩容cells需要获取锁,0表示无锁状态,1表示其他线程已经持有了锁
        cells == as == null  是成立的
        casCellsBusy:通过CAS操作修改cellsBusy的值,CAS成功代表获取锁,
        返回true,第一次进来没人抢占cell单元格,肯定返回true
        **/
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) { 
            //是否初始化的标记
                boolean init = false;
                try {                           // Initialize table(新建cells)
                        // 前面else if中进行了判断,这里再次判断,采用双端检索的机制
                        if (cells == as) {
                                //如果上面条件都执行成功就会执行数组的初始化及赋值操作,Cell[] rs = new Cell[2]标识数组的长度为2
                                Cell[] rs = new Cell[2];
                                //rs[h & 1] = new Cell(x)表示创建一个新的cell元素,value是x值,默认为1
                                //h & 1 类似于我们之前hashmap常用到的计算散列桶index的算法,
                                //通常都是hash&(table.len-1),同hashmap一个意思
                                //看这次的value是落在0还是1
                                rs[h & 1] = new Cell(x);
                                cells = rs;
                                init = true;
                        }
                } finally {
                        cellsBusy = 0;
                }
                if (init)
                        break;
        }
```

#### CASE3：兜底(多个线程尝试CAS修改失败的线程会走这个分支)

```Java
        //CASE3:cells正在进行初始化,则尝试直接在基数base上进行累加操作
        //这种情况是cell中都CAS失败了,有一个兜底的方法
        //该分支实现直接操作base基数,将值累加到base上,
        //也即其他线程正在初始化,多个线程正在更新base的值
        else if (casBase(v = base, ((fn == null) ? v + x :
                                                                fn.applyAsLong(v, x))))
                break;     
```

#### CASE1：Cell数组不再为空且可能存在Cell数组扩容

```Java
for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0) { // CASE1:cells已经初始化了
            // 当前线程的hash值运算后映射得到的Cell单元为null,说明该Cell没有被使用
                if ((a = as[(n - 1) & h]) == null) {
                        //Cell[]数组没有正在扩容
                        if (cellsBusy == 0) {       // Try to attach new Cell
                                //先创建一个Cell
                                Cell r = new Cell(x);   // Optimistically create
                                //尝试加锁,加锁后cellsBusy=1
                                if (cellsBusy == 0 && casCellsBusy()) { 
                                        boolean created = false;
                                        try {               // Recheck under lock
                                                Cell[] rs; int m, j; //将cell单元赋值到Cell[]数组上
                                                //在有锁的情况下再检测一遍之前的判断 
                                                if ((rs = cells) != null &&
                                                        (m = rs.length) > 0 &&
                                                        rs[j = (m - 1) & h] == null) {
                                                        rs[j] = r;
                                                        created = true;
                                                }
                                        } finally {
                                                cellsBusy = 0;//释放锁
                                        }
                                        if (created)
                                                break;
                                        continue;           // Slot is now non-empty
                                }
                        }
                        collide = false;
                }
                /**
                wasUncontended表示cells初始化后,当前线程竞争修改失败
                wasUncontended=false,表示竞争激烈,需要扩容,这里只是重新设置了这个值为true,
                紧接着执行advanceProbe(h)重置当前线程的hash,重新循环
                */
                else if (!wasUncontended)       // CAS already known to fail
                        wasUncontended = true;      // Continue after rehash
                //说明当前线程对应的数组中有了数据,也重置过hash值
                //这时通过CAS操作尝试对当前数中的value值进行累加x操作,x默认为1,如果CAS成功则直接跳出循环
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                                                         fn.applyAsLong(v, x))))
                        break;
                //如果n大于CPU最大数量,不可扩容,并通过下面的h=advanceProbe(h)方法修改线程的probe再重新尝试
                else if (n >= NCPU || cells != as)
                        collide = false;    //扩容标识设置为false,标识永远不会再扩容
                //如果扩容意向collide是false则修改它为true,然后重新计算当前线程的hash值继续循环
                else if (!collide) 
                        collide = true;
                //锁状态为0并且将锁状态修改为1(持有锁) 
                else if (cellsBusy == 0 && casCellsBusy()) { 
                        try {
                                if (cells == as) {      // Expand table unless stale
                                        //按位左移1位来操作,扩容大小为之前容量的两倍
                                        Cell[] rs = new Cell[n << 1];
                                        for (int i = 0; i < n; ++i)
                                                //扩容后将之前数组的元素拷贝到新数组中
                                                rs[i] = as[i];
                                        cells = rs; 
                                }
                        } finally {
                                //释放锁设置cellsBusy=0,设置扩容状态,然后进行循环执行
                                cellsBusy = 0;
                        }
                        collide = false;
                        continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
        }
```

### LongAdder#sum

LongAdder#`sum( )`会将所有Cell数组中的value和base累加作为返回值，核心的思想就是将之前AtomicLong一个value的更新压力分散到多个value中去,从而降级更新热点

```Java
    public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

为啥高并发下sum的值不精确？sum执行时,并没有限制对base和cells的更新(一句要命的话)。所以LongAdder不是强一致性,它是最终一致性的

- 首先,最终返回的sum局部变量,初始被赋值为base,而最终返回时,很可能base已经被更新了,而此时局部变量sum不会更新,造成不一致
    
- 其次,这里对cell的读取也无法保证是最后一次写入的值。所以,sum方法只是在没有并发的情况下,可以获得正确的结果