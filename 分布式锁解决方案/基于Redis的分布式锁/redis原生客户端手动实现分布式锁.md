
Redis并不是天生就默认实现了分布式锁，比如以下程序，即使使用了Redis，依然是有并发问题的，出现了超卖现象
```java
public void deduct(){  
    //1,查询库存信息  
    String stock = this.redisTemplate.opsForvalue().get ("stock");  
    //2.判断库存是否充足  
    if (stock !=null &&stock.length()!=0)(  
            Integer st = Integer.valueOf(stock);  
      
    if (st > 0){  
    //3.扣减库存  
        this.redisTemplate.opsForvalue ().set ("stock",String.valueof(--st));  
    }  
}
```
上述代码只是简单的用redis存了一下数据，当然会有并发现象了，只不过性能肯定是比MySQL要好的
# redis乐观锁实现分布式锁
在当前事务执行期间：
- 如果没有其他的并发事务修改，则当前事务执行成功
- 如果没有其他的并发事务修改，则当前事务执行失败，但是其他并发事务的修改执行成功
watch指令用于监控
multi指令用于开启redis事务
exec用于关闭redis事务
如，正常情况没有其他的并发事务修改，当前事务执行成功返回ok
![[Pasted image 20240123103052.png]]
如，异常情况有其他的并发事务修改，会造成当前事务执行失败，取消当前事务并返回nil（这个逻辑跟MySQL事务的逻辑正好相反）
![[Pasted image 20240123105107.png]]
![[Pasted image 20240123105137.png]]
![[Pasted image 20240123105244.png]]
redis乐观锁的Java代码
```java
public void deduct() {

    this.redisTemplate.execute(new SessionCallback() {
        @Override
        public Object execute(RedisOperations operations) throws DataAccessException {
            operations.watch("stock");
            // 1. 查询库存信息
            Object stock = operations.opsForValue().get("stock");
            // 2. 判断库存是否充足
            int st = 0;
            if (stock != null && (st = Integer.parseInt(stock.toString())) > 0) {
                // 3. 扣减库存
                operations.multi();
                operations.opsForValue().set("stock", String.valueOf(--st));
                List exec = operations.exec();
                if (exec == null || exec.size() == 0) {
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    //重试，这里是递归重试
                    deduct();
                }
                return exec;
            }
            return null;
        }
    });
}
```
redis乐观锁解决了并发超卖问题，压测5000个并发
```shell
set stock 5000
```
![[Pasted image 20240123110026.png]]
```shell
get stock
0
```
# 以下基于redis排他锁setnx实现分布式锁
![[Pasted image 20240123111146.png]]


# redis分布式锁Java代码
借助于redis中的命令setnx(key, value)，key不存在就新增，存在就什么都不做。同时有多个客户端发送setnx命令，只有一个客户端可以成功，返回1（true）；其他的客户端返回0（false）。
1. 多个客户端同时获取锁（setnx），只有一个并发能够获取到
2. 获取成功，执行业务逻辑，执行完成释放锁（del）
3. 其他客户端等待重试
![[1606626611922.png]]

