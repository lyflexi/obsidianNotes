# synchronized 关键字使用方式

**synchronized 关键字最主要的三种使用方式：**

**1.修饰实例方法:** 作用于当前对象实例加锁，进入同步代码前要获得 **当前对象实例的锁**

```Java
synchronized void method() {
  //业务代码
}
```

但是构造方法不能使用 synchronized 关键字修饰。构造方法本身就属于线程安全的，不存在同步的构造方法一说。

面试题：当线程1进入到一个对象的synchronized方法A后，线程2是否可以进入到此对象的synchronized方法B?

不能，线程2只能访问该对象的非同步方法。因为执行同步方法时需要获得对象的锁，而线程1在进入`sychronized`修饰的方A时已经获取到了锁，线程2只能等待，无法进入到`synchronized`修饰的方法B，但可以进入到其他非`synchronized`修饰的方法。

**2.修饰静态方法:** 也就是给当前类加锁，会作用于类的所有对象实例 ，进入同步代码前要获得 **当前 class 的锁**。==静态成员不属于任何一个实例对象==。所以，如果一个线程 A 调用一个实例对象的非静态 synchronized 方法，而线程 B 需要调用这个实例对象所属类的静态 synchronized 方法，是允许的，不会发生互斥现象，**因为访问静态 synchronized 方法占用的锁是当前类的锁，而访问非静态 synchronized 方法占用的锁是当前实例对象锁**。

```Java
synchronized static void method() {
//业务代码
}
```

**3.修饰代码块** ：指定加锁对象，对给定对象/类加锁。

synchronized(this|object) 表示进入同步代码库前要获得**给定对象的锁**。

synchronized(类.class) 表示进入同步代码前要获得 **当前 class 的锁**

```Java
synchronized(this) {
  //业务代码
}
```

# synchronized 关键字三大特性

使用synchronized ，以下三大特性均可以保证！

- 原子性：一个或多个操作要么全部执行成功，要么全部执行失败。`synchronized`关键字可以保证只有一个线程拿到锁，访问共享资源。
    
- 可见性：当一个线程对共享变量进行修改后，其他线程可以立刻看到。
    
- 有序性：程序的执行顺序会按照代码的先后顺序执行。
    

## 原子性分析

不言自明

## 可见性分析

我们来分析下什么是可见性，比如下面的代码示例：

```Java
/**
 * 变量的内存可见性例子
 *
 * @author star
 */
public class VolatileExample {

    /**
     * main 方法作为一个主线程
     */
    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        // 开启线程
        myThread.start();

        // （不加synchronized ）你会发现永远都不会输出有点东西这一段代码，
        //  按道理线程MyThread改了flag变量，主线程Main也能访问到的呀？
        for (; ; ) {
            if (myThread.isFlag()) {
                System.out.println("有点东西");
            }
        }
//     （通过加synchronized 可以解决不可见性问题 ）
//      因为当一个线程进入 synchronized 代码块后，线程获取到锁，会清空本地内存，
//      然后从主内存中拷贝共享变量的最新值到本地内存作为副本，将修改后的副本值刷新到主内存中，执行代码，最后线程释放锁。
//      for (; ; ) {
//          synchronized (myThread) {
//              if (myThread.isFlag()) {
//                  System.out.println("主线程访问到 flag 变量");
//                }
//          }
//      }       
    }

}

/**
 * 子线程类
 */
class MyThread extends Thread {

    private boolean flag = false;

    @Override
    public void run() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 修改变量值
        flag = true;
        System.out.println("flag = " + flag);
    }

    public boolean isFlag() {
        return flag;
    }

    public void setFlag(boolean flag) {
        this.flag = flag;
    }
}
```

在 JDK1.2 之前，Java 的内存模型实现总是从主存（即共享内存）读取变量，是不需要进行特别的注意的。而在当前的 Java 内存模型下，JMM规定：

- 所有的共享变量都存储于主内存。（这里所说的共享变量指的是实例变量和类变量，不包含局部变量，因为局部变量是线程私有的，因此不存在竞争问题）
    
- 每一个线程还存在自己的工作内存，线程的工作内存，保留了被线程使用的变量的工作副本。
    
- 线程对变量的所有的操作（读，取）都必须在工作内存中完成，而不能直接读写主内存中的变量。
    
