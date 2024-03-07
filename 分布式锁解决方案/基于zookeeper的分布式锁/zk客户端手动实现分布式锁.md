实现分布式锁目前有三种流行方案，分别为基于数据库、Redis、Zookeeper的方案。这里主要介绍基于zk怎么实现分布式锁。在实现分布式锁之前，先回顾zookeeper的相关知识点
压缩包安装（也可以使用docker安装替代）：把zk安装包上传到/opt目录下，并切换到/opt目录下，执行以下指令
```shell
# 解压
tar -zxvf zookeeper-3.7.0-bin.tar.gz
# 重命名
mv apache-zookeeper-3.7.0-bin/ zookeeper
# 打开zookeeper根目录
cd /opt/zookeeper
# 创建一个数据目录，备用
mkdir data
# 打开zk的配置目录
cd /opt/zookeeper/conf
# copy配置文件，zk启动时会加载zoo.cfg文件
cp zoo_sample.cfg zoo.cfg
# 编辑配置文件
vim zoo.cfg
# 修改dataDir参数为之前创建的数据目录：/opt/zookeeper/data
# 切换到bin目录
cd /opt/zookeeper/bin
# 启动 
./zkServer.sh start
./zkServer.sh status # 查看启动状态
./zkServer.sh stop # 停止
./zkServer.sh restart # 重启
./zkCli.sh # 进入zk客户端
```
Zookeeper提供一个多层级的节点命名空间（节点称为znode），每个节点都用一个以斜杠（/）分隔的路径表示，而且每个节点都有父节点（根节点除外），非常类似于文件系统。并且每个节点都是唯一的。
znode节点有四种类型：
- PERSISTENT：永久节点。客户端与zookeeper断开连接后，该节点依旧存在
- EPHEMERAL：临时节点。客户端与zookeeper断开连接后，该节点被删除
- PERSISTENT_SEQUENTIAL：永久节点、序列化。可以重复设置节点，Zookeeper会给该节点名称进行顺序编号
- EPHEMERAL_SEQUENTIAL：临时节点、序列化。可以重复设置节点，Zookeeper给该节点名称进行顺序编号

