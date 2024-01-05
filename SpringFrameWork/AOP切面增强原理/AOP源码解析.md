SpringAOP的实现原理是动态代理，最终放入容器的是代理类的对象，而不是Bean本身的对象。

AOP增强定位于Bean生命周期当中的后置处理操作`BeanPostProcessor`，如下图所示
![[Pasted image 20240105145914.png]]

接下来让我们从@EnableAspectJAutoProxy注解入手，去束流AOP的原理：

1. 开启@EnableAspectJAutoProxy，注入Bean的后置处理器AnnotationAwareAspectJAutoProxyCreator同时AnnotationAwareAspectJAutoProxyCreator是一个InstantiationAwareBeanPostProcessor类型的后置处理器；
    
2. 找被切面增强的Advisor
    
    1. org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findEligibleAdvisors
        
    2. org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors
        
    3. org.springframework.aop.support.AopUtils#findAdvisorsThatCanApply
        
3. 代理对象的创建
    
    1. org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization
        
    2. org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#wrapIfNecessary
        
    3. org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#createProxy
        
    4. org.springframework.aop.framework.JdkDynamicAopProxy#getProxy(java.lang.ClassLoader)
        
4. 代理方法的执行
    
    1. org.springframework.aop.framework.JdkDynamicAopProxy#invoke
        
    2. org.springframework.aop.framework.ReflectiveMethodInvocation#proceed，interceptorsAndDynamicMethodMatchers中第一个advice为org.springframework.aop.interceptor.ExposeInvocationInterceptor。
        
5. 拦截器链MethodInterceptor
    
    1. ExposeInvocationInterceptor
        
    2. 环绕通知的执行
        
    3. 前置通知的执行
        
    4. 后置通知的执行
        
    5. 返回后通知的执行
        
    6. 异常通知的执行
## 一、创建与注册后置处理器AnnotationAwareAspectJAutoProxyCreator

1. 传入配置类，创建ioc容器
    
2. 注册配置类，调用refresh（）刷新容器；
    
3. registerBeanPostProcessors(beanFactory);注册bean的后置处理器来方便拦截bean的创建；注册BeanPostProcessor，实际上就是创建BeanPostProcessor对象，保存在容器中；
    
    1. 先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor
        
    2. 给容器中加别的BeanPostProcessor
        
    3. 优先注册实现了PriorityOrdered接口的BeanPostProcessor；
        
    4. 再给容器中注册实现了Ordered接口的BeanPostProcessor；
        
    5. 注册没实现优先级接口的BeanPostProcessor；创建internalAutoProxyCreator的BeanPostProcessor【`AnnotationAwareAspectJAutoProxyCreator`】
        
        1. 创建Bean的实例Instantiation
            
        2. populateBean；给bean的各种属性赋值
            
        3. initializeBean：初始化bean；
            
            1. invokeAwareMethods()：处理Aware接口的方法回调
                
            2. applyBeanPostProcessorsBeforeInitialization()：应用后置处理器的postProcessBeforeInitialization（）
                
            3. invokeInitMethods()；执行自定义的初始化方法
                
            4. applyBeanPostProcessorsAfterInitialization()；执行后置处理器的postProcessAfterInitialization（）；
                
        4. BeanPostProcessor(`AnnotationAwareAspectJAutoProxyCreator`)创建成功；aspectJAdvisorsBuilder
            
        5. 把`AnnotationAwareAspectJAutoProxyCreator`注册到BeanFactory中；beanFactory.addBeanPostProcessor(postProcessor); =======以上是创建和注册AnnotationAwareAspectJAutoProxyCreator的过程========



@EnableAspectJAutoProxy注解用于开启AOP功能，它使用@Import注解向Spring容器中注入了一个类型为`AspectJAutoProxyRegistrar`的`class`
![[Pasted image 20240105151241.png]]

`AspectJAutoProxyRegistrar.class`实现了`ImportBeanDefinitionRegistrar`接口，而`ImportBeanDefinitionRegistrar`是spring提供的扩展点之一，主要用来向容器中注入`BeanDefinition`，Spring会根据BeanDefinion来生成Bean。通过`AspectJAutoProxyRegistrar`向IOC容器中注册一个`AnnotationAwareAspectJAutoProxyCreator`组件
![[Pasted image 20240105151248.png]]

