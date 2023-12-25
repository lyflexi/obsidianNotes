信号量Semaphore 可以用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用资源。

通常用于那些资源有明确访问数量限制的场景：

- 比如：数据库连接池，同时进行连接的线程有数量限制，连接不能超过一定的数量，当连接达到了限制数量后，后面的线程只能排队等前面的线程释放了数据库连接才能获得数据库连接。
    
- 比如：停车场场景，车位数量有限，同时只能容纳多少台车，车位满了之后只有等里面的车离开停车场外面的车才可以进入。可以把它简单的理解成我们停车场入口立着的那个显示屏，每有一辆车进入停车场显示屏就会显示剩余车位减1，每有一辆车从停车场出去，显示屏上显示的剩余车辆就会加1，当显示屏上的剩余车位为0时，停车场入口的栏杆就不会再打开，车辆就无法进入停车场了，直到有一辆车从停车场出去为止。
    
- 比如：接口限流，应用限流，商品限流…
    

Semaphore常用方法说明

```Java
acquire()  
获取一个令牌，在获取到令牌、或者被其他线程调用中断之前线程一直处于阻塞状态。

tryAcquire(long timeout, TimeUnit unit)
尝试在指定时间内获得令牌,返回获取令牌成功或失败，不阻塞线程。

release()
释放一个令牌，唤醒一个获取令牌不成功的阻塞线程。

hasQueuedThreads()
等待队列里是否还存在等待线程。

getQueueLength()
获取等待队列里阻塞的线程数。

drainPermits()
清空令牌把可用令牌数置为0，返回清空令牌的数量。

availablePermits()
返回可用的令牌数量。

// .....其他的自己看源码
```

# 用semaphore 实现停车场提示牌功能

每个停车场入口都有一个提示牌，上面显示着停车场的剩余车位还有多少，当剩余车位为0时，不允许车辆进入停车场，直到停车场里面有车离开停车场，这时提示牌上会显示新的剩余车位数。

业务场景 ：

- 停车场容纳总停车量10。
    
- 当一辆车进入停车场后，显示牌的剩余车位数响应的减1.
    
- 每有一辆车驶出停车场后，显示牌的剩余车位数响应的加1。
    
- 停车场剩余车位不足时，车辆只能在外面等待。
    

```Java
public class TestCar {

    //停车场同时容纳的车辆10
    private  static  Semaphore semaphore=new Semaphore(10);

    public static void main(String[] args) {

        //模拟100辆车进入停车场
        for(int i=0;i<100;i++){

            Thread thread=new Thread(new Runnable() {
                public void run() {
                    try {
                        System.out.println("===="+Thread.currentThread().getName()+"来到停车场");
                        if(semaphore.availablePermits()==0){
                            System.out.println("车位不足，请耐心等待");
                        }
                        semaphore.acquire();//获取令牌尝试进入停车场
                        System.out.println(Thread.currentThread().getName()+"成功进入停车场");
                        Thread.sleep(new Random().nextInt(10000));//模拟车辆在停车场停留的时间
                        System.out.println(Thread.currentThread().getName()+"驶出停车场");
                        semaphore.release();//释放令牌，腾出停车场车位
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            },i+"号车");

            thread.start();

        }

    }
}
```

# 用semaphore 实现防止商品超卖

```Java
package com.limiting.semaphore;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * 秒杀防止商品超卖现象
 */
public class SemaphoreCommodity {

    //商品池
  private   Map<String, Semaphore> map=new ConcurrentHashMap<>();

    //初始化商品池
    public SemaphoreCommodity() {
        //手机10部
        map.put("phone",new Semaphore(10));
        //电脑4台
        map.put("computer",new Semaphore(4));
    }

    /**
     *
     * @param name 商品名称
     * @return 购买是否成功
     */
    public boolean getbuy(String name) throws Exception {

        Semaphore semaphore = map.get(name);
        while (true) {
            int availablePermit = semaphore.availablePermits();
            if (availablePermit==0) {
                //商品售空
                return  false;
            }
            boolean b = semaphore.tryAcquire(1, TimeUnit.SECONDS);
            if (b) {
                System.out.println("抢到商品了");
                ///处理逻辑
                return  true;
            }

        }

    }

    public static void main(String[] args) throws Exception {
        SemaphoreCommodity semaphoreCommodity=new SemaphoreCommodity();
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                try {
                    System.out.println(semaphoreCommodity.getbuy("computer"));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }


    }



}
```

# 用semaphore 实现接口限流

切面注解`SemaphoreDoc`

```Java
package com.limiting.semaphore;

import java.lang.annotation.*;

@Documented

@Target({ElementType.METHOD})//作用:方法
@Retention(RetentionPolicy.RUNTIME)
public @interface SemaphoreDoc {

    String key(); //建议设置，不然可能发生不同方法重复限流现象
    int limit() default 3;
    int blockingTime() default 3;

}
```

切面类`SemaphoreAop`

```Java
package com.limiting.semaphore;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;


import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

@Component
@Aspect
public class SemaphoreAop {

    //这里需要注意了，这个是将自己自定义注解作为切点的根据，路径一定要写正确了
    @Pointcut(value = "@annotation(com.limiting.semaphore.SemaphoreDoc)")
    public void semaphoreDoc() {

    }
    //限流池
  private static  Map<String, Semaphore> map=new ConcurrentHashMap<>();


    @Around("semaphoreDoc()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        Object res = null;
        MethodSignature signature = (MethodSignature)joinPoint.getSignature();
        SemaphoreDoc annotation  = signature.getMethod().getAnnotation(SemaphoreDoc.class);
        int blockingTime = annotation.blockingTime();
        int limit = annotation.limit();
        String key = annotation.key();


        StringBuilder name = new StringBuilder(key+signature.getMethod().getName());//方法名
        for (String parameterName : signature.getParameterNames()) {
            name.append(parameterName);
        }

        Semaphore semaphore = map.get(name.toString());
        if (semaphore == null) {
            Semaphore semaphore1 = new Semaphore(limit);
            map.put(name.toString(),semaphore1);
            semaphore=semaphore1;
        }

        try {


            //获取令牌
            boolean b = semaphore.tryAcquire(blockingTime, TimeUnit.SECONDS);
            if (b) {//如果拿到令牌了那么执行方法
                try {
                    res = joinPoint.proceed();
                } catch (Throwable e) {
                    e.printStackTrace();
                }
            } else {
                //在一定时间内拿不到令牌那么就访问失败
                throw  new Exception("访问超时,目前请求人数过多请稍后在试");
            }

        } finally {
            //释放令牌，腾出位置
            semaphore.release();
        }

        return  res;
    }

}
```