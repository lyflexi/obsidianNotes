我们知道，jdk打娘胎出来在`${JAVA_HOME/bin}`下有很多命令行工具，如：
- jps：查看正在运行的Java进程
- jstat：查看JVM统计信息
- jinfo：实时查看JVM配置参数，在 Java 9 及更高版本中支持参数动态修改但也并非所有JVM参数都支持动态修改，修改某些参数最好还是重启JVM。
- jmap：用于生成堆内存映射或堆转储文件（heap dump）
- jstack：打印JVM中的线程快照
![[Pasted image 20240303105959.png]]
获取命令具体参数选项，可以通过`command -help` 获取。例如：
```shell
[root@localhost bin]# jps -help
usage: jps [-help]       
jps [-q] [-mlvV] [<hostid>]

Definitions:    
<hostid>:      <hostname>[:<port>]
```
# JVM脚本命令
## jps：查看正在运行的Java进程
通过`jps`命令获取Java进程的进程ID（PID）以及主类名称或JAR文件的完整路径名。
[参数选项]：
- `-l`：输出主类或者JAR的完全路径名。例如，运行`jps -l`命令将列出所有Java进程及其对应的主类名称或JAR文件的完整路径名。
- `-v`：输出JVM参数。列出每个Java进程的JVM参数信息。
- `-m`：输出JVM启动时传递给main()方法的参数。
- `-V`：（特定环境或版本可能支持）提供特定于该环境或版本的输出或功能。
- `<hostid>`：指定要查询 Java 进程信息的远程主机。
测试用例：
```shell
[root@localhost ~]# jps -mlvV10217 demo-jvm-1.0-SNAPSHOT.jar10285 sun.tools.jps.Jps -mlvV -Dapplication.home=/usr/local/java/jdk1.8.0_221 -Xms8m
```
结果演示：
![[Pasted image 20240303110726.png]]
## jstat：查看JVM统计信息

`jstat` 是 Java 虚拟机（JVM）自带的监控工具，用于查看 HotSpot JVM 的性能统计信息。
```shell
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```
[参数选项]：
基础参数：
- `-t`：展示从虚拟机运行到现在的性能数据
- `-h<lines>`：每隔lines行展示行头部信息
- `<vmid>`：JVM进程的虚拟机标识符。
- `<interval>`：两次统计信息收集之间的时间间隔（秒）。
- `<count>`：收集统计信息的次数。
性能参数：
- `-class`：显示类加载器的统计信息。
- `-compiler`：显示即时编译器的统计信息。
- `-gc`：显示垃圾收集的统计信息。
- `-gccapacity`：显示各内存池的容量和使用情况。
- `-gccause`：显示上一次或当前垃圾收集的原因及相关统计信息。
- `-gcnew`：显示新生代的垃圾收集统计信息。
- `-gcnewcapacity`：显示新生代内存池的容量和使用情况。
- `-gcold`：显示老年代的垃圾收集统计信息。
- `-gcoldcapacity`：显示老年代内存池的容量和使用情况。
- `-gcpermcapacity`：显示永久代的容量和使用情况（Java 8 及以后版本可能不再适用）。
- `-printcompilation`：输出被即时编译器编译的方法信息。
测试用例：
通过jstat工具监控指定Java进程（进程ID为10217）的垃圾收集（GC）统计信息，并限制输出结果的行数为2行。每1秒收集一次数据，总共收集3次。
```shell
[root@localhost ~]# jstat -gc -h 2 10217 1s 3
```
结果演示：
![[Pasted image 20240303111504.png]]
## jmap：导出内存映像文件&内存使用情况
`jmap` 用于生成堆内存映射或堆转储文件（heap dump）。这些文件可以用于后续的堆分析，帮助开发者诊断内存泄漏、内存溢出等问题。
```
jmap [option] <pid>
```
其中，`<pid>` 是要分析的 Java 进程的进程 ID。
[参数选项]：
- `-heap`：打印出堆内存的概要信息，包括各代（新生代、老年代）的使用情况、GC 配置等。
- `-histo`：打印堆内存的直方图，列出每个类的实例数量和总字节大小。帮助识别哪些类占用了大量内存。
- `-dump:<dump-options>`：生成堆转储文件。`<dump-options>` 可以是 `live`（仅转储活动的对象）、`format=b`（二进制格式）、`file=<filename>`（指定转储文件的名称）。
- `-finalizerinfo`：打印出正在等待 Finalizer 线程执行的对象信息。
- `-clstats`：打印类加载器的统计信息，包括加载的类数量、卸载的类数量等。
- `-printcompilation`：打印出即时编译器编译的代码信息。
例如，要生成一个名为 `heapdump.hprof` 的堆转储文件，可以使用以下命令：
```shell
jmap -dump:format=b,file=heapdump.hprof <pid>
```
注意: 生成堆转储文件可能会对正在运行的 JVM 产生性能影响，特别是在堆内存很大的情况下。因此，建议在系统负载较低或处于可接受的范围内时执行此操作。
## jinfo：实时查看和修改JVM配置参数
`jinfo` 用于实时查看和修改运行中的 Java 进程的 JVM 配置参数。
[参数选项]：
- `-flag <name>`        打印指定名称的 VM 标志的值
- `-flag [+|-]<name>`   启用或禁用指定名称的 VM 标志
- `-flag <name>=<value>`将指定名称的 VM 标志设置为给定值
- `-Flags`             打印 VM 标志
- `-sysprops`          打印 Java 系统属性
- `<no option>`        同时打印以上两者
- `-h` 或 `-help`      打印此帮助信息
测试用例，实时查看指定进程如10217的JVM配置信息
```shell
#查看所有 JVM 标志和系统属性
jinfo 10217
#查看特定 JVM 标志的值
jinfo -flag MaxHeapSize 10217
#将最大堆内存大小设置为 2GB：
jinfo -flag MaxHeapSize=2g 10217
#查看所有 Java 系统属性
jinfo -sysprops 10217
```
## jstack：打印JVM中线程快照

