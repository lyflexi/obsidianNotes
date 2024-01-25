引入依赖
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
因此，用户进入购物车的时候，需要调用商品模块中该商品的最新价格，两价格之差需要展示到用户购物车中，以刺激用户消费，因此购物车模块需要如下设计：
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
# 执行调试
我们在`List<ItemDTO> items = itemClient.queryItemByIds(itemIds)`这行打上断点，
远程调用客户端itemClient是一个代理对象Proxy，里面实现了我们熟悉的老朋友InvocationHandler
![[Pasted image 20240125191350.png]]
# FeignInvocationHandler
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

# 开启feign的连接池
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
## 验证
`cart-service`再次向远程item-service服务发起请求
![[Pasted image 20240125195357.png]]