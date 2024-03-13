小故事 - 为什么需要 wait，此时没有烟，由于条件不满足，小南不能继续进行计算，但小南如果一直占用着锁，其它人就得一直阻塞，效率太低
![[Pasted image 20240313140900.png]]
于是老王单开了一间休息室（调用 wait 方法），让小南到休息室（WaitSet）等着去了，但这时锁释放开， 其它人可以由老王随机安排进屋 直到小M将烟送来，大叫一声你的烟到了 （调用 notify 方法）
![[Pasted image 20240313140944.png]]
小南于是可以离开休息室，为了公平起见，小南需要重新进入竞争锁的队列
![[Pasted image 20240313141050.png]]
规范术语描述如下，锁的持有者线程Owner发现条件不满足，调用wait方法，即可进入WaitSet变为WAITING状态，BLOCKED和WAITING的线程都处于阻塞状态，不占用CPU时间片，区别如下：
- BLOCKED线程会在Owner线程释放锁时唤醒
- WAITING线程会在Owner线程调用notify或notifyAll时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入EntryList重新竞争

# wait() 与 notify()
- obj.wait() 让进入 object 监视器的线程到 waitSet 等待 
- obj.notify() 在 object 上正在 waitSet 等待的线程中挑一个唤醒 
- obj.notifyAll() 让 object 上正在 waitSet 等待的线程全部唤醒
它们都是线程之间进行协作的手段，都属于 Object 对象的方法。必须获得此对象的synchronized锁，才能调用这几个方法
```java
package org.lyflexi.synchronize.waitAndNotify2;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/13 14:21  
 */public class TestWaitNotify {  
    final static Object obj = new Object();  
  
    public static void main(String[] args) throws InterruptedException {  
  
        new Thread(() -> {  
            synchronized (obj) {  
                System.out.println("执行....");  
                try {  
                    obj.wait(); // 让线程在obj上一直等待下去  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
                System.out.println("其它代码....");  
            }  
        },"t1").start();  
  
        new Thread(() -> {  
            synchronized (obj) {  
                System.out.println("执行....");  
                try {  
                    obj.wait(); // 让线程在obj上一直等待下去  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
                System.out.println("其它代码....");  
            }  
        },"t2").start();  
  
        // 主线程0.5秒后执行  
        Thread.sleep(500);  
        System.out.println("唤醒 obj 上其它线程");  
        synchronized (obj) {  
//            obj.notify(); // 唤醒obj上一个线程  
            obj.notifyAll(); // 唤醒obj上所有等待线程  
        }  
    }  
}
```
打印信息如下：
```shell
执行....
执行....
唤醒 obj 上其它线程
其它代码....
其它代码....

Process finished with exit code 0
```
# sleep(long n) 与 wait(long n) 
1) sleep 是 Thread 方法，而 wait 是 Object 的方法 
2) sleep 不需要强制和 synchronized 配合使用，但 wait 需要 和 synchronized 一起用 
3) sleep 在睡眠的同时不会释放对象锁的，但 wait 在等待的时候会释放对象锁 更合理
4) 相同点：它们的状态都是 TIMED_WAITING

