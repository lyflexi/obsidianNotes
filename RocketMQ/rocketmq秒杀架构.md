内置tomcat默认支持200个并发线程，假设处理一个业务请求耗时50ms，则1秒内能够处理`20*200=4000`个请求，这就是QPS  
```yml
server:  
    tomcat:  
        threads:  
            max: 400 #调大为为400个并发线程
```
即使是堆tomcat，使用nginx做负载均衡，对于超大并发也是顶不住的，因为nginx本身也是有压力限制的
![[07-秒杀.jpg]]
# 秒杀架构设计
技术选型:springBoot + Redis + Mysql + RocketMq + Security ...  
  
设计: (秒杀抢优惠券..)  
- 设计seckill-web接收处理秒杀请求  ，对用户进行去重，并预扣减库存
- 设计seckill-service处理秒杀真实业务的  ，redis分布式锁
  
TOC项目部署细节:  
假设用户量: 50w，服务器带宽100M拉满，8核16G服务器6台，seckill-web : 4台    seckill-service 2台  
- 日活量: 5千-2.5万   1%-5%  ，这个统计数据库用户表的登录时间
- QPS约等于日活量: 2w左右   这个统计方式是自己打日志或者统计nginx的访问日志(access.log) ，对相同的时间log 2023-4-24 16:58:11 进行分组统计
技术要点：  
1.通过redis的setnx对用户和商品做去重判断，防止用户刷接口的行为  
2.每天晚上8点通过定时任务 把mysql中参与秒杀的库存商品，同步到redis中去，做库存的预扣减，提升接口性能  
3.通过RocketMq消息中间件的异步消息，来将秒杀的业务异步化，进一步提升性能  
4.seckill-service使用MQ的并发消费模式将消息并发的取到Java程序，并且设置合理的线程数量，快速处理队列中堆积的消息  
5.使用redis的分布式锁+自旋锁，对商品的库存进行并发控制，真正的减少db压力  
6.使用声明式事务注解Transactional，并且设置异常回滚类型，控制数据库的原子性操作  
7.使用jmeter压测工具，对秒杀接口进行压力测试，在8C16G的服务器上，qps2k+，达到压测预期  
8.使用sentinel的热点参数限流规则，针对爆款商品和普通商品的区别，区分限制
![[秒杀架构.png]]
![[Pasted image 20240123094757.png]]
![[Pasted image 20240123094811.png]]
# seckill-web模块
yml配置
```yml
server:  
    port: 8081  
    tomcat:  
        threads:  
            max: 400  
spring:  
    data:  
        redis:  
            host: 47.103.44.163  
            port: 6379  
            database: 0  
rocketmq:  
    name-server: 47.103.44.163:9876  
    producer:  
        group: seckill-producer-group  
#        access-key: rocketmq2  
#        secret-key: 12345678
```

SeckillController设计：
1. 根据从MySQL同步过来的库存，进行预扣减
2. 发送消息至seckill-service模块执行真正的扣减
```java
package org.lyflexi.seckill_web.controller;  

/**  
 * @Author: DLJD  
 * @Date: 2023/4/24  
 */@RestController  
public class SeckillController {  
  
  
    @Autowired  
    private StringRedisTemplate redisTemplate;  
  
    @Resource  
    private RocketMQTemplate rocketMQTemplate;  
  
    //CAS java无锁的   原子性 安全的  
    AtomicInteger userIdAt = new AtomicInteger(0);  
  
    /**  
     * 1.用户去重  
     * 2.库存的预扣减  
     * 3.消息放入mq  
     * 秒杀不是一个单独的系统  
     * 都是大项目的某一个小的功能模块  
     *  
     * @param goodsId  
     * @param userId  真实的项目中 要做登录的 不要穿这个参数  
     * @return  
     */  
    @GetMapping("seckill")  
    public String doSecKill(Integer goodsId /*, Integer userId*/) {  
        // log 2023-4-24 16:58:11  
        // log 2023-4-24 16:58:11        int userId = userIdAt.incrementAndGet();  
        // uk uniqueKey = [yyyyMMdd] +  userId + goodsId  
        String uk = userId + "-" + goodsId;  
        // 这一步使用setIfAbsent只是用来给秒杀用户去重，分布式锁在另外的seckill_service模块做的  
        Boolean flag = redisTemplate.opsForValue().setIfAbsent("uk:" + uk, "");  
        if (!flag) {  
            return "您已经参与过该商品的抢购，请参与其他商品O(∩_∩)O~";  
        }  
        //redis的线程安全操作api：decrement  
        Long restCount = redisTemplate.opsForValue().decrement("goodsId:" + goodsId);  
        if (restCount < 0) {  
            // 保证我的redis的库存 最小值是0  
            redisTemplate.opsForValue().increment("goodsId:" + goodsId);  
            return "该商品已经被抢完,下次早点来(●ˇ∀ˇ●)";  
        }  
        // 方mq 异步处理  
        rocketMQTemplate.asyncSend("seckillTopic3", uk, new SendCallback() {  
            @Override  
            public void onSuccess(SendResult sendResult) {  
                System.out.println("发送成功");  
            }  
  
            @Override  
            public void onException(Throwable throwable) {  
                System.out.println("发送失败:" + throwable.getMessage());  
                System.out.println("用户的id:" + userId + "商品id" + goodsId);  
            }  
        });  
        return "正在拼命抢购中,请稍后去订单中心查看";  
    }  
  
  
    /**  
     * 抢一个付费的商品  
     * 1.先扣减库存  再付费  | 如果不付费 库存需要回滚  
     * 2.先付费  再扣减库存  | 如果库存不足  则退费  
     */  
  
}
```

