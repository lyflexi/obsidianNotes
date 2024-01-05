`BeanFactoryPostProcessor`接口是Bean工厂的后置处理器
![[Pasted image 20231226150111.png]]

# BeanFactoryPostProcessor的调用时机分析

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

# 代码验证

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
MyBeanFactoryPostProcessor...postProcessBeanFactory...
当前BeanFactory中有9个Bean
[org.springframework.context.annotation.internalConfigurationAnnotationProcessor,...,
        ...,org.springframework.context.event.internalEventListenerFactory,extConfig,myBeanFactoryPostProcessor,blue}
blue...constructor
```

细心一点看的话，从Bean的定义信息中还能看到Blue组件注册到容器中的名字，只是此刻还没创建对象，说明BeanFactoryPostProcessor是在所有的Bean定义信息都被加载之后，Bean的创建与初始化之前，调用的