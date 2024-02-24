`AbstractQueuedSynchronizer(AQS)`，翻译过来叫做抽象队列同步器，它提供了一套可用于实现锁同步机制的框架，不夸张地说，`AQS`是`JUC`同步框架的基石。`AQS`通过一个`FIFO`双向队列维护线程同步状态，实现类只需要继承该类，并重写指定方法即可实现一套线程同步机制。我相信你应该看过源码了，共享资源指的是`state`状态位：

- `state=0`，没占用是0
- `state=1`，占用了是1
- `state>1`，大于1是可重入锁

# 同步队列AQS：CLH变体的虚拟双向队列
![[Pasted image 20231225160355.png]]
## 节点包装
在`AQS`中如果线程获取资源失败，会包装成一个节点挂载到AQS队列上，`AQS`中定义了`Node`类用于包装线程。
![[Pasted image 20231225161340.png]]

`Node`主要包含5个核心字段：

- `waitStatus`：当前节点状态，该字段共有5种取值：
    
    - `0`。节点初始化时的状态。
        
    - `CANCELLED = 1`。节点引用线程由于等待超时或被打断时的状态。
        
    - `SIGNAL = -1`。后继节点线程需要被唤醒时的当前节点状态。
        
    - `CONDITION = -2`。当节点线程进入`condition`队列时的状态。(见`ConditionObject`)
        
    - `PROPAGATE = -3`。仅在释放共享锁`releaseShared`时对头节点使用。(见共享锁分析)
        
- `prev`：前驱节点。
    
- `next`：后继节点。
    
- `thread`：引用线程，头节点不包含线程。
    
- `nextWaiter`：`condition`条件队列。(见`ConditionObject`)
    

## 模板方法

`AQS`内部封装了队列维护逻辑，==采用模版设计模式提供实现类以下方法：==

```Java
tryAcquire(int);        // 尝试获取独占锁，可获取返回true，否则false
tryRelease(int);        // 尝试释放独占锁，可释放返回true，否则false
tryAcquireShared(int);  // 尝试以共享方式获取锁，失败返回负数，只能获取一次返回0，否则返回个数
tryReleaseShared(int);  // 尝试释放共享锁，可获取返回true，否则false
isHeldExclusively();    // 判断线程是否独占资源
```

AQS的方法只是空方法，比如
![[Pasted image 20231225160450.png]]
- 如实现类想要实现独占锁功能，则可只实现`tryAcquire/tryRelease`
- 如实现类想要实现共享锁功能，则可只实现`tryAcquireShared/tryReleaseShared`

虽然实现`tryAcquire/tryRelease/tryAcquireShared/tryReleaseShared`可自行设定逻辑，但建议使用`compareAndSetState`方法对`state`变量进行操作以实现同步类。

如下是一个简单的同步锁实现示例：示例代码简单通过`AQS`实现一个互斥操作，线程1获取`mutex`后，线程2的`acquire`陷入阻塞，直到线程1释放。其中`tryAcquire/acquire/tryRelease/release`的`arg`参数可按实现逻辑自定义传入值，无具体要求。

```Java
public class Mutex extends AbstractQueuedSynchronizer {
    
    @Override
    public boolean tryAcquire(int arg) {
        return compareAndSetState(0, 1);
    }
    
    @Override
    public boolean tryRelease(int arg) {
        return compareAndSetState(1, 0);
    }
    
    public static void main(String[] args) {
        final Mutex mutex = new Mutex();
        
        new Thread(() -> {
            System.out.println("thread1 acquire mutex");
            mutex.acquire(1);
            // 获取资源后sleep保持
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch(InterruptedException ignore) {
                
            }
            mutex.release(1);
            System.out.println("thread1 release mutex");
        }).start();
        
        new Thread(() -> {
            // 保证线程2在线程1启动后执行
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch(InterruptedException ignore) {
                
            }
            // 等待线程1 sleep结束释放资源
            mutex.acquire(1);
            System.out.println("thread2 acquire mutex");
            mutex.release(1);
        }).start()
    }
}
```
# 内置条件队列ConditionObject
但是需要注意，有了AQS队列之后，也不能少了等待队列，而AQS使用的是条件等待队列Condition，因此基于AQS的juc包，远远比基于ObjectMonitor的synchronized关键字更加强大，对比如下：
- AQS：AQS锁阻塞队列+条件等待队列Condition，因为支持多个条件，所以支持多个等待队列Condition
- ObjectMonitor：`_EntryList阻塞队列`+`_WaitSet等待队列`，只支持1个`_WaitSet`等待队列

