# 数据结构

JDK1.8 之前 HashMap 由 数组+链表 组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。
JDK1.8 之后 HashMap 的组成多了红黑树，在满足下面两个条件之后，会执行链表转红黑树操作，以此来加快搜索速度。
>这里简单地回顾一下红黑树，它是一种平衡的二叉树搜索树，类似地还有AVL树。两者核心的区别是AVL树追求“绝对平衡”，在插入、删除节点时，成本要高于红黑树，但也因此拥有了更好的查询性能，适用于读多写少的场景。然而，对于HashMap而言，读写操作其实难分伯仲，因此选择红黑树也算是在读写性能上的一种折中。
>
>红黑树的“平衡”的意思并不是说两个子树的叶子节点个数精确一样，而是说两个子树的高度不会差太多（对于AVL树，任何一个节点的两个子树高度差不会超过1；对于红黑树，则是不会相差两倍以上），从而在这样的树中进行搜索的话即便在最坏情况下也会很高效，这就足够了。

红黑树的生成条件，注意：下面两个条件存在先后顺序：
1. 随着`putVal`的添加，当链表长度大于阈值时（阈值默认为 8），会首先调用`treeifyBin`方法
2. `treeifyBin`方法当中会判断HashMap 数组长度是否超过 64，若超过64则会执行链表转红黑树操作以减少搜索时间，否则只是简单执行`resize`方法对数组进行扩容。
![[Pasted image 20231225103846.png]]

![[Pasted image 20231225103853.png]]

![[Pasted image 20231225103922.png]]

那么为什么要引入红黑树来替代链表呢？虽然链表的插入性能是O(1)，但查询性能确是O(n)，当哈希冲突元素非常多时，这种查询性能是难以接受的。因此，在JDK1.8中，如果冲突链上的元素数量大于8，并且哈希桶数组的长度大于64时，会使用红黑树代替链表来解决哈希冲突，使查询具备O(logn)的性能。此时的节点会被封装成TreeNode而不再是Node。

声明：贯穿下文
- Node就是Map数组节点，俗称桶
- Node数组就是`Node<K,V>[] tab`，俗称桶数组，每个桶可能转换为链表Or红黑树
- 下文我们根据JDK1.8源码去剖析HashMap，我们从扰动函数入手。

# 全局变量

`HashMap`定义的全局变量如下：

```Java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    // 序列号
    private static final long serialVersionUID = 362498820763181265L;
    // 默认的初始容量是16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;//16
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认的填充因子
    static final float DEFAULT_INITIAL_CAPACITY = 0.75f;
    // 当桶(bucket)上的结点数大于这个值时会转成红黑树
    static final int TREEIFY_THRESHOLD = 8;
    // 当桶(bucket)上的结点数小于这个值时树转链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 桶中结构转化为红黑树对应的table的最小大小
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 存储元素的数组，总是2的幂次倍
    transient Node<k,v>[] table;
    // 存放具体元素的集
    transient Set<map.entry<k,v>> entrySet;
    // 存放元素的个数，注意这个不等于数组的长度。
    transient int size;
    // 每次扩容和更改map结构的计数器
    transient int modCount;
    // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;
    // 加载因子
    final float loadFactor;
}
```

默认`DEFAULT_INITIAL_CAPACITY`和`DEFAULT_LOAD_FACTOR` ：
给定的默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 16 * 0.75 = 12 就需要将当前 16 的容量进行扩容，而扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能。

成员变量`threshold`是数组扩容阈值，`threshold = capacity * loadFactor`，当 Size>=threshold的时候，那么就要考虑对数组的扩增了。 但是，源码当中注释中强调了
![[Pasted image 20231225103950.png]]

翻译过来是：`threshold`在数组还没有被分配的情况下还可以充当初始数组容量，特别的`threshold=0`的时候代表 `DEFAULT_INITIAL_CAPACITY`（16）

# 懒初始化

HashMap 中有四个构造方法，它们分别如下：

