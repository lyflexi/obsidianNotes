
# 场景一：Spring当中配置了多例模式
```java
package org.lyflexi.jvmlock.service;  
  
import org.lyflexi.jvmlock.pojo.Stock;  
import org.springframework.stereotype.Service;  
  
import java.util.concurrent.locks.ReentrantLock;  
  
@Service
//在springboot中，如果我们的服务类没有实现接口，要想开启多例模式还要设置proxyMode = ScopedProxyMode.TARGET_CLASS
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class StockService {  
  
    private Stock stock = new Stock();  
  
    private ReentrantLock lock = new ReentrantLock();  
  
    public void deduct(){  
        lock.lock();  
        try {  
            stock.setStock(stock.getStock() - 1);  
            System.out.println("库存余量：" + stock.getStock());  
        } finally {  
            lock.unlock();  
        }  
    }  
}

```

为了方便起见，我就不连数据库了，直接定义个pojo，这跟MySQL一个道理
```java
@Data  
public class Stock {  
  
    private Integer stock = 5000;  
}
```
# 场景二：开启了事务
我们的解锁代码在事务commit之前生效，这导致了默认隔离级别RR下，会导致其他并发请求读取未提交的数据，又造成了超卖现象
```java
@Service
public class StockService {  
  
    private Stock stock = new Stock();  
  
    private ReentrantLock lock = new ReentrantLock();  
	@Transaction
    public void deduct(){  
        lock.lock();  
        try {  
            stock.setStock(stock.getStock() - 1);  
            System.out.println("库存余量：" + stock.getStock());  
        } finally {  
            lock.unlock();  
        }  
    }  
}

@Data  
public class Stock {  
  
    private Integer stock = 5000;  
}
```
解决方法，在事务方法前后加锁
```java
lock.lock();
service.deduct()
lock.unlock(); 
//或者
synchronized (this) {
	service.deduct()
}

```
# 场景三：分布式场景
详见分布式锁解决方案