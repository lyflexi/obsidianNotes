前面章节讲的是内置IOC容器的标准初始化，以及执行IOC容器的后置处理器

这一节注册的是Bean的后置处理器BeanPostProcessor
![[Pasted image 20240105111700.png]]
从该方法上的描述上，翻译过来就是注册bean的后置处理器，拦截bean的创建过程。

> 题外话，在创建AOP的核心类时，就是调用这个方法来进行处理的。

跟踪源码，如下图所示，可以看到在该方法里面会调用PostProcessorRegistrationDelegate类的registerBeanPostProcessors方法。
![[Pasted image 20240105112233.png]]
==PostProcessorRegistrationDelegate==类里面一堆的静态方法，多次见于：
1. invokeBeanFactoryPostProcessors方法中调用了PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()
2. registerBeanPostProcessors方法中调用了PostProcessorRegistrationDelegate.registerBeanPostProcessors()
打开PostProcessorRegistrationDelegate的源码，翻译过来是作为==AbstractApplicationContext's post-processor handling的委托类==
```java
Delegate for AbstractApplicationContext's post-processor handling.
Since:
4.0
Author:
Juergen Hoeller, Sam Brannen, Stephane Nicoll
```


# registerBeanPostProcessors分析
```java
public static void registerBeanPostProcessors(  
       ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {  
  
    // WARNING: Although it may appear that the body of this method can be easily  
    // refactored to avoid the use of multiple loops and multiple lists, the use    // of multiple lists and multiple passes over the names of processors is    // intentional. We must ensure that we honor the contracts for PriorityOrdered    // and Ordered processors. Specifically, we must NOT cause processors to be    // instantiated (via getBean() invocations) or registered in the ApplicationContext    // in the wrong order.    //    // Before submitting a pull request (PR) to change this method, please review the    // list of all declined PRs involving changes to PostProcessorRegistrationDelegate    // to ensure that your proposal does not result in a breaking change:    // https://github.com/spring-projects/spring-framework/issues?q=PostProcessorRegistrationDelegate+is%3Aclosed+label%3A%22status%3A+declined%22  
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);  
  
    // Register BeanPostProcessorChecker that logs an info message when  
    // a bean is created during BeanPostProcessor instantiation, i.e. when    // a bean is not eligible for getting processed by all BeanPostProcessors.    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;  
    beanFactory.addBeanPostProcessor(  
          new BeanPostProcessorChecker(beanFactory, postProcessorNames, beanProcessorTargetCount));  
  
    // Separate between BeanPostProcessors that implement PriorityOrdered,  
    // Ordered, and the rest.    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();  
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();  
    List<String> orderedPostProcessorNames = new ArrayList<>();  
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();  
    for (String ppName : postProcessorNames) {  
       if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {  
          BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);  
          priorityOrderedPostProcessors.add(pp);  
          if (pp instanceof MergedBeanDefinitionPostProcessor) {  
             internalPostProcessors.add(pp);  
          }  
       }  
       else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {  
          orderedPostProcessorNames.add(ppName);  
       }  
       else {  
          nonOrderedPostProcessorNames.add(ppName);  
       }  
    }  
  
    // First, register the BeanPostProcessors that implement PriorityOrdered.  
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);  
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);  
  
    // Next, register the BeanPostProcessors that implement Ordered.  
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());  
    for (String ppName : orderedPostProcessorNames) {  
       BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);  
       orderedPostProcessors.add(pp);  
       if (pp instanceof MergedBeanDefinitionPostProcessor) {  
          internalPostProcessors.add(pp);  
       }  
    }  
    sortPostProcessors(orderedPostProcessors, beanFactory);  
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);  
  
    // Now, register all regular BeanPostProcessors.  
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());  
    for (String ppName : nonOrderedPostProcessorNames) {  
       BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);  
       nonOrderedPostProcessors.add(pp);  
       if (pp instanceof MergedBeanDefinitionPostProcessor) {  
          internalPostProcessors.add(pp);  
       }  
    }  
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);  
  
    // Finally, re-register all internal BeanPostProcessors.  
    sortPostProcessors(internalPostProcessors, beanFactory);  
    registerBeanPostProcessors(beanFactory, internalPostProcessors);  
  
    // Re-register post-processor for detecting inner beans as ApplicationListeners,  
    // moving it to the end of the processor chain (for picking up proxies etc).    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));  
}
```
## 获取所有BeanPostProcessor组件的名字

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
```

这里，我会挑出如下BeanPostProcessor的几个子接口将其罗列出来，目的是为了告诉大家BeanPostProcessor接口旗下确实是有非常多的子接口，而且这些不同接口类型的BeanPostProcessor在bean创建前后的执行时机是不一样的，虽然它们都是后置处理器。

- DestructionAwareBeanPostProcessor：该接口我们之前是不是说过啊？它是销毁bean的后置处理器
    
- InstantiationAwareBeanPostProcessor
    
- SmartInstantiationAwareBeanPostProcessor
    
- MergedBeanDefinitionPostProcessor
    

## 添加BeanPostProcessorChecker后置处理器
```java
// Register BeanPostProcessorChecker that logs an info message when  
// a bean is created during BeanPostProcessor instantiation, i.e. when  
// a bean is not eligible for getting processed by all BeanPostProcessors.  
int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;  
beanFactory.addBeanPostProcessor(  
       new BeanPostProcessorChecker(beanFactory, postProcessorNames, beanProcessorTargetCount));