`AQS`中`Node`除了组成阻塞队列外，还在条件队列Condition中得到应用，`AQS`维护了内部类`ConditionObject`，`ConditionObject`的核心定义为：`ConditionObject`通过`Node`也构成了一个`FIFO`的队列，那么`ConditionObject`为`AQS`提供了怎样的功能呢？

```Java
public class ConditionObject implements Condition, java.io.Serializable {
    ... 
    private transient Node firstWaiter;
    private transient Node lastWaiter;
    ...
}
```

查看`Condition`接口的定义，可以看到其定义的方法与`Object`类的`wait/notify/notifyAll`功能是一致的。
```Java
public interface Condition {
    ...
    void await() throws InterruptedException;
    void signal();
    void signalAll();
    ...
}
```


而`AQS`通过`ConditionObject`同样也提供了`wait/notify`机制的阻塞队列。

1. 在条件队列`Condition`中，`Node`采用`nextWaiter`组成单向链表，
    
2. 当持有锁的线程发起`condition.await`调用后，会包装为`Node`挂载到Condition条件阻塞队列中；
    
3. 当对应`condition.signal`被触发后，==条件队列==中的节点将被唤醒并挂载到AQS锁阻塞队列中。由于`AQS`是`JUC`同步框架的基石，因此`ConditionObject`本质上依然是利用AQS的==锁阻塞（同步）队列==完成的线程同步，只不过`ConditionObject`本身提供一些条件唤醒的能力。略...
![[Pasted image 20231225161215.png]]
# 从ReentrantLock开始解读AQS

本次讲解我们走最常用的ReentrantLock的`lock`/`unlock`方法作为案例突破口
![[Pasted image 20231225160518.png]]

```Java
package com.bilibili.juc.aqs;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class AQSDemo
{
    public static void main(String[] args)
    {
        ReentrantLock reentrantLock = new ReentrantLock();//非公平锁

        // A B C三个顾客，去银行办理业务，A先到，此时窗口空无一人，他优先获得办理窗口的机会，办理业务。
        // A 耗时严重，估计长期占有窗口
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                System.out.println("----come in A");
                //暂停20分钟线程,超长业务
                try { TimeUnit.MINUTES.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
            }finally {
                reentrantLock.unlock();
            }
        },"A").start();

        //B是第2个顾客，B一看到受理窗口被A占用，只能去候客区等待，进入AQS队列，等待着A办理完成，尝试去抢占受理窗口。
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                System.out.println("----come in B");
            }finally {
                reentrantLock.unlock();
            }
        },"B").start();


        //C是第3个顾客，C一看到受理窗口被A占用，只能去候客区等待，进入AQS队列，等待着A办理完成，尝试去抢占受理窗口,前面是B顾客，FIFO
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                System.out.println("----come in C");
            }finally {
                reentrantLock.unlock();
            }
        },"C").start();

        //后续顾客DEFG。。。。。。。以此类推
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                //。。。。。。
            }finally {
                reentrantLock.unlock();
            }
        },"D").start();
    }
}
```

在创建完公平锁/非公平锁后，`ReentrantLock` 会调用`sync.lock`方法进行加锁，最终都会调用到AQS的`acquire`方法
![[Pasted image 20231225160528.png]]

