  

**过滤器**：用于把你传入的request、response提前过滤掉一些信息，或者提前设置一些参数，然后再传入servlet或者struts的action进行业务逻辑，比如过滤掉非法url（不是login.do的地址请求，如果用户没有登陆都过滤掉），或者在传入servlet或者 struts的action前统一设置字符集，或者去除掉一些非法字符。==过滤器Filter是基于函数回调机制去实现的==

**拦截器** ：是指的面向切面编程AOP，基于Java的动态代理机制的。就是在你的service方法前调用一个方法，或者在service方法后调用一个方法，甚至在你抛出异常的时候做业务逻辑的操作。==比如动态代理dynamicProxy就是拦截器的简单实现，在你调用方法前打印出字符串（或者做其它业务逻辑的操作），也可以在你调用方法后打印出字符串，甚至在你抛出异常的时候做业务逻辑的操作。==

首先区分Filter和Intercepter的规范：

- 过滤器是在Servlet规范中定义的，是Servlet容器支持的。因此Filter只能用于Web程序中，对Servlet请求起作用。
    
- Spring容器内的，是Spring框架支持的。因此Intercepter既可以用于Web程序中的Controller，也可以用于Application、Swing程序中。
    

# 过滤器的原理分析
## 实现Filter 接口

过滤器需要实现Filter 接口，看到Filter 接口中定义了三个方法。

- init() ：该方法在容器启动初始化过滤器时被调用，它在 Filter 的整个生命周期只会被调用一次。注意：这个方法必须执行成功，否则过滤器会不起作用。
    
- ==doFilter() ：容器中的每一次请求都会调用该方法， 该方法传入了FilterChain接口，用来调用下一个过滤器 Filter。FilterChain 就是回调接口==
    
- destroy()： 当容器销毁 过滤器实例时调用该方法，一般在方法中销毁或关闭资源，在过滤器 Filter 的整个生命周期也只会被调用一次
    

```Java
@Component
public class MyFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

        System.out.println("Filter 初始化");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

        System.out.println("Filter 处理中");
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

        System.out.println("Filter 后置");
    }
}
```

## 传入回调接口FilterChain
`FilterChain` 内部也有一个 doFilter() 方法就是回调方法。

```Java
public interface FilterChain {
    void doFilter(ServletRequest var1, ServletResponse var2) throws IOException, ServletException;
}
```

ApplicationFilterChain是FilterChain 接口的实现类， ApplicationFilterChain里面能拿到我们自定义的MyFilter1、MyFilter2...MyFilterN 类，在其内部回调方法doFilter()里调用我们自定义的MyFilter1、MyFilter2...MyFilterN 过滤器，并执行各自的doFilter() 方法。

```Java
public final class ApplicationFilterChain implements FilterChain {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response) {
            ...//省略
            internalDoFilter(request,response);
    }

    private void internalDoFilter(ServletRequest request, ServletResponse response){
    if (pos < n) {
            //获取第pos个filter            ApplicationFilterConfig filterConfig = filters[pos++];            Filter filter = filterConfig.getFilter();
            ...
            filter.doFilter(request, response, this);
        }
    }

}
```

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjYyZjYwOTFjZTFmZjVlMDk1ZjU2MjFlZDBmNmM1MDRfMDRkb3YzeXVHOTZWM3p3NUNRTWpyU0JITjgxRjlzVkxfVG9rZW46QmMzb2JPczFYb3RMaXF4Q24ySWM1U2JvbnJmXzE3MDQ3MDE5OTk6MTcwNDcwNTU5OV9WNA)

  

# 拦截器的原理分析

Java反射-->dynamicProxy-->AOP-->拦截器

## 实现HandlerInterceptor接口

编写拦截器类需要实现`HandlerInterceptor`接口，三个必须实现的方法，且方法的执行有先后顺序

1. preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) 。在请求被处理之前进行调用，是否需要将当前的请求拦截下来
    
    1. 如果返回false，请求将会终止。
        
    2. 返回true，请求将会继续`Object handler`执行目标方法实例。
        
2. ==postHandle(HttpServletRequest request, HttpServletResponse response, Object handler，ModelAndView modelAndView) 。真正的业务处理==。postHandle用于在请求被处理之后进行调用，`ModelAndView modelAndView`是指将被呈现在网页上的对象，可以通过修改这个对象实现不同角色跳向不同的网页或不同的消息提示。
    
3. afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler，Exception ex) 。在请求结束之后调用，一般用于关闭流、资源连接等
    

```Java
package org.springframework.web.servlet;  
public interface HandlerInterceptor {  
    boolean preHandle(  
            HttpServletRequest request, HttpServletResponse response,   
            Object handler)   
            throws Exception;  
  
    void postHandle(  
            HttpServletRequest request, HttpServletResponse response,   
            Object handler, ModelAndView modelAndView)   
            throws Exception;  
  
    void afterCompletion(  
            HttpServletRequest request, HttpServletResponse response,   
            Object handler, Exception ex)  
            throws Exception;  
}
```
# Filter和Intercepter的对比总结
## 触发时机比较

