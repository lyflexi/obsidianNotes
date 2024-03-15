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
# 虚假唤醒与处理方式while+notifyAll
将notify替换为notifyAll之后，打印信息如下：
虽然小女线程被正确的唤醒了，但是小南线程也被唤醒了小南线程的等待条件并没有满足，这种情况称之为称之为虚假唤醒，最终导致的结果就是小南没干成活程序却结束了
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

# 虚假唤醒与处理方式while+notifyAll
Java源码下载链接：http://hg.openjdk.java.net/jdk8/jdk8/hotspot

在Hotspot虚拟机中，锁的实现依赖于对象监视器src/share/vm/runtime/objectMonitor.hpp。wait/notify等方法就依赖于`ObjectMonitor`对象，所以wait/notify等方法必需在获得ObjectMonitor锁的代码块内调用，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。
## 原生方法wait/notify-重要！

在Java程序中，`synchronized`解决了多线程竞争的问题(原子性)。例如，对于一个任务管理器，多个线程同时往队列中添加任务，可以用`synchronized`加锁：
```java
class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
    }
}
```

但是`synchronized`并没有解决多线程协调的问题。
仍然以上面的`TaskQueue`为例，我们再编写一个`getTask()`方法取出队列的第一个任务：

```java
class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
    }

    public synchronized String getTask() {
        while (queue.isEmpty()) {
        }
        return queue.remove();
    }
}
```

上述代码看上去没有问题：`getTask()`内部先判断队列是否为空，如果为空，就循环等待，直到另一个线程往队列中放入了一个任务，`while()`循环退出，就可以返回队列的元素了。

但实际上`while()`循环永远不会退出。因为线程在执行`while()`循环时，已经在`getTask()`入口获取了`this`锁，其他线程根本无法调用`addTask()`，因为`addTask()`执行条件也是获取`this`锁。

因此，执行上述代码，线程会在`getTask()`中因为死循环而100%占用CPU资源。

如果深入思考一下，我们想要的执行效果是：
- 线程1可以调用`addTask()`不断往队列中添加任务；
- 线程2可以调用`getTask()`从队列中获取任务。如果队列为空，则`getTask()`应该等待，直到队列中至少有一个任务时再返回。

因此，多线程协调运行的原则就是：当条件不满足时，线程进入等待状态；当条件满足时，线程被唤醒，继续执行任务。

对于上述`TaskQueue`，我们先改造`getTask()`方法，在条件不满足时，线程进入等待状态，直到将来某个时刻，线程从等待状态被其他线程唤醒后，`wait()`方法才会返回，然后，继续执行下一条语句。
```java
public synchronized String getTask() {
    while (queue.isEmpty()) {
        // 释放this锁:
        this.wait();
        // 重新获取this锁
    }
    return queue.remove();
}
```
有些仔细的童鞋会指出：即使线程在`getTask()`内部等待，其他线程如果拿不到`this`锁，照样无法执行`addTask()`，肿么办？
这个问题的关键就在于`wait()`方法的执行机制非常复杂：
- `wait()`方法调用时，会释放线程获得的锁
- `wait()`方法返回后，线程又会重新试图获得锁。

现在我们面临第二个问题：如何让等待的线程被重新唤醒，然后从`wait()`方法返回？答案是在相同的锁对象上调用`notify()`方法。我们修改`addTask()`如下：
```java
public synchronized void addTask(String s) {
    this.queue.add(s);
    this.notify(); // 唤醒在this锁等待的线程
}
```

我们来看一个完整的例子：这个例子中，我们重点关注`addTask()`方法，内部调用了`this.notifyAll()`而不是`this.notify()`，使用`notifyAll()`将唤醒所有当前正在`this`锁等待的线程，而`notify()`只会唤醒其中一个（具体哪个依赖操作系统，有一定的随机性）。这是因为可能有多个线程正在`getTask()`方法内部的`wait()`中等待，使用`notifyAll()`将一次性全部唤醒。通常来说，`notifyAll()`更安全。有些时候，如果我们的代码逻辑考虑不周，用`notify()`会导致只唤醒了一个线程，而其他线程可能永远等待下去醒不过来了。
```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        var q = new TaskQueue();
        var ts = new ArrayList<Thread>();
        for (int i=0; i<5; i++) {//5个消费者线程
            var t = new Thread() {
                public void run() {
                    // 执行task:
                    while (true) {
                        try {
                            String s = q.getTask();
                            System.out.println("execute task: " + s);
                        } catch (InterruptedException e) {
                            return;
                        }
                    }
                }
            };
            t.start();
            ts.add(t);
        }
        var add = new Thread(() -> {//1个消费者线程
            for (int i=0; i<10; i++) {
                // 放入task:
                String s = "t-" + Math.random();
                System.out.println("add task: " + s);
                q.addTask(s);
                try { Thread.sleep(100); } catch(InterruptedException e) {}
            }
        });
        add.start();
        add.join();
        Thread.sleep(100);
        for (var t : ts) {
            t.interrupt();
        }
    }
}

class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
        this.notifyAll();
    }

    public synchronized String getTask() throws InterruptedException {
        while (queue.isEmpty()) {
            this.wait();
        }
        return queue.remove();
    }
}

```
打印如下：
```shell
add task: t-0.9219689522548505
execute task: t-0.9219689522548505
add task: t-0.271813708432178
execute task: t-0.271813708432178
add task: t-0.8644114626575558
execute task: t-0.8644114626575558
add task: t-0.32891791164139694
execute task: t-0.32891791164139694
add task: t-0.011217614729680414
execute task: t-0.011217614729680414
add task: t-0.06744186559914589
execute task: t-0.06744186559914589
add task: t-0.5643259633238602
execute task: t-0.5643259633238602
add task: t-0.9453978156714784
execute task: t-0.9453978156714784
add task: t-0.6959170743585548
execute task: t-0.6959170743585548
add task: t-0.6532340394528874
execute task: t-0.6532340394528874

Process finished with exit code 0
```
但是，注意到`wait()`方法返回时需要重新获得`this`锁。假设当前有3个线程被唤醒，唤醒后，首先要等待执行`addTask()`的线程结束此方法后，才能释放`this`锁，随后，这3个线程中只能有一个获取到`this`锁，剩下两个将继续等待。

再注意到我们在`while()`循环中调用`wait()`，而不是`if`语句，这是因为线程被唤醒时，需要再次获取this锁。多个线程被唤醒后，只有一个线程能获取this锁，此刻，该线程执行queue.remove()可以获取到队列的元素，然而，剩下的线程如果获取this锁后执行queue.remove()，此刻队列可能已经没有任何元素了，所以，要始终在while循环中wait()，并且每次被唤醒后拿到this锁就必须再次判断：

```java
public synchronized String getTask() throws InterruptedException {
	while (queue.isEmpty()) {
		this.wait();
	}
	return queue.remove();
}
```

所以，正确编写多线程代码是非常困难的，需要仔细考虑的条件非常多，任何一个地方考虑不周，都会导致多线程运行时不正常。总结以下，`wait`和`notify`用于多线程协调运行：
- 在`synchronized`内部可以调用`wait()`使线程进入等待状态；
- 必须在已获得的锁对象上调用`wait()`方法；
- 在`synchronized`内部可以调用`notify()`或`notifyAll()`唤醒其他等待线程；
- 必须在已获得的锁对象上调用`notify()`或`notifyAll()`方法；
- 已唤醒的线程还需要重新获得锁后才能继续执行。