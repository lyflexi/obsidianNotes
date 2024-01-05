![[Pasted image 20240101212508.png]]
Bean 生命周期的整个执行过程描述如下：
1. 实例化接口InstantiationAwareBeanPostProcessor 
2. addSingletonFactory，会获取原始对象的AOP早期引用getEarlyBeanReference，并存入三级缓存`singletonFactories`中
3. populateBean设置对象属性
4. 初始化接口BeanPostProcessor
5. DisposableBean销毁接口

# 实例化Instantiation 
实例化接口InstantiationAwareBeanPostProcessor

1. 调用者通过`getBean(beanName)` 请求某一个Bean
	
2. **实例化Before：**如果容器注册了 `org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor` 接口，则在实例化Bean 之前，将调用 postProcessBeforeInstantiation（）方法。根据配置情况调用Bean构造函数 或工厂方法实例化Bean
	
3. **实例化After：如果容器注册了`InstantiationAwareBeanPostProcessor` 接口，那么在实例化Bean之后，调用该接口的postProcessAfterInstantiation（）方法，可在这里对已经实例化的Bean 进行一些操作

# addSingletonFactory

会获取原始对象的AOP早期引用getEarlyBeanReference，并存入三级缓存`singletonFactories`中
# populateBean设置对象属性：
根据传入的`RootBeanDefinition mbd`（xml配置信息和@Configuration配置信息），那么IOC容器在这一步着手将配置值设置到Bean 对应的属性中去。
1. ==支持外部自定义属性注入，控制是否继续给 Bean 设置属性值==。由用户实现扩展点`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`
2. ==注入属性到 PropertyValues 中（按名字装配或按类型装配）==，在这注意：常规的根据 Bean 配置的依赖注入方式完成注入，resolvedAutowireMode 默认是0，即不走以下逻辑 autowireByName 或者 autowireByType，只有在xml配置文件中声明了 autowire="byName"或者 autowire="byType" 才会走
3. ==对解析完但未设置的属性进行再处理（InstantiationAwareBeanPostProcessor#postProcessProperties）==
	1. @Autowired属性填充（Autowired注入的也是Bean的属性）：如果Bean中有使用@Autowired注入的属性，则会进行属性填充。前提是要保证属性实例已经存在于Spring容器当中
	2. @Resource属性填充
4. 在populateBean的最后有前置判断条件 pvs != null，所以==只有执行了 autowireByName 或者 autowireByType 的属性注入，才会调用该方法（applyPropertyValues）将PropertyValues中的属性值设置到 BeanWrapper 中==。

话不多说，直接看看 populateBean 方法的源码实现：
```java
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
					//如果用户实现的postProcessBeanDefinitionRegistry返回false，则框架提前结束
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

好，下面我们一一分析其实现逻辑

## 支持外部自定义属性注入

```java
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

```

在这主要是让用户可以自定义属性注入。比如用户实现一个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过 postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。

==但如果postProcessAfterInstantiation调用返回 false，表示不必继续进行依赖注入，框架提前结束，直接返回==

## 注入属性到 PropertyValues 中

在这注意：==常规的根据 Bean 配置的依赖注入方式完成注入，resolvedAutowireMode 默认是0，即不走以下逻辑 autowireByName 或者 autowireByType==。只有在配置文件中声明了 autowire="byName" 才会进行至 autowireByName 方法。
```java
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
		// 根据 BeanType 进行 autowiring 自动装配处理
		// ps: <bean id="A" class="com.mrlsm.spring.A"  autowire="byType"></bean>
		if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}

```

那么我们继续跟进 autowireByName 方法
```java
protected void autowireByName(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	// 获取要注入的非简单类型的属性名称
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		// 检测是否存在与 propertyName 相关的 bean 或 BeanDefinition。
		// 若存在，则调用 BeanFactory.getBean 方法获取 bean 实例
		if (containsBean(propertyName)) {
			// 从容器中获取相应的 bean 实例
			Object bean = getBean(propertyName);
			// 将解析出的 bean 存入到属性值列表 pvs 中
			pvs.add(propertyName, bean);
			// 注册依赖关系
			registerDependentBean(propertyName, beanName);
			// logger
		} 
		// logger
	}
}