公平锁与非公平锁的lock()方法唯一的区别就在于公平锁在获取同步状态时多了一个限制条件`hasQueuedPredecessors()`，公平锁加锁时依据该条件判断等待队列中是否存在前驱节点
![[Pasted image 20231225160535.png]]

# 非公平锁为例解读AQS

ReentrantLock默认为非公平锁，因为非公平锁有助于提高吞吐量

- 若`state=0`，太好了没人占锁，直接CAS修改状态位state，然后当前线程A占锁`setExclusiveOwnerThread(Thread.currentThread());`
    ![[Pasted image 20231225160547.png]]
    
- 否则，调用acquire去尝试获得锁，分三步`tryAcquire`|`addWaiter`|`acquireQueued`
    ![[Pasted image 20231225160558.png]]
    
## acquire（独占模式）

### tryAcquire

nonfairTryAcquire(acquires)
另外tryAcquire还会执行重入操作

return false
![[Pasted image 20231225160621.png]]

所以!tryAcquire(arg)为true

所以`if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`继续推进条件，走下一步方法`addWaiter`

### addWaiter

`addWaiter`将线程包装为独占节点，尾插式加入到队列中。==此时队列为空，当前线程是B，因此直接进入enq方法==
![[Pasted image 20231225160636.png]]

线程A的时候enq会先添加一个空的头节点，用于创建队列，然后enq继续自旋将线程B入队
![[Pasted image 20231225160643.png]]

将线程B添加到AQS队列，目前线程A持有锁，所以将线程B入队列，==而且线程B来的时候才刚开始创建队列==
![[Pasted image 20231225160650.png]]

将线程C添加到AQS队列
![[Pasted image 20240123142148.png]]
![[Pasted image 20240123142214.png]]
### acquireQueued

`acquireQueued`方法意味着==线程B和线程C==加入阻塞队列后，再次尝试是否可以竞争到锁state，核心逻辑为`shouldParkAfterFailedAcquire`和`parkAndCheckInterrupt`
![[Pasted image 20231225160704.png]]
线程C的前驱节点不是head，所以进入`if(`_`shouldParkAfterFailedAcquire`_`(p, node) &&parkAndCheckInterrupt())`判断
- 前驱节点B默认初始状态等于0，此时线程C将其前驱节点B的状态设为SIGNAL是==-1<0==，表示前驱节点需要被唤醒，返回`true`后继续向下推进`parkAndCheckInterrupt`方法，线程C自身被`park`陷入阻塞。
```java
private final boolean parkAndCheckInterrupt() {  
    LockSupport.park(this);  
    return Thread.interrupted();  
}
```
线程B的前驱节点是head，所以线程B再次尝试tryAcquire来竞争线程A持有的锁
![[Pasted image 20231225160945.png]]
此时线程B，C均在阻塞队列，均等待被唤醒unpark。线程在阻塞队列等呀等，如果不想等了就直接走了，就会调用`cancelAcquire`方法将`node.waitStatus = Node.CANCELLED（1）;`
![[Pasted image 20231225160746.png]]

## release（独占模式）
`release`流程较为简单，主要方法是`unparkSuccessor`，尝试释放`tryRelease`成功后，即从头结点开始唤醒其后继节点
![[Pasted image 20231225160806.png]]
### tryRelease
可重入次数--：int c = getState() - releases;  
判断当前线程是否是锁持有线程，保证不会删错锁，否则抛出IllegalMonitorStateException异常
```java
protected final boolean tryRelease(int releases) {  
    int c = getState() - releases;  
    if (Thread.currentThread() != getExclusiveOwnerThread())  
        throw new IllegalMonitorStateException();  
    boolean free = false;  
    if (c == 0) {  
        free = true;  
        setExclusiveOwnerThread(null);  
    }  
    setState(c);  
    return free;  
}
```
### unparkSuccessor
```java
private void unparkSuccessor(Node node) {  
    /*  
     * If status is negative (i.e., possibly needing signal) try     * to clear in anticipation of signalling.  It is OK if this     * fails or if status is changed by waiting thread.     */    
    int ws = node.waitStatus;  
    if (ws < 0)  
        compareAndSetWaitStatus(node, ws, 0);  
  
    /*  
     * Thread to unpark is held in successor, which is normally     * just the next node.  But if cancelled or apparently null,     * traverse backwards from tail to find the actual     * non-cancelled successor.     */    
    Node s = node.next;  
    if (s == null || s.waitStatus > 0) {  
        s = null;  
        for (Node t = tail; t != null && t != node; t = t.prev)  
            if (t.waitStatus <= 0)  
                s = t;  
    }  
    if (s != null)  
        LockSupport.unpark(s.thread);  
}
```


