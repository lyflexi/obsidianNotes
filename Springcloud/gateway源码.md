背景交代：

> 至此，我们将黑马商城拆分为5个微服务：  
> 用户服务  
> 商品服务  
> 购物车服务  
> 交易服务  
> 支付服务  
> 还有横向拆分的hm-api，统一存放所有的feign远程调用客户端  


服务拆分之后，相信大家在与前端联调的时候发现了两个问题：  
1. 由于每个微服务都有不同的地址或端口，入口不同，请求不同数据时要访问不同的入口，需要维护多个入口地址，麻烦  
2. 而微服务拆分后，每个微服务都独立部署，每个微服务都需要编写登录校验、用户信息获取的功能吗？回顾单体架构时我们只需要完成一次用户登录、身份校验，就可以在所有业务中获取到用户信息，单体架构的用户信息传递方案是springmvc拦截器和ThreadLocal存储
	- 定义用户上下文UserContext，持有ThreadLocal引用
	- 定义springmvc拦截器，将保存在用户上下文中的UserContext的ThreadLocal中
	- 在Controller层获取UserContext的ThreadLocal信息
```java
package com.hmall.interceptor;  
  
import com.hmall.common.utils.UserContext;  
import com.hmall.utils.JwtTool;  
import lombok.RequiredArgsConstructor;  
import org.springframework.web.servlet.HandlerInterceptor;  
  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
  
@RequiredArgsConstructor  
public class LoginInterceptor implements HandlerInterceptor {  
  
    private final JwtTool jwtTool;  
  
    @Override  
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {  
        // 1.获取请求头中的 token        String token = request.getHeader("authorization");  
        // 2.校验token  
        Long userId = jwtTool.parseToken(token);  
        // 3.存入上下文  
        UserContext.setUser(userId);  
        // 4.放行  
        return true;  
    }  
  
    @Override  
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {  
        // 清理用户  
        UserContext.removeUser();  
    }  
}


package com.hmall.common.utils;  
  
public class UserContext {  
    private static final ThreadLocal<Long> tl = new ThreadLocal<>();  
  
    /**  
     * 保存当前登录用户信息到ThreadLocal  
     * @param userId 用户id  
     */    public static void setUser(Long userId) {  
        tl.set(userId);  
    }  
  
    /**  
     * 获取当前登录用户信息  
     * @return 用户id  
     */    public static Long getUser() {  
        return tl.get();  
    }  
  
    /**  
     * 移除当前登录用户信息  
     */  
    public static void removeUser(){  
        tl.remove();  
    }  
}

@ApiOperation("查询当前用户地址列表")  
@GetMapping  
public List<AddressDTO> findMyAddresses() {  
    // 1.查询列表  
    List<Address> list = addressService.query().eq("user_id", UserContext.getUser()).list();  
    // 2.判空  
    if (CollUtils.isEmpty(list)) {  
        return CollUtils.emptyList();  
    }  
    // 3.转vo  
    return BeanUtils.copyList(list, AddressDTO.class);  
}
```
我们会通过分布式网关技术解决上述问题，新建hm-gateway微服务模块
1. 网关鉴权，解决统一登录校验和用户信息获取的问题。  
2. 网关路由转发，解决前端请求入口的问题。  
还有一点，对于每一个模块比如商品模块item-service，生产环境往往会对这个模块做集群部署，此时==网关还能够在后端进行负载均衡寻址==
# uri+predicates解决请求转发问题
网关调用nacos，实时更新服务列表  
![[Pasted image 20240126104602.png]]
## gate为我们提供好的predicates
查看gateway的自动配置属性GatewayProperties
```java

package org.springframework.cloud.gateway.config;  

/**  
 * @author Spencer Gibb  
 */@ConfigurationProperties(GatewayProperties.PREFIX)  
@Validated  
public class GatewayProperties {  
  
    /**  
     * Properties prefix.     */    
    public static final String PREFIX = "spring.cloud.gateway";  
  
    private final Log logger = LogFactory.getLog(getClass());  
  
    /**  
     * List of Routes.     */    
    @NotNull  
    @Valid    
    private List<RouteDefinition> routes = new ArrayList<>();  
  
    /**  
     * List of filter definitions that are applied to every route.     */    
    private List<FilterDefinition> defaultFilters = new ArrayList<>();  
  
    private List<MediaType> streamingMediaTypes = Arrays.asList(MediaType.TEXT_EVENT_STREAM,  
          MediaType.APPLICATION_STREAM_JSON, new MediaType("application", "grpc"),  
          new MediaType("application", "grpc+protobuf"), new MediaType("application", "grpc+json"));  
  
    /**  
     * Option to fail on route definition errors, defaults to true. Otherwise, a warning     * is logged.     */    
    private boolean failOnRouteDefinitionError = true;  
  

    @Override  
    public String toString() {  
       return new ToStringCreator(this).append("routes", routes).append("defaultFilters", defaultFilters)  
             .append("streamingMediaTypes", streamingMediaTypes)  
             .append("failOnRouteDefinitionError", failOnRouteDefinitionError).toString();  
  
    }  
  
}
```
这些路由信息都被保存在了`List<RouteDefinition>`中
```java
package org.springframework.cloud.gateway.route;  

  
/**  
 * @author Spencer Gibb  
 */@Validated  
public class RouteDefinition {  
  
    private String id;  
  
    @NotEmpty  
    @Valid    private List<PredicateDefinition> predicates = new ArrayList<>();  
  
    @Valid  
    private List<FilterDefinition> filters = new ArrayList<>();  
  
    @NotNull  
    private URI uri;  
  
    private Map<String, Object> metadata = new HashMap<>();  
  
    private int order = 0;  
}
```

