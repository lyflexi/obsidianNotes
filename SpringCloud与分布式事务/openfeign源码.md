Feign是Spring Cloud组件中一个轻量级RESTful的HTTP服务客户端，Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务。Feign的使用方式是：使用Feign的注解定义接口，调用接口，就可以调用服务注册中心的服务。

2019年feign停更，springcloud F 及F版本以上 ，springboot 2.0 以上基本上使用openfeign，目前大多数新项目都用openfeign：
- ==自SpringCloud2020版本开始，已经弃用内置的Ribbon，改用Spring自己开源的Spring Cloud LoadBalancer(响应式)了，我们使用的OpenFeign的也已经与其整合。==
-  ==OpenFeign在Feign的基础上支持了SpringMVC的注解，因此OpenFeign可以在FeignClient定义的远程调用接口上声明@RequestMapping等SpringMVC注解==
- 作为消费者，OpenFeign通过动态代理的方式进行远程调用。
- 作为生产者，OpenFeign通过反射的方式生成服务实例。

引入openFeign依赖
```xml
<!--openFeign-->  
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-openfeign</artifactId>  
</dependency>  
<!--负载均衡器-->  
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>  
</dependency>
```
背景交代，在分布式商城系统中，存在两个被拆分过的服务，分别是：
- item-service，商品模块，集群部署，假设有两个item-service应用实例，端口分别为8081和8083
- cart-service，购物车模块，端口为8082
购物车数据库表存储的只是用户加入商品到购物车那一时刻的快照，商品的价格在此后是存在浮动的，有可能涨价也有可能降价
因此，用户进入购物车的时候，需要额外调用商品模块查询该商品的最新价格，两价格之差需要展示到用户购物车中，以刺激用户消费，因此购物车模块需要如下设计：
# feign远程调用（不含用户信息）
购物车模块要声明ItemClient，用来远程调用item-service
```java
@FeignClient("item-service")  
public interface ItemClient {  
  
    @GetMapping("/items")  
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids);  
}
```
购物车刷新逻辑：
```java
@Autowired  
ItemClient itemClient;

@Override  
public List<CartVO> queryMyCarts() {  
    // 1.查询我的购物车列表  
    List<Cart> carts = lambdaQuery().eq(Cart::getUserId, 1L /*TODO UserContext.getUser()*/).list();  
    if (CollUtils.isEmpty(carts)) {  
        return CollUtils.emptyList();  
    }  
    // 2.转换VO  
    List<CartVO> vos = BeanUtils.copyList(carts, CartVO.class);  
    // 3.处理VO中的商品信息  
    handleCartItems(vos);  
    // 4.返回  
    return vos;  
}  
//第三次升级，升级restTemplate为openfeign  
private void handleCartItems(List<CartVO> vos) {  
    // 1.获取商品id  
    Set<Long> itemIds = vos.stream().map(CartVO::getItemId).collect(Collectors.toSet());  
    // 2.查询商品  
    // List<ItemDTO> items = itemService.queryItemByIds(itemIds);  
    //一行代码feign  
    List<ItemDTO> items = itemClient.queryItemByIds(itemIds);  
    if (CollUtils.isEmpty(items)) {  
        return;  
    }  
    // 3.转为 id 到 item的map  
    Map<Long, ItemDTO> itemMap = items.stream().collect(Collectors.toMap(ItemDTO::getId, Function.identity()));  
    // 4.写入vo  
    for (CartVO v : vos) {  
        ItemDTO item = itemMap.get(v.getItemId());  
        if (item == null) {  
            continue;  
        }  
        v.setNewPrice(item.getPrice());  
        v.setStatus(item.getStatus());  
        v.setStock(item.getStock());  
    }  
}
```
# feign服务间user信息传递
gateway通过路由转发解决了前端请求入口的问题。  
gateway还通过gateway在过滤器中调用exchange.mutate方法向传统的SpringMVC拦截器传递了用户登录信息，解决了后端统一认证鉴权的问题。
但是还有个问题gateway是无法解决的，那就是微服务之间feign如何传递用户信息?虽然每个微服务都引入了hm-common（SpringMvc：UserInfoInterceptor+UserContext），但每个微服务都有各自的UserContext为自己所用，不同微服务之间用户信息无法传递。

