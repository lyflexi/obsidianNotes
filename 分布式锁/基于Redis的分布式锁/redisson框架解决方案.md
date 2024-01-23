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