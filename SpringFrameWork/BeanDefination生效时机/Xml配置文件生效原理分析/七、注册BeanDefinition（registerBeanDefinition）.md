1、通过解析bean标签得到创建BeanDefinition的过程已经简单地讲完了，那么下面processBeanDefinition方法（DefaultBeanDefinitionDocumentReader类里）。创建完Bean Definition并封装成BeanDefinitionHolder以后，就调用BeanDefinitionReaderUtils.registerBeanDefinition进行注册
```java
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册BeanDefinition的holder对象
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```
# BeanDefinitionReaderUtils.registerBeanDefinition
## getReaderContext

2、getReaderContext的这个ReaderContext里的reader其实就是XmlBeanDefinitionReader对象。
![[Pasted image 20231228172046.png]]
## getRegistry
而这个reader对象在创建时的传的是DefaultListAbleBeanFactory对象，所以就是把BeanDefinition注册到DefaultListAbleBeanFactory容器中。
![[Pasted image 20231228172259.png]]
## registry.registerBeanDefinition
进入BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
![[Pasted image 20231228172521.png]]

把DefaultListableBeanFactory的registerBeanDefinition方法简化一下就很清晰了，==就是把这个BeanDefinition put到beanDefinitionMap中key就是beanName==
```java
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		//对BeanDefinition进行校验
		if(校验通过) {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}

```
结语：本节也比较简单，讲述了创建BeanDefinition后spring把这个BeanDefinition注册到DefaultListableBeanFactory的beanDefinitionMap中。这样，spring就完成了BeanDefinition的收集。

---
至此， spring 完成了对bean标签的解析与处理，就是获取到 xml 中 bean 标签里配置的各种属性，封装成一个 BeanDefinition 对象，然后把这个对象存到我们 IOC 容器的 beanDefinitionMap、beanDefinitionNames 中，为之后在 IOC 容器中使用与获取创造条件，即完成 ConfigurableListableBeanFactory 的创建。



