在 Java 早期版本中，synchronized 属于重量级锁，效率低下。为什么呢？
![[Pasted image 20231225135351.png]]
Java基于监视器锁Monitor对象来实现重量级锁的，但是因为监视器锁是依赖于底层的操作系统的Mutex Lock来实现的。所以如果要挂起或者唤醒一个线程都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高。如果同步代码块中内容过于简单，这种切换的时间可能比用户代码执行的时间还长。

庆幸的是在 Java 6 之后 Java 官方对从 JVM 层面对 synchronized 较大优化，所以现在的 synchronized 锁效率也优化得很不错了。JDK1.6 对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、偏向锁、轻量级锁等技术来减少锁操作的开销。



对象头(Header)，对象头中包含三部分内容：
1. MarkWord ，又叫运行时元数据，用于存储对象自身的运行时数据，如 HashCode、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID即JavaThread、偏向时间戳epoch等等。
2. 类型指针：虚拟机通过这个指针确定该对象是哪个类的实例。虚拟机通过这个指针来确定这个对象是哪个类的实例Customer cust = new Customer();
3. 如果是数组对象的话，对象头还有一部分是存储数组的长度。
		![[Pasted image 20231225150724.png]]

变量所占长度说明如下表：

| 变量                   | 变量所占长度     |
| -------------------- | ---------- |
| Mark Word            | 32bit（8字节） |
| ClassMetadataAddress | 32bit（8字节） |
| Array length         | 32bit（8字节） |
# MarkWord
多线程下 synchronized 的加锁就是对同一个对象的对象头中的 MarkWord 中的变量进行CAS操作。

在运行期间，MarkWord会根据对象的锁标志位复用自己的存储空间，Mark Word的内存布局如下，无锁：MarkWord存储的是25位unused和31位hashcode，计算完一次hashcode之后的打印信息如下，前8个字节都是markword：
```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 66 01 d1 (00000001 01100110 00000001 11010001) (-788437503)
      4     4        (object header)                           17 00 00 00 (00010111 00000000 00000000 00000000) (23)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
计算完一次hashcode之后，hashcode从markword中移除
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
## 001无锁：unused+hashcode
## 101偏向锁【偏向线程ID】

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

## 00轻量级锁【MarkWord指向栈中Lock Record】

引入轻量级锁的目的：在多线程交替执行同步代码块时（未发生竞争），避免使用互斥量Monitor（重量锁）带来的性能消耗。

轻量级锁的执行过程：在抢锁线程进入临界区之前，如果内置锁（临界区的同步对象）没有被锁定，JVM首先将在抢锁线程的栈帧中建立一个锁记录（Displaced Mark Word），用于存储对象目前Mark Word的拷贝，然后抢锁线程将使用CAS自旋操作，尝试将内置锁对象头的Mark Word的ptr_to_lock_record（锁记录指针）更新为抢锁线程栈帧中锁记录的地址
![[Pasted image 20240313103406.png]]
如果抢锁成功，线程就拥有了这个对象锁。然后JVM将Mark Word中的lock标记位改为00（轻量级锁标志），即表示该对象处于轻量级锁状态。JVM会将Mark Word中原来的锁对象信息（如哈希码等）保存在抢锁线程锁记录的Displaced Mark Word（可以理解为放错地方的Mark Word）字段中，再将抢锁线程中锁记录的owner指针指向锁对象
![[Pasted image 20240313103516.png]]

为什么复制对象头的部分信息到线程堆栈中的锁记录的Displaced Mark Word字段？
因为内置锁对象的Mark Word的结构会有所变化，Mark Word将会出现一个指向锁记录的指针，而不再存着无锁状态下的锁对象哈希码等信息，所以必须将这些信息暂存起来，供后面在锁释放时使用。

正是因为Synchronized的优化，此时Synchronized依然没有使用到Monitor，只有重量级Synchronized才会使用到更为底层的Monitor

如果争抢轻量级锁失败，则表示多个线程存在竞争，该线程通过自旋去不断的尝试CAS获得锁，自旋超过一定次数，轻量级锁升级为重量级锁。


## 10重量级锁【MarkWord指向底层的Monitor】
精妙的图示：
- obj是对象实例位于堆空间，虽然是共享的，但线程们必须操作同一把锁才有效
- 不同的Thread内部持有的目标方法栈帧位于栈空间，线程私有
- Monitor位于操作系统底层
![[Pasted image 20240312161518.png]]
- 刚开始Monitor中Owner为null，锁对象只有1把obj位于堆空间
- 当Thread-2执行synchronized(obj)，会将对象头中的MarkWord与底层Monitor相关联，同将Monitor的所有者Owner置为Thread-2，Monitor中只能有一个Owner
- 在Thread-2上锁的过程中，如果Thread-1,Thread-3也来执行synchronized(obj),就会进入EntryList BLOCKED
- Thread-2执行完同步代码块的内容，然后唤醒EntryList中等待的线程来竞争锁，竞争的时是非公平的
- 图中WaitSet是之前获得过锁，但条件不满足进入WAITING状态的线程，与wait-notify有关，涉及到线程间通信
# 编写测试用例总结
## 不计算hashcode，默认可偏向
//大端模式，是指数据的高字节保存在内存的低地址中，而数据的低字节保存在内存的高地址中，这样的存储模式有点儿类似于把数据当作字符串顺序处理：地址由小向大增加，而数据从高位往低位放；这和我们的阅读习惯一致。  
//小端模式-默认，是指数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中，这种存储模式将地址的高低和数据位权有效地结合起来，高地址部分权值高，低地址部分权值低。
```java
package org.lyflexi.synchronize.upgrade;  
  
