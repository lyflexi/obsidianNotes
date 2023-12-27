接下来为使用内部BeanFactory 做准备
![[Pasted image 20231227173529.png]]

prepareBeanFactory(beanFactory)方法用于给内部BeanFactory 做些标准的设置：
==入参beanFactory正是IOC生命周期的上一步obtainFreshBeanFactory()的返回值ConfigurableListableBeanFactory beanFactory==
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {  
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());  
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));  
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));  
  
    // Configure the bean factory with context callbacks.  
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));  
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);  
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);  
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);  
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);  
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);  
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);  
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);  
  
    // BeanFactory interface not registered as resolvable type in a plain factory.  
    // MessageSource registered (and found for autowiring) as a bean.   
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);  
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);  
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);  
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);  
  
    // Register early post-processor for detecting inner beans as ApplicationListeners.  
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));  
  
    // Detect a LoadTimeWeaver and prepare for weaving, if found.  
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {  
       beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));  
       // Set a temporary ClassLoader for type matching.  
       beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));  
    }  
  
    // Register default environment beans.  
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {  
       beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());  
    }  
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {  
       beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());  
    }  
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {  
       beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());  
    }  
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {  
       beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());  
    }  
}
```
1. 为内部的BeanFactory设置类加载器，beanFactory.setBeanClassLoader(getClassLoader());
2. 设置表达式解析器，beanFactory.setBeanExpressionResolver
3. 设置属性编辑器，beanFactory.addPropertyEditorRegistrar
4. 添加BeanPostProcessor(ApplicationContextAwareProcessor),对Spring 中各种Aware接口的支持,在初始化Bean前，调用Bean实现的Aware接口方法。
5. 注册指定的依赖类型和对应的value
	1. beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory); 
	2. beanFactory.registerResolvableDependency(ResourceLoader.class, this);那么在类中自动注入ResourceLoader类型的对象，就会拿到当前IOC容器。
	3. beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	4. beanFactory.registerResolvableDependency(ApplicationContext.class, this);
6. 添加BeanPostProcessor(ApplicationListenerDetector),用于收集实现了ApplicationListener接口的Bean
7. 注入一些其它信息的bean，比如environment、systemProperties等