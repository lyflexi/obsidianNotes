按类型自动注入可能会导致多个候选者，所以往往需要对选择过程进行针对的控制。

实现这一目标的方法之一是使用Spring的@Primary注解。@Primary表示如果在候选者中正好有一个主要Bean存在，它就会成为自动注入的值。

```Java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }

    @Bean
    public MovieCatalog secondMovieCatalog() { ... }

    // ...
}
```

通过前面的配置，下面的MovieRecommender被自动注入主Bean：firstMovieCatalog

```Java
public class MovieRecommender {

    @Autowired
    private MovieCatalog movieCatalog;

    // ...
}
```

