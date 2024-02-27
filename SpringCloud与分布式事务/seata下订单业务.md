业务背景：首先我们看看项目中的下单业务整体流程：
![[Pasted image 20240127100701.png]]
由于订单、购物车、商品分别在三个不同的微服务，而每个微服务都有自己独立的数据库，因此下单过程中就会跨多个数据库完成业务。而每个微服务都会执行自己的本地事务：
- 交易服务：下单事务
- 购物车服务：清理购物车事务
- 库存服务：扣减库存事务==（有可能此分支事务失败）==
每个微服务的本地事务，也可以称为**分支事务**。整个业务中多个有关联的分支事务一起就组成了**全局事务**。我们必须保证整个全局事务同时成功或失败。

我们知道每一个分支事务就是传统的单体事务，都可以满足ACID特性，但全局事务跨越多个服务、多个数据库，是否还能满足呢？

我们来做一个测试，先进入购物车页面：
![[Pasted image 20240127100807.png]]
目前有4个购物车，我们选择其中的两种商品结算下单，进入订单结算页面：
![[Pasted image 20240127100818.png]]
然后将下单的某个商品的库存修改为`0`：
![[Pasted image 20240127100831.png]]
然后，提交订单，最终因库存不足导致下单失败：
![[Pasted image 20240127100840.png]]
然后我们去查看购物车列表，发现购物车数据依然被清空了，并未回滚：
![[Pasted image 20240127100850.png]]
事务并未遵循ACID的原则，归其原因就是参与事务的多个子业务在不同的微服务，跨越了不同的数据库。虽然每个单独的业务都能在本地遵循ACID，但是它们互相之间没有感知，不知道有人失败了，无法保证最终结果的统一，也就无法遵循ACID的事务特性了。

这就是分布式事务问题，出现以下情况之一就可能产生分布式事务问题：
- 业务跨多个服务实现
- 业务跨多个数据源实现
接下来这一章我们就一起来研究下如何解决分布式事务问题。
# Seata的工作架构
https://seata.io/zh-cn/docs/overview/what-is-seata.html
其实分布式事务产生的一个重要原因，就是参与事务的多个分支事务互相无感知，不知道彼此的执行状态。因此解决分布式事务的思想非常简单：

就是找一个统一的事务协调者，与多个分支事务通信，检测每个分支事务的执行状态，保证全局事务下的每一个分支事务同时成功或失败即可。大多数的分布式事务框架都是基于这个理论来实现的。

Seata也不例外，在Seata的事务管理中有三个重要的角色：

- TC (Transaction Coordinator) - 事务协调者：维护全局和分支事务的状态，协调全局事务提交或回滚。
- TM (Transaction Manager) - 事务管理器全局事务：定义全局事务的范围、开始全局事务、提交或回滚全局事务。
- RM (Resource Manager) - 资源管理器分支事务：管理分支事务，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。
Seata的工作架构如图所示：
![[Pasted image 20240127101041.png]]
其中，TM**和**RM可以理解为Seata的客户端部分，引入Seata依赖到参与事务的微服务中即可。将来TM和RM就会协助微服务，实现本地分支事务与TC之间交互，实现事务的提交或回滚。

而**TC**服务则是事务协调中心，是一个独立的微服务，需要单独部署。见我的Docker应用部署篇
# 微服务集成Seata客户端

