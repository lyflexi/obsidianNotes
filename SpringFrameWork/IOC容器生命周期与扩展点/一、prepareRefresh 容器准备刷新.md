我们先来看一下如下的一个单元测试类（例如IOCTest_Ext）。

```Java
package com.meimeixia.test;

import org.junit.Test;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import com.meimeixia.ext.ExtConfig;

public class IOCTest_Ext {
        
        @Test
        public void test01() {
                AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ExtConfig.class);
                
                // 发布一个事件
                applicationContext.publishEvent(new ApplicationEvent(new String("我发布的事件")) {
                });
                
                // 关闭容器
                applicationContext.close();
        }

}
```

我们知道如下这样一行代码是来new一个IOC容器的，而且还可以看到传入了一个配置类。

```Java
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ExtConfig.class);
```

我们不妨点进去AnnotationConfigApplicationContext类的有参构造方法里面去看一看，我们将核心的关注点放在refresh方法上，也即刷新容器
![[Pasted image 20231226175401.png]]

接下来，我们在刷新容器的方法上打上一个断点，如下图所示，重点分析一下刷新容器这个方法里面到底做了些什么事。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YTlhYmI2OGFlMTUzMTA5MTFhNzMwZjU0OWI0NzE3YWZfeFBaYmVQRU9CMjFLdldWaHBOcjh2M0NCVGFxR01sU3hfVG9rZW46RTFSbGJmb3B3b0JUMTR4M3A4cWMzZnJBbmVnXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

我们以debug的方式运行IOCTest_Ext测试类中的test01方法，如下图所示，程序现在停到了标注断点的refresh方法处。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YjJkOTg5MTQ5NmIzNmU5MjU2MWUzOTU2MDUzNGYyOWRfWVlaYzBOdndaTExMYUlGb0RTa0Ewc0Q5N0xsaFZBSnVfVG9rZW46RlczbGJXS1kzb09IMnF4NG52emM1NDQybnJkXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

# prepareRefresh刷新容器前的预处理工作

按下F5快捷键进入refresh方法里面，如下图所示，可以看到映入眼帘的是一个线程安全的锁机制，除此之外，你还能看到第一个方法，即prepareRefresh方法，顾名思义，它是来执行刷新容器前的预处理工作的。

那么问题来了，刷新容器前的这个预处理工作它到底都做了哪些事呢？下面我们就来详细说说。

## initPropertySources

按下F6快捷键让程序往下运行，运行到prepareRefresh方法处时，按下F5快捷键进入该方法里面，如下图所示，可以看到会先清理一些缓存，我们的关注点不在这儿，所以略过。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2I4YTFkMjljNzBhMTM4Y2IxODk0NTA4NzVjOTg0ZWZfRlhRbU12Sm5ha2pIZmlHaGVGZGIyUUFsZnp3RHBqTTdfVG9rZW46QUJSdmJyY0tSb29kTW94QXRBSmNJYkJTbkVkXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

继续按下F6快捷键让程序往下运行，运行到super.prepareRefresh()这行代码处，这儿也是来执行刷新容器前的预处理工作的。按下F5快捷键进入该方法里面，如下图所示，我们可以看到它里面都做了些什么预处理工作。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YWY2NzcwM2NjZWUyYmQ5ZGMzMTJjODllNmJmOTQ3NWRfQTFYRHhsZENqNm10ZXV5SEdsc0d1eUp1Tm1GbEdCTzZfVG9rZW46V0RVOGJ5SnhQb1dZZW54RDF0RWNpVDFJbmdXXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

发现就是先记录下当前时间，然后设置下当前容器是否是关闭、是否是活跃的等状态，除此之外，还会打印当前容器的刷新日志。

如果你要是不信的话，那么可以按下F6快捷键让程序往下运行，直至运行到initPropertySources方法处，你便能看到Eclipse控制台打印出了一些当前容器的刷新日志，如下图所示。

这时，我们看到了第一个方法，即initPropertySources方法。那么，它里面做了些啥事呢？

initPropertySources() 子类自定义个性化的属性设置的方法.

顾名思义，该方法是来初始化一些属性设置的。那么，该方法里面究竟做了些啥事呢？我们不妨进去一探究竟，按下F5快捷键进入该方法中，如下图所示，发现它是空的，没有做任何事情。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MmMzMThjZTY4YWZmYWFkNDMwNWVmZTdlNWRkNzMzMTJfNGZ5REVOR29JaXFpdThNcm1QdEFFYjgyNlRwaE1QMkdfVG9rZW46RTNNN2JyR2hob3BJVjd4UGsxS2M4YzcwbnFiXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