## 递归重试
递归的本地是jvm栈压栈，当前方法调用压在栈底，因此if (!lock){this.deduct();}执行之后必须加else，否则等栈顶方法执行结束退栈，栈底方法被唤醒继续向下执行的时候仍然会引发商品超卖
```java
@Service
public class StockService {

    @Autowired
    private StockMapper stockMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void deduct() {
        // 加锁setnx
        Boolean lock = this.redisTemplate.opsForValue().setIfAbsent("lock", "111");
        // 重试：递归调用
        if (!lock){
            try {
                Thread.sleep(50);
                this.deduct();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } else {
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
            } finally {
                // 解锁
                this.redisTemplate.delete("lock");
            }
        }
    }
}
```
另外，递归重试也容易导致栈内存溢出
![[Pasted image 20240123113140.png]]
## 自旋重试
解锁语句注意要放在finally代码块内，this.redisTemplate.delete("lock");
```java
@Service
public class StockService {

    @Autowired
    private StockMapper stockMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void deduct() {
        // 加锁setnx
        Boolean lock = this.redisTemplate.opsForValue().setIfAbsent("lock", "111");
        // 重试：自旋
		while (!this.redisTemplate.opsForValue().setIfAbsent("lock", "111")){
		    try {
		        Thread.sleep(40);
		    } catch (InterruptedException e) {
		        e.printStackTrace();
		    }
		}
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
		} finally {
			// 解锁
			this.redisTemplate.delete("lock");
		}
	}
}
```
# 防死锁-加过期时间
问题：setnx刚刚获取到锁，当前服务器宕机，导致del无法释放锁，进而即使服务器恢复正常之后，其他并发线程依旧获取不到锁
解决：给锁设置过期时间，自动释放锁。
设置过期时间两种方式：
1. 通过expire设置过期时间：expire key seconds或者expire key milliseconds（缺乏原子性：如果在setnx和expire之间出现异常，锁也无法释放）
2. 使用set指令设置过期时间：set key value ex 3 nx（保证了原子性，既达到setnx的效果，又设置了过期时间）
set其实是一个非常复杂的指令：set key value ex seconds/px milliseconds nx|xx
- setIfAbsent相当于底层的nx，表示如果不存在则加锁
- setIfPresent相当于底层的xx，表示如果存在再加锁
```java
@Service
public class StockService {

    @Autowired
    private StockMapper stockMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void deduct() {
        // 加锁setnx，自旋
		while (!this.redisTemplate.opsForValue().setIfAbsent("lock", "111", 3, TimeUnit.SECONDS)){
		    try {
		        Thread.sleep(40);
		    } catch (InterruptedException e) {
		        e.printStackTrace();
		    }
		}
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
		} finally {
			// 解锁
			this.redisTemplate.delete("lock");
		}
	}
}
```

