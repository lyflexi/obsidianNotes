我们知道对于普通的 Java 对象来说，它们的生命周期就是：

- 实例化
    
- 该对象不再被使用时通过垃圾回收机制进行回收
    

而对于 Spring Bean 的生命周期来说：

- 实例化 Instantiation
    
- 属性赋值 Populate
    
- 初始化 Initialization
    
- 销毁 Destruction
    

# 源码定位

SpringBean的创建入口是doCreate()方法方法，大家跟我来找找

## AbstractApplicationContext

首先来到我们最熟悉的`AbstractApplicationContext#refresh`方法，IOC容器刷新，即初始化IOC容器。
![[Pasted image 20231228213943.png]]

在初始化IOC容器的最后，来了一句`finishBeanFactoryInitialization`，也就是初始化剩下的单实例Bean，我们进入方法查看，最主要的调用就是从IOC容器中获取Bean：`beanFactory.getBean`
![[Pasted image 20231228213952.png]]

## AbstractBeanFactory#getBean！！！
![[Pasted image 20231228213958.png]]

在doGetBean中有就会有创建Bean的调用了，`createBean`，但是`createBean`方法在当前的抽象类AbstractBeanFactory当中是没有实现的，AbstractBeanFactory把`createBean`的方法实现交给了子类`AbstractAutowireCapableBeanFactory`
![[Pasted image 20231228214005.png]]

## AbstractAutowireCapableBeanFactory

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
          isSingletonCurrentlyInCreation(beanName));
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

是不是很清爽了？至于 InstantiationAwareBeanPostProcessor、BeanPostProcessor、以及其他的类，在我看来只不过是IOC容器对主流程四个步骤的一系列扩展点而已。其中

1. InstantiationAwareBeanPostProcessor是作用于Bean实例化阶段的前后
    
2. BeanPostProcessor是作用于Bean初始化阶段的前后。
    

这些接口的实现类是独立于 Bean 的，并且会注册到 Spring 容器中。在 Spring 容器创建任何 Bean 的时候，这些后处理器都会发生作用。

![[Pasted image 20240101213534.png]]
# 扩展点分析

## **一、InstantiationAwareBeanPostProcessor**

我们翻一下源码发现 InstantiationAwareBeanPostProcessor 是继承了 BeanPostProcessor，并且新增了三个方法

- `postProcessBeforeInstantiation`
    
- `postProcessAfterInstantiation`
    
- `postProcessProperties`，（postProcessPropertyValues已废弃，现在都是`postProcessProperties`）
    

```Java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
       return null;
    }
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
       return true;
    }
    @Nullable
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
          throws BeansException {

       return pvs;
    }
}
```

### `postProcessBeforeInstantiation` 调用点

`postProcessBeforeInstantiation(Class<?> beanClass, String beanName)`在实例化之前调用。

但如果`postProcessBeforeInstantiation`的返回值如果不为null，那么后续的Bean的创建流程【实例化、初始化afterProperties】都不会执行，而是调用`postProcessAfterInitialization`之后直接使用返回的快捷Bean。
![[Pasted image 20231228214151.png]]

apply的意思是遍历调用，因此

1. 遍历调用postProcessBeforeInstantiation
    
2. 遍历调用postProcessAfterInitialization
![[Pasted image 20231228214213.png]]

### postProcessAfterInstantiation 调用点

进入`populateBean`方法内部，`postProcessAfterInstantiation` 的调用时机在：

1. 正常情况下在实例化之后
    
2. 在执行populateBean之前调用
    

如果有指定的bean的时候`postProcessAfterInstantiation` 一定返回`false`

- 那么后续的属性填充和属性依赖注入【populateBean】将不会执行，即postProcessProperties将不会执行
    
- 但是初始化和BeanPostProcessor的仍然会执行。
    

