线程的六大状态图如下：
![[Pasted image 20240318115304.png]]

假设有线程 Thread t 
# 情况 1 NEW --> RUNNABLE 
当调用 t.start() 方法时，由 NEW --> RUNNABLE
# RUNNABLE <--> WAITING
## 情况 2 持有syn锁
t 线程用 synchronized(obj) 获取了对象锁后 
- 调用 obj.wait() 方法时，t 线程从 RUNNABLE --> WAITING 
- 调用 obj.notify() ， obj.notifyAll() ， t.interrupt()打断wait 时 
	- 竞争锁成功，t 线程从 WAITING --> RUNNABLE 
	- 竞争锁失败，t 线程从 WAITING --> BLOCKED
调试代码：
```java
package org.lyflexi.multiThread.sixState;  
  
import lombok.extern.slf4j.Slf4j;  
  
@Slf4j(topic = "c.TestWaitNotify")  
public class TestWaitNotify {  
    final static Object obj = new Object();  
  
    public static void main(String[] args) throws InterruptedException{  
  
        new Thread(() -> {  
            synchronized (obj) {  
                log.debug("执行....");  
                try {  
                    obj.wait(); // 让线程在obj上一直等待下去  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
                log.debug("其它代码....");  
            }  
        },"t1").start();  
  
        new Thread(() -> {  
            synchronized (obj) {  
                log.debug("执行....");  
                try {  
                    obj.wait(); // 让线程在obj上一直等待下去  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
                log.debug("其它代码....");  
            }  
        },"t2").start();  
  
        // 主线程两秒后执行  
        Thread.sleep(500);  
        log.debug("唤醒 obj 上其它线程");  
        synchronized (obj) {  
            obj.notifyAll(); // 唤醒obj上所有等待线程  
        }  
    }  
}
```
调试记录：断点设置如下：
![[Pasted image 20240318121059.png]]
注意要让主线程断点设置为Thread模式，这样不会影响子线程t1和t2的执行
![[Pasted image 20240318120959.png]]
Runnable在IDEA调试工具中对应的状态是Running
Blocked在IDEA调试工具中对应的状态Monitor
阶段一：Main线程Running，t1和t2线程均Waitting表示等待
![[Pasted image 20240318121315.png]]
阶段二：Main线程执行了notifyAll之后在释放}锁之前，t1和t2线程均Monitor表示阻塞
![[Pasted image 20240318121435.png]]
阶段三：Main释放}锁，t2线程抢到锁进入Running，t2线程枪锁失败仍然是Monitor
![[Pasted image 20240318121542.png]]
阶段四：t2释放锁之后线程结束之前依然是Running，t2线程终于获得锁也进入Running了
![[Pasted image 20240318121657.png]]
## 情况 3 不持有锁join
当前线程调用 t.join() 方法时，当前线程从 RUNNABLE --> WAITIN
t 线程运行结束，或调用了当前线程的 interrupt() 时，当前线程从 WAITING --> RUNNABLE
## 情况 4 不持有锁park()
当前线程调用 LockSupport.park() 方法会让当前线程从 RUNNABLE --> WAITING 
调用 LockSupport.unpark(目标线程) 或调用了线程的 interrupt() ，会让目标线程从 WAITING --> RUNNABLE



# RUNNABLE <--> TIMED_WAITING

## 情况 5 持有syn锁
t 线程用 synchronized(obj) 获取了对象锁后 
- 调用 obj.wait(long n) 方法时，t 线程从 RUNNABLE --> TIMED_WAITING 
- t 线程等待时间超过了 n 毫秒，或调用 obj.notify() ， obj.notifyAll() ， t.interrupt() 时 
	- 竞争锁成功，t 线程从 TIMED_WAITING --> RUNNABLE 
	- 竞争锁失败，t 线程从 TIMED_WAITING --> BLOCKED
## 情况 6 不持有锁join
- 当前线程调用 t.join(long n) 方法时，当前线程从 RUNNABLE --> TIMED_WAITING 注意是当前线程在t 线程对象的监视器上等待 
- 当前线程等待时间超过了 n 毫秒，或t 线程运行结束，或调用了当前线程的 interrupt() 时，当前线程从 TIMED_WAITING --> RUNNABLE

## 情况 7 不持有锁parkNanos
- 当前线程调用 LockSupport.parkNanos(long nanos) 或 LockSupport.parkUntil(long millis) 时，当前线 程从 RUNNABLE --> TIMED_WAITING 
- 调用 LockSupport.unpark(目标线程) 或调用了线程 的 interrupt() ，或是等待超时，会让目标线程从 TIMED_WAITING--> RUNNABLE

## 情况 8 不持有锁sleep
- 当前线程调用 Thread.sleep(long n) ，当前线程从 RUNNABLE --> TIMED_WAITING 
- 当前线程等待时间超过了 n 毫秒，当前线程从 TIMED_WAITING --> RUNNABLE

# 情况 9 RUNNABLE <--> BLOCKED 
- t 线程用 synchronized(obj) 获取了对象锁时如果竞争失败，从 RUNNABLE --> BLOCKED 
- 持 obj 锁线程的同步代码块执行完毕，会唤醒该对象上所有 BLOCKED 的线程重新竞争，如果其中 t 线程竞争 成功，从 BLOCKED --> RUNNABLE ，其它失败的线程仍然 BLOCKED
# 情况 10 RUNNABLE <--> TERMINATED 
当前线程所有代码运行完毕，进入 TERMINATED