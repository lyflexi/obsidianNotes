Aware接口是个空接口，接口注释中说到一个bean有资格由Spring以一种特点的框架机制（通过回调）来通知
```java
/*A marker superinterface indicating that a bean is eligible to be notified by the Spring container of a particular framework object through a callback-style method. The actual method signature is determined by individual subinterfaces but should typically consist of just one void-returning method that accepts a single argument.
*/
public interface Aware {  
  
}
```
接下来以BeanFactoryAware和ApplicationContextAware这两个接口为例，分析Aware的生效时机，在以下两处打上断点：
- BeanFactoryAware#setBeanFactory
- ApplicationContextAware#setApplicationContext
```java
package org.lyflexi.debug_springframework.aware;  
  
import org.springframework.beans.BeansException;  
import org.springframework.beans.factory.BeanFactory;  
import org.springframework.beans.factory.BeanFactoryAware;  
import org.springframework.context.ApplicationContext;  
import org.springframework.context.ApplicationContextAware;  
import org.springframework.stereotype.Component;  
  
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

# BeanFactoryAware
可以看到，setBeanFactory为于栈顶，SomeServiceNonSpring的第21行（main方法）位于栈底
![[Pasted image 20240212184916.png]]
==因此对于BeanFactoryAware结论就是，生效时机位于bean的初始化initializeBean方法中的第一步：invokeAwareMethods(beanName, bean);  ==
```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {  
	//initializeBean第一步
    invokeAwareMethods(beanName, bean);  
  
    Object wrappedBean = bean;  
    if (mbd == null || !mbd.isSynthetic()) {  
    //initializeBean第二步
       wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
    }  
  
    try {  
    //initializeBean第三步
       invokeInitMethods(beanName, wrappedBean, mbd);  
    }  
    catch (Throwable ex) {  
       throw new BeanCreationException(  
             (mbd != null ? mbd.getResourceDescription() : null), beanName, ex.getMessage(), ex);  
    }  
    if (mbd == null || !mbd.isSynthetic()) {  
    //initializeBean第四步
       wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
    }  
  
    return wrappedBean;  
}
```
进入invokeAwareMethods方法查看，回调方法BeanFactoryAware.setBeanFactory被执行，一目了然
```java
private void invokeAwareMethods(String beanName, Object bean) {  
    if (bean instanceof Aware) {  
       if (bean instanceof BeanNameAware beanNameAware) {  
          beanNameAware.setBeanName(beanName);  
       }  
       if (bean instanceof BeanClassLoaderAware beanClassLoaderAware) {  
          ClassLoader bcl = getBeanClassLoader();  
          if (bcl != null) {  
             beanClassLoaderAware.setBeanClassLoader(bcl);  
          }  
       }  
       if (bean instanceof BeanFactoryAware beanFactoryAware) {  
		  //BeanFactoryAware接口方法的回调处
          beanFactoryAware.setBeanFactory(AbstractAutowireCapableBeanFactory.this);  
       }  
    }  
}
```
# ApplicationContextAware
Resume Program放行上一个断点，来到ApplicationContextAware#setApplicationContext
![[Pasted image 20240212190135.png]]
发现来到了invokeAwareInterfaces方法，咦？这个方法有点陌生，不慌我们逐步分析
```java
private void invokeAwareInterfaces(Object bean) {  
    if (bean instanceof Aware) {  
       if (bean instanceof EnvironmentAware environmentAware) {  
          environmentAware.setEnvironment(this.applicationContext.getEnvironment());  
       }  
       if (bean instanceof EmbeddedValueResolverAware embeddedValueResolverAware) {  
          embeddedValueResolverAware.setEmbeddedValueResolver(this.embeddedValueResolver);  
       }  
       if (bean instanceof ResourceLoaderAware resourceLoaderAware) {  
          resourceLoaderAware.setResourceLoader(this.applicationContext);  
       }  
       if (bean instanceof ApplicationEventPublisherAware applicationEventPublisherAware) {  
          applicationEventPublisherAware.setApplicationEventPublisher(this.applicationContext);  
       }  
       if (bean instanceof MessageSourceAware messageSourceAware) {  
          messageSourceAware.setMessageSource(this.applicationContext);  
       }  
       if (bean instanceof ApplicationStartupAware applicationStartupAware) {  
          applicationStartupAware.setApplicationStartup(this.applicationContext.getApplicationStartup());  
       }  
       if (bean instanceof ApplicationContextAware applicationContextAware) {  
	      //ApplicationContextAware接口方法的回调处
          applicationContextAware.setApplicationContext(this.applicationContext);  
       }  
    }  
}
```
==跟踪堆栈信息，逐步退栈，来到了initializeBean方法的第二步，初始化Bean的前置处理applyBeanPostProcessorsBeforeInitialization==
```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {  
	//initializeBean第一步
    invokeAwareMethods(beanName, bean);  
  
    Object wrappedBean = bean;  
    if (mbd == null || !mbd.isSynthetic()) {  
    //initializeBean第二步
       wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
    }  
  
    try {  
    //initializeBean第三步
       invokeInitMethods(beanName, wrappedBean, mbd);  
    }  
    catch (Throwable ex) {  
       throw new BeanCreationException(  
             (mbd != null ? mbd.getResourceDescription() : null), beanName, ex.getMessage(), ex);  
    }  
    if (mbd == null || !mbd.isSynthetic()) {  
    //initializeBean第四步
       wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
    }  
  
    return wrappedBean;  
}
```
> 在Spring框架中，`initializeBean` 方法通常用于初始化一个Bean。在这个过程中，可能会调用 `mbd.isSynthetic()` 来判断某个Bean的属性或方法是否是合成的（synthetic）。
> 
> 合成属性或方法是那些由编译器生成的，而不是开发者显式定义的。这些属性或方法通常是为了满足某些特定的语言特性或框架需求而自动添加的。例如，Java中的内部类可以访问外部类的私有成员，这是因为编译器会为内部类生成一个隐式的、私有的、包含对外部类成员的引用的合成字段。
> 
> 在Spring中，`mbd` 是 `org.springframework.beans.factory.config.MergedBeanDefinition` 的缩写，它表示合并后的Bean定义。`MergedBeanDefinition` 包含了Bean的定义信息，包括Bean的类、构造函数参数、属性值等。
> 
> 当Spring创建和初始化一个Bean时，它会检查每个属性或方法是否是合成的。如果 `mbd.isSynthetic()` 返回 `true`，则表示该属性或方法是合成的。这可能会影响到Spring如何处理这个属性或方法。例如，Spring可能不会注入一个合成属性的值，因为合成属性通常不是开发者关心的部分，而是编译器为了实现特定功能而自动添加的。
> 
> 总的来说，`mbd.isSynthetic()` 在 `initializeBean` 方法中用于判断Bean的属性或方法是否是合成的，这有助于Spring更准确地处理Bean的初始化过程。

进入applyBeanPostProcessorsBeforeInitialization方法，容器中存在一个专门的后置处理器ApplicationContextAwareProcessor通过回调ApplicationContextAware接口，在Bean真正的初始化invokeInitMethods之前进行拦截（也不算拦截，就是额外做一些扩展操作）
![[Pasted image 20240212192435.png]]
postProcessBeforeInitialization
```java
@Override  
@Nullable  
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
    if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||  
          bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||  
          bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||  
          bean instanceof ApplicationStartupAware)) {  
       return bean;  
    }  
  
    invokeAwareInterfaces(bean);  
    return bean;  
}
```