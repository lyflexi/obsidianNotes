我们知道对于普通的 Java 对象来说，它们的生命周期就是：

- 实例化
    
- 该对象不再被使用时通过垃圾回收机制进行回收
    

而对于 Spring Bean 的生命周期来说：

- 实例化 Instantiation
- addSingletonFactory，会获取原始对象的AOP早期引用getEarlyBeanReference，并存入三级缓存`singletonFactories`中，此时还未进行属性填充，用于解决循环依赖
- 属性赋值 Populate
- 初始化 Initialization
- 销毁 Destruction



SpringBean的创建入口是doCreate()方法方法，大家跟我来找找

# AbstractApplicationContext

首先来到我们最熟悉的`AbstractApplicationContext#refresh`方法，IOC容器刷新，即初始化IOC容器。
![[Pasted image 20231228213943.png]]

在初始化IOC容器的最后，来了一句finishBeanFactoryInitialization，也就是初始化剩下的单实例Bean，我们进入方法查看，
来到finishBeanFactoryInitialization的最后一行beanFactory.preInstantiateSingletons();
![[Pasted image 20240127184524.png]]
继续跟踪，preInstantiateSingletons终于调用了getBean方法
![[Pasted image 20240127184612.png]]
# AbstractBeanFactory#getBean！！！
![[Pasted image 20231228213958.png]]

在doGetBean中有就会有创建Bean的调用了，`createBean`，但是`createBean`方法在当前的抽象类AbstractBeanFactory当中是没有实现的，AbstractBeanFactory把`createBean`的方法实现交给了子类`AbstractAutowireCapableBeanFactory`
![[Pasted image 20231228214005.png]]

# AbstractAutowireCapableBeanFactory

createBean方法处有一处`doCreateBean`
![[Pasted image 20231228214012.png]]

`doCreateBean`方法如下：
1. instanceWrapper = createBeanInstance(beanName, mbd, args);
2. populateBean(beanName, mbd, instanceWrapper);
3. exposedObject = initializeBean(beanName, exposedObject, mbd);
4. registerDisposableBeanIfNecessary(beanName, bean, mbd);

```Java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
       throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
       instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
       // 实例化阶段
       instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
       mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
       if (!mbd.postProcessed) {
          try {
             applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
          }
          catch (Throwable ex) {
             throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                   "Post-processing of merged bean definition failed", ex);
          }
          mbd.markAsPostProcessed();
       }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
          isSingletonCurrentlyInCreation(beanName));、
          
    //将早期引用（半成品对象）存入三级缓存，用于解决循环依赖
    if (earlySingletonExposure) {
       if (logger.isTraceEnabled()) {
          logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
       }
       addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
       // 属性赋值阶段
       populateBean(beanName, mbd, instanceWrapper);
       
       // 初始化阶段
       exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
       if (ex instanceof BeanCreationException bce && beanName.equals(bce.getBeanName())) {
          throw bce;
       }
       else {
          throw new BeanCreationException(mbd.getResourceDescription(), beanName, ex.getMessage(), ex);
       }
    }

    if (earlySingletonExposure) {
       Object earlySingletonReference = getSingleton(beanName, false);
       if (earlySingletonReference != null) {
          if (exposedObject == bean) {
             exposedObject = earlySingletonReference;
          }
          else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
             String[] dependentBeans = getDependentBeans(beanName);
             Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
             for (String dependentBean : dependentBeans) {
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                   actualDependentBeans.add(dependentBean);
                }
             }
             if (!actualDependentBeans.isEmpty()) {
                throw new BeanCurrentlyInCreationException(beanName,
                      "Bean with name '" + beanName + "' has been injected into other beans [" +
                      StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                      "] in its raw version as part of a circular reference, but has eventually been " +
                      "wrapped. This means that said other beans do not use the final version of the " +
                      "bean. This is often the result of over-eager type matching - consider using " +
                      "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
             }
          }
       }
    }

    // Register bean as disposable.
    try {
       //销毁bean
       registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
       throw new BeanCreationException(
             mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

是不是很清爽了？