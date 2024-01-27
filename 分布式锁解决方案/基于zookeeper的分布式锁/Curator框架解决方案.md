Curator是netflix公司开源的一套zookeeper客户端，目前是Apache的顶级项目。与Zookeeper提供的原生客户端相比，Curator的抽象层次更高，简化了Zookeeper客户端的开发量。Curator解决了很多zookeeper客户端非常底层的细节开发工作，包括连接重连、反复注册wathcer和NodeExistsException 异常等。

通过查看官方文档，可以发现Curator主要解决了三类问题：
- 封装ZooKeeper client与ZooKeeper server之间的连接处理
- 提供了一套Fluent风格的操作API
- 提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装，这些实现都遵循了zk的最佳实践，并考虑了各种极端情况


Curator由一系列的模块构成，对于一般开发者而言，常用的是curator-framework和curator-recipes：
- curator-framework：提供了常见的zk相关的底层操作
- curator-recipes：提供了一些zk的典型使用场景的参考。本节重点关注的分布式锁就是该包提供的

引入依赖：
```XML
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.3.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.3.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
    <version>3.7.0</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.4</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

> 最新版本的curator 4.3.0支持zookeeper 3.4.x和3.5，但是需要注意curator传递进来的依赖，需要和实际服务器端使用的版本相符，以我们目前使用的zookeeper 3.4.14为例。

但是我使用以我们目前使用的zookeeper3.7.0好像也没问题

添加curator客户端配置：
```java
@Configuration
public class CuratorConfig {

    @Bean
    public CuratorFramework curatorFramework(){
        // 重试策略，这里使用的是指数补偿重试策略，重试3次，初始重试间隔1000ms，每次重试之后重试间隔递增。
        RetryPolicy retry = new ExponentialBackoffRetry(1000, 3);
        // 初始化Curator客户端：指定链接信息 及 重试策略
        CuratorFramework client = CuratorFrameworkFactory.newClient("172.16.116.100:2181", retry);
        client.start(); // 开始链接，如果不调用该方法，很多方法无法工作
        return client;
    }
}
```
# 框架源码追踪
以最常用的分布式可重入锁InterProcessMutex为例

public InterProcessMutex(CuratorFramework client, String path)
public void acquire()
public void release()

InterProcessMutex
	basePath：初始化锁时指定的节点路径
	internals：LockInternals对象，加锁 解锁
	ConcurrentMap<Thread, LockData> threadData：记录了重入信息
	class LockData {
		Thread lockPath lockCount
	}
	

LockInternals
	maxLeases：租约，值为1
	basePath：初始化锁时指定的节点路径
	path：basePath + "/lock-"
	
加锁流程：InterProcessMutex.acquire() --> InterProcessMutex.internalLock() --> LockInternals.attemptLock()
# 可重入锁InterProcessMutex
Reentrant和JDK的ReentrantLock类似， 意味着同一个客户端在拥有锁的同时，可以多次获取，不会被阻塞。它是由类**InterProcessMutex**来实现。
```java
// 常用构造方法
public InterProcessMutex(CuratorFramework client, String path)
// 获取锁
public void acquire();
// 带超时时间的可重入锁
public boolean acquire(long time, TimeUnit unit);
// 释放锁
public void release();
```
改造service测试方法：
```java
@Autowired  
private CuratorFramework curatorFramework;  
@Autowired  
StringRedisTemplate redisTemplate;  
  
/*测试可重入锁InterProcessMutex*/  
public void checkAndLock() {  
    InterProcessMutex mutex = new InterProcessMutex(curatorFramework, "/curator/lock");  
    try {  
        // 加锁  
        mutex.acquire();  
  
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
  
         this.testSub(mutex);  
  
        // 释放锁  
        mutex.release();  
    } catch (Exception e) {  
        e.printStackTrace();  
    }  
}  
  
public void testSub(InterProcessMutex mutex) {  
  
    try {  
        mutex.acquire();  
        System.out.println("测试可重入锁。。。。");  
        mutex.release();  
    } catch (Exception e) {  
        e.printStackTrace();  
    }  
}
```
注意：如想重入，不可再new新的InterProcessMutex对象。
压力测试结果：
![[1607069431523.png]]

# 不可重入锁InterProcessSemaphoreMutex
具体实现：InterProcessSemaphoreMutex。与InterProcessMutex调用方法类似，区别在于该锁是不可重入的，在同一个线程中不可重入。
```java
public InterProcessSemaphoreMutex(CuratorFramework client, String path);
public void acquire();
public boolean acquire(long time, TimeUnit unit);
public void release();
```
案例：
```java
@Autowired
private CuratorFramework curatorFramework;