- if (ws < 0)，则_compareAndSetWaitStatus_(node, ws, 0);
    
- 如后继节点被取消`CANCELLED = 1`，则转为从尾部开始找阻塞的节点将其唤醒。
![[Pasted image 20231225160832.png]]

阻塞节点B被唤醒后，再次进入`acquireQueued`中的`for(;;)`重新竞争资源，竞争成功的线程开始重新设置头节点，==由于是独占模式，因此只唤醒了B线程，只有B线程抢占到资源，本次唤醒与C线程无关==
![[Pasted image 20231225161004.png]]

`setHead`代码执行完毕后，会出现如下图所示

- 原有的傀儡节点离开，有助于GC
    
- 线程B真正的执行业务去了，留下了线程B的空壳节点成为新的傀儡节点
![[Pasted image 20231225161012.png]]

# 共享模式分析

## acquireShared & releaseShared

```Java
public final void acquireShared(int arg) {
    // 负数表示获取共享锁失败，不同于tryAcquire的bool返回
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

`acquireShared`和`releaseShared`整体流程与独占锁类似，`tryAcquireShared`获取失败后以`Node.SHARED`挂载到队尾阻塞，直到队头节点将其唤醒。==在`doAcquireShared`与独占锁不同的是，由于共享锁是可以被多个线程获取的，因此在首个阻塞节点被唤醒后，会通过`setHeadAndPropagate`传递唤醒后续的阻塞节点。==

```Java
// doAcquireShared核心代码
final Node node = addWaiter(Node.SHARED);
...
for (;;) {
    final Node p = node.predecessor();
    if (p == head) {
        int r = tryAcquireShared(arg);
        if (r >= 0) {
            // r>=0 表示获取锁成功，调整头结点并传递唤醒
            setHeadAndPropagate(node, r);
        }
    }
    ...
}
```

`setHeadAndPropagate`和`doReleaseShared`构成共享锁唤醒的核心逻辑。
![[Pasted image 20231225161043.png]]

这两方法的逻辑较为简单，不再进行展开，主要对`setheadAndPropagate`的多节点唤醒判断逻辑做出分析。

进入`setHeadAndPropagate`，首先需要明确的是，该函数的传入参数`propagate`一定是非负数，接下来其唤醒主要为两个判断逻辑：

- 如果`propagate > 0`，表示存在多个共享锁可以获取，可直接进行`doReleaseShared`唤醒阻塞节点。
    
- 如果`propagate = 0`，表示仅当前节点可被唤醒，则有两种情况：
    
    - `h == null || h.waitStatus < 0`，通常情况下`h != null`，现给出`h.waitStatus < 0`的场景。
    ![[Pasted image 20231225161111.png]]
    
    - `(h = head) == null || h.waitStatus < 0`的场景执行序列如下：
    ![[Pasted image 20231225161135.png]]
    

> 独占锁共享锁小结
> 
> 1、独占锁共享锁默认都是非公平获取策略，可能被插队。
> 
> 2、独占锁只有一个线程可获取，其他线程均被阻塞在队列中；共享锁可以有多个线程获取。
> 
> 3、独占锁释放仅唤醒一个阻塞节点，共享锁可以根据可用数量，一次唤醒多个阻塞节点



