![[Pasted image 20240101212508.png]]
Bean 生命周期的整个执行过程描述如下：
1. 实例化接口InstantiationAwareBeanPostProcessor 
2. addSingletonFactory，会获取原始对象的AOP早期引用getEarlyBeanReference，并存入三级缓存`singletonFactories`中，此时还未进行属性填充，用于解决循环依赖
3. populateBean设置对象属性
4. 初始化接口BeanPostProcessor
5. DisposableBean销毁接口

# 实例化Instantiation
实例化接口InstantiationAwareBeanPostProcessor

1. 调用者通过`getBean(beanName)` 请求某一个Bean
2. 实例化Before：如果容器注册了 `org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor` 接口，则在实例化Bean 之前，将调用 postProcessBeforeInstantiation（）方法。根据配置情况调用Bean构造函数 或工厂方法实例化Bean
3. 实例化After：如果容器注册了`InstantiationAwareBeanPostProcessor` 接口，那么在实例化Bean之后，调用该接口的postProcessAfterInstantiation（）方法，可在这里对已经实例化的Bean 进行一些操作

# addSingletonFactory

会获取原始对象的AOP早期引用getEarlyBeanReference，并存入三级缓存`singletonFactories`中，用于解决循环依赖
# populateBean设置对象属性：
根据传入的`RootBeanDefinition mbd`（xml配置信息和@Configuration配置信息），那么IOC容器在这一步着手将配置值设置到Bean 对应的属性中去。
1. 由用户实现扩展点`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`返回false来控制是否继续给 Bean 设置属性值。或者返回true享受最后一次机会在属性注入前修改 Bean 的属性值
2. ==创建PropertyValues（pvs）并注入属性到 pvs 中（按名字装配或按类型装配），autowireByName/autowireByType==
3. autowireByName/autowireByType之后，使用AutowiredAnnotationBeanPostProcessor#postProcessProperties检查、修改或补充bean的属性信息，以满足特定的需求。实现`postProcessProperties`的类比较多，这里列举两个：
	 - AutowiredAnnotationBeanPostProcessor#postProcessProperties：主要注册带有 @Autowired，@Value，@Lookup，@Inject 注解的属性
	 - CommonAnnotationBeanPostProcessor#postProcessProperties 主要注册带有 @Resource 注解的属性，流程与AutowiredAnnotationBeanPostProcessor#postProcessProperties类似
4. 在populateBean的最后有前置判断条件 pvs != null，执行applyPropertyValues

话不多说，直接看看 populateBean 方法的源码实现：
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	···
	// 给 InstantiationAwareBeanPostProcessors 最后一次机会在属性注入前修改 Bean 的属性值，
	// 主要是让用户可以自定义属性注入。比如用户实现一个 InstantiationAwareBeanPostProcessor 类型的后置处理器，
	// 并通过 postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。

	//也可以控制是否继续填充 Bean
	//具体是用户重写postProcessAfterInstantiation 方法返回 false,表示不必继续进行依赖注入，直接返回
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					//如果用户实现的postProcessBeanDefinitionRegistry返回false，则框架提前结束
					return;
				}
			}
		}
	}
    //属性值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);  
  
	int resolvedAutowireMode = mbd.getResolvedAutowireMode();  
	// 属性填充，根据@Autowired注入属性
	if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {  
	    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);  
	    // Add property values based on autowire by name if applicable.  
	    if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {  
	       autowireByName(beanName, mbd, bw, newPvs);  
	    }  
	    // Add property values based on autowire by type if applicable.  
	    if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {  
	       autowireByType(beanName, mbd, bw, newPvs);  
	    }  
	    pvs = newPvs;  
	}  
	// 扩展点：属性填充之后，根据AutowiredAnnotationBeanPostProcessor检查、修改或补充bean的属性信息，以满足特定的需求。
	if (hasInstantiationAwareBeanPostProcessors()) {  
	    if (pvs == null) {  
	       pvs = mbd.getPropertyValues();  
	    }  
	    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
			//默认提供的实现里会调用AutowiredAnnotationBeanPostProcessor的postProcessProperties()方法直接给对象中的属性赋值
			//	AutowiredAnnotationBeanPostProcessor会处理@Autowired、@Value、 @Inject 注解
			//	CommonAnnotationBeanPostProcessor会处理@Resource注解
	       PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);  
	       if (pvsToUse == null) {  
	          return;  
	       }  
	       pvs = pvsToUse;  
	    }  
	}


    //依赖项检查，确保所有公开的属性都已设置
    //依赖项检查可以是对象引用、“简单”属性或所有(两者都是)
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        //3.属性赋值（把pvs里的属性值给bean进行赋值!!!!!）
        //autowire自动注入，其实只是收集注入的值，放到pvs中，最终赋值的动作还是在这里！！
        //PropertyValues中的值优先级最高，如果当前Bean中的BeanDefinition中设置了PropertyValues，那么最终将是PropertyValues中的值，覆盖@Autowired的
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}

