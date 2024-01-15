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
## spi打破了双亲委派机制

上述例子中，通过ServiceLoader.load(ISearch.class) 来加载ISearch接口的实现类，==我们知道Java加载类都离不开类加载器，查看ServiceLoader.load()方法的源码就会发现==，spi加载类是通过java.lang.Thread#setContextClassLoader方法设置了线程上下文加载器
```java
public static <S> ServiceLoader<S> load(Class<S> service,  
                                        ClassLoader loader)  
{  
    return new ServiceLoader<>(service, loader);  
}  

//load是个重载方法
public static <S> ServiceLoader<S> load(Class<S> service) {  
    ClassLoader cl = Thread.currentThread().getContextClassLoader();  
    return ServiceLoader.load(service, cl);  
}
```

正常情况下，我们的Java类若未设置类加载器，则会从父线程中继承，在应用程序全局都未设置的情况下，默认是应用程序类加载器，即
1. JDK 中的本地方法类一般由根加载器（Bootstrp loader）装载，Java核心类库位于<JAVA_HOME>\lib目录中
2. JDK 中内部实现的扩展类一般由扩展加载器（ExtClassLoader ）实现装载，位于<JAVA_HOME>\lib\ext目录
3. 而程序中的类文件则由系统加载器（AppClassLoader ）实现装载。

==而SPI使用了线程上下文加载器加载所需的SPI代码，实际上是父类加载器请求子类加载器来完成加载类的动作，打破了双亲委派模型的层次结构。==

# SPI业界的应用案例

## 日志框架slf4j

SPI的实际应用，最常见的应该是日志框架slf4j，它就是个日志门面，并不提供具体的实现，需要绑定其他具体实现。例如可使用log4j2作为具体的绑定器，只需要在 pom 中引入slf4j-log4j12，就可以使用具体功能。
```xml
```text
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.3</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>2.0.3</version>
</dependency>
```
引入项目后，点开它的 jar 包看一下具体结构：
### slf4j-api（门面）定义spi
首先看下门面依赖的jar包结构，在spi目录中定义了许多的契约接口
![[Pasted image 20240114195753.png]]
### slf4j-log4j12（厂商）实现spi
在来看下具体的厂商实现的jar，slf4j-log4j12
![[Pasted image 20240114200902.png]]
jar 包的META-INF.services里面，通过 SPI 注入了Reload4jServiceProvider这个实现类，它实现了SLF4JServiceProvider这一接口，在它的初始化方法initialize()中，会完成初始化等工作，后续可以继续获取到LoggerFactory和Logger等具体日志对象。
```java
  
public class Reload4jServiceProvider implements SLF4JServiceProvider {  
    public static String REQUESTED_API_VERSION = "2.0.99";  
    private ILoggerFactory loggerFactory;  
    private IMarkerFactory markerFactory;  
    private MDCAdapter mdcAdapter;  
  
    public Reload4jServiceProvider() {  
        try {  
            Level var1 = Level.TRACE;  
        } catch (NoSuchFieldError var2) {  
            Util.report("This version of SLF4J requires log4j version 1.2.12 or later. See also http://www.slf4j.org/codes.html#log4j_version");  
        }  
  
    }  
  
    public void initialize() {  
        this.loggerFactory = new Reload4jLoggerFactory();  
        this.markerFactory = new BasicMarkerFactory();  
        this.mdcAdapter = new Reload4jMDCAdapter();  
    }  
  
    public ILoggerFactory getLoggerFactory() {  
        return this.loggerFactory;  
    }  
  
    public IMarkerFactory getMarkerFactory() {  
        return this.markerFactory;  
    }  
  
    public MDCAdapter getMDCAdapter() {  
        return this.mdcAdapter;  
    }  
  
    public String getRequestedApiVersion() {  
        return REQUESTED_API_VERSION;  
    }  
}
```

## 数据库驱动mysql-connector-java
DriverManager是JDBC里管理数据库驱动的的工具类。一个数据库可能会存在不同实现的数据库驱动。我们在使用特定的驱动实现时，通过一个简单的配置就而不用修改代码就可以达到效果。 
引入依赖：
```xml
<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->  
<dependency>  
    <groupId>mysql</groupId>  
    <artifactId>mysql-connector-java</artifactId>  
    <version>5.1.44</version>  
