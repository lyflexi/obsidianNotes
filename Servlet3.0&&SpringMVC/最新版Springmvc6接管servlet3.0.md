导入新版的springmvc 6.1.2包和jakarta.servlet-api5.0.0
```xml
<!--tomcat10+对应的servlet-->  
<dependency>  
  <groupId>org.springframework</groupId>  
  <artifactId>spring-webmvc</artifactId>  
  <version>6.1.2</version>  
</dependency>  
<dependency>  
  <groupId>jakarta.servlet</groupId>  
  <artifactId>jakarta.servlet-api</artifactId>  
  <version>5.0.0</version>  
  <scope>provided</scope>  
</dependency>
```
查看webjar包，发现springmvc为我们绑定好了Servlet的可插拔组件信息，
![[Pasted image 20240110105818.png]]
打开jakarta.servlet.ServletContainerInitializer文件信息，绑定了一个SpringServletContainerInitializer
```java
org.springframework.web.SpringServletContainerInitializer
```
# springmvc绑定SpringServletContainerInitializer
查看SpringServletContainerInitializer的onStart为我们做了什么事情：
1. 通过注解@HandlesTypes注入了感兴趣的类WebApplicationInitializer.class
2. 遍历出所有的感兴趣类型waiClass，并以此反射出实例
3. 感兴趣的类也属于Initializer，因此再遍历出所有实例的initializer，以次调用实例的onStartup方法
```java
@HandlesTypes(WebApplicationInitializer.class)  
public class SpringServletContainerInitializer implements ServletContainerInitializer {  
	@Override  
    public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)  
          throws ServletException {  
  
       List<WebApplicationInitializer> initializers = Collections.emptyList();  
  
       if (webAppInitializerClasses != null) {  
          initializers = new ArrayList<>(webAppInitializerClasses.size());  
          for (Class<?> waiClass : webAppInitializerClasses) {  
             // Be defensive: Some servlet containers provide us with invalid classes,  
             // no matter what @HandlesTypes says...             if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&  
                   WebApplicationInitializer.class.isAssignableFrom(waiClass)) {  
                try {  
                   initializers.add((WebApplicationInitializer)  
                         ReflectionUtils.accessibleConstructor(waiClass).newInstance());  
                }  
                catch (Throwable ex) {  
                   throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);  
                }  
             }  
          }  
       }  
  
       if (initializers.isEmpty()) {  
          servletContext.log("No Spring WebApplicationInitializer types detected on classpath");  
          return;  
       }  
  
       servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");  
       AnnotationAwareOrderComparator.sort(initializers);  
       for (WebApplicationInitializer initializer : initializers) {  
          initializer.onStartup(servletContext);  
       }  
    }  
  
}
```