- 不同线程之间也不能直接访问对方工作内存中的变量，线程间变量的值的传递需要通过主内存中转来完成。
    

一句话总结就是，线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。
![[Pasted image 20231225134449.png]]

然而，JMM 这样的规定可能会导致线程对共享变量的修改没有即时更新到主内存，或者线程没能够即时将共享变量的最新值同步到工作内存中，从而使得线程在使用共享变量的值时，该值并不是最新的。比如本地内存中读取的 i=1，两个线程做了 1++运算完之后再写回 Main Memory 之后 i=2，而正确结果应该是 i=3。

## 有序性分析

一个好的内存模型实际上会放松对处理器和编译器规则的束缚，在不改变程序执行结果的前提下，编译器和处理器常常会对指令进行重排序，以尽可能提高执行效率。
![[Pasted image 20231225134456.png]]

单看下面的程序好像没有问题，最后 i 的值是 1。但是为了提高性能，编译器和处理器常常会在不改变数据依赖的情况下对指令做重排序。假设线程 A 在执行时被重排序成先执行代码 2，再执行代码 1；而线程 B 在线程 A 执行完代码 2 后，读取了 flag 变量。由于条件判断为真，线程 B 将读取变量 a。此时，变量 a 还根本没有被线程 A 写入，那么 i 最后的值是 0，导致执行结果不正确。

线程 A：

```Java
int a = 0;
// 线程 A
a = 1;           // 1
flag = true;     // 2
```

线程 B：

```Java
// 线程 B
if (flag) { // 3
  int i = a; // 4
}
```

synchronized 会保证上述程序顺序执行，不会出现问题

# synchronized 关键字的底层原理

## synchronized 可重入锁原理Monitor

在jdk1.6之前，`synchronized`只能实现重量级锁，Java虚拟机是基于Monitor对象来实现重量级锁的。因为监视器锁(monitor)是依赖于底层的操作系统的Mutex Lock来实现的，挂起线程和恢复线程都需要转入内核态去完成，阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态切换需要耗费处理器时间，如果同步代码块中内容过于简单，这种切换的时间可能比用户代码执行的时间还长”，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。
![[Pasted image 20231225135351.png]]

在Hotspot虚拟机中，ObjectMonitor源码是用C++语言编写的，首先我们先下载Hotspot的源码，源码下载链接：[http://hg.openjdk.java.net/jdk8/jdk8/hotspot](https://link.zhihu.com/?target=http%3A//hg.openjdk.java.net/jdk8/jdk8/hotspot)，找到`ObjectMonitor.hpp`文件，路径是src/share/vm/runtime/objectMonitor.hpp。通过`ObjectMonitor.hpp`源码我们知道，wait/notify等方法也依赖于`ObjectMonitor`对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。这里只是简单介绍下其数据结构

```C++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //锁的计数器，获取锁时count数值加1，释放锁时count值减1，直到
    _waiters      = 0, //等待线程数
    _recursions   = 0; //锁的重入次数
    _object       = NULL; 
    _owner        = NULL; //指向持有ObjectMonitor对象的线程地址
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; //阻塞在EntryList上的单向线程列表
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

其中 `_owner`、`_count`、`_recursions`、`_WaitSet`和`_EntryList` 字段比较重要，它们之间的转换关系如下图
![[Pasted image 20231225134517.png]]
![[Pasted image 20231225134525.png]]

synchronized是可重入锁，从上图可以总结获取Monitor和释放Monitor的流程如下：

1. 当多个线程同时访问同步代码块时，首先会进入到`EntryList`中，然后通过CAS的方式尝试将Monitor中的owner字段设置为当前线程，同时`count`加1
    
    1. 若发现之前的owner的值就是指向当前线程的，`recursions`也需要加1。
        
    2. 如果CAS尝试获取锁失败，则进入到`EntryList`中。
        
2. 当前线程执行完同步代码块时，则会释放锁，count减1，recursions减1。
    
    1. 当recursions的值为0时，说明线程已经释放了锁，`_owner`置为null。
        
    2. 当recursions的值不为0时，说明线程还没有从多层synchronized中完全退出，`_owner`保持不变。
        
3. 还有一种情况count--recursions--，当获取锁的线程调用`wait()`方法，则会将owner设置为null，同时count减1，recursions减1，将当前线程加入到`WaitSet`中，等待被唤醒。

## synchronized 同步语句块的情况

下面通过 JDK 自带的 命令查看字节码信息来验证synchronized原理

1. 首先切换到类的对应目录执行 javac SynchronizedDemo.java 命令生成编译后的 .class 文件
    
2. 然后执行javap -c -s -v -l SynchronizedDemo.class命令查看 SynchronizedDemo 类的相关字节码信息。
    

```Java
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("synchronized 代码块");
        }
    }
}
```

从下面我们可以看出：

**synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中：**

- **monitorenter 指令指向同步代码块的开始位置**
    
- **monitorexit 指令则指明同步代码块的结束位置。**
    

synchronized用的是监视器锁（monitor），是依赖于底层的操作系统的 Mutex Lock 来实现的。当执行 monitorenter 指令时，线程试图获取 **对象监视器 monitor** 的持有权。
![[Pasted image 20231225135440.png]]

## synchronized 修饰方法的的情况

```Java
public class SynchronizedDemo2 {
    public synchronized void method() {
        System.out.println("synchronized 方法");
    }
}
```

synchronized 修饰的方法并没有 monitorenter 指令和 monitorexit 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，

该标识指明了该方法是一个同步方法。JVM 通过该 `flags：ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。
![[Pasted image 20231225135457.png]]

