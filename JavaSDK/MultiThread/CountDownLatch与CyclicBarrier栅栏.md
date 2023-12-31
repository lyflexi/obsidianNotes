![[Pasted image 20231225161409.png]]

# CountDownLatch原理介绍

CountDownLatch(闭锁) 做减法，主要涉及两个方法：

- countDown()，当其它线程调用countDown方法会将计数器减1，但是不会阻塞
    
- await()，当前线程调用await方法时会被阻塞，等待计数器的值变为0时，当前线程才被唤醒继续执行
    

示例代码如下：

```Java
//需求:要求6个线程都执行完了,mian线程最后执行
public class CountDownLatchDemo {
    public static void main(String[] args) throws Exception{

        CountDownLatch countDownLatch=new CountDownLatch(6);
        for (int i = 1; i <=6; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"\t");
                countDownLatch.countDown();
            },i+"").start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName()+"\t班长关门走人,main线程是班长");
    }
}
```

CountDownLatch是基于AQS实现的，它的实现机制很简单：

- 当我们在构建CountDownLatch对象时，传入的值其实就会赋值给 AQS 的关键变量state
    
- 执行CountDownLatch的countDown方法时，其实就是利用CAS 将state 减一
    
- 执行await方法时，其实就是判断state是否为0
    
    - 如果state不为0，则将调用await方法的当前线程加入到阻塞队列中，将该线程阻塞掉（除了头结点）
        
    - AQS头节点会一直自旋等待state为0，当state为0时，头节点把剩余的在队列中阻塞的节点也一并唤醒。
        

# CyclicBarrier原理介绍

回到CyclicBarrier上，代码也不难，只有await方法。从源码不难发现的是：

- 它没有像CountDownLatch和ReentrantLock使用AQS的state变量，而是使用CyclicBarrier内部维护的内部维护count变量
    
- 同时CyclicBarrier借助ReentrantLock加上Condition实现等待唤醒的功能
    

## parties变量和condition队列

在构建CyclicBarrier时，传入的值是parties变量，同时也会赋值给CyclicBarrier内部维护count变量（这是可以复用的关键）

```Java
//parties表示屏障拦截的线程数量，当屏障撤销时，先执行barrierAction，然后在释放所有线程
public CyclicBarrier(int parties, Runnable barrierAction)
//barrierAction默认为null
public CyclicBarrier(int parties)
```

1. 每次调用await时，会将count -1 ，操作count值是直接使用ReentrantLock来保证线程安全性
    
    1. 如果count不为0，则添加到condition队列中
        
    2. 如果count等于0时，则把节点从condition队列添加至AQS的队列中并进行全部唤醒，并且将parties的值重新赋值为count的值（实现复用）
        

## 阻塞任务线程而非主线程

CountDownLatch和CyclicBarrier都是线程同步的工具类。可以发现这两者的等待主体是不一样的。

- CountDownLatch调用await()通常是主线程
    
- CyclicBarrier调用await()是在任务线程调用的，所以，CyclicBarrier中的阻塞的是任务的线程，而主线程是不受影响的。
    

## 共同抵达时的执行操作

CyclicBarrier构造函数支持一个可选的Runnable barrierAction命令，在一组线程中的最后一个线程到达之后，但在释放所有线程之前执行一次barrierAction。（若在继续所有参与线程之前更新共享状态，此屏障操作很有用）

下面是CyclicBarrier的示例程序，一个寝室四个人约好了去球场打球

1. 由于四个人准备工作不同，所以约好在楼下集合
    
2. 四个人集合好之后且在四人线程释放之前，由CyclicBarrier触发一起出发去球场的动作。
    

```Java
    private static final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(4, 10, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
    //当拦截线程数达到4时，便优先执行barrierAction，然后再执行被拦截的线程。
    private static final CyclicBarrier cb = new CyclicBarrier(4, () -> System.out.println("寝室四兄弟一起出发去球场"));

    private static class MyThread extends Thread {
        private String name;
        public MyThread(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            System.out.println(name + "开始从宿舍出发");
            try {
                cb.await();
                //线程的具体业务操作
                TimeUnit.SECONDS.sleep(1);
                System.out.println(name + "从楼底下出发");
                TimeUnit.SECONDS.sleep(1);
                System.out.println(name + "到达操场");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        String[] str = {"李明", "王强", "刘凯", "赵杰"};
        for (int i = 0; i < 4; i++) {
            threadPool.execute(new MyThread(str[i]));
        }
        try {
            Thread.sleep(4000);
            System.out.println("四个人一起到达球场，现在开始打球");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
    }
输出：
王强开始从宿舍出发
刘凯开始从宿舍出发
李明开始从宿舍出发
赵杰开始从宿舍出发
寝室四兄弟一起出发去球场
李明从楼底下出发
刘凯从楼底下出发
王强从楼底下出发
赵杰从楼底下出发
赵杰到达操场
刘凯到达操场
王强到达操场
李明到达操场
四个人一起到达球场，现在开始打球
```

## 屏障复用

CyclicBarrier是可循环利用的屏障，顾名思义，说明该类创建的对象可以复用。

现在对CyclicBarrier进行复用…又来了一拨人，看看愿不愿意一起打：

```Java
    private static final ThreadPoolExecutor threadPool = new ThreadPoolExecutor(4, 10, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
    //当拦截线程数达到4时，便优先执行barrierAction，然后再执行被拦截的线程。
    private static final CyclicBarrier cb = new CyclicBarrier(4, () -> System.out.println("寝室四兄弟一起出发去球场"));

    private static class MyThread extends Thread {
        private String name;
        public MyThread(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            System.out.println(name + "开始从宿舍出发");
            try {
                cb.await();
                TimeUnit.SECONDS.sleep(1);
                System.out.println(name + "从楼底下出发");
                TimeUnit.SECONDS.sleep(1);
                System.out.println(name + "到达操场");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        String[] str = {"李明", "王强", "刘凯", "赵杰"};
        for (int i = 0; i < 4; i++) {
            threadPool.execute(new MyThread(str[i]));
        }
        try {
            Thread.sleep(4000);
            System.out.println("四个人一起到达球场，现在开始打球");
            System.out.println();
            System.out.println("现在对CyclicBarrier进行复用.....");
            System.out.println("又来了一拨人，看看愿不愿意一起打：");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        String[] str1= {"王二","洪光","雷兵","赵三"};
        for (int i = 0; i < 4; i++) {
            threadPool.execute(new MyThread(str1[i]));
        }
        try {
            Thread.sleep(4000);
            System.out.println("四个人一起到达球场，表示愿意一起打球，现在八个人开始打球");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

李明开始从宿舍出发
刘凯开始从宿舍出发
王强开始从宿舍出发
赵杰开始从宿舍出发
寝室四兄弟一起出发去球场
李明从楼底下出发
赵杰从楼底下出发
刘凯从楼底下出发
王强从楼底下出发
李明到达操场
赵杰到达操场
刘凯到达操场
王强到达操场
四个人一起到达球场，现在开始打球

现在对CyclicBarrier进行复用…
又来了一拨人，看看愿不愿意一起打：
王二开始从宿舍出发
洪光开始从宿舍出发
赵三开始从宿舍出发
雷兵开始从宿舍出发
寝室四兄弟一起出发去球场
雷兵从楼底下出发
赵三从楼底下出发
王二从楼底下出发
洪光从楼底下出发
洪光到达操场
赵三到达操场
王二到达操场
雷兵到达操场
四个人一起到达球场，表示愿意一起打球，现在八个人开始打球
```