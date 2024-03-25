对于Lock，支持多个Condition等待队列，以此为突破点
- 让每个线程各自关联各自的Condition
- 由于每个condition中只有1个线程，因此无需while(!条件)去防止虚假唤醒，这一点是比sync版更优化的地方
- 唤醒下个线程的Condition
面向对象封装AwaitSignal类，AwaitSignal继承自ReentrantLock
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.byCondition;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.Condition;  
import java.util.concurrent.locks.ReentrantLock;  
@Slf4j(topic = "c.ReentrantLock")  
class AwaitSignal extends ReentrantLock {  
    private int loopNumber;  
  
    public AwaitSignal(int loopNumber) {  
        this.loopNumber = loopNumber;  
    }  
  
    // 参数1 打印内容， 参数2 进入哪一间休息室, 参数3 下一间休息室  
    public void print(String str, Condition current, Condition next) {  
        for (int i = 0; i < loopNumber; i++) {  
            lock();  
            //由于每个condition中只有1个线程，因此无需while(!条件)去防止虚假唤醒，这一点是比sync版更优化的地方
            try {  
                current.await();  
                System.out.print(str);  
                next.signal();  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            } finally {  
                unlock();  
            }  
        }  
    }  

	//唤醒第一个condition
    public void start(Condition first) {  
        this.lock();  
        try {  
            log.debug("start");  
            first.signal();  
        } finally {  
            this.unlock();  
        }  
    }  
  
}
```
测试类TestByWaitNotify
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.byCondition;  
  
import java.util.concurrent.locks.Condition;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 21:41  
 */public class TestAwaitSignal {  
    public static void main(String[] args) throws InterruptedException {  
        AwaitSignal as = new AwaitSignal(5);  
        Condition aWaitSet = as.newCondition();  
        Condition bWaitSet = as.newCondition();  
        Condition cWaitSet = as.newCondition();  
        new Thread(() -> {  
            as.print("a", aWaitSet, bWaitSet);  
        }).start();  
        new Thread(() -> {  
            as.print("b", bWaitSet, cWaitSet);  
        }).start();  
        new Thread(() -> {  
            as.print("c", cWaitSet, aWaitSet);  
        }).start();  
        as.start(aWaitSet);  
  
    }  
}
```
打印信息如下：
```shell
21:42:59.665 c.ReentrantLock [main] - start
abcabcabcabcabc
Process finished with exit code 0

```