考虑如下场景：
![[Pasted image 20240129153229.png]]
传统的开发方式是在Controller中注入多种Service，组合调用多种Service来实现请求操作，这种方式不符合设计模式的开闭原则
- 对修改关闭
- 对扩展开放
每次新增业务，都要在Controller中修改业务代码
# 事件驱动开发
可以通过事件驱动开发来优化上述逻辑，将各个Service视为事件监听者，我们只需配置一个发布者Publisher即可
- **事件发布**：通过ApplicationEventPublisherAware给我们提供，或者注入ApplicationEventMulticaster
- **事件监听**：@Component+实现ApplicationListener接口，或者@Component + @EventListener
![[Pasted image 20240129153450.png]]
# EventPublisher

```JAVA
package org.lyflexi.debug_springboot.eventdriver.publisher;  
  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationEvent;  
import org.springframework.context.ApplicationEventPublisher;  
import org.springframework.context.ApplicationEventPublisherAware;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 18:49  
 */  
@Service  
public class EventPublisher implements ApplicationEventPublisherAware {  
  
    /**  
     * 底层发送事件用的组件，SpringBoot会通过ApplicationEventPublisherAware接口自动注入给我们  
     * 事件是广播出去的。所有监听这个事件的监听器都可以收到  
     */  
    ApplicationEventPublisher applicationEventPublisher;  
  
    /**  
     * 所有事件都可以发  
     * @param event  
     */  
    public void sendEvent(ApplicationEvent event) {  
        //调用底层API发送事件  
        applicationEventPublisher.publishEvent(event);  
    }  
  
    /**  
     * 会被自动调用，把真正发事件的底层组组件给我们注入进来  
     * @param applicationEventPublisher event publisher to be used by this object  
     */    @Override  
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {  
        this.applicationEventPublisher = applicationEventPublisher;  
    }  
}
```
# LoginSuccessEvent
```JAVA
package org.lyflexi.debug_springboot.eventdriver;  
  
import org.springframework.context.ApplicationEvent;  
  
/**  
 * @author lfy  
 * @Description  登录成功事件。所有事件都推荐继承 ApplicationEvent  
 * @create 2023-04-24 18:51  
 */public class LoginSuccessEvent  extends ApplicationEvent {  
  
    /**  
     *     * @param source  代表是谁登录成了  
     */  
    public LoginSuccessEvent(UserEntity source) {  
        super(source);  
    }  
}

package org.lyflexi.debug_springboot.eventdriver;  
  
import lombok.AllArgsConstructor;  
import lombok.Data;  
import lombok.NoArgsConstructor;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 18:52  
 */@AllArgsConstructor  
@NoArgsConstructor  
@Data  
public class UserEntity {  
  
    private String username;  
    private String passwd;  
}
```


# subscriber
## AccountService
```JAVA
package org.lyflexi.debug_springboot.eventdriver.subscriber;  
  
  
import org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent;  
import org.lyflexi.debug_springboot.eventdriver.UserEntity;  
import org.springframework.context.ApplicationListener;  
import org.springframework.core.annotation.Order;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 18:43  
 */@Order(2)  
@Service  
public class AccountService implements ApplicationListener<LoginSuccessEvent> {  
  
  
  
  
    public void addAccountScore(String username){  
        System.out.println(username +" 加了1分");  
    }  
  
    @Override  
    public void onApplicationEvent(LoginSuccessEvent event) {  
        System.out.println("=====  AccountService  收到事件 =====");  
  
        UserEntity source = (UserEntity) event.getSource();  
        addAccountScore(source.getUsername());  
    }  
}
```
## CouponService
```JAVA
package org.lyflexi.debug_springboot.eventdriver.subscriber;  
  
  
import org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent;  
import org.lyflexi.debug_springboot.eventdriver.UserEntity;  
import org.springframework.context.event.EventListener;  
import org.springframework.core.annotation.Order;  
import org.springframework.scheduling.annotation.Async;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 18:44  
 */  
@Service  
public class CouponService {  
    public CouponService(){  
        System.out.println("构造器调用");  
    }  
  
    @Async  
    @Order(1)  
    @EventListener  
    public void onEvent(LoginSuccessEvent loginSuccessEvent){  
        System.out.println("===== CouponService ====感知到事件"+loginSuccessEvent);  
        UserEntity source = (UserEntity) loginSuccessEvent.getSource();  
        sendCoupon(source.getUsername());  
    }  
  
    public void sendCoupon(String username){  
        System.out.println(username + " 随机得到了一张优惠券");  
    }  
}
```

