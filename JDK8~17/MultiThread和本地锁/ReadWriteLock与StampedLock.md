![[Pasted image 20231225161445.png]]
# ReadWriteLock

## 读读共享

大实际生活中多实际场景是“读/读”线程间并不存在互斥关系, 只有“读/写”线程或”写/写”线程间的操作需要互斥的。因此引入ReentrantReadWriteLock。

ReentrantReadWriteLock它允许读读共存，但是读写和写写依然是互斥的，即

- 一个资源可以被多个读操作访问
    
- 一个资源只能被一个写操作访问
    

在读多写少情景之下，读写锁具有较高的性能体现。比如下面的代码示例，读读不互斥，大家一起看电影

```Java
package com.bilibili.juc.rwlock;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class MyResource 
{
    //模拟一个简单的缓存
    Map<String,String> map = new HashMap<>();
    //=====ReentrantLock 等价于 =====synchronized，之前讲解过
    Lock lock = new ReentrantLock();
    //=====ReentrantReadWriteLock 一体两面，读写互斥，读读共享
    ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void write(String key ,String value)
    {
        rwLock.writeLock().lock();
        try
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"正在写入");
            map.put(key,value);
            //暂停毫秒
            try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"完成写入");
        }finally {
            rwLock.writeLock().unlock();
        }
    }

    public void read(String key)
    {
        rwLock.readLock().lock();
        try
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"正在读取");
            String result = map.get(key);
            //暂停200毫秒
            try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"完成读取"+"\t"+result);
        }finally {
            rwLock.readLock().unlock();
        }
    }


}


/**
 * @auther zzyy
 * @create 2022-04-08 18:18
 */
public class ReentrantReadWriteLockDemo
{
    public static void main(String[] args)
    {
        MyResource myResource = new MyResource();

        for (int i = 1; i <=10; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.write(finalI +"", finalI +"");
            },String.valueOf(i)).start();
        }

        for (int i = 1; i <=10; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.read(finalI +"");
            },String.valueOf(i)).start();
        }

    }
}
```

输出信息如下，写操作无法被打断：

- 写写不共存
    
- 读写不共存
    

但是读操作可以共同读

```Java
lock.ReadWriteLockDemo
5        正在写入
5        完成写入
2        正在写入
2        完成写入
6        正在写入
6        完成写入
4        正在写入
4        完成写入
1        正在写入
1        完成写入
7        正在写入
7        完成写入
3        正在写入
3        完成写入
9        正在写入
9        完成写入
8        正在写入
8        完成写入
10        正在写入
10        完成写入
1        正在读取
2        正在读取
3        正在读取
8        正在读取
10        正在读取
4        正在读取
5        正在读取
6        正在读取
7        正在读取
9        正在读取
2        完成读取        2
10        完成读取        10
7        完成读取        7
9        完成读取        9
4        完成读取        4
6        完成读取        6
1        完成读取        1
5        完成读取        5
3        完成读取        3
8        完成读取        8

Process finished with exit code 0
```

## 锁降级策略

`ReentrantReadWriteLock`的经典应用场景是锁降级策略，将写入锁降级为读锁。当同一个线程持有了写锁，在没有释放写锁的情况下，它还可以继续获得读锁。降级的规则顺序如下：

1. 先获取写锁
    
2. 然后获取读锁，
    
3. 最后再释放写锁。
    

如果释放了写锁，那么就完全转换为读锁。
![[Pasted image 20231225161509.png]]

下面这个代码示例就是一个完整的锁降级的过程。代码中声明了一个volatile类型的布尔值`cacheValid`变量，目的是保证数据可见性。

1. 在processCachedData 方法中，会首先获取到读锁，也就是rwl.readLock().lock()，它去判断当前的缓存是否有效，如果有效那么就直接跳过整个 if 语句，如果已经失效，代表我们需要更新这个缓存了。由于我们需要更新缓存，所以之前获取到的读锁是不够用的，我们需要获取写锁。
    
2. 在获取写锁之前，我们首先释放读锁，然后利用 rwl.writeLock().lock() 来获取到写锁。然后是经典的 try finally 语句，在 try 语句中我们==再次判断缓存是否有效==，因为在刚才释放读锁和获取写锁的过程中，可能有其他线程抢先修改了数据，所以在此我们需要进行二次判断。
    
3. 如果我们发现缓存是无效的，就用 new Object() 这样的方式来示意，获取到了新的数据内容，并把缓存的标记位设置为 ture，让缓存变得有效。由于我们后续希望打印出 data 的值，所以不能在此处释放掉所有的锁。
    
4. ==我们的选择是在不释放写锁的情况下直接获取读锁==，也就是rwl.readLock().lock() 这行语句所做的事情，然后，在持有读锁的情况下释放写锁，最后，在最下面的 try 中把 data 的值打印出来。
    

