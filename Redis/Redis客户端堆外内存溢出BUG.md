> 堆外内存直接受操作系统管理（而不是虚拟机），这样做的结果就是能够在一定程度上减少垃圾回收对应用程序造成的影响。主要有：
> 1. 方法区。方法区属于非堆内存。它存储每个类结构，如运行时常数池、字段和方法数据，以及方法和构造方法的代码。它是在 Java 虚拟机启动时创建的。方法区在逻辑上属于堆，但 Java 虚拟机实现可以选择不对其进行回收或压缩。与堆类似，方法区的内存不需要是连续空间，因此方法区的大小可以固定，也可以扩大和缩小
> 2. 除了方法区外，Java 虚拟机实现可能需要用于内部处理或优化的内存，这种内存也是堆外内存。例如，JIT 编译器需要内存来存储从 Java 虚拟机代码转换而来的本机代码，从而获得高性能。

lettuce和jedis都是操作redis的底层客户端，springboot2.0以后默认使用`lettuce`作为操作redis的客户端，`lettuce`使用netty进行网络通信。

由于lettuce的bug，当你进行压力测试时后期，Redis出现堆外内存溢出OutOf DirectMemoryError，翻译过来叫直接内存，但是业界喜欢称直接内存为堆外内存。

临时的解决方式是，netty可以使用-Dio.netty.maxDirectMemory设置堆外内存，但netty如果没有指定堆外内存，默认使用Xms的值

但由于是lettuce的bug造成，不建议使用-Dio.netty.maxDirectMemory去调大虚拟机堆外内存，治标不治本。因为它没有及时释放内存，即使最后升级lettuce客户端，依旧解决不了问题

因此最终的解决方案是剔除`lettuce`：切换使用`jedis`。
![[Pasted image 20240120175400.png]]
排除redis默认使用lettuce客户端，引入jedis,解决堆外内存溢出
```XML 
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
	<exclusions>
		<exclusion>
			<groupId>io.lettuce</groupId>
			<artifactId>lettuce-core</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
</dependency>
```
