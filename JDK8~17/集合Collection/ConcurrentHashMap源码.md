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
![[Pasted image 20240322180245.png]]

数据结构：**Segment 数组 + HashEntry 数组 + 链表**

# 1.8ConcurrentHashMap
在JDK1.8中，ConcurrentHashMap摒弃了分段锁，将锁的级别控制在了更细粒度的哈希桶数组元素级别，也就是说只需要锁住这个链表头节点（红黑树的根节点），就不会影响其他的哈希桶数组元素的读写，大大提高了并发度。
![[Pasted image 20240322180315.png]]
同时使用了乐观锁cas+悲观锁synchronized共存的实现方式
- 不存在冲突时，乐观锁cas
- 存在冲突时，解决冲突使用悲观锁DCL

## volatile
### volatile数组，扩容可见
哈希桶数组table用 volatile 修饰主要是保证在数组扩容的时候保证可见性。
![[Pasted image 20240322194411.png]]
注意：get 方法不需要加锁与 volatile 修饰的哈希桶数组table无关
### volatile节点，get可见
Node 的元素 value 和指针 next 使用 volatile 修饰，为了保证在多线程环境下线程A修改节点的 value 或者新增节点的时候是对线程B可见的。
![[Pasted image 20240322194358.png]]
这也是get 方法不需要加synchronized锁的原因，因为共享数据已经被volatile 修饰为可见了
## put方法逻辑
putVal大致可以分为以下步骤：
1. 根据 key 计算出 hash 值，根据`(n - 1) & hash` 确定元素存放在哪个桶中
2. 判断是否需要进行初始化，依旧是懒加载机制
3. 定位到 Node，拿到首节点 f，判断首节点f：
	- 如果为  null  ，则通过 CAS 无锁的方式尝试添加；tabAt+casTabAt
	- 如果为 `f.hash = MOVED = -1` ，说明其他线程在扩容，参与一起扩容helpTransfer
	- 如果都不满足 ，synchronized 锁住 f 节点，判断是链表还是红黑树，遍历插入；
源码如下：
![[Pasted image 20240322193238.png]]
### 不存在冲突则直接cas
当发现头节点为null时，直接casTabAt进行写操作
![[Pasted image 20231225105244.png]]
上面涉及到了两个重要的操作，tabAt与casTabAt
tabAt对应于UnSafe.getObjectVolatile
```Java
@SuppressWarnings("unchecked")  
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {  
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);  
}  
```
casTabAt对应于UnSafe.compareAndSwapObject
```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,  
                                    Node<K,V> c, Node<K,V> v) {  
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);  
}
```

### 存在冲突则上DCL锁
继续往下看put方法的逻辑，看到这里只有在发生哈希冲突的情况下才使用synchronized锁定头节点
1. 当put的元素在哈希桶数组中存在，并且不处于扩容状态时，则使用synchronized锁定哈希桶数组中第i个位置中的第一个元素f（头节点）
2. 接着进行DCL(double check lock)，类似于线程安全单例模式的思想。防止当前线程获取到锁之后进行扩容操作改变了元素的位置
3. 校验通过后，会遍历当前冲突链上的元素，并选择合适的位置进行put操作。
![[Pasted image 20231225105403.png]]
## 不支持key和value为null的原因？★★★
==我们知道用于单线程状态的 HashMap允许多个null值和一个null键==
- key可以为null，hash函数中明确处理了key为null的情况，但是由于key不能重复null键只可能存在1个
- value可以为null，可以用`containsKey(key)` 去判断到底是否包含了这个 null 。
Doug Lea和Effective Java作者Josh Bloch共同参与了HashMap的设计，Josh Bloch允许key和value为null，
![[Pasted image 20240322200444.png]]
==但是ConcurrentHashMap的key和value均不能为null：==
- 我们先来说value 为什么不能为 null。因为 ConcurrentHashMap 是用于多线程的 ，如果`ConcurrentHashMap.get(key)`得到了 null ，这就无法判断，是映射的value是 null ，还是没有找到对应的key而为 null ，就有了二义性。
- key也不可以为null，这个问题讨论的价值不大，因为ConcurrentHashMap的源码就是这样写的，Doug Lea在设计之初就不允许null的key存在。
![[Pasted image 20240322200414.png]]
ConcurrentHashMap的作者只有Doug Lea，Doug Lea十分讨厌null，也确实nullkey和nullvalue是一种很不合理的设计，因此ConcurrentHashMap的key和value均不能为null，传送门->[这道面试题我真不知道面试官想要的回答是什么 (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzg3NjU3NTkwMQ==&mid=2247505071&idx=1&sn=5b9bbe01a71cbfae4d277dd21afd6714&source=41#wechat_redirect)
## 并发扩容helpTransfer？待研究
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