参与分布式事务的每一个微服务都需要集成Seata
## trade-service
引入依赖：
```XML
  <!--seata-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
  </dependency>
```
配置文件：
```yaml
seata:  
  registry: # TC服务注册中心的配置，微服务根据这些信息去注册中心获取tc服务地址  
    type: nacos # 注册中心类型 nacos    nacos:  
      server-addr: 192.168.18.100:8848 # nacos地址  
      namespace: "" # namespace，默认为空  
      group: DEFAULT_GROUP # 分组，默认是DEFAULT_GROUP  
      application: seata-server # seata服务名称  
      username: nacos  
      password: nacos  
  tx-service-group: hmall # 事务组名称  
  service:  
    vgroup-mapping: # 事务组与tc集群的映射关系  
      hmall: "default"
```
## cart-service
引入依赖：
```XML
  <!--seata-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
  </dependency>
```
配置文件：
```yaml
seata:  
  registry: # TC服务注册中心的配置，微服务根据这些信息去注册中心获取tc服务地址  
    type: nacos # 注册中心类型 nacos    nacos:  
      server-addr: 192.168.18.100:8848 # nacos地址  
      namespace: "" # namespace，默认为空  
      group: DEFAULT_GROUP # 分组，默认是DEFAULT_GROUP  
      application: seata-server # seata服务名称  
      username: nacos  
      password: nacos  
  tx-service-group: hmall # 事务组名称  
  service:  
    vgroup-mapping: # 事务组与tc集群的映射关系  
      hmall: "default"
```
## item-service
我们以`trade-service`为例。
引入依赖：
```XML
  <!--seata-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
  </dependency>
```
配置文件：
```yaml
seata:  
  registry: # TC服务注册中心的配置，微服务根据这些信息去注册中心获取tc服务地址  
    type: nacos # 注册中心类型 nacos    nacos:  
      server-addr: 192.168.18.100:8848 # nacos地址  
      namespace: "" # namespace，默认为空  
      group: DEFAULT_GROUP # 分组，默认是DEFAULT_GROUP  
      application: seata-server # seata服务名称  
      username: nacos  
      password: nacos  
  tx-service-group: hmall # 事务组名称  
  service:  
    vgroup-mapping: # 事务组与tc集群的映射关系  
      hmall: "default"
```

重启上述三个服务，并查看TC服务的日志
```shell
docker logs -f seata
```
发现trade-service、cart-service、item-service的TM还有RM就有注册成功的提示
![[Pasted image 20240127100611.png]]

# XA二阶段提交
Seata对原始的XA模式做了简单的封装和改造，以适应自己的事务模型，基本架构如图：
![[Pasted image 20240127115543.png]]
`RM`一阶段的工作：
1. 注册分支事务到`TC`
2. 执行分支业务sql但不提交，只是向报告执行状态`TC`
3. 傻等TC的全局性反馈

`RM`一阶段的工作：
1. 注册分支事务到`TC`
2. 执行分支业务sql但不提交，只是向报告执行状态`TC`
3. 傻等TC的全局性反馈
  
`TC`二阶段的工作：
1. `TC`检测各分支事务执行状态
    1. 如果都成功，通知所有RM提交事务
    2. 如果有失败，通知所有RM回滚事务