背景介绍：有些业务是比较复杂的，请求到达微服务后还需要调用其它多个微服务。比如下单业务trade的流程如下：
![[Pasted image 20240126151304.png]]
1. 创建订单，就位于当前交易服务trade-service
2. trade-service需要调用商品服务扣减库存，扣减库存无所谓，因为不需要用户信息
3. trade-service还需要调用购物车服务清理用户购物车。而清理购物车时必须知道当前登录的用户身份。
==但是，下单业务trade调用购物车时并没有传递用户信息，购物车服务从UserContex中获取到的用户信息为null，无法知道当前用户是谁！==
```shell
#购物车删除sql如下
queryWrapper.lambda()  
        .eq(Cart::getUserId, UserContext.getUser())  
        .in(Cart::getItemId, itemIds);
#购物车删除日志，可见user_id为null，sql返回0删除失败
14:55:05:361 DEBUG 18456 --- [nio-8082-exec-1] com.hmall.cart.mapper.CartMapper.delete  : ==>  Preparing: DELETE FROM cart WHERE (user_id = ? AND item_id IN (?))
14:55:05:361 DEBUG 18456 --- [nio-8082-exec-1] com.hmall.cart.mapper.CartMapper.delete  : ==> Parameters: null, 14741770661(Long)
14:55:05:362 DEBUG 18456 --- [nio-8082-exec-1] com.hmall.cart.mapper.CartMapper.delete  : <==    Updates: 0
```
原因解释：现在只有从gateway过来的请求才会进行请求过滤、token解析用户以及用户信息的传递，传递给下游的Springmvc拦截器存入UserContext，微服务才能拿到用户信息。对于购物车服务而言，来自于gateway的请求肯定早就结束了或者就没开始，归属于各自服务的tomcat请求结束后用户信息肯定已经被移除了UserContext.removeUser()
```java
package com.hmall.common.interceptor;  
  
  
public class UserInfoInterceptor implements HandlerInterceptor {  
    @Override  
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {  
        // 1.获取请求头中的用户信息  
        String userInfo = request.getHeader("user-info");  
        // 2.判断是否为空  
        if (StrUtil.isNotBlank(userInfo)) {  
            // 不为空，保存到ThreadLocal  
            UserContext.setUser(Long.valueOf(userInfo));  
        }  
        // 3.放行  
        return true;  
    }  
  
    @Override  
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {  
        // 移除用户  
        UserContext.removeUser();  
    }  
}
```

