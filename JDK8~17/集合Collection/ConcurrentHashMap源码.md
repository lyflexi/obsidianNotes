Java 中有 HashTable、Collections.synchronizedMap、以及 ConcurrentHashMap 可以实现线程安全的 Map。
- HashTable 是直接在操作方法上加 synchronized 关键字，锁住整个数组，粒度比较大
- Collections.synchronizedMap 是使用 Collections 集合工具的内部类，通过传入 Map 封装出一个 SynchronizedMap 对象，内部定义了一个对象锁，方法内通过对象锁实现；
- ConcurrentHashMap 使用分段锁，降低了锁粒度，让并发度大大提高。
三种Map的使用方式并没有区别：
```Java
package com.company.listIn_security;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

//HashMap不安全
//解决办法：
//Map<String, String> map = new ConcurrentHashMap<String, String>();
//Map<String, String> map = Collections.synchronizedMap(new HashMap<String, String>());
public class MapTest {
    public static void main(String[] args) {

        Map<String, String> map = new HashMap<String, String>();//报错：java.util.ConcurrentModificationException
        //Map<String, String> map = new ConcurrentHashMap<String, String>();
        //Map<String, String> map = Collections.synchronizedMap(new HashMap<String, String>());

        for (int i = 1; i <= 30; i++) {
            new Thread(()->{
                map.put(Thread.currentThread().getName(),
                        UUID.randomUUID().toString().substring(0,5));
                System.out.println(map);
            },String.valueOf(i)).start();
        }
    }
}


```
# HashTable悲观锁|全表锁

HashTable中采用了全表锁，即所有操作均上锁，串行执行，如下图中的put方法所示，采用synchronized关键字修饰。这样虽然保证了线程安全，但是在多核处理器时代也极大地影响了计算性能，这也致使HashTable逐渐淡出开发者们的视野。
![[Pasted image 20231225105201.png]]
# 1.7ConcurrentHashMap分段锁|segment

针对HashTable中锁粒度过粗的问题，在JDK1.8之前，ConcurrentHashMap引入了分段锁机制。整体的存储结构如下图所示，在原有结构的基础上拆分出多个segment，每个segment下再挂载原来的entrys（`HashMap`中经常提到的哈希桶数组），每次操作只需要锁定元素所在的segment，不需要锁定整个表。因此，锁定的范围更小，并发度也会得到提升。
![[Pasted image 20231225105210.png]]

数据结构：**Segment 数组 + HashEntry 数组 + 链表**

# 1.8ConcurrentHashMap
## 一、乐观锁CAS

虽然引入了分段锁的机制，即可以保证线程安全，又可以解决锁粒度过粗导致的性能低下问题，但是对于追求极致性能的工程师来说，这还不是性能的天花板。因此，在JDK1.8中，ConcurrentHashMap摒弃了分段锁，使用了乐观锁的实现方式。放弃分段锁的原因主要有以下几点：
- 使用segment之后，会增加ConcurrentHashMap的存储空间。
- 当单个segment过大时，并发性能会急剧下降。

**ConcurrentHashMap在JDK1.8中的实现废弃了之前的segment结构，沿用了与HashMap中的类似的Node数组结构。**
![[Pasted image 20231225105233.png]]

ConcurrentHashMap中的乐观锁是采用synchronized+CAS进行实现的。这里主要看下put的相关代码。

当put的元素在哈希桶数组中不存在时，则直接casTabAt进行写操作。
![[Pasted image 20231225105244.png]]

### tabAt计算出指定元素的起始内存地址

上面涉及到了两个重要的操作，`tabAt`与`casTabAt`。可以看出，这里面都使用了Unsafe类的方法。Unsafe这个类在日常的开发过程中比较罕见。我们通常对Java语言的认知是：Java语言是安全的，所有操作都基于JVM，在安全可控的范围内进行。然而，Unsafe这个类会打破这个边界，使Java拥有C的能力，可以操作任意内存地址，是一把双刃剑。这里使用到了前文中所提到的ASHIFT，来计算出指定元素的起始内存地址，再通过getObjectVolatile与compareAndSwapObject分别进行取值与CAS操作。

ConcurrentHashMap中的`((long)i << ASHIFT) + ABASE)`是用来计算哈希数组中某个元素在实际内存中的初始位置，通过((long)i << ASHIFT) + ABASE)进行计算，便可得到数组中第i个元素的起始内存地址。

```Java
    @SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```

位于ConcurrentHashMap源码的末尾处，声明了private static final sun.misc.Unsafe _U_

- `ABASE`由Unsafe类获取。ABASE = U.arrayBaseOffset(ak);
- `ASHIFT`由31 - Integer.numberOfLeadingZeros(scale);计算出来的，用总位数减去前导0的个数能够获取给定值的最高有效位数
    - ASHIFT采取的计算方式是31与scale前导0的数量做差
    - 其中`scale`就是哈希桶数组Node[]中每个元素的大小，由Unsafe类获取。int scale = U.arrayIndexScale(ak);
![[Pasted image 20231225105258.png]]

