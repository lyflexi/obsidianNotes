
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
# redis乐观锁介绍
在当前事务执行期间：
- 如果没有其他的并发事务修改，则当前事务执行成功
- 如果没有其他的并发事务修改，则当前事务执行失败，但是其他并发事务的修改执行成功
watch指令用于监控
multi指令用于开启redis事务
exec用于关闭redis事务
如，正常情况没有其他的并发事务修改，当前事务执行成功返回ok
![[Pasted image 20240123103052.png]]
如，异常情况有其他的并发事务修改，会造成当前事务执行失败，取消当前事务并返回nil
![[Pasted image 20240123105107.png]]
![[Pasted image 20240123105137.png]]
![[Pasted image 20240123105244.png]]
# redis乐观锁Java代码
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