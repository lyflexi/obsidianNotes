停止线程显得尤为重要，如取消一个耗时操作。因此，Java提供了一种用于停止线程的机制即中断。中断只是一种协作机制，==jdk中断函数并不是真正的中断，仅仅是设置了个中断标志，中断的实现完全需要程序员自己实现。每个线程对象中都有一个标识，仅用于标识线程是否被中断==
- true：表示中断
- false：表示未中断

若要中断协商一个线程，你需要手动调用该线程的interrupt方法，该方法也仅仅是将线程对象的中断标识设为true。需要搭配其他的中断API进行实现。interrupt可以在别的线程中调用，也可以在自己的线程中调用

Thread的中断API如下：

```Java
/**
具体来说，线程t1线程t2，当线程t2调用t1.interrupt()时，开始中断协商：
1.如果t1线程处于正常活动状态，当线程t2调用t1.interrupt()时，那么会将t1线程的中断标志设置为true，仅此而已。t1线程将继续正常运行不受影响
2.如果t1线程已经结束停止了，则线程t2调用t1.interrupt()返回false，对t1不会造成任何影响因为t1早就停止了，不活动的线程不会受到任何影响，
3.如果t1线程处于被阻塞状态，例如处于sleep，wait，join等状态，当线程t2调用t1.interrupt()时，那么t1线程将立即退出被阻塞状态并且抛出interruptedException异常并且把中断状态清除，事后调用isInterrupted将返回false
*/
public void interrupt()


/**
通过检查中断标志位判断当前线程是否被中断
若中断标志为true，则isInterrupted返回true
若中断标志为false，则isInterrupted返回false
*/
public boolean isInterrupted() 



/**上述两个实例方法足够应对大部分场景了************************************************************************************
再补充一个静态方法，这个方法做了两件事：返回当前线程的中断状态，并将当前线程的中断状态清除即设为false
假设有两个线程t1、t2，线程t2调用了t1.interrupt()方法。后面如果我们连续调用两次interrupted方法，第一次会返回true，第二次将返回false
*/
public static boolean interrupted() 
```

# 测试打断running线程
interrupt+isInterrupted实现
```Java
/**  
 * @Author: ly  
 * @Date: 2024/2/24 19:35  
 */public class InterruptTest {  
    static volatile boolean isStop = false;  
    static AtomicBoolean atomicBoolean = new AtomicBoolean(false);  
  
    public static void main(String[] args) {  
        Thread t1 = new Thread(() -> {  
            while (true) {  
                if (Thread.currentThread().isInterrupted()) {  
                    System.out.println(Thread.currentThread().getName() + "\t isInterrupted()返回true，并清除中断状态设置为flase，程序停止");  
                    break;  
                }  
                System.out.println("t1 -----hello interrupt api");  
            }  
        }, "t1");  
        t1.start();  
  
        System.out.println("-----t1的默认中断标志位：" + t1.isInterrupted());  
  
        //暂停毫秒  
        try {  
            TimeUnit.MILLISECONDS.sleep(20);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
  
        //t2向t1发出协商，将t1的中断标志位设为true希望t1停下来  
        new Thread(() -> {  
            t1.interrupt();  
        }, "t2").start();  
        //t1.interrupt();，t1也可以自己给自己协商  
  
    }  
}
```

