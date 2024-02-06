# Thread启动线程的两种方式
在jdk中，Thread类本来就实现了Runnable接口，同时又通过构造注入依赖了Runnable对象
```java
public class Thread implements Runnable {
	...
	/* What will be run. */  
	private Runnable target;
	
	public Thread(String name) {  
	    this(null, null, name, 0);  
	}
	
	public Thread(Runnable target, String name) {  
	    this(null, target, name, 0);  
	}

	/**  
	 * If this thread was constructed using a separate * {@code Runnable} run object, then that  
	 * {@code Runnable} object's {@code run} method is called;  
	 * otherwise, this method does nothing and returns. * <p>  
	 * Subclasses of {@code Thread} should override this method.  
	 * * @see     #start()  
	 * @see     #stop()  
	 * @see     #Thread(ThreadGroup, Runnable, String)  
	 */
	@Override  
	public void run() {  
	    if (target != null) {  
	        target.run();  
	    }  
	}
	public synchronized void start() {  
		if (threadStatus != 0)  
			throw new IllegalThreadStateException();  
		group.add(this);  
		
		boolean started = false;  
		try {  
			start0();  //本地方法，应该就是请求CPU分配时间片
			started = true;  
		} finally {  
			try {  
				if (!started) {  
					group.threadStartFailed(this);  
				}  
			} catch (Throwable ignore) {  
			
			}  
		}  
	}
}
```
因此通过看Thread类的源码，Thread启动线程就有两种方式，但是两种方式都是要通过Thread.start()来启动线程
- 方式1：继承Thread重写run方法，最后执行Thread,.start
- 方式2 ：构造Thread并传入Runnable，最后执行Thread,.start
方式1：继承Thread重写run方法，最后执行Thread,.start
```Java
// 方式1：继承Thread类
package org.lyflexi.thread;  
  
/**  
 * @Author: ly  
 * @Date: 2024/2/6 13:58  
 */public class ExtendsThread extends Thread {  
  
    @Override  
    public void run() {  
        System.out.println("线程运行中...");  
    }  
  
  
    public static void main(String[] args) {  
        ExtendsThread extendsThread = new ExtendsThread();  
        extendsThread.start();  
    }  
  
}


```
方式2 ：构造Thread并传入Runnable，最后执行Thread,.start
```java
package org.lyflexi.thread;  
  
/**  
 * @Author: ly  
 * @Date: 2024/2/6 13:59  
 */public class DependRunnable {  
    public static void main(String[] args) {  
        Thread lambdaThread = new Thread(() -> System.out.println("使用lambda表达式创建的线程运行中..."));  
        lambdaThread.start();  
    }  
}
```
注意：Thread千万不能通过直接调用run()方法来启动线程！否则将失去创建线程的效果
```java
public class ExtendThreadDemo extends Thread {
    //定一个线程的名称
    private String name;

    //带参数的构造方法
    ExtendThreadDemo(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        System.out.println("[ExtendThreadDemo]我是线程：ExtendThreadDemo-" + name);
    }

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            ExtendThreadDemo threadDemo = new ExtendThreadDemo(String.valueOf(i));
            threadDemo.run();
        }
    }
}

```
打印如下：
```shell
[ExtendThreadDemo]我是线程：ExtendThreadDemo-0
[ExtendThreadDemo]我是线程：ExtendThreadDemo-1
[ExtendThreadDemo]我是线程：ExtendThreadDemo-2
[ExtendThreadDemo]我是线程：ExtendThreadDemo-3
[ExtendThreadDemo]我是线程：ExtendThreadDemo-4
```
这里线程打印结果的顺序是我们同步启动线程的顺序(无论试多少次都是这个结果)，这说明线程没有异步启动，而是同步执行了里面的run()方法而已。
方式3：创建一个带返回结果的线程任务FutureTask：
其实方式3本质上也称作方式2，因为FutureTask是Runnable接口的实现类
![[Pasted image 20240206140336.png]]
给Thread传入FutureTask之前，需要额外使用Callable接口构造出FutureTask
```java
package org.lyflexi.thread;  
  
import java.util.concurrent.Callable;  
import java.util.concurrent.ExecutionException;  
import java.util.concurrent.FutureTask;  
  
/**  
 * @Author: ly  
 * @Date: 2024/2/6 14:00  
 */public class DependFutureTask {  
    public static void main(String[] args) {  
        //3.创建Callable接口实现类的对象  
        MyCallable mc = new MyCallable();  
        //4.将此Callable接口实现类的对象作为传递到FutureTask构造器中，创建FutureTask的对象  
        FutureTask ft = new FutureTask<>(mc);  
        //5.将FutureTask的对象作为参数传递到Thread类的构造器中，创建Thread对象，并调用start()  
        new Thread(ft).start();  
  
        System.out.println("主线程在执行任务");  
  
        //6.get()接收返回值  
        try {  
            Object sum = ft.get();  
            System.out.println("和为："+sum);  
        } catch (ExecutionException | InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
  
        System.out.println("所有任务执行完毕");  
    }  
    //1.创建一个实现Callable的实现类  
    static class MyCallable implements Callable {  
        //2.实现call方法，将此线程需要执行的操作声明在call()中  
        @Override  
        public Object call() throws Exception {  
            System.out.println("子线程在进行计算");  
            int sum = 0;  
            for (int i = 0;i < 5; i++) {  
                sum = sum + i;  
            }  
            return sum;  
        }  
    }  
}
```
打印结果：
```shell
主线程在执行任务
子线程在进行计算
和为：10
所有任务执行完毕
```
# Thread的线程同步方法join()
通过Thread同时创建多个线程时，由于各个子线程的任务耗时不同，各个子线程的完成时间一般是不可预知的，那如何同步各个线程的执行先后顺序呢？

