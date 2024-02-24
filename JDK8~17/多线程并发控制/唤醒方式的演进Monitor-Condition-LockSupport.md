线程等待与唤醒的方式演进：
![[Pasted image 20231225160229.png]]
LockSupport它的解决的痛点：
- LockSupport在代码书写方面不用加锁，也即不用持有锁块
- 唤醒与阻塞不存在先后顺序，即使先调用唤醒再调用阻塞也不会导致程序卡死报异常，因为unpark获得了一个凭证，之后再调用park方法，就可以名正言顺的依据凭证消费
LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。LockSupport类使用了一种名为Permit(许可)的概念来做到阻塞和唤醒线程的功能，每个线程都有一个许可(permit)，permit只有两个值1和零，默认是零。可以把许可看成是一种(0,1)信号量(Semaphore)，但与Semaphore不同的是，许可的累加上限是1
- 阻塞方法park：permit默认是0，所以一开始调用park()方法，当前线程就会阻塞，直到别的线程将当前线程的permit设置为1时, park方法会被唤醒，然后会将permit再次设置为0并返回
- 唤醒方法unpark：就会将thread线程的许可permit设置成1(注意多次调用unpark方法,不会累加，permit值上限是1)会自动唤醒thread线程，即之前阻塞中的LockSupport.park()方法失效
# synchronized时期：必须持有同一把锁

线程1：objectLock.wait();

线程2：objectLock.notify();

正确的等待唤醒顺序：线程t1先等待，线程t2后唤醒t1

若线程t2先唤醒t1，线程t1后调用等待，则会抛出两个异常：

wait异常

notify异常

```Java
package com.bilibili.juc.LockSupport;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @auther zzyy
 * @create 2022-01-20 16:14
 */
public class LockSupportDemo
{

    public static void main(String[] args)
    {
        Object objectLock = new Object();

        new Thread(() -> {
            synchronized (objectLock){
                try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
                System.out.println(Thread.currentThread().getName()+"\t ----come in");
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+"\t ----被唤醒");
            }
        },"t1").start();

        //暂停几秒钟线程
        //try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            synchronized (objectLock){
                objectLock.notify();
                System.out.println(Thread.currentThread().getName()+"\t ----发出通知");
            }
        },"t2").start();

    }
}
打印：
wait异常
notify异常
```

# Lock时期：必须持有同一把锁

线程1：condition.await();

线程2：condition.signal();

正确的等待唤醒顺序：线程t1先等待，线程t2后唤醒t1

若线程t2先唤醒t1，线程t1后调用等待，则会抛出两个异常：

await异常

signal异常

```Java
package com.bilibili.juc.LockSupport;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @auther zzyy
 * @create 2022-01-20 16:14
 */
public class LockSupportDemo
{
    public static void main(String[] args)
    {
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        new Thread(() -> {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            lock.lock();
            try
            {
                System.out.println(Thread.currentThread().getName()+"\t ----come in");
                condition.await();
                System.out.println(Thread.currentThread().getName()+"\t ----被唤醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        },"t1").start();

        //暂停几秒钟线程
        //try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            lock.lock();
            try
            {
                condition.signal();
                System.out.println(Thread.currentThread().getName()+"\t ----发出通知");
            }finally {
                lock.unlock();
            }
        },"t2").start();

    }
}
打印：
await异常
signal异常
```

# LockSupport时期

- 无需使用锁块（synchronized或者lock）
    
- 等待唤醒顺序谁先谁后无所谓，都不会报错，正常执行
    

```Java
package com.bilibili.juc.LockSupport;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @auther zzyy
 * @create 2022-01-20 16:14
 */
public class LockSupportDemo {
    static int x = 0;
    static int y = 0;

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "\t ----come in" + System.currentTimeMillis());
            LockSupport.park();
            System.out.println(Thread.currentThread().getName() + "\t ----被唤醒" + System.currentTimeMillis());
        }, "t1");
        t1.start();

        //暂停几秒钟线程
        //try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            LockSupport.unpark(t1);
            System.out.println(Thread.currentThread().getName() + "\t ----发出通知");
        }, "t2").start();

    }
}
```

打印信息：

```Shell
t2         ----发出通知
t1         ----come in1702804699937
t1         ----被唤醒1702804699944
```