# synchronized的阻塞通知方法

## Object的wait方法

`wait`方法是Object类的方法，native方法。当一个线程调用wait的时候，会释放同步锁，然后该线程进入等待状态。其他挂起的线程会竞争这个锁，得到锁的继续执行。

## Thread的join方法

`join`是Thread的方法。 当前线程停止执行，一直等到新join进来的线程执行完毕，才会继续执行！！新join进来的线程相当于强行插队了

## Object的notify方法

# synchronized与ReentrantLock

## 定义区别

- synchronized是Java中的关键字。
    
- Lock是java的一个interface接口
    
- synchronized 依赖于 JVM
    
- ReentrantLock 依赖于 API。
    

```Java
//Lock的接口如下：
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

## 可重入锁

==可重入锁：两者都是可重入锁。==

**“可重入锁”** 指的是自己可以再次**自己获取自己**的内部锁。比如一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，如果不可锁重入的话，可能就会造成死锁。同一个线程每次获取锁，锁的计数器都自增 1，所以要等到锁的计数器下降为 0 时才能释放锁。

通俗来说：当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁，请求就会成功.

```Java
package com.test.reen;

// 演示可重入锁是什么意思，可重入，就是可以重复获取相同的锁，synchronized和ReentrantLock都是可重入的
// 可重入降低了编程复杂性
public class WhatReentrant {
    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (this) {
                    System.out.println("第1次获取锁，这个锁是：" + this);
                    int index = 1;
                    while (true) {
                        synchronized (this) {
                                System.out.println("第" + (++index) + "次获取锁，这个锁是：" + this);
                        }
                        if (index == 10) {
                                break;
                        }
                    }
                }
            }
        }).start();
    }
}
```

## 自动释放

synchronized是由JDK实现的，不需要程序员编写代码去控制加锁和释放；synchronized修饰的代码在执行异常时，jdk会自动释放线程占有的锁，不需要程序员去控制释放锁，因此只要服务器不崩溃不会导致死锁现象发生；但是，当Lock发生异常时，如果程序没有通过unLock()去释放锁，则很可能造成死锁现象，因此Lock一般都是在finally块中释放锁；格式如下：

```Java
//Lock一般都是在finally块中释放锁；格式如下：
Lock lock = new LockImpl; // new 一个Lock的实现类
lock.lock();  // 加锁
try{
    //todo
}catch(Exception ex){
     // todo
}finally{
    lock.unlock();   //释放锁
}
```

## 响应中断

Lock可以让等待锁的线程响应中断处理，如tryLock(long time, TimeUnit unit)在参数时间内未获取锁，则立即退出尝试获取锁，去执行else内的代码块。而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够中断，程序员无法控制；

```Java
public void runThread(Thread t){
    //lock对象调用trylock(long time , TimeUnit unit)方法尝试获取锁
    try {
        //注意，这个方法需要抛出中断异常
        if(lock.tryLock(2000L,TimeUnit.MILLISECONDS)){
                //获锁成功代码段
                System.out.println("线程"+t.getName()+"获取锁成功");
                try {
                        //执行的代码
                        Thread.sleep(4000);
                } catch (Exception e) {
                        //异常处理内容，比如中断异常
                } finally {
                        //获取锁成功之后，一定记住加finally并unlock()方法,释放锁
                        System.out.println("线程"+t.getName()+"释放锁");
                        lock.unlock();
                }
            }else{
                //获锁失败代码段
                //具体获取锁失败的回复响应
                System.out.println("线程"+t.getName()+"获取锁失败");
             }
        } catch (InterruptedException e) {
            e.printStackTrace();
         }
}
```

ReentrantLock另外提供了一种能够中断等待锁的线程的机制，通过 lock.lockInterruptibly() 来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。

## 公平与否

synchronized 和 ReentrantLock 默认是非公平锁

什么是公平锁和非公平锁？

非公平锁：是指在多线程获取锁的顺序并不是按照申请锁的顺序,有可能后申请的线程比先申请的线程优先获取到锁,在高并发的情况下,有可能造成饥饿现象

公平锁：是指多个线程按照申请锁的顺序来获取锁类似排队打饭先来后到。所谓的公平锁就是先等待的线程先获得锁。

可实现公平锁 : ==ReentrantLock正义之锁==，它可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。

ReentrantLock默认情况也是非公平的，可以创建 ReentrantLock的时候通过构造方法ReentrantLock(boolean fair)传入boolean 来制定是否是公平的。

排队抢票案例：使用公平锁会有什么问题？锁饥饿:我们使用5个线程买100张票,使用ReentrantLock默认是非公平锁,获取到的结果可能都是A线程在出售这100张票,会导致B、C、D、E线程发生锁饥饿

```Java
class Ticket {
    private int number = 50;

