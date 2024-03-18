固定运行顺序 比如，必须先 2 后 1 打印
```java
package org.lyflexi.multiThread.designParten.syncParten.fixedSequence.byLock;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.Condition;  
import java.util.concurrent.locks.ReentrantLock;  
  
@Slf4j(topic = "c.FixedSequenceByLock")  
public class FixedSequenceByLock {  
    static ReentrantLock lock = new ReentrantLock();  
    static Condition condition = lock.newCondition();  
  
    // 关键点，设置布尔值，表示 t2 是否运行过  
    static boolean t2runned = false;  
  
  
    public static void main(String[] args) {  
  
        Thread t1 = new Thread(() -> {  
            lock.lock();  
  
            try {  
                log.debug("线程1进入try块");  
                while (!t2runned) {  
                    try {  
                        condition.await();  
                    } catch (InterruptedException e) {  
                        throw new RuntimeException(e);  
                    }  
                }  
                log.debug("1");  
            }finally {  
                lock.unlock();  
            }  
        }, "t1");  
  
  
        Thread t2 = new Thread(() -> {  
            lock.lock();  
  
            try {  
                log.debug("2");  
                t2runned = true;  
                condition.signal();  
            } finally {  
                lock.unlock();  
            }  
  
  
        }, "t2");  
  
        t1.start();  
        t2.start();  
    }  
}
```
打印信息如下：
```shell
20:56:53.143 c.FixedSequenceByLock [t1] - 线程1进入try块
20:56:53.145 c.FixedSequenceByLock [t2] - 2
20:56:53.145 c.FixedSequenceByLock [t1] - 1

Process finished with exit code 0
```