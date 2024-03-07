openfeign和Dubbo都可以实现远程调用，他们都依赖注册中心，他们都支持负载均衡，服务熔断。

openfeign的Dubbo的区别如下，主要特点：
- 使用粒度，openfeign是基于http使用@FeignClient注解来标识远程调用服务的接口时需要加@RequestMapping，Dubbo是基于方法层面使用Dubbo请求的时候不需要添加@RequestMapping
- 并发方面，openfeign通过TCP短连接的方式进行通信不适合高并发的访问，Dubbo使用TCP长连接和Netty的方式适合高并发。
- 配置方面，Feign追求的是简洁，少侵入，因为就服务端而言不需要做任何额外的操作，而Dubbo的服务端需要配置开放的Dubbo接口。
- 熔断方面，openfeign依赖于Hystix或者Sentinel，而Dubbo内置熔断策略
- 负载均衡方面，Ribbon提供4种策略，随机、轮询、空闲策略、响应时间策略。Dubbo也提供4种策略分别是基于权重随机算法的 RandomLoadBalance、基于加权轮询算法的 RoundRobinLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance

长连接和短连接区别是什么，为什么基于短连接的Feign就不适合高并发的访问呢，为什么长连接的Dubbo就可以。
# 重点对比，长连接和短连接

说到长连接和短连接，很多人都会想到Http长连接和短连接的相关知识，例如Http1.0默认使用短连接，Http1.1默认使用长连接呀之类的，其实Http根本不分长连接和短连接，或者说“ Http的长短连接，其实是在Tcp连接中实现的 ”。

- HTTP属于应用层协议，在传输层使用TCP协议，在网络层使用IP协议。 
- IP协议主要解决网络路由和寻址问题
- TCP协议主要解决如何在IP层之上可靠地传递数据包，使得网络上接收端收到发送端所发出的所有包，并且接受顺序与发送顺序一致。

真正负责建立连接的是TCP，只有负责传输的这一层才需要建立连接！！！！！
- 对于短连接，客户端和服务器每进行一次HTTP操作，就建立一次连接，任务结束就中断连接。当客户端浏览器访问的某个HTML或其他类型的Web页中包含有其他的Web资源（如JavaScript文件、图像文件、CSS文件等），每遇到这样一个Web资源，浏览器就会重新建立一个HTTP会话。
- 对于长连接，当一个网页打开完成后，客户端和服务器之间用于传输HTTP数据的TCP连接不会关闭，客户端再次访问这个服务器时，会继续使用这一条已经建立的TCP连接，这就节省了很多TCP连接建立和断开的消耗。长连接会在响应头加入这行代码Connection:keep-alive。Keep-Alive可以在不同的服务器软件（如Apache）中人为设定。最后，实现长连接需要客户端和服务端都支持长连接。

那么综合长短连接来说呢，Feign是短连接的，那我们面对高并发的请求的时候，每请求一次数据都要建立一次连接，那肯定就不适用了。
# 重点对比，一致性哈希策略

普通哈希策略（Nginx中的粘性哈希策略）针对请求 ip 地址进行哈希，由此映射到对应的服务器上。但是缺点是服务集群扩缩容的时候，相当于服务数量N变了，会导致大量请求重新哈希重新分配，因此该策略不够稳定影响面比较大，后果是有的时候某些数据只存在于特定的服务节点上重新哈希会导致命中失败，不得不人工做数据的迁移。所以扩容时通常采用翻倍扩容，这样顶多只会发生 50% 的请求重新分配，可以将损失降到最小。

而一致性 hash 算法更加适合动态环境下的负载均衡，能够减少服务器加入和退出对整个集群的影响，提高系统的稳定性和可扩展性。

一致性 hash 算法由麻省理工学院的 Karger 及其合作者于 1997 年提出的，算法提出之初是用于大规模缓存系统的负载均衡。它的工作过程是这样的，首先根据 ip 或者其他的信息为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 232 - 1] 的圆环上。当有查询或写入请求时，则为缓存项的 key 生成一个 hash 值。然后查找第一个大于或等于该 hash 值的缓存节点，并到这个节点中查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入缓存时，为缓存项查找另一个大于其 hash 值的缓存节点即可。大致效果如下图所示，比如下面绿色点对应的缓存项将会被存储到 cache-2 节点中。由于 cache-3 挂了，原本应该存到该节点中的缓存项最终会存储到 cache-4 节点中。


![[Pasted image 20240307175940.png]]
在一致性哈希算法中，不管是增加节点，还是宕机节点，受影响的区间仅仅是增加或者宕机服务器在哈希环空间中，逆时针方向遇到的第一台服务器之间的区间，其它区间不会受到影响。

但是一致性哈希也是存在问题的，当节点很少的时候可能会出现这样的分布情况，A 服务会承担大部分请求。这种情况就叫做数据倾斜。
![[Pasted image 20240307180910.png]]
那如何解决数据倾斜的问题呢？

加入虚拟节点。

首先一个服务器根据需要可以有多个虚拟节点。假设每一台服务器都有 n 个虚拟节点。那么哈希计算时，可以使用 IP + 端口 + 编号的形式进行哈希值计算。其中的编号就是 0 到 n 的数字。由于 IP + 端口是一样的，所以这 n 个节点都是指向的同一台机器。
![[Pasted image 20240307181336.png]]
在没有加入虚拟节点之前，A 服务器承担了绝大多数的请求。但是假设每个服务器有一个虚拟节点（A-1，B-1，C-1），经过哈希计算后落在了如上图所示的位置。那么 A 服务器的承担的请求就在一定程度上（图中标注了五角星的部分）分摊给了 B-1、C-1 虚拟节点，实际上就是分摊给了 B、C 服务器。
在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。



