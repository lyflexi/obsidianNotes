
除了Bean生命周期的4+1个主要过程之外，下面讲Spring在Bean的整个生命周期中穿插的两个重要的扩展点（Bean后置处理器接口）
- InstantiationAwareBeanPostProcessor
- BeanPostProcessor
其中InstantiationAwareBeanPostProcessor是作用于Bean实例化阶段的前后
```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {  
  
    @Nullable  
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {  
       return null;  
    }  
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {  
       return true;  
    }  
    //postProcessPropertyValues已废弃，现在都是`postProcessProperties`
    @Nullable  
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)  
          throws BeansException {  
  
       return pvs;  
    } 
}
```
其中BeanPostProcessor是作用于Bean初始化阶段的前后。
```java
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
这些接口的实现类是独立于 Bean 的，并且会注册到 Spring 容器中。在 Spring 容器创建任何 Bean 的时候，这些后处理器都会发生作用。
![[Pasted image 20240101213534.png]]

# 一、InstantiationAwareBeanPostProcessor

## postProcessBeforeInstantiation 调用点

`postProcessBeforeInstantiation(Class<?> beanClass, String beanName)`在Bean实例化之前调用。

在进入doCreateBean方法之前，如果`postProcessBeforeInstantiation`的返回值如果不为null，那么后续的Bean的创建流程[实例化、属性填充，初始化]都不会执行，而是调用`postProcessAfterInitialization`之后直接使用返回的快捷Bean。
![[Pasted image 20231228214151.png]]

apply的意思是遍历调用，因此
1. 遍历调用postProcessBeforeInstantiation
2. 遍历调用postProcessAfterInitialization：此方法自然是Bean 初始化完毕之后的最后调用的方法，在这里调用是为了和之后的正常加载的 bean 保持一致性，因为这里要直接返回Bean以后再也没有处理的机会了
![[Pasted image 20231228214213.png]]

## postProcessAfterInstantiation 调用点

进入`populateBean`方法内部，`postProcessAfterInstantiation` 的调用时机在Bean实例化之后，执行populateBean之前调用

如果用户重写`postProcessAfterInstantiation` 返回`false`，那么后续Bean的属性填充populateBean便不会执行，但是Bean初始化仍然会执行。
```Java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	···
	// 给 InstantiationAwareBeanPostProcessors 最后一次机会在属性注入前修改 Bean 的属性值
	// 主要是让用户可以自定义属性注入。比如用户实现一个 InstantiationAwareBeanPostProcessor 类型的后置处理器，
	// 并通过 postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。

	//也可以控制是否继续填充 Bean
	//具体通过调用 postProcessAfterInstantiation 方法，如果调用返回 false,表示不必继续进行依赖注入，直接返回
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

## postProcessProperties调用时机（只负责注解注入，不负责xml注入）

在Spring Framework中，`autowireByType` 和 `postProcessProperties` 是两种不同的机制，它们的作用和实现方式也不同。

1. `autowireByType` 是一种自动装配（autowiring）的方式，它允许Spring根据类型自动装配bean之间的依赖关系。当一个bean被标记为需要自动装配时，Spring会尝试寻找与该bean类型匹配的其他bean，并将它们注入到该bean中。这种自动装配的方式基于bean的类型信息，而不是名称。通常，你可以使用`@Autowired`注解来启用`autowireByType`，或者在XML配置中使用`autowire="byType"`。
    
2. `postProcessProperties` 是一个Bean后处理器（InstantiationAwareBeanPostProcessor），它允许在Spring容器实例化bean并将其属性设置之后，对这些属性进行一些定制化的处理。当Spring实例化一个bean并设置其属性后，会调用所有注册的`postProcessProperties`方法，从而允许开发人员对bean的属性进行定制化处理。通常情况下，`postProcessProperties`方法可以用来检查、修改或补充bean的属性信息，以满足特定的需求。

```Java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	···

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
				// 当Spring实例化一个bean并@Autowired设置其属性后，会调用所有注册的`postProcessProperties`方法，从而允许开发人员对bean的属性进行定制化处理。通常情况下，`postProcessProperties`方法可以用来检查、修改或补充bean的属性信息，以满足特定的需求。
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


在populateBean设置对象属性的过程中，`InstantiationAwareBeanPostProcessor`接口的`postProcessProperties`方法用于@Autowired正常填充属性之后，对
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
AutowiredMethodElement
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

# 二、BeanPostProcessor

进入`initializeBean`内部
![[Pasted image 20231228214542.png]]

## postProcessBeforeInitialization调用时机

1. 首先获取到所有的后置处理器 getBeanPostProcessors()
    
2. 在 for 循环中依次调用后置处理器的方法 processor.postProcessBeforeInitialization(result, beanName);
    
3. 进入 postProcessBeforeInitialization 方法
    
![[Pasted image 20231228214547.png]]

查看`postProcessBeforeInitialization`实现，进入`ApplicationContextAwareProcessor`

首先判断此 bean 是不是实现了各种的Aware

1. 如果bean 实现了各种的Aware，就向容器中导入相关的上下文环境，目的是为了 Bean 实例能够获取到相关的上下文。
    
2. 如果bean 没有实现各种Aware，就直接返回Bean。
![[Pasted image 20231228214558.png]]
使用了Spring Aware 你的Bean将会和Spring框架耦合，Spring Aware 的目的是为了让Bean获取Spring容器的服务。

- 如果Bean 实现了 BeanNameAware 接口 则将调用 setBeanName 接口方法，将配置文件中该Bean 的对应名称设置到Bean 中去
- 实现BeanClassLoaderAware接口，调用setBeanClassLoader方法
- ==如果Bean 实现了 BeanFactoryAware 接口 则将调用 setBeanFactory 接口方法，将BeanFactory容器实例设置到Bean 中去，将BeanFactory实例传进来；==
- ==如果bean实现了ApplicationContextAware接口，它的setApplicationContext()方法将被调用，将应用上下文的引用传入到bean中；==
- 如果bean实现了MessageSourceAware：获得message source，这也可以获得文本信息
- 如果bean实现了applicationEventPulisherAware：应用事件发布器，可以发布事件，
- 如果bean实现了ResourceLoaderAware： 获得资源加载器，可以获得外部资源文件的内容；
    

## postProcessAfterInitialization调用时机

Bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；

如果BeanFactory 装配了 org.springframework.beans.factory.config.BeanPostProcessor 后处理 ，则将调用`postProcessAfterInitialization(Object bean, String beanName)` 方法，容器再次获得了对Bean 的加工机会。此时bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；  ==此处非常重要，Spring 的 AOP 就是利用after实现的，代理对象的创建时机就位于Bean的初始化之后==
