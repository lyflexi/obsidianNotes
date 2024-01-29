# 模板设计模式
DispatcherServlet用到了模板设计模式，模板设计模式主要为了代码复用：
- 父类已经实现的方法，子类可以复用
- 父类没有实现的方法，留给子类扩展
对于框架而言，我们向Spring中注册的一般是子类组件，则生效的就是子类组件，但是源码当中引用的是父类或者接口，这就是面向接口编程的精髓
![[Pasted image 20240108214816.png]]

# 策略模式Handler

## Spring MVC中的策略模式

在Spring MVC中，通过DispatcherServlet.properties配置文件，会把映射器HandlerMapping、适配器Adapter、视图解析器ViewReslover、异常处理器、文件处理器等等给初始化掉。

此后，DispatcherServlet 则能够根据请求信息调用 HandlerMapping，解析请求对应的 Handler（也就是我们平常说的 Controller 控制器）后做不同的处理。为什么要在 Spring MVC 中使用策略模式？ Spring MVC 中的 Controller 种类众多，不同类型的 Controller 通过不同的方法来对请求进行处理。如果不利用策略模式的话，DispatcherServlet 直接获取对应类型的 Controller，需要的自行来判断，像下面这段代码一样：if语句泛滥

```Java
if(mappedHandler.getHandler() instanceof MultiActionController){  
   ((MultiActionController)mappedHandler.getHandler()).xxx  
}else if(mappedHandler.getHandler() instanceof XXX){  
    ...  
}else if(...){  
   ...  
}  
```

假如我们再增加一个 Controller类型就要在上面代码中再加入一行 判断语句，这种形式就使得程序难以维护，也违反了设计模式中的开闭原则

设计模式中的开闭原则：对扩展开放，对修改关闭。

## 业务代码中的策略模式

我们在写代码的时候，常常会遇到不同的情况不同处理，通常情况下我们会使用if...else if ....else.... 但随着程序的不断扩展，可能if...else if 会越来越多，可维护性就会不断降低，而且代码的可读性也会越来越差，所以这里推荐大家使用策略模式。在这里以Java的项目来进行演示。
![[Pasted image 20240108215228.png]]

以订单的多状态处理来进行设计，提供 `IHander`类进行统一的handler的定义，和抽象`AbstractOrderStatusHandler`进行handler的默认的一些方法的定义（在抽象类这里首次实现方法，以便后面的具体实现类可以复用方法）

- OrderCommitStatusHandler（具体的实现handler）
    
- OrderPayStatusHandler（具体的实现handler）
    
- Order...StatusHandler（具体的实现handler）
    

通过Strategy进行控制，利用Map的特性根据handler的键获取对应的真正的处理handler实现的常用的策略模式