</dependency>
```
查看mysql-connector-java的jar包，通过META-INF/services/java.sql.Driver加载了com.mysql.jdbc.Driver  
![[Pasted image 20240114200812.png]]

查看com.mysql.jdbc.Driver  源码，发现源码当中使用DriverManager的静态方法加载了该厂商的实现类com.mysql.jdbc.Driver。
### java.sql.Driver（门面）定义spi
```java

package java.sql;  
  
import java.util.logging.Logger;  
  

 public interface Driver {  
  

     
     Connection connect(String url, java.util.Properties info)  
        throws SQLException;  
     boolean acceptsURL(String url) throws SQLException;  

     DriverPropertyInfo[] getPropertyInfo(String url, java.util.Properties info)  
                         throws SQLException;  

     int getMajorVersion();  
  

     int getMinorVersion();  
  
   
     boolean jdbcCompliant();  
  
    //------------------------- JDBC 4.1 -----------------------------------  
  

     public Logger getParentLogger() throws SQLFeatureNotSupportedException;  
}
```
### com.mysql.jdbc.Driver（厂商）实现spi
com.mysql.jdbc.Driver
Driver extends NonRegisteringDriver，实现了java.sql.Driver规定的的所有接口
```java
package com.mysql.jdbc;  
  
import java.sql.SQLException;

public class Driver extends NonRegisteringDriver implements java.sql.Driver {  
    //  
    // Register ourselves with the DriverManager    //    
    static {  
        try {  
            java.sql.DriverManager.registerDriver(new Driver());  
        } catch (SQLException E) {  
            throw new RuntimeException("Can't register driver!");  
        }  
    }  
  
    /**  
     * Construct a new driver and register it with DriverManager     ** @throws SQLException  
     *             if a database error occurs.     */ 

	
     public Driver() throws SQLException {  
        // Required for Class.forName().newInstance()  
    }  
}
```
//由于ServiceLoader的底层是用无参的反射方法newInstance()来创建com.mysql.jdbc.Driver实例，==因此com.mysql.jdbc.Driver提供了一个无参的构造==
```java
//jdk8
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

#### 第一步：java.sql.DriverManager jdk使用ServiceLoader创建厂商驱动
查看DriverManager源码，jvm在启动的时候，根加载器会加载java.sql.DriverManager，首先执行java.sql.DriverManager的静态代码块
```java
 static {  
    loadInitialDrivers();  
    println("JDBC DriverManager initialized");  
}
```

