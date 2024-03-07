LinkedHashMap记录了元素插入顺序，除了保存当前对象的引用外，还保存了其上一个元素before和下一个元素after的引用，从而在HashMap的基础上构成了双向链接列表。这样就能按照插入的顺序遍历原本无序的HashMap。LinkedHashMap的有序可以按两种顺序排列，一种是按照插入的顺序FIFO（先进先出），一种是按照读取的顺序LRU（Least Recently Used，优先淘汰最近最少使用的元素）

以下是LinkedHashMap的源码构造函数：

```Java
    /**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

默认情况下，accessOrder是false的即FIFO，如果设置成true就可实现LRU。LinkedHashMap内部是靠建立一个双向链表来维护这个顺序的，在每次访问、删除、插入后，都会调用一个函数来进行 双向链表的维护 ，准确的来说，是有三个函数来做这件事`Callbacks to allow LinkedHashMap post-actions`，这三个函数都统称为 回调函数 ，这三个函数分别是：

```Java
//其作用就是在访问元素之后，将该元素放到双向链表的尾巴处(所以这个函数只有在按照读取的顺序的时候才会执行)，之所以提这个，是建议大家去看看，如何优美的实现在双向链表中将指定元素放入链尾！
void afterNodeAccess(Node<K,V> p) { }
//其作用就是在删除元素之后，将元素从双向链表中删除，还是非常建议大家去看看这个函数的，很优美的方式在双向链表中删除节点！
void afterNodeRemoval(Node<K,V> p) { }
//这个才是我们题目中会用到的，在插入新元素之后，需要回调函数判断是否需要移除一直不用的某些元素！
void afterNodeInsertion(boolean evict) { }
```

在`LinkedHashMap`当中，`afterNodeInsertion`会调用`removeEldestEntry`，默认的removeEldestEntry方法永远返回false，意味着LinkedHashMap默认采用扩容机制。从源码可以看出，它本身就是返回false的，所以只要在子类中重写这个方法就可以按照我们自己的定义去实现FIFO和LRU

```Java
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```

我们需要复写`removeEldestEntry`，使之`return size() > SIZE;`

```Java
    //重写淘汰机制
    @Override
    protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {
        return size() > SIZE;  //如果缓存存储达到最大值删除最后一个
    }
```

# 实现FIFO

继承LinkedHashMap实现FIFO缓存机制：

```Java
package com.brickworkers;

import java.util.LinkedHashMap;

public class FIFOCache<K,V> extends LinkedHashMap<K, V>{

    private static final long serialVersionUID = 436014030358073695L;

    private final int SIZE;

    public FIFOCache(int size) {
        super();//调用父类无参构造，不启用LRU规则
        SIZE = size;
    }

    //重写淘汰机制
    @Override
    protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {
        return size() > SIZE;  //如果缓存存储达到最大值删除最后一个
    }
}
```

# 实现LRU

继承LinkedHashMap实现LRU缓存机制：

```Java
package com.brickworkers;

import java.util.LinkedHashMap;

public class LRUCache<K,V> extends LinkedHashMap<K, V> {
    private static final long serialVersionUID = 5853563362972200456L;

    private final int SIZE;

    public LRUCache(int size) {
        super(size, 0.75f, true);  //int initialCapacity, float loadFactor, boolean accessOrder这3个分别表示容量，加载因子和是否启用LRU规则
        SIZE = size;
    }

    //LinkedHashMap有一个removeEldestEntry(Map.Entry eldest)方法，默认返回false
    //通过覆盖这个方法，加入一定的条件，满足条件返回true。当put进新的值方法返回true时，便移除该map中最老的键和值。避免内耗
    @Override
    protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {
        return size() > SIZE;
    }
}
```