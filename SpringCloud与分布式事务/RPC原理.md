> 我有新的感悟!
> 
> A场景: http网络数据传输，只是数据 
> B场景:方法调用，提供了计算
> C场景:A+B就是RPC!!!

RPC（Remote Procedure Call）是指远程过程调用，但是分布式服务器节点由于不在一个内存空间，不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。客户端机器A想要服务端机器B执行一个给定的任务（方法名），这个任务的执行环境（参数）由客户端A给出。任务的执行在服务端B发生，客户端A只关心任务结果并不关心执行过程。那么如何实现这个功能呢？

实现远程调用需要涉及：
- 服务端开辟线程（保持服务运行）、服务端反射（因为要通过注册的类名去实例化服务）
- 网络通信Socket、IO流、序列化
- 客户端动态代理dynamicProxy（获取服务的通用代理，1个动态代理可以代理任意服务端接口）
# 服务端

首先定义一个接口，表示服务端可以帮忙执行的任务列表

```Java
public interface HelloService {
    String hello(String name);
    String bye();
}
```

然后定义一个实现类，表示服务端真正提供的服务

```Java
public class HelloServiceImpl implements HelloService {
    @Override
    public String hello(String name) {
        return "Hello" + name;
    }
    @Override
    public String bye() {
        return "Bye Bye!";
    }
}
```

定义服务端进程，接收每一个客户端发来的请求，根据网络传入参数调用服务类对应的服务

```Java
package rpc;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RpcServer {
    private ExecutorService threadPool;
    private static final int DEFAULT_THREAD_NUM = 10;

    public RpcServer() { //构造函数生成一个线程池
        threadPool = Executors.newFixedThreadPool(DEFAULT_THREAD_NUM);
    }

    public void register(Object service, int port) { //服务端主动注册服务

        try {
            System.out.println("server starts...");
            // TCP socket_server
            ServerSocket server = new ServerSocket(port);
            Socket socket = null; //接收客户端发来的请求
            while ((socket = server.accept()) != null) {
                System.out.println("client connected");
                threadPool.execute(new Processor(socket, service)); //线程池拿出一个线程执行任务  任务一定要实现Runnable接口
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    class Processor implements Runnable {
        Socket socket;
        Object service;

        public Processor(Socket socket, Object service) {
            this.socket = socket;
            this.service = service;
        }

        public void process() {

        }

        @Override
        public void run() { //真正的服务
            try {
                // 解码获取客户端的请求参数信息
                ObjectInputStream in = new ObjectInputStream(socket.getInputStream());
                String methodName = in.readUTF(); // 获取方法名
                Class<?>[] parameterTypes = (Class<?>[]) in.readObject(); // 参数类型
                Object[] parameters = (Object[]) in.readObject(); // 参数
                // 从注册的service获取服务类，
                // 并且根据客户端传入的方法名以及参数调用服务类的相关方法
                Method method = service.getClass().getMethod(methodName, parameterTypes);                 
                try {
                    // 方法调用invoke
                    Object result = method.invoke(service, parameters);
                    // 初始化输出即返回值
                    ObjectOutputStream out = new ObjectOutputStream(socket.getOutputStream());
                    // 结果返回值，供客户端接收
                    out.writeObject(result);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
        }
    }
}
```

开启服务进程

```Java
package rpc;

public class Main {
    public static void main(String[] args) {
        HelloService helloService = new HelloServiceImpl();
        RpcServer server = new RpcServer();
        server.register(helloService, 50001);
    }
}
```

# 客户端
定义客户端进程，通过接口的代理向服务端发送任务请求

```Java
package rpc;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.net.Socket;

public class RpcClient {
    public static void main(String[] args) {
        // 生成接口的代理类对象 （代理类字节码文件是运行时生成的）
        HelloService helloService = getClient(HelloService.class, "127.0.0.1", 50001);
       
        // 代理类通过内部invoke方法调用接口方法hello,bye 
        System.out.println(helloService.hello("chenqihang"));
        System.out.println(helloService.bye());
    }

    @SuppressWarnings("unchecked")
    public static <T> T getClient(Class<T> clazz, String ip, int port) {
        return (T) Proxy.newProxyInstance(RpcClient.class.getClassLoader(), new Class<?>[]{clazz}, new InvocationHandler() {
            @Override
            public Object invoke(Object arg0, Method arg1, Object[] arg2) throws Throwable {
                    //连上服务端
                Socket socket = new Socket(ip, port);
                //客户端期望访问远程HelloService服务，因此将HelloService类信息编码发送到服务端
                ObjectOutputStream out = new ObjectOutputStream(socket.getOutputStream());
                
                out.writeUTF(arg1.getName()); //访问远程方法名：hello bye

                out.writeObject(arg1.getParameterTypes()); //发送方法参数类型

                out.writeObject(arg2);//发送方法参数：chenqihang " "
                /**
                Java反射是本地调用，HelloService服务在服务端的本地，因此
                arg1.invoke(service,args)需要在服务端被调用并返回结果给客户端
                */
                ObjectInputStream in = new ObjectInputStream(socket.getInputStream());                  

                return in.readObject();
            }
        });

    }
}
```

先开启服务端进程，再执行客户端进程。通过上述代码就实现了客户端进程访问服务端进程Say Hello和Say Bye的功能

# RPC开源框架

当然这只是RPC的一个小的执行框架，相信聪明的同学们已经发现似乎上述过程有些技术是多余的呢，比如代理技术，直接通过反射机制获取信息再传递到服务端似乎也能完成需要的功能，没错这是对的，但是抛弃动态代理的代码是不是通用性就降低了呢？一个好的RPC框架既需要满足需要的功能，还需要方便用户使用。

为了提高效率和方便使用国内外的互联网公司及研究机构做了大量的工作。
- Feign 来自SpringCloud
- Dubbo 来自阿里巴巴 http://dubbo.I/O/
- Motan 新浪微博自用 https://github.com/weibocom/motan
- Dubbox 当当基于 dubbo 的 https://github.com/dangdangdotcom/dubbox
- rpcx 基于 Golang 的 https://github.com/smallnest/rpcx
- Thrift from facebook https://thrift.apache.org
- Avro from hadoop https://avro.apache.org
- Finagle by twitter https://twitter.github.I/O/finagle
- gRPC by Google http://www.grpc.I/O (Google inside use Stuppy)
- Hessian from cuacho http://hessian.caucho.com
- Coral Service inside amazon (not open sourced)