打印信息如下：
```shell
....
t1 -----hello interrupt api
t1 -----hello interrupt api
t1 -----hello interrupt api
t1 -----hello interrupt api
t1 -----hello interrupt api
t1 -----hello interrupt api
t1 -----hello interrupt api
t1	 isInterrupted()返回true，并清除中断状态设置为flase，程序停止
```
# 测试打断sleep线程
如果t1线程处于被阻塞状态，例如处于sleep，wait，join等状态，当线程t2调用t1.interrupt()时，那么t1线程将立即退出被阻塞状态并且抛出interruptedException异常并且把中断状态清除，因此事后调用isInterrupted将返回false
```java
package org.lyflexi.thread.interrupt;  
  
import lombok.extern.slf4j.Slf4j;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/14 9:54  
 */@Slf4j(topic = "c.Test7")  
@Slf4j(topic = "c.Test7")  
public class InterruptSleep {  
    public static void main(String[] args) throws InterruptedException {  
        Thread t1 = new Thread("t1") {  
            @Override  
            public void run() {  
                log.debug("enter sleep...");  
                try {  
                    Thread.sleep(2000);  
                } catch (InterruptedException e) {  
                    log.debug("wake up...");  
                    e.printStackTrace();  
                }  
            }  
        };  
        t1.start();  
  
        Thread.sleep(1000);  
        log.debug("interrupt...");  
        t1.interrupt();  
        log.debug("打断标记:{}", t1.isInterrupted());  
    }  
}
```
打印信息如下：
```java
10:54:47.703 c.Test7 [t1] - enter sleep...
10:54:48.711 c.Test7 [main] - interrupt...
10:54:48.711 c.Test7 [t1] - wake up...
10:54:48.711 c.Test7 [main] - 打断标记:false
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at org.lyflexi.thread.interrupt.InterruptSleep$1.run(InterruptSleep.java:17)
```
# 测试打断park线程
LockSupport.park(); 被interrupte打断之后，无法再次进入park，原因是interrupte设置了中断标志为true
```java
package org.lyflexi.thread.interrupt;  
  
import lombok.extern.slf4j.Slf4j;  
  
import java.util.concurrent.locks.LockSupport;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/14 12:22  
 */@Slf4j(topic = "c.Test14")  
public class InterruptPark {  

    private static void test() throws InterruptedException {  
        Thread t1 = new Thread(() -> {  
            log.debug("park...");  
            LockSupport.park();  
            log.debug("unpark...");  
            log.debug("打断状态：{}", Thread.currentThread().isInterrupted());  
  
            LockSupport.park();  
            log.debug("unpark...");  
        }, "t1");  
        t1.start();  
  
        Thread.sleep(1000);  
        t1.interrupt();  
  
    }  
  
    public static void main(String[] args) throws InterruptedException {  
        test();  
    }  
}
```
打印信息如下：程序终止
```shell
12:27:57.243 c.Test14 [t1] - park...
12:27:58.252 c.Test14 [t1] - unpark...
12:27:58.252 c.Test14 [t1] - 打断状态：true
12:27:58.257 c.Test14 [t1] - unpark...

Process finished with exit code 0
```
要想让LockSupport.park();中断之后还可以再次进入LockSupport.park();，必须事后调用静态方法Thread.interrupted()将中断标志设为false
```java
private static void test2() {  
    Thread t1 = new Thread(() -> {  
        for (int i = 0; i < 5; i++) {  
            log.debug("park..."+i);  
            LockSupport.park();  
            log.debug("打断状态：{}", Thread.interrupted());  
        }  
    });  
    t1.start();  
  
  
    try {  
        Thread.sleep(1000);  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    }  
    t1.interrupt();  
}
```
测试程序不变，打印如下：程序并未终止
```shell
12:29:40.058 c.Test14 [Thread-0] - park...0
12:29:41.059 c.Test14 [Thread-0] - 打断状态：true
12:29:41.063 c.Test14 [Thread-0] - park...1
```
# 两阶段终止模式（多线程设计模式）
 两阶段终止模式Two Phase Termination，在一个线程 T1 中如何“优雅”终止线程 T2？这里的【优雅】指的是给 T2 一个料理后事的机会。 错误思路有两种 
 - 使用线程对象的 stop() 方法停止线程 stop 方法会真正杀死线程，但是通过stop杀死的线程不会释放锁， 可能会造成死锁其它线程将永远无法获取锁 ，由于stop不安全已经被淘汰了
 - 使用 System.exit(int) 方法停止线程 目的仅是停止一个线程，但这种做法会让整个程序都停止

