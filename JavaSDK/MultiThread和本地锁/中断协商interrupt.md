停止线程显得尤为重要，如取消一个耗时操作。因此，Java提供了一种用于停止线程的机制即中断。中断只是一种协作机制，==Java没有给中断增加任何语法，中断的实现完全需要程序员自己实现。每个线程对象中都有一个标识，仅用于标识线程是否被中断==

- true：表示中断
    
- false：表示未中断
    

若要中断协商一个线程，你需要手动调用该线程的interrupt方法，该方法也仅仅是将线程对象的中断标识设为true。需要搭配其他的中断API进行实现。interrupt可以在别的线程中调用，也可以在自己的线程中调用

Thread的中断API如下：

```Java
/**
具体来说，线程t1线程t2，当线程t2调用t1.interrupt()时，开始中断协商：
如果t1线程处于正常活动状态，当线程t2调用t1.interrupt()时，那么会将t1线程的中断标志设置为true，仅此而已。t1线程将继续正常运行不受影响
如果t1线程处于被阻塞状态，例如处于sleep，wait，join等状态，当线程t2调用t1.interrupt()时，那么t1线程将立即退出被阻塞状态并且抛出interruptedException异常并且把中断状态清除
不活动的线程不会受到任何影响，如果t1已经结束停止了，则线程t2调用t1.interrupt()返回false，对t1不会造成任何影响因为t1早就停止了
*/
public void interrupt()


/**
通过检查中断标志位判断当前线程是否被中断
若中断标志为true，则isInterrupted返回true
若中断标志为false，则isInterrupted返回false
*/
public boolean isInterrupted() 


/**
这个方法做了两件事：
1.返回当前线程的中断状态 
2.将当前线程的中断状态清除，即设为false
假设有两个线程A、B，线程B调用了interrupt方法。后面如果我们连续调用两次interrupted方法，
第一次会返回true，然后这个方法会将中断标识位设置位false,
所以第二次调用interrupted将返回false
*/
public static boolean interrupted() 
```

# interrupt+isInterrupted实现中断机制

```Java
package com.bilibili.juc.interrupt;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * @auther zzyy
 * @create 2022-01-19 11:42
 */
public class InterruptDemo {
    static volatile boolean isStop = false;
    static AtomicBoolean atomicBoolean = new AtomicBoolean(false);

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            while (true) {
                if (Thread.currentThread().isInterrupted()) {
                    System.out.println(Thread.currentThread().getName() + "\t isInterrupted()被修改为true，程序停止");
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