`jstack(JVM Stack Trace)`：用于生成虚拟机指定进程当前时刻的线程快照(虚拟机堆栈跟踪)。
```
jstack <pid>
```
线程快照就是当前虚拟机内指定进程的每一条线程正在执行的方法堆栈的集合。可用于定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等问题。就可以用jstack显示各个线程调用的堆栈情况。
[参数选项]：
- `-F`：强制生成线程堆栈。当 `jstack <pid>` 没有响应（进程挂起）时使用。
- `-m`：如果调用到本地方法的话，可以显示C/C++的堆栈
- `-l`：打印关于锁的额外信息。
测试用例：
```
jstack 10217
```
演示效果：
![[Pasted image 20240303112319.png]]
# JVM监控工具
使用命令行工具或组合能帮您获取目标Java应用性能相关的基础信息，但它们无法获取方法级别的分析数据，如方法间的调用关系、各方法的调用次数和调用时间等（这对定位应用性能瓶颈至关重要）。而且结果展示不够直观。

为此，JDK提供了一些内存泄漏的分析工具，如`jconsole，jvisualvm`等，用于辅助开发人员定位问题，但是这些工具很多时候并不足以满足快速定位的需求。

## jvisualvm
为方便测试，我们在window下使用Visual VM ：{JAVA_HOME}/bin/jvisualvm.exe 来演示效果（功能性详细信息，请自行发掘）
比如：我们在程序中写一个最简单OOM的例子。
```java
public static void main(String[] args) {
    List<Object> list = new ArrayList<>();
    while (true) {
        list.add(new Object());
    }
}
```
运行一段时间，点击运行jvisualvm.exe，显示内存使用情况
![[Pasted image 20240303112547.png]]
后台运行情况：
```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```
尽管直观地看到CPU、线程数、对空间的直观变化。但是这些信息有时候并不能满足我们的分析和定位需求。因此，下边介绍2个重常用的第三方工具。
## eclipse MAT
`MAT: MAT(Memory Analyzer Tool)`: 基于Eclipse的内存分析工具，是一个快速、功能丰富的Java heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。
官网下载地址：https://eclipse.dev/mat/downloads.php
为演示效果，设置JVM参数：
```
java -jar -Xms8m -Xmx8m -XX:+PrintGC -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:H
```
运行一段时间，出现OOM异常,并生成了转存文件：java_pid11469.hprof。
![[Pasted image 20240303112641.png]]
本地用mat工具导入并打开刚刚的转存文件，辅助我们定位OOM具体发生在哪里
![[Pasted image 20240303112655.png]]
事实上，这和后台进程显示是一致的。
![[Pasted image 20240303112720.png]]
## 阿里 Arthas
`Arthas`:Alibaba开源的Java诊断工具。
官方使用教程：https://arthas.aliyun.com/
功能特性：
- 支持在线排查，无需重启
- 动态跟踪Java代码
- 实时监控JVM状态
当你遇到以下类似问题时，Arthas可以帮助你解决（根据学习资料整理）：
1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2. 修改的代码是否生效？难道没commit？或者分支搞错了？
3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5. 如何以全局视角来查看系统的运行状况？
6. 有什么办法可以监控到JVM的实时运行状态？
7. 怎么快速定位应用的热点，生成火焰图？
相关诊断指令：https://arthas.aliyun.com/doc/commands.html
用户案例：https://github.com/alibaba/arthas/issues?q=label%3Auser-case