`RM`二阶段的工作：
- 接收`TC`指令，提交或回滚事务
## 开启XA模式
trade-service、cart-service、item-service三服务均开启XA模式
```yaml
seata:
	data-source-proxy-mode: XA #开启数据源代理的XA模式
```
trade-service创建订单接口开启全局事务@GlobalTransactional
```java
    @Override  
//    @Transactional  
	//开启全局事务
    @GlobalTransactional  
    public Long createOrder(OrderFormDTO orderFormDTO) {  
        // 1.订单数据  
        Order order = new Order();  
        // 1.1.查询商品  
        List<OrderDetailDTO> detailDTOS = orderFormDTO.getDetails();  
        // 1.2.获取商品id和数量的Map  
        Map<Long, Integer> itemNumMap = detailDTOS.stream()  
                .collect(Collectors.toMap(OrderDetailDTO::getItemId, OrderDetailDTO::getNum));  
        Set<Long> itemIds = itemNumMap.keySet();  
        // 1.3.查询商品  
        List<ItemDTO> items = itemClient.queryItemByIds(itemIds);  
        if (items == null || items.size() < itemIds.size()) {  
            throw new BadRequestException("商品不存在");  
        }  
        // 1.4.基于商品价格、购买数量计算商品总价：totalFee  
        int total = 0;  
        for (ItemDTO item : items) {  
            total += item.getPrice() * itemNumMap.get(item.getId());  
        }  
        order.setTotalFee(total);  
        // 1.5.其它属性  
        order.setPaymentType(orderFormDTO.getPaymentType());  
        order.setUserId(UserContext.getUser());  
        order.setStatus(1);  
        // 1.6.将Order写入数据库order表中  
        save(order);  
  
        // 2.保存订单详情  
        List<OrderDetail> details = buildDetails(order.getId(), items, itemNumMap);  
        detailService.saveBatch(details);  
  
        // 3.清理购物车商品  
        cartClient.deleteCartItemByIds(itemIds);  
  
  
        // 4.扣减库存  
        try {  
            itemClient.deductStock(detailDTOS);  
        } catch (Exception e) {  
            throw new RuntimeException("库存不足！");  
        }  
        return order.getId();  
  
    }
```
cart-service删除购物车接口开启普通事务
```java
@Override  
@Transactional  
public void removeByItemIds(Collection<Long> itemIds) {  
    // 1.构建删除条件，userId和itemId  
    QueryWrapper<Cart> queryWrapper = new QueryWrapper<Cart>();  
    queryWrapper.lambda()  
            .eq(Cart::getUserId, UserContext.getUser())  
            .in(Cart::getItemId, itemIds);  
    // 2.删除  
    remove(queryWrapper);  
}
```
item-service扣库存业务开启普通事务
```java
    @Override  
    @Transactional    
    public void deductStock(List<OrderDetailDTO> items) {  
        String sqlStatement = "com.hmall.mapper.item.ItemMapper.updateStock";  
        boolean r = false;  
        try {  
            r = executeBatch(items, (sqlSession, entity) -> sqlSession.update(sqlStatement, entity));  
        } catch (Exception e) {  
//            log.error("更新库存异常", e);  
//            return;  
            //这里要想让提交订单业务的分布式事务生效，必须抛出异常，不能私自捕获  
            throw new BizIllegalException("库存不足！");  
        }  
        if (!r) {  
            throw new BizIllegalException("库存不足！");  
        }  
    }
```
## XA演示
一开始康佳（KONIKA）的库存量为1000，用户添加了购物车，准备买5件
但是用户准备下单之前，数据库中康佳（KONIKA）的库存被改为了3
![[Pasted image 20240127105805.png]]

此时提交订单，必定失败
![[Pasted image 20240127105637.png]]分布式事务生效了
返回购物车页面，购物车业务也回滚，购物车没有被清除
![[Pasted image 20240127110134.png]]
![[Pasted image 20240127105859.png]]
同时看IDEA控制台购物车服务的日志，二阶段回滚Branch Rollbacked result: PhaseTwo_Rollbacked成功
```shell
11:44:00:113 DEBUG 17220 --- [nio-8082-exec-9] com.hmall.cart.mapper.CartMapper.delete  : ==>  Preparing: DELETE FROM cart WHERE (user_id = ? AND item_id IN (?,?))
11:44:00:113 DEBUG 17220 --- [nio-8082-exec-9] com.hmall.cart.mapper.CartMapper.delete  : ==> Parameters: 1(Long), 14741770661(Long), 100001905417(Long)
11:44:00:115 DEBUG 17220 --- [nio-8082-exec-9] com.hmall.cart.mapper.CartMapper.delete  : <==    Updates: 2
11:44:00:131  INFO 17220 --- [h_RMROLE_1_4_24] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:xid=192.168.18.100:8099:36511650651529232,branchId=36511650651529236,branchType=XA,resourceId=jdbc:mysql://192.168.18.100:3308/hm-cart,applicationData=null
11:44:00:132  INFO 17220 --- [h_RMROLE_1_4_24] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.18.100:8099:36511650651529232 36511650651529236 jdbc:mysql://192.168.18.100:3308/hm-cart
11:44:00:135  INFO 17220 --- [h_RMROLE_1_4_24] i.s.rm.datasource.xa.ResourceManagerXA   : 192.168.18.100:8099:36511650651529232-36511650651529236 was rollbacked
11:44:00:135  INFO 17220 --- [h_RMROLE_1_4_24] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
```

