
Sentinel是阿里巴巴开源的一款服务保护框架，目前已经加入SpringCloudAlibaba中。官方网站：https://sentinelguard.io/zh-cn/
Sentinel 的使用可以分为两个部分:
- **核心库**（maven依赖）：不依赖任何框架/库，能够运行于 Java 8 及以上的版本的运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。在项目中引入依赖即可实现服务限流、隔离、熔断等功能。
- **控制台**（Dashboard，jar包）：Dashboard 主要负责管理推送规则、监控、管理机器信息等。
为了方便监控微服务，我们先把Sentinel的控制台搭建出来。下载地址：https://github.com/alibaba/Sentinel/releases
将sentinel-dashboard-1.8.6.jar包放在任意非中文、不包含特殊字符的目录下，然后运行如下命令启动控制台：
```shell
java -Dserver.port=8090 -Dcsp.sentinel.dashboard.server=localhost:8090 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.6.jar
```
访问http://localhost:8090页面，就可以看到sentinel的控制台了：登录密码sentinel/sentinel
![[Pasted image 20240126202605.png]]
# sentinel服务整合
业务回顾：防止购物车页面爆刷
我们在`cart-service`模块中整合sentinel，连接`sentinel-dashboard`控制台，步骤如下：
1）引入sentinel依赖
```XML
<!--sentinel-->
<dependency>
    <groupId>com.alibaba.cloud</groupId> 
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```
2）配置控制台

修改application.yaml文件，添加下面内容：
```YAML
spring:
  cloud: 
    sentinel:
      transport:
        dashboard: localhost:8090
```
3）访问`cart-service`的任意端点
重启`cart-service`，然后访问查询购物车接口http://localhost:18080/cart.html，sentinel的客户端就会将服务访问的信息提交到`sentinel-dashboard`控制台。并展示出统计信息：
![[Pasted image 20240126202830.png]]
点击簇点链路菜单，会看到下面的页面：
![[Pasted image 20240126202842.png]]
所谓簇点链路，就是单机调用链路，是每一个被`Sentinel`监控到的请求。默认情况下，`Sentinel`会监控`SpringMVC`的每一个`Endpoint`（接口）。

因此，我们看到`/carts`这个接口路径就是其中一个簇点，我们可以对其进行限流、熔断、隔离等保护措施。

不过，需要注意的是，我们的后端Controller接口是按照Restful风格设计，因此购物车的查询、删除、修改等接口全部都是`/carts`路径：
![[Pasted image 20240126202913.png]]
默认情况下Sentinel会把路径作为簇点资源的名称，无法区分路径相同但请求方式不同的接口，查询、删除、修改等都被识别为一个簇点资源，这显然是不合适的。

所以我们可以选择打开Sentinel的请求方式前缀，把`请求方式 + 请求路径`作为簇点资源名：

首先，在`cart-service`的`application.yml`中添加下面的配置：
```YAML
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8090
      http-method-specify: true # 开启请求方式前缀
```
然后，重启服务，通过页面访问购物车的相关接口，可以看到sentinel控制台的簇点链路发生了变化：
![[Pasted image 20240126202946.png]]


# 请求入口处：限流与配置429
在簇点链路后面点击流控按钮，即可对其做限流配置，这样就把查询购物车列表这个簇点资源的流量限制在了每秒6个，也就是最大QPS为6.
![[Pasted image 20240126203347.png]]
我们利用Jemeter做限流测试，我们每秒发出1000/100=10个请求：
![[Pasted image 20240126203057.png]]
最终监控结果如下，可以看出`GET:/carts`这个接口的通过QPS稳定在6附近，而拒绝的QPS在4附近，符合我们的预期。
![[Pasted image 20240126203434.png]]
另外，jmeter的查看结果树展示了==被拒绝的QPS返回值是异常代码429==
![[Pasted image 20240126200739.png]]
jmeter的汇总报告展示了错误率为40%左右，这就意味着每钟的10个请求当中有4个被拒绝，同样符合我们的预期
![[Pasted image 20240126203722.png]]
# feign服务间：线程隔离500

查询购物车的时候需要查询商品，这属于跨服务间的远程调用，如果远程调用查询商品服务接口经常超时，我们可以从根源上限制购物车的查询接口可使用的最大线程资源数比如只分配20个线程，这样即便查询商品服务出现故障，不会影响到购物车的增/删/改接口。

对于购物车服务自身而言，达到了线程隔离的效果

![[Pasted image 20240126203949.png]]
接下来，我们对查询商品的FeignClient接口做线程隔离。
## OpenFeign整合Sentinel

修改cart-service模块的application.yml文件，开启Feign的sentinel功能：
```YAML
feign:
  sentinel:
    enabled: true # 开启feign对sentinel的支持
```
然后重启cart-service服务，可以看到查询商品的FeignClient自动变成了一个簇点资源：
![[Pasted image 20240126204132.png]]
## 线程隔离配置

