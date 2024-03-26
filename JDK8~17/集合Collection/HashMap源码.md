# 数据结构

JDK1.8 之前 HashMap 由 数组+链表Node<K,V>组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。
```java
transient Node<K,V>[] table;

static class Node<K,V> implements Map.Entry<K,V> {  
    final int hash;  
    final K key;  
    V value;  
    Node<K,V> next;  
  
    Node(int hash, K key, V value, Node<K,V> next) {  
        this.hash = hash;  
        this.key = key;  
        this.value = value;  
        this.next = next;  
    }  
  
    public final K getKey()        { return key; }  
    public final V getValue()      { return value; }  
    public final String toString() { return key + "=" + value; }  
  
    public final int hashCode() {  
        return Objects.hashCode(key) ^ Objects.hashCode(value);  
    }  
  
    public final V setValue(V newValue) {  
        V oldValue = value;  
        value = newValue;  
        return oldValue;  
    }  
  
    public final boolean equals(Object o) {  
        if (o == this)  
            return true;  
        if (o instanceof Map.Entry) {  
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;  
            if (Objects.equals(key, e.getKey()) &&  
                Objects.equals(value, e.getValue()))  
                return true;  
        }  
        return false;  
    }  
}
```

JDK1.8 之后 HashMap 的组成多了红黑树TreeNode<K,V>，在满足下面两个条件之后，会执行链表转红黑树操作，以此来加快搜索速度。

>这里简单地回顾一下红黑树，它是一种平衡的二叉树搜索树，类似地还有AVL树。两者核心的区别是AVL树追求“绝对平衡”，在插入、删除节点时，成本要高于红黑树，但也因此拥有了更好的查询性能，适用于读多写少的场景。然而，对于HashMap而言，读写操作其实难分伯仲，因此选择红黑树也算是在读写性能上的一种折中。
>
>红黑树的“平衡”的意思并不是说两个子树的叶子节点个数精确一样，而是说两个子树的高度不会差太多（对于AVL树，任何一个节点的两个子树高度差不会超过1；对于红黑树，则是不会相差两倍以上），从而在这样的树中进行搜索的话即便在最坏情况下也会很高效，这就足够了。

红黑树的生成条件，注意：下面两个条件存在先后顺序：
1. 随着`putVal`的添加，当链表长度大于阈值时（阈值默认为 8），会首先调用`treeifyBin`方法
2. `treeifyBin`方法当中会判断HashMap 数组长度是否超过 64，若超过64则会执行链表转红黑树操作以减少搜索时间，否则只是简单执行`resize`方法对数组进行扩容。
![[Pasted image 20231225103846.png]]

![[Pasted image 20231225103853.png]]

那么为什么要引入红黑树来替代链表呢？虽然链表的插入性能是O(1)，但查询性能确是O(n)，当哈希冲突元素非常多时，这种查询性能是难以接受的。因此，在JDK1.8中，如果冲突链上的元素数量大于8，并且哈希桶数组的长度大于64时，会使用红黑树代替链表来解决哈希冲突，使查询具备O(logn)的性能。此时的节点会被封装成TreeNode而不再是Node。

下文我们根据JDK1.8源码去剖析HashMap，我们从全局变量入手。

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

成员变量`threshold`默认是0，它就是数组扩容阈值，但是在第一次putVal的时候它才会被赋值为`threshold = capacity * loadFactor`（懒加载）。

另外，源码中的threshold注释中强调了`threshold`在数组还没有被分配的情况下还可以充当初始数组容量，特别的`threshold=0`的时候代表 `DEFAULT_INITIAL_CAPACITY`（16）
![[Pasted image 20231225103950.png]]
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

## HashMap()无参构造

以默认构造函数为例，通过new创建出HashMap的时候只会设置默认负载因子，并不会对其进行初始化操作，只有在首次使用put添加元素的时候才会进行初始化（resize方法第一次调用）。
![[Pasted image 20231225104045.png]]
resize()中会设置默认的初始化容量DEFAULT_INITIAL_CAPACITY为16，扩容的阈值为`0.75x16 = 12`，最后创建容量为16的`Node`数组，并赋值给成员变量哈希桶`table`，即完成了HashMap的初始化操作
![[Pasted image 20231225104054.png]]

## HashMap(int initialCapacity)

### tableSizeFor
当用户使用其余三种构造方法创建HashMap的时候，用户传入自定义容量`initialCapacity`，但JDK并不使用用户传入的`initialCapacity`，==而是触发`tableSizeFor(initialCapacity)`方法将输入的cap的进行加工==，转换为“大于等于给定值cap的最小2的整数次幂”作为最终容量。
```java
/**  
 * Returns a power of two size for the given target capacity. 
 * */
static final int tableSizeFor(int cap) {  
    int n = cap - 1;  
    n |= n >>> 1;  //先计算无符号右移`>>>`，后计算或运算|
    n |= n >>> 2;  
    n |= n >>> 4;  
    n |= n >>> 8;  
    n |= n >>> 16;  
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;  
}
```

