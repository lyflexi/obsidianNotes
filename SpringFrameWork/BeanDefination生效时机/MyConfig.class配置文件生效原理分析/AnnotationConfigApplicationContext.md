==BeanDefinition 的注册时机剖析：一共有两个时机！！！==
xml配置文件和注解配置类（以@Configuration声明）
1. 父类AbstractApplicationContext.java的refresh的obtainFreshBeanFactory();，这个我们之前以ClassPathXmlApplicationContext类为切入点，深入分析过==xml的生效原理就是走的obtainFreshBeanFactory()：
	1. refreshBeanFactory()：将 BeanDefinition注册到beanDefinitionMap表中
	2. obtainFreshBeanFactory()；最终返回内部的工厂（ConfigurableListableBeanFactory）DefaultListableBeanFactory。==ConfigurableListableBeanFactory是spring的发动机，以list集合的方式操作bean==
2. 来看父类AbstractApplicationContext的另一个子类AnnotationConfigApplicationContext，构造方法`AnnotationConfigApplicationContext(Class<?>... componentClasses)，`componentClasses接收的是注解配置类==MyConfig.class。也是将 BeanDefinition注册到beanDefinitionMap表中==
![[Pasted image 20231228120315.png]]

跟踪register(componentClasses)方法，总的来看：
1. 首先需要构造描述bean实例化信息的`BeanDefinition`对象，需要将注解配置类信息转化为`AnnotatedGenericBeanDefinition` 类型，此处的`AnnotatedGenericBeanDefinition` 就是一种`BeanDefinition`类型，包含了Bean的构造函数参数，属性值以及添加的注解信息。  
2. 设置`BeanDefinition`属性，完成对`@Scope、@Lazy、@Primary`等注解的处理  
3. 最后通过`registerBeanDefinition()`方法完成Bean的注册。

```Java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
       @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
       @Nullable BeanDefinitionCustomizer[] customizers) {

    // 将注解配置类信息转换成一种 BeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
       return;
    }

    abd.setAttribute(ConfigurationClassUtils.CANDIDATE_ATTRIBUTE, Boolean.TRUE);
    abd.setInstanceSupplier(supplier);
    // 获取bean的作用域元数据
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    // 将bean的作用域写回 BeanDefinition
    abd.setScope(scopeMetadata.getScopeName());
    // 生成 beanName
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
    // 解析AnnotatedGenericBeanDefinition 中的 @lazy 和 @Primary注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // 处理@Qualifier 注解
    if (qualifiers != null) {
       for (Class<? extends Annotation> qualifier : qualifiers) {
          if (Primary.class == qualifier) {
             abd.setPrimary(true);
          }
          else if (Lazy.class == qualifier) {
             abd.setLazyInit(true);
          }
          else {
             abd.addQualifier(new AutowireCandidateQualifier(qualifier));
          }
       }
    }
    if (customizers != null) {
       for (BeanDefinitionCustomizer customizer : customizers) {
          customizer.customize(abd);
       }
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 注册 BeanDefinition到beanDefinitionMap表中
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```
