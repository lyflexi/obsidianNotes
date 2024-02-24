# 一、如何保证消息的顺序？
RocketMQ的生产者支持发送非常多类型的消息：
- 同步消息（默认），Producer每发送一个消息，都要等待broker的回复SendResult，send(Message msg)
- 异步消息：Producer发送消息之后，无需等待broker回复，send(Message msg,  SendCallback sendCallback)
- 单向消息：这种方式主要用在不关心发送结果的场景例如日志信息的发送，这种方式吞吐量很大，但是存在消息丢失的风险sendOneway(Message msg)
- 延时消息：发送消息前可以给消息设置延时时间，message.setDelayTimeLevel(3)，表示延时3秒后才能够被消费。用于下订单业务，提交了一个订单就可以发送一个延时消息，30min后去检查这个订单的状态，如果还是未付款就重置订单状态，表示取消订单释放库存。
- 批量消息：将多条消息发送到broker里面相同的队列中集中存放，`send(Collection<Message> msgs)`
==MQ自带的发送策略都无法保证消息的顺序性，顺序性只有我们开发人员自己保证！==
模拟一个订单的发送流程，发送的消息顺序是订单号111 消息流程 下订单->物流->签收  ，如何保证顺序性？
- 要求发送方必须，把属于同一个订单的多条消息，强制放在同一个队列当中
- 要求接收方必须，禁用并发接收方式，启用单线程接收模式
## 发送方保证发送的顺序
创建个消息实体类：MsgModel
```java
@AllArgsConstructor  
@NoArgsConstructor  
public class MsgModel {  
  
    private String orderSn;  
    private Integer userId;  
    private String desc; // 下单 短信 物流  
  
    // xxx  
}

//初始化消息实体，模拟了两个订单的生命周期
private List<MsgModel> msgModels = Arrays.asList(  
        new MsgModel("qwer", 1, "下单"),  
        new MsgModel("qwer", 1, "短信"),  
        new MsgModel("qwer", 1, "物流"), 
         
        new MsgModel("zxcv", 2, "下单"),  
        new MsgModel("zxcv", 2, "短信"),  
        new MsgModel("zxcv", 2, "物流")  
); 
```

发送方：send(Message msg, MessageQueueSelector selector, Object arg)，其中selector会读到Object arg参数的值，利用这个特点，我们可以给Object arg传入订单号
```java
@Test  
public void orderlyProducer() throws Exception {  
    DefaultMQProducer producer = new DefaultMQProducer("orderly-producer-group");  
    producer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
    producer.start();  
    // 发送顺序消息  发送时要确保有序 并且要发到同一个队列下面去  
    msgModels.forEach(msgModel -> {  
        Message message = new Message("orderlyTopic", msgModel.toString().getBytes());  
        try {  
  
            producer.send(message, new MessageQueueSelector() {  
                // 发 相同的订单号去相同的队列  
                @Override  
                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {  
                    // 在这里利用订单号orderSn选择队列  
                    int hashCode = arg.toString().hashCode();  
                    int i = hashCode % mqs.size();  
                    return mqs.get(i);  
                }  
            }, msgModel.getOrderSn());  
  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    });  
    producer.shutdown();  
    System.out.println("发送完成");  
}
```
## 接收方保证接收的顺序

```java
@Test  
public void orderlyConsumer() throws Exception {  
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("orderly-consumer-group");  
    consumer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
    consumer.subscribe("orderlyTopic", "*");  
    // MessageListenerConcurrently 并发模式 多线程的  ，多个消费者线程争抢队列中的消息 
    // MessageListenerOrderly 顺序模式 单线程的 ，而且如果消息接收失败，会无限重试接收Integer.Max_Value  
    consumer.registerMessageListener(new MessageListenerOrderly() {  
        @Override  
        public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {  
            System.out.println("线程id:" + Thread.currentThread().getId());  
            System.out.println(new String(msgs.get(0).getBody()));  
            return ConsumeOrderlyStatus.SUCCESS;  
        }  
    });  
    consumer.start();  
    System.in.read();  
}
```

# 二、消息过滤场景，Tag与Topic区分？

==一个应用尽可能用一个Topic，而消息子类型则可以用tags来标识==。

tags主要用于以下场景：我们往一个Topic里面发送消息的时候，根据业务逻辑，有可能可能需要更进一步的区分，比如带有tagA标签的被A消费，带有tagB标签的被B消费。还有在事务监听的类里面，只要是事务消息都要走同一个监听，我们也需要通过Tag过滤才区别对待