过滤器 和 拦截器的触发时机也不同，我们看下边这张图。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YjAyNjc2ZWU3YzQ5N2FkYjBkMzNkOTE0NGQ0ZjBmNjFfdHREMjEwS0ZGVVpOQW1zS1gxQ3ROb3p4TEpGQUhaRktfVG9rZW46T1pXWmJPN1Bkb2g4V1Z4cTUydWNZb3BvblVlXzE3MDQ3MDE5OTk6MTcwNDcwNTU5OV9WNA)

- 过滤器Filter是在请求进入Servlet容器前，或者退出Servlet容器后，对Servlet进行处理。
	- 初始化init()必须返回true，下面的doFilter方法才有效
	- dofilter()只能在容器初始化之后被调用一次。
	- 容器销毁的时候，调用destroy()
    
- 拦截器 Interceptor 是在请求穿过servlet后，在进入Controller的时候进行处理的，拦截器可以多次被调用。拦截器不仅能够深入到方法前后、而且能够深入到异常抛出前后等
    - Controller之前预处理
    - Controller 中渲染了对应的视图
    - Controller 之后结束调用，一般用于关闭流、资源连接等

## 多实例顺序控制比较

实际开发过程中，会出现多个过滤器或拦截器同时存在的情况，有时我们希望某个过滤器或拦截器能优先执行，就涉及到它们的执行顺序。

### 过滤器顺序控制

过滤器用@Order注解控制执行顺序，通过@Order控制过滤器的级别，值越小级别越高越先执行。

```Java
@Order(Ordered.HIGHEST_PRECEDENCE)
@Component
public class MyFilter2 implements Filter {
}
```

### 拦截器顺序控制

拦截器默认的执行顺序，就是它的注册顺序，也可以通过Order手动设置控制，值越小越先执行。

```Java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new MyInterceptor2()).addPathPatterns("/**").order(2);
    registry.addInterceptor(new MyInterceptor1()).addPathPatterns("/**").order(1);
    registry.addInterceptor(new MyInterceptor()).addPathPatterns("/**").order(3);
}
```

优先级高的的拦截器 preHandle() 方法先执行，但是优先级高的postHandle()和afterCompletion()方法反而会后执行。如果实际开发中严格要求执行顺序，那就需要特别注意这一点。

```Shell
Interceptor1 前置
Interceptor2 前置
Interceptor 前置
我是controller
Interceptor 处理中
Interceptor2 处理中
Interceptor1 处理中
Interceptor 后置
Interceptor2 处理后
Interceptor1 处理后
```

那为什么会这样呢？ 得到答案就只能看源码了，我们要知道controller 中所有的请求都要经过核心组件`DispatcherServlet`路由，都会执行它的 doDispatch() 方法，而拦截器postHandle()、preHandle()方法便是在其中调用的。

```Java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    
        try {
         ...........
            try {
           
                // 获取可以执行当前Handler的适配器
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

                // Process last-modified header, if supported by the handler.
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if (logger.isDebugEnabled()) {
                        logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
                    }
                    if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }
                // 注意： 执行Interceptor中PreHandle()方法
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }

                // 注意：执行Handle【包括我们的业务逻辑，当抛出异常时会被Try、catch到】
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }
                applyDefaultViewName(processedRequest, mv);

                // 注意：执行Interceptor中PostHandle 方法【抛出异常时无法执行】
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            }
        }
        ...........
    }

```

看看两个方法applyPreHandle(）、applyPostHandle(）具体是如何被调用的，发现两个方法中在调用拦截器数组 HandlerInterceptor[] 时，循环的顺序竟然是相反的。。。，导致postHandle()、preHandle() 方法执行的顺序相反

```Java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HandlerInterceptor[] interceptors = this.getInterceptors();
    if(!ObjectUtils.isEmpty(interceptors)) {
        for(int i = 0; i < interceptors.length; this.interceptorIndex = i++) {
            HandlerInterceptor interceptor = interceptors[i];
            if(!interceptor.preHandle(request, response, this.handler)) {
                this.triggerAfterCompletion(request, response, (Exception)null);
                return false;
            }
        }
    }

    return true;
}



void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv) throws Exception {
        HandlerInterceptor[] interceptors = this.getInterceptors();
        if(!ObjectUtils.isEmpty(interceptors)) {
            for(int i = interceptors.length - 1; i >= 0; --i) {
                HandlerInterceptor interceptor = interceptors[i];
                interceptor.postHandle(request, response, this.handler, mv);
            }
        }
    }
```