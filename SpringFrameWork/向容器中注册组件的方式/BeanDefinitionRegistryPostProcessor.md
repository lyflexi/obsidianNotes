BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的一个子接口：

当用户自定义BeanDefinitionRegistryPostProcessor的时候，必将同时实现两个方法

- ==postProcessBeanDefinitionRegistry，今天的重点：==
    - 它的参数是BeanDefinitionRegistry，顾名思义就是与BeanDefinition注册相关的。
    - 参数BeanDefinitionRegistry有一个方法叫做registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
- 继承来的postProcessBeanFactory

```Java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {


    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;


    @Override
    default void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    }

}
```
# BeanDefinitionRegistryPostProcessor调用时机分析
跟BeanFactoryPostProcessor一样，还是看IOC容器刷新AbstractApplicationContext#refresh，从invokeBeanFactoryPostProcessors方法入手，跟进源码：
1. BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry(registry) 方法。
    1. invokeBeanFactoryPostProcessors一开始，首先触发postProcessBeanDefinitionRegistry(registry) 方法
    2. 然后再来触发它们的postProcessBeanFactory方法，对于这些 Bean 按照 PriorityOrdered 接口、Ordered、没有排序接口的实例分别进行处理。
2. 调用 BeanFactoryPostProcessor#postProcessBeanFactory(beanFactory) 方法。
    1. 依次触发它们的postProcessBeanFactory方法。对于这些 Bean 按照 PriorityOrdered 接口、Ordered、没有排序接口的实例分别进行处理。
    ![[Pasted image 20231226151342.png]]
完整的invokeBeanFactoryPostProcessors方法如下：
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
}
```

# 代码验证

接下来编写测试代码进行验证。编写一个类MyBeanDefinitionRegistryPostProcessor，实现BeanDefinitionRegistryPostProcessor接口

```Java
package com.meimeixia.ext;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.stereotype.Component;

import com.meimeixia.bean.Blue;

// 记住，我们这个组件写完之后，一定别忘了给它加在容器中
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
                // TODO Auto-generated method stub
                System.out.println("MyBeanDefinitionRegistryPostProcessor...bean的数量：" + beanFactory.getBeanDefinitionCount());
        }

        /**
         * 这个BeanDefinitionRegistry就是Bean定义信息的保存中心，这个注册中心里面存储了所有的bean定义信息，
         * 以后，BeanFactory就是按照BeanDefinitionRegistry里面保存的每一个bean定义信息来创建bean实例的。
         * 
         * bean定义信息包括有哪些呢？有这些，这个bean是单例的还是多例的、bean的类型是什么以及bean的id是什么。
         * 也就是说，这些信息都是存在BeanDefinitionRegistry里面的。
         */
        @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
                // TODO Auto-generated method stub
                System.out.println("postProcessBeanDefinitionRegistry...bean的数量：" + registry.getBeanDefinitionCount());
                // 除了查看bean的数量之外，我们还可以给容器里面注册一些bean，我们以前也简单地用过
                /*
                 * 第一个参数：我们将要给容器中注册的bean的名字
                 * 第二个参数：BeanDefinition对象
                 */
                // RootBeanDefinition beanDefinition = new RootBeanDefinition(Blue.class); // 现在我准备给容器中添加一个Blue对象
                // 咱们也可以用另外一种办法，即使用BeanDefinitionBuilder这个构建器生成一个BeanDefinition对象，很显然，这两种方法的效果都是一样的
                AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(Blue.class).getBeanDefinition();
                registry.registerBeanDefinition("hello", beanDefinition);
        }

}
```

接下来，我们就来测试一下以上类里面的两个方法是什么时候执行的。运行IOCTest_Ext测试类，我们可以得出这样两个结论
- ==BeanDefinitionRegistryPostProcessor接口是优先于BeanFactoryPostProcessor接口执行的==
- ==并且，在BeanDefinitionRegistryPostProcessor接口中，postProcessBeanDefinitionRegistry方法优先于postProcessBeanFactory方法调用==

```Shell
postProcessBeanDefinitionRegistry...bean的数量: 10
MyBeanDefinitionRegistryPostProcessor...bean的数量：11
---
MyBeanFactoryPostProcessor...postProcessBeanFactory...
当前BeanFactory中有9个Bean
[org.springframework.context.annotation.internalConfigurationAnnotationProcessor,...,
                ...,org.springframework.context.event.internalEventListenerFactory,extConfig,myBeanFactoryPostProcessor,blue}
blue...constructor
```

可以看到，首先`postProcessBeanDefinitionRegistry`方法先执行，拿到IOC容器中Bean的数量（即10）之后向IOC容器中注册一个hello组件。

接着是`postProcessBeanFactory`方法再执行，该方法只是打印了一下IOC容器中Bean的数量11（算上hello）

除此之外，MyBeanDefinitionRegistryPostProcessor类里面的方法都执行完了以后，才轮到之前写的MyBeanFactoryPostProcessor来执行