tags可以由应用自由设置，只有生产者在发送消息设置了tags
```java
message.setTags("TagA")
```
消费方在订阅消息时才可以利用tags通过broker做消息过滤
## 订阅关系一致性（consumer-Topic-Tag）
订阅关系：一个消费者组订阅一个 Topic 的某一个 Tag，这种记录被称为订阅关系。

订阅关系一致：同一个消费者组下所有消费者实例所订阅的Topic、Tag必须完全一致。如果==订阅关系（消费者组名-Topic-Tag）==不一致，会导致消费消息紊乱，甚至消息丢失。
### 正确订阅关系示例
1. 订阅一个Topic且订阅一个Tag
如下图所示，同一Group ID下的三个Consumer实例C1、C2和C3分别都订阅了TopicA，且订阅TopicA的Tag也都是Tag1，符合订阅关系一致原则。
![[Pasted image 20240120110026.png]]
**正确示例代码一**

C1、C2、C3的订阅关系一致，即C1、C2、C3订阅消息的代码必须完全一致，代码示例如下：

```JAVA
    Properties properties = new Properties();
    properties.put(PropertyKeyConst.GROUP_ID, "GID_test_1");
    Consumer consumer = ONSFactory.createConsumer(properties);
    consumer.subscribe("TopicA", "Tag1", new MessageListener() {
        public Action consume(Message message, ConsumeContext context) {
            System.out.println(message.getMsgID());
            return Action.CommitMessage;
        }
    }); 
```

2. 订阅一个Topic且订阅多个Tag

如下图所示，同一Group ID下的三个Consumer实例C1、C2和C3分别都订阅了TopicB，订阅TopicB的Tag也都是Tag2和Tag3，表示订阅TopicB中所有Tag为Tag2或Tag3的消息，且顺序一致都是Tag2||Tag3，符合订阅关系一致性原则。
![[Pasted image 20240120110155.png]]
**正确示例代码二**

C1、C2、C3的订阅关系一致，即C1、C2、C3订阅消息的代码必须完全一致，代码示例如下：

```java
    Properties properties = new Properties();
    properties.put(PropertyKeyConst.GROUP_ID, "GID_test_2");
    Consumer consumer = ONSFactory.createConsumer(properties);
    consumer.subscribe("TopicB", "Tag2||Tag3", new MessageListener() {
        public Action consume(Message message, ConsumeContext context) {
            System.out.println(message.getMsgID());
            return Action.CommitMessage;
        }
    });   
```

3. 订阅多个Topic且订阅多个Tag

如下图所示，同一Group ID下的三个Consumer实例C1、C2和C3分别都订阅了TopicA和TopicB，
- 订阅的TopicA都未指定Tag，即订阅TopicA中的所有消息，
- 订阅的TopicB的Tag都是Tag2和Tag3，表示订阅TopicB中所有Tag为Tag2或Tag3的消息，且顺序一致都是Tag2||Tag3，
符合订阅关系一致原则。
![[Pasted image 20240120110239.png]]
**正确示例代码三**

C1、C2、C3的订阅关系一致，即C1、C2、C3订阅消息的代码必须完全一致，代码示例如下：

```JAVA
    Properties properties = new Properties();
    properties.put(PropertyKeyConst.GROUP_ID, "GID_test_3");
    Consumer consumer = ONSFactory.createConsumer(properties);
    consumer.subscribe("TopicA", "*", new MessageListener() {
        public Action consume(Message message, ConsumeContext context) {
            System.out.println(message.getMsgID());
            return Action.CommitMessage;
        }
    });     
    consumer.subscribe("TopicB", "Tag2||Tag3", new MessageListener() {
        public Action consume(Message message, ConsumeContext context) {
            System.out.println(message.getMsgID());
            return Action.CommitMessage;
        }
    });    
```

### 错误订阅关系排查

- 同一Group ID下的Consumer实例订阅的Topic不同
- 同一Group ID下的Consumer实例订阅的Topic相同，但订阅的Tag不一致
## 什么时候该用 Topic，什么时候该用 Tag？粒度怎么把控

1.消息类型是否一致：如普通消息、事务消息、定时（延时）消息、顺序消息，不同的消息类型使用不同的 Topic，无法通过 Tag 进行区分。

