IOC即**控制反转**，也称为**依赖注入**，是指将**对象的创建**或者**依赖关系的引用**从具体的对象控制转为框架或者IOC容器来完成，也就是依赖对象的获得被反转了。可以简单理解为原来由我们来创建对象，现在由Spring来创建并控制对象。

xml的注入方式分为：set方法注入、构造方法注入

它的注入类型分为：值类型注入（8种基本数据类型）和引用类型注入（将依赖对象注入）

以下是set方法注入引用类型的简单样例：
```java
<bean name="userService" class="com.luban.service.UserService">
	<property name="orderService" ref="orderService"/>
</bean>

```

以下是构造方法注入引用类型的简单样例：
```java
<bean name="userService" class="com.luban.service.UserService">
	<constructor-arg index="0" ref="orderService"/>
</bean>

```
以下是注入基本类型的简单样例：
```java
    <bean id="userBean" class="org.lyflexi.debug_springframework.beanlifecircle.UserBean" init-method="myInit" destroy-method="myDestroy">  
        <!-- 构造函数依赖注入 -->  
        <constructor-arg index="0" type="java.lang.Integer" value="1"/>  
        <constructor-arg index="1" type="java.lang.Integer" value="18"/>  
  
        <!-- setter方法依赖注入 -->  
        <property name="id" value="2"/>  
        <property name="age" value="19"/>  
  
    </bean>  
```

Xml方式存在的缺点如下：

1. Xml文件配置起来比较麻烦，既要维护代码又要维护配置文件，开发效率低；项目中配置文件过多，维护起来比较困难；
    
2. 程序编译期间无法对配置项的正确性进行验证，只能在运行期发现并且出错之后不易排查；
    
3. 解析Xml时，无论是将Xml一次性装进内存，还是一行一行解析，都会占用内存资源，影响性能。