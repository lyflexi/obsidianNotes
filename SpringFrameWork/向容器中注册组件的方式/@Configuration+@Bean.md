
Spring 2.5开始出现了一系列注解，除了我们经常使用的@Controller、@Service、@Repository、@Component 之外，还有一些其他比较常用实用技巧，比如当我们需要引入第三方的jar包时，可以用@Bean注解来标注，同时需要搭配@Configuration来使用。

- @Configuration用来声明一个配置类，可以理解为Xml的`<beans>`标签
    
- @Bean 用来声明一个Bean，将其加入到Spring容器中，可以理解为Xml的`<bean>`标签
    
RedisConfig配置类如下：
```Java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<String, Object>();
        //这里可以根据传入的redisConnectionFactory自定义RedisTemplate的依赖注入规则哦
        ......
        return redisTemplate;
    }
}
```

启动类（测试类）加载RedisConfig.class
```java

  
  
public class BeanLifeCycleTest {  
  
    public static void main(String[] args) {  

        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(RedisConfig.class);  
  
        ((AbstractApplicationContext) applicationContext).close();  
    }  
}
```