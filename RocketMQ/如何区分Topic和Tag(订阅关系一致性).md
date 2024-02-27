==一个应用尽可能用一个Topic，而消息子类型则可以用tags来标识==。

tags主要用于以下场景：我们往一个Topic里面发送消息的时候，根据业务逻辑，有可能可能需要更进一步的区分，比如带有tagA标签的被A消费，带有tagB标签的被B消费。还有在事务监听的类里面，只要是事务消息都要走同一个监听，我们也需要通过Tag过滤才区别对待

tags可以由应用自由设置，生产者在发送消息时设置了tags
```java
message.setTags("TagA")
```
消费方在订阅消息时就可以利用tags通过broker做消息过滤
# 订阅关系一致性（consumer-Topic-Tag）
订阅关系：一个消费者组订阅一个 Topic 的某一个 Tag，这种记录被称为订阅关系。形式化表示为[消费者组名-Topic-Tag]

订阅关系一致：同一个消费者组下所有消费者实例所订阅的Topic以及Tag必须完全一致。如果订阅关系[消费者组名-Topic-Tag]不一致，会导致消费消息紊乱，甚至消息丢失。
## 订阅同一个Topic中的同一个Tag
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

## 订阅同一个Topic中的多个Tag

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

## 订阅多个Topic中的多个Tag

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

错误订阅关系情况排查
- 同一Group ID下的Consumer实例订阅的Topic不同
- 同一Group ID下的Consumer实例订阅的Topic相同，但订阅的Tag不一致
# 什么时候该用 Topic，什么时候该用 Tag？粒度怎么把控

1.消息类型是否一致：如普通消息、事务消息、定时（延时）消息、顺序消息，不同的消息类型使用不同的 Topic，无法通过 Tag 进行区分。

2.消息优先级是否一致：如同样是物流消息，盒马必须小时内送达，天猫超市 24 小时内送达，淘宝物流则相对会慢一些，不同优先级的消息用不同的 Topic 进行区分。

3.消息量级是否相当：有些业务消息虽然量小但是实时性要求高，如果跟某些万亿量级的消息使用同一个 Topic，则有可能会因为过长的等待时间而“饿死”，此时需要将不同量级的消息进行拆分，使用不同的 Topic。

总结：不同的业务应该使用不同的Topic，那么我们还要使用tag进行区分

4.业务是否相关联：没有直接关联的消息，如淘宝交易消息，京东物流消息使用不同的 Topic 进行区分；而同样是天猫交易消息，电器类订单、女装类订单、化妆品类订单的消息可以用 Tag 进行区分。

总的来说，针对消息分类，您可以选择创建多个 Topic，或者在同一个 Topic 下创建多个 Tag。但通常情况下，不同的 Topic 之间的消息没有必然的联系，而 Tag 则用来区分同一个 Topic 下相互关联的消息，例如全集和子集的关系、流程先后的关系。