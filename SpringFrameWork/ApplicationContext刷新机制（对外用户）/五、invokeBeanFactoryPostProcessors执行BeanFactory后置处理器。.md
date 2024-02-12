在上一讲中，我们详细地分析了一下BeanFactory的创建以及预准备工作的流程。紧接上一讲，我们就要来看看接下来又做了哪些工作。
现在，程序已经运行到了下面这行代码处了。
![[Pasted image 20240105101955.png]]

可以看到这儿会执行一个叫invokeBeanFactoryPostProcessors的方法，是来执行BeanFactory的后置处理器BeanFactoryPostProcessor的。
看看BeanFactoryPostProcessor的源码注释
![[Pasted image 20240105102318.png]]

而BeanFactory标准初始化正是前面的prepareBeanFactory方法

接下来，我们就来看看invokeBeanFactoryPostProcessors这个方法里面到底做了哪些事，也就是看一下BeanFactoryPostProcessor的整个执行过程

其实，当你看完这篇文章之后，你就知道了在invokeBeanFactoryPostProcessors方法里面主要就是执行了BeanFactoryPostProcessors#postProcessBeanFactory方法。

```Java

@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory var1) throws BeansException;
}
```

或者BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry和postProcessBeanFactory这俩方法，

```Java
//注意这里是接口继承，不是接口实现
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry var1) throws BeansException;
}
```

tips：先执行BeanDefinitionRegistryPostProcessor的方法


我们可以按下F5快捷键进入invokeBeanFactoryPostProcessors方法里面去瞧一瞧，如下图所示，可以看到现在程序来到了如下这行代码处。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=ODIxNTU4ZTJlYmM0ZjFjOTlkZmU5ZDQyYWM0Zjc2MDhfVDBCU05IZEIzdmlGVFVxVGI3aVllT1pIbXZMMjEzU0NfVG9rZW46Ujg1OWJSWjlLb1NpZEJ4ZGV3RGNXd3g5bmZmXzE3MDQ0MjA5ODU6MTcwNDQyNDU4NV9WNA)

再次进入PostProcessorRegistrationDelegate的invokeBeanFactoryPostProcessors方法，这个方法可太熟悉了，臭长臭长

![[Pasted image 20240105102745.png]]
大家一定要注意哟！这里会先来判断我们这个beanFactory是不是BeanDefinitionRegistry。之前我们在上一讲中就已经说过了，我们生成的BeanFactory对象是==DefaultListableBeanFactory类型的==，而且还==使用了ConfigurableListableBeanFactory接口进行接收==。这里我们就来看下DefaultListableBeanFactory类是不是实现了BeanDefinitionRegistry接口：是！

![[Pasted image 20240105103700.png]]
  
自然地，程序就会进入到if判断语句中，进来以后呢，我们来大致地分析一下下面的流程。

首先，映入眼帘的是一个for循环，它是来循环遍历invokeBeanFactoryPostProcessors方法中的第二个参数的，即beanFactoryPostProcessors。

其实呢，就是拿到所有的BeanFactoryPostProcessor，再挨个遍历出来。然后，再来以遍历出来的每一个BeanFactoryPostProcessor是否实现了BeanDefinitionRegistryPostProcessor接口为依据将其分别存放于以下两个箭头所指向的LinkedList中，其中实现了BeanDefinitionRegistryPostProcessor接口的还会被直接调用。即`registryProcessor.postProcessBeanDefinitionRegistry(registry);`
![[Pasted image 20240105104044.png]]
往下看这段源码超级长：


## 后置处理器BeanDefinitionRegistryPostProcessor调用时机