这就是一个非常典型的利用锁的降级功能的代码。

```Java
public class CachedData {

    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    void processCachedData() {
        rwl.readLock().lock();
        if (!cacheValid) {
            //在获取写锁之前，必须首先释放读锁。
            rwl.readLock().unlock();
            rwl.writeLock().lock();
            try {
                //这里需要再次判断数据的有效性,因为在我们释放读锁和获取写锁的空隙之内，可能有其他线程修改了数据。
                if (!cacheValid) {
                    data = new Object();
                    cacheValid = true;
                }
                //在不释放写锁的情况下，直接获取读锁，这就是读写锁的降级。
                rwl.readLock().lock();
            } finally {
                //释放了写锁，但是依然持有读锁
                rwl.writeLock().unlock();
            }
        }
        
        try {
            System.out.println(data);
        } finally {
            //释放读锁
            rwl.readLock().unlock();
        }
    }
}
```

问题来了，那我一直持有最高等级的写锁不就可以了吗？这样谁都没办法来影响到我自己的工作，永远是线程安全的。
- ==因为如果我们在刚才的方法中，一直使用写锁，最后才释放写锁的话，虽然确实是线程安全的，但是也是没有必要的，因为我们只有一处修改数据的代码==。
```Java
data = new Object();
```

- 而且==后面我们对于 data 仅仅是读取。如果还一直使用写锁的话，就不能让多个线程同时来读取了，持有写锁是浪费资源的，降低了整体的效率，所以这个时候利用锁的降级是很好的办法，可以提高整体性能。==

>ReadWriteLock支持锁的降级，不支持升级
>
>假设有 A，B 和 C 三个线程，它们都已持有读锁。假设线程 A 尝试从读锁升级到写锁。那么它必须等待 B 和 C 释放掉已经获取到的读锁。如果随着时间推移，B 和 C 逐渐释放了它们的读锁，此时线程 A 确实是可以成功升级并获取写锁。
>
>但是我们考虑一种特殊情况。假设线程 A 和 B 都想升级到写锁，那么对于线程 A 而言，它需要等待其他所有线程，包括线程 B 在内释放读锁。而线程 B 也需要等待所有的线程，包括线程 A 释放读锁。这就是一种非常典型的死锁的情况。谁都愿不愿意率先释放掉自己手中的锁。
>
>但是读写锁的升级并不是不可能的，也有可以实现的方案，如果我们保证每次只有一个线程可以升级，那么就可以保证线程安全。只不过最常见的 ReentrantReadWriteLock 对此并不支持。

## 写锁饥饿问题

读读共享是优点，但是与此同时也造成了写操作的饥饿现象

倘若读的时候别的线程还能修改的话，那读的数据就是脏数据了。因此读锁没有完成之前，写锁无法获得
![[Pasted image 20231225161555.png]]

```Java
package com.bilibili.juc.rwlock;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class MyResource //资源类，模拟一个简单的缓存
{
    Map<String,String> map = new HashMap<>();
    //=====ReentrantLock 等价于 =====synchronized，之前讲解过
    Lock lock = new ReentrantLock();
    //=====ReentrantReadWriteLock 一体两面，读写互斥，读读共享
    ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void write(String key ,String value)
    {
        rwLock.writeLock().lock();
        try
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"正在写入");
            map.put(key,value);
            //暂停毫秒
            try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"完成写入");
        }finally {
            rwLock.writeLock().unlock();
        }
    }

    public void read(String key)
    {
        rwLock.readLock().lock();
        try
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"正在读取");
            String result = map.get(key);
            //暂停200毫秒
            try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { e.printStackTrace(); }

            //暂停2000毫秒,演示读锁没有完成之前，写锁无法获得
            try { TimeUnit.MILLISECONDS.sleep(2000); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"完成读取"+"\t"+result);
        }finally {
            rwLock.readLock().unlock();
        }
    }


}


/**
 * @auther zzyy
 * @create 2022-04-08 18:18
 */
public class ReentrantReadWriteLockDemo
{
    public static void main(String[] args)
    {
        MyResource myResource = new MyResource();

        for (int i = 1; i <=10; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.write(finalI +"", finalI +"");
            },String.valueOf(i)).start();
        }

        for (int i = 1; i <=10; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.read(finalI +"");
            },String.valueOf(i)).start();
        }

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        for (int i = 1; i <=3; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.write(finalI +"", finalI +"");
            },"新写锁线程->"+String.valueOf(i)).start();
        }
    }
}
```

打印信息：