除了Path类的断言之外，RoutePredicateFactory一共提供了如下十二种断言规则，
每一种断言的参数均以逗号隔开

|**名称**|**说明**|**示例**|
|---|---|---|
|After|是某个时间点后的请求|- After=2037-01-20T17:42:47.789-07:00（America/Denver）|
|Before|是某个时间点之前的请求|- Before=2031-04-13T15:14:47.433+08:00（Asia/Shanghai）|
|Between|是某两个时间点之前的请求|- Between=2037-01-20T17:42:47.789-07:00（America/Denver）, 2037-01-21T17:42:47.789-07:00（America/Denver）|
|Cookie|请求必须包含某些cookie|- Cookie=chocolate, ch.p|
|Header|请求必须包含某些header|- Header=X-Request-Id, \d+|
|Host|请求必须是访问某个host（域名）|- Host=`**`.somehost.org,`**`.anotherhost.org|
|Method|请求方式必须是指定方式|- Method=GET,POST|
|Path|请求路径必须符合指定规则|- Path=/red/{segment},/blue/`**`|
|Query|请求参数必须包含指定参数|- Query=name, Jack或者- Query=name|
|RemoteAddr|请求者的ip必须是指定范围|- RemoteAddr=192.168.1.1/24|
|weight|权重处理||
## 使用内置predicates解决请求转发问题
该项目路由配置如下：
```shell
server:  
  port: 8080  
spring:  
  application:  
    name: gateway  
  cloud:  
    nacos:  
      server-addr: 192.168.18.100:8848  
    gateway:  
      routes:  
        - id: item # 路由规则id，自定义，唯一  
          uri: lb://item-service # 路由的目标服务，lb代表负载均衡，会从注册中心拉取服务列表  
          predicates: # 路由断言，判断当前请求是否符合当前规则，符合则路由到目标服务  
            - Path=/items/**,/search/** # 这里是以请求路径作为判断规则  
        - id: cart  
          uri: lb://cart-service  
          predicates:  
            - Path=/carts/**  
        - id: user  
          uri: lb://user-service  
          predicates:  
            - Path=/users/**,/addresses/**  
        - id: trade  
          uri: lb://trade-service  
          predicates:  
            - Path=/orders/**  
        - id: pay  
          uri: lb://pay-service  
          predicates:  
            - Path=/pay-orders/**
```
# filter解决登录校验问题
## geteway为我们提供好的filter
Gateway内置了很多的GatewayFilter，详情可以参考官方文档：https://docs.spring.io/spring-cloud-gateway/docs/3.1.7/reference/html/#gatewayfilter-factories。我们可以配置yml文件来使这些内置的filter生效
GatewayProperties里面保存了全局filter
![[Pasted image 20240126103251.png]]
全局filter的yml配置类似于这样：
```YAML
spring:
  cloud:
    gateway:
      default-filters: # default-filters下的过滤器可以作用于所有路由
        - AddRequestHeader=key, value
      routes:
      - id: test_route
        uri: lb://test-service
        predicates:
          -Path=/test/**
```
另外，RouteDefinition里面保存了局部filter
![[Pasted image 20240126103356.png]]
局部filter的yml配置类似于这样：
```YAML
spring:
  cloud:
    gateway:
      routes:
      - id: test_route
        uri: lb://test-service
        predicates:
          -Path=/test/**
        filters:
          - AddRequestHeader=key, value # 逗号之前是请求头的key，逗号之后是value
```

