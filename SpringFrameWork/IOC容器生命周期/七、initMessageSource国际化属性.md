来到了方法initMessageSource()
顾名思义，该方法是来初始化MessageSource组件的。对于Spring MVC而言，该方法主要是来做国际化功能的，如消息绑定、消息解析等。
![[Pasted image 20240105114443.png]]
# initMessageSource分析
![[Pasted image 20240105114551.png]]
## 获取BeanFactory

按下F5快捷键进入到initMessageSource方法里面，如下图所示，可以看到一开始是先来获取BeanFactory的。
```java
ConfigurableListableBeanFactory beanFactory = getBeanFactory();
```
而这个BeanFactory，我们之前早就准备好了。

### 看容器中是否有id为messageSource，类型是MessageSource的组件

按下F6快捷键让程序继续往下运行，会发现有一个判断，即判断BeanFactory中是否有一个id为messageSource的组件。
![[Pasted image 20240105114814.png]]

如果有的话，那么会从BeanFactory中获取到id为messageSource，类型是MessageSource的组件，并将其赋值给this.messageSource。这可以从下面这行代码看出。
```java
ConfigurableListableBeanFactory beanFactory = getBeanFactory();  
if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {  
    this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);  
    // Make MessageSource aware of parent MessageSource.  
    if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource hms &&  
          hms.getParentMessageSource() == null) {  
       // Only set parent context as parent MessageSource if no parent MessageSource  
       // registered already.       
       hms.setParentMessageSource(getInternalParentMessageSource());  
    }  
    if (logger.isTraceEnabled()) {  
       logger.trace("Using MessageSource [" + this.messageSource + "]");  
    }  
}
```
很显然，容器刚开始创建的时候，肯定是还没有的，所以程序会来到下面的else语句中。
那么Spring自己会创建一个DelegatingMessageSource类型的组件，并把创建好的组件注册在容器中
![[Pasted image 20240105114934.png]]

那么问题来了，这种MessageSource类型的组件有啥作用呢？我们不妨查看一下MessageSource接口的源码，如下图所示，它里面定义了很多重载的getMessage方法，该方法可以从配置文件（特别是国际化配置文件）中取出某一个key所对应的值。
![[Pasted image 20240105125224.png]]也就是说，这种MessageSource类型的组件的作用一般是取出国际化配置文件中某个key所对应的值，而且还能按照区域信息获取哟~