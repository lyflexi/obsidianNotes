# API

API在我们日常开发工作中是比较直观可以看到的，比如在 Spring 项目中，我们通常习惯在写 service 层代码前，添加一个接口层，对于 service 的调用一般也都是基于接口操作，通过依赖注入，可以使用接口实现类的实例。

简单形容就是这样的：
![[Pasted image 20240114151919.png]]
如上图所示，服务调用方无需关心接口的定义与实现，只进行调用即可，**接口、实现类都是由服务提供方提供**。服务提供方提供的接口与其实现方法就可称为**API**，API中所定义的接口无论是在概念上还是具体实现，都更接近服务提供方（实现方），通常接口与实现类在同一包中；
# SPI

如果我们将接口的定义放在调用方，服务的调用方定义一个接口规范，可以由不同的服务提供者实现。并且，调用方能够通过某种机制来发现服务提供方，通过调用接口使用服务提供方提供的功能，这就是SPI的思想。

SPI 的全称是Service Provider Interface，字面意思就是**服务提供者的接口**，是由服务提供者定义的接口。
![[Pasted image 20240114151959.png]]
服务提供方按接口规范实现服务，服务调用方通过某种机制为这个接口寻找到这个服务， SPI的特点很明显：接口的定义（调用方提供）与具体实现是隔离的（服务提供方提供），使用接口的实现类需要依赖某种服务发现机制。

通过对比，我们可以看出接口在**API**与**SPI**中的含义还是有很大的不同，总的来说，API 中的接口是更像是服务提供者给调用者的一个功能列表，==而 SPI 中更多强调的是，服务调用者对服务实现的一种约束。==

# 如何实现SPI
下面就结合一个示例来具体讲讲。若有这样一个需求，需要使用一个接口来完成内容查找服务，接口的具体实现交给其他服务提供方，实现可能是基于文件系统的查找，也可能是基于数据库的查找。
```xml
<modules>  
    <module>ProviderA</module>  
    <module>ProviderB</module>  
    <module>Defining-the-specification</module>  
    <module>searchofServiceLoader</module>  
</modules>
```
## Defining-the-specification模块
**（1）定义接口**

作为服务调用方，需要先定义一套接口规范，用来规范之后的服务提供方按规范来实现接口。这样，不管是谁提供的实现方法，调用方都可以按相同的方式来调用接口。
```java
package spi;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/14 13:31  
 */import java.util.List;  
// 查找服务接口  
public interface Search {  
    // 按关键字查询内容方法  
    String searchDoc(String keyword);  
}
```
将它打包发布mvn clean install，确保maven仓库中有该jar包，之后提供者在项目中就可以引入这个 jar 包了。
## 第三方服务厂商
当服务的提供者，提供了服务接口的一种实现之后，只需要在自己的jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件，该文件的内容就是实现该服务接口的具体实现类。而当外部程序引入厂商提供的依赖时，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并加载实现类，完成依赖的注入，这就是Java SPI的服务发现机制。
### ProviderA模块
**（2）服务实现商File**
```java
package spi;  
  
  
import spi.Search;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/14 13:53  
 */public class FileSearch implements Search {  
  
    @Override  
    public String searchDoc(String keyword) {  
        return "文件查找：" + keyword;  
    }  
}
```
引入Defining-the-specification
```xml
<dependency>  
    <groupId>org.lyflexi</groupId>  
    <artifactId>Defining-the-specification</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
    <scope>compile</scope>  
</dependency>
```

![[Pasted image 20240114165212.png]]
### ProviderB模块
**（3）服务实现商Database**
```java
package spi;  
  
import spi.Search;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/14 13:58  
 */
public class DatabaseSearch implements Search {  
    @Override  
    public String searchDoc(String keyword) {  
        return "数据库查找：" + keyword;  
    }  
}
```
引入Defining-the-specification
```xml
<dependency>  
    <groupId>org.lyflexi</groupId>  
    <artifactId>Defining-the-specification</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
    <scope>compile</scope>  
</dependency>
```
![[Pasted image 20240114165340.png]]
## searchofServiceLoader模块
**（4）服务发现**
我们在Spring项目中使用API时，会使用Spring的依赖注入（DI）来实现“服务发现”，同样地，SPI的重点也是如何让调用方发现接口的具体实现，也就是SPI的服务发现机制。

SPI的服务发现机制是由ServiceLoader提供，ServiceLoader是Java在JDK 6中引进的新特性，它主要是用来发现并加载一系列的service provider。==ServiceLoader会扫描服务厂商的jar包当中的META-INF/services/目录，该目录定义了厂商实现类的信息，ServiceLoader再通过反射实例化厂商的实现类==
```java
package spi;  
  
/**  
 * @Author: ly  
 * @Date: 2024/1/14 14:36  
 */  
import java.util.ServiceLoader;  
  
public class SearchDoc {  
  
    public static void main(String[] args) {  
        new SearchDoc().searchDocByKeyWord("hello world");  
    }  
  
    public void searchDocByKeyWord(String keyWord) {  
  
        ServiceLoader<Search> searchServiceLoader = ServiceLoader.load(Search.class);  
  
        for (Search search : searchServiceLoader){  
            String doc = search.searchDoc(keyWord);  
            System.out.println(doc);  
        }  
    }  
}
```
打印信息：
```java
Connected to the target VM, address: '127.0.0.1:61035', transport: 'socket'
文件查找：hello world
数据库查找：hello world
Disconnected from the target VM, address: '127.0.0.1:61035', transport: 'socket'

Process finished with exit code 0
```