我们继续看下前导0的数量是怎么计算出来的，numberOfLeadingZeros是Integer的静态方法，还是沿用找规律的方式一探究竟。
![[Pasted image 20231225105317.png]]

其中n初始值为1

假设 i = 0000 0000 0000 0100 0000 0000 0000 0000

```Java
i >>> 16  0000 0000 0000 0000 0000 0000 0000 0100   不为0 
i >>> 24  0000 0000 0000 0000 0000 0000 0000 0000   等于0
```

右移了24位等于0，说明24位到31位之间肯定全为0，即n = 1 + 8 = 9，由于高8位全为0，并且已经将信息记录至n中，因此可以舍弃高8位，即 i <<= 8。此时，

```Java
i = 0000 0100 0000 0000 0000 0000 0000 0000
```

类似地，i >>> 28 也等于0，说明28位到31位全为0，n = 9 + 4 = 13，舍弃高4位。此时，

```Java
i = 0100 0000 0000 0000 0000 0000 0000 0000
```

继续运算，

```Java
i >>> 30  0000 0000 0000 0000 0000 0000 0000 0001   不为0 
i >>> 31  0000 0000 0000 0000 0000 0000 0000 0000   等于0
```

最终可得出n = 13，即有13个前导0。

`n -= i >>> 31`是检查最高位31位是否是1，因为n初始化值是1，如果最高位是1，则不存在前置0，即n = n - 1 = 0。

总结一下，以上的操作其实是基于二分法的思想来定位二进制中1的最高位，先看高16位，若为0，说明1存在于低16位；反之存在高16位。由此将搜索范围由32位（确切的说是31位）减少至16位，进而再一分为二，校验高8位与低8位，以此类推。

计算过程中校验的位数依次为16、8、4、2、1，加起来刚好为31。为什么是31不是32呢？因为前置0的数量为32的情况下i只能为0，在前面的if条件中已经进行过滤。

这样一来，非0值的情况下，前置0只能出现在高31位，因此只需要校验高31位即可。最终，用总位数减去计算出来的前导0的数量，即可得出二进制的最高有效位数。

代码中使用的是31 - Integer.numberOfLeadingZeros(scale)，而不是总位数32，这是为了能够得到哈希桶数组中第i个元素的起始内存地址，方便进行CAS等操作。

#### Unsafe方法.getObjectVolatile

在获取哈希桶数组中指定位置的元素时为什么不能直接get而是要使用getObjectVolatile呢？

虽然ConcurrentHashMap中的Node数组`table`是由volatile修饰的，可以保证可见性，但是Node数组中元素是不具备可见性的。因此，在获取数据时通过Unsafe的方法`getObjectVolatile`直接到主存中拿，保证获取的数据是最新的。
![[Pasted image 20231225105331.png]]

>在JVM的内存模型中，每个线程有自己的工作内存，也就是栈中的局部变量表，它是主存的一份copy。因此，线程1对某个共享资源进行了更新操作，并写入到主存，而线程2的工作内存之中可能还是旧值，脏数据便产生了。Java中的volatile是用来解决上述问题，保证可见性，任意线程对volatile关键字修饰的变量进行更新时，会使其它线程中该变量的副本失效，需要从主存中获取最新值。

### casTabAt进行并发写操作

#### Unsafe方法.compareAndSetReference

## 二、扩容操作helpTransfer
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {  
    if (key == null || value == null) throw new NullPointerException();  
    int hash = spread(key.hashCode());  
    int binCount = 0;  
    for (Node<K,V>[] tab = table;;) {  
        Node<K,V> f; int n, i, fh;  
        if (tab == null || (n = tab.length) == 0)  
            tab = initTable();  
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  
            if (casTabAt(tab, i, null,  
                         new Node<K,V>(hash, key, value, null)))  
                break;                   // no lock when adding to empty bin  
        }  
        else if ((fh = f.hash) == MOVED)  
            tab = helpTransfer(tab, f);
        else {...}
```

## 三、Synchronized+DoubleCheck思想

继续往下看put方法的逻辑：

1. 当put的元素在哈希桶数组中存在，并且不处于扩容状态时，则使用synchronized锁定哈希桶数组中第i个位置中的第一个元素f（头节点）
2. 接着进行DCL(double check lock)，类似于线程安全单例模式的思想。防止当前线程获取到锁之后进行扩容操作改变了元素的位置
3. 校验通过后，会遍历当前冲突链上的元素，并选择合适的位置进行put操作。

![[Pasted image 20231225105403.png]]

最后总结下ConcurrentHashMap：
- **悲观->乐观锁：这里只有在发生哈希冲突的情况下才使用synchronized锁定头节点
- **粗粒度->细粒度锁：只在特定场景下锁定其中一个哈希桶，这是比分段锁更细粒度的锁实现，降低锁了的影响范围。
- **数据结构和HashMap一样，依然是Node 数组 + 链表 / 红黑树**。当冲突链表达到一定长度8，且数组长度大于64的时候，链表会转换成红黑树。

Java Map针对并发场景解决方案的演进方向可以归结为，从悲观锁到乐观锁，从粗粒度锁到细粒度锁，这也可以作为我们在日常并发编程中的指导方针。