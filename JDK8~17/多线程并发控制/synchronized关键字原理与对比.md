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
答：不能，线程2只能访问该对象的非同步方法。因为执行同步方法时需要获得对象的锁，而线程1在进入`sychronized`修饰的方A时已经获取到了锁，线程2只能等待，无法进入到`synchronized`修饰的方法B，但可以进入到其他非`synchronized`修饰的方法。

**2.修饰静态方法:** 也就是给当前类加锁，会作用于类的所有对象实例 ，进入同步代码前要获得 **当前 class 的锁**。==静态成员不属于任何一个实例对象==。所以，如果一个线程 A 调用一个实例对象的非静态 synchronized 方法，而线程 B 需要调用这个实例对象所属类的静态 synchronized 方法，是允许的，不会发生互斥现象，**因为访问静态 synchronized 方法占用的锁是当前类的锁，而访问非静态 synchronized 方法占用的锁是当前实例对象锁**。

```Java
synchronized static void method() {
//业务代码
}
```

**3.修饰代码块** ：指定加锁对象，对给定对象/类加锁。

synchronized(this|object) 表示进入同步代码块前要获得**给定对象的锁**。

synchronized(类.class) 表示进入同步代码前要获得 **当前 class 的锁**

```Java
synchronized(this) {
  //业务代码
}
```

# synchronized 关键字三大特性

使用synchronized ，以下三大特性均可以保证！
- 原子性（同步）：一个或多个操作要么全部执行成功，要么全部执行失败。`synchronized`关键字可以保证只有一个线程拿到锁，访问共享资源。
- 可见性：当一个线程对共享变量进行修改后，其他线程可以立刻看到。
- 有序性：程序的执行顺序会按照代码的先后顺序执行。

## 原子性分析

不言自明

## 可见性分析

我们来分析下什么是可见性，比如下面的代码示例：

```Java
/**  
 * @Author: ly  
 * @Date: 2024/2/24 17:00  
 */public class VolatileExample {  
  
    /**  
     * main 方法作为一个主线程  
     */  
    public static void main(String[] args) {  
        MyThread myThread = new MyThread();  
        // 开启线程  
        myThread.start();  
  
        // （不加synchronized ）你会发现主线程永远都不会输出有点东西这一段代码，  
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
  
  
  
    /**  
     * 子线程类  
     */  
    static class MyThread extends Thread {  
  
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
}
```

在 JDK1.2 之前，Java 的内存模型实现总是从主存（即共享内存）读取变量，是不需要进行特别的注意的。

而在当前的 Java 内存模型下，不仅线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。（这里所说的共享变量指的是实例变量和类变量，不包含局部变量，因为局部变量是线程私有的，因此不存在竞争问题）
- 线程对变量的所有的操作（读，取）都必须在工作内存中完成，而不能直接读写主内存中的变量。
- 不同线程之间也不能直接访问对方工作内存中的变量，线程间变量的值的传递需要通过主内存中转来完成。
![[Pasted image 20231225134449.png]]

然而，JMM 这样的规定可能会导致线程对共享变量的修改没有即时更新到主内存，或者线程没能够即时将共享变量的最新值同步到工作内存中，从而使得线程在使用共享变量的值时，该值并不是最新的。这也是为什么上述代码（不加synchronized ）你会发现主线程永远都不会输出有点东西这一段代码。

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

在jdk1.6之前，`synchronized`只能实现重量级锁，Java虚拟机是基于Monitor对象来实现重量级锁的。因为监视器锁(monitor)是依赖于底层的操作系统的Mutex Lock来实现的，挂起线程和恢复线程都需要转入内核态去完成，阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态切换需要耗费处理器时间，如果同步代码块中内容过于简单，这种切换的时间可能比用户代码执行的时间还长”，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。
![[Pasted image 20231225135351.png]]

在Hotspot虚拟机中，ObjectMonitor源码是用C++语言编写的，首先我们先下载Hotspot的源码，源码下载链接：http://hg.openjdk.java.net/jdk8/jdk8/hotspot，找到src/share/vm/runtime/objectMonitor.hpp文件。通过`ObjectMonitor.hpp`源码我们知道，wait/notify等方法就依赖于`ObjectMonitor`对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。
## ObjectMonitor#wait/notify测试

