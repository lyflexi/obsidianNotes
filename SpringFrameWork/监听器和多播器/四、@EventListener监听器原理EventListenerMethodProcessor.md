# EventListenerMethodProcessor

我们可以点进去@EventListener这个注解里面去看一看，如下图所示，可以看到这个注解上面有一大堆的描述，描述中有一个醒目的字眼，即参考EventListenerMethodProcessor。意思可能是说，如果你想搞清楚@EventListener注解的内部工作原理，那么可以参考EventListenerMethodProcessor这个类。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MTFiOWZhNzM0YjI1YThmNmE1OGM4YmI0N2M5ZTUzOTlfcWNVdzVOb2dNbFN6SEE3cWpZSUtiMFRVb0VLMVlVeWxfVG9rZW46STAxc2JtbTJVbzFpR3J4MXlpcmNEVHFpbm5jXzE3MDQ1NTIwMTk6MTcwNDU1NTYxOV9WNA)

EventListenerMethodProcessor是啥呢？它就是一个处理器，其作用是来解析方法上的@EventListener注解的。
打开EventListenerMethodProcessor的diagram，虽然EventListenerMethodProcessor实现了三个接口：
- SmartInitializingSingleton, 
- ApplicationContextAware, 
- BeanFactoryPostProcessor
但是最关键的是==SmartInitializingSingleton==
![[Pasted image 20240106225504.png]]
SmartInitializingSingleton接口的定义信息如下，==它只含一个方法afterSingletonsInstantiated== 
翻译描述信息如下：
- 该方法是在所有的单实例bean已经全部被创建完了以后才会被执行。==这提示我们要分析IOC刷新的finishBeanFactoryInitialization方法==
- 紧接着下面一段的描述还说了，该接口的也是个回调接口
![[Pasted image 20240106230205.png]]

因此接下来我们就研究afterSingletonsInstantiated是在哪里被回调的即可

 afterSingletonsInstantiated回调追踪
# finishBeanFactoryInitialization
继续跟进代码，看refresh里面调用的finishBeanFactoryInitialization方法，顾名思义，该方法就是来完成BeanFactory的初始化工作的。也就是来初始化所有剩下的那些单实例bean的
![[Pasted image 20240106230257.png]]
继续跟进代码，太熟悉了，调用preInstantiateSingletons
![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=OTk1YmE1NmYwZTZmOWE2Y2M5NTlhMGQ0ZTAyYTQ0Y2JfS2REaFF4NUJOckZ6Rm1sdkpLczlFQVFoZ0ZUYzcxOThfVG9rZW46UFdka2J3cEJWbzRON214SXNPZmNMOEVubjZmXzE3MDQ1NTIwMjA6MTcwNDU1NTYyMF9WNA)

## preInstantiateSingletons
有两个大的for循环：
1. // Trigger initialization of all non-lazy singleton beans... 这块是getBean，也就是走的Bean生命周期
2. // Trigger post-initialization callback for all applicable beans... ==这块执行了回调方法afterSingletonsInstantiated==
```java
@Override  
public void preInstantiateSingletons() throws BeansException {  
    if (logger.isTraceEnabled()) {  
       logger.trace("Pre-instantiating singletons in " + this);  
    }  
  
    // Iterate over a copy to allow for init methods which in turn register new bean definitions.  
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);  
  
    // Trigger initialization of all non-lazy singleton beans...  
    for (String beanName : beanNames) {  
       RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);  
       if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {  
          if (isFactoryBean(beanName)) {  
             Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);  
             if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {  
                getBean(beanName);  
             }  
          }  
          else {  
             getBean(beanName);  
          }  
       }  
    }  


//在这里！
    // Trigger post-initialization callback for all applicable beans...  
    for (String beanName : beanNames) {  
       Object singletonInstance = getSingleton(beanName);  
       if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {  
          StartupStep smartInitialize = getApplicationStartup().start("spring.beans.smart-initialize")  
                .tag("beanName", beanName);  
        //回调方法afterSingletonsInstantiated
          smartSingleton.afterSingletonsInstantiated();  
          smartInitialize.end();  
       }  
    }  
}
```
### // Trigger post-initialization callback for all applicable beans...
在第一个for循环中（getBean所在的for循环），所有的单实例bean都已经全部创建完了。