==AnnotationAwareAspectJAutoProxyCreator属于InstantiationAwareBeanPostProcessor
，相当于我们给容器中注册了一个InstantiationAwareBeanPostProcessor，这可是Bean生命周期中的第一个扩展点呀！！！==
![[Pasted image 20240105151301.png]]


# 二、代理对象创建 postProcessAfterInitialization
finishBeanFactoryInitialization(beanFactory);完成BeanFactory初始化工作；创建剩下的单实例bean。遍历获取容器中所有的Bean，依次创建对象getBean(beanName)。getBean->doGetBean()->getSingleton()-> 创建bean，doCreateBean：Bean生命周期的两个扩展点：

1. InstantiationAwareBeanPostProcessor是在创建Bean实例之前先尝试用后置处理器返回对象
    
    1. ==InstantiationAwareBeanPostProcessor在所有bean实例化之前会有一个拦截，会调用postProcessBeforeInstantiation()。先从缓存中获取代理对象==，如果能获取到，说明bean是之前被创建过的，直接使用，否则再创建；只要创建好的Bean都会被缓存起来。
        
    2. 上面的AnnotationAwareAspectJAutoProxyCreator其实就是InstantiationAwareBeanPostProcessor。
        
2. ==BeanPostProcessor是在Bean对象创建完成初始化前后调用的，创建增强对象：==
    
    1. ==postProcessAfterInitialization方法return wrapIfNecessary(bean, beanName, cacheKey);方法会判断该Bean是否注册了切面，若是，则生成代理对象注入到容器中，下面是代理对象的生成逻辑：==
		![[Pasted image 20240105151315.png]]
    1. 获取当前bean的所有拦截器（通知方法） Object[] specificInterceptors
        
        1. 找到候选的所有的增强器（找哪些通知方法是需要切入当前bean方法的）
            
        2. 获取到能在bean使用的增强器。
            
        3. 给增强器advisor排序，
            
        4. 给增强器advisor转换为拦截器Interceptor
            
    3. 创建当前bean的代理对象；
        
        1. 获取所有增强器（通知方法）
            
        2. 保存到proxyFactory
            
        3. 创建代理对象：Spring自动决定
            
            1. JdkDynamicAopProxy(config);jdk动态代理；
                
            2. ObjenesisCglibAopProxy(config);cglib的动态代理；
                
    4. 以后容器中获取到的就是这个组件的代理对象，执行目标方法的时候，代理对象就会执行通知方法的流程；
        

其实，在实例化createBeanInstance之后，initializeBean之前，doCreateBean方法中调用了这么一段代码addSingletonFactory来给三级缓存中添加早期引用
```java
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
```
getEarlyBeanReference方法使用一个Map earlyBeanReferences（注意：非三级缓存的map）保存了早期引用对象
```java
private final Map<Object, Object> earlyBeanReferences = new ConcurrentHashMap<>(16);

@Override  
public Object getEarlyBeanReference(Object bean, String beanName) {  
    Object cacheKey = getCacheKey(bean.getClass(), beanName);  
    this.earlyBeanReferences.put(cacheKey, bean);  
    return wrapIfNecessary(bean, beanName, cacheKey);  
}
```

==接下来着重分析Bean的后置处理器`AbstractAutoProxyCreator`的`postProcessAfterInitialization`方法：==

因此在AbstractAutoProxyCreator#postProcessAfterInitialization方法中，才会判断if (this.earlyProxyReferences.remove(cacheKey) != bean)？

- 如果earlyProxyReferences中的cacheKey对应的bean不等于当前bean，则按需生成代理对象wrapIfNecessary
- 如果earlyProxyReferences中的cacheKey对应的bean等于当前bean，则直接返回当前bean即可

```Java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        // 先从缓存中获取代理对象，    
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        //如果之前调用过getEarlyBeanReference获取包装目标对象到AOP代理对象（如果需要），则不再执行
        //cacheKey保证不会重复生成代理对象
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                // 包装目标对象到AOP代理对象（如果需要）
                return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```




按需生成代理对象，org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#wrapIfNecessary

