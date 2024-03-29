之前我们一直在讨论NIO的网络Socket通信，还剩下一种channel没有讲，那就是 FileChannel，用来读写磁盘文件


传统的 IO 将一个文件通过 socket 写出要经历如下步骤
```java
File f = new File("helloword/data.txt");
RandomAccessFile file = new RandomAccessFile(file, "r");

byte[] buf = new byte[(int)f.length()];
//通过fileChannel将文件写入buffer
file.read(buf);

Socket socket = ...;
socket.getOutputStream().write(buf);
```
java 本身并不具备 IO 读写能力，因此 read 方法调用后，要从 java 程序的用户态切换至内核态，去调用操作系统（Kernel）的读能力，将数据读入内核缓冲区。这期间用户线程阻塞，上述代码的内部工作流程是这样的：
![[0024.png]]
1. DMA：操作系统使用 DMA（Direct Memory Access）来实现读磁盘文件，其间也不会使用 cpu（DMA 也可以理解为硬件单元，用来解放 cpu 完成文件 IO）
2. 从内核态切换回用户态，将数据从内核缓冲区读入用户缓冲区，即用户buffer（byte[] buf），==这期间 cpu 会参与拷贝==，无法利用 DMA
3. 调用 write 方法，这时将数据从用户缓冲区（byte[] buf）写入 socket 缓冲区，==cpu 会参与拷贝==
4. DMA：接下来要向网卡写数据，这项能力 java 又不具备，因此又得从用户态切换至内核态，调用操作系统的写能力，使用 DMA 将 socket 缓冲区的数据写入网卡，不会使用 cpu
可以看到DMA已经解放了部分的CPU操作，但是中间环节仍然比较多较多
- 用户态与内核态的切换发生了 3 次，这个操作比较重量级
- 数据拷贝了共 4 次
- 操作用户缓冲区需要cpu参与两次


如何提升IO的性能呢？

# mmap()优化-rocketmq
mmap（memory map）是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。简单地说就是内核缓冲区和应用缓冲区进行映射，用户在操作应用缓冲区时就好像在操作内核缓冲区。

NIO通过 DirectByteBuf实现了mmap()的效果， 将堆外内存映射到 jvm 内存中来直接访问使用，减少了一次数据拷贝：
- ByteBuffer.allocate(10) HeapByteBuffer 使用的还是 jvm 堆内存
- ByteBuffer.allocateDirect(10) DirectByteBuffer 使用的是操作系统内存（堆外内存）

![[0025.png]]
DirectByteBuffer内存的特点（了解）：
- 这块内存不受 jvm 垃圾回收的影响，因此内存地址固定，有助于 IO 读写
- java 中的 DirectByteBuf 对象仅维护了此内存的虚引用，内存回收分成两步
    - DirectByteBuf 对象被垃圾回收，将虚引用加入引用队列
    - 通过专门线程访问引用队列，根据虚引用释放堆外内存
最后，即使使用了DirectByteBuf，并没有减少用户态与内核态的切换次数，至此：
- 用户态与内核态的切换发生了 3 次
- 数据拷贝了共 3 次
# linux 2.1优化- sendFile
进一步优化，底层采用了 linux 2.1 后提供的 sendFile 方法，java 中对应着两个 channel 调用 transferTo/transferFrom 方法拷贝数据：具体流程如下：
1. java 调用 transferTo 方法后，要从 java 程序的用户态切换至内核态，使用 DMA将数据读入内核缓冲区，不会使用 cpu
2. 数据从内核缓冲区传输到 socket 缓冲区，cpu 会参与拷贝
3. 最后使用 DMA 将 socket 缓冲区的数据写入网卡，不会使用 cpu
![[0026.png]]
至此：
- 只发生了一次用户态与内核态的切换
- 数据拷贝了 3 次，此时实现了零拷贝
==零拷贝，零拷贝在操作系统中还是会有拷贝的，“零”只是体现操作系统内核不再需要拷贝buffer到用户程序（jvm）了==
# linux 2.4优化- sendFile+SG-DMA

进一步优化（linux 2.4），进一步引入了SG-DMA
![[0027.png]]
1. java 调用 transferTo 方法后，要从 java 程序的用户态切换至内核态，使用 DMA将数据读入内核缓冲区，不会使用 cpu
2. 只会将一些 offset 和 length 信息拷入 socket 缓冲区，几乎无消耗
3. 使用 SG-DMA 将 内核缓冲区的数据跳过socket缓冲区，直接写入网卡，不会使用 cpu
可以通过以下命令查看网卡是否支持scatter-gather特性：
![[Pasted image 20231225132819.png]]
整个过程仅只发生了一次用户态与内核态的切换，数据拷贝了 2 次。所谓的【零拷贝】，并不是真正无拷贝，而是在不会拷贝重复数据到 jvm 内存中，零拷贝的优点有
- 更少的用户态与内核态的切换
- 全程不利用 cpu 计算，减少 cpu 缓存伪共享
- 零拷贝适合小文件传输