## 自定义过滤器
一般我们的过滤器只是用Pre即可，也即前置过滤
![[Pasted image 20240126104423.png]]

### 自定义GatewayFilter
自定义PrintAnyGatewayFilterFactory类名必须以“GatewayFilterFactory”结尾
```java
package com.hmall.gateway.filter;  
  
import lombok.Data;  
import org.springframework.cloud.gateway.filter.GatewayFilter;  
import org.springframework.cloud.gateway.filter.GatewayFilterChain;  
import org.springframework.cloud.gateway.filter.OrderedGatewayFilter;  
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;  
import org.springframework.stereotype.Component;  
import org.springframework.web.server.ServerWebExchange;  
import reactor.core.publisher.Mono;  
  
import java.util.List;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/26 11:48  
 */  
@Component  
public class PrintAnyGatewayFilterFactory // 父类泛型是内部类的Config类型  
        extends AbstractGatewayFilterFactory<PrintAnyGatewayFilterFactory.Config> {  
  
    @Override  
    public GatewayFilter apply(Config config) {  
        // OrderedGatewayFilter是GatewayFilter的子类，包含两个参数：  
        // - GatewayFilter：过滤器  
        // - int order值：值越小，过滤器执行优先级越高  
        return new OrderedGatewayFilter(new GatewayFilter() {  
            @Override  
            public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
                // 获取config值  
                String a = config.getA();  
                String b = config.getB();  
                String c = config.getC();  
                // 编写过滤器逻辑  
                System.out.println("a = " + a);  
                System.out.println("b = " + b);  
                System.out.println("c = " + c);  
                // 放行  
                return chain.filter(exchange);  
            }  
        }, 100);  
    }  
  
    // 自定义配置属性，成员变量名称很重要，下面会用到  
    @Data  
    static class Config{  
        private String a;  
        private String b;  
        private String c;  
    }  
    // 将变量名称依次返回，顺序很重要，将来读取参数时需要按顺序获取  
    @Override  
    public List<String> shortcutFieldOrder() {  
        return List.of("a", "b", "c");  
    }  
    // 返回当前配置类的类型，也就是内部的Config  
    @Override  
    public Class<Config> getConfigClass() {  
        return Config.class;  
    }  
  
}
```
自定义PrintAnyGatewayFilterFactory定义出来之后，需要在yml配置文件中指定自己，规则如下：
- 过滤器名字是PrintAny
- 既可以放到全局过滤器位置生效，也可以放在局部过滤器位置生效
```YAML
#全局生效
spring:
  cloud:
    gateway:
      default-filters:
		- PrintAny=1,2,3 # 注意，这里多个参数以","隔开，将来会按照shortcutFieldOrder()方法返回的参数顺序依次复制
#局部生效
spring:
  cloud:
    gateway:
      routes:
      - id: test_route
        uri: lb://test-service
        predicates:
          -Path=/test/**
        filters:
          - PrintAny=1,2,3 
```

上面这种配置方式参数必须严格按照shortcutFieldOrder()方法的返回参数名顺序来赋值。