# 防误删-解铃还须系铃人
问题：虽然锁是同一把锁，但是加锁的人可不是同一个人，因此解锁的时候可能会释放其他服务器的锁。
场景：如果业务逻辑的执行时间是7s。锁过期时间是3s，执行流程如下
1. index1业务逻辑没执行完，3秒后锁被自动释放。
2. index2获取到锁，执行业务逻辑，3秒后锁被自动释放。
3. index3获取到锁，执行业务逻辑
4. index1业务逻辑执行完成，开始调用del释放锁，这时释放的是index3的锁，导致index3的业务只执行1s就被别人释放。
...最终等于没锁的情况。
解决：setnx获取锁时，设置一个指定的唯一值（例如：uuid）；释放前获取这个值，判断是否自己的锁
![[1606707959639.png]]
==不使用threadID作为锁的唯一区分的原因是，集群环境下不同线程的threadID还是会重复，线程ID类似于1，2，3，4，5.....==
下面设置uuid
```java
@Service
public class StockService {

    @Autowired
    private StockMapper stockMapper;

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void deduct() {
	    //加uuid
	    String uuid = UUID.randomUUID().toString();
        // 加锁setnx，自旋
		while (!this.redisTemplate.opsForValue().setIfAbsent("lock", uuid, 3, TimeUnit.SECONDS)){
		    try {
		        Thread.sleep(40);
		    } catch (InterruptedException e) {
		        e.printStackTrace();
		    }
		}
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
		} finally {
			// 解锁
			if(StringUtils.equals(this.redisTemplate.opsForValue().get ("lock"),uuid)){
				this.redisTemplate.delete("lock");
			}

		}
	}
}
```
# 保证删除原子性-lua脚本
问题：删除操作缺乏原子性。
场景：
1. index1执行删除时，查询到的lock值确实和uuid相等
2. index1执行删除前，lock刚好过期时间已到，被redis自动释放
3. index2获取了lock
4. index1执行删除，此时会把index2的lock删除
解决方案：没有一个命令可以同时做到判断 + 删除，所有只能通过其他方式实现（LUA脚本）
```shell
EVAL script numkeys key [key ...] arg [arg ...]  
script：lua脚本字符串，这段Lua脚本不需要（也不应该）定义函数。  
numkeys：lua脚本中KEYS数组的大小  
key [key ...]：KEYS数组中的元素  
arg [arg ...]：ARGV数组中的元素
```
案例1：动态传参，前5个是key，剩下的5个是arg
```shell
EVAL "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 5 10 20 30 40 50 60 70 80 90  
# 输出：10 20 60 70  
```
注意：脚本里使用的所有键都应该由 KEYS 数组来传递。但并不是强制性的，代价是这样写出的脚本不能被 Redis 集群所兼容。
案例2：调用redis类库方法
```shell
EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 bbb 20
```
![[1600610957600.png]]
学到这里基本可以应付redis分布式锁所需要的脚本知识了。
案例3：pcall函数的使用（了解），因为set方法写成了sets，肯定会报错。
```shell
-- 当call() 在执行命令的过程中发生错误时，脚本会停止执行，并返回一个脚本错误，输出错误信息  
EVAL "return redis.call('sets', KEYS[1], ARGV[1]), redis.call('set', KEYS[2], ARGV[2])" 2 bbb ccc 20 30  
-- pcall函数不影响后续指令的执行  
EVAL "return redis.pcall('sets', KEYS[1], ARGV[1]), redis.pcall('set', KEYS[2], ARGV[2])" 2 bbb ccc 20 30
```
![[1600612707202.png]]
，接下来，删除锁的LUA脚本就可以这样写：
```shell
if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end
```
代码实现：
```java
public void deduct() {  
    String uuid = UUID.randomUUID().toString();  
    // 加锁setnx  
    while (!this.redisTemplate.opsForValue().setIfAbsent("lock", uuid, 3, TimeUnit.SECONDS)) {  
        // 重试：循环  
        try {  
            Thread.sleep(50);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
    try {  
        // this.redisTemplate.expire("lock", 3, TimeUnit.SECONDS);  
        // 1. 查询库存信息  
        String stock = redisTemplate.opsForValue().get("stock").toString();  
​  
        // 2. 判断库存是否充足  
        if (stock != null && stock.length() != 0) {  
            Integer st = Integer.valueOf(stock);  
            if (st > 0) {  
                // 3.扣减库存  
                redisTemplate.opsForValue().set("stock", String.valueOf(--st));  
            }  
        }  
    } finally {  
        // 先判断是否自己的锁，再解锁  
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] " +  
            "then " +  
            "   return redis.call('del', KEYS[1]) " +  
            "else " +  
            "   return 0 " +  
            "end";  
        this.redisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList("lock"), uuid);  
    }  
}
```

压力测试，不会出现超卖现象

# 实现可重入锁-lua脚本+Hash
由于上述加锁命令使用了 SETNX ，一旦键存在就无法再设置成功，这就导致后续同一线程内继续加锁，将会加锁失败。当一个线程执行一段代码成功获取锁之后，继续执行时，又遇到加锁的子任务代码，可重入性就保证线程能继续执行，而不可重入就是需要等待锁释放之后，再次获取锁成功，才能继续往下执行。

用一段 Java 代码解释可重入：

```java
public synchronized void a() {  
    b();  
}  
​  
public synchronized void b() {  
    // pass  
}
```

假设 X 线程在 a 方法获取锁之后，继续执行 b 方法，如果此时不可重入，线程就必须等待锁释放，再次争抢锁。

锁明明是被 X 线程拥有，却还需要等待自己释放锁，这就又会造成死锁

可重入性就可以解决这个尴尬的问题，当线程拥有锁之后，往后再遇到加锁方法，直接将加锁次数加 1，然后再执行方法逻辑。退出加锁方法之后，加锁次数再减 1，当加锁次数为 0 时，锁才被真正的释放。

可以看到可重入锁最大特性就是计数，计算加锁的次数。所以当可重入锁需要在分布式环境实现时，我们也就需要统计加锁次数。

解决方案：redis + Hash
key: lock，Hash的key
arg1: uuid+threadID，Hash的Hash1，可重入计数
arg2: expire 30

### 加锁脚本

