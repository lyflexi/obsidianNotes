messageID是rocketmq默认给消息生成的ID，我们也可以自己手动的去给消息添加key，最好使用自己定义的key，原因如下：
- 虽然说messageID一定是全局唯一标识符，但是实际使用中，可能会存在相同的消息有两个不同messageID的情况，比如producer主动重发、或因客户端重投机制导致的重发等，这种情况就需要使业务字段进行重复消费。
- 另外，使用自定义key对我们程序员来说也是可控的
```java
Message(String topic, String tags, String keys, byte[] body)
```
## 消费者重复消费问题
一般情况下，producer不会重复投递消息，因为producer与broker建立的是TCP连接，TCP的三次握手机制保证了producer投递消息的稳定性，所以说producer重发是小概率事件
![[Pasted image 20240119205509.png]]
==消息的重复消费一定是在consumer方，已知consumer方消费消息存在两种模式，这两种模式都有可能重复消费消息：==
- BROADCASTING(广播)模式下，所有注册的消费者都会消费，而这些消费者通常是集群部署的一个个微服务，这样就会多台机器重复消费。
- CLUSTERING（负载均衡）模式下：
	- 有可能一个topic被多个consumerGroup订阅，也会重复消费。
	- CLUSTERING（负载均衡）模式下还存在重新负载均衡情况，即使broker中的一个队列在同一个consumerGroup下，但是只要任何一个消费者新上线后，同组的所有消费者要重新负载均衡（反之一个消费者掉线后，也会重新负载）。一个队列所对应的新的消费者要获取之前消费的offset（偏移量，也就是消息消费的点位），但是消费者消费消息，和broker更新offset是分两步操作，此时之前的消费者可能已经消费了一条消息，但是并没有把offset提交给broker，那么新的消费者可能会重新消费一次。
![[Pasted image 20240119214159.png]]

## 幂等性原则铺垫
无论是微服务中各个子系统相互之间的调用，还是客户端对服务端的调用，都存在网络延迟等问题，会导致重复请求接口
1. 网络延时
2. 消息丢失
幂等性原则：
- 重复的写操作必须保证操作只执行一次
- 重复的读操作必须保证返回同一个结果

==但是，多次执行请求有可能会对业务产生预期之外的后果，比如重复调用支付接口会导致多次扣钱。==

幂等性的解决有以下方案：
### MySQL唯一索引

创建去重表，去重表设置为唯一索引，唯一性索引多次插入就会报错，从而保证幂等性。
1. 创建mysql去重表表tb_lock用于并发占坑，比如给lock_name字段创建唯一性索引，线程尝试获取锁tb_lock（insert）
2. 创建mysql表tb_service，这是我们真正的业务数据表
3. 获取tb_lock的锁成功，则执行tb_service表的业务逻辑，执行完成释放锁tb_lock（delete再删除去重表中的这条数据）
4. 其他线程等待重试
### MySQL悲观锁for update

加行锁，互斥锁
### MySQL乐观版锁本号

在业务表中新建一个版本字段version，int类型。

服务A调用服务B需要更新的时候，需要提前查询到version，然后作为参数传过去。

数据更新的时候将version + 1，接口如果发生重试的时候，version已经发生变更，那么也能保证幂等性。
### MySQL支付状态位State，类似于乐观锁

增加支付状态，0代表待支付 ， 1代表支付中 ， 2代表支付成功，3代表支付失败

SQL语句通过查询并重设状态位State值来保证支付幂等

```sql
update order set status = 1 where status =0 and orderId = “201251487987”
```

- SQL返回影响数 1 代表修改成功，可以支付，继续执行支付业务代码
    
- SQL返回影响数 0 代表修改失败，该订单已经不是待支付订单了。

## MQ如何利用幂等性避免消息重复消费
### 解决方案1,MQ+MySQL存储唯一性索引
那么如果在CLUSTERING（负载均衡）模式下，并且在同一个消费者组中，不希望一条消息被重复消费，改怎么办呢？

对于本身并发较低的场景，不会对MySQL服务造成压力，可以直接使用MySQL的唯一性索引保存

