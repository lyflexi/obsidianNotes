# 进程线程对比

进程和线程的主要差别在于它们是不同的操作系统资源管理方式，区别如下：

1. **内存空间：系统在运行的时候会为每个进程分配不同的内存空间；而对线程而言，除了CPU 外，系统不会为线程分配内存（线程所使用的资源来自其所属进程的资源），线程组之间只能共享资源；
    
2. **切换开销：进程和线程切换时,需要切换进程和线程的上下文，进程的上下文切换时间开销远远大于线程上下文切换时间,耗费资源较大,效率要差一些；
    
3. **依存关系：每个进程都有独立的程序运行入口、顺序执行序列和程序出口，但是线程不能够独立执行,必须依存在应用程序中,由应用程序提供多个线程执行控制；
    
4. 一个进程崩溃后,在保护模式下不会对其他进程产生影响，但是一个线程崩溃会影响整个进程都死掉。所以多进程要比多线程健壮。
    

jps全称Java Virtual Machine Process Status Tool，是Java提供的一个用来显示当前所有Java进程的命令。与之对应，linux 系统里也有一个ps命令

Java的每一个程序，均独占一个Java虚拟机实例，且都是一个独立的进程，每个进程都有自己的id。jps命令的位置在 JAVA_HOME/bin/ 下，jps命令常用参数如下：
- 无参数（ -V ） 默认显示 pid、应用程序 main class 类名
- -q：只显示 pid
- -m：显示 pid、应用程序`main class`类名和传递给main方法的参数，在嵌入式jvm上可能是null
- -l：显示pid 和应用程序`main class`的完整包名或者 应用程序的jar路径
- -v：显示pid 、应用程序`main class`类名和传递给 jvm 的参数
![[Pasted image 20231225133353.png]]

使用`man jps`可以查看 jps命令手册
![[Pasted image 20231225133358.png]]

# 线程协程对比

我们知道操作系统在线程等待IO的时候，会阻塞当前线程，切换到其它线程，这样在当前线程等待IO的过程中，其它线程可以继续执行。当系统线程较少的时候没有什么问题，但是当线程数量非常多的时候，却产生了问题。

- 一是系统线程会占用非常多的内存空间，
    
- 二是过多的线程切换会占用大量的系统时间
![[Pasted image 20231225133408.png]]

协程刚好可以解决上述2个问题。协程运行在线程之上，当一个协程执行完成后，可以选择主动让出，让另一个协程运行在当前线程之上。
- ==协程并没有增加线程数量，只是在线程的基础之上通过分时复用的方式运行多个协程。==
- ==协程的切换在用户态完成，切换的代价比线程从用户态到内核态的代价小很多。==
![[Pasted image 20231225133420.png]]

我们只需要启动100个线程，每个线程上运行100个协程，这样不仅减少了线程切换开销，而且还能够同时处理10000个读取数据库的任务，很好的解决了上述任务。

==需要注意的是协程只有和异步IO结合起来，才能发挥最大的威力。==

假设协程调用了一个阻塞IO操作，这时候会发生什么？实际上操作系统并不知道协程的存在，它只知道线程，因此在协程调用阻塞IO操作的时候，操作系统会让线程进入阻塞状态，当前的协程和其它绑定在该线程之上的协程都会陷入阻塞而得不到调度，这往往是不能接受的。因此在协程中最好不要调用导致线程阻塞的操作。

# Thread用法

```Java
    Thread01 thread = new Thread01();
    thread.start();//启动线程
    // 继承Thread
    public static class T1 extends Thread{
        @Override
        public void run(){
            System.out.println("当前线程："+Thread.currentThread().getId());
            System.out.println("T1");
        }

    }
```

## 插队方法join()

join方法是Thread类中的一个方法，该方法的定义是等待该线程执行直到终止。其实就说join方法将挂起当前线程的执行，直到调用join方法的线程执行完成。

### join阻塞子线程的情况

join实例（以一道面试题为例）：现在有T1、T2、T3三个线程，你怎样保证T2在T1执行完之后执行，T3在T2执行完后执行？

这个问题是网上很热门的面试题（这里除了join还有很多方法能够实现，只是使用join是最简单的方案），下面是实现的代码：

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

### join阻塞主线程的情况

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

# Runnable接口用法

```Java
// 构造注入Runnable接口
Runable01 runable01 = new Runable01();
new Thread(runable01).start();
// 实现Runnable接口
public static class R1 implements Runnable{
    @Override
    public void run(){
        System.out.println("当前线程："+Thread.currentThread().getId());
        System.out.println("R1");
    }

}
```

# `Callable<T>`接口用法

```Java
    // 实现Callable接口 （可以拿到返回结果，可以处理异常）
    public static class C1 implements Callable<Integer> {

        @Override
        public Integer call() throws Exception {
            System.out.println("当前线程："+Thread.currentThread().getId());
            System.out.println("C1");
            return 66;
        }
    }
```


Runnable自 Java 1.0 以来一直存在，但Callable仅在 Java 1.5 中引入，目的就是为了来处理Runnable不支持的用例。

- **Runnable 接口**不会返回结果或抛出检查异常
    
- 但是**Callable 接口**可以返回结果或抛出检查异常。
    
- 工具类 Executors 可以实现 Runnable 对象和 Callable 对象之间的相互转换。（Executors.callable（Runnable task）或 Executors.callable（Runnable task，Object resule））。
    

所以，如果任务不需要返回结果或抛出异常推荐使用 **Runnable 接口**，这样代码看起来会更加简洁。

# FutureTask用法

FutureTask三功能合一：

- 实现了Runnable接口
    
- 实现了Future接口
    
- 构造注入了`Callable<T>`，提供了Callable功能，FutureTask往往搭配Callable接口使用

![[Pasted image 20231225133626.png]]

示例用法

```Java
//实现Callable接口 + FutureTask （可以拿到返回结果，可以处理异常）
FutureTask<Integer> futureTask = new FutureTask<>(new Callable01());
new Thread(futureTask).start();
//阻塞等待整个线程执行完成，获取返回结果
Integer integer = futureTask.get();    
```