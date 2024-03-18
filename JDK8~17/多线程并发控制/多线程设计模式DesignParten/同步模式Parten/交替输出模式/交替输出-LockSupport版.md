线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。要求输出 abcabcabcabcabc 怎么实现

# LockSupport初版封装
封装ParkUnpark如下：
//使用ParkUnpark，print这里无需传递current线程，因为park()这个API就是阻塞当前线程
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.byLockSupport;  
  
import java.util.concurrent.locks.LockSupport;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 22:01  
 */public class ParkUnpark {  
    //使用ParkUnpark这里无需传递current线程，因为park()这个API就是阻塞当前线程  
    public void print(String str, Thread next) {  
        for (int i = 0; i < loopNumber; i++) {  
            LockSupport.park();  
            System.out.print(str);  
            LockSupport.unpark(next);  
        }  
    }  
  
    private int loopNumber;  
  
    public ParkUnpark(int loopNumber) {  
        this.loopNumber = loopNumber;  
    }  
}
```
测试程序如下TestParkUnpark：
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.byLockSupport;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.LockSupport;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 22:02  
 */
@Slf4j(topic = "c.TestParkUnpark")  
public class TestParkUnpark {  
    static Thread t1;  
    static Thread t2;  
    static Thread t3;  
    public static void main(String[] args) {  
        ParkUnpark pu = new ParkUnpark(5);  
        t1 = new Thread(() -> {  
            pu.print("a", t2);  
        });  
        t2 = new Thread(() -> {  
            pu.print("b", t3);  
        });  
        t3 = new Thread(() -> {  
            pu.print("c", t1);  
        });  
        t1.start();  
        t2.start();  
        t3.start();  
  
        //启动者  
        LockSupport.unpark(t1);  
    }  
}
```
打印如下：
```shell
abcabcabcabcabc
Process finished with exit code 0
```
# LockSupport进一步封装
//进一步封装线程数组于SyncPark内部，让我们的SyncPark更加健壮，通用性更强
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.byLockSupport2;  
  
import java.util.concurrent.locks.LockSupport;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 22:00  
 */public class ParkUnpark2 {  
    private int loopNumber;  
    private Thread[] threads;  
    public ParkUnpark2(int loopNumber) {  
        this.loopNumber = loopNumber;  
    }  
    //进一步封装线程数组于ParkUnpark2内部，让我们的SyncPark更加健壮  
    public void setThreads(Thread... threads) {  
        this.threads = threads;  
    }  
    public void print(String str) {  
        for (int i = 0; i < loopNumber; i++) {  
            LockSupport.park();  
            System.out.print(str);  
            LockSupport.unpark(nextThread());  
        }  
    }  
    //根据Thread.currentThread()寻找下一个线程  
    private Thread nextThread() {  
        Thread current = Thread.currentThread();  
        int index = 0;  
        for (int i = 0; i < threads.length; i++) {  
            if(threads[i] == current) {  
                index = i;  
                break;  
            }  
        }  
        if(index < threads.length - 1) {  
            return threads[index+1];  
        } else {  
            return threads[0];  
        }  
    }  
    public void start() {  
        for (Thread thread : threads) {  
            thread.start();  
        }  
        LockSupport.unpark(threads[0]);  
    }  
}
```
测试程序如下：
```java
package org.lyflexi.multiThread.designParten.syncParten.alternatelySequence.byLockSupport2;  
  
import lombok.extern.slf4j.Slf4j;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/18 22:02  
 */@Slf4j(topic = "c.TestParkUnpark")  
public class TestParkUnpark2 {  
  
  
    public static void main(String[] args) {  
        ParkUnpark2 parkUnpark2 = new ParkUnpark2(5);  
        Thread t1 = new Thread(() -> {  
            parkUnpark2.print("a");  
        });  
        Thread t2 = new Thread(() -> {  
            parkUnpark2.print("b");  
        });  
        Thread t3 = new Thread(() -> {  
            parkUnpark2.print("c");  
        });  
        parkUnpark2.setThreads(t1, t2, t3);  
        //启动者  
        parkUnpark2.start();  
  
    }  
}
```
打印如下：
```shell
abcabcabcabcabc
Process finished with exit code 0
```