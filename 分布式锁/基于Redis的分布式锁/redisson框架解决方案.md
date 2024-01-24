Redisson它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。
- BitSet, Set, Multimap, SortedSet, Map, List, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, AtomicLong, CountDownLatch, 
- Publish / Subscribe, Bloom filter, Remote service, Spring cache, Executor service, Live Object service, Scheduler service
Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。
![[image-20220501155501783.png]]
官方文档地址：https://github.com/redisson/redisson/wiki
引入依赖
```xml
<dependency>  
    <groupId>org.redisson</groupId>  
    <artifactId>redisson</artifactId>  
    <version>3.23.0</version>  
</dependency>
```
添加配置
```java
package org.lyflexi.redissonclient.config;  
  
import org.redisson.Redisson;  
import org.redisson.api.RedissonClient;  
import org.redisson.config.Config;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/23 20:05  
 */@Configuration  
public class RedissonConfig {  
  
    @Bean  
    public RedissonClient redissonClient(){  
        Config config = new Config();  
        // 可以用"rediss://"来启用SSL连接  
        config.useSingleServer().setAddress("redis://192.168.18.100:6379");  
        return Redisson.create(config);  
    }  
}
```
# 可重入锁RLock
跟踪RLock#lock()方法源码，发现确实可重入，redisson基于redis实现了一套可重入锁，与AQS十分的像但不是AQS
使用方式如下，`RLock`对象完全符合Java的Lock规范。只有拥有锁的进程才能解锁，其他进程解锁则会抛出`IllegalMonitorStateException`错误。
```java
package org.lyflexi.redissonclient.service;  
  
  
import org.redisson.api.RLock;  
import org.redisson.api.RedissonClient;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.stereotype.Service;  
  
import java.util.concurrent.TimeUnit;  
  
@Service  
public class StockService {  
  
    @Autowired  
    StringRedisTemplate redisTemplate;  
    @Autowired  
    RedissonClient redissonClient;  
    public void deduct() {  
        RLock lock = redissonClient.getLock("lock");  
        lock.lock();  
  
        try {  
            // 1. 查询库存信息  
            String stock = redisTemplate.opsForValue().get("stock").toString();  
  
            // 2. 判断库存是否充足  
            if (stock != null && stock.length() != 0) {  
                Integer st = Integer.valueOf(stock);  
                if (st > 0) {  
                    // 3.扣减库存  
                    redisTemplate.opsForValue().set("stock", String.valueOf(--st));  
                }  
            }  
            this.test();  
        } finally {  
            lock.unlock();  
        }  
    }  
    public void test() {  
        RLock redisLock = this.redissonClient.getLock("lock");  
        redisLock.lock();  
        System.out.println("测试可重入锁");  
        redisLock.unlock();  
    }  
}
```
另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间（过期时间）。超过这个时间后锁便自动解开了。
```java
// 加锁以后10秒钟自动解锁
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}
```

redisson内置看门狗续期，只要占锁成功，就会启动一个定时任务每隔 10 秒重新给锁设置默认30 秒的过期时间，过期时间可以通过修改`Config.lockWatchdogTimeout`来指定。如下图所示：
![[Pasted image 20240124085411.png]]
看门狗并不会造成死锁，原因有两点：
- 当前线程业务执行结束，看门狗会释放，不会一直续期
- 当服务器宕机后，看门狗也会相应死去，因为锁的有效期是 30 秒（宕机之前的锁占用时间+后续锁占用的时间），所以30s到了之后就会自动释放锁
```java
package org.lyflexi.redissonclient.service;  

@Service  
public class StockService {  
  
    @Autowired  
    StringRedisTemplate redisTemplate;  
    @Autowired  
    RedissonClient redissonClient;  
    public void deduct() {  
        RLock lock = redissonClient.getLock("lock");  
        lock.lock();  
  
        try {  
            // 1. 查询库存信息  
            String stock = redisTemplate.opsForValue().get("stock").toString();  
  
            // 2. 判断库存是否充足  
            if (stock != null && stock.length() != 0) {  
                Integer st = Integer.valueOf(stock);  
                if (st > 0) {  
                    // 3.扣减库存  
                    redisTemplate.opsForValue().set("stock", String.valueOf(--st));  
                }  
            }  
            //触发看门狗自动续期  
            try {  
                TimeUnit.SECONDS.sleep(1000);  
            } catch (InterruptedException e) {  
                throw new RuntimeException(e);  
            }  
        } finally {  
            lock.unlock();  
        }  
    }  
  
}
```
# 公平锁FairLock
基于Redis的Redisson分布式可重入公平锁也是实现了`java.util.concurrent.locks.Lock`接口的一种`RLock`对象。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockRx.html)的接口。它保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。所有请求线程会在一个队列中排队，当某个线程出现宕机时，Redisson会等待5秒后继续下一个线程，也就是说如果前面有5个线程都处于等待状态，那么后面的线程会等待至少25秒。
```java
RLock fairLock = redisson.getFairLock("anyLock");
// 最常见的使用方法
fairLock.lock();

// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
fairLock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = fairLock.tryLock(100, 10, TimeUnit.SECONDS);
fairLock.unlock();
```

