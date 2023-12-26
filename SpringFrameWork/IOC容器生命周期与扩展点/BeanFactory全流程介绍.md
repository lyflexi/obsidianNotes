Spring IOC ：

- Bean定义的定位,Bean 可能定义在XML中，或者一个注解，或者其他形式。这些都被用Resource来定位, IOC容器读取Resource获取BeanDefinition 注册到 Bean定义注册表中。
- ==第一次向容器getBean操作会触发Bean的创建过程，实列化一个Bean时，要根据BeanDefinition中类信息等实列化Bean.==
- ==将实列化的Bean放到单例Bean缓存内。==
- ==此后再次获取向容器getBean就会从缓存中获取。==

![[Pasted image 20231226153338.png]]

refresh方法是容器刷新的入口，方法定义在AbstractApplicationContext 中

```Java
public void refresh() throws BeansException, IllegalStateException {
//同步代码块 。
        synchronized (this.startupShutdownMonitor) {
            // 准备刷新.
            prepareRefresh();
            // Tell the subclass to refresh the internal bean factory.
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);
            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);
                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);
                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);
                // Initialize message source for this context.
                initMessageSource();
                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();
                // Initialize other special beans in specific context subclasses.
                onRefresh();
                // Check for listener beans and register them.
                registerListeners();
                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);
                // Last step: publish corresponding event.
                finishRefresh();
            }
            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }
                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();
                // Reset 'active' flag.
                cancelRefresh(ex);
                // Propagate exception to caller.
                throw ex;
            }
            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
```

# 一、prepareRefresh 准备刷新

1. 设置启动时间,设计容器激活标志(active=true)
    
2. 初始化 properties 资源
    
3. 验证必须存在的properties(validateRequiredProperties)
    
4. 初始earlyApplicationEnvents，用于收集已经产生的ApplicationEnvents.
    

# 二、obtainFreshBeanFactory 获取默认容器

1. 调用子类的refeshBeanFactory()，SpringBoot中采用默认的实现，设置BeanFactory的SerializationId，设置refreshed标志为true。
    
2. 获取BeanFactory，程序启动就会给我们创建GenericApplicationContext这个BeanFactory，GenericApplicationContext是AbstractApplicationContext的子类
    
3. XmlWebApplicationContext ，AnnotationConfigApplicationContext 会在这一步加载BeanDefinition
    

# 三、prepareBeanFactory 【标准初始化】

为了使用==内部的BeanFactory== ，为内部的BeanFactory 做些标准的设置

1. 为内部的BeanFactory设置类加载器
    
2. 设置表达式解析器，属性编辑器。
    
3. 为内部BeanFactory注册后置处理器BeanPostProcessor(ApplicationContextAwareProcessor，ApplicationListenerDetector),
    
    1. ApplicationContextAwareProcessor：对Spring 中各种Aware接口的支持，在初始化Bean前，调用Bean实现的Aware接口方法。
        
    2. ApplicationListenerDetector：用于收集实现了ApplicationListener接口的Bean
        
4. 注册指定的依赖类型和对应的value,
    1. 例如:beanFactory.registerResolvableDependency(ResourceLoader.class, this)，那么在类中自动注入ResourceLoader类型的对象，就会拿到当前IOC容器。
5. 注入一些其它信息的bean，比如environment、systemProperties等
    

# 四、postProcessBeanFactory为工厂注册后置处理器

用于在BeanFactory 完成标准的初始化之后修改BeanFactory。不同容器根据自己的需求添加特殊的后置处理器， EmbeddedWebApplicationContext 容器在这里添加了对ServletContextAware支持的Bean后置处理器WebApplicationContextServletContextAwareProcessor。

# 五、invokeBeanFactoryPostProcessors执行BeanFactory后置处理器。

实例化并且执行所有已经注册到BeanFactory中的 BeanFactoryPostProcessor。支持按照Order排序。
执行逻辑是进一步委托给PostProcessorRegistrationDelegate的静态方法invokeBeanFactoryPostProcessors

执行顺序：
1. 执行ApplicationContext初始化器注册的BeanDefinitionRegistryPostProcessor类型的处理器。BeanDefinitionRegistryPostProcessor 接口继承自BeanFactoryPostProcessor，并且会先于BeanFactoryPostProcessor 执行。
	1. 执行实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessor。
	2. 执行实现了Ordered的BeanDefinitionRegistryPostProcessor
	3. 执行继承自BeanFactoryPostProcessor 的接口方法
2. 获取所有的BeanFactoryPostProcessor，按照1.1-1.2的顺序，执行BeanFactoryPostProcessor的接口方法