join方法是Thread类中的一个方法，调用t.join()将挂起当前线程的执行，直到引用线程t执行完成。
## join阻塞子线程的情况

join实例（以一道面试题为例）：现在有T1、T2、T3三个线程，你怎样保证T2在T1执行完之后执行，T3在T2执行完后执行？

这个问题是网上很热门的面试题（这里除了join还有很多方法能够实现比如CompletableFulture，只是使用join是最简单的方案），下面是实现的代码：

```Java
/**
 * @author wcc
 * @date 2021/8/21 20:46
 * 现在有T1、T2、T3三个线程，你怎样保证T2在T1执行完后执行，T3在T2执行完后执行？
 */
public class JoinDemo {

    public static void main(String[] args) {
        //初始化线程1，由于后续有匿名内部类调用这个局部变量，需要用final修饰
        //这里不用final修饰也不会报错的原因 是因为jdk1.8对其进行了优化
        /*
        在局部变量没有重新赋值的情况下，它默认局部变量为final类型，认为你只是忘记了加final声明了而已。
        如果你重新给局部变量改变了值或者引用，那就无法默认为final了
         */
        Thread t1=new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("t1 is running...");
            }
        });

        //初始化线程二
        Thread t2=new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t1.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("t2 is running...");
                }
            }
        });

        //初始化线程三
        Thread t3=new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t2.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("t3 is running...");
                }
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }

}
输出：
t1 is running...
t2 is running...
t3 is running...
```

## join阻塞主线程的情况

在很多情况下，主线程创建并启动子线程，如果子线程中要进行大量的耗时运算，主线程将可能早于子线程结束。如果主线程需要知道子线程的执行结果时，就需要等待子线程执行结束了。主线程可以sleep(xx),但这样的xx时间不好确定，因为子线程的执行时间不确定，join()方法比较合适这个场景。

代码如下：
```Java
public static void main(String[] args) {
    Thread t = new Thread(() -> System.out.println("test"));
    t.start();
    System.out.println("main1");
    try {
        t.join();
    } catch (Exception e) {
        e.printStackTrace();
    }
    System.out.println("main2");
}
输出：
main1
test
main2
OR
test
main1
main2
```

main1和test虽然会随机输出，但join方法后的代码（输出main2），一定会在线程t的run方法输出test之后执行。