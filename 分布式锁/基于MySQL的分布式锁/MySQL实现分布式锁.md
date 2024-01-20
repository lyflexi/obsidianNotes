MySQL实现思路
1. 创建mysql表tb_lock用于并发占坑，比如给lock_name字段创建唯一性索引，线程尝试同时获取tb_lock的锁（insert）
2. 创建mysql表tb_service，这是我们真正的业务数据表
3. 获取tb_lock的锁成功，则执行tb_service表的业务逻辑，执行完成释放锁（delete）
4. 其他线程等待重试

![[1606620944823.png]]
相关代码示例：
```java
@Service
public class StockService {
	@Autowired
	private StockMapper stockMapper;
	@Autowired
	private LockMapper lockMapper;
	/**
	* 数据库分布式锁
	*/
	public void checkAndLock() {
		// 加锁
		Lock lock = new Lock(null, "lock", this.getClass().getName(), new Date(), null);
		try {
			this.lockMapper.insert(lock);
		} catch (Exception ex) {
			// 获取锁失败，则重试
			try {
				Thread.sleep(50);
				this.checkAndLock();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		// 先查询库存是否充足
		Stock stock = this.stockMapper.selectById(1L);
		// 再减库存
		if (stock != null && stock.getCount() > 0){
			stock.setCount(stock.getCount() - 1);
			this.stockMapper.updateById(stock);
		}
		// 释放锁
		this.lockMapper.deleteById(lock.getId());
	}
}
```
使用Jmeter压力测试结果：
![[Pasted image 20240120221401.png]]
可以看到性能感人。mysql数据库库存余量为0，可以保证线程安全。


缺点：
1. 这把锁强依赖数据库的可用性，数据库是一个单点，一旦数据库挂掉，会导致业务系统不可用。
    
    解决方案：给 锁数据库 搭建主备
    
2. 这把锁没有失效时间，一旦解锁操作失败，就会导致锁记录一直在数据库中，其他线程无法再获得到锁。
    
    解决方案：只要做一个定时任务，每隔一定时间把数据库中的超时数据清理一遍。
    
3. 这把锁是非重入的，同一个线程在没有释放锁之前无法再次获得该锁。因为数据中数据已经存在了。
    
    解决方案：记录获取锁的主机信息和线程信息，如果相同线程要获取锁，直接重入。
    
4. 受制于数据库性能，并发能力有限。mysql太慢了！