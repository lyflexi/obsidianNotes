对于update形如count = count - 1操作，MySQL会自动的给行记录上锁，因此我们安心的减当前商品行的stock字段即可
```java
@Repository  
public interface StockMapper {  
    @Update(" update db_stock set count = count - {#count} where product_code = #{#productCode} and count>={#count}")  
    public void updateStock(@Param("productCode") Integer productCode,@Param("count") Integer count) ;  
}
```
Java业务代码，设置每次减1件库存
```java
@Service  
public class OneSqlService {  
  
    @Autowired  
    StockMapper stockMapper;  
      
    @Transactional  
    public void deduct(){  
        stockMapper.updateStock(1001,1);  
    }  
}
```