执行`./zkCli.sh` 进入zk客户端，演示四种节点的常用操作：
- create创建节点
- ls查看节点下的子节点
- stat查看节点的状态
- get查看节点的内容
- delete删除节点
```shell
[zk: localhost:2181(CONNECTED) 0] create /aa test  # 创建持久化节点
Created /aa
[zk: localhost:2181(CONNECTED) 1] create -s /bb test  # 创建持久序列化节点
Created /bb0000000001
[zk: localhost:2181(CONNECTED) 2] create -e /cc test  # 创建临时节点
Created /cc
[zk: localhost:2181(CONNECTED) 3] create -e -s /dd test  # 创建临时序列化节点
Created /dd0000000003
[zk: localhost:2181(CONNECTED) 4] ls /   # 查看某个节点下的子节点
[aa, bb0000000001, cc, dd0000000003, zookeeper]
[zk: localhost:2181(CONNECTED) 5] stat /  # 查看某个节点的状态
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x5
cversion = 3
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 5
[zk: localhost:2181(CONNECTED) 6] get /aa  # 查看某个节点的内容
test
[zk: localhost:2181(CONNECTED) 11] delete /aa  # 删除某个节点
[zk: localhost:2181(CONNECTED) 7] ls /  # 再次查看
[bb0000000001, cc, dd0000000003, zookeeper]
```
节点事件监听：在当前zk客户端1执行正常操作时，我们可以新起客户端2设置事件监听，当客户端1的节点数据或结构变化时，zookeeper会返回结果通知到客户端2。当前zookeeper针对节点的监听有如下四种事件：
```shell
1. 节点创建：stat -w /xx
    当/xx节点创建时：NodeCreated
2. 节点删除：stat -w /xx
    当/xx节点删除时：NodeDeleted
3. 节点数据修改：get -w /xx
    当/xx节点数据发生变化时：NodeDataChanged
4. 子节点变更：ls -w /xx
    当/xx节点的子节点创建或者删除时：NodeChildChanged
```
# 引入zookeeper依赖
引入Java依赖zookeeper
```xml
<dependencies>  
    <dependency>        
	    <groupId>org.apache.zookeeper</groupId>  
        <artifactId>zookeeper</artifactId>  
        <exclusions>            
	        <exclusion>                
		        <groupId>org.slf4j</groupId>  
                <artifactId>slf4j-log4j12</artifactId>  
            </exclusion>        
        </exclusions>        
        <version>3.7.0</version>  
    </dependency>    
    <dependency>        
	    <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-data-redis</artifactId>  
    </dependency>
</dependencies>
```
常用api及其方法
```java
public class ZkTest {

    public static void main(String[] args) throws KeeperException, InterruptedException {

        // 获取zookeeper链接
        CountDownLatch countDownLatch = new CountDownLatch(1);
        ZooKeeper zooKeeper = null;
        try {
            zooKeeper = new ZooKeeper("172.16.116.100:2181", 30000, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (Event.KeeperState.SyncConnected.equals(event.getState()) 
                            && Event.EventType.None.equals(event.getType())) {
                        System.out.println("获取链接成功。。。。。。" + event);
                        countDownLatch.countDown();
                    }
                }
            });

            countDownLatch.await();
        } catch (Exception e) {
            e.printStackTrace();
        }

        // 创建一个节点，1-节点路径 2-节点内容 3-节点的访问权限 4-节点类型
        // OPEN_ACL_UNSAFE：任何人可以操作该节点
        // CREATOR_ALL_ACL：创建者拥有所有访问权限
        // READ_ACL_UNSAFE: 任何人都可以读取该节点
        // zooKeeper.create("/atguigu/aa", "haha~~".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        zooKeeper.create("/test", "haha~~".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        // zooKeeper.create("/atguigu/cc", "haha~~".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);
        // zooKeeper.create("/atguigu/dd", "haha~~".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        // zooKeeper.create("/atguigu/dd", "haha~~".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        // zooKeeper.create("/atguigu/dd", "haha~~".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

        // 判断节点是否存在
        Stat stat = zooKeeper.exists("/test", true);
        if (stat != null){
            System.out.println("当前节点存在！" + stat.getVersion());
        } else {
            System.out.println("当前节点不存在！");
        }

        // 判断节点是否存在，同时添加监听
        zooKeeper.exists("/test", event -> {
        });

        // 获取一个节点的数据
        byte[] data = zooKeeper.getData("/atguigu/ss0000000001", false, null);
        System.out.println(new String(data));

        // 查询一个节点的所有子节点
        List<String> children = zooKeeper.getChildren("/test", false);
        System.out.println(children);

        // 更新
        zooKeeper.setData("/test", "wawa...".getBytes(), stat.getVersion());

        // 删除一个节点
        //zooKeeper.delete("/test", -1);

        if (zooKeeper != null){
            zooKeeper.close();
        }
    }
}
```
# zk分布式锁实现思路-临时节点
分布式锁的步骤：
1. 获取锁：create一个节点。多个请求同时添加一个相同的临时节点，只有一个可以添加成功。添加成功的获取到锁
2. 执行业务逻辑
3. 删除锁：delete一个节点。完成业务流程后，删除节点释放锁。
4. 重试：没有获取到锁的请求重试