因此最终生效的是WebApplicationInitializer.class，我们只需要编写接口实现WebApplicationInitializer即可
## WebApplicationInitializer.class
WebApplicationInitializer接口旗下三层抽象类，接下来依次分析AbstractContextLoaderInitializer，AbstractDispatcherServletInitializer，AbstractAnnotationConfigDispatcherServletInitializer分别干了什么事
![[Pasted image 20240110110802.png]]
### AbstractContextLoaderInitializer
AbstractContextLoaderInitializer：
- 创建根容器；createRootApplicationContext()；  
- 创建监听器，并且将监听器添加到根容器当中
这些操作跟过去配置applicationContext.xml的效果是一样的
```java
@Override  
public void onStartup(ServletContext servletContext) throws ServletException {  
    registerContextLoaderListener(servletContext);  
}  
  
/**  
 * Register a {@link ContextLoaderListener} against the given servlet context. The  
 * {@code ContextLoaderListener} is initialized with the application context returned  
 * from the {@link #createRootApplicationContext()} template method.  
 * @param servletContext the servlet context to register the listener against  
 */protected void registerContextLoaderListener(ServletContext servletContext) {  
    WebApplicationContext rootAppContext = createRootApplicationContext();  
    if (rootAppContext != null) {  
       ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);  
       listener.setContextInitializers(getRootApplicationContextInitializers());  
       servletContext.addListener(listener);  
    }  
    else {  
       logger.debug("No ContextLoaderListener registered, as " +  
             "createRootApplicationContext() did not return an application context");  
    }  
}
```
### AbstractDispatcherServletInitializer
AbstractDispatcherServletInitializer：  
- 创建一个web的ioc容器；createServletApplicationContext();  
- 创建了DispatcherServlet；并且将DispatcherServlet添加到web的ioc容器当中
- getServletMappings()配置映射信息。  ==但是getServletMappings()方法并没有实现，是为了留给子类扩展==
```java
@Override  
public void onStartup(ServletContext servletContext) throws ServletException {  
    super.onStartup(servletContext);  
    registerDispatcherServlet(servletContext);  
}  
  
/**  
 * Register a {@link DispatcherServlet} against the given servlet context.  
 * <p>This method will create a {@code DispatcherServlet} with the name returned by  
 * {@link #getServletName()}, initializing it with the application context returned  
 * from {@link #createServletApplicationContext()}, and mapping it to the patterns  
 * returned from {@link #getServletMappings()}.  
 * <p>Further customization can be achieved by overriding {@link  
 * #customizeRegistration(ServletRegistration.Dynamic)} or  
 * {@link #createDispatcherServlet(WebApplicationContext)}.  
 * @param servletContext the context to register the servlet against  
 */protected void registerDispatcherServlet(ServletContext servletContext) {  
    String servletName = getServletName();  
    Assert.state(StringUtils.hasLength(servletName), "getServletName() must not return null or empty");  
  
    WebApplicationContext servletAppContext = createServletApplicationContext();  
    Assert.state(servletAppContext != null, "createServletApplicationContext() must not return null");  
  
    FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);  
    Assert.state(dispatcherServlet != null, "createDispatcherServlet(WebApplicationContext) must not return null");  
    dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());  
  
    ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);  
    if (registration == null) {  
       throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +  
             "Check if there is another servlet registered under the same name.");  
    }  
  
    registration.setLoadOnStartup(1);  
    registration.addMapping(getServletMappings());  
    registration.setAsyncSupported(isAsyncSupported());  
  
    Filter[] filters = getServletFilters();  
    if (!ObjectUtils.isEmpty(filters)) {  
       for (Filter filter : filters) {  
          registerServletFilter(servletContext, filter);  
       }  
    }  
  
    customizeRegistration(registration);  
}


protected abstract String[] getServletMappings();
```
### AbstractAnnotationConfigDispatcherServletInitializer：注解方式配置的DispatcherServlet初始化器  
1. 创建根容器：重写了createRootApplicationContext()  
	1. getRootConfigClasses();传入一个配置类  
1. 创建web的ioc容器： 重写了createServletApplicationContext();  
	1. 获取配置类；getServletConfigClasses();  传入一个配置类  

```java
@Override  
@Nullable  
protected WebApplicationContext createRootApplicationContext() {  
    Class<?>[] configClasses = getRootConfigClasses();  
    if (!ObjectUtils.isEmpty(configClasses)) {  
       AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();  
       context.register(configClasses);  
       return context;  
    }  
    else {  
       return null;  
    }  
}  
  
/**  
 * {@inheritDoc}  
 * <p>This implementation creates an {@link AnnotationConfigWebApplicationContext},  
 * providing it the annotated classes returned by {@link #getServletConfigClasses()}.  
 */@Override  
protected WebApplicationContext createServletApplicationContext() {  
    AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();  
    Class<?>[] configClasses = getServletConfigClasses();  
    if (!ObjectUtils.isEmpty(configClasses)) {  
       context.register(configClasses);  
    }  
    return context;  
}
```

# 实现接管，定制servlet
以注解方式来启动SpringMVC；干脆继承功能最多的这个抽象类AbstractAnnotationConfigDispatcherServletInitializer；  
实现抽象方法指定DispatcherServlet的配置信息；  
```java
package org.lyflexi.debug_springmvc.mvcintegration;  
  
import org.lyflexi.debug_springmvc.mvcintegration.config.AppConfig;  
import org.lyflexi.debug_springmvc.mvcintegration.config.RootConfig;  
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;  
  
//web容器启动的时候创建对象；调用方法来初始化容器以前前端控制器  
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {  
  
    //获取根容器的配置类；（相当于Spring的配置文件）   父容器；  
    @Override  
    protected Class<?>[] getRootConfigClasses() {  
       // TODO Auto-generated method stub  
       return new Class<?>[]{RootConfig.class};  
    }  
  
    //获取web容器的配置类（相当于SpringMVC的配置文件）  子容器；  
    @Override  
    protected Class<?>[] getServletConfigClasses() {  
       // TODO Auto-generated method stub  
       return new Class<?>[]{AppConfig.class};  
    }  
  
    //获取DispatcherServlet的映射信息  
    //  /：拦截所有请求（包括静态资源（xx.js,xx.png）），但是不包括*.jsp；  
    //  /*：拦截所有请求；连*.jsp页面都拦截；jsp页面是tomcat的jsp引擎解析的；  
    @Override  
    protected String[] getServletMappings() {  
       // TODO Auto-generated method stub  
       return new String[]{"/"};  
    }  
  
}


```
## 根配置类RootConfig如下：

