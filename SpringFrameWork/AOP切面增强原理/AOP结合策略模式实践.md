# 定义切面注解

定义切面注解`AiPassiveMsg`

```Java
package com.iwhalecloud.aiFactory.common.aspect;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @description:
 * @author: hasee
 * @create: 2022-04-24 18:16
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AiPassiveMsg {
    / 场景类型,具体场景和环节使用对应的类型前缀,用点拼接.系统=sys,安全=security,审核=audit.xxx.xxxx; */
    String sceneType() default "";

    /** 替换参数json字符串 */
    String param() default "";

    /** 用户id ["1001","1002","1003"] */
    String[] users() default {};
}
```

# 定义切面类

```Java
package com.iwhalecloud.aiFactory.aspect;

import com.iwhalecloud.aiFactory.aspect.msghandler.AiPassiveMsgHandler;
import com.iwhalecloud.aiFactory.aspect.msghandler.MsgHandlerContext;
import com.iwhalecloud.aiFactory.auth.CurrentUserHolder;
import com.iwhalecloud.aiFactory.common.aspect.AiPassiveMsg;
import com.iwhalecloud.aiFactory.common.util.LoggerUtil;
import com.iwhalecloud.aiFactory.system.service.api.dto.response.UserInfoDto;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.lang.reflect.Method;
import java.util.HashMap;

/**
 * @author hasee
 * @Description 消息中心埋点
 * @since 2022/3/14 14:30
 */
@Aspect
@Component
public class AiPassiveMsgAspect {


    @Autowired
    MsgHandlerContext msgHandlerContext;

    @Pointcut("@annotation(com.iwhalecloud.aiFactory.common.aspect.AiPassiveMsg)")
    public void cut() {
        LoggerUtil.info("被动消息切面切入");
    }

    @AfterReturning("cut()")
    public void addAiInformation(JoinPoint joinPoint) throws IOException {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        // 获取注解参数
        AiPassiveMsg annotation = method.getAnnotation(AiPassiveMsg.class);
        // 通过枚举值或者方法的调用
        String sceneType = annotation.sceneType();
        String[] users = annotation.users();


        UserInfoDto currentUser = CurrentUserHolder.getCurrentUser();
        Object[] joinPointArgs = joinPoint.getArgs();


        AiPassiveMsgHandler instance = msgHandlerContext.getInstance(sceneType);
        HashMap<String, Object> map = new HashMap<>();
        map.put("sceneType",sceneType);
        map.put("currentUser",currentUser);
        map.put("joinPointArgs",joinPointArgs);

        instance.handle(map);
    }


}
```

其中`@AfterReturning("cut()")`执行切入点动作

但是`MsgHandlerContext`是什么？

# 定义业务类

业务方法`applyPublish`

```Java
    @Override
    @Transactional(value = "inferenceTransactionManager", rollbackFor = Exception.class)
    @AiPassiveMsg(sceneType = "audit.serviceVersion.publish")
    public void applyPublish(AirServicePublishBeRequest request) {
    ...
    }
```

业务方法`approvePublish`

```Java
    @Override
    @Transactional("inferenceTransactionManager")
    @AiPassiveMsg(sceneType = "audit.serviceVersion.publish.audit")
    public void approvePublish(AirInferenceApprovedInfo request) {
    ...
    }
```

# 策略模式

打在业务方法上的`@AiPassiveMsg(sceneType = "audit.serviceVersion.publish")` 或者 `@AiPassiveMsg(sceneType = "audit.serviceVersion.publish.audit")`我们应该做出怎么样的处理呢？这里就要定义抽象处理器Handler以及它的子类具体实现类， 即 `AiPassiveMsgHandler`，`PublishServiceVersionMsgHandler`，`ApprovePublishServiceVersionMsgHandler`

## AiPassiveMsgHandler

![[Pasted image 20240106101345.png]]

`AiPassiveMsgHandler`

```Java
package com.iwhalecloud.aiFactory.aspect.msghandler;

import java.io.IOException;
import java.util.Map;

/**
 * @description:
 * @author: hasee
 * @create: 2022-04-25 09:15
 **/
public abstract class AiPassiveMsgHandler {
    public abstract Map<String, Object> handle(Map<String, Object> params) throws IOException;
}
```

