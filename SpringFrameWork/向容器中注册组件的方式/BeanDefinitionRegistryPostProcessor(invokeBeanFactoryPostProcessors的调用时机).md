`BeanFactoryPostProcessor`接口是Bean工厂的后置处理器
![[Pasted image 20231226150111.png]]

# BeanFactoryPostProcessor通过beanFactory注册bean

`BeanFactoryPostProcessor`接口优先于Bean的创建和初始化

Spring首先从IOC容器中找到所有类型是BeanFactoryPostProcessor的组件，根据BeanFactoryPostProcessor类型组件的组件名字，然后再来执行它们其中的方法postProcessBeanFactory，而且是在初始化创建Bean之前执行。接下来，我们就以debug的方式来看一下BeanFactoryPostProcessor的调用时机。
![[Pasted image 20231226150215.png]]

从invokeBeanFactoryPostProcessors(beanFactory)入手，跟进代码，发现又调用了一个invokeBeanFactoryPostProcessors方法，如下图所示。
![[Pasted image 20231226150223.png]]
![[Pasted image 20231226150234.png]]
继续跟进代码，这里是PostProcessorRegistrationDelegate类invokeBeanFactoryPostProcessors方法：
- 首先拿到所有BeanFactoryPostProcessor组件的名字String[] postProcessorNames
- 然后，来挨个看相应名字的BeanFactoryPostProcessor组件，哪些是实现了PriorityOrdered接口的，哪些是实现了Ordered接口的，以及哪些是什么接口都没有实现的，将不同优先级的BeanFactoryPostProcessor组件给分离出来，装到三个容器中
    - `priorityOrderedPostProcessors`
    - `orderedPostProcessorNames`
    - `nonOrderedPostProcessorNames`
- 最后，分别按不同的执行顺序执行三种不同的BeanFactoryPostProcessor组件。
    - `invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);`
    - `invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);`
    - `invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);`

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


接下来，我们来编写一个案例来验证一下以上说的内容。自定义MyBeanFactoryPostProcessor实现我们上面说的BeanFactoryPostProcessor接口。注意，我们自己编写的MyBeanFactoryPostProcessor类要想让Spring知道，并且还要能被使用起来，那么它一定就得被加在容器中，为此，我们可以在其上标注一个@Component注解。

```Java
package com.meimeixia.ext;

import java.util.Arrays;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;

@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
                System.out.println("MyBeanFactoryPostProcessor...postProcessBeanFactory..."); // 这个时候我们所有的bean还没被创建
                // 但是我们可以看一下通过Spring给我们传过来的这个beanFactory，我们能拿到什么
                int count = beanFactory.getBeanDefinitionCount(); // 我们能拿到有几个bean定义
                String[] names = beanFactory.getBeanDefinitionNames(); // 除此之外，我们还能拿到每一个bean定义的名字
                System.out.println("当前BeanFactory中有" + count + "个Bean");
                System.out.println(Arrays.asList(names));
        }

}
```

然后，创建一个配置类，例如ExtConfig，记得还要在该配置类上使用@ComponentScan注解来配置包扫描哟！

当然了，我们也可以使用@Bean注解向容器中注入咱自己写的组件，例如，在这里，我们可以向容器中注入一个Blue组件。

```Java
package com.meimeixia.ext;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

import com.meimeixia.bean.Blue;

/**
 * 
 * @author liayun
 *
 */
@ComponentScan("com.meimeixia.ext")
@Configuration
public class ExtConfig {

        @Bean
        public Blue blue() {
                return new Blue();
        }
        
}
//上面这个Blue组件其实就是一个非常普通的组件，代码如下所示：在创建Blue对象的时候，无参构造器会有相应打印。
public class Blue {

        public Blue() {
                System.out.println("blue...constructor");
        }
        
        public void init() {
                System.out.println("blue...init...");
        }
        
        public void destory() {
                System.out.println("blue...destory...");
        }
        
}
```

接着，编写一个单元测试类，例如IOCTest_Ext，来进行测试。测试啥呢？其实就是来验证一下BeanFactoryPostProcessor的调用时机。以确认咱们自己编写的BeanFactoryPostProcessor要在所有的Bean定义已经被加载，但是还未创建Bean对象的时候工作？

```Java
package com.meimeixia.test;

import org.junit.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import com.meimeixia.ext.ExtConfig;

public class IOCTest_Ext {
        
        @Test
        public void test01() {
                AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ExtConfig.class);
                
                // 关闭容器
                applicationContext.close();
        }

}
```

运行IOCTest_Ext类中的test01方法，可以看到Eclipse控制台打印出了如下内容。哎呀！咱们自己编写的BeanFactoryPostProcessor在Blue类的无参构造器创建Blue对象之前就已经工作了。

```Shell
postProcessBeanDefinitionRegistry...bean的数量：9
MyBeanDefinitionRegistryPostProcessor...bean的数量：10
MyBeanFactoryPostProcessor...postProcessBeanFactory...
当前BeanFactory中有10 个Bean
[org.springframework.context.annotation.internalConfigurationAnnotationProcessor, org.springframework.context.annotation.internalAutowiredAnnotationProcessor, org.springframework.context.annotation.internalCommonAnnotationProcessor, org.springframework.context.event.internalEventListenerProcessor, org.springframework.context.event.internalEventListenerFactory, extConfig, myBeanDefinitionRegistryPostProcessor, myBeanFactoryPostProcessor, blue, hello]
blue...constructor
blue...constructor
```

细心一点看的话，从Bean的定义信息中还能看到Blue组件注册到容器中的名字，只是此刻还没创建对象，说明BeanFactoryPostProcessor是在所有的Bean定义信息都被加载之后，Bean的创建与初始化之前，调用的

BeanFactoryPostProcessor通过方法参数beanFactory来注册bean，如下
```java
@Component
public class MsgHandlerProcessor implements BeanFactoryPostProcessor {

    private static final String HANDLER_PACKAGE = "com.iwhalecloud.aiFactory.aspect.msghandler.process";

    /*
     * 扫描@PassiveMsgHandlerType，MsgHandlerContext，将其注册到spring容器
     *
     * @param beanFactory bean工厂
     */
     
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        Map<String, Class> handlerMap = Maps.newHashMapWithExpectedSize(3);
        //只扫描带有@PassiveMsgHandlerType注解的handler
        ClassScaner.scan(HANDLER_PACKAGE, PassiveMsgHandlerType.class).forEach(clazz -> {
            // 获取注解中的类型值
            String type = clazz.getAnnotation(PassiveMsgHandlerType.class).value();
            // 将注解中的类型值作为key，对应的类作为value，保存在Map中
            handlerMap.put(type, clazz);
            beanFactory.registerSingleton(clazz.getName(),clazz);
        });
        // 创建自己的HandlerContext，也将其注册到spring容器中，HandlerContext用于后续获取handler
        MsgHandlerContext context = new MsgHandlerContext(handlerMap);
        beanFactory.registerSingleton(MsgHandlerContext.class.getName(), context);
    }

}
```


# BeanDefinitionRegistryPostProcessor通过registry注册Bean
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

# 调用时机验证与对比

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
                // 咱们也可以用另外一种办法，即使用BeanDefinitionBuilder来生成一个BeanDefinition对象，很显然，这两种方法的效果都是一样的
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