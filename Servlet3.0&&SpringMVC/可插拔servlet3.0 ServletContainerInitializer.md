
  
Servlet3.0官方支持第三方插件植入，Shared libraries（共享库） / runtimes pluggability（运行时插件能力）:  

==可插拔能力可以抛弃web.xml再结合spring的注解==，从而实现了接口的插拔，此时要想扩展功能就必须实现servlet为我们提供的ServletContainerInitializer接口，示例如下：
```java
package lyflexi.servlet_initializer;  
  
  
  
import jakarta.servlet.*;  
import jakarta.servlet.annotation.HandlesTypes;  
import jakarta.servlet.ServletException;  
import lyflexi.service.HelloService;  
  
import java.util.EnumSet;  
import java.util.Set;  
  
//容器启动的时候会将@HandlesTypes指定的这个类型下面的子类（实现类，子接口等）传递过来；  
//传入感兴趣的类型；  
@HandlesTypes(value={HelloService.class})  
public class MyServletContainerInitializer implements ServletContainerInitializer {  
  
    /**  
     * 应用启动的时候，会运行onStartup方法；  
     *   
     * Set<Class<?>> arg0：感兴趣的类型的所有子类型；  
     * ServletContext arg1:代表当前Web应用的ServletContext；一个Web应用一个ServletContext；  
     *  
     * 使用编码的方式，给项目中注册组件的方式  
     *        必须在项目启动的时候来添加，启动后再添加是无效的  
     *        1）、使用ServletContainerInitializer得到的ServletContext，注册Web组件（Servlet、Filter、Listener）  
     *        2）、使用ServletContextListener得到的ServletContext；，注册Web组件（Servlet、Filter、Listener）  
     */  
    @Override  
    public void onStartup(Set<Class<?>> arg0, ServletContext sc) throws ServletException {  
       System.out.println("onStartup。。。");  
       // TODO Auto-generated method stub  
       System.out.println("感兴趣的类型：");  
       for (Class<?> claz : arg0) {  
          System.out.println(claz);  
       }  
         
       //注册组件  返回ServletRegistration.Dynamic  
       ServletRegistration.Dynamic servlet = sc.addServlet("userServlet", new UserServlet());  
       //配置servlet的映射信息  
       servlet.addMapping("/user");  
  
  
       //注册Listener  
       sc.addListener(UserListener.class);  
  
  
       //注册Filter  返回FilterRegistration.Dynamic  
       FilterRegistration.Dynamic filter = sc.addFilter("userFilter", UserFilter.class);  
       //配置Filter的映射信息  
       filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");  
    }  
}
```
打印如下：
```java
感兴趣的类型：
interface lyflexi.service.HelloServiceExt
class lyflexi.service.HelloServiceImpl
class lyflexi.service.AbstractHelloService
```
@HandlesTypes注解传入的感兴趣类型（包含所有的子类型），会传递给重写方法onStartup的第一个参数`Set<Class<?>> arg0`，后期可以遍历arg0，依次反射出每一个感兴趣的类型

重写方法onStartup的第二个参数是上下文sc，通过sc我们可以注册servlet以及servlet的组件（如Listener和Filter）
# 如何生效
Servlet容器启动会扫描，当前应用里面每一个jar包的ServletContainerInitializer的实现 
1. 老版本tomcat10-javax.servlet_initializer-api下必须绑定在META-INF/services/javax.servlet_initializer.ServletContainerInitializer  
2. 新版本tomcat10+jakarta.servlet-api下必须绑定在，META-INF/services/jakarta.servlet_initializer.ServletContainerInitializer ，同时META-INF要放在idea的resources目录中

文件名就是jakarta.servlet_initializer.ServletContainerInitializer
文件的内容就是ServletContainerInitializer实现类的全类名lyflexi.servlet_initializer.MyServletContainerInitializer； 