# seckill-service模块
## 数据同步DataSync
```java
package org.lyflexi.seckill_service.config;  
   
  
/**  
 * @Author: DLJD  
 * @Date: 2023/4/24  
 * 1.每天10点 晚上8点 通过定时任务 将mysql的库存 同步到redis中去  
 * 2.为了测试方便 希望项目启动的时候 就同步数据  
 */  
@Component  
public class DataSync {  
  
    @Autowired  
    private GoodsMapper goodsMapper;  
  
    @Autowired  
    private StringRedisTemplate redisTemplate;  
  
//    @Scheduled(cron = "0 0 10 0 0 ?")  
//    public void initData(){  
//    }  
  
    /**  
     * 为了测试方便，就不做定时任务了，我让这个方法在项目启动就以后  
     * 并且再这个类的属性注入完毕以后执行  
     * bean生命周期了  
     * 实例化 new  
     * 属性赋值  
     * 初始化  (前PostConstruct/中InitializingBean/后BeanPostProcessor)  
     * 使用  
     * 销毁  
     * ----------  
     * 定位不一样  
     */  
    @PostConstruct  
    public void initData() {  
        List<Goods> goodsList = goodsMapper.selectSeckillGoods();  
        if (CollectionUtils.isEmpty(goodsList)) {  
            return;  
        }  
        goodsList.forEach(goods -> {  
            redisTemplate.opsForValue().set("goodsId:" + goods.getGoodsId(), goods.getTotalStocks().toString());  
        });  
    }  
  
  
}
```
## 消费监听MQ
### 方案一：jvm锁
MySQL默认隔离级别是RR，所以一定要在事务块外面加锁：保证第一个并发先提交事务，第二个再获取锁，这样才可以实现并发安全，要不然锁不住  
```java
@Override  
public void onMessage(MessageExt message) {  
	String msg = new String(message.getBody());  
	// userId + "-" + goodsId  
	Integer userId = Integer.parseInt(msg.split("-")[0]);  
	Integer goodsId = Integer.parseInt(msg.split("-")[1]);  

	synchronized (this) {  
		goodsService.realSeckill(userId, goodsId);  
	}  
}
```
service：
```java
/**  
 *     *  常规的 方案  
 *  锁加载调用方法的地方 要加载事务外面  
 * @param userId  
 * @param goodsId  
 */  
@Override  
@Transactional(rollbackFor = Exception.class) // rr  
public void realSeckill(Integer userId, Integer goodsId) {  
	// 扣减库存  插入订单表  
	Goods goods = goodsMapper.selectByPrimaryKey(goodsId);  
	int finalStock = goods.getTotalStocks() - 1;  
	if (finalStock < 0) {  
		// 只是记录日志 让代码停下来   这里的异常用户无法感知  
		throw new RuntimeException("库存不足：" + goodsId);  
	}  
	goods.setTotalStocks(finalStock);  
	goods.setUpdateTime(new Date());  
	// update goods set stocks =  1 where id = 1  没有行锁，完全靠完全的jvm锁控制  
	int i = goodsMapper.updateByPrimaryKey(goods);  
	if (i > 0) {  
		Order order = new Order();  
		order.setGoodsid(goodsId);  
		order.setUserid(userId);  
		order.setCreatetime(new Date());  
		orderMapper.insert(order);  
	}  
}
```
另外jvm锁没法集群  ，当前服务的锁对象EntrySet/WaitSet的作用域是当前服务内部，无法适用于分布式的情况
### 方案二：MySQL分布式锁
mysql(行锁)   可以做分布式锁，对于update操作，必须是类似于update goods set total_stocks = total_stocks - 1 这种情况才能触发行锁
```java
@Override  
public void onMessage(MessageExt message) {  
    String msg = new String(message.getBody());  
    // userId + "-" + goodsId  
    Integer userId = Integer.parseInt(msg.split("-")[0]);  
    Integer goodsId = Integer.parseInt(msg.split("-")[1]);  
    goodsService.realSeckill(userId, goodsId);  
}
```
service：
```java
@Override  
@Transactional(rollbackFor = Exception.class)  
public void realSeckill(Integer userId, Integer goodsId) {  
    // update goods set total_stocks = total_stocks - 1 where goods_id = goodsId and total_stocks - 1 >= 0;  
    // 通过mysql来控制锁  
    int i = goodsMapper.updateStock(goodsId);  
    if (i > 0) {  
        Order order = new Order();  
        order.setGoodsid(goodsId);  
        order.setUserid(userId);  
        order.setCreatetime(new Date());  
        orderMapper.insert(order);  
    }  
}
```
xml：
```java
<update id="updateStock">  
  update  goods set total_stocks = total_stocks - 1 ,update_time = now() where goods_id = #{value} and total_stocks - 1 >= 0  
</update>
```
但mysql不适合并发较大场景  ，性能感人
### 方案三：Redis分布式自旋锁
分布式锁必须给定自动过期时间。如果没有定时自动解锁，假设第一个线程拿到redis锁后,还未释放锁,此时系统宕机,锁就永久存在redis,竞争线程即使在系统恢复后也无法获得锁
```java
package org.lyflexi.seckill_service.listener;  
  
import org.apache.rocketmq.common.message.MessageExt;  
import org.apache.rocketmq.spring.annotation.ConsumeMode;  
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;  
import org.apache.rocketmq.spring.core.RocketMQListener;  
import org.lyflexi.seckill_service.service.GoodsService;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.stereotype.Component;  
  
import java.time.Duration;  
  
/**  
 * @Author: DLJD  
 * @Date: 2023/4/24  
 */@Component  
@RocketMQMessageListener(topic = "seckillTopic3",  
        consumerGroup = "seckill-consumer-group2",  
        consumeMode = ConsumeMode.CONCURRENTLY,  
        consumeThreadNumber = 40  
)  
public class SeckillListener implements RocketMQListener<MessageExt> {  
  
    @Autowired  
    private GoodsService goodsService;  
  
    @Autowired  
    private StringRedisTemplate redisTemplate;  
  
    int ZX_TIME = 20000;  
  
    /**  
     * 扣减库存  
     * 写订单表  
     *  
     * @param message  
     */  
//    @Override  
//    public void onMessage(MessageExt message) {  
//        String msg = new String(message.getBody());  
//        // userId + "-" + goodsId  
//        Integer userId = Integer.parseInt(msg.split("-")[0]);  
//        Integer goodsId = Integer.parseInt(msg.split("-")[1]);  
//        // 方案一: MySQL默认隔离级别是RR，所以一定要在事务块外面加锁：保证第一个并发先提交事务，第二个再获取锁，这样才可以实现并发安全，要不然锁不住  
//        但是jvm锁没法集群  
    // jvm  EntrySet WaitSet  
//        synchronized (this) {  
//            goodsService.realSeckill(userId, goodsId);  
//        }  
//    }  
  
  
    // 方案二  分布式锁  mysql(行锁)   可以做分布式锁，但不适合并发较大场景  
//    @Override  
//    public void onMessage(MessageExt message) {  
//        String msg = new String(message.getBody());  
//        // userId + "-" + goodsId  
//        Integer userId = Integer.parseInt(msg.split("-")[0]);  
//        Integer goodsId = Integer.parseInt(msg.split("-")[1]);  
//        goodsService.realSeckill(userId, goodsId);  
//    }  
  
    // 方案三: redis setnx 分布式锁  压力会分摊到redis和程序中执行  缓解db的压力  
    @Override  
    public void onMessage(MessageExt message) {  
        String msg = new String(message.getBody());  
        Integer userId = Integer.parseInt(msg.split("-")[0]);  
        Integer goodsId = Integer.parseInt(msg.split("-")[1]);  
        int currentThreadTime = 0;  
        while (true) {  
            // redis分布式锁，这里给一个key的过期时间,可以避免死锁的发生  
            Boolean flag = redisTemplate.opsForValue().setIfAbsent("lock:" + goodsId, "", Duration.ofSeconds(30));  
            if (flag) {  
                // 拿到锁成功  
                try {  
                    goodsService.realSeckill(userId, goodsId);  
                    return;  
                } finally {  
                    // 删除  
                    redisTemplate.delete("lock:" + goodsId);  
                }  
            } else {  
                //自选次数+1  
                currentThreadTime += 200;  
                try {  
                    //这次没拿到锁，我们认定紧接着下次拿到锁的概率还是很小，所以我们让该线程睡一会  
                    Thread.sleep(200L);  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
            }  
        }  
    }  
  
}
```
# 演示现象
## 库存刷新
首先启动定时任务，将MySQL库存刷新到redis
![[Pasted image 20240123094703.png]]
## jmeter压测
然后jmeter压测3000并发给到seckill-web模块：
- seckill-web模块用户去重
- seckill-web模块预处理redis库存
- seckill-web模块发送成功”用户id+商品id“消息给seckill-service模块
![[Pasted image 20240123095624.png]]
![[Pasted image 20240123095100.png]]
![[Pasted image 20240123095357.png]]
## seckill-service验证
3000条消息已经完全被消费
![[Pasted image 20240123095529.png]]
数据库新增3000条订单
![[Pasted image 20240123095827.png]]
最重要的是库存表商品余量为0，没有超卖现象发生
![[Pasted image 20240123095931.png]]