还有一种用法，无需按照这个顺序，就是手动指定参数名：
```YAML
#全局生效
spring:
  cloud:
    gateway:
      default-filters:
		- name: PrintAny
		  args: # 手动指定参数名，无需按照参数顺序
			a: 1
			b: 2
			c: 3
#局部生效
spring:
  cloud:
    gateway:
      routes:
      - id: test_route
        uri: lb://test-service
        predicates:
          -Path=/test/**
        filters:
			- name: PrintAny
			  args: # 手动指定参数名，无需按照参数顺序
				a: 1
				b: 2
				c: 3
```
### 自定义GlobalFilter
自定义GlobalFilter则简单很多，实现GlobalFilter即可，不用配置yml文件，但是却无法设置动态参数，
```java
package com.hmall.gateway.filter;  
  
import org.springframework.cloud.gateway.filter.GatewayFilterChain;  
import org.springframework.cloud.gateway.filter.GlobalFilter;  
import org.springframework.core.Ordered;  
import org.springframework.http.server.reactive.ServerHttpResponse;  
import org.springframework.stereotype.Component;  
import org.springframework.web.server.ServerWebExchange;  
import reactor.core.publisher.Mono;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/26 11:47  
 */@Component  
public class MyGlobalFilter implements GlobalFilter, Ordered {  
    @Override  
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
        // 编写过滤器逻辑  
        System.out.println("未登录，无法访问");  
        // 放行  
        // return chain.filter(exchange);  
  
        // 拦截  
        ServerHttpResponse response = exchange.getResponse();  
        response.setRawStatusCode(401);  
        return response.setComplete();  
    }  
  
    @Override  
    public int getOrder() {  
        // 过滤器执行顺序，值越小，优先级越高  
        return 0;  
    }  
}
```
## 使用自定义GlobalFilter过滤请求并解析token
登录校验需要用到JWT，而且JWT的加密需要秘钥和加密工具。
- `AuthProperties`：配置登录校验需要拦截的路径，因为不是所有的路径都需要登录才能访问
- `JwtProperties`：定义与JWT工具有关的属性，比如秘钥文件位置
- `SecurityConfig`：工具的自动装配
- `JwtTool`：JWT工具，其中包含了校验和解析`token`的功能
- `hmall.jks`：秘钥文件
其中`AuthProperties`和`JwtProperties`所需的属性要在`application.yaml`中配置：
```shell
hm:
  jwt:
    location: classpath:hmall.jks # 秘钥地址
    alias: hmall # 秘钥别名
    password: hmall123 # 秘钥文件密码
    tokenTTL: 30m # 登录有效期
  auth:
    excludePaths: # 无需登录校验的路径
      - /search/**
      - /users/login
      - /items/**
```

==我们需要在网关转发之前做请求过滤==，对于不需要权限的请求全部放行，对于需要权限的请求进行过滤
==我们还需要在网关转发之前做登录校验==，所谓登录校验就是解析token，步骤如下：
- 网关拦截器取出请求头中的authorization即token
- 网关调用jwtTool解析token生成userid
```Java
package com.hmall.gateway.filter;

import com.hmall.common.exception.UnauthorizedException;
import com.hmall.common.utils.CollUtils;
import com.hmall.gateway.config.AuthProperties;
import com.hmall.gateway.util.JwtTool;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;

@Component
//推荐使用构造参数注入
@RequiredArgsConstructor
@EnableConfigurationProperties(AuthProperties.class)
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    private final JwtTool jwtTool;

    private final AuthProperties authProperties;

    private final AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1.获取Request
        ServerHttpRequest request = exchange.getRequest();
        // 2.判断是否不需要拦截
        if(isExclude(request.getPath().toString())){
            // 无需拦截，直接放行
            return chain.filter(exchange);
        }
        // 3.获取请求头中的token
        String token = null;
        List<String> headers = request.getHeaders().get("authorization");
        if (!CollUtils.isEmpty(headers)) {
            token = headers.get(0);
        }
        // 4.校验并解析token
        Long userId = null;
        try {
            userId = jwtTool.parseToken(token);
        } catch (UnauthorizedException e) {
            // 如果无效，拦截
            ServerHttpResponse response = exchange.getResponse();
            response.setRawStatusCode(401);
            return response.setComplete();
        }

        // TODO 5.如果有效，传递用户信息
        System.out.println("userId = " + userId);
        // 6.放行
        return chain.filter(exchange);
    }

    private boolean isExclude(String antPath) {
        for (String pathPattern : authProperties.getExcludePaths()) {
            if(antPathMatcher.match(pathPattern, antPath)){
                return true;
            }
        }
        return false;
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```
重启测试，会发现访问/items开头的路径，未登录状态下不会被拦截。
![[Pasted image 20240126135008.png]]
访问其他路径则，未登录状态下请求会被拦截，并且返回`401`状态码：
![[Pasted image 20240126135039.png]]


# 下游微服务如何获取网关filter中的用户
至此，网关已经可以完成请求过滤，登录校验（token解析）并获取登录用户身份信息。但是当网关将请求转发到微服务时，微服务又该如何获取用户身份呢？

过去，用户请求直接打到SpringMVC，SpringMVC拦截器保存了用户信息到UserContext的ThreadLocal中，我们的下游服务就可以在Controller的位置从UserContext中获取的用户信息。

==现在，SpringMVC拦截器前面挡了一层gateway的filter，用户信息没有直达SpringMVC拦截器==，这造成的后果就是原本的UserContext失效了
![[Pasted image 20240126135857.png]]
![[Pasted image 20240126132303.png]]