>`>>`表示右移，移出的部分将被抛弃。如果该数为正，则高位补0，若为负数，则高位补1。
	>1. 对于正数右移：假设我们有一个正数 `5`，其二进制表示为 `0000 0101`，如果我们对它进行右移一位，即 `5 >> 1`，则得到 `0000 0010`，即 `2`。
	>2. 对于负数右移：假设我们有一个负数 `-5`，其二进制表示为 `1111 1011`（这里使用补码表示法：负数=绝对值的反码+1）。如果我们对其进行右移一位，即 `-5 >> 1`，同时高位补 `1`，因此得到 `1111 1101`，即 `-3`。
>
>`>>>`表示无符号右移，也叫忽略符号位右移，也叫逻辑右移，即若该数为正，则高位补0，而若该数为负数，则右移后高位同样补0

当cap为3时，则n=2，tableSizeFor整个计算过程如下：
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

hash 方法返回hash值，hash值用来计算 `(n - 1) & hash` 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

JDK 1.8 HashMap 的 hash 方法源码：
```Java
    static final int hash(Object key) {
      int h;
	  //从这里也可以看出来，HashMap的key是可以为null的，key为null时扰动函数`return 0`;
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

# 插入过程（四大步骤）
`putVal`方法，根据`(n - 1) & hash` 确定元素存放在哪个桶中，插入过程分为四大步：
1. 如果通过HashMap的无参构造方法创建的table，则在此处扩容，即HashMap懒加载
2. (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)，代码末尾统一
3. 桶中已经存在元素
	1. 比较桶中第一个元素(数组中的结点)的hash值相等，key相等，将第一个元素赋值给e，用e来记录，代码末尾统一覆盖。若比较桶中第一个元素(数组中的结点)hash值不相等即key不相等，沿着链表向下找
	2. 为红黑树结点吗，直接插入或者记录冲突节点e
	3. 为链表结点吗，直接插入（尾插）或者记录冲突节点e
	4. 若e != null，最后执行统一的覆盖操作，表示在桶中找到key值、hash值与插入元素相等的结点
4. 如果上面没有return，来到了最后一段代码，说明没有冲突，只是往数组里面新增元素，因此需要结构性修改++modCount
```Java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)//将老数组table（可以看作所有哈希桶的根节点）复制给tab
    //1. 如果通过HashMap的无参构造方法创建的table，则在此处扩容，即HashMap懒加载
        n = (tab = resize()).length;
    //2. (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //3.桶中已经存在元素
    else {
        Node<K,V> e; K k;
        // 3.1比较桶中第一个元素(数组中的结点)的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
                // 将第一个元素赋值给e，用e来记录，代码末尾统一覆盖
                e = p;
        // 比较桶中第一个元素(数组中的结点)hash值不相等即key不相等，沿着链表向下找
        // 3.2为红黑树结点吗
        else if (p instanceof TreeNode)
            // putTreeVal返回null表示直接插入, 或者返回冲突节点赋值给e由e来记录，代码末尾统一覆盖
            // putTreeVal和put方法的注释如下：Returns:the previous value associated with key, or null if there was no mapping for key. (A null return can also indicate that the map previously associated null with key.)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 3.3为链表结点吗
        else {
            // 在链表最末插入结点，或者提前遍历到冲突节点赋值给e，用e来记录。代码末尾统一覆盖
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
                //当两个键的 `hashCode()` 方法返回相同的哈希码，并且它们的 `equals()` 方法返回 true 时，HashMap 将认为这两个键是相等的
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
        // 4. 覆盖操作在此e != null：表示在桶中找到key值、hash值与插入元素相等的结点
        if (e != null) {
            // 记录e的value
            V oldValue = e.value;
            // onlyIfAbsent为false，或者旧值为null
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 如果上面没有冲突，说明是新增元素到了普通数组里面，需要结构性修改数组大小
    ++modCount;
    // 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
}
```

对 putVal 方法添加元素的分析如下，流程图来自于互联网并不是很准确，纠正如下，红黑树少了个覆盖操作图上漏掉了
- 头节点的key存在，则覆盖
- 链表节点存在key，则覆盖
- 红黑树节点存在key，则覆盖
![[Pasted image 20231225104924.png]]
# 如何判断key相等！重要
注意：怎么比较key相等？要求两个键的 `hashCode()` 方法返回相同的哈希码，并且它们的 `equals()` 方法返回 true 时，HashMap 才认为这两个键是相等的，截取putVal的如下代码：
```java
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 判断链表中结点的key值与插入的元素的key值是否相等
                //当两个键的 `hashCode()` 方法返回相同的哈希码，并且它们的 `equals()` 方法返回 true 时，HashMap 将认为这两个键是相等的
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
```
以下是一个简单的示例，演示了如何在 Java 中使用自定义对象作为 HashMap 的键，并重写 `hashCode()` 和 `equals()` 方法来比较键的相等性：
```java
import java.util.HashMap;

class MyKey {
    private int id;

    public MyKey(int id) {
        this.id = id;
    }

    // 重写 hashCode() 方法
    @Override
    public int hashCode() {
        // 这里可以根据实际情况生成哈希码
        return id;
    }

    // 重写 equals() 方法
    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null || getClass() != obj.getClass())
            return false;
        MyKey other = (MyKey) obj;
        return id == other.id;
    }
}