# SPI原理

ServiceLoader定义了如下成员变量，==并实现了Iterable接口==
- 实现iterator()方法
```java
public final class ServiceLoader<S>  
    implements Iterable<S>  
{  
    //扫描厂商jar包中契约接口文件org.lyflexi.spi.ISearch所在的路径
    private static final String PREFIX = "META-INF/services/";  
  
    // The class or interface representing the service being loaded  
    private final Class<S> service;  
  
    // The class loader used to locate, load, and instantiate providers  
    private final ClassLoader loader;  
  
    // The access control context taken when the ServiceLoader is created  
    private final AccessControlContext acc;  
  
    // Cached providers, in instantiation order  保存厂商提供的目标实现类
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();  
  
    // The current lazy-lookup iterator   内部类LazyIterator的单向引用
    private LazyIterator lookupIterator;

	//正是因为ServiceLoader持有了内部类LazyIterator的单向引用lookupIterator，
	//所以iterator()的实现，基于了内部类LazyIterator
	public Iterator<S> iterator() {  
	    return new Iterator<S>() {  
	  
	        Iterator<Map.Entry<String,S>> knownProviders  
	            = providers.entrySet().iterator();  
	  
	        public boolean hasNext() {  
	            if (knownProviders.hasNext())  
	                return true;  
	            return lookupIterator.hasNext();  
	        }  
	  
	        public S next() {  
	            if (knownProviders.hasNext())  
	                return knownProviders.next().getValue();  
	            return lookupIterator.next();  
	        }  
	  
	        public void remove() {  
	            throw new UnsupportedOperationException();  
	        }  
	  
	    };  
	}
}
```
内部类LazyIterator定义，LazyIterator实现了Iterator接口，==注意Iterator接口和Iterable接口是两个不同的接口==
- 实现boolean hasNext();
- 实现E next();
- 实现void remove()
```java

	private class LazyIterator  
	    implements Iterator<S>  
	{  
	  
	    Class<S> service;  
	    ClassLoader loader;  
	    Enumeration<URL> configs = null;  
	    Iterator<String> pending = null;  
	    String nextName = null;  
	  
	    private LazyIterator(Class<S> service, ClassLoader loader) {  
	        this.service = service;  
	        this.loader = loader;  
	    }  
	  
	    private boolean hasNextService() {  
	        if (nextName != null) {  
	            return true;  
	        }  
	        if (configs == null) {  
	            try {  
	                String fullName = PREFIX + service.getName();  
	                if (loader == null)  
	                    configs = ClassLoader.getSystemResources(fullName);  
	                else  
	                    configs = loader.getResources(fullName);  
	            } catch (IOException x) {  
	                fail(service, "Error locating configuration files", x);  
	            }  
	        }  
	        while ((pending == null) || !pending.hasNext()) {  
	            if (!configs.hasMoreElements()) {  
	                return false;  
	            }  
	            pending = parse(service, configs.nextElement());  
	        }  
	        nextName = pending.next();  
	        return true;  
	    }  
	  
	    private S nextService() {  
	        if (!hasNextService())  
	            throw new NoSuchElementException();  
	        String cn = nextName;  
	        nextName = null;  
	        Class<?> c = null;  
	        try {  
	            c = Class.forName(cn, false, loader);  
	        } catch (ClassNotFoundException x) {  
	            fail(service,  
	                 "Provider " + cn + " not found");  
	        }  
	        if (!service.isAssignableFrom(c)) {  
	            fail(service,  
	                 "Provider " + cn  + " not a subtype");  
	        }  
	        try {  
	            S p = service.cast(c.newInstance());  
	            providers.put(cn, p);  
	            return p;  
	        } catch (Throwable x) {  
	            fail(service,  
	                 "Provider " + cn + " could not be instantiated",  
	                 x);  
	        }  
	        throw new Error();          // This cannot happen  
	    }  
	  
	    public boolean hasNext() {  
	        if (acc == null) {  
	            return hasNextService();  
	        } else {  
	            PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {  
	                public Boolean run() { return hasNextService(); }  
	            };  
	            return AccessController.doPrivileged(action, acc);  
	        }  
	    }  
	  
	    public S next() {  
	        if (acc == null) {  
	            return nextService();  
	        } else {  
	            PrivilegedAction<S> action = new PrivilegedAction<S>() {  
	                public S run() { return nextService(); }  
	            };  
	            return AccessController.doPrivileged(action, acc);  
	        }  
	    }  
	  
	    public void remove() {  
	        throw new UnsupportedOperationException();  
	    }  
	  
	}
```
该LazyIterator会找出所有的目标实现类`Class<?> c`，最终通过反射实例化目标类，存入LinkedHashMap providers中，供ServiceLoader的iterator()方法使用
```java
	        try {  
	            S p = service.cast(c.newInstance());  
	            providers.put(cn, p);  
	            return p;  
	        } catch (Throwable x) {  
	            fail(service,  
	                 "Provider " + cn + " could not be instantiated",  
	                 x);  
	        }  
```