```Java
    // 默认构造函数。
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all   other fields defaulted
     }

     // 包含另一个“Map”的构造函数
     public HashMap(Map<? extends K, ? extends V> m) {
         this.loadFactor = DEFAULT_LOAD_FACTOR;
         putMapEntries(m, false);
     }

     // 指定“容量大小”的构造函数
     public HashMap(int initialCapacity) {
         this(initialCapacity, DEFAULT_LOAD_FACTOR);
     }

     // 指定“容量大小”和“加载因子”的构造函数
     public HashMap(int initialCapacity, float loadFactor) {
         if (initialCapacity < 0)
             throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
         if (initialCapacity > MAXIMUM_CAPACITY)
             initialCapacity = MAXIMUM_CAPACITY;
         if (loadFactor <= 0 || Float.isNaN(loadFactor))
             throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
         this.loadFactor = loadFactor;
         this.threshold = tableSizeFor(initialCapacity);
     }
```

## HashMap()默认构造

以默认构造函数为例，通过new创建出HashMap的时候只会设置默认负载因子，并不会对其进行初始化操作，只有在首次使用put添加元素的时候才会进行初始化（resize方法第一次调用）。
![[Pasted image 20231225104045.png]]

resize()中会设置默认的初始化容量DEFAULT_INITIAL_CAPACITY为16，扩容的阈值为`0.75x16 = 12`，最后创建容量为16的`Node`数组，并赋值给成员变量哈希桶`table`，即完成了HashMap的初始化操作
![[Pasted image 20231225104054.png]]

## 自定义容量构造HashMap(int initialCapacity)

### tableSizeFor

当用户使用其余三种构造方法创建HashMap的时候，用户传入自定义容量`initialCapacity`，但JDK并不使用用户传入的`initialCapacity`，==而是触发`tableSizeFor(initialCapacity)`方法将输入的cap的进行加工==，转换为“大于等于给定值cap的最小2的整数次幂”作为最终容量。

`static final int tableSizeFor(int cap) {...}`返回“大于等于给定值cap的最小2的整数次幂”

提示：==无符号右移`>>>`的优先级是最高的==

>`>>`表示右移，移出的部分将被抛弃。如果该数为正，则高位补0，若为负数，则高位补1。如0000 1111(15)右移2位的结果是0000 0011(3)，又或者0001 1010(18)右移3位的结果是0000 0011(3)。
>`>>>`表示无符号右移，也叫逻辑右移，即若该数为正，则高位补0，而若该数为负数，则右移后高位同样补0。

tableSizeFor根据输入容量大小cap来计算最终哈希桶数组的容量大小，计算的意义找到“大于等于给定值cap的最小2的整数次幂”。整个过程是：

1. 找到cap对应二进制中最高位的1，
2. 然后每次以2倍的步长（依次移位1、2、4、8、16）复制最高位1到后面的所有低位，把最高位1后面的所有位全部置为1，
3. 最后进行+1，即完成了进位
![[Pasted image 20231225104730.png]]

当cap为3时，则n=2，计算过程如下：

```Java
n = 2
n |= n >>> 1       010  | 001 = 011   n = 3
n |= n >>> 2       011  | 000 = 011   n = 3
n |= n >>> 4       011  | 000 = 011   n = 3
….
n = n + 1 = 4
```

当cap为5时，则n=4，计算过程如下：

```Java
n = 4
n |= n >>> 1    0100 | 0010 = 0110  n = 6
n |= n >>> 2    0110 | 0001 = 0111  n = 7
….
n = n + 1 = 8
```

JDK虽然要求返回“大于等于给定值cap的最小2的整数次幂”，但为什么必须要2的整数次幂呢？

答案是，为了提高计算与存储效率，使每个元素对应hash值能够准确落入哈希桶数组给定的范围区间内。