```

可以看到该方法主要是获取要注入的非简单类型的属性名称，根据属性名称去容器中获取相应的 bean 实例，然后注册相应的依赖关系。

我们先来看看获取要注入的非简单类型的属性名称 unsatisfiedNonSimpleProperties 方法
```java
protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
	Set<String> result = new TreeSet<>();
	PropertyValues pvs = mbd.getPropertyValues();
	PropertyDescriptor[] pds = bw.getPropertyDescriptors();
	for (PropertyDescriptor pd : pds) {
		if (pd.getWriteMethod() != null && !isExcludedFromDependencyCheck(pd) && !pvs.contains(pd.getName()) &&
				!BeanUtils.isSimpleProperty(pd.getPropertyType())) {
			result.add(pd.getName());
		}
	}
	return StringUtils.toStringArray(result);
}

```

该方法主要是获取非简单类型属性的名称，且该属性未被配置在配置文件中。

那什么是简单类型呢？Spring 认为的 简单类型 属性有哪些，如下：

1. CharSequence 接口的实现类，比如 String
2. Enum
3. Date
4. URI/URL
5. Number 的继承类，比如 Integer/Long
6. byte/short/int... 等基本类型
7. Locale
8. 以上所有类型的数组形式，比如 String[]、Date[]、int[] 等等

除了要求非简单类型的属性外，还要求属性未在配置文件中配置过，也就是 pvs.contains(pd.getName()) = false。 也就是上面源码中 if 中的判断条件。

好的，看完 autowireByName 方法之后，我们在看看根据 BeanType 进行 autowiring 自动装配处理的 autowireByType 方法：

```java
protected void autowireByType(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	// 获取的属性类型转换器
	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}
	// 用来存放解析的要注入的属性名
	Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
	// 获取要注入的属性名称（非简单属性（8种原始类型、字符、URL等都是简单属性））
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		try {
			// 获取指定属性名称的属性 Descriptor(Descriptor用来记载属性的getter setter type等情况)
			PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
			// 不对 Object 类型的属性进行装配注入，即如果属性类型为 Object，则忽略，不做解析
			if (Object.class != pd.getPropertyType()) {
				// 获取属性的 setter 方法
				MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
				// 对于继承了 PriorityOrdered 的 post-processor，不允许立即初始化(热加载)
				boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
				// 创建一个要被注入的依赖描述，方便提供统一的访问
				DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
				// 根据容器的 BeanDefinition 解析依赖关系，返回所有要被注入的 Bean 实例
				Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
				if (autowiredArgument != null) {
					// 将解析出的 bean 存入到属性值列表 pvs 中
					pvs.add(propertyName, autowiredArgument);
				}
				for (String autowiredBeanName : autowiredBeanNames) {
					// 注册依赖关系
					registerDependentBean(autowiredBeanName, beanName);
					if (logger.isTraceEnabled()) {
						logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
								propertyName + "' to bean named '" + autowiredBeanName + "'");
					}
				}
				// 清除已注入属性的记录
				autowiredBeanNames.clear();
			}
		}
		catch (BeansException ex) {
			throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
		}
	}
}

```

该方法要比 autowireByName 方法稍微复杂，复杂在属性对应的 bean 的 name 是不确定的，要去做相应的解析。

该方法的开始也是去获取非简单类型属性的名称，根据名称去创建一个要被注入的依赖描述，方便提供统一的访问，根据容器的 BeanDefinition 解析依赖关系，返回所有要被注入的 Bean 实例，最后都是将解析出的 bean 存入到属性值列表 pvs 中，然后执行 registerDependentBean 方法来注册依赖关系。

## 对解析完但未设置的属性进行再处理

在这里主要是对 @Autowired 标记的属性进行依赖注入，我们看下源码：
```java
// 容器是否注册了 InstantiationAwareBeanPostProcessor
boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
// 是否进行依赖检查，默认为 false, 该判断为了后续是否进行依赖检查。
boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

