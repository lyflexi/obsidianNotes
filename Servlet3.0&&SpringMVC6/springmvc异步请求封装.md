
springmvc对异步servlet做了进一步的封装，如下，
异步Springmvc提供了两种实现方式：
1. `Callable<>`
2. `DeferredResult<>`
# 方式Callable
* 1、控制器返回Callable  
* 2、Spring异步处理，将Callable 提交到 TaskExecutor 使用一个隔离的线程进行执行  
* 3、DispatcherServlet和所有的Filter退出web容器的线程，但是response 保持打开状态；  
* 4、Callable返回结果，SpringMVC将请求重新派发给容器，恢复之前的处理，唤醒DispatcherServlet，==因此每一次请求preHandle会触发两次 == 
* 5、根据Callable返回的结果。SpringMVC继续进行视图渲染流程等（从收请求-视图渲染）。 
完整的请求处理过程如下：
 一次请求：      `preHandle.../springmvc_annotation/async01`
==可以看到主线程的用的线程池是http-nio-8080-exec==
主线程开始...`Thread[http-nio-8080-exec-4,5,main]==>1565836350257`
主线程结束...`Thread[http-nio-8080-exec-4,5,main]==>1565836350258`
=========DispatcherServlet及所有的Filter退出线程...============================  

================等待Callable执行，可以看到副线程的用的线程池是MvcAsync1==========  
副线程开始...`Thread[MvcAsync1,5,main]==>1565836350269`
······       副线程开始...`Thread[MvcAsync1,5,main]==>1565836352269`
================Callable执行完成==========  

================再次收到之前重发过来的请求========  
preHandle.../springmvc-annotation/async01       
postHandle...目标方法就不用再次执行了，（因为Callable的之前的返回值就是目标方法的返回值）  
afterCompletion...  

```java
package org.lyflexi.debug_springmvc.mvcintegration.controller;  
  
import org.lyflexi.debug_springmvc.mvcintegration.queue.DeferredResultQueue;  
import org.springframework.stereotype.Controller;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.ResponseBody;  
import org.springframework.web.context.request.async.DeferredResult;  
  
import java.util.UUID;  
import java.util.concurrent.Callable;  
  
  
@Controller  
public class AsyncController {  
  
    /**  
     * 1、控制器返回Callable  
     * 2、Spring异步处理，将Callable 提交到 TaskExecutor 使用一个隔离的线程进行执行  
     * 3、DispatcherServlet和所有的Filter退出web容器的线程，但是response 保持打开状态；  
     * 4、Callable返回结果，SpringMVC将请求重新派发给容器，恢复之前的处理，唤醒DispatcherServlet，因此每一次请求preHandle会触发两次  
     * 5、根据Callable返回的结果。SpringMVC继续进行视图渲染流程等（从收请求-视图渲染）。  
  
  
     一次请求：      preHandle.../springmvc_annotation/async01  
      主线程开始...Thread[http-nio-8080-exec-4,5,main]==>1565836350257  
      主线程结束...Thread[http-nio-8080-exec-4,5,main]==>1565836350258  
       =========DispatcherServlet及所有的Filter退出线程============================  
  
       ================等待Callable执行==========  
       副线程开始...Thread[MvcAsync1,5,main]==>1565836350269  
       ······       副线程开始...Thread[MvcAsync1,5,main]==>1565836352269  
       ================Callable执行完成==========  
  
       ================再次收到之前重发过来的请求========  
       preHandle.../springmvc-annotation/async01       postHandle...目标方法就不用再次执行了，（因为Callable的之前的返回值就是目标方法的返回值）  
       afterCompletion...  
  
     可以看到，异步mvc场景下，普通的拦截器不能够拦截目标方法，因此就需要异步的拦截器（略了）  
       异步的拦截器:  
          1）、原生API的AsyncListener  
          2）、SpringMVC：实现AsyncHandlerInterceptor；  
     * @return  
     */  
    @ResponseBody  
    @RequestMapping("/async01")  
    public Callable<String> async01(){  
       System.out.println("主线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());  
  
       Callable<String> callable = ()->{  
          System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());  
          Thread.sleep(2000);  
            System.out.println("······");  
          System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());  
          return "Callable<String> async01()";  
       };  
  
       System.out.println("主线程结束..."+Thread.currentThread()+"==>"+System.currentTimeMillis());  
       return callable;  
    }  
  
}
```
打印信息如下，副线程确实执行了2000ms
```java
preHandle.../debug_springmvc_war/async01
主线程开始...Thread[http-nio-8081-exec-4,5,main]==>1704857963025
主线程结束...Thread[http-nio-8081-exec-4,5,main]==>1704857963025
副线程开始...Thread[MvcAsync1,5,main]==>1704857963033
······
副线程结束...Thread[MvcAsync1,5,main]==>1704857965039
preHandle.../debug_springmvc_war/async01
postHandle...
afterCompletion...
```
我们还可以看到，异步mvc场景下，普通的拦截器不能够很好的拦截目标方法（真正的目标方法被异步执行了），因此就需要异步的拦截器（略了）  
异步的拦截器:  
  1）、原生API的AsyncListener  
  2）、SpringMVC：实现AsyncHandlerInterceptor；  
# 方式DeferredResult
假设有如下的场景，//由于tomcat线程无法创建业务场景中的订单，但是tomcat线程可以把创建订单的消息发送给消息中间件，
- 让订单由另外一个任务去创建
- 等订单创建好之后，再唤醒主线程
因此我们有必要维护一个消息队列DeferredResultQueue
```java
package org.lyflexi.debug_springmvc.mvcintegration.queue;  
  
import org.springframework.web.context.request.async.DeferredResult;  
  
import java.util.Queue;  
import java.util.concurrent.ConcurrentLinkedQueue;  
  
public class DeferredResultQueue {  
      
    private static Queue<DeferredResult<Object>> queue = new ConcurrentLinkedQueue<DeferredResult<Object>>();  
      
    public static void save(DeferredResult<Object> deferredResult){  
       queue.add(deferredResult);  
    }  
      
    public static DeferredResult<Object> get( ){  
       return queue.poll();  
    }  
  
}
```
![[Pasted image 20240110114058.png]]
DeferredResult的异步示例如下：
- 先发起/createOrder请求，如果等待三秒后订单还没创建成功，则/createOrder请求返回fail
- 先发起/createOrder请求，在三秒内再发起先发起/create请求并创建成功订单，回设deferredResult值唤醒主任务/createOrder请求
```java
package org.lyflexi.debug_springmvc.mvcintegration.controller;  
  
import org.lyflexi.debug_springmvc.mvcintegration.queue.DeferredResultQueue;  
import org.springframework.stereotype.Controller;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.ResponseBody;  
import org.springframework.web.context.request.async.DeferredResult;  
  
import java.util.UUID;  
import java.util.concurrent.Callable;  
  
  
@Controller  
public class AsyncController {  
  
  
    @ResponseBody  
    @RequestMapping("/createOrder")  
    public DeferredResult<Object> createOrder(){  
  
       DeferredResult<Object> deferredResult = new DeferredResult<>((long)3000, "create fail...");  
       //由于tomcat线程无法创建业务场景中的订单，但是tomcat线程可以把创建订单的消息发送给消息中间件  
       DeferredResultQueue.save(deferredResult);  
       return deferredResult;  
    }  
  
  
    @ResponseBody  
    @RequestMapping("/create")  
    public String create(){  
       //创建订单  
       String order = UUID.randomUUID().toString();  
       DeferredResult<Object> deferredResult = DeferredResultQueue.get();  
       deferredResult.setResult(order);  
       return "success===>"+order;  
    }   
}
```