Redis 提供了 Hash （哈希表）这种可以存储键值对数据结构。所以我们可以使用 Redis Hash 存储的锁的重入次数，然后利用 lua 脚本判断逻辑。
1. 如果锁不存在，则设置锁hset，设置过期时间
2. 如果锁存在，则重入锁hincrby，重置过期时间
```shell
if (redis.call('exists', 'lock') == 0 
then  
    redis.call('hset', 'lock', uuid, 1);  
    redis.call('expire', 'lock', 30);  
    return 1; 
else if redis.call('hexists', KEYS[1], ARGV[1]) == 1
then
    redis.call('hincrby', 'lock', uuid, 1);  
    redis.call('expire', 'lock', 30);  
    return 1; 
else  
    return 0;  
end
```
因为hincrby可以将hset合并，再将脚本配置为动态参数，因此上述脚本优化为
```shell
if (redis.call('exists', KEYS[1]) == 0 or redis.call('hexists', KEYS[1], ARGV[1]) == 1)   
then  
    redis.call('hincrby', KEYS[1], ARGV[1], 1);  
    redis.call('expire', KEYS[1], ARGV[2]);  
    return 1;  
else  
    return 0;  
end
```
## 解锁脚本
```shell
-- 判断 hash set 可重入 key 的值是否等于 0
-- 返回 nil 代表 自己的锁已不存在，在尝试解其他线程的锁，解锁失败
-- 返回 0 代表 可重入次数被减 1
-- 返回 1 代表 该可重入 key 解锁成功
if(redis.call('hexists', KEYS[1], ARGV[1]) == 0) then 
    return nil; 
else if(redis.call('hincrby', KEYS[1], ARGV[1], -1) > 0) then 
    return 0; 
else 
    redis.call('del', KEYS[1]); 
    return 1; 
end;
```