`PublishServiceVersionMsgHandler`

```Java
package com.iwhalecloud.aiFactory.aspect.msghandler.process;

import com.iwhalecloud.aiFactory.aiResource.aimessage.MsgTmplService;
import com.iwhalecloud.aiFactory.aspect.msghandler.AiPassiveMsgHandler;
import com.iwhalecloud.aiFactory.aspect.msghandler.PassiveMsgHandlerType;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

/**
 * @description:
 * @author: hasee
 * @create: 2022-04-25 09:23
 **/
@Component
@PassiveMsgHandlerType("audit.serviceVersion.publish")
public class PublishServiceVersionMsgHandler extends AiPassiveMsgHandler {
    @Autowired
    MsgTmplService msgTmplService;
    @Override
    public Map<String, Object> handle(Map<String, Object> params) {
        try {
            msgTmplService.serviceVersionPublishAspect(params);
        }
        catch (RuntimeException e) {
        }

        return new HashMap<>();
    }
}
```

`ApprovePublishServiceVersionMsgHandler`

```Java
package com.iwhalecloud.aiFactory.aspect.msghandler.process;

import com.iwhalecloud.aiFactory.aiResource.aimessage.MsgTmplService;
import com.iwhalecloud.aiFactory.aspect.msghandler.AiPassiveMsgHandler;
import com.iwhalecloud.aiFactory.aspect.msghandler.PassiveMsgHandlerType;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;



/**
 * @description:
 * @author: hasee
 * @create: 2022-04-25 09:23
 **/
@Component
@PassiveMsgHandlerType("audit.serviceVersion.publish.audit")
public class ApprovePublishServiceVersionMsgHandler extends AiPassiveMsgHandler {
    @Autowired
    MsgTmplService msgTmplService;
    @Override
    public Map<String, Object> handle(Map<String, Object> params) {
        try {
            msgTmplService.serviceVersionAuditAspect(params);
        }
        catch (RuntimeException e) {
        }
        return new HashMap<>();
    }
}
```

## MsgHandlerProcessor注册器

但是`MsgHandlerContext`当中的`handlerMap`也就是我们具体的一个个的实际`hanlder`是何时加入到容器的呢？这里就需要定义实现了Bean工厂的`MsgHandlerProcessor`即注册器，实现`postProcessBeanFactory`方法。定义包扫描，扫描hanlder所在包当中打上`@PassiveMsgHandlerType`注解的`hanlder`。使具体的hanlder处理器`PublishServiceVersionMsgHandler`和`ApprovePublishServiceVersionMsgHandler`在程序启动的时候就加载到Spring容器当中。

`PublishServiceVersionMsgHandler`和`ApprovePublishServiceVersionMsgHandler`要打上自定义注解`@PassiveMsgHandlerType`，并且注解`@PassiveMsgHandlerType`的value值要与业务内`@AiPassiveMsg`注解的sceneType值保持匹配，因为注册阶段`MsgHandlerProcessor`会根据`@PassiveMsgHandlerType`的value值作为key进行hanlder的注册，加载阶段`MsgHandlerContext`会根据`@AiPassiveMsg`的sceneType值作为key进行获取。==`@PassiveMsgHandlerType`的`value`值和`@AiPassiveMsg`的`sceneType`值保持一致，能确保获取到的hanlder是符合预期的hanlder==

