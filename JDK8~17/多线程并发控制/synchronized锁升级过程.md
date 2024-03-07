在 Java 早期版本中，synchronized 属于重量级锁，效率低下。为什么呢？
![[Pasted image 20231225135351.png]]
Java基于监视器锁Monitor对象来实现重量级锁的，但是因为监视器锁是依赖于底层的操作系统的Mutex Lock来实现的。所以如果要挂起或者唤醒一个线程都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高。如果同步代码块中内容过于简单，这种切换的时间可能比用户代码执行的时间还长。

庆幸的是在 Java 6 之后 Java 官方对从 JVM 层面对 synchronized 较大优化，所以现在的 synchronized 锁效率也优化得很不错了。JDK1.6 对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、偏向锁、轻量级锁等技术来减少锁操作的开销。



对象头(Header)，对象头中包含三部分内容：
1. MarkWord 
2. 类型指针：虚拟机通过这个指针确定该对象是哪个类的实例。虚拟机通过这个指针来确定这个对象是哪个类的实例Customer cust = new Customer();
		![[Pasted image 20231225150724.png]]
3. 如果是数组对象的话，对象头还有一部分是存储数组的长度。

| 变量 | 说明 | 变量所占长度 |
| ---- | ---- | ---- |
| Mark Word | Mark Word 用于存储对象自身的运行时数据，如 HashCode、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID即JavaThread、偏向时间戳epoch等等。 | 32bit（8字节） |
| Class MetadataAddress | 存储到对象类型数据的指针 | 32bit（8字节） |
| Array length | 数组的长度 | 32bit（8字节） |
# MarkWord
多线程下 synchronized 的加锁就是对同一个对象的对象头中的 MarkWord 中的变量进行CAS操作。

在运行期间，MarkWord会根据对象的锁标志位复用自己的存储空间，Mark Word的内存布局如下：
- 无锁：MarkWord存储的是==unused未使用；==
- 偏向锁 : MarkWord存储的是==偏向的线程ID==，也即`JavaThread*`，另外Epoch表示偏向锁的时间戳
- 轻量锁 : MarkWord存储的是==指向线程栈中Lock Record的指针;==，Lock Record是当前对象MarkWord的一份拷贝
- 重量锁 : MarkWord存储的是==指向堆中的monitor对象的指针;==
![[Pasted image 20231225150557.png]]

例如GC年龄采用4个bit存储，最大为15，正好对应于MaxTenuringThreshold参数默认值就是15
所以假如我们手动设置最大分代年龄：-XX:MaxTenuringThreshold=16，JVM会报错，JVM启动失败
![[Pasted image 20231225150623.png]]
# 锁的升级过程四阶段

因为synchronized是通过进入和退出Monitor对象来实现代码块同步和方法同步的，而Monitor是依靠底层操作系统的`Mutex Lock`来实现的，属于重量级锁。操作系统实现线程之间的切换需要从用户态转换到内核态，这个切换成本比较高，对性能影响较大。

在JDK1.6中，为了减少获得锁和释放锁带来的性能消耗，引入了偏向锁和轻量级锁，锁的状态变成了四种，如下图所示。锁的状态会随着竞争激烈逐渐升级，但通常情况下，锁的状态只能升级不能降级。这种只能升级不能降级的策略是为了提高获得锁和释放锁的效率。
![[Pasted image 20231225161313.png]]
![[Pasted image 20231225151210.png]]
## 01无锁：unused未使用
## 01偏向锁【偏向线程ID】

Hotspot 的作者经过研究发现，大多数的多线程的情况下，锁不仅不存在多线程竞争，反倒是经常存在锁由同一个线程多次获得的情况，偏向锁就是在这种情况下出现的，它的出现是为了解决只有在一个线程执行同步时提高性能。
![[Pasted image 20231225151222.png]]

偏向锁的获取与竞争流程：

1. 检查对象头中Mark Word是否为可偏向状态，如果不是则直接升级为轻量级锁。
2. 如果是，判断Mark Work中的线程ID是否指向当前线程，如果是，则执行同步代码块。如果不是，则进行CAS操作竞争锁
3. 判断锁对象是否处于无锁状态，即原偏向锁的线程如果已经退出了STW临界区，表示同步代码已经执行完了。竞争锁的线程会进行CAS操作替代原来线程的ThreadID
4. 如果获得偏向锁的线程还处于临界区之内，表示同步代码还未执行完，将原偏向锁的线程升级为轻量级锁。

