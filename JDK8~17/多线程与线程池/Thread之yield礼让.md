yield//让当前线程让出部分时间片，相当于在cpu紧张的情况下当前线程的优先级降低
```java
package org.lyflexi.thread.yield;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/14 10:23  
 */public class YieldTest {  
    public static void main(String[] args) {  
        Runnable task1 = () -> {  
            int count = 0;  
            for (;;) {  
                System.out.println("---->1 " + count++);  
            }  
        };  
        Runnable task2 = () -> {  
            int count = 0;  
            for (;;) {  
                Thread.yield();//让当前的task2让出部分时间片，相当于在cpu紧张的情况下当前task2的优先级降低  
                System.out.println("              ---->2 " + count++);  
            }  
        };  
        Thread t1 = new Thread(task1, "t1");  
        Thread t2 = new Thread(task2, "t2");  
//        t1.setPriority(Thread.MIN_PRIORITY);  
//        t2.setPriority(Thread.MAX_PRIORITY);  
        t1.start();  
        t2.start();  
    }  
}
```
打印信息如下：线程2的计算结果明显小于线程1
```shell
。。。
---->1 350439
---->1 350440
---->1 350441
              ---->2 106484
              ---->2 106485
---->1 350442
---->1 350443
---->1 350444
。。。
```

