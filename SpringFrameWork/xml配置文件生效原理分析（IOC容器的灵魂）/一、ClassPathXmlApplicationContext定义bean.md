
1、定义Aclass，重写构造方法和toString方法
```java
public class A {
    A(){
        System.out.println("我被初始化了。。");
    }

    @Override
    public String toString() {
        return "I am A class";
    }
}
```

2、通过XML配置创建bean
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--    定义一个A的bean，默认是单例-->
    <bean id="a" class="com.sise.course1.A"></bean>
</beans>

```
3、编写启动类，这里我使用的是maven的项目结构，所以把xml放到resources文件夹里就可以直接识别。
```java
public class XmlApplication {
    public static void main(String[] args) {
        //通过指定xml加载上下文
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
       //启动容器
        context.refresh();
        //从容器中获取A的bean
        A bean = context.getBean(A.class);
        System.out.println(bean.toString());
    }
}


```

这是一个spring使用xml定义bean的简单方式。
# refresh()
## obtainFreshBeanFactory
启动IOC容器的主要代码就是context.refresh()。
点进这个方法你会发现，refresh是父类AbstractApplicationContext的方法，它采用了模板的设计模式，定义了一套spring容器启动时的抽象方法以及调用顺序。我们先不关心其他的，就只看obtainFreshBeanFactory方法。
```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			prepareRefresh();

			// 本节重点，obtainFreshBeanFactory方法创建BeanFactory容器
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			prepareBeanFactory(beanFactory);

			try {
				postProcessBeanFactory(beanFactory);
				invokeBeanFactoryPostProcessors(beanFactory);
				registerBeanPostProcessors(beanFactory);
				initMessageSource();
				initApplicationEventMulticaster();
				onRefresh();
				registerListeners();
				finishBeanFactoryInitialization(beanFactory);
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}
				destroyBeans();
				cancelRefresh(ex);
				throw ex;
			}

			finally {
				resetCommonCaches();
			}
		}
	}
	

```
### refreshBeanFactory
obtainFreshBeanFactory方法在父类AbstractApplicationContext已经给出了实现，首先会调到refreshBeanFactory方法，而refreshBeanFactory方法是个抽象方法，所以会调用到子类AbstractRefreshableApplicationContext的refreshBeanFactory方法。
```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}

```
#### loadBeanDefinitions

子类AbstractRefreshableApplicationContext的refreshBeanFactory方法调用了一个==loadBeanDefinitions的抽象方法，这个loadBeanDefinitions方法在AbstractXmlApplicationContext中实现了xml文件的加载以及把xml中的bean标签封装成BeanDefinitions并注册到spring容器。==
```java
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}

```
# ClassPathXmlApplicationContext中对loadBeanDefinitions的实现。
```java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 创建xml解析器
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		//通过xml解析器加载BeanDefinitions
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

```
结语：在main方法调用context.refresh() 时，会调用到父类AbstractApplicationContext的refresh方法，而refresh方法调用obtainFreshBeanFactory方法初始化beanfactory容器，在这个方法里又会调用到refreshBeanFactory。refreshBeanFactory是个抽象方法，在子类AbstractRefreshableApplicationContext实现了这个接口，并调用loadBeanDefinitions这个抽象方法。最终AbstractXmlApplicationContext对loadBeanDefinitions接口实现了xml的解析以及对BeanDefinitions的封装和注册。（你可以借助下图理解各个类之间的关系）
![[Pasted image 20231228154100.png]]
