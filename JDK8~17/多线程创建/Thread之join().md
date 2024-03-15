# Thread的线程同步方法join()
通过Thread同时创建多个线程时，由于各个子线程的任务耗时不同，各个子线程的完成时间一般是不可预知的，那如何同步各个线程的执行先后顺序呢？

join方法是Thread类中的一个方法，join的底层源码也是wait，因此调用t.join()将挂起当前线程的执行，直到引用线程t执行完成。
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