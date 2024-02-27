
梳理官方strater的依赖关系，比如spring-boot-starter-web的核心是引入了spring-boot-starter，我们称之为stater的starter，最终才会有spring-boot-autoconfigure的自动配置能力
![[Pasted image 20240131113129.png]]
自动配置类MultipartAutoConfiguration中的yam参数配置最终都是映射到某个类上，这个类由@EnableConfigurationProperties(MultipartProperties.class)指定，如文件上传相关的MultipartProperties，这个类会在容器中创建对象。
![[Pasted image 20240131113644.png]]

==因此我们要想自定义starter，只需引入关键中间人spring-boot-starter即可==
官方starter命名规范：`spring-boot-starter-xxx
民间starter命名规范：`xxx-spring-boot-starter`

自定义starter要实现的效果：任何项目导入此`starter`都可以使用其中的Service，并且允许用户在application.properties中修改配置参数
# 1.基本抽取-firstlevel
1. 创建自定义starter项目debug_springboot_robotstarter，引入spring-boot-starter基础依赖，新版IDEA新建项目会自动帮我们引入
2. 编写业务组件功能，但是==其他项目的springboot默认扫描规则扫不到自定义starter中的业务组件（默认的扫描包路径不同），因此我们要在robotstarter中编写RobotAutoConfiguration手动向spring容器导入自己的业务组件RobotService==
3. 自定义starter的主程序类可以删除。
业务代码如下：
RobotProperties
```java
@ConfigurationProperties(prefix = "robot")  //此属性类和配置文件指定前缀绑定
@Component
@Data
public class RobotProperties {

    private String name;
    private String age;
    private String email;
}
```

RobotService
```java
package org.lyflexi.thirdlevel_robotstarter.service;  
  
  
import org.lyflexi.thirdlevel_robotstarter.properties.RobotProperties;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Service;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-27 19:58  
 */
@Service  
public class RobotService {  
  
    @Autowired  
    RobotProperties robotProperties;  
  
    public String sayHello(){  
        return "你好：名字：【"+robotProperties.getName()+"】;年龄：【"+robotProperties.getAge()+"】";  
    }  
}
```
RobotController
```java
package org.lyflexi.thirdlevel_robotstarter.controller;  
  
  
  
import org.lyflexi.thirdlevel_robotstarter.service.RobotService;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-27 20:02  
 */
@RestController  
public class RobotController {  
  
    @Autowired  
    RobotService robotService;  
  
    @GetMapping("/robot/hello")  
    public String sayHello(){  
        String s = robotService.sayHello();  
        return s;  
    }  
}
```
RobotAutoConfiguration
```java
package org.lyflexi;  
  
  
  
  
import org.lyflexi.thirdlevel_robotstarter.controller.RobotController;  
import org.lyflexi.thirdlevel_robotstarter.properties.RobotProperties;  
import org.lyflexi.thirdlevel_robotstarter.service.RobotService;  
import org.springframework.boot.context.properties.EnableConfigurationProperties;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.context.annotation.Import;  
  
/**  
 * @author lfy  
 * @Description  
 * @create 2023-04-27 20:15  
 */
//给容器中导入Robot功能要用的所有组件  
@Import({RobotService.class})  
//把组件导入到容器中，与@Import效果一样  
@EnableConfigurationProperties(RobotProperties.class)  
@Configuration  
public class RobotAutoConfiguration {  
  
    @Bean //把组件导入到容器中，与@Import效果一样  
    public RobotController robotController(){  
        return new RobotController();  
    }  
}
```


其他项目引用这个`starter`，直接导入这个 `RobotAutoConfiguration`，就能把这个场景的组件导入进来
```java
package org.lyflexi.debug_springboot;  
  
import org.springframework.boot.SpringApplication;  
import org.springframework.boot.autoconfigure.ImportAutoConfiguration;  
import org.springframework.boot.autoconfigure.SpringBootApplication;  
import org.springframework.context.annotation.Import;  
  
@Import(RobotAutoConfiguration.class)  
@SpringBootApplication  
  
public class DebugSpringbootApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(DebugSpringbootApplication.class, args);  
    }  
  
}
```
用户自定义application.properties配置参数
```shell
robot.name=i'm robot  
robot.age=19  
robot.email=haha@qq.com
```
测试starter的导入效果：浏览器访问http://localhost:8080/robot/hello
浏览器输出：
```shell
你好：名字：【i'm robot】;年龄：【19】
```

# 2.使用@EnableXxx机制-secondlevel
定义EnableRobot注解，该注解的本质也是把RobotAutoConfiguration放入spring容器中
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import(RobotAutoConfiguration.class)
public @interface EnableRobot {


}
```

其他项目引入`starter`不再需要手动指定RobotAutoConfiguration.class，只需开启这个注解即可 @EnableRobot
```java
package org.lyflexi.debug_springboot;  
  
import org.springframework.boot.SpringApplication;  
import org.springframework.boot.autoconfigure.ImportAutoConfiguration;  
import org.springframework.boot.autoconfigure.SpringBootApplication;  
import org.springframework.context.annotation.Import;  
@EnableRobot  
@SpringBootApplication  
  
public class DebugSpringbootApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(DebugSpringbootApplication.class, args);  
    }  
  
}
```
用户自定义application.properties配置参数
```shell
robot.name=i'm robot  
robot.age=19  
robot.email=haha@qq.com
```
测试starter的导入效果：浏览器访问http://localhost:8080/robot/hello
浏览器输出：
```shell
你好：名字：【i'm robot】;年龄：【19】
```

# 3.利用SpringBoot的SPI机制-secondlevel

依赖SpringBoot的SPI机制，自定义starter中创建META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 文件，spi文件指定我们自动配置类的全类名org.lyflexi.RobotAutoConfiguration即可
```shell
org.lyflexi.RobotAutoConfiguration
```
其他项目引入`starter`之后，完全自动化获得自定义starter的功能，任何配置都不需要，达到原生springboot-starter一样的效果
```java
package org.lyflexi.debug_springboot;  
  
import org.springframework.boot.SpringApplication;  
import org.springframework.boot.autoconfigure.ImportAutoConfiguration;  
import org.springframework.boot.autoconfigure.SpringBootApplication;  
import org.springframework.context.annotation.Import;  
  

@SpringBootApplication  
public class DebugSpringbootApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(DebugSpringbootApplication.class, args);  
    }  
  
}
```
用户自定义application.properties配置参数
```shell
robot.name=i'm robot  
robot.age=19  
robot.email=haha@qq.com
```
测试starter的导入效果：浏览器访问http://localhost:8080/robot/hello
浏览器输出：
```shell
你好：名字：【i'm robot】;年龄：【19】
```