考虑下面的场景，首先使用sleep实现：
```java
package org.lyflexi.synchronize.sleepAndWait;  
  
public class TestSleep {  
    static final Object room = new Object();  
    static boolean hasCigarette = false; // 有没有烟  
  
    public static void main(String[] args) throws InterruptedException {  
        new Thread(() -> {  
            synchronized (room) {  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                if (!hasCigarette) {  
                    System.out.println("没烟，先歇会！"+System.currentTimeMillis());  
                    try {  
                        Thread.sleep(2000);  
                    } catch (InterruptedException e) {  
                        throw new RuntimeException(e);  
                    }  
                }  
                System.out.println("有烟没？[{}]"+hasCigarette);  
                if (hasCigarette) {  
                    System.out.println("可以开始干活了"+System.currentTimeMillis());  
                }  
            }  
        }, "小南").start();  
  
        for (int i = 0; i < 5; i++) {  
            new Thread(() -> {  
                synchronized (room) {  
                    System.out.println("可以开始干活了");  
                }  
            }, "其它人").start();  
        }  
  
        Thread.sleep(1);  
        new Thread(() -> {  
            // 这里不能加 synchronized (room)否则烟永远送不到，因为sleep不会释放锁  
//            synchronized (room) {  
//                hasCigarette = true;  
//                System.out.println("烟到了噢！");  
//            }  
            hasCigarette = true;  
            System.out.println("烟到了噢！");  
        }, "送烟的").start();  
    }  
  
}
```
打印信息，可以看到存在两个问题：
- 即使烟只用了1秒就提前送到了，由于sleep在阻塞小南线程无法提前唤醒，必须死等2秒钟
- 小南线程阻塞期间不释放锁，两秒内导致其他无关线程也干不了活
```shell
有烟没？[{}]false
没烟，先歇会！1710314695301
烟到了噢！
有烟没？[{}]true
可以开始干活了1710314697316
可以开始干活了
可以开始干活了
可以开始干活了
可以开始干活了
可以开始干活了
```

下面通过wait和notify组合，优化上述问题：
```java
package org.lyflexi.synchronize.sleepAndWait;  
  
  
public class TestWait {  
    static final Object room = new Object();  
    static boolean hasCigarette = false;  
  
  
    public static void main(String[] args) throws InterruptedException{  
        new Thread(() -> {  
            synchronized (room) {  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                if (!hasCigarette) {  
                    System.out.println("没烟，先歇会！"+System.currentTimeMillis());  
                    try {  
                        room.wait(2000);  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                if (hasCigarette) {  
                    System.out.println("可以开始干活了"+System.currentTimeMillis());  
                }  
            }  
        }, "小南").start();  
  
        for (int i = 0; i < 5; i++) {  
            new Thread(() -> {  
                synchronized (room) {  
                    System.out.println("可以开始干活了");  
                }  
            }, "其它人").start();  
        }  
  
        Thread.sleep(1000);  
        new Thread(() -> {  
            synchronized (room) {  
                hasCigarette = true;  
                System.out.println("烟到了噢！");  
                room.notify();  
            }  
        }, "送烟的").start();  
    }  
  
}
```
打印信息如下
- 时间差仅仅是1秒，小南线程就被唤醒了
- 小南线程阻塞期间，wait释放了锁，不影响其他无关线程使用cpu
```shell
有烟没？[{}]false
没烟，先歇会！1710314722570
可以开始干活了
可以开始干活了
可以开始干活了
可以开始干活了
可以开始干活了
烟到了噢！
有烟没？[{}]true
可以开始干活了1710314723584
```
# notify随机唤醒