接下来，点击查询商品的FeignClient对应的簇点资源后面的流控按钮：注意，这里勾选的是并发线程数限制，也就是说这个查询功能最多使用5个线程，而不是5QPS。
![[Pasted image 20240126200624.png]]
接下来我们给查询商品接口做睡眠500ms来模拟查询商品的接口每秒处理2个请求，5个线程资源的实际QPS在10左右，而超出的请求自然会被拒绝
```java
@ApiOperation("根据id批量查询商品")  
@GetMapping  
public List<ItemDTO> queryItemByIds(@RequestParam("ids") List<Long> ids){  
    try {  
        Thread.sleep(500);  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    }  
    return itemService.queryItemByIds(ids);  
}
```


我们利用Jemeter测试，每秒发送50个请求：
![[Pasted image 20240126204540.png]]
最终测试结果如下：
![[Pasted image 20240126202155.png]]
另外，还可以测试在并发查询/carts接口接口的情况下，虽然查询/carts接口的时间为530ms左右，但并不影响/carts接口的增删改，/carts接口的增删改速度很快只有30ms左右，达到了线程隔离的作用
![[Pasted image 20240126201733.png]]
还可以在sentinel中观察，通过页面访问购物车的其它接口，例如添加购物车、修改购物车商品数量，发现不受影响：
![[Pasted image 20240126205003.png]]
响应时间非常短，这就证明线程隔离起到了作用，尽管查询购物车这个接口并发很高，但是它能使用的线程资源被限制了，因此不会影响到该服务自身的其他接口。

另外，jmeter的结果树展示被拒绝的请求错误代码为==500==
![[Pasted image 20240126202216.png]]


由于查询商品的功能耗时较高（我们模拟了500毫秒延时），再加上线程隔离限定了线程数为5，导致接口吞吐能力有限，最终QPS只有10左右。这就导致了超出的QPS上限的请求就只能抛出异常==500==，从而导致购物车的查询失败。但从业务角度来说，即便没有查询到最新的商品信息，购物车也应该展示给用户，用户体验更好。也就是给查询失败设置一个**降级处理**逻辑。
## 编写降级逻辑-等待fallback

触发限流或熔断后的请求不一定要直接报错，也可以返回一些默认数据或者友好提示，用户体验会更好。
给FeignClient编写失败后的降级逻辑有两种方式：
- 方式一：FallbackClass，无法对远程调用的异常做处理
- 方式二：FallbackFactory，可以对远程调用的异常做处理，我们一般选择这种方式。
这里我们演示方式二的失败降级处理。
步骤一：在hm-api模块中给`ItemClient`定义降级处理类，实现`FallbackFactor`，代码如下：
```Java
package com.hmall.api.client.fallback;

import com.hmall.api.client.ItemClient;
import com.hmall.api.dto.ItemDTO;
import com.hmall.api.dto.OrderDetailDTO;
import com.hmall.common.exception.BizIllegalException;
import com.hmall.common.utils.CollUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.openfeign.FallbackFactory;

import java.util.Collection;
import java.util.List;

@Slf4j
public class ItemClientFallback implements FallbackFactory<ItemClient> {
    @Override
    public ItemClient create(Throwable cause) {
        return new ItemClient() {
            @Override
            public List<ItemDTO> queryItemByIds(Collection<Long> ids) {
                log.error("远程调用ItemClient#queryItemByIds方法出现异常，参数：{}", ids, cause);
                // 查询购物车允许失败，查询失败，返回空集合
                return CollUtils.emptyList();
            }

            @Override
            public void deductStock(List<OrderDetailDTO> items) {
                // 库存扣减业务需要触发事务回滚，查询失败，抛出异常
                throw new BizIllegalException(cause);
            }
        };
    }
}
```
步骤二：在`hm-api`模块中的`com.hmall.api.config.DefaultFeignConfig`类中将`ItemClientFallback`注册为一个`Bean`：
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
import com.hmall.api.client.fallback.ItemClientFallback;  
  
/*-  
Feign默认的日志级别就是NONE，所以默认我们看不到请求日志。  
- NONE：不记录任何日志信息，这是默认值。  
- BASIC：仅记录请求的方法，URL以及响应状态码和执行时间  
- HEADERS：在BASIC的基础上，额外记录了请求和响应的头信息  
- FULL：记录所有请求和响应的明细，包括头信息、请求体、元数据。  
*/  
public class DefaultFeignConfig {  
    @Bean  
    public Logger.Level feignLogLevel() {  
        return Logger.Level.FULL;  
    }  
  
