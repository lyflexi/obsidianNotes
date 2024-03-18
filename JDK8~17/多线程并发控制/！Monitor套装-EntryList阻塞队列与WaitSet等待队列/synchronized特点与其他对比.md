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

## 有序性分析（宏观有序性）来自于原子性
synchronized保证的有序性是多个线程之间的有序性，即被加锁的内容要按照顺序被多个线程执行。但是其内部的同步代码还是会发生重排序，所以在单例模式中，对单实例singleton的修饰volitile不可少。

[@ZealTalk](https://www.zhihu.com/people/724179916ce56e22decb86dc11782703)说的是 synchronized 可以防止指令重排，这个观点不对的，来讨论这个问题先，先看看 Java 里的操作无序现象是什么：

> 《深入理解 Java 虚拟机》- P374：如果在一个线程观察另一个线程，所有操作都是无序的指的是 “指令重排序” 和 “工作内存与主内存同步延迟” 现象

Java 里只有 volatile 变量是能实现禁止指令重排的

synchronized 虽然不能禁止指令重排，但也能保证有序性？这个有序性是相对语义来看的，线程与线程间，每一个 synchronized 块可以看成是一个原子操作，它保证每个时刻只有一个线程执行同步代码，所以它可以解决上面引述的工作内存和主内存同步延迟现象引发的无序

所以，synchronized 和 volatile 的有序性是两个角度来看的：

- synchronized 是因为块与块之间看起来是原子操作，块与块之间有序可见，其本质是可见性
- volatile 是在底层通过内存屏障防止指令重排的，变量前后之间的指令与指令之间有序可见

同时，synchronized 和 volatile 有序性不同也是因为其实现原理不同：

- synchronized 靠操作系统内核互斥锁实现的，相当于 JMM 中的 lock 和 unlock。退出代码块时一定会刷新变量回主内存
- volatile 靠插入内存屏障指令防止其后面的指令跑到它前面去了

总而言之就是， synchronized 块里的非原子操作依旧可能发生指令重排


> 《Java 并发编程实战》 P287 指出，有 synchronized 无 volatile 的 DCL 会出现的情况：线程可能看到引用的当前值，但对象的状态值确少失效的，这意味着线程可以看到对象处于无效或错误的状态

Stack OverFlow也给出了更详细的解释，为何双重校验单例还要加上 volatile 的原因....[Why is volatile used in double checked locking](https://link.zhihu.com/?target=https%3A//stackoverflow.com/questions/7855700/why-is-volatile-used-in-double-checked-locking)

因为 new 这个指令是非原子操作，底层是分成几条指令来执行的，加上 volatile 是禁止指令重排，保证别的线程读到的时候一定是状态和引用正常的、一个完整的对象，防止其他线程看到的是类似引用有了，内存资源却还没分配的对象。值得一提的是 javap -v 编译出来的字节码指令还不是全部指令，它里面的 new 指令还是能更细分的，因为 volatile 的内存屏障指令还要深入到汇编的层次插入的
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
ReentrantLock基本语法
```java
// 获取锁  
reentrantLock.lock();  
try {  
	// 临界区  
} finally {  
	// 释放锁  
	reentrantLock.unlock();  
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

## 尝试一次
Lock在获取锁的时候尝试只获取一次的API是tryLock()
```java
package org.lyflexi.syncVsReentrantLock;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.ReentrantLock;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 17:02  
 */@Slf4j(topic = "c.TesttryLock")  
public class TesttryLock {  
    public static void main(String[] args) throws InterruptedException {  
        ReentrantLock lock = new ReentrantLock();  
        Thread t1 = new Thread(() -> {  
            log.debug("启动...");  
            if (!lock.tryLock()) {  
                log.debug("获取立刻失败，返回");  
                return;  
            }  
            try {  
                log.debug("获得了锁");  
            } finally {  
                lock.unlock();  
            }  
        }, "t1");  
  
        lock.lock();  
        log.debug("获得了锁");  
        t1.start();  
        try {  
            Thread.sleep(2);  
        } finally {  
            lock.unlock();  
        }  
    }  
}
```
打印信息如下：
```shell
17:02:40.926 c.TesttryLock [main] - 获得了锁
17:02:40.928 c.TesttryLock [t1] - 启动...
17:02:40.928 c.TesttryLock [t1] - 获取立刻失败，返回

Process finished with exit code 0
```
## 超时退出
Lock在获取锁的时候可以设置超时时间，而synchronized在获取锁的时候无法设置超时时间，使用synchronized时，等待的线程会一直等待下去；
Lock在获取锁的时候设置超时时间的API是tryLock(long time, TimeUnit unit)
```Java
package org.lyflexi.syncVsReentrantLock;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.TimeUnit;  
import java.util.concurrent.locks.ReentrantLock;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 17:02  
 */@Slf4j(topic = "c.TesttryLock")  
public class TesttryLockHasTime {  
    public static void main(String[] args) throws InterruptedException {  
        ReentrantLock lock = new ReentrantLock();  
        Thread t1 = new Thread(() -> {  
            log.debug("启动...");  
            try {  
                if (!lock.tryLock(1, TimeUnit.SECONDS)) {  
                    log.debug("获取等待 1s 后失败，返回");  
                    return;  
                }  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
            try {  
                log.debug("获得了锁");  
            } finally {  
                lock.unlock();  
            }  
        }, "t1");  
  
        lock.lock();  
        log.debug("获得了锁");  
        t1.start();  
        try {  
            Thread.sleep(2000);  
        } finally {  
            lock.unlock();  
        }  
    }  
}
```
打印信息如下：
```shell
17:04:30.372 c.TesttryLock [main] - 获得了锁
17:04:30.374 c.TesttryLock [t1] - 启动...
17:04:31.385 c.TesttryLock [t1] - 获取等待 1s 后失败，返回

Process finished with exit code 0
```
## 响应中断
ReentrantLock可打断AQD，synchronized可打断WaitSet而非EntryList
- ReentrantLock另外提供了一种能够中断等待锁的线程的机制，用于打断在阻塞队列AQS中一直等待获取锁的线程，通过 lock.lockInterruptibly() 来实现这个机制。
- 注意，原本synchronized的打断的是调用wait/join方法后位于等待队列WaitSet中的等待线程，而非EntryList中的阻塞线程，因为调用wait的时候synchronized已经释放掉锁不再参与争抢了
ReentrantLock#lockInterruptibly测试中断程序如下：
```java
package org.lyflexi.syncVsReentrantLock;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.ReentrantLock;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 16:43  
 */@Slf4j(topic = "c.TestInterrupt")  
public class TestlockInterruptibly {  
    public static void main(String[] args) {  
  
        ReentrantLock lock = new ReentrantLock();  
  
        Thread t1 = new Thread(() -> {  
            log.debug("启动...");  
            try {  
                lock.lockInterruptibly();  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
                log.debug("等锁的过程中被打断,子线程返回");  
                return;  
            }  
            try {  
                log.debug("获得了锁");  
            } finally {  
                lock.unlock();  
            }  
        }, "t1");  
  
  
        lock.lock();  
        log.debug("主线程获得了锁");  
        t1.start();  
        try {  
            Thread.sleep(1000);  
            t1.interrupt();  
            log.debug("执行打断");  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        } finally {  
            lock.unlock();  
        }  
  
    }  
}
```
打印信息如下：
```shell
16:58:03.063 c.TestInterrupt [main] - 主线程获得了锁
16:58:03.065 c.TestInterrupt [t1] - 启动...
16:58:04.070 c.TestInterrupt [main] - 执行打断
16:58:04.071 c.TestInterrupt [t1] - 等锁的过程中被打断,子线程返回
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at org.lyflexi.syncVsReentrantLock.TestlockInterruptibly.lambda$main$0(TestlockInterruptibly.java:20)
	at java.lang.Thread.run(Thread.java:750)

Process finished with exit code 0
```

## 公平与否

synchronized 和 ReentrantLock 默认是非公平锁，因为非公平锁性能较高：

恢复挂起的线程到真正锁的获取还是有时间差的，从开发人员来看这个时间微乎其微，但是从CPU的角度来看，这个时间存在的还是很明显的，这个时间可以支持非公平锁更充分的利用CPU的时间片，不要死板的排队等待（公平效率低），尽量减少CPU空闲状态时间

可实现公平锁 : ==ReentrantLock正义之锁==，它可以指定是公平锁还是非公平锁，可以构造ReentrantLock(boolean fair)的时候传入true代表公平锁。而synchronized只能是非公平锁。

- Lock公平锁出现的本意是为了解决线程饥饿问题，先入AQS锁阻塞队列的线程在唤醒后优先先使用锁，不过公平锁使用较少，一般没必要使用
- 为了更高的吞吐量往往使用非公平锁，节省很多线程切换时间，吞吐量自然就上去了。

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

## 多条件等待|选择性通知|分组唤醒->正确使用姿势
synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待
ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤 醒
- synchronized关键字使用`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。但是ObjectMonitor不支持选择性通知
- ReentrantLock可实现选择性通知：借助于Condition接口与newCondition()方法，其他线程对象可以注册在指定的Condition中，Condition实例的signalAll()方法只会唤醒注册在该Condition实例中的所有等待线程。
ReentrantLock的正确使用姿势如下，注意事项与synchronized一样
- 使用await 和 signal之前都需要获得锁 
- await 执行后，会释放锁，进入 conditionObject 等待 
- await 的线程被唤醒（或打断、或超时）之后会重新竞争 lock 锁 
- 竞争 lock 锁成功后，从 await 后继续执行
```java
package org.lyflexi.syncVsReentrantLock;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.Condition;  
import java.util.concurrent.locks.ReentrantLock;  
  
  
@Slf4j(topic = "c.TestCondition")  
public class TestLockCondition {  
    static boolean hasCigarette = false;  
    static boolean hasTakeout = false;  
    // 锁，代表这个大房子  
    static ReentrantLock ROOM = new ReentrantLock();  
    // 等待烟的休息室  
    static Condition waitCigaretteSet = ROOM.newCondition();  
    // 等外卖的休息室  
    static Condition waitTakeoutSet = ROOM.newCondition();  
  
    public static void main(String[] args) {  
  
  
        new Thread(() -> {  
            ROOM.lock();  
            try {  
                log.debug("有烟没？[{}]", hasCigarette);  
                while (!hasCigarette) {  
                    log.debug("没烟，先歇会！");  
                    try {  
                        waitCigaretteSet.await();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                log.debug("可以开始干活了");  
            } finally {  
                ROOM.unlock();  
            }  
        }, "小南").start();  
  
        new Thread(() -> {  
            ROOM.lock();  
            try {  
                log.debug("外卖送到没？[{}]", hasTakeout);  
                while (!hasTakeout) {  
                    log.debug("没外卖，先歇会！");  
                    try {  
                        waitTakeoutSet.await();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                log.debug("可以开始干活了");  
            } finally {  
                ROOM.unlock();  
            }  
        }, "小女").start();  
  
  
  
  
        try {  
            Thread.sleep(1000);  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
        new Thread(() -> {  
            ROOM.lock();  
            try {  
                hasTakeout = true;  
                waitTakeoutSet.signal();//signal同样需要在lock块内使用  
            } finally {  
                ROOM.unlock();  
            }  
        }, "送外卖的").start();  
  
        try {  
            Thread.sleep(1000);  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
  
        new Thread(() -> {  
            ROOM.lock();  
            try {  
                hasCigarette = true;  
                waitCigaretteSet.signal();//signal同样需要在lock块内使用  
            } finally {  
                ROOM.unlock();  
            }  
        }, "送烟的").start();  
    }  
  
}
```
打印信息如下：
```shell
19:59:55.572 c.TestCondition [小南] - 有烟没？[false]
19:59:55.576 c.TestCondition [小南] - 没烟，先歇会！
19:59:55.576 c.TestCondition [小女] - 外卖送到没？[false]
19:59:55.576 c.TestCondition [小女] - 没外卖，先歇会！
19:59:56.575 c.TestCondition [小女] - 可以开始干活了
19:59:57.584 c.TestCondition [小南] - 可以开始干活了

Process finished with exit code 0
```
## 锁拥有者线程对象
Synchronized获取锁的时候，先判断共享资源`_count`:
- 如果`_count`为0，则当前线程获取锁，并设置 `_owner` 为当前线程
- 如果`_count`不为0，且`_owner` 为当前线程，则执行可重入， `_recursions+1` 

ReentrantLock获取锁的时候，先判断共享资源state:
- 如果state是0，则调用setExclusiveOwnerThread(current); 设置锁的拥有者为当前线程current = Thread.currentThread();
- 如果state不是0，会调用到getExclusiveOwnerThread()判断当前锁是否已拥有，如果是则`int nextc = c + acquires`，acquire是1
```java
/**  
 * Performs non-fair tryLock.  tryAcquire is implemented in 
 * subclasses, but both need nonfair try for trylock method. */
final boolean nonfairTryAcquire(int acquires) {  
    final Thread current = Thread.currentThread();  
    int c = getState();  
    if (c == 0) {  
        if (compareAndSetState(0, acquires)) {  
            setExclusiveOwnerThread(current);  
            return true;  
        }  
    }  
    else if (current == getExclusiveOwnerThread()) {  
        int nextc = c + acquires;  
        if (nextc < 0) // overflow  
            throw new Error("Maximum lock count exceeded");  
        setState(nextc);  
        return true;  
    }  
    return false;  
}
```
其中setExclusiveOwnerThread和getExclusiveOwnerThread方法是抽象类AbstractOwnableSynchronizer的方法，这两个方法操作的对象都是成员变量`private transient Thread exclusiveOwnerThread; `
```java
public abstract class AbstractOwnableSynchronizer  
    implements java.io.Serializable {  
  
    /** Use serial ID even though all fields transient. */  
    private static final long serialVersionUID = 3737899427754241961L;  
  
    /**  
     * Empty constructor for use by subclasses.     */    
    protected AbstractOwnableSynchronizer() { }  
  
    /**  
     * The current owner of exclusive mode synchronization.     */    
    private transient Thread exclusiveOwnerThread;  
  
    /**  
     * Sets the thread that currently owns exclusive access.     * A {@code null} argument indicates that no thread owns access.  
     * This method does not otherwise impose any synchronization or     * {@code volatile} field accesses.  
     * @param thread the owner thread  
     */    
    protected final void setExclusiveOwnerThread(Thread thread) {  
        exclusiveOwnerThread = thread;  
    }  
  
    /**  
     * Returns the thread last set by {@code setExclusiveOwnerThread},  
     * or {@code null} if never set.  This method does not otherwise  
     * impose any synchronization or {@code volatile} field accesses.  
     * @return the owner thread  
     */    
    protected final Thread getExclusiveOwnerThread() {  
        return exclusiveOwnerThread;  
    }  
}
```
# synchronized与volatile

synchronized 关键字和 volatile 关键字区别如下：
- 可见性方面：synchronized和volatile 关键字都可以保证共享变量的可见性。
- 有序性方面：synchronized的有序性是线程之间的宏观有序性，volitile的有序性是线程内的代码片段的微观执行有序性
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

## 有序性分析(volitile微观有序性)

一个好的内存模型实际上会放松对处理器和编译器规则的束缚，在不改变程序执行结果的前提下，编译器和处理器常常会对指令进行重排序，以尽可能提高执行效率。
![[Pasted image 20231225134456.png]]

单看下面的程序好像没有问题，最后 i 的值是 1。但是在多线程下为了提高性能，编译器会对指令做重排序：假设线程 A 在执行时被重排序成先执行代码 2，再执行代码 1；而线程 B 在线程 A 执行完代码 2 后，读取了 flag 变量。由于条件判断为真，线程 B 将读取变量 a。此时，变量 a 还根本没有被线程 A 写入，那么 i 最后的值是 0，导致执行结果不正确。

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

a = 1和flag = true进行了重排序，所以导致了在多线程环境下会出问题


volatile 提供了禁止重排序的规则如下所示：
- 当第一个操作为volatile读时，不论第二个操作是什么，都不能重排序。
- ==当第二个操作为volatile写时，不论第一个操作是什么，都不能重排序。==
- 当第一个操作为volatile写时，第二个操作为volatile读时，不能重排。
这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。还是上面这个例子， 使用 volatile 修饰 flag 变量后，禁止了代码1和代码2的重排序，也就保证了线程B的执行结果。

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