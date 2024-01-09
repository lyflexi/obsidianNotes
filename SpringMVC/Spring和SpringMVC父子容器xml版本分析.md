在Spring和Springmvc进行整合的时候，一般情况下我们会使用不同的配置文件来配置Spring和SpringMVC，因此我们的应用中会存在至少2个ApplicationContext实例，由于是在web应用中，因此最终实例化的是ApplicationContext的子接口WebApplicationContext。如下图所示：
![[Pasted image 20240108160337.png]]

上图中显示了2个WebApplicationContext实例，为了进行区分，分别称之为：Servlet WebApplicationContext、Root WebApplicationContext。 其中：

- Servlet WebApplicationContext：这是对J2EE三层架构中的web层进行配置，如控制器(controller)、视图解析器(view resolvers)等相关的bean。通过spring mvc中提供的DispatchServlet来加载配置，通常情况下，配置文件的名称为`spring-servlet.xml`。
    
- Root WebApplicationContext：这是对J2EE三层架构中的service层、dao层进行配置，如业务bean，数据源(DataSource)等。通常情况下，配置文件的名称为`applicationContext.xml`。在web应用中，其一般通过`ContextLoaderListener`来加载。
    

以下是一个web.xml配置案例：

```XML
<?xml version="1.0" encoding="UTF-8"?>  
  
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaeehttp://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">  
    
    <!—创建Root WebApplicationContext-->
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>/WEB-INF/spring/applicationContext.xml</param-value>  
    </context-param>  
  
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>  
    
    <!—创建Servlet WebApplicationContext-->
    <servlet>  
        <servlet-name>dispatcher</servlet-name>  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <init-param>  
            <param-name>contextConfigLocation</param-name>  
            <param-value>/WEB-INF/spring/spring-servlet.xml</param-value>  
        </init-param>  
        <load-on-startup>1</load-on-startup>  
    </servlet>  
    <!—Servlet 映射-->
    <servlet-mapping>  
        <servlet-name>dispatcher</servlet-name>  
        <url-pattern>/*</url-pattern>  
    </servlet-mapping>  
  
</web-app>
```

在上面的配置中：
1. 父容器Root WebApplicationContext会先被初始化，根据`<context-param>`元素中contextConfigLocation参数指定的配置文件路径，在这里就是`"/WEB-INF/spring/applicationContext.xml”`。 
	1. 内部创建ContextLoaderListener，并添加到父容器当中
2. 子容器Servlet WebApplicationContext后背初始化，会根据`<init-param>`元素中contextConfigLocation参数指定的配置文件路径，即`"/WEB-INF/spring/spring-servlet.xml”`
	1. 内部创建DispatcherServlet，并添加到子容器当中
	2. 注册servlet信息，如映射规则等
注意事项：
- 当我们尝试从child context(即：Servlet WebApplicationContext)中获取一个bean时，如果找不到，则会委派给parent context (即Root WebApplicationContext)来查找。
- 如果第一步中，我们没有通过ContextLoaderListener来创建Root WebApplicationContext，那么Servlet WebApplicationContext的parent就是null，也就是没有parent context。

# **为什么要有父子容器**

笔者理解，父子容器的作用主要是划分框架边界。

在J2EE三层架构中，在service层我们一般使用spring框架， 而在web层则有多种选择，如spring mvc、struts等。因此，通常对于web层我们会使用单独的配置文件。例如在上面的案例中，一开始我们使用spring-servlet.xml来配置web层，使用applicationContext.xml来配置service、dao层。如果现在我们想把web层从spring mvc替换成struts，那么只需要将spring-servlet.xml替换成Struts的配置文件struts.xml即可，而applicationContext.xml不需要改变。

==事实上，如果你的项目确定了只使用spring和spring mvc的话，你甚至可以将service 、dao、web层的bean都放到子容器spring-servlet.xml中进行配置，并不是一定要将service、dao层的配置单独放到applicationContext.xml中，然后使用ContextLoaderListener来加载==。在这种情况下，就没有了Root WebApplicationContext，只有Servlet WebApplicationContext。

# 正确配置父子容器的包扫描

## **为什么不能在Spring的applicationContext.xml中配置全局扫描**

如果都在Spring容器中，这时的SpringMVC容器中没有对象。因为在解析@ReqestMapping解析过程中，initHandlerMethods()函数只是对Spring MVC 容器中的bean进行处理的，并没有去查找父容器的bean。因此不会对父容器中含有@RequestMapping注解的函数进行处理，更不会生成相应的处理器Handler（映射器），适配器Adapter，也就会找不到映射对象，映射关系，因此在页面上就会出现404的错误。

## **为什么不能同时扫描**

会在两个父子IOC容器中生成大量的相同bean，这就会造成内存资源的浪费。

## 正确**的扫描**

因为@RequestMapping一般会和@Controller搭配使。为了防止重复注册bean，建议在spring-servlet.xml配置文件中只扫描含有Controller bean的包，其它的共用bean的注册定义到applicationContext.xml文件中。

applicationContext.xml：排除子容器的Controller组件`exclude-filter type``="annotation"`

```XML
<context:component-scan base-package="org.xuan.springmvc"
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
                                
</context:component-scan>
```

spring-servlet.xml

```XML
<context:component-scan base-package="org.xuan.springmvc.controller" use-default-filters="false"
          <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

# 父子配置文件独立

Spring容器导入的properties配置文件，只能在Spring容器中用而在SpringMVC容器中不能读取到。

需要在SpringMVC 的配置文件中重新进行导入properties文件，并且同样在父容器Spring中不能被使用。

导入后使用@Value("${key}")在java类中进行读取。