解决方案：由于微服务获取用户信息是通过Springmvc拦截器在请求头中读取，因此要想实现微服务之间的用户信息传递，就必须在微服务发起调用时把用户信息存入请求头。Feign的本质是restTemplate，也是http，这里要借助Feign中提供的一个拦截器接口：`feign.RequestInterceptor`，实现此接口给restTemplate中放入用户信息"user-info"
```Java
public interface RequestInterceptor {

  /**
   * Called for every request. 
   * Add data using methods on the supplied {@link RequestTemplate}.
   */
  void apply(RequestTemplate template);
}
```
在`com.hmall.api.config.DefaultFeignConfig`中添加一个Bean：RequestInterceptor
```java
package com.hmall.api.config;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/25 20:33  
 */  
import com.hmall.common.utils.UserContext;  
import feign.Logger;  
import feign.RequestInterceptor;  
import feign.RequestTemplate;  
import org.springframework.context.annotation.Bean;  
public class DefaultFeignConfig {  
    @Bean  
    public Logger.Level feignLogLevel(){  
        return Logger.Level.FULL;  
    }  
    @Bean  
    public RequestInterceptor userInfoRequestInterceptor(){  
        return new RequestInterceptor() {  
            @Override  
            public void apply(RequestTemplate template) {  
                // 获取登录用户  
                Long userId = UserContext.getUser();  
                if(userId == null) {  
                    // 如果为空则直接跳过  
                    return;  
                }  
                // 如果不为空则放入请求头中，传递给下游微服务  
                template.header("user-info", userId.toString());  
            }  
        };  
    }  
}
```
并且在trade-service（feign客户端使用方）主启动类上指定feign配置类，让当前服务trade-service的用户信息添加到feign请求内部发送
```java
@EnableFeignClients(basePackages = "com.hmall.api.client", defaultConfiguration = DefaultFeignConfig.class)  
@MapperScan("com.hmall.trade.mapper")  
@SpringBootApplication  
public class TradeApplication {  
    public static void main(String[] args) {  
        SpringApplication.run(TradeApplication.class, args);  
    }  
}
```
重启项目，测试下单，发现购物车删除成功
```shell
16:53:37:467 DEBUG 7832 --- [nio-8082-exec-8] com.hmall.cart.mapper.CartMapper.delete  : ==>  Preparing: DELETE FROM cart WHERE (user_id = ? AND item_id IN (?))
16:53:37:468 DEBUG 7832 --- [nio-8082-exec-8] com.hmall.cart.mapper.CartMapper.delete  : ==> Parameters: 1(Long), 40305713537(Long)
16:53:37:473 DEBUG 7832 --- [nio-8082-exec-8] com.hmall.cart.mapper.CartMapper.delete  : <==    Updates: 1
```
# FeignInvocationHandler
我们在`List<ItemDTO> items = itemClient.queryItemByIds(itemIds)`这行打上断点，
远程调用客户端itemClient是一个代理对象Proxy，里面实现了我们熟悉的老朋友InvocationHandler
![[Pasted image 20240125191350.png]]
进入FeignInvocationHandler的invoke方法
```java
@Override  
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
  if ("equals".equals(method.getName())) {  
    try {  
      Object otherHandler =  
          args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;  
      return equals(otherHandler);  
    } catch (IllegalArgumentException e) {  
      return false;  
    }  
  } else if ("hashCode".equals(method.getName())) {  
    return hashCode();  
  } else if ("toString".equals(method.getName())) {  
    return toString();  
  }  
  
  return dispatch.get(method).invoke(args);  
}
```
# SynchronousMethodHandler
来到return dispatch.get(method).invoke(args)，进入了SynchronousMethodHandler的invoke方法
## invoke
```java
@Override  
public Object invoke(Object[] argv) throws Throwable {  
  RequestTemplate template = buildTemplateFromArgs.create(argv);  
  Options options = findOptions(argv);  
  Retryer retryer = this.retryer.clone();  
  while (true) {  
    try {  
      return executeAndDecode(template, options);  
    } catch (RetryableException e) {  
      try {  
        retryer.continueOrPropagate(e);  
      } catch (RetryableException th) {  
        Throwable cause = th.getCause();  
        if (propagationPolicy == UNWRAP && cause != null) {  
          throw cause;  
        } else {  
          throw th;  
        }  
      }  
      if (logLevel != Logger.Level.NONE) {  
        logger.logRetry(metadata.configKey(), logLevel);  
      }  
      continue;  
    }  
  }  
}
```
buildTemplateFromArgs.create执行过后返回了RequestTemplate对象，RequestTemplate保存了“GET /items?ids=100000006163 HTTP/1.1”，
RequestTemplate只有远程方法信息，却没有请求地址信息：
- 没有请求ip
- 没有请求端口
![[Pasted image 20240125191846.png]]
继续向下执行到return executeAndDecode(template, options)这行

## executeAndDecode
方法的第一行执行了Request request = targetRequest(template)，返回了Rrquest对象，request里面保存了“GET http://item-service/items?ids=100000006163 HTTP/1.1”，可以看到request比template多了目标服务名item-service
![[Pasted image 20240125192300.png]]
继续向下执行到response = client.execute(request, options)这行，进入
# FeignBlockingLoadBalancerClient
execute方法首先从request中拿到原始url：http://item-service/items?ids=100000006163
然后从原始url中取出服务id也就是远程服务名item-service
```java
@Override  
public Response execute(Request request, Request.Options options) throws IOException {  
    final URI originalUri = URI.create(request.url());  
    String serviceId = originalUri.getHost();  
    Assert.state(serviceId != null, "Request URI does not contain a valid hostname: " + originalUri);  
    String hint = getHint(serviceId);  
    DefaultRequest<RequestDataContext> lbRequest = new DefaultRequest<>(  
          new RequestDataContext(buildRequestData(request), hint));  
    Set<LoadBalancerLifecycle> supportedLifecycleProcessors = LoadBalancerLifecycleValidator  
          .getSupportedLifecycleProcessors(  
                loadBalancerClientFactory.getInstances(serviceId, LoadBalancerLifecycle.class),  
                RequestDataContext.class, ResponseData.class, ServiceInstance.class);  
    supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onStart(lbRequest));  
    //负载均衡获取远程服务中的一个实例
    ServiceInstance instance = loadBalancerClient.choose(serviceId, lbRequest);  
    org.springframework.cloud.client.loadbalancer.Response<ServiceInstance> lbResponse = new DefaultResponse(  
          instance);  
    if (instance == null) {  
       String message = "Load balancer does not contain an instance for the service " + serviceId;  
       if (LOG.isWarnEnabled()) {  
          LOG.warn(message);  
       }  
       supportedLifecycleProcessors.forEach(lifecycle -> lifecycle  
             .onComplete(new CompletionContext<ResponseData, ServiceInstance, RequestDataContext>(  
                   CompletionContext.Status.DISCARD, lbRequest, lbResponse)));  
       return Response.builder().request(request).status(HttpStatus.SERVICE_UNAVAILABLE.value())  
             .body(message, StandardCharsets.UTF_8).build();  
    }  
    String reconstructedUrl = loadBalancerClient.reconstructURI(instance, originalUri).toString();  
    Request newRequest = buildRequest(request, reconstructedUrl);  
    LoadBalancerProperties loadBalancerProperties = loadBalancerClientFactory.getProperties(serviceId);  
    return executeWithLoadBalancerLifecycleProcessing(delegate, options, newRequest, lbRequest, lbResponse,  
          supportedLifecycleProcessors, loadBalancerProperties.isUseRawStatusCodeInResponseData());  
}
```
## loadBalancerClient.choose
继续向下执行，来到了ServiceInstance instance = loadBalancerClient.choose(serviceId, lbRequest)，负载均衡获取远程服务中的一个实例
此时获取的实例instance信息的端口为8081
![[Pasted image 20240125193151.png]]
我们放行程序，重新发起请求调用，看什么时候能够负载均衡到远程的8083服务端口：可以看到我又尝试了多次请求之后，8083也有了提供服务的机会
![[Pasted image 20240125193420.png]]
我们知道。openfeign是对resttemplate做了封装，他的本质还是执行的http请求调用，
但是到现在为止，只知道原始请求uri：http://item-service/items?ids=100000006163并没有远程服务ip
## loadBalancerClient.reconstructURI
程序继续向下执行，来到了loadBalancerClient.reconstructURI(instance, originalUri).toString()返回reconstructedUrl，终于拿到了真正的服务地址
http://192.168.127.1:8083/items?ids=100000006163
![[Pasted image 20240125193923.png]]