import org.openjdk.jol.info.ClassLayout;  
  
import java.util.concurrent.TimeUnit;  
  
  
public class A_Biasable  
{  
    public static void main(String[] args)  
    {  
        //先睡眠5秒，保证开启偏向锁  
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }  
        Object o = new Object();    
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
        o.hashCode();  
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
    }  
}
```
打印信息：
第一组的锁标志位是101表示无锁状态下的可偏向
第二组计算了hashcode，之后锁标志位是001表示无锁状态下的不可偏向
```shell
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 66 01 d1 (00000001 01100110 00000001 11010001) (-788437503)
      4     4        (object header)                           17 00 00 00 (00010111 00000000 00000000 00000000) (23)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


Process finished with exit code 0
```
## 计算了hashcode，则不可偏向，升级为LightWeight

当一个无锁对象已经计算过一致性哈希码后，它就再也无法进入偏向锁状态了，直接升级为轻量级锁。

升级为轻量级锁时，JVM会在当前线程的栈帧中创建一个Mark Word拷贝叫做Lock Record，该拷贝中包含identity hash code和GC年龄

```Java
public class B_LightWeight  
{  
    public static void main(String[] args)  
    {  
        //先睡眠5秒，保证开启偏向锁  
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }  
        Object o = new Object();  
  
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
        o.hashCode();  
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
        synchronized (o){  
            System.out.println(ClassLayout.parseInstance(o).toPrintable());  
        }  
  
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
    }  
}
```
打印信息如下，第三组打印信息的最后一个字节代表升级为了轻量级锁11101000，synchronized不支持锁的降级，最后虽然回归无锁状态，但是最后一组打印信息00000001表示锁已经不可偏向了
```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 66 01 d1 (00000001 01100110 00000001 11010001) (-788437503)
      4     4        (object header)                           17 00 00 00 (00010111 00000000 00000000 00000000) (23)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           e8 f4 ff 22 (11101000 11110100 11111111 00100010) (587199720)
      4     4        (object header)                           e4 00 00 00 (11100100 00000000 00000000 00000000) (228)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 66 01 d1 (00000001 01100110 00000001 11010001) (-788437503)
      4     4        (object header)                           17 00 00 00 (00010111 00000000 00000000 00000000) (23)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