    @Bean  
    public RequestInterceptor userInfoRequestInterceptor() {  
        return new RequestInterceptor() {  
            @Override  
            public void apply(RequestTemplate template) {  
                // 获取登录用户  
                Long userId = UserContext.getUser();  
                if (userId == null) {  
                    // 如果为空则直接跳过  
                    return;  
                }  
                // 如果不为空则放入请求头中，传递给下游微服务  
                template.header("user-info", userId.toString());  
            }  
        };  
    }  
  
    @Bean  
    public ItemClientFallback itemClientFallback() {  
        return new ItemClientFallback();  
  
    }  
}
```
步骤三：在`hm-api`模块中的`ItemClient`接口中使用`ItemClientFallbackFactory`：注解声明fallbackFactory = ItemClientFallback.class
```java
package com.hmall.api.client;  
  
  
  
import com.hmall.api.dto.ItemDTO;  
import com.hmall.api.dto.OrderDetailDTO;  
import org.springframework.cloud.openfeign.FeignClient;  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.PutMapping;  
import org.springframework.web.bind.annotation.RequestBody;  
import org.springframework.web.bind.annotation.RequestParam;  
import com.hmall.api.client.fallback.ItemClientFallback;  
import java.util.Collection;  
import java.util.List;  
  
@FeignClient(value = "item-service",fallbackFactory = ItemClientFallback.class)  
public interface ItemClient {  
  
    @GetMapping("/items")  
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids);  
  
    @PutMapping("/items/stock/deduct")  
    void deductStock(@RequestBody List<OrderDetailDTO> items);  
}
```
重启后，再次测试，发现被限流的请求不再报错500，走了降级逻辑：
![[Pasted image 20240126210952.png]]
但是未被限流的请求延时依然很高，因为我们feign调用了不健康的接口-查询商品延时了500ms，导致最终的平局响应时间较长。
![[Pasted image 20240126211240.png]]
# feign服务间：熔断快速fallback
由于查询商品的延迟较高（模拟的500ms），从而导致查询购物车的响应时间也变的很长。这样不仅拖慢了购物车服务，消耗了购物车服务的更多资源，而且用户体验也很差。对于商品服务这种不太健康的接口，我们应该直接停止调用，直接走降级逻辑，避免影响到当前服务。也就是将商品查询接口熔断。
## 熔断状态机
Sentinel中的断路器可以统计某个接口的慢请求或者异常请求比例。
- 当这些比例超出阈值时，就会熔断该接口，即拦截访问该接口的一切请求，降级处理；
- 当该接口恢复正常时，再放行对于该接口的请求。
断路器的工作状态切换有一个状态机来控制：
![[Pasted image 20240126213002.png]]
状态机包括三个状态：
- **closed**：关闭状态，断路器放行所有请求，并开始统计异常比例、慢请求比例。超过阈值则切换到open状态
- **open**：打开状态，服务调用被熔断，访问被熔断服务的请求会被拒绝，快速失败，直接走降级逻辑。Open状态持续一段时间后会进入half-open状态
- **half-open**：半开状态，放行一次请求，根据执行结果来判断接下来的操作。
    - 请求成功：则切换到closed状态
    - 请求失败：则切换到open状态
## 熔断策略配置
我们可以在控制台通过点击簇点后的熔断按钮来配置熔断策略：

![[Pasted image 20240126213032.png]]
在弹出的表格中这样填写：
![[Pasted image 20240126213050.png]]、
这种是按照慢调用比例来做熔断，上述配置的含义是：
- RT超过200毫秒的请求调用就是慢调用
- 统计最近1000ms内的最少10次请求，如果慢调用比例不低于0.5，即超过5个慢调用，则触发熔断
- 熔断持续时长20s
配置完成后，再次利用Jemeter测试，为了观察断路器状态机的变化，我们把jmeter并发数量适当减小一点，减少为40个并发/s
![[Pasted image 20240126213501.png]]
可以发现在一开始一段时间是允许访问的
![[Pasted image 20240126213616.png]]
过了一段时间，达到预先设定的熔断阈值即最近1000ms内慢调用数超过了`10*0.5=5`，则触发熔断，查询商品服务的接口通过QPS直接为0
![[Pasted image 20240126213626.png]]
触发熔断状态等了20s，这个20s是我们设置的熔断时间，sentinel会进入half-open半开状态，放行一次请求，根据执行结果来判断接下来的操作。
![[Pasted image 20240126213651.png]]
很不幸，这次的结果依旧是慢调用（垃圾的）feign请求，继续熔断
![[Pasted image 20240126214114.png]]

## 编写降级逻辑-迅速fallback
激动人心的是，此时整个购物车查询服务的平均响应时间特别短，这是因为熔断之后立即进入fallback降级逻辑，没有丝毫的等待，也因此没有受到一丁点feign慢调用的影响
![[Pasted image 20240126214159.png]]
![[Pasted image 20240126214214.png]]