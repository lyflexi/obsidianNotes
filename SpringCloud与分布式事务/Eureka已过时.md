Eureka是SpringCloud中的一个负责服务注册与发现的组件。Eureka中分为Server和Client：
- Server是服务的注册与发现中心，Server端无需向注册中心注册，因为Server本身就是注册中心
- Client既可以作为服务的生产者，又可以作为服务的消费者。生产者Client和消费者Client都必须向注册中心注册

Eureka结构如下图：
![[Pasted image 20240127124436.png]]

# 生产者与消费者流程分析

生产者与消费者执行流程说明：
- 服务生产者集成在Eureka客户端，启动时，生产者发送注册请求到Eureka Server。==同时生产者有心跳机制==
    - 生产者每30秒发送一次续约请求到Eureka Server。心跳间隔默认30s：`eureka.instance.lease-renewal-interval-in-seconds=5`
    - 若Eureka Server超过90秒未收到生产者的续约请求，则Eureka服务端会将其从服务列表中踢除；心跳超时时间默认90s：`eureka.instance.lease-expiration-duration-in-seconds=10`
- 服务消费者也是集成在Eureka客户端，==启动时消费者会周期性的从服务中心获取对端的服务清单信息缓存到本地==
    - 是否从Eureka Server获取服务实例清单：`fetch-registry: true`
        - 服务消费者中，消费者每隔30秒，会从Eureka Server获取服务实例列表，其中，首次是全量查询，后续为增量查询；获取到服务实例列表后，消费者会将服务实例写入到本地缓存中；从Eureka Server获得服务清单的间隔时间是：`registry-fetch-interval-seconds: 15`
    - 服务消费者调用服务提供者时，Eureka会从本地缓存获取对应的实例信息，并进行远程接口调用restTemplate。
    - 别忘了消费者也要向Eureka Server更新自己实例信息，更新间隔时间为：`instance-info-replication-interval-seconds: 15`（因为消费者有一天也可能作为生产者，所以要也及时更新自己的信息）

# Eureka的自我保护机制
通常情况下，如果 Eureka Server 在一定的 90s 内没有接收到某个微服务实例的心跳，会注销该实例。

自我保护机制主要体现在Eureka Client和Eureka Server之间存在网络分区的情况下发挥保护作用。生产环境下服务之间通常都是跨网络调用，网络通信往往会面临着各种问题，虽然微服务状态正常，但是网络分区发生故障， Eureka Client和Eureka Server无法进行通信，此时Eureka Client无法向Eureka Server发起注册和续约请求，一段时间后Eureka Server注册表中的服务实例租约出现大量过期，Eureka Client就可能面临被Eureka Server剔除的危险。

然而此时的Eureka Client可能是处于健康状态的（可接受服务访问），固定时间内大量实例被注销，可能会严重威胁整个微服务架构的可用性。为了解决这个问题，Eureka 开发了自我保护机制，自我保护机制的触发机制如下：
- 当个别客户端出现心跳失联时，则认为是客户端的问题，剔除掉客户端；
- Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否超过 85%，如果超过 85%，则认为可能是网络问题，Eureka Server 即会进入自我保护机制。

Eureka Server 触发自我保护机制后，页面会出现提示：
```shell
EMERGENCYI EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY’RE NOT.
RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.
```
当Eureka在处于我保护状态下：
- Eureka Server 不再从注册列表中移除因为长时间没收到心跳而应该过期的服务。
    - 但是如果在保护期内刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，即会调用失败。对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。
- Eureka Server 仍然能够接受新服务的注册和查询请求，但是不会被同步到其它Server 节点上(即保证当前节点依然可用)
- 当网络稳定时，当前实例新的注册信息会被同步到其它节点中（保证Server 节点最终一致性）

通过在 Eureka Server 配置如下参数，开启或者关闭保护机制，生产环境建议打开：
```shell
eureka.server.enable-self-preservation=true
```

# Eureka实战演练

## 服务中心Server:8761

添加`spring-cloud-starter-netflix-eureka-server`依赖

```XML
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.0.6.RELEASE</version>
  <relativePath/> <!-- lookup parent from repository -->
</parent>
<properties>
  <java.version>1.8</java.version>
  <spring-cloud.version>Finchley.SR2</spring-cloud.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <exclusions>
      <exclusion>
        <groupId>org.junit.vintage</groupId>
        <artifactId>junit-vintage-engine</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
</dependencies>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

在SpringBoot启动类上添加注解`@EnableEurekaServer`

```Java
@SpringBootApplication
//开启服务治理功能
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

集成Security框架，实现Eureka密码登录

```Java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
        http.authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .httpBasic();
    }
}
```

配置文件：