==确定数组下标采用的算法是 hash & (n - 1)，n即为哈希桶数组的大小==。由于其总是2的整数次幂，这意味着n-1的二进制形式永远都是0000111111的形式，即从最低位开始，连续出现多个1，该二进制与任何值进行&运算都会使该值映射到指定区间[0, n-1]。
- 比如：当n=8时，n-1对应的二进制为0111，任何与0111进行&运算都会落入[0,7]的范围内，即落入给定的8个哈希桶中，存储空间利用率100%。
- 举个反例，当n=7，n-1对应的二进制为0110，任何与0110进行&运算会落入到第0、6、4、2个哈希桶，而不是[0,6]的区间范围内，少了1、3、5三个哈希桶，这导致存储空间利用率只有不到60%，同时也增加了哈希碰撞的几率。

# hash方法（扰动函数）

所谓扰动函数指的就是 HashMap 的 hash 方法。hash方法是对key 的 hashCode值进行再加工，因此 hash 方法是就为了防止一些实现比较差的 hashCode() 方法 ，换句话说使用扰动函数之后可以减少碰撞。

>hash 方法返回hash值，然后通过 `(n - 1) & hash` 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

JDK 1.8 HashMap 的 hash 方法源码：

从这里也可以看出来，HashMap的key是可以为null的，key为null时扰动函数`return 0`;

```Java
    static final int hash(Object key) {
      int h;
      // key.hashCode()：返回散列值也就是hashcode
      // ^ ：按位异或
      // >>>:无符号右移，忽略符号位，空位都以0补齐
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```

对比一下 JDK1.7 的 HashMap 的 hash 方法源码。相比于 JDK1.8 的 hash 方法 ，JDK 1.7 的 hash 方法的性能会稍差一点点，因为毕竟扰动了 4 次。