因此，往下翻阅preInstantiateSingletons方法，在第二个for循环里面，beanNames里面存储的是创建好的所有bean的名字。

因此，在下面这个for循环中，咱们所要做的事就是
1. 获取所有创建好的单实例bean，
2. 然后判断每一个bean对象是否是SmartInitializingSingleton这个接口类型的，
3. 如果是，将bean做一个强转为SmartInitializingSingleton
4. 调用SmartInitializingSingleton里面的afterSingletonsInstantiated回调方法

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmZhMzBhODM2Y2Q5MDFkYjljMWM3ZGExNjk5NzY2NjVfbXB0eXlKekhMcHlncUY1WUFOeFdIOUEyYUFxcjAyUWxfVG9rZW46WmlkbWJtaHhOb2ZLNUZ4czViRWNaMXp3bllmXzE3MDQ1NTIwMjA6MTcwNDU1NTYyMF9WNA)
#### afterSingletonsInstantiated回调实现

```java
public void afterSingletonsInstantiated() {  
。。。。。。
     processBean(beanName, type);  
。。。。。。
}
```

##### processBean
```java
private void processBean(final String beanName, final Class<?> targetType) {  
    if (!this.nonAnnotatedClasses.contains(targetType) &&  
          AnnotationUtils.isCandidateClass(targetType, EventListener.class) &&  
          !isSpringContainerClass(targetType)) {  
  
       Map<Method, EventListener> annotatedMethods = null;  
       try {  
          annotatedMethods = MethodIntrospector.selectMethods(targetType,  
                (MethodIntrospector.MetadataLookup<EventListener>) method ->  
                      AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));  
       }  
       catch (Throwable ex) {  
          // An unresolvable type in a method signature, probably from a lazy bean - let's ignore it.  
          if (logger.isDebugEnabled()) {  
             logger.debug("Could not resolve methods for bean with name '" + beanName + "'", ex);  
          }  
       }  
  
       if (CollectionUtils.isEmpty(annotatedMethods)) {  
          this.nonAnnotatedClasses.add(targetType);  
          if (logger.isTraceEnabled()) {  
             logger.trace("No @EventListener annotations found on bean class: " + targetType.getName());  
          }  
       }  
       else {  
          // Non-empty set of methods  
          ConfigurableApplicationContext context = this.applicationContext;  
          Assert.state(context != null, "No ApplicationContext set");  
          List<EventListenerFactory> factories = this.eventListenerFactories;  
          Assert.state(factories != null, "EventListenerFactory List not initialized");  
          for (Method method : annotatedMethods.keySet()) {  
             for (EventListenerFactory factory : factories) {  
                if (factory.supportsMethod(method)) {  
                   Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));  
                   ApplicationListener<?> applicationListener =  
                         factory.createApplicationListener(beanName, targetType, methodToUse);  
                   if (applicationListener instanceof ApplicationListenerMethodAdapter alma) {  
                      alma.init(context, this.evaluator);  
                   }  
                   context.addApplicationListener(applicationListener);  
                   break;  
                }  
             }  
          }  
          if (logger.isDebugEnabled()) {  
             logger.debug(annotatedMethods.size() + " @EventListener methods processed on bean '" +  
                   beanName + "': " + annotatedMethods);  
          }  
       }  
    }  
}
```
首先解析注解EventListener.class，获得annotatedMethods，annotatedMethods信息如下，
- key:目标方法信息，public void MyApplicationListenerByAnnoatation.listen(org.springframework.context.ApplicationEvent)
- value:目标方法的注解参数，classes={org.springframework.context.ApplicationEvent.class}
![[Pasted image 20240106231849.png]]
然后遍历annotatedMethods的keySet集合，依次将key（目标method）封装成applicationListener，最终装入当前上下文context
```java
private ConfigurableApplicationContext applicationContext;
```
只不过debug的时候，context是我们测试类中创建的org.springframework.context.annotation.AnnotationConfigApplicationContext
==时刻谨记，ConfigurableApplicationContext是IOC容器的发动机==
![[Pasted image 20240106232801.png]]
# finishRefresh
紧接着便会来调用finishRefresh方法，容器已经创建完了，此时就会来发布容器已经刷新完成的事件。
![[Pasted image 20240106231233.png]]
因此，**SmartInitializingSingleton接口的调用时机有点类似于ContextRefreshedEvent事件，即在容器刷新完成以后，便会回调该接口**。