    private Lock lock = new ReentrantLock(true); //默认用的是false非公平锁，设置为true公平锁
    public void sale() {
        lock.lock();
        try {
            if(number > 0) {
                System.out.println(Thread.currentThread().getName()+"\t 卖出第: "+(number--)+"\t 还剩下: "+number);
            }
        }finally {
            lock.unlock();
        }
    }
}
public class SaleTicketDemo {
    public static void main(String[] args) {
        Ticket ticket = new Ticket();
        new Thread(() -> { for (int i = 1; i <=55; i++) ticket.sale(); },"a").start();
        new Thread(() -> { for (int i = 1; i <=55; i++) ticket.sale(); },"b").start();
        new Thread(() -> { for (int i = 1; i <=55; i++) ticket.sale(); },"c").start();
        new Thread(() -> { for (int i = 1; i <=55; i++) ticket.sale(); },"d").start();
        new Thread(() -> { for (int i = 1; i <=55; i++) ticket.sale(); },"e").start();
    }
}
```

为什么默认非公平？

减少了线程的开销线程的开销：恢复挂起的线程到真正锁的获取还是有时间差的,从开发人员来看这个时间微乎其微,但是从CPU的角度来看,这个时间存在的还是很明显的,所以非公平锁能更充分的利用CPU的时间片,尽量减少CPU空闲状态时间

非公平锁的优点？

使用多线程很重要的考量点是线程切换的开销,当采用非公平锁时,当一个线程请求锁获取同步状态,然后释放同步状态,因为不需要考虑是否还有前驱节点,所以刚释放锁的线程在此刻再次获取同步状态的概率就变得非常大了,所以就减少了线程的开销线程的开销

什么时候用公平？什么时候用非公平？

如果为了更高的吞吐量,很显然非公平锁是比较合适的,因为节省很多线程切换时间,吞吐量自然就上去了。否则那就用公平锁,大家公平使用

## 悲观与否

synchronized关键字和Lock的实现类都是悲观锁

```Java
//悲观锁的调用方式synchronized
public synchronized void m1(){
        //加锁后的业务逻辑
}
//悲观锁的调用方式ReentrantLock
//保证多个线程使用的是同一个lock对象的前提下
ReetrantLock lock=new ReentrantLock();
public void m2(){
        lock.lock();
        try{
                //操作同步资源
        }finally{
                lock.unlock();
        }
}
```

①. 悲观锁

什么是悲观锁：认为自己在使用数据的时候一定有别的线程来修改数据,因此在获取数据的时候会先加锁,确保数据不会被别的线程修改

适合写操作多的场景，先加锁可以保证写操作时数据正确(写操作包括增删改)、显式的锁定之后再操作同步资源

②. 乐观锁：乐观锁一般有两种实现方式，采用版本号机制（Version）或CAS（Compare and Swap）算法实现，最常采用的是CAS算法。以CAS为例，==乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。如果这个数据没有被更新,当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则本次CAS失败==。乐观锁在Java中通过使用无锁编程来实现，适合读操作多的场景,不加锁的特点能够使其读操作的性能大幅度提升。Java原子类中的递增操作就通过CAS自旋实现的。

## 锁的范围

- Lock锁的范围有局限性，仅适用于代码块范围
    
- 而synchronized可以锁住代码块、对象实例、类；
    

## 选择性通知|分组唤醒

- synchronized关键字的`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。在使用notify()/notifyAll()方法进行通知时，无法选择性通知
    