```YAML
spring:
  application:
# 服务名
    name: eureka-server
# security安全认证配置
  security:
    user:
      name: yangle
      password: 123
server:
  port: 8761
eureka:
  client:
#  该应用为注册中心，不需要向注册中心注册自己
    register-with-eureka: false
#    关闭检索服务的功能，只需要维护服务
    fetch-registry: false
```

项目启动后，访问http://localhost:8761，输入用户名和密码 yangle 123

目前我们还没有创建服务，因此下图Application栏目没有服务实例
![[Pasted image 20240127130043.png]]

## Client生产者:8081-order-service

添加`spring-cloud-starter-netflix-eureka-client`依赖：

```XML
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.SR2</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

然后在启动类中添加注解`@EnableEurekaClient`

```Java
@SpringBootApplication
//开启服务发现功能
@EnableDiscoveryClient
public class EurekaClientProducterApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClientProducterApplication.class, args);
    }

}
```

添加一个订单服务

```Python

@RestController
public class OrderService {
    @RequestMapping("getOrder")
    public String getOrder(){
        return "{code:0,data:{}}";
    }
}
```

在配置文件中添加配置信息：

```Properties
spring:
  application:
    name: eureka-client-order-service
server:
  port: 8081
eureka:
  client:
    serviceUrl:
#    指定注册中心
      defaultZone: http://yangle:123@localhost:8761/eureka
  instance:
#可选
    preferIpAddress: true
#实例ID支持自定义，可选
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

我们重新打开http://localhost:8761，会看到生产者已经注册到注册中心里面了
![[Pasted image 20240127130055.png]]

## Client消费者:8082-consumer-service

将restTemplate注入到容器中，SpringRestTemplate是Spring提供的用于访问Rest服务的客端，RestTemplate提供了多种便捷访问远程Http服务的方法，能够大大提高客户端的编写效率，所以很多客户端比如Android或者第三方服务商都是使用RestTemplate做远程服务调用

```Java
@SpringBootApplication
public class EurekaClientConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaClientConsumerApplication.class, args);
    }

    @Bean(name = "restTemplate")
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    @Bean(name = "restTemplate2")
    @LoadBalanced//开启负载均衡，需要使用IDEA复制多份消费者
    public RestTemplate getRestTemplate2() {
        return new RestTemplate();
    }
}
```

添加一个接口去调用生产者提供的服务

```Java
@RestController
public class OrderConsumerService {
    @Autowired
    @Qualifier("restTemplate")
    private RestTemplate restTemplate;
    @Autowired
    @Qualifier("restTemplate2")
    private RestTemplate restTemplate2;
    @RequestMapping("getOrder")
    public String getOrder(){
        return restTemplate.getForObject("http://localhost:8081/getOrder",String.class);
    }
    //远程调用实现了负载均衡的服务，不需要指定服务端口，只需要指定使用的服务名称
    @RequestMapping("getOrderForLoadBalence")
    public String getOrderForLoadBalence(){
        return restTemplate2.getForObject("http://eureka-client-order-service/getOrder",String.class);
    }
}
```

配置文件中添加配置信息，将自己也注册进服务注册中心：

```YAML
spring:
  application:
    name: eureka-client-order-consumer-service
server:
  port: 8082
eureka:
  client:
    serviceUrl:
      defaultZone: http://yangle:123@localhost:8761/eureka
  instance:
    preferIpAddress: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

查看eureka界面http://localhost:8761/，看到消费者也注册进来了
![[Pasted image 20240127130105.png]]

# 高可用服务中心
集群中的Srever分片会以异步的方式互相复制各自的状态；

复制eureka-server项目，更名为eureka-server-slave，eureka-server-slave的配置文件如下，将端口从8761改为8762，同时defaultZone指向8761

```YAML
spring:
  application:
    name: eureka-server-slave
  security:
    user:
      name: yangle
      password: 123
server:
  port: 8762
eureka:
  client:
    serviceUrl:
      defaultZone: http://yangle:123@localhost:8761/eureka
```

修改eureka-server的配置文件，将defaultZone指向8762

```YAML
spring:
  application:
    name: eureka-server
  security:
    user:
      name: yangle
      password: 123
server:
  port: 8761
eureka:
  client:
    serviceUrl:
      defaultZone: http://yangle:123@localhost:8762/eureka
```

修改producter的配置文件，向服务中心集群进行注册

```YAML
spring:
  application:
    name: eureka-client-order-service
server:
  port: 8081
eureka:
  client:
    serviceUrl:
      defaultZone: http://yangle:123@localhost:8761/eureka,http://yangle:123@localhost:8762/eureka
  instance:
    preferIpAddress: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

修改consumer的配置文件，向服务中心集群进行注册

```YAML
spring:
  application:
    name: eureka-client-order-consumer-service
server:
  port: 8082
eureka:
  client:
    serviceUrl:
      defaultZone: http://yangle:123@localhost:8761/eureka,http://yangle:123@localhost:8762/eureka
  instance:
    preferIpAddress: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```