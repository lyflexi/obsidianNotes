servlet3.0时期，servlet为我们提供了异步servlet支持。
在过去同步servlet时期，每当一个新的请求到来，都要占用一个servlet处理线程从开始到处理结束。
即使servlet维护了一个线程池，但是过多的消耗线程池中的线程，也是十分浪费资源的
# 同步servlet
同步servlet:
```java
package lyflexi.servlet_annotation;  
  
import jakarta.servlet.ServletException;  
import jakarta.servlet.annotation.WebServlet;  
import jakarta.servlet.http.HttpServlet;  
import jakarta.servlet.http.HttpServletRequest;  
import jakarta.servlet.http.HttpServletResponse;  
import java.io.IOException;  
  
@WebServlet("/hello")  
public class HelloServlet extends HttpServlet {  
      
    @Override  
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {  
       // TODO Auto-generated method stub  
       //super.doGet(req, resp);  
       System.out.println(Thread.currentThread()+" start...");  
       try {  
          sayHello();  
       } catch (Exception e) {  
          e.printStackTrace();  
       }  
       resp.getWriter().write("hello...");  
       System.out.println(Thread.currentThread()+" end...");  
    }  
      
    public void sayHello() throws Exception{  
       System.out.println(Thread.currentThread()+" processing...");  
       Thread.sleep(3000);  
    }  
}
```
# 异步servlet
异步servlet，提供了异步支持类AsyncContext
![[servlet3.0支持异步请求，springmvc做了封装.png]]
```java
package lyflexi.servlet_annotation;  
  
import jakarta.servlet.AsyncContext;  
import jakarta.servlet.ServletException;  
import jakarta.servlet.ServletResponse;  
import jakarta.servlet.annotation.WebServlet;  
import jakarta.servlet.http.HttpServlet;  
import jakarta.servlet.http.HttpServletRequest;  
import jakarta.servlet.http.HttpServletResponse;  
import java.io.IOException;  
  
@WebServlet(value = "/async", asyncSupported = true)  
public class HelloAsyncServlet extends HttpServlet {  
  
    @Override  
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {  
        //1、支持异步处理asyncSupported=true  
        //2、开启异步模式  
        System.out.println("主线程开始。。。" + Thread.currentThread() + "==>" + System.currentTimeMillis());  
        AsyncContext startAsync = req.startAsync();  
  
        //3、业务逻辑进行异步处理;开始异步处理  
        startAsync.start(new Runnable() {  
            @Override  
            public void run() {  
                try {  
                    System.out.println("副线程开始。。。" + Thread.currentThread() + "==>" + System.currentTimeMillis());  
                    sayHello();  
                    startAsync.complete();  
//                    //获取到异步上下文,这里不好使？  
//                    AsyncContext asyncContext = req.getAsyncContext();  
//                    //4、获取响应  
//                    ServletResponse response = asyncContext.getResponse();  
                    ServletResponse response = startAsync.getResponse();  
                    response.getWriter().write("hello async...");  
                    System.out.println("副线程结束。。。" + Thread.currentThread() + "==>" + System.currentTimeMillis());  
                } catch (Exception e) {  
                }  
            }  
        });  
        System.out.println("主线程结束。。。" + Thread.currentThread() + "==>" + System.currentTimeMillis());  
    }  
  
    public void sayHello() throws Exception {  
        System.out.println(Thread.currentThread() + " processing...");  
        Thread.sleep(1000);//1s  
    }  
}
```