## executeWithLoadBalancerLifecycleProcessing
程序的最后，终于要发送请求了，来到了excute方法的最后一行，传人了delegate（翻译过来叫委托），它是个Client接口
```java
public class FeignBlockingLoadBalancerClient implements Client {  
  
    private static final Log LOG = LogFactory.getLog(FeignBlockingLoadBalancerClient.class);  
  
    private final Client delegate;  
  
    private final LoadBalancerClient loadBalancerClient;  
  
    private final LoadBalancerClientFactory loadBalancerClientFactory;

	@Override  
	public Response execute(Request request, Request.Options options) throws IOException {  
		...
		return executeWithLoadBalancerLifecycleProcessing(delegate, options, newRequest, lbRequest, lbResponse, supportedLifecycleProcessors, loadBalancerProperties.isUseRawStatusCodeInResponseData());
	}
}
```

我们查看下delegate具体是个什么，是Client接口的内部类Default
![[Pasted image 20240125194651.png]]
# Default（HttpURLConnection）
Default的底层用的是HttpURLConnection发送请求
```java
@Override  
public Response execute(Request request, Options options) throws IOException {  
  HttpURLConnection connection = convertAndSend(request, options);  
  return convertResponse(connection, request);  
}
```

# 引入feign的连接池
Feign底层发起http请求，依赖于其它的框架。其底层支持的http客户端实现包括：
- HttpURLConnection：默认实现，不支持连接池
- Apache HttpClient ：支持连接池
- OKHttp：支持连接池

