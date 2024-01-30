天下所有语言的执行过程分为两种：
- 编译型语言Compile：编译器，快，将源代码转换为机器码
- 解释型语言Interpretor：解释器，慢，因为每行代码都需要被解释执行

# Intercepter和Jit
但是Java是个比较特殊的语言，他是半编译半解释型语言，与之对应的既有jvm执行引擎Interpretor解释器，又有JIT即时编辑器：
- 默认jvm使用Interpretor解释器来逐行解释源代码（字节码）文件
- 当jvm探测到热点代码时，就会触发JIT即时编辑器，将热点源代码（字节码）翻译成机器码并存入热点缓存CodeCache中
Java代码默认解释执行，只有热点代码才会编译成机器码，见下图：
![[Pasted image 20240130132635.png]]
# Client Compiler和Server Compiler
JVM中集成了==两种JIT编译器，Client Compiler和Server Compiler==，它们的作用也不同。Client Compiler注重启动速度和局部的优化，Server Compiler则更加关注全局的优化，性能会更好，但由于会进行更多的全局分析，所以启动速度会变慢。两种编译器有着不同的应用场景，在虚拟机中同时发挥作用。
- HotSpot VM带有一个Client Compiler C1编译器。这种编译器启动速度快，但是性能比较Server Compiler来说会差一些。
- Hotspot虚拟机中使用的Server Compiler有两种：C2和Graal。
# Tiered Compiler分层编译技术
Java 7开始引入了分层编译(Tiered Compiler)的概念，它结合了C1和C2的优势，追求启动速度和峰值性能的一个平衡。执行java -version
可以看到mixed mode模式，也就是jdk默认支持分层编译
```shell
OpenJDK Runtime Environment Corretto-8.392.08.1 (build 1.8.0_392-b08)
OpenJDK 64-Bit Server VM Corretto-8.392.08.1 (build 25.392-b08, mixed mode)
```
分层编译将JVM的执行状态分为了五个层次。五个层级分别是：
- 解释执行。
- 执行不带profiling的C1代码。
- 执行仅带方法调用次数以及循环回边执行次数profiling的C1代码。
- 执行带所有profiling的C1代码。
- 执行C2代码。
profiling就是jvm执行引擎中的Profiler工具，能够收集能够反映程序执行状态的数据。其中最基本的统计数据就是方法的调用次数，以及循环回边的执行次数。
![[Pasted image 20240130135835.png]]

执行过程会在上述五个层级中递进或者回退，转换关系如下，转换关系也是五种：
- 图中第①条路径，代表编译的一般情况，热点方法从解释执行到被3层的C1编译，最后被4层的C2编译。
- 如果方法比较小（比如Java服务中常见的getter/setter方法），3层的profiling没有收集到有价值的数据，JVM就会断定该方法对于C1代码和C2代码的执行效率相同，就会执行图中第②条路径。在这种情况下，JVM会在3层编译之后，放弃进入C2编译，退回到1层的C1编译运行。
- 在C1忙碌的情况下，执行图中第③条路径，在解释执行过程中对程序进行profiling ，根据信息直接由第4层的C2编译。
- 前文提到C1中的执行效率是1层C1>2层C1>3层C1，第3层一般要比第2层慢35%以上，所以在C2忙碌的情况下，执行图中第④条路径。这时方法会被2层的C1编译，以减少方法在3层C1的执行时间。
- 如果编译器做了一些比较激进的优化，比如循环分支的预测，在实际运行时发现预测出错，这时就会进行反优化，重新进入解释执行，图中第⑤条执行路径代表的就是反优化。