自然的，商品扣库存业务异常，扣库存业务也得回滚，数据库康佳（KONIKA）的库存量还是3，没有减成功：
![[Pasted image 20240127110100.png]]
另外，看看IDEA控制台商品服务的二阶段回滚Branch Rollbacked result: PhaseTwo_Rollbacked生效了
```shell
com.hmall.common.exception.BizIllegalException: 库存不足！
	at com.hmall.item.service.impl.ItemServiceImpl.deductStock(ItemServiceImpl.java:37) ~[classes/:na]
	at com.hmall.item.service.impl.ItemServiceImpl$$FastClassBySpringCGLIB$$fe4365a9.invoke(<generated>) ~[classes/:na]
	...

10:56:23:813  INFO 704 --- [h_RMROLE_1_1_24] i.s.c.r.p.c.RmBranchRollbackProcessor    : rm handle branch rollback process:xid=192.168.18.100:8099:36511650651529227,branchId=36511650651529228,branchType=XA,resourceId=jdbc:mysql://192.168.18.100:3308/hm-item,applicationData=null
10:56:23:815  INFO 704 --- [h_RMROLE_1_1_24] io.seata.rm.AbstractRMHandler            : Branch Rollbacking: 192.168.18.100:8099:36511650651529227 36511650651529228 jdbc:mysql://192.168.18.100:3308/hm-item
10:56:23:817  INFO 704 --- [h_RMROLE_1_1_24] i.s.rm.datasource.xa.ResourceManagerXA   : 192.168.18.100:8099:36511650651529227-36511650651529228 was rollbacked
10:56:23:817  INFO 704 --- [h_RMROLE_1_1_24] io.seata.rm.AbstractRMHandler            : Branch Rollbacked result: PhaseTwo_Rollbacked
10:58:52:603 DEBUG 704 --- [nio-8081-exec-6] c.h.i.mapper.ItemMapper.selectBatchIds   : ==>  Preparing: SELECT id,name,price,stock,image,category,brand,spec,sold,comment_count,isAD,status,create_time,update_time,creater,updater FROM item WHERE id IN ( ? , ? )
10:58:52:603 DEBUG 704 --- [nio-8081-exec-6] c.h.i.mapper.ItemMapper.selectBatchIds   : ==> Parameters: 14741770661(Long), 100001905417(Long)
10:58:52:605 DEBUG 704 --- [nio-8081-exec-6] c.h.i.mapper.ItemMapper.selectBatchIds   : <==      Total: 2
```

`XA`模式的优点是什么？
- 事务的强一致性，满足ACID原则
- 常用数据库都支持，实现简单，并且没有代码侵入

`XA`模式的缺点是什么？
- ==分布式feing接口调用需要等待二阶段结束才释放，因此一阶段需要锁定数据库资源一直傻等，性能较差==

同时XA需要依赖于数据库本身提供的事务机制

# AT二阶段提交
==Seata主推的是AT模式，AT模式同样是分阶段提交的事务模型，不过缺弥补了XA模型中资源锁定周期过长的缺陷。==
![[Pasted image 20240127120240.png]]
阶段一`RM`的工作：
- 注册分支事务
- 记录undo-log（数据快照）
- 执行业务sql并提交
- 报告事务状态

阶段二全局提交时`RM`的工作：
- 删除undo-log即可

阶段二全局回滚时`RM`的工作：
- 根据undo-log恢复数据到更新前

## 开启AT模式
trade-service、cart-service、item-service三服务均开启AT模式
```yaml
seata:
	data-source-proxy-mode: AT #开启数据源代理的XA模式
```
trade-service、cart-service、item-service三服务对应各自的数据库均导入seata-at.sql表，用于存储undolog
```sql
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8mb4 COMMENT ='AT transaction mode undo table';

```
![[Pasted image 20240127153043.png]]
## AT演示
我们用一个真实的业务来梳理下AT模式的原理。比如，现在有一个数据库表，记录用户余额：

|**id** |**money** |
|---|---|
|1|100|
其中一个分支业务要执行的SQL为：
```SQL
 update tb_account set money = money - 10 where id = 1
```
AT模式下，当前分支事务执行流程图如下：
![[Pasted image 20240127120947.png]]
一阶段：
1. `TM`发起并注册全局事务到`TC`
2. `TM`调用分支事务
3. 分支事务准备执行业务SQL
4. `RM`拦截业务SQL，根据where条件查询原始数据，形成快照。
```JSON
{
  "id": 1, "money": 100
}
```
5. `RM`执行业务SQL，提交本地事务，释放数据库锁。此时 money = 90
6. `RM`报告本地事务状态给`TC`