```java
  
/*请注意，wait()和notify()方法必须在synchronized块或synchronized方法中使用，否则将抛出IllegalMonitorStateException异常。*/  
public class WaitAndNotify {  
    public static void main(String[] args) {  
        Counter counter = new Counter();  
  
        Thread incrementThread = new Thread(() -> {  
            for (int i = 0; i < 10; i++) {  
                counter.increment();  
            }  
        });  
  
        Thread decrementThread = new Thread(() -> {  
            for (int i = 0; i < 10; i++) {  
                counter.decrement();  
            }  
        });  
  
        incrementThread.start();  
        decrementThread.start();  
    }  
  
    static public class Counter {  
        private int count = 0;  
        private final Object lock = new Object();  
  
        public void increment() {  
            synchronized (lock) {  
                if (count == 10) {  
                    try {  
                        lock.wait();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                count++;  
                System.out.println("Count incremented. Current count: " + count);  
                lock.notify();  
            }  
        }  
  
        public void decrement() {  
            synchronized (lock) {  
                if (count == 0) {  
                    try {  
                        lock.wait();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                count--;  
                System.out.println("Count decremented. Current count: " + count);  
                lock.notify();  
            }  
        }  
    }  
}
```
## ObjectMonitor数据结构与可重入原理
下面简单介绍下ObjectMonitor.hpp数据结构
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

其中 `_owner`、`_count`、`_recursions`、`_WaitSet`和`_EntryList` 字段比较重要
### `_count`和`_recursions`的区别
其中`_count`和`_recursions`的区别如下：
1. `_count`：
    - `_count` 表示当前持有该锁的线程数量。
    - 当 `_count` 为 0 时，表示锁当前未被任何线程持有，其他线程可以尝试获取该锁。
    - 当 `_count` 大于 0 时，表示有一个或多个线程正在持有该锁。
2. `_recursions`：
    - `_recursions` 表示当前线程持有该锁的重入次数。
    - 当一个线程重复获取同一把锁时，它的 `_recursions` 会递增。
    - `_recursions` 主要用于跟踪同一线程对锁的重入情况，允许同一线程多次获取同一把锁而不会导致死锁。

举例来说，假设有一个方法 A 中包含同步块，线程 T1 首次执行该方法时获取了锁，此时 `_count` 为 1，`_recursions` 为 1。然后方法 A 中又调用了另一个同步方法 B，B 方法中同样有一个同步块，这时如果线程 T1 再次获取这个锁，那么 `_count` 不变，但 `_recursions` 会增加。当线程 T1 退出方法 B 后，`_recursions` 会减少，当 `_recursions` 为 0 时，锁才会被释放，`_count` 也会减少，其他线程才有机会获取该锁。

总之，`_count` 表示当前持有锁的线程数，而 `_recursions` 表示当前线程对锁的重入次数。==在标准的Java虚拟机实现中，`_count` 通常不会大于 1。因为Java的锁通常是独占锁，一次只能由一个线程持有。因此，在大多数情况下，`_count` 的最大值为 1。如果 `_count` 大于 1，则意味着某些异常情况发生了，比如可能是虚拟机的实现错误或者是某些非标准的行为。==

### `_EntryList`和`_WaitSet`的区别
1. `_EntryList`：存储了已经获取到锁的线程列表（锁池队列）。
    - 当一个线程成功获取到锁时，它会被加入到 `_EntryList` 中。==虽说`_count` 的最大值为 1，但是`_EntryList`中往往存在许多等待锁的线程，因此`_EntryList`的长度远远不止1==
    - 当该线程释放锁时它会从 `_EntryList` 中移除，同时唤醒 `_WaitSet` 中的线程让新的等待线程参与锁竞争。
2. `_WaitSet`：存储了等待获取锁的线程列表（等待队列）。
    - 当一个线程尝试获取锁但无法立即获得时（锁被其他线程持有），它会被加入到 `_WaitSet` 中，进入等待状态。
    - 当持有锁的线程释放锁时，会从 `_WaitSet` 中选择一个或多个线程唤醒，从 `_WaitSet` 移动到 `_EntryList`让它们尝试再次获取锁。
