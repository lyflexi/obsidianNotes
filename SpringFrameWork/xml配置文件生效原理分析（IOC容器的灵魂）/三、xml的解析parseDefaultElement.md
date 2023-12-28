1、接着上一节，我们继续看XmlBeanDefinitionReader的loadBeanDefinitions代码（代码被我简化成伪代码）
# XmlBeanDefinitionReader
```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		
		if (resourceLoader instanceof ResourcePatternResolver) {
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				//加载多个xml文件
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				return count;
			}
			catch (IOException ex) {
				...
			}
		}
		else {
			Resource resource = resourceLoader.getResource(location);
			//加载一个xml文件
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			return count;
		}
	}

```
2、==可以看到loadBeanDefinitions加载的是Resource数组，也是循环调用loadBeanDefinitions去加载xml文件==
```java
@Override  
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {  
    Assert.notNull(resources, "Resource array must not be null");  
    int count = 0;  
    for (Resource resource : resources) {  
    //接口BeanDefinitionReader的方法loadBeanDefinitions在XmlBeanDefinitionReader给出了实现。
       count += loadBeanDefinitions(resource);  
    }  
    return count;  
}
```
XmlBeanDefinitionReader#loadBeanDefinitions(EncodedResource encodedResource)
```java


public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException("check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
			    //通过Resource拿到xml文件的输入流封装成InputResource对象，
			    //再调doLoadBeanDefinitions方法进行解析
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			...
		}
		finally {
			...
		}
	}

```
## doLoadBeanDefinitions
3、doLoadBeanDefinitions方法就很清晰明了，通过inputSource获取到这个xml的Document对象，然后就调用registerBeanDefinitions对这个xml的bean对象进行注册，而registerBeanDefinitions方法里创建了一个BeanDefinitionDocumentReader对象对doc进行解析注册
```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
			Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (Exception ex) {
		   ...
		}
	}
	
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}


```
# DefaultBeanDefinitionDocumentReader

4、documentReader.registerBeanDefinitions方法就是调用了doRegisterBeanDefinitions方法进行xml的处理，而doRegisterBeanDefinitions里又调用了parseBeanDefinitions方法，parseBeanDefinitions方法就是真正的对xml进行解析的方法。
```java
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
	
	protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
		//可以对root对象进行扩展
		preProcessXml(root);
		//开始解析xml
		parseBeanDefinitions(root, this.delegate);
		//可以对root对象进行扩展
		postProcessXml(root);

		this.delegate = parent;
	}


```
## parseBeanDefinitions
5、兜兜转转那么久终于到了parseBeanDefinitions方法，这个方法就是真正的解析xml的方法了。这个方法就是把root节点下的所有子节点循环遍历然后调用parseDefaultElement方法解析这些子节点
```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}

```
### parseDefaultElement
6、来到parseDefaultElement就可以看到这里会对import、alias、bean以及beans标签的处理，由于impo标签和beans标签最终还是会变成bean标签的解析，所以就不展开了讲了，而alias标签也就给beanName起个别名也不展开讲
```java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {//处理import标签
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {//处理alias标签
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {//处理bean标签
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {//处理beans标签
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

```

下一节的重点是bean标签如何封装成BeanDefinition然后注册到spring容器的。
