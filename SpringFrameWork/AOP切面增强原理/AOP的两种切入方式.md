AOP是指将某段代码“动态”的切入到“指定方法”的“指定位置”并且生效的一种编程方式，Spring利用动态代理对AOP做了增强
要想搭建AOP环境，首先需要在项目的pom.xml文件中引入AOP的依赖

```XML
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aspects</artifactId>
  <version>4.3.12.RELEASE</version>
</dependency>
```

AOP的实现有两种方式：

- 方法切入
    
- 注解切入
    

# 方法切入方式

在com.meimeixia.aop包下创建一个业务逻辑类，例如MathCalculator，用于处理数学计算上的一些逻辑。

现在，我们希望在这个业务逻辑类中的除法运算之前，记录一下日志，例如记录一下哪个方法运行了，用的参数是什么，运行结束之后它的返回值又是什么，顺便可以将其打印出来，还有如果运行出异常了，那么就捕获一下异常信息。

```Java
package com.meimeixia.aop;
public class MathCalculator {
        public int div(int i, int j) {
                System.out.println("MathCalculator...div...");
                return i / j;
        }
}
```

如果需要切面类来动态地感知目标类方法的运行情况，那么就需要使用Spring AOP中的一系列通知方法了。在com.meimeixia.aop包下创建一个切面类，例如LogAspects，在该切面类中定义几个打印日志的方法，以这些方法来动态地感知MathCalculator类中的div()方法的运行情况。

AOP中的通知方法及其对应的注解与含义如下：

- 前置通知（对应的注解是@Before）：在目标方法运行之前运行
    
- 后置通知（对应的注解是@After）：在目标方法运行结束之后运行，无论目标方法是正常结束还是异常结束都会执行
    
- 返回通知（对应的注解是@AfterReturning）：在目标方法正常返回之后运行
    
- 异常通知（对应的注解是@AfterThrowing）：在目标方法运行出现异常之后运行
    
- 环绕通知（对应的注解是@Around）：动态代理，我们可以直接手动推进目标方法运行（joinPoint.procced()）
    

切入点表达式指定在哪个方法切入

- `@Pointcut("execution(public int com.meimeixia.aop.MathCalculator.*(..))")`
    
- `@Pointcut("execution(public int com.meimeixia.aop.MathCalculator.div(int, int))`
    

```Java
package com.meimeixia.aop;

import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;

/**
 * 切面类
 * 
 * @Aspect：告诉Spring当前类是一个切面类，而不是一些其他普通的类
 * @author liuyan
 *
 */
@Aspect
public class LogAspects {
        
        // 如果切入点表达式都一样的情况下，那么我们可以抽取出一个公共的切入点表达式
        @Pointcut("execution(public int com.meimeixia.aop.MathCalculator.*(..))")
        public void pointCut() {}
        
        @Before("pointCut()")
        public void logStart(JoinPoint joinPoint) {
                Object[] args = joinPoint.getArgs(); // 拿到参数列表，即目标方法运行需要的参数列表
                System.out.println(joinPoint.getSignature().getName() + "@Before参数列表是：{" + Arrays.asList(args) + "}");
        }
        
        // 在目标方法（即div方法）结束时被调用
    @After("pointCut()")
        public void logEnd(JoinPoint joinPoint) {
                System.out.println(joinPoint.getSignature().getName() + "结束......@After");
        }
        
    /*
         * 如果方法正常返回，我们还想拿返回值，那么返回值又应该怎么拿呢？
         */
        @AfterReturning(value="pointCut()", returning="result") // returning来指定我们这个方法的参数谁来封装返回值
    // 一定要注意：JoinPoint这个参数要出现在参数列表的第一位，否则Spring也是无法识别的
        public void logReturn(JoinPoint joinPoint, Object result) { 
                System.out.println(joinPoint.getSignature().getName() + "正常返回@AfterReturning，结果是：{" + result + "}");
        }
        // 在目标方法（即div方法）出现异常，被调用
    //@After("com.meimeixia.aop.LogAspects.pointCut()")
    @AfterThrowing(value="pointCut()", throwing="exception")
    public void logException(JoinPoint joinPoint, Exception exception) {        
        System.out.println(joinPoint.getSignature().getName() + "出现异常......异常信息：{" + exception + "}");
    }
}
```

接下来需要将目标类和切面类加入到IOC容器。新建一个配置类，例如MainConfigOfAOP，并使用@Configuration注解标注这是一个Spring的配置类，同时使用@EnableAspectJAutoProxy注解开启AOP代理模式。在MainConfigOfAOP配置类中，使用@Bean注解将业务逻辑类和切面类都加入到IOC容器

