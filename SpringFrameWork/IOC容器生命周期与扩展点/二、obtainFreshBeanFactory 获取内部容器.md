# obtainFreshBeanFactory 获取BeanFactory
看注释，翻译过来是“告诉子类去刷新内部的BeanFactory”
当前refresh方法位于AbstractApplicationContext，为什么AbstractApplicationContext的子类能够刷新内部的BeanFactory呢？
带着这个问题往下分析

![[Pasted image 20231227170607.png]]

## refreshBeanFactory()

发现首先调用了一个叫refreshBeanFactory的方法，该方法见名思义，应该是来刷新BeanFactory的。那么，该方法里面又做了哪些事呢？
```java
/**  
 * Tell the subclass to refresh the internal bean factory. * @return the fresh BeanFactory instance  
 * @see #refreshBeanFactory()  
 * @see #getBeanFactory()  
 */protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {  
    refreshBeanFactory();  
    return getBeanFactory();  
}
```
在 refreshBeanFactory();  打上断点，发现程序来到了==子类`GenericApplicationContext`里面==
1. 先判断是不是重复刷新了
2. 然后调用this.beanFactory.setSerializationId(getId());
### 判断容器是否被重复刷新
```java
if (!this.refreshed.compareAndSet(false, true)) {  
    throw new IllegalStateException(  
          "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");  
}
```

### 给内部容器设置序列号this.beanFactory.setSerializationId(getId());
程序运行到这里，有一个大大的疑问，那就是我们的beanFactory不是还没创建么，怎么在这儿又开始调用方法了呢，难道是已经创建了吗？
![[Pasted image 20231227171108.png]]

==这就是我们说的获取内部BeanFactory，虽然用户的BeanFactory还没创建，但是Spring内部自带的BeanFactory已经在此之前被创建了==

我们向上翻阅GenericApplicationContext类的代码，发现原来是在这个类的无参构造方法里面，就已经实例化了beanFactory这个对象。也就是说，在创建GenericApplicationContext对象时，无参构造器里面就new出来了beanFactory这个对象，而且该对象还是DefaultListableBeanFactory类型的。
```java
/**  
 * Create a new GenericApplicationContext. * @see #registerBeanDefinition  
 * @see #refresh  
 */public GenericApplicationContext() {  
    this.beanFactory = new DefaultListableBeanFactory();  
}
```

既然如此，那么究竟是在什么地方调用了GenericApplicationContext类的无参构造方法的呢？

其实我们的单元测试类（例如IOCTest_Ext）中使用的AnnotationConfigApplicationContext又是GenericApplicationContext的子类
![[Pasted image 20231227171947.png]]
AnnotationConfigApplicationContext的有参构造器中有一句关键代码this()
```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {  
    this();  
    register(componentClasses);  
    refresh();  
}
```
this()进一步调用了自己的无参构造器：
```java
public AnnotationConfigApplicationContext() {  
    StartupStep createAnnotatedBeanDefReader = getApplicationStartup().start("spring.context.annotated-bean-reader.create");  
    this.reader = new AnnotatedBeanDefinitionReader(this);  
    createAnnotatedBeanDefReader.end();  
    this.scanner = new ClassPathBeanDefinitionScanner(this);  
}
```
所以，当我们实例化AnnotationConfigApplicationContext时
1. 首先调用AnnotationConfigApplicationContext的有参构造`AnnotationConfigApplicationContext(Class<?>... componentClasses)`
2. this()调用AnnotationConfigApplicationContext的无参构造
3. ==AnnotationConfigApplicationContext的无参构造默认会触发其父类的构造方法（这个是Java本身的特性）==，相应地这时我们的==内部BeanFactory即GenericApplicationContext就也实例化了。==
## getBeanFactory()

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {  
    refreshBeanFactory();  
    return getBeanFactory();  
}
```
进入GenericApplicationContext的getBeanFactory()方法：

```java
/**  
 * Return the single internal BeanFactory held by this context * (as ConfigurableListableBeanFactory). */@Override  
public final ConfigurableListableBeanFactory getBeanFactory() {  
    return this.beanFactory;  
}
```
翻译上述代码，返回由GenericApplicationContext持有的内部BeanFactory（DefaultListableBeanFactory）作为ConfigurableListableBeanFactory
![[Pasted image 20231227213604.png]]