# 联锁MultiLock
基于Redis的Redisson分布式联锁[`RedissonMultiLock`](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/RedissonMultiLock.html)对象可以将多个`RLock`对象关联为一个联锁，每个`RLock`对象实例可以来自于不同的Redisson实例。
```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 所有的锁都上锁成功才算成功。
lock.lock();
...
lock.unlock();
```
# 红锁RedLock
基于Redis的Redisson红锁`RedissonRedLock`对象实现了[Redlock](http://redis.cn/topics/distlock.html)介绍的加锁算法。该对象也可以用来将多个`RLock`对象关联为一个红锁，每个`RLock`对象实例可以来自于不同的Redisson实例。
```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 红锁在半数以上节点上加锁成功就算成功。
lock.lock();
...
lock.unlock();
```
# 读写锁ReadWriteLock
基于Redis的Redisson分布式可重入读写锁[`RReadWriteLock`](http://static.javadoc.io/org.redisson/redisson/3.4.3/org/redisson/api/RReadWriteLock.html) Java对象实现了`java.util.concurrent.locks.ReadWriteLock`接口。其中读锁和写锁都继承了[RLock](https://github.com/redisson/redisson/wiki/8.-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E5%92%8C%E5%90%8C%E6%AD%A5%E5%99%A8#81-%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81reentrant-lock)接口。

分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。
```java
RReadWriteLock rwlock = redisson.getReadWriteLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();

// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```
添加测试方法如下，打开开两个浏览器窗口测试：
- 同时访问写：一个写完之后，等待一会儿（约10s），另一个写开始
- 同时访问读：不用等待
- 先写后读：读要等待（约10s）写完成
- 先读后写：写要等待（约10s）读完成
```java
@GetMapping("test/read")
public String testRead(){
    String msg = stockService.testRead();

    return "测试读";
}

@GetMapping("test/write")
public String testWrite(){
    String msg = stockService.testWrite();

    return "测试写";
}



public String testRead() {
    RReadWriteLock rwLock = this.redissonClient.getReadWriteLock("rwLock");
    rwLock.readLock().lock(10, TimeUnit.SECONDS);

    System.out.println("测试读锁。。。。");
    // rwLock.readLock().unlock();

    return null;
}

public String testWrite() {
    RReadWriteLock rwLock = this.redissonClient.getReadWriteLock("rwLock");
    rwLock.writeLock().lock(10, TimeUnit.SECONDS);

    System.out.println("测试写锁。。。。");
    // rwLock.writeLock().unlock();

    return null;
}
```
# 信号量Semaphore限流
基于Redis的Redisson的分布式信号量（[Semaphore](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphore.html)）Java对象`RSemaphore`采用了与`java.util.concurrent.Semaphore`相似的接口和用法。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreRx.html)的接口。
```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.trySetPermits(3);
semaphore.acquire();
semaphore.release();
```
添加测试方法：
```java
@GetMapping("test/semaphore")
public String testSemaphore(){
    this.stockService.testSemaphore();

    return "测试信号量";
}


public void testSemaphore() {
    RSemaphore semaphore = this.redissonClient.getSemaphore("semaphore");
    semaphore.trySetPermits(3);
    try {
        semaphore.acquire();

        TimeUnit.SECONDS.sleep(5);
        System.out.println(System.currentTimeMillis());

        semaphore.release();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```
jmeter测试
![[1606961296212.png]]
控制台效果，一次只进来3个并发处理。即使在分布式场景下，多台服务器同一时刻处理的并发数之和也是3
```java

1606960790234
1606960800337
1606960800443


1606960790328
1606960795332
1606960800245


1606960790433
1606960795238
1606960795437

1606960805248
```
# 闭锁CountDownLatch
基于Redisson的Redisson分布式闭锁（[CountDownLatch](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RCountDownLatch.html)）Java对象`RCountDownLatch`采用了与`java.util.concurrent.CountDownLatch`相似的接口和用法。
```java
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.trySetCount(1);
latch.await();

// 在其他线程或其他JVM里
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.countDown();
```
添加测试方法，需要两个方法：一个等待，一个计数countDown
```java
@GetMapping("test/latch")
public String testLatch(){
    this.stockService.testLatch();

    return "班长锁门。。。";
}

@GetMapping("test/countdown")
public String testCountDown(){
    this.stockService.testCountDown();

    return "出来了一位同学";
}


public void testLatch() {
    RCountDownLatch latch = this.redissonClient.getCountDownLatch("latch");
    latch.trySetCount(6);

    try {
        latch.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}

public void testCountDown() {
    RCountDownLatch latch = this.redissonClient.getCountDownLatch("latch");
    latch.countDown();
}
```
重启测试，打开两个页面：
- 先执行test/latch请求，创建RCountDownLatch，并阻塞
- 再执行test/countdown，当test/countdown请求执行6次之后，test/latch请求才会放行。