```

# 初始化invokeInitMethods
    
1. 初始化Before：如果BeanFactory 装配了 org.springframework.beans.factory.config.BeanPostProcessor 后处理 ，则将调用`postProcessBeforeInitialization`方法 对Bean 进行加工操作，==`postProcessBeforeInitialization`当中调用了`invokeAwareMethod`方法，这意味着使用了Spring Aware 你的Bean将会和Spring框架耦合，Spring Aware 的目的是为了让Bean获取Spring容器的服务。==
	![[Pasted image 20240105114206.png]]
	
	1. 如果 Bean 实现了 BeanNameAware 接口，则 Spring 调用 Bean 的 setBeanName() 方法传入当前 Bean 的 id 值。
	2. 如果 Bean 实现了 BeanFactoryAware 接口，则 Spring 调用 setBeanFactory() 方法传入当前工厂实例的引用。
	3. 如果 Bean 实现了 ApplicationContextAware 接口，则 Spring 调用 setApplicationContext() 方法传入当前 ApplicationContext 实例的引用。
	4. 实现BeanClassLoaderAware接口，调用setBeanClassLoader方法
	5. 如果bean实现了ResourceLoaderAware： 获得资源加载器，可以获得外部资源文件的内容；
	6. 如果bean实现了MessageSourceAware：获得message source，这也可以获得文本信息
	7. 如果bean实现了applicationEventPulisherAware：应用事件发布器，可以发布事件，
		
2. 初始化方法invokeInitMethods：
	因为 Aware 方法都是执行在初始化方法之前，所以可以在初始化方法中放心大胆的使用 Aware 接口获取的资源，这也是我们自定义扩展 Spring 的常用方式。
	1. 如果Bean实现了 InitializingBean 接口，则将调用接口的afterPropertiesSet 方法。
	2. 类似的如果在applicationContext.xml配置文件中通过init-method属性指定了初始化方法，则将执行指定的初始化方法
	
3. 初始化After：如果BeanFactory 装配了 org.springframework.beans.factory.config.BeanPostProcessor 后处理 ，则将调用`postProcessAfterInitialization(Object bean, String beanName)` 方法，容器再次获得了对Bean 的加工机会。此时bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；  ==此处非常重要，Spring 的 AOP 就是利用after实现的，代理对象的创建时机就位于Bean的初始化之后==
# DisposableBean销毁接口

DisposableBean 类似于 InitializingBean，对应生命周期的销毁阶段，以ConfigurableApplicationContext#close()方法作为入口，
1. 若bean实现了DisposableBean接口，spring将调用它的distroy()接口方法。
2. 同样的，如果bean使用了destroy-method属性声明了销毁方法，则该方法被调用；

Bean的生命周期到此结束啦！！！


---

如果在`<Bean>` 中使用Scope指定了作用范围
1. scope = "prototype" 则将Bean返回给调用者， 由调用者管理Bean 的生命周期，此时Bean是多实例的，多实例Bean在使用getBean获取的时候才创建对象
	
2. 如果 scope = "singleton" ,则将Bean放入 IOC容器的缓存池中，由Spring管理Bean的生命周期，此时Bean都是单实例的，对于单实例Bean对象来说，在Spring容器创建完成后就会对单实例Bean进行实例化

> Spring中的bean默认都是单例singleton，单例模式默认会在容器初始化的过程中，解析xml或注解，创建出所有的单例bean，这样的机制在bean比较少时问题不大，但一旦bean非常多时，spring需要在启动的过程中花费大量的时间来创建bean ，花费大量的空间存储bean，但这些bean可能很久都用不上，这种在启动时在时间和空间上的浪费显得非常的不值得。
> 
> 因此，Spring提供了懒加载机制。所谓的懒加载机制就是可以指定bean不在启动时立即创建，而是在后续第一次用到时才创建，从而减轻在启动过程中对时间和内存的消耗。懒加载机制只对单例bean有作用，对于多例bean设置懒加载没有意义，因为多例bean本来就是在用户使用时才创建的，而且Spring不负责多实例的生命周期。
> 
> 多例bean在用户每次使用时，Spring会每次都重新生成一个新的对象给请求方，虽然这种类型的对象的实例化以及属性设置等工作都是由容器负责的，但是只要准备完毕，并且对象实例返回给用户之后，容器就不在拥有当前对象的引用，用户需要自己负责当前对象后继生命周期的管理工作，包括该对象的销毁。也就是说，容器每次返回请求方该对象的一个新的实例之后，就由这个对象“自生自灭”。



