正规的域名访问流程是输入www.baidu.com，然后由网络DNS解析出对应的ip地址

我们现在通过配置`C:\Windows\System32\drivers\etc\host`文件添加映射规则，也可达到同样的效果
![[Pasted image 20240127164012.png]]

Windows根据域名匹配到nginx服务器，然后通过`nginx.conf`的配置字段`location`再定位到后端product服务
![[Pasted image 20240127164023.png]]

但这样做不好，分布式项目后期会有好多微服务，来回的修改配置`location`很不方便

# 分布式网关

所以，应该将`proxy_pass`字段指定给微服务网关，在后端由微服务网关帮我们转发定位服务

Spring网关配置如下：

```YAML
# 任何以mall.com结尾的域名转发到mall-product
- id: mall_route
  uri: lb://mall-product
  predicates:
    - Host=**.gulimall.com,gulimall.com,item.gulimall.com
```

`nginx.conf`配置如下

```PowerShell
#gzip  on;
upstream gulimall{
        server 192.168.18.1:88;
}
```

step1:我们发送域名给nginx的时候是会带上当然是会带上host

step2:但是nginx再次转发给我们的微服务网关的时候，是会丢失域名信息的，

step3:我们的网关微服务这时候是以域名进行断言的，请求头丢失之后网关就无法根据host进行路由了
![[Pasted image 20240127164038.png]]

# 解决请求头丢失问题

所以在ngnix的`location`字段当中需要再次显式的指定host

```PowerShell
location / {
        proxy_pass http://gulimall;
        # 转发给上游服务器的时候带上请求头信息
        proxy_set_header Host $host;
}
```

![[Pasted image 20240127164047.png]]

根据域名访问商品服务的接口：http://gulimall.com/api/product/category/list/tree
![[Pasted image 20240127164054.png]]