```Java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
            return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
    }

    // Create proxy if we have advice.
    /**
     * @see AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean(java.lang.Class, java.lang.String, org.springframework.aop.TargetSource)
     */
    // 获取与当前Bean匹配的切面
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 创建代理
            Object proxy = createProxy(
                            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
    }

    // 缓存
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

即使当前的`abstract class AbstractAutoProxyCreator`还未实现`getAdvicesAndAdvisorsForBean`方法，但是没关系，有子类`AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator`并且实现了`getAdvicesAndAdvisorsForBean`方法
![[Pasted image 20240105151356.png]]

因此实际中使用的时候肯定用的是子类`AbstractAdvisorAutoProxyCreator` 。但为什么还要定义抽象父类`AbstractAutoProxyCreator`呢？

原因是这里使用的抽象模板设计模式：

- 父类是个抽象类
    
- 抽象父类可以实现部分方法，这部分方法是为了让子类代码复用
    
- 子类再去完成父类没有实现的方法即可
    
- 子类依然可以是个抽象类，依然可以留下一些方法不去实现
    

## 找到切面Bean对应的所有增强方法Advisor

找被切面Bean增强的所有Advisor

### AbstractAdvisorAutoProxyCreator

来到`AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator`
![[Pasted image 20240105151402.png]]
![[Pasted image 20240105151408.png]]

AbstractAdvisorAutoProxyCreator#findEligibleAdvisors

```Java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
        /**
         * @see AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors()
         */
        // 获取容器中所有的切面Advisor
        // 这里返回的切面中的方法已经是有序的了，
    //先按注解顺序（Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class），再按方法名称
        List<Advisor> candidateAdvisors = findCandidateAdvisors();
        // 获取所有能够作用于当前Bean上的Advisor
        List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
        /**
         * @see AspectJAwareAdvisorAutoProxyCreator#extendAdvisors(java.util.List)
         */
        // 往集合第一个位置加入了一个DefaultPointcutAdvisor
        extendAdvisors(eligibleAdvisors);
        if (!eligibleAdvisors.isEmpty()) {
                /**
                 * @see AspectJAwareAdvisorAutoProxyCreator#sortAdvisors(java.util.List)
                 */
                // 这里是对切面进行排序，例如有@Order注解或者实现了Ordered接口
                eligibleAdvisors = sortAdvisors(eligibleAdvisors);
        }
        return eligibleAdvisors;
}
```

#### findCandidateAdvisors

`AbstractAdvisorAutoProxyCreator`中的成员变量：

`private BeanFactoryAdvisorRetrievalHelper advisorRetrievalHelper`翻译过来叫做`advisor找回助手`
![[Pasted image 20240105151415.png]]

追溯`this.advisorRetrievalHelper.findAdvisorBeans()`到下方代码，非常精妙

这段代码有个成员变量缓存cachedAdvisorBeanNames，初次if (advisorNames == null) 为true，那么就往缓存中写入符合条件的AdvisorBeanNames，以后if (advisorNames == null) 为false，就直接从缓存中获取了。以空间换时间！！！

```Java
public List<Advisor> findAdvisorBeans() {
    // Determine list of advisor bean names, if not cached already.
    String[] advisorNames = this.cachedAdvisorBeanNames;
    if (advisorNames == null) {
       // Do not initialize FactoryBeans here: We need to leave all regular beans
       // uninitialized to let the auto-proxy creator apply to them!
       advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
             this.beanFactory, Advisor.class, true, false);
       this.cachedAdvisorBeanNames = advisorNames;
    }
    if (advisorNames.length == 0) {
       return new ArrayList<>();
    }

    List<Advisor> advisors = new ArrayList<>();
    for (String name : advisorNames) {
       if (isEligibleBean(name)) {
          if (this.beanFactory.isCurrentlyInCreation(name)) {
             if (logger.isTraceEnabled()) {
                logger.trace("Skipping currently created advisor '" + name + "'");
             }
          }
          else {
             try {
                advisors.add(this.beanFactory.getBean(name, Advisor.class));
             }
             catch (BeanCreationException ex) {
                Throwable rootCause = ex.getMostSpecificCause();
                if (rootCause instanceof BeanCurrentlyInCreationException bce) {
                   String bceBeanName = bce.getBeanName();
                   if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                      if (logger.isTraceEnabled()) {
                         logger.trace("Skipping advisor '" + name +
                               "' with dependency on currently created bean: " + ex.getMessage());
                      }
                      // Ignore: indicates a reference back to the bean we're trying to advise.
                      // We want to find advisors other than the currently created bean itself.
                      continue;
                   }
                }
                throw ex;
             }
          }
       }
    }
    return advisors;
}
```

#### findAdvisorsThatCanApply
![[Pasted image 20240105151423.png]]

org.springframework.aop.support.AopUtils#findAdvisorsThatCanApply

```Java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
        if (candidateAdvisors.isEmpty()) {
                return candidateAdvisors;
        }
        List<Advisor> eligibleAdvisors = new ArrayList<>();
        // InstantiationModelAwarePointcutAdvisorImpl
        for (Advisor candidate : candidateAdvisors) {
            if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
                // IntroductionAdvisor类型为引入切面，具体类型为DeclareParentsAdvisor
                eligibleAdvisors.add(candidate);
            }
        }
        boolean hasIntroductions = !eligibleAdvisors.isEmpty();
        for (Advisor candidate : candidateAdvisors) {
            if (candidate instanceof IntroductionAdvisor) {
                // already processed
                continue;
            }
            // PointCut中的ClassFilter.match 匹配类
            // PointCut中的MethodMatcher.match 匹配方法
            if (canApply(candidate, clazz, hasIntroductions)) {
                // @Aspect，类型为InstantiationModelAwarePointcutAdvisorImpl
                eligibleAdvisors.add(candidate);
            }
        }
        return eligibleAdvisors;
}
```

##### 创建代理对象

在为业务逻辑组件创建代理对象的时候，使用的是cglib来创建动态代理的。

当然了，如果业务逻辑类有实现接口，那么就使用jdk来创建动态代理。

```Java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                @Nullable Object[] specificInterceptors, TargetSource targetSource) {

        if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
                AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
        }

        // 创建代理工厂
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.copyFrom(this);

        if (!proxyFactory.isProxyTargetClass()) {
                // 进来说明proxyTargetClass=false，指定JDK代理
                if (shouldProxyTargetClass(beanClass, beanName)) {
                        // 进来这里说明BD中有个属性preserveTargetClass=true，可以BD中属性设置的优先级最高
                        proxyFactory.setProxyTargetClass(true);
                }
                else {
                        // 这里会判断bean有没有实现接口，没有就只能使用CGlib
                        evaluateProxyInterfaces(beanClass, proxyFactory);
                }
        }

        Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
        proxyFactory.addAdvisors(advisors); // 切面
        proxyFactory.setTargetSource(targetSource); // 目标对象
        customizeProxyFactory(proxyFactory);

        proxyFactory.setFrozen(this.freezeProxy);
        if (advisorsPreFiltered()) {
                proxyFactory.setPreFiltered(true);
        }

        // 使用JDK或者CGlib创建代理对象
        return proxyFactory.getProxy(getProxyClassLoader());
}
```

# 三、目标方法的执行

这个代理对象创建完以后，IOC容器也就创建完了。接下来，便要来执行目标方法了。
容器中保存了组件的代理对象（cglib增强后的对象），这个对象里面保存了详细信息（比如增强器，目标对象，xxx）；

1. CglibAopProxy.intercept();拦截目标方法的执行
    
2. 获取将要执行的目标方法拦截器链；`List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);`
    
    1. 将增强器转为拦截器`List<MethodInterceptor>`；遍历所有的增强器advisor，调用registry.getInterceptors(advisor);转换完成返回MethodInterceptor数组；
        
    2. `List<Object> interceptorList`保存所有拦截器 5个，一个默认的ExposeInvocationInterceptor 和 4个增强器；
        
3. 把需要执行的目标对象，目标方法，拦截器链等信息传入创建一个 CglibMethodInvocation 对象，并调用 Object retVal = mi.proceed();火炬方法
    

当Bean对象初始化完成之后，`postProcessAfterInitialization`方法会判断该Bean是否注册了切面，若是，则生成代理对象注入到容器中





此时，其实是代理对象来执行目标方法。createAopProxy() 方法决定了是使用 JDK 还是 Cglib 来做动态代理：

![[Pasted image 20240105151443.png]]

- 如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理
    
- 如果目标对象没有实现了接口，会使用 CGLIB 动态代理
    

```Java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

        @Override
        public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
                if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
                    Class<?> targetClass = config.getTargetClass();
                    if (targetClass == null) {
                            throw new AopConfigException("TargetSource cannot determine target class: " +
                                            "Either an interface or a target is required for proxy creation.");
                    }
                    if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                            return new JdkDynamicAopProxy(config);
                    }
                    return new ObjenesisCglibAopProxy(config);
                }
                else {
                    return new JdkDynamicAopProxy(config);
                }
        }
  .......
}
```

## JdkDynamicAopProxy

JDK动态代理时候`InvocationHandler`的实现类为org.springframework.aop.framework.JdkDynamicAopProxy。
![[Pasted image 20240105151449.png]]

那么代理方法的调用肯定会进入JdkDynamicAopProxy.invoke()方法
![[Pasted image 20240105151454.png]]
![[Pasted image 20240105151459.png]]

```Java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;

        TargetSource targetSource = this.advised.targetSource;
        Object target = null;

        try {
                if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
                        // The target does not implement the equals(Object) method itself.
                        return equals(args[0]);
                }
                else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                        // The target does not implement the hashCode() method itself.
                        return hashCode();
                }
                else if (method.getDeclaringClass() == DecoratingProxy.class) {
                        // There is only getDecoratedClass() declared -> dispatch to proxy config.
                        return AopProxyUtils.ultimateTargetClass(this.advised);
                }
                else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                        // Service invocations on ProxyConfig with the proxy config...
                        return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
                }

                Object retVal;

                if (this.advised.exposeProxy) {
                        // Make invocation available if necessary.
                        oldProxy = AopContext.setCurrentProxy(proxy);
                        setProxyContext = true;
                }

                // Get as late as possible to minimize the time we "own" the target,
                // in case it comes from a pool.
                target = targetSource.getTarget(); // 目标对象
                Class<?> targetClass = (target != null ? target.getClass() : null); // 目标对象的类型

                // Get the interception chain for this method.
                // 这里会对方法进行匹配，因为不是目标对象中的所有方法都需要增强
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

                // Check whether we have any advice. If we don't, we can fallback on direct
                // reflective invocation of the target, and avoid creating a MethodInvocation.
                if (chain.isEmpty()) {
                        // We can skip creating a MethodInvocation: just invoke the target directly
                        // Note that the final invoker must be an InvokerInterceptor so we know it does
                        // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
                        // 没有匹配的切面，直接通过反射调用目标对象的目标方法
                        Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                        retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
                }
                else {
                        // We need to create a method invocation...
                        MethodInvocation invocation =
                                        new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                        // Proceed to the joinpoint through the interceptor chain.
                        /**
                         * @see ReflectiveMethodInvocation#proceed()
                         */
                        // 这里才是增强的调用，重点，火炬的传递
                        retVal = invocation.proceed();
                }

                // Massage return value if necessary.
                Class<?> returnType = method.getReturnType();
                if (retVal != null && retVal == target &&
                                returnType != Object.class && returnType.isInstance(proxy) &&
                                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                        // Special case: it returned "this" and the return type of the method
                        // is type-compatible. Note that we can't help if the target sets
                        // a reference to itself in another returned object.
                        retVal = proxy;
                }
                else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                        throw new AopInvocationException(
                                        "Null return value from advice does not match primitive return type for: " + method);
                }
                return retVal;
        }
        finally {
                if (target != null && !targetSource.isStatic()) {
                        // Must have come from TargetSource.
                        targetSource.releaseTarget(target);
                }
                if (setProxyContext) {
                        // Restore old proxy.
                        AopContext.setCurrentProxy(oldProxy);
                }
        }
}
```

## CglibAopProxy（可选，提前剧透）

如果目标对象没有实现接口，则使用CglibAopProxy类的内部类的DynamicAdvisedInterceptor#intercept()方法来拦截目标方法的执行

```Java
/**
 * General purpose AOP callback. Used when the target is dynamic or when the
 * proxy is not frozen.
 */
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

    private final AdvisedSupport advised;

    public DynamicAdvisedInterceptor(AdvisedSupport advised) {
       this.advised = advised;
    }

    @Override
    @Nullable
    public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
       Object oldProxy = null;
       boolean setProxyContext = false;
       Object target = null;
       TargetSource targetSource = this.advised.getTargetSource();
       try {
          if (this.advised.exposeProxy) {
             // Make invocation available if necessary.
             oldProxy = AopContext.setCurrentProxy(proxy);
             setProxyContext = true;
          }
          // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
          target = targetSource.getTarget();
          Class<?> targetClass = (target != null ? target.getClass() : null);
          List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
          Object retVal;
          // Check whether we only have one InvokerInterceptor: that is,
          // no real advice, but just reflective invocation of the target.
          if (chain.isEmpty()) {
             // We can skip creating a MethodInvocation: just invoke the target directly.
             // Note that the final invoker must be an InvokerInterceptor, so we know
             // it does nothing but a reflective operation on the target, and no hot
             // swapping or fancy proxying.
             Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
             retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
          }
          else {
             // We need to create a method invocation...
             retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
          }
          return processReturnType(proxy, target, method, args, retVal);
       }
       finally {
          if (target != null && !targetSource.isStatic()) {
             targetSource.releaseTarget(target);
          }
          if (setProxyContext) {
             // Restore old proxy.
             AopContext.setCurrentProxy(oldProxy);
          }
       }
    }