loadInitialDrivers方法中创建了ServiceLoader，ServiceLoader#iterator()的底层迭代出厂商所有的mysql驱动实现类并通过反射创建驱动
```java
//jdk8

/**  
 * Load the initial JDBC drivers by checking the System property * jdbc.properties and then use the {@code ServiceLoader} mechanism  
 */
 static {  
    loadInitialDrivers();  
    println("JDBC DriverManager initialized");  
}

private static void loadInitialDrivers() {  
    String drivers;  
    try {  
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {  
            public String run() {  
                return System.getProperty("jdbc.drivers");  
            }  
        });  
    } catch (Exception ex) {  
        drivers = null;  
    }  
    // If the driver is packaged as a Service Provider, load it.  
    // Get all the drivers through the classloader    // exposed as a java.sql.Driver.class service.    // ServiceLoader.load() replaces the sun.misc.Providers()  
    AccessController.doPrivileged(new PrivilegedAction<Void>() {  
        public Void run() {  
  
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);  
            Iterator<Driver> driversIterator = loadedDrivers.iterator();  
  
            /* Load these drivers, so that they can be instantiated.  
             * It may be the case that the driver class may not be there             * i.e. there may be a packaged driver with the service class             * as implementation of java.sql.Driver but the actual class             * may be missing. In that case a java.util.ServiceConfigurationError             * will be thrown at runtime by the VM trying to locate             * and load the service.             *             * Adding a try catch block to catch those runtime errors             * if driver not available in classpath but it's             * packaged as service and that service is there in classpath.             */            
            try{  
                while(driversIterator.hasNext()) {  
                    driversIterator.next();  
                }  
            } catch(Throwable t) {  
            // Do nothing  
            }  
            return null;  
        }  
    });  
  
    println("DriverManager.initialize: jdbc.drivers = " + drivers);  
  
    if (drivers == null || drivers.equals("")) {  
        return;  
    }  
    //jdk保障机制，jdk帮我们加载驱动，即使用户忘记Class.forName("com.mysql.jdbc.Driver");,依然保证驱动加载成功 
    String[] driversList = drivers.split(":");  
    println("number of Drivers:" + driversList.length);  
    for (String aDriver : driversList) {  
        try {  
            println("DriverManager.Initialize: loading " + aDriver);  
            Class.forName(aDriver, true,  
                    ClassLoader.getSystemClassLoader());  
        } catch (Exception ex) {  
            println("DriverManager.Initialize: load failed: " + ex);  
        }  
    }  
}
```
但是与此同时DriverManage内部创建的ServiceLoader打破了Java的双亲委派机制：
因为DriverManager是java.sql包中的，所以DriverManager本身是被启动类根加载器加载的，只是在加载DriverManager类的时候会触发调用static方法，在static方法中使用的是SPI机制（创建了ServiceLoader打破了Java的双亲委派模型）来切换到上下文类加载器，然后使用上下文类加载器来加载classpath下的MySQL驱动（反射实现厂商类c.newInstance()）。
![[Pasted image 20240114204416.png]]
#### 多了一步：java.sql.DriverManager 注册驱动给用户使用，因此用户最终使用的是java.sql.DriverManager的API#getConnection
==用户可以这样去注册Driver==
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class TempMain {
	// 连接数据库URL格式为：jdbc协议:数据库子协议:主机:端口/连接的数据库名
	// 和HTTP协议类似
	private static String url = "jdbc:mysql://localhost:3306/mydb";
	private static String user = "root";// 用户名
	private static String password = "root";// 密码

	// 第三种方法：使用驱动管理器类连接数据库
	public static void main(String[] args) throws SQLException, ClassNotFoundException {
		// 1.通过得到字节码对象的方式加载静态代码块，从而注册驱动程序
		Class.forName("com.mysql.jdbc.Driver"); // 参数是字节码

		// 2.连接到具体的数据库
		Connection conn = DriverManager.getConnection(url, user, password);
		System.out.println(conn);
		//输出：com.mysql.jdbc.JDBC4Connection@50675690，表明连接成功
	}
}

```
==我们在运用Class.forName("com.mysql.jdbc.Driver")加载com.mysql.jdbc.Driver的字节码的时候，会自动执行其中的静态代码把driver注册到DriverManager中==
```java
package com.mysql.jdbc;  
  
import java.sql.SQLException;

public class Driver extends NonRegisteringDriver implements java.sql.Driver {  
    //  
    // Register ourselves with the DriverManager    //    
    static {  
        try {  
            java.sql.DriverManager.registerDriver(new Driver());  
        } catch (SQLException E) {  
            throw new RuntimeException("Can't register driver!");  
        }  
    }  
  
    /**  
     * Construct a new driver and register it with DriverManager     ** @throws SQLException  
     *             if a database error occurs.     */ 

	
     public Driver() throws SQLException {  
        // Required for Class.forName().newInstance()  
    }  
}
```
进而DriverManager可以拿到Driver的引用，同时DriverManager对成员变量Driver做了一层包装`private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();`，进而我们可以使用DriverManager#getConnection方便的操作数据库连接
```java
//java8
public class DriverManager {
	// List of registered JDBC drivers  
	private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();  
	private static volatile int loginTimeout = 0;  
	private static volatile java.io.PrintWriter logWriter = null;  
	private static volatile java.io.PrintStream logStream = null;  
	// Used in println() to synchronize logWriter  
	private final static  Object logSync = new Object();  
	  