二阶段：
1. `TM`通知`TC`事务结束
2. `TC`检查分支事务状态
    1. 如果都成功，则立即删除快照
    2. ==如果有其他任意分支事务失败==，则所有分支事务都需要回滚。当前事务读取快照数据（{"id": 1, "money": 100}），将快照恢复到数据库。此时数据库再次恢复为100


简述`AT`模式与`XA`模式最大的区别是什么？
- `XA`模式一阶段不提交事务，锁定资源；`AT`模式一阶段直接提交，不锁定资源。
- `XA`模式依赖分支数据库事务实现回滚；`AT`模式利用数据快照实现数据回滚。
- `XA`模式强一致；由于`AT`模式快照的补偿机制会存在短暂的不一致现象，又叫最终一致

可见AT模式性能更好，因此企业90%的分布式事务都可以用AT模式来解决。
==但是AT 模式同样也要基于支持本地 ACID 事务的关系型数据库==
# 手工补偿TCC（了解）
根据两阶段行为模式的不同，我们将分支事务划分为 Automatic (Branch) Transaction Mode 和 Manual (Branch) Transaction Mode.

AT 模式基于支持本地 ACID 事务的关系型数据库：

- 一阶段 prepare 行为：在本地事务中，一并提交业务数据更新和相应回滚日志记录。
- 二阶段 commit 行为：马上成功结束，**自动**异步批量清理回滚日志。
- 二阶段 rollback 行为：通过回滚日志，**自动**生成补偿操作，完成数据回滚。

相应的，TCC 模式，不依赖于底层数据资源的事务支持：

- 一阶段 try 行为：调用 **自定义** 的 prepare 逻辑。
- 二阶段 confirm 行为：调用 **自定义** 的 commit 逻辑，表示成功。
- 二阶段 cancel 行为：调用 **自定义** 的 rollback 逻辑，表示失败。

所谓 TCC 模式，是指支持把 **自定义** 的分支事务纳入到全局事务的管理中，TCC的手动补流程图如下：
![[Pasted image 20240127140357.png]]
第一个分支事务异常导致全局事务回滚，但是另外一个分支是未执行`try`操作的，直接执行了`cancel`操作反而会导致数据错误。因此，这种情况下，尽管`cancel`方法要执行，但其中不能做任何回滚操作，这就是**空回滚**。

对于整个空回滚的分支事务，将来try方法阻塞结束依然会执行。但是整个全局事务其实已经结束了，因此永远不会再有confirm或cancel，也就是说这个事务执行了一半，处于**悬挂状态**，这就是业务悬挂问题。

以上问题都需要我们在手工编写try、cancel方法时处理。


# 长事务治理SAGA（了解）

Saga的核心思想是将长事务T拆分为多个本地短事务`{T1，T2，T3，Tn}`依次执行，此后任何一个子事务执行失败，Saga事务协调器就会反序调用`rollback`补偿操作`{C3，C2，C1}`。拆分后的本地事务无锁，子事务只需按序执行`commit`，不用像XA事务那样长期锁定资源账户Accont
![[Pasted image 20240127121359.png]]

# seata踩坑
分布式事务虽然生效了

但是目前TC也就是seata-server服务用到的seata数据库中的四张表一直没有数据

以及AT模式下，三大业务数据库trade-service、cart-service和trade-service中的各自额undolog表也是空的

暂时没有解决...
# 分布式事务的究极解决方案
说了这么多，其实分布式事务的究极解决方案就是尽量不要产生分布式事务。因为分布式事务底层涉及TM客户端与TC服务之间的网络通信，如果是AT的话还涉及undolog的创建，存储以及执行undolog，十分消耗资源与性能

消息队列通知cart-service和item-service执行各自的操作，cart-service和item-service服务接收到消息尝试执行删除购物车以及扣减库存，如果item-service服务扣减库存失败，人工捕获反复重试，直到成功为止，这就是兜底方案