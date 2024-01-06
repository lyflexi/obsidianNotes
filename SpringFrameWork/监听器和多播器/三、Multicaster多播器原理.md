其实，在讲Listener原理的时候，执行了这句话：this.applicationEventMulticaster.multicastEvent(applicationEvent, eventType);
==通过this.applicationEventMulticaster拿到事件多播器，并且调用multicastEvent方法把该事件事件广播给多个监听器==
但是这句话执行有两个前提：
1. 容器中已经存在多播器this.applicationEventMulticaster
2. 容器中的listener已经被添加到了this.applicationEventMulticaster中
我们需要从IOC容器的生命周期中去分析以上两个前提，是在何时准备好的
# initApplicationEventMulticaster
来看看这个refresh方法具体都做了哪些事。该方法我们已经很熟悉了，如下图所示，可以看到在该方法中会调非常多的方法，其中就有一个叫initApplicationEventMulticaster的方法，顾名思义，它就是来初始化ApplicationEventMulticaster的。
![[Pasted image 20240106221311.png]]

那么，初始化ApplicationEventMulticaster的逻辑又是怎样的呢？我们也可以来看一看，进入initApplicationEventMulticaster方法里面，如下图所示。其实很简单，它就是先判断IOC容器（也就是BeanFactory）中是否有id等于常量applicationEventMulticaster【APPLICATION_EVENT_MULTICASTER_BEAN_NAME】的组件：
- 如果IOC容器中有id等于applicationEventMulticaster的组件，那么就会通过getBean方法直接拿到这个组件；其实说白了，在整个事件派发的过程中，我们可以自定义事件多播器。
- 如果没有，那么就重新new一个SimpleApplicationEventMulticaster类型的事件多播器兜底，然后再把这个事件多播器注册到容器中，也就是说，这相当于我们自己给容器中注册了一个事件多播器，这样，以后我们就可以在其他组件要派发事件的时候，自动注入这个事件多播器就行了。
![[Pasted image 20240106221325.png]]

以上就是我们这个事件多播器它是怎么拿到的。
# registerListener

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=NjQwNjVkZDc0OTZlYjY0NzNiZmEwN2IyY2UyMmViZjZfS1k0N1hiQTloaWdJVlIyaE1iQmt6d2YxdnJqTjA0WVJfVG9rZW46VVRSZGJCN05PbzJXc254bmdzVWNpY3hUbm1mXzE3MDQ1NTAyNTM6MTcwNDU1Mzg1M19WNA)

registerListeners()会遍历所有的listener，并将listener添加到多播器中
![[Pasted image 20240106221736.png]]

addApplicationListener其实是将监听器添加到了SimpleApplicationEventMulticaster.java的内部类DefaultListenerRetriever中的成员变量applicationListeners中了
```java

public abstract class AbstractApplicationEventMulticaster  
       implements ApplicationEventMulticaster, BeanClassLoaderAware, BeanFactoryAware {

	@Override  
	public void addApplicationListener(ApplicationListener<?> listener) {  
	    synchronized (this.defaultRetriever) {  
	       // Explicitly remove target for a proxy, if registered already,  
	       // in order to avoid double invocations of the same listener.       Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);  
	       if (singletonTarget instanceof ApplicationListener) {  
	          this.defaultRetriever.applicationListeners.remove(singletonTarget);  
	       }  
	       this.defaultRetriever.applicationListeners.add(listener);  
	       this.retrieverCache.clear();  
	    }  
	}
	
	
	private class DefaultListenerRetriever {  
	  
	    public final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();  
	  
	    public final Set<String> applicationListenerBeans = new LinkedHashSet<>();  
	  
	    public Collection<ApplicationListener<?>> getApplicationListeners() {  
	       List<ApplicationListener<?>> allListeners = new ArrayList<>(  
	             this.applicationListeners.size() + this.applicationListenerBeans.size());  
	       allListeners.addAll(this.applicationListeners);  
	       if (!this.applicationListenerBeans.isEmpty()) {  
	          BeanFactory beanFactory = getBeanFactory();  
	          for (String listenerBeanName : this.applicationListenerBeans) {  
	             try {  
	                ApplicationListener<?> listener =  
	                      beanFactory.getBean(listenerBeanName, ApplicationListener.class);  
	                if (!allListeners.contains(listener)) {  
	                   allListeners.add(listener);  
	                }  
	             }  
	             catch (NoSuchBeanDefinitionException ex) {  
	                // Singleton listener instance (without backing bean definition) disappeared -  
	                // probably in the middle of the destruction phase             }  
	          }  
	       }  
	       AnnotationAwareOrderComparator.sort(allListeners);  
	       return allListeners;  
	    }  
	}
}

```

结语：
如此这般，多播器才能够执行==向各个监听器派发事件==：this.applicationEventMulticaster.multicastEvent(applicationEvent, eventType);
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
同理getApplicationListeners也是从this.defaultRetriever.applicationListeners中取所有的listener，
只不过getApplicationListeners这个方法过于复杂了，这里就不展开分析了