public void deduct() {

    InterProcessSemaphoreMutex mutex = new InterProcessSemaphoreMutex(curatorFramework, "/curator/lock");
    try {
        mutex.acquire();
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
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            mutex.release();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
# 可重入读写锁InterProcessReadWriteLock
类似JDK的ReentrantReadWriteLock。一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁。这也意味着写锁可以降级成读锁。从读锁升级成写锁是不成的。主要实现类InterProcessReadWriteLock：
```java
// 构造方法
public InterProcessReadWriteLock(CuratorFramework client, String basePath);
// 获取读锁对象
InterProcessMutex readLock();
// 获取写锁对象
InterProcessMutex writeLock();
```
curator的读写锁自然也是读读共享，读写和写写互斥
但是不同的是，curator的写锁在释放之前会一直阻塞请求线程，而读锁不会
```java
public void testZkReadLock() {
    try {
        InterProcessReadWriteLock rwlock = new InterProcessReadWriteLock(curatorFramework, "/curator/rwlock");
        rwlock.readLock().acquire(10, TimeUnit.SECONDS);
        // TODO：一顿读的操作。。。。
        //rwlock.readLock().unlock();
    } catch (Exception e) {
        e.printStackTrace();
    }
}

public void testZkWriteLock() {
    try {
        InterProcessReadWriteLock rwlock = new InterProcessReadWriteLock(curatorFramework, "/curator/rwlock");
        rwlock.writeLock().acquire(10, TimeUnit.SECONDS);
        // TODO：一顿写的操作。。。。
        //rwlock.writeLock().unlock();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
Controller添加方法：
```java
@GetMapping("stock/testZkReadLock")  
public String testZkReadLock(){  
    stockService.testZkReadLock();  
    return "hello stock deduct！！";  
}  
@GetMapping("stock/testZkWriteLock")  
public String testZkWriteLock(){  
    stockService.testZkWriteLock();  
    return "hello stock deduct！！";  
}
```
# 联锁InterProcessMultiLock
Multi Shared Lock是一个锁的容器。当调用acquire， 所有的锁都会被acquire，如果请求失败，所有的锁都会被release。同样调用release时所有的锁都被release(失败被忽略)。基本上，它就是组锁的代表，在它上面的请求释放操作都会传递给它包含的所有的锁。实现类InterProcessMultiLock：
```java
// 构造函数需要包含的锁的集合，或者一组ZooKeeper的path
public InterProcessMultiLock(List<InterProcessLock> locks);
public InterProcessMultiLock(CuratorFramework client, List<String> paths);

// 获取锁
public void acquire();
public boolean acquire(long time, TimeUnit unit);

// 释放锁
public synchronized void release();
```
# 信号量InterProcessSemaphoreV2
一个计数的信号量类似JDK的Semaphore。==JDK中Semaphore维护的一组许可(permits)，而Cubator中称之为租约(Lease)==。注意，所有的实例必须使用相同的numberOfLeases值。调用acquire会返回一个租约对象。客户端必须在finally中close这些租约对象，否则这些租约会丢失掉。但是，如果客户端session由于某种原因比如crash丢掉， 那么这些客户端持有的租约会自动close， 这样其它客户端可以继续使用这些租约。主要实现类InterProcessSemaphoreV2：
```java
// 构造方法
public InterProcessSemaphoreV2(CuratorFramework client, String path, int maxLeases);

// 注意一次你可以请求多个租约，如果Semaphore当前的租约不够，则请求线程会被阻塞。
// 同时还提供了超时的重载方法
public Lease acquire();
public Collection<Lease> acquire(int qty);
public Lease acquire(long time, TimeUnit unit);
public Collection<Lease> acquire(int qty, long time, TimeUnit unit)

// 租约还可以通过下面的方式返还
public void returnAll(Collection<Lease> leases);
public void returnLease(Lease lease);
```
案例代码：
StockController中添加方法：
```java
@GetMapping("test/semaphore")
public String testSemaphore(){
    this.stockService.testSemaphore();
    return "hello Semaphore";
}
```
StockService中添加方法：
```java
public void testSemaphore() {
    // 设置资源量 限流的线程数
    InterProcessSemaphoreV2 semaphoreV2 = new InterProcessSemaphoreV2(curatorFramework, "/locks/semaphore", 5);
    try {
        Lease acquire = semaphoreV2.acquire();// 获取资源，获取资源成功的线程可以继续处理业务操作。否则会被阻塞住
        this.redisTemplate.opsForList().rightPush("log", "10010获取了资源，开始处理业务逻辑。" + Thread.currentThread().getName());
        TimeUnit.SECONDS.sleep(10 + new Random().nextInt(10));
        this.redisTemplate.opsForList().rightPush("log", "10010处理完业务逻辑，释放资源=====================" + Thread.currentThread().getName());
        semaphoreV2.returnLease(acquire); // 手动释放资源，后续请求线程就可以获取该资源
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
# 共享计数器
利用ZooKeeper可以实现一个集群共享的计数器。只要使用相同的path就可以得到最新的计数器值， 这是由ZooKeeper的一致性保证的。这个功能可以实现ConuntDownLanch。Curator有两种计数器：
- SharedCount
- DistributedAtomicNumber
## SharedCount
共享计数器SharedCount相关方法如下：
```java
// 构造方法
public SharedCount(CuratorFramework client, String path, int seedValue);
// 获取共享计数的值
public int getCount();
// 设置共享计数的值
public void setCount(int newCount) throws Exception;
// 当版本号没有变化时，才会更新共享变量的值
public boolean  trySetCount(VersionedValue<Integer> previous, int newCount);
// 通过监听器监听共享计数的变化
public void addListener(SharedCountListener listener);
public void addListener(final SharedCountListener listener, Executor executor);
// 共享计数在使用之前必须开启
public void start() throws Exception;
// 关闭共享计数
public void close() throws IOException;
```
使用案例：

StockController：
```java
@GetMapping("test/zk/share/count")
public String testZkShareCount(){
    this.stockService.testZkShareCount();
    return "hello shareData";
}
```
StockService：
```java
public void testZkShareCount() {
    try {
        // 第三个参数是共享计数的初始值
        SharedCount sharedCount = new SharedCount(curatorFramework, "/curator/count", 0);
        // 启动共享计数器
        sharedCount.start();
        // 获取共享计数的值
        int count = sharedCount.getCount();
        // 修改共享计数的值
        int random = new Random().nextInt(1000);
        sharedCount.setCount(random);
        System.out.println("我获取了共享计数的初始值：" + count + "，并把计数器的值改为：" + random);
        sharedCount.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
## DistributedAtomicNumber
DistributedAtomicNumber接口是分布式原子数值类型的抽象，定义了分布式原子数值类型需要提供的方法。
DistributedAtomicNumber接口有两个实现：`DistributedAtomicLong` 和 `DistributedAtomicInteger`
![[image-20220711225708066.png]]
这两个实现将各种原子操作的执行委托给了`DistributedAtomicValue`，所以这两种实现是类似的，只不过表示的数值类型不同而已。这里以`DistributedAtomicLong` 为例进行演示

DistributedAtomicLong除了计数的范围比SharedCount大了之外，比SharedCount更简单易用。它首先尝试使用乐观锁的方式设置计数器， 如果不成功(比如期间计数器已经被其它client更新了)， 它使用InterProcessMutex方式来更新计数值。此计数器有一系列的操作：

- get(): 获取当前值
- increment()：加一
- decrement(): 减一
- add()：增加特定的值
- subtract(): 减去特定的值
- trySet(): 尝试设置计数值
- forceSet(): 强制设置计数值

你必须检查返回结果的succeeded()， 它代表此操作是否成功。如果操作成功， preValue()代表操作前的值， postValue()代表操作后的值。
# 栅栏barrier
DistributedBarrier构造函数中barrierPath参数用来确定一个栅栏，只要barrierPath参数相同(路径相同)就是同一个栅栏。通常情况下栅栏的使用如下：
1. 主client设置一个栅栏
2. 其他客户端就会调用waitOnBarrier()等待栅栏移除，程序处理线程阻塞
3. 主client移除栅栏，其他客户端的处理程序就会同时继续运行。
## DistributedBarrier
DistributedBarrier类的主要方法如下：
```java
setBarrier() - 设置栅栏
waitOnBarrier() - 等待栅栏移除
removeBarrier() - 移除栅栏
```
## DistributedDoubleBarrier双栅栏
DistributedDoubleBarrier双栅栏，允许客户端在计算的开始和结束时同步。当足够的进程加入到双栅栏时，进程开始计算，当计算完成时，离开栅栏。DistributedDoubleBarrier实现了双栅栏的功能。构造函数如下：
```java
// client - the client
// barrierPath - path to use
// memberQty - the number of members in the barrier
public DistributedDoubleBarrier(CuratorFramework client, String barrierPath, int memberQty);

enter()、enter(long maxWait, TimeUnit unit) - 等待同时进入栅栏
leave()、leave(long maxWait, TimeUnit unit) - 等待同时离开栅栏
```
memberQty是成员数量，当enter方法被调用时，成员被阻塞，直到所有的成员都调用了enter。当leave方法被调用时，它也阻塞调用线程，直到所有的成员都调用了leave。

注意：参数memberQty的值只是一个阈值，而不是一个限制值。当等待栅栏的数量大于或等于这个值栅栏就会打开！

与栅栏(DistributedBarrier)一样,双栅栏的barrierPath参数也是用来确定是否是同一个栅栏的，双栅栏的使用情况如下：

1. 从多个客户端在同一个路径上创建双栅栏(DistributedDoubleBarrier),然后调用enter()方法，等待栅栏数量达到memberQty时就可以进入栅栏。
2. 栅栏数量达到memberQty，多个客户端同时停止阻塞继续运行，直到执行leave()方法，等待memberQty个数量的栅栏同时阻塞到leave()方法中。
3. memberQty个数量的栅栏同时阻塞到leave()方法中，多个客户端的leave()方法停止阻塞，继续运行。