下面我们就模拟一个监控程序，每隔两秒执行监控记录，流程图如下：
如果我们希望优雅的终止监控，那么流程中需要考虑两个打断点：
- Thread.sleep(1000);//可能的打断点1 ，代表睡眠期间被打断
- log.debug("执行监控记录");//可能的打断点2  ，代表真实业务执行期间被打断

![[Pasted image 20240314121807.png]]
TwoPhaseTermination代码如下
```java
package org.lyflexi.thread.interrupt.designParten;  
  
import lombok.extern.slf4j.Slf4j;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/14 11:35  
 */@Slf4j(topic = "c.TwoPhaseTermination")  
public class TwoPhaseTermination {  
    // 监控线程  
    private Thread monitorThread;  
  
    // 启动监控线程  
    public void start() {  
  
        monitorThread = new Thread(() -> {  
            while (true) {  
                Thread current = Thread.currentThread();  
                // 是否被打断  
                if (current.isInterrupted()) {  
                    log.debug("料理后事");  
                    break;  
                }  
                try {  
                    Thread.sleep(1000);//可能的打断点1  
                    log.debug("执行监控记录");//可能的打断点2  
                } catch (InterruptedException e) {  
                    e.printStackTrace();   
                }  
            }  
        }, "monitor");  
        monitorThread.start();  
    }  
  
    // 停止监控线程  
    public void stop() {  
        monitorThread.interrupt();  
    }  
}
```
测试程序
```java
public class Test {  
    public static void main(String[] args) throws InterruptedException {  
        TwoPhaseTermination tpt = new TwoPhaseTermination();  
        tpt.start();  
        Thread.sleep(3500);  
        System.out.println("停止监控");  
  
        tpt.stop();  
    }  
}
```
打印信息如下：可见打断sleep之后抛出了异常，但是monitor线程并未终止，这不符合我们的预期，原因是打断sleep之后中断标记置为false
```shell
12:13:39.565 c.TwoPhaseTermination [monitor] - 执行监控记录
12:13:40.579 c.TwoPhaseTermination [monitor] - 执行监控记录
12:13:41.587 c.TwoPhaseTermination [monitor] - 执行监控记录
停止监控
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at org.lyflexi.thread.interrupt.designParten.TwoPhaseTermination.lambda$start$0(TwoPhaseTermination.java:26)
	at java.lang.Thread.run(Thread.java:750)
12:13:43.061 c.TwoPhaseTermination [monitor] - 执行监控记录
12:13:44.065 c.TwoPhaseTermination [monitor] - 执行监控记录
12:13:45.077 c.TwoPhaseTermination [monitor] - 执行监控记录
12:13:46.087 c.TwoPhaseTermination [monitor] - 执行监控记录
12:13:47.100 c.TwoPhaseTermination [monitor] - 执行监控记录
....。。。。。
```
TwoPhaseTermination程序修改如下：
```java
public void start() {  
  
    monitorThread = new Thread(() -> {  
        while (true) {  
            Thread current = Thread.currentThread();  
            // 是否被打断  
            if (current.isInterrupted()) {  
                log.debug("料理后事");  
                break;  
            }  
            try {  
                Thread.sleep(1000);//可能的打断点1  
                log.debug("执行监控记录");//可能的打断点2  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
                current.interrupt();  
            }  
        }  
    }, "monitor");  
    monitorThread.start();  
}
```
测试程序不变，打印如下
```shell
12:16:41.679 c.TwoPhaseTermination [monitor] - 执行监控记录
12:16:42.696 c.TwoPhaseTermination [monitor] - 执行监控记录
12:16:43.711 c.TwoPhaseTermination [monitor] - 执行监控记录
停止监控
12:16:44.181 c.TwoPhaseTermination [monitor] - 料理后事
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at org.lyflexi.thread.interrupt.designParten.TwoPhaseTermination.lambda$start$0(TwoPhaseTermination.java:26)
	at java.lang.Thread.run(Thread.java:750)

Process finished with exit code 0
```