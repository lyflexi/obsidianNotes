在同一个事务当中
- 先select ...  for update，获得行锁
- 再执行无锁的更新操作
```java
package org.lyflexi.mysqlock.forUpdateLock;  
  
import org.lyflexi.mysqlock.pojo.Stock;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.transaction.annotation.Transactional;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/25 23:44  
 */public class ForUpdateService {  
  
    @Autowired  
    StockMapper stockMapper;  
  
    @Transactional//开启事务  
    public void deduct(){  
        //先加锁  
        Stock stock = stockMapper.selectStockForUpdate(1001);  
        if (stock!=null&&stock.getStock()>0){  
            //不在sql里做stock-1操作，因此下面是无锁更新  
            stock.setStock(stock.getStock()-1);  
            stockMapper.updateStock(stock);  
        }  
    }  
}
```
对应的sql语句如下：
```java
/**  
 * @Author: ly  
 * @Date: 2024/3/25 21:36  
 */@Repository  
public interface StockMapper {  
  
    @Update(" select * from db_stock where id = #{id} for update")  
    public Stock selectStockForUpdate(@Param("id") Integer id) ;  
    @Update(" update db_stock set db_stock.stock = #{stock.stock}")  
    public void updateStock(@Param("stock") Stock stock) ;  
}
```