假如现在不止1个线程在等待呢，使用notify的时候只能是随机唤醒一个线程，有一定概率唤醒的不是我们想要的线程
```java
package org.lyflexi.synchronize.notifyAll;  
  
public class TestNotify {  
    static final Object room = new Object();  
    static boolean hasCigarette = false;  
    static boolean hasTakeout = false;  
  
    // 虚假唤醒  
    public static void main(String[] args) throws InterruptedException{  
        new Thread(() -> {  
            synchronized (room) {  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                if (!hasCigarette) {  
                    System.out.println("没烟，先歇会！"+System.currentTimeMillis());  
                    try {  
                        room.wait();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                if (hasCigarette) {  
                    System.out.println("可以开始干活了"+System.currentTimeMillis());  
                } else {  
                    System.out.println("没干成活...");  
                }  
            }  
        }, "小南").start();  
  
        new Thread(() -> {  
            synchronized (room) {  
                Thread thread = Thread.currentThread();  
                System.out.println("外卖送到没？[{}]"+ hasTakeout);  
                if (!hasTakeout) {  
                    System.out.println("没外卖，先歇会！"+System.currentTimeMillis());  
                    try {  
                        room.wait();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                System.out.println("外卖送到没？[{}]"+ hasTakeout);  
                if (hasTakeout) {  
                    System.out.println("可以开始干活了"+System.currentTimeMillis());  
                } else {  
                    System.out.println("没干成活...");  
                }  
            }  
        }, "小女").start();  
  
        Thread.sleep(1000);  
        new Thread(() -> {  
            synchronized (room) {  
                hasTakeout = true;  
                System.out.println("外卖到了噢！");  
                room.notify();  
            }  
        }, "送外卖的").start();  
  
  
    }  
  
}

```
打印信息：
```shell
有烟没？[{}]false
没烟，先歇会！1710315258900
外卖送到没？[{}]false
没外卖，先歇会！1710315258900
外卖到了噢！
有烟没？[{}]false
没干成活...
```
# 虚假唤醒与处理方式
将notify替换为notifyAll之后，打印信息如下：
虽然小女线程被正确的唤醒了，但是小南线程也被唤醒了小南线程的等待条件并没有满足所以小南没干成活程序却结束了，因此小南被称之为虚假唤醒
```shell
有烟没？[{}]false
没烟，先歇会！1710315470318
外卖送到没？[{}]false
没外卖，先歇会！1710315470319
外卖到了噢！
外卖送到没？[{}]true
可以开始干活了1710315471320
有烟没？[{}]false
没干成活...

Process finished with exit code 0
```
解决虚假唤醒的方式是将if条件判断改为while条件判断，形如while+wait+notifyAll组合的方式
```java
synchronized(lock) {  
    while(条件不成立) {  
    lock.wait();  
	}  
	// 干活  
}  

//另一个线程  
synchronized(lock) {  
    lock.notifyAll();  
}
```
完整代码如下：
```java
package org.lyflexi.synchronize.notifyAll;  
  
public class TestNotifyAllByWhile {  
    static final Object room = new Object();  
    static boolean hasCigarette = false;  
    static boolean hasTakeout = false;  
  
    // 虚假唤醒  
    public static void main(String[] args) throws InterruptedException{  
        new Thread(() -> {  
            synchronized (room) {  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                while (!hasCigarette) {  
                    System.out.println("没烟，先歇会！");  
                    try {  
                        room.wait();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                System.out.println("有烟没？[{}]"+ hasCigarette);  
                if (hasCigarette) {  
                    System.out.println("可以开始干活了"+System.currentTimeMillis());  
                } else {  
                    System.out.println("没干成活...");  
                }  
            }  
        }, "小南").start();  
  
        new Thread(() -> {  
            synchronized (room) {  
                Thread thread = Thread.currentThread();  
                System.out.println("外卖送到没？[{}]"+ hasTakeout);  
                if (!hasTakeout) {  
                    System.out.println("没外卖，先歇会！"+System.currentTimeMillis());  
                    try {  
                        room.wait();  
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
                System.out.println("外卖送到没？[{}]"+ hasTakeout);  
                if (hasTakeout) {  
                    System.out.println("可以开始干活了"+System.currentTimeMillis());  
                } else {  
                    System.out.println("没干成活...");  
                }  
            }  
        }, "小女").start();  
  
        Thread.sleep(1000);  
        new Thread(() -> {  
            synchronized (room) {  
                hasTakeout = true;  
                System.out.println("外卖到了噢！");  
                room.notifyAll();  
            }  
        }, "送外卖的").start();  
  
  
    }  
  
}
```
打印信息如下：可以看到程序并没有结束，只要while (!hasCigarette)，小南就会一直wait，直到小南条件满足为止
```shell
有烟没？[{}]false
没烟，先歇会！
外卖送到没？[{}]false
没外卖，先歇会！1710315790103
外卖到了噢！
外卖送到没？[{}]true
可以开始干活了1710315791110
没烟，先歇会！
```