==在MySQL中建立去重表order_oper_log，设置唯一索引字段order_sn，将key值映射到order_sn字段，多次插入重复的order_sn就会报错==
![[Pasted image 20240120124655.png]]
业务表还不变，是我们的xxx_service，只是多了一个去重表order_oper_log
```java
package org.lyflexi.arocketmqdemo;  
  
@SpringBootTest  
class ARocketmqDemoApplicationTests {  
  
    @Autowired  
    private JdbcTemplate jdbcTemplate;  
  
  
    @Test  
    void repeatProducer() throws Exception {  
        DefaultMQProducer producer = new DefaultMQProducer("repeat-producer-group");  
        producer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
        producer.start();  
        String key = UUID.randomUUID().toString();  
        System.out.println(key);  
        // 测试 发两个key一样的消息（真实环境下producer不会重复发送，而是consumer会重复消费）  
        Message m1 = new Message("repeatTopic", null, key, "扣减库存-1".getBytes());  
        Message m1Repeat = new Message("repeatTopic", null, key, "扣减库存-1".getBytes());  
        producer.send(m1);  
        producer.send(m1Repeat);  
        System.out.println("发送成功");  
        producer.shutdown();  
    }  
  
    /**  
     * 幂等性(mysql的唯一索引, redis(setnx) )  
     * 多次操作产生的影响均和第一次操作产生的影响相同  
     * 新增:普通的新增操作  是非幂等的,唯一索引的新增,是幂等的  
     * 修改:看情况  
     * 查询: 是幂等操作  
     * 删除:是幂等操作  
     * ---------------------  
     * 我们设计一个去重表 对消息的唯一key添加唯一索引  
     * 每次消费消息的时候 先插入数据库 如果成功则执行业务逻辑 [如果业务逻辑执行报错 则删除这个去重表记录]  
     * 如果插入失败 则说明消息来过了,直接签收了  
     *  
     * @throws Exception  
     */    @Test  
    void repeatConsumer() throws Exception {  
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("repeat-consumer-group");  
        consumer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
        consumer.subscribe("repeatTopic", "*");  
        consumer.registerMessageListener(new MessageListenerConcurrently() {  
            @Override  
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {  
                // 先拿key  
                MessageExt messageExt = msgs.get(0);  
                String keys = messageExt.getKeys();  
                // 原生方式操作  
                Connection connection = null;  
                try {  
                    connection = DriverManager.getConnection("jdbc:mysql://47.103.44.163:3306/test?serverTimezone=GMT%2B8&useSSL=false", "root", "123456");  
                } catch (SQLException e) {  
                    e.printStackTrace();  
                }  
                PreparedStatement statement = null;  
  
                try {  
                    // 先插入数据库 （去重表），将key值映射到数据表的唯一索引字段order_sn
                    statement = connection.prepareStatement("insert into order_oper_log(`type`, `order_sn`, `user`) values (1,'" + keys + "','123')");  
                } catch (SQLException e) {  
                    e.printStackTrace();  
                }  
  
                try {  
                    statement.executeUpdate();  
                } catch (SQLException e) {  
                    System.out.println("executeUpdate");  
                    if (e instanceof SQLIntegrityConstraintViolationException) {  
                        // 唯一索引冲突异常  
                        System.out.println("该消息来过了");  
                        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
                    }  
                    e.printStackTrace();  
                }  
  
                // 处理业务逻辑   （另外的业务表）         
                System.out.println(new String(messageExt.getBody()));  
                System.out.println(keys);  

                //最后别忘了解锁，删除掉这个去重表记录 delete from order_oper_log where order_sn = keys;      
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
            }  
        });  
        consumer.start();  
        System.in.read();  
    }  
  
  
}
```
### 解决方案2,MQ+Redis存储唯一key（秒杀用户去重案例）

consumer面对两个重复的消息key，第一次拿到key之后先保存到redis中，接下来正常执行业务逻辑，执行结束删除redis
如果很快第二次再次拿到相同的消息key，则先去redis中比较，如果redis中存在这个key(说明已经在消费中了)，则本次不做重复消费

redis-string结构，将消息key值为key，消息值为value，并设置过期时间，这个过期时间要能够承受住redis服务器异常时间，假如设置过期时间为10分钟，如果redis服务器断了20分钟，那么未消费的数据都会丢了
```java
//伪代码如下：
    public void receiveMessage(Message message) throws UnsupportedEncodingException {
	    //拿到msg
		//拿到消息的key
	 
        //判断redis中是否已经存在当前key，如果存在，提前return结束
        if (messageRedisValue.equals(key)) {
            return;
        }
        
        //否则，以队列名字为key，以消息key为value，将消息key存入redis
        redisUtil.set("key",msg,30L);
        //执行业务逻辑
    }
```