```Java
public static void invokeBeanFactoryPostProcessors(
       ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {



    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    Set<String> processedBeans = new HashSet<>();

    if (beanFactory instanceof BeanDefinitionRegistry registry) {
       List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
       List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

       // registryProcessors && regularPostProcessors 
       for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
          if (postProcessor instanceof BeanDefinitionRegistryPostProcessor registryProcessor) {
             registryProcessor.postProcessBeanDefinitionRegistry(registry);
             registryProcessors.add(registryProcessor);
          }
          else {
             regularPostProcessors.add(postProcessor);
          }
       }

       // Do not initialize FactoryBeans here: We need to leave all regular beans
       // uninitialized to let the bean factory post-processors apply to them!
       // Separate between BeanDefinitionRegistryPostProcessors that implement
       // PriorityOrdered, Ordered, and the rest.
       List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
       
// Invoke BeanDefinitionRegistryPostProcessors orderd
       // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
       String[] postProcessorNames =
             beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
       for (String ppName : postProcessorNames) {
          if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
             currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
             processedBeans.add(ppName);
          }
       }
       sortPostProcessors(currentRegistryProcessors, beanFactory);
       registryProcessors.addAll(currentRegistryProcessors);
       invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
       currentRegistryProcessors.clear();

       // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
       postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
       for (String ppName : postProcessorNames) {
          if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
             currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
             processedBeans.add(ppName);
          }
       }
       sortPostProcessors(currentRegistryProcessors, beanFactory);
       registryProcessors.addAll(currentRegistryProcessors);
       invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
       currentRegistryProcessors.clear();

       // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
       boolean reiterate = true;
       while (reiterate) {
          reiterate = false;
          postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
          for (String ppName : postProcessorNames) {
             if (!processedBeans.contains(ppName)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
                reiterate = true;
             }
          }
          sortPostProcessors(currentRegistryProcessors, beanFactory);
          registryProcessors.addAll(currentRegistryProcessors);
          invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
          currentRegistryProcessors.clear();
       }
// end orderd
       // registryProcessors（List<BeanDefinitionRegistryPostProcessor>）
       invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        // regularPostProcessors（List<BeanFactoryPostProcessor>）
       invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }
    else{父接口BeanFactoryPostProcessor....也很长}
```

1. invokeBeanDefinitionRegistryPostProcessors：执行BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry
2. invokeBeanFactoryPostProcessors(registryProcessors, beanFactory)：执行BeanDefinitionRegistryPostProcessor的postProcessBeanFactory
3. invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory)：执行BeanFactoryPostProcessor的postProcessBeanFactory

# 后置处理器BeanFactoryPostProcessor调用时机

```Java
public static void invokeBeanFactoryPostProcessors(
    ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {


			if (beanFactory instanceof BeanDefinitionRegistry registry) {...很长：Invoke BeanDefinitionRegistryPostProcessors first, if any...}

            else {
            // Invoke factory processors registered with the context instance.
            invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
            }

            // Do not initialize FactoryBeans here: We need to leave all regular beans
            // uninitialized to let the bean factory post-processors apply to them!
            String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

            // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
            // Ordered, and the rest.
            List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
            List<String> orderedPostProcessorNames = new ArrayList<>();
            List<String> nonOrderedPostProcessorNames = new ArrayList<>();
            for (String ppName : postProcessorNames) {
            if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
            }
            else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            }
            else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
            }
            else {
            nonOrderedPostProcessorNames.add(ppName);
            }
            }

            // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
            sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
            invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

            // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
            List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
            for (String postProcessorName : orderedPostProcessorNames) {
            orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
            }
            sortPostProcessors(orderedPostProcessors, beanFactory);
            invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

            // Finally, invoke all other BeanFactoryPostProcessors.
            List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
            for (String postProcessorName : nonOrderedPostProcessorNames) {
            nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
            }
            invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

            // Clear cached merged bean definitions since the post-processors might have
            // modified the original metadata, e.g. replacing placeholders in values...
            beanFactory.clearMetadataCache();
            }
```

BeanDefinitionRegistryPostProcessor后置处理器是要优先于BeanFactoryPostProcessor后置处理器执行的。
invokeBeanFactoryPostProcessors方法最主要的核心作用就是执行了
1. BeanDefinitionRegistryPostProcessor
	1. postProcessBeanDefinitionRegistry方法
	2. postProcessBeanFactory方法
2. BeanFactoryPostProcessors的postProcessBeanFactory方法。