```Java
package com.meimeixia.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

import com.meimeixia.aop.LogAspects;
import com.meimeixia.aop.MathCalculator;

/**
 * AOP：面向切面编程，其底层就是动态代理
 *      指在程序运行期间动态地将某段代码切入到指定方法指定位置进行运行的编程方式。
 *        
 * @author liayun
 *
 */
@EnableAspectJAutoProxy
@Configuration
public class MainConfigOfAOP {
        
        // 将业务逻辑类（目标方法所在类）加入到容器中
        @Bean
        public MathCalculator calculator() {
                return new MathCalculator();
        }
        
        // 将切面类加入到容器中
        @Bean
        public LogAspects logAspects() {
                return new LogAspects();
        }

}
```

最后创建测试类进行单元测试类

```Java
package com.meimeixia.test;

import org.junit.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import com.meimeixia.aop.MathCalculator;
import com.meimeixia.config.MainConfigOfAOP;

public class IOCTest_AOP {
        
        @Test
        public void test01() {
                AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfAOP.class);
                
                // 不要自己创建这个对象
                // MathCalculator mathCalculator = new MathCalculator();
                // mathCalculator.div(1, 1);
                
                // 我们要使用Spring容器中的组件
                MathCalculator mathCalculator = applicationContext.getBean(MathCalculator.class);
                mathCalculator.div(1, 1);
                
                // 关闭容器
                applicationContext.close();
        }

}
```

打印结果：
```java
div运行。。。@Before:参数列表是：{[1, 1]}
MathCalculator...div...
div正常返回。。。@AfterReturning:运行结果：{1}
div结束。。。@After

Process finished with exit code 0
```

接下来，我们就在MathCalculator类的div()方法中模拟抛出一个除零异常，来测试下异常情况，如下所示。

```Java
package com.meimeixia.test;

import org.junit.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import com.meimeixia.aop.MathCalculator;
import com.meimeixia.config.MainConfigOfAOP;

public class IOCTest_AOP {
        
        @Test
        public void test01() {
                AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfAOP.class);
                
                // 不要自己创建这个对象
                // MathCalculator mathCalculator = new MathCalculator();
                // mathCalculator.div(1, 1);
                
                // 我们要使用Spring容器中的组件
                MathCalculator mathCalculator = applicationContext.getBean(MathCalculator.class);
                mathCalculator.div(1, 0);
                
                // 关闭容器
                applicationContext.close();
        }

}
```

此时，我们运行以上test01()方法，AOP捕获到的异常信息如下
打印结果：
```java
div运行。。。@Before:参数列表是：{[1, 0]}
MathCalculator...div...
div异常。。。异常信息：{java.lang.ArithmeticException: / by zero}
div结束。。。@After

java.lang.ArithmeticException: / by zero
```


# 注解切入方式（定义顺序：切面>业务）

自定义注解`@interface AiPassiveMsg`

```Java
package com.iwhalecloud.aiFactory.common.aspect;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @description:
 * @author: liuyan
 * @create: 2022-04-24 18:16
 /
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

然后定义切面类，在注解驱动的AOP场景下，切入点表达式要指向自定义注解：`@Pointcut("@annotation(com.iwhalecloud.aiFactory.common.aspect.AiPassiveMsg)")`

```Java
package com.iwhalecloud.aiFactory.aspect;

import com.iwhalecloud.aiFactory.auth.CurrentUserHolder;
import com.iwhalecloud.aiFactory.common.aspect.AiPassiveMsg;
import com.iwhalecloud.aiFactory.common.util.LoggerUtil;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

/**
 * @author liuyan
 * @Description 消息中心埋点
 * @since 2022/3/14 14:30
 */
@Aspect
@Component
public class AiPassiveMsgAspect {

    //后续可以注入Serivce
    @Pointcut("@annotation(com.iwhalecloud.aiFactory.common.aspect.AiPassiveMsg)")
    public void cut() {
        LoggerUtil.info("被动消息切面切入");
    }

    @AfterReturning("cut()")
    public void addAiInformation(JoinPoint joinPoint) {
        MethodSignature methodSignature = (MethodSignature)joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        // 获取注解参数
        AiPassiveMsg annotation = method.getAnnotation(AiPassiveMsg.class);
        //解析注解
        String sceneType = annotation.sceneType();
        //TODO 后续可以调用service服务
        Long currentUserId = CurrentUserHolder.getCurrentUserId();
        Object[] args = joinPoint.getArgs();
    }
}
```

给目标业务类打上自定义注解`@AiPassiveMsg`

```Java

@Override
@AiPassiveMsg(sceneType = "audit.serviceVersion.deployment")
@Transactional(value = "inferenceTransactionManager", rollbackFor = Exception.class)
public AirServiceVersionInstanceResponse examineDeploymentInfo(AirServiceVersionInstanceRequest request) {
    ...
}
```