为什么要设置上述两个队列？在Java中，线程状态被划分为多种，通过查看Thread内的枚举类State，我们可以看到Java线程其实是六种状态
```java
public enum State {
        NEW,
        RUNNABLE,

		/*
		*Thread state for a thread blocked waiting for a monitor lock. A thread in the blocked state is waiting for a           *monitor lock to enter a synchronized block/method or reenter a synchronized block/method after calling 
		*Object.wait.
		*/
        BLOCKED,
        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,
        TIMED_WAITING,
        TERMINATED;
    }
```
#### Java线程的六大状态图示：
![[Pasted image 20240224182840.png]]
线程状态区分为阻塞和等待两种，对比如下：
- 阻塞状态（Blocked）通常发生在线程试图获取对象的同步锁，但该锁被其他线程占用时。在这种情况下，线程会进入锁池，等待锁被释放。阻塞状态是线程主动放弃CPU使用权，暂时停止运行的状态。
- 等待状态（Waiting）则通常发生在线程执行了某些方法，如wait()、join()或sleep()，这些方法会使线程进入等待状态。例如，当线程执行wait()方法时，它会被放入等待池中。==与阻塞状态不同的是，等待状态下的线程通常需要其他线程调用了该对象的notify()或notifyAll()方法来唤醒它，或者等待的时间超时，线程才可以继续执行。==
### 获取Monitor和释放Monitor的整体流程

依据`_recursions`，所以synchronized是可重入锁，synchronized获取Monitor和释放Monitor的流程如下：
![[Pasted image 20231225134517.png]]
![[Pasted image 20231225134525.png]]
  
获取 Monitor 流程：
1. 当一个线程尝试获取某个对象的 Monitor 时，首先检查该 Monitor 是否已被其他线程持有。
2. 如果 Monitor 未被其他线程持有，则当前线程获取该 Monitor，`_EntryList`入队，设置 `_owner` 为当前线程，并将 `_count` 设置为 1，表示当前线程持有该锁，并将 `_recursions` 设置为 0，表示尚未进行重入。
3. 如果 Monitor 已被其他线程持有，则当前线程会被放入该 Monitor 的 `_WaitSet` 中，进入等待状态，直到 Monitor 被释放。

释放 Monitor 流程：
1. 当持有 Monitor 的线程希望释放 Monitor 时，首先会检查是否有其他线程在等待获取该 Monitor。
2. 如果没有等待的线程，则直接释放该 Monitor，将 `_owner` 设置为 null，清空 `_EntryList`，并将 `_count` 设置为 0。
3. 如果有等待的线程，则从 `_WaitSet` 中选择一个或多个线程唤醒，让它们尝试再次获取 Monitor。同时，当前持有 Monitor 的线程会释放该 Monitor，并将 `_owner` 设置为 null，清空 `_EntryList`，并将 `_count` 设置为 0。

在获取 Monitor 和释放 Monitor 的过程中，需要注意以下几点：

- 获取 Monitor 时，如果是同一线程再次获取 Monitor（即重入），则会增加 `_recursions`，而不会加入到 `_EntryList` 中。
- 当释放 Monitor 时，需要根据情况决定是否唤醒等待线程，以及如何选择唤醒的线程。


## synchronized 同步语句块的情况
测试程序：
```Java
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("synchronized 代码块");
        }
    }
}
```
下面通过 JDK 自带的 命令查看字节码信息来验证synchronized原理
1. 首先切换到类的对应目录执行 javac SynchronizedDemo.java 命令生成编译后的 .class 文件
2. 然后执行javap -c -s -v -l SynchronizedDemo.class命令查看 SynchronizedDemo 类的相关字节码信息。

从下面我们可以看出：
**synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中：**
- **monitorenter 指令指向同步代码块的开始位置**
- **monitorexit 指令则指明同步代码块的结束位置。**
    

synchronized用的是监视器锁（monitor），是依赖于底层的操作系统的 Mutex Lock 来实现的。当执行 monitorenter 指令时，线程试图获取 **对象监视器 monitor** 的持有权。
![[Pasted image 20231225135440.png]]

