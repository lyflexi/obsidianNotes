# 线程的六种状态两条队列
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
## 阻塞队列（Blocked）- 用于共享资源互斥访问
线程被阻塞了，“阻塞状态”与“等待状态”的区别是：“阻塞状态”一般在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生，在程序等待进入同步区域的时候，线程将进入这种阻塞状态。
## 等待队列（Waiting）- 用户同步线程通信
处于这种状态的线程不会被分配CPU执行时间，它们要等待被其他线程显式地唤醒，用于线程间通信。某一线程因为调用下列方法之一而处于等待状态：
- 不带超时值的 Object.wait ()
- 不带超时值的 Thread.join ()
- LockSupport.park ()
==与阻塞状态不同的是，等待状态下的线程通常需要其他线程显式调用该对象的notify()或notifyAll()方法来唤醒它。==


# Monitor工作原理-核心态切换
synchronized重量级锁整体的同步流程如下，精妙的图示：
- obj是对象实例位于堆空间，虽然是共享的，但线程们必须操作同一把锁才有效
- 不同的Thread相当于独立的栈，内部持有的目标方法栈帧线程私有
- Monitor位于操作系统底层，访问Monitor需要经历用户态->内核态的转换
![[Pasted image 20240312161518.png]]
- 刚开始Monitor中Owner为null，锁对象只有1把obj位于堆空间
- 当Thread-2执行synchronized(obj)，会将对象头中的MarkWord与底层Monitor相关联，同将Monitor的所有者Owner置为Thread-2，Monitor中只能有一个Owner
- 在Thread-2上锁的过程中，如果Thread-1,Thread-3也来执行synchronized(obj),就会进入EntryList BLOCKED
- Thread-2执行完同步代码块的内容，然后唤醒EntryList中等待的线程来竞争锁，竞争的时是非公平的
- 图中WaitSet是之前获得过锁但条件不满足进入WAITING状态的线程，与wait-notify有关，涉及到线程间通信，本文暂不讨论
# 可重入分析Monitor_recursions
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