对于偏向锁，在线程获取偏向锁时，会用Thread ID和epoch值覆盖identity hash code所在的位置。如果一个对象的hashCode()方法已经被调用过一次之后，这个对象还能被设置偏向锁么？答案是不能。因为如果可以的化，那Mark Word中的identity hash code必然会被偏向线程Id给覆盖，这就会造成同一个对象前后两次调用hashCode()方法得到的结果不一致。
## 同步块内计算hashcode，则不可偏向，越级为heavyWeight

当一个对象当前正处于偏向锁状态，又收到需要计算其一致性哈希码请求时，这时候会直接越级至重量级锁。在重量级锁的实现中，对象头指向了重量级锁的监视器`Object Monitor`，`Object Monitor`类里有字段可以记录无锁状态(标志为“01”)下的Mark Word，其中自然可以存储原来的哈希码，包括GC年龄
```Java
public class C_HeavyWeight  
{  
    public static void main(String[] args)  
    {  
        //先睡眠5秒，保证开启偏向锁  
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }  
        Object o = new Object();  
  
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
  
        synchronized (o){  
            System.out.println(ClassLayout.parseInstance(o).toPrintable());  
            o.hashCode();  
            System.out.println(ClassLayout.parseInstance(o).toPrintable());  
        }  
  
        System.out.println(ClassLayout.parseInstance(o).toPrintable());  
    }  
}
```
打印信息如下：第三组打印信息11101010表示从偏向锁越级为了重量级锁，第四组打印信息表示仍然是重量级锁11101010不可降级
```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 48 c2 2d (00000101 01001000 11000010 00101101) (767707141)
      4     4        (object header)                           46 02 00 00 (01000110 00000010 00000000 00000000) (582)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           ea 01 db 51 (11101010 00000001 11011011 01010001) (1373307370)
      4     4        (object header)                           46 02 00 00 (01000110 00000010 00000000 00000000) (582)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           ea 01 db 51 (11101010 00000001 11011011 01010001) (1373307370)
      4     4        (object header)                           46 02 00 00 (01000110 00000010 00000000 00000000) (582)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


Process finished with exit code 0
```


总结一下
- 没有锁 : 自由自在
- 偏向锁：唯我独尊，多数情况(独得恩宠)。适用于单线程适用的情况，在不存在锁竞争的时候进入同步方法/代码块则使用偏向锁。
- 轻量级锁：楚汉争霸，交替执行(自旋锁，避免线程上下文切换)。偏向锁存在竞争时升级为轻量级锁，轻量级锁采用的是自旋锁适用于竞争较不激烈的情况，如果同步代码块执行时间很短的话，采用轻量级锁虽然会占用cpu资源但是相对比使用重量级锁还是更高效。
- 重量级锁：群雄逐鹿，阻塞同步(传统阻塞锁池Monitor_Entrylist)。如果同步代码块执行时间很长或者竞争特别剧烈了，那么使用轻量级锁自旋带来的性能消耗就比使用重量级锁更严重，这时候就需要升级为重量级锁







# 锁升级的过程中hashCode()去哪了

锁升级的过程中，markword中已经没有位置再保存哈希码了，那么哈希码信息被移动到哪里去了呢？在Java语言里面一个对象如果计算过哈希码，就应该一直保持该值不变，否则很多依赖对象哈希码的API都可能存在出错风险。

1. 锁升级为偏向锁后，Mark Word中保存的是偏向线程ThreadID，和时间戳epoch，网上也没有找到偏向锁状态下hashcode的存储位置究竟在哪里
2. 锁升级为轻量级锁后，Mark Word中保存的是指向线程栈帧里的锁记录指针，指向Lock Record记录，Lock Record记录里面保存了hashcode
3. 锁升级为重量级锁后，Mark Word中保存的是重量级锁指针，指向了堆中的重量级锁的监视器`Object Monitor`，`Object Monitor`里面保存了hashcode
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