## synchronized 修饰方法的的情况
测试程序
```Java
public class SynchronizedDemo2 {
    public synchronized void method() {
        System.out.println("synchronized 方法");
    }
}
```
Asynchronized 修饰方法通过该 `flags：ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。
![[Pasted image 20231225135457.png]]
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

synchronized修饰的代码在执行异常时，jdk会自动释放线程占有的锁，不需要程序员去控制释放锁。

但是，当Lock发生异常时，如果程序没有通过unLock()去释放锁，则很可能造成死锁现象，因此Lock一般都是在finally块中释放锁；格式如下：

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

## 超时退出

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
## 响应中断
ReentrantLock另外提供了一种能够中断等待锁的线程的机制，通过 lock.lockInterruptibly() 来实现这个机制。也就是说正在等待的线程可以选择直接放弃等待，改为处理其他事情。
## 公平与否

synchronized 和 ReentrantLock 默认是非公平锁，因为非公平锁性能较高：

恢复挂起的线程到真正锁的获取还是有时间差的，从开发人员来看这个时间微乎其微，但是从CPU的角度来看，这个时间存在的还是很明显的，这个时间可以支持非公平锁更充分的利用CPU的时间片，不要死板的排队等待（公平效率低），尽量减少CPU空闲状态时间

可实现公平锁 : ==ReentrantLock正义之锁==，它可以指定是公平锁还是非公平锁，可以构造ReentrantLock(boolean fair)的时候传入true代表公平锁。而synchronized只能是非公平锁。

什么时候用公平？什么时候用非公平？

如果为了更高的吞吐量，很显然非公平锁是比较合适的,因为节省很多线程切换时间,吞吐量自然就上去了。否则那就用公平锁,大家公平使用

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

②. 乐观锁：乐观锁一般有两种实现方式，采用版本号机制（Version）或CAS（Compare and Swap）算法实现，最常采用的是CAS算法。以CAS为例，==乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁。只是在更新数据的时候去判断之前有没有别的线程更新了这个数据，如果这个数据没有被更新，当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则本次CAS失败==。乐观锁在Java中通过使用无锁编程来实现，适合读操作多的场景,不加锁的特点能够使其读操作的性能大幅度提升。Java原子类中的递增操作就通过CAS自旋实现的。
## 锁的范围
- Lock锁的范围有局限性，仅适用于代码块范围
- 而synchronized可以锁住代码块、对象实例、类；

## 选择性通知|分组唤醒

- synchronized关键字使用`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。但是ObjectMonitor不支持选择性通知
- ReentrantLock可实现选择性通知：借助于Condition接口与newCondition()方法，其他线程对象可以注册在指定的Condition中，Condition实例的signalAll()方法只会唤醒注册在该Condition实例中的所有等待线程。

# synchronized与volatile

synchronized 关键字和 volatile 关键字区别如下：
- 可见性方面：synchronized和volatile 关键字都可以保证共享变量的可见性。
- 有序性方面：synchronized和volatile 关键字都可以保证代码片段的执行有序性
- 原子性方面：synchronized 可以保证代码片段的原子性。
    - volatile 可以使纯赋值操作是原子的如 boolean flag = true; falg = false。
    - 但volatile 不可以保证代码片段的原子性比如类似于 count++ 这种复合操作

三大特点的对比见下文：

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

对`volatile`变量简单的读取和写回操作是立即可见的，但对于复合操作（count++），`volatile`不能确保其原子性，假设有一个计数器类，count被声明为volatile
```java
public class Counter {
    private volatile int count = 0;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}

```
现在假设有两个线程同时对这个计数器进行增加操作：
```java
Counter counter = new Counter();

// 线程1增加计数器值
Thread thread1 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
});

// 线程2增加计数器值
Thread thread2 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
});

thread1.start();
thread2.start();

// 等待两个线程执行完成
thread1.join();
thread2.join();

System.out.println("Final count: " + counter.getCount());

```
理想情况下，如果`increment()`方法是原子的，那么最终计数器的值应该是2000。但是，由于`increment()`方法不是原子的，它涉及到读取、递增和写回三个步骤，而`volatile`只确保了读取和写回的可见性，所以两个线程同时对计数器执行`increment()`操作时，会产生竞态条件，导致最终的计数器值小于2000。