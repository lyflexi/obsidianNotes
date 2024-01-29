SpringBoot 已经默认配置好了Web开发场景常用功能。我们直接使用即可。

我们还可以手动实现WebMvcConfigurer或者配置WebMvcConfigurer，去改写mvc的默认规则
```java
package org.lyflexi.debug_springboot.takeovermvc;  
  

//修改mvc的默认配置，手自一体化mvc  

@Configuration //这是一个配置类,给容器中放一个 WebMvcConfigurer 组件，就能自定义底层  
public class MyWebMvcConfigurer  /*implements WebMvcConfigurer*/ {  
  
    @Bean  
    public WebMvcConfigurer webMvcConfigurer(){  
        return new WebMvcConfigurer() {  
            @Override 
            //配置静态资源  
            public void addResourceHandlers(ResourceHandlerRegistry registry) {  
                registry.addResourceHandler("/static/**")  
                        .addResourceLocations("classpath:/a/", "classpath:/b/")  
                        .setCacheControl(CacheControl.maxAge(1180, TimeUnit.SECONDS));  
            }  
  
            @Override 
            //配置拦截器  
            public void addInterceptors(InterceptorRegistry registry) {  
  
            }  
  
            @Override 
            //系统提供默认的MessageConverter 功能有限，仅用于返回json或者xml数据。
            //额外增加新的内容协商功能比如yaml，必须增加新的`HttpMessageConverter`
            public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {  
                converters.add(new MyYamlHttpMessageConverter());  
            }  
        };  
    }  
    class MyYamlHttpMessageConverter extends AbstractHttpMessageConverter<Object> {  
  
        private ObjectMapper objectMapper = null; //把对象转成yaml  
  
        public MyYamlHttpMessageConverter(){  
            //告诉SpringBoot这个MessageConverter支持哪种媒体类型  //媒体类型  
            super(new MediaType("text", "yaml", Charset.forName("UTF-8")));  
            YAMLFactory factory = new YAMLFactory()  
                    .disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER);  
            this.objectMapper = new ObjectMapper(factory);  
        }  
  
        @Override  
        protected boolean supports(Class<?> clazz) {  
            //只要是对象类型，不是基本类型  
            return true;  
        }  
  
        @Override  //@RequestBody  
        protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {  
            return null;  
        }  
  
        @Override //@ResponseBody 把对象怎么写出去  
        protected void writeInternal(Object methodReturnValue, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {  
  
            //try-with写法，自动关流  
            try(OutputStream os = outputMessage.getBody()){  
                this.objectMapper.writeValue(os,methodReturnValue);  
            }  
  
        }  
    }  
}
```
我们还可以配置`WebMvcConfigurer`并标注  `@EnableWebMvc`表示禁用默认配置
```java
package org.lyflexi.debug_springboot.takeovermvc;  
  
import org.springframework.context.annotation.Configuration;  
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;  
  

@EnableWebMvc  //禁用mvc的默认功能，全手动mvc  
@Configuration  
public class MyEnableWebMvcConfigurer implements WebMvcConfigurer {  
  
}
```


因此我们有以下三种接管方式

| 方式 | 用法 | 效果 |
| ---- | ---- | ---- |
| **全自动** | 直接编写控制器逻辑 | 全部使用**自动配置默认效果** |
| **手自一体** | 配置`WebMvcConfigurer | **保留自动配置效果**  <br>**手动设置mvc部分功能**   |
| **全手动** | 配置`WebMvcConfigurer`并标注  <br>`@EnableWebMvc`表示禁用默认配置 | **禁用自动配置效果**  <br>**全手动设置** |

# WebMvcAutoConfiguration到底自动配置了哪些默认行为

`WebMvcAutoConfiguration`是web场景下面的mvc自动配置类，该自动配置场景给我们配置了很多默认行为
## 首先，配置两个filter

1. 支持RESTful的filter：HiddenHttpMethodFilter
2. 支持非POST请求，请求体携带数据：FormContentFilter
```java
@Bean  
@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)  
@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled")  
public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {  
    return new OrderedHiddenHttpMethodFilter();  
}  
  
