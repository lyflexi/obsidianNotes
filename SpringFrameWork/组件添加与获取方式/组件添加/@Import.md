@Import的value 有三种类型的Class.
1. ImportBeanDefinitionRegistrar.class 导入类定义注册者.
2. 1. ImportSelector.class 导入选择器。
3. 如果不是上面两种类型 就会被当作普通的configuration 类 注册到容器。 例如
	1. @EnableScheduling通过@Import将SchedulingConfiguration.class注册到容器中
	2. @EnableSpringConfigured通过@Import将SpringConfiguredConfiguration.class注册到容器中
# ImportBeanDefinitionRegistrar.class
找到`ImportBeanDefinitionRegistrar`接口，接口方法`registerBeanDefinitions`便是注入Bean的入口，接口方法参数已经由Spring自动注入
![[Pasted image 20231226145458.png]]

还是转回我们的自定义场景，可以通过注入`ImportBeanDefinitionRegistrar`接口的自定义实现，来定制化我们的业务

```Java
@Configuration
@Import(value = {MyImportBeanDefinitionRegistrar.class})
public class MyConfig {
    
}

public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                        BeanDefinitionRegistry registry) {
            RootBeanDefinition tDefinition = new RootBeanDefinition(Teacher.class);
            // 注册 Bean，并指定bean的名称和类型
            registry.registerBeanDefinition("teacher", tDefinition);
        }
    }
}
```

无独有偶，`ImportBeanDefinitionRegistrar`接口也为我们注入了Spring的切面配置。从`EnableAspectJAutoProxy`注解源码入手，发现注解参数`AspectJAutoProxyRegistrar.class实现了ImportBeanDefinitionRegistrar`接口。

```Java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    ......
}
```

`AspectJAutoProxyRegistrar.class`最终向IOC容器中注册一个`AnnotationAwareAspectJAutoProxyCreator`组件
![[Pasted image 20231226145506.png]]

# @Import+ImportSelector接口

@Import的value值是一个数组，一个一个注入比较繁琐，因此我们可以搭配ImportSelector接口来使用，用法如下：

```Java
//通过@Import注解，指定自定义的ImportSelector实现并将其注入到Spring容器中
@Configuration
@Import(MyImportSelector.class)
public class MyConfig {
    
}

//自定义selectImports方法，返回自定义的数组
public class MyImportSelector implements ImportSelector {
 @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{"org.springframework.demo.model.Teacher","org.springframework.demo.model.Student"};
    }
}
```

# 普通的configuration 类

我们在翻看Spring源码的过程中，经常会看到@Import注解，它也可以用来将第三方jar包注入Spring，但是它只可以作用在类上。
## SchedulingConfiguration.class
![[Pasted image 20231226160606.png]]
## SpringConfiguredConfiguration.class
例如在注解接口EnableSpringConfigured上就包含了`@Import(SpringConfiguredConfiguration.class)`注解，用于将SpringConfiguredConfiguration配置文件加载进Spring容器。
![[Pasted image 20231226161117.png]]