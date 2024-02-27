# @SpringBootApplication入手

@SpringBootApplication是一个3合一注解：

```Java
//@SpringBootApplication
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan("com.atguigu.boot")
public class MainApplication {
...
}
```

## @SpringBootConfiguration

```Java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.springframework.boot;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.AliasFor;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration//代表当前是一个配置类
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```

## @ComponentScan

指定扫描哪些包，Spring注解；

## @EnableAutoConfiguration

```Java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```

### 导入主程序类所在的包及其子包@AutoConfigurationPackage

翻译过来叫自动配置包？指定了默认的包规则，看源码

```Java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {}
```

它导入了一个Registrar，这是个内部类，点进去看源码，debug调试
![[Pasted image 20240129100329.png]]

利用Registrar给容器中导入一系列组件，将指定的一个包`com.atguigu.boot`下的所有组件导入进来

默认是主程序类MainApplication 所在包下。
![[Pasted image 20240129100337.png]]

### 导入spring.factories当中的SPI配置类@Import(AutoConfigurationImportSelector.class)
- selectImports
	- 利用getAutoConfigurationEntry(annotationMetadata);给容器中批量导入一些组件
	    - 调用`List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)`获取到所有需要导入到容器中的配置类 130个
	        - 利用工厂加载 `Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader)`；得到所有的组件
最终是从META-INF/spring.factories位置来加载一个文件。 默认扫描我们当前系统里面所有META-INF/spring.factories位置的文件 ，META-INF/spring.factories==（SPI文件）==，位于spring-boot-autoconfigure-2.3.4.RELEASE.jar包当中META-INF/spring.factories


下面开始梳理流程，点进AutoConfigurationImportSelector去看源码selectImports方法
```Java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
	}
	AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
	return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

getAutoConfigurationEntry方法

```Java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = getConfigurationClassFilter().filter(configurations);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```

getCandidateConfigurations方法

```Java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
					getBeanClassLoader());
	Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
					+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}
```

loadFactoryNames方法

```Java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
	String factoryTypeName = factoryType.getName();
	return (List)loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

loadSpringFactories方法

```Java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
	MultiValueMap<String, String> result = cache.get(classLoader);
	if (result != null) {
			return result;
	}

	try {
		Enumeration<URL> urls = (classLoader != null ?
						classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
						ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		result = new LinkedMultiValueMap<>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			UrlResource resource = new UrlResource(url);
			Properties properties = PropertiesLoaderUtils.loadProperties(resource);
			for (Map.Entry<?, ?> entry : properties.entrySet()) {
				String factoryTypeName = ((String) entry.getKey()).trim();
				for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
					result.add(factoryTypeName, factoryImplementationName.trim());
				}
			}
		}
		cache.put(classLoader, result);
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load factories from location [" +
						FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}
```

加载了一个配置文件叫"META-INF/spring.factories"
![[Pasted image 20240129100643.png]]

在这一步拿到所有的配置，返回上层我们看一共有130个
![[Pasted image 20240129100649.png]]

默认扫描我们当前系统里面所有spring.factories位置的文件，spring.factories位于spring-boot-autoconfigure-2.3.4.RELEASE.jar包当中META-INF/spring.factories，spring.factories文件里面写死了spring-boot一启动就要给容器中加载的所有配置类。这其实是SPI机制：
- 主人定义一系列的接口或抽象类，
- 然后各大厂商（starter）通过在classpath（META-INF/spring.factories）中声明该实现类
- 最后主人引入各大厂商（starter），来实现对组件的动态发现和加载。
![[Pasted image 20240129100749.png]]==需要注意的是，全场景的自动配置都在 `spring-boot-autoconfigure`这个包，在这个包的org目录中已经把所有的场景的自动配置类都写好了，但是这些实现的自动配置类都是条件生效的，因此默认不是全都开启的，只有导入哪个场景就开启哪个场景相关的自动配置==，比如我们没有引入batch批处理场景spring-boot-starter-batch，batch相关不会生效
![[Pasted image 20240131112937.png]]
# 以web场景为例分析自动配置

![[Pasted image 20240129121329.png]]
每一个场景可不止一个自动配置类，比如引入web场景spring-boot-starter-web，spring.factories中以下14个web相关的自动配置类就会生效
![[Pasted image 20240129100800.png]]
下面我就挑其中的两个自动配置类做个分析：
- DispatcherServletAutoConfiguration
- HttpEncodingAutoConfiguration
## DispatcherServletAutoConfiguration
这个自动配置类它配置了一个Bean叫做MultipartResolver
![[Pasted image 20240129100959.png]]

源码解读：以条件注入的方式给容器中添加了文件上传解析器；并且返回的resolver叫multipartResolver，因为我们的bean方法名叫multipartResolver，当于强制的给用户配置的视图解析器重命名为multipartResolver

```Java
    @Bean
    @ConditionalOnBean(MultipartResolver.class)  //容器中有这个类型组件时@Bean生效
    @ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME) //容器中没有这个名字 multipartResolver 的组件时@Bean生效
    //给@Bean标注的方法传入了对象参数，这个参数的值就会从容器中找。
    public MultipartResolver multipartResolver(MultipartResolver resolver) { 
      //SpringMVC multipartResolver。防止有些用户配置的文件上传解析器名字不叫multipartResolver，不符合规范
      // Detect if the user has created a MultipartResolver but named it incorrectly
      return resolver;
    }
```

## HttpEncodingAutoConfiguration
`@EnableConfigurationProperties(ServerProperties.class)`标注在配置类上
![[Pasted image 20240129101057.png]]

`@ConfigurationProperties(prefix = "server")`标注在参数类上
因此用户能够根据`prefix` 去yaml/properties中对应字段的配置
![[Pasted image 20240129101104.png]]

# 用户如何定制化组件

总结一下spring.factories中的自动配置类是怎么生效的：
- SpringBoot先加载所有场景的所有自动配置类 xxxxxAutoConfiguration
- 每个自动配置类是个配置类@Configuration
- 每个自动配置类按照条件进行生效@ConditionalOnXXXX
- 生效的配置类就会给容器中装配很多组件@Bean，容器放了这些组件，相当于这些功能就有了
- 更激动人心的是，自动配置类会为我们绑定参数配置文件，
    - 类注解`@EnableConfigurationProperties(ServerProperties.class)`
    - 参数注解`@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)`

ConditionalOnMissingBean意味着Spring的一种设计模式条件装载，因此，SpringBoot默认会在底层配好所有的组件。但是如果用户自己配置了以用户的优先。比如`HttpEncodingAutoConfiguration`中自带了默认的`CharacterEncodingFilter`组件。用户可以直接自己@Bean替换底层Spring默认的组件，
![[Pasted image 20240129101808.png]]
# springboot3版本更新

注：springboot3不再使用spring.factories去声明自动配置类了，取而代之的是META-INF/spring目录下的org.springframework.boot.autoconfigure.AutoConfiguration.imports。
![[Pasted image 20240129101802.png]]
spring.factories仍做保留，META-INF目录下spring.factories只用来声明监听器