2.消息优先级是否一致：如同样是物流消息，盒马必须小时内送达，天猫超市 24 小时内送达，淘宝物流则相对会慢一些，不同优先级的消息用不同的 Topic 进行区分。

3.消息量级是否相当：有些业务消息虽然量小但是实时性要求高，如果跟某些万亿量级的消息使用同一个 Topic，则有可能会因为过长的等待时间而“饿死”，此时需要将不同量级的消息进行拆分，使用不同的 Topic。

总结：不同的业务应该使用不同的Topic，那么我们还要使用tag进行区分

4.业务是否相关联：没有直接关联的消息，如淘宝交易消息，京东物流消息使用不同的 Topic 进行区分；而同样是天猫交易消息，电器类订单、女装类订单、化妆品类订单的消息可以用 Tag 进行区分。

总的来说，针对消息分类，您可以选择创建多个 Topic，或者在同一个 Topic 下创建多个 Tag。但通常情况下，不同的 Topic 之间的消息没有必然的联系，而 Tag 则用来区分同一个 Topic 下相互关联的消息，例如全集和子集的关系、流程先后的关系。
# 三、死信消息解决方案？
在消费者return ConsumeConcurrentlyStatus.RECONSUME_LATER;后就会执行重试，当前msg会重新被放回消息队列，等待下一次被消费
## 死信消息模拟
==但是如果consumer重试次数达到默认的重试次数，该消息就会直接被扔到信队列DLQ中去，DLQ中的消息topic名字为%DLQ%consumergrouopname==
```java
@Test  
public void retryConsumer() throws Exception {  
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("retry-consumer-group");  
    consumer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
    consumer.subscribe("retryTopic", "*");  
    // 设定重试次数  
    consumer.setMaxReconsumeTimes(2);  
    consumer.registerMessageListener(new MessageListenerConcurrently() {  
        @Override  
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {  
            MessageExt messageExt = msgs.get(0);  
            System.out.println(new Date());  
            System.out.println(messageExt.getReconsumeTimes());  
            System.out.println(new String(messageExt.getBody()));  
            // 业务报错了 返回null 返回 RECONSUME_LATER 都会重试  
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;  
        }  
    });  
    consumer.start();  
    System.in.read();  
}


@Test  
public void retryDeadConsumer() throws Exception {  
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("retry-dead-consumer-group");  
    consumer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
    consumer.subscribe("%DLQ%retry-consumer-group", "*");  
    consumer.registerMessageListener(new MessageListenerConcurrently() {  
        @Override  
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {  
            MessageExt messageExt = msgs.get(0);  
            System.out.println(new Date());  
            System.out.println(new String(messageExt.getBody()));  
            System.out.println("死信队列，兜底方案，记录到特别的位置 文件 mysql 通知人工处理");  
            // 业务报错了 返回null 返回 RECONSUME_LATER 都会重试  
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
        }  
    });  
    consumer.start();  
    System.in.read();  
}
```
## 解决方案
因此，我们在实际生产过程中对死信消息的解决方案是，一般重试3-5次，如果还没有消费成功，则可以把消息签收了，通知人工等处理
```java
@Test  
public void retryConsumer2() throws Exception {  
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("retry-consumer-group");  
    consumer.setNamesrvAddr(MqConstant.NAME_SRV_ADDR);  
    consumer.subscribe("retryTopic", "*");  
    // 设定重试次数  
    consumer.registerMessageListener(new MessageListenerConcurrently() {  
        @Override  
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {  
            MessageExt messageExt = msgs.get(0);  
            System.out.println(new Date());  
            // 业务处理  
            try {  
                handleDb();  
            } catch (Exception e) {  
                // 重试  
                int reconsumeTimes = messageExt.getReconsumeTimes();  
                if (reconsumeTimes >= 3) {  
                    // 不要重试了  
                    System.out.println("记录到特别的位置 文件 mysql 通知人工处理");  
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
                }  
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;  
            }  
            // 业务报错了 返回null 返回 RECONSUME_LATER 都会重试  
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
        }  
    });  
    consumer.start();  
    System.in.read();  
}  
  
private void handleDb() {  
    int i = 10 / 0;  
}
```
# 四、重复消费解决方案，key与messageID区分？

