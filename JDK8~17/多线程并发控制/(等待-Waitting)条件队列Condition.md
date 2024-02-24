Object监视器之于synchronized：
java.lang.Object中定义了一组监视器方法，例如wait()、wait(long timeout)、wait(long timeout, int nanos)、notify()、notifyAll()而object是任何对象的超类，所以任意java对象都拥有这组监视器方法。这些方法再配合上synchronized同步关键字，我们就可以实现等待/通知机制。

Condition之于Lock：
Condition是一个接口，定义在juc中（java.util.concurrent.locks.Condition），它的主要功能类似于wait()/notify()，但是Condition其实现比wait()/notify()使用更加灵活，简洁、适用场景更加丰富。Condition接口中提供了await()、await(long time, TimeUnit unit)、signal()、signalAll()等方法的定义，这些方法的实现配合上Lock也可实现等待/通知机制。

Condition接口中定义的方法如下，具体方法定义的含义请看方法上的注释
![[Pasted image 20240224125604.png]]
获取Condition必须通过Lock的newCondition方法，这个在Lock接口定义中可以找到结论。
![[Pasted image 20240224125714.png]]

# Condtion源码分析
## condition.await()源码解析
condition.await()的实现位于AbstractQueuedSynchronizer中的ConditionObject

一个await方法涵盖：阻塞（AQS队列->Condition队列） 和  唤醒（Condition队列->AQS队列）

- 调用Condition的await()或者awaitXxxx()会将当前线程构建成Node节点加入Condition的等待队列
- await()方法在调用之前，线程一定获取到了锁，因此addConditionWaiter()无需CAS也可以保证线程安全
- 在阻塞自己之前，必须先释放锁fullyRelease(node)，防止死锁
- isOnSyncQueue(node)用于判断节点是否在AQS同步队列中，如果从Condition的等待队列移动到了AQS的同步队列证明被执行了signal()
- LockSupport.park(this)阻塞自己之后，线程被唤醒的方式有unpark和中断，通过checkInterruptWhileWaiting(node)判断当前线程被唤醒是否是因为中断，如果中断则退出while循环
- 线程从被唤醒后，必须通过acquireQueued(node, savedState)重新获取锁
- 如果线程从await()或者awaitXxxI()方法返回，表明线程又重新获取了Condition相关的锁。
![[Pasted image 20240224130836.png]]
![[Pasted image 20240224130849.png]]
## condition.signal()源码解析

调用Condition的signal()方法，会唤醒在Condition等待队列中的线程节点（唤醒的是等待时间最长的首节点），==唤醒节点之前会将其移至同步队列中（这里要注意先加入同步队列再唤醒该节点）。==

![[Pasted image 20240224130942.png]]
![[Pasted image 20240224130950.png]]

# 图解同步队列与等待队列的关系

==Object监视器锁和Condition接口其中很大的一个区别在于Object的监视模型上，一个对象只拥有一个同步队列和等待队列，这样的模型一个很大的问题在于它不太适用于编写带有多个条件谓词的并发对象（可以简单理解为复杂的带高级功能的）==；而并发包中的Lock组合了Condition对象，使得其可以拥有一个同步队列和多个等待队列（一个Condition中有一个等待队列，不同的Condition队列又被称作条件队列，因此可以做到以条件队列为单位选择性唤醒队列）。


下面就通过图来说明Condition的等待队列和AQS同步队列和等待队列的关系


Condition等待队列图示
![[Pasted image 20240224131210.png]]
![[Pasted image 20240224131221.png]]

## 图解await()方法如何加入等待队列

前面讲过，调用await()方法的前提是获取到了Lock对应的锁，也正是因为这个await()操作是在获取锁的前提下进行的，所以等待队列节点的构造并未使用CAS，因为它的前提条件就是线程互斥（安全）的；

所以加入Condition等待队列的线程，可以理解为在AQS同步器中重新获取到锁的首节点线程被移植（这里的移植不是将以前的节点加入，是通过以前节点的信息构造一个新的线程节点加入到等待队列）到了Condition的等待队列中，其图如下。
![[Pasted image 20240224131407.png]]


## 图解signal()方法如何移出等待队列

signal()方法会将等待队列中的等待时间最长的节点（首节点），移动到同步队列尾部，加入同步队列的代码是 enq(final Node node)，通过CAS来线程安全的移动。移动完成之后线程再使用LockSupport.unpark(node.thread)唤醒该节点中等待的线程。线程节点被移动至同步队列中后，线程可以参与同步状态的竞争，如果竞争成功，线程将会从await()方法返回。其图解如下。
![[Pasted image 20240224131418.png]]