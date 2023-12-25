聊聊Object obj = new Object()

添加分析对象工具JOL的pom依赖：

```XML
        <!--
        JAVA object layout
        官网:http://openjdk.java.net/projects/code-tools/jol/
        定位:分析对象在JVM的大小和分布
        -->
        <dependency>
            <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-core</artifactId>
            <version>0.9</version>
        </dependency>
```

测试类：

```Java
public class JOLDemo{
    public static void main(String[] args){
        Object o = new Object();
        System.out.println( ClassLayout.parseInstance(o).toPrintable());
    }
}
```

# 默认开启类型指针压缩

结果呈现说明：
![[Pasted image 20231225154713.png]]

其中loss due to the next object alignment：就是对齐填充花费的内存

但是为什么类型指针只有4个字节呢？上文不是说好了MarkWord+类型指针==8+8==16字节吗？

控制台执行命令`java -XX:+PrintCommandLineFlags -version`，查看JVM运行时参数：

其中发现了一个参数`-XX:+UseCompressedClassPointers`，说明JVM默认开启了类型指针压缩！！！

```Java
E:\Project\source-bulldozer>java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=265473728 -XX:MaxHeapSize=4247579648 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC
java version "1.8.0_211"
Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)
```

手动关闭压缩再看看 `-XX:-UseCompressedClassPointers`
![[Pasted image 20231225154721.png]]
![[Pasted image 20231225154728.png]]

# 关闭类型指针压缩测试

关闭了类型指针压缩之后，最后再测试一个带有实例数据的对象的内存布局，满足我们的心理预期

对象头：8+8=16字节

- 示例数据：这里占用了int(4)+boolean(1)+boolean(1)=6字节
    
- 对齐填充：2字节
    

```Java
package com.bilibili.juc.objecthead;

import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.vm.VM;

/**
 * @auther zzyy
 * @create 2022-03-06 16:48
 */
public class JOLDemo
{
    public static void main(String[] args)
    {
        Object o = new Object();//16 bytes

        //System.out.println(ClassLayout.parseInstance(o).toPrintable());

        Customer c1 = new Customer();//16 bytes
        System.out.println(ClassLayout.parseInstance(c1).toPrintable());



    }
}


class Customer
{
    //示例数据：这里占用了int(4)+boolean(1)+boolean(1)=6字节
    int id;
    boolean flag = false;
    boolean flag2 = false;
}

输出：
com.bilibili.juc.objecthead.Customer object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0     4           (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4           (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4           (object header)                           90 35 50 1c (10010000 00110101 01010000 00011100) (475018640)
     12     4           (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     4       int Customer.id                               0
     20     1   boolean Customer.flag                             false
     21     1   boolean Customer.flag2                            false
     22     2           (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 2 bytes external = 2 bytes total


Process finished with exit code 0
```

再开启指针压缩：
![[Pasted image 20231225154757.png]]

```Java
com.bilibili.juc.objecthead.JOLDemo
com.bilibili.juc.objecthead.Customer object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0     4           (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4           (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4           (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4       int Customer.id                               0
     16     1   boolean Customer.flag                             false
     17     1   boolean Customer.flag2                            false
     18     6           (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 6 bytes external = 6 bytes total


Process finished with exit code 0
```

再来看下带有实例数据的对象的内存布局：

- 对象头：8+4=12字节
    
- 实例数据：实际情况实际占用，这里占用了int(4)+boolean(1)+boolean(1)=6字节
    
- 对齐填充：6字节

发现压缩的类型指针的4字节最后又通过对齐填充补回来了，这种情况类型指针压缩好似又有点鸡肋，哈哈