# JVM调优参数

## 设置堆、栈、方法区等内存大小

- `-Xmx4g`: 设置进程占用的最大堆空间大小为4GB，超出后会导致OutOfMemoryError。
- `-Xms2g`: 设置初始化堆空间大小为2GB。
- -XX:MaxNewSize=256m：新生代分配最小256m 的内存
- -XX:NewSize=1024m：新生代分配最大1024m的内存
- `-Xmn1g`: 设置年轻代大小为1GB，意味着`MaxNewSize==NewSize`
- `-XX:NewRatio=n`: 设置年轻代和老年代空间大小的比值。
- `-Xss512k`: 设置每个线程占用的内存大小为512KB。
- `-XX:SurvivorRatio=n`: 设置年轻代中Eden区与Survivor区的比值，例如n=4时，Eden和Survivor的比值为4:2。
- `-XX:MetaspaceSize=512m`: 设置元空间（Metaspace）的初始大小为512MB。
- `-XX:MaxMetaspaceSize=512m`: 设置元空间（Metaspace）增长的上限，防止无限制地使用本地内存。
- `-XX:MinMetaspaceFreeRatio=N`: 设置Metaspace空闲空间的最小比例，控制Metaspace的增长速度。
- `-XX:MaxMetaspaceFreeRatio=N`: 设置Metaspace空闲空间的最大比例，控制Metaspace的释放。
- `-XX:MaxMetaspaceExpansion=N`: 设置Metaspace增长时的最大幅度。
## 设置垃圾收集器

- `-XX:+UseSerialGC`: 设置使用串行收集器。
- `-XX:+UseParallelGC`: 设置使用并行收集器。
- `-XX:+UseParNewGC`：设置使用并行新生代收集器。
- `-XX:+UseParalledlOldGC`: 设置使用并行年老代收集器。
- `-XX:+UseConcMarkSweepGC`: 设置使用并发收集器。
- ==`-XX:ParallelGCThreads=n`: 设置并行收集器使用的gc线程数。这个并行gc参数只要是并行 GC 都可以使用，不只是 ParNew。==
- `-XX:MaxGCPauseMillis=n`: 设置并行收集的最大暂停时间。
- `-XX:GCTimeRatio=n`: 设置垃圾回收时间占程序运行时间的百分比，1/(1+n)。
- `-XX:+DisableExplicitGC`: 禁止外部调用`System.gc()`。
- `-XX:MaxTenuringThreshold`: 设置年轻代对象复制到老年代前的最大复制次数。
## 垃圾回收信息统计
使用以下参数，我们可以记录_GC_活动：
- `-XX:+PrintGC`: 打印垃圾回收信息。
- `-XX:+PrintGCDetails`: 打印详细的垃圾回收信息。
- `-XX:+PrintGCTimeStamps`: 打印每次垃圾回收前程序未中断的执行时间。
- `-Xloggc:filepath`: 把GC日志存入指定文件。
- `-XX:+PrintGCApplicationStoppedTime`: 打印垃圾回收期间程序暂停的时间。
- `-XX:+PrintGCApplicationConcurrentTime`: 打印每次垃圾回收前程序未中断的执行时间。
- `-XX:+PrintHeapAtGC`: 打印GC前后的详细堆栈信息。
- `-XX:+HeapDumpOnOutOfMemoryError`: 在OutOfMemoryError时生成堆转储。
- `-XX:HeapDumpPath=/dump`: 设置堆转储文件的路径。
