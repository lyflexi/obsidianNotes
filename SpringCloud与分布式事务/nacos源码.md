服务拆分为多个微服务之后，各个微服务各自维护配置文件，导致了配置文件重复 

解决方案：nacos

Nacos统一配置管理，解决服务拆分两大痛点
- 统一配置文件管理，多个微服务之间无需重复配置某些参数，共享配置
- 配置热更新支持，无需重启服务即可生效，降低运维成本

在需要开启nacos配置中心的模块中引入依赖：
```XML
  <!--nacos统一配置管理-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  </dependency>
  <!--读取bootstrap文件-->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bootstrap</artifactId>
  </dependency>
```


动态路由配置：我们无法利用上面普通的配置热更新来实现路由更新，因为网关的路由配置全部是在项目启动时由org.springframework.cloud.gateway.route.CompositeRouteDefinitionLocator在项目启动的时候加载，并且一经加载就会缓存到内存中的路由表内（一个Map），不会改变。因此要想实现动态路由配置，我们必须监听Nacos的配置变更，然后手动把最新的路由更新到路由表中。这里有两个难点：
- 如何监听Nacos配置变更？在Nacos官网中给出了手动监听Nacos配置变更的SDK：https://nacos.io/zh-cn/docs/sdk.html
- 如何把路由信息更新到路由表？更新路由要用到`org.springframework.cloud.gateway.route.RouteDefinitionWriter`这个接口：