参照redis分布式锁的特点：互斥 排他
1. 防死锁：节点必须是临时的，客户端宕机后，过了一定时间zookeeper没有收到客户端的心跳包判断会话失效，将临时节点删除从而释放锁。    
2. 可重入锁：借助于ThreadLocal
3. 防误删：因为不需要设置过期时间，也就不存在误删问题。
4. 加锁/解锁要具备原子性
5. 单点问题：使用zookeeper可以有效的解决单点问题，ZK一般是集群部署的。
6. 集群问题：zookeeper集群是强一致性的，只要集群中有半数以上的机器存活，就可以对外提供服务。
## 建立连接
由于zookeeper获取链接是一个耗时过程，这里可以在项目启动时，初始化链接，并且只初始化一次。借助于spring特性，代码实现如下：
```java
@Component
public class ZkClient {

    private static final String connectString = "172.16.116.100:2181";

    private static final String ROOT_PATH = "/distributed";

    private ZooKeeper zooKeeper;

    @PostConstruct
    public void init(){
        try {
            // 连接zookeeper服务器
            this.zooKeeper = new ZooKeeper(connectString, 30000, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    System.out.println("获取链接成功！！");
                }
            });

            // 创建分布式锁根节点
            if (this.zooKeeper.exists(ROOT_PATH, false) == null){
                this.zooKeeper.create(ROOT_PATH, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
        } catch (Exception e) {
            System.out.println("获取链接失败！");
            e.printStackTrace();
        }
    }

    @PreDestroy
    public void destroy(){
        try {
            if (zooKeeper != null){
                zooKeeper.close();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 初始化zk分布式锁对象方法
     * @param lockName
     * @return
     */
    public ZkDistributedLock getZkDistributedLock(String lockName){
        return new ZkDistributedLock(zooKeeper, lockName);
    }
}
```
## 自旋锁
zk分布式锁具体实现：
- 在zk里面一个节点代表着一个全路径名
- 为了预防死锁，我们创建临时节点EPHEMERAL
与Redis分布式锁一样，我们只需要key，而value没有实际意义因此这里随便设置为null
```java
public class ZkDistributedLock {

    private static final String ROOT_PATH = "/distributed";

    private String path;

    private ZooKeeper zooKeeper;

    public ZkDistributedLock(ZooKeeper zooKeeper, String lockName){
        this.zooKeeper = zooKeeper;
        //在zk里面一个节点代表着一个全路径名
        this.path = ROOT_PATH + "/" + lockName;
    }

    public void lock(){
        try {
            zooKeeper.create(path, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        } catch (Exception e) {
            // 重试
            try {
                Thread.sleep(200);
                lock();
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
        }
    }

    public void unlock(){
        try {
            this.zooKeeper.delete(path, 0);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }
}
```
改造StockService的checkAndLock方法：
```java
package org.lyflexi.zkhandsondistrilock.service;  
  
import org.lyflexi.zkhandsondistrilock.lock.ZkClient;  
import org.lyflexi.zkhandsondistrilock.lock.ZkDistributedLock;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.stereotype.Service;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/24 15:11  
 */@Service  
public class StockService {  
    @Autowired  
    private ZkClient client;  
    @Autowired  
    StringRedisTemplate redisTemplate;  
  
    public void checkAndLock() {  
        // 加锁，获取锁失败重试  
        ZkDistributedLock lock = this.client.getZkDistributedLock("lock");  
        lock.lock();  
  
        // 1. 查询库存信息  
        String stock = redisTemplate.opsForValue().get("stock").toString();  
        // 2. 判断库存是否充足  
        if (stock != null && stock.length() != 0) {  
            Integer st = Integer.valueOf(stock);  
            if (st > 0) {  
                // 3.扣减库存  
                redisTemplate.opsForValue().set("stock", String.valueOf(--st));  
            }  
        }  
  
        // 释放锁  
        lock.unlock();  
    }  
}
```
Jmeter压力测试：
![[1607046072239.png]]
性能一般，mysql数据库的库存余量为0（注意：所有测试之前都要先修改库存量为5000）
基本实现存在的问题：
1. 性能一般（比mysql分布式锁略好）
2. 不可重入
接下来首先来提高性能
# 性能优化-临时序列化节点
上面实现中由于重试操作使用的是自旋，在并发高的情况下会严重影响Cpu性能：
![[1607048160051.png]]
试想：每个请求要想正常的执行完成，最终都是要创建节点，如果能够避免争抢必然可以提高性能。
==这里借助于zk的临时序列化节点，实现阻塞锁，并且加入监听机制==
1. 在同一目录下生成多个序列化子节点，序列号小的节点先获取到锁
2. 监听同一目录下的序列化节点中的前驱节点，若前驱节点释放，则当前序列化节点获取到锁
![[1607048783043.png]]
## 阻塞锁实现-SEQUENTIAL
代码实现：
1. 主要修改了构造方法，每个并发请求来都先创建EPHEMERAL_SEQUENTIAL序列化节点
2. 主要修改了lock方法，添加了getPreNode获取前置节点的逻辑。
```java
public class ZkDistributedLock {

    private static final String ROOT_PATH = "/distributed";

    private String path;

    private ZooKeeper zooKeeper;

    public ZkDistributedLock(ZooKeeper zooKeeper, String lockName){
        try {
            this.zooKeeper = zooKeeper;
            this.path = zooKeeper.create(ROOT_PATH + "/" + lockName + "-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void lock(){
        String preNode = getPreNode(path);
        // 如果该节点没有前一个节点，说明该节点是最小节点，放行执行业务逻辑
        if (StringUtils.isEmpty(preNode)){
            return ;
        }
        try {
            Thread.sleep(20);
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
        lock();
    }

    public void unlock(){
        try {
            this.zooKeeper.delete(path, 0);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    /**  
     * 获取指定节点的前节点：指的是同一目录下的序列化节点中的前驱节点，序列化节点会从小到大排序  
     * @param path  
     * @return  
     */  
    private String getPreNode(String path){  
  
        try {  
            // 获取当前节点的序列化号43423525256  
            Long curSerial = Long.valueOf(StringUtils.substringAfterLast(path, "-"));  
            // 获取根路径distributed下的所有序列化子节点名称，如lock-43423525256，但是有可能是别的资源的锁如lockxx-111,lockyy-111  
            List<String> children = this.zooKeeper.getChildren(ROOT_PATH, false);  
  
            // 判空  
            if (CollectionUtils.isEmpty(children)){  
                return null;  
            }  
            //对children去重，让当前资源和当前锁匹配  
            List<String> nodes = children.stream().filter(node -> StringUtils.startsWith(node, lockName + '-')).collect(Collectors.toList());  
  
            if (CollectionUtils.isEmpty(nodes)){  
                return null;  
            }  
  
              
            Long flag = 0L;  
            String preNode = null;  
            for (String node : nodes) {  
                // 获取每个节点的序列化号  
                Long serial = Long.valueOf(StringUtils.substringAfterLast(node, "-"));  
                // 获取前一个节点
                if (serial < curSerial && serial > flag){  
                    flag = serial;  
                    preNode = node;  
                }  
            }  
  
            return preNode;  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
        return null;  
    }  
}
```
==特别需要注意的是在getPreNode的实现逻辑中，一定要对//children去重，让当前资源和当前锁匹配，因为同一个根目录distributed下很有可能还存在其他资源的锁，当前资源的锁和其他资源的锁是两码事==
jmeter测试
![[1607051896117.png]]
性能反而更弱了。

