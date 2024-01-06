ApplicationListener按照字面意思，它应该是Spring里面的应用监听器，也就是Spring为我们提供的基于事件驱动开发的功能。它的作用主要是来监听IOC容器中发布的一些事件，==只要事件发生便会来触发该监听器的回调，从而来完成事件驱动模型的开发。==

==要知道的是，监听器监听事件的时机是在IOC容器刷新完成之后：==，监听器的监听顺序如下
1.  IOC容器刷新完成事件（Spring自带事件）
2.  用户自己发布的其他事件
3.  IOC容器关闭事件（Spring自带事件）

Spring提供了两种监听方式，分别是
- ApplicationListener接口
- @Listener注解
# 监听器的使用
## ApplicationListener
我们看一下ApplicationListener的源码
- 其中`<E extends ApplicationEvent>`就是要发布的事件类型，包含很多子事件类型
    
- 其中`void onApplicationEvent(E event);`就是回调函数。

![[Pasted image 20240106200506.png]]

编写一个类来实现ApplicationListener接口，例如MyApplicationListener，这实际上就是写了一个监听器。

```Java
package com.meimeixia.ext;

import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

// 当然了，监听器这东西要工作，我们还得把它添加在容器中
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {

        // 当容器中发布此事件以后，下面这个方法就会被触发
        @Override
        public void onApplicationEvent(ApplicationEvent event) {
                // TODO Auto-generated method stub
                System.out.println("收到事件：" + event);
        }

}
```
## @Listener注解

```java
package org.lyflexi.debug_springframework.listener;  
  
import org.springframework.context.ApplicationEvent;  
import org.springframework.context.event.EventListener;  
import org.springframework.stereotype.Service;  
  
@Service  
public class MyApplicationListenerByAnnoatation {  
  
    @EventListener(classes={ApplicationEvent.class})  
    public void listen(ApplicationEvent event){  
       System.out.println("监听方式二：UserService的@EventListener注解监听到的事件："+event);  
    }  
  
}
```
## 配置类
配置类只是为了只当包扫描路径
```java
package org.lyflexi.debug_springframework.listener;  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/6 20:30  
 */@ComponentScan("org.lyflexi.debug_springframework.listener")  
//@Configuration  
public class ConfigListener {  
}
```
## 测试类

上面通过ApplicationListener接口和@EventListener注解，注册了两个监听器。此后只要容器中有相关事件发布，那么我们就能监听到这个事件：
- Spring默认会在容器刷新时和容器关闭时发布两个事件。`ContextRefreshedEvent`和`ContextClosedEvent`
    - ContextRefreshedEvent：容器刷新完成事件。即容器刷新完成（此时，所有bean都已完全创建），便会发布该事件。
    - ContextClosedEvent：容器关闭事件。即容器关闭时，便会发布该事件。
- 此外，用户还可以自定义一个事件，并调用publishEvent方法去发布。

运行以下test01/test02方法
```Java
package org.lyflexi.debug_springframework.listener;  
  
import org.junit.jupiter.api.Test;  
import org.lyflexi.debug_springframework.beanfactorylifecircle.ExtConfig;  
import org.lyflexi.debug_springframework.tx.TxConfig;  
import org.springframework.context.ApplicationEvent;  
import org.springframework.context.annotation.AnnotationConfigApplicationContext;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/6 20:18  
 */public class ListenerTest {  
      
    /*  
    * 有参构造AnnotationConfigApplicationContext(ConfigListener.class)  
    * */    @Test  
    public void test01() {  
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ConfigListener.class);  
//配置类ConfigListener添加了包扫描，因此无需再手动注册listener  
//        applicationContext.addApplicationListener(new MyApplicationListener());  
//        applicationContext.register(MyApplicationListenerByAnnoatation.class);  
//有参构造AnnotationConfigApplicationContext中已经调用了refresh()，因此不可再次手动调用refresh()，refresh()只能调用一次  
/*          public AnnotationConfigApplicationContext(Class<?>... componentClasses) {  
            this();            register(componentClasses);            refresh();        }*///        applicationContext.refresh();  
        //发布事件  
        applicationContext.publishEvent(new ApplicationEvent(new String("我发布的事件")) {  
        });  
  
        applicationContext.close();  
    }  
  
    /*  
     * 无参构造AnnotationConfigApplicationContext()  
     * */    @Test  
    public void test02() {  
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();  
  
        applicationContext.addApplicationListener(new MyApplicationListener());  
        applicationContext.register(MyApplicationListenerByAnnoatation.class);  
  
        applicationContext.refresh();  
        //发布事件  
        applicationContext.publishEvent(new ApplicationEvent(new String("我发布的事件")) {  
        });  
  
        applicationContext.close();  
    }  
}
```

看到Idea控制台打印出了如下内容：
```java
监听方式一：MyApplicationListener接口收到事件：org.springframework.context.event.ContextRefreshedEvent[source=org.springframework.context.annotation.AnnotationConfigApplicationContext@6302bbb1, started on Sat Jan 06 21:20:40 CST 2024]
监听方式二：UserService的@EventListener注解监听到的事件：org.springframework.context.event.ContextRefreshedEvent[source=org.springframework.context.annotation.AnnotationConfigApplicationContext@6302bbb1, started on Sat Jan 06 21:20:40 CST 2024]

监听方式一：MyApplicationListener接口收到事件：org.lyflexi.debug_springframework.listener.ListenerTest$2[source=我发布的事件]
监听方式二：UserService的@EventListener注解监听到的事件：org.lyflexi.debug_springframework.listener.ListenerTest$2[source=我发布的事件]

监听方式一：MyApplicationListener接口收到事件：org.springframework.context.event.ContextClosedEvent[source=org.springframework.context.annotation.AnnotationConfigApplicationContext@6302bbb1, started on Sat Jan 06 21:20:40 CST 2024]
监听方式二：UserService的@EventListener注解监听到的事件：org.springframework.context.event.ContextClosedEvent[source=org.springframework.context.annotation.AnnotationConfigApplicationContext@6302bbb1, started on Sat Jan 06 21:20:40 CST 2024]

Process finished with exit code 0
```
哎，可以看到我们收到了三个事件，这三个事件分别是

- org.springframework.context.event.ContextRefreshedEvent代表容器已经刷新完成事件
    
- 我发布的事件
    
- org.springframework.context.event.ContextClosedEvent代表容器关闭事件。