但是，我们要注意该方法是protected类型的，这意味着它是留给子类自定义个性化的属性设置的。例如，我们可以自己来写一个AnnotationConfigApplicationContext的子类，在容器刷新的时候，重写这个方法，这样，我们就可以在子类（也叫子容器）的该方法中自定义一些个性化的属性设置了。

  

这个方法只有在子类自定义的时候有用，只不过现在它还是空的，里面啥也没做。

## getEnvironment()

`validateRequiredProperties()`获取其环境变量，然后校验属性的合法性

继续按下F6快捷键让程序往下运行，直至运行到以下这行代码处。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MTZmN2E4MTBhOTAyMWZkZGM3NDdlNzQ4YTFiZWRiYjhfb3V5Y1UwNk94WWtJSE52VFF6ako5eEJhZmI3S3FkYnVfVG9rZW46UWRxZWJxVm5ab0VzVG14Uzl0R2NtR25LbjVnXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

这行代码的意思很容易知道，前面不是自定义了一些个性化的属性吗？这儿就是来校验这些属性的合法性的。

那么是怎么来进行属性校验的呢？首先是要来获取其环境变量，你可以按下F5快捷键进入getEnvironment方法中去看看，如下图所示，可以看到该方法就是用来获取其环境变量的。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MGIwNDk4MTE5YWVjZDY0MGFlNjdjMjFkZjI1NDU5YmJfMnFOTzVDQWF6d04xUEt1bE5GOHc1Y3hzaHh2cDBGWlZfVG9rZW46U2kxZGJ4NlBab3ViVmh4UTVQN2NHQjJPbnFlXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

继续按下F6快捷键让程序往下运行，让程序再次运行到getEnvironment().validateRequiredProperties()这行代码处。然后，再次按下F5快捷键进入validateRequiredProperties方法中去看看，如下图所示，可以看到就是使用属性解析器来进行属性校验的。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YWQ1MmJlOTkzZjczOWE1N2M4NmYxY2E2ZDkzY2M1NzBfeGUxWXpjVWZCOGV0UkdRb2JwQ2JjNDA4cXpkV2tZOU9fVG9rZW46TjVwSWJPOWJGb1owRUN4UDVOOGMzNGVQblZmXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

只不过，我们现在没有自定义什么属性，所以，此时并没有做任何属性校验工作。

## 保存容器中早期的事件

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MjM1ZTdmODEwOWIzZmU1NjYxYjJmNzUzMzc5OGExN2JfNXNsbDBMa3JkWDh3NFBQbWliUWVDdzBqVjRYdVZGYUNfVG9rZW46VVZuUGJnYUdDb3d3YTd4NDh4SGNnd1ljblhoXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

这儿是new了一个LinkedHashSet，它主要是来临时保存一些容器中早期的事件的。如果有事件发生，那么就存放在这个LinkedHashSet里面，这样，当事件派发器好了以后，直接用事件派发器把这些事件都派发出去。

  

总结一下就是一句话，即允许收集早期的容器事件，等待事件派发器可用之后，即可进行发布。

  

至此，我们就分析完了prepareRefresh方法，以上就是该方法所做的事情。我们发现这个方法和BeanFactory并没有太大关系，因此，接下来我们还得来看下一个方法，即obtainFreshBeanFactory方法。

# obtainFreshBeanFactory 获取BeanFactory

继续按下F6快捷键让程序往下运行，直至运行至以下这行代码处。

获取BeanFactory对象`obtainFreshBeanFactory`

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDVlNGQ5ZTg5ZWQyMGQ2MjNlZGU4Nzg3NTVlZTBmNGNfSFBpZTZ1enBXSWUxaEdRdEhldFNiU2s2emh2RXEzTVVfVG9rZW46UUZwZGJ6SEdIb1NlREp4a2NvUmM0UUlGbkxiXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

可以看到一个叫obtainFreshBeanFactory的方法，顾名思义，它是来获取BeanFactory的实例的。接下来，我们就来看看该方法里面究竟做了哪些事。

## refreshBeanFactory()

创建BeanFactory对象，并为其设置一个序列化id

按下F5快捷键进入该方法中，如下图所示，可以看到其获取BeanFactory实例的过程是下面这样子的。

发现首先调用了一个叫refreshBeanFactory的方法，该方法见名思义，应该是来刷新BeanFactory的。那么，该方法里面又做了哪些事呢？

## GenericApplicationContext

我们可以按下F5快捷键进入该方法中去看看，如下图所示，发现程序来到了GenericApplicationContext类里面。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YzZkOTkwYWI0MjM4YTgyYzcxM2Q4MTFiYzc2NTFlYTRfakdSbkRRekU3OFdBcHhCcndiYWdlaTRhbk5saUdoYWJfVG9rZW46T0NBcmI4WGtTbzhGMnp4Y3pCV2NmVzdMbktlXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

