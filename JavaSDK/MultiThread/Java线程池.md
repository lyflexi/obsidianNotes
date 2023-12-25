**池化技术相比大家已经屡见不鲜了，线程池、数据库连接池、Http 连接池等等都是对这个思想的应用。池化技术的思想**

- **主要是为了减少每次获取资源的消耗，提高对资源的利用率。**
    
- **线程池**提供了一种限制和管理资源（包括执行一个任务）。
    
- 每个**线程池**还维护一些基本统计信息，例如已完成任务的数量。
    

这里借用《Java 并发编程的艺术》提到的来说一下**使用线程池的好处**：

- **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
    
- **提高响应速度**。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
    
- **提高线程的可管理性**。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。
    

# Executors 创建线程池

Executors 工具类中的方法如图所示：
![[Pasted image 20231225133731.png]]

**通过 Executor 框架的工具类 Executors 来实现**我们可以创建三种类型的 ThreadPoolExecutor：

- **CachedThreadPool：** 该方法返回一个可根据实际情况调整线程数量的线程池。
    
    - 线程池的线程数量不确定，但若有空闲线程可以复用，则会优先使用可复用的线程。
        
    - 若所有线程均在工作，又有新的任务提交，则会创建新的线程处理任务。
        
    - 所有线程在当前任务执行完毕后，将返回线程池进行复用。
        
- **ScheduledThreadPoolExecutor** 主要用来在给定的延迟后运行任务，或者定期执行任务。 这个在实际项目中基本不会被用到，也不推荐使用，大家只需要简单了解一下它的思想即可。
    
- **FixedThreadPool** ： 该方法返回一个固定线程数量的线程池。该线程池中的线程数量始终不变。
    
    - 当有一个新的任务提交时，线程池中若有空闲线程，则立即执行。
        
    - 若没有，则新的任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务。
        
- **SingleThreadExecutor：** 方法返回一个只有一个线程的线程池。
    
    - 当有一个新的任务提交时，任务会被保存在一个任务队列中，
        
    - 待线程空闲，按先入先出的顺序执行队列中的任务。
        

但是《阿里巴巴 Java 开发手册》中强制线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。Executors 返回线程池对象的弊端如下：

- **CachedThreadPool 和 ScheduledThreadPool** ： **允许创建的线程数量为 Integer.MAX_VALUE** ，可能会创建大量线程导致 OOM。
    
- **FixedThreadPool 和 SingleThreadExecutor** ： **允许请求的队列长度为 Integer.MAX_VALUE** ，可能堆积大量的请求，从而导致 OOM。
    

# ThreadPoolExecutor 创建线程池

![[Pasted image 20231225133749.png]]

ThreadPoolExecutor 类中提供的四个构造方法。我们来看最长的那个，其余三个都是在这个构造方法的基础上产生（其他几个构造方法说白点都是给定某些默认参数的构造方法比如默认制定拒绝策略是什么），这里就不贴代码讲了，比较简单。

```Java
/**
 * 用给定的初始参数创建一个新的ThreadPoolExecutor。
 */
public ThreadPoolExecutor(int corePoolSize,
                      int maximumPoolSize,
                      long keepAliveTime,
                      TimeUnit unit,
                      BlockingQueue<Runnable> workQueue,
                      ThreadFactory threadFactory,
                      RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
            throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

## 构造函数参数分析

1. corePoolSize : 核心线程数线程数定义了最小可以同时运行的线程数量。
    
2. workQueue: 当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在阻塞队列中。注意，阻塞队列中的线程暂时不会执行
    
3. maximumPoolSize : 当队列中存放的任务达到阻塞队列容量的时候，当前还可以新运行`maximumPoolSize -corePoolSize` 个线程数。
    
4. keepAliveTime：当线程池中的线程数量大于 corePoolSize 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 keepAliveTime才会被回收销毁；
    
5. unit : keepAliveTime 参数的时间单位。
    
6. threadFactory :executor 创建新线程的时候会用到。
    
7. RejectedExecutionHandler拒绝策略：如果当前队列已经被放满了任务，并且运行的线程数量达到最大线程数量，ThreadPoolTaskExecutor 将采取拒绝策略策略
    
    1. AbortPolicy【默认策略】：抛出 `RejectedExecutionException`异常来拒绝新任务的处理。这代表你将丢失对这个任务的处理。
        
    2. DiscardPolicy： 直接拒绝任务，不抛出错误也不会给你任何的通知
        
    3. DiscardOldestPolicy： 只要还有任务新增，一直会丢弃阻塞队列workQueue的最老的任务，并将新的任务加入
        
    4. **CallerRunsPolicy【提供可伸缩队列】**：该策略直接在调用者线程中，运行当前的被丢弃的任务。==调用者线程指的是执行`execute`方法的线程，也即创建当前线程池的线程，特别的当调用者是主线程的时候，指的就是main线程。==
    

例题举例1：一个线程池 core 7； max 20 ，queue：50，100并发进来怎么分配的；

1. 7个会立即得到执行，
    
2. 50个会进入队列，
    
3. 再开13个进行执行。
    
4. 剩下的30个就使用拒绝策略。如果不想抛弃还要执行，拒绝策略采用CallerRunsPolicy；
    

例题举例2：我们在代码中模拟了 10 个任务，我们配置的核心线程数为 5 、等待队列容量为 100 ，

1. 所以每次只可能存在 5 个任务同时执行，
    
2. 剩下的 5 个任务会被放到等待队列中去。当前的 5 个任务执行完成后，才会执行剩下的 5 个任务。
    
3. ==等待队列装不满，所以用不到maximumPoolSize==
    

# 线程池execute()和 submit()的区别

- `void execute(Runnable command);`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；
- `<T> Future<T> submit(Callable<T> task);`方法用于提交需要返回值的任务。线程池会返回一个 Future 类型的对象，通过这个 Future 对象可以判断任务是否执行成功，并且可以通过 Future 的 `get()`方法来获取返回值：
    - `get()`方法会阻塞当前线程直到任务完成，
    - `get（long timeout，TimeUnit unit）`方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。