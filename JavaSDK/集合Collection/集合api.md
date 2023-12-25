# List操作

```Java
package com.company.listIn_security;

import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

//List不安全
public class ListTest {

    public static void main(String[] args) {
        /**
         * 解决办法：
         * 1、List<String> list = new Vector();
         * 2、List<String> list = Collections.synchronizedList(new ArrayList<>());
         * 3、List<String> list = new CopyOnWriteArrayList<>();
         */

        //List<String> list = new ArrayList<String>();
        //List<String> list = new Vector();
        //List<String> list = Collections.synchronizedList(new ArrayList<>());
        List<String> list = new CopyOnWriteArrayList<>();

        for (int i = 1; i <= 10; i++) {
            new Thread(()->{
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}


之后会报异常：
java.util.ConcurrentModificationException 并发修改异常
导致原因：多线程并发争抢统一资源（这里指list实例），且没加锁
解决办法：
List list = new Vector<>();//vector默认是安全的
or
List list = Collections.synchronizedList(new ArrayList<>());
or
List list = new CopyOnWriteArrayList<>()；//CopyOnWriteArrayList,写入时复制
#CopyOnWrite通俗地讲，当我们往容器中添加一个元素的时候，不是直接添加，而是对当前容器copy，复制一个容器，
#在这个复制的容器中添加元素，添加完之后，再将引用指向这个新容器。缺点是1.内存占用问题，产生了两个容器。
```

Vector的add方法有Synchronized修饰，效率都比较低

```Java
 public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
```

CopyOnWriteArrayList的add方法用的是Lock锁，其add()方法里面加了锁ReentrantLock：写前复制，length+1

```Java
public boolean add(E e) {
        final ReentrantLock lock = this.lock; //加锁
        lock.lock();
        try {
            Object[] elements = getArray();  //先得到老版本的集合
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

# Map操作

```Java
package map;

import java.util.Collection;
import java.util.HashMap;
import java.util.Set;

public class HashMapDemo {

    public static void main(String[] args) {
        HashMap<String, String> map = new HashMap<String, String>();
        // 键不能重复，值可以重复
        map.put("san", "张三");
        map.put("si", "李四");
        map.put("wu", "王五");
        map.put("wang", "老王");
        map.put("wang", "老王2");// 老王被覆盖
        map.put("lao", "老王");
        System.out.println("-------直接输出hashmap:-------");
        System.out.println(map);
        /**
         * 遍历HashMap
         */
        // 1.获取Map中的所有键
        System.out.println("-------foreach获取Map中所有的键:------");
        Set<String> keys = map.keySet();
        for (String key : keys) {
            System.out.print(key+"  ");
        }
        System.out.println();//换行
        // 2.获取Map中所有值
        System.out.println("-------foreach获取Map中所有的值:------");
        Collection<String> values = map.values();
        for (String value : values) {
            System.out.print(value+"  ");
        }
        System.out.println();//换行
        // 3.得到key的值的同时得到key所对应的值
        System.out.println("-------得到key的值的同时得到key所对应的值:-------");
        Set<String> keys2 = map.keySet();
        for (String key : keys2) {
            System.out.print(key + "：" + map.get(key)+"   ");

        }
        /**
         * 4.如果既要遍历key又要value，那么建议这种方式，因为如果先获取keySet然后再执行map.get(key)，map内部会执行两次遍历。
         * 一次是在获取keySet的时候，一次是在遍历所有key的时候。
         */
        // 当我调用put(key,value)方法的时候，首先会把key和value封装到Entry这个静态内部类对象中，再把Entry对象再添加到数组中，
        //所以我们想获取map中的所有键值对，我们只要获取数组中的所有Entry对象，接下来
        //调用Entry对象中的getKey()和getValue()方法就能获取键值对了
        Set<java.util.Map.Entry<String, String>> entrys = map.entrySet();
        for (java.util.Map.Entry<String, String> entry : entrys) {
            System.out.println(entry.getKey() + "--" + entry.getValue());
        }

        /**
         * HashMap其他常用方法
         */
        System.out.println("after map.size()："+map.size());
        System.out.println("after map.isEmpty()："+map.isEmpty());
        System.out.println(map.remove("san"));
        System.out.println("after map.remove()："+map);
        System.out.println("after map.get(si)："+map.get("si"));
        System.out.println("after map.containsKey(si)："+map.containsKey("si"));
        System.out.println("after containsValue(李四)："+map.containsValue("李四"));
        System.out.println(map.replace("si", "李四2"));
        System.out.println("after map.replace(si, 李四2):"+map);
    }

}
```

# Set操作

```Java
package com.company.listIn_security;

import java.util.*;
import java.util.concurrent.CopyOnWriteArraySet;

//set不安全
public class SetTest {
    public static void main(String[] args) {
        //同List
        //1.Set<String> set = new HashSet<String>();
        //2.Set<String> set = Collections.synchronizedSet(new HashSet<String>());
        //3.Set<String> set = new CopyOnWriteArraySet(new HashSet<String>());

        Set<String> set = new CopyOnWriteArraySet(new HashSet<String>());
        for (int i = 1; i <= 10; i++) {
            new Thread(()->{
                set.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(set);
            },String.valueOf(i)).start();
        }
    }
}

同样是报错：
java.util.ConcurrentModificationException
解决办法：
Set<String> set = Collections.synchronizedSet(new HashSet<String>());
or
Set<String> set = new CopyOnWriteArraySet(new HashSet<String>());
```