## HahaService
```JAVA
package org.lyflexi.debug_springboot.eventdriver.subscriber;  
  
import org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent;  
import org.springframework.context.event.EventListener;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 19:04  
 */@Service  
public class HahaService {  
  
    @EventListener  
    public void onEvent(LoginSuccessEvent event){  
        System.out.println("=== HahaService === 感知到事件"+event);  
        //调用业务  
    }  
}
```
## SysService
```JAVA
package org.lyflexi.debug_springboot.eventdriver.subscriber;  
  
  
import org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent;  
import org.lyflexi.debug_springboot.eventdriver.UserEntity;  
import org.springframework.context.event.EventListener;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 18:44  
 */@Service  
public class SysService {  
  
    @EventListener  
    public void haha(LoginSuccessEvent event){  
        System.out.println("==== SysService ===感知到事件"+event);  
        UserEntity source = (UserEntity) event.getSource();  
        recordLog(source.getUsername());  
    }  
  
    public void recordLog(String username){  
        System.out.println(username + "登录信息已被记录");  
    }  
}
```

# 简化Controller操作
```java
package org.lyflexi.debug_springboot.eventdriver;  
  
  
import org.lyflexi.debug_springboot.eventdriver.publisher.EventPublisher;  
import org.lyflexi.debug_springboot.eventdriver.subscriber.AccountService;  
import org.lyflexi.debug_springboot.eventdriver.subscriber.CouponService;  
import org.lyflexi.debug_springboot.eventdriver.subscriber.SysService;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationEventPublisher;  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.RequestParam;  
import org.springframework.web.bind.annotation.RestController;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-24 18:41  
 */@RestController  
public class LoginController {  
  
    @Autowired  
    AccountService accountService;  
  
    @Autowired  
    CouponService couponService;  
  
    @Autowired  
    SysService sysService;  
  
    @Autowired  
    EventPublisher eventPublisher;  
  
  
  
    /**  
     * 增加业务  
     * @param username  
     * @param passwd  
     * @return  
     */  
    @GetMapping("/login")  
    public String login(@RequestParam("username") String username,  
                        @RequestParam("passwd")String passwd){  
  
  
  
        //1、创建事件信息  
        LoginSuccessEvent event = new LoginSuccessEvent(new UserEntity(username, passwd));  
        //2、发送事件  
        eventPublisher.sendEvent(event);  
  
  
        //1、账户服务自动签到加积分  
//        accountService.addAccountScore(username);  
//        //2、优惠服务随机发放优惠券  
//        couponService.sendCoupon(username);  
//        //3、系统服务登记用户登录的信息  
//        sysService.recordLog(username);  
  
        //业务处理  
        System.out.println("登录业务处理完成....");  
        //设计模式：对新增开放，对修改关闭  
        //xxx  
        //xxx        //xxx        return username+"登录成功";  
    }  
}
```
# 请求测试

浏览器访问请求：http://localhost:8080/login?username=LY&passwd=123
浏览器返回结果如下：
```SHELL
LY登录成功
```
IDEA控制它台打印如下：
```shell
2024-01-29T15:40:34.178+08:00  INFO 14384 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2024-01-29T15:40:34.178+08:00  INFO 14384 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2024-01-29T15:40:34.179+08:00  INFO 14384 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
===== CouponService ====感知到事件org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent[source=UserEntity(username=LY, passwd=123)]
LY 随机得到了一张优惠券
=====  AccountService  收到事件 =====
LY 加了1分
=== HahaService === 感知到事件org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent[source=UserEntity(username=LY, passwd=123)]
==== SysService ===感知到事件org.lyflexi.debug_springboot.eventdriver.LoginSuccessEvent[source=UserEntity(username=LY, passwd=123)]
LY登录信息已被记录
登录业务处理完成....
```