```Java
package com.iwhalecloud.aiFactory.aspect.msghandler;

import com.google.common.collect.Maps;
import com.iwhalecloud.aiFactory.common.aspect.AiPassiveMsg;
import com.iwhalecloud.aiFactory.common.util.ClassScaner;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * @description:
 * @author: hasee
 * @create: 2019-06-14 15:02
 */
 
@Component
public class MsgHandlerProcessor implements BeanFactoryPostProcessor {

    private static final String HANDLER_PACKAGE = "com.iwhalecloud.aiFactory.aspect.msghandler.process";

    /*
     * 扫描@PassiveMsgHandlerType，MsgHandlerContext，将其注册到spring容器
     *
     * @param beanFactory bean工厂
     */
     
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        Map<String, Class> handlerMap = Maps.newHashMapWithExpectedSize(3);
        //只扫描带有@PassiveMsgHandlerType注解的handler
        ClassScaner.scan(HANDLER_PACKAGE, PassiveMsgHandlerType.class).forEach(clazz -> {
            // 获取注解中的类型值
            String type = clazz.getAnnotation(PassiveMsgHandlerType.class).value();
            // 将注解中的类型值作为key，对应的类作为value，保存在Map中
            handlerMap.put(type, clazz);
            beanFactory.registerSingleton(clazz.getName(),clazz);
        });
        // 创建自己的HandlerContext，也将其注册到spring容器中，HandlerContext用于后续获取handler
        MsgHandlerContext context = new MsgHandlerContext(handlerMap);
        beanFactory.registerSingleton(MsgHandlerContext.class.getName(), context);
    }

}
```

### ClassScaner包扫描

实现了`ResourceLoaderAware`接口的`setResourceLoader(ResourceLoader var1)`方法

```Java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package org.springframework.context;

import org.springframework.beans.factory.Aware;
import org.springframework.core.io.ResourceLoader;

public interface ResourceLoaderAware extends Aware {
    void setResourceLoader(ResourceLoader var1);
}
```

`ClassScaner`：

```Java
package com.iwhalecloud.aiFactory.common.util;

import org.apache.commons.lang3.ArrayUtils;
import org.springframework.beans.factory.BeanDefinitionStoreException;
import org.springframework.context.ResourceLoaderAware;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.io.support.ResourcePatternResolver;
import org.springframework.core.io.support.ResourcePatternUtils;
import org.springframework.core.type.classreading.CachingMetadataReaderFactory;
import org.springframework.core.type.classreading.MetadataReader;
import org.springframework.core.type.classreading.MetadataReaderFactory;
import org.springframework.core.type.filter.AnnotationTypeFilter;
import org.springframework.core.type.filter.TypeFilter;
import org.springframework.util.StringUtils;
import org.springframework.util.SystemPropertyUtils;

import java.io.IOException;
import java.lang.annotation.Annotation;
import java.util.HashSet;
import java.util.LinkedList;
import java.util.List;
import java.util.Set;

/**
 * @description:
 * @author: hasee
 * @create: 2019-06-14 15:06
 **/
public class ClassScaner implements ResourceLoaderAware {

    private final List<TypeFilter> includeFilters = new LinkedList<TypeFilter>();
    private final List<TypeFilter> excludeFilters = new LinkedList<TypeFilter>();

    private ResourcePatternResolver resourcePatternResolver = new PathMatchingResourcePatternResolver();
    private MetadataReaderFactory metadataReaderFactory = new CachingMetadataReaderFactory(this.resourcePatternResolver);

    @SafeVarargs
    public static Set<Class<?>> scan(String[] basePackages, Class<? extends Annotation>... annotations) {
        ClassScaner cs = new ClassScaner();

        if (ArrayUtils.isNotEmpty(annotations)) {
            for (Class anno : annotations) {
                cs.addIncludeFilter(new AnnotationTypeFilter(anno));
            }
        }

        Set<Class<?>> classes = new HashSet<>();
        for (String s : basePackages) {
            classes.addAll(cs.doScan(s));
        }

        return classes;
    }

    @SafeVarargs
    public static Set<Class<?>> scan(String basePackages, Class<? extends Annotation>... annotations) {
        return ClassScaner.scan(StringUtils.tokenizeToStringArray(basePackages, ",; \t\n"), annotations);
    }

    public final ResourceLoader getResourceLoader() {
        return this.resourcePatternResolver;
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourcePatternResolver = ResourcePatternUtils
                .getResourcePatternResolver(resourceLoader);
        this.metadataReaderFactory = new CachingMetadataReaderFactory(
                resourceLoader);
    }

    public void addIncludeFilter(TypeFilter includeFilter) {
        this.includeFilters.add(includeFilter);
    }

    public void addExcludeFilter(TypeFilter excludeFilter) {
        this.excludeFilters.add(0, excludeFilter);
    }

    public void resetFilters(boolean useDefaultFilters) {
        this.includeFilters.clear();
        this.excludeFilters.clear();
    }

    public Set<Class<?>> doScan(String basePackage) {
        Set<Class<?>> classes = new HashSet<>();
        try {
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX
                    + org.springframework.util.ClassUtils
                    .convertClassNameToResourcePath(SystemPropertyUtils
                            .resolvePlaceholders(basePackage))
                    + "/**/*.class";
            Resource[] resources = this.resourcePatternResolver
                    .getResources(packageSearchPath);

            for (int i = 0; i < resources.length; i++) {
                Resource resource = resources[i];
                if (resource.isReadable()) {
                    MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
                    if ((includeFilters.isEmpty() && excludeFilters.isEmpty()) || matches(metadataReader)) {
                        try {
                            classes.add(Class.forName(metadataReader
                                    .getClassMetadata().getClassName()));
                        }
                        catch (ClassNotFoundException e) {
                            LoggerUtil.info(e.getMessage());
                        }
                    }
                }
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "I/O failure during classpath scanning", ex);
        }
        return classes;
    }

    protected boolean matches(MetadataReader metadataReader) throws IOException {
        for (TypeFilter tf : this.excludeFilters) {
            if (tf.match(metadataReader, this.metadataReaderFactory)) {
                return false;
            }
        }
        for (TypeFilter tf : this.includeFilters) {
            if (tf.match(metadataReader, this.metadataReaderFactory)) {
                return true;
            }
        }
        return false;
    }

}
```

