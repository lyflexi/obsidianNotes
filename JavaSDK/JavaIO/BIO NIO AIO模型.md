阻塞IO（BIO）、同步非阻塞 IO（NIO）、异步非堵塞的 IO（AIO）：

- BIO 就是传统的`java.io`包，它是基于流模型实现的，交互的方式是同步、阻塞方式，也就是说在读入输入流或者输出流时，在读写动作完成之前，线程会一直阻塞在那里，它们之间的调用时可靠的线性顺序。它的有点就是代码比较简单、直观；缺点就是 IO 的效率和扩展性很低，容易成为应用性能瓶颈。
    
- NIO 是Java 1.4引入的`java.nio`包，提供了`Channel`、`Selector`、`Buffer`等新的抽象，可以构建多路复用的、同步非阻塞IO程序，同时提供了更接近操作系统底层高性能的数据操作方式。
    
- AIO 是 Java1.7之后引入的包，是NIO的升级版本，提供了异步非堵塞的IO操作方式，所以人们叫它 AIO（Asynchronous IO），异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。
    

# 阻塞（Blocking IO）

1. 应用程序想要读取数据就会调用recvfrom
    
2. recvfrom会通知OS来执行，OS就会判断数据报是否准备好(比如判断是否收到了一个完整的UDP报文，如果收到UDP报文不完整，那么就继续等待)。
    
3. 当数据包准备好了之后，OS就会将数据从内核空间拷贝到用户空间(因为我们的用户程序只能获取用户空间的内存，无法直接获取内核空间的内存)。
    
4. 拷贝完成之后socket.read()就会解除阻塞，并得到read的结果。
![[Pasted image 20231225131700.png]]

阻塞发生在两个地方：

- OS等待数据报准备好。
    
- 将数据从内核空间拷贝到用户空间。
    

多线程情况下，JavaBIO明确线程是较为重量级的资源：

- 当并发量大，而后端服务或客户端处理数据慢过时就会产生产生大量线程处于等待中，即上述的阻塞，是非常严重的资源浪费。
    
- 此外，线程的切换也会导致cpu资源的浪费，单机内存限制也无法过多的线程，只能单向以流的形式读取数据。

![[Pasted image 20231225131713.png]]

```Java
public class ServerTcpSocket {
    static byte[] bytes = new byte[1024];

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        try {
            // 1.创建一个ServerSocket连接
            final ServerSocket serverSocket = new ServerSocket();
            // 2.绑定端口号
            serverSocket.bind(new InetSocketAddress(8080));
            // 3.当前线程放弃cpu资源等待获取数据
            System.out.println("等待获取数据...");
            while (true) {
                final Socket socket = serverSocket.accept();
                executorService.execute(new Runnable() {
                    public void run() {
                        try {
                            System.out.println("获取到数据...");
                            // 4.读取数据
                            int read = socket.getInputStream().read(bytes);
                            String result = new String(bytes);
                            System.out.println(result);
                        } catch (Exception e) {

                        }
                    }
                });

            }
        } catch (Exception e) {

        }
    }
}
```

# 非阻塞（Non-Blocking IO）

非阻塞I/O 不管是否有获取到数据，都会立马获取结果，如果没有获取数据的话、那么就不间断的循环重试，但是我们整个应用程序不会实现阻塞。

当我设置非阻塞后，我们的socket.read()方法就会立即得到一个返回结果(成功 or 失败)，我们可以根据返回结果执行不同的逻辑，比如在失败时，我们可以做一些其他的事情。但事实上这种方式也是低效的，因为我们不得不使用轮询方法区一直问OS：“我的数据好了没啊”。也就是说NIO 不会在recvfrom也就是socket.read()时候阻塞，但是还是会在将数据从内核空间拷贝到用户空间阻塞。一定要注意这个地方，Non-Blocking还是会阻塞的
![[Pasted image 20231225131722.png]]

多线程情况下，JavaNIO有三大组成部分：`Buffer`、`Channel`、`Selector`，通过事件驱动模式实现了什么时候有数据可读的问题。

- Buffer：缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。用于和Channel通道进行双向交互。如你所知，数据是从通道读入缓冲区，从缓冲区写入到通道中的。
    
- Channel：有ServerSocketChannel、SocketChannel、FileChannel、DatagramChannel，只有FileChannel无法设置成非阻塞模式，其他Channel都可以设置为非阻塞模式。
    
- Selector：是Java NIO中能够检测一到多个NIO通道Channel的选择器，通道将关心的事件注册到Selector上，Selector能够知晓通道是否为这些事件诸如读写事件做好数据准备。这样，一个单独的后台线程可以管理多个channel，从而管理多个网络连接。
![[Pasted image 20231225131731.png]]

NIO使用单线程或者只使用少量的多线程，多个连接共用一个线程，消耗的线程资源会大幅减小

```Java
public class ServerNioTcpSocket {
    static ByteBuffer byteBuffer = ByteBuffer.allocate(512);

    public static void main(String[] args) {
        try {
            // 1.创建一个ServerSocketChannel连接
            final ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            // 2.绑定端口号
            serverSocketChannel.bind(new InetSocketAddress(8080));
            // 设置为非阻塞式
            serverSocketChannel.configureBlocking(false);
            // 非阻塞式
            SocketChannel socketChannel = serverSocketChannel.accept();
            if (socketChannel != null) {
                int j = socketChannel.read(byteBuffer);
                if (j > 0) {
                    byte[] bytes = Arrays.copyOf(byteBuffer.array(), byteBuffer.limit());
                    System.out.println("获取到数据" + new String(bytes));
                }
            }
            System.out.println("程序执行完毕..");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# 异步I/O(Asynchronous IO)

Asynchronous IO调用中是真正的无阻塞，其他IO model中多少会有点阻塞。

1. 程序发起read操作之后，立刻就可以开始去做其它的事。
    
2. 而在内核角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。
    
3. 然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，
    
4. 当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。

![[Pasted image 20231225131742.png]]