```java
package org.lyflexi.debug_springmvc.mvcintegration.config;  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.ComponentScan.Filter;  
import org.springframework.context.annotation.FilterType;  
import org.springframework.stereotype.Controller;  
  
//Spring的容器不扫描controller;父容器  
@ComponentScan(value="org.lyflexi.debug_springmvc.mvcintegration",excludeFilters={  
       @Filter(type=FilterType.ANNOTATION,classes={Controller.class})  
})  
public class RootConfig {  
  
}
```
## Web配置类AppConfig如下：
  - @EnableWebMvc:开启SpringMVC定制配置功能；等效于xml版本的<mvc:annotation-driven/>； 
  - 配置servlet组件（视图解析器、视图映射、静态资源映射、拦截器。。。）
```java
package org.lyflexi.debug_springmvc.mvcintegration.config;  
  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.ComponentScan.Filter;  
import org.springframework.context.annotation.FilterType;  
import org.springframework.stereotype.Controller;  
import org.springframework.web.servlet.config.annotation.*;  
  
//SpringMVC只扫描Controller；子容器  
//要想满足只扫描，还需要禁用默认的过滤规则useDefaultFilters=false ；  

@ComponentScan(value="org.lyflexi.debug_springmvc.mvcintegration",includeFilters={  
       @Filter(type=FilterType.ANNOTATION,classes={Controller.class})  
},useDefaultFilters=false)  
@EnableWebMvc  
//public class AppConfig  extends WebMvcConfigurerAdapter  {  
//WebMvcConfigurerAdapter该抽象类从Spring5.0，也就是SpringBoot2.0后，已被弃用。替代方案幽有两种：  
//1.实现WebMvcConfigurer接口  
//2.继承WebMvcConfigurationSupport类  
public class AppConfig  implements WebMvcConfigurer  {  
    //实现WebMvcConfigurerAdapter，相当于手写mvc.xml，来定制mvc  
  
    //定制  
    //视图解析器  
    @Override  
    public void configureViewResolvers(ViewResolverRegistry registry) {  
       // TODO Auto-generated method stub  
       //由controller的return返回的字符串拼接jsp路径，默认所有的jsp页面都从 /WEB-INF/ 下找  
//     registry.jsp();  
       //由controller的return返回的字符串拼接jsp路径，改成从/WEB-INF/views/下找jsp  
       registry.jsp("/WEB-INF/views/", ".jsp");  
    }  
      
    //静态资源访问  
    @Override  
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {  
       // TODO Auto-generated method stub  
       //将SpringMVC处理不了的静态资源请求交给tomcat；比如png图片 等效于xml时代的<mvc:default-servlet-handler/>  
       configurer.enable();  
    }  
      
    //拦截器  
    @Override  
    public void addInterceptors(InterceptorRegistry registry) {  
       // TODO Auto-generated method stub  
//     super.addInterceptors(registry);  
       registry.addInterceptor(new MyFirstInterceptor()).addPathPatterns("/**");  
    }  
}
```
最后写一个controller测试
```java
package org.lyflexi.debug_springmvc.mvcintegration.controller;  
  
  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Controller;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.ResponseBody;  
import org.lyflexi.debug_springmvc.mvcintegration.service.HelloService;  
  
@Controller  
public class HelloController {  
      
    @Autowired  
    HelloService helloService;  
      
      
    @ResponseBody  
    @RequestMapping("/hello")  
    public String hello(){  
       String hello = helloService.sayHello("tomcat..");  
       return hello;  
    }  
      
  
    @RequestMapping("/suc")  
    public String success(){  
       //相当于找/WEB-INF/views/success.jsp  
       return "success";  
    }  
      
  
}
```
