注解注入的实现是依赖Bean生命周期的第一个扩展点InstantiationAwareBeanPostProcessor来实现的
InstantiationAwareBeanPostProcessor接口有三个方法：
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
与@Autowired和@Resource依赖注入相关的是InstantiationAwareBeanPostProcessor接口的第三个方法：
default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) 
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {...}

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    // 实例化之后，属性填充之前
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {...}

    //属性值，用来通过set方法注入的值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

	...

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
	
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        //@Autowired和@Resource注解注入
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            //默认提供的实现里会调用AutowiredAnnotationBeanPostProcessor的postProcessProperties()方法直接给对象中的属性赋值
            //	AutowiredAnnotationBeanPostProcessor内部并不会处理pvs，直接返回了
            //	AutowiredAnnotationBeanPostProcessor会处理@Autowired、@Value、 @Inject 注解
            //	CommonAnnotationBeanPostProcessor会处理@Resource注解
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                if (filteredPds == null) {
                    //获取筛选后的propertydescriptor，属性描述器集合(排除被忽略的依赖项类型 或 在被忽略的依赖项接口上 定义的属性)
                    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }
                //过时了，在工厂将给定的属性值应用到给定的bean之前，对它们进行后处理。
                //允许检查是否满足了所有依赖项，例如基于bean属性设置器上的“Required”注解。
                //	默认实现RequiredAnnotationBeanPostProcessor中，支持给"setXXX"方法加上@Required注解
                //	标记为“必需的”，简单来说，就是要求pvs属性值中一定要存在 给@Required标记的set方法注入的值  
                pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    return;
                }
            }
            pvs = pvsToUse;
        }
    }

    //依赖项检查，确保所有公开的属性都已设置
    //依赖项检查可以是对象引用、“简单”属性(原语和String)或所有(两者都是)
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        //3.属性赋值（把pvs里的属性值给bean进行赋值!!!!!）
        //autowire自动注入，其实只是收集注入的值，放到pvs中，最终赋值的动作还是在这里！！
        //PropertyValues中的值优先级最高
        //	如果当前Bean中的BeanDefinition中设置了PropertyValues，那么最终将是PropertyValues中的值，覆盖@Autowired的
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}

```
在populateBean设置对象属性的过程中，`InstantiationAwareBeanPostProcessor`接口的`postProcessProperties`方法用于填充属性，前提是要保证注入的实例已经存在于Spring容器当中。实现`postProcessProperties`的类比较多，这里重点讲两个：
 - AutowiredAnnotationBeanPostProcessor#postProcessProperties：主要注册 ==带有 @Autowired，@Value，@Lookup，@Inject 注解的属性。==
 - CommonAnnotationBeanPostProcessor#postProcessProperties ： ==主要注册带有 @Resource 注解的属性==，流程与AutowiredAnnotationBeanPostProcessor#postProcessProperties`类似

# AutowiredAnnotationBeanPostProcessor#postProcessProperties

@Autowired支持属性注入（最常用），构造器注入，setter注入。

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

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MTJjZGM0MTk3ZThjMDgzYjM4MjRiZjc5NGQ2MTJhNWVfd3JwR0lyYVVVaEFqSVJaR1NsY0xoQnRQeFhkVEhrY2tfVG9rZW46RmUxT2JQYXR1bzMxQnd4MHVBSWNhejdtbjZnXzE3MDM3NDcxMzQ6MTcwMzc1MDczNF9WNA)

将依赖注入信息封装到InjectionMetadata中

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=M2U5OTZmOTJiNTYzZTg2MDFjMDg4OTQwYjgwZDg2OTlfMmR3WnBFTzlUVFRueEVFb2liUFNHYzlyM29FTGZYUU1fVG9rZW46STVYRWJpSUhxb3NhWGF4UVFkOGNuYkpXbmlkXzE3MDM3NDcxMzQ6MTcwMzc1MDczNF9WNA)

而后从InjectionMetadata中遍历出所有的AutowiredElement进行反射注入！

AutowiredFieldElement和AutowiredMethodElement都属于AutowiredElement

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MmFjNjUwYWE4YWFkMzZmMWJlNzQ2NTJiYTQ2NGRkNTFfT2dkNTlTV3F1bnZHMlRMUjZObTd6Vnk2UzhzSlJwTjNfVG9rZW46QVVWV2JzYzZub1ZrMTV4RE13YWMyQk5Nbm1oXzE3MDM3NDcxMzQ6MTcwMzc1MDczNF9WNA)

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YzQwMzY3YTQ5ZDM4MzBhOWQwMjA3OWVjNzRlOWZmNWVfZW90dG5wZXRwdjUwVkZHZFpldHFKUHpVWjlRUHg4b0JfVG9rZW46SmNkVWJrTzBRb09td3B4Z0F0M2NrelFJbjZlXzE3MDM3NDcxMzQ6MTcwMzc1MDczNF9WNA)

# CommonAnnotationBeanPostProcessor#postProcessProperties

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