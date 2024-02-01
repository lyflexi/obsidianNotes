> CORS 全称跨域资源共享策略，Cross-origin resource sharing，跨域是在域名环境下由新的端口那个服务器限制的，不是浏览器限制的。

现在88端口是我们的分布式网关服务端口，目前作为流量的统一入口，被Nginx配置为上游服务器地址。登录输入账号密码验证码之后报错：出现了跨域的问题，就是说前端Vue项目是8001端口，却要跳转到后端项目的88端口，出现跨域问题Access-Control-Allow-Origin
![[Pasted image 20240127164144.png]]

```Shell
:8001/#/login:1 Access to XMLHttpRequest at 'http://localhost:88/api/sys/login' from origin 'http://localhost:8001' has been blocked by CORS policy: 
Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

一个合法的跨域请求的实现是通过预检请求先发送一个预检请求preflight request探路，收到响应允许跨域后再发送真实请求

```plain
跨域请求流程：
非简单请求(PUT、DELETE)等，需要先发送预检请求


       -----1、预检请求、OPTIONS ------>
       <----2、服务器响应允许跨域 ------
浏览器 |                               |  服务器
       -----3、正式发送真实请求 -------->
       <----4、响应数据   --------------
```

# 同源策略

问题描述：已拦截跨源请求：同源策略禁止8001端口页面读取位于 [http://localhost:88/api/sys/login](http://localhost:88/api/sys/login) 的远程资源。

问题原因：Access-Control-Allow-Origin'

问题分析：这是一种跨域问题。访问的域名或端口和原来请求的域名端口一旦不同，请求就会被限制
- 跨域：指的是浏览器不能执行其他网站的脚本。它是由浏览器的同源策略造成的，是浏览器对js施加的安全限制
- 同源策略：是指协议、域名、端囗都要相同，其中有一个不同都会产生跨域；


|   |   |   |
|---|---|---|
|URL|说明|是否允许通信|
|[http://www.a.com/a.js](http://www.a.com/a.js)  <br>[http://www.a.com/b.js](http://www.a.com/b.js)|同一域名下|允许|
|[http://www.a.com/lab/a.js](http://www.a.com/lab/a.js)  <br>[http://www.a.com/script/b.js](http://www.a.com/script/b.js)|同一域名下不同文件夹|允许|
|[http://www.a.com:8000/a.js](http://www.a.com:8000/a.js)  <br>[http://www.a.com/b.js](http://www.a.com/b.js)|同一域名，不同端口|不允许|
|[http://www.a.com/a.js](http://www.a.com/a.js)  <br>[https://www.a.com/b.js](https://www.a.com/b.js)|同一域名，不同协议|不允许|
|[http://www.a.com/a.js](http://www.a.com/a.js)  <br>[http://70.32.92.74/b.js](http://70.32.92.74/b.js)|域名和域名对应ip|不允许|
|[http://www.a.com/a.js](http://www.a.com/a.js)  <br>[http://script.a.com/b.js](http://script.a.com/b.js)|主域相同，子域不同|不允许|
|[http://www.a.com/a.js](http://www.a.com/a.js)  <br>[http://a.com/b.js](http://a.com/b.js)|同一域名，不同二级域名（同上）|不允许（cookie这种情况下也不允许访问）|
|[http://www.cnblogs.com/a.js](http://www.cnblogs.com/a.js)  <br>[http://www.a.com/b.js](http://www.a.com/b.js)|不同域名|不允许|

# 响应头字段

设置前端admin和后端gateway允许跨域，添加响应头字段：参考：[https://blog.csdn.net/qq_38128179/article/details/84956552](https://blog.csdn.net/qq_38128179/article/details/84956552)

- Access-Control-Allow-Origin ： 支持哪些来源的请求跨域
    
- Access-Control-Allow-Method ： 支持那些方法跨域
    
- Access-Control-Allow-Credentials ：跨域请求默认不包含cookie，设置为true可以包含cookie
    
- Access-Control-Expose-Headers ： 跨域请求暴露的字段
    
    - CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段： Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma 如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。
        
- Access-Control-Max-Age ：表明该响应的有效时间为多少秒。在有效时间内，浏览器无须为同一请求再次发起预检请求。请注意，浏览器自身维护了一个最大有效时间，如果该首部字段的值超过了最大有效时间，将失效
    

设置响应头是为了告诉预检请求能跨域，有两种设置方式

- 方法1：让后端服务器告诉预检请求能跨域
    
- 方法2：让nginx服务器告诉预检请求能跨域
    

# 跨域的后端解决方案

编写Java配置，在服务端配置允许跨域

在网关中定义“`GulimallCorsConfiguration`”类，该类用来做过滤，允许所有的请求跨域。

```Java
package com.atguigu.gulimall.gateway.config;

@Configuration // gateway
public class GulimallCorsConfiguration {

    @Bean // 添加过滤器
    public CorsWebFilter corsWebFilter(){
        // 基于url跨域，选择reactive包下的
        UrlBasedCorsConfigurationSource source=new UrlBasedCorsConfigurationSource();
        // 跨域配置信息
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        // 允许跨域的头
        corsConfiguration.addAllowedHeader("*");
        // 允许跨域的请求方式
        corsConfiguration.addAllowedMethod("*");
        // 允许跨域的请求来源
        corsConfiguration.addAllowedOrigin("*");
        // 是否允许携带cookie跨域
        corsConfiguration.setAllowCredentials(true);
        
       // 任意url都要进行跨域配置
        source.registerCorsConfiguration("/**",corsConfiguration);
        return new CorsWebFilter(source);
    }
}
```

再次访问：[http://localhost:8001/#/login](http://localhost:8001/#/login)通过
![[Pasted image 20240127164254.png]]

# 跨域的前端解决方案

将以上配置信息添加到nginx配置文件当中，效果与后端配置`GulimallCorsConfiguration`等价