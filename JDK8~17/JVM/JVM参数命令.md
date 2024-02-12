在本篇文章中，你将掌握最常用的 JVM 参数配置。

# 堆内存相关

Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**

## 指定堆内存–Xms和-Xmx

与性能有关的最常见实践之一是根据应用程序要求初始化堆内存。如果我们需要指定最小和最大堆大小（推荐显示指定大小），以下参数可以帮助你实现：

```Java
-Xms<heap size>[unit] //heap size 表示要初始化内存的具体大小。unit 表示要初始化内存的单位。单位为g(GB) 、m（MB）、k（KB）。
-Xmx<heap size>[unit]

//如果我们要为JVM分配最小2 GB和最大5 GB的堆内存大小，我们的参数应该这样来写：
-Xms2G -Xmx5G
```

## 指定新生代内存(Young Generation)

根据[Oracle官方文档open in new window](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/sizing.html)，在堆总可用内存配置完成之后，第二大影响因素是为 Young Generation 在堆内存所占的比例。默认情况下，YG 的最小大小为 1310 _MB_，最大大小为_无限制_。

一共有两种指定 新生代内存(Young Ceneration)大小的方法：

**1.通过-XX:NewSize和-XX:MaxNewSize指定**

```Java
-XX:NewSize=<young size>[unit] 
-XX:MaxNewSize=<young size>[unit]
//如果我们要为 新生代分配 最小256m 的内存，最大 1024m的内存我们的参数应该这样来写：
-XX:NewSize=256m
-XX:MaxNewSize=1024m
```

**2.通过`-Xmn<young size>[unit]` 指定

举个栗子🌰，如果我们要为 新生代分配256m的内存（NewSize与MaxNewSize设为一致），我们的参数应该这样来写：

```Java
-Xmn256m  
```

GC 调优策略中很重要的一条经验总结是这样说的：

将新对象预留在新生代，由于 Full GC 的成本远高于 Minor GC，因此尽可能将对象分配在新生代是明智的做法，实际项目中根据 GC 日志分析新生代空间大小分配是否合理，适当通过“-Xmn”命令调节新生代大小，最大限度降低新对象直接进入老年代的情况。

## 指定新老代内存的比例

另外，你还可以通过`-XX:NewRatio=<int>`来设置新生代和老年代内存的比值。

比如下面的参数就是设置新生代（包括Eden和两个Survivor区）与老年代的比值为1。也就是说：新生代与老年代所占比值为1：1，新生代占整个堆栈的 1/2。

```Java
-XX:NewRatio=1 
```

# 方法区相关

如果我们没有指定 Metaspace 的大小，随着更多类的创建，虚拟机会耗尽所有可用的系统内存

**JDK 1.8 之前永久代还没被彻底移除的时候通常通过下面这些参数来调节方法区大小。**

```Java
-XX:PermSize=N //方法区 (永久代) 初始大小
-XX:MaxPermSize=N //方法区 (永久代) 最大大小,超过这个值将会抛出 OutOfMemoryError 异常:java.lang.OutOfMemoryError: PermGen
```

**JDK 1.8 的时候，方法区（HotSpot 的永久代）被彻底移除了（JDK1.7 就已经开始了），取而代之是元空间，元空间使用的是本地内存。**

```Java
-XX:MetaspaceSize=N //设置 Metaspace 的初始（和最小大小）
-XX:MaxMetaspaceSize=N //设置 Metaspace 的最大大小，如果不指定大小的话，随着更多类的创建，虚拟机会耗尽所有可用的系统内存。
```

# 垃圾收集相关

## 垃圾回收器

为了提高应用程序的稳定性，选择正确的[垃圾收集open in new window](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)算法至关重要。

JVM具有四种类型的_GC_实现：

- 串行垃圾收集器
    
- 并行垃圾收集器
    
- CMS垃圾收集器
    
- G1垃圾收集器
    

可以使用以下参数声明这些实现：

```Java
-XX:+UseSerialGC
-XX:+UseParallelGC
-XX:+UseParNewGC
-XX:+UseG1GC
```

有关垃圾回收实施的更多详细信息，请参见[此处open in new window](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/JVM%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6.md)。

## GC记录

为了严格监控应用程序的运行状况，我们应该始终检查JVM的_垃圾回收_性能。最简单的方法是以人类可读的格式记录_GC_活动。

使用以下参数，我们可以记录_GC_活动：

```Java
-XX:+UseGCLogFileRotation 
-XX:NumberOfGCLogFiles=< number of log files > 
-XX:GCLogFileSize=< file size >[ unit ]
-Xloggc:/path/to/gc.log
```