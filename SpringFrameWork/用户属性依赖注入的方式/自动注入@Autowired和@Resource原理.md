来看Spring属性注入的源码片段：
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {...}

	// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the  
	// state of the bean before properties are set. This can be used, for example,  
	// to support styles of field injection.  
	// 属性填充之前，提供一个修改属性的机会
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
	    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {  
	       if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {  
	          return;  
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
# autowireByType源码

在Spring Framework中，`autowireByType` 和 `postProcessProperties` 是两种不同的机制，它们的作用和实现方式也不同。

1. `autowireByType` 是一种自动装配（autowiring）的方式，它允许Spring根据类型自动装配bean之间的依赖关系。当一个bean被标记为需要自动装配时，Spring会尝试寻找与该bean类型匹配的其他bean，并将它们注入到该bean中。这种自动装配的方式基于bean的类型信息，而不是名称。通常，你可以使用`@Autowired`注解来启用`autowireByType`，或者在XML配置中使用`autowire="byType"`。
    
2. `postProcessProperties` 是一个Bean后处理器（InstantiationAwareBeanPostProcessor），它允许在Spring容器实例化bean并将其属性设置之后，对这些属性进行一些定制化的处理。当Spring实例化一个bean并设置其属性后，会调用所有注册的`postProcessProperties`方法，从而允许开发人员对bean的属性进行定制化处理。通常情况下，`postProcessProperties`方法可以用来检查、修改或补充bean的属性信息，以满足特定的需求。
# postProcessProperties源码
postProcessProperties(PropertyValues pvs, Object bean, String beanName) 是扩展点InstantiationAwareBeanPostProcessor的第三个方法：
```java

 public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {  

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
实现`postProcessProperties`的类比较多，这里重点讲两个：
 - AutowiredAnnotationBeanPostProcessor#postProcessProperties：主要注册 ==带有 @Autowired，@Value，@Lookup，@Inject 注解的属性。==
 - CommonAnnotationBeanPostProcessor#postProcessProperties ： ==主要注册带有 @Resource 注解的属性==，流程与AutowiredAnnotationBeanPostProcessor#postProcessProperties`类似

## AutowiredAnnotationBeanPostProcessor#postProcessProperties

@Autowired支持三种注入方式：
- 属性注入（最常用），
- 构造器注入，
- setter注入。

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
![[Pasted image 20240226201445.png]]
将依赖注入信息封装到InjectionMetadata中
![[Pasted image 20240226201450.png]]
而后从InjectionMetadata中遍历出所有的AutowiredElement进行反射注入！InjectionMetadata的两个实现类是AutowiredFieldElement和AutowiredMethodElement
![[Pasted image 20240226201458.png]]
AutowiredMethodElement：
![[Pasted image 20240226201509.png]]

## CommonAnnotationBeanPostProcessor#postProcessProperties

CommonAnnotationBeanPostProcessor#postProcessProperties主要注册带有 @Resource 注解的属性，流程与`AutowiredAnnotationBeanPostProcessor#postProcessProperties`类似

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
# @Autowired和@Resource注解用法对比


@Autowired和@Resource注解都是用来注入Bean的，@Autowired注解是Spring提供的，而@Resource注解是J2EE本身提供的
![[Pasted image 20231226152620.png]]
- @Autowird注解优先通过byType方式注入
- @Resource注解优先通过byName方式注入

## byType


首先有一个接口UserService和两个实现类UserServiceImpl1和UserServiceImpl2，并且这两个实现类已经加入到Spring的IOC容器中了

```Java
@Service
public class UserServiceImpl1 implements UserService
@Service
public class UserServiceImpl2 implements UserService
```

通过@Autowired注入使用

```Java
@Autowired
private UserService userService;
```

根据上面的步骤，可以很容易判断出，直接这么使用是会报错的

原因：首先通过byType注入，判断**UserService**类型有两个实现，无法确定具体是哪一个，

解决方式一：

```Java
// 方式一：改变变量名
@Autowired
private UserService userServiceImpl1;
```

解决方式二：

```Java
// 方式二：配合@Qualifier注解来显式指定name值
@Autowired
@Qualifier(value = "userServiceImpl1")
private UserService userService;
```

另外，使用@Autowired注解注入对象的前提是对象本身在IOC容器中存在，否则需要加上属性`required=false`，表示忽略当前要注入的Bean，如果有直接注入，没有则跳过，不会报错
## byName

@Resource优先通过byName注入，如果没有匹配则通过byType注入

```Java
@Service
public class UserServiceImpl1 implements UserService
@Service
public class UserServiceImpl2 implements UserService
```

通过@Resource注入使用

```Java
@Resource
private UserService userService;
```

通过byName匹配，首字母小写的变量名userService无法匹配IOC容器中任何一个id（这里指的userServiceImpl1和userServiceImpl2）报错

@Resource可以通过参数`name`显式指定所需的对象ID
```Java
// 1. 默认方式：byName，既没指定name属性，也没指定type属性：默认通过byName方式注入，如果byName匹配失败，则使用byType方式注入（也就是上面的那个例子）
@Resource  
private UserService userDao; 
// 2. 指定byName，指定name属性：通过byName方式注入，把name属性和IOC容器中的id去匹配，匹配失败则报错
@Resource(name="userService")  
private UserService userService; 
```

另外，@Resource可以通过参数`type`显式指定对象类型，以达到byType同样的注入效果
```java
// 1. 指定byType，指定type属性：通过byType方式注入，在IOC容器中匹配对应的类型，如果匹配不到或者匹配到多个则报错
@Resource(type=UserService.class)  
private UserService userService; 
// 2. 指定byName和byType，同时指定name属性和type属性：在IOC容器中匹配，名字和类型同时匹配则成功，否则失败
@Resource(name="userService",type=UserService.class)  
private UserService userService;
```