```Java
2        正在写入
2        完成写入
4        正在写入
4        完成写入
1        正在写入
1        完成写入
3        正在写入
3        完成写入
5        正在写入
5        完成写入
6        正在写入
6        完成写入
8        正在写入
8        完成写入
7        正在写入
7        完成写入
9        正在写入
9        完成写入
10        正在写入
10        完成写入
1        正在读取
2        正在读取
3        正在读取
4        正在读取
5        正在读取
6        正在读取
8        正在读取
7        正在读取
9        正在读取
10        正在读取
3        完成读取        3
8        完成读取        8
7        完成读取        7
2        完成读取        2
10        完成读取        10
5        完成读取        5
1        完成读取        1
4        完成读取        4
9        完成读取        9
6        完成读取        6
新写锁线程->1        正在写入
新写锁线程->1        完成写入
新写锁线程->2        正在写入
新写锁线程->2        完成写入
新写锁线程->3        正在写入
新写锁线程->3        完成写入

Process finished with exit code 0
```

# StampedLock

StampedLock是由读写锁写锁的饥饿问题引出的， ReentrantReadWriteLock实现了读写分离，但是一旦读操作比较多的时候，想要获取写锁就变得比较困难了，假如当前1000个线程，999个读，1个写，有可能999个读取线程长时间抢到了锁，那1个写线程就悲剧了，因为当前有可能会一直存在读锁，而无法获得写锁，根本没机会写,o(T_T)。尽管对`ReentrantReadWriteLock`使用“公平”策略可以一定程度上缓解这个问题，但是"公平"策略是以牺牲系统吞吐量为代价的

```Java
new ReentrantReadWriteLock(true)
```

StampedLock横空出世 ，StampedLock是JDK1.8中新增的一个读写锁, 也是对JDK1.5中的读写锁的优化
>- 所有获取锁的方法，都返回一个邮戳(Stamp)，Stamp为零表示获取失败，其余都表示成功；
>- 所有释放锁的方法，都需要一个邮戳(Stamp)，这个Stamp必须是和成功获取锁时得到的Stamp一致
>- StampedLock是不可重入的，没有Re开头。如果强行重入，会造成死锁。
>- StampedLock 的悲观读锁和写锁都不支持条件变量Condition，这个也需要注意。
>- 使用 StampedLock一定不要调用中断操作，即不要调用interrupt()方法

StampedLock有三种访问模式：
1. Reading（读模式悲观） : 功能和ReentrantReadWriteLock的读锁类似
2. Writing（写模式悲观） : 功能和ReentrantReadWriteLock的写锁类似
3. ==Optimistic reading(乐观读模式) : 无锁机制，获取读锁之后其他线程再尝试获取写锁时不会被阻塞，这其实是对读写锁的优化，很乐观认为读取时没人修改，假如被修改再实现升级为悲观读模式==

## 邮戳锁乐观读演示
1. 获取stamp ：`long stamp = stampedLock.tryOptimisticRead();`
2. 校验：`validate public boolean validate(long stamp)`
    - 如果自获得给定标记以来没有在写入模式下获取锁定，则方法validate(long)返回 true。
    - 如果标记代表当前持有的锁，则始终返回true。

读线程代码块后面暂停6秒钟线程，使得写线程不可介入

```Java
package com.bilibili.juc.rwlock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.StampedLock;

/**
 * @auther zzyy
 *
 * StampedLock = ReentrantReadWriteLock + 读的过程中也允许获取写锁介入
 */
public class StampedLockDemo
{
    static int number = 37;
    static StampedLock stampedLock = new StampedLock();

    public void write()
    {
        long stamp = stampedLock.writeLock();
        System.out.println(Thread.currentThread().getName()+"\t"+"写线程准备修改");
        try
        {
            number = number + 13;
        }finally {
            stampedLock.unlockWrite(stamp);
        }
        System.out.println(Thread.currentThread().getName()+"\t"+"写线程结束修改");
    }

    //乐观读，读的过程中也允许获取写锁介入
    public void tryOptimisticRead()
    {
        long stamp = stampedLock.tryOptimisticRead();
        int result = number;
        //故意间隔4秒钟，很乐观认为读取中没有其它线程修改过number值，具体靠判断
        System.out.println("4秒前stampedLock.validate方法值(true无修改，false有修改)"+"\t"+stampedLock.validate(stamp));
        for (int i = 0; i < 4; i++) {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"正在读取... "+i+" 秒" +
                    "后stampedLock.validate方法值(true无修改，false有修改)"+"\t"+stampedLock.validate(stamp));
        }
        if(!stampedLock.validate(stamp))
        {
            System.out.println("有人修改过------有写操作");
            stamp = stampedLock.readLock();//从乐观读 升级为 悲观读
            try
            {
                System.out.println("从乐观读 升级为 悲观读");
                result = number;
                System.out.println("重新悲观读后result："+result);
            }finally {
                stampedLock.unlockRead(stamp);
            }
        }
        System.out.println(Thread.currentThread().getName()+"\t"+" finally value: "+result);
    }


    public static void main(String[] args)
    {
        StampedLockDemo resource = new StampedLockDemo();

        new Thread(() -> {
            resource.tryOptimisticRead();
        },"readThread").start();


        //暂停6秒钟线程，使得写线程不可介入
        try { TimeUnit.SECONDS.sleep(6); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName()+"\t"+"----come in");
            resource.write();
        },"writeThread").start();
        
        try { TimeUnit.SECONDS.sleep(4); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println(Thread.currentThread().getName()+"\t"+"number:" +number);

    }
}
```