@Bean  
@ConditionalOnMissingBean(FormContentFilter.class)  
@ConditionalOnProperty(prefix = "spring.mvc.formcontent.filter", name = "enabled", matchIfMissing = true)  
public OrderedFormContentFilter formContentFilter() {  
    return new OrderedFormContentFilter();  
}
```
## 然后，导入EnableWebMvcConfiguration

导入`EnableWebMvcConfiguration`，配置如下默认行为
1. `RequestMappingHandlerAdapter`
2. `WelcomePageHandlerMapping`： 欢迎页功能支持（模板引擎目录、静态资源目录放index.html），项目访问/ 就默认展示这个页面.
3. `RequestMappingHandlerMapping`：找每个请求由谁处理的映射关系
4. `ExceptionHandlerExceptionResolver`：默认的异常解析器
5. `LocaleResolver`：国际化解析器
6. `ThemeResolver`：主题解析器
7. `FlashMapManager`：临时数据共享
8. `FormattingConversionService`： 数据格式化 、类型转化
9. `Validator`： 数据校验`JSR303`提供的数据校验功能
10. `WebBindingInitializer`：请求参数的封装与绑定
11. `ContentNegotiationManager`：内容协商管理器
## 最后，配置出默认的WebMvcConfigurer行为
WebMvcAutoConfigurationAdapter配置生效，它是一个`WebMvcConfigurer`，定义扩展SpringMVC底层功能
![[Pasted image 20240129113704.png]]
1. 定义好 `WebMvcConfigurer` **底层组件默认功能；所有功能详见列表**
2. 视图解析器：`InternalResourceViewResolver`
3. 视图解析器：`BeanNameViewResolver`，此时视图名（controller方法的返回值字符串）就是组件名
4. 内容协商解析器：`ContentNegotiatingViewResolver`
5. 请求上下文过滤器：`RequestContextFilter`: 任意位置直接获取当前请求
6. 静态资源链规则
7. `ProblemDetailsExceptionHandler`：错误详情，SpringMVC内部场景异常被它捕获
### WebMvcConfigurer 功能列表

|   |   |   |   |
|---|---|---|---|
|提供方法|核心参数|功能|默认|
|addFormatters|FormatterRegistry|**格式化器**：支持属性上@NumberFormat和@DatetimeFormat的数据类型转换|GenericConversionService|
|getValidator|无|**数据校验**：校验 Controller 上使用@Valid标注的参数合法性。需要导入starter-validator|无|
|addInterceptors|InterceptorRegistry|**拦截器**：拦截收到的所有请求|无|
|configureContentNegotiation|ContentNegotiationConfigurer|**内容协商**：支持多种数据格式返回。需要配合支持这种类型的HttpMessageConverter|支持 json|
|configureMessageConverters|`List<HttpMessageConverter<?>>` |**消息转换器**：标注@ResponseBody的返回值会利用MessageConverter直接写出去|8 个，支持byte，string,multipart,resource，json|
|addViewControllers|ViewControllerRegistry|**视图映射**：直接将请求路径与物理视图映射。用于无 java 业务逻辑的直接视图页渲染|无  `<br><mvc:view-controller>`|
|configureViewResolvers|ViewResolverRegistry|**视图解析器**：逻辑视图转为物理视图|ViewResolverComposite|
|addResourceHandlers|ResourceHandlerRegistry|**静态资源处理**：静态资源路径映射、缓存控制|ResourceHandlerRegistry|
|configureDefaultServletHandling|DefaultServletHandlerConfigurer|**默认 Servlet**：可以覆盖 Tomcat 的DefaultServlet。让DispatcherServlet拦截/|无|
|configurePathMatch|PathMatchConfigurer|**路径匹配**：自定义 URL 路径匹配。可以自动为所有路径加上指定前缀，比如 /api|无|
|configureAsyncSupport|AsyncSupportConfigurer|**异步支持**：|TaskExecutionAutoConfiguration|
|addCorsMappings|CorsRegistry|**跨域**：|无|
|addArgumentResolvers|List<HandlerMethodArgumentResolver>|**参数解析器**：|mvc 默认提供|
|addReturnValueHandlers|List<HandlerMethodReturnValueHandler>|**返回值解析器**：|mvc 默认提供|
|configureHandlerExceptionResolvers|List<HandlerExceptionResolver>|**异常处理器**：|默认 3 个  <br>ExceptionHandlerExceptionResolver  <br>ResponseStatusExceptionResolver  <br>DefaultHandlerExceptionResolver|
|getMessageCodesResolver|无|**消息码解析器**：国际化使用|无|


# @EnableWebMvc 全面禁用了默认行为

`@EnableWebMvc`给容器中导入 `DelegatingWebMvcConfiguration`组件，DelegatingWebMvcConfiguration他是 WebMvcConfigurationSupport
```java
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.TYPE)  
@Documented  
@Import(DelegatingWebMvcConfiguration.class)  
public @interface EnableWebMvc {  
}
```

mvc的自动配置类`WebMvcAutoConfiguration`有一个核心的条件注解, `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`

因此@EnableWebMvc 导入 `WebMvcConfigurationSupport` 导致 `WebMvcAutoConfiguration` 失效。所以禁用了默认行为