PropertyDescriptor[] filteredPds = null;
if (hasInstAwareBpps) {
	if (pvs == null) {
		pvs = mbd.getPropertyValues();
	}
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			// 对 @Autowired 标记的属性进行依赖注入
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

```

我们跟进依赖注入的方法 postProcessProperties ，他的实现类为：AutowiredAnnotationBeanPostProcessor
```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
	// 获取指定类中 @Autowired 相关注解的元信息
	InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
	try {
		// 对 Bean 的属性进行自动注入
		metadata.inject(bean, beanName, pvs);
	}
	// catch ···
	return pvs;
}

```

inject 方法我们不陌生，进入发现就是遍历每个字段并进行注入，最终调用 invoke 方法进行反射的处理，我们看看获取指定类中 @Autowired 相关注解的元信息的方法：
```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
	String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
	// 从容器中查找是否有给定类的 autowire 相关注解元信息
	InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
	if (InjectionMetadata.needsRefresh(metadata, clazz)) {
		synchronized (this.injectionMetadataCache) {
			metadata = this.injectionMetadataCache.get(cacheKey);
			if (InjectionMetadata.needsRefresh(metadata, clazz)) {
				if (metadata != null) {
					metadata.clear(pvs);
				}
                // 解析给定类 @Autowired 相关注解元信息
				metadata = buildAutowiringMetadata(clazz);
				// 将得到的给定类 autowire 相关注解元信息存储在容器缓存中
				this.injectionMetadataCache.put(cacheKey, metadata);
			}
		}
	}
	return metadata;
}

```

可以看到该方法主要就是解析给定类 @Autowired 相关注解元信息，从而将得到的给定类 autowire 相关注解元信息存储在容器缓存中。

## 将 PropertyValues 中的属性值设置到 BeanWrapper 中

在这我们可以看到有前置判断条件 pvs != null，所以只有执行了 autowireByName 或者 autowireByType 的属性注入，才会调用该方法（applyPropertyValues），最终将属性注入到 Bean 的 Wrapper 实例里。源码如下：
```java
if (pvs != null) {
	applyPropertyValues(beanName, mbd, bw, pvs);
}

protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
	// 前置判断，属性列表 zise 为 0 直接返回
	if (pvs.isEmpty()) {
		return;
	}
	if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
		// 设置安全上下文，JDK安全机制
		((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
	}

	MutablePropertyValues mpvs = null;
	List<PropertyValue> original;

	if (pvs instanceof MutablePropertyValues) {
		mpvs = (MutablePropertyValues) pvs;
		if (mpvs.isConverted()) {
			// 若属性值已经转换了，则直接赋值
			try {
				bw.setPropertyValues(mpvs);
				return;
			}
			catch (BeansException ex) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Error setting property values", ex);
			}
		}
		// 若没有被转换，先将没转换前的原始类型值给记录下来
		original = mpvs.getPropertyValueList();
	}
	else {
		// 如果 pvs 不是 MutablePropertyValues 类型，则直接使用原始类型
		original = Arrays.asList(pvs.getPropertyValues());
	}
	// 获取用户的自定义类型转换
	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}
	// 创建一个 Bean 定义属性值解析器，将 BeanDefinition 中的属性值解析为 Bean 实例对象的实际值
	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

	// 为属性需要解析的值创建一个副本，将副本的数据注入 Bean 实例
	List<PropertyValue> deepCopy = new ArrayList<>(original.size());
	boolean resolveNecessary = false;
	for (PropertyValue pv : original) {
		// 不需要转换的属性值直接添加
		if (pv.isConverted()) {
			deepCopy.add(pv);
		}
		else {
			String propertyName = pv.getName();
			// 保留转换前的属性值
			Object originalValue = pv.getValue();
			// 如果是被 Autowired 标记的实例，则把 Bean 里面关于 set 此属性的方法给记录下来，包装成 DependencyDescriptor
			if (originalValue == AutowiredPropertyMarker.INSTANCE) {
				Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
				if (writeMethod == null) {
					throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
				}
				originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
			}
			// 转换属性值，如将引用转换为容器中实例化的对象引用
			Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
			Object convertedValue = resolvedValue;
			// 属性值是否可以转换
			boolean convertible = bw.isWritableProperty(propertyName) &&
					!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
			if (convertible) {
				// 使用用户自定义的类型转换器转换属性值
				convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
			}
			// 存储转换后的属性值，避免每次属性注入时的转换工作
			if (resolvedValue == originalValue) {
				if (convertible) {
					pv.setConvertedValue(convertedValue);
				}
				deepCopy.add(pv);
			}
			else if (convertible && originalValue instanceof TypedStringValue &&
					!((TypedStringValue) originalValue).isDynamic() &&
					!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
				pv.setConvertedValue(convertedValue);
				deepCopy.add(pv);
			}
			else {
				resolveNecessary = true;
				deepCopy.add(new PropertyValue(pv, convertedValue));
			}
		}
	}
	if (mpvs != null && !resolveNecessary) {
		// 标记属性已经转换过
		mpvs.setConverted();
	}

	try {
		// 进行属性的依赖注入，通过Java的反射机制根据 set 方法把属性注入到 bean 里。
		bw.setPropertyValues(new MutablePropertyValues(deepCopy));
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Error setting property values", ex);
	}
}

