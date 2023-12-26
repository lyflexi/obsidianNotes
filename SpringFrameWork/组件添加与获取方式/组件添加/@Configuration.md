IOC即**控制反转**，也称为**依赖注入**，是指将**对象的创建**或者**依赖关系的引用**从具体的对象控制转为框架或者IOC容器来完成，也就是依赖对象的获得被反转了。可以简单理解为原来由我们来创建对象，现在由Spring来创建并控制对象。

# Xml 方式

依稀记得最早接触Spring的时候，用的还是SSH框架，不知道大家对这个还有印象吗？所有的Bean的注入得依靠Xml文件来完成。

它的注入方式分为：set方法注入、构造方法注入、字段注入

它的注入类型分为：值类型注入（8种基本数据类型）和引用类型注入（将依赖对象注入）

以下是set方法注入的简单样例

```xml
<bean name="teacher" class="org.springframework.demo.model.Teacher">
    <property name="name" value="阿Q"></property>
</bean>
```

对应的实体类代码

```Java
public class Teacher {
        private String name;
         public void setName(String name) {
                  this.name = name;
    }
}
```

Xml方式存在的缺点如下：

1. Xml文件配置起来比较麻烦，既要维护代码又要维护配置文件，开发效率低；项目中配置文件过多，维护起来比较困难；
    
2. 程序编译期间无法对配置项的正确性进行验证，只能在运行期发现并且出错之后不易排查；
    
3. 解析Xml时，无论是将Xml一次性装进内存，还是一行一行解析，都会占用内存资源，影响性能。
    

# @Configuration + @Bean

随着Spring的发展，Spring 2.5开始出现了一系列注解，除了我们经常使用的@Controller、@Service、@Repository、@Component 之外，还有一些其他比较常用实用技巧，比如当我们需要引入第三方的jar包时，可以用@Bean注解来标注，同时需要搭配@Configuration来使用。

- @Configuration用来声明一个配置类，可以理解为Xml的`<beans>`标签
    
- @Bean 用来声明一个Bean，将其加入到Spring容器中，可以理解为Xml的`<bean>`标签
    

```Java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<String, Object>();
        ......
        return redisTemplate;
    }
}
```