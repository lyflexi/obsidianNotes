
classpath:beanlifecircle/applicationContext.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">  
<!-- 向Spring容器注入MyInstantiationAwareBeanPostProcessor --> 
    <bean class="org.lyflexi.debug_springframework.beanlifecircle.MyInstantiationAwareBeanPostProcessor" />  
<!-- 向Spring容器注入MyBeanPostProcessor --> 
    <bean class="org.lyflexi.debug_springframework.beanlifecircle.MyBeanPostProcessor" />  
<!-- 向Spring容器注入MyBeanFactoryPostProcessorOfBeanLifeTest --> 
    <bean class="org.lyflexi.debug_springframework.beanlifecircle.MyBeanFactoryPostProcessorOfBeanLifeTest" />  
  
</beans>
```

启动类（测试类）加载xml
```java
public class BeanLifeCycleTest {  
  
    public static void main(String[] args) {  
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:beanlifecircle/applicationContext.xml");  

        ((AbstractApplicationContext) applicationContext).close();  
    }  
}
```