因此我们通常会使用带有连接池的客户端来代替默认的HttpURLConnection。比如OK Http.
## 引入OKHttp连接池
在`cart-service`的`pom.xml`中引入依赖：
```xml
<!--OK http 的依赖 -->
<dependency>
  <groupId>io.github.openfeign</groupId>
  <artifactId>feign-okhttp</artifactId>
</dependency>
```
在`cart-service`的`application.yml`配置文件中开启Feign的连接池功能：
```shell
feign:
  okhttp:
    enabled: true # 开启OKHttp功能
```
重启`cart-service`服务，连接池就生效了。
## 验证连接池
`cart-service`再次向远程item-service服务发起请求
![[Pasted image 20240125195357.png]]
# OpenFeign负载均衡原理loadBalancerClient.choose
所以feign负载均衡的关键就是FeignBlockingLoadBalancerClient#execute方法里调用了loadBalancerClient来通过lb的方式选择服务实例
```java
ServiceInstance instance = loadBalancerClient.choose(serviceId, lbRequest);
```
## loadBalancerClient
loadBalancerClient的类型是`org.springframework.cloud.client.loadbalancer.LoadBalancerClient`，这是`Spring-Cloud-Common`模块中定义的接口，只有一个实现类：
![[Pasted image 20240127142728.png]]
而这里的`org.springframework.cloud.client.loadbalancer.BlockingLoadBalancerClient`正是`Spring-Cloud-LoadBalancer`模块下的一个类：
![[Pasted image 20240127142915.png]]
我们跟入`BlockingLoadBalancerClient#choose()`方法：发现feign的底层，用的是响应式负载均衡器ReactiveLoadBalancer：
`ReactiveLoadBalancer`是`Spring-Cloud-Common`组件中定义的负载均衡器接口规范，而`Spring-Cloud-Loadbalancer`组件给出了两个个实现：
- RoundRobinLoadBalancer
- RoundLoadBalancer
==这也印证了文章开头处，自SpringCloud2020版本开始，已经弃用内置的Ribbon，改用Spring自己开源的Spring Cloud LoadBalancer(响应式)了，我们使用的OpenFeign的也已经与其整合。==
![[Pasted image 20240127143210.png]]
此处用的是默认实现`RoundRobinLoadBalancer`，即轮询负载均衡器：下图中代码的核心逻辑如下：
- 根据serviceId找到这个服务采用的负载均衡器（`ReactiveLoadBalancer`），也就是说我们可以给每个服务配不同的负载均衡算法。
- 利用负载均衡器（`ReactiveLoadBalancer`）中的负载均衡算法，选出一个服务实例
![[Pasted image 20240127143141.png]]
## RoundRobinLoadBalancer
默认RoundRobinLoadBalancer的负载均衡器choose的核心逻辑如下：
![[Pasted image 20240127143338.png]]
核心流程就是两步：
- 利用`ServiceInstanceListSupplier#get()`方法拉取服务的实例列表，这一步是采用响应式编程
- 利用本类，也就是`RoundRobinLoadBalancer`的`getInstanceResponse()`方法挑选一个实例，这里采用了轮询算法来挑选。
## ServiceInstanceListSupplier
这里的ServiceInstanceListSupplier有很多实现：
![[Pasted image 20240127143441.png]]
其中CachingServiceInstanceListSupplier采用了装饰模式，加了服务实例列表缓存，避免每次都要去注册中心拉取服务实例列表。而其内部是基于`DiscoveryClientServiceInstanceListSupplier`来实现的。

在这个DiscoveryClientServiceInstanceListSupplier类的构造函数中，就会异步的基于DiscoveryClient去拉取服务的实例列表：
![[Pasted image 20240127143510.png]]
# Feign的一些自定义配置

## 超时时间设置

OpenFeign默认超时时间为1s，超过1s就会返回错误页面。

如果我们的接口处理业务确实超过1s，就需要对接口进行超时配置，如下：

```YAML
ribbon: #设置feign客户端连接所用的超时时间，适用于网络状况正常情况下，两端连接所用时间
  ReadTimeout: 1500 #指的是建立连接所用时间
  ConnectTimeout: 1500 #指建立连接后从服务读取到可用资源所用时间
```

## 日志级别设置

Feign的日志级别分为四种：
- NONE：不记录任何日志信息，默认
- BASIC：仅记录请求的方法，URL以及响应状态和执行时间
- HEADERS：在BASIC的基础上，加上请求与响应头信息
- FULL：记录所有请求和响应的明细，包括头信息、请求体、响应信息，元数据

基于配置文件修改feign的日志级别
```YAML
#针对单个服务：
feign:  
  client:
    config: 
      someservice: # 针对某个微服务的配置
        loggerLevel: FULL #  日志级别
#针对针对所有服务
feign:  
  client:
    config: 
      default: # 这里用default就是全局配置
        loggerLevel: FULL #  日志级别
```

基于Java代码修改feign的日志级别，只需要创建自定义的@Bean覆盖默认Bean即可。基于代码方式修改feign的日志级别需要定义配置类比如`DefaultFeignConfiguration`
```Java
import feign.Logger;
import org.springframework.context.annotation.Bean;

public class DefaultFeignConfiguration {

    @Bean
    public Logger.Level logLevel() {
        return Logger.Level.BASIC;
    }
}
```
接下来，要让日志级别生效，还需要配置这个类。有两种方式：
- 局部生效：在某个`FeignClient`中配置，只对当前`FeignClient`生效
```java
```Java
@FeignClient(value = "item-service", configuration = DefaultFeignConfig.class)
```
- 全局生效：在启动类上的`@EnableFeignClients`中配置，针对所有`FeignClient`生效。
```Java
@EnableFeignClients(defaultConfiguration = DefaultFeignConfig.class)
```
