non-blocking io 非阻塞 IO，NIO的三大组件是：
- Channel
- Buffer
- Selector
# Channel & Buffer

channel 有一点类似于 stream，它就是读写数据的==双向交互通道==，可以从 channel 将数据读入 buffer，也可以将 buffer 的数据写入 channel，而之前的 stream 要么是输入，要么是输出，channel 比 stream 更为底层
![[1 1.png]]
常见的 Channel 有
- FileChannel：处理文件
- DatagramChannel：处理UDP
- SocketChannel：处理TCP
- ServerSocketChannel：处理TCP

buffer 则用来缓冲读写数据，常见的 buffer 有
- ByteBuffer
    - MappedByteBuffer
    - DirectByteBuffer
    - HeapByteBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer
- CharBuffer

# Selector

selector 单从字面意思不好理解，需要结合服务器的设计演化来理解它的用途
## 服务器多线程版设计
多线程版缺点：
- 内存占用高，每开一个线程都是需要消耗内存的，比如开一个线程需要1M，那么开1000个线程就需要1G内存
- 线程上下文切换成本高，线程的最终执行需要依靠CPU，当CPU的性能不足以同时处理比如1前个线程的时候，其余线程就会等待执行，这就是线程的上下文切换成本
因此服务器多线程版设计只适用于连接数比较少的场景
![[1.png]]
## 服务器线程池版设计
线程池版缺点，虽然线程池中的线程实现了复用，但是在当前socket连接断开之前，即使该socket连接没有任何读写请求，线程也不能去处理其他socket，线程仅能处理当前者一个socket 连接，

因此线程池版的设计也仅适合短连接场景，比如HTTP连接，来一个请求执行结束就立马断开，不过多的霸占线程资源
![[1 2.png]]
## 服务器selector版设计
selector 的作用就是配合一个线程来管理多个 channel，获取这些 channel 上发生的事件，==这些 channel 工作在非阻塞模式下，不会让线程吊死在一个 channel 上。==
![[1 3.png]]
调用 selector 的 select() 会阻塞直到 channel 发生了读写就绪事件，这些事件一旦发生，select 方法就会返回这些事件交给 thread 来处理

# ByteBuffer 内存结构
使用ByteBuffer的正确姿势
1. 向 buffer 写入数据，例如调用 channel.read(buffer)或者buffer.put(byte b)
2. 调用 flip() 切换至**读模式**，然后调用 buffer.get() 从buffer中读取数据
3. 调用 clear() 或 compact() 切换至**写模式**
4. 重复 1~3 步骤
流程演示，初始状态给buffer分配10个字节的内存空间，Bytebuffer buf = ByteBuffer.allocate(10);
![[0021.png]]
调用 channel.read(buffer)，下图表示写入了 4 个字节后的状态，position 是写入后的位置，limit 等于容量，
![[0018.png]]
flip 动作发生后，position 切换为读取位置，limit 切换为读取限制
![[0019.png]]
调用 buffer.get()四次，从头开始读取 4 个字节后的buffer状态
![[0020.png]]
clear 动作发生后，状态如下
![[0021 1.png]]
compact 方法，是把读取完的部分清空，未读完的部分向前移动到头，然后切换至写模式
![[0022.png]]