而且，我们还可以看到在以上refreshBeanFactory方法中，会先判断是不是重复刷新了。

## this.beanFactory.setSerializationId(getId())

于是，我们继续按下F6快捷键让程序往下运行，发现程序并没有进入到if判断语句中，而是来到了下面这行代码处。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=YmQxY2YxYjhjMjYzMWZmMTQyODlhOWM4NWEwM2M2NThfRFlMbGZLbWFSUGtLY1ZHZHIwSHlleEp0S3lKQkpXU1dfVG9rZW46QnhYUGJGUUR4b1pKQlN4Y3hIbWM4MGsxbmZnXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

程序运行到这里，你会不会有一个大大的疑问，那就是我们的beanFactory不是还没创建么，怎么在这儿又开始调用方法了呢，难道是已经创建了吗？

## new DefaultListableBeanFactory()

我们向上翻阅GenericApplicationContext类的代码，发现原来是在这个类的无参构造方法里面，就已经实例化了beanFactory这个对象。也就是说，在创建GenericApplicationContext对象时，无参构造器里面就new出来了beanFactory这个对象。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjkyZjBhZTJlYjczNzFiYTMxMWIxMTA3YjcwMjcxN2FfdmEza2VRdFRPUEJWQ3dNT1IwREpJTGNlc3BnSzBXRmlfVG9rZW46U1NadmJ1RGMwb3pUaEp4dlFXRGNYQXVCbjhnXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

相当于我们做了非常核心的一步，即创建了一个beanFactory对象，而且该对象还是DefaultListableBeanFactory类型的。

  

现在，我们已经知道了在GenericApplicationContext这个类的无参构造方法里面，就已经实例化了beanFactory这个对象。那么，你可能会有疑问，究竟是在什么地方调用GenericApplicationContext类的无参构造方法的呢？

  

这时，我们可以去看一下我们的单元测试类（例如IOCTest_Ext），如下图所示。

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=OGZkY2Y0NmQ5Y2NlYzM5YjlhMjc4OGJkNGMwYzJkYWZfTGdQWkhraXB1dldUb3lMekFzcmpVNVdGSkpOb0gyVGRfVG9rZW46SVV2Q2IxRER2b0w1Yk54YzQ4WGNXbk01bmllXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

只要点进去AnnotationConfigApplicationContext类里面去看一看，你就知道大概了，如下图所示，原来AnnotationConfigApplicationContext类继承了GenericApplicationContext这个类，所以，当我们实例化AnnotationConfigApplicationContext时就会调用其父类的构造方法，相应地这时就会对我们的BeanFactory进行实例化了。

  

这里是Spring当中第一种使父类生效方式：

父类是个类同时我们又实现了继承该父类的子类。

java中的一个简单继承问题：

父类: public class FatherClass{ public FatherClass(){ System.out.println("FatherClass Create"); } } 子类: public class ChildClass extends FatherClass{ public ChildClass(){ System.out.println("ChildClass Create"); } public static void main(String[] args){ FatherClass fc = new FatherClass(); ChildClass cc = new ChildClass(); } } 结果如下: FatherClass Create FatherClass Create ChildClass Create

剖析：

因为FatherClass fc = new FatherClass(); 所以输出第一个FatherClass Create，这个应该没有什么难理解的。 ChildClass cc = new ChildClass();这个时候去调用构造函数之前，因为有继承关系所以java会先将你要继承的那个类分配内存空间，所以FatherClass类的构造函数执行了一遍，这也就是第二个FatherClass Create。等需要继承的那个类已经有内存空间了以后，再执行ChildClass类的构造函数，输出ChildClass Create。相当于使抽象源码生效

---

TIPS:

Spring当中第二种父类生效方式，父类是个接口，即实现implements该接口的子类。第二种特性在springboot源码当中十分常见，相当于我们往容器当中注入了一个自定义的Bean。相当于使抽象源码生效

  

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=NDE5ODBmMWVjZWU1YTNmMDMxYWIzZDczMTc1ZGU3Y2ZfbUxEV2hwY0l4TENxVnJtRkZQSTRWM2U3aDgxa0FXSzdfVG9rZW46V2RTbWJKc1BSb2s5aFN4MDVJeGN5VnQ2bnFoXzE3MDM1ODQzMjg6MTcwMzU4NzkyOF9WNA)

this()调用

```Java
    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this();
        this.register(componentClasses);
        this.refresh();
    }
```

# prepareBeanFactory

# postProcessBeanFactory