==介绍BeanFactoryPostProcessor 接口==：
针对BeanFactory 的处理，允许使用者修改容器中的BeanFactory definitions，例如： 
- ServletComponentRegisteringPostProcessor提供对@WebFilter,@WebServlet,@WebListener注解的扫描，并注册到容器中。
- ==ConfigurationClassPostProcessor是SpringBoot自动配置功能的驱动者，对@Configuration,@Bean,@ComponentScan,@Import,@ImportResource,@PropertySource注解解析（对用上述注解指定的类进行动态代理的增强）。如果没有ConfigurationClassPostProcessor，那么这些神奇的自动配置注解都不在起作用==
	- ConfigurationClassPostProcessor后置处理器 实现了 PriorityOrdered接口，优先级最高的后置处理器，在SpringBoot自动配置的实现中起到举足轻重的作用。
		1. 处理PropertySources 注解 加载property资源
		    
		2. 处理ComponentScans和ComponentScan注解 扫描指定包下所有@Component注解。包括@Component，@Service，@Controller,@Repository,@Configuration,@ManagedBean
		    
		3. 处理@Import注解类 解析所有包含@Import注解。spring中`@Enable****`注解的实现依赖。@Import的value 有三种类型的Class.
		    
		    1. ImportSelector 导入选择器。 执行非DeferredImportSelector接口的，收集配置类。 收集DeferredImportSelector接口的实现。稍后执行。
		        
		    2. ImportBeanDefinitionRegistrar 导入类定义注册者 收集ImportBeanDefinitionRegistrar对象。
		        
		    3. 如果不是上面两种类型 就会被当作普通的configuration 类 注册到容器。 例如@EnableScheduling
		        
		4. 处理ImportResource注解 收集@ImportResource中指定的配置文件信息。
		    
		5. 收集@Bean注解的方法
		    
		6. 处理接口默认方法
		    
		7. 处理父类 父类className 不是"java"开头，并且是未处理过的类型。获取父类循环1-7。
		    
		8. 处理3.1中收集的DeferredImportSelector。
		    
		9. 利用ConfigurationClassBeanDefinitionReader 处理每个Configuration类的@Bean和第4步收集的资源和3.2中收集的注册者。

==注意使用时不能对任何BeanDefinition 进行实列化操作。==
# 六、registerBeanPostProcessors 注册bean后置处理器

1. 委托给PostProcessorRegistrationDelegate的静态方法registerBeanPostProcessors执行
    
    1. 按照 PriorityOrdered，Ordered,普通，MergedBeanDefinitionPostProcessor 添加到 beanFactory 的beanPostProcessors属性中。最后添加ApplicationListenerDetector。
        
2. BeanPostProcessors 接口：
    
    1. 该接口在Bean的实例化过程中被调用，例如MergedBeanDefinitionPostProcessor 接口继承了BeanPostProcessors接口, 会在 bean 实列化 之后 属性注入前 执行。例如 对@Autowired @Resource @Value 注解的解析
    2. 该接口在对一个对象进行初始化 前后 被调用。例如实现了 EnvironmentAware,EmbeddedValueResolverAware，ResourceLoaderAware，ApplicationContextAware等Aware就是这个时候 被执行的。

# 七、initMessageSource

在Spring容器中初始化一些国际化相关的属性。

# 八、initApplicationEventMulticaster

在容器中初始化Application事件广播器。

# 九、onRefresh

onRefresh是一个模板方法，留给子类容器扩展，不同的容器做不同的事情。例如：
容器AnnotationConfigEmbeddedWebApplicationContext中会调用createEmbeddedServletContainer方法去创建内置的Servlet容器。 EmbeddedServletContainerAutoConfiguration 类中定义了 Spring boot 支持的三种 Servlet容器。 目前SpringBoot只支持3种内置的Servlet容器：
1. Tomcat
    
2. Jetty
    
3. Undertow
    

# 十、registerListeners

1. 将所有的 ApplicationListener 实现类注册 到 ApplicationEventMulticaster中,觉得是观察者模式。
    
2. 把已经产生的事件广播出去。
    

# 十一、finishBeanFactoryInitialization

实列化所有 非懒加载的 类（剩下的单实例Bean）

# 十二、finishRefresh

1. 完成容器的初始化过程,发布相应事件。
    
2. 启动容器的声明周期处理器。管理容器声明周期。
    
3. 发布 ContextRefreshedEvent事件。
    
4. 启动内嵌的Servlet容器。
    
5. 发布容器启动事件。