由于网关发送请求到微服务依然采用的是`Http`请求，因此我们可以将用户信息以请求头的方式传递到下游微服务。然后微服务可以从请求头中获取登录用户信息。流程图如下：
![[Pasted image 20240126140815.png]]
#### 2.1 网关改写请求头，放入userid，转发给下游服务
网关调用exchange.mutate改写请求头，放入userid，将user信息传递给下游服务
![[Pasted image 20240126141912.png]]
filter执行exchange.mutate() 方法重写请求头，用户信息定义为"user-info"，值为userID
```java
@Override  
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
    // 1.获取Request  
    ServerHttpRequest request = exchange.getRequest();  
    // 2.判断是否不需要拦截  
    if(isExclude(request.getPath().toString())){  
        // 无需拦截，直接放行  
        return chain.filter(exchange);  
    }  
    // 3.获取请求头中的token  
    String token = null;  
    List<String> headers = request.getHeaders().get("authorization");  
    if (!CollUtils.isEmpty(headers)) {  
        token = headers.get(0);  
    }  
    // 4.校验并解析token  
    Long userId = null;  
    try {  
        userId = jwtTool.parseToken(token);  
    } catch (UnauthorizedException e) {  
        // 如果无效，拦截  
        ServerHttpResponse response = exchange.getResponse();  
        response.setRawStatusCode(401);  
        //终止拦截器，并终止后续拦截器，并终止请求  
        return response.setComplete();  
    }  
  
    //  5.如果有效，传递用户信息  
    String userInfo = userId.toString();  
    ServerWebExchange ex = exchange.mutate()  
            .request(b -> b.header("user-info", userInfo))  
            .build();  
    // 6.放行  
    return chain.filter(ex);  
}
```

#### 2.2 下游定义springmvc拦截器，接收filter请求
编写springmvc拦截器以及UserContext，拦截请求获取用户信息，保存到ThreadLocal后放行。直接放行，此处不做拦截，因为前面的gateway的filter相当于已经帮我们拦截过了

由于下游微服务众多，所以要把springmvc拦截器的逻辑抽取到公共模块hm-common，其他微服务模块均引入公共模块即可

![[Pasted image 20240126131802.png]]
springmvc拦截器UserInfoInterceptor如下
```Java
package com.hmall.common.interceptor;

import cn.hutool.core.util.StrUtil;
import com.hmall.common.utils.UserContext;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

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


package com.hmall.common.utils;  
  
public class UserContext {  
    private static final ThreadLocal<Long> tl = new ThreadLocal<>();  
  
    /**  
     * 保存当前登录用户信息到ThreadLocal  
     * @param userId 用户id  
     */    public static void setUser(Long userId) {  
        tl.set(userId);  
    }  
  
    /**  
     * 获取当前登录用户信息  
     * @return 用户id  
     */    public static Long getUser() {  
        return tl.get();  
    }  
  
    /**  
     * 移除当前登录用户信息  
     */  
    public static void removeUser(){  
        tl.remove();  
    }  
}
```
MvcConfig如下，
```java
package com.hmall.common.config;  
  
import com.hmall.common.interceptor.UserInfoInterceptor;  
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.web.servlet.DispatcherServlet;  
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;  
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;  
  
@Configuration  
@ConditionalOnClass(DispatcherServlet.class)  
public class MvcConfig implements WebMvcConfigurer {  
    @Override  
    public void addInterceptors(InterceptorRegistry registry) {  
        registry.addInterceptor(new UserInfoInterceptor());  
    }  
}
```

不过，需要注意的是，这个配置类默认是不会生效的，因为它所在的包是`com.hmall.common.config`，与其它springboot微服务默认的扫描包路径不一致，无法被扫描到，因此无法生效。
基于SpringBoot的自动装配原理，我们要将其添加到`resources`目录下的`META-INF/spring.factories`文件中：
![[Pasted image 20240126144115.png]]


此时项目启动，hm-gateway模块报错，说找不到WebMvcConfigurer
注意hm-gateway前面过滤请求以及解析token需要JwtTool和AuthProperties，因此hm-gateway引入了hm-common模块，由于springcloud-gateway底层用的是webflux响应式编程，与传统的MVC（Servlet）是冲突的，==因此一定要对这里的MvcConfig加上@ConditionalOnClass(DispatcherServlet.class) 注解==