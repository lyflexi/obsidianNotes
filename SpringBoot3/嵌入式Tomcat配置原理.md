Servlet容器：管理、运行Servlet组件（Servlet、Filter、Listener）的环境


引入web场景spring-boot-starter-web，spring.factories中以下14个web相关的自动配置类就会生效
![[Pasted image 20240129100800.png]]
其中与tomcat相关的是这两个自动配置类：
- `ServletWebServerFactoryAutoConfiguration`
- `EmbeddedWebServerFactoryCustomizerAutoConfiguration`
# ServletWebServerFactoryAutoConfiguration
`ServletWebServerFactoryAutoConfiguration` 自动配置了嵌入式容器场景
```java
@AutoConfiguration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
    
}
```
ServletWebServerFactoryAutoConfiguration自动配置了如下内容：
1. 绑定了`ServerProperties`配置类，前缀是 `server`，因此修改yml的`server`配置就可以修改服务器参数
2. 通过ServletWebServerFactoryConfiguration导入了 嵌入式的三大服务器 `Tomcat`、`Jetty`、`Undertow`
## ServletWebServerFactoryConfiguration
ServletWebServerFactoryConfiguration定义如下：
导入的 `Tomcat`、`Jetty`、`Undertow` 都有条件注解。系统中有这个类才行（也就是导了包），Jetty和Undertow默认不生效
![[Pasted image 20240129103848.png]]
默认 `Tomcat`配置生效。并且给容器中放 TomcatServletWebServerFactory
![[Pasted image 20240129103800.png]]
### TomcatServletWebServerFactory
TomcatServletWebServerFactory工厂通过getWebServer方法，来创建tomcatweb服务器
```java
@Override  
public WebServer getWebServer(ServletContextInitializer... initializers) {  
    if (this.disableMBeanRegistry) {  
       Registry.disableRegistry();  
    }  
    Tomcat tomcat = new Tomcat();  
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");  
    tomcat.setBaseDir(baseDir.getAbsolutePath());  
    for (LifecycleListener listener : this.serverLifecycleListeners) {  
       tomcat.getServer().addLifecycleListener(listener);  
    }  
    Connector connector = new Connector(this.protocol);  
    connector.setThrowOnFailure(true);  
    tomcat.getService().addConnector(connector);  
    customizeConnector(connector);  
    tomcat.setConnector(connector);  
    tomcat.getHost().setAutoDeploy(false);  
    configureEngine(tomcat.getEngine());  
    for (Connector additionalConnector : this.additionalTomcatConnectors) {  
       tomcat.getService().addConnector(additionalConnector);  
    }  
    prepareContext(tomcat.getHost(), initializers);  
    return getTomcatWebServer(tomcat);  
}
```

在getWebServer处打上断点，debug启动SpringBoot
# WebServer的创建时机：

从TomcatServletWebServerFactory的getWebServer方法处回溯Debug栈
## Spring子容器：ServletWebServerApplicationContext
```java
private void createWebServer() {  
    WebServer webServer = this.webServer;  
    ServletContext servletContext = getServletContext();  
    if (webServer == null && servletContext == null) {  
       StartupStep createWebServer = getApplicationStartup().start("spring.boot.webserver.create");  
       ServletWebServerFactory factory = getWebServerFactory();  
       createWebServer.tag("factory", factory.getClass().toString());  
       this.webServer = factory.getWebServer(getSelfInitializer());  
       createWebServer.end();  
       getBeanFactory().registerSingleton("webServerGracefulShutdown",  
             new WebServerGracefulShutdownLifecycle(this.webServer));  
       getBeanFactory().registerSingleton("webServerStartStop",  
             new WebServerStartStopLifecycle(this, this.webServer));  
    }  
    else if (servletContext != null) {  
       try {  
          getSelfInitializer().onStartup(servletContext);  
       }  
       catch (ServletException ex) {  
          throw new ApplicationContextException("Cannot initialize servlet context", ex);  
       }  
    }  
    initPropertySources();  
}
```

继续跟踪createWebServer，来到ServletWebServerApplicationContext的onRefresh方法：
ServletWebServerApplicationContext是AbstractApplicationContext的子类，重写的onRefresh如下
```java
@Override  
protected void onRefresh() {  
    super.onRefresh();  
    try {  
       createWebServer();  
    }  
    catch (Throwable ex) {  
       throw new ApplicationContextException("Unable to start web server", ex);  
    }  
}
```
super.onRefresh()正式调用的父类AbstractApplicationContext的onRefresh() 
```java
refresh(){
......
	//容器刷新 十二大步中的第九步，刷新子容器会调用 `onRefresh()`// Initialize other special beans in specific context subclasses.
	onRefresh() 
......
}
```

# 切换服务器

切换服务器；
```xml
<properties>
    <servlet-api.version>3.1.0</servlet-api.version>
</properties>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- Exclude the Tomcat dependency -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- Use Jetty instead -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