>全局安全点STW表示一种状态，该状态下所有线程都处于暂停状态。

==偏向锁和无锁的标志位都是01，这也就说明了偏向锁好似无锁一样高效==

偏向锁在JDK1.6之后是默认开启的，但是启动时间有4000ms延迟，执行Linux命令查看JVM启动参数：`java -XX:+PrintFlagsInitial | grep BiasedLock*`

```Shell
//关闭偏向锁延时，让其在程序启动时立刻启动
-XX:BiasedLockingStartupDelay=0，
//关闭偏向锁，关闭之后程序默认会直接进入轻量级锁状态
-XX:-UseBiasedLocking
```
![[Pasted image 20231225151446.png]]

## 00轻量级锁【获取方式为CAS+自旋】【MarkWord指向栈中Lock Record】

引入轻量级锁的目的：在多线程交替执行同步代码块时（未发生竞争），避免使用互斥量（重量锁）带来的性能消耗。
轻量级锁的获取与竞争流程如下：

1. 首先判断当前对象是否处于一个无锁的状态01，如果是，==Java虚拟机将在当前线程的栈帧建立一个锁记录（Lock Record），官方叫做Displaced Mark Word，用于存储对象目前的Mark Word的拷贝==，如图所示。
![[Pasted image 20231225151554.png]]

2. 线程获得了这个对象的锁
	- 将Lock Record中的owner指向当前对象Object，
	- 使用CAS操作将对象的Object对象头中的Mark Word更新为指向Lock Record的指针，
	- 修改Mark Word状态为00表示轻量级锁
	- 如果当前线程已经持有了当前对象的锁，这是一次重入，直接执行同步代码块。

3. 如果第二步未执行成功，则表示多个线程存在竞争，该==线程通过CAS自旋尝试获得锁，即重复步骤2，自旋超过一定次数，轻量级锁升级为重量级锁。==

## 10重量级锁【MarkWord指向堆中monitor】

- 偏向锁：适用于单线程适用的情况，在不存在锁竞争的时候进入同步方法/代码块则使用偏向锁。
- 轻量级锁：适用于竞争较不激烈的情况(这和乐观锁的使用范围类似)，存在竞争时升级为轻量级锁，轻量级锁采用的是自旋锁，如果同步方法/代码块执行时间很短的话，采用轻量级锁虽然会占用cpu资源但是相对比使用重量级锁还是更高效。
- 重量级锁：适用于竞争激烈的情况，如果同步方法/代码块执行时间很长，那么使用轻量级锁自旋带来的性能消耗就比使用重量级锁更严重，这时候就需要升级为重量级锁


总结下锁升级的口诀：

- 没有锁 : 自由自在
- 偏向锁 : 唯我独尊，多数情况(独得恩宠)
- 轻量锁 : 楚汉争霸，交替执行(自旋锁，避免线程上下文切换)
- 重量锁 : 群雄逐鹿，阻塞同步(传统阻塞锁池)

# 锁升级的过程中hashCode()去哪了

锁升级的过程中已经没有位置再保存哈希码了

1. 锁升级为偏向锁后，Mark Word中保存的是偏向线程ThreadID，和时间戳epoch
2. 锁升级为轻量级锁后，Mark Word中保存的是==指向线程栈帧里的锁记录指针==，指向Lock Record记录
3. 锁升级为重量级锁后，Mark Word中保存的是重量级锁指针，==指向了堆中的重量级锁的监视器`Object Monitor`==

那么哈希码信息被移动到哪里去了呢?

>用书中的一段话来描述锁和hashcode 之间的关系 ：
>在Java语言里面一个对象如果计算过哈希码，就应该一直保持该值不变，否则很多依赖对象哈希码的API都可能存在出错风险。

## 无锁状态计算hashcode，跳过偏向锁

当一个无锁对象已经计算过一致性哈希码后，它就再也无法进入偏向锁状态了，直接升级为轻量级锁。

升级为轻量级锁时，JVM会在当前线程的栈帧中创建一个锁记录(Lock Record)空间，用于存储锁对象的Mark Word拷贝，

- 该拷贝中包含identity hash code和GC年龄
    
- 释放锁后会将这些信息写回到堆中的对象头
    