## java代码
由于加解锁代码量相对较多，这里可以封装成工具类并引入工厂模式
工厂DistributedLockClient，便于框架扩展
```java
@Component
public class DistributedLockClient {

    @Autowired
    private StringRedisTemplate redisTemplate;

    private String uuid;

    public DistributedLockClient() {
        this.uuid = UUID.randomUUID().toString();
    }

    public DistributedRedisLock getRedisLock(String lockName){
        return new DistributedRedisLock(redisTemplate, lockName, uuid);
    }
}
```
DistributedRedisLock实现如下：
==只有保证可重入方法获取到的uuid和外面方法的uuid相同，才代表同一把锁，才能够重入，所以上面的uuid必须只生成一次（Spring单实例DistributedLockClient）==
但是这又带来了新的问题，并发线程获取的锁如何区分唯一性？
解决方案：
- uuid不再作为锁的唯一标识，而是作为服务的唯一标识，这可以保证集群情况下并发线程获取的锁id具有唯一性
- 新的uuid由uuid和当前线程id拼接而成this.uuid = uuid + ":" + Thread.currentThread().getId()，这可以保证可重入锁
```java
public class DistributedRedisLock implements Lock {

    private StringRedisTemplate redisTemplate;

    private String lockName;

    private String uuid;

    private long expire = 30;

    public DistributedRedisLock(StringRedisTemplate redisTemplate, String lockName, String uuid) {
        this.redisTemplate = redisTemplate;
        this.lockName = lockName;
        this.uuid = uuid;
    }

    @Override
    public void lock() {
        this.tryLock();
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        try {
            return this.tryLock(-1L, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 加锁方法
     * @param time
     * @param unit
     * @return
     * @throws InterruptedException
     */
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        if (time != -1){
            this.expire = unit.toSeconds(time);
        }
        String script = "if redis.call('exists', KEYS[1]) == 0 or redis.call('hexists', KEYS[1], ARGV[1]) == 1 " +
                "then " +
                "   redis.call('hincrby', KEYS[1], ARGV[1], 1) " +
                "   redis.call('expire', KEYS[1], ARGV[2]) " +
                "   return 1 " +
                "else " +
                "   return 0 " +
                "end";
        while (!this.redisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList(lockName), getId(), String.valueOf(expire))){
            Thread.sleep(50);
        }
        return true;
    }

    /**
     * 解锁方法
     */
    @Override
    public void unlock() {
        String script = "if redis.call('hexists', KEYS[1], ARGV[1]) == 0 " +
                "then " +
                "   return nil " +
                "elseif redis.call('hincrby', KEYS[1], ARGV[1], -1) == 0 " +
                "then " +
                "   return redis.call('del', KEYS[1]) " +
                "else " +
                "   return 0 " +
                "end";
        Long flag = this.redisTemplate.execute(new DefaultRedisScript<>(script, Long.class), Arrays.asList(lockName), getId());
        if (flag == null){
            throw new IllegalMonitorStateException("this lock doesn't belong to you!");
        }
    }

    @Override
    public Condition newCondition() {
        return null;
    }

    /**
     * 给线程拼接唯一标识
     * @return
     */
    String getId(){
        return uuid + ":" + Thread.currentThread().getId();
    }
}
```
在业务代码中使用，测试了5000并发完全扛得住
```java
public void deduct() {
    DistributedRedisLock redisLock = this.distributedLockClient.getRedisLock("lock");
    redisLock.lock();

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
    } finally {
        redisLock.unlock();
    }
}
```
可重入性测试：
```java
public void deduct() {
    DistributedRedisLock redisLock = this.distributedLockClient.getRedisLock("lock");
    redisLock.lock();

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
        redisLock.unlock();
    }
}

public void test() {
    DistributedRedisLock redisLock = this.distributedLockClient.getRedisLock("lock");
    redisLock.lock();
	System.out.println("测试可重入锁");
	redisLock.unlock();
}
```
# 自动续期-lua脚本+Timer
问题：业务比较长，一开始设定的过期时间可能不够用
解决：自动续期
## 自动续期脚本
```java
if(redis.call('hexists', KEYS[1], ARGV[1]) == 1) then 
    redis.call('expire', KEYS[1], ARGV[2]); 
    return 1; 
else 
    return 0; 
end
```
key: lock
arg: uuid+threadID
arg: expire 30
## Java代码
在工具类RedisDistributeLock中添加renewExpire方法，由于自动续期写在了定时器Timer子线程当中，会导致uuid+threadID是子线程的uuid+threadID
因此，我们应当把拼串操作uuid+threadID提前到DistributedRedisLock构造器
```java
public class DistributedRedisLock implements Lock {

    private StringRedisTemplate redisTemplate;

    private String lockName;

    private String uuid;

    private long expire = 30;

    public DistributedRedisLock(StringRedisTemplate redisTemplate, String lockName, String uuid) {
        this.redisTemplate = redisTemplate;
        this.lockName = lockName;
        this.uuid = uuid + ":" + Thread.currentThread().getId();
    }

    @Override
    public void lock() {
        this.tryLock();
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        try {
            return this.tryLock(-1L, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 加锁方法
     * @param time
     * @param unit
     * @return
     * @throws InterruptedException
     */
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        if (time != -1){
            this.expire = unit.toSeconds(time);
        }
        String script = "if redis.call('exists', KEYS[1]) == 0 or redis.call('hexists', KEYS[1], ARGV[1]) == 1 " +
                "then " +
                "   redis.call('hincrby', KEYS[1], ARGV[1], 1) " +
                "   redis.call('expire', KEYS[1], ARGV[2]) " +
                "   return 1 " +
                "else " +
                "   return 0 " +
                "end";
        while (!this.redisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList(lockName), uuid, String.valueOf(expire))){
            Thread.sleep(50);
        }
        // 加锁成功，返回之前，开启定时器自动续期
        this.renewExpire();
        return true;
    }

    /**
     * 解锁方法
     */
    @Override
    public void unlock() {
        String script = "if redis.call('hexists', KEYS[1], ARGV[1]) == 0 " +
                "then " +
                "   return nil " +
                "elseif redis.call('hincrby', KEYS[1], ARGV[1], -1) == 0 " +
                "then " +
                "   return redis.call('del', KEYS[1]) " +
                "else " +
                "   return 0 " +
                "end";
        Long flag = this.redisTemplate.execute(new DefaultRedisScript<>(script, Long.class), Arrays.asList(lockName), uuid);
        if (flag == null){
            throw new IllegalMonitorStateException("this lock doesn't belong to you!");
        }
    }

    @Override
    public Condition newCondition() {
        return null;
    }

    // String getId(){
    //     return this.uuid + ":" + Thread.currentThread().getId();
    // }

    private void renewExpire(){
        String script = "if redis.call('hexists', KEYS[1], ARGV[1]) == 1 " +
                "then " +
                "   return redis.call('expire', KEYS[1], ARGV[2]) " +
                "else " +
                "   return 0 " +
                "end";
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                if (redisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList(lockName), uuid, String.valueOf(expire))) {
                    renewExpire();
                }
            }
        }, this.expire * 1000 / 3);
    }
}
```
用户程序添加睡眠业务，用来测试自动续期：
```java
try {  
    TimeUnit.SECONDS.sleep(1000);  
} catch (InterruptedException e) {  
    throw new RuntimeException(e);  
}
```
完整代码如下：
```java
package org.lyflexi.redistemplate.service;  

@Service  
public class StockService {  
  
    @Autowired  
    StringRedisTemplate redisTemplate;  
    @Autowired  
    DistributedLockClient distributedLockClient;  
    public void deduct() {  
        DistributedRedisLock redisLock = this.distributedLockClient.getRedisLock("lock");  
        redisLock.lock();  
  
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
            try {  
                TimeUnit.SECONDS.sleep(1000);  
            } catch (InterruptedException e) {  
                throw new RuntimeException(e);  
            }  
        } finally {  
            redisLock.unlock();  
        }  
    }  
    public void test() {  
        DistributedRedisLock redisLock = this.distributedLockClient.getRedisLock("lock");  
        redisLock.lock();  
        System.out.println("测试可重入锁");  
        redisLock.unlock();  
    }  
}
```