一致性哈希在 Dubbo中的应用如下，这里相同颜色的节点均属于同一个服务提供者，比如 Invoker1-1，Invoker1-2，……, Invoker1-160。这样做的目的是通过引入虚拟节点，让 Invoker 在圆环上分散开来，避免数据倾斜问题。所谓数据倾斜是指，由于节点不够分散，导致大量请求落到了同一个节点上，而其他节点只会接收到了少量请求的情况。

![](https://pic1.zhimg.com/80/v2-e9a0deba68ba57d4ce9bba8eae875a20_720w.webp)
到这里背景知识就普及完了，接下来开始分析源码。我们先从 ConsistentHashLoadBalance 的 doSelect 方法开始看起，如下doSelect 方法主要做了一些前置工作，比如检测 invokers 列表是不是变动过，以及创建 ConsistentHashSelector。这些工作做完后，接下来开始调用 ConsistentHashSelector 的 select 方法执行负载均衡逻辑。
```java
public class ConsistentHashLoadBalance extends AbstractLoadBalance {


    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = 
        new ConcurrentHashMap<String, ConsistentHashSelector<?>>();


    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String methodName = RpcUtils.getMethodName(invocation);
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;


        // 获取 invokers 原始的 hashcode
        int identityHashCode = System.identityHashCode(invokers);
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        // 如果 invokers 是一个新的 List 对象，意味着服务提供者数量发生了变化，可能新增也可能减少了。
        // 此时 selector.identityHashCode != identityHashCode 条件成立
        if (selector == null || selector.identityHashCode != identityHashCode) {
            // 创建新的 ConsistentHashSelector
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }


        // 调用 ConsistentHashSelector 的 select 方法选择 Invoker
        return selector.select(invocation);
    }
    
    private static final class ConsistentHashSelector<T> {...}
}
```
在分析ConsistentHashSelector#select 方法之前，我们先来看一下一致性 hash 选择器 ConsistentHashSelector 的初始化过程，如下：ConsistentHashSelector 的构造方法执行了一系列的初始化逻辑，比如从配置中获取虚拟节点数以及参与 hash 计算的参数下标，默认情况下只使用第一个参数进行 hash。需要特别说明的是，ConsistentHashLoadBalance 的负载均衡逻辑只受参数值影响，具有相同参数值的请求将会被分配给同一个服务提供者。ConsistentHashLoadBalance 不 关系权重，因此使用时需要注意一下。在获取虚拟节点数和参数下标配置后，接下来要做的事情是计算虚拟节点 hash 值，并将虚拟节点存储到 TreeMap 中。到此，ConsistentHashSelector 初始化工作就完成了。
```java
private static final class ConsistentHashSelector<T> {


    // 使用 TreeMap 存储 Invoker 虚拟节点
    private final TreeMap<Long, Invoker<T>> virtualInvokers;


    private final int replicaNumber;


    private final int identityHashCode;


    private final int[] argumentIndex;


    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
        this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
        this.identityHashCode = identityHashCode;
        URL url = invokers.get(0).getUrl();
        // 获取虚拟节点数，默认为160
        this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
        // 获取参与 hash 计算的参数下标值，默认对第一个参数进行 hash 运算
        String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
        argumentIndex = new int[index.length];
        for (int i = 0; i < index.length; i++) {
            argumentIndex[i] = Integer.parseInt(index[i]);
        }
        for (Invoker<T> invoker : invokers) {
            String address = invoker.getUrl().getAddress();
            for (int i = 0; i < replicaNumber / 4; i++) {
                // 对 address + i 进行 md5 运算，得到一个长度为16的字节数组
                byte[] digest = md5(address + i);
                // 对 digest 部分字节进行4次 hash 运算，得到四个不同的 long 型正整数
                for (int h = 0; h < 4; h++) {
                    // h = 0 时，取 digest 中下标为 0 ~ 3 的4个字节进行位运算
                    // h = 1 时，取 digest 中下标为 4 ~ 7 的4个字节进行位运算
                    // h = 2, h = 3 时过程同上
                    long m = hash(digest, h);
                    // 将 hash 到 invoker 的映射关系存储到 virtualInvokers 中，
                    // virtualInvokers 需要提供高效的查询操作，因此选用 TreeMap 作为存储结构
                    virtualInvokers.put(m, invoker);
                }
            }
        }
    }
}
```
接下来，我们最后看ConsistentHashSelector#select方法的逻辑。如下，选择的过程相对比较简单了。首先是对参数进行 md5 以及 hash 运算，得到一个 hash 值。然后再拿这个值到 TreeMap 中查找目标 Invoker 即可。
```java
public Invoker<T> select(Invocation invocation) {
    // 将参数转为 key
    String key = toKey(invocation.getArguments());
    // 对参数 key 进行 md5 运算
    byte[] digest = md5(key);
    // 取 digest 数组的前四个字节进行 hash 运算，再将 hash 值传给 selectForKey 方法，
    // 寻找合适的 Invoker
    return selectForKey(hash(digest, 0));
}


private Invoker<T> selectForKey(long hash) {
    // 到 TreeMap 中查找第一个节点值大于或等于当前 hash 的 Invoker
    Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();
    // 如果 hash 大于 Invoker 在圆环上最大的位置，此时 entry = null，
    // 需要将 TreeMap 的头节点赋值给 entry
    if (entry == null) {
        entry = virtualInvokers.firstEntry();
    }


    // 返回 Invoker
    return entry.getValue();
}
```

到此关于 ConsistentHashLoadBalance 就分析完了。在阅读 ConsistentHashLoadBalance 源码之前，建议读者先补充背景知识，不然看懂代码逻辑会有很大难度。



