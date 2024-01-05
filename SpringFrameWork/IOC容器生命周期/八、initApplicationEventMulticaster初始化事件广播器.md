来到了initApplicationEventMulticaster：顾名思义，该方法是来初始化事件派发器的。
![[Pasted image 20240105125420.png]]


按下F5快捷键进入到initApplicationEventMulticaster方法里面，如下图所示，跟initMessageSource的逻辑很像，可以看到一开始是先来获取BeanFactory的。
# 获取BeanFactory
```java
public static final String APPLICATION_EVENT_MULTICASTER_BEAN_NAME = "applicationEventMulticaster";
```
![[Pasted image 20240105125558.png]]

## 看容器中是否有id为applicationEventMulticaster，类型是ApplicationEventMulticaster的组件

如果有的话，那么会从BeanFactory中获取到id为applicationEventMulticaster，类型是ApplicationEventMulticaster的组件，并将其赋值给this.applicationEventMulticaster。这可以从下面这行代码看出。

也就是说，如果我们之前已经在容器中配置了一个事件派发器，那么此刻就能从BeanFactory中获取到该事件派发器了。

很显然，容器刚开始创建的时候，肯定是还没有的，所以程序会来到下面的else语句中。
![[Pasted image 20240105125723.png]]
若没有，则创建一个SimpleApplicationEventMulticaster类型的组件，并把创建好的组件注册在容器中

如果没有的话，那么Spring自己会创建一个SimpleApplicationEventMulticaster类型的对象，即一个简单的事件派发器。

然后，把创建好的事件派发器组件注册到容器中，即添加到BeanFactory中，所执行的是下面这行代码。

这样，我们以后其他组件要使用事件派发器，直接自动注入这个事件派发器组件即可。