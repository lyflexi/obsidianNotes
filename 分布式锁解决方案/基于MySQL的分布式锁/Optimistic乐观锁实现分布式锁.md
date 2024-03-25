MySQL本身并没有实现乐观锁，我们需要手动在表中添加版本号来实现cas：
- 比较并交换
- 交换失败则重试，递归无需while或者自旋需要while
```java
package org.lyflexi.mysqlock.optimisticLocking;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.transaction.annotation.Transactional;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/25 23:44  
 */public class OptimisticService {  
  
    @Autowired  
    StockMapper stockMapper;  
  
  
    public void checkAndLock() {  
  
        // 先查询库存是否充足  
        Stock stock = this.stockMapper.selectById(1L);  
  
        // 再减库存  
        if (stock != null && stock.getCount() > 0){  
            // 获取版本号  
            Long version = stock.getVersion();  
  
            stock.setCount(stock.getCount() - 1);  
            // 每次更新 版本号 + 1            
            stock.setVersion(stock.getVersion() + 1);  
            // 更新之前先判断是否是之前查询的那个版本，如果不是重试  
            if (this.stockMapper.updateStock(stock)==0) {//返回0表示cas失败  
                checkAndLock();  
            }  
        }  
    }  
}
```
相应的sql如下:
```java
/**  
 * @Author: ly  
 * @Date: 2024/3/25 21:36  
 */@Repository  
public interface StockMapper {  
  
    @Update(" select * from db_stock where id = #{id}")  
    public Stock selectById(@Param("id") Long id) ;  
    @Update(" update db_stock set db_stock.stock = #{stock.stock} where db_stock.stock = #{stock.id} and db_stock.versoin = #{stock.version}")  
    public int updateStock(@Param("stock") Stock stock) ;  
}
```
javaBean如下：
```java
package org.lyflexi.mysqlock.optimisticLocking;  
  
import lombok.Data;  
  
@Data  
public class Stock {  
    private Integer productCode;  
    private Long version ;  
    private Integer count;  
  
}
```