	/* Prevent the DriverManager class from being instantiated. */  
	private DriverManager(){}  
	  
	  
	/**  
	 * Load the initial JDBC drivers by checking the System property * jdbc.properties and then use the {@code ServiceLoader} mechanism  
	 */static {  
	    loadInitialDrivers();  
	    println("JDBC DriverManager initialized");  
	}



		//  Worker method called by the public getConnection() methods.  
	private static Connection getConnection(String url, java.util.Properties info, Class<?> caller) throws SQLException {  
	    /*  
	     * When callerCl is null, we should check the application's     * (which is invoking this class indirectly)     * classloader, so that the JDBC driver class outside rt.jar     * can be loaded from here.     */    
	    ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;  
	    synchronized(DriverManager.class) {  
	        // synchronize loading of the correct classloader.  
	        if (callerCL == null) {  
	            callerCL = Thread.currentThread().getContextClassLoader();  
	        }  
	    }  
	  
	    if(url == null) {  
	        throw new SQLException("The url cannot be null", "08001");  
	    }  
	  
	    println("DriverManager.getConnection(\"" + url + "\")");  
	  
	    // Walk through the loaded registeredDrivers attempting to make a connection.  
	    // Remember the first exception that gets raised so we can reraise it.    
	    SQLException reason = null;  
	  
	    for(DriverInfo aDriver : registeredDrivers) {  
	        // If the caller does not have permission to load the driver then  
	        // skip it.        
	        if(isDriverAllowed(aDriver.driver, callerCL)) {  
	            try {  
	                println("    trying " + aDriver.driver.getClass().getName());  
	                Connection con = aDriver.driver.connect(url, info);  
	                if (con != null) {  
	                    // Success!  
	                    println("getConnection returning " + aDriver.driver.getClass().getName());  
	                    return (con);  
	                }  
	            } catch (SQLException ex) {  
	                if (reason == null) {  
	                    reason = ex;  
	                }  
	            }  
	  
	        } else {  
	            println("    skipping: " + aDriver.getClass().getName());  
	        }  
	  
	    }  
	  
	    // if we got here nobody could connect.  
	    if (reason != null)    {  
	        println("getConnection failed: " + reason);  
	        throw reason;  
	    }  
	  
	    println("getConnection: no suitable driver found for "+ url);  
	    throw new SQLException("No suitable driver found for "+ url, "08001");  
	}
}

```

上述代码可以清楚的看到，DriverManager#getConnection可以方便操作数据库连接，但是DriverManager#getConnection其实依然使用的是注册进来的com.mysql.jdbc.Driver驱动（registeredDrivers）。因此接下来要探讨的问题是？Driver是如何被包装成registeredDrivers的
```java
public static synchronized void registerDriver(java.sql.Driver driver)  
    throws SQLException {  
  
    registerDriver(driver, null);  
}  
  
/**  
 * Registers the given driver with the {@code DriverManager}.  
 * A newly-loaded driver class should call * the method {@code registerDriver} to make itself  
 * known to the {@code DriverManager}. If the driver is currently  
 * registered, no action is taken. * * @param driver the new JDBC Driver that is to be registered with the  
 *               {@code DriverManager}  
 * @param da     the {@code DriverAction} implementation to be used when  
 *               {@code DriverManager#deregisterDriver} is called  
 * @exception SQLException if a database access error occurs  
 * @exception NullPointerException if {@code driver} is null  
 * @since 1.8  
 */public static synchronized void registerDriver(java.sql.Driver driver,  
        DriverAction da)  
    throws SQLException {  
  
    /* Register the driver if it has not already been added to our list */  
    if(driver != null) {  
        registeredDrivers.addIfAbsent(new DriverInfo(driver, da));  
    } else {  
        // This is for compatibility with the original DriverManager  
        throw new NullPointerException();  
    }  
  
    println("registerDriver: " + driver);  
  
}
```
