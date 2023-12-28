
@Primary是按类型使用自动注入主Bean的一种有效方式。当你需要对选择过程进行更多控制时，你可以使用Spring的@Qualifier注解。你可以将限定符的值与特定的参数联系起来，缩小类型匹配的范围，从而为每个注入参数选择一个特定的bean。如下面的例子所示：

# 定义Qualifier
容器中存在修饰词为`main`和`action`的两个`SimpleMovieCatalog`

```Java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/> (1)

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/> (2)

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```
# 依据Qualifier针对性依赖注入
## 方式一、原始注解
使用`@Qualifier`注解注入名字叫做的`main`的`SimpleMovieCatalog`

```Java
public class MovieRecommender {

    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog;

    // ...
}
```

你也可以在构造函数参数或方法参数上指定@Qualifier注解，如以下例子所示：

```Java
public class MovieRecommender {

    private final MovieCatalog movieCatalog;

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main") MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

## 方式二、自定义注解
你可以创建你自己的自定义限定符注解，你需要为你的注解打上@Qualifier注解，自定义限定符注解的默认字段叫做`value()`

```Java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

    String value();
}
```

然后你可以在自动注入的字段和参数上提供自定义限定词，如下例所示：

```Java
public class MovieRecommender {

    @Autowired
    @Genre("Action")
    private MovieCatalog actionCatalog;

    private MovieCatalog comedyCatalog;

    @Autowired
    public void setComedyCatalog(@Genre("Comedy") MovieCatalog comedyCatalog) {
        this.comedyCatalog = comedyCatalog;
    }

    // ...
}
```

自定义注解也可以指定多个属性值，如下面的例子所示：

```Java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MovieQualifier {

    String genre();

    Format format();
}
```

其中`Format`是一个枚举，定义如下：

```Java
public enum Format {
    VHS, DVD, BLURAY
}
```

要自动注入的字段用自定义限定符注解`@Qualifier`修饰，并包括两个属性的值：`genre`和`format`，如下面的例子所示：

```Java
public class MovieRecommender {

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Action")
    private MovieCatalog actionVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Comedy")
    private MovieCatalog comedyVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.DVD, genre="Action")
    private MovieCatalog actionDvdCatalog;

    @Autowired
    @MovieQualifier(format=Format.BLURAY, genre="Comedy")
    private MovieCatalog comedyBluRayCatalog;

    // ...
}
```