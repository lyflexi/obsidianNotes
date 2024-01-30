当今Java应用面临启动速度慢的问题：大型云平台要求每一种应用都必须秒级启动。每个应用都要求效率高。而jar包默认解释执行，只有热点代码才编译成机器码；因此jar包初始启动速度慢，初始能够应对的处理请求数量少。

于是GraalVM营运而生，GraalVM是一个高性能的JDK，同时还提供JavaScript、Python和许多其他流行语言的支持。GraalVM带来了AOT（Ahead Of Time）技术，旨在加速用Java和其他JVM语言编写的应用程序的执行。终于，Java应用也能提前被编译成机器码，随时急速启动，一启动就急速运行，达到最高性能
![[Pasted image 20240130144626.png]]
==AOT的产物是原生镜像native-image，即适配本机平台的可执行文件（机器码、本地镜像）==。因此，服务器无需再单独安装Java环境，可以在当前平台直接运行。

GraalVM提供了两种运行Java应用程序的方式：
1. 半替换：使用Graal替换HotSpot JVM内置的C1/C2即时编译器（JIT），具体针对的是C2编译器，而解释器Interpreter依然保留
2. 全替换：作为预先编译的本机可执行文件运行，这就是AOT（Ahead Of Time，本地镜像）。这才是最佳实践

环境安装：Windows平台需要安装
- VisualStudio提供集成环境
- GraalVM + native-image工具
环境安装：Linux平台需要安装
- gcc，glibc-devel，zlib-devel提供集成环境
- GraalVM + native-image工具
环境安装：Mac平台需要安装
- Xcode提供集成环境
- GraalVM + native-image工具

参照文档：https://www.yuque.com/leifengyang/springboot3/xy9gqc2ezocvz4wn#uCZ7T