```Java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	···
	// 给 InstantiationAwareBeanPostProcessors 最后一次机会在属性注入前修改 Bean 的属性值，也可以控制是否继续填充 Bean
	// 具体通过调用 postProcessAfterInstantiation 方法，如果调用返回 false,表示不必继续进行依赖注入，直接返回
	// 主要是让用户可以自定义属性注入。比如用户实现一个 InstantiationAwareBeanPostProcessor 类型的后置处理器，
	// 并通过 postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					return;
				}
			}
		}
	}
	// pvs 是一个 MutablePropertyValues 实例，里面实现了 PropertyValues 接口，
	// 提供属性的读写操作实现，同时可以通过调用构造函数实现深拷贝
	// 获取 BeanDefinition 里面为 Bean 设置上的属性值
	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
	// 根据 Bean 配置的依赖注入方式完成注入，默认是0，即不走以下逻辑，所有的依赖注入都需要在 xml 文件中有显式的配置
	// 如果设置了相关的依赖装配方式，会遍历 Bean 中的属性，根据类型或名称来完成相应注入，无需额外配置
	int resolvedAutowireMode = mbd.getResolvedAutowireMode();
	if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// 根据 beanName 进行 autowiring 自动装配处理
		// ps: <bean id="A" class="com.mrlsm.spring.A"  autowire="byName"></bean>
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// 根据 Bean 的类型进行 autowiring 自动装配处理
		// ps: <bean id="A" class="com.mrlsm.spring.A"  autowire="byType"></bean>
		if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}
	// 容器是否注册了 InstantiationAwareBeanPostProcessor
	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
	// 是否进行依赖检查，默认为 false
	boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

	PropertyDescriptor[] filteredPds = null;
	if (hasInstAwareBpps) {
		if (pvs == null) {
			pvs = mbd.getPropertyValues();
		}
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				// 在这里会对 @Autowired 标记的属性进行依赖注入
				PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
				if (pvsToUse == null) {
					if (filteredPds == null) {
						filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
					}
					// 对解析完但未设置的属性再进行处理
					pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						return;
					}
				}
				pvs = pvsToUse;
			}
		}
	}
	// 依赖检查，对应 depend-on 属性，3.0已经弃用此属性
	if (needsDepCheck) {
		// 过滤出所有需要进行依赖检查的属性编辑器
		if (filteredPds == null) {
			filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		}
		checkDependencies(beanName, mbd, filteredPds, pvs);
	}

	if (pvs != null) {
		// 最终将属性注入到 Bean 的 Wrapper 实例里，这里的注入主要是供显式配置了 autowiredbyName 或者 ByType 的属性注入，
		// 针对注解来讲，由于在 AutowiredAnnotationBeanPostProcessor 已经完成了注入，
		// 所以此处不执行
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}

```

### postProcessProperties调用时机（只负责注解注入，不负责xml注入）

在populateBean设置对象属性的过程中，`InstantiationAwareBeanPostProcessor`接口的`postProcessProperties`方法用于填充自动注入注解注入的属性，前提是要保证注入的实例已经存在于Spring容器当中。
实现`postProcessProperties`的类比较多，这里重点讲两个：
- `AutowiredAnnotationBeanPostProcessor#postProcessProperties`：主要注册 带有 @Autowired，@Value，@Lookup，@Inject 注解的属性。==其中@Autowired支持属性注入（最常用），构造器注入，setter注入。==
```Java
public class ClassA {
    //@Autowired属性注入（最常用），此种方式注入的字段classB无法在ClassA外访问
    @Autowired
    private ClassB classB;
    //@Autowired构造器注入，通过构造器注入可以解决注入对象的外部可见性的问题
    private ClassC classC;
    @Autowired
    public ClassA (ClassC classC) {
      this.classC= classC;
    }
    //@Autowired的setter注入
    private ClassD classD;
    @Autowired
    public setClassD (ClassD  classD) {
      this.classD= classD;
    }
 }
  
```

在`AutowiredAnnotationBeanPostProcessor`构造器中，初始化了对@Autowired，@Value，@Lookup，@Inject 注解的支持
![[Pasted image 20231228214423.png]]

将依赖注入信息封装到InjectionMetadata中
![[Pasted image 20231228214431.png]]

而后从InjectionMetadata中遍历出所有的AutowiredElement进行反射注入！

AutowiredFieldElement和AutowiredMethodElement都属于AutowiredElement
![[Pasted image 20231228214436.png]]
![[Pasted image 20231228214443.png]]

- `CommonAnnotationBeanPostProcessor#postProcessProperties` ： 主要注册带有 @Resource 注解的属性，流程与`AutowiredAnnotationBeanPostProcessor#postProcessProperties`类似，只不过 @Resource
    - 注入由@EJB 和@Resource修饰的属性
    - 注入由@EJB 和@Resource修饰的方法