==原因：虽然不用自旋争抢创建节点了，但是会自旋判断自己是最小的节点，因为对于getPreNode来说假如当前有1000个节点在等待锁，如果获得锁的线程释放锁时，这1000个并发线程都会被唤醒，这种情况称为“羊群效应”；在这种羊群效应中，zookeeper需要通知1000个客户端，这会阻塞zookeeper的其他的操作，这个判断逻辑更复杂更耗时。==

最好的情况应该只唤醒新的最小节点对应的并发线程。

解决方案：监听。
## 监听阻塞锁实现-Watcher
应该怎么做呢？在设置事件监听时，每个客户端应该对刚好在它之前的子节点设置事件监听，举例说明：假如子节点列表为/distributed/lock-0000000000、/distributed/lock-0000000001、/distributed/lock-0000000002，
- 序号为1的客户端监听序号为0的子节点删除消息，
- 序号为2的监听序号为1的子节点删除消息。
所以调整后的分布式锁算法流程如下：
- 客户端连接zookeeper，并在/distributed下创建临时的子节点，并且让他们先排好序sort，比如第一个客户端对应的子节点为/distributed/lock-0000000000，第二个为/distributed/lock-0000000001，以此类推；
- 客户端获取/distributed下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，如果是则认为获得锁，否则监听刚好在自己之前一位的子节点删除消息，获得子节点变更通知后重复此步骤直至获得锁；
- 执行业务代码；
- 完成业务流程后，删除对应的子节点释放锁。
改造ZkDistributedLock的lock方法：
```java
package org.lyflexi.zkhandsondistrilock.lock;  
  
import org.apache.commons.lang3.StringUtils;  
import org.apache.zookeeper.*;  
import org.springframework.util.CollectionUtils;  
  
import java.util.Collections;  
import java.util.List;  
import java.util.concurrent.CountDownLatch;  
import java.util.stream.Collectors;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/24 15:10  
 */public class ZkDistributedLock {  
  
    private static final String ROOT_PATH = "/distributed";  
  
    private String path;  
  
    private ZooKeeper zooKeeper;  
    private String lockName;  
  
    public ZkDistributedLock(ZooKeeper zooKeeper, String lockName){  
        try {  
            this.zooKeeper = zooKeeper;  
            this.lockName = lockName;  
            //每次请求来都来创建序列号锁，lockName还是相同，但是序列号不同  
            this.path = zooKeeper.create(ROOT_PATH + "/" + lockName + "-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
  
    public void lock(){  
        try {  
            String preNode = getPreNode(path);  
            // 如果该节点没有前一个节点，说明该节点是最小节点，获取到锁,放行执行业务逻辑  
            if (StringUtils.isEmpty(preNode)){  
                return ;  
            } else {  
                CountDownLatch countDownLatch = new CountDownLatch(1);  
                //二次校验前驱节点，如果在此期间前驱节点持有的锁释放，那么当前节点获取到锁  
                if (this.zooKeeper.exists(ROOT_PATH + "/" + preNode, new Watcher(){  
                    @Override  
                    public void process(WatchedEvent event) {  
                        countDownLatch.countDown();  
                    }  
                }) == null) {  
                    //当前获取到锁,放行执行业务逻辑  
                    return;  
                }  
                countDownLatch.await();  
                return;  
            }  
        } catch (Exception e) {  
            e.printStackTrace();  
            // 重新检查。是否获取到锁  
            try {  
                Thread.sleep(200);  
            } catch (InterruptedException ex) {  
                ex.printStackTrace();  
            }  
            lock();  
        }  
    }  
  
    public void unlock(){  
        try {  
            this.zooKeeper.delete(path, 0);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        }  
    }  
  
    /**  
     * 获取指定节点的前节点：指的是同一目录下的序列化节点中的前驱节点，序列化节点会从小到大排序  
     * @param path  
     * @return  
     */  
    private String getPreNode(String path){  
  
        try {  
            // 获取当前节点的序列化号43423525256  
            Long curSerial = Long.valueOf(StringUtils.substringAfterLast(path, "-"));  
            List<String> children = this.zooKeeper.getChildren(ROOT_PATH, false);  
            // 判空  
            if (CollectionUtils.isEmpty(children)){  
                throw new IllegalMonitorStateException("非法操作");  
            }  
            List<String> nodes = children.stream().filter(node -> StringUtils.startsWith(node, lockName + '-')).collect(Collectors.toList());  
  
            if (CollectionUtils.isEmpty(nodes)){  
                throw new IllegalMonitorStateException("非法操作");  
            }  
            //拍好序  
            Collections.sort(nodes);  
            //获取当前序列节点的索引  
            String currNodeName = StringUtils.substringAfterLast(path, "/");  
            int index = Collections.binarySearch(nodes, currNodeName);  
            if (index<0){  
                throw new IllegalMonitorStateException("非法操作");  
            }else if (index>0){  
                return nodes.get(index-1);  
            }  
            //index=0，没有前置节点，说明自己就是前置节点，返回null，放行  
            return null;  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
        return null;  
    }  
}
```
压力测试效果如下：
![[1607052541669.png]]
由此可见性能提高不少，接近于redis的分布式锁
# 可重入实现-ThreadLocal
实现方式1.在节点的内容中记录服务器、线程以及重入信息，这类似于Redis分布式可重入的实现方式
实现方式2.ThreadLocal：线程的局部变量，线程私有，能够用于当前线程的重入计数我们使用这种方案：
1. 修改构造方法：如果我这次是重入的，那就不要再重新创建锁了 
2. 修改lock：如果ThreadLocal不为0则重入，否则置为1表示第一次获取锁
3. 修改unlock方法：如果ThreadLocal为0记得remove，因为ZkDistributedLock是单实例的导致ThreadLocal也是单实例的，所以可能存在复用
```java
package org.lyflexi.zkhandsondistrilock.lock;  
  
import org.apache.commons.lang3.StringUtils;  
import org.apache.zookeeper.*;  
import org.springframework.util.CollectionUtils;  
  
import java.util.Collections;  
import java.util.List;  
import java.util.concurrent.CountDownLatch;  
import java.util.stream.Collectors;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/24 15:10  
 */public class ZkDistributedLock {  
  
    private static final String ROOT_PATH = "/distributed";  
  
    private String path;  
  
    private ZooKeeper zooKeeper;  
    private String lockName;  
  
    private static final ThreadLocal<Integer> THREAD_LOCAL = new ThreadLocal<>();  
  
    public ZkDistributedLock(ZooKeeper zooKeeper, String lockName){  
        try {  
            this.zooKeeper = zooKeeper;  
            this.lockName = lockName;  
            //每次请求来都来创建序列号锁，lockName还是相同，但是序列号不同  
//            this.path = zooKeeper.create(ROOT_PATH + "/" + lockName + "-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);  
            //如果我这次是重入的，那就不要再重新创建锁了  
            if (THREAD_LOCAL.get() == null || THREAD_LOCAL.get() == 0){  
                this.path = zooKeeper.create(ROOT_PATH + "/" + lockName + "-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);  
            }  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
  
    public void lock(){  
        Integer flag = THREAD_LOCAL.get();  
        if (flag != null && flag > 0) {  
            THREAD_LOCAL.set(flag + 1);  
            return;  
        }  
        try {  
            String preNode = getPreNode(path);  
  
            if (!StringUtils.isEmpty(preNode)){  
                CountDownLatch countDownLatch = new CountDownLatch(1);  
                //二次校验前驱节点，如果在此期间前驱节点持有的锁释放，那么当前节点获取到锁  
                if (this.zooKeeper.exists(ROOT_PATH + "/" + preNode, new Watcher(){  
                    @Override  
                    public void process(WatchedEvent event) {  
                        countDownLatch.countDown();  
                    }  
                }) == null) {  
                    //前驱节点结束，当前节点获取到锁,放行执行业务逻辑  
                    //首次获取  
                    THREAD_LOCAL.set(1);  
                    return;  
                }  
                //如果监听到前驱节点，则当前线程阻塞，阻塞锁  
                countDownLatch.await();  
                THREAD_LOCAL.set(1);  
                return;  
            }  
  
            // 如果该节点没有前一个节点，说明该节点是最小节点，获取到锁,放行执行业务逻辑  
            THREAD_LOCAL.set(1);  
            return;  
        } catch (Exception e) {  
            e.printStackTrace();  
            // 重新检查。是否获取到锁  
            try {  
                Thread.sleep(200);  
            } catch (InterruptedException ex) {  
                ex.printStackTrace();  
            }  
//            lock();  
        }  
    }  
  
    public void unlock(){  
        try {  
            THREAD_LOCAL.set(THREAD_LOCAL.get() - 1);  
            if (THREAD_LOCAL.get() == 0) {  
                this.zooKeeper.delete(path, 0);  
                THREAD_LOCAL.remove();  
            }  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        }  
    }  
  
    /**  
     * 获取指定节点的前节点：指的是同一目录下的序列化节点中的前驱节点，序列化节点会从小到大排序  
     * @param path  
     * @return  
     */  
    private String getPreNode(String path){  
  
        try {  
            // 获取当前节点的序列化号43423525256  
            Long curSerial = Long.valueOf(StringUtils.substringAfterLast(path, "-"));  
            // 获取根路径distributed下的所有序列化子节点名称，如lock-43423525256，但是有可能是别的资源的锁如lockxx-111,lockyy-111  
            List<String> children = this.zooKeeper.getChildren(ROOT_PATH, false);  
  
            // 判空  
            if (CollectionUtils.isEmpty(children)){  
                throw new IllegalMonitorStateException("非法操作");  
            }  
            //对children去重，让当前资源和当前锁匹配  
            List<String> nodes = children.stream().filter(node -> StringUtils.startsWith(node, lockName + '-')).collect(Collectors.toList());  
  
            if (CollectionUtils.isEmpty(nodes)){  
                throw new IllegalMonitorStateException("非法操作");  
            }  
            //拍好序  
            Collections.sort(nodes);  
            //获取当前序列节点的索引  
            String currNodeName = StringUtils.substringAfterLast(path, "/");  
            int index = Collections.binarySearch(nodes, currNodeName);  
            if (index<0){  
                throw new IllegalMonitorStateException("非法操作");  
            }else if (index>0){  
                return nodes.get(index-1);  
            }  
            //index=0，没有前置节点，说明自己就是前置节点，返回null，放行  
            return null;  
        } catch (KeeperException e) {  
            e.printStackTrace();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
        return null;  
    }  
}
```
业务类测试可重入
```java
package org.lyflexi.zkhandsondistrilock.service;  
  
import org.lyflexi.zkhandsondistrilock.lock.ZkClient;  
import org.lyflexi.zkhandsondistrilock.lock.ZkDistributedLock;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.stereotype.Service;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/24 15:11  
 */@Service  
public class StockService {  
    @Autowired  
    private ZkClient client;  
    @Autowired  
    StringRedisTemplate redisTemplate;  
  
    public void checkAndLock() {  
        // 加锁，获取锁失败重试  
        ZkDistributedLock lock = this.client.getZkDistributedLock("lock");  
        lock.lock();  
  
        // 1. 查询库存信息  
        String stock = redisTemplate.opsForValue().get("stock").toString();  
        // 2. 判断库存是否充足  
        if (stock != null && stock.length() != 0) {  
            Integer st = Integer.valueOf(stock);  
            if (st > 0) {  
                // 3.扣减库存  
                redisTemplate.opsForValue().set("stock", String.valueOf(--st));  
            }  
        }  
  
        this.test();  
        // 释放锁  
        lock.unlock();  
    }  
  
  
    public void test() {  
        ZkDistributedLock lock = this.client.getZkDistributedLock("lock");  
        lock.lock();  
        System.out.println("测试可重入锁");  
        lock.unlock();  
    }  
}
```