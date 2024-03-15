# Thread启动线程的三种方式
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
- 方式1：继承Thread重写run方法，最后执行Thread.start
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
