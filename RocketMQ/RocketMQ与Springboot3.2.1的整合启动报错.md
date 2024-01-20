依赖信息如下：
```xml
<parent>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-parent</artifactId>  
    <version>3.2.1</version>  
    <relativePath/> <!-- lookup parent from repository -->  
</parent>


<!--bootmq，最新版仅支持2.2.3-->
<dependency>  
    <groupId>org.apache.rocketmq</groupId>  
    <artifactId>rocketmq-spring-boot-starter</artifactId>  
    <version>2.2.3</version>  
</dependency>
```
报错信息如下：
```java
***************************
APPLICATION FAILED TO START
***************************

Description:

Field rocketMQTemplate in com.spt.message.service.MqProducerService required a bean of type 'org.apache.rocketmq.spring.core.RocketMQTemplate' that could not be found.

The injection point has the following annotations:
    - @org.springframework.beans.factory.annotation.Autowired(required=true)


Action:

Consider defining a bean of type 'org.apache.rocketmq.spring.core.RocketMQTemplate' in your configuration.
```
导致这个问题的原因是：

Springboot-3.0已经放弃了spring.plants自动装配，它被/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports所取代，但是rocketmq-spring-boot-starter还停留在过时的装配方式，打开rocketmq-spring-boot的jar一看便知
![[Pasted image 20240120144412.png]]

解决方案，或者说是临时性的兼容方案是，用户手动添加文件/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports，内容跟spring.factories的内容一样
![[Pasted image 20240120144607.png]]
