按下F6快捷键让程序继续往下运行，直至运行到下面这行代码处。
![[Pasted image 20240105125858.png]]

于是，我们按下F5快捷键进入到以上onRefresh方法里面去看一看，如下图所示，发现它里面是空的。
```java
/**  
 * Template method which can be overridden to add context-specific refresh work. * Called on initialization of special beans, before instantiation of singletons. * <p>This implementation is empty.  
 * @throws BeansException in case of errors  
 * @see #refresh()  
 */protected void onRefresh() throws BeansException {  
    // For subclasses: do nothing by default.  
}
```

你是不是觉得很熟悉，因为我们之前就见到过两次类似这样的空方法，一次是我们在做容器刷新前的预处理工作时，可以让子类自定义个性化的属性设置，另一次是在BeanFactory创建并预处理完成以后，可以让子类做进一步的设置。我的朋友，你现在记起来了吗？😂

同理，以上onRefresh方法就是留给子类来重写的，这样是为了给我们留下一定的弹性，当子类ApplicationContext（也可以说是子容器）重写该方法后，在容器刷新的时候就可以再自定义一些逻辑了，比如给容器中多注册一些组件之类的。



// Initialize other special beans in specific context subclasses.
剧透一下，这里在未来的springboot篇，就是在这个时机，由AbstractApplicationContext的子容器ServletWebServerApplicationContext创建了tomcat环境