打印信息：

```Shell
4秒前stampedLock.validate方法值(true无修改，false有修改)        true
readThread        正在读取... 0 秒后stampedLock.validate方法值(true无修改，false有修改)        true
readThread        正在读取... 1 秒后stampedLock.validate方法值(true无修改，false有修改)        true
readThread        正在读取... 2 秒后stampedLock.validate方法值(true无修改，false有修改)        true
readThread        正在读取... 3 秒后stampedLock.validate方法值(true无修改，false有修改)        true
readThread         finally value: 37
writeThread        ----come in
writeThread        写线程准备修改
writeThread        写线程结束修改
main        number:50
    
Process finished with exit code 0
```
## 写线程接入，此后乐观读升级为悲观读

读线程代码块后面短暂的暂停2秒钟线程，使得写线程可介入

若`tryOptimisticRead`发现写线程介入了，则升级乐观读锁tryOptimisticRead为悲观读锁readLock

```Java
package com.bilibili.juc.rwlock;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.StampedLock;

/**
 * @auther zzyy
 *
 * StampedLock = ReentrantReadWriteLock + 读的过程中也允许获取写锁介入
 */
public class StampedLockDemo
{
    static int number = 37;
    static StampedLock stampedLock = new StampedLock();

    public void write()
    {
        long stamp = stampedLock.writeLock();
        System.out.println(Thread.currentThread().getName()+"\t"+"写线程准备修改");
        try
        {
            number = number + 13;
        }finally {
            stampedLock.unlockWrite(stamp);
        }
        System.out.println(Thread.currentThread().getName()+"\t"+"写线程结束修改");
    }



    //乐观读，读的过程中也允许获取写锁介入
    public void tryOptimisticRead()
    {
        long stamp = stampedLock.tryOptimisticRead();
        int result = number;
        //故意间隔4秒钟，很乐观认为读取中没有其它线程修改过number值，具体靠判断
        System.out.println("4秒前stampedLock.validate方法值(true无修改，false有修改)"+"\t"+stampedLock.validate(stamp));
        for (int i = 0; i < 4; i++) {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"正在读取... "+i+" 秒" +
                    "后stampedLock.validate方法值(true无修改，false有修改)"+"\t"+stampedLock.validate(stamp));
        }
        if(!stampedLock.validate(stamp))
        {
            System.out.println("有人修改过------有写操作");
            stamp = stampedLock.readLock();//从乐观读 升级为 悲观读
            try
            {
                System.out.println("从乐观读 升级为 悲观读");
                result = number;
                System.out.println("重新悲观读后result："+result);
            }finally {
                stampedLock.unlockRead(stamp);
            }
        }
        System.out.println(Thread.currentThread().getName()+"\t"+" finally value: "+result);
    }


    public static void main(String[] args)
    {
        StampedLockDemo resource = new StampedLockDemo();


        new Thread(() -> {
            resource.tryOptimisticRead();
        },"readThread").start();

        //暂停2秒钟线程,写线程可以介入，演示
        try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }


        new Thread(() -> {
            System.out.println(Thread.currentThread().getName()+"\t"+"----come in");
            resource.write();
        },"writeThread").start();


    }
}
```

打印信息：

```Shell

4秒前stampedLock.validate方法值(true无修改，false有修改)        true
readThread        正在读取... 0 秒后stampedLock.validate方法值(true无修改，false有修改)        true
writeThread        ----come in
writeThread        写线程准备修改
writeThread        写线程结束修改
readThread        正在读取... 1 秒后stampedLock.validate方法值(true无修改，false有修改)        false
readThread        正在读取... 2 秒后stampedLock.validate方法值(true无修改，false有修改)        false
readThread        正在读取... 3 秒后stampedLock.validate方法值(true无修改，false有修改)        false
有人修改过------有写操作
从乐观读 升级为 悲观读
重新悲观读后result：50
readThread         finally value: 50

Process finished with exit code 0
```