```

现在向beanFactory中添加了一个BeanPostProcessorChecker类型的后置处理器，它是来检查所有BeanPostProcessor组件的。

## 按分好类的优先级顺序来注册BeanPostProcessor

继续按下F6快捷键让程序往下运行，在这一过程中，可以看到后置处理器也可以按照
是否实现了PriorityOrdered接口、Ordered接口以及没有实现这两个接口这三种情况进行分类。
不再赘述

```java
// Separate between BeanPostProcessors that implement PriorityOrdered,  
// Ordered, and the rest.  
List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();  
List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();  
List<String> orderedPostProcessorNames = new ArrayList<>();  
List<String> nonOrderedPostProcessorNames = new ArrayList<>();  
for (String ppName : postProcessorNames) {  
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {  
       BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);  
       priorityOrderedPostProcessors.add(pp);  
       if (pp instanceof MergedBeanDefinitionPostProcessor) {  
          internalPostProcessors.add(pp);  
       }  
    }  
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {  
       orderedPostProcessorNames.add(ppName);  
    }  
    else {  
       nonOrderedPostProcessorNames.add(ppName);  
    }  
}  
  
// First, register the BeanPostProcessors that implement PriorityOrdered.  
sortPostProcessors(priorityOrderedPostProcessors, beanFactory);  
registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);  
  
// Next, register the BeanPostProcessors that implement Ordered.  
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());  
for (String ppName : orderedPostProcessorNames) {  
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);  
    orderedPostProcessors.add(pp);  
    if (pp instanceof MergedBeanDefinitionPostProcessor) {  
       internalPostProcessors.add(pp);  
    }  
}  
sortPostProcessors(orderedPostProcessors, beanFactory);  
registerBeanPostProcessors(beanFactory, orderedPostProcessors);  
  
// Now, register all regular BeanPostProcessors.  
List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());  
for (String ppName : nonOrderedPostProcessorNames) {  
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);  
    nonOrderedPostProcessors.add(pp);  
    if (pp instanceof MergedBeanDefinitionPostProcessor) {  
       internalPostProcessors.add(pp);  
    }  
}  
registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
```
## 注册internalPostProcessors类型后置处理器

最后，再来注册MergedBeanDefinitionPostProcessor这种类型的BeanPostProcessor，因为名字为internalPostProcessors的ArrayList集合中存放的就是这种类型的BeanPostProcessor。
```java
// Finally, re-register all internal BeanPostProcessors.  
sortPostProcessors(internalPostProcessors, beanFactory);  
registerBeanPostProcessors(beanFactory, internalPostProcessors);
```
![[Pasted image 20240105113404.png]]
## 注册ApplicationListenerDetector类型后置处理器
除此之外，还会向beanFactory中添加一个ApplicationListenerDetector类型的BeanPostProcessor。
```java
// Re-register post-processor for detecting inner beans as ApplicationListeners,  
// moving it to the end of the processor chain (for picking up proxies etc).  
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
```

我们不妨点进ApplicationListenerDetector类的源码里面去看一看，如下图所示，它里面有一个postProcessAfterInitialization方法，该方法是在bean创建初始化之后，探测该bean是不是ApplicationListener的。
![[Pasted image 20240105113635.png]]
也就是说，该方法的作用是检查哪些bean是监听器的。如果是，那么会将该bean放在容器中保存起来。

最最后，我得多提一嘴，**以上只是来注册bean的后置处理器，即只是向beanFactory中添加了所有这些bean的后置处理器，而并不会执行它们。**