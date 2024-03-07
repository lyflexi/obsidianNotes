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
Java源码下载链接：http://hg.openjdk.java.net/jdk8/jdk8/hotspot

在Hotspot虚拟机中，锁的实现依赖于对象监视器src/share/vm/runtime/objectMonitor.hpp。wait/notify等方法就依赖于`ObjectMonitor`对象，所以wait/notify等方法必需在获得ObjectMonitor锁的代码块内调用，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。

## 原生方法wait/notify-重要！

在Java程序中，`synchronized`解决了多线程竞争的问题(原子性)。例如，对于一个任务管理器，多个线程同时往队列中添加任务，可以用`synchronized`加锁：
```java
class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
    }
}
```

但是`synchronized`并没有解决多线程协调的问题。
仍然以上面的`TaskQueue`为例，我们再编写一个`getTask()`方法取出队列的第一个任务：

```java
class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
    }

    public synchronized String getTask() {
        while (queue.isEmpty()) {
        }
        return queue.remove();
    }
}
```

上述代码看上去没有问题：`getTask()`内部先判断队列是否为空，如果为空，就循环等待，直到另一个线程往队列中放入了一个任务，`while()`循环退出，就可以返回队列的元素了。

但实际上`while()`循环永远不会退出。因为线程在执行`while()`循环时，已经在`getTask()`入口获取了`this`锁，其他线程根本无法调用`addTask()`，因为`addTask()`执行条件也是获取`this`锁。

因此，执行上述代码，线程会在`getTask()`中因为死循环而100%占用CPU资源。

如果深入思考一下，我们想要的执行效果是：
- 线程1可以调用`addTask()`不断往队列中添加任务；
- 线程2可以调用`getTask()`从队列中获取任务。如果队列为空，则`getTask()`应该等待，直到队列中至少有一个任务时再返回。

因此，多线程协调运行的原则就是：当条件不满足时，线程进入等待状态；当条件满足时，线程被唤醒，继续执行任务。

对于上述`TaskQueue`，我们先改造`getTask()`方法，在条件不满足时，线程进入等待状态，直到将来某个时刻，线程从等待状态被其他线程唤醒后，`wait()`方法才会返回，然后，继续执行下一条语句。
```java
public synchronized String getTask() {
    while (queue.isEmpty()) {
        // 释放this锁:
        this.wait();
        // 重新获取this锁
    }
    return queue.remove();
}
```
有些仔细的童鞋会指出：即使线程在`getTask()`内部等待，其他线程如果拿不到`this`锁，照样无法执行`addTask()`，肿么办？
这个问题的关键就在于`wait()`方法的执行机制非常复杂：
- `wait()`方法调用时，会释放线程获得的锁
- `wait()`方法返回后，线程又会重新试图获得锁。

现在我们面临第二个问题：如何让等待的线程被重新唤醒，然后从`wait()`方法返回？答案是在相同的锁对象上调用`notify()`方法。我们修改`addTask()`如下：
```java
public synchronized void addTask(String s) {
    this.queue.add(s);
    this.notify(); // 唤醒在this锁等待的线程
}
```

我们来看一个完整的例子：这个例子中，我们重点关注`addTask()`方法，内部调用了`this.notifyAll()`而不是`this.notify()`，使用`notifyAll()`将唤醒所有当前正在`this`锁等待的线程，而`notify()`只会唤醒其中一个（具体哪个依赖操作系统，有一定的随机性）。这是因为可能有多个线程正在`getTask()`方法内部的`wait()`中等待，使用`notifyAll()`将一次性全部唤醒。通常来说，`notifyAll()`更安全。有些时候，如果我们的代码逻辑考虑不周，用`notify()`会导致只唤醒了一个线程，而其他线程可能永远等待下去醒不过来了。
```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        var q = new TaskQueue();
        var ts = new ArrayList<Thread>();
        for (int i=0; i<5; i++) {//5个消费者线程
            var t = new Thread() {
                public void run() {
                    // 执行task:
                    while (true) {
                        try {
                            String s = q.getTask();
                            System.out.println("execute task: " + s);
                        } catch (InterruptedException e) {
                            return;
                        }
                    }
                }
            };
            t.start();
            ts.add(t);
        }
        var add = new Thread(() -> {//1个消费者线程
            for (int i=0; i<10; i++) {
                // 放入task:
                String s = "t-" + Math.random();
                System.out.println("add task: " + s);
                q.addTask(s);
                try { Thread.sleep(100); } catch(InterruptedException e) {}
            }
        });
        add.start();
        add.join();
        Thread.sleep(100);
        for (var t : ts) {
            t.interrupt();
        }
    }
}

class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
        this.notifyAll();
    }

    public synchronized String getTask() throws InterruptedException {
        while (queue.isEmpty()) {
            this.wait();
        }
        return queue.remove();
    }
}

```
打印如下：
```shell
add task: t-0.9219689522548505
execute task: t-0.9219689522548505
add task: t-0.271813708432178
execute task: t-0.271813708432178
add task: t-0.8644114626575558
execute task: t-0.8644114626575558
add task: t-0.32891791164139694
execute task: t-0.32891791164139694
add task: t-0.011217614729680414
execute task: t-0.011217614729680414
add task: t-0.06744186559914589
execute task: t-0.06744186559914589
add task: t-0.5643259633238602
execute task: t-0.5643259633238602
add task: t-0.9453978156714784
execute task: t-0.9453978156714784
add task: t-0.6959170743585548
execute task: t-0.6959170743585548
add task: t-0.6532340394528874
execute task: t-0.6532340394528874

Process finished with exit code 0
```
但是，注意到`wait()`方法返回时需要重新获得`this`锁。假设当前有3个线程被唤醒，唤醒后，首先要等待执行`addTask()`的线程结束此方法后，才能释放`this`锁，随后，这3个线程中只能有一个获取到`this`锁，剩下两个将继续等待。