```Java
private InjectionMetadata buildResourceMetadata(Class<?> clazz) {
    if (!AnnotationUtils.isCandidateClass(clazz, resourceAnnotationTypes)) {
       return InjectionMetadata.EMPTY;
    }

    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
       final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
        //注入由@EJB 和@Resource修饰的属性 
       ReflectionUtils.doWithLocalFields(targetClass, field -> {
          if (ejbAnnotationType != null && field.isAnnotationPresent(ejbAnnotationType)) {
             if (Modifier.isStatic(field.getModifiers())) {
                throw new IllegalStateException("@EJB annotation is not supported on static fields");
             }
             currElements.add(new EjbRefElement(field, field, null));
          }
          else if (jakartaResourceType != null && field.isAnnotationPresent(jakartaResourceType)) {
             if (Modifier.isStatic(field.getModifiers())) {
                throw new IllegalStateException("@Resource annotation is not supported on static fields");
             }
             if (!this.ignoredResourceTypes.contains(field.getType().getName())) {
                currElements.add(new ResourceElement(field, field, null));
             }
          }
          else if (javaxResourceType != null && field.isAnnotationPresent(javaxResourceType)) {
             if (Modifier.isStatic(field.getModifiers())) {
                throw new IllegalStateException("@Resource annotation is not supported on static fields");
             }
             if (!this.ignoredResourceTypes.contains(field.getType().getName())) {
                currElements.add(new LegacyResourceElement(field, field, null));
             }
          }
       });
        //注入由@EJB 和@Resource修饰的方法 
       ReflectionUtils.doWithLocalMethods(targetClass, method -> {
          Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
          if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
             return;
          }
          if (method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
             if (ejbAnnotationType != null && bridgedMethod.isAnnotationPresent(ejbAnnotationType)) {
                if (Modifier.isStatic(method.getModifiers())) {
                   throw new IllegalStateException("@EJB annotation is not supported on static methods");
                }
                if (method.getParameterCount() != 1) {
                   throw new IllegalStateException("@EJB annotation requires a single-arg method: " + method);
                }
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                currElements.add(new EjbRefElement(method, bridgedMethod, pd));
             }
             else if (jakartaResourceType != null && bridgedMethod.isAnnotationPresent(jakartaResourceType)) {
                if (Modifier.isStatic(method.getModifiers())) {
                   throw new IllegalStateException("@Resource annotation is not supported on static methods");
                }
                Class<?>[] paramTypes = method.getParameterTypes();
                if (paramTypes.length != 1) {
                   throw new IllegalStateException("@Resource annotation requires a single-arg method: " + method);
                }
                if (!this.ignoredResourceTypes.contains(paramTypes[0].getName())) {
                   PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                   currElements.add(new ResourceElement(method, bridgedMethod, pd));
                }
             }
             else if (javaxResourceType != null && bridgedMethod.isAnnotationPresent(javaxResourceType)) {
                if (Modifier.isStatic(method.getModifiers())) {
                   throw new IllegalStateException("@Resource annotation is not supported on static methods");
                }
                Class<?>[] paramTypes = method.getParameterTypes();
                if (paramTypes.length != 1) {
                   throw new IllegalStateException("@Resource annotation requires a single-arg method: " + method);
                }
                if (!this.ignoredResourceTypes.contains(paramTypes[0].getName())) {
                   PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                   currElements.add(new LegacyResourceElement(method, bridgedMethod, pd));
                }
             }
          }
       });

       elements.addAll(0, currElements);
       targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return InjectionMetadata.forElements(elements, clazz);
}
```

## 二、BeanPostProcessor

BeanPostProcessor 中有两个方法：

1. postProcessBeforeInitialization
    
2. postProcessAfterInitialization
    

```Java
public interface BeanPostProcessor {

    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
       return bean;
    }
    
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
       return bean;
    }

}
```

进入`initializeBean`内部
![[Pasted image 20231228214542.png]]

### postProcessBeforeInitialization调用时机

1. 首先获取到所有的后置处理器 getBeanPostProcessors()
    
2. 在 for 循环中依次调用后置处理器的方法 processor.postProcessBeforeInitialization(result, beanName);
    
3. 进入 postProcessBeforeInitialization 方法
    
![[Pasted image 20231228214547.png]]

查看`postProcessBeforeInitialization`实现，进入`ApplicationContextAwareProcessor`

首先判断此 bean 是不是实现了各种的Aware

1. 如果bean 实现了各种的Aware，就向容器中导入相关的上下文环境，目的是为了 Bean 实例能够获取到相关的上下文。
    
2. 如果bean 没有实现各种Aware，就直接返回Bean。
![[Pasted image 20231228214558.png]]
使用了Spring Aware 你的Bean将会和Spring框架耦合，==Spring Aware 的目的是为了让Bean获取Spring容器的服务。==

- 如果Bean 实现了 BeanNameAware 接口 则将调用 setBeanName 接口方法，将配置文件中该Bean 的对应名称设置到Bean 中去
    
- 实现BeanClassLoaderAware接口，调用setBeanClassLoader方法
    
- 如果Bean 实现了 BeanFactoryAware 接口 则将调用 setBeanFactory 接口方法，将BeanFactory容器实例设置到Bean 中去，将BeanFactory实例传进来；
    
- 如果bean实现了ApplicationContextAware接口，它的setApplicationContext()方法将被调用，将应用上下文的引用传入到bean中；
    
- 如果bean实现了MessageSourceAware：获得message source，这也可以获得文本信息
    
- 如果bean实现了applicationEventPulisherAware：应用事件发布器，可以发布事件，
    
- 如果bean实现了ResourceLoaderAware： 获得资源加载器，可以获得外部资源文件的内容；
    

### postProcessAfterInitialization调用时机

再次获得对Bean 的加工机会。

此时Bean的初始化方法`invokeInitMethods`已经执行完毕，Bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；