```Java
package com.bilibili.juc.syncup;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

/**
 * @auther zzyy
 */
public class SynchronizedUpDemo
{
    public static void main(String[] args)
    {
        //先睡眠5秒，保证开启偏向锁
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }

        Object o = new Object();
        System.out.println("本应是偏向锁");
        System.out.println(ClassLayout.parseInstance(o).toPrintable());

        o.hashCode();//没有重写，一致性哈希，重写后无效,当一个对象已经计算过identity hash code，它就无法进入偏向锁状态；

        synchronized (o){
            System.out.println("本应是偏向锁，但是由于计算过一致性哈希，会直接升级为轻量级锁");
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
}
```

## 偏向锁状态计算hashcode，跳过轻量级锁

验证：而当一个对象当前正处于偏向锁状态，又收到需要计算其一致性哈希码请求时，它的偏向状态会被立即撤销,并且锁会膨胀为重量级锁。在重量级锁的实现中，对象头指向了重量级锁的监视器`Object Monitor`

- `Object Monitor`类里有字段可以记录无锁状态(标志为“01”)下的Mark Word，其中自然可以存储原来的哈希码，包括GC年龄
    
- 释放锁后也会将这些信息写回到对象头。
    

```Java
package com.bilibili.juc.syncup;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

/**
 * @auther zzyy
 */
public class SynchronizedUpDemo
{
    public static void main(String[] args)
    {

        //先睡眠5秒，保证开启偏向锁
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
        Object o = new Object();

        synchronized (o){
            o.hashCode();//没有重写，一致性哈希，重写后无效
            System.out.println("偏向锁过程中遇到一致性哈希计算请求，立马撤销偏向模式，膨胀为重量级锁");
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
}
```

# synchronized锁消除与锁粗化

JIT即时编译器会对加锁代码进行优化，如下：

1. 锁消除：多线程加锁加的不是同一把锁，相当于没加
    
2. 锁粗化：一个线程体内首尾相邻加了多次同一把锁，相当于加了一次锁
    

## 锁消除

```Java
package com.bilibili.juc.syncup;

/**
 * @auther zzyy
 * 锁消除：Object o是每次new出来的，都不一样
 * 从JIT角度看相当于无视它，synchronized (o)不存在了,
 */
public class LockClearUPDemo
{
    static Object objectLock = new Object();

    public void m1()
    {
        /*synchronized (objectLock){
            System.out.println("-----hello LockClearUPDemo");
        }*/
        //锁消除问题，JIT编译器会无视它 ，synchronized(o),每次new出来的，不存在了，非正常的。
        Object o = new Object();

        synchronized (o){
            System.out.println("-----hello LockClearUPDemo"+"\t"+o.hashCode()+"\t"+objectLock.hashCode());
        }
    }

    public static void main(String[] args)
    {
        LockClearUPDemo lockClearUPDemo = new LockClearUPDemo();

        for (int i = 1; i <=10; i++) {
            new Thread(() -> {
                lockClearUPDemo.m1();
            },String.valueOf(i)).start();
        }
    }
}
输出：
-----hello LockClearUPDemo        81436791        169692828
-----hello LockClearUPDemo        1650373908        169692828
-----hello LockClearUPDemo        40850379        169692828
-----hello LockClearUPDemo        1334955338        169692828
-----hello LockClearUPDemo        811147219        169692828
-----hello LockClearUPDemo        48799585        169692828
-----hello LockClearUPDemo        314312744        169692828
-----hello LockClearUPDemo        808072541        169692828
-----hello LockClearUPDemo        1787413096        169692828
-----hello LockClearUPDemo        1977837199        169692828

Process finished with exit code 0
```

## 锁粗化

```Java
package com.bilibili.juc.syncup;

/**
 * @auther zzyy
 * 锁粗化
 * 假如方法中首尾相接，前后相邻的都是同一个锁对象，那JIT编译器就会把这几个synchronized块合并成一个大块，
 * 加粗加大范围，一次申请锁使用即可，避免次次的申请和释放锁，提升了性能
 */
public class LockBigDemo
{
    static Object objectLock = new Object();

    public static void main(String[] args)
    {
        new Thread(() -> {
            synchronized (objectLock){
                System.out.println("111111");
            }
            synchronized (objectLock){
                System.out.println("222222");
            }
            synchronized (objectLock){
                System.out.println("333333");
            }
            synchronized (objectLock){
                System.out.println("444444");
            }

            synchronized (objectLock){
                System.out.println("111111");
                System.out.println("222222");
                System.out.println("333333");
                System.out.println("444444");
            }

        },"t1").start();
    }
}
```