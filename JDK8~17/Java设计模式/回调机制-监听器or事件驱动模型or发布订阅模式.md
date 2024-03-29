>在计算机程序设计中，回调函数是指通过函数参数传递到其它代码的某一块可执行代码的引用，换句话说回调函数其实就是一个被作为参数传递的函数，==这一设计允许了底层代码在合适的时机调用在高层定义的子程序。==
> - 回调函数可以用于中断处理，操作系统在触发中断的时候执行用户传入的回调函数

当程序跑起来时，一般情况下，应用程序（application program）会时常通过API调用库里所预先备好的函数。

但是有些库函数（library function）却要求应用先传给它一个函数，好在合适的时候调用，以完成目标任务。

这个被传入的、后又被调用的函数就称为回调函数（callback function）


# C语言中的回调实现

在C程序设计中，回调函数是通过函数指针实现的。函数指针其实就是一个变量，这个变量存放的是什么呢？存放的是一个函数的入口地址；

它的定义方式为：`函数返回值类型 （*指针变量名）（函数参数.......）;`，在一般情况下，我们为了方便通常会使用typedef关键字，比如说

```C
typedef int （*ptrfunction）(int,int);
ptrfunction  func;//在这里func就是一个函数指针，函数的返回值为int类型，且有两个int类型的参数；
```

当然我们现在只是定义这个函数指针，还尚未给它赋值

下面是函数指针的示例用法，将函数指针作为回调函数的形参
![[Pasted image 20231225131159.png]]

假设main函数就是系统底层，那么系统底层就会在合适的时机调用`Register_CallbackFunc`，并执行用户传入的函数`add`

# Java语言中的回调实现

用户先自定义个回调接口：ICallback ，用于向系统底层传递“函数指针”

```Java
package callback;
import java.util.Map;
/*用户自定义的回调接口*/
public interface ICallback {

    public void callback(Map<String, Object> params);
}
```

## Java实现异步回调

Java语言回调实现一般是异步处理的一种技术。下面模拟了主线程传递回调接口，给系统底层的`AsyncNettyTest.doStm`，并由`AsyncNettyTest.doStm`线程结束的时候执行接收的回调函数

```Java
package callback;
 
import java.util.HashMap;
import java.util.Map;  
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;
public class AsyncNettyTest{
  
    static ExecutorService es = Executors.newFixedThreadPool(2);  
  
    public static void doStm(final ICallback callback) {  
        // 初始化一个线程  
        Thread t = new Thread() {  
            public void run() {  
  
                // 这里是业务逻辑处理  
                System.out.println("子线任务执行:"+Thread.currentThread().getId());
  
                // 为了能看出效果 ，让当前线程阻塞5秒  
                try {  
                    Thread.sleep(1000);
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
  
                // 处理完业务逻辑，  
                Map<String, Object> params = new HashMap<String, Object>();
                params.put("a1", "这是我返回的参数字符串...");
                callback.callback(params);
            };  
        };  
  
        es.execute(t);
        //一定要调用这个方法，不然executorService.isTerminated()永远不为true
        es.shutdown();
    }
  
    public static void main(String[] args) {  
        doStm((params)->{
            System.out.println("单个线程也已经处理完毕了，返回参数a1=" + params.get("a1"));
        });
  
        System.out.println("主线任务已经执行完了:"+Thread.currentThread().getId());
    }  
}  
```

执行结果，可以看到主线程在子线程之前就结束了，证明了异步性

```Java
子线任务执行:11
主线任务已经执行完了:1
单个线程也已经处理完毕了，返回参数a1=这是我返回的参数字符串...
 
Process finished with exit code 0
```

## Java实现同步回调

Java在java.util.concurrent包中附带了Future接口，给线程池的`submit`方法传递一个JDK自带的`Callable`的回调接口，线程池的`submit(Callable<T> task)`方法将会返回一个回调的Future，Futures是一个抽象的概念，它表示一个值，该值可能在某一点变得可用。一个Future要么获得计算完的结果，要么获得计算失败后的异常。

线程池execute()和 submit()的区别

1. `void execute(Runnable command);`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；
2. ==`<T> Future<T> submit(Callable<T> task);`==方法用于提交需要返回值的任务。线程池会返回一个 Future 类型的对象，通过这个 Future 对象可以判断任务是否执行成功，并且可以通过 Future 的 get()方法来获取返回值
    1. get()方法会阻塞当前线程直到任务完成，
    2. get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

```Java
package callback;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.*;

public class SyncNettyTest {
    private static ExecutorService es = Executors.newFixedThreadPool(2);


    public static void sysFunc(final ICallback callback) {
        Callable<Netty> c = new Callable<SyncNettyTest.Netty>() {

            @Override
            public Netty call() throws Exception {
                System.out.println("子线任务执行:"+Thread.currentThread().getId());
                //这里是你的业务逻辑处理

                //让当前线程阻塞1秒看下效果
                Thread.sleep(1000);
                return new Netty("张三");
            }
        };

        //记得要用submit，执行Callable对象
        Future<Netty> fn = es.submit(c);
        //一定要调用这个方法，不然executorService.isTerminated()永远不为true
        es.shutdown();

        //无限循环等待任务处理完毕  如果已经处理完毕 isDone返回true
        while (!fn.isDone()) {
            try {
                //处理完毕后返回的结果
                Netty nt = fn.get();
                System.out.println("future获取返回值阻塞结束:"+nt.name);
                Map<String, Object> params = new HashMap<String, Object>();
                params.put("a1", "这是我返回的参数字符串...");
                callback.callback(params);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) {
        sysFunc((params)->{
            System.out.println("单个线程也已经处理完毕了，返回参数a1=" + params.get("a1"));
        });

        System.out.println("主线任务已经执行完了:"+Thread.currentThread().getId());
    }



    static class Netty {
        private String name;
        private Netty(String name) {
            this.name = name;
        }
    }

}
```

执行结果，可以看到主线程永远在子线程之后结束，证明了同步性

```Java
子线任务执行:22
future获取返回值阻塞结束:张三
单个线程也已经处理完毕了，返回参数a1=这是我返回的参数字符串...
主线任务已经执行完了:1

Process finished with exit code 0
```

# Spring框架中的回调机制-监听器/事件驱动模型/发布订阅模式

回调机制在Spring当中广泛的用于事件驱动模式的实现，比如监听器ApplicationListener接口（高层定义）中定义了回调函数`onApplicationEvent`，同时框架将Listener接口传递到了底层SimpleApplicationEventMulticaster，一旦有`ApplicationEvent`类型的事件发布时SimpleApplicationEventMulticaster（底层框架）就会触发doInvokeListener，就会执行Listener接口中的回调函数，该回调机制有个特殊的叫法：发布订阅模式
![[Pasted image 20231225131304.png]]