下载地址： [官网地址](http://tomcat.apache.org/).  
# tomcat环境设置
1、如图，大家可以任意选择版本进行安装，但是推荐使用Tomcat8版本及以上版本，但是也不用追求最新版本。这里为了给大家示范，我选择 **Tomcat10.1.7** 版本的，如图，单击即可。
![[Pasted image 20240109124709.png]]
![[Pasted image 20240109125114.png]]
配置环境变量，我喜欢把解压后的安装包直接放在Windows用户目录下：C:\Users\hasee\apache-tomcat-10.1.7
1. 新建环境变量CATALINA_HOME
```Java
C:\Users\hasee\apache-tomcat-10.1.7
```
2. 添加环境变量Path
```java
%CATALINA_HOME%\bin

%CATALINA_HOME%\lib

```

启动tomcat即可
![[Pasted image 20240109125519.png]]
启动成功：
![[Pasted image 20240109125535.png]]
访问http://localhost:8080/
![[Pasted image 20240109140918.png]]
# tomcat与IDEA2023.3.2集成
新建mavenmoudle：debug_servlet
- 指定jdk17，
- 指定org.apache.maven.archetypes:maven-archetype-webapp
创建出来的项目是没有java目录的，右键新建一个Java目录（默认作为Java根目录）
创建一个MyServlet.java类
```java
/**  
 * @Author: ly  
 * @Date: 2024/1/9 12:11  
 */  
  
  
import jakarta.servlet.ServletException;  
import jakarta.servlet.annotation.WebServlet;  
import jakarta.servlet.http.HttpServlet;  
import jakarta.servlet.http.HttpServletRequest;  
import jakarta.servlet.http.HttpServletResponse;  
  
import java.io.IOException;  
  
  
@WebServlet(name = "MyServlet", urlPatterns = "/myservlet")  
public class MyServlet extends HttpServlet {  
    @Override  
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {  
        System.out.println("doGet====================");  
    }  
  
    @Override  
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {  
        System.out.println("doPost====================");  
    }  
}
```
添加Servlet的依赖库，这里需要Servlet的版本注意区分！

Tomcat 10是第一个不再使用javax.servlet和相关包的版本。在Tomcat 10中，Servlet API已经迁移到了Jakarta EE命名空间（jakarta.servlet）。这是因为Java EE已经转移到了Eclipse基金会，并更名为Jakarta EE。因此，Servlet API也需要进行相应的更改。
```xml
    <!--tomcat 10+-->  
    <dependency>  
      <groupId>jakarta.servlet</groupId>  
      <artifactId>jakarta.servlet-api</artifactId>  
      <version>5.0.0</version>  
      <scope>provided</scope>  
    </dependency>    <!--tomcat 10之前版本-->  
<!--    <dependency>-->  
<!--      <groupId>javax.servlet</groupId>-->  
<!--      <artifactId>javax.servlet-api</artifactId>-->  
<!--      <version>4.0.1</version>-->  
<!--    </dependency>-->  
<!--    <dependency>-->  
<!--      <groupId>javax.servlet</groupId>-->  
<!--      <artifactId>javax.servlet-api</artifactId>-->  
<!--      <version>3.0.1</version>-->  
<!--    </dependency>-->
```
关联本地tomcat
![[Pasted image 20240109140747.png]]
Deployment部署方式设置成debug_servlet:war
Artifact将当前tomcat与指定的IDEAmoudle绑定
![[Pasted image 20240109141029.png]]注意：当添加完Artifact后，即servlet-test:war，在编辑configuration的server下的url，会自动给从http://localhost:8080变成http://localhost:8080/debug_servlet_war/，
![[Pasted image 20240109141048.png]]
所以最终访问路径就变成了http://localhost:8080/debug_servlet_war/myservlet
![[Pasted image 20240109141146.png]]