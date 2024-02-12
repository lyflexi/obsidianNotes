有的时候，当一个非Spring管理的类需要用到Spring中的组件的时候是无法通过@Autowired注入相关组件的，那怎么办？

hutool工具包中的SpringUtil提供了现成的解决方案：
```xml
<dependency>  
    <groupId>cn.hutool</groupId>  
    <artifactId>hutool-all</artifactId>  
    <version>5.4.6</version>  
</dependency>
```
如果是spring项目，要手动开启hutool包扫描，将cn.hutool.extra.spring包纳入Spring容器生效
```java
@Configuration  
@ComponentScan({"org.lyflexi.debug_springframework.aware","cn.hutool.extra.spring"})  
public class AwareConfig {  
  
    @Bean  
    public DogBean dogBean(){  
        return new DogBean();  
    }  
  
}
```
使用方式很简单，SpringUtil.getBean是个重载方法，既可以通过BeanName获取，也可以通过BeanType获取
```java
/**  
 * @Author: ly  
 * @Date: 2024/2/12 17:42  
 */  
//当前的SomeServiceNonSpring是非Spring管理的类，无法使用@Autowired注入所需要的组件，怎么办？  
//现成的解决方案是：通过hutool提供的SpringUtil.getBean方法获取
public class SomeServiceNonSpring {  
  
    public static void main(String[] args) {  
        //只是向Spring容器中注入配置类AwareConfig.class，让配置类生效  
        AnnotationConfigApplicationContext applicationContext  = new AnnotationConfigApplicationContext(AwareConfig.class);  
  
        //可以看到我们并没有通过applicationContext来获取Bean，而是通过SpringUtil来获取Bean  
        DogBean bean1 = SpringUtil.getBean(DogBean.class);  
        System.out.println(bean1);  
    }  
}
```
打印如下：
```shell
org.lyflexi.debug_springframework.aware.DogBean@e350b40
```
# 手写SpringUtil
请查阅SpringUtil的源码，我们发现SpringUtil实现了ApplicationContextAware接口，实现的是setApplicationContext方法，将上下文赋给了SpringUtil的静态成员变量SpringUtil.applicationContext
```java
@Override  
public void setApplicationContext(ApplicationContext applicationContext) {  
    SpringUtil.applicationContext = applicationContext;  
}
```
是不是恍然大悟了，接下来我们就手写SpringUtil，起名为MySpringUtil，==除了ApplicationContextAware接口外，我们再额外实现一个BeanFactoryAware接口，这两个Aware接口都可以获取IOC容器：==
- ApplicationContext的定位是供外部用户使用，通过重载方法将其传给MySpringUtil的静态成员变量applicationContext
- BeanFactory的定位是供Spring内部使用，通过重载方法将其传给MySpringUtil的静态成员变量beanFactory
同时自己提供两个静态获取Bean的方法：
- `getBeanByApplicationContextAware(Class<T> clazz)`
- `getBeanByBeanFactoryAware(Class<T> clazz)`
```java
/**  
 * @Author: ly  
 * @Date: 2024/2/12 11:35  
 */  
@Component  
public class MySpringUtil implements ApplicationContextAware, BeanFactoryAware {  
  
    private static BeanFactory beanFactory;//供Spring内部使用的IOC容器  
  
    private static ApplicationContext applicationContext;//供Spring外部用户们使用的IOC容器  
  
    @Override  
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {  
        MySpringUtil.beanFactory = beanFactory;  
  
    }  
  
    @Override  
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {  
        MySpringUtil.applicationContext = applicationContext;  
    }  
  
    public static <T> T getBeanByApplicationContextAware(Class<T> clazz) {  
        return MySpringUtil.applicationContext.getBean(clazz);  
    }  
  
    public static <T> T getBeanByBeanFactoryAware(Class<T> clazz) {  
        return MySpringUtil.beanFactory.getBean(clazz);  
    }  
}
```
最后别忘了对MySpringUtil加上@Component注解，虽说Aware的使用场景是“供非Spring管理的类使用”，但不代表MySpringUtil本身就不被Spring管理了哟!

顺其自然的，对于Spring项目，也要扫描MySpringUtil所在包“org.lyflexi.debug_springframework.aware”
```java
@Configuration  
@ComponentScan({"org.lyflexi.debug_springframework.aware","cn.hutool.extra.spring"})  
public class AwareConfig {  
  
    @Bean  
    public DogBean dogBean(){  
        return new DogBean();  
    }  
  
}
```
使用方式如下：
```java
//当前的SomeServiceNonSpring是非Spring管理的类，无法使用@Autowired注入所需要的组件，怎么办？  
public class SomeServiceNonSpring {  
  
    public static void main(String[] args) {  
        //只是向Spring容器中注入配置类AwareConfig.class，让配置类生效  
        AnnotationConfigApplicationContext applicationContext  = new AnnotationConfigApplicationContext(AwareConfig.class);  
  
        //可以看到我们并没有通过applicationContext来获取Bean，而是通过SpringUtil来获取Bean  
        DogBean bean1 = SpringUtil.getBean(DogBean.class);  
        System.out.println(bean1);  
  
        //使用我们自己手写的MySpringUtil.getBeanByApplicationContextAware  
        DogBean bean2 = MySpringUtil.getBeanByApplicationContextAware(DogBean.class);  
        System.out.println(bean2);  
        //使用我们自己手写的MySpringUtil.getBeanByApplicationContextAware  
        DogBean bean3 = MySpringUtil.getBeanByBeanFactoryAware(DogBean.class);  
        System.out.println(bean3);  
    }  
}
```
打印信息如下：
```java
org.lyflexi.debug_springframework.aware.DogBean@e350b40
org.lyflexi.debug_springframework.aware.DogBean@e350b40
org.lyflexi.debug_springframework.aware.DogBean@e350b40
```
完整代码见github