messageID是rocketmq默认给消息生成的ID，我们也可以自己手动的去给消息添加key，最好使用自己定义的key，原因如下：
- 虽然说messageID一定是全局唯一标识符，但是实际使用中，可能会存在相同的消息有两个不同messageID的情况，比如producer主动重发、或因客户端重投机制导致的重发等，这种情况就需要使业务字段进行重复消费。
- 另外，使用自定义key对我们程序员来说也是可控的
```java
Message(String topic, String tags, String keys, byte[] body)
```
## 消费者问题
一般情况下，producer不会重复投递消息，因为producer与broker建立的是TCP连接，TCP的三次握手机制保证了producer投递消息的稳定性：
![[Pasted image 20240119205509.png]]
producer重发是小概率事件，==因此，消息的重复消费一定是在consumer方==

consumer方消费消息存在两种模式：
- BROADCASTING(广播)模式下，所有注册的消费者都会消费，而这些消费者通常是集群部署的一个个微服务，这样就会多台机器重复消费。
- CLUSTERING（负载均衡）模式下，有可能一个topic被多个consumerGroup订阅，也会重复消费。
==另外在CLUSTERING（负载均衡）模式下还存在重新负载均衡情况==，即使同一个consumerGroup下，broker中的一个队列只会分配给同一个consumerGroup，看起来好像是不会重复消费。但是，有个特殊情况：一个消费者新上线后，同组的所有消费者要重新负载均衡（反之一个消费者掉线后，也会重新负载）。一个队列所对应的新的消费者要获取之前消费的offset（偏移量，也就是消息消费的点位），但是消费者消费消息，和broker更新offset是分两步操作，此时之前的消费者可能已经消费了一条消息，但是并没有把offset提交给broker，那么新的消费者可能会重新消费一次。
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
### 解决方案2,MQ+Redis存储唯一key

consumer面对两个重复的消息key，第一次拿到key之后先保存到redis中，接下来正常执行业务逻辑
如果第二次再次拿到相同的消息key，则先去redis中比较，如果redis中存在这个key，则抛弃这个消息

对于Redis数据库本身，又细化为两种方案：
方案一：redis-list结构，类似于MySQL存储
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
        redisUtil.lpush("queueName",key);
        //执行业务逻辑
    }
```
这个方案很好，但是随着mq的消息增加，redis数据越来越多，需要去清除redis数据。

方案二：redis-string结构，将消息key值为key，消息值为value，并设置过期时间，这个过期时间要能够承受住redis服务器异常时间，假如设置过期时间为10分钟，如果redis服务器断了20分钟，那么未消费的数据都会丢了
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

# 五、消息堆积问题的解决方案？
一般认为单条队列消息差值>=1w时算堆积，消息堆积产生的原因与解决方案
1. 生产太快了，生产方可以做业务限流
2. 适当的设置最大的消费线程数量，如果是IO密集型程序线程数设置为2n，如果是计算密集型程序线程数设置为n+1)
3. 也可以单纯的增加消费者数量，但是超过队列数量的消费者是没有意义的，因此需要动态扩容队列数量，从而增加消费者数量
最后还有可能是消费者程序出现问题，排查消费者程序的问题

# 六、消息丢失问题的解决方案？
1. 方案一：producer在发送消息的同时会向磁盘进行刷盘（硬盘），rocketmq有两种刷盘机制，同步刷盘和异步刷盘。要想减少消息的丢失情况，最好将mq的刷盘机制设置为同步刷盘，因为异步刷盘会利用buffer进行缓冲，buffer属于内存是不稳定的
2. 方案二：记录消息日志
	1. producer每发一次消息，都最好将发送日志保存下来，比如MySQL ，mq_log表【key，creatTime，status】初始status=0
	2. consumer每消费一条消息，也去mq_log表更新status=1
	3. 创建定时任务，每天执行，遍历mq_log中status依然为0的记录，这些记录被认为是丢失了，请求producer重新补发
	4. 因为有可能消息并没有丢失，而是挤压在了broker，所以consumer要自觉的开启幂等性判断，防止消息重复消费
3. 方案三：开启rocketmq的trace功能
	1. 在broker.conf中开启消息追踪traceTopicEnable=true，重启broker
	2. 生产者配置yml开启消息轨迹，enable-msg-trace: true
	3. 消费者开启消息轨迹功能，可以给单独的某一个消费者开启enableMsgTrace = true
	4. rocket面板可以查看某一Topic的轨迹

