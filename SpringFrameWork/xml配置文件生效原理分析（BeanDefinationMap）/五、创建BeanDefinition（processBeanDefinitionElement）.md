1、上一节我们知道了parseDefaultElement会解析xml标签，其中bean标签的处理主要是通过processBeanDefinition方法
# processBeanDefinition
## parseBeanDefinitionElement(ele)
现在我们主要关注BeanDefinitionParserDelegate的parseBeanDefinitionElement方法，这个方法会让我们得到一个Bean Definition对象。

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //本节重点，如何通过bean标签得到一个BeanDefinitionHolder
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				...
			}
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}

```
## parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean)重载
2、parseBeanDefinitionElement(Element ele)是一个包装方法，它具体逻辑是封装参数调用重载的parseBeanDefinitionElement(ele, null)，
```java
   //方法包装
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
```

而在parseBeanDefinitionElement(ele, null)方法里也做了包装，继续调用到重载的parseBeanDefinitionElement(ele, beanName, containingBean)方法。看来spring最喜欢做的事情就是包装了。。。
```java

   //实现bean标签解析的主要方法
   	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
   	    //获取bean标签的id属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
		//获取bean标签的name属性
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

        //如果id不为空那么beanName使用id值，如果id为空，那么beanName则使用第一个别名
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}

        //主要方法，处理bean标签得到一个AbstractBeanDefinition 对象
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}

```
### parseBeanDefinitionElement(Element ele, String beanName, @Nullable BeanDefinition containingBean)重载
3、parseBeanDefinitionElement重载方法负责了对BeanDefinition对象的创建以及它的属性填充功能。
```java
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
		    //开始创建AbstractBeanDefinition 对象
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			
			//对AbstractBeanDefinition进行属性填充，如scope、abstract、lazy等等，BeanDefinition对象
			//就是对bean的一个定义，它定义了这个bean是否是单例的、是否是抽象的、是否是懒加载等等，
			//而之所以spring没有直接用class也是因为class对象能含有spring的bean的全部属性定义。
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			//对bean便签的子标签的处理
			//    <bean id="a" class="com.sise.course1.A">
            //       <meta key="" value=""/>
            //    </bean>
			parseMetaElements(ele, bd);
			//在BeanDefinition中记录lookup-method属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			//在BeanDefinition中记录replaced-method等属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			//在BeanDefinition中记录构造器的参数注入属性,如：
			//    <bean id="a" class="com.sise.course1.A">
       		//		 <constructor-arg name="i" value="1"></constructor-arg>
    		//	  </bean>
			parseConstructorArgElements(ele, bd);
			//在BeanDefinition中记录property属性
			parsePropertyElements(ele, bd);
			//在BeanDefinition中记录qualifier属性
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (Exception ex) {
		    ....
		}

		return null;
	}

```
#### createBeanDefinition
4、最后，看看BeanDefinition的创建
```java
	//也是一个包装方法
	protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
			throws ClassNotFoundException {

		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}

	//创建BeanDefinition对象的逻辑
	public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {
		//直接new一个GenericBeanDefinition对象，所以我们的BeanDefinition对象一般都是
		//GenericBeanDefinition对象
		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setParentName(parentName);
		if (className != null) {
			if (classLoader != null) {
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			}
			else {
				bd.setBeanClassName(className);
			}
		}
		return bd;
	}


```
5、结语：本节主要讲述了bean标签如何被解析为一个BeanDefinition对象的，并且还讲述了对BeanDefinition的属性填充。在阅读spring源码时要注意spring比较喜欢对方法进行包装，想要看到具体的实现逻辑需要深追到实现的方法。