```Java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

# 插入过程

`putVal`方法，根据`(n - 1) & hash` 确定元素存放在哪个桶中，

```Java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 桶中已经存在元素
    else {
        Node<K,V> e; K k;
        // 比较桶中第一个元素(数组中的结点)的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
                // 将第一个元素赋值给e，用e来记录
                e = p;
        // hash值不相等，即key不相等；为红黑树结点
        else if (p instanceof TreeNode)
            // 放入树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 为链表结点
        else {
            // 在链表最末插入结点
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    // 在尾部插入新结点
                    p.next = newNode(hash, key, value, null);
                    // 结点数量达到阈值(默认为 8 )，执行 treeifyBin 方法
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    // 跳出循环
                    break;
                }
                // 判断链表中结点的key值与插入的元素的key值是否相等
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
        // 覆盖操作在此：表示在桶中找到key值、hash值与插入元素相等的结点
        if (e != null) {
            // 记录e的value
            V oldValue = e.value;
            // onlyIfAbsent为false或者旧值为null
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 结构性修改
    ++modCount;
    // 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
}
```

**对 putVal 方法添加元素的分析如下：**
![[Pasted image 20231225104924.png]]

关于HashMap链表插入问题，java7及之前之前是头插法，当时写这个代码的作者认为后来的值被查找的可能性更大一点，提升查找的效率。但是java7及之前HashMap的在并发扩容的时候会导致链表成环的问题。所以在执行get的时候，会触发死循环，引起CPU的100%问题，所以一定要避免在并发环境下使用HashMap。

HashMap的死链问题：

头插法在扩容时会改变链表中元素原本的顺序，线程T1执行之后，链表中的节点顺序发生了改变。但线程T2对于发生的一切还是不可知的，所以它指向的节点引用依然没变。如图所示，T2指向的是A节点，T2.next指向的是B节点。
![[Pasted image 20231225104933.png]]

==因此在jdk8及之后采用尾插入法并且引入了高低位链表==，扩容时会保持链表元素原本的顺序，因此在扩容时就不会出现链表成环的问题了。但Java8即便如此，HashMap也是不能在并发场景下使用的，因为还存在一个并发修改问题

# 扩容细节

扩容是通过resize方法来实现的。扩容发生在putVal方法的最后，即写入元素之后才会判断是否需要扩容操作，当自增后的size大于之前所计算好的阈值threshold，即执行resize操作。resize的扩容会先创建一个更大容量的数组，
1. 扩容后的数组应该为原数组的两倍，通过位运算<<1进行容量扩充，即扩容1倍，
2. 并且这里的数组大小必须是2的幂，同时新的阈值newThr也扩容为老阈值的1倍。

然后遍历原table，重新计算所有的节点的hash值对应的下标，即rehash操作，最后将节点转移到新table中`Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];`。节点转移时总共存在三种情况：
1. 不存在冲突：哈希桶数组中某个位置只有1个元素，即不存在哈希冲突时，则直接将该元素copy至新哈希桶数组的对应位置即可。
2. 红黑树扩容：哈希桶数组中某个位置的节点为树节点时，则执行红黑树的扩容操作。
3. ==链表扩容，引入高低位链表进行链表拆分。==
![[Pasted image 20231225105012.png]]

resize扩容最重要的操作就是第三种情况，在JDK1.8中，为了避免之前版本中并发扩容所导致的死链问题，将头插法替换为尾插法，尾插法在链表扩容的过程中不会改变原链表的顺序，因此避免了并发死链的问题。

同时，JDK1.8中引入了高低位链表进行链表拆分扩容。高低位链表这里定义了4个变量：loHead, loTail ,hiHead , hiTail，说明扩容后的数据结构存在两个链表。
1. **链表拆分：**找到拆分后仍处于同一个桶的节点，
2. **高低位链表各自成链：**将这些节点重新连接起来。
3. **放置头节点：**将拆分完的链表放进桶里的操作，比较简单，只需要将头节点放进桶里就ok了

```Java
    HashMap.Node<K,V> loHead = null, loTail = null;
    HashMap.Node<K,V> hiHead = null, hiTail = null;
    HashMap.Node<K,V> next;
            //遍历该桶
        do {
        next = e.next;
        //找出拆分后仍处在同一个桶中的节点，将这些节点重新连接起来。
        if ((e.hash & oldCap) == 0) {
            if (loTail == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
        }
        else {
            if (hiTail == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
        }
    } while ((e = next) != null);
        //最后这段代码是将拆分完的链表放进桶里的操作，比较简单，只需要将头节点放进桶里就ok了，
        if (loTail != null) {
        loTail.next = null;
        newTab[j] = loHead;
    }
        if (hiTail != null) {
        hiTail.next = null;
        newTab[j + oldCap] = hiHead;
    }
```

举例如下：

假如hash值h二进制为：0010 1100 1111 0001 1110 1110

旧容量length为16，length-1=0000 1111

那么扩容前索引位置为：h&(length-1)=1110=14

  

扩容后length为32，新length-1=0001 1111

hash值对新容量计算索引时，可以看出新容量-1之后最后四位二进制是不变的，与原容量时计算一致，唯一变化的是倒数第五位的二进制值，本例中hash对应新索引不变还是14，因为hash的倒数第五位为0

但假如hash为0100 0001 0010，旧容量中索引为0010=2，新容量下索引为0001 0010=18=2+16。
![[Pasted image 20231225105038.png]]

==可见在扩容后，索引位置要么不变，要么移动旧容量个位置。==

而判断的依据就是h值与扩容后容量-1的最高位对应位上的值，而扩容后容量-1的最高位其实就是扩容前容量的最高位。只要计算hash值在旧容量最高位对应的二进制是1还是0，是1则会移动到高位索引(原索引位置+原容量)，是零则在低位索引也就是原位置。
即代码中的`if ((e.hash & oldCap) == 0)`：
- 如果=0，则代表h对应位上的值也是0，不移动位置
- 如果=1，则代表h对应位上的值也是1，则需要移动到高位索引。

高低位链表这种实现降低了对共享资源newTab的访问频次，先组织冲突节点，最后再放入newTab的指定位置。避免了JDK1.8之前每遍历一个元素就放入newTab中，从而导致并发扩容下的死链问题

# get 方法

```Java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 数组元素相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶中不止一个节点
        if ((e = first.next) != null) {
            // 在树中get
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 在链表中get
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```