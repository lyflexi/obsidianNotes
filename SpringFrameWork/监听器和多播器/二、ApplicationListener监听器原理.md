onApplicationEvent是监听器的回调方法，用户只需实现该回调方法，监听器就能够生效。

```java

@FunctionalInterface  
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {  
  
	void onApplicationEvent(E event);  
  
	default boolean supportsAsyncExecution() {  
       return true;  
    }  
  
  
	static <T> ApplicationListener<PayloadApplicationEvent<T>> forPayload(Consumer<T> consumer) {  
       return event -> consumer.accept(event.getPayload());  
    }  
  
}
```
对于回调机制而言，onApplicationEvent必然被Spring传入到了框架底层才得以触发，下面就来分析onApplicationEvent究竟是在哪里被Spring执行了的

还是在MyApplicationListener的onApplicationEvent处打上断点
![[Pasted image 20240106215742.png]]
打开调试堆栈信息面板，发现在AbstractApplicationContext的第451行，publishEvent方法内执行了
this.applicationEventMulticaster.multicastEvent(applicationEvent, eventType);
通过this.applicationEventMulticaster拿到事件多播器，并且调用==multicastEvent方法把该事件事件广播给多个监听器==
![[Pasted image 20240106215914.png]]

# multicastEvent

来跟一下`multicastEvent`方法:
在for循环中，遍历出所有的ApplicationListener，最终执行invokeListener方法
```java
@Override  
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {  
    ResolvableType type = (eventType != null ? eventType : ResolvableType.forInstance(event));  
    Executor executor = getTaskExecutor();  
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {  
       if (executor != null && listener.supportsAsyncExecution()) {  
          executor.execute(() -> invokeListener(listener, event));  
       }  
       else {  
          invokeListener(listener, event);  
       }  
    }  
}
```
## invokeListener

我们继续跟进代码，invokeListener方法里面调用了一个叫doInvokeListener的方法。
```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {  
    ErrorHandler errorHandler = getErrorHandler();  
    if (errorHandler != null) {  
       try {  
          doInvokeListener(listener, event);  
       }  
       catch (Throwable err) {  
          errorHandler.handleError(err);  
       }  
    }  
    else {  
       doInvokeListener(listener, event);  
    }  
}
```

### doInvokeListener
最终在doInvokeListener方法中，触发了我们外部的回调函数onApplicationEvent
![[Pasted image 20240106220545.png]]