public class Main {
    public static void main(String[] args) {
        HashMap<MyKey, String> hashMap = new HashMap<>();

        MyKey key1 = new MyKey(1);
        MyKey key2 = new MyKey(1);

        // 添加键值对到哈希映射
        hashMap.put(key1, "Value1");

        // 使用键2查询值
        String value = hashMap.get(key2);

        System.out.println("Value corresponding to key2: " + value); // 输出 Value1
    }
}

```
输出信息如下，尽管我们使用了不同的 `MyKey` 对象实例（`key1` 和 `key2`），但我们重写了equals方法和hashCode方法
- hashcode相同
- equals返回也相同
所以它们被认为是相等的键，所以我们可以成功地使用 `key2` 来检索哈希映射中的值。
```shell
输出 Value1
```
# 扩容细节（高低位链表）

## 1.7并发扩容死链问题
关于HashMap链表插入操作，java7及之前之前是头插法，当时写这个代码的作者认为后来的值被查找的可能性更大一点，提升查找的效率。因为HashMap插入操作是不支持并发的，所以头插法会导致HashMap的在并发扩容的时候会导致链表成环的问题。导致线上在执行get的时候，会触发死循环，引起CPU的100%问题，所以一定要避免在并发环境下使用HashMap。
死链产生的两个条件：
1. hashmap对数组进行扩容，尾插法会改变节点的顺序
2. 又恰好扩容前多线程并发访问了hashmap，就会导致在扩容后出现死链现象
阶段一：扩容前扩容前t2线程休眠
![[Pasted image 20240326214246.png]]
阶段二：t1线程顺利参与了扩容整个流程
![[Pasted image 20240326214531.png]]
阶段三：t2醒过来，死链发生
![[Pasted image 20240326215000.png]]
阶段四：如果当前链表后面还有元素，则无法抵达，cpu循环跑满100%
![[Pasted image 20240326215126.png]]
因此在JDK1.8中，为了避免之前版本中并发扩容所导致的死链问题，将头插法替换为尾插法，尾插法在链表扩容的过程中不会改变原链表的顺序，因此避免了并发插入下的死链问题。头插法->尾插法

但即使是这样，HashMap仍然不是线程安全的，因为还存在并发修改问题，要想完全的线程安全则必须是ConCurrentHashMap
## l.8HashMap扩容优化
扩容是通过resize方法来实现的。扩容发生在putVal方法的最后，即写入元素之后才会判断是否需要扩容操作，当自增后的size大于之前所计算好的阈值threshold，即执行resize操作。
1. resize的扩容会先创建一个更大容量的数组newTable，容量为原数组的两倍，通过位运算<<1进行容量扩充，即扩容1倍，
2. 同时新的阈值newThr也扩容为老阈值的2倍。

然后遍历原table，重新计算所有的节点的hash值对应的下标，即rehash操作，最后将节点转移到新table中`Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];`。节点转移时总共存在三种情况：
1. 不存在冲突：哈希桶数组中只有第1个元素，即不存在哈希冲突时，则直接将该元素copy至新哈希桶数组的对应位置即可。
2. 红黑树扩容：哈希桶数组中某个位置的节点为树节点时，则执行红黑树的扩容操作。
3. ==链表扩容，引入高低位链表进行链表拆分。==
![[Pasted image 20231225105012.png]]

resize扩容最重要的操作就是第三种情况，高低位链表进行链表拆分扩容。高低位链表这里定义了4个变量：loHead, loTail ,hiHead , hiTail，说明扩容后的数据结构存在两个链表。
1. 链表拆分：找到拆分后仍处于同一个桶的节点，
2. 高低位链表各自成链：将这些节点重新连接起来。
3. 放置头节点：将拆分完的链表放进桶里的操作，比较简单，只需要将头节点放进桶里就ok了
高低位链表扩容源码如下：
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

举例如下，之所以元素B，hash值为0100 0001 0010，扩容后被移动了旧Length个距离，是因为h&(newLength-1)=18
18=2+16，16来自于
- hash的倒数第五位是1
- newLength-1的最高位也是1
- 最高位正好也是倒数第五位，因此二者的与运算也是1
newLength-1的最高位其实就是扩容前容量的最高位，所以上述移动16的判断条件可以简化为`if ((e.hash & oldCap) == 1)`
![[Pasted image 20231225105038.png]]

==可见在扩容后，索引位置要么不变，要么移动旧容量个位置==。综上所述，通过即计算hash值在旧容量最高位对应的二进制是1还是0
- 如果(e.hash & oldCap)=0，扩容后不移动位置
- 如果(e.hash & oldCap)=1，扩容后需要移动到高位索引。
高低位链表这种实现降低了对共享资源newTab的访问频次，先组织冲突节点，最后再放入newTab的指定位置。避免了JDK1.8之前每遍历一个元素就放入newTab中，从而容易导致并发扩容下的死链问题
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