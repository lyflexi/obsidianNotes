线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。要求输出 abcabcabcabcabc 怎么实现

面向对象封装AwaitSignal类，AwaitSignal继承自ReentrantLock
由于每个condition中只有1个线程，因此无需while(!条件)去防止虚假唤醒，这一点是比sync版更优化的地方
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
  
    //            参数1 打印内容， 参数2 进入哪一间休息室, 参数3 下一间休息室  
    public void print(String str, Condition current, Condition next) {  
        for (int i = 0; i < loopNumber; i++) {  
            lock();  
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
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.bySync;  
  
import lombok.extern.slf4j.Slf4j;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 21:23  
 */
@Slf4j(topic = "c.TestByWaitNotify")  
public class TestByWaitNotify {  
    public static void main(String[] args) {  
        WaitNotify wn = new WaitNotify(1, 5);  
        new Thread(() -> {  
            wn.print("a", 1, 2);  
        }).start();  
        new Thread(() -> {  
            wn.print("b", 2, 3);  
        }).start();  
        new Thread(() -> {  
            wn.print("c", 3, 1);  
        }).start();  
    }  
}
```
打印信息如下：
```shell
21:42:59.665 c.ReentrantLock [main] - start
abcabcabcabcabc
Process finished with exit code 0

```