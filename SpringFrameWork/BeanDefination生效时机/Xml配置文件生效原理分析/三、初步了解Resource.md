1、接着看上一节的loadBeanDefinitions，
- 这个方法的前几行都是==构建beanDefinitionReader==，
- 最后一行则是调用loadBeanDefinitions进行xml的解析和注册(loadBeanDefinitions是个重载方法)，下面重点看这个方法。
```java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

```
![[Pasted image 20231228154801.png]]
## ClassPathXmlApplicationContext
2、loadBeanDefinitions(beanDefinitionReader)主要就是调用XmlBeanDefinitionReader的loadBeanDefinitions方法，加载xml之前会先通过getConfigLocations去获取我们的xml路径
```java
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}

	protected String[] getConfigLocations() {
		return (this.configLocations != null ? this.configLocations : getDefaultConfigLocations());
	}

```

3、==至于configLocations的值是怎么来的，我们先看回main方法在new ClassPathXmlApplicationContext时把xml给传了进去。然后在构造方法里调用setConfigLocations，把xml给设置到configLocations==
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

public class ClassPathXmlApplicationContext{
    //构造方法
	public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		//调用重载的构造方法
		this(new String[] {configLocation}, true, null);
	}
	
	//重载的构造方法
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		//把xml设置到configLocations
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}

  //父类方法，把locations设置到configLocations
	public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
}

```
# XmlBeanDefinitionReader
4、XmlBeanDefinitionReader的loadBeanDefinitions方法也是个重载方法，通过遍历locations，最终会调用到下面的loadBeanDefinitions方法。这个方法虽然有点长，但其实就是用resourceLoader解析location得到Resource或者Resource数组，那什么是Resource呢？
```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
	}

```
5、Resource其实就是spring对一些资源文件的一个统一定义，我们知道能通过它获取到文件流供我们操作就可以了。
```java
public interface Resource extends InputStreamSource {
    boolean exists();

    default boolean isReadable() {
        return this.exists();
    }

    default boolean isOpen() {
        return false;
    }

    default boolean isFile() {
        return false;
    }

    URL getURL() throws IOException;

    URI getURI() throws IOException;

    File getFile() throws IOException;

    default ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(this.getInputStream());
    }

    long contentLength() throws IOException;

    long lastModified() throws IOException;

    Resource createRelative(String var1) throws IOException;

    @Nullable
    String getFilename();

    String getDescription();
}

```
6、结语：本节主要讲了在new ClassPathXmlApplicationContext时会把指定的xml赋值给configLocations，然后在refresh创建beanfactory时构建一个XmlBeanDefinitionReader，这个reader在执行loadBeanDefinitions方法时会遍历configLocations，然后在加载前会把这些location封装成Resource对象。