再注意到我们在`while()`循环中调用`wait()`，而不是`if`语句，这是因为线程被唤醒时，需要再次获取this锁。多个线程被唤醒后，只有一个线程能获取this锁，此刻，该线程执行queue.remove()可以获取到队列的元素，然而，剩下的线程如果获取this锁后执行queue.remove()，此刻队列可能已经没有任何元素了，所以，要始终在while循环中wait()，并且每次被唤醒后拿到this锁就必须再次判断：

```java
public synchronized String getTask() throws InterruptedException {
	while (queue.isEmpty()) {
		this.wait();
	}
	return queue.remove();
}
```

所以，正确编写多线程代码是非常困难的，需要仔细考虑的条件非常多，任何一个地方考虑不周，都会导致多线程运行时不正常。总结以下，`wait`和`notify`用于多线程协调运行：
- 在`synchronized`内部可以调用`wait()`使线程进入等待状态；
- 必须在已获得的锁对象上调用`wait()`方法；
- 在`synchronized`内部可以调用`notify()`或`notifyAll()`唤醒其他等待线程；
- 必须在已获得的锁对象上调用`notify()`或`notifyAll()`方法；
- 已唤醒的线程还需要重新获得锁后才能继续执行。
## 线程的六种状态两条队列
从上面的例子中我们能够感受到，synchronized保证原子性依赖于锁阻塞队列，wait/notify协调多线程工作用到的是等待队列

所以在Java中线程的六大状态中，有三大状态都与锁阻塞队列和线程等待队列有关：
- 锁阻塞队列：对应于BLOCKED状态
- 线程等待队列：对应于无限期等待WAITING状态和限期等待TIMED_WAITING状态
```java
public enum State {
	NEW,
	RUNNABLE,
	BLOCKED,
	WAITING,
	TIMED_WAITING,//既可以被其他线程显式地唤醒，也可以在一定时间之后会由系统自动唤醒
	TERMINATED;
}
```
Java线程的六大状态图示：
![[Pasted image 20240224182840.png]]
最重要的阻塞状态与等待状态的对比如下：
#### 阻塞队列（Blocked）
线程被阻塞了，“阻塞状态”与“等待状态”的区别是：“阻塞状态”一般在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生，在程序等待进入同步区域的时候，线程将进入这种阻塞状态。
#### 等待队列（Waiting）：
处于这种状态的线程不会被分配CPU执行时间，它们要等待被其他线程显式地唤醒。某一线程因为调用下列方法之一而处于等待状态：
- 不带超时值的 Object.wait ()
- 不带超时值的 Thread.join ()
- LockSupport.park ()
==与阻塞状态不同的是，等待状态下的线程通常需要其他线程显式调用该对象的notify()或notifyAll()方法来唤醒它。==
## ObjectMonitor五大字段与可重入原理
下面简单介绍下ObjectMonitor.hpp数据结构
```C++
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; 
    _waiters      = 0, 
    _recursions   = 0; 
    _object       = NULL; 
    _owner        = NULL;
    _WaitSet      = NULL; 
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; 
    FreeNext      = NULL ;
    _EntryList    = NULL ; 
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

其中 `_owner`、`_count`、`_recursions`、`_WaitSet`和`_EntryList` 字段比较重要
- `_owner`指向持有ObjectMonitor对象的线程地址，也即代表了持有锁的当前线程
- `_count`锁的计数器， `_count` 为 0 时表示锁当前未被任何线程持有，其他线程可以尝试获取该锁。一般是互斥锁，该值最大为1。该值不负责可重入计数。
- `_recursions`：锁的重入次数
- `_WaitSet`处于wait状态的线程，会被加入到_WaitSet
- `_EntryList`处于等待锁block状态的线程，会被加入到_EntryList

依据`_recursions`，所以synchronized是可重入锁，synchronized获取Monitor和释放Monitor的流程如下：
![[Pasted image 20231225134517.png]]
![[Pasted image 20231225134525.png]]
  
获取 Monitor 流程：
1. 如果 Monitor 已被其他线程持有，则当前线程会被放入该 Monitor 的 `_EntryList` 中，表示阻塞
2. 直到 Monitor 被释放，`_count`为0，当前线程与其他阻塞线程一同竞争锁，看是否能争抢到锁
3. 当前线程获取到锁，将`_count` 设置为 1，并设置 `_owner` 指向当前线程，并将 `_recursions` 初始化为 0，表示尚未进行重入。
4. 同一个线程根据`_owner`判断是否允许重入，如果`_owner` 为当前线程，则执行可重入， `_recursions+1` 
5. 上述过程与`_WaitSet`无关
释放 Monitor 流程：
1. `_recursions`大于0，则先执行`_recursions-1`，直至`_recursions`为0
2. 释放锁，将 `_owner` 设置为 null，并将 `_count` 设置为 0。此时允许 `_EntryList`中的阻塞线程来争抢这把锁

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
![[Pasted image 20231225135440.png]]
synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中：
- monitorenter 指令指向同步代码块的开始位置，当执行 monitorenter 指令时，线程试图获取 对象监视器 monitor 的持有权。
- monitorexit 指令则指明同步代码块的结束位置。
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