```

集体操作已经在上面标注清楚，主要就是通过进行转换操作，生成最终需要注入的类型的对象，之后调用 setPropertyValues 方法通过Java的反射机制根据 set 方法把属性注入到 bean 里。


# 初始化接口BeanPostProcessor
    
1. **初始化Before：如果BeanFactory 装配了 org.springframework.beans.factory.config.BeanPostProcessor 后处理 ，则将调用`postProcessBeforeInitialization`方法 对Bean 进行加工操作，==`postProcessBeforeInitialization`当中调用了`invokeAwareMethod`方法：使用了Spring Aware 你的Bean将会和Spring框架耦合，Spring Aware 的目的是为了让Bean获取Spring容器的服务。==
	![[Pasted image 20240105114206.png]]
	
	1. 如果 Bean 实现了 BeanNameAware 接口，则 Spring 调用 Bean 的 setBeanName() 方法传入当前 Bean 的 id 值。
		
	2. 如果 Bean 实现了 BeanFactoryAware 接口，则 Spring 调用 setBeanFactory() 方法传入当前工厂实例的引用。
		
	3. 如果 Bean 实现了 ApplicationContextAware 接口，则 Spring 调用 setApplicationContext() 方法传入当前 ApplicationContext 实例的引用。
		
	4. 实现BeanClassLoaderAware接口，调用setBeanClassLoader方法
		
	5. 如果bean实现了ResourceLoaderAware： 获得资源加载器，可以获得外部资源文件的内容；
		
	6. 如果bean实现了MessageSourceAware：获得message source，这也可以获得文本信息
		
	7. 如果bean实现了applicationEventPulisherAware：应用事件发布器，可以发布事件，
		
2. **初始化方法invokeInitMethods，**因为 Aware 方法都是执行在初始化方法之前，所以可以在初始化方法中放心大胆的使用 Aware 接口获取的资源，这也是我们自定义扩展 Spring 的常用方式。如果Bean实现了 InitializingBean 接口：
	
	1. 则将调用接口的afterPropertiesSet 方法。
		
	2. 类似的如果在applicationContext.xml配置文件中通过init-method属性指定了初始化方法，则将执行指定的初始化方法
		
3. **初始化After：**如果BeanFactory 装配了 org.springframework.beans.factory.config.BeanPostProcessor 后处理 ，则将调用`postProcessAfterInitialization(Object bean, String beanName)` 方法，容器再次获得了对Bean 的加工机会。此时bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；  ==此处非常重要，Spring 的 AOP 就是利用after实现的，代理对象的创建时机就位于Bean的初始化之后==
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