```

CglibAopProxy的内部类CglibMethodInvocation实现了火炬`ReflectiveMethodInvocation`，调用火炬方法`super.proceed()`
![[Pasted image 20240105151511.png]]

五个拦截器的invoke方法中（第一个拦截器`ExposeInvocationInterceptor`没有调增强，单纯调火炬）

- 调用了火炬
    
- 调用了CglibAopProxy增强
    

拦截器的invoke方法返回，被火炬传递到下一个拦截器
![[Pasted image 20240105151519.png]]

## ReflectiveMethodInvocation#proceed火炬

以JdkDynamicAopProxy的invoke方法为例，继续往下分析
![[Pasted image 20240105151526.png]]

成员变量索引currentInterceptorIndex的值初始为-1
![[Pasted image 20240105151531.png]]

`interceptorsAndDynamicMethodMatchers`中第一个advice为`org.springframework.aop.interceptor.ExposeInvocationInterceptor`。

```Java
public Object proceed() throws Throwable {
        // We start with an index of -1 and increment early.
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
                // 执行到最后一个Advice，才会到这里执行目标方法
                return invokeJoinpoint();
        }

        Object interceptorOrInterceptionAdvice =
                        this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            // Evaluate dynamic method matcher here: static part will already have
            // been evaluated and found to match.
            // dm.isRuntime()=true的走这
            InterceptorAndDynamicMethodMatcher dm =
                            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
            Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
            if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
                    return dm.interceptor.invoke(this);
            }
            else {
                    // Dynamic matching failed.
                    // Skip this interceptor and invoke the next in the chain.
                    return proceed();
            }
        }
        else {
                // It's an interceptor, so we just invoke it: The pointcut will have
                // been evaluated statically before this object was constructed.
                // 走这，第一次调用ExposeInvocationInterceptor#invoke
                return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
}
```

## ExposeInvocationInterceptor缓存火炬

MethodInvocation中封装了目标对象，目标方法，方法参数等信息。ExposeInvocationInterceptor#invoke方法干了俩件事

1. 就是将MethodInvocation加入到了ThreadLocal中：
    
    1. `ThreadLocal` 的特点是存在它里边的数据，哪个线程存的，哪个线程才能访问到。
        
    2. 这样后续可以在其他地方（before/after/return/throw）使用。ExposeInvocationInterceptor#currentInvocation获取到MethodInvocation：`ReflectiveMethodInvocation`
    ![[Pasted image 20240105151542.png]]
    
2. 循环调用return mi.proceed();传递火炬
    

```Java
    private static final ThreadLocal<MethodInvocation> invocation = new NamedThreadLocal<>("Current AOP method invocation");

    public Object invoke(MethodInvocation mi) throws Throwable {
        MethodInvocation oldInvocation = invocation.get();
        invocation.set(mi);
        try {
                return mi.proceed();
        }
        finally {
                invocation.set(oldInvocation);
        }
    }
    
    public static MethodInvocation currentInvocation() throws IllegalStateException {
        MethodInvocation mi = invocation.get();
        if (mi == null) {
           throw new IllegalStateException(
                 "No MethodInvocation found: Check that an AOP invocation is in progress and that the " +
                 "ExposeInvocationInterceptor is upfront in the interceptor chain. Specifically, note that " +
                 "advices with order HIGHEST_PRECEDENCE will execute before ExposeInvocationInterceptor! " +
                 "In addition, ExposeInvocationInterceptor and ExposeInvocationInterceptor.currentInvocation() " +
                 "must be invoked from the same thread.");
            }
            return mi;
        }
}
```