### PassiveMsgHandlerType注解

`PassiveMsgHandlerType`注解，各个具体的`hanlder`实现类跟据`PassiveMsgHandlerType`标识，包扫描工具类就是根据此注解找到我们各个具体的`hanlder`实现类。

```Java
package com.iwhalecloud.aiFactory.aspect.msghandler;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface PassiveMsgHandlerType {
    String value();
}
```

## MsgHandlerContext加载器

怎么从容器中读取这些hanlder处理器呢？

这里就需要定义加载器，即上下文类`MsgHandlerContext`。

`AiPassiveMsgHandler instance = msgHandlerContext.getInstance(sceneType);`以`sceneType`名为`key`获取到hanlder的`clazz`对象

```Java
package com.iwhalecloud.aiFactory.aspect.msghandler;

import com.iwhalecloud.aiFactory.common.web.SpringContextUtil;

import java.util.Map;

/**
 * @description:
 * @author: hasee
 * @create: 2022-04-25 09:13
 **/
//单实例MsgHandlerContext
public class MsgHandlerContext {

	//里面也是单实例handlerMap
    private Map<String, Class> handlerMap;

    public MsgHandlerContext(Map<String, Class> handlerMap) {
        this.handlerMap = handlerMap;
    }

    public AiPassiveMsgHandler getInstance(String type) {
        Class clazz = handlerMap.get(type);
        if (clazz == null) {
            throw new IllegalArgumentException("not found handler for type: " + type);
        }

        return (AiPassiveMsgHandler) SpringContextUtil.getBean(clazz);
    }
}
```

# 策略模式优势

我们在写代码的时候，常常会遇到不同的情况不同处理，通常情况下我们会使用if...else if ....else.... 但随着程序的不断扩展，可能if...else if 会越来越多，可维护性就会不断降低，而且代码的可读性也会越来越差，所以这里推荐大家使用策略模式。在这里以Java的项目来进行演示。
![[Pasted image 20240106101438.png]]

以订单的多状态处理来进行设计：

- 提供 IHander类进行统一的handler的定义
    
- 抽象AbstractOrderStatusHandler进行handler的默认的一些方法的定义
    
- OrderCommitStatusHandler（具体的实现handler）
    
- OrderPayStatusHandler
    
- Order...StatusHandler
    
- 通过Strategy进行控制，利用Map的特性根据handler的键获取对应的真正的处理handler实现的常用的策略模式