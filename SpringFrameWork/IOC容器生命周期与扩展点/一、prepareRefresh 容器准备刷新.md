我们先来看一下如下的一个单元测试类（例如IOCTest_Ext）。

```Java
package com.meimeixia.test;

import org.junit.Test;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import com.meimeixia.ext.ExtConfig;

public class IOCTest_Ext {
        
        @Test
        public void test01() {
                AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ExtConfig.class);
                
                // 发布一个事件
                applicationContext.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
                });
                
                // 关闭容器
                applicationContext.close();
        }

}
```

我们知道如下这样一行代码是来new一个IOC容器的，而且还可以看到传入了一个配置类。

```Java
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ExtConfig.class);
```

我们不妨点进去AnnotationConfigApplicationContext类的有参构造方法里面去看一看，我们将核心的关注点放在refresh方法上，也即刷新容器
![[Pasted image 20231226175401.png]]

接下来，我们在刷新容器的方法上打上一个断点，如下图所示，重点分析一下刷新容器这个方法里面到底做了些什么事。
映入眼帘的是一个线程安全的锁机制，
除此之外，你还能看到第一个方法，即prepareRefresh方法，顾名思义，它是来执行刷新容器前的预处理工作的。
![[Pasted image 20231227165029.png]]

# prepareRefresh刷新容器前的预处理工作
![[Pasted image 20231227165237.png]]
## initPropertySources
initPropertySources() ：这是个空方法，没有做任何事情。交由子类自定义个性化的属性设置
例如，我们可以自己来写一个AnnotationConfigApplicationContext的子类，在容器刷新的时候，重写这个方法，这样，子类（也叫子容器）的该方法就会生效。这就是框架给我们预留的扩展点。==有点类似于模板设计模式==
![[Pasted image 20231227165359.png]]

这个方法只有在子类自定义的时候有用，只不过现在它还是空的，里面啥也没做。

## validateRequiredProperties()
看注释，翻译过来就是校验属性的合法性
```java
// Validate that all properties marked as required are resolvable:  
// see ConfigurablePropertyResolver#setRequiredProperties  
getEnvironment().validateRequiredProperties();
```
首先进入通过getEnvironment()获取环境变量，然后调用validateRequiredProperties()校验属性的合法性

只不过，我们现在没有自定义什么属性，所以，此时并没有做任何属性校验工作。
## this.earlyApplicationEvents = new LinkedHashSet<>();
看注释，允许收集早期的事件信息ApplicationEvents
```java
// Allow for the collection of early ApplicationEvents,  
// to be published once the multicaster is available...  
this.earlyApplicationEvents = new LinkedHashSet<>();
```
`private Set<ApplicationEvent> earlyApplicationEvents`用来保存容器中早期的事件，
如果有事件发生，那么就存放在这个LinkedHashSet里面，这样，当事件派发器好了以后，直接用事件派发器把这些事件都派发出去。


至此，我们就分析完了prepareRefresh方法，以上就是该方法所做的事情。我们发现这个方法和BeanFactory并没有太大关系，因此，接下来我们还得来看下一个方法，即obtainFreshBeanFactory方法。