- ReentrantLock可实现选择性通知：借助于Condition接口与newCondition()方法，其他线程对象可以注册在指定的Condition中，Condition实例的signalAll()方法只会唤醒注册在该Condition实例中的所有等待线程。
    

# synchronized与volatile

synchronized 关键字和 volatile 关键字区别如下：

- 可见性方面：synchronized和volatile 关键字都可以保证共享变量的可见性。
    
- 有序性方面：synchronized和volatile 关键字都可以保证代码片段的执行有序性
    
- 原子性方面：synchronized 可以保证代码片段的原子性。
    
    - volatile 可以使纯赋值操作是原子的如 boolean flag = true; falg = false。
        
    - 但volatile 不可以保证代码片段的原子性比如类似于 flag = !flag 这种复合操作
        

## 可见性分析(volatile具备)

使用 volatile 修饰共享变量后，同样保证了可见性，原理是总线嗅探机制，在现代计算机中，CPU 的速度是极高的，如果 CPU 需要存取数据时都直接与内存打交道，那么对于CPU而言就是大材小用了，这是一种极大的浪费，所以，为了提高处理速度，CPU 不直接和内存进行通信，而是在 CPU 与内存之间加入很多寄存器，多级缓存，它们比内存的存取速度高得多，这样就解决了 CPU 运算速度和内存读取速度不一致问题。
![[Pasted image 20231225135624.png]]

为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，而==嗅探是实现缓存一致性的常见机制==。每个处理器通过监听在总线上传播的数据来检查自己的缓存值是不是过期了，如果处理器发现自己缓存行对应的内存地址修改，就会将当前处理器的缓存行设置无效状态，当处理器使用这个数据时，会重新从主内存中把数据读到处理器缓存中。

基于 CPU 缓存一致性协议，JVM 实现了 volatile 的可见性，但由于总线嗅探机制，会不断的监听总线，如果大量使用 volatile 会引起==总线风暴==。所以，volatile 的使用要适合具体场景。

另外，还需要注意的是，缓存的一致性问题是多缓存导致的，不是多处理器导致。

## 有序性分析(volatile具备)

volatile 提供了 happens-before 保证，==换句话说，对 `volatile 变量 V` 的写入及之前的写操作 happens-before 所有其他线程后续对 V 的读操作，即写后读==，这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。下面这个例子中， 使用 volatile 修饰 flag 变量后，禁止了代码1和代码2的重排序，也就保证了线程B的执行结果。

线程 A：

```Java
int a = 0;
// 线程 A
a = 1;           // 1
volatile flag = true;     // 2
```

线程 B：

```Java
// 线程 B
if (flag) { // 3
  int i = a; // 4
}
```

重排序禁止规则如下所示：
- 当第一个操作为volatile读时，不论第二个操作是什么，都不能重排序。
- 当第二个操作为volatile写时，不论第一个操作是什么，都不能重排序。
- 当第一个操作为volatile写时，第二个操作为volatile读时，不能重排。

禁止重排序原理如下，Java编译器会根据volatile在适当的位置会插入内存屏障指令来禁止特定类型的处理器重排序。一共有四种内存屏障指令：
- LoadLoad屏障：Load1；LoadLoadBarriers；Load2
- StoreStore屏障：Store1；StoreStoreBarriers；Store2
- LoadStore屏障：Load1；LoadStoreBarriers；Store2
- StoreLoad屏障：Store1；StoreLoadBarriers；Load2；
## 原子性分析(volatile 不具备)

volatile 不具备原子性，因此volatile 无法代替synchronized