# refreshBeanFactory
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