# 红锁算法
生产中redis往往会做集群，在redis集群状态下，我们的分布式锁存在以下的问题：
1. 并发线程A从master获取到锁，master会默认根据rdb或者aof向各个slave节点同步并发线程的锁信息
2. 在master将锁同步到slave之前，master宕掉了。
3. slave节点被晋级为master节点，
4. 但由于步骤2，此时的新晋master并不知道锁已经被并发线程A拿到了，因此并发线程B又重复获取了并发线程A持有的锁，又会出现并发超卖问题

解决集群下锁失效，参照redis官方网站针对redlock文档：https://redis.io/topics/distlock

在算法的分布式版本中，我们假设有N个Redis服务器，这些节点是完全独立的。将N设置为5是一个合理的值，因此需要在不同的计算机或虚拟机上运行5个Redis主服务器，确保它们以独立的方式发生故障。
![[Pasted image 20240123185647.png]]
为了获取锁，应用程序并发线程执行以下操作：
1. 并发线程以毫秒为单位获取当前时间的时间戳，作为起始时间。
2. 并发线程尝试在所有N个实例中顺序使用相同的键名、相同的随机值来获取锁定。每个实例尝试获取锁都需要时间，客户端应该设置一个远小于总锁定时间的超时时间。例如，如果自动释放时间为30秒，则尝试获取锁的超时时间不能超过30秒。这样可以防止并发线程长时间与故障状态的Redis节点进行通信：如果某个实例不可用，尽快尝试与下一个实例进行通信。
3. 计算获取锁所花费的时间，用当前时间（步骤3的当前时间），减去在步骤1中获得的起始时间。当且仅当客户端能够在半数以上实例（至少3个）中获取锁时，并且获取锁所花费的总时间小于锁有效时间，则认为已获取锁。
4. 如果获取了锁，计算剩余锁定时间，用锁的有效时间，减去步骤3中计算出的获取锁所花费的时间。
5. 如果客户端由于某种原因（无法锁定N / 2 + 1个实例或有效时间为负）而